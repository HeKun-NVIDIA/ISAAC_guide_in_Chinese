# Jetbot应用示例

本节介绍如何将 Isaac SDK 与 NVIDIA 新的高性能模拟平台 Omniverse 集成，以让 [Jetbot](https://www.nvidia.com/en-in/autonomous-machines/embedded-systems/jetbot-ai-robot-kit/) 在模拟中跟随球。 本节作为使用三个 [Jetbot](https://www.nvidia.com/en-in/autonomous-machines/embedded-systems/jetbot-ai-robot-kit/) 应用程序进入 Omniverse 和 Isaac SDK 的 Python API 的有价值的切入点。 所有示例应用程序都存在于 `jetbot_jupyter_notebook`。

## 构建并运行 Jetbot Jupyter Notebook
在 Isaac SDK 存储库中，运行 `jetbot_jupyter_notebook Jupyter` 应用程序：

```bash
bob@desktop:~/isaac/sdk$ bazel run apps/jetbot:jetbot_jupyter_notebook

```
您的 Web 浏览器应打开 Jupyter notebook 文档。 如果没有，请在控制台上搜索链接：它看起来像 `http://localhost:8888/notebooks/jetbot_notebook.ipynb`。 在浏览器中打开该链接。

## 使用虚拟游戏手柄远程控制 Jetbot
此示例演示如何使用 Omniverse 和 Jupyter notebook 远程控制 Jetbot。

* `Omniverse 中的 Jetbot`：按照基于 NVIDIA Omniverse 构建的文档 Isaac Sim 启动模拟器并在 `omni:/Isaac/Samples/Isaac_SDK/Robots/Jetbot_REB.usd` 打开舞台。 启动仿真和 Robot Engine Bridge。

    在 Jupyter Notebook 中，按照单元格启动 SDK 应用程序。 一旦连接到模拟器，您就可以使用虚拟游戏手柄从 Omniverse 中的站点移动 Jetbot。

    ![](https://docs.nvidia.com/isaac/_images/jetbot_remote.jpg)


现在我们要在 Omniverse 中搭建一个训练环境。

## 构建训练环境
随着 Jetbot 模型正常工作并能够通过 Isaac SDK 控制它，我们现在可以训练检测模型，它允许机器人识别并随后跟随一个球。 当我们希望最终将经过训练的模型和随附的控制逻辑部署到真实的 Jetbot 时，在 Omniverse 中构建的训练场景能够在物理世界中重新创建非常重要。 本节中构建的模拟环境是为了模仿我们创建的真实世界环境，因此您可以选择以不同方式设计您的环境。 NVIDIA 建议使用纸板箱或枕头的边缘作为环境的边界。

我们通过导航到“[菜单](https://docs.omniverse.nvidia.com/app_create/app_create/interface.html#menu-bar)”工具栏中的“[创建](https://docs.omniverse.nvidia.com/app_create/app_create/interface.html#create-menu)”>“网格”>“球体”，通过添加 5 个立方体网格（对应于 1 层和 4 面）来开始构建场景。 默认情况下，立方体的尺寸为 100 厘米，因此使用“详细信息”面板适当地缩放和平移它们以创建所需大小的盒子。 由于墙壁的银色默认网格颜色在现实中很难重现，我们创建了一种新材料，并将新 OmniPBR 材料的颜色和粗糙度属性调整为类似于纸，并将其应用于 5 个立方体网格。 生成的场景如下图所示：

![](https://docs.nvidia.com/isaac/_images/jetbot_training_environment1.jpg)

然后可以添加资产以引入检测模型的障碍； 检测模型不能将这些资产误认为是球体。 从[内容管理器](https://docs.omniverse.nvidia.com/app_create/app_create/interface.html#content-manager)中，一些代表普通家居用品的资产被拖放到舞台上。 添加资产的网格定位为不与地板相交。

接下来，我们在 Jetbot 将跟随的球的模拟中创建表示。 球体网格被添加到场景中并放置在 Xform 元素中以允许使用域随机化。 使用[语义模式编辑器](https://docs.omniverse.nvidia.com/app_isaacsim/app_isaacsim/camera.html#semantic-schema-editor)添加了用于目标检测检测的类标签。 最后，将[ Sphere Lights](https://docs.omniverse.nvidia.com/app_create/app_create/lighting.html#sphere-light) 和 *jetbot.usd* 文件添加到场景中。

![](https://docs.nvidia.com/isaac/_images/jetbot_training_environment2.jpg)

如果上面显示的场景用于生成训练数据和训练检测模型，那么除非部署 Jetbot 的物理环境与上述模拟场景完全匹配，否则真实 Jetbot 使用训练模型进行推理的能力将受到影响。 在物理世界中重建场景的复杂细节将非常困难。 因此，重要的是创建一个能够将其训练概括和应用到类似物理环境的检测模型。 为此，将域随机化 (DR) 组件添加到场景中的实体，创建更加多样化的训练数据集，从而提高检测模型的鲁棒性。

场景中显示的所有项目都可以在纸盒范围内自由移动，并分别使用 DR 移动和旋转组件绕其 Z 轴旋转。 颜色组件应用于球体网格，允许训练检测模型来检测任何颜色的球。 球体灯中添加了光和运动组件，因此可以使用各种阴影和光强度捕获训练数据。 请注意，Jetbot 模型可以移动和旋转，因此可以从多个位置和角度捕获训练数据。 所有域随机化组件的持续时间均设置为 0.3 秒。 将场景另存为 jetbot_inference.usd。 有了适当的模拟环境，现在可以收集数据并训练检测模型。


## 训练检测模型
要生成数据集并训练检测模型，请参阅 Isaac SDK 文档中的使用 [DetectNetv2 推理流程进行目标检测](https://docs.nvidia.com/isaac/packages/detect_net/doc/detect_net.html#object-detection-with-detect-net)，注意以下差异。 与其使用 Unity3D 生成训练图像，不如使用 Omniverse。 Isaac SDK 中 packages/ml/apps/generate_kitti_dataset 下的 generate_kitti_dataset.app.json 文件被更改为生成 50000 个训练图像和 500 个测试图像。 在运行 generate_kitti_dataset 应用程序之前，确保将 Omniverse 视口中的相机切换到 Jetbot 的第一人称视角，创建 Robot Engine Bridge 应用程序，并启动模拟。 请注意，在使用迁移学习工具包 (TLT) 训练检测模型之前，您必须安装 TensorRT、CUDA 和 CuDNN，并确保遵循所有安装说明。

与迁移学习工具包一起使用的文本文件被修改为仅检测“球体”对象。 此外，由于在数据集生成过程中经常创建重复图像，epochs 的数量从 100 减少到 20。跟踪对象检测管道，直到导出训练模型（.etlt 文件）。 您现在可以在我们的 Isaac 应用程序中使用经过训练的模型来执行推理。

## 在模拟中运行推理
此示例演示如何使用现有的经过训练的模型、Omniverse 和 Jupyter Notebook 对目标进行推理。

* Omniverse 中的 Jetbot：按照基于 NVIDIA Omniverse 构建的文档 Isaac Sim 启动模拟器并在 `omni:/Isaac/Samples/Isaac_SDK/Scenario/jetbot_inference.usd` 打开舞台。 启动仿真和 Robot Engine Bridge。

* 在 Jupyter Notebook 中，按照单元格启动 SDK 应用程序。 连接到模拟器后，您可以在可视窗口中查看推理输出。

![](https://docs.nvidia.com/isaac/_images/jetbot_inference.jpg)

## Jetbot 在模拟中自主跟随物体
此示例使用 Omniverse 和 Jupyter notebook 演示了 Autonomoulsy Follow Ball Object。

* Omniverse 中的 Jetbot：按照基于 NVIDIA Omniverse 构建的文档 Isaac Sim 启动模拟器并在 `omni:/Isaac/Samples/Isaac_SDK/Scenario/jetbot_follow_me.usd` 打开舞台。 启动仿真和 **Robot Engine Bridge**。

    在 Jupyter Notebook 中，按照单元格启动 SDK 应用程序。 一旦它连接到模拟器，您就可以在 Omniverse 中移动球并在观察窗口中检查 Jetbot 是否正在跟随球。

    ![](https://docs.nvidia.com/isaac/_images/jetbot_following_object.jpg)







































