## 概述
1. ROS 2 的所有命令行指令都以 ros2 作为主入口点，并采用 "动词-名词" 的结构。
2. 例如 ros2 topic list，其中 topic 是动词（操作的对象类别），list 是子动词（具体的操作）
## 核心交互与调试
### ros2 node：节点管理
用于查看和管理正在运行的 ROS 节点
1. ros2 node list: 列出当前 ROS 域中所有可见的活跃节点
2. ros2 node info <node_name>: 显示特定节点的详细信息，包括它的订阅、发布、服务和动作
### ros2 topic：话题通信
用于检查和操作 ROS 话题，这是最主要的数据传输方式
1. ros2 topic list: 列出所有活跃的话题
2. ros2 topic list -t: 在列出话题的同时，显示它们的消息类型（例如 std_msgs/msg/String）
3. ros2 topic info <topic_name>: 显示特定话题的信息，包括其消息类型、发布者数量和订阅者数量
4. ros2 topic echo <topic_name>: (极其常用) 实时打印某个话题上发布的数据
5. ros2 topic pub <topic_name> <msg_type> '<msg_data>': (极其常用) 向某个话题发布单条消息。<msg_data> 必须使用 YAML 格式
```Bash
// 示例 (发布一个字符串)
ros2 topic pub /chatter std_msgs/msg/String "data: 'Hello world'"
// 示例 (发布一个点)
ros2 topic pub /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.1}, angular: {z: 0.5}}"
```
1. ros2 topic hz <topic_name>: 测量某个话题上消息发布的平均频率 (Hz)
### ros2 service：服务通信
用于检查和调用 ROS 服务（Services），这是一种请求/响应的通信模式
1. ros2 service list: 列出所有可用的服务
2. ros2 service list -t: 在列出服务的同时，显示它们的服务类型
3. ros2 service type <service_name>: 显示特定服务的类型
4. ros2 service call <service_name> <srv_type> '<request_data>': (极其常用) 调用一个服务并等待响应。请求数据同样使用 YAML 格式
```Bash
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 5, b: 10}"
```
### ros2 action：动作通信
用于检查和调用 ROS 动作，这是一种用于长时间运行任务（如导航）并提供反馈的通信模式
1. ros2 action list: 列出所有可用的动作
2. ros2 action info <action_name>: 显示特定动作的信息
3. ros2 action send_goal <action_name> <action_type> '<goal_data>': (极其常用) 向一个动作服务器发送一个目标
```Bash
//示例 (导航): 添加 --feedback 标志可以在发送目标后实时打印反馈信
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose "{pose: {stamp: {sec: 0}, frame_id: 'map', pose: {position: {x: 1.0, y: 1.0, z: 0.0}, orientation: {w: 1.0}}}}"
```
### ros2 param：参数管理
用于动态地获取或设置节点的参数
1. ros2 param list: 列出所有节点及其拥有的参数
2. ros2 param get <node_name> <param_name>: 获取某个节点上特定参数的值
3. ros2 param set <node_name> <param_name> <value>: (极其常用) 动态设置某个节点上特定参数的值
4. ros2 param dump <node_name>: 将某个节点的所有参数值转储（保存）到一个 YAML 文件中
## 执行与启动
### ros2 run：运行单个节点
用于从一个包中启动一个单独的可执行文件（节点）
1. ros2 run <package_name> <executable_name>: 这是 ros2 run 的标准用法
```Bash
ros2 run turtlesim turtlesim_node
```
2. ros2 run 也可以用来传递 ROS 特定的参数，例如
```Bash
ros2 run my_package my_node --ros-args -r __node:=my_new_node_name (重命名节点)
ros2 run my_package my_node --ros-args -p my_param:=my_value (设置参数)
```
### ros2 launch：启动复杂系统
用于启动一个或多个节点，通常通过 Python 或 XML 编写的 launch 文件 来定义。这是启动一个完整机器人系统的标准方式
1. ros2 launch <package_name> <launch_file_name>: 启动一个 launch 文件
```Bash
ros2 launch turtlesim multisim.launch.py
```
2. Launch 文件可以接受参数
```Bash
ros2 launch my_package my_launch.py my_arg:=my_value
```
## 包与接口
用于管理和查看代码包以及定义的消息、服务和动作接口
### ros2 pkg：软件包管理
1. ros2 pkg list: 列出系统中所有已安装的 ROS 2 软件包
2. ros2 pkg prefix <package_name>: 显示某个包的安装路径（这在查找配置文件时很有用）
3. ros2 pkg executables <package_name>: 列出某个包中所有可执行文件（即可以 ros2 run 的文件）
4. ros2 pkg create: (不常用，通常用 ament 或 colcon 创建，但它是一个快捷方式)
### ros2 interface：接口定义
用于查看 msg (消息), srv (服务), 和 action (动作) 的数据结构定义
1. ros2 interface list: 列出所有可用的接口类型S
2. ros2 interface show <interface_name>: (极其常用) 显示一个接口的具体定义
```Bash
//示例 (查看字符串消息)
ros2 interface show std_msgs/msg/String (将显示 string data)
//示例 (查看服务)
ros2 interface show example_interfaces/srv/AddTwoInts (将显示请求、--- 分隔符和响应)
```
1. ros2 interface package <package_name>: 列出特定包中定义的所有接口
## 数据记录与回放
### ros2 bag：数据包 (Rosbag)
用于记录和回放 ROS 2 系统中的数据，这对于调试、测试和仿真至关重要
1. ros2 bag record <topic1> <topic2> ...: 记录指定话题的数据
2. ros2 bag record -a: 记录所有活跃话题的数据
3. ros2 bag play <bag_directory>: 回放一个之前录制的数据包
4. ros2 bag info <bag_directory>: 显示一个数据包文件的信息（例如包含的话题、时长等）
## 系统诊断与管理
### ros2 doctor：系统诊断
一个方便的工具，用于检查 ROS2 环境和网络设置是否存在常见问题
1. ros2 doctor: 运行一系列检查，并报告警告或错误
2. ros2 doctor --report: 生成一个更详细的系统状态报告
### ros2 daemon：后台守护进程
ros2 CLI 依赖一个后台守护进程来快速发现节点和话题。通常不需要直接操作它，但有时它可能会卡住
1. ros2 daemon stop: 停止守护进程
2. ros2 daemon start: 启动守护进程
3. ros2 daemon status: 检查守护进程的状态
> 如果 ros2 node list 或 ros2 topic list 突然显示为空，而节点正在运行，尝试重启守护进程 (ros2 daemon stop 后再运行一个 ros2 命令) 可能会解决问题
## 帮助系统
1. ros2 -h: 查看所有可用的动词 (verbs)
2. ros2 <verb> -h: 查看某个动词（例如 topic）所有可用的子动词
3. ros2 <verb> <sub-verb> -h: 查看一个完整命令（例如 ros2 topic pub）的详细用法和选项