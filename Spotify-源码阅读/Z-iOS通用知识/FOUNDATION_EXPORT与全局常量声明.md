# FOUNDATION_EXPORT 与全局常量声明

> 关联源码笔记：[[SPMusicRepository]]

## 核心概念

在 Objective-C 中，向外暴露全局字符串常量（如通知名 `NSNotificationName`、错误域 `NSErrorDomain` 等）的标准写法是**`.h` 声明 + `.m` 定义**：

```objc
// .h 头文件声明
FOUNDATION_EXPORT NSNotificationName const SPMusicSourceDidChangeNotification;

// .m 实现文件定义
NSNotificationName const SPMusicSourceDidChangeNotification = @"com.dyyxs.spotify.music.sourceDidChange";
```

逐词拆解：
1. **`FOUNDATION_EXPORT`**
   - Apple Foundation 提供的导出宏，在 OC 中等价于 `extern`，在 Objective-C++（`.mm`）中自动展开为 `extern "C"`（防止 C++ Name Mangling 导致符号链接失败）。
   - 语义是“仅声明外部符号，实际定义与赋值在 `.m` 中”，让所有 `#import` 该 `.h` 的文件共享同一个全局符号。
2. **`NSNotificationName`（类型化字符串枚举）**
   - 本质是 `typedef NSString * NSNotificationName NS_EXTENSIBLE_STRING_ENUM;`。
   - 相比直接写 `NSString *`，能提供更明确的语义提示和编译期类型检查，同时支持优雅地桥接到 Swift（如 `Notification.Name`）。
3. **`const` 位置**
   - 写在指针类型之后（等价于 `NSString * const`），锁定**指针本身不可重新赋值**，防止外部代码篡改全局常量指向。

### 为什么优先用 `FOUNDATION_EXPORT ... const` 而不是 `#define`？
- **类型安全**：编译器会检查类型，`#define` 只是纯文本替换。
- **单一地址**：全局只有一份字符串指针，可直接用 `==` 比较地址，不会在各编译单元重复展开。
- **调试与混编友好**：符号在调试器（LLDB）中可见，且支持 Swift `NS_EXTENSIBLE_STRING_ENUM` 桥接。

---

## 在本项目中的实际应用

- **源码位置**：`Spotify/Data/Repository/SPMusicRepository.h:L13`、`Spotify/Data/Repository/SPMusicRepository.m:L11-L12`
- **实现方式**：
  - `.h` 用 `FOUNDATION_EXPORT NSNotificationName const SPMusicSourceDidChangeNotification;` 导出数据源切换通知名。
  - `.m` 中完成实际字符串定义，在 `setSource:` 切换数据源并清理缓存后，通过 `NSNotificationCenter` 在主线程广播该通知；同文件第 11 行也以同样方式定义了 `SPMusicErrorDomain`（对应 `SPMusicDataSource.h` 中的 `FOUNDATION_EXPORT NSErrorDomain const SPMusicErrorDomain;`）。

---

## 关键问题沉淀
- **Q: 如果把 `const` 写在 `NSString` 前面（`const NSString *`）会怎样？**
  - `const NSString *` 修饰的是指针指向的内容（对本就不可变的 `NSString` 意义不大），指针本身仍可被改写；只有写在类型后的 `NSString * const`（即 `NSNotificationName const`）才真正保护全局常量指针不被重新赋值。
