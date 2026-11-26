# 第 3 层：Player 播放器内核架构层 (`Spotify/Player/`)

> [← 返回架构总控台](README.md) · [← 上一层：Data 数据层](02-Data数据与网络架构层.md) · **当前位置**：区块一（层级架构） > 第 3 层 Player 播放器内核层
> **按需跳转到重点实现逻辑**：[专题 ④：歌曲缓存与上下首 0ms 预加载](04-歌曲缓存与上下首0ms预加载.md) · [专题 ⑤：后台播放与锁屏灵动岛集成](05-后台播放与锁屏灵动岛集成.md) · [专题 ③：三级 LRU 缓存体系 (L1/L2)](03-三级LRU缓存体系实现.md)

---

## 1. 本层局部架构图

`Spotify/Player/` 目录由 **1 个对外门面 (`SPPlayerService`)** 和 **4 个对内专职子系统 (`SPPlayQueue` / `SPAudioCacheManager` / `SPAVPlayerEngine` / `SPSystemMediaController`)** 组成：

```mermaid
flowchart TD
    UICallers["UI 控制器 / MiniPlayer / NowPlaying / 锁屏远程指令"] -->|调用播放命令| PlayerService["SPPlayerService (单例门面)<br/>遵循 &lt;SPAudioEngineDelegate&gt;"]
    PlayerService -->|NSHashTable 多播通知| Observers["@protocol SPPlayerObserver<br/>(RootVC / MiniPlayer / NowPlaying / Detail / Home 等8个订阅者)"]

    PlayerService -->|1. 队列与洗牌管理| Queue["SPPlayQueue<br/>originalTracks + playOrderTracks<br/>Fisher-Yates 锚点洗牌"]
    PlayerService -->|2. 解析可播Asset & 预加载相邻曲目| CacheMgr["SPAudioCacheManager<br/>L1: resolvedURLCache(64) + assetCache(6)<br/>L2: diskIndexCache(40首/200MB) + manifest.json"]
    PlayerService -->|3. 下发音频加载/播放/Seek指令| EngineProto["@protocol SPAudioEngine"]
    PlayerService -->|4. 激活会话 & 5秒节流同步锁屏| SysCtrl["SPSystemMediaController<br/>AVAudioSession + MPRemoteCommandCenter<br/>+ MPNowPlayingInfoCenter"]

    EngineProto <|.. AVEngine["SPAVPlayerEngine<br/>AVPlayer + AVPlayerItem KVO<br/>+ 0.25s PeriodicTimeObserver"]
    AVEngine -.->|SPAudioEngineDelegate 汇报状态/进度/结束| PlayerService
    SysCtrl -.->|锁屏/耳机按键/拖拽进度条回调| PlayerService
```

---

## 2. 本层核心类与协议职责清单

### 2.1 `SPPlayerService` — 播放器唯一门面与状态总控
- **源码文件**：[SPPlayerService.h](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.h) / [SPPlayerService.m](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.m#L35-L385)
- **对外暴露的状态属性 (readonly)**：
  - `currentTrack` / `previousTrack` / `nextTrack` / `upNextTracks`（代理自 `SPPlayQueue`）
  - `state` (`SPPlaybackState`: `Idle` / `Loading` / `Playing` / `Paused` / `Failed`)、`playing`
  - `currentTime` / `duration` / `bufferedTime`、`lastPlaybackUsedCache`、`contextTitle`
  - `repeatMode` (`Off` / `All` / `One`)、`shuffleEnabled`、`preloadEnabled`
- **核心内部方法与设计机制**：
  | 方法名 | 行号定位 | 核心机制 |
  | :--- | :--- | :--- |
  | `-addObserver:` / `-notify:` | [L68-L80](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.m#L68-L80) | 基于 `[NSHashTable weakObjectsHashTable]` 向所有实现 `SPPlayerObserver` 的 UI 组件一对多广播状态 |
  | `-prepareTracks:startIndex:contextTitle:` | [L108-L116](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.m#L108-L116) | 首页首屏加载后调用，预置播放队列但保持 `Paused`，让底部 `SPMiniPlayerView` 启动即有默认歌曲展示 |
  | `-startCurrentTrackWithAutoPlay:` | [L234-L284](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.m#L234-L284) | 核心起播流程：`loadTicket++` 防竞态 → 申请后台过渡 Task → 激活 AudioSession → 调 `SPAudioCacheManager` 解析 `AVURLAsset` → 喂给 `engine` → 记录历史 → 触发 `preloadAdjacentTracks` |
  | `-beginTransitionBackgroundTask` | [L286-L300](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.m#L286-L300) | 向系统申请 `SPTrackTransition` 后台任务，防止后台自动切歌在请求新 URL 的网络空窗期被 iOS 挂起 |
  | `-handlePlaybackFailureWithMessage:` | [L314-L325](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayerService.m#L314-L325) | 遇到无版权或失效歌曲时弹出 Toast `"暂无播放地址，已自动跳过"` 并自动切下一首（`consecutiveFailures` 防止全队列无版权时死循环） |

### 2.2 `SPPlayQueue` — 播放队列与锚点洗牌状态机
- **源码文件**：[SPPlayQueue.h](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayQueue.h) / [SPPlayQueue.m](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayQueue.m#L14-L162)
- **核心设计**：
  - 双数组结构：`originalTracks` 保存传入歌单/专辑的原始顺序；`playOrderTracks` 保存当前实际播放顺序。
  - **锚点 Fisher-Yates 洗牌** ([`-shuffledTracksFrom:anchor:anchorIndex:`](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayQueue.m#L144-L160))：当用户正在听第 $k$ 首歌时开启随机播放，算法先将当前歌曲 `anchor` 从池中取出，对剩余歌曲执行标准 Fisher-Yates 随机洗牌，再把 `anchor` 插回 `index 0` 并令 `currentIndex = 0`。这样开启随机瞬间不会切断当前正在听的歌，且关闭随机时能通过 `[originalTracks indexOfObject:current]` 精准找回原列表位置！
  - **插队支持** ([`-insertNextTrack:`](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayQueue.m#L98-L110))：将指定歌曲直接插入 `playOrderTracks` 的 `currentIndex + 1` 位置。
  - **自然播完 vs 手动下一首** ([`-advanceToNextByAutoPlay:`](file:///Users/dyyxs/项目/Spotify/Spotify/Player/SPPlayQueue.m#L112-L126))：当处于单曲循环 (`SPRepeatModeOne`) 时，只有自然播完 (`isAutoPlay == YES`) 才会停留在当前曲目重播；用户手动点“下一首” (`isAutoPlay == NO`) 则正常前进到下一首。

### 2.3 `SPAudioCacheManager` — L1/L2 音频缓存与上下首预加载器
- **源码文件**：[SPAudioCacheManager.h](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Cache/SPAudioCacheManager.h) / [SPAudioCacheManager.m](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Cache/SPAudioCacheManager.m#L40-L445)
- **核心能力**：
  - 管理 **L1 内存缓存**（64 条带 15 分钟 TTL 的 `resolvedURLCache` + 6 首预热 `AVURLAsset` 的 `assetCache`）与 **L2 磁盘缓存**（`Caches/SPAudioCache/` 下最多 40 首 / `200MB` 音频文件 + `manifest.json` 持久化 MRU 索引）。
  - 提供 [`-resolvePlayableAssetForTrack:completion:`](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Cache/SPAudioCacheManager.m#L270-L332) 四级命中漏斗与 [`-preloadTracks:`](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Cache/SPAudioCacheManager.m#L334-L365) 相邻曲目预加载。
  - 详见 [专题 ④：歌曲缓存与上下首 0ms 预加载](04-歌曲缓存与上下首0ms预加载.md)。

### 2.4 `@protocol SPAudioEngine` & `SPAVPlayerEngine` — AVPlayer 内核封装
- **源码文件**：[SPAudioEngine.h](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Engine/SPAudioEngine.h) / [SPAVPlayerEngine.m](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Engine/SPAVPlayerEngine.m#L26-L325)
- **核心机制**：
  - 统一管理 `AVPlayerItem` 的 KVO 观察（`status`、`loadedTimeRanges`）与 `AVPlayer` 的 `timeControlStatus` KVO，以及 `CMTimeMake(1, 4)`（每 0.25 秒一次）的进度回调。
  - 在 [`-teardownCurrentItem`](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Engine/SPAVPlayerEngine.m#L162-L172) 中安全卸载旧 Item 的 KVO 与通知，防止切歌或销毁时 KVO 崩溃。
  - 实现未就绪状态下的 `pendingSeekTime` 暂存与 Seek 释放后 `0.35s` 内的旧时间戳防闪回过滤 ([L289-L294](file:///Users/dyyxs/项目/Spotify/Spotify/Player/Engine/SPAVPlayerEngine.m#L289-L294))。

### 2.5 `SPSystemMediaController` — 音频会话、锁屏与灵动岛控制器
- **源码文件**：[SPSystemMediaController.h](file:///Users/dyyxs/项目/Spotify/Spotify/Player/System/SPSystemMediaController.h) / [SPSystemMediaController.m](file:///Users/dyyxs/项目/Spotify/Spotify/Player/System/SPSystemMediaController.m#L20-L271)
- **核心机制**：封装 `AVAudioSession`、`MPRemoteCommandCenter` 与 `MPNowPlayingInfoCenter`。详见 [专题 ⑤：后台播放与锁屏灵动岛集成](05-后台播放与锁屏灵动岛集成.md)。

---

> **下一层导航**：[前往第 4 层：Core 基础组件与 Models 领域模型层 →](04-Core基础组件与Models领域模型层.md)
