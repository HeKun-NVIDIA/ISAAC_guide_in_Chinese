# Isaac机器人引擎简介

## 基础
本节介绍如何使用 Isaac 机器人引擎。 它介绍了相关术语并解释了 Isaac 应用程序的结构。

Isaac 应用程序由 JavaScript Object Notation (JSON) 文件定义。 以下示例位于` /sdk/apps/tutorials/ping/ping.app.json` 的 Isaac SDK 中，显示了基本格式。

```json
{
  "name": "ping",
  "modules": [
    "//apps/tutorials/ping:ping_components",
    "sight"
  ],
  "graph": {
    "nodes": [
      {
        "name": "ping",
        "components": [
          {
            "name": "ping",
            "type": "isaac::Ping"
          }
        ]
      }
    ],
    "edges": []
  },
  "config": {
    "ping" : {
      "ping" : {
        "message": "My own hello world!",
        "tick_period" : "1Hz"
      }
    }
  }
}
```
Isaac 应用程序由四个部分定义：

* `name` 是一个包含应用程序名称的字符串。

* `modules` 是正在使用的库列表。 在上面的示例中，我们包含 `//apps/tutorials/ping:ping_components` 以便`“apps/tutorials/ping/Ping.cpp”`可用。 Isaac SDK 带有高质量和经过良好测试的包，可以作为模块导入。 本教程后面将显示一个更长的模块示例。

* `graph`有两个小节来定义应用程序的功能：

    * `nodes` 是我们应用程序的基本构建块。 在这个简单的例子中，我们只有一个名为“ping”的节点，它只有一个组件。 请注意，此组件的类型 `isaac::Ping` 与 `apps/tutorials/ping/Ping.hpp` 的最后一行匹配。 一个典型的 Isaac 应用程序有多个节点，每个节点通常有多个组件。

    * `edges` 将组件连接在一起并使它们之间能够通信。 此示例不需要连接任何组件。

* `config` 允许您根据您的用例调整每个节点的参数。 在此示例中，指定“ping”分量应以 1 赫兹tick。


定义应用程序后，接下来您需要创建一个 makefile。 以下是与 ping 应用程序关联的 BUILD 文件。

```json
load("//bzl:module.bzl", "isaac_app", "isaac_cc_module")

isaac_cc_module(
    name = "ping_components",
    srcs = ["Ping.cpp"],
    hdrs = ["Ping.hpp"],
)

isaac_app(
    name = "ping",
    data = ["fast_ping.json"],
    modules = [
        "sight",
        "//apps/tutorials/ping:ping_components",
    ],
)
```

## Codelets
Codelets 是使用 Isaac 构建的机器人应用程序的基本构建块。 Isaac SDK 包括您可以在您的应用程序中使用的各种小代码。 下面的示例说明了如何导出Codelets。

为了更好地理解Codelets，请参阅[了解Codelets](https://docs.nvidia.com/isaac/doc/engine/components.html#understanding-codelets)部分。

以下 Ping.hpp 和 Ping.cpp 清单显示了位于 sdk/apps/tutorials/ping 的示例Codelets的源代码：

```cpp
// This is the header file located at sdk/apps/tutorials/ping/Ping.hpp

#pragma once

#include <string>

#include "engine/alice/alice_codelet.hpp"

namespace isaac {

// A simple C++ codelet that prints periodically
class Ping : public alice::Codelet {
 public:
  // Has whatever needs to be run in the beginning of the program
  void start() override;
  // Has whatever needs to be run repeatedly
  void tick() override;

  // Message to be printed at every tick
  ISAAC_PARAM(std::string, message, "Hello World!");
};

}  // namespace isaac

ISAAC_ALICE_REGISTER_CODELET(isaac::Ping);
```
```cpp
// This is the C++ file located at apps/tutorials/ping/Ping.cpp

#include "Ping.hpp"

namespace isaac {

void Ping::start() {
  // This part will be run once in the beginning of the program

  // We can tick periodically, on every message, or blocking. The tick period is set in the
  // json ping.app.json file. You can for example change the value there to change the tick
  // frequency of the node.
  // Alternatively you can also overwrite configuration with an existing configuration file like
  // in the example file fast_ping.json. Run the application like this to use an additional config
  // file:
  //   bazel run //apps/tutorials/ping -- --config apps/tutorials/ping/fast_ping.json
  tickPeriodically();
}

void Ping::tick() {
  // This part will be run at every tick. We are ticking periodically in this example.

  // Print the desired message to the console
  LOG_INFO(get_message().c_str());
}

}  // namespace isaac

```

## 完整的应用
下面显示的应用程序（位于 `/sdk/apps/tutorials/proportional_control_cpp`）功能更强大，具有具有多个节点的图、具有多个组件的节点、节点之间的边以及接收和传输消息的Codelets。

首先查看 JSON 文件以了解边缘是如何定义的。 请注意，此文件比上面的 ping 示例更长，但它遵循完全相同的语法。

```json
{
  "name": "proportional_control_cpp",
  "modules": [
    "//apps/tutorials/proportional_control_cpp:proportional_control_cpp_codelet",
    "navigation",
    "segway",
    "sight"
  ],
  "config": {
    "cpp_controller": {
      "isaac.ProportionalControlCpp": {
        "tick_period": "10ms"
      }
    },
    "segway_rmp": {
      "isaac.SegwayRmpDriver": {
        "ip": "192.168.0.40",
        "tick_period": "20ms",
        "speed_limit_angular": 1.0,
        "speed_limit_linear": 1.0,
        "flip_orientation": true
      },
      "isaac.alice.Failsafe": {
        "name": "robot_failsafe"
      }
    },
    "odometry": {
      "isaac.navigation.DifferentialBaseOdometry": {
        "tick_period": "100Hz"
      }
    },
    "websight": {
      "WebsightServer": {
        "port": 3000,
        "ui_config": {
          "windows": {
            "Proportional Control C++": {
              "renderer": "plot",
              "channels": [
                { "name": "proportional_control_cpp/cpp_controller/isaac.ProportionalControlCpp/reference (m)" },
                { "name": "proportional_control_cpp/cpp_controller/isaac.ProportionalControlCpp/position (m)" }
              ]
            }
          }
        }
      }
    }
  },
  "graph": {
    "nodes": [
      {
        "name": "cpp_controller",
        "components": [
          {
            "name": "message_ledger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "isaac.ProportionalControlCpp",
            "type": "isaac::ProportionalControlCpp"
          }
        ]
      },
      {
        "name": "segway_rmp",
        "components": [
          {
            "name": "message_ledger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "isaac.SegwayRmpDriver",
            "type": "isaac::SegwayRmpDriver"
          },
          {
            "name": "isaac.alice.Failsafe",
            "type": "isaac::alice::Failsafe"
          }
        ]
      },
      {
        "name": "odometry",
        "components": [
          {
            "name": "message_ledger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "isaac.navigation.DifferentialBaseOdometry",
            "type": "isaac::navigation::DifferentialBaseOdometry"
          }
        ]
      },
      {
        "name": "commander",
        "subgraph": "packages/navigation/apps/differential_base_commander.subgraph.json"
      }
    ],
    "edges": [
      {
        "source": "segway_rmp/isaac.SegwayRmpDriver/segway_state",
        "target": "odometry/isaac.navigation.DifferentialBaseOdometry/state"
      },
      {
        "source": "odometry/isaac.navigation.DifferentialBaseOdometry/odometry",
        "target": "cpp_controller/isaac.ProportionalControlCpp/odometry"
      },
      {
        "source": "cpp_controller/isaac.ProportionalControlCpp/cmd",
        "target": "commander.subgraph/interface/control"
      },
      {
        "source": "commander.subgraph/interface/command",
        "target": "segway_rmp/isaac.SegwayRmpDriver/segway_cmd"
      }
    ]
  }
}

```

文件 isaac_app 与 ping 示例非常相似。 但是，添加了此应用程序所需的模块，如“segway”。


```json
load("//bzl:module.bzl", "isaac_app", "isaac_cc_module")

isaac_cc_module(
    name = "proportional_control_cpp_codelet",
    srcs = ["ProportionalControlCpp.cpp"],
    hdrs = ["ProportionalControlCpp.hpp"],
    visibility = ["//visibility:public"],
    deps = [
        "//messages/state:differential_base",
        "//packages/engine_gems/state:io",
    ],
)

isaac_app(
    name = "proportional_control_cpp",
    data = [
        "//packages/navigation/apps:differential_base_commander_subgraph",
    ],
    modules = [
        "//apps/tutorials/proportional_control_cpp:proportional_control_cpp_codelet",
        "navigation",
        "segway",
        "sensors:joystick",
        "sight",
        "viewers",
    ],
)

```

下面的 ProportionalControlCpp 小码可以通过 `ISAAC_PROTO_RX `和 `ISAAC_PROTO_TX` 宏以及 JSON 文件中的关联边与其他组件进行通信。

```cpp
// This is the header file located at
// sdk/apps/tutorials/proportional_control_cpp/ProportionalControlCpp.hpp

  #pragma once

  #include "engine/alice/alice_codelet.hpp"
  #include "messages/differential_base.capnp.h"
  #include "messages/state.capnp.h"

  namespace isaac {

  // A C++ codelet for proportional control
  //
  // We receive odometry information, from which we extract the x position. Then, using refence and
  // gain parameters that are provided by the user, we compute and publish a linear speed command
  // using `control = gain * (reference - position)`
  class ProportionalControlCpp : public alice::Codelet {
   public:
    // Has whatever needs to be run in the beginning of the program
    void start() override;
    // Has whatever needs to be run repeatedly
    void tick() override;

    // List of messages this codelet recevies
    ISAAC_PROTO_RX(Odometry2Proto, odometry);
    // List of messages this codelet transmits
    ISAAC_PROTO_TX(StateProto, cmd);

    // Gain for the proportional controller
    ISAAC_PARAM(double, gain, 1.0);
    // Reference for the controller
    ISAAC_PARAM(double, desired_position_meters, 1.0);
  };

  }  // namespace isaac

  ISAAC_ALICE_REGISTER_CODELET(isaac::ProportionalControlCpp);

```

```cpp
// This is the C++ file located at
// apps/tutorials/proportional_control_cpp/ProportionalControlCpp.cpp

#include "ProportionalControlCpp.hpp"

#include "messages/math.hpp"
#include "messages/state/differential_base.hpp"
#include "packages/engine_gems/state/io.hpp"

namespace isaac {

void ProportionalControlCpp::start() {
  // This part will be run once in the beginning of the program

  // Print some information
  LOG_INFO("Please head to the Sight website at <IP>:<PORT> to see how I am doing.");
  LOG_INFO("<IP> is the Internet Protocol address where the app is running,");
  LOG_INFO("and <PORT> is set in the config file, typically to '3000'.");
  LOG_INFO("By default, local link is 'localhost:3000'.");

  // We can tick periodically, on every message, or blocking. See documentation for details.
  tickPeriodically();
}

void ProportionalControlCpp::tick() {
  // This part will be run at every tick. We are ticking periodically in this example.

  // Nothing to do if we haven't received odometry data yet
  if (!rx_odometry().available()) {
    return;
  }

  // Read parameters that can be set through Sight webpage
  const double reference = get_desired_position_meters();
  const double gain = get_gain();

  // Read odometry message received
  const auto& odom_reader = rx_odometry().getProto();
  const Pose2d odometry_T_robot = FromProto(odom_reader.getOdomTRobot());
  const double position = odometry_T_robot.translation.x();

  // Compute the control action
  const double control = gain * (reference - position);

  // Show some data in Sight
  show("reference (m)", reference);
  show("position (m)", position);
  show("control", control);
  show("gain", gain);

  // Publish control command
  messages::DifferentialBaseControl command;
  command.linear_speed() = control;
  command.angular_speed() = 0.0;  // This simple example sets zero angular speed
  ToProto(command, tx_cmd().initProto(), tx_cmd().buffers());
  tx_cmd().publish();
}

}  // namespace isaac

```




















