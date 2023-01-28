# Isaac C API

Isaac C API 允许您使用 Isaac SDK 功能，而无需将您的软件堆栈与 ISAAC SDK 完全集成。 通过使用 C API，您不必为代码库使用 Bazel，也不必编写 Isaac codelet。 它还使您能够使用 C++ 以外的编程语言与 Isaac 应用程序进行通信。

* 示例 1：您想将 Isaac 导航堆栈与 Carter 一起使用，但想从您自己的软件堆栈发送目标命令。 您可以通过 C API 向 Carter 应用程序发送 GoTo 命令来实现此目的。 这只需要对现有的 Carter 应用程序进行少量修改。

* 示例 2：您想使用 Isaac 3D 物体姿态估计来检测物体的姿态。 但是，您有自己的代码来获取图像，并且希望在 Isaac 应用程序之外使用估计的姿势。 您可以通过基于 3D 对象姿态估计子图创建一个小型 Isaac 应用程序来实现此目的：您将通过 C API 发送相机图像并通过 C API 接收估计的姿态。

* 示例 3：您想使用自定义模拟环境来评估 Isaac 应用程序。 您可以通过 C API 将传感器信息从模拟器发送到 Isaac 应用程序，并通过 C API 从应用程序接收命令。


## 程序流程和消息格式
使用 Isaac C API，您可以向 Isaac 应用程序发送消息并从中接收消息。 C API 负责启动和停止 Isaac 应用程序：您不能在没有 Isaac 应用程序的情况下使用 C API。 Isaac 应用程序是从 Isaac 使用的通用应用程序、图形和配置 JSON 文件加载的。 有关示例，请参阅[示例部分](https://docs.nvidia.com/isaac/packages/engine_c_api/doc/c_api.html#c-api-samples)。

通过 C API 发送的消息使用“JSON & buffers”格式。 在这种格式中，大部分高级消息数据存储在 JSON 对象中，而大数据块（例如图像和张量）存储在字节缓冲区中。 消息的 JSON 格式直接基于 Isaac 使用的相应 Cap'n Proto 消息。 您可以在本文档的消息 API 部分中查看消息列表。 [缓冲区布局](https://docs.nvidia.com/isaac/packages/engine_c_api/doc/c_api.html#c-api-buffer-layout)和[示例消息](https://docs.nvidia.com/isaac/packages/engine_c_api/doc/c_api.html#c-api-example-messages)部分提供了有关图像和张量的 JSON 格式和缓冲区布局的更多详细信息。

C API 的每个实例都与单个 Isaac 应用程序接口。 如果您想使用同一个 C 程序访问多个独立的 Isaac 功能，您应该将所有必需的节点添加到单个 Isaac 应用程序中。 这确保您只有一个调度程序在运行。 虽然可以从不同的进程启动多个 Isaac 应用程序，但您最终会得到多个独立的调度程序，这可能不是最佳的或者需要调整调度程序参数。


## ROS
如果您想与现有的 ROS 应用程序通信，则不必使用 C API。 Isaac 可以通过 Isaac ROS 网桥直接与 ROS 通信。

## 示例
Isaac SDK 包含两个示例应用程序，用于说明如何使用 C API：

* `//apps/tutorials/c_api/viewer.c`

* `//apps/tutorials/c_api/depth.c`

使用以下命令运行示例：

```bash
bazel run //apps/tutorials/c_api:c_api_viewer
bazel run //apps/tutorials/c_api:c_api_depth
```


`viewer.c` 示例使用 C 代码创建图像并将其发送到位于 `//apps/tutorials/c_api/viewer.app.json` 中的 Isaac 应用程序。 Isaac 应用程序使用带有 ImageViewer codelet 的单个节点来可视化通过 C 发送的图像。

`depth.c` 示例在 `//apps/tutorials/c_api/depth.app.json` 中运行 Isaac 应用程序，它有一个节点运行 CameraGenerator codelet，创建一个虚拟深度图像。 C 代码接收这些消息并在控制台上打印中心像素的深度。

为了方便起见，这些示例使用 Bazel 来编译 C 源代码文件。 接下来我们将向您展示一个完全独立的示例，您的自定义代码不需要 Bazel。

## 独立示例
使用以下命令构建 C API 并将其部署到本地计算机上的文件夹（从 `sdk/` 子目录）：

```bash
./../engine/engine/build/deploy.sh -p //packages/engine_c_api:isaac_engine_c_api-pkg -d x86_64 -h localhost --deploy_path ~
```

链接代码时，必须为 C API 和“sight”模块提供共享库的路径，它们位于：

* `<deploy_path>/engine/alice/c_api/libisaac_c_api.so`

* `<deploy_path>/packages/sight/libsight_module.so`

以下是使用 GCC 编译的命令行示例：

```bash
gcc c_api_example.c \
   -L./engine/alice/c_api -lisaac_c_api \
   -L./packages/sight -lsight_module \
   -o c_api_example

export LD_LIBRARY_PATH=./engine/alice/c_api:./packages/sight:$LD_LIBRARY_PATH

./c_api_example
```

此示例显示了一个简单的ping-pong操作，您可以在其中通过 C API 将消息发送到最小的 Isaac 应用程序，并通过 C API 接收这些相同的消息。

## 启动和停止应用程序
每个使用 C API 的 Isaac 应用程序都需要一个 JSON 应用程序文件； 例如，上面的示例具有相应的` viewer.app.json` 和 `depth.app.json` 文件。 使用 JSON 应用程序文件的应用程序图来定义您希望使用 C API 与之通信的节点。

要启动 Isaac 应用程序，请按如下方式加载 JSON 文件：

```cpp
isaac_handle_t app;
isaac_create_application("", "path/to/config.app.json", 0, 0, 0, 0, &app);
isaac_start_application(app);
```

要关闭应用程序，请使用以下命令：

```cpp
isaac_stop_application(app);
isaac_destroy_application(&app);
```

## 向 Isaac 应用程序发布消息
Isaac 在内部使用 [Cap'n Proto](https://capnproto.org/) 消息； 但是，由于这种消息格式并未得到普遍支持，因此 C API 将消息有效负载公开为 JSON。 如果接收消息的节点需要 Cap'n Proto 消息，则需要启用消息转换为 Cap'n Proto 格式。

按照以下步骤向 Isaac 发布消息：

1. 按如下方式创建新消息：

    ```cpp
    isaac_uuid_t uuid;
    isaac_create_message(app, &uuid);
    isaac_write_message_json(app, &uuid, &json);
    ```

2. 设置消息的原型 ID 并启用 JSON 到原型的转换，如下所示：

    ```cpp
    isaac_set_message_proto_id(app, &uuid, proto_id);
    isaac_set_message_auto_convert(app, &uuid, isaac_message_type_proto);
    ```
    `isaac_set_message_proto_id()` 调用需要消息的原型 ID。 要确定消息的原型 ID，请将消息类型添加到 `//apps/tutorials/c_api:typeid` 应用程序，然后通过 `bazel run //apps/tutorials/c_api:typeid` 运行该应用程序。 原型 ID 将显示在终端上。

3. 使用 `isaac_publish_message()` 调用将消息发布到 Isaac 应用程序节点。 该消息将作为 Cap'n Proto 消息转发到节点。

    ```cpp
    isaac_publish_message(app, "node_name", "component_name", "key", &uuid);
    ```

    有关消息发布的示例，请参见 //apps/tutorials/c_api/viewer.c。

## 从 Isaac 应用程序接收消息
要从 JSON 应用程序文件中定义的任何节点接收最新消息，请使用以下调用：

```cpp
isaac_receive_latest_new_message(app, "node_name", "component_name", "key", &uuid);
```

`isaac_receive_latest_new_message()` 调用返回“`成功`”代码 (`isaac_error_success`) 或“no message available”代码 (`isaac_error_no_message_available`)。

获取消息内容如下：
```cpp
isaac_const_json_t json = isaac_create_null_const_json();
isaac_get_message_json(app, &uuid, &json);
```
如果消息源使用 Cap'n Proto 发布消息，消息会自动转换为 JSON。

处理完消息后，立即使用 `isaac_release_message() `调用，它允许 Isaac 回收用于消息的任何资源：

```cpp
isaac_release_message(app, &uuid);
```

如果要保留任何消息数据，请在发布消息之前复制它。

有关接收消息的示例，请参见 `//apps/tutorials/c_api/depth.c`。

## 语言环境设置
Isaac 应用程序的区域设置自动设置为 `en_US.UTF-8`，以防止在将 JSON 消息与原型消息相互转换时出现十进制转换错误。 因此，为与 Isaac 节点通信而生成的 JSON 文件必须与 `en_US.UTF-8` 语言环境兼容。

## 示例消息
执行以下命令在 /tmp 文件夹中生成示例 JSON 文件：
```bash
bazel run apps/samples/proto_to_json
```
这些 JSON 文件（详见下文）对应于 Isaac 中使用的通用原型。

## ImageProto
运行上面的命令后，在生成的文件`/tmp/color_camera_proto.json`中可以找到需要的JSON格式。 为方便起见，将 JSON 粘贴在此处：

```json
{
  "channels": 3,
  "cols": 1920,
  "dataBufferIndex": 0,
  "elementType": "uint8",
  "rows": 1080
}
```
这条 `Cap’n Proto` 消息的名称是 [ImageProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#imageproto)，原型 ID 是 16240193334533147313。

有关二维缓冲区布局的说明，请参阅[图像缓冲区](https://docs.nvidia.com/isaac/packages/engine_c_api/doc/c_api.html#tensor-buffers)部分。

## RangeScanProto
* 文件名：`range_scan_proto.json`

* 原型描述：[RangeScanProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#rangescanproto)

* 原型编号：`11901202900662173387`

有关[缓冲区布局](https://docs.nvidia.com/isaac/packages/engine_c_api/doc/c_api.html#tensor-buffers)的说明，请参阅下面的张量缓冲区部分。 在这种情况下，有一个缓冲区用于范围，另一个缓冲区用于强度。 在示例 JSON 中，缓冲区大小为 16x8，因为垂直光束角 (theta) 列表有 16 个成员，而有八个水平射线切片（与 `phi` 角相关联）。


## StateProto (messages::DifferentialBaseDynamics)
* 文件名：/tmp/differential_base_state_proto.json

* 原型描述：[StateProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#stateproto)

* 原型编号：13177870757040999364

有[关缓冲区布局](https://docs.nvidia.com/isaac/packages/engine_c_api/doc/c_api.html#tensor-buffers)的说明，请参阅下面的张量缓冲区部分。 在这种情况下，缓冲区是一个 1x1x4 张量，其中包含以下值：

1. 线速度

2. 角速度

3. 直线加速度

4. 角加速度

## StateProto (messages::DifferentialBaseControl)
* 文件名：differential_base_control_proto.json

* 原型描述：[StateProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#stateproto)

* 原型编号：13177870757040999364

有关缓冲区布局的说明，请参阅下面的张量缓冲区部分。 在这种情况下，缓冲区是一个 1x1x2 张量，其中包含以下值：

1. 线速度

2. 角速度


## 缓冲区布局
带有缓冲区的消息原型必须存储包含数据的缓冲区的索引。 例如，ImageProto 中的“dataBufferIndex”是包含图像数据的缓冲区的索引。

## 图像缓冲器
下面是 720p RGB 图像缓冲区的图示。 每个（行、列）位置都包含有关像素的信息。 Isaac 图像缓冲区按行优先顺序排列。

![](https://docs.nvidia.com/isaac/_images/image_buffer.jpg)



## 张量缓冲器
在一维张量中，元素按顺序堆叠：

![](https://docs.nvidia.com/isaac/_images/tensor_buffer.jpg)

二维张量类似于上面描述的图像缓冲区：每个 RGB 值都可以表示为一个元组元素。 如果图像是灰度图像，则每个元素都将是一个数字。 二维张量缓冲区的第一个元素索引为 (0,0)，第二个元素索引为 (0,1)，依此类推。

类似地，三维张量具有以下顺序：

`(0, 0, 0), (0, 0, 1), (0, 0, 2), …. (0, 1, 0) …… (1, 0, 0) … (max_0, max_1, max_2),`

其中 max_i 是维度“i”中的最大索引。

3D 张量可以表示 RGB 图像，每个（行、列、通道）索引指向 0 到 255 之间的值。










