# 使用Sight的远程操纵杆

控制机器人运动的传统方法是使用直接连接到机器人的操纵杆。 但是，在 Isaac 中，Virtual Gamepad Sight 小部件可用于通过网络向机器人发送模拟传统操纵杆轴的信号。 该小部件允许三种输入机制，一次只能使用其中一种：

* [浏览器识别标准](https://w3c.github.io/gamepad/#remapping)游戏手柄控制器连接到运行 Sight Client 的机器，它使用浏览器的 [Gamepad API](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad_API)。

* 连接到运行 Sight Client 的计算机的鼠标。 鼠标与小部件中的虚拟鼠标交互，模拟操纵杆轴。

* 连接到运行 Sight 客户端的计算机的键盘。 方向键分配在键盘上以模拟操纵杆轴。

## 设置 Isaac 应用程序以使用虚拟游戏手柄小部件

要从 Sight 使用 Virtual Gamepad Widget，请在应用程序的 JSON 文件中设置 VirtualGamepadBridge，如下所示：

1. 添加以下节点：

```json
{
  "name": "virtual_gamepad_bridge",
  "components": [
    {
      "name": "message_ledger",
      "type": "isaac::alice::MessageLedger"
    },
    {
      "name": "VirtualGamepadBridge",
      "type": "isaac::navigation::VirtualGamepadBridge"
    }
  ]
}
```
2. 将以下配置参数添加到节点：

```json
"virtual_gamepad_bridge": {
  "VirtualGamepadBridge": {
    "tick_period": "100ms"
  }
}
```
3. 添加连接以启用 Sight 和后端之间的通信：

```json
{
  "source": "websight/WebsightServer/virtual_gamepad",
  "target": "virtual_gamepad_bridge/VirtualGamepadBridge/request"
},
{
  "source": "virtual_gamepad_bridge/VirtualGamepadBridge/reply",
  "target": "websight/WebsightServer/virtual_gamepad_reply"
}
```

4. 将连接添加到处理操纵杆类消息的小码（例如：RobotRemoteControl）：

```json
{
  "source": "virtual_gamepad_bridge/VirtualGamepadBridge/joystick",
  "target": "carter_joystick/isaac.navigation.RobotRemoteControl/js_state"
}

```
5. 在您的应用程序构建文件中包含导航模块：

```json
isaac_app(
    name = "...",
    ...
    modules = ["navigation"],
)

```
6. 在您的应用程序 json 文件中包含导航模块：
```json
modules = ["navigation"]
```

VirtualGamepadBridge 具有以下配置参数：

* tick_period：周期性 tick() 函数调用之间的间隔。

* sight_widget_connection_timeout：自动与 Sight Virtual Gamepad Widget 实例断开连接之前等待的时间（以秒为单位）。 默认值为 30 秒。

* num_virtual_buttons：前端鼠标或键盘正在模拟的模拟操纵杆上的按钮数。 默认值为 12。

* deadman_button：故障安全按钮的按钮编号。 此按钮编号用于从模拟操纵杆的鼠标或键盘生成的消息中。 默认值为 4。


## 关于虚拟游戏手柄小部件
以下是可见的虚拟游戏手柄小部件：

![](https://docs.nvidia.com/isaac/_images/virtual-gamepad-main.png)

小部件的元素

* **Widget ID**：每个小部件实例的唯一 ID，用于确保只有一个这样的小部件实例连接到后端，以便一次只有一个用户处于控制之中。

* **连接到后端按钮**：单击以与后端建立实时且唯一的连接。 当前连接状态在按钮上显示为绿色或红色图标。 连接成功后，模式面板显示在中央。 如果小部件保持断开连接，模式面板将保持隐藏状态以禁止用户交互。

* **模式选择器按钮**：单击以选择操作模式。 所选/可见模式激活为当前操作模式。 所选模式的按钮从灰色变为绿色。

* **Virtual Mousepad Joystick dial**：使用连接的鼠标或触摸板与使用光标的转盘交互，将模拟的操纵杆信号发送到后端。 有关详细信息，请参阅操作模式。

* **Virtual Keypad Joystick presentation**：启用后，显示当前在连接的键盘上按下的键。 这只是被按下的键的视觉呈现。 这些不是可以用鼠标或触摸板点击的按钮。 有关详细信息，请参阅操作模式。

* **Backend Connected to Widget 显示**：显示当前连接到后端的 widget 实例的 ID，作为哪个用户在控制中的指示。

* **Connected Standard/Remapped Gamepads 显示**：显示运行 Sight 的浏览器当前识别的游戏手柄数量。 仅当浏览器识别单个控制器时，此小部件才允许使用连接的控制器进行控制。 如果连接了多个控件，则必须断开除一个以外的所有控件。


## 使用虚拟游戏手柄小部件
如果设置 Isaac 应用程序以使用 Virtual Gamepad Widget 中的过程未完成，则在启动应用程序时不会在后端实例化任何 VirtualGamepadBridge codelet。 在这种情况下，Sight 小部件将保持禁用状态并显示以下消息：

![](https://docs.nvidia.com/isaac/_images/virtual-gamepad-disabled.png)


设置完成后，可以在执行以下步骤后使用 Virtual Gamepad Widget：

1. 单击“连接到后端”按钮连接到后端。

2. 通过单击相应的模式按钮选择所需的模式。 模式的状态由其按钮的颜色指示。 有关模式的说明，请参阅[操作模式](https://docs.nvidia.com/isaac/packages/navigation/doc/virtual-gamepad.html#modes-of-operation)。

3. 要断开您的小部件实例与后端的连接，请单击之前用于连接的相同按钮。

这种小部件只有一个实例可以保持与后端的连接。 这可以防止具有多个 Sight 实例的多个 Sight 用户同时尝试移动同一个机器人。

## 运作模式

### 手柄模式

![](https://docs.nvidia.com/isaac/_images/virtual-gamepad-controller-mode-on.png)


在 Gamepad 模式下，控制器或游戏手柄通过无线或 USB 连接到运行 Sight 的机器，并可通过浏览器的 [Gamepad API](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad_API) 识别来控制机器人。 请注意，在此操作模式下只能连接一个[标准控制器](https://w3c.github.io/gamepad/#remapping)。 如果只有一个标准控制器保持连接，则游戏手柄图标变为绿色，否则图标颜色保持灰色，表示没有控制器可用。 游戏手柄的功能就像直接连接到机器人一样。

### 鼠标模式

![](https://docs.nvidia.com/isaac/_images/virtual-gamepad-mousepad-mode-on.png)

要将连接的鼠标或触摸板与鼠标转盘一起使用，请左键单击并按住中间的深色小圆圈。 按住左键单击的同时，移动光标以移动较大圆形表盘中的黑圈。

当黑色圆圈位于表盘上的绿色区域时，小部件将模拟操纵杆消息发送到后端。 如果它在灰色区域，则不会向后端发送任何消息。

只需按下左键即可移动黑色圆圈，否则它会返回其在表盘上的中心位置，表示没有操纵杆活动。

如果光标移出表盘半径，黑圈会自行重置到中心位置。

黑圈越靠近外围，模拟的操纵杆轴值越接近其极值。 例如，对于较慢的速度，黑圈需要保持在靠近表盘中心的绿色区域内。 对于更高的速度，黑圈必须靠近外围。

### 键盘模式

![](https://docs.nvidia.com/isaac/_images/virtual-gamepad-keypad-mode-on.png)


连接键盘上的 W、S、A、D 键或它们的任意组合可用于向机器人发送模拟操纵杆轴消息。

UI 指示何时按下或释放方向键。














