# Isaac Jetson Nano 入门

本节介绍如何在 Jetson Nano 设备上运行 Isaac SDK 示例应用程序。 有关如何开始使用 Nano 的一般说明，请参阅 [Jetson Nano 开发工具包入门](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit)。

## 获取 IP 地址
获取Jetson Nano的IP地址：

1. 连接键盘、鼠标和显示器，并启动设备，如 [Jetson Nano 开发工具包入门的设置和首次启动部分](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit#setup)所示。

2. 在终端提示符下，输入以下命令：

```bash
bob@jetson:~/$ ip addr show
```

## 在 Jetson Nano 上运行示例应用程序
本节介绍在 Jetson Nano 上运行示例应用程序的步骤。 第一个示例不需要任何外围设备。 第二个示例是一个更有用的应用程序，需要连接相机。 第三个示例演示了如何部署 TensorFlow 模型并在设备上运行推理。

可以使用此处描述的方法部署和运行其他应用程序。

## Ping
1. 按照[应用程序控制台选项](https://docs.nvidia.com/isaac/doc/getting_started.html#deployment-device)中的说明将 `//apps/tutorials/ping:ping-pkg` 部署到机器人。

2. 切换到 Jetson Nano 上的目录并使用以下命令运行应用程序：

```bash
bob@jetson:~/$ cd deploy/<bob>/ping-pkg/
bob@jetson:~/deploy/<bob>/ping-pkg$ ./apps/tutorials/ping/ping
```

`<bob>` 是您在主机系统上的用户名。 你应该看到“Hello World!” 每 1.5 秒打印一次。

## OpenCV 边缘检测
1. 对于此示例，将相机连接到 Jetson Nano 上的 USB 端口之一。

2. 按照应用程序控制台选项中的说明，将 `//apps/tutorials/opencv_edge_detection:opencv_edge_detection-pkg` 部署到机器人。

3. 切换到 Jetson Nano 上的目录并使用以下命令运行应用程序：

```bash
bob@jetson:~/$ cd deploy/<bob>/opencv_edge_detection-pkg/
bob@jetson:~/deploy/<bob>/opencv_edge_detection-pkg/$ ./apps/tutorials/opencv_edge_detection/opencv_edge_detection
```
4. 要查看结果，请在浏览器中加载 `http://<nano_ip>:3000/`。 确保在加载网页时应用程序正在运行。

## 使用 TensorRT 进行 TensorFlow 模型推理
1. 按照应用程序控制台选项中的说明，将 `//apps/samples/ball_segmentation:inference_tensorrt-pkg` 部署到机器人。

2. 切换到 Jetson Nano 上的目录并使用以下命令运行应用程序：

```bash
bob@jetson:~/$ cd deploy/<bob>/inference_tensorrt-pkg/
bob@jetson:~/deploy/<bob>/inference_tensorrt-pkg/$ ./apps/samples/ball_segmentation/inference_tensorrt
```

3. 要查看结果，请在浏览器中加载 `http://<nano_ip>:3000/`。 确保在加载网页时应用程序正在运行。

## 经常遇到的问题
### 如何刷入 Jetson Nano
Nano 可以通过两种方式刷入：

* 通过 SD 卡：请参阅 [Jetson Nano 开发工具包入门中的程序](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit)。

* 通过 SDK Manager：参见 [SDK Manager 用户指南](https://docs.nvidia.com/sdk-manager)中的步骤

使用上面的文档来确定适合您的用例的最佳方法。

## 如何连接 Wi-Fi/蓝牙卡
Intel 8265 卡用于 Wi-Fi 和蓝牙连接。 以下步骤描述了如何为 Jetson Nano 安装 Wi-Fi/蓝牙卡。

1. 要访问载板上的 M.2 插槽，请卸下侧面的两个螺丝并用双手打开 SODIMM 闩锁。

![](https://docs.nvidia.com/isaac/_images/nano-wifi-1.jpg)

2. 当 Jetson Nano 模块弹出时，将其轻轻滑出。

![](https://docs.nvidia.com/isaac/_images/nano-wifi-2.jpg)
3. 在将卡插入 M.2 插座之前，取出英特尔无线网卡并将天线连接到其 U.FL 插座。 将天线连接到插座上需要耐心和习惯。 用指甲轻轻用力。 一旦你处于正确的位置，你不需要太大的力量来夹紧它。

![](https://docs.nvidia.com/isaac/_images/nano-wifi-3.jpg)
4. 将 Intel 8265 卡滑入插槽。

![](https://docs.nvidia.com/isaac/_images/nano-wifi-3.jpg)
5. 用螺丝固定 Intel 8265 卡，更换 Jetson Nano 模块。 确保在每种情况下都使用正确的螺钉。

![](https://docs.nvidia.com/isaac/_images/nano-wifi-4.jpg)

































