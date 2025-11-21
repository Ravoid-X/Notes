## 概述
1. 本质是一个抽象层，定义了一套规范，描述了 ROS2 需要一个中间件提供哪些功能，但不关心这些功能具体是如何实现的
2. 为了实现最大的可移植性和最稳定的 ABI（应用程序二进制接口），这套接口被定义为纯 C 语言的 API
3. 这个 RMW 接口（定义在 ros2/rmw 包中）本身不包含任何 DDS 或任何特定通信库的代码，只是一堆 .h 头文件
## 切换 RMW 实现
可以在不重新编译应用程序代码的情况下，切换底层的 DDS 实现，通过环境变量 RMW_IMPLEMENTATION 实现
1. 假设系统中安装了两个 RMW 实现：ros-humble-rmw-fastrtps-cpp，ros-humble-rmw-cyclonedds-cpp
2. 当 ROS2 节点启动时，rcl 层会检查 RMW_IMPLEMENTATION 环境变量
3. 如果设置：export RMW_IMPLEMENTATION=rmw_fastrtps_cpp。ROS2 会在运行时动态加载 librmw_fastrtps_cpp.so 这个适配器库。
4. 所有 rcl 发出的 rmw_ 调用（如 rmw_publish）都会被路由到 Fast DDS 的实现上。
### DDS 选择
1. Fast DDS: 功能丰富，eProsima（DDS 厂商）维护。
2. CycloneDDS: 性能稳定，Eclipse 基金会（社区驱动）维护。
3. Connext DDS: 商业版，提供安全和实时性认证，用于高可靠性产品
## rosidl_typesupport
1. RMW 接口只处理动作和模式（发布、订阅、服务、QoS），但不处理数据本身\
（1）rmw_publish() 只接受一个 const void* (原始字节指针) 作为要发送的数据\
（2）rmw_take() 只返回一个 void* (原始字节指针) 作为收到的数据
2. rosidl_typesupport 负责把 std_msgs::msg::String 对象序列化成原始字节，把收到的字节反序列化回 String 对象
3. 一个完整的 RMW 实现总是成对出现：RMW 实现包 (e.g., rmw_fastrtps_cpp)：提供 API 适配器；TypeSupport 包 (e.g., rosidl_typesupport_fastrtps_c)：提供消息的序列化/反序列化适配器
4. 当切换 RMW_IMPLEMENTATION 时，ROS2 也会自动查找并加载与之配套的 typesupport 库