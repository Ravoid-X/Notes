## 概述
1. TF2（Transform Library）是 ROS2 中用于 跟踪和管理多个坐标系之间关系的核心库
2. T一个分布式系统，允许多个 ROS2 节点发布它们所知道的坐标系关系，而其他任何节点都可以监听并查询这些关系
## 概念
### 坐标系
一个 frame 就是一个坐标系，用一个字符串 ID（frame_id）来标识，例如 "base_link" 或 "map"
### 变换
一个 Transform 定义了两个坐标系之间的数学关系，包含
1. 平移：一个三维向量 (x, y, z)，表示原点的偏移
2. 旋转：一个四元数 (x, y, z, w)，表示姿态的旋转
### geometry_msgs::msg::TransformStamped
TF2 在 ROS2 中实际传递的消息类型，不仅包含 Transform，还包含了元数据：
1. header.stamp：时间戳
2. header.frame_id：父坐标系，变换的“来源”坐标系（例如 "odom"）。
3. child_frame_id：子坐标系，变换的“目标”坐标系（例如 "base_link"）
>TF2 只存储 从 parent 到 child 的变换
### TF 树
1. 所有的 frame 通过 TransformStamped 消息连接起来，形成一个树状结构
2. 规则：一个 frame 只能有一个父 frame。如 map -> odom -> base_link -> laser_scanner
3. 当需要计算变换时，会自动沿着树查找路径，并将路径上的所有变换（包括反向变换）链式相乘，得到最终结果
### 时间和缓冲区
1. 不仅仅存储最新的变换，还会缓冲一段时间内的所有变换
2. 假设相机数据在 t=10.0 时刻到达，激光雷达数据在 t=10.1 时刻到达。如果想融合这些数据，需要知道它们各自对应时刻的变换
## 组件
### tf2_ros::Buffer (缓冲区)
在后台运行，存储所有接收到的变换。提供了核心的 lookupTransform() 和 transform() (用于变换数据) API
### tf2_ros::TransformListener (监听器)
1. 一个辅助类，唯一工作就是订阅 /tf 和 /tf_static 话题，并将接收到的所有 TransformStamped 消息喂给 Buffer
2. 常规用法：在节点中同时创建 Buffer 和 TransformListener，并将 Listener 绑定到 Buffer
### tf2_ros::TransformBroadcaster (发布器)
发布动态变换，会向 /tf 话题发布消息，适用于变换频繁变化的情况
### tf2_ros::StaticTransformBroadcaster (静态发布器)
1. 发布静态变换，如果一个变换永远不变，就必须使用这个发布器
2. 会向 /tf_static 话题发布一个锁存的消息，非常高效
## 示例
创建一个节点，模拟一个机器人底座 (base_link) 相对于里程计 (odom) 坐标系做圆周运动。这是一个动态变换
```C++
#include <rclcpp/rclcpp.hpp>
#include <geometry_msgs/msg/transform_stamped.hpp>
#include <tf2_ros/transform_broadcaster.h>
#include <tf2/LinearMath/Quaternion.h> // 用于 RPY 到四元数的转换
#include <string>
#include <cmath>

class DynamicTFBroadcaster : public rclcpp::Node{
public:
    DynamicTFBroadcaster() : Node("dynamic_tf2_broadcaster"){
        // 1. 初始化 TransformBroadcaster
        tf_broadcaster_ = std::make_unique<tf2_ros::TransformBroadcaster>(this);
        // 2. 创建一个定时器，周期性地调用 handle_timer 回调函数
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(100), // 10 Hz
            std::bind(&DynamicTFBroadcaster::handle_timer, this));
        RCLCPP_INFO(this->get_logger(), "Dynamic TF broadcaster started.");
    }

private:
    void handle_timer(){
        // 获取当前时间
        rclcpp::Time now = this->get_clock()->now();
        double t = now.seconds();
        // 模拟圆周运动
        double x = cos(t) * 1.0; // 半径 1.0 米
        double y = sin(t) * 1.0;
        double z = 0.0;
        double roll = 0.0;
        double pitch = 0.0;
        double yaw = t; // 机器人也在自转
        // 3. 创建 TransformStamped 消息
        geometry_msgs::msg::TransformStamped t_stamped;
        t_stamped.header.stamp = now;
        t_stamped.header.frame_id = "odom"; // 父坐标系
        t_stamped.child_frame_id = "base_link"; // 子坐标系
        t_stamped.transform.translation.x = x;
        t_stamped.transform.translation.y = y;
        t_stamped.transform.translation.z = z;
        // 使用 tf2::Quaternion 帮助类从欧拉角 (Roll, Pitch, Yaw) 转换
        tf2::Quaternion q;
        q.setRPY(roll, pitch, yaw); // 绕Z轴旋转
        t_stamped.transform.rotation.x = q.x();
        t_stamped.transform.rotation.y = q.y();
        t_stamped.transform.rotation.z = q.z();
        t_stamped.transform.rotation.w = q.w();
        // 4. 发送变换
        tf_broadcaster_->sendTransform(t_stamped);
        // (可选) 顺便发布一个静态变换 (例如 laser_scanner 相对于 base_link)
        // 注意：在实际项目中，静态变换应该由 StaticTransformBroadcaster 在构造函数中发布一次
        // if (static_broadcaster_ == nullptr) {
        //     static_broadcaster_ = std::make_unique<tf2_ros::StaticTransformBroadcaster>(this);
        //     geometry_msgs::msg::TransformStamped s_stamped;
        //     s_stamped.header.stamp = now;
        //     s_stamped.header.frame_id = "base_link";
        //     s_stamped.child_frame_id = "laser_scanner";
        //     s_stamped.transform.translation.x = 0.1; // 激光雷达在 base_link 前方 0.1米
        //     s_stamped.transform.rotation.w = 1.0; // 无旋转
        //     static_broadcaster_->sendTransform(s_stamped);
        // }
    }
    std::unique_ptr<tf2_ros::TransformBroadcaster> tf_broadcaster_;
    rclcpp::TimerBase::SharedPtr timer_;
    // std::unique_ptr<tf2_ros::StaticTransformBroadcaster> static_broadcaster_;
};

int main(int argc, char **argv){
    rclcpp::init(argc, argv);
    auto node = std::make_shared<DynamicTFBroadcaster>();
    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```