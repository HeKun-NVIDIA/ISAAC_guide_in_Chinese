# 在 C++ 中开发 Codelet

本教程的目标是用 C++ 开发两个小码：第一个实际上是一台“ping”的机器，而第二个侦听并摄取“ping”消息。 对于本教程，不需要外部依赖项或特殊硬件。

## 创建新应用程序
每个 Issac Robotics 应用程序都需要一个应用程序 JSON 文件和一个 Bazel 构建文件。 在编写小码之前，我们将首先创建这些文件。

### 为应用程序创建一个新目录
使用类似于以下的命令在 `isaac/sdk/packages` 目录中创建一个文件夹：

```bash
bob@desktop:~/isaac/sdk/packages/$ mkdir ping
```


对于本教程的其余部分，每当您被要求创建一个新文件时，请将其直接放入此文件夹中。 更复杂的包将有子文件夹，但对于本教程，一切都保持简单。

### 创建应用程序 JSON 文件
每个 Isaac 应用程序都基于一个 JSON 文件，它描述了应用程序的依赖关系、节点图和消息流； 它还包含自定义配置数据。 创建一个名为 ping.app.json 的新 JSON 文件并指定其名称，如以下代码片段所示：

```json
{
  "name": "ping"
}

```

### 创建 Bazel 构建文件
接下来，创建用于编译和运行应用程序的 Bazel 构建文件。 Bazel 为大型项目提供了非常好的依赖管理和出色的构建速度，而且 `Bazel` 构建文件非常容易编写。 使用名为 `ping` 的新应用程序目标创建名为 BUILD 的文件，如下所示：

```json
load("@com_nvidia_isaac_sdk//bzl:module.bzl", "isaac_app", "isaac_cc_module")

isaac_app(
     name = "ping"
)

```

### 使用 Bazel 构建应用程序
现在您可以通过在 sdk 目录中运行以下命令来构建应用程序：
```bash
bob@desktop:~/isaac/sdk/$ bazel build packages/ping:ping
```
该命令可能需要一些时间，因为 Isaac 机器人引擎的所有外部依赖项都已下载并编译。 一段时间后，第一次构建应该会成功，输出类似于以下内容：

```bash
bob@desktop:~/isaac/sdk/$ bazel build packages/ping:ping
INFO: Analyzed target //packages/ping:ping (69 packages loaded, 4167 targets configured).
INFO: Found 1 target...
Target //packages/ping:ping up-to-date:
  bazel-bin/packages/ping/run_ping
  bazel-bin/packages/ping/ping
INFO: Elapsed time: 10.602s, Critical Path: 8.60s
INFO: 3 processes: 3 linux-sandbox.
INFO: Build completed successfully, 8 total actions
```
接下来，您可以通过执行以下命令来运行您的新应用程序：

```bash
bob@desktop:~/isaac/sdk/$ bazel run packages/ping:ping
```

这将启动 ping 应用程序并使其保持运行。 您可以通过在控制台中按 **Ctrl+C**来停止正在运行的应用程序。 这将正常关闭应用程序。

您会注意到并没有发生太多事情，因为我们还没有应用程序图。 接下来我们将为应用程序创建一些节点。

## 创建节点
Isaac 应用程序由许多并行运行的节点组成。 它们可以使用 Isaac 机器人引擎提供的各种其他机制相互发送消息或相互交互。 节点是轻量级的，不需要自己的进程，甚至不需要自己的线程。

要自定义 ping 节点的行为，我们必须为其配备组件。 我们将创建我们自己的组件，称为“Ping”。

### 创建codelet.hpp 文件
在 ping 目录下新建一个名为 Ping.hpp 的文件，内容如下：

```cpp
#pragma once
#include "engine/alice/alice_codelet.hpp"
class Ping : public isaac::alice::Codelet {
 public:
  void start() override;
  void tick() override;
  void stop() override;
};
ISAAC_ALICE_REGISTER_CODELET(Ping);

```
Codelets 提供三个可以重载的主要函数：start、tick 和stop。 当一个节点启动时，首先调用所有附加的小码的启动函数。 例如，start 是分配资源的好地方。 您可以将小代码配置为周期性地或每次收到新消息时打勾。 大多数功能随后由 tick 函数执行。

最后，当节点停止时，将调用停止函数。 您应该在停止功能中释放所有先前分配的资源。 不要使用构造函数或析构函数：您无权访问构造函数中的任何 Isaac 机器人引擎功能（例如配置）。

您创建的每个自定义小代码都需要在 Isaac 机器人引擎中注册。 这是在文件末尾使用 ISAAC_ALICE_REGISTER_CODELET 宏完成的。 如果您的小代码在命名空间内，则必须提供完全限定的类型名称，例如 ISAAC_ALICE_REGISTER_CODELET(foo::bar::MyCodelet);。

### 创建codelet.cpp 文件
要向 codelet 添加一些功能，请创建一个名为 Ping.cpp 的源文件，其中包含以下功能：
```cpp
#include "Ping.hpp"
void Ping::start() {}
void Ping::tick() {}
void Ping::stop() {}

```
### 定义 tick() 行为
小码可以用不同的方式tick，但现在我们将使用周期性tick，这可以通过调用 Ping::start 函数中的 tickPeriodically 函数来实现。 在 Ping.cpp 的启动函数中添加如下代码：

```cpp
void Ping::start() {
  tickPeriodically();
}
```
### 添加日志消息
为了验证确实发生了某些事情，我们将在小码tick时打印一条消息。 Isaac SDK 包括用于记录数据的实用函数； LOG_INFO 可用于在控制台上打印消息。 它遵循 printf 风格的语法。 在Ping.cpp中添加tick函数如下图：
```cpp
void Ping::tick() {
  LOG_INFO("ping");
}
```
### 将组件添加到 BUILD 文件
将组件作为模块添加到BUILD文件中，如下图：

```cpp
isaac_app(
  ...
)

isaac_cc_module(
  name = "ping_components",
  srcs = ["Ping.cpp"],
  hdrs = ["Ping.hpp"],
)
```
Isaac 模块定义了一个共享库，它封装了一组小码，可以被不同的应用程序使用。

### 向 JSON 文件添加一个新节点
要在应用程序中使用 Ping codelet，我们首先需要在应用程序 JSON 文件中创建一个新节点：

```json
{
  "name": "ping",
  "graph": {
    "nodes": [
      {
        "name": "ping",
        "components": []
      }
    ],
    "edges": []
  }
}

```
### 将 Ping 小代码添加到节点
每个节点都可以包含多个组件，这些组件定义了它的功能。 通过在组件数组中添加一个新部分，将 Ping 小代码添加到节点：

```json

{
  "name": "ping",
  "graph": {
    "nodes": [
      {
        "name": "ping",
        "components": [
          {
            "name": "ping",
            "type": "Ping"
          }
         ]
      }
    ],
    "edges": []
  }
}
```

应用程序图通常具有连接不同节点的边，这决定了节点之间的消息传递顺序。 因为此应用程序没有任何其他节点，所以我们将边缘留空。

### 将组件添加到模块列表
如果您尝试运行此应用程序，它会崩溃并显示错误消息 `Could not load component ‘Ping’`。 发生这种情况是因为应用程序中使用的所有组件都必须添加到模块列表中。 您需要在 BUILD 文件和应用程序 JSON 文件中执行此操作：
```json

load("@com_nvidia_isaac_sdk//bzl:module.bzl", "isaac_app", "isaac_cc_module")

isaac_app(
    name = "ping",
    modules = ["//packages/ping:ping_components"]
)
```

```json
{
  "name": "ping",
  "modules": [
    "//packages/ping:ping_components"
  ],
  "graph": {
    ...
  }
}

```

如果您现在运行该应用程序，您将收到一条不同的恐慌消息：“`未找到参数‘ping/ping/tick_period’或类型错误`”。 出现此消息是因为我们需要在配置部分设置 Ping codelet 的滴答周期。 我们将在下一节中执行此操作。

## 配置
大多数代码需要各种参数来自定义行为。 例如，您可能希望为我们的 ping 机器的用户提供更改tick周期的选项。 在 Isaac 框架中，这可以通过配置来实现。

让我们在应用程序 JSON 文件的“配置”部分指定节拍周期，以便我们最终可以运行应用程序。

```json
{
  "name": "ping",
  "modules": [
    "//packages/ping:ping_components"
  ],
  "graph": {
    ...
  },
  "config": {
    "ping" : {
      "ping" : {
        "tick_period" : "1Hz"
      }
    }
  }
}

```
每个配置参数都由三个元素引用：节点名称、组件名称和参数名称。 在这种情况下，我们在节点 ping 中设置组件 ping 的参数 tick_period。

**注意**

配置值必须与组件 API 中指定的数据类型相匹配。 有关预期的数据类型，请参阅组件 API 概述或component.hpp 文件。 另请注意，整数值被接受为一种双精度值。

现在应用程序将成功运行并每秒打印一次 ping。 您应该会看到类似于下面代码片段的输出。 您可以按 **Ctrl+C** 正常停止应用程序。

```bash
bob@desktop:~/isaac/sdk/packages/ping$ bazel run ping
2019-03-24 17:09:39.726 DEBUG   engine/alice/backend/codelet_backend.cpp@61: Starting codelet 'ping/ping' ...
2019-03-24 17:09:39.726 DEBUG   engine/alice/backend/codelet_backend.cpp@73: Starting codelet 'ping/ping' DONE
2019-03-24 17:09:39.726 DEBUG   engine/alice/backend/codelet_backend.cpp@291: Starting job for codelet 'ping/ping'
2019-03-24 17:09:39.726 INFO    packages/ping/Ping.cpp@8: ping
2019-03-24 17:09:40.727 INFO    packages/ping/Ping.cpp@8: ping
2019-03-24 17:09:41.726 INFO    packages/ping/Ping.cpp@8: ping
```
### 向codelet  .hpp 文件添加新参数
tick_period 参数是自动为我们创建的，但我们也可以创建自己的参数来自定义小码的行为。 将参数添加到您的小代码，如下所示：

```cpp
class Ping : public isaac::alice::Codelet {
 public:
  void start() override;
  void tick() override;
  void stop() override;
  ISAAC_PARAM(std::string, message, "Hello World!");
};

```
ISAAC_PARAM 采用三个参数：

* 参数的类型，通常为 double、int、bool 或 std::string。

* 参数名称，用于访问或指定参数。

* 参数的默认值。 如果没有给出默认值，并且没有通过配置文件指定参数，则程序会在访问参数时断言。

ISAAC_PARAM 宏创建一个名为 get_message 的访问器和更多代码以将参数与系统的其余部分正确连接。

### 使用 codelet .hpp 文件中的新参数
我们现在可以在 tick() 函数中使用参数而不是硬编码值。 调用 get_message() 以检索消息参数的值：
```cpp

void tick() {
  LOG_INFO(get_message().c_str());
}
```

### 配置JSON文件中的参数
下一步是为节点添加配置。 使用节点名称 (ping)、组件名称 (ping) 和参数名称 (message) 来指定所需的值。

```json
{
  "name": "ping",
  "modules": [
    "//packages/ping:ping_components"
  ],
  "graph": {
    ...
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

就是这样！ 您现在有一个可以定期打印自定义消息的应用程序。 使用以下命令运行应用程序：

```bash
bob@desktop:~/isaac/sdk/packages/ping$ bazel run ping
```

正如预期的那样，codelet 会在命令行上定期打印消息。

## 发送信息
自定义小码 Ping 现在正在愉快地滴答作响。 为了让其他节点对 ping 做出反应，Ping 小码必须发送其他小码可以接收的消息。

### 将 ISAAC_PROTO_TX 宏添加到小代码 .hpp 文件
发布消息很容易。 使用 ISAAC_PROTO_TX 宏来指定小码正在发布消息。 将其添加到 Ping.hpp 中，如下所示：
```cpp
 #pragma once

 #include "engine/alice/alice.hpp"
 #include "messages/ping.capnp.h"

 class Ping : public isaac::alice::Codelet {
  public:
   ...

   ISAAC_PARAM(std::string, message, "Hello World!");
   ISAAC_PROTO_TX(PingProto, ping);
 };

ISAAC_ALICE_REGISTER_CODELET(Ping);

```
`ISAAC_PROTO_TX` 宏有两个参数。 第一个参数指定要发布的消息。 在这里，使用 Isaac 消息 API 附带的 `PingProto` 消息。 通过包含相应的头文件来访问 PingProto。 第二个参数指定我们要在其下发布消息的频道的名称。

### 修改 codelet .cpp 文件中的 tick() 函数
接下来，更改 `tick()` 函数以发布消息而不是打印到控制台。 Isaac SDK 目前支持 [cap'n'proto ](https://capnproto.org/)消息。 `Protos` 是一种独立于平台和语言的表示和序列化数据的方式。 创建消息是通过调用 `ISAAC_PROTO_TX` 宏创建的访问器上的 `initProto` 函数启动的。 此函数返回一个 `cap'n'proto` 构建器对象，可用于将数据直接写入原型。

`ProtoPing` 消息有一个名为 `message` 的字段，类型为字符串，因此在这种情况下，我们可以使用 `.setMessage()` 函数向原型写入一些文本。 填充原型后，我们可以通过调用发布函数来发送消息。 这会立即将消息发送到任何连接的接收器。

将 Ping.cpp 中的 .tick() 函数更改为以下内容：

```cpp
...
void Ping::tick() {
  // create and publish a ping message
  auto proto = tx_ping().initProto();
  proto.setMessage(get_message());
  tx_ping().publish();
}
...

```
### 将 MessageLedger 组件添加到节点
最后，升级节点（在 JSON 文件中）以支持消息传递。 Isaac SDK 中的节点默认是轻量级对象，需要最少的强制组件设置，您应用程序中的某些节点可能不需要发送或接收消息。

要在节点上启用消息传递，我们需要添加一个名为 MessageLedger 的组件。 该组件处理传入和传出消息并将它们中继到其他节点中的 MessageLedger 组件。

```json
{
  "name": "ping",
  "graph": {
    "nodes": [
      {
        "name": "ping",
        "components": [
          {
            "name": "message_ledger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "ping",
            "type": "Ping"
          }
         ]
      }
    ],
    "edges": []
},
"config": {
  ...
}

```
构建并运行应用程序。 似乎没有任何反应，因为现在没有任何内容连接到您的频道。 当您发布一条消息时，没有人会收到它并对其做出反应。 我们将在下一节中解决这个问题。

## 接收信息
您需要一个可以接收 `ping` 消息并以某种方式对其做出反应的节点。 为此，让我们创建一个 `Pong codelet`，它由 `Ping` 发送的消息触发。

### 为 `Pong` 创建一个 `codelet .hpp` 文件
使用以下内容创建一个名为 `Pong.hpp` 的新文件：
```cpp
#pragma once
#include "engine/alice/alice.hpp"
#include "messages/ping.capnp.h"

class Pong : public isaac::alice::Codelet {
 public:
  void start() override;
  void tick() override;

  // An incoming message channel on which we receive pings.
  ISAAC_PROTO_RX(PingProto, trigger);

  // Specifies how many times we print 'PONG' when we are triggered
  ISAAC_PARAM(int, count, 3);
};

ISAAC_ALICE_REGISTER_CODELET(Pong);

```
### 将 Pong 组件添加到 BUILD 文件中
需要将 Pong codelet 添加到 ping_components 模块才能进行编译。 将它们添加到 BUILD 文件中，如下所示（我们将在本节后面创建 Pong.cpp 文件）：

```cpp
isaac_cc_module(
  name = "ping_components",
  srcs = [
    "Ping.cpp",
    "Pong.cpp"
  ],
  hdrs = [
    "Ping.hpp",
    "Pong.hpp"
  ],
)

```

### 在 JSON 文件中创建一个 Pong 节点
在应用程序 JSON 文件中，创建第二个节点并将新的 Pong codelet 附加到它。

```json
"nodes": [
  {
    "name": "ping",
    "components": [
      {
        "name": "message_ledger",
        "type": "isaac::alice::MessageLedger"
      },
      {
        "name": "ping",
        "type": "Ping"
      }
    ]
  },
  {
    "name": "pong",
    "components": [
      {
        "name": "message_ledger",
        "type": "isaac::alice::MessageLedger"
      },
      {
        "name": "pong",
        "type": "Pong"
      }
    ]
  }
],

```

### 向 JSON 文件添加edge 
边缘将接收 RX 通道连接到发送 TX 通道。 一个传输通道可以将数据传输到多个接收器。 一个接收通道也可以接收来自多个发射器的数据； 但是，这是有警告的，不鼓励这样做。

与参数类似，通道由三个元素引用：节点名称、组件名称和通道名称。 可以通过将edge 添加到应用程序 JSON 文件中的边缘部分来创建边缘。 这里source是发送通道的全名，target是接收通道的全名。

使用边连接 Ping 和 Pong 节点：

```json
{
  "name": "ping",
  "modules": [
    "ping:ping_components"
  ],
  "graph": {
    "nodes": [
      {
        "name": "ping",
        "components": [
          {
            "name": "message_ledger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "ping",
            "type": "Ping"
          }
        ]
      },
      {
        "name": "pong",
        "components": [
          {
            "name": "message_ledger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "pong",
            "type": "Pong"
          }
        ]
      }
    ],
    "edges": [
      {
        "source": "ping/ping/ping",
        "target": "pong/pong/trigger"
      }
    ]
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
### 为 Pong 创建一个 codelet .cpp
剩下的最后一项任务是设置` Pong codelet` 在收到 ping 时执行某些操作。 创建一个名为 `Pong.cpp` 的新文件。 调用 `start`() 中的 `tickOnMessage`() 函数，以指示小代码在每次收到该频道上的新消息时进行标记。 在 `tick`() 中，我们添加了打印“PONG!”的功能。 与 `Pong` 头文件中的 `count` 参数定义的次数一样多：

```cpp
#include "Pong.hpp"

#include <cstdio>

void Pong::start() {
  tickOnMessage(rx_trigger());
}

void Pong::tick() {
  // Parse the message we received
  auto proto = rx_trigger().getProto();
  const std::string message = proto.getMessage();

  // Print the desired number of 'PONG!' to the console
  const int num_beeps = get_count();
  std::printf("%s:", message.c_str());
  for (int i = 0; i < num_beeps; i++) {
    std::printf(" PONG!");
  }
  if (num_beeps > 0) {
    std::printf("\n");
  }
}

```
通过使用 `tickOnMessage`() 而不是 `tickPeriodically`()，我们指示 `codelet` 仅在传入数据通道上收到新消息时才打勾，在本例中为触发器。 `tick` 函数现在仅在您收到新消息时执行。 这是由艾萨克机器人引擎保证的。

运行应用程序。 您应该看到每次 `Pong` 小代码从 `Ping` 小代码接收到 ping 消息时都会生成“`pong`”。 通过更改配置文件中的参数，您可以更改创建 ping 的时间间隔，更改与每个 `ping` 一起发送的消息，并在每次收到 `ping` 时或多或少地打印 `pong`。

## 通过网络发送消息
如果 `Ping` 和 `Pong` 组件运行在不同的设备上，我们需要网络连接。 `TcpPublisher` 和 `TcpSubscriber` 节点促进网络连接，如下所示：
```json
{
  "name": "ping",
  "modules": ["engine_tcp_udp"],
  "graph": {
    "nodes": [
      ...
      {
        "name": "pub",
        "components": [
          {
            "name": "message_ledger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "tcp_publisher",
            "type": "isaac::alice::TcpPublisher"
          }
        ]
      }
    ],
    "edges": [
      {
        "source": "ping/ping/ping",
        "target": "pub/tcp_publisher/tunnel"
      }
    ]
  },
  "config": {
    ...
    "pub": {
      "tcp_publisher": {
        "port": 5005
      }
    }
  }
}

```

`port` 参数指定接受连接的网络端口。 确保它在设备上可用。 `另一方面，TcpSubscriber` 可以在 JSON 文件中设置时传递消息，如下所示：

```json
{
  "name": "pong",
  ...
  "graph": {
    "nodes": [
      ...
      {
        "name": "sub",
        "components": [
          {
            "name": "message_ledger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "tcp_receiver",
            "type": "isaac::alice::TcpSubscriber"
          }
        ]
      }
    ],
    "edges": [
      {
        "source": "sub/tcp_receiver/tunnel",
        "target": "pong/pong/trigger"
      }
    ]
  },
  "config": {
    ...
    "sub": {
      "tcp_receiver": {
        "port": 5005,
        "reconnect_interval": 0.5,
        "host": "127.0.0.1"
      }
    }
  }
}

```

主机参数指定要侦听的 IP 地址。 确保主机和端口指定运行 Ping 的设备的开放端口和 IP 地址。 在不同的设备上运行这些应用程序以查看通过网络传送的消息。

这只是一个非常简单的应用程序的快速入门。 一个真实世界的应用程序由几十个节点组成，每个节点都有多个组件和一个或多个小码。 Codelet 接收多种类型的消息，调用专门的库来解决计算难题，并再次发布它们的结果以供其他节点使用。

















