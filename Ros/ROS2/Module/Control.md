## 引出
ros2_control 是 ROS2 中用于机器人硬件抽象的标准框架，允许上层的“控制算法”与底层的“硬件驱动”解耦
## 优势
### 可重用性
1. 控制器重用：差分驱动（diff_drive_controller）、机械臂轨迹（joint_trajectory_controller）这些控制算法是通用的。可以在任何机器人上使用这些标准控制器，而无需关心底层硬件。
2. 硬件接口重用：为机器人写好了一次硬件接口，就可以无缝接入任何 ros2_control 兼容的控制器
### 实时性
将 ROS2 的非实时世界（如 Rviz、Python 脚本）与机器人的实时控制循环（RT Loop）严格分开
### 仿真集成
ros2_control 的设计（特别是 gazebo_ros2_control 插件）允许在仿真和实体之间使用完全相同的控制器配置，几乎不需要更改上层代码
## ControllerManager (控制器管理器)
### 概念
ros2_control 的总指挥
### 原理
1. 是一个 ROS2 节点（通常作为组件加载到另一个进程中），负责加载、激活、停用和卸载其他所有控制器
2. 拥有并运行“实时控制循环”。ControllerManager 就像一个节拍器，以固定的高频率运行，并精确地协调 read(), update(), write() 这三个阶段
## HardwareInterface (硬件接口)
### 概念
开发者或机器人制造商需要编写的核心代码，是连接 ros2_control 框架和物理硬件（或仿真器）的驱动程序
### 原理
1. 必须从 hardware_interface::SystemInterface (或 ActuatorInterface, SensorInterface) 继承一个 C++ 类
2. 需要实现几个关键的虚函数，最重要的是\
（1）read(time, period)：从硬件读取数据（例如，编码器读数、IMU 数据），并更新 ros2_control 的状态接口
（2）write(time, period)：从 ros2_control 的命令接口获取数据（例如，目标速度），并发送给物理硬件（例如，通过 CAN、UART 或共享内存）
（3）on_init(), on_activate(), on_deactivate()：管理硬件连接的生命周期（例如，打开/关闭串口）
## Controller (控制器)
### 概念
控制算法的实现，一个可重用、通用的插件
### 原理
1. 从 controller_interface::ControllerInterface 继承，实现了 update(time, period) 函数
2. 工作是读取状态，计算逻辑，写入命令
3. 非实时示例：订阅 /cmd_vel 话题，在 ROS 2 回调中接收 Twist 消息并缓存它
4. 实时(在 update() 中)示例：
（1）读取状态：从 HardwareInterface 提供的状态接口读取当前轮子的位置/速度
（2）计算逻辑：根据缓存的 Twist 指令 和 当前的轮子状态，使用运动学模型计算出每个轮子应该达到的目标速度
（3）写入命令：将这个目标速度写入到命令接口，以便 HardwareInterface 在 write() 阶段将其发送到硬件
## URDF 中的 <ros2_control> 标签
### 概念
URDF 中有一个特殊的 <ros2_control> 标签，ros2_control 通过解析机器人的 URDF 文件来了解硬件
### 原理
1. hardware 插件：在这里指定 HardwareInterface 类的插件名称，ControllerManager 会在启动时加载这个插件
2. joint 和 interface：必须在这里声明你硬件暴露了哪些接口
（1）state 接口：硬件能提供的数据（例如 position, velocity）
（2）command 接口：硬件能接收的指令（例如 position, velocity, effort）
3. 示例
```XML
<ros2_control name="MyRobotSystem" type="system">
    <hardware>
        <plugin>my_robot_driver/MyRobotHardwareInterface</plugin>
        <param name="serial_port">/dev/ttyUSB0</param>
    </hardware>
    <joint name="left_wheel_joint">
        <command_interface name="velocity"/> <state_interface name="position"/>   <state_interface name="velocity"/>   </joint>
    <joint name="right_wheel_joint">
        <command_interface name="velocity"/>
        <state_interface name="position"/>
        <state_interface name="velocity"/>
    </joint>
</ros2_control>
```
## 实时控制循环
ros2_control 的核心，ControllerManager 以固定频率不断执行这个循环。必须在实时安全的线程中运行，不能有任何内存分配、I/O 阻塞或不确定的计算\
假设现在是 $t=100ms$ 时刻的循环
### read()
1. ControllerManager 调用 HardwareInterface 的 read() 函数
2. 立即从硬件（例如编码器寄存器）读取 $t=99ms$ 时的实际位置和速度
3. 将这些值写入到内存中的状态接口（例如 state_interfaces_["left_wheel_joint"]["position"] = 1.23）
4. 此时，所有控制器都能看到硬件的最新状态
### update()
1. ControllerManager 挨个调用所有已激活的 Controller 的 update() 函数
2. JointStateBroadcaster (一个特殊控制器)：update() 被调用，读取状态接口（1.23），并将其发布到（非实时的）/joint_states 话题，供 RViz 等工具使用
3. DiffDriveController (主控制器)：update() 被调用\
（1）读取状态接口（1.23）以了解当前状态\
（2）读取它在（非实时）ROS 回调中缓存的最新 /cmd_vel 指令\
（3）执行控制算法（运动学逆解），计算出 $t=110ms$ 时刻的目标速度（例如 1.5 rad/s）\
（4）将这个目标写入到内存中的命令接口（command_interfaces_["left_wheel_joint"]["velocity"] = 1.5）
4. 此时，所有控制器都已计算完毕，硬件的目标指令已准备就绪
### write()
1. ControllerManager 调用 HardwareInterface 的 write() 函数
2. 从命令接口读取目标值（1.5），将这个值转换成硬件指令（例如 SetMotorSpeed(1.5)）并通过串口/CAN 总线发送出去
3. 此时，硬件已经收到了新指令，并开始在 $t=100ms$ 到 $t=110ms$ 之间执行它
### 循环
循环结束，等待 10ms，在 $t=110ms$ 时刻重复此过程