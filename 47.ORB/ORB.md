# ORB
这个 gem 提供了一个特征检测器和描述符提取器。

功能用于以下应用程序：

* 使用运动结构的 3D 重建

* 视觉里程计（运动跟踪）和 SLAM

* 基于内容的图像检索

* 图像对齐和全景拼接

“ORB”代表“Oriented FAST and rotated BRIEF”。 这表明 ORB 基于特征检测器 FAST 和二进制描述符 BRIEF。

Rublee 等人的原始出版物，标题为“ORB：SIFT 或 SURF 的有效替代品”，可在此处找到：http://www.willowgarage.com/sites/default/files/orb_final.pdf

与其他特征类型相比，ORB 具有以下关键特性：

* 高效的; 出色的性能质量权衡

* 抗图像噪声

* 旋转不变性

* 多尺度

Isaac SDK ORB gem 紧跟原始出版物。 除此之外，它还：

* 通过在 GPU 上运行来实现 CUDA 中的大部分流水线以获得更高的性能

* 添加空间正则化（“网格过滤”）步骤，通过在图像上更均匀地分布关键点来修复其他实现（例如 OpenCV 的）的基本缺陷。 这对于运动跟踪等应用具有显着优势。

算法步骤如下：

1. 将输入图像下采样到不同的比例级别

2. 在所有级别上提取 FAST 特征

3. 应用网格过滤

4. 提取特征方向

5. 提取描述符


## Gem 提供的类型
### 关键点
关键点具有以下属性：

* x, y：从中提取的图像中的关键点坐标。 注意：由于有多个比例级别，请使用实用函数 GetCoordsAtTopLevel 获取最顶层（初始）图像级别的坐标。

* response：特征的“强度”。 较高的分数意味着较高的局部对比度。

* angle：方向角（以弧度为单位）

* scale：支撑区域的半径（以像素为单位）

* level：从中提取特征的比例级别。 从零开始。

关键点类型是关键点的向量。

### 描述符
描述符具有以下属性：

* id：允许将描述符连接回其起源的关键点的 ID

* elements：二进制描述符数据，打包到一个小的元素缓冲区中。

Descriptors 类型是描述符的向量。

## 如何使用 Gem（界面）
gem 提供了一个函数 ExtractOrbFeatures，用于从图像中提取 ORB 特征和描述符。

作为输入参数，必须传递图像。

作为输出参数，必须传递关键点和描述符对象。

这些是可以设置的附加参数：

* 最大限 features (default 1500): 需要保留的最佳特征数量。

* FAST 阈值（默认值 30）：像素必须展示的最小局部对比度阈值才能被视为特征

* 网格单元格数线性（默认 8）：水平/垂直方向上用于空间正则化的箱数。 箱子的总数是参数的平方。 更高的数字意味着更均匀地分布特征。 设置为 1 将禁用正则化。

* 下采样因子（默认 0.7）：后续比例级别的大小，作为当前级别的比率。 允许介于 0.5（含）和 1.0（不含）之间的值。

* 最大限levels (default 4): 使用多少比例级别。

## 构建包
运行以下命令来构建 gem、codelet 和示例应用程序：

```bash
$ bazel build //packages/orb/...
```


## Isaac Codelets


ORB 包提供了一个单独的小代码，ExtractAndVisualizeOrb，其唯一目的是展示如何将 gem 与 ZED 相机一起使用的示例。

gem 当然可以直接在应用程序中使用，而无需通过 codelet。

![](https://docs.nvidia.com/isaac/_images/orb_websight.png)


### 示例应用程序
该示例应用程序从 ZED 相机接收到的图像中提取 ORB 特征，并使用 Websight 将它们可视化。

无论您在什么平台上运行示例应用程序，请务必先连接 ZED 相机。 在应用程序运行时，通过导航到 http://localhost:3000 在浏览器中打开 Isaac Sight。 如果您在 Jetson 平台上运行应用程序，请确保使用 Jetson 系统的 IP 地址而不是 localhost。

### 主机设备
运行以下命令启动演示：

```bash
$ bazel run //packages/orb/apps/orb_demo
```
### 嵌入式 Jetson 设备
按照应用程序控制台选项中的说明将 `//packages/orb/apps/orb_demo:orb_demo-pkg` 部署到机器人。

使用以下命令通过 SSH 连接到设备：
```bash
$ ssh nvidia@ROBOTIP
```
切换到部署目录后，使用以下命令在设备上运行演示应用程序：

```bash
$ cd orb_demo-pkg/
$ ./packages/orb/apps/orb_demo/orb_demo
```

































