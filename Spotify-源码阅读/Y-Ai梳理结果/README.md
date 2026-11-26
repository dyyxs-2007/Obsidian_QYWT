# Spotify iOS 仿写项目 · 架构总控与核心实现按需拆解中心

> [!TIP]
> **双形态阅读指南（拒绝一股脑长文堆砌，支持按层级下钻与按路径跳转）**
> 1. **交互式双区块可视化看板**：可以在侧边栏直接打开交互式 Artifact，或在浏览器双击打开本目录下的 [Spotify架构总控与核心链路看板.html](file:///Users/dyyxs/项目/学习app-Spotify/Spotify架构总控与核心链路看板.html)（左栏看总控与层级架构，右栏看调用链路与源码级详解，支持双向联动定位）。
> 2. **模块化 Markdown 知识库（当前入口）**：本文档作为**全局总控台**，下方分为**「区块一：按层级进入子系统架构」**与**「区块二：重点实现路径与特殊场景详解」**两大分区，所有卡片均支持按需点击跳转，并附带直达 Xcode 源码具体行号的 `file://` 链接。

---

## 全局总控：五层架构与三大设计支柱

整个 [Spotify](file:///Users/dyyxs/项目/Spotify/Spotify) 项目采用 **Objective-C + UIKit（纯代码 Masonry）** 构建，通过**「双门面 (Dual Facade) + 协议插座 (`@protocol` Socket) + 弱引用多播观察者 (`NSHashTable`)」**三大支柱，将 60+ 个源文件划分为职责边界极其清晰的 5 层：

```mermaid
flowchart TD
    subgraph L1["第 1 层：App 入口与根容器层 (Spotify/App/)"]
        AppDelegate["AppDelegate<br/>启动注入数据源与认证源"]
        SceneDelegate["SceneDelegate<br/>引导页 / 登录门禁 / 主界面路由"]
        RootVC["SPRootViewController<br/>三明治容器：底层Tab + 悬浮Mini条 + 顶层零抖动抽屉"]
        NavVC["SPNavigationController<br/>隐藏系统导航栏 + 自接管侧滑返回手势"]
    end

    subgraph L2["第 2 层：Modules 业务界面层 (Spotify/Modules/)"]
        HomeVC["SPHomeViewController<br/>CompositionalLayout + 无限刷新"]
        SearchVC["SPSearchViewController<br/>分类网格 + 防抖综合搜索"]
        LibraryVC["SPLibraryViewController<br/>个人歌单 + 远端推荐双区"]
        DetailVC["SPDetailViewController<br/>歌单/专辑/艺人多态详情"]
        MiniView["SPMiniPlayerView<br/>三槽位横滑切歌 + 色彩插值"]
        NowPlayingVC["SPNowPlayingViewController<br/>大封面三槽位 + 实时歌词卡"]
        Transition["SPNowPlayingTransition<br/>封面跨层级弹簧飞行转场"]
    end

    subgraph L3["第 3 层：Player 播放服务与内核层 (Spotify/Player/)"]
        PlayerFacade["SPPlayerService (播放唯一门面)<br/>loadTicket防竞态 + NSHashTable多播 + 后台Task保活"]
        PlayQueue["SPPlayQueue<br/>Fisher-Yates锚点洗牌 + 插队"]
        AudioCache["SPAudioCacheManager<br/>L1内存热缓存 + L2磁盘LRU + 上下首预加载"]
        AudioEngine["@protocol SPAudioEngine<br/>实现类：SPAVPlayerEngine (AVPlayer + KVO)"]
        SysMedia["SPSystemMediaController<br/>AVAudioSession + 锁屏控制 + 灵动岛"]
    end

    subgraph L4["第 4 层：Data 数据仓库与多源层 (Spotify/Data/)"]
        RepoFacade["SPMusicRepository (数据唯一门面)<br/>内置5个 L3 SPLRUCache + 主线程统一派发"]
        DataSourceProto["@protocol SPMusicDataSource (数据插座)"]
        NeteaseSrc["SPNeteaseMusicSource<br/>SPNeteaseSession + SPNeteaseParser"]
        MockSrc["SPMockMusicSource<br/>mock_data.json + YYModel"]
        LocalStores["SPLibraryStore (点赞/自建歌单)<br/>SPHistoryStore (最近50首记录)"]
    end

    subgraph L5["第 5 层：Core 基础底座与 Models 领域层 (Core/ & Models/)"]
        MediaProto["@protocol SPMediaItem<br/>SPTrack / SPAlbum / SPPlaylist / SPArtist"]
        LRUCache["SPLRUCache<br/>os_unfair_lock + 双向链表 + 哈希表 O(1)"]
        HTTPClient["SPHTTPClient<br/>NSURLSession + timestamp防缓存 + cookie透传"]
        AuthSys["SPAuthService + @protocol SPAuthProvider<br/>SPLocalAuthProvider / SPNeteaseAuthProvider"]
        Extensions["UIImageView+SPCover (Runtime关联对象防闪烁)<br/>UIImage+SPDominantColor (16x16色相桶取色)"]
    end

    AppDelegate --> RepoFacade
    AppDelegate --> AuthSys
    SceneDelegate --> RootVC
    RootVC --> HomeVC & SearchVC & LibraryVC & MiniView & NowPlayingVC
    MiniView -.->|自定义转场| Transition -.-> NowPlayingVC

    HomeVC & SearchVC & LibraryVC & DetailVC & NowPlayingVC -->|只调数据门面| RepoFacade
    HomeVC & DetailVC & MiniView & NowPlayingVC -->|只调播放门面 / 订阅 SPPlayerObserver| PlayerFacade

    PlayerFacade --> PlayQueue & AudioCache & AudioEngine & SysMedia
    AudioCache -->|实时换取播放URL| RepoFacade

    RepoFacade --> DataSourceProto
    DataSourceProto <|.. NeteaseSrc & MockSrc
    NeteaseSrc --> HTTPClient
    AudioCache & RepoFacade & Extensions --> LRUCache
    HomeVC & DetailVC & MiniView & NowPlayingVC --> MediaProto
```

---

## 协议速查总表：7 大 `@protocol` 提供给谁、谁实现了它？

| 协议名称 | 定义位置 | 核心作用 | 哪些类实现了该协议 (Implementers) | 接口提供给谁调用 (Consumers) | 深度解析跳转 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`SPMusicDataSource`** | [SPMusicDataSource.h](file:///Users/dyyxs/项目/Spotify/Spotify/Data/SPMusicDataSource.h#L29-L72) | 数据层“插座”（12 个异步数据读取接口） | 1. [SPNeteaseMusicSource](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Netease/SPNeteaseMusicSource.m#L15-L1042)（网易云 API）<br/>2. [SPMockMusicSource](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Mock/SPMockMusicSource.m#L21-L363)（本地 JSON）<br/>3. [SPMusicRepository](file:///Users/dyyxs/项目/Spotify/Spotify/Data/Repository/SPMusicRepository.m#L29-L316)（门面装饰器） | 所有业务 VC（`Home`/`Search`/`Library`/`Detail`/`NowPlaying`）及 `SPAudioCacheManager`，统一通过 `[SPMusicRepository shared]` 调用 | [查看协议供需详解](01-协议接口供需全景与Runtime实战.md) |
| **`SPMediaItem`** | [SPMediaItem.h](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPMediaItem.h#L20-L31) | 抹平歌曲/专辑/歌单/艺人的 UI 渲染差异 | [SPTrack](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPTrack.m#L33-L45)、[SPAlbum](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPAlbum.m#L27-L44)、[SPPlaylist](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPPlaylist.m#L24-L45)、[SPArtist](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPArtist.m#L12-L21) | `SPMediaCardCell`、`SPHomeShortcutCell`、`SPMediaRowView`、`UIImageView+SPCover`、`SPDetailViewController` | [查看多态模型详解](04-Core基础组件与Models领域模型层.md) |
| **`SPPlayerObserver`** | [SPPlayerService.h](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.h#L32-L39) | 播放器状态一对多弱引用广播（5 个 `@optional` 方法） | `SPRootViewController`、`SPMiniPlayerView`、`SPNowPlayingViewController`、`SPDetailViewController`、`SPHomeViewController`、`SPSearchViewController`、`SPHistoryViewController`、`SPQueueViewController` | 仅由 [SPPlayerService](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.m#L71-L80) 内部通过 `NSHashTable` + `-notify:(SEL)selector` 动态派发 | [查看多播观察者详解](01-协议接口供需全景与Runtime实战.md) |
| **`SPAudioEngine`** | [SPAudioEngine.h](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Engine/SPAudioEngine.h#L26-L41) | 音频解码内核控制接口（加载/播放/暂停/Seek） | [SPAVPlayerEngine](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Engine/SPAVPlayerEngine.m#L26-L158) | 仅由 [SPPlayerService](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.m#L48-L49) 持有并下发播放控制指令 | [查看播放内核层](03-Player播放器内核架构层.md) |
| **`SPAudioEngineDelegate`** | [SPAudioEngine.h](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Engine/SPAudioEngine.h#L16-L24) | 引擎向播放服务汇报状态、进度、自然播完与错误 | [SPPlayerService](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.m#L347-L383) | 仅由 [SPAVPlayerEngine](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Engine/SPAVPlayerEngine.m#L174-L189) 在 KVO/通知触发时向上回调 | [查看播放内核层](03-Player播放器内核架构层.md) |
| **`SPAuthProvider`** | [SPAuthProvider.h](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Auth/SPAuthProvider.h#L19-L45) | 认证通道“插座”（验证码/注册/密码登录） | [SPLocalAuthProvider](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Auth/SPLocalAuthProvider.m#L41-L352)（本地 Keychain+SHA256）、[SPNeteaseAuthProvider](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Auth/SPNeteaseAuthProvider.m#L20-L274)（网易云 API+扫码） | 仅由 [SPAuthService](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Auth/SPAuthService.m#L90-L140) 持有并转发登录/注册请求 | [查看认证体系详解](04-Core基础组件与Models领域模型层.md) |
| **`SPAuthObserver`** | [SPAuthService.h](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Auth/SPAuthService.h#L16-L20) | 登录用户态变更广播 (`-authServiceDidChangeUser:`) | `SPUserAvatarButton`、`SPSideDrawerViewController`、`SPLibraryViewController`、`SPSettingsViewController` | 仅由 [SPAuthService](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Auth/SPAuthService.m#L43-L52) 在登录/登出/恢复会话时广播 | [查看认证体系详解](04-Core基础组件与Models领域模型层.md) |

---

## 区块一：按层级进入子系统架构（看全局各个小部分的架构）

点击下方任意分层文档，按层级进入查看该层内部的所有类、类职责、方法签名与上下游依赖关系：

1. **[第 1 层：App 与根容器架构层](01-App与容器架构层.md)**
   - 包含：`AppDelegate`、`SceneDelegate`、`SPRootViewController`（含 `SPInternalTabBarController` & `SPTabItemButton`）、`SPNavigationController`。
2. **[第 2 层：Data 数据与网络架构层](02-Data数据与网络架构层.md)**
   - 包含：`SPMusicDataSource`、`SPMusicRepository`、`SPNeteaseMusicSource`、`SPNeteaseSession`、`SPNeteaseParser`、`SPMockMusicSource`、`SPLibraryStore`、`SPHistoryStore`。
3. **[第 3 层：Player 播放器内核架构层](03-Player播放器内核架构层.md)**
   - 包含：`SPPlayerService`、`SPPlayerObserver`、`SPAudioEngine`、`SPAVPlayerEngine`、`SPPlayQueue`、`SPAudioCacheManager`、`SPSystemMediaController`。
4. **[第 4 层：Core 基础组件与 Models 领域模型层](04-Core基础组件与Models领域模型层.md)**
   - 包含：`SPMediaItem` 与领域模型（`SPTrack` / `SPAlbum` / `SPPlaylist` / `SPArtist` / `SPLyricDocument` / `SPFeedModels`）、`SPLRUCache`、`SPHTTPClient`、`SPAuthService` 认证体系、`SPTheme`、通用 UI 组件与 Runtime 分类扩展。
5. **[第 5 层：Modules 业务界面模块层](05-Modules业务界面模块层.md)**
   - 包含：`Home`、`Search`、`Library`、`Detail`、`MiniPlayer`、`NowPlaying`、`Transition`、`Drawer`、`Settings`、`Auth`、`Onboarding` 11 个界面模块的组装方式。

---

## 区块二：重点实现路径与特殊场景详解（按需跳转看核心代码逻辑）

针对你点名的 **协议与 Runtime 供需、网络申请、三级缓存、歌曲缓存与预加载、后台播放、无限刷新、界面架构** 七大核心场景，点击下方专题按调用路径逐层深入：

1. **[专题 ①：7 大协议接口供需全景与 OC Runtime 实战](01-协议接口供需全景与Runtime实战.md)**
   - 详解：每个协议谁提供、谁实现、谁调用；`UIImageView+SPCover` 的 `objc_setAssociatedObject` 防 Cell 复用闪烁；`SPPlayerService` 的 `NSHashTable weakObjectsHashTable` + `performSelector:` 多播派发。
2. **[专题 ②：网络申请、匿名 Cookie 防惊群与多路并发降级](02-网络申请与多源并发聚合.md)**
   - 详解：`SPHTTPClient` 的 `timestamp` 防缓存与 `cookie` 透传；`SPNeteaseSession` 的 `pendingWaiters` 并发等待队列合并匿名注册请求；`dispatch_group_t` 6 路并发聚合首页；搜索 `1018` 与播放地址 `/song/url/v1` 双重自动降级。
3. **[专题 ③：手写 O(1) SPLRUCache 与 L1 / L2 / L3 三级缓存体系](03-三级LRU缓存体系实现.md)**
   - 详解：`os_unfair_lock` + 双向链表 + 哈希表实现；为什么 `evictionHandler` 必须在锁外触发；L1 内存缓存（15 分钟 TTL）、L2 磁盘音频文件（200MB 上限 + `manifest.json` 跨进程恢复 MRU）、L3 元数据预热与 `16x16` 色相桶封面主色调缓存。
4. **[专题 ④：歌曲缓存漏斗、防竞态 Ticket 与上下首 0ms 预加载](04-歌曲缓存与上下首0ms预加载.md)**
   - 详解：`loadTicket` 递增令牌解决快速切歌音画串台；`resolvePlayableAssetForTrack:` 四级命中漏斗；低优先级后台下载与 `.tmp` 原子重命名落盘；`preloadAdjacentTracks` 异步预热 `playable`/`duration` 容器头。
5. **[专题 ⑤：后台播放保活、锁屏控制中心与灵动岛集成](05-后台播放与锁屏灵动岛集成.md)**
   - 详解：`AVAudioSessionCategoryPlayback` 与 `UIBackgroundModes`；**后台自动切歌网络空窗期 `beginBackgroundTaskWithName:@"SPTrackTransition"` 防挂起保活**；禁用快进快退稳定展示灵动岛上下首；`MPNowPlayingInfoCenter` 5 秒节流与 `SDImageCache` 同步读取首帧封面；来电打断自动续播与拔耳机自动暂停。
6. **[专题 ⑥：首页无限刷新（上滑渐进分页）、并发保序与无闪烁插入](06-首页无限刷新与渐进分页.md)**
   - 详解：`UICollectionViewCompositionalLayout` 全局底部补充视图 `SPHomeLoadMoreFooterView`；距底 `280pt` 无感预触发；三分类独立分页字典与 `categoryRequestToken`；`slots` 固定槽位数组 + `@synchronized` 解决 `dispatch_group` 并发乱序；`performWithoutAnimation` + `insertSections:` 局部无闪烁增量刷新。
7. **[专题 ⑦：界面架构、三明治根容器、三槽位横滑切歌与封面飞行转场](07-界面架构与丝滑转场交互.md)**
   - 详解：`SPRootViewController` 隐藏原生 TabBar + 动态 `additionalSafeAreaInsets.bottom` 避让 MiniPlayer；底层静止零抖动抽屉与“先无动画 Push 再滑出抽屉”单段转场；全屏右滑返回与左边缘抽屉手势仲裁；`SPMiniPlayerView` 与 `SPNowPlayingViewController` 三槽位 (`prev`/`center`/`next`) 无限横滑、实时背景色彩插值与归位瞬间画面接力；`SPNowPlayingTransition` 封面跨层级弹簧飞行动画。
