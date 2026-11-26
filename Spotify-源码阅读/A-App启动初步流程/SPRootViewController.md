# SPRootViewController

> 源码：`Spotify/App/SPRootViewController.m`
> 阶段：阶段一：启动与主骨架 (App)

## 核心笔记

### 1. 底部常驻 Dock 架构（MiniPlayer + 自定义 TabBar + 延伸背板与 1px 顶线）
- **结论**：`setupBottomDock` 把「渐变遮罩 + 延伸背板/顶线 + 自定义 TabBar + 悬浮 MiniPlayer」作为**跨 Tab 常驻的统一 Dock 整体**来管理，保证层级高于所有子页面且共享安全区避让。
- **代码定位**：`SPRootViewController.m:L227-L303`、`L325-L353`、`L631-L652`；`SPMiniPlayerView.h:L12-L16`、`SPMiniPlayerView.m:L596-L605`
- **要点**：
  - **为什么 MiniPlayer 放在 `setupBottomDock`**：放在 Root 容器层才能在切 Tab / push 详情页时常驻悬浮；其底边直接锚定在 `_customTabBar.mas_top - 8`，并与 TabBar 一起通过 `updateTabDockMetrics` 统一写入 `additionalSafeAreaInsets.bottom`。
  - **`tapHandler` 为什么在 RootVC 绑定**：View 与路由解耦（依赖反转）。`SPMiniPlayerView` 是纯 `UIView`，点击/上滑/待机点播放都只抛 `tapHandler()`；实际 `presentNowPlaying`（含自定义 Hero 转场 `nowPlayingTransition`）交由 `SPRootViewController` 调度。用 `__weak typeof(self) weakSelf` 打破 `RootVC → miniPlayer → tapHandler → RootVC` 循环引用。
  - **`_tabBarOpaqueBackdrop` 与 `_tabBarTopHairline`**：`_customTabBar`（52pt）透明且底边停在 SafeArea 内防按钮撞小黑条；底下垫一层 `opaque = YES` 的 `_tabBarOpaqueBackdrop` 从 TabBar 顶部一直延伸到屏幕物理底边，防止 Home Indicator 区域露底。`_tabBarTopHairline` 高度为 `1.0 / UIScreen.mainScreen.scale`（精确 1 物理像素），负责分隔底栏与上方内容。
  - **`_dockFadeScrim` 与 `resolvedColorWithTraitCollection:`**：`solidDockColor` 是动态色（`colorWithDynamicProvider:`），传给底层 `CAGradientLayer`（`SPGradientView`）前需先用 `resolvedColorWithTraitCollection:` 解析出当前深/浅模式下的确定静态色 `resolvedDock`，再派生出 `alpha = 0.0 → 0.86 → 1.0`（对应 `locations: 0 / 0.55 / 1.0`）的竖直渐变幕布垫在 MiniPlayer 背后，消除 MiniPlayer 与 TabBar 间 8pt 缝隙露底并让列表自然溶入底栏。

### 2. `tabController` 的单向同步与 OC 类扩展属性重声明
- **结论**：`self.tabController` 就是私有子类 `SPInternalTabBarController`（隐藏系统原生 `tabBar`，作为 `selectedIndex` 的唯一数据源），通过 `.h` / `.m` 属性重声明隐藏子类细节，并在 `setupBottomDock` 末尾手动补做一次初始高亮同步。
- **代码定位**：`SPRootViewController.h:L15`；`SPRootViewController.m:L28-L54`、`L135`、`L202-L221`、`L300`、`L305-L323`
- **要点**：
  - **单向数据流**：点击自定义底栏（`tabButtonTapped:`）、抽屉或空白播放页切 Tab 时，都统一只改 `self.tabController.selectedIndex` → 触发重写的 `setSelectedIndex:` → 回调 `onSelectionChanged` 反向刷新 `_tabButtons` 高亮。
  - **为什么 `L300` 还要手动调一次 `syncCustomTabSelection:`**：初始化时序上先跑 `setupTabs`（系统默认停在 `0`，此时 `_tabButtons` 还没创建），后跑 `setupBottomDock`（刚建出的 3 个按钮默认全未选中），不补这一行刚进首页底栏会全灰。
  - **`.h` 声明父类 `readonly`、`.m` 改私有子类 `readwrite`**：属于 OC **Class Extension 属性重声明 + 子类协变**（编译期合并为同一个 `_tabController` ivar，并非两个属性覆盖）。对外隐藏内部子类并防篡改，对内可赋值并访问子类属性（初始化在 `setupTabs` 的 `L203` 局部 `tab` → `L220` 赋给 `self.tabController`）。规则：允许 `readonly → readwrite`、父类 → 子类，禁止反向缩权或改内存/原子语义。

### 3. 侧边抽屉叠加层与跟手拖拽算法（`handleDrawerPan:`）
- **结论**：`edgePan`（左缘右滑打开）和 `closePan`（抽屉展开时左滑关闭）**共用同一个回调 `handleDrawerPan:`**，通过 `baseX` 将相对位移 `translation.x` 统一映射为 `[0.0, 1.0]` 的展开进度 `progress`，松手时综合**瞬时速度（`velocity.x`）**与**位置阈值（`progress > 0.45`）**决定吸附展开或收起。
- **代码定位**：`SPRootViewController.m:L457-L570`
- **要点**：
  - **为什么有 `baseX = self.drawerOpen ? w : 0`**：
    - `translationInView:` 返回的是**相对于本次手指按下起点**的偏移量（右滑为正，左滑为负）。
    - 若从关闭态拉开（`drawerOpen == NO`）：起点 `baseX = 0`，向右滑 `translation.x > 0`，当前抽屉右边缘位置 `currentX = 0 + translation.x`。
    - 若从打开态推回（`drawerOpen == YES`）：起点抽屉已经全开在 `baseX = w`，向左滑 `translation.x < 0`，当前位置 `currentX = w + translation.x`。
    - 用 `MIN(MAX(..., 0), w)` 钳制在 `[0, w]` 后除以 `w`，就把“拉开”和“推回”统一成了同一个 `0.0（全关）~ 1.0（全开）` 的 `progress`。
  - **手势三阶段状态机（`UIGestureRecognizerState`）**：
    - `Began`（刚识别成功）：置 `isAnimatingDrawer = YES`（锁住 `viewDidLayoutSubviews` 防止布局刷新把抽屉打回原位），取消遮罩和抽屉的 `hidden`，刷最新用户数据，先把进度对齐到起始态（`0` 或 `1`）。
    - `Changed`（手指移动中）：每帧实时调用 `applyDrawerProgress:progress`，用 `CGAffineTransformMakeTranslation(-w * (1 - progress), 0)` 平移抽屉并同步遮罩 `alpha`（只改 GPU `transform` 不改 AutoLayout 约束，保证 60/120fps 零抖动）。
    - `Ended / Cancelled`（松手或被打断）：判断 `BOOL shouldOpen = (velocity.x > 400) || (velocity.x >= -400 && progress > 0.45)`：
      1. **快甩优先**：向右甩速度 `> 400pt/s`（哪怕只拉了一点点）→ 直接开；向左甩速度 `< -400pt/s`（哪怕还在大半开位置）→ 直接关。
      2. **慢拖看位置**：速度在 `[-400, 400]` 之间（慢慢拖后停住松手），只要拉出超过 `45%`（`progress > 0.45`）就自动吸开，否则弹回关闭。

### 4. 页面转场编排（抽屉单段跳页 + 全屏播放页自定义 Hero 转场挂载）
- **结论**：`SPRootViewController` 将复杂的封面飞跃动画抽离到独立的 `SPNowPlayingTransition`（遵守 `UIViewControllerTransitioningDelegate`），自身只做轻量组装；同时在首帧布局与抽屉跳页中使用无动画操作消除视觉闪烁与两段割裂。
- **代码定位**：`SPRootViewController.m:L174-L177`、`L437-L449`、`L578-L620`、`L631-L652`；`SPNowPlayingTransition.h:L12-L19`
- **要点**：
  - **把转场动画交给独立类（`SPNowPlayingTransition`）**：
    1. 设置 `vc.modalPresentationStyle = UIModalPresentationCustom` + `vc.transitioningDelegate = self.nowPlayingTransition`，UIKit 在 `present` / `dismiss` 时就会交由 `SPNowPlayingTransition` 执行“MiniPlayer 小封面 ↔ 全屏大封面”的 Hero 缩放飞跃动画。
    2. **为什么 `nowPlayingTransition` 必须是 `strong` 属性**：UIKit 的 `vc.transitioningDelegate` 是 **`weak` 引用**，若只用局部变量创建，方法结束即释放，关闭页面时转场就会失效。
    3. **为什么 `sourceCoverProvider` 传 Block**：每次触发转场的瞬间才动态获取当前 `miniPlayer.coverView` 的最新位置与图片，同时让转场类只依赖通用 `UIView *` 而不耦合 `SPMiniPlayerView`。
    4. **`dismissDragProgressHandler` 与 `emptyActionHandler` 中介桥接**：全屏页下拉手势实时把进度 `0~1` 抛给 `dismissDragProgressHandler`，由 RootVC 转调 `nowPlayingTransition` 同步淡化底层黑罩（`0.48 * (1 - progress)`）；空白待机播放页点「去首页 (`0`) / 去音乐库 (`2`)」则通过 `emptyActionHandler` 回调 RootVC 修改 `tabController.selectedIndex`，保持 `SPNowPlayingViewController` 与外部容器零耦合。
  - **`[UIView performWithoutAnimation:]` 防首帧乱飞与抽屉单段跳转**：
    1. `viewDidLoad` 与 `updateMiniPlayerVisibilityAnimated:` 中用 `performWithoutAnimation:` 包裹 `layoutIfNeeded`，强制首帧约束 0 秒定格，防止被外层转场动画卷入导致控件从左上角 `(0,0)` 飞到底部。
    2. `handleDrawerAction:` 点击抽屉菜单跳二级页时，先在底层导航栈执行 `[nav pushViewController:targetVC animated:NO]`（无动画就位），再调用 `[self closeSideDrawerAnimated:YES]` 滑出抽屉直接显露目标页，避免“先关抽屉闪一下首页、再推入新页面”的两段动画割裂。
