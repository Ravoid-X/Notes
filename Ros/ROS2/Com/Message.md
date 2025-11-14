## 定义
一种数据结构的描述，定义了发布者和订阅者之间交换的数据的字段和类型
## .msg 文件
1. Message是使用简单的、语言无关的 .msg 文件来定义的
2. 这些文件通常存放在一个包的 msg/ 目录下
3. 示例 (std_msgs/msg/String.msg)，只定义了一个名为 data 的 string 字段
```
string data
```
## 语言无关性
1. 在 .msg 文件中定义一次数据结构，当构建时，ROS2 的代码生成器会自动将这个 .msg 文件“翻译”成多种语言的实现\
（1）C++: 生成 .hpp 头文件 (例如 std_msgs/msg/string.hpp)\
（2）Python: 生成 .py 模块
2. 使得 C++ 节点可以无缝地与 Python 节点通信，只要它们都使用相同的 .msg 定义
## 支持的字段类型
### 基本类型
bool, byte, char, float32, float64, int8, int16, int32, int64, uint8, uint16, uint32, uint64, string
### 数组
1. int32[] (动态大小的数组 / std::vector)
2. int32[5] (固定大小的数组 / std::array)
3. string[<=5] (有界字符串数组)
### 嵌套
1. 一个 Message 可以包含其他 Message，这是构建复杂数据结构的方式
2. 例如 geometry_msgs/msg/Pose.msg 包含了 Point 和 Quaternion 两个其他的 Message
```
Point position
Quaternion orientation
```
### Header
1. 一个特殊的、被广泛使用的 Message：std_msgs/msg/Header
2. 包含两个字段
```
builtin_interfaces/Time stamp  # 时间戳
string frame_id               # 坐标系ID
```
3. 几乎所有与现实世界相关的数据（如传感器数据、坐标变换）都应该在 Message 中包含一个 Header 字段，以便进行时间和空间上的同步
### 接口包
1. 专门用于存放 .msg (以及 srv/action) 文件的包通常被称为“接口包”
2. 常见的官方接口包有\
（1）std_msgs: 标准消息 (String, Int32, Header, ColorRGBA...)\
（2）geometry_msgs: 几何消息 (Point, Vector3, Pose, Twist, Transform...)\
（3）sensor_msgs: 传感器消息 (Image, LaserScan, PointCloud2, Imu...)\
（4）nav_msgs: 导航消息 (Odometry, Path, MapMetaData...)
## 示例
### 目标
1. 创建一个接口包 my_robot_interfaces。
2. 在该包中定义一个自定义消息 HardwareStatus.msg，用于报告硬件状态。
3. 创建一个 C++ 节点 hardware_publisher 来发布此消息。
4. 创建一个 C++ 节点 hardware_subscriber 来订阅此消息
### 创建接口包
1. 进入工作空间，并使用 ros2 pkg create 创建一个纯接口包
```Bash
cd ~/ros2_ws/src
ros2 pkg create my_robot_interfaces --build-type ament_cmake
```
2. 在包里创建 msg 目录
```Bash
mkdir my_robot_interfaces/msg
```
3. 创建 .msg 文件 my_robot_interfaces/msg/HardwareStatus.msg
```
# 这是一个自定义的消息，用于描述硬件状态
int64 temperature_celsius
string hardware_id
bool is_motor_on
```
### 配置 package.xml 和 CMakeLists.txt
最关键的一步，目的是告诉 colcon 如何处理 .msg 文件
1. 编辑 my_robot_interfaces/package.xml：在<buildtool_depend>ament_cmake</buildtool_depend> 之后添加
```XML
<build_depend>rosidl_default_generators</build_depend>
<exec_depend>rosidl_default_runtime</exec_depend>
<member_of_group>rosidl_interface_packages</member_of_group>
```
2. my_robot_interfaces/CMakeLists.txt：\
在 find_package(ament_cmake REQUIRED) 之后添加
```CMake
# 找到代码生成器
find_package(rosidl_default_generators REQUIRED)
```
在文件的末尾添加
```CMake
# 这是核心：告诉 ament/rosidl 
# 在 "msg/" 目录下查找接口文件并生成代码
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/HardwareStatus.msg"
)
```
### 创建发布者
1. 在 src 目录下创建第二个包，用于存放 C++ 节点
```Bash
cd ~/ros2_ws/src
ros2 pkg create hardware_publisher --build-type ament_cmake --dependencies rclcpp my_robot_interfaces
```
>添加了 my_robot_interfaces 作为依赖项
2. 编辑 hardware_publisher/src/publisher_node.cpp
```C++
#include "rclcpp/rclcpp.hpp"
// 1. 包含自动生成的 .hpp 头文件
// 格式：<package_name>/msg/<message_name_snake_case>.hpp
#include "my_robot_interfaces/msg/hardware_status.hpp" 
#include <chrono>
using namespace std::chrono_literals;

class HardwarePublisherNode : public rclcpp::Node{
public:
    HardwarePublisherNode() : Node("hardware_publisher")  {
        // 2. 创建发布者时，模板参数使用 Message 类型
        publisher_ = this->create_publisher<my_robot_interfaces::msg::HardwareStatus>(
        "hardware_status", 10);
        timer_ = this->create_wall_timer(
            1s, std::bind(&HardwarePublisherNode::timer_callback, this));
        RCLCPP_INFO(this->get_logger(), "Hardware Publisher 节点已启动.");
    }

private:
    void timer_callback()  {
        // 3. 实例化 Message 对象
        auto msg = my_robot_interfaces::msg::HardwareStatus();
        // 4. 填充 Message 的字段
        // 注意：字段名称与 .msg 文件中定义的完全一致
        msg.temperature_celsius = 42;
        msg.hardware_id = "motor_-FL";
        msg.is_motor_on = true;
        // 5. 发布消息
        publisher_->publish(msg);
    }
    rclcpp::Publisher<my_robot_interfaces::msg::HardwareStatus>::SharedPtr publisher_;
    rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char **argv){
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<HardwarePublisherNode>());
    rclcpp::shutdown();
    return 0;
}
```
3. 配置 hardware_publisher/CMakeLists.txt\
添加可执行文件并链接
```CMake
add_executable(publisher_node src/publisher_node.cpp)
ament_target_dependencies(publisher_node rclcpp my_robot_interfaces)

install(TARGETS
  publisher_node
  DESTINATION lib/${PROJECT_NAME}
)
```
### 创建订阅者
1. 在 src 目录下创建第三个包（也可以把订阅者放在 hardware_publisher 包里，作为另一个节点）
```Bash
cd ~/ros2_ws/src
ros2 pkg create hardware_subscriber --build-type ament_cmake --dependencies rclcpp my_robot_interfaces
```
2. 编辑 hardware_subscriber/src/subscriber_node.cpp
```C++
#include "rclcpp/rclcpp.hpp"
// 1. 同样包含自动生成的 .hpp 头文件
#include "my_robot_interfaces/msg/hardware_status.hpp" 

using std::placeholders::_1;

class HardwareSubscriberNode : public rclcpp::Node{
public:
    HardwareSubscriberNode() : Node("hardware_subscriber"){
        // 2. 创建订阅者，使用相同的 Message 类型
        // 回调函数的参数类型是 const Message::SharedPtr
        subscription_ = this->create_subscription<my_robot_interfaces::msg::HardwareStatus>(
        "hardware_status", 10, 
        std::bind(&HardwareSubscriberNode::topic_callback, this, _1));
        RCLCPP_INFO(this->get_logger(), "Hardware Subscriber 节点已启动.");
    }
private:
    // 3. 订阅回调函数
    void topic_callback(const my_robot_interfaces::msg::HardwareStatus::SharedPtr msg)const{
        // 4. 访问 Message 的字段
        RCLCPP_INFO(this->get_logger(), "收到状态：ID='%s', Temp=%ldC, MotorOn=%s",
        msg->hardware_id.c_str(),
        msg->temperature_celsius,
        msg->is_motor_on ? "true" : "false");
    }
    rclcpp::Subscription<my_robot_interfaces::msg::HardwareStatus>::SharedPtr subscription_;
};

int main(int argc, char **argv){
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<HardwareSubscriberNode>());
    rclcpp::shutdown();
    return 0;
}
```
3. 配置 hardware_subscriber/CMakeLists.txt，步骤与 Publisher 相同
### 构建和运行
1. 回到工作空间根目录 ~/ros2_ws，colcon 会自动发现依赖关系。
2. 它会先构建 my_robot_interfaces (生成 C++ 代码)，然后才构建 hardware_publisher 和 hardware_subscriber
```Bash
cd ~/ros2_ws
colcon build --packages-up-to hardware_publisher hardware_subscriber
```
3. 运行\
终端1 (激活环境并运行 Publisher)
```Bash
source install/setup.bash
ros2 run hardware_publisher publisher_node
```
终端2 (激活环境并运行 Subscriber)
```Bash
source install/setup.bash
ros2 run hardware_subscriber subscriber_node
```