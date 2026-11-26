# SPMusicRepository

> 源码：`Spotify/Data/Repository/SPMusicRepository.h` / `SPMusicRepository.m`
> 阶段：阶段三 · 数据与播放心脏

## 核心笔记

### 切换数据源时的双层缓存清理策略（`setSource:`）
- **结论**：`clearMetadataCaches` 与 `SPAudioCacheManager.clearMemoryCaches` 并非重复清理，而是分别清理**业务元数据层**与**音频流热缓存层**两个独立管家的内存缓存，同时刻意保留磁盘已落盘的音频文件。
- **代码定位**：`SPMusicRepository.m:L52-L60`、`SPAudioCacheManager.m:L443-L446`
- **要点**：
  - `[self clearMetadataCaches]`：清空 `SPMusicRepository` 自身的 **UI/业务元数据缓存**（首页分区、歌单、专辑、歌手、搜索结果、歌词），确保切源后界面重拉新源数据、不串源。
  - `[[SPAudioCacheManager shared] clearMemoryCaches]`：清空音频缓存管家的 **L1 内存热缓存**（`resolvedURLCache` 临时播放直链 + `assetCache` 预热的 `AVURLAsset`），释放旧源预加载占用的内存并防止临时直链混用；但**不调用 `clearAllCaches`**，保留 L2 沙盒磁盘已下载的音频文件，切回原数据源时仍可离线秒播。

### 歌单查询中的“数据源路由” vs “多级缓存”（`SPLibraryStore` 与 `playlistCache`）
- **结论**：先查 `SPLibraryStore` 再查 `playlistCache` **不是一级/二级缓存关系**，而是**本地真实数据源（Store）与远程歌单内存缓存（L1 Cache）的业务分流路由**。
- **代码定位**：`SPMusicRepository.m:L72-L76`、`L216-L226`；`SPLibraryStore.m:L86-L90`、`L370-L394`
- **要点**：
  - **`SPLibraryStore` 是持久化数据源（Source of Truth），不是缓存**：文件落在 `Documents/` 目录，存的是用户本地自建歌单（`user_pl_*`）和“我喜欢的音乐”，服务器上根本不存在、绝不可被 LRU 淘汰（删了用户数据就真丢了）。
  - **`playlistCache` 才是真正的 L1 内存缓存**：存的是从网络数据源拉回来的公共/推荐歌单，丢了随时可重拉。
  - **统一入口分流**：详情页只认 `playlistId`，Repository 先拦截归属本地的歌单直接返回（避免拿本地 ID 去请求服务器报 404），不属于本地的才走 `playlistCache -> 网络数据源`。

### 异步请求防切源竞态模板与网络分层（`refreshHomeSectionsForCategory:`）
- **结论**：`SPMusicRepository` 本身不写任何 HTTP 代码（网络已下沉封装在 `SPHTTPClient` + `SPNeteaseMusicSource` 中），它只负责**数据源快照防竞态、主线程收敛与缓存写入/预热**。
- **代码定位**：`SPMusicRepository.m:L119-L135`、`L158-L170`
- **要点**：
  - **防切源串数据（快照比对）**：发起请求前先用局部变量 `currentSource = self.source` 拍下当前源快照；异步回调回来后先判断 `if (weakSelf.source != currentSource) return;`，若请求途中用户已切换了数据源，直接丢弃旧源慢请求结果，防止污染新源缓存与 UI。
  - **线程与锁**：`SP_MAIN_WORK` 统一把回调切回主线程；写入普通可变字典 `cachedHomeSectionsByCategory` 时用 `@synchronized` 加锁防多线程读写崩溃，随后调用 `warmCachesFromHomeSections:` 把首页里已含曲目的歌单/专辑预热进 LRU 缓存。
  - **为何不在类内再把这套模板抽成一个万能函数**：OC 缺乏优雅的块返回泛型，各个接口的 `completion` 参数签名不同（单对象、数组+分页布尔值、歌手+曲目双返回值），且写缓存逻辑各异；强行用 `id` 抹平类型会丧失编译期类型安全，保留 10 行显式模板更直观好调试。
