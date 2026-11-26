# additionalSafeAreaInsets 避让技巧

## 核心概念

- `UIViewController.additionalSafeAreaInsets` 允许父容器或当前控制器在系统安全区（Safe Area）之外追加自定义内边距。
- 修改父容器或子控制器的 `additionalSafeAreaInsets` 后，UIKit 会自动更新 `view.safeAreaInsets`，并向下传递到子控制器和遵循 Safe Area 的 `UIScrollView` / Auto Layout 约束，无需各业务页手动硬编码底部高度。

---

## 在本项目中的实际应用
- **使用场景（顶部/底部常驻 MiniPlayer 与自定义 TabBar 统一避让）**：
  - 项目在 `SPRootViewController` 中隐藏了系统原生 `UITabBar`（`self.tabBar.hidden = YES`），改用悬浮在底部的「自定义 `_customTabBar`（`52pt`）+ 常驻 `SPMiniPlayerView`（`56pt`）+ 上下间距（`16pt`）」组合。
  - 通过统一给 `tabController.viewControllers`（首页 / 搜索 / 音乐库三个 `UINavigationController`）设置 `vc.additionalSafeAreaInsets.bottom = 52 + 56 + 16 = 124pt`，所有子页面的 `UITableView` / `UICollectionView` 会自动把 `adjustedContentInset.bottom` 撑开 `124pt`，列表滑到底时最后一行刚好停在 MiniPlayer 上方，零业务侵入。
- **源码位置**：`Spotify/App/SPRootViewController.m:L127`、`L325-L337`、`L437-L449`
- **动态显隐 MiniPlayer 与安全区联动注意事项**：
  1. 在 `viewDidLoad`、`viewSafeAreaInsetsDidChange` 以及 `updateMiniPlayerVisibilityAnimated:` 中同步调用 `updateTabDockMetrics`。
  2. 自定义底栏按钮区 `_customTabBar` 底部锚定在 `mas_safeAreaLayoutGuideBottom`，而背景背板 `_tabBarOpaqueBackdrop` 则从 `_customTabBar.mas_top` 一直延伸到 `contentContainer` 物理底边（覆盖 Home Indicator 区域），配合 `_dockFadeScrim` 渐变幕布彻底杜绝底部露底。

---

## 关键问题沉淀
- **Q: 为什么不直接在每个页面的 `tableView.contentInset` 里写死底部 `124pt`？**
  - 如果由各业务页自己写死 `contentInset.bottom`，不仅每个新 push 的二级页（歌单详情、历史记录、设置页）都要重复写一遍，一旦以后 MiniPlayer 高度调整或动态隐藏，所有页面都会失步；在根容器一层改 `additionalSafeAreaInsets.bottom`，UIKit 会自动沿着控制器树向下传播给所有子页面与滚动视图。
