# 使用强化学习的DollyDocking
**注意**

**此应用程序是实验性的，并且可能会在不同版本之间发生重大变化。**

Dolly Docking 应用程序的目标是使用深度神经网络 (DNN) 教会机器人在放置在机器人视线范围内的手推车下方导航。 DNN 的输入是机器人前方环境的占用网格的历史记录，以及目标姿态、速度和加速度向量。 神经网络的输出是接下来三个时间步长的速度曲线。

此应用程序为 Isaac SDK 中的模块化强化学习工作流程提供了参考。 它展示了如何使用多代理场景训练策略 (DNN)，然后使用冻结模型部署它们。 本文档首先描述了如何快速开始推理和训练，然后介绍了有关神经网络策略、训练工作流、小码和健身房状态机的详细信息。

深度学习策略允许用户将基于学习的规划方法与经典控制方法相结合，有效地利用两全其美。 通过模拟和从模拟中的这些错误中学习，可以使 DNN 策略对来自计算机视觉模型以及测距和状态估计的不一致和错误具有鲁棒性。 强化学习算法通常用于学习最佳策略的探索和开发策略可以在环境地图不可用或环境代价地图的生成或计算并非微不足道的情况下提供帮助。


## 快速开始
### 推理
Dolly Docking 应用程序包括两个示例场景，用于在 Isaac Sim Unity3D 中执行推理：未来工厂场景和多代理场景。

### 未来场景工厂
导航到 `isaac_sim_unity3d/builds` 文件夹并运行以下命令：
```bash
bob@desktop:~/isaac_sim_unity3d/builds$ ./factory_of_the_future.x86_64 --scene Factory01 --scenario 6
```

要为场景运行推理，请打开另一个终端窗口，导航到 Isaac SDK 根文件夹，然后运行以下命令：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/rl/apps/dolly_navigation/py_components:dolly_navigation -- --config='packages/rl/apps/dolly_navigation/py_components/fof_pb_inference.json'

```
在模拟器中，机器人和手推车在工厂车间生成，机器人反复导航到手推车的中心，然后在中心停了一段时间，然后再次重生并重复动作。 模拟中至少需要约 10 FPS 才能获得最佳推理性能。 如果机器人撞到小车的轮子，它会在其起始位置重生，小车的方向略有不同。

### 多代理场景
导航到 Isaac Sim Unity3D 根文件夹并从构建文件夹运行以下命令：
```bash
bob@desktop:~/isaac_sim_unity3d/builds$ ./sample.x86_64 --scene dolly_docking_training
```

要为场景运行推理，请打开另一个终端窗口，导航到 Isaac SDK 根文件夹，然后运行以下命令：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/rl/apps/dolly_navigation/py_components:dolly_navigation -- --config='packages/rl/apps/dolly_navigation/py_components/multi_agent_pb_inference.json'

```
在模拟器中，场景中生成了九个机器人和手推车，以及随机放置在手推车边缘的墙壁。 场景中的大多数机器人反复导航到手推车的中心，然后在中心停留一段时间，然后重生并重复动作。 如果机器人撞到小车的轮子或墙壁，它会在新的起始位置重生，小车的方向也会不同。

### 训练
要执行训练，导航到 Isaac Sim Unity3D 根文件夹并从构建文件夹运行以下命令
```bash
bob@desktop:~/isaac_sim_unity3d/builds$ ./sample.x86_64 --scene dolly_docking_training
```
要开始训练，请打开另一个终端窗口，导航到 Isaac SDK 根文件夹，然后输入以下命令：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/rl/apps/dolly_navigation/py_components:dolly_navigation
```
这将启动一个 TensorFlow 实例并使用通过 TCP 接收的数据对其进行训练。 可以通过位于 /packages/rl/apps/dolly_navigation/py_components/trainer.config.json 的配置文件更改训练配置。

训练开始后，Tensorflow 默认会定期输出日志到/tmp/rl_logs 文件夹，检查点输出到/tmp/rl_checkpoints 文件夹。 我们建议将这些更改为 trainer.config.json 文件中的持久存储路径。

每个检查点实例由三个文件组成：

* .meta 文件：表示模型的图结构。

* .data 文件：存储所有已保存变量的值。

* .index 文件：存储变量名称和形状的列表。

要查看 Tensorboard 上的训练进度，请运行以下命令并在浏览器中打开 http://localhost:6006。
```bash
tensorboard --logdir=/tmp/rl_logs
```
rl_logs 目录还将每个时期的奖励存储在一个 .txt 文件中。 在开始增加之前，前几次迭代的奖励往往会减少。

要停止训练，请使用 Ctrl+C 终止应用程序。

**注意**

模型训练需要大量资源。 我们建议在 NVIDIA DGX 或至少具有 100 GB RAM 的多 GPU 虚拟机实例上训练神经网络。 即使使用功能强大的机器，处理数据和训练模型也需要大量时间。 如果不定期删除，保存的检查点会占用大量磁盘空间。


### 利用训练的模型运行推理
训练结束后，生成的检查点文件（`*.meta、*.index、*.data`）可用于推理。

要在多代理场景中使用生成的检查点运行推理，请编辑位于 `packages/rl/apps/dolly_navigation/py_components/multi_agent_py_inference.json` 的 JSON 文件。 在模拟运行时，修改 JSON 变量 restore_path 以包含存储的检查点的路径。

**提示**

如果checkpoint在/tmp/rl_checkpoints文件夹中存储为agent42.ckpt.meta、agent42.ckpt.index和agent42.ckpt.data，则restore_path参数需要设置为/tmp/rl_checkpoints/agent42.ckpt

然后运行以下命令来运行推理：
```bash
bob@desktop:~/isaac/sdk$ bazel run packages/rl/apps/dolly_navigation/py_components:dolly_navigation -- --config='packages/rl/apps/dolly_navigation/py_components/multi_agent_py_inference.json'
```
要在未来工厂模拟中运行推理，您需要编辑位于 packages/rl/apps/dolly_navigation/py_components/fof_py_inference.json 的 JSON 文件。 修改 JSON 变量 restore_path 以包含存储的检查点的路径。 然后，在如上所述启动工厂模拟后，使用以下命令运行 Isaac SDK 应用程序进行推理：

```bash
bob@desktop:~/isaac/sdk$ bazel run packages/rl/apps/dolly_navigation/py_components:dolly_navigation -- --config='packages/rl/apps/dolly_navigation/py_components/fof_py_inference.json'

```

## Codelets
Isaac SDK 中的强化学习工作流程的灵感来自 OpenAI Gym 界面。

以下小代码从模拟中收集信息以运行强化学习循环：

* [差分基础里程计](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-navigation-differentialbaseodometry)

* [RangeScanFlattening](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-perception-rangescanflattening)

* [RangeScanToObservationMap](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-navigation-rangescantoobservationmap)

* [DollyDockingState解码器](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-dollydockingstatedecoder)

* [DollyDockingAux解码器](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-dollydockingauxdecoder)

一旦累积了状态信息，它就会通过相当于 OpenAI Gym 的小码传递，然后发送到样本累加器进行存储或发送到 Python 代码进行推理：

* [TensorAggregator](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-tensoraggregator)

* [TensorDeaggregator](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-tensordeaggregator)

* [StateMachineGymFlow](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-statemachinegymflow)

* [DollyDockingBirth](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-dollydockingbirth)

* [DollyDockingDeath](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-dollydockingdeath)

* [DollyDockingReward](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-dollydockingreward)

* [DollyDockingStateNoiser](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-dollydockingstatenoiser)

* [TemporalBatching](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-temporalbatching)

一旦 Gym 发布预测，它就会通过以下小码流：

* [TensorToCompositeVelocityProfile](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-rl-tensortocompositevelocityprofile)

* [DifferentialBaseVelocityIntegrator](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-planner-differentialbasevelocityintegrator)

* [CompositeToDifferentialTrajectoryConverter](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-utils-compositetodifferentialtrajectoryconverter)

* DifferentialBaseControl

小码连接如下所示：

![](https://docs.nvidia.com/isaac/_images/rl_architecture.jpg)

**注意**

辅助张量允许用户存储来自模拟或其他 SDK 组件的信息和标志，并将其传递给管道中的所有后续小码。 isaac::rl::DollyDockingAuxDecoder 在对接管道中用于此目的。

强化学习应用程序使用以下消息：

* [CollisionProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#collisionproto)

* [CompositeProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#compositeproto)

* [DifferentialTrajectoryPlanProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#differentialtrajectoryplanproto)

* [FlatscanProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#flatscanproto)

* [Odometry2Proto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#odometry2proto)

* [ImageProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#imageproto)

* [RangeScanProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#rangescanproto)

* [RigidBody3GroupProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#rigidbody3groupproto)

* [StateProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#stateproto)

* [TensorProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#tensorproto)

## 模拟
Isaac SDK 和模拟器使用发布/订阅架构进行通信：通过在创建数据的一侧设置 TCP 发布者并在数据摄取的一侧设置 TCP 订阅者，数据在两个进程之间来回传递。

对于Unity 3D模拟，发布ground truth数据的应用是packages/navsim/apps/navsim.app.json。 这是由 NavSim 中的 dolly_docking_training 场景直接加载的。

应用程序使用 TcpPublisher 将传感器和环境数据发布到用户定义的端口。 此数据由训练应用程序使用，后者又将传送和控制命令发送到 NavSim 应用程序，这些命令通过 TcpSubscriber 节点接收。

在训练场景中，每个机器人发布其状态和环境消息，并在通道名称末尾附加索引号，从索引 1 开始。例如，第一个机器人的传送消息在名为 teleport1 的 TCP 通道上接收 等等。 SDK 和仿真之间的所有通道名称和连接都可以在用于小车对接的 Python 应用程序中获得。

能够通过模拟生成无限的数据点是一项强大的资产，弥合了将模拟机器人技术与真实实验分开的“现实差距”。 域随机化试图通过提高各种数据的可用性来弥合现实差距。 在训练场景中，域随机化可以通过以下几种方式实现：

* 占用网格随机化：在小车和机器人周围生成方块以获得不同的激光雷达地图。

* 推车随机化：旋转推车并在每次运行前将其从中心姿势平移。

* Target pose noise randomization：在dolly的目标pose上添加noise来模拟pose detection的错误。

* 里程计噪声随机化：为了模拟硬件中常见的里程计噪声，将随机噪声添加到从模拟计算的里程计信息中

## Isaac SDK 中的健身房状态机流程
Isaac SDK 强化学习工作流程的核心是一个名为 Gym 的状态机，它控制多智能体场景中所有机器人的整个生命周期，并为策略提供训练数据。 状态机执行六个不同的步骤，如下图所示，并提供三个组件——出生、死亡和奖励——其子组件可在运行时动态插入状态机。 这些组件的基类位于 packages/rl/base_components 中。 Gym 期望每个代理都有一份这些组件的副本，并在适当的阶段调用它们的功能。

![](https://docs.nvidia.com/isaac/_images/gym_state_machine.jpg)

状态机的步骤如下：

1. 当状态机开始时，它需要在模拟中以初始姿势生成代理及其环境（推车、墙壁和障碍物）。 为此，它为模拟中的每个代理调用一次 Birth::spawn 函数。 DollyDockingBirth 组件（isaac::rl::Birth 的子组件）负责向模拟传送消息发送消息，这些消息将代理及其环境置于所需的姿势。 它还通过在发布传送消息之前将噪声附加到对象姿势来负责场景中的域随机化。

2. 接下来，状态机等待从 TensorAggregator codelet 接收聚合状态张量。 该张量包含模拟中所有代理的最新状态，以及可能需要的任何辅助信息。

3. 一旦接收到最新的代理状态，状态机就会评估是否应该杀死和重生任何代理。 它为模拟中的每个代理调用一次 Death::is_dead 函数。 DollyDockingDeath 组件（isaac::rl::Death 的子组件）包含决定什么构成无效代理状态的逻辑：例如，如果代理从模拟中的碰撞中注册 CollisionProto，或者代理已经存活太久 . 函数返回 true 的代理在下一步之前重置为新姿势。

4. 在决定一个代理人是死是活之后，状态机收集每个代理人的奖励。 它为模拟中的每个代理调用一次 Reward::evaluate 函数。 DollyDockingReward 组件（isaac::rl::Reward 的子组件）包含根据最近结束的转换向代理分配奖励（浮动）的逻辑。

5. 一旦收集到与上次转换相关的所有数据，就会以聚合形式发布当前状态、奖励和死亡标志以及辅助张量。

6. 神经网络接收代理的当前状态并执行前向传递以输出动作张量，其中包含代理应在不久的将来达到的目标速度。 这个动作张量通过 Gym 传递给 TensorDeaggregator。

循环再次从步骤 1 继续，仅重置在最后一步中刚刚死亡的那些代理。

## 强化学习政策
OpenAI [Spinning Up](https://spinningup.openai.com/en/latest/) 很好地介绍了强化学习。 Spinning Up 提供了流行的强化学习算法的清晰简洁的[实现](https://github.com/openai/spinningup)。 由于其样本效率和对各种环境和 sim2real 应用程序的鲁棒性，[Soft Actor Critic](https://arxiv.org/abs/1801.01290) 算法用于训练对接策略。

深度神经网络策略设计如下：
![](https://docs.nvidia.com/isaac/_images/policy_neural_network.jpg)

该网络接收过去三个时间步长的环境占用图，并将它们传递给卷积骨干网络。 主干的扁平化输出附加到小车的目标姿势（在机器人框架中），以及最后三个时间步长的速度和加速度矢量。 这个组合的线性张量然后通过两个完全连接的层，每个层的大小为 256。然后它输出一个大小为 6 的一维张量，它被认为是接下来三个时间步长的目标速度。

## JSON 管道参数
下表描述了 /packages/rl/apps/dolly_navigation/py_components 中的 JSON 文件参数：

* action_dimension：神经网络输出张量的大小（在这种情况下，是 3 个未来时间步长的线速度和角速度）

* action_scale：神经网络的输出范围（在本例中为+0.5 到-0.5）。 此输出由 TensorToCompositeVelocityProfile 小代码缩放为真实机器人输出，并具有以下可配置参数。

* agent_spawn_randomization：机器人姿态从其中心在 x、y 和角度坐标中的随机化。 该参数列出了每个坐标中允许的最大位移

* agents_per_row：场景中每一行的agent数量

* angle_allowance：agent在被杀死前可以在轴的任一方向上旋转的最大允许弧度

* aux_dimension：辅助张量的大小

* aux_end_of_episode_flag：辅助标志的位置，表示情节或试验是否因为代理已达到其最大年龄而结束

* batch_size：神经网络的训练批量大小

* bias：奖励方程中x坐标、y坐标和角度（弧度）的系数：ax+b|y|−(|y|/|c.tan(|angle|)|)

* buffer_threshold：训练开始前样本累加器收集的最小样本数

* cell_size: 动态观测图中cell的大小，单位为米

* checkpoint_directory：存放tensorflow checkpoints的目录

* collision_penalty：与场景物体碰撞的惩罚

* delay_sending：Gym 状态机启动前等待的时间。 这确保所有节点都有足够的时间启动。

* dividing_space：模拟中两个机器人小车设置之间沿 x 和 y 坐标的距离

* experience_buffer_size：经验缓冲区/样本累加器的大小

* gamma：强化学习的奖励折扣因子，通常固定为0.99

* ideal_docking_pose：机器人相对于 (x,y) 中小车中心的完美对接坐标

* learning_rate：强化学习策略的目标学习率

* look_back：存储在状态张量中作为神经网络输入的过去步数

* max_episode_length：构成单次试验的最大步数

* max_steps_per_epoch：每个epoch的训练迭代次数

* mode：以多种形式之一运行 Isaac 应用程序。 将其值设置为“train”用于训练新策略，“inference_pb”用于冻结模型推理，“inference_py”用于 python 检查点推理

* num_agents_per_sim：单个模拟器实例中存在的代理数。 请注意，特定场景中的代理数量是固定的，因此更改此配置不会增加或减少场景中的代理数量。

* observation_map_dimension：由 flatscan 生成并馈送到神经网络的正方形占用图的一侧

* obstacle_scale_randomization：块从其原始大小的随机化比例。 此参数列出了比例矢量元素中允许的最大位移

* obstacle_separation：在应用随机化之前，障碍物（墙壁、方块等）从小车中心沿 x 和 y 坐标的生成位置

* obstacle_spawn_randomization：墙壁和方块在 x、y 和角度坐标中从其中心的姿势随机化。 此参数列出了每个坐标中允许的最大位移

* output_angular_velocity_range：重新调整接收到的角速度指数的范围

* output_linear_velocity_range：重新调整接收到的线速度指数的范围

* polyak：平均系数（通常在 0.98 - 0.99 之间）。 此参数确定有多少重复网络权重被复制到原始网络。

* reaction_time：Gym 应记录状态的操作消息发布后的延迟秒数


* restore_path：恢复 tensorflow 检查点或冻结模型的路径

* reward_clip_range：如果超出，奖励值将被修剪的范围。 这些值在数字刻度的两侧被修剪。

* sim_instances：并行运行的Unity实例数

* start_coordinate：在模拟中生成代理的坐标

* success_reward：在实现目标的每个时间步收到的奖励

* target_pose_noise：添加到 dolly 的地面真值姿势以模拟姿势检测错误的噪声

* target_separation：target(dolly) 和机器人之间沿 x 和 y 坐标的距离

* target_spawn_randomization：目标推车从其中心在 x、y 和角度坐标中的姿态随机化。 此参数列出了每个坐标中允许的最大位移

* tensorboard_log_directory：存放所有日志的目录

* tick_period：流水线中小码的滴答周期

* timestamp_profile：附加到神经网络的每个预测输出的目标时间步长

* tolerance：从推车中心沿 x 和 y 方向的增量被认为是成功对接结束姿势

* use_pretrained_model：一个标志，如果为真，则表示需要恢复检查点模型

* wall_thickness：集成平面扫描时命中标记为实体的区域的厚度

* x_allowance：代理在被杀死之前可以沿着各自的负x方向和正x方向移动的最大允许距离

* y_allowance：agent在被杀死之前可以沿着各自的正负y方向移动的最大允许距离





















