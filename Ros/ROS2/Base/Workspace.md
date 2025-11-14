## 概述
1. ws 不是一个复杂的软件，而是一个遵循特定规范的目录结构，以及一套用于管理这个结构的工具（主要是 colcon）
2. 简单来说，工作空间是一个容器，用于存放、组织、构建和安装正在开发的一组 ROS2 软件包
## 原理
### 基础环境
1. 当安装 ROS2 时，所有核心软件包（如 rclcpp, std_msgs, ros2 topic 工具）都被安装在一个系统目录中（例如 /opt/ros/humble/）
2. 当在终端运行 source /opt/ros/humble/setup.bash 时，就在“激活”这个基础环境
3. 本质上是告诉操作系统：当寻找 ROS 相关的可执行文件、库或 Python 模块时，请去 /opt/ros/humble/install/ 目录中查找
### 工作空间
1. 工作空间是第二个环境，它“叠加”在基础环境之上
2. 构建完工作空间并运行 source ./install/setup.bash 时，就在告诉操作系统：请优先在工作空间（./install/）里查找 ROS 包。如果找不到，再去基础环境里找。
### 覆盖
以上就是覆盖的核心原理，它允许
1. 添加新包：创建的 my_robot_controller 包在基础环境里没有，但因为激活了工作空间，系统现在能找到它了
2. 覆盖现有包： 如果从 GitHub 下载了 rclcpp 的源码，在工作空间里修改并构建了它，那么系统将使用修改后的版本，而不是原始版本（高级用法）
## 标准格式
### 目录
一个标准的 ROS2 工作空间包含以下四个关键的顶级目录
```
my_ros2_ws/
├── src/           # 源码空间 (Source Space)
├── build/         # 构建空间 (Build Space)
├── install/       # 安装空间 (Install Space)
└── log/           # 日志空间 (Log Space)
```
### src
1. 存放软件包的源代码
2. 可以通过 git clone 把代码库克隆到这里，或者使用 ros2 pkg create 在这里创建新包
3. 构建工具会递归地扫描此目录，查找所有包含 package.xml 文件的子目录，并将它们识别为 ROS 包
### build
1. 存放构建过程中产生的中间文件
2. colcon 会在这里为 src 中的每一个包创建一个子目录。在包的子目录中，colcon 会调用底层的构建系统（如CMake, Make, Python setup.py）
3. 永远不应该手动编辑这个目录。如果构建出错，可以安全地删除这个目录（或特定包的子目录）然后重新构建
### install
1. 存放构建完成后的最终产物（可执行文件、库、Python脚本、.msg 定义、launch 文件等）
2. 这个目录的结构完全模仿了 ROS2 的基础安装目录（/opt/ros/humble/）。会看到 bin/、lib/、share/ 等\
（1）share/：存放 package.xml、launch 文件、urdf 模型、msg/srv/action 定义等\
（2）lib/：存放编译好的库文件（.so）和Python包\
（3）bin/：存放编译好的可执行文件（C++节点）\
（4）include/：存放C++头文件
3. source install/setup.bash 这个命令之所以能工作，就是因为它激活了这个 install 目录，将其添加到了环境路径中
### log
1. 存放 colcon 构建过程中的日志文件
2. 如果构建失败，这是首先应该查看的地方。它会详细记录每个包的配置、编译和安装日志