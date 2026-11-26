# UIFeedbackGenerator 触觉反馈机制

> 关联源码笔记：[[SPRootViewController]]

## 核心概念

`UIFeedbackGenerator`（iOS 10+，位于 `UIKit`）是系统提供的 Taptic Engine 线性马达触觉反馈抽象基类，通过三大具体子类模拟不同物理手感（仅真机生效，模拟器无感）：

1. **`UIImpactFeedbackGenerator`（物理敲击/碰撞感）**
   - 初始化时指定强度档位：`Light`（轻）、`Medium`（中）、`Heavy`（重）、`Soft`（柔，iOS 13+）、`Rigid`（硬，iOS 13+）。
   - 调用 `[gen impactOccurred]` 触发一次震动（iOS 13+ 还支持 `impactOccurredWithIntensity:` 按 0.0~1.0 微调力度）。
   - 适用场景：按钮点击、点赞、弹窗、吸附贴合。
2. **`UISelectionFeedbackGenerator`（刻度拨动感）**
   - 调用 `[gen selectionChanged]` 触发极轻微的“咯噔”感。
   - 适用场景：滚轮选择、开关切换、横滑切歌/分段控件切换。
3. **`UINotificationFeedbackGenerator`（状态结果通知）**
   - 调用 `[gen notificationOccurred:]` 传入 `Success` / `Warning` / `Error`，分别对应系统标准节奏震动。
   - 适用场景：表单提交成功、密码错误拦截、下载完成。

---

## 在本项目中的实际应用

- **源码位置**：`Spotify/Core/Utils/SPHaptics.h`、`Spotify/Core/Utils/SPHaptics.m:L14-L53`
- **实现方式**：
  - 封装为纯类方法工具类（无需 `alloc/init` 实例）：对外提供 `+light`、`+medium`、`+selection` 和全局开关类属性 `@property (class) BOOL enabled`。
  - 开关状态通过文件级静态变量 + `@synchronized (self)` 懒加载读取 `NSUserDefaults`（`sp_haptics_enabled_v1`），用户在「设置与隐私」页关闭触感反馈后，全 App 震动一键静音。
- **关键注意事项**：
  - **主线程调用**：`UIFeedbackGenerator` 属于 UIKit，应在主线程触发。
  - **零延迟预热（`prepare`）**：对时间极敏感的交互（如手势拖拽吸附）可提前持有生成器并调用 `[gen prepare]` 唤醒 Taptic Engine（唤醒态维持约 1~2 秒），减少几十毫秒的马达起振延迟；普通点击直接临时创建即可。
  - **与系统 `UISwitch` 的冲突**：原生 `UISwitch` 在真机上会强制触发无法关闭的系统震动，因此本项目自定义了 `SPToggleSwitch` 并统一走 `SPHaptics` 管控开关。
