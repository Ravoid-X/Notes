## 概述
1. DWA (Dynamic Window Approach) 是一种基于速度空间的局部路径规划算法
2. 能同时考虑到机器人的运动学约束和动力学约束，生成平滑且避障的轨迹
3. 核心思想是在速度空间（v, $\omega$）中采样，假设机器人在极短的时间间隔 $\Delta t$ 内做匀速运动，通过推演轨迹来评估好坏
## 核心约束
需要在速度空间 $(v, \omega)$ 中确定一个采样范围，这个范围由三个限制条件的交集决定
### 物理极限 $V_s$
机器人电机能提供的最大最小速度
$$V_s = \{ (v, \omega) | v_{min} \le v \le v_{max}, \omega_{min} \le \omega \le \omega_{max} \}$$
### 动力学约束 $V_d$
$V_d$：考虑到机器人的加速度限制，在下一个时间片 $\Delta t$ 内，机器人实际能达到的速度范围
$$V_d = \{ (v, \omega) | v_{curr} - a_{v,dec} \Delta t \le v \le v_{curr} + a_{v,acc} \Delta t, \dots \}$$
>"Dynamic Window" 名字的由来，因为这个窗口随着当前速度 $v_{curr}$ 动态变化
### 安全约束 $V_a$
如果以当前速度 $(v, \omega)$ 行驶，必须保证在遇到障碍物前能减速到 0
$$V_a = \{ (v, \omega) | v \le \sqrt{2 \cdot dist(v, \omega) \cdot a_{v,dec}}, \dots \}$$
### 采样空间 $V_r$
$$V_r = V_s \cap V_d \cap V_a$$
## 评价函数
### 函数
在 $V_r$ 中采样得到多组 $(v, \omega)$，分别推演其未来一段时间的轨迹，利用评价函数 $G(v, \omega)$ 打分
$$G(v, \omega) = \sigma (\alpha \cdot \text{heading}(v, \omega) + \beta \cdot \text{dist}(v, \omega) + \gamma \cdot \text{velocity}(v, \omega))$$
### Heading (方位角评价)
衡量轨迹末端朝向与目标点的对齐程度。越对齐，得分越高
### Dist (避障评价)
轨迹上离最近障碍物的距离。距离越远，得分越高。如果碰撞，得分为负无穷
### Velocity (速度评价)
鼓励机器人以更快的速度运行（在安全前提下）
## 算法步骤
1. 获取状态: 读取机器人当前位姿 $(x, y, \theta)$、当前速度 $(v_{curr}, \omega_{curr})$ 以及传感器数据（障碍物信息）
2. 计算动态窗口: 根据加速度限制和当前速度，计算出当前可行的速度范围 $[v_{min\_d}, v_{max\_d}]$ 和 $[\omega_{min\_d}, \omega_{max\_d}]$
3. 速度采样: 在动态窗口内离散化采样若干组速度 $(v_i, \omega_j)$
4. 轨迹推演: 利用运动学模型（如差速模型），推算每组速度在预测时间内的轨迹点。模型示例如下
$$x_{t+1} = x_t + v \cos(\theta_t) dt$$
$$y_{t+1} = y_t + v \sin(\theta_t) dt$$
$$\theta_{t+1} = \theta_t + \omega dt$$
5. 轨迹评分: 对每条推演轨迹计算 $G(v, \omega)$
6. 选择最优: 选取分数最高的那组速度 $(v^*, \omega^*)$
7. 发布指令: 将 $(v^*, \omega^*)$ 下发给底层控制器
## 代码示例 
模拟了障碍物和目标点，没有引入复杂的 Costmap2D 依赖
### dwa_planner_node.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <geometry_msgs/msg/twist.hpp>
#include <geometry_msgs/msg/pose_stamped.hpp>
#include <nav_msgs/msg/odometry.hpp>
#include <tf2/utils.h>
#include <vector>
#include <cmath>
#include <algorithm>
#include <limits>

// --- 配置参数结构体 ---
struct Config {
    double max_speed = 1.0;        // [m/s]
    double min_speed = 0.0;        // [m/s]
    double max_yaw_rate = 40.0 * M_PI / 180.0; // [rad/s]
    double max_accel = 0.2;        // [m/ss]
    double max_yaw_accel = 40.0 * M_PI / 180.0; // [rad/ss]    
    double v_resolution = 0.01;    // 速度采样分辨率
    double yaw_resolution = 0.1 * M_PI / 180.0; // 角速度采样分辨率    
    double dt = 0.1;               // 运动模型步长 [s]
    double predict_time = 3.0;     // 预测总时长 [s]    
    // 评价函数权重
    double to_goal_cost_gain = 0.15;
    double speed_cost_gain = 1.0;
    double obstacle_cost_gain = 1.0;
    double robot_radius = 0.5;     // [m]
};
// --- 机器人状态 ---
struct State {
    double x = 0.0;
    double y = 0.0;
    double yaw = 0.0;
    double v = 0.0;
    double w = 0.0;
};
// --- 动态窗口 ---
struct Window {
    double min_v;
    double max_v;
    double min_w;
    double max_w;
};

class DWAPlanner : public rclcpp::Node {
public:
    DWAPlanner() : Node("dwa_planner_node") {
        // 1. 初始化发布者和订阅者
        cmd_pub_ = this->create_publisher<geometry_msgs::msg::Twist>("cmd_vel", 10);
        odom_sub_ = this->create_subscription<nav_msgs::msg::Odometry>(
            "odom", 10, std::bind(&DWAPlanner::odom_callback, this, std::placeholders::_1));
        // 2. 模拟目标点 (在实际 Nav2 中由 Global Planner 提供)
        goal_.x = 10.0;
        goal_.y = 10.0;
        // 3. 模拟障碍物列表 (x, y)
        obstacles_ = {{5.0, 5.0}, {7.0, 6.0}, {3.0, 4.0}};
        // 4. 定时器循环控制 (10Hz)
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(100), std::bind(&DWAPlanner::control_loop, this));
        RCLCPP_INFO(this->get_logger(), "DWA Planner Started. Goal: (%.2f, %.2f)", goal_.x, goal_.y);
    }
private:
    Config cfg_;
    State state_;
    State goal_;
    std::vector<std::pair<double, double>> obstacles_;    
    rclcpp::Publisher<geometry_msgs::msg::Twist>::SharedPtr cmd_pub_;
    rclcpp::Subscription<nav_msgs::msg::Odometry>::SharedPtr odom_sub_;
    rclcpp::TimerBase::SharedPtr timer_;
    void odom_callback(const nav_msgs::msg::Odometry::SharedPtr msg) {
        state_.x = msg->pose.pose.position.x;
        state_.y = msg->pose.pose.position.y;        
        // 简单的四元数转欧拉角
        tf2::Quaternion q(
            msg->pose.pose.orientation.x,
            msg->pose.pose.orientation.y,
            msg->pose.pose.orientation.z,
            msg->pose.pose.orientation.w);
        state_.yaw = tf2::impl::getYaw(q);

        state_.v = msg->twist.twist.linear.x;
        state_.w = msg->twist.twist.angular.z;
    }
    // --- 核心：计算动态窗口 ---
    Window calc_dynamic_window() {
        Window window;
        // 1. 车辆物理极限
        double Vs_min_v = cfg_.min_speed;
        double Vs_max_v = cfg_.max_speed;
        double Vs_min_w = -cfg_.max_yaw_rate;
        double Vs_max_w = cfg_.max_yaw_rate;
        // 2. 动力学限制 (当前速度能加/减速到的范围)
        // 注意：这里使用的是控制周期的 dt (比如0.1s)
        double Vd_min_v = state_.v - cfg_.max_accel * cfg_.dt;
        double Vd_max_v = state_.v + cfg_.max_accel * cfg_.dt;
        double Vd_min_w = state_.w - cfg_.max_yaw_accel * cfg_.dt;
        double Vd_max_w = state_.w + cfg_.max_yaw_accel * cfg_.dt;
        // 3. 取交集
        window.min_v = std::max(Vs_min_v, Vd_min_v);
        window.max_v = std::min(Vs_max_v, Vd_max_v);
        window.min_w = std::max(Vs_min_w, Vd_min_w);
        window.max_w = std::min(Vs_max_w, Vd_max_w);

        return window;
    }
    // --- 核心：推演轨迹 ---
    std::vector<State> predict_trajectory(double v, double w) {
        std::vector<State> trajectory;
        State temp_state = state_;
        double time = 0;
        while (time <= cfg_.predict_time) {
            // 简单的运动学模型更新
            temp_state.yaw += w * cfg_.dt;
            temp_state.x += v * std::cos(temp_state.yaw) * cfg_.dt;
            temp_state.y += v * std::sin(temp_state.yaw) * cfg_.dt;
            temp_state.v = v;
            temp_state.w = w;            
            trajectory.push_back(temp_state);
            time += cfg_.dt;
        }
        return trajectory;
    }
    // --- 核心：计算轨迹代价 ---
    double calc_cost(const std::vector<State>& traj) {
        if (traj.empty()) return std::numeric_limits<double>::infinity();
        // 1. Heading Cost (目标朝向代价)
        State last_state = traj.back();
        double dx = goal_.x - last_state.x;
        double dy = goal_.y - last_state.y;
        double error_angle = std::atan2(dy, dx) - last_state.yaw;
        double cost_angle = std::abs(std::atan2(std::sin(error_angle), std::cos(error_angle)));
        // 2. Dist Cost (障碍物距离代价)
        double min_dist = std::numeric_limits<double>::max();
        for (const auto& p : traj) {
            for (const auto& obs : obstacles_) {
                double dist = std::hypot(p.x - obs.first, p.y - obs.second);
                if (dist <= cfg_.robot_radius) {
                    return std::numeric_limits<double>::infinity(); // 碰撞
                }
                if (dist < min_dist) min_dist = dist;
            }
        }
        // DWA 论文通常取倒数，或者最大化距离。这里为了统一 Cost 越小越好
        double cost_obs = 1.0 / (min_dist + 1e-6);
        // 3. Velocity Cost (速度代价 - 速度越大代价越小)
        double cost_vel = cfg_.max_speed - traj.back().v;
        // 总代价
        double total_cost = 
            cfg_.to_goal_cost_gain * cost_angle +
            cfg_.obstacle_cost_gain * cost_obs +
            cfg_.speed_cost_gain * cost_vel;

        return total_cost;
    }
    void control_loop() {
        // 判断是否到达目标
        double dist_to_goal = std::hypot(state_.x - goal_.x, state_.y - goal_.y);
        if (dist_to_goal < 0.5) {
            geometry_msgs::msg::Twist stop_cmd;
            cmd_pub_->publish(stop_cmd);
            RCLCPP_INFO_THROTTLE(this->get_logger(), *this->get_clock(), 2000, "Goal Reached!");
            return;
        }
        Window dw = calc_dynamic_window();
        double best_v = 0.0;
        double best_w = 0.0;
        double min_cost = std::numeric_limits<double>::infinity();
        // 采样循环
        for (double v = dw.min_v; v <= dw.max_v; v += cfg_.v_resolution) {
            for (double w = dw.min_w; w <= dw.max_w; w += cfg_.yaw_resolution) {                
                std::vector<State> traj = predict_trajectory(v, w);
                double cost = calc_cost(traj);
                if (cost < min_cost) {
                    min_cost = cost;
                    best_v = v;
                    best_w = w;
                }
            }
        }
        // 发布最优速度
        geometry_msgs::msg::Twist cmd;
        cmd.linear.x = best_v;
        cmd.angular.z = best_w;
        cmd_pub_->publish(cmd);        
        // 调试输出
        // RCLCPP_INFO(this->get_logger(), "Best v: %.2f, w: %.2f, Cost: %.2f", best_v, best_w, min_cost);
    }
};
int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<DWAPlanner>());
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.8)
project(dwa_planner_cpp)

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_geometry_msgs REQUIRED)

add_executable(dwa_planner_node src/dwa_planner_node.cpp)
ament_target_dependencies(dwa_planner_node
    rclcpp
    geometry_msgs
    nav_msgs
    tf2
    tf2_geometry_msgs
)
install(TARGETS dwa_planner_node
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```