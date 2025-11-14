## 概述
ROS 中最核心、最常用的通信模式，基于发布/订阅模型，允许系统中的不同部分（节点）以解耦和异步的方式交换数据
## 概念
### 话题
1. 一个命名的总线，节点可以通过它来发送和接收数据，本身不存储数据（有特例），只是一个路由机制。
2. 就像一个公告板，一个节点可以往上面“张贴”信息，而其他节点可以“订阅”这个公告板来查看信息
### 消息
1. 在 Topic 上传输的数据的结构，每个Topic都强绑定一种特定的消息类型，消息类型在 .msg 文件中定义
2. 例如，发布 GPS 位置可能使用 sensor_msgs/msg/NavSatFix 类型，发布简单调试文本可能使用 std_msgs/msg/String 类型
### 发布者
1. 一个发布者是节点内的一个对象，它连接到一个特定的 Topic，并向该 Topic 发送 特定类型的消息
2. 一个节点可以有多个发布者，发布到不同的 Topic
### 订阅者
1. 一个订阅者是节点内的一个对象，它连接到一个特定的 Topic，并接收该 Topic 上的消息
2. 当有消息发布到该 Topic 时，订阅者的回调函数会被自动触发
3. 一个节点可以有多个订阅者，订阅不同的 Topic
### 核心关系
Topic 是多对多 (M-to-N) 的
1. 一个 Topic 可以有多个发布者（例如，多个传感器都发布到 /diagnostics 话题）
2. 一个 Topic 可以有多个订阅者（例如，日志系统和控制系统都订阅 /robot_pose 话题）
## 原理
### 发布/订阅模型
Topic 的核心，关键在于“解耦”
1. 空间解耦: 发布者不知道订阅者的存在，订阅者也不知道发布者的存在。它们都只知道 Topic 的名称
2. 时间解耦: 发布者和订阅者不需要同时运行。发布者可以发布消息，即使没有任何订阅者；订阅者可以订阅，即使还没有发布者（只是收不到消息）
3. 异步: 发布者发送消息后不会等待订阅者确认，它即发即忘，然后继续执行自己的任务
### DDS & QoS
详见 DDS.md 和 QoS.md
## 示例
### 实现目标
创建两个节点：
1. minimal_publisher (发布者): 每秒发布一次 "Hello, world! [N]" 到 chatter 话题。
2. minimal_subscriber (订阅者): 订阅 chatter 话题，并打印它收到的所有消息
### minimal_publisher (发布者)
```C++
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"
#include <chrono>
#include <memory>

// 使用 chrono_literals，以便使用 500ms 这样的时间表示
using namespace std::chrono_literals;

class MinimalPublisher : public rclcpp::Node{
public:
    MinimalPublisher():Node("minimal_publisher"),count_(0){
        // 1. 创建发布者
        publisher_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);
        // 2. 创建一个定时器
        timer_ = this->create_wall_timer(500ms, 
            std::bind(&MinimalPublisher::timer_callback, this));
    }

private:
    void timer_callback() {
        // 3. 创建一个消息对象
        auto message = std_msgs::msg::String();
        // 4. 填充消息数据
        message.data = "Hello, world! " + std::to_string(count_++);
        // 5. 打印日志 (可选，但推荐)
        // 使用 RCLCPP_INFO 宏来打印日志，它会自动包含节点名和时间戳
        RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", message.data.c_str());
        // 6. 发布消息
        publisher_->publish(message);
    }
    rclcpp::TimerBase::SharedPtr timer_; // 定时器
    rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_; // 发布者
    size_t count_; // 消息计数器
};

int main(int argc, char * argv[]){
    // 1. 初始化 ROS2 C++ 客户端库
    rclcpp::init(argc, argv);
    // 2. 创建 MinimalPublisher 节点实例并运行
    // rclcpp::spin() 会“阻塞”在这里，使节点保持活动状态，
    rclcpp::spin(std::make_shared<MinimalPublisher>());
    // 3. 关闭 ROS2 客户端库
    rclcpp::shutdown();
    return 0;
}
```
### minimal_subscriber (订阅者)
```C++
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"
#include <memory>

// 使用 placeholders，用于 std::bind
using std::placeholders::_1;

class MinimalSubscriber : public rclcpp::Node{
public:
    MinimalSubscriber():Node("minimal_subscriber"){
        // 1. 创建订阅者
        // 将收到的消息 (用 _1 占位符表示) 作为参数传递给该方法
        subscription_ = this->create_subscription<std_msgs::msg::String>(
        "chatter", 10, std::bind(&MinimalSubscriber::topic_callback, this, _1));
    }
private:
    // 参数类型必须是 const MessageType::SharedPtr
    void topic_callback(const std_msgs::msg::String::SharedPtr msg)const{
        RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg->data.c_str());
    }
    rclcpp::Subscription<std_msgs::msg::String>::SharedPtr subscription_;
};

int main(int argc, char * argv[]){
    // 1. 初始化 ROS2 C++ 客户端库
    rclcpp::init(argc, argv);
    // 2. 创建 MinimalSubscriber 节点实例并运行
    // rclcpp::spin() 使节点保持活动状态，等待消息到达并触发回调
    rclcpp::spin(std::make_shared<MinimalSubscriber>());
    // 3. 关闭 ROS2 客户端库
    rclcpp::shutdown();
    return 0;
}
```