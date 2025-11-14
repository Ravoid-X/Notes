## 概述
1. 采用完全去中心化的对等 (Peer-to-Peer) 架构，节点使用一种称为“发现”的机制自动找到彼此。
2. 使用 DDS（Data Distribution Service，数据分发服务）作为其核心通信骨干，这一转变带来了 ROS2 的核心特性：增强的容错性、多机器人协作通信支持、安全加密通信、精细的 QoS 以及节点生命周期管理。
## 架构
技术栈自底向上分为五层
### DDS
1. 由 DDS 供应商（如 eProsima, RTI, Eclipse）提供的具体通信库，例如 eProsima Fast DDS、Eclipse Cyclone DDS 或 RTI Connext DDS。
2. 负责所有网络数据包的发送、接收、序列化、反序列化以及节点发现。
### RMW
1. 中间件接口，定义了一个最小的 C API，DDS 供应商必须实现这个接口才能将其库插入到 ROS2 中。
2. 例如，rmw_fastrtps_cpp  和 rmw_cyclonedds_cpp  就是该接口的实现
### RCL
1. 客户端库，是一个通用的 C API，位于 RMW 层之上。
2. 实现了所有 ROS 的核心概念（如节点、发布者、订阅者、服务、参数等），但完全与特定的 DDS 供应商解耦，只调用 RMW 层的抽象 C API 
### 特定语言客户端库
1. 机器人开发者最常直接交互的层，主要是 rclcpp (C++) 和 rclpy (Python)
2. 负责调用 rcl 的 C API，并以各自语言的惯用方式（如 C++ 的类、std::shared_ptr，或 Python 的类和上下文管理器）将其封装，提供给用户。
### 应用层
用户编写的机器人应用程序，由 ROS2 节点和组件组成