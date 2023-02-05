# GMapping应用程序
GMapping 是一个使用 OpenSlam 软件库的地图生成工具。 该应用程序允许您创建地图以在其他应用程序中使用。

GMapping 应用程序使用 Carter 参考机器人的 LIDAR 功能。

**注意**

建图是一项计算密集型和存储密集型活动，可能需要微调才能生成合适的建筑地图。 为获得最佳结果，请记录您的建图日志并离线调整 GMapping 参数。

根据机器人的能力，利用里程计或机器人姿势的姿势树。 使用惯性测量单元 (IMU) 来改进结果。

虽然您可以在桌面上运行 GMapping 应用程序，但对于实际的地图绘制，它必须部署并在 Carter 机器人上运行。

## 部署和运行 GMapping 应用程序
1. 按照[应用程序控制台选项](https://docs.nvidia.com/isaac/doc/getting_started.html#deployment-device)中的说明将 //apps/carter/gmapping:gmapping-pkg 部署到机器人。

2. 在选择目标机器人配置文件的同时运行 GMapping 应用程序：
```bash
bob@carter1:~/gmapping-pkg$ ./apps/carter/gmapping/gmapping --more "apps/carter/robots/carter_1.json"

```
## 输出
示例应用程序在整个运行过程中将地图写入 /tmp/map.img_<N>.png 文件夹。 使用以下配置参数为 Gmapping codelet 指定输出路径：

```json
"config": {
  "gmapping.gmapping": {
    "GMapping": {
      "file_path": "/tmp"
    }
  }
}
```
## 建图示例
下面的图像序列说明了随着时间的推移，从第一次激光雷达数据捕获到建筑物地图的建图过程。

![](https://docs.nvidia.com/isaac/_images/map1.png)![](https://docs.nvidia.com/isaac/_images/map2.png)![](https://docs.nvidia.com/isaac/_images/map3.png)![](https://docs.nvidia.com/isaac/_images/map4.png)![](https://docs.nvidia.com/isaac/_images/map5.png)![](https://docs.nvidia.com/isaac/_images/map6.png)![](https://docs.nvidia.com/isaac/_images/map7.png)

## 建图建议
机器人在建图过程中移动的速度会影响结果。 速度越慢，LIDAR 样本的数量越多，从而提高了准确性。 避免急转弯。 配置机器人以限制最大线速度和角速度。

定期匹配和关闭路径循环，以纠正测绘期间里程计和惯性测量中的漂移和错误。 匹配深度是有限的。 在可能的情况下，绕建筑物中的街区导航，例如隔间畜栏和大型建筑元素。 无需驾车穿过已绘制地图的区域； 它会增加噪音。

从帧到帧保持足够的锚点。 这在离开或进入新区域或转入走廊时尤为重要。 避免驾驶太靠近墙壁。 选择与您的建筑拓扑一样高的（匹配）粒子数量，您可以在不丢失扫描匹配或仅恢复到里程计的情况下，这会导致糟糕的建图结果。

使用足够长的范围来帮助维护锚点，但使用较小的更新范围来绘制清晰的地图图像。 理想情况下，您的更新范围不应大于要建图的最大区域长度的一半。

在使用日志建图应用程序完成真实世界捕获后，记录您的扫描和里程计通道以重播和创建具有不同参数的新地图。 调整配置参数以针对您的建图用例进行试验。

使用 GMapping 或 logamppings 生成的地图时，修剪地图图像的灰色边缘。 这减少了文件的大小并提高了使用地图图像的算法的性能。 修改后，将这些地图保存为灰度、压缩、PNG 格式，以显着减小文件大小。











































