# Cartographer

Cartographer 是 Google 的一个（同步定位和映射）SLAM 系统，能够进行 2D 或 3D SLAM。 Isaac SDK 结合了 Cartographer 以提供映射功能。

Cartographer 和其他第三方 SLAM 系统可能需要调整（独立于 Isaac SDK）才能在某些应用程序中获得有用的结果。

Cartographer 需要的计算资源可能超过边缘设备。 因此，所提供的示例采用记录的日志数据，而不是直接在边缘设备上运行。 假设 Carter 机器人硬件，示例应用程序执行 2D 映射。


## 数据要求
Flat Lidar Scan 作为 `FlatRangeScanProto` 消息。

* 里程计框架中的激光雷达姿势。 在 Carter 机器人上，它由 DifferentialBaseDynamics 类型的 StateProto 消息中的 DifferentialBaseOdometry codelet 提供。

* 记录和回放日志请参考记录和回放。

## 配置
有关 Cartographer 示例应用程序的配置，请参阅 `IsaacSDK/apps/carter/log_cartographer/log_cartographer.app.json`。


```json
"tick_dt": 0.25,
"num_visible_submaps": 100
"lua_configuration_basename": "carter.lua",

```

* num_visible_submaps 是映射期间要渲染的子图数。 如果需要，减少此数字以节省 CPU 周期。

* tick_dt 是将激光雷达数据输入 Cartographer 之间的秒数。 更频繁地提供数据可能会提高生成地图的质量，同时相应地增加资源成本。

* lua_configuration_basename 指向包含 Cartographer 参数的 LUA 脚本。 有关参数的详细信息，请参阅 [Cartographer Configuration](https://google-cartographer.readthedocs.io/en/latest/configuration.html) 和 Cartographer Tuning Guide。 如有必要，检查并修改 IsaacSDK/apps/carter/log_cartographer/carter.lua 中的参数。

## 启动 Cartographer 示例应用程序
使用以下命令启动示例应用程序：

```bash
bazel run apps/carter/log_cartographer
```

## 监控和可视化
在 Web 浏览器中打开以下 URL：
```bash
http://localhost:3000
```

![](https://docs.nvidia.com/isaac/_images/cartographer_sight.png)

* 该应用程序允许您指定日志输入的路径，如 Replay Widget 中所述。

* 当 Cartographer codelet 的滴答时间长于预设的滴答间隔 tick_dt 时，结果可能会降低。 在这种情况下，减少 num_visible_submaps 或禁用子图的可视化可能会有所帮助。

## 输出
退出时，示例应用程序将最终地图写入 /tmp/submap_merged.png。 使用以下配置参数为 Cartographer codelet 指定输出路径：

```bash
"output_path": "PATH/TO/WRITE"
```



















































