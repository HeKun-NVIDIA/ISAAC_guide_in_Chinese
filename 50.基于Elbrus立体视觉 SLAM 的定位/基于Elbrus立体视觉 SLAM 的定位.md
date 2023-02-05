# 基于Elbrus立体视觉 VSLAM 的定位

Elbrus 基于两项核心技术：视觉里程计 (VO) 和同步定位与地图绘制 (SLAM)。

视觉里程计是一种用于估计相机相对于其起始位置的位置的方法。 此方法具有迭代性质：在每次迭代时，它都会考虑两个相应的输入帧（立体声对）。 在两个框架上，它都找到了一组关键点。 匹配这两组中的关键点可以估计相机在帧之间的过渡和相对旋转。

Simultaneous Localization and Mapping 是一种建立在 VO 预测之上的方法。 它旨在通过利用先前看到的轨迹部分的知识来提高 VO 估计的质量。 它检测当前场景是否在过去见过，即相机运动的循环，并运行额外的优化程序来调整先前获得的姿势。

除了视觉数据，Elbrus 还可以使用惯性测量单元 (IMU) 进行测量。 当 VO 无法估计姿势时，它会自动切换到 IMU——例如，当相机前面有暗光或长固体表面时。 在视觉条件不佳的情况下，使用 IMU 通常会显着提高性能。

Elbrus 提供实时跟踪性能：VGA 分辨率超过 60 FPS。 对于 [KITTI 基准测试](http://www.cvlibs.net/datasets/kitti/)，该算法实现了约 1% 的定位漂移和 0.003 度/米运动的方向误差。

Elbrus 允许在各种环境和不同用例中进行稳健跟踪：室内、室外、空中、HMD、汽车和机器人。

![](https://docs.nvidia.com/isaac/_images/elbrus_visual_slam_zed_image2.jpg)


## 架构
Elbrus 实现了一个基于关键帧的 Stereo VIO 和 SLAM 架构，这是一个两层系统：所有输入帧的一个小子集被用作关键帧并由额外的算法处理，而其他帧通过二维跟踪快速解决已经 选定的意见。 端到端跟踪管道包含两个主要组件：2D 和 3D。

![](https://docs.nvidia.com/isaac/_images/elbrus_visual_slam_arch.jpg)


在顶端：

* VIO 将 camera extrinsics 和 observations 推向 SLAM。

* SLAM 将节点和边添加到姿势图 (PG)。

* SLAM 创建地标并将它们存储在地标空间索引 (LSI) 中。

在底部，SLAM 周期性地执行 Loop Closure Solving。 如果成功：

* LAM 向姿势图 (PG) 添加边。

* SLAM 做姿势图优化。

立体视觉惯性里程计 (Stereo VIO) 使用从立体相机装置获得的成像数据检索左侧相机相对于其起始位置的 3D 姿态。 立体相机装备需要两个具有已知内部校准的相机，它们彼此刚性连接并牢固地安装到机器人框架上。 左右相机之间的转换是已知的，并且图像采集的时间是同步的。

Simultaneous Localization and Mapping (SLAM) 建立在 Stereo VIO 的结果之上。 它在姿势图上进行了额外的优化过程，从而提高了准确性，反而消耗了一些额外的资源。 使用 SLAM 是一个选项，可以禁用。

当以 30 或 60 fps 的速度记录立体图像，并且以 200-300 Hz（如果可用）记录 IMU 角速度和线性加速度测量值时，Elbrus 保证最佳跟踪精度。 立体 VIO 和 SLAM 的不准确性小于 1% 的平移漂移和约 0.03 度/米的角运动误差，这是针对 KITTI 基准测量的，该基准以 10 fps 的速度记录，每帧的分辨率为 1382x512。


![](https://docs.nvidia.com/isaac/_images/elbrus_visual_slam_pipe.jpg)


在图像输入严重退化的情况下（灯被关闭、驾驶时颠簸处的剧烈运动模糊以及其他可能的情况），额外的运动估计算法将确保姿势跟踪的可接受质量：

* IMU 读数积分器提供可接受的姿势跟踪质量，持续时间约为 ~< 1 秒。

* 在 IMU 出现故障的情况下，恒速积分器会继续提供 Stereo VIO 在出现故障之前报告的最后一个线速度和角速度。 这提供了约 0.5 秒的可接受姿势跟踪质量。

## 嵌入式高保真
由于所有相机都有镜头，镜头失真始终存在，使画面中的物体倾斜。 在 ImageWarp 小代码中实现了一种通用镜头失真算法。 然而，对于视觉里程计跟踪，Elbrus 库带有一个内置的不失真算法，它提供了一种更有效的方法来处理原始（失真的）相机图像。 Elbrus 可以跟踪失真图像上的 2D 特征，并将不失真限制为浮点坐标中的选定特征。 这比在跟踪之前执行的所有图像像素的不失真要快得多并且更准确。

**注意**

如果您使用其他需要未失真图像的小码，则需要改用 ImageWarp 小码。

要使用 Elbrus 不失真，请在 ElbrusVisualSLAM GEM 中设置 left.distortion 和 right.distortion（请参阅 ImageProto）输入。 支持以下主要 DistortionModel 选项：

* 具有三个径向和两个切向畸变系数的布朗畸变模型：(r0 r1 r2 t0 t1)

* 具有四个径向畸变系数的鱼眼（广角）畸变：（r0、r1、r2、r3）

有关详细信息，请参阅 [DistortionProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#distortionproto) 文档。

**注意**

“elbrus_visual_slam_zed”示例应用程序使用 ZED 相机，它在 StereoLabs SDK 中执行软件不失真。


## 嵌入式降噪
如果您的应用程序或环境由于光线不足而产生嘈杂的图像，Elbrus 可能会选择太多不正确的特征点。 在这种情况下，启用 ElbrusVisualSLAM GEM 中的 `denoise_input_images` 选项以提高跟踪速度和准确性进行去噪。

## 惯性测量单元 (IMU) 集成
立体声 VIO 使用从 IMU 获得的测量值，该 IMU 牢固地安装在相机装备或机器人底座上。 如果视觉里程计由于图像输入的严重退化而失败，位置跟踪将在 IMU 输入上进行，持续时间最多为一秒。 只要视频输入中断持续时间少于一秒，IMU 集成就能确保无缝姿势更新。

要使 IMU 集成与 Stereo VIO 配合使用，机器人必须在应用程序开始时处于水平位置，否则 Stereo VIO 维护的世界坐标系 (WCS) 中的起始姿势和重力加速度矢量将不正确。

## SLAM 与纯视觉里程计
Elbrus 为用户提供了两种可能的模式：带 SLAM 的视觉里程计（默认）或纯视觉里程计。 在相机路径经常被包围或纠缠的情况下，使用 SLAM 更为有利。 尽管 SLAM 增加了 VO 的计算资源需求，但它通常会显着减少最终位置漂移。

另一方面，使用纯 VO 可能对相对较短和笔直的相机轨迹有用。 此选择将减少 CPU 使用率并提高工作速度，而不会导致基本姿势漂移。


![](https://docs.nvidia.com/isaac/_images/odom_vs_slam.jpg)



## 使用立体相机示例应用程序
Isaac SDK 包括以下示例应用程序，它们演示了 Stereo VIO 和 SLAM 与机器人社区中流行的第三方立体相机的集成：

* packages/visual_slam/apps/elbrus_visual_slam_zed_python.py：此 Python 应用程序演示了 SLAM 与 ZED 和 ZED Mini (ZED-M) 相机的集成。

* packages/visual_slam/apps/elbrus_visual_slam_zed.app.json：此 JSON 示例应用程序演示了 SLAM 与配备 IMU 的 ZED-M 相机的集成。

* packages/visual_slam/apps/elbrus_visual_slam.app.json：此 JSON 示例应用程序演示了 SLAM 与配备 IMU 的 ZED-M 相机的集成。

* packages/visual_slam/apps/elbrus_visual_slam_realsense.py：此 Python 应用程序演示了 SLAM 与英特尔实感 435 相机的集成。

要运行视觉里程计，环境不应该是没有特征的（如普通的白墙）。 Isaac SDK 的 SVO 分析可见特征。 另一种方法是使用传感器融合方法来处理此类环境。

要尝试 ZED 示例应用程序之一，首先将 [ZED 相机](https://docs.nvidia.com/isaac/packages/sensors/doc/zedcamera.html#zed-camera)连接到您的主机系统或 Jetson 设备，并确保它按照 ZED 相机文档中的描述工作。

要试用 RealSense 435 示例应用程序，首先将 RealSense 摄像头连接到您的主机系统或 Jetson 设备，并确保它按照 RealSense 摄像头文档中的描述工作。

以下部分描述了运行其中一个示例应用程序所需的步骤。

**重要**

如果要将常规 ZED 相机与 JSON 示例应用程序一起使用，则需要在运行它之前编辑 packages/visual_slam/elbrus_visual_slam_zed.app.json 应用程序：更改 codelet 配置参数 zed/zed_camera/enable_imu 和 elbrus_visual_slam_zed/elbrus_visual_slam_zed/process_imu_readings 从“true”到“false”。



## 源代码
Isaac SDK 包括 Elbrus 立体声跟踪器作为由小码封装的动态库。 包装 Elbrus 立体跟踪器的 Isaac codelet 接收一对输入图像、相机内在参数和 IMU 测量值（如果可用）。 如果视觉跟踪成功，则小代码将左摄像机相对于世界框架的姿势发布为 Pose3d 消息，其时间戳等于左框架的时间戳。

如果视觉跟踪丢失，左摄像机姿势的发布将中断，直到跟踪恢复。 视觉跟踪恢复后，将恢复发布左摄像机位姿，但无法保证估计的摄像机位姿与世界坐标系中左摄像机的实际位姿相对应。 使用视觉里程计小代码的应用程序必须检测相机姿势更新的中断并启动外部重新定位算法。

Elbrus codelet 有两个字段用于将左相机的姿势写入姿势树：

* odom_pose - 对应于通过视觉里程计获得的姿势。

* slam_pose - 对应于 SLAM 获得的姿势。

还有两个不同的姿势发布者用于与其他小码交互：

* left_camera_vio_pose 用于视觉里程计姿势。

* left_camera_slam_pose 用于 SLAM 姿势。

如果禁用 SLAM，将使用视觉里程计姿势而不是 SLAM 姿势。

## 在 x86_64 主机系统上运行基于立体视觉 SLAM 的本地化示例应用程序
* 使用以下命令为带 IMU 的 ZED 相机构建并运行 JSON 示例应用程序：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/visual_slam/apps:elbrus_visual_slam_zed
```


* 使用以下命令为 ZED 相机构建并运行 Python 示例应用程序：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/visual_slam/apps:elbrus_visual_slam_zed_python -- --imu
```

* 使用以下命令为不带 IMU 的 ZED 相机构建并运行 Python 示例应用程序：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/visual_slam/apps:elbrus_visual_slam_zed_python
```

* 使用以下命令为 Realsense 435 摄像头构建并运行 Python 示例应用程序：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/visual_slam/apps:elbrus_visual_slam_realsense

```
其中 bob 是您在主机系统上的用户名。

## 使用模拟器示例应用程序
Isaac SDK 包括以下示例应用程序，演示了立体视觉里程计与 Isaac Sim Omniverse 和 Isaac Sim Unity3D 的集成。 请注意，这些应用演示了纯立体视觉里程计，没有 IMU 测量集成。

* packages/visual_slam/apps/elbrus_visual_slam_sim_lidar.py：此 Python 应用程序演示了基于立体视觉 SLAM 的本地化与虚拟 Carter 机器人的集成，并使用了激光雷达导航堆栈。 运行此应用程序时可以传递几个命令行参数：例如，使用 `--sim <simulator_name>` 选择模拟器，使用 --record 记录模拟器数据。 有关可用命令行参数的更多信息，您可以查看以下文件：packages/visual_slam/apps/elbrus_visual_slam_sim_lidar.py

* packages/visual_slam/apps/elbrus_visual_slam_sim_joystick.py：此 Python 应用程序演示了基于立体视觉 SLAM 的定位与虚拟 Carter 机器人的集成，需要手动控制导航。 这个应用程序有一堆命令行参数，可以在运行这个应用程序时传递。 例如。 `--sim <simulator_name>` 选择模拟器，--record 记录模拟器数据。 有关可用命令行参数的更多信息，您可以查看以下文件：packages/visual_slam/apps/elbrus_visual_slam_sim_joystick.py 此应用程序需要的资源比 elbrus_visual_slam_sim_lidar 少得多。

接下来的部分描述了在 Isaac Sim Unity3D 中运行示例应用程序的步骤。

## 将 elbrus_visual_slam_sim_lidar 示例应用程序与 Isaac Sim Unity3D 结合使用
1. 按照 [Isaac Sim Unity3D 设置说明](https://docs.nvidia.com/isaac/doc/setup.html#setup-isaac-unity3d)中的说明下载并解压 Unity Player（“播放模式”）构建。

2. 使用以下命令启动中型仓库场景的 Isaac Sim 模拟：
```bash
bob@desktop:~isaac_sim_unity3d/builds$ ./sample.x86_64 -screen-quality Medium -screen-width 854 -screen-height 480 --scene medium_warehouse --scenario 2 --framerate 30
```

3. 在单独的终端中输入以下命令以运行 elbrus_visual_slam_sim_lidar Isaac 应用程序：
```bash
bob@desktop:~$ cd isaac/sdk
bob@desktop:~/isaac/sdk$ bazel run packages/visual_slam/apps:elbrus_visual_slam_sim_lidar
```

4. 打开 http://localhost:3000/ 以通过 Isaac Sight 监控应用程序。

5. 右键单击 elbrus_visual_slam_sim_lidar - Map View Sight 窗口并选择设置。

6. 在设置中，单击选择标记下拉菜单并选择“pose_as_goal”。

7. 单击添加标记。

8. 单击更新。 标记将添加到地图中。 您可能需要放大地图才能看到新标记。 机器人不会立即开始导航到标记。

9. 单击标记并将其拖动到地图上的新位置。 机器人将开始导航到标记位置。 有关详细信息，请参阅[交互式标记](https://docs.nvidia.com/isaac/packages/sight/doc/interactiveMarkers.html#interactive-markers)。 另一种选择是在右侧的“应用程序配置”工具箱中将 goals.goal_behavior->desired_behavior 从 pose 切换为 random。

10. 启用所需的通道以确保立体视觉 SLAM 小部件的平滑可视化。 避免一次启用所有应用程序通道，因为这可能会导致 Sight 延迟问题，当应用程序将过多数据流式传输到 Sight 时会发生这种情况。 激光雷达导航堆栈通道的可视化与此基于立体视觉 SLAM 的定位示例应用程序的目的无关。 有关其他详细信息，请查看[常见问题解答](https://docs.nvidia.com/isaac/doc/faq.html#isaac-faq)页面。

**重要**

如果您在运行模拟时遇到错误，请尝试更新已部署的 Isaac SDK navsim 包，其中包含要在 Unity 中运行的 C API 和 NavSim 应用程序。 使用以下命令将 `//packages/navsim/apps:navsim-pkg` 重新部署到 Isaac Sim Unity3D：

```bash
bob@desktop:~$ cd isaac/sdk
bob@desktop:~/isaac/sdk$ ./../engine/engine/build/deploy.sh -p //packages/navsim/apps:navsim-pkg -d x86_64 -h localhost --deploy_path ~/isaac_sim_unity3d/builds/sample_Data/StreamingAssets
```
![](https://docs.nvidia.com/isaac/_images/elbrus_visual_slam_sim_lidar.jpg)



## 将 elbrus_visual_slam_sim_joystick 示例应用程序与 Isaac Sim Unity3D 结合使用
1. 按照 [Isaac Sim Unity3D 设置说明](https://docs.nvidia.com/isaac/doc/setup.html#setup-isaac-unity3d)中的说明下载并解压 Unity Player（“播放模式”）构建。

2. 使用以下命令启动中型仓库场景的 Isaac Sim 模拟：
```bash
bob@desktop:~isaac_sim_unity3d/builds$ ./sample.x86_64 -screen-quality Medium -screen-width 854 -screen-height 480 --scene medium_warehouse --scenario 2 --framerate 30
```

3. 在单独的终端中输入以下命令以运行 elbrus_visual_slam_sim_joystick 应用程序：
```bash
bob@desktop:~$ cd isaac/sdk
bob@desktop:~/isaac/sdk$ bazel run packages/visual_slam/apps:elbrus_visual_slam_sim_joystick

```
4. 打开 http://localhost:3000/ 以通过 Isaac Sight 监控应用程序。

5. 使用 Virtual Gamepad 窗口在地图上导航机器人：首先，单击左侧的 Virtual Gamepad，然后单击小部件上的 Connect to Backend。 选择键盘并使用“wasd”键导航机器人。 有关详细信息，请参阅使用 Sight 的远程操纵杆。

接下来的部分将介绍如何在 Isaac Sim Omniverse 中运行示例应用程序。

## 将 elbrus_visual_slam_sim_lidar 示例应用程序与 Isaac Sim Omniverse 结合使用
1. 完成 [Isaac Sim Omniverse 入门](https://docs.nvidia.com/isaac/doc/simulation/ovkit.html#isaac-sim-omniversekit-getting-started)部分中描述的步骤。

2. 启动 Isaac Sim Omniverse 并打开以下场景：`omniverse:/Isaac/Samples/Isaac_SDK/Scenario/carter_warehouse_with_forklift_stereo.usd`

3. 通过单击“创建应用程序”按钮启动 Robot Engine Bridge，然后按播放。 桥应该创建一个新的视口，总共有两个视口，每个相机一个（左和右）。

4. 在单独的终端中输入以下命令以运行 elbrus_visual_slam_sim_lidar Isaac 应用程序：
```bash
bob@desktop:~$ cd isaac/sdk
bob@desktop:~/isaac/sdk$ bazel run packages/visual_slam/apps:elbrus_visual_slam_sim_lidar -- --map=apps/assets/maps/virtual_test_warehouse_1.json --sim=isaac_sim_ov
```
5. 打开 http://localhost:3000/ 以通过 Isaac Sight 监控应用程序。

6. 右键单击 elbrus_visual_slam_sim_lidar - Map View Sight 窗口并选择设置。

7. 在设置中，单击选择标记下拉菜单并选择“pose_as_goal”。

8. 单击添加标记。

9. 单击更新。 标记将添加到地图中。 您可能需要放大地图才能看到新标记。 机器人不会立即开始导航到标记。

10. 单击标记并将其拖动到地图上的新位置。 机器人将开始导航到标记位置。 有关详细信息，请参阅交互式标记。 另一个选项是在右侧的应用程序配置工具箱中将“goals.goal_behavior->desired_behavior”从“pose”切换为“random”。

11. 启用所需的通道以确保立体视觉 SLAM 小部件的平滑可视化。 避免一次启用所有应用程序通道，因为这可能会导致 Sight 延迟问题，当应用程序将过多数据流式传输到 Sight 时会发生这种情况。 激光雷达导航堆栈通道的可视化与此示例应用程序的目的无关。 有关其他详细信息，请参阅常见问题解答页面。

![](https://docs.nvidia.com/isaac/_images/elbrus_visual_slam_sim_lidar.jpg)



## 使用elbrus_visual_slam_sim_joystick示例申请与isaac sim omniverse
1. 完成：参考文献中描述的步骤：“入门<get-started>”部分for Isaac Sim Omniverse。

2. 启动Isaac Sim Omniverse并打开以下场景：Omniverse：/iisaac/samples/isaac_sdk/scenario/carter_warehouse_with_with_forklifts_stereo.usd

3. 通过单击“创建应用程序”按钮并按Play启动机器人引擎桥。 桥梁应创建一个新的视口，总共有两个视口，一个用于每个相机（左右）。

4. 在单独的终端中输入以下命令以运行ELBRUS_VISUAL_SLAM_SIM_JOYSTICK应用程序：
```bash
bob@desktop:~$ cd isaac/sdk
bob@desktop:~/isaac/sdk$ bazel run packages/visual_slam/apps:elbrus_visual_slam_sim_joystick -- --map=apps/assets/maps/virtual_test_warehouse_1.json --sim=isaac_sim_ov
```

5. 打开http：// localhost：3000/以通过ISAAC瞄准镜监视应用程序。

6. 使用虚拟GamePad窗口在地图周围浏览机器人：首先，单击左侧的Virtual GamePad，然后单击“连接”以在小部件上的后端。 选择键盘并使用“ WASD”键来浏览机器人。 有关更多信息，请参见[使用视觉的远程操纵杆](https://docs.nvidia.com/isaac/packages/navigation/doc/virtual-gamepad.html#virtual-gamepad)。

## 从WebSight中的应用程序查看输出
当应用程序运行时，通过导航到http：// localhost：3000，在浏览器中打开ISAAC瞄准镜。 如果要在Jetson上运行该应用程序，请使用Jetson系统的IP地址而不是Localhost。

如下所示，您应该看到类似的图片。 注意相机姿势3D视图中显示的彩色摄像头挫败感。

![](https://docs.nvidia.com/isaac/_images/elbrus_visual_slam_zed_image1.jpg)













