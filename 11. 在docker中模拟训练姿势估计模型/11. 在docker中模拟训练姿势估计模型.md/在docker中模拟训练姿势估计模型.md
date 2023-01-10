# 在docker中模拟训练姿势估计模型

物体检测和 3D 姿态估计在机器人技术中起着至关重要的作用。 导航、对象操作和检查等各种应用都需要它们。 Isaac SDK 中的 3D 对象姿态估计应用程序提供了一个框架，可以在模拟中完全训练任何模型的姿态估计，并在模拟和现实世界中测试和运行推理。

要详细了解 Isaac SDK 中 pose_cnn_decoder 的内部工作原理，您可以查阅文档：

使用此 Docker 映像，您将能够训练姿势估计网络并将其用于 Isaac SDK 应用程序中的推理。

## 怎么运行的
结合使用模拟、Isaac SDK 应用程序和您自己的 3D 模型，我们将首先使用 [Docker 中的模拟训练目标检测](https://docs.nvidia.com/isaac/doc/tutorials/training_in_docker.html#training-in-docker)创建一个对象检测网络。

然后我们将使用 Isaac SDK 的姿势估计训练应用程序以及模拟场景来生成供应用程序使用的样本。

生成足够的样本后，训练停止并存储经过训练的网络。 您可以访问应用程序生成的数据和模型。

最后，我们会将这个经过训练的模型转换为可用于推理的格式，并设置要处理的视频源。

## 主机设置
### 硬件要求
* NVIDIA Pascal GPU 或更新版本。 GTX 1080Ti 和 Titan V 是推荐的最低 GPU。

### 软件要求
* Ubuntu 18.04

* Docker 19.03 或更新版本

* NVIDIA CUDA 驱动程序

* NVIDIA NGC 帐户和 API 密钥

运行以下脚本以在 Ubuntu 18.04 桌面上安装软件要求。 您可以将以下命令复制并粘贴到终端窗口：

```bash
#Docker CE repository
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

#NVIDIA CUDA Drivers
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/cuda-ubuntu1804.pin
sudo mv cuda-ubuntu1804.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget http://developer.download.nvidia.com/compute/cuda/10.2/Prod/local_installers/cuda-repo-ubuntu1804-10-2-local-10.2.89-440.33.01_1.0-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu1804-10-2-local-10.2.89-440.33.01_1.0-1_amd64.deb
sudo apt-key add /var/cuda-repo-10-2-local-10.2.89-440.33.01/7fa2af80.pub

#NVIDIA Docker
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

#Install packages
sudo apt-get update
sudo apt-get install docker-ce
sudo apt-get -y install cuda
sudo apt-get install -y nvidia-container-toolkit

#Add yourself to the docker group
sudo usermod -a -G docker $(id -nu)
echo "All installed"
```

安装所有软件后，重新启动计算机以加载 NVIDIA CUDA 驱动程序：

```bash
sudo shutdown -r now

```
重新登录到您的 Ubuntu 桌面环境并打开一个新的终端窗口。 使用以下命令验证安装是否顺利：

```bash
docker run --gpus all nvidia/cuda:10.0-base nvidia-smi

```

您应该会看到一条消息，指示来自您的 GPU 和 CUDA 库的一些统计信息。 以下是此命令的示例输出。

```bash
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 440.33.01    Driver Version: 440.33.01    CUDA Version: 10.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  TITAN V             On   | 00000000:17:00.0 Off |                  N/A |
| 30%   44C    P8    26W / 250W |      0MiB / 12066MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  TITAN V             On   | 00000000:65:00.0  On |                  N/A |
| 30%   44C    P8    27W / 250W |    404MiB / 12063MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    1      2983      G   /usr/lib/xorg/Xorg                           256MiB |
|    1      3117      G   /usr/bin/gnome-shell                         145MiB |
+-----------------------------------------------------------------------------+
```

如果您遇到问题，请参阅 Isaac 常见问题解答或访问论坛。

### NGC Docker 注册表设置
NGC 注册表托管用于 AI 的 Docker 图像，以及用于 HPC、AI 和 NVIDIA 及合作伙伴的其他技术的模型、数据集和工具。 要使用本教程，您需要拥有一个帐户并创建一个 API 密钥。 这将允许您下载 isaac-experiments 图像，以及用于迁移学习的预训练模型。

访问 [NGC](https://ngc.nvidia.com/) 以设置新帐户。 登录后，访问 [API 密钥创建页面](https://ngc.nvidia.com/setup/api-key)并按照屏幕上的说明进行操作。

妥善保管 API 密钥； 我们将在设置过程中使用它几次。

使用以下命令（包括您的 API 密钥）登录 NGC Docker 注册表：


### 第一次运行
该图像创建了许多文件，为您提供定制和控制模拟和训练行为的机会。 我们建议创建一个单独的文件夹来保存您的实验。

运行以下命令创建 `isaac-experiments` 文件夹并为容器创建启动脚本。

```bash
#Create a new folder to hold the generated data and trained models.
mkdir ~/isaac-experiments
cd ~/isaac-experiments
#Deploy the startup helper script to your experiments folder.
docker run -u $(id -u) -v $PWD:/workspace nvcr.io/nvidia/isaac-ml-3dpose:2020.2 -s
```

将在您的 `isaac-experiments `文件夹中创建一个名为 `start.sh` 的新脚本。

每次运行 `start.sh `脚本时都需要 root 密码，因为它将用于配置 X 服务器以允许从 Docker 内部进行通信。

首先，在与上面相同的终端窗口中运行以下命令：

```bash
./start.sh

```


![](https://docs.nvidia.com/isaac/_images/isaac_experiments_folder.png)

最后，您将看到以下消息：

```bash
***************************************************************************************************
* Open your browser and go to:                                                                    *
* http://localhost:8888/notebooks/object_detection_from_sim.ipynb?token=object-detection-from-sim *
*                                                                                                 *
***************************************************************************************************
Press Ctrl-C twice to exit
```

按住 Control 键并单击链接以打开浏览器。 此链接会将您带到可以运行的 Jupyter notebook。 Jupyter 笔记本将是您控制数据集生成和实验流程的主要方式。

## 数据集生成配置
### 您的工作区
请注意，在 Docker 容器内，您的 `~/isaac-experiments` 文件夹安装为 `/workspace`。

每当你看到从 docker 内部调用这个文件夹时，它实际上指的是你主机中的一个文件夹，你对这个文件夹中的文件所做的任何更改都会立即反映出来。 这是在容器和您的机器之间传递文件、可执行文件和脚本的好方法。

### Jupyter 变量设置
在 Jupyter Notebook 的顶部，您需要配置一些变量。 这些控制数据集的生成方式：

* KEY 这可以是任何字符串。 我们提供了一个默认密钥，但您应该设置一个唯一的密钥。 您不需要为每个实验更改它，但您需要将它提供给使用您的训练模型进行推理的应用程序。

* USER_EXPERIMENT_DIR 训练过程中生成的文件的保存位置

* DATA_DIR 将定位数据集的位置。

* SPECS_DIR 实验规范的文件夹，它控制所有高级训练参数

* TRAINING_TESTS 要生成的测试图像的数量。 在评估训练模型期间使用。

* TRAINING_SAMPLES 要生成的训练图像和标签的数量。 更大的集合意味着更精确的点，但需要更长的时间来处理。


## 开始训练
在 Jupyter notebook 中设置参数后，请按照每个单元格上的说明运行它。

### 添加您自己的 3D 模型
如果您有想要包含在图像数据集中的 3D 模型，只需将其以 FBX 格式放在 isaac-experiments/models 文件夹中。 不带扩展名的文件名将用作数据集上的标签。 请保持 3D 模型与网球模型的比例大致相同； 一个好的经验法则是使用您可以在书桌或桌子上找到的物品。

您可以添加的模型类型有一些限制。

* 此模式不支持透明纹理。 如果您的模型中有透明纹理，它们可能无法正确渲染或标记。

* 纹理应嵌入 FBX 模型中。 如果您的纹理看起来缺失，请尝试使用嵌入的纹理重新生成您的模型。 不支持将模型纹理添加为附加文件。


## 故障排除
* 意外删除了一个文件，但现在没有任何效果：如果您删除了其中一个提供的文件，笔记本电脑可能会开始抛出错误。 如果发生这种情况，您不必丢失所有工作。 只需重新启动 Docker 容器，启动脚本就会用一个新的副本替换任何所需的文件。

* 生成数据集时出现黑色窗口：这是正常现象，因为模拟需要访问连接到 GPU 的真实显示器。 为此，我们将内部模拟连接到您主机上的真实 X 服务器。 您可以安全地忽略此窗口——一旦数据集生成完成，它就会消失。

* 查看 Isaac Websight 时，没有图表或没有显示任何信息：尝试启用左侧边栏上的所有禁用通道。 这应该恢复图表。

## 接下来
如果您想探索使用 Isaac 开发机器人应用程序，请查看本文档中的其他教程。

您可以继续使用此 Docker 映像作为您的开发容器，但如果您在主机上执行完整安装，您将获得更好的性能。
































































