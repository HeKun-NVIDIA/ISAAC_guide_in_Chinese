# 使用DOPE进行3D物体姿态估计

[深度对象姿态估计 (DOPE:Deep Object Pose Estimation)](https://arxiv.org/abs/1809.10790) 从单个 RGB 图像执行已知对象的检测和 3D 姿态估计。 它使用深度学习方法来预测对象 3D 边界框的角点和质心的图像关键点，并使用 PnP 后处理来估计 3D 姿态。 该算法不同于现有的Pose CNN模型； 因此，它为 Isaac SDK 中的 3D 姿势估计工具集提供了更多多样性。

![](https://docs.nvidia.com/isaac/_images/model.png)

## 推理
Isaac SDK 包支持使用 TensorRT 的 DOPE 推理和使用 DopeDecoder codelet 的 C++ 后处理。 每个 DOPE 网络模型都支持检测单个对象类的多个实例。

![](https://docs.nvidia.com/isaac/_images/inference_pipeline.png)


以下命令使用默认示例模型（在 OpenImages v4 上预训练 VGG19）和图像运行涂料推理应用程序以检测 YCB 对象破解器 (003_cracker)：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/object_pose_estimation/apps/dope:dope_inference
```
在 localhost:3000 打开 Sight 以查看结果。
![](https://docs.nvidia.com/isaac/_images/inference_sample.png)

使用 Realsense 摄像头使用命令实时运行推理：

```bash
bob@desktop:~/isaac/sdk$ bazel run packages/object_pose_estimation/apps/dope:dope_inference -- --mode realsense
```

对其他 YCB 对象的推理
[Deep_Object_Pose github](https://github.com/NVlabs/Deep_Object_Pose) 为额外的 [YCB 对象](https://www.ycbbenchmarks.com/object-models/)提供预训练的 torch 模型（请注意，这些模型使用在 ImageNet 上预训练的 pytorch model zoo 中的 VGG19）。 要使用不同的模型运行推理应用程序，请将番茄汤 (YCB 005_tomator_soup_can) 的权重从 [Deep_Object_Pose github](https://github.com/NVlabs/Deep_Object_Pose) 下载到 /tmp/soup_60.pth。 使用以下命令将torch模型转换为 ONNX 模型：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/object_pose_estimation/apps/dope:dope_model -- --input /tmp/soup_60.pth

```
这会在 /tmp/soup_60.onnx 中生成 ONNX 模型。 将推理应用程序与此模型结合使用：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/object_pose_estimation/apps/dope:dope_inference -- --mode realsense --model /tmp/soup_60.onnx --box 0.06766 0.102 0.0677 --label soup

```
请注意，新对象的 3D 边界框大小必须与 --box 一起提供，作为 DopeDecoder Codelet 执行 PnP 的输入。 YCB 模型的边界框大小可以在 [Deep_Object_Pose 配置](https://github.com/NVlabs/Deep_Object_Pose/blob/master/config/config_pose.yaml)或 [YCB 论文](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=7251504)中找到。

## 训练
DOPE 训练目前不是 Isaac SDK 的一部分。 参考[Deep_Object_Pose github scripts/train.py](https://github.com/NVlabs/Deep_Object_Pose)中的torch训练脚本。

对于托管在 Deep_Object_Pose github 上的任何预训练模型，或使用 Deep_Object_Pose github 中的训练脚本训练的模型，请使用：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/object_pose_estimation/apps/dope:dope_model -- --input <path to torch model>

```

将torch模型转换为 ONNX 模型以与 Isaac SDK 的 DOPE 推理管道一起使用。

要使用自定义对象进行训练，请参阅 Omniverse Isaac Sim 的合成数据生成工具。[ FAT 数据集](https://research.nvidia.com/sites/default/files/pubs/2018-06_Falling-Things/readme_0.txt)中指定了离线训练数据格式。




































