## 概述
机器人路径规划中最经典、直观的算法之一，虽然其思想简单，但在实际工程中（尤其是局部避障）仍然非常有效
## 核心思想
将机器人在环境中的运动视为在虚拟力场中的运动
### 目标点
产生引力场，像磁铁一样吸引机器人
### 障碍物
产生斥力场，像排斥磁铁一样推开机器人
### 合力
机器人受到引力和斥力的叠加作用，沿着合力方向下降（即势能降低的方向）运动，最终到达目标点
## 数学模型
定义机器人的位置为 $q = (x, y)$，目标点为 $q_{goal}$，障碍物位置为 $q_{obs}$
### 引力场
通常使用抛物线势场，因为其导数（力）是线性的，随着距离接近零，力也趋近于零，保证机器人能稳定停在目标点
#### 势能函数 $U_{att}(q)$
$$U_{att}(q) = \frac{1}{2} \xi \rho^2(q, q_{goal})$$
#### 引力 $F_{att}(q)$ (势能的负梯度)
$$F_{att}(q) = -\nabla U_{att}(q) = \xi (q_{goal} - q)$$
$\xi$: 引力增益系数 (正常数)\
$\rho(q, q_{goal})$: 机器人与目标点的欧几里得距离
### 斥力场
通常只在障碍物的一定范围内 ($d_0$) 作用。距离越近，斥力趋于无穷大；超过影响距离，斥力为 0
#### 势能函数 $U_{rep}(q)$
$$U_{rep}(q) = \begin{cases} \frac{1}{2} \eta (\frac{1}{\rho(q, q_{obs})} - \frac{1}{d_0})^2 & \text{if } \rho(q, q_{obs}) \leq d_0 \\ 0 & \text{if } \rho(q, q_{obs}) > d_0 \end{cases}$$
#### 斥力 $F_{rep}(q)$
$$F_{rep}(q) = -\nabla U_{rep}(q) = \begin{cases} \eta (\frac{1}{\rho(q, q_{obs})} - \frac{1}{d_0}) \frac{1}{\rho^2(q, q_{obs})} \nabla \rho(q, q_{obs}) & \text{if } \rho \leq d_0 \\ 0 & \text{if } \rho > d_0 \end{cases}$$
$\eta$: 斥力增益系数\
$d_0$: 障碍物影响距离阈值\
$\nabla \rho(q, q_{obs})$: 从障碍物指向机器人的单位向量
### 总势场与合力
$$U_{total}(q) = U_{att}(q) + \sum U_{rep}(q)$$
$$F_{total}(q) = F_{att}(q) + \sum F_{rep}(q)$$
## 算法步骤
1. 初始化: 设置机器人起点、目标点坐标、引力系数 $\xi$、斥力系数 $\eta$、障碍物影响距离 $d_0$、步长或速度上限
2. 感知环境: 获取当前机器人位置以及传感器探测到的障碍物位置
3. 计算引力: 根据当前位置和目标点计算 $F_{att}$
4. 计算斥力: 遍历所有探测到的障碍物，计算每个障碍物产生的 $F_{rep}$ 并求矢量和
5. 合力合成: 计算 $F_{total} = F_{att} + F_{rep}$
6. 运动控制:\
（1）将合力方向作为机器人的期望航向角\
（2）将合力大小（或由于运动学限制截断后的值）映射为线速度
7. 迭代: 更新机器人位置，重复步骤2-6，直到到达目标点（距离小于阈值）
## 问题
### 局部极小值
1. 当引力和斥力大小相等、方向相反且共线时（例如机器人、障碍物、目标点在一条直线上），合力为 0，机器人停止运动，但未到达目标
2. 解决方案\
（1）模拟退火/随机游走: 陷入局部极小值时添加随机扰动\
（2）全局规划结合: 使用 A* 或 RRT 生成全局路径，APF 仅作为局部控制器跟踪该路径
## 代码示例
硬编码了几个虚拟障碍物，并直接发布 cmd_vel
### 
```C++
#include <rclcpp/rclcpp.hpp>
#include <geometry_msgs/msg/twist.hpp>
#include <nav_msgs/msg/odometry.hpp>
#include <tf2/LinearMath/Quaternion.h>
#include <tf2/LinearMath/Matrix3x3.h>
#include <cmath>
#include <vector>

// 定义简单的 2D 向量结构体辅助计算
struct Vector2D {
    double x;
    double y;
    double norm() const { return std::sqrt(x * x + y * y); }
    Vector2D operator+(const Vector2D& other) const { return {x + other.x, y + other.y}; }
    Vector2D operator-(const Vector2D& other) const { return {x - other.x, y - other.y}; }
    Vector2D operator*(double scalar) const { return {x * scalar, y * scalar}; }
};
class APFNode : public rclcpp::Node {
public:
    APFNode() : Node("apf_planner_node") {
        // 1. 初始化参数
        this->declare_parameter("k_att", 2.0); // 引力系数
        this->declare_parameter("k_rep", 15.0); // 斥力系数 (通常需要调得比引力大)
        this->declare_parameter("dist_limit", 2.0); // 障碍物影响范围
        this->declare_parameter("goal_x", 5.0);
        this->declare_parameter("goal_y", 5.0);
        k_att_ = this->get_parameter("k_att").as_double();
        k_rep_ = this->get_parameter("k_rep").as_double();
        dist_limit_ = this->get_parameter("dist_limit").as_double();
        goal_ = {this->get_parameter("goal_x").as_double(), this->get_parameter("goal_y").as_double()};
        // 2. 设置虚拟障碍物 (实际应用中应来自 LaserScan 或 Costmap)
        obstacles_.push_back({2.5, 2.5}); // 位于路径中间的一个障碍物
        obstacles_.push_back({3.5, 4.0});
        // 3. ROS 通信
        pub_cmd_vel_ = this->create_publisher<geometry_msgs::msg::Twist>("cmd_vel", 10);
        sub_odom_ = this->create_subscription<nav_msgs::msg::Odometry>(
            "odom", 10, std::bind(&APFNode::odomCallback, this, std::placeholders::_1));
        // 定时器用于控制循环 (50ms = 20Hz)
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(50), std::bind(&APFNode::controlLoop, this));            
        RCLCPP_INFO(this->get_logger(), "APF Node Started. Goal: (%.2f, %.2f)", goal_.x, goal_.y);
    }
private:
    // 参数变量
    double k_att_, k_rep_, dist_limit_;
    Vector2D goal_;
    std::vector<Vector2D> obstacles_;    
    // 机器人状态
    Vector2D current_pos_ = {0.0, 0.0};
    double current_yaw_ = 0.0;
    bool odom_received_ = false;
    // ROS 句柄
    rclcpp::Publisher<geometry_msgs::msg::Twist>::SharedPtr pub_cmd_vel_;
    rclcpp::Subscription<nav_msgs::msg::Odometry>::SharedPtr sub_odom_;
    rclcpp::TimerBase::SharedPtr timer_;
    // 里程计回调：更新机器人位置
    void odomCallback(const nav_msgs::msg::Odometry::SharedPtr msg) {
        current_pos_.x = msg->pose.pose.position.x;
        current_pos_.y = msg->pose.pose.position.y;
        // 四元数转欧拉角 (Yaw)
        tf2::Quaternion q(
            msg->pose.pose.orientation.x,
            msg->pose.pose.orientation.y,
            msg->pose.pose.orientation.z,
            msg->pose.pose.orientation.w);
        tf2::Matrix3x3 m(q);
        double roll, pitch;
        m.getRPY(roll, pitch, current_yaw_);        
        odom_received_ = true;
    }
    // 计算引力
    Vector2D computeAttractiveForce() {
        // F_att = k_att * (q_goal - q_current)
        // 这是一个指向目标的向量
        Vector2D diff = goal_ - current_pos_;
        return diff * k_att_; 
    }
    // 计算斥力
    Vector2D computeRepulsiveForce() {
        Vector2D total_rep = {0.0, 0.0};
        for (const auto& obs : obstacles_) {
            Vector2D diff = current_pos_ - obs; // 向量：障碍物指向机器人
            double dist = diff.norm();
            if (dist <= dist_limit_) {
                // 简单的斥力模型公式实现
                // F_rep = k_rep * (1/dist - 1/dist_limit) * (1/dist^2) * unit_vector                
                double rep_mag = k_rep_ * (1.0 / dist - 1.0 / dist_limit_) * (1.0 / (dist * dist));                
                // 归一化方向向量
                Vector2D unit_vec = {diff.x / dist, diff.y / dist};                
                total_rep = total_rep + (unit_vec * rep_mag);
            }
        }
        return total_rep;
    }
    void controlLoop() {
        if (!odom_received_) {
            RCLCPP_WARN_THROTTLE(this->get_logger(), *this->get_clock(), 2000, "Waiting for Odom...");
            return;
        }
        // 1. 检查是否到达目标
        double dist_to_goal = (goal_ - current_pos_).norm();
        if (dist_to_goal < 0.2) {
            stopRobot();
            RCLCPP_INFO_THROTTLE(this->get_logger(), *this->get_clock(), 2000, "Goal Reached!");
            return;
        }
        // 2. 计算合力
        Vector2D f_att = computeAttractiveForce();
        Vector2D f_rep = computeRepulsiveForce();
        Vector2D f_total = f_att + f_rep;
        // 3. 运动学转换：将力转换为速度
        // 这里的逻辑很简单：合力的方向决定角速度，合力的大小决定线速度        
        double desired_yaw = std::atan2(f_total.y, f_total.x);
        double yaw_error = desired_yaw - current_yaw_;
        // 角度归一化到 [-PI, PI]
        while (yaw_error > M_PI) yaw_error -= 2 * M_PI;
        while (yaw_error < -M_PI) yaw_error += 2 * M_PI;
        geometry_msgs::msg::Twist cmd;        
        // 角速度控制 (P控制器)
        cmd.angular.z = 2.0 * yaw_error; 
        // 线速度控制
        // 稍微复杂的逻辑：如果角度偏差大，就减慢线速度（原地转向），否则全速前进
        // 限制最大力的大小，防止飞出去
        double force_mag = std::min(f_total.norm(), 1.0); 
        if (std::abs(yaw_error) > 0.5) {
            cmd.linear.x = 0.0; // 优先转向
        } else {
            cmd.linear.x = force_mag * 0.5; // 简单的增益
        }
        pub_cmd_vel_->publish(cmd);        
        // Debug 日志
        // RCLCPP_INFO(this->get_logger(), "Pos: (%.2f, %.2f) | F_att: %.2f | F_rep: %.2f | Cmd: v=%.2f w=%.2f", 
        //     current_pos_.x, current_pos_.y, f_att.norm(), f_rep.norm(), cmd.linear.x, cmd.angular.z);
    }
    void stopRobot() {
        geometry_msgs::msg::Twist cmd;
        cmd.linear.x = 0.0;
        cmd.angular.z = 0.0;
        pub_cmd_vel_->publish(cmd);
    }
};
int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    auto node = std::make_shared<APFNode>();
    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.8)
project(apf_planner)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_geometry_msgs REQUIRED)

add_executable(apf_node src/apf_node.cpp)
target_include_directories(apf_node PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)
target_compile_features(apf_node PUBLIC c_std_17)

ament_target_dependencies(apf_node
    rclcpp
    geometry_msgs
    nav_msgs
    tf2
    tf2_geometry_msgs
)

install(TARGETS apf_node
    DESTINATION lib/${PROJECT_NAME})

ament_package()
```
### package.xml
```XML
<depend>rclcpp</depend>
<depend>geometry_msgs</depend>
<depend>nav_msgs</depend>
<depend>tf2</depend>
<depend>tf2_geometry_msgs</depend>
```