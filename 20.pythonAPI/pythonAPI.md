# Isaac Python接口(Python API)

虽然 Isaac SDK 的大部分部分都是用 C++ 编码的，但您可以选择使用 Python 构建您的应用程序。 本文档介绍了 Isaac SDK 的 Python API。 Python API 允许您执行以下操作：

* 在 Python 中创建、管理和运行 Isaac 应用程序。

* 在 Python 中访问记录的 Isaac 日志数据。

* 使用 Python 中的行为树实现复杂的机器人行为。

## 在 Python 中创建应用程序
Isaac SDK 包括几个使用 Python API 编码的示例应用程序，例如 `packages/flatsim/apps:flatsim`。 下面的示例以一个 Python 应用程序框架开始。

首先，需要一个 Bazel 目标以及 Python 代码本身：
```json
load("@com_nvidia_isaac_sdk//bzl:py.bzl", "isaac_py_app")

isaac_py_app(
    name = "foo_app",
    srcs = [ "foo_app.py" ],
    data = [ "foo_subgraph" ],
    modules=[ 'viewers' ],
    deps = [ "//packages/pyalice" ],
)
```

除了 src 和数据等通用定义外，模块还引入了 Isaac SDK 附带的指定模块。

要在 PC 上运行该应用程序，请使用以下命令：

```bash
bazel run //apps/foo_app:foo_app
```

要在 Jetson 设备上运行它，首先使用 `deploy.sh` 脚本（来自 `sdk/ `子目录）部署它：

```bash
./../engine/engine/build/deploy.sh -h <jetson_ip> -p //apps/foo_app:foo_app-pkg -d jetpack45
```

其中 jetson_ip 是 Jetson 设备的 IP。

然后，在 Jetson 设备上运行这个应用程序：

```bash
./run ./apps/foo_app/foo_app.py
```

要使用 Python API 创建 Isaac 应用程序实例，请使用“`engine.pyalice.Application`”导入并创建实例。 与 C++ 一样，init 函数可以将应用程序 JSON 文件的路径作为参数：
```python
from engine.pyalice import Application

app = Application(name="foo_app")
```
使用应用程序实例，您可以像使用 `*.app.json` 文件一样加载 Isaac 子图：

```json
app.load("apps/foo_app/foo.subgraph.json", prefix="foo")
```
除了加载预先编写的计算图外，Python API 还可以使事情变得更加灵活。 例如，Isaac SDK 中的许多小码都是由位于 packages 文件夹中的模块提供的，这些模块必须通过 `*.app.json` 和 `*.subgraph.json` 文件加载，就像在 C++ 应用程序中一样：

```json
{
  "name": "foo_app",
  "modules": [
    "message_generators",
    "viewers"
  ]
}
```
此处 `message_generators` 模块提供虚拟小码，用于发布预配置消息以用于测试目的。 查看器模块提供在 Sight 中可视化消息的小码。

使用 Python API，除了在 JSON 文件中指定模块外，您还可以在创建应用程序和认为有必要时加载模块：
```python
app = Application(name="foo_app", modules=["message_generators"])
app.load_module('viewers')
```

**注意**

确保在从模块提供的小码创建实例或加载任何使用小码的子图之前加载模块。

与其 C++ 对应物一样，应用程序使用由组件组成的节点来管理计算图。 现在，让我们创建一个节点，并从我们刚刚在上面加载的查看器模块提供的 ImageViewer codelet 中附加一个组件：
```python
node = app.add(name='viewer')
component = node.add(name='ImageViewer',
                     ctype=app.registry.isaac.viewers.ImageViewer)
```
在这里，app.add() 函数返回一个节点实例，而 node.add() 函数返回一个组件实例。 这些实例也可以按如下方式检索：
```python
node = app.nodes['viewer']
component = node['ImageViewer']
```
现在将 target_fps 的配置参数设置为 15fps。 有关其配置参数的详细信息，请参阅 `isaac.viewers.ImageViewer` API 条目。

```python
component.config.target_fps = 15.0
```

同样，您可以从发布消息的 `CameraGenerator codelet` 创建一个组件，并按如下方式配置它。 注意`CameraGenerator`是`message_generators`模块提供的，需要提前加载。

```python
image_node = app.add(name='camera')
camera_generator = node.add(name='CameraGenerator',
                            ctype=app.registry.isaac.message_generators.CameraGenerator)
camera_generator.config.rows = 480
camera_generator.config.cols = 640
camera_generator.config.tick_period = '15Hz'
```

现在我们有一个发布消息的生成器组件和一个可视化消息的查看器组件。 我们可以连接这些组件，以便将生成的消息发送到查看器组件：

```python
app.connect(camera_generator, "color_left", component, "color")
app.connect('camera/CameraGenerator', 'color_left', 'viewer/ImageViewer', 'image')
```

在这里，可以使用上述实例或它们的名称来指定组件。

通过上面的代码，我们现在有了一个完整的应用程序图。 您可以使用 `run() `函数运行它。 调用不带参数的 `run()` 允许它无限期地运行。 您还可以指定它运行一段时间（以秒为单位）或在特定节点不再运行时停止：

```python
app.run()

app.run(10.0)

app.run('foo_node')
```

在所有情况下，按 Ctrl-C 将停止应用程序。

## 访问Cask Logs
`Cask` 是 Isaac SDK 中使用的记录格式。 记录和回放日志请参考：记录和回放。 可以在 `apps/samples/camera/record_dummy.py` 中找到用于记录日志的示例应用程序。

假设你在/path/to/log/文件夹下有记录的日志，你可以用Python加载日志，如下：

```python
from isaac import Cask, Message
cask = Cask('/path/to/log/')

# List all channels recorded
series = cask.channels['foo_channel']:    # Looks for channel named 'foo_channel'
for msg in series:                        # Goes through every messages one by one in recorded order
   print(msg.proto)
   print(msg.uuid)
   print(msg.acqtime)
   print(msg.pubtime)
```

## 行为树
Isaac SDK 具有一个称为行为树的特殊模块，它提供了不同的小码，可用于管理复杂应用程序行为的其他小码。 `例如，TimerBehavior` 可以启动一个特定的小代码，并在关闭它之前让它运行指定的持续时间。 另一方面，SwitchBehavior 可用于在预配置模式之间切换行为。

在创建和操作 Behavior codelet 之前，确保模块已加载：

```python
app.load_module('behavior_tree')
```

行为小码也可以由其他行为小码管理——您可以通过堆叠行为来创建相当复杂的功能。

您可以通过使用 Python API 创建和配置这些行为代码来实现更大的灵活性。

使用Python开发Codelet的示例请参考：[Developing Codelets in Python](https://docs.nvidia.com/isaac/doc/tutorials/ping_py.html#pycodelet)。





















