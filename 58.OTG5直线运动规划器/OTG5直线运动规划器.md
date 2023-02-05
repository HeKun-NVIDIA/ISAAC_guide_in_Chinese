## OTG5 直线运动规划器
用于直线运动的在线轨迹生成 - V 型 (OTG5) 规划器允许线性运动，同时明确防止曲线。 这在即使与一般直线运动方向有轻微偏差也会导致意外结果的情况下很有用。 目前，Isaac SDK 中可用的 OTG5 规划器（在//packages/otg5 下）支持差分轴距车辆。 它接受全局目标姿势并将其转换为分为四个运动阶段的线性轨迹：

1. 停止机器人，等待它静止：codelet rotate_translate_rotate_state_machine/RotateTranslateRotateStateMachine 使用其 thresholds_stationary 参数（Vector2d）来决定（根据当前里程计状态）机器人是否静止。 第一个值是以 m/s 为单位的最大线速度。 第二个值是以 rad/s 为单位的最大角速度。

2. 向目标位置旋转：机器人向目标位置旋转（以转角最短的方向），直到其方向偏离不超过状态机参数 thresholds_direction_angle 的第一个值。

3. 向目标位置线性移动：机器人向目标位置线性移动，直到其距离偏离不超过状态机参数 threshold_longitudinal_distance 的第一个值。

4. 在目标位置向目标角度旋转：机器人向目标角度（转向角度最短的方向）旋转，直到其方向偏离不超过状态机参数 thresholds_direction_angle 的第一个值。

thresholds_direction_angle 和 thresholds_longitudinal_angle 的第二个值用于确定当前机器人姿态和目标姿态之间的增量是否已经改变到足以证明重新启动运动阶段状态机是合理的。

OTG5 规划器的当前实现允许机器人在向前（默认）或向后（通过将状态机的 drive_backwards 参数设置为真）执行上述运动阶段。


## 最大值和期望值的配置
为确保 OTG5 小码正常工作，需要配置最大和最小速度以及最大加速度和加加速度。 在 dual_otg5/DualOtg5 下配置 OTG5 的组件 API 条目中描述的参数。

## OTG5 的 Flatsim 演示
可以在 //packages/flatsim/apps/demo_7.json 中找到展示 OTG5 集成到应用程序中的 flatsim 演示应用程序。 要运行它，请在 bazel 工作区中执行以下命令：
```bash
bazel run //packages/flatsim/apps:flatsim -- --demo demo_7

```
启用参数commander.robot_remote/isaac.navigation.RobotRemoteControl/disable_deadman_switch，允许OTG5规划器控制模拟机器人。 车辆也可以通过 Isaac Sight 中的虚拟游戏手柄进行控制。