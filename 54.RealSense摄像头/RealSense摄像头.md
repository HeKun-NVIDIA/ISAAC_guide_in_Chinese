# RealSense相机
英特尔RealSense 435 摄像头是一款立体摄像头，可借助红外发射器计算深度。 可以通过多种方式配置 RealSense 摄像头小代码，以与各种 Isaac SDK GEM 配合使用：

* 作为带有 GEM 的常规彩色相机，可处理单个相机图像——例如，用于物体检测。

* 作为深度相机为 Superpixels 等 GEM 提供颜色和深度图像。

* 作为[单色立体相机](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-visual-slam-elbrusvisualslam)，为立体视觉里程计等 GEM 提供灰度左右红外相机图像。

## RealsenseCamera Codelet
Isaac SDK 提供 RealsenseCamera codelet 以通过 librealsense SDK 访问来自 RealSense 摄像头的数据流。 在 Jetson 上，所有依赖项都由 Isaac SDK 自动处理，不需要手动安装。

小码可以通过各种参数进行配置。 最值得注意的是，您可以通过 rows 和 cols 参数更改所需的分辨率，并通过 rgb_framerate 和 depth_framerate 参数更改所需的帧率。 请注意，如 codelet 文档中所述，仅支持某些模式。 如果连接了多个摄像头，则需要通过 dev_index 或 serial_number 参数指定所需的设备。

## 示例应用程序
ISAAC 提供了一个基本的示例应用程序来测试您的实感摄像头。 您可以使用以下命令运行它：


```bash
bob@desktop:~/isaac/sdk$ bazel run //apps/samples/realsense_camera

```

应用程序运行后，您可以使用 Isaac Sight 查看相机捕获的图像。

## 故障排除
### 固件注意事项
RealSense 摄像头的固件版本可能已过时。 如上所述，您可以通过运行 RealSense 示例应用程序来检查相机的固件版本。 如果固件版本不是推荐的版本，应用程序会向控制台打印一条消息。

如果您需要更新固件，请按照官方网站上的说明进行操作：https://dev.intelrealsense.com/docs/firmware-update-tool。

推荐固件版本为 5.11.15。

## 通过 USB 3.0 电缆使用 USB 3.0 端口
RealSense 摄像头需要 USB3.0+ 连接。 USB 3.0 端口和电缆可以通过端口和电缆连接器内部的蓝色组件来识别。 Jetson Xavier USB-C 端口是 USB 3.0 端口。

使用 USB-C 端口测试 RealSense 摄像头揭示了稳定性问题，使用未供电的 USB 集线器可以解决这些问题。 众所周知，AmazonBasics USB 3.1 Type-C 到 4 端口铝制集线器连接器可以可靠地与 RealSense 摄像头配合使用。

众所周知，Accell USB C-to-A 和 C-to-C SuperSpeed+ USB 3.1 Gen 2 (10 Gbps) 电缆可以可靠地与 RealSense 摄像头配合使用。

## x86_64 Linux 主机设置
必须在 x86_64 Linux 主机上安装自定义 Linux 内核模块补丁和 UDEV 规则，RealSense 435 摄像头才能正常工作。

我们建议手动设置 Linux 内核模块，因为 Isaac SDK 使用 2.29 版本的 librealsense。 要执行手动设置，请遵循英特尔实感 SDK 设置指南。

## 设置电源模型
运行 sudo nvpmodel -m 0 启用所有内核，并增加缩放最小时钟频率。 此更改在重新启动后仍然存在。 运行 sudo nvpmodel -q 以显示当前设置。 在某些情况下，这会增加处理能力并可能导致帧丢失和图像模糊减少。



























