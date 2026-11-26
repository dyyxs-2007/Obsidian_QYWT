# 跟手拖拽手势与 performWithoutAnimation 防飞

> 关联源码笔记：[[SPRootViewController]]

## 核心概念

1. **`UIPanGestureRecognizer` 双向归一化与吸附判定公式**
   - 连续手势的 `translationInView:` 永远返回**相对于本次手指按下起点**的位移增量。
   - 当同一个手势回调既要处理“拉开（从 `0` 到 `w`）”又要处理“推回（从 `w` 到 `0`）”时，用起始基准值 `baseX = isOpen ? w : 0` 叠加 `translation.x`，再夹逼到 `[0, w]` 并除以 `w`，即可统一成 `0.0 ~ 1.0` 的标准化进度 `progress`。
   - 松手（`Ended / Cancelled`）判定应同时兼顾**快甩（Flick，看 `velocity`）**与**慢拖（Drag，看 `progress`）**：
     `BOOL shouldOpen = (velocity.x > V_THRESHOLD) || (velocity.x >= -V_THRESHOLD && progress > P_THRESHOLD);`
2. **`+[UIView performWithoutAnimation:]` 动画免疫罩**
   - UIKit 在动画上下文执行期间，会把块内一切视图的 `frame` / `alpha` / `transform` 变化都录制成补间动画。
   - 刚用 AutoLayout / Masonry 添加完约束但尚未排版过的视图，`frame` 默认为 `CGRectZero (0,0,0,0)`。若首次排版被卷入外层动画，控件就会从屏幕左上角 `(0,0)` 斜飞到目标位置。
   - 用 `[UIView performWithoutAnimation:^{ [view layoutIfNeeded]; }]` 可临时禁用动画事务（底层联动 `[CATransaction setDisableActions:YES]`），强制首帧坐标 0 秒瞬移到位。

---

## 在本项目中的实际应用

- **源码位置**：`Spotify/App/SPRootViewController.m:L174-L177`、`L437-L449`、`L457-L570`
- **实现方式**：
  - 侧边抽屉的 `edgePan`（左缘右滑开）与 `closePan`（左滑关）共用 `handleDrawerPan:`，跟手过程只修改 `drawerContainerView.transform = CGAffineTransformMakeTranslation(-w * (1 - progress), 0)` 与 `drawerDimView.alpha`，不改 AutoLayout 约束，保证 120Hz 零抖动。
  - `viewDidLoad` 末尾与 `updateMiniPlayerVisibilityAnimated:` 中均使用 `performWithoutAnimation:` 冲刷 `layoutIfNeeded`，彻底解决首次进主页时底部 MiniPlayer 乱飞的问题。

---

## 关键问题沉淀
- **Q: 为什么跟手拖拽时要置 `isAnimatingDrawer = YES` 并在 `viewDidLayoutSubviews` 中避让？**
  - 手势拖动过程中如果触发了父视图 `viewDidLayoutSubviews`，若不加状态锁直接重置关闭态 `transform`，会导致手势跟手画面瞬间闪回原位。
