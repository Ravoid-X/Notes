## 概述
1. 一套通信策略，允许精细地调整 ROS2 节点（通过 Topic、Service 等）之间交换数据的方式
2. 现代机器人系统是复杂的、分布式的，并且经常运行在不完美的网络环境中，不同类型的数据对通信有截然不同的要求，所以需要 Qos
## 原理
1. QoS 策略本质上是 DDS 提供的子集，当发布者和订阅者在同一个 Topic 上连接时，它们底层的 DDS 实体会交换各自的 QoS 配置，RMW 会检查它们的配置是否兼容
2. QoS 策略不兼容是 ROS2 中“为什么节点 A 发布了，节点 B 却收不到”的最常见原因之一
## 核心策略
### Reliability (可靠性)
定义了数据传递的保证级别
#### RMW_QOS_POLICY_RELIABILITY_RELIABLE (可靠)
1. 保证消息一定会被传递（只要网络允许）。如果数据包丢失，DDS层会自动重传
2. 适用：绝对不能丢失的数据，例如：控制命令、服务调用、参数更改
3. 代价：延迟可能更高（因为需要确认和重传），网络开销更大
#### RMW_QOS_POLICY_RELIABILITY_BEST_EFFORT (尽力而为)
1. 发布者即发即忘，尽力发送数据，但不保证接收方一定能收到。如果数据包丢失，它就永远丢失了
2. 适用：高频率、数据量大的传感器数据，例如：激光雷达、图像、IMU
3. 优势：延迟最低，网络效率最高
### Durability (持久性)
定义了发布者应如何处理历史消息，特别是对于后来才加入的订阅者
#### RMW_QOS_POLICY_DURABILITY_VOLATILE (易失性)
默认策略。订阅者只能收到在它加入网络之后发布者发布的消息，会错过所有历史消息
#### RMW_QOS_POLICY_DURABILITY_TRANSIENT_LOCAL (瞬态本地)
1. 这是 ROS1 中锁存行为的 ROS2 版本。发布者会在本地缓存它发布的最后 N 条消息（N 由 Depth 策略定义）
2. 当一个新的订阅者连接到这个Topic时，发布者会立即将这些缓存的消息发送给这个新来的订阅者
3. 适用：状态性质的数据，--*例如：地图 (/map)、配置参数、机器人当前状态
### History & Depth (历史与队列深度)
定义了在发送和接收端，消息队列（缓冲区）的行为
#### RMW_QOS_POLICY_HISTORY_KEEP_LAST (保留最后 N 个)
1. 最常用的策略，与一个 depth 值配合使用，如 History = KEEP_LAST, Depth = 10
2. 适用：绝大多数场景。depth=1 意味着只关心最新数据，depth=10 提供了一个小的缓冲
#### RMW_QOS_POLICY_HISTORY_KEEP_ALL (保留所有)
1. 保留所有历史消息，直到DDS的资源限制
2. 适用于极少数情况（如数据记录）
3. 极度危险，如果订阅者处理不及时，发布者的内存会被无限增长，最终导致系统崩溃
## 策略兼容性
最好保持发布者和订阅者的 QoS 策略一致，或者使用 ROS2 提供的预定义 QoS 配置集
### Reliability
1. Pub (Reliable) 可以连接 Sub (Reliable)
2. Pub (Best Effort) 可以连接 Sub (Best Effort)
3. Pub (Reliable) 可以连接 Sub (Best Effort) (订阅者“降级”了要求)
4. Pub (Best Effort) 不能连接 Sub (ReliABLE) (订阅者要求太高，发布者无法满足)
### Durability
1. Pub (Transient Local) 可以连接 Sub (Volatile) (订阅者只是不索要历史数据)
2. Pub (Volatile) 不能连接 Sub (Transient Local) (订阅者索要历史数据，但发布者没有)
## 预定义的 QoS 配置集
### 系统默认
rclcpp::SystemDefaultsQoS()
1. Reliability: RELIABLE
2. Durability: VOLATILE
3. History: KEEP_LAST, Depth: 10
### 传感器数据
rclcpp::SensorDataQoS()
1. Reliability: BEST_EFFORT (关键！用于高频数据)
2. Durability: VOLATILE
3. History: KEEP_LAST, Depth: 5
### 参数
rclcpp::ParametersQoS()
1. Reliability: RELIABLE
2. Durability: TRANSIENT_LOCAL (关键！新节点能获取到参数)
3. History: KEEP_LAST, Depth: 1000
### 服务
rclcpp::ServicesQoS()
1. Reliability: RELIABLE
2. Durability: VOLATILE
## 示例
```C++
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

/*
 * 在 C++ 中，QoS 对象通常作为 create_publisher/create_subscription 的
 * 最后一个参数传入。
 */
class MyNode : public rclcpp::Node{
public:
    MyNode() : Node("my_qos_node"){
        // 1. 使用预定义的 "SensorData" 配置
        auto sensor_qos = rclcpp::SensorDataQoS();
        lidar_sub_ = this->create_subscription<std_msgs::msg::String>(
        "lidar_scan", sensor_qos, /* ... callback ... */);
        // 2. 使用预定义的 "Parameters" 配置 (即 ROS1 的 Latching)
        auto latching_qos = rclcpp::ParametersQoS();
        map_pub_ = this->create_publisher<std_msgs::msg::String>("map", latching_qos);
        // 3. 手动创建自定义QoS
        // 创建一个队列深度为 5，可靠(Reliable)，持久(Transient Local)的QoS
        rclcpp::QoS custom_qos(rclcpp::KeepLast(5)); // 基于 KeepLast(5) 创建
        custom_qos.reliable();                       // 设置为 Reliable
        custom_qos.transient_local();                // 设置为 Transient Local
        custom_pub_ = this->create_publisher<std_msgs::msg::String>("custom_topic", custom_qos);
        // 4. (最常见) 简单地指定队列深度
        // 这将使用系统默认值（Reliable, Volatile），但只修改深度
        // rclcpp::QoS(10) 等同于 rclcpp::SystemDefaultsQoS().keep_last(10)
        default_pub_ = this->create_publisher<std_msgs::msg::String>("chatter", rclcpp::QoS(10));
    }
    // ...
};
```