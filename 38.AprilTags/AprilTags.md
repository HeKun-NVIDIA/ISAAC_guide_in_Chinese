# AprilTags
AprilTags 是一种流行的基准标记形式。 它在机器人技术中有广泛的应用，包括对象跟踪、视觉定位、SLAM 精度评估和人机交互。 Isaac 通过利用 GPU 加速提供实时 AprilTag 检测，同时实现高解码稳健性。

除了检测之外，Isaac 还对所有检测到的标签进行标签姿态估计。 我们根据相机内在参数、标签的大小和标签角的像素坐标计算标签姿势的估计值，返回标签相对于相机的旋转和平移。 具体来说，鉴于以下情况：

* X 和 Y 中的相机焦距，以像素（每弧度）为单位。

* 相机主点 X 和 Y，以图像中像素 (0,0) 的像素为单位。

* 标记大小 W，其中标记为正方形 W x W，以调用者方便的单位表示。 建议使用米或厘米。

Isaac SDK 返回以下内容：

* 所有检测到的 AprilTag 的标签 ID，格式为 `<tagFamily>_<tagID>`。 例如，对于标签系列 Tag36h11 和 ID 7，返回的标签 ID 是 tag36h11_7。

* 在 2018.3 版本中，支持 tag36h11 标签系列。 计划在未来的版本中将该算法扩展到其他标签系列。

* 观察到的标签的像素坐标，从左上角开始，依次是右上角、右下角、左下角。

* 表示标签相对于相机框架的方向的四元数； 和

* 一个 3维向量，指示标签中心相对于相机位置的位置，使用与指定标签尺寸相同的单位。

相对于相机的坐标系为：

* 右撇子

* X轴向右

* Y轴向上

* Z轴向内

* 列扫描旋转矩阵，即 X、Y 和 Z 轴的重新映射列表。

该估计的精度与到标签的距离成反比。

## 源码
Isaac 使用静态库形式的 AprilTags 检测代码。

AprilTags 检测和姿势估计被包装为 Isaac codelet，并且在 Isaac 存储库中可用。

## Isaac Codelet
包装 AprilTags 检测的 Isaac codelet 获取输入图像，并发布检测到的标签列表以及标签角的坐标。 它还使用相机内在特性、输入标签大小和检测到的标签的坐标来估计这些标签的姿态。 相对于相机的位置，姿势由四元数和平移向量表示。

## 运行示例应用程序
AprilTags 示例应用程序使用 Realsense 立体相机。 首先将相机连接到主机系统或您正在使用的 Jetson 平台。 然后使用以下过程之一运行包含的示例应用程序。



### 在主机系统上运行示例应用程序
1. 使用以下命令构建示例应用程序：
```bash
bob@desktop:~/isaac/sdk$ bazel build //apps/samples/april_tags
```

2. 使用以下命令运行示例应用程序：
```bash
bob@desktop:~/isaac/sdk$ bazel run //apps/samples/april_tags
```
### 在 Jetson 上运行应用程序
1. 在主机上构建一个包，然后将其部署到 Jetson 系统中。

2. 按照应用程序控制台选项中的说明将 `//apps/samples/april_tags:april_tags-pkg` 部署到机器人。

3. 登录到 Jetson 系统并使用以下命令运行应用程序：
```bash
bob@jetson:~/$ cd deploy/bob/april_tags-pkg
bob@jetson:~/deploy/bob/april_tags-pkg$./apps/samples/april_tags/april_tags
```
其中“`bob`”是您在主机系统上的用户名。


## 在 Websight 中查看应用程序的输出
在应用程序运行时，通过导航到` http://localhost:3000` 在浏览器中打开 Isaac Sight。 （如果在 Jetson 平台上运行应用程序，请确保使用 Jetson 系统的 IP 地址而不是本地主机。）

在 Websight 中，一个名为 Tags 的窗口显示输入图像，其中一个绿色半透明矩形覆盖在检测到的 AprilTags 之上：

![](https://docs.nvidia.com/isaac/_images/image1.png)




































