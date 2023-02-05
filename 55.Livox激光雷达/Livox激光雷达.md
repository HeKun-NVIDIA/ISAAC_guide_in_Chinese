# Livox 激光雷达

Livox 激光雷达
Isaac SDK 支持使用 Livox LIDAR，包括兼容的驱动程序和示例应用程序。

## 支持的硬件和固件
Isaac SDK 中包含的 Livox LIDAR 驱动程序支持 Livox Mid-40 并符合固件版本 03.04.0000。

## 在桌面上设置和运行示例应用程序
Isaac SDK 包含一个示例应用程序，演示了 Livox LIDAR 驱动程序和相关组件的用法。

要设置示例，请执行以下步骤：

1. 打开 Livox LIDAR 并将其连接到主机 PC。

2. 从以下网站下载并安装 Livox Viewer 工具：https://www.livoxtech.com/。

3. 按照 Livox 快速入门说明将 LIDAR 配置为使用您选择的静态 IP 地址，例如 192.168.1.13。

4. 按照 Livox 快速入门说明将 LIDAR 固件升级到版本 03.04.0000。

5. 在示例应用程序配置文件 packages/livox/apps/livox_lidar_visualization/livox_lidar_visualization.app.json 中，更新 LIDAR 的 IP 地址。 在这个例子中：
```json
"config": {
  "livox_lidar_mid-40": {
    "driver": {
      "device_ip": "192.168.1.13",
```

6. 使用以下命令启动示例应用程序：
```bash
bob@desktop:~/isaac/sdk$ bazel run //packages/livox/apps/livox_lidar_visualization
```
应用程序开始正常运行并稳定下来。

## 在机器人上设置和运行示例应用程序
要在机器人上部署和运行示例应用程序，请执行以下步骤：

将 `//packages/livox/apps/livox_livox_visualization:livox_livox_visualization-pkg` 部署到机器人，如应用程序控制台选项中所述。

使用以下命令在机器人上运行示例应用程序：
```bash
bob@jetson:~/$ ./packages/livox/apps/livox_livox_visualization/livox_livox_visualization
```

## 查看正在运行的应用程序
1. 通过在桌面或机器人上运行 livox_LIDAR 应用程序，通过加载 localhost:3000 在 Web 浏览器中启动 Sight 应用程序。 随时间变化的点计数等指标以及显示的其他信息将类似于以下内容：

![](https://docs.nvidia.com/isaac/_images/livox_sight.png)


2. 启用点云查看器以显示传入数据：

![](https://docs.nvidia.com/isaac/_images/livox_lidar.png)


## 将来
Isaac SDK 中的 Livox LIDAR 是另一种可用于机器人应用程序的工具。 您的应用程序可能会集成 LIDAR 以用于导航、感知、深度或其他用途。

在下面的示例中，反射率信息在 10 秒的曝光时间段内传输到颜色通道中，用于累积云中总共一百万个点。

![](https://docs.nvidia.com/isaac/_images/livox_reflectivity.png)

关于Livox LIDAR的更多详细信息，请访问以下网站：https://www.livoxtech.com/










































