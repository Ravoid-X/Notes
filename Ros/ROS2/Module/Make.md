## 概述
1. ROS1 的 catkin_make 是一个一体式的 CMake 流程，而 ROS2 的系统是一个分层、解耦、可插拔的元构建系统
2. 有三个核心组件：ament、colcon 和底层的 CMake/setuptools
## 解决问题
### 查找与排序
如何找到工作空间中的所有功能包，并根据它们的依赖关系确定一个正确的编译顺序
### 调用与执行
如何为不同类型（C++或Python）的功能包调用各自对应的构建工具（如CMake或setuptools）
### 隔离与安装
如何确保每个包的编译过程互不干扰（隔离），并如何将编译产物（可执行文件、库、Python脚本、launch文件等）安装到统一的、可被 ROS2 激活的目录中
## Colcon (构建协调器)
### 原理
1. 在命令行中直接交互的工具（例如 colcon build）
2. 一个构建协调工具，本身不编译任何代码
### 核心任务
1. 发现包：递归扫描 src 目录，通过查找 package.xml 文件来识别所有的功能包
2. 解析依赖：读取每个 package.xml 中的依赖项（如 <build_depend>）
3. 拓扑排序：根据依赖关系生成一个有向无环图，确定一个无冲突的编译顺序。例如，如果 pkg_B 依赖 pkg_A，Colcon 保证 pkg_A 一定在 pkg_B 之前被构建
4. 调用执行：按顺序，为每个功能包调用其底层的构建系统（由 package.xml 中的 <build_type> 指定，通常是 ament_cmake 或 ament_python）
5. 隔离环境：为每个包的构建和安装创建独立目录（如 build/pkg_A 和 install/pkg_A），防止相互污染
## Ament (构建系统抽象)
### 原理
一个构建系统框架，提供了一系列工具和 API，用于简化 ROS2 包的构建。不是一个单一的工具，而是一组宏、钩子和脚本
### 核心任务
ament_cmake (针对 C++)
1. 一组 CMake 宏，在 CMakeLists.txt 中使用 find_package(ament_cmake REQUIRED) 和 ament_package() 时，就引入了这些宏
2. 提供了如 ament_target_dependencies() 这样的便利函数，这个函数会自动处理依赖项的头文件包含路径、库链接、编译器定义等，远比手写 target_include_directories 和 target_link_libraries 简单且健壮。
3. 标准化了 install 规则，确保可执行文件、库、package.xml 等都被安装到 install 空间的正确位置
## CMake / setuptools (底层构建工具)
实际的编译器和打包器
### CMake (C++)
一个跨平台的构建脚本生成器。读取 CMakeLists.txt，生成特定平台（如 Linux）的 Makefile（或其他，如 Ninja 文件），然后 make（或 ninja）工具会调用 C++ 编译器（如 g++）来编译代码
### setuptools (Python)
Python 社区标准的打包和分发库，读取 setup.py 和 setup.cfg，处理Python模块的安装