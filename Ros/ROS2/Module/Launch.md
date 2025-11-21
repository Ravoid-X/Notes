## 概述
1. 在一个复杂的机器人系统中，可能需要同时运行几十个节点，每个节点都需要特定的配置（参数、重映射、命名空间等）
2. Launch 系统就是为了自动化这个过程，用于启动和管理一个或多个 ROS2 节点，使其可配置、可复用。
3. 与 ROS1 主要使用 XML 不同，ROS2 推荐使用 Python 脚本 来编写 Launch 文件。
## Python 即 Launch
1. 一个 .py Launch 文件本质上是一个 Python 脚本，运行 ros2 launch ... 时，ROS2 实际上是在执行这个 Python 脚本
2. 这个脚本的唯一目的是构建并返回一个 LaunchDescription 对象
## 声明式系统
1. 不是在 Python 脚本里立即运行节点，而是在描述一个系统应该是什么样的（如启动节点、设置参数）
2. Python 脚本执行完毕后，会返回一个包含所有这些描述的 LaunchDescription 对象
3. Launch 服务（launch_ros）会解析这个对象，并真正地去执行声明的动作
4. 这种解耦允许 Launch 系统在执行前分析整个系统布局、检查依赖关系、处理事件，甚至在运行时监控和重启节点
## 核心构建块：Action
Launch 系统中可以执行的最小单元，最重要的 Action 包括
1. Node: 启动一个 ROS2 节点
2. IncludeLaunchDescription: 包含另一个 Launch 文件（实现复用）
3. DeclareLaunchArgument: 声明一个可以从命令行传入的参数
4. ExecuteProcess: 执行一个任意的 shell 命令
5. RegisterEventHandler: 注册一个事件处理器（例如：当某个节点退出时，执行另一个动作）
## 参数化与复用
为了使 Launch 文件可复用，不能硬编码所有值
### DeclareLaunchArgument (声明启动参数)
用于在 Launch 文件顶部定义一个变量，可以设置默认值
```Python
DeclareLaunchArgument('use_sim_time', default_value='true')
```
### LaunchConfiguration (使用启动参数)
用于在 Launch 文件内部获取 DeclareLaunchArgument 声明的变量的值
```Python
# 注意：'use_sim_time' 的值不是立即获得的，它是一个“承诺”
# launch 系统在执行时会将其替换为真实的值（'true' 或从命令行传入的值）
use_sim_time_config = LaunchConfiguration('use_sim_time')
```
### Substitutions (替换机制)
LaunchConfiguration 是最常用的一种替换。它告诉系统：“在运行时，请把这个占位符替换成实际的值。” 其他有用的替换还包括：
1. TextSubstitution(text='...'): 插入静态文本
2. FindExecutable(name='...'): 查找可执行文件的完整路径
3. PathJoinSubstitution([...]): 拼接路径
## 示例
```Python
# 1. 导入必要的模块
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
# 2. 定义一个固定的入口函数
def generate_launch_description():
    # 3. 在函数内部，创建 LaunchDescription 对象
    ld = LaunchDescription()
    # 4. 声明启动参数 (Actions)
    use_sim_time_arg = DeclareLaunchArgument(
        'use_sim_time',
        default_value='false',
        description='Use simulation (Gazebo) clock'
    )
    # 5. 创建节点 (Actions)
    my_node = Node(
        package='my_package',
        executable='my_node_executable',
        name='custom_node_name',
        output='screen',
        parameters=[
            # 使用 LaunchConfiguration 来传递参数
            {'use_sim_time': LaunchConfiguration('use_sim_time')}
        ],
        remappings=[
            ('/input/topic', '/remapped/topic')
        ]
    )
    # 6. 将所有 Actions 添加到 LaunchDescription
    ld.add_action(use_sim_time_arg)
    ld.add_action(my_node)
    # 7. 返回 LaunchDescription 对象
    return ld
```
### 分析
1. package: 节点所在的功能包名称 (e.g., turtlesim)
2. executable: 节点的可执行文件名称 (e.g., turtlesim_node)
3. name: (可选) 覆盖节点内部设置的默认名称
4. namespace: (可选) 为节点设置命名空间 (e.g., /my_robot)
5. output='screen': (常用) 将节点的 stdout/stderr (如 RCLCPP_INFO) 打印到终端
6. parameters: (关键) 传递参数：
（1）传递静态值: parameters=[{'param_name': 'value'}]
（2）传递动态值: parameters=[{'param_name': LaunchConfiguration('my_arg')}]
（3）加载 YAML 文件: parameters=[PathJoinSubstitution([... , 'my_params.yaml'])]
7. remappings: (关键) 话题/服务/动作的重映射
## 运行
### 构建功能包
```Bash
colcon build --packages-select my_launch_pkg
source install/setup.bash
```
### 使用默认值运行
```Bash
ros2 launch my_launch_pkg start_turtlesim.launch.py
```
### 传递参数运行
```Bash
ros2 launch my_launch_pkg start_turtlesim.launch.py background_r:=0
```