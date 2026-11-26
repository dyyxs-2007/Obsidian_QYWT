# 第 2 层：Data 数据与网络架构层 (`Spotify/Data/`)

> [← 返回架构总控台](README.md) · [← 上一层：App 与容器层](01-App与容器架构层.md) · **当前位置**：区块一（层级架构） > 第 2 层 Data 数据层
> **按需跳转到重点实现逻辑**：[专题 ②：网络申请与多源并发聚合](02-网络申请与多源并发聚合.md) · [专题 ③：三级 LRU 缓存体系 (L3 元数据缓存)](03-三级LRU缓存体系实现.md) · [专题 ⑥：首页无限刷新与 slots 保序](06-首页无限刷新与渐进分页.md)

---

## 1. 本层局部架构图

`Spotify/Data/` 采用 **「Repository 门面 + DataSource 协议插座 + 本地持久化 Store」** 结构。UI 层永远只访问 `SPMusicRepository` 和 `SPLibraryStore` / `SPHistoryStore`，对底层是网易云 API 还是本地 Mock 完全无感知：

```mermaid
flowchart TD
    UI["所有 UI 控制器 & SPAudioCacheManager"] -->|唯一数据读取入口| Repo["SPMusicRepository (单例门面)<br/>同样实现 &lt;SPMusicDataSource&gt;"]

    subgraph L3Cache["Repository 内置 L3 LRU 内存缓存"]
        C1["playlistCache (50)"]
        C2["albumCache (50)"]
        C3["artistTracksCache (40)"]
        C4["searchCache (60)"]
        C5["lyricsCache (60)"]
    end

    Repo --> L3Cache
    Repo -->|缓存未命中时转发给 self.source| Proto["@protocol SPMusicDataSource<br/>(12个标准异步接口)"]

    Proto <|.. Netease["SPNeteaseMusicSource<br/>真实网易云 API 数据源"]
    Proto <|.. Mock["SPMockMusicSource<br/>本地 mock_data.json 数据源"]

    Netease --> Session["SPNeteaseSession<br/>用户Cookie(Keychain) + 12h游客Cookie<br/>pendingWaiters 并发防惊群"]
    Netease --> Parser["SPNeteaseParser<br/>防御性JSON解析 + 封面HTTPS降采样 + 播客合成"]
    Netease --> HTTP["SPHTTPClient (Core/Network)<br/>NSURLSession + timestamp防缓存"]

    UI -->|点赞/自建歌单/关注艺人| LibStore["SPLibraryStore<br/>Documents/sp_user_library_v1.json"]
    Repo -->|查本地自建歌单| LibStore
```

---

## 2. 本层核心协议与类职责拆解

### 2.1 `@protocol SPMusicDataSource` — 数据层核心插座
- **源码文件**：[SPMusicDataSource.h](file:///Users/dyyxs/项目/Spotify/Spotify/Data/SPMusicDataSource.h#L29-L72)
- **设计约定**：实现方可以在任意后台队列回调，由 `SPMusicRepository` 统一通过 `SP_MAIN_WORK` 宏切回主线程，并定义了统一错误域 `SPMusicErrorDomain`（含 `SPMusicErrorNoCopyright = 451` 无版权错误码）。
- **12 个协议接口全表**：
  1. `-sourceName`：返回数据源名称（如 `"网易云音乐 API (msapi.awup.cn)"` 或 `"Mock"`），供设置页展示。
  2. `-fetchHomeSectionsWithCompletion:` & `-fetchHomeSectionsForCategory:completion:`：获取首页指定分类（全部/音乐/播客）的首屏分区数组 `NSArray<SPHomeSection *> *`。
  3. `-fetchHomeMoreSectionsForPage:completion:` & `-fetchHomeMoreSectionsForCategory:page:completion:`：首页上滑无限刷新分页接口（`page` 从 1 开始，回调 `sections` + `BOOL hasMore`）。
  4. `-fetchSearchCategoriesWithCompletion:`：获取搜索页彩色分类卡片 `NSArray<SPSearchCategory *> *`。
  5. `-searchWithKeyword:completion:`：综合关键词搜索，返回包含 `tracks`/`artists`/`albums`/`playlists` 的 `SPSearchResult *`。
  6. `-fetchPlaylistWithId:completion:`：获取歌单（或 `dj:` 前缀的播客电台）详情及完整曲目。
  7. `-fetchAlbumWithId:completion:`：获取专辑详情及曲目列表。
  8. `-fetchArtistTopTracksWithId:completion:`：获取艺人信息与热门单曲。
  9. `-fetchLibraryItemsWithCompletion:`：获取“音乐库”页面的推荐歌单/专辑/艺人聚合列表。
  10. `-fetchPlayURLForTrack:completion:`：**播放前实时换取音频流 URL**（因网易云音频 URL 有 15~20 分钟防盗链时效，不能写死在模型里）。
  11. `-fetchLyricsForTrack:completion:`：获取解析好的双语同步歌词 `SPLyricDocument *`。

### 2.2 `SPMusicRepository` — 数据门面与 L3 缓存装饰器
- **源码文件**：[SPMusicRepository.h](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Repository/SPMusicRepository.h) / [SPMusicRepository.m](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Repository/SPMusicRepository.m#L29-L316)
- **实现的协议**：`<SPMusicDataSource>`
- **核心亮点**：
  - **热切换数据源** ([`-setSource:`](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Repository/SPMusicRepository.m#L50-L58))：当在设置页替换 `repo.source` 时，自动调用 `clearMetadataCaches` 清空 L3 缓存、调用 `[[SPAudioCacheManager shared] clearMemoryCaches]` 清空 L1 内存音频缓存（保留 L2 磁盘已下载的 mp3），并在主线程发出 `SPMusicSourceDidChangeNotification`，触发首页、搜索页、音乐库立即自动刷新。
  - **防异步串台保护**：每次转发请求前先用局部变量 `id<SPMusicDataSource> currentSource = self.source;` 捕获当前源，异步回调回来后校验 `if (weakSelf.source != currentSource) return;`，防止切换数据源瞬间旧源的慢请求污染新源缓存。
  - **首页自动预热详情缓存** ([`-warmCachesFromHomeSections:`](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Repository/SPMusicRepository.m#L148-L160))：首页加载出歌单/专辑后，若已包含曲目则直接写入 `playlistCache` / `albumCache`；详情页通过 [`-cachedPlaylistForId:`](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Repository/SPMusicRepository.m#L70-L78) 首帧同步读取。

### 2.3 `SPNeteaseMusicSource` / `SPNeteaseSession` / `SPNeteaseParser` — 网易云数据源三剑客
- **`SPNeteaseMusicSource`** ([SPNeteaseMusicSource.m](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseMusicSource.m#L15-L1042))：
  - 首页「全部 / 音乐 / 播客」三套 `dispatch_group_t` 多路并发请求聚合 ([L61-L369](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseMusicSource.m#L61-L369))。
  - 上滑加载更多分区的 `slots` 固定槽位并发收集 ([L376-L625](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseMusicSource.m#L376-L625))。
  - 综合搜索 `/search?type=1018` 失败自动降级 4 路 `/cloudsearch` ([L666-L748](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseMusicSource.m#L666-L748))。
  - 歌单详情并发请求 `/playlist/detail` + `/playlist/track/all?limit=50`，播客电台 (`dj:` 前缀 ID) 自动路由到 `/dj/detail` + `/dj/program` ([L752-L848](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseMusicSource.m#L752-L848))。
  - 播放地址 `/song/url/v1` 失败自动降级旧版 `/song/url?br=320000` ([L967-L1002](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseMusicSource.m#L967-L1002))。
- **`SPNeteaseSession`** ([SPNeteaseSession.m](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseSession.m#L25-L156))：
  - 用 Keychain 持久化 `userCookie` 与 `userId`，用 `NSUserDefaults` 持久化 12 小时有效期的游客 `anonymousCookie`。
  - [`-ensureCookieWithClient:completion:`](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseSession.m#L109-L149)：通过 `pendingWaiters` 回调队列 + `fetchingAnonCookie` 状态锁合并并发的 `/register/anonimous` 请求。
- **`SPNeteaseParser`** ([SPNeteaseParser.m](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseParser.m#L8-L334))：
  - [`+normalizedCoverURL:size:`](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseParser.m#L35-L46)：将 `http://` 强制升级为 `https://`，并为 `music.126.net` 域名自动追加 `?param=500y500` 服务端缩略图裁剪参数，大幅节省流量与解码内存。
  - [`+podcastRadioFromJSON:` & `+podcastEpisodeTrackFromJSON:fallbackRadio:`](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseParser.m#L189-L249)：将网易云播客电台和声音节目(`mainSong`)无缝适配为 `SPPlaylist`（带 `dj:` 前缀）和 `SPTrack`，从而直接复用歌单详情页与播放器，零额外 UI 分支！

### 2.4 `SPMockMusicSource` — 离线仿真数据源
- **源码文件**：[SPMockMusicSource.h](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Mock/SPMockMusicSource.h) / [SPMockMusicSource.m](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Mock/SPMockMusicSource.m#L21-L363)
- **核心机制**：在专用串行队列 `com.dyyxs.spotify.mock` 上懒加载 `Resources/MockData/mock_data.json`，通过 `YYModel` 解析出艺人、专辑、歌单与曲目，并利用 [`-resolveRefs:`](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Mock/SPMockMusicSource.m#L96-L107) 解析 JSON 中的 `{ "type": "playlist", "id": "pl_1" }` 引用关系，配合 `dispatch_after` 模拟 `0.4s` 真实网络延迟。

### 2.5 `SPLibraryStore` & `SPHistoryStore` — 本地个人数据持久化
- **`SPLibraryStore`** ([SPLibraryStore.m](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Repository/SPLibraryStore.m#L25-L344))：
  - 将「已点赞的歌曲 (`likedTracks`)」「用户自建歌单 (`userPlaylists`，ID 以 `user_pl_` 开头)」「已关注的艺人 (`followedArtists`)」通过 `YYModel` 序列化保存在 `Documents/sp_user_library_v1.json`。
  - 首次安装自动创建一个默认自建歌单「我的收藏歌单」([L75-L85](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Repository/SPLibraryStore.m#L75-L85))；任何增删操作自动发出 `SPLibraryDidUpdateNotification` 通知详情页、大播放页、音乐库同步刷新。
- **`SPHistoryStore`** ([SPHistoryStore.m](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Repository/SPHistoryStore.m#L17-L83))：
  - 在 `NSUserDefaults` 中维护最近播放的 50 首 `SPTrack`（重复播放同一首歌会先去重再插到 index 0），每次 `SPPlayerService` 成功解析并加载歌曲时自动调用 `[[SPHistoryStore shared] recordTrack:track]`。

---

> **下一层导航**：[前往第 3 层：Player 播放器内核架构层 →](03-Player播放器内核架构层.md)
