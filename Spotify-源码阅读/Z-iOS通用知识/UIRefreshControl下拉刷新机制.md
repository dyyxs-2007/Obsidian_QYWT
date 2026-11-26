# UIRefreshControl下拉刷新机制

> 关联源码笔记：[[SPHomeViewController]]

## 核心概念

`UIRefreshControl` 是 UIKit 提供的系统原生**顶部下拉刷新控件**（继承自 `UIControl`），可直接挂载到任意 `UIScrollView` 及其子类（`UITableView`、`UICollectionView`）上。

- **绑定方式**：iOS 10+ 直接赋值给 `scrollView.refreshControl`，系统自动接管顶部藏匿、下拉跟随露头、触发刷新及回弹动画，无需手动计算 `contentInset`。
- **事件触发**：当用户下拉超过触发阈值时，发出 `UIControlEventValueChanged` 事件，通知 Target 执行刷新回调。
- **生命周期收尾**：异步刷新结束后（**无论成功或失败**），必须手动在主线程调用 `[refreshControl endRefreshing]` 收起加载菊花；可用 `refreshControl.isRefreshing` 判断当前是否处于刷新态。
- **适用边界**：
  - 仅支持**顶部下拉刷新**，不支持底部上拉加载更多（分页加载需自行通过 `UICollectionView` Footer / `UITableView` `tableFooterView` 或滚动到底部回调实现）。
  - 要求宿主的 `alwaysBounceVertical = YES`（列表内容不足一屏时也能下拉触发）。

---

## 在本项目中的实际应用

- **源码位置**：
  - 首页配置：`Spotify/Modules/Home/SPHomeViewController.m:L231-L234`
  - 刷新触发与收尾：`Spotify/Modules/Home/SPHomeViewController.m:L357-L380`
  - 其他使用处：`SPLibraryViewController.m:L253`、`SPActivityViewController.m:L81`
- **实现方式**：
  ```objc
  _refreshControl = [UIRefreshControl new];
  _refreshControl.tintColor = SPTheme.accentColor;
  [_refreshControl addTarget:self action:@selector(refreshData) forControlEvents:UIControlEventValueChanged];
  _collectionView.refreshControl = _refreshControl;
  ```
  - 下拉触发 `refreshData` 后，调用 `SPMusicRepository` 的 `refreshHomeSectionsForCategory:`（绕过内存缓存强制拉新），在网络回调中执行 `[strongSelf.refreshControl endRefreshing]`。
- **关键注意事项**：
  - 异步请求**失败分支也必须调用 `endRefreshing`**，否则转圈动画会卡死在顶部无法收回。
  - 若需要自定义复杂动画或上下拉一体化体验，生产项目通常会选用 `MJRefresh`；若仅需轻量原生下拉刷新，直接用 `UIRefreshControl` 零依赖且最贴合 iOS 交互手势。
