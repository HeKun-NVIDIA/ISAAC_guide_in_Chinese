# 在Python中开发Codelets

虽然就性能而言，编写小码的最佳语言是 C++，但并非应用程序的所有小码都需要使用相同的语言。 Isaac SDK 还支持 Python codelets，或 pyCodelets，适合那些更熟悉 Python 的人。 本节向您展示如何执行以下操作：

* 以 Isaac SDK 中包含的 ping_python 为例运行 Python codelets

* 创建 Python 小码

本节还介绍了使用 Python codelet 部署到目标系统的运行脚本，以及用于 C++ codelet 的 JSON 和 Bazel BUILD 文件与用于 Python codelet 的 JSON 和 Bazel BUILD 文件之间的差异。

## 运行 Python Codelet
可以在 apps/tutorials/ping_python/ 目录中找到[基础](https://docs.nvidia.com/isaac/doc/engine/alice.html#ping-cpp)部分中描述的 Ping codelet 的 Python 版本。

可以通过执行以下命令在您的系统上运行此应用程序：

```bash
bob@desktop:~/isaac/sdk$ bazel run //apps/tutorials/ping_python
```
如果您想在 Jetson 设备上运行该应用程序，您必须按照这些说明进行操作

1. 按照应用程序控制台选项中的说明将 `//apps/tutorials/ping_python:ping_python-pkg` 部署到机器人。

2. 使用以下命令切换到 Jetson 上已部署包的目录：
    ```bash
    bob@desktop:~/$ cd ~/deploy/bob/ping_python-pkg

    Where "bob" is your username on the host system.
    ```

3. 通过执行以下命令运行应用程序：

    ```bash
    bob@desktop:~/deploy/bob/ping_python-pkg/$ ./run apps/tutorials/ping_python/ping_python.py
    ```
当您通过任一方法运行小码时，“Hello World!” 每 1.5 秒打印一次消息。 修改 apps/tutorials/ping_python/ping_python.py 中的脚本并再次运行它以查看更改的效果。

**注意**

如果显示`“ImportError: No module named capnp”`错误，请确保使用在机器人上安装依赖项中提到的 `install_dependencies_jetson` 脚本安装了 `pycapnp`。

下面显示了[一个更完整的示例](https://docs.nvidia.com/isaac/doc/engine/alice.html#p-control-cpp)，即完整应用部分中描述的比例控制小代码的 Python 版本。 以下 Python 脚本在功能上等效于 main.cpp、ProportionalControlCpp.hpp 和 ProportionalControlCpp.cpp 的组合：

```python
'''
Copyright (c) 2019, NVIDIA CORPORATION. All rights reserved.

NVIDIA CORPORATION and its licensors retain all intellectual property
and proprietary rights in and to this software, related documentation
and any modifications thereto. Any use, reproduction, disclosure or
distribution of this software and related documentation without an express
license agreement from NVIDIA CORPORATION is strictly prohibited.
'''

from isaac import *


# A Python codelet for proportional control
# For comparison, please see the same logic in C++ at "ProportionalControlCpp.cpp".
#
# We receive odometry information, from which we extract the x position. Then, using refence and
# gain parameters that are provided by the user, we compute and publish a linear speed command
# using `control = gain * (reference - position)`
class ProportionalControlPython(Codelet):
    def start(self):
        # This part will be run once in the beginning of the program

        # Input and output messages for the Codelet. We'll make connections in the json file.
        self.rx = self.isaac_proto_rx("Odometry2Proto", "odometry")
        self.tx = self.isaac_proto_tx("StateProto", "cmd")    # DifferentialBaseControl type

        # Print some information
        print("Please head to the Sight website at <IP>:<PORT> to see how I am doing.")
        print("<IP> is the Internet Protocol address where the app is running,")
        print("and <PORT> is set in the config file, typically to '3000'.")
        print("By default, local link is 'localhost:3000'.")

        # We can tick periodically, on every message, or blocking. See documentation for details.
        self.tick_periodically(0.01)

    def tick(self):
        # This part will be run at every tick. We are ticking periodically in this example.

        # Try to get the odometry message.
        # Nothing to do if we haven't received odometry data yet
        rx_message = self.rx.message
        if rx_message is None:
            return

        # Read parameters that can be set through Sight webpage
        reference = self.config["desired_position_meters"]
        gain = self.config["gain"]

        # Read odometry message received
        position = rx_message.proto.odomTRobot.translation.x

        # Compute the control action
        control = gain * (reference - position)

        # Show some data in Sight
        self.show("reference (m)", reference)
        self.show("position (m)", position)
        self.show("control", control)
        self.show("gain", gain)

        # Publish control command
        tx_message = self.tx.init()
        data = tx_message.proto.init('data', 2)
        data[0] = control    # linear speed
        data[1] = 0.0    # This simple example sets zero angular speed
        self.tx.publish()


def main():
    app = Application(
        "apps/tutorials/proportional_control_python/proportional_control_python.app.json")
    app.nodes["py_controller"].add(ProportionalControlPython)
    app.connect('odometry/isaac.navigation.DifferentialBaseOdometry', 'odometry',
                'py_controller/PyCodelet', 'odometry')
    app.connect('py_controller/PyCodelet', 'cmd', 'commander.subgraph/interface',
                'control')
    app.run()


if __name__ == '__main__':
    main()
```

Python 中的 import 语句类似于 C++ 中的预处理器 #include 语句。 与 ProportionalControlCpp.cpp 一样，codelet 在 start 和 tick 函数中定义。 使用 Isaac 参数 desired_position_meters 和 gain，其值要么在 JSON 文件中配置，要么在运行时通过 Sight 设置。

在每次滴答时，如果收到里程计消息，则会为机器人计算并发布适当的命令。 一些重要数据显示在 Sight 中。

main 函数只是在运行应用程序之前加载图形和配置文件，就像 main.cpp 在 C++ codelet 中所做的那样。 此外，Python 主函数还为 PyCodelet 创建edges。


## 创建 Python Codelet
创建 Python codelet 时，请遵循类似于以下的过程。

### 创建工作区
将 `apps/tutorials/proportional_control_python/` 或另一个现有的基于 Python 的小代码复制到 `apps/<your_app_name>` 作为模板或起点。

如果您在本教程中使用未经修改的比例控制小代码，则需要 Carter 机器人或等效物。 有关详细信息，请参阅 [NVIDIA Carter](https://docs.nvidia.com/isaac/doc/tutorials/carter_hardware.html#carter-hardware)。

重命名文件以反映您的小代码的名称，而不是您复制的小代码。

### 创建 Bazel 构建文件
1. 在与用作起点的其他文件一起复制的` apps/<your_app_name>/BUILD` 中，将所有 `proportional_control_python` 字符串替换为 `<your_app_name>`。

2. 根据您使用的 C++ codelets 修改 `py_binary` 规则中的数据属性。

    例如，如果您要省略或删除 `apps/tutorials/proportional_control_python/BUILD` 中的 `//packages/segway`，并运行 codelet，则会显示错误 `Component with typename 'isaac::SegwayRmpDriver' not registered`，因为 `Proportional Control codelet  (proportional_control_python.graph.json) `使用基于 C++ 的segway codelet。

3. 由于我们的应用程序位于 apps 而不是 apps/tutorials 中，因此删除 `//apps/tutorials:py_init` 的规范，保留 `//apps:py_init`。

    如果不是将应用程序向上移动到 apps，而是将其移动到 `apps/tutorials/tutorials_sub`，`apps/tutorials/tutorials_sub` 中的 BUILD 文件必须在所有三个目录中指定 py_init，`//apps:py_init, //apps/tutorials:py_init `和`//apps/tutorials/tutorials_sub:py_init`。 每个目录还需要一份 `__init__.py`。


### 创建 Python Codelet
1. 在您的 `<your_app_name>.py` 中，将 `import apps.tutorials.proportional_control_python` 替换为` import apps.<your_app_name>`。

2. 根据需要重命名和修改 `ProportionalControlPython` 类。 您可以在此文件中定义多个 Python codelet。

3. 在主函数中，将所有 `proportional_control_python` 字符串替换为 `<your_app_name>`。 您必须使用类名注册所有 pyCodelet，例如我们用作起点的文件中的 `ProportionalControlPython。` 修改节点名称，在本例中为 `py_controller`，以匹配您在 `graph.json` 文件中选择的名称。

    您的主要功能将类似于以下内容：
    ```python
    def main():
    app = Application(app_filename = "my_new_app.app.json")
    app.register({"my_py_node1": PyCodeletType1, "my_py_node2a": PyCodeletType2, "my_py_node2b": PyCodeletType2})
    app.run()
    ```
4. 根据您的小码，在 `apps/<your_app_name>/<your_app_name>`.`app.json` 的图形部分添加或删除节点、组件或边。

5. 根据需要在 `apps/<your_app_name>/<your_app_name>.app.json` 的配置部分配置节点和组件。 确保用新小代码的名称替换小代码名称的所有实例。

如[运行 Python Codelet](https://docs.nvidia.com/isaac/doc/tutorials/ping_py.html#running-a-python-codelet) 中所述，在本地运行 codelet 或在 Jetson 系统上部署和运行它。

### 运行脚本
与包含一个或多个 Python codelet 的 Isaac 应用程序的部署（使用 deploy.sh）一起提供的运行脚本执行以下功能：

1. 检查 Python 脚本的文件名是否以“.py”结尾

2. 验证每个目录都有一个 __init__.py 文件

3. 使用以下命令运行应用程序。

```bash
PYTHONPATH=$PWD:$PWD/packages/pyalice:$PWD/packages/ml python
```


当我们使用以下命令时，这些功能由运行脚本执行：

```bash
./run apps/tutorials/proportional_control_python/proportional_control_python.py
```

## Python Codelets 的 JSON 和 BUILD 文件
Python codelet 的 JSON 文件与 C++ codelet 的 JSON 文件非常相似，只是 Python codelet 的组件类型始终是 `isaac::alice::PyCodelet`。

Bazel BUILD 文件有些不同，如下例所示：

```json
load("@com_nvidia_isaac_sdk//bzl:module.bzl", "isaac_pkg")

py_binary(
    name = "proportional_control_python",
    srcs = [
        "__init__.py",
        "proportional_control_python.py",
    ],
    data = [
        "proportional_control_python.config.json",
        "proportional_control_python.graph.json",
        "//apps:py_init",
        "//apps/tutorials:py_init",
        "//packages/navigation:libnavigation_module.so",
        "//packages/segway:libsegway_module.so",
        "//packages/sensors:libjoystick_module.so",
    ],
    deps = ["//packages/pyalice"],
)

isaac_pkg(
    name = "proportional_control_python-pkg",
    srcs = ["proportional_control_python"],
)

```

通过在 py_binary 规则的数据中指定相应的模块，可以使用 C++ codelets。 例如，`//packages/segway:libsegway_module.so` 是使用 `isaac::SegwayRmpDriver` 类型的 C++ Codelet 所必需的。 忽略或忘记此依赖项会导致在应用程序运行时显示错误 `Component with typename ‘isaac::SegwayRmpDriver’ not registered`。

`isaac_pkg` 规则负责打包所有文件并创建一个存档，该存档可以使用部署脚本传输到目标设备。





























































