# ZED相机
Isaac SDK 支持 StereoLabs ZED 和 ZED Mini (ZED-M) 以及 ZED2 立体相机。

使用本节中的程序下载出厂校准文件或在相机上执行本地校准。

**重要的**

ZED 相机的本地校准是从立体视觉里程计等计算机视觉应用中获得准确结果的必要条件。

应将特殊的 UDEV 规则添加到 Linux 系统，以便用户应用程序与 ZED-M IMU 一起工作。 这是 ZED SDK 添加的规则：

```bash
bob@desktop:~/isaac/sdk$ cat /etc/udev/rules.d/99-slabs.rules
# HIDAPI/libusb
SUBSYSTEM=="usb", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f681", MODE="0666"
SUBSYSTEM=="usb", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f781", MODE="0666"
SUBSYSTEM=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="df11", MODE="0666"
# HIDAPI/hidraw
KERNEL=="hidraw*", ATTRS{busnum}=="1", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f681", MODE="0666"
KERNEL=="hidraw*", ATTRS{busnum}=="1", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f781", MODE="0666"
KERNEL=="hidraw*", ATTRS{busnum}=="1", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="df11", MODE="0666"
# blacklist for usb autosuspend
# http://kernel.org/doc/Documentation/usb/power-management.txt
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f780", TEST=="power/control", ATTR{power/control}="on"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f781", TEST=="power/control", ATTR{power/control}="on"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="0424", ATTRS{idProduct}=="2512", TEST=="power/control", ATTR{power/control}="on"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f780", TEST=="power/autosuspend", ATTR{power/autosuspend}="-1"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f781", TEST=="power/autosuspend", ATTR{power/autosuspend}="-1"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="0424", ATTRS{idProduct}=="2512", TEST=="power/autosuspend", ATTR{power/autosuspend}="-1"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f780", TEST=="power/autosuspend_delay_ms", ATTR{power/autosuspend_delay_ms}="-1"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="2b03", ATTRS{idProduct}=="f781", TEST=="power/autosuspend_delay_ms", ATTR{power/autosuspend_delay_ms}="-1"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="0424", ATTRS{idProduct}=="2512", TEST=="power/autosuspend_delay_ms", ATTR{power/autosuspend_delay_ms}="-1"
```


**注意**

这些说明中的文件路径使用 ZED SDK 的默认安装位置：`/usr/local/zed`。 如果您在其他位置安装了 ZED SDK，则需要相应地更改 ZED SDK 文件路径。

## Codelets
* isaac::ZedCamera：发布由 ZED 相机拍摄的彩色或单色立体对图像。 它不提供对深度感应、位置跟踪、空间映射或 ZED SDK 的其他高级功能的访问。

* isaac::zed::ZedImuReader：发布从 ZED-M IMU 捕获的 IMU 读数（线性加速度和角速度）。 此小代码应添加到与 isaac::ZedCamera 小代码相同的节点。 线加速度和角速度向量在COORDINATE_SYSTEM_IMAGE中给出，这是[OpenCV](http://docs.opencv.org/2.4/modules/calib3d/doc/camera_calibration_and_3d_reconstruction.html)中使用的标准右手坐标系：X坐标指向右侧，Y指向底部，Z指向左侧远离光轴 相机：


![](https://docs.nvidia.com/isaac/_images/zed_coordinate_systems.jpg)



## 支持的固件
* ZED 相机支持并测试了相机固件版本 1142。

* ZED-M 相机支持和测试相机固件版本 1523 和 IMU 固件版本 515。

使用 ZED Explorer SDK 工具根据需要将相机固件升级或降级到支持的版本。 ZED 资源管理器可执行文件位于 /usr/local/zed/tools/。

## 下载出厂校准文件
1. ZED 相机需要校准文件才能工作。 校准文件的名称是 `SN<serial_number>.conf`，其中“`<serial_number>`”是相机的序列号。

2. 每个 ZED 相机都由唯一的序列号标识，并在出厂时进行了校准。 在 ZED 相机盒上找到序列号，或运行 bazel run apps/samples/zed_camera 将相机序列号打印到标准输出。

3. 使用以下命令下载 ZED 相机的出厂校准文件：
```bash
bob@desktop:~/isaac/sdk$ ./engine/engine/build/scripts/download_zed_calibration.sh -s <zed_camera_serial_number>
```

4. 将下载的 SN<serial_number>.conf 文件复制到` /usr/local/zed/settings` 目录。

5. 运行 bazel run apps/samples/zed_camera 并在 Chrome 浏览器中打开 Isaac Sight，网址为 http://localhost:3000/。 您应该可以正确看到相机图像。

6. 如果下载的标定文件复制成功，但zed_camera app输出如下信息，则需要进行本地标定。

```bash
2020-10-16 20:11:34.662 ERROR external/com_nvidia_isaac_engine/engine/alice/components/Codelet.cpp@229: Component 'zed/zed_camera' of type 'isaac::ZedCamera' reported FAILURE:

Calibration file was not found for your zed camera.
```


## 通过本地校准提高相机精度
1. 将 ZED 相机连接到主机 PC 并从以下网站安装 ZED SDK：https://www.stereolabs.com/developers/release/。

2. 启动 ZED 校准实用程序：
```bash
bob@desktop:~/isaac/sdk$ /usr/local/zed/tools/ZED\ Calibration
```
3. 按照屏幕上的说明执行相机校准。 校准实用程序将新的 `SN<serial_number>.conf `校准文件写入默认设置目录 (/usr/local/zed/settings)，覆盖现有文件。

4. 将校准文件复制到 NVIDIA Jetson 板上的 /usr/local/zed/settings 目录。

## 为相机校准文件指定自定义位置
如果需要，请使用 ZedCamera codelet 中的 settings_folder_path 参数为包含相机校准文件的目录指定自定义路径。

































