# 创建Isaac应用
本教程将指导您完成使用 Isaac SDK 创建机器人应用程序的过程，以视频输入的 [OpenCV 边缘检测](https://docs.opencv.org/trunk/da/d22/tutorial_py_canny.html)处理为例。

本教程将指导您完成以下步骤：

1. 使用 Isaac SDK 显示 USB 摄像头源。

2. 使用 OpenCV 边缘检测处理相机提要并显示处理后的提要。

3. 使用 OpenCV 边缘检测处理来自 Isaac Sim Unity3D 的模拟输入。

## 预安装
本教程需要以下硬件/软件：

1. 安装了 Issac SDK 和所有先决条件以及 Isaac Sim Unity3D 的 Linux x86 机器。 有关详细信息，请参阅[设置](https://docs.nvidia.com/isaac/doc/setup.html#setup-isaac)页面。

2. 与 Video4Linux2 (V4L2) 驱动程序兼容的 USB 摄像头


## 显示相机源
第一步，我们将使用 Isaac V4L2 组件在 Linux x86 机器上捕获摄像头源并将其显示在 Websight 中。

**注意**

此应用程序的完整 JSON 和构建文件可在 `//apps/samples/v4l2_camera` 获得。


![](https://docs.nvidia.com/isaac/_images/camera_feed_example.jpg)

## 创建应用程序文件
在 Isaac `//apps` 目录的新文件夹中创建一个名为 `v4l2_camera.app.json` 的 JSON 应用程序文件。 添加如下所示的内容：

```json
{
   "name": "v4l2_camera",
   "modules": [
     "sensors:v4l2_camera",
     "sight",
     "viewers"
   ],
   "graph": {
       "nodes": [
         {
           "name": "camera",
           "components": [
             {
               "name": "V4L2Camera",
               "type": "isaac::V4L2Camera"
             }
           ]
         },
         {
           "name": "viewer",
           "components": [
             {
               "name": "ImageViewer",
               "type": "isaac::viewers::ImageViewer"
             }
           ]
         }
       ]
}
```

应用程序`“graph”`定义了两个节点：一个`“camera”`节点，它使用 `V4L2Camera` 组件捕获相机提要，以及一个`“viewer”`节点，它使用 `ImageViewer `组件在 Websight 中显示提要。 `“modules”`部分包括应用程序中的这些组件。

## 启用节点间通信
要启用`“camera”`和`“viewer”`节点之间的通信，请将 MessageLedger 组件添加到它们：


```json
{
   "name": "v4l2_camera",
   "modules": [
     "sensors:v4l2_camera",
     "sight",
     "viewers"
   ],
   "graph": {
       "nodes": [
         {
           "name": "camera",
           "components": [
             {
               "name": "MessageLedger",
               "type": "isaac::alice::MessageLedger"
             },
             {
               "name": "V4L2Camera",
               "type": "isaac::V4L2Camera"
             }
           ]
         },
         {
           "name": "viewer",
           "components": [
             {
               "name": "MessageLedger",
               "type": "isaac::alice::MessageLedger"
             },
             {
               "name": "ImageViewer",
               "type": "isaac::viewers::ImageViewer"
             }
           ]
         }
       ],
}
```

使用`“edges”`小节将源`“camera”`节点连接到目标`“viewer”`节点。

```json
"graph": {
    "nodes": [
      ...
    ],
    "edges": [
      {
        "source": "camera/V4L2Camera/frame",
        "target": "viewer/ImageViewer/image"
      }
    ]
```

## 配置组件
添加`“config”`部分以设置 USB 摄像头和 Websight 服务器的参数。 在`“websight”`部分中，`“windows”`子部分创建了一个窗口，用于在 Websight 中呈现摄像头源。

```json
{
  "name": "v4l2_camera",
  "modules": [
    "sensors:v4l2_camera",
    "sight",
    "viewers"
  ],
  "graph": {
  ...
  },
  "config": {
    "camera": {
      "V4L2Camera": {
        "device_id": 0,
        "rows": 480,
        "cols": 640,
        "rate_hz": 30
      }
    },
    "websight": {
      "WebsightServer": {
        "port": 3000,
        "ui_config": {
          "windows": {
            "Camera": {
              "renderer": "2d",
              "channels": [
                { "name": "v4l2_camera/viewer/ImageViewer/image" }
              ]
            }
          }
        }
      }
    }
  },

```

## 创建 Bazel 构建文件
在与 JSON 应用程序文件相同的目录中创建一个名为 BUILD 的文件。 BUILD 文件指定应用程序的“名称”和应用程序使用的 Isaac 模块。 应用程序名称应与 `<name>.app.json` 应用程序文件名的 `<name>` 相匹配。
```json
load("//bzl:module.bzl", "isaac_app")

isaac_app(
    name = "v4l2_camera",
    modules = [
        "sensors:v4l2_camera",
        "sight",
        "viewers",
    ],
)

```


## 运行应用程序
打开控制台窗口，导航到 Isaac 目录，然后按照入门页面中的描述运行应用程序：
```json
bob@desktop:~/isaac/sdk$ bazel run //apps/samples/v4l2_camera
```

## 查看相机源
通过导航到 `http://localhost:3000/`，在 Web 浏览器中打开 `Websight`。 您应该在 `Websight` 的窗口中看到视频流。 如果不这样做，请确保选中左侧的 `Channels > v4l2_camera` 框。

![](https://docs.nvidia.com/isaac/_images/sight_v4l2.jpg)

要停止应用程序，请在控制台中按 Ctrl+C。

## 处理相机源
接下来，我们将构建在上一节中创建的应用程序图，以对相机源执行边缘检测。


![](https://docs.nvidia.com/isaac/_images/edge_detector_example.jpg)


**注意**

此应用程序的完整 JSON 和构建文件可在 `//apps/tutorials/opencv_edge_detection` 获得。 JSON 文件名为 `opencv_edge_detection.app.json`。

## 添加边缘检测节点
将第三个节点“edge_detector”添加到应用程序图中。 该节点将使用 Isaac EdgeDetector 组件对 V4L2 摄像头源执行边缘检测。

```json
"graph": {
    "nodes": [
      {
        "name": "camera",
        "components": [
          {
            "name": "MessageLedger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "V4L2Camera",
            "type": "isaac::V4L2Camera"
          }
        ]
      },
      {
        "name": "edge_detector",
        "components": [
          {
            "name": "MessageLedger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "EdgeDetector",
            "type": "isaac::opencv::EdgeDetector"
          }
        ]
      },
      {
        "name": "viewer",
        "components": [
          {
            "name": "MessageLedger",
            "type": "isaac::alice::MessageLedger"
          },
          {
            "name": "ImageViewer",
            "type": "isaac::viewers::ImageViewer"
          }
        ]
      }
    ],

```

您还需要在`“modules”`部分添加 EdgeDetector 组件：

```json
"modules": [
    "//apps/tutorials/opencv_edge_detection:edge_detector",
    "sensors:v4l2_camera",
    "sight",
    "viewers"
],

```
## 修改应用程序边缘
修改`“edges”`部分，使`“camera”`节点将提要传递给`“edge_detector”`节点，然后由`“edge_detector”`节点将处理后的提要发送给`“viewer”`节点。

```json
"graph": {
    "nodes": [
      ...
    ],
    "edges": [
      {
        "source": "camera/V4L2Camera/frame",
        "target": "edge_detector/EdgeDetector/input_image"
      },
      {
        "source": "edge_detector/EdgeDetector/output_image",
        "target": "viewer/ImageViewer/image"
      }
    ]

```


## 修改构建文件
在应用程序构建文件中包含 EdgeDetector codelet：

```json
load("//bzl:module.bzl", "isaac_app", "isaac_cc_module")

isaac_app(
    name = "opencv_edge_detection",
    modules = [
        "//apps/tutorials/opencv_edge_detection:edge_detector",
        "sensors:v4l2_camera",
        "sight",
        "viewers",
    ],
)

```


## 运行应用程序
运行应用程序：

```bash
bob@desktop:~/isaac/sdk$ bazel run //apps/tutorials/opencv_edge_detection
```

打开Websight。 您应该看到带有边缘检测的视频流。 如果不是这样，请确保选中左侧的 `Channels > opencv_edge_detection `框。

![](https://docs.nvidia.com/isaac/_images/sight_edge_detection.jpg)

## 处理来自模拟的输入
在这最后一部分中，我们将把我们的边缘检测应用程序与模拟机器人集成在一起。

![](https://docs.nvidia.com/isaac/_images/simple_opencv_unity3d.jpg)

**注意**

此应用程序的完整 JSON 和构建文件可在 //apps/tutorials/opencv_edge_detection 获得。 JSON 文件名为 opencv_unity3d.app.json。

## 更换相机节点
从应用程序图中删除“camera”节点。 将其替换为“simulation”节点，如下所示。 此节点中的子图允许应用程序访问来自 Unity3D 模拟的各种数据流，就好像它们是真实的传感器一样。 此子图还支持向模拟中的 Carter 机器人发送命令，但构建 App 教程中未涵盖该主题。

```json
"graph": {
  "nodes": [
    {
      "name": "simulation",
      "subgraph": "packages/navsim/apps/navsim_navigation.subgraph.json"
    },
    {
      "name": "edge_detector",
      "components": [
        {
          "name": "MessageLedger",
          "type": "isaac::alice::MessageLedger"
        },
        {
          "name": "EdgeDetector",
          "type": "isaac::opencv::EdgeDetector"
        }
      ]
    },
   ...

```
## 添加第二个 Camera_Viewer 节点
为了将边缘检测输出与标准相机进行比较，我们希望在 Websight 中看到这两个流。 为此，添加第二个使用 ImageViewer 组件的节点。 在下面的示例中，两个节点分别命名为“edge_camera_viewer”和“camera_viewer”。


```json
"graph": {
  "nodes": [
    {
      "name": "simulation",
      "subgraph": "packages/navsim/apps/navsim_navigation.subgraph.json"
    },
    {
      "name": "edge_detector",
      "components": [
        {
          "name": "MessageLedger",
          "type": "isaac::alice::MessageLedger"
        },
        {
          "name": "EdgeDetector",
          "type": "isaac::opencv::EdgeDetector"
        }
      ]
    },
    {
      "name": "edge_camera_viewer",
      "components": [
        {
          "name": "MessageLedger",
          "type": "isaac::alice::MessageLedger"
        },
        {
          "name": "ImageViewer",
          "type": "isaac::viewers::ImageViewer"
        }
      ]
    },
    {
      "name": "camera_viewer",
      "components": [
        {
          "name": "MessageLedger",
          "type": "isaac::alice::MessageLedger"
        },
        {
          "name": "ImageViewer",
          "type": "isaac::viewers::ImageViewer"
        }
      ]
    }
  ...

```
## 修改应用程序edges
在“edges”部分，修改第一“edges”，以便“edge_detector”节点接收来自模拟的输入。 为每个“查看器”节点包含一条边：“edge_camera_viewer”节点连接到“edge_detector”节点，而“camera_viewer”直接从模拟接收流。
```json
"edges": [
      {
        "source": "simulation.interface/output/color",
        "target": "edge_detector/EdgeDetector/input_image"
      },
      {
        "source": "edge_detector/EdgeDetector/output_image",
        "target": "edge_camera_viewer/ImageViewer/image"
      },
      {
        "source": "simulation.interface/output/color",
        "target": "camera_viewer/ImageViewer/image"
      }
    ]

```
## 添加 WebSight 配置
在“config”部分，添加另一个窗口以显示另一个摄像头源。 请注意，您需要更改通道“name”以匹配修改后的节点名称。

```json
"config": {
   "websight": {
         "WebsightServer": {
           "port": 3000,
           "ui_config": {
             "windows": {
               "Edge Detection": {
                 "renderer": "2d",
                 "dims": {
                   "width": 640,
                   "height": 480
                 },
                 "channels": [
                   {
                     "name": "opencv_unity3d/edge_camera_viewer/ImageViewer/image"
                   }
                 ]
               },
               "Color Camera": {
                 "renderer": "2d",
                 "dims": {
                   "width": 640,
                   "height": 480
                 },
                 "channels": [
                   {
                     "name": "opencv_unity3d/camera_viewer/ImageViewer/image"
                   }
                 ]
               }
             }
           }
         }
       }

```


我们还建议配置每个 ImageViewer 组件以减少资源消耗并使视频大小易于管理：

```json
"config": {
    "edge_camera_viewer": {
      "ImageViewer": {
        "target_fps": 20,
        "reduce_scale": 4
      }
    },
    "camera_viewer": {
      "ImageViewer": {
        "target_fps": 20,
        "reduce_scale": 4
      }
    },

```
## 修改构建文件
要使用与 Unity3D 交互的“`navsim_navigation`”子图，您需要将其添加到 BUILD 文件中：
```json
data = [
    "//packages/navsim/apps:navsim_navigation_subgraph",
],
modules = [
    "//apps/tutorials/opencv_edge_detection:edge_detector",
    "viewers"
],

```
## 启动Unity3D
在运行您的应用程序之前，您需要在 Unity3D 中启动“small_warehouse”示例场景：

```bash
bob@desktop:~/isaac_sim_unity3d$ cd builds
bob@desktop:~/isaac_sim_unity3d/builds$ ./sample.x86_64 --scene small_warehouse
```
Unity 窗口应打开，显示带有 Carter 机器人的仓库的俯视图。

## 运行应用程序
运行 Isaac 应用程序，它将从模拟中接收传感器数据：
```bash
bob@desktop:~/isaac/sdk$ bazel run //apps/tutorials/opencv_edge_detection:opencv_unity3d
```
在 Websight 中，您应该会看到模拟 Carter 的标准摄像头视图，以及经过边缘检测处理的视图。

![](https://docs.nvidia.com/isaac/_images/sight_unity3d_edge_detection.jpg)


## 接下来
现在您已经构建了几个应用程序，您可以准备探索 Isaac SDK 的其他方面：

* 创建小码：本教程使用预先存在的小码来构建应用程序。 按照[在 C++ 中开发小码教程](https://docs.nvidia.com/isaac/doc/tutorials/ping.html#cplusplus-ping)学习如何构建您自己的小码。

* 在真实机器人上运行应用程序：模拟之后，下一步是让您的应用程序在真实机器人上运行。 Isaac SDK 提供 [NVIDIA Kaya](https://docs.nvidia.com/isaac/doc/tutorials/assemble_kaya.html#kaya-hardware) 和 [NVIDIA Carter](https://docs.nvidia.com/isaac/doc/tutorials/carter_hardware.html#carter-hardware) 参考设计，以构建具有完整导航和感知能力的机器人。 请参阅 [Jetson Nano 入门页面](https://docs.nvidia.com/isaac/doc/tutorials/nano.html#get-started-nano)，了解如何将 Isaac 应用程序发布到 Jetson 设备。















