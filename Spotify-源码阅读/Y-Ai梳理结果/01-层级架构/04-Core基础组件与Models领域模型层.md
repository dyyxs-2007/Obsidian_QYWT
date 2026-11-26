# 第 4 层：Core 基础组件与 Models 领域模型层 (`Spotify/Core/` & `Spotify/Models/`)

> [← 返回架构总控台](README.md) · [← 上一层：Player 播放内核层](03-Player播放器内核架构层.md) · **当前位置**：区块一（层级架构） > 第 4 层 Core 基础与 Models 领域层
> **按需跳转到重点实现逻辑**：[专题 ①：协议供需与 Runtime 实战](01-协议接口供需全景与Runtime实战.md) · [专题 ③：手写 O(1) SPLRUCache 与三级缓存](03-三级LRU缓存体系实现.md)

---

## 1. 本层局部架构图

`Core/` 与 `Models/` 是整个工程的地基，不依赖任何上层业务模块（`Modules` / `Player` / `Data`），向上层提供 **多态领域模型**、**$O(1)$ LRU 缓存容器**、**HTTP 客户端**、**Keychain 认证体系**、**主题 Token**、**Runtime 图像分类** 及 **通用 UI 原子控件**：

```mermaid
flowchart TD
    subgraph Models["Spotify/Models/ (纯净领域模型层)"]
        MediaProto["@protocol SPMediaItem<br/>(-mediaType, -identifier, -title, -subtitle, -coverURL, -themeColor)"]
        Track["SPTrack (单曲)"]
        Album["SPAlbum (专辑)"]
        Playlist["SPPlaylist (歌单 / 播客电台)"]
        Artist["SPArtist (艺人)"]
        Lyric["SPLyricDocument & SPLyricLine<br/>双语 LRC 解析与时间轴定位"]
        Feed["SPHomeSection / SPSearchCategory / SPSearchResult"]
        MediaProto <|.. Track & Album & Playlist & Artist
    end

    subgraph CoreInfra["Spotify/Core/ (基础技术底座)"]
        LRU["Cache/SPLRUCache<br/>os_unfair_lock + 双向链表 + NSMutableDictionary"]
        HTTP["Network/SPHTTPClient<br/>NSURLSession + timestamp防缓存 + cookie透传"]
        Auth["Auth/SPAuthService + @protocol SPAuthProvider<br/>SPLocalAuthProvider (Keychain+SHA256)<br/>SPNeteaseAuthProvider (API+扫码)"]
        Theme["Theme/SPTheme<br/>动态深浅色 Token + SF Symbols + 等宽数字字体"]
        Ext["Extensions/<br/>UIImageView+SPCover (Runtime关联对象防闪烁)<br/>UIImage+SPDominantColor (16x16色相桶取色)<br/>UIColor+SPHex (明度钳制与色彩插值)"]
        Comp["Components/<br/>SPMediaCells / SPMediaRowView / SPNowPlayingIndicatorView<br/>SPGradientView / SPSkeletonView / SPScaleButton / SPToast"]
    end
```

---

## 2. Models 领域模型层详解 (`Spotify/Models/`)

### 2.1 `@protocol SPMediaItem` 与四大媒体实体
- **协议定义**：[SPMediaItem.h](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPMediaItem.h#L20-L31)
- **四个实现类的多态映射对照**：
  | 模型类 | 源码文件 | `mediaType` | `title` | `subtitle` | `coverURL` |
  | :--- | :--- | :--- | :--- | :--- | :--- |
  | **`SPTrack`** | [SPTrack.m](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPTrack.m#L33-L45) | `SPMediaTypeTrack` | `self.name` | `[self artistNames]`（多个歌手以 `", "` 拼接） | 优先 `trackCoverURL`，为空回退 `album.coverURL` |
  | **`SPAlbum`** | [SPAlbum.m](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPAlbum.m#L27-L44) | `SPMediaTypeAlbum` | `self.name` | `@"专辑 · 歌手名"`（或 `"EP · 歌手名"`） | `self.coverURL` |
  | **`SPPlaylist`** | [SPPlaylist.m](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPPlaylist.m#L24-L45) | `SPMediaTypePlaylist` | `self.name` | 播客 (`dj:` 前缀) 返回 `@"播客 · 主播"`；普通歌单返回 `@"歌单 · 创作者"` | `self.coverURL` |
  | **`SPArtist`** | [SPArtist.m](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPArtist.m#L12-L21) | `SPMediaTypeArtist` | `self.name` | `@"艺人"` | `self.coverURL` |

### 2.2 `SPLyricDocument` & `SPLyricLine` — 双语 LRC 歌词解析与时间轴匹配
- **源码文件**：[SPLyricModel.h](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPLyricModel.h) / [SPLyricModel.m](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPLyricModel.m)
- **核心方法**：
  - [`+parseLRC:translationLRC:`](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPLyricModel.m)：使用正则表达式匹配 `[mm:ss.xx]` 时间戳，解析原文歌词与翻译歌词（`tlyric`），按毫秒时间戳将中文翻译对齐挂载到对应的 `SPLyricLine.translation` 上，并按时间升序排序。
  - [`-activeLineIndexForTime:`](file:///Users/dyyxs/项目/Spotify/Spotify/Models/SPLyricModel.m)：根据播放器当前 `currentTime`，快速定位当前应当高亮居中的歌词行下标。

---

## 3. Core 基础底座层详解 (`Spotify/Core/`)

### 3.1 `Core/Cache/SPLRUCache` — 线程安全 $O(1)$ LRU 缓存容器
- **源码文件**：[SPLRUCache.h](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Cache/SPLRUCache.h) / [SPLRUCache.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Cache/SPLRUCache.m#L33-L229)
- **核心设计**：`os_unfair_lock` + 双向链表 (`SPLRUNode`) + 哈希表 (`NSMutableDictionary`)。支持 `maxCount` 与 `maxCost` 双重限制、系统内存警告自动减半清理、锁外触发 `evictionHandler` 及导出 `allKeysInMRUOrder`。详见 [专题 ③：三级 LRU 缓存体系实现](03-三级LRU缓存体系实现.md)。

### 3.2 `Core/Network/SPHTTPClient` — 轻量 JSON 网络客户端
- **源码文件**：[SPHTTPClient.h](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Network/SPHTTPClient.h) / [SPHTTPClient.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Network/SPHTTPClient.m#L15-L145)
- **核心设计**：封装 `NSURLSession`，支持 `bypassCache` 注入毫秒级 `timestamp`、`cookie` 参数透传、HTTP 状态码与业务 `code == 200` 双重校验。详见 [专题 ②：网络申请与多源并发聚合](02-网络申请与多源并发聚合.md)。

### 3.3 `Core/Auth/` — 认证门面、策略插座与 Keychain 存储
- **`SPAuthService`** ([SPAuthService.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Auth/SPAuthService.m#L20-L147))：全局认证门面，通过 `NSHashTable<id<SPAuthObserver>>` 广播用户登录态变更，并用 `SPKeychainStore` 持久化当前 `SPUser` 会话。
- **`SPLocalAuthProvider`** ([SPLocalAuthProvider.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Auth/SPLocalAuthProvider.m#L41-L352))：本地安全认证实现，使用 `SecRandomCopyBytes` 生成 16 字节随机 Salt + `CC_SHA256` 密码哈希存入 Keychain，内置 `demo@spotify.local`（密码 `Spotify123`）体验账号与 6 位本地验证码闭环。
- **`SPNeteaseAuthProvider`** ([SPNeteaseAuthProvider.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Auth/SPNeteaseAuthProvider.m#L20-L274))：对接网易云手机/邮箱密码登录、短信验证码登录，以及二维码扫码登录（`/login/qr/key` → `/login/qr/create` → `/login/qr/check` 轮询 800/801/802/803 状态码 → `/login/status` 获取用户资料并将 `cookie` 写入 `SPNeteaseSession`）。

### 3.4 `Core/Extensions/` — 图像、取色与色彩插值分类
- **`UIImageView+SPCover`** ([UIImageView+SPCover.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Extensions/UIImageView+SPCover.m#L13-L184))：
  - 使用 Runtime `objc_getAssociatedObject` / `objc_setAssociatedObject` 挂载 `SPCoverCurrentURLKey` 防止 Cell 复用重复刷图闪烁。
  - 使用 `UIGraphicsImageRenderer` 纯代码绘制「已点赞的歌曲」蓝紫渐变红心封面 (`+sp_likedSongsCoverImage`) 与自建空歌单默认音符封面 (`+sp_defaultPlaylistCoverImageForColor:`)。
- **`UIImage+SPDominantColor`** ([UIImage+SPDominantColor.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Extensions/UIImage+SPDominantColor.m#L9-L156))：
  - `16x16` 像素降采样 + 12 色相桶饱和度加权算法提取封面主色调，并用 200 容量的 `SPLRUCache` 缓存。
- **`UIColor+SPHex`** ([UIColor+SPHex.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Extensions/UIColor+SPHex.m))：
  - 提供 `-sp_darkenedBy:`、`-sp_clampedForDarkBackground`（钳制 HSB 亮度与饱和度，确保深色背景下白字可读）以及 `-sp_interpolatedToColor:fraction:`（供 MiniPlayer 和大播放页横滑切歌时实时插值渐变背景色）。

### 3.5 `Core/Components/` & `Core/Theme/` — 通用 UI 组件与主题系统
- **`SPTheme`** ([SPTheme.h](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Theme/SPTheme.h) / [SPTheme.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Theme/SPTheme.m))：基于 `UIColor colorWithDynamicProvider:` 提供自适应深色/浅色模式的语义化颜色 Token，切换主题时对 `UIWindow` 执行 `0.28s` 交叉溶解过渡并发送 `SPThemeDidChangeNotification`。
- **`SPMediaCells` & `SPMediaRowView`** ([SPMediaCells.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Components/SPMediaCells.m), [SPMediaRowView.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Components/SPMediaRowView.m))：只面向 `id<SPMediaItem>` 配置的通用卡片与列表行组件，内置 `SPNowPlayingIndicatorView`（3 根用 `CABasicAnimation` 驱动的绿色跳动音律柱）。
- **`SPScaleButton`** ([SPScaleButton.m](file:///Users/dyyxs/项目/Spotify/Spotify/Core/Components/SPScaleButton.m))：重写 `setHighlighted:` 实现按压 `0.94` 倍弹性缩放，支持 `opticalHorizontalOffset`（播放三角形图标向右微调 `1.5~2.0pt` 实现视觉居中）及 SF Symbol 切换动画。

---

> **下一层导航**：[前往第 5 层：Modules 业务界面模块层 →](05-Modules业务界面模块层.md)
