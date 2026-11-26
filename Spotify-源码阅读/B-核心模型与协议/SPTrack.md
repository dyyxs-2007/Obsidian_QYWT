# SPTrack

> 源码：`Spotify/Models/SPTrack.h`、`Spotify/Models/SPTrack.m`
> 阶段：阶段二：核心模型与协议 (Models & Protocols)

## 核心笔记

### 1. `[self.artists valueForKey:@"name"]` KVC 集合映射与歌手名拼接
- **结论**：利用 `NSArray` 的 **KVC 集合批量取值特性**（等价于 `map`），一行代码将 `NSArray<SPArtist *> *` 映射为 `NSArray<NSString *> *`，再用 `componentsJoinedByString:@", "` 拼成合唱歌手字符串。
- **代码定位**：`SPTrack.m:L21-L23`（同样见于 `SPAlbum.m:L23-L25`）
- **要点**：
  - **执行拆解**：
    1. `[self.artists valueForKey:@"name"]`：对 `NSArray` 调 `valueForKey:` 时，系统会自动遍历数组内每个 `SPArtist` 元素执行 `[artist valueForKey:@"name"]`，返回 `@[@"周杰伦", @"阿信"]`。
    2. `componentsJoinedByString:@", "`：将字符串数组用逗号空格连接为 `@"周杰伦, 阿信"`。
    3. `?: @""`：当 `self.artists` 为 `nil` 时兜底返回空串 `@""`。
  - **KVC 边界注意**：若数组中某个元素的目标属性为 `nil`，`[NSArray valueForKey:]` 会在结果数组中插入 `[NSNull null]`；本项目在 `SPNeteaseParser` 解析层已保证 `artist.name` 非空。
