# OC 类扩展属性重声明与协变

> 关联源码笔记：[[SPRootViewController]]

## 核心概念

在 Objective-C 中，可以在 `.h` 公开头文件声明一个属性，再在 `.m` 的**类扩展（Class Extension，`@interface ClassName ()`）**中对同名属性进行**重声明（Property Redeclaration）**：

1. **编译期合并（非运行时覆盖）**
   - 与普通 Category（`@interface Foo (Bar)`，运行时合并方法、不能加实例变量）不同，Class Extension 在**编译期**就会与主类 `@interface` 合并为同一份类定义。
   - 编译器只会生成**唯一一个底层 `_ivar`**，并根据重声明自动补齐 `setter` 方法。
2. **重声明合法规则**
   - ✅ **读写权限放宽**：`.h` 声明 `readonly` → `.m` 重声明为 `readwrite`（默认）。对外只暴露 getter 防篡改，对内合成 setter 供自身赋值。
   - ✅ **类型协变收窄（Covariance）**：`.h` 声明为抽象父类 / 不可变容器（如 `UITabBarController *`、`NSArray *`）→ `.m` 收窄为具体私有子类 / 可变容器（如 `SPInternalTabBarController *`、`NSMutableArray *`）。符合里氏替换原则，对外隐藏私有实现细节。
   - ❌ **禁止项**：不能把 `readwrite` 缩回 `readonly`；不能更改内存或原子修饰符（如 `strong` 改 `copy`、`nonatomic` 改 `atomic`）；不能把类型放宽成父类或无关类。

---

## 在本项目中的实际应用

- **源码位置**：`Spotify/App/SPRootViewController.h:L14-L16`、`Spotify/App/SPRootViewController.m:L133-L154`
- **实现方式**：
  - `.h` 对外暴露 `@property (nonatomic, strong, readonly) UITabBarController *tabController;`。
  - `.m` 内部定義私有子类 `SPInternalTabBarController`（隐藏原生 `tabBar` 并增加 `onSelectionChanged` 回调），并在 `SPRootViewController ()` 中重声明为 `@property (nonatomic, strong) SPInternalTabBarController *tabController;`。
  - 外部模块（如 `SPSelfTestRunner`）只需当成标准 `UITabBarController` 读取 `selectedIndex`，而 `SPRootViewController` 内部既能赋初值，又能直接访问子类的 `onSelectionChanged`。

---

## 关键问题沉淀
- **Q: 在 Xcode 中点击 `self.tabController` 跳到了 `.h` 的 `UITabBarController *`，怎么判断它运行时到底是哪个类？**
  - 看 `.m` 顶部的 `@interface ClassName ()` 是否有同名属性重声明，以及 `init` / `viewDidLoad`（如 `setupTabs`）中实际 `[XXX new]` 赋进去的实例类型。
