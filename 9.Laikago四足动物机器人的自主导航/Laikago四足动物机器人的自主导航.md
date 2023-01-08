# Laikago 四足机器人的自主导航

开发智能机器人系统是一项多学科的工作，集成了动力学、控制、计算机视觉、人工智能等。 很难掌握所有这些领域。 即使你掌握了所有这些，也需要花费大量时间才能正确和稳健。

为了帮助机器人专家加速智能机器人的开发，NVIDIA Isaac SDK 包含参考应用程序和平台。 其中一个平台是 Kaya，一种三轮完整的自主机器人。 Laikago 应用程序是使用 Kaya 作为参考构建的，以创建可以导航和避开障碍物的自主机器。

Laikago 是由 Unitree Robotics 制造的四足机器人。 它具有用于在微控制器单元 (MCU) 中行走和平衡的运动控制算法。 它还提供一个作为可选包的安装座，其中包括用于大脑的 [NVIDIA Jetson TX2](https://developer.nvidia.com/embedded/jetson-tx2) 模块，允许用户开发自定义软件并访问运动控制和传感器数据。 开箱即用，它没有任何用于映射、定位或避障的传感器或软件。

此应用程序使用 Velodyne VLP-16 激光雷达进行感知，并将 Isaac SDK 导航堆栈与 Unitree Robotics API 集成。 所有计算都在 TX2 内部完成。 运动控制器以 500Hz 运行，而导航堆栈需要 50% 的 CPU。 运行此应用程序时，Laikago 以 0.6 m/s 的峰值速度行走。
![](https://docs.nvidia.com/isaac/_images/laikago_photo.jpg)


## 升级硬件
您需要将激光雷达和接口盒安装到 Laikago。 我们根据 Unitree GitHub 存储库中的 Laikago CAD 文件设计了 3D 打印支架。 Laikago 提供 19V 输出，高于 VLP-16 的工作电压。

**注意**

最新的 VLP-16 支持高达 32V 的规格。

我们建议使用 DC-DC 转换器来降低电压，并使用 USB 转以太网将激光雷达传感器连接到 TX2。 您还可以安装一个兼容的摄像头用于物体检测，但在这个示例应用程序中没有应用。 下图显示了整体设置。
![](https://docs.nvidia.com/isaac/_images/laikago_vlp16.jpg)


Isaac SDK 导航和感知堆栈与 Jetson 板的传感器品牌和类型无关。 例如，Kaya 使用相同的导航堆栈，但在 Jetson Nano 上运行，并使用摄像头而不是激光雷达传感器进行定位。 Kaya 的许多感知算法也适用于简单的网络摄像头。

## 软件概述
此应用程序主要使用 Isaac SDK 导航堆栈，其中包括地图、定位、全局路径规划、控制、避障、里程计和路径跟踪。 Isaac SDK 还包括激光雷达驱动程序和 Laikago SDK，因此不需要额外的库或依赖项。

下图显示了设计层次结构。 所有圆框都包含在 Isaac SDK 中。 矩形框指定机器人硬件。 Laikago 驱动主要用于将 Isaac SDK 的消息传递给 Laikago SDK。
![](https://docs.nvidia.com/isaac/_images/laikago_app_graph.png)


## 运行 Laikago 导航应用程序
1. 确保 Jetson 设备已按照[设置](https://docs.nvidia.com/isaac/doc/setup.html#setup-isaac)文档中的详细信息进行设置。

2. 构建一个 ARM 目标并使用以下命令将其部署到机器人：

    ```bash
    bob@desktop:~/isaac/sdk$ ./../engine/engine/build/deploy.sh --remote_user <username> -p //packages/laikago/apps:laikago_navigate-pkg -d jetpack45 -h <robot_ip_address>
    ```

    其中 `<username> `是您在机器人上的用户名（默认为 nvidia），`<robot_ip_address>` 是机器人的 IP 地址。

3. 使用以下命令通过 SSH 连接到机器人：

    ```bash
    bob@desktop:~/isaac/sdk$ ssh <username>@<robot_ip_address>
    ```
    其中 `<username>` 是您在机器人上的用户名（默认为 nvidia），`<robot_ip_address>` 是机器人的 IP 地址。

4. 使用以下命令在机器人上运行 Laikago 应用程序：

    ```bash
    bob@jetson:~/$ ./packages/laikago/apps/laikago_navigate --config <map_config_json> --graph <map_graph_json>
    ```
    `<map_config_json>` 和 `<map_graph_json>` 是地图文件。 `apps/assets/maps` 文件夹中提供了示例。

    连接到蓝牙操纵杆控制器。 我们在此示例中使用 NVIDIA Shield 控制器。 这可用于向 Laikago 发送定向命令并触发自主导航模式。

    **注意**

    默认情况下，机器人处于“站立”模式。 当方向命令通过一个小阈值时，机器人将开始行走。 有关详细信息，请参阅[操纵杆](https://docs.nvidia.com/isaac/packages/sensors/doc/joystick.html#joystick)部分。

6. 在 `<robot_ip>:3000` 的浏览器中打开 Isaac Sight。 您应该会看到 Laikago 所在的地图。 使用操纵杆移动 Laikago 并观察地图更新。

















































