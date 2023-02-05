# ROS
Isaac 和[机器人操作系统](https://www.ros.org/) (ROS) 都使用消息传递来处理各自系统不同部分之间的通信。 Isaac 和 ROS 之间的通信需要在两个系统之间创建消息转换层。 本节介绍该层的概述和工作流程。

## 安装ROS
为了使用 ROS Bridge，您必须在您的设备上安装 ROS。 例如，如果你想在 Jetson 上使用 ROS Bridge，你必须在 Jetson 上安装 ROS。 如果你想在桌面上使用 ROS，你必须在桌面上安装 ROS。

您安装的 ROS 版本必须与您设备的操作系统相匹配。 对于您的桌面系统或 Jetson Xavier（均运行 Ubuntu 18.04），您必须安装 ROS Melodic Morenia。

要安装 ROS，请按照 [ROS 网页](http://wiki.ros.org/ROS/Installation)上的说明进行操作。

## 创建和使用自定义 ROS 包
Isaac 不需要大部分 ROS 功能，因此包含的软件包仅提供对通常随默认 ROS 安装一起安装的消息的支持，如下所列。
```json
roscpp rospy actionlib_msgs control_msgs diagnostic_msgs geometry_msgs
map_msgs nav_msgs pcl_msgs sensor_msgs shape_msgs std_msgs stereo_msgs
tf2_geometry_msgs tf2_msgs trajectory_msgs visualization_msgs
```
ROS Bridge 所需的第三方库仅在桌面和 Jetson Xavier 上受支持。

如果上述消息类型足以满足您的应用，请继续进行下一小节。 否则，为了使用自定义 ROS 消息，有必要生成自定义包并将 bazel 构建系统指向这样的包。 首先生成一个包含所需包的自定义工作区。 请参阅 ./engine/engine/build/scripts/ros_package_generation.sh 以获取有关如何创建具有正确路径结构的自定义包的指南。 一旦这样的包存在，修改 third_party/ros.bzl 以指向新的工作区。

首先注释掉平台特定的默认包，它应该类似于以下内容：

```cpp
isaac_http_archive(
    name = "isaac_ros_bridge_x86_64",
    build_file = clean_dep("//third_party:ros.BUILD"),
    sha256 = "b2a6c2373fe2f02f3896586fec0c11eea83dff432e65787f4fbc0ef82100070a",
    url = "https://developer.nvidia.com/isaac/download/third_party/ros_kinetic_x86_64.tar.gz",
)
```
将特定于平台的包替换为以下内容以指向包含自定义包的新工作区：

```cpp
isaac_new_local_repository(
    name='isaac_ros_bridge_x86_64',
    path='path to the workspace',
    build_file = clean_dep("//third_party:ros.BUILD"),
)
```
新部分中的 name 和 build_file 字段必须与被替换部分的字段相匹配。

## 创建 ROS Bridge
“ros_bridge”包提供了一个消息转换器库，可以很容易地创建新的。 得益于模块化设计，用户可以创建具有不同连接和转换器配置的各种桥接器，并在需要时轻松创建新的转换器。 虽然本节的其余部分将解释和构建此包，但如果用户想直接使用 [roscpp](http://wiki.ros.org/roscpp)，示例应用程序“[navigation_rosbridge](https://docs.nvidia.com/isaac/doc/tutorials/samples.html#navigation-rosbridge)”的小码说明了如何执行此操作。

Isaac-ROS bridge包括：

1. 一个且只有一个 RosNode codelet，

2. 与 RosNode 小码位于同一节点中的 TimeSynchronizer 小码，

3. 根据需要尽可能多的消息和姿势转换器小码，

4. 一旦 RosNode 与 roscore 建立连接，就会启动转换器的行为树。

## Ros节点
RosNode 是初始化 ROS 节点并等待 roscore 启动的 Isaac codelet。 每个带有 ROS 桥的 Isaac 应用程序都需要有一个且只有一个节点具有这种类型的单个组件。

## 时间同步器
允许在 Isaac 表示法和 ROS 表示法之间转换时间戳。

## 消息转换器基地
Isaac 为用户提供了基类来快速开发典型的转换器：

1. ProtoToRosConverter：这是将 Isaac 原型消息转换为 ROS 消息并将其发布到 ROS 的小码的基类。 请检查 OdometryToRos 转换器以查看有关如何使用 ProtoToRosConverter 创建新转换器的示例：我们需要做的就是定义一个 protoToRos 函数。

2. RosToProtoConverter：这是接收 ROS 消息并将其转换为 Isaac 原型消息的小码的基类。 请检查 RosToDifferentialBaseCommand 转换器以查看有关如何使用 RosToProtoConverter 创建新转换器的示例：我们需要做的就是定义一个 rosToProto 函数。

## 姿势同步
Isaac [姿势树](https://docs.nvidia.com/isaac/doc/engine/components.html#pose-tree)和 ROS [tf2](http://wiki.ros.org/tf2) 可以使用下面的代码同步：

1. PosesToRos：对于姿势映射列表，此小代码从 Isaac Pose Tree 读取姿势并将其写入 ROS tf2。

2. RosToPoses：对于姿势映射列表，此小代码从 ROS tf2 读取转换并将它们写入 Isaac 姿势树。

## 自定义小码
如果所需的小码与上述小码或基类不匹配，您可以轻松创建更高级的小码。 例如，GoalToRosAction 接收到两条 Isaac 消息，运行一个 ROS 动作客户端，并发布一条 Isaac 消息。

## 示例：将 ROS 导航堆栈与 Isaac 结合使用
请查看 `packages/ros_bridge/apps/ros_navigation/ros_bridge.subgraph.json` 以获取有关如何使用 RosNode 和转换器小代码的示例。 请注意，行为树确保转换器在 RosNode 完成初始化 ROS 连接后启动。

此示例子图用于通过 Isaac 模拟器或通过 Isaac 与真实机器人一起运行 [ROS 导航堆栈](http://wiki.ros.org/navigation)。 让我们看看 TurtleBot 3 Waffle Pi 由 Flatsim 中的 TurtleBot 3 的 ROS 导航堆栈导航。

1. 除了 ROS 之外，还为 [TurtleBot 3](http://emanual.robotis.com/docs/en/platform/turtlebot3/overview/) 安装 [ROS 导航堆栈](http://wiki.ros.org/turtlebot3_navigation)。

2. 按照 Virtual Navigation with TurtleBot3 中的说明，为 Isaac 小型仓库场景运行以下命令：
    ```bash
    bob@desktop:~/isaac/sdk$ TURTLEBOT3_MODEL=waffle_pi roslaunch turtlebot3_navigation turtlebot3_navigation.launch map_file:=$(realpath packages/ros_bridge/maps/small_warehouse.yaml)

    ```

    您不需要自己启动 roscore，roslaunch 会为您完成。

3. 运行具有 flatsim 和 ROS 桥的 Isaac 应用程序：
    ```bash
    bob@desktop:~/isaac/sdk$ bazel run packages/ros_bridge/apps:ros_to_navigation_flatsim -- --more apps/assets/maps/virtual_small_warehouse.json --config ros_navigation:packages/ros_bridge/maps/small_warehouse_map_transformation.config.json,ros_navigation:packages/ros_bridge/apps/ros_to_navigation_turtlebot3_waffle_pi.config.json
    ```
4. 打开 http://localhost:3000/ 通过 Isaac 进行监控。 通过 ROS 监视 RViz 窗口以进行监视。 请注意，RViz 可能会抱怨“没有从 `[wheel_left_link]` 到 `[map]` 的转换”。 此信息不由桥提供，因为它不被 ROS 导航堆栈使用。 但是，如果需要，它（和其他数据）可以像其他转换一样发布。

5. 机器人现在应该导航到目标，可以通过拖动 Sight 上“地图”窗口的“pose_as_goal”标记轻松修改。


**注意**

由于 ROS 导航堆栈中的问题，ROS 可能会打印如下警告。

```bash
Warning: Invalid argument "/map" passed to canTransform argument target_frame in tf2 frame_ids cannot start with a '/' like:
at line 134 in /tmp/binarydeb/ros-melodic-tf2-0.6.5/src/buffer_core.cpp
```
在这种情况下，请应用 https://github.com/ROBOTIS-GIT/turtlebot3/pull/402/files 中显示的更改，即从 global_costmap_params.yaml 和 local_costmap_params.yaml 的框架名称中删除“/”字符。 如果您安装了 turtlebot3 包，这些文件可能存在于 /opt/ros/melodic/share/turtlebot3_navigation/param/，或者存在于您克隆 turtlebot3 存储库的位置。 有关详细信息，请查看 https://github.com/ros-planning/navigation/issues/794 上的讨论。

**注意**


如果 turtlebot3_navigation 没有为您安装 dwa-local-planner，ROS 可能会运行失败并显示以下消息：
```bash
[FATAL] [1567169699.791910573]: Failed to create the dwa_local_planner/DWAPlannerROS planner, are you sure it is properly registered and that the containing library is built? Exception: According to the loaded plugin descriptions the class dwa_local_planner/DWAPlannerROS with base class type nav_core::BaseLocalPlanner does not exist. Declared types are  base_local_planner/TrajectoryPlannerROS
```
在这种情况下，为您的 ROS 发行版安装缺少的依赖项，如下所示：
```bash
sudo apt install ros-<your_distro>-dwa-local-planner

```
例如，Melodic Morenia 的命令是
```bash
sudo apt install ros-melodic-dwa-local-planner
```

## 在此示例上构建
* 要使用 isaac_sim_unity3d 而不是 Flatsim 进行模拟，请通过运行以下命令启动小型仓库场景：
    ```bash
    bob@desktop:~isaac_sim_unity3d/builds$ ./sample.x86_64 --scene small_warehouse --scenarioFile ~/isaac/sdk/packages/navsim/scenarios/turtlebot3_waffle_pi.json --scenario 0
    ```

* 然后，在单独的终端上输入以下命令以运行与 Unity 和 ROS 通信的应用程序：
    ```
    bashbob@desktop:~/isaac/sdk$ bazel run packages/ros_bridge/apps:ros_to_navigation_unity3d -- --more apps/assets/maps/virtual_small_warehouse.json --config ros_navigation:packages/ros_bridge/maps/small_warehouse_map_transformation.config.json, ros_navigation:packages/ros_bridge/apps/ros_to_navigation_turtlebot3_waffle_pi.config.json
    ```

* 使用与上述相同的命令运行 ROS：
    ```bash
    bob@desktop:~/isaac/sdk$ TURTLEBOT3_MODEL=waffle_pi roslaunch turtlebot3_navigation turtlebot3_navigation.launch 地图文件:=$(realpath packages/ros_bridge/maps/small_warehouse.yaml)
    ```
    和以前一样，可以通过拖动 Sight 中的“pose_as_goal”标记来修改目标。

* 要在不同的地图上运行，只需在上面的命令中更改地图文件的路径（如果您不使用 flatsim，则在模拟器窗口中更改）。

* 要模拟不同的机器人，请在上面的命令中更改机器人配置（如果您不使用 flatsim，则在模拟器窗口中更改）。 在这种情况下，还应更新 ROS 导航堆栈。


## 将 Isaac 映射转换为 ROS 映射
Isaac 和 ROS 之间的地图文件扩展名和地图框架约定不同：

* Isaac 使用便携式网络图形 (.png) 地图，而 ROS 导航堆栈使用便携式灰度图格式 (.pgm)。

* (0, 0)坐标对应Isaac map的左上角，x方向指向下方，而ROS的map_server的frame依赖于.yaml文件的origin参数。

以下是生成供 ROS 使用的 .yaml 和 .pgm 文件的步骤。 反向转换类似。

1. 转换 Isaac 地图图像：
    ```bash
    bob@desktop:~/isaac/sdk$ convert apps/assets/maps/virtual_small_warehouse.png -flatten packages/ros_bridge/maps/small_warehouse.pgm
    ```
    如果愿意，您也可以在此命令中使用旋转标志。 我们将在步骤 3 中处理框架转换。

2. 创建一个 packages/ros_bridge/maps/small_warehouse.yaml 文件，读取
    ```bash
    image: ./small_warehouse.pgm
    resolution: 0.1
    origin: [0.0, 0.0, 0.0]
    negate: 0
    occupied_thresh: 0.65
    free_thresh: 0.196
    ```

    您可以根据需要修改原点。 但是，分辨率字段应与 apps/assets/maps/virtual_small_warehouse.json 中的 cell_size 匹配。

3. 找到从Isaac map frame到ROS map frame的正确转换创建packages/ros_bridge/maps/small_warehouse_map_transformation.config.json：
    ```json
    {
    "pose_initializers": {
        "map_transformation": {
          "lhs_frame": "ros_map",
          "rhs_frame": "map",
          "pose": {
            "rotation": { "yaw_degrees": 180 },
            "translation": [19.3, 24.3, 0.0]
          }
        }
    }
    }

    ```

**注意**

[ROS Answers](https://answers.ros.org/question/69019/how-to-point-and-click-on-rviz-map-and-output-the-position/?answer=69295#post-id-69295) 中描述了找到这种转换的一种方法：

1. 启动 ROS 导航：
```bash
bob@desktop:~/isaac/sdk$ TURTLEBOT3_MODEL=waffle_pi roslaunch turtlebot3_navigation turtlebot3_navigation.launch map_file:=$(realpath packages/ros_bridge/maps/small_warehouse.yaml)

```

2. 在单独的终端中，键入以下内容：
```bash
bob@desktop:~/isaac/sdk$ rostopic echo /move_base_simple/goal

```
3. 给出对应于上述 Isaac 地图框架的 2D 导航目标（即地图的左上角，指向下方）。 用rostopic在终端上打印的pose是ros_map_T_map。



































