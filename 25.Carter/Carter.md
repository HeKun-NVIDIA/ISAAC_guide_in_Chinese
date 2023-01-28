# Carter机器人

Carter 是一个机器人平台，使用 Segway 的差分底座、用于 3D 范围扫描的 Velodyne P16、ZED 相机、IMU 和 Jetson TX2 作为系统的核心。 与定制安装支架一起，它为 Isaac 导航堆栈提供了一个强大而稳健的演示平台。

![](https://docs.nvidia.com/isaac/_images/image16.png)


运行 Carter 应用程序
1. 按照在 Carter 上设置软件部分的详细信息设置 Carter 机器人。

    **注意**

    这些 Carter 软件设置说明特定于 Carter 2.0 版。 有关 Carter 1.0 版说明，请参阅 Isaac SDK 2020.2 文档 <https://docs.nvidia.com/isaac/archive/2020.2/doc/index.html>。

2. 构建一个 ARM 目标并使用以下命令将其部署到机器人：

    ```bash
    bob@desktop:~/isaac/sdk$ ./../engine/engine/build/deploy.sh --remote_user <username> -p //apps/carter:carter-pkg -d jetpack45 -h <robot_ip_address>
    ```
    其中 `<username>` 是您在机器人上的用户名（默认为 nvidia），`<robot_ip_address> `是机器人的 IP 地址。

3. 使用以下命令通过 SSH 连接到机器人：

    ```bash
    bob@desktop:~/isaac/sdk$ ssh <username>@<robot_ip_address>

    ```
    其中 `<username>` 是您在机器人上的用户名（默认为 nvidia），`<robot_ip_address>` 是机器人的 IP 地址。

4. 使用以下命令在机器人上运行 carter 应用程序：

    ```bash
    bob@jetson:~/$ ./run apps/carter/carter.py --map_json <map_file> --robot_json <robot_file>
    ```

**注意**

默认情况下，机器人会尝试驶向随机选择的目标。 密切监视机器人并按下游戏手柄上的 deadman 开关以激活机器人。 有关详细信息，请参阅操纵杆部分。

## Carter 应用节点
carter 应用程序使用导航堆栈中的所有节点并添加以下特定于机器人的节点：

* **LidarDriver**：从网络端口上的 Velodyne P16 接收 3D 范围扫描并将其发布为 LidarScan 消息的节点。

* **RangeScanFlattening**：将激光雷达消息展平为平面二维范围扫描，由导航堆栈使用。

* **SegwayRMPDriver**：与 Segway 硬件通信，用于向 Segway 发送控制命令并接收从 Segway 传回的车轮里程计数据。

* **操纵杆**：一种通用的操纵杆驱动程序，可从连接的游戏手柄发布用户输入。

* **SegwayJoystick**：使用操纵杆数据，检查 deadman 开关并根据用户输入决定自动或手动控制模式。

![](https://docs.nvidia.com/isaac/_images/image22.png)














































