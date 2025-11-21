## 概述
1. TrajOpt (Trajectory Optimization) 是一种基于序列凸优化的运动规划算法，广泛应用于机械臂和移动机器人的路径规划中
2. 相比于基于采样的算法（如 RRT*），TrajOpt 能够直接生成光滑且无碰撞的轨迹；相比于 CHOMP（基于梯度下降），TrajOpt 处理硬约束的能力更强
## 核心原理
核心思想是将运动规划问题建模为一个受约束的非线性优化问题，并通过序列二次规划 (SQP) 的方法进行求解
### 数学建模
将机器人的轨迹表示为一系列离散的时间步
$$X = [x_0, x_1, \dots, x_T]$$
$x_t \in \mathbb{R}^n$ 是机器人在 $t$ 时刻的关节构型
### 优化问题
TrajOpt 的目标是求解以下优化问题
$$\begin{aligned}
\min_{x} \quad & f(x) & \text{(目标函数：平滑度)} \\
\text{s.t.} \quad & g(x) \leq 0 & \text{(不等式约束：避障)} \\
& h(x) = 0 & \text{(等式约束：起始点、末端点、关节极限)}
\end{aligned}$$
### 目标函数
通常选择轨迹的最小速度或最小加速度作为优化目标，以保证轨迹平滑
$$f(x) = \sum_{t=0}^{T-1} || x_{t+1} - x_t ||^2 \quad \text{(最短路径/最小速度)}$$
### 避障约束
1. 最关键的创新在于如何处理避障，不使用二值的碰撞检测（碰/没碰），而是使用带符号距离场 (SDF)
2. 对于每个连杆上的检测点 $p$，其到最近障碍物的距离为 $sd(p)$，希望 $sd(p) > d_{safe}$
3. 为了将非凸的避障约束转化为凸约束，TrajOpt 在当前的轨迹点 $x_i$ 处进行线性化
$$sd(p(x)) \approx sd(p(x_i)) + \nabla sd(p(x_i)) \cdot J(x_i) \cdot (x - x_i)$$
$\nabla sd$: 距离场梯度（指示推开障碍物的方向）\
$J(x_i)$: 机器人的运动学雅可比矩阵

4. 这使得复杂的几何避障变成了一个线性的不等式约束 $A x \leq b$，可以很容易地嵌入到凸优化求解器中
### 连续碰撞检测
1. 不仅检查离散点 $x_t$，还检查 $x_t$ 到 $x_{t+1}$ 之间的扫掠体积
2. 通过将连杆建模为胶囊体并计算胶囊体移动过程中的凸包来实现，防止了穿墙现象
## 算法步骤
采用 SQP (Sequential Quadratic Programming) 框架，引入了 Trust Region (信赖域) 的概念
### 初始化
生成一条简单的初值轨迹（例如起点到终点的直线插值），通常是有碰撞的
### 主循环
1. 计算碰撞信息: 对当前轨迹的每个点，计算与环境的最近距离（SDF）和法向量
2. 凸近似：将非线性的避障约束在当前点线性化，将目标函数近似为二次型
3. 构建 QP 问题
$$\min_{\Delta x} \frac{1}{2} \Delta x^T H \Delta x + g^T \Delta x + \text{惩罚项}$$
$$\text{s.t. } A \Delta x \leq b$$
4. 求解 QP: 使用求解器（如 Gurobi, OSQP, BPMPD）算出步长 $\Delta x$
5. 信赖域检查\
（1）检查新的轨迹 $x_{new} = x_{old} + \Delta x$ 是否真的降低了 Cost\
（2）如果线性化误差太大，则缩小信赖域，拒绝更新或减小步长\
（3）如果改进显著，则接受更新并扩大信赖域
### 终止条件
当 $\Delta x$ 足够小或约束满足度达到阈值时停止
## 代码示例
在 ROS 2 工业级应用中，TrajOpt 通常集成在 Tesseract 规划框架中。下面的代码模拟了 TrajOpt 的核心逻辑：使用梯度信息将轨迹“推”出障碍物，同时保持平滑
### trajopt_demo.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <Eigen/Dense>
#include <vector>
#include <iostream>
#include <cmath>
using namespace Eigen;

class TrajOptDemo : public rclcpp::Node {
public:
    TrajOptDemo() : Node("trajopt_simple_solver") {
        // 1. 定义问题
        // 起点 (0,0), 终点 (10, 10)
        // 障碍物: 圆形, 圆心(5, 5), 半径 2.0
        start_pos_ << 0.0, 0.0;
        goal_pos_ << 10.0, 10.0;
        obstacle_pos_ << 5.0, 5.0;
        obstacle_radius_ = 2.5; // 稍微留点余量
        // 初始化轨迹 (直线插值)
        int num_waypoints = 20;
        trajectory_.resize(num_waypoints);
        for (int i = 0; i < num_waypoints; ++i) {
            double t = (double)i / (num_waypoints - 1);
            trajectory_[i] = start_pos_ + t * (goal_pos_ - start_pos_);
        }
        RCLCPP_INFO(this->get_logger(), "开始优化... 初始Cost: %f", calculateTotalCost());        
        // 2. 执行优化循环 (模拟 SQP 的迭代过程)
        optimize();
    }
private:
    std::vector<Vector2d> trajectory_;
    Vector2d start_pos_;
    Vector2d goal_pos_;
    Vector2d obstacle_pos_;
    double obstacle_radius_;
    // 权重参数
    const double w_smoothness_ = 1.0;  // 平滑度权重
    const double w_collision_ = 10.0;  // 碰撞惩罚权重
    const double learning_rate_ = 0.05; // 模拟步长 (Trust Region)
    void optimize() {
        for (int iter = 0; iter < 50; ++iter) { // 迭代50次
            std::vector<Vector2d> gradients(trajectory_.size(), Vector2d::Zero());
            // 计算梯度 (原理核心)
            for (size_t i = 1; i < trajectory_.size() - 1; ++i) {
                // A. 平滑度梯度 (Smoothness Gradient)
                // 目标是最小化 ||x_i - x_{i-1}||^2 + ||x_{i+1} - x_i||^2
                // 导数近似为: 2*x_i - x_{i-1} - x_{i+1} (类似于拉普拉斯算子)
                Vector2d grad_smooth = 2 * (2 * trajectory_[i] - trajectory_[i-1] - trajectory_[i+1]);
                // B. 碰撞梯度 (Collision Gradient - 线性化约束)
                // TrajOpt 使用 Signed Distance Field (SDF)
                Vector2d diff = trajectory_[i] - obstacle_pos_;
                double dist = diff.norm();
                Vector2d grad_coll = Vector2d::Zero();
                // 如果在障碍物影响范围内 (dist < radius)
                // 这相当于 Hinge Loss 约束
                if (dist < obstacle_radius_) {
                    // 梯度方向指向圆外 (归一化向量)
                    Vector2d normal = diff.normalized();
                    // 惩罚力度与侵入深度成正比
                    // 在 TrajOpt 中，这通常通过求解 QP 处理，这里用梯度下降模拟
                    grad_coll = -1.0 * normal * (obstacle_radius_ - dist); 
                }
                // 总梯度
                gradients[i] = w_smoothness_ * grad_smooth + w_collision_ * grad_coll;
            }
            // C. 更新轨迹 (Update Step)
            double max_update = 0.0;
            for (size_t i = 1; i < trajectory_.size() - 1; ++i) {
                trajectory_[i] -= learning_rate_ * gradients[i];
                max_update = std::max(max_update, gradients[i].norm());
            }
            if (iter % 10 == 0) {
                RCLCPP_INFO(this->get_logger(), "Iter %d: Max Gradient: %f", iter, max_update);
                printTrajectorySample();
            }
        }        
        RCLCPP_INFO(this->get_logger(), "优化完成。");
        printTrajectorySample();
    }
    double calculateTotalCost() {
        // 简单的 Cost 计算用于调试
        double cost = 0;
        for(size_t i=0; i<trajectory_.size()-1; ++i) {
            cost += (trajectory_[i+1] - trajectory_[i]).squaredNorm();
        }
        return cost;
    }
    void printTrajectorySample() {
        // 打印中间点位置，看是否绕过了 (5,5)
        int mid_idx = trajectory_.size() / 2;
        RCLCPP_INFO(this->get_logger(), "Mid Point: [%.2f, %.2f] (Obstacle at [5.0, 5.0])", 
            trajectory_[mid_idx].x(), trajectory_[mid_idx].y());
    }
};
int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<TrajOptDemo>());
    rclcpp::shutdown();
    return 0;
}
```
### package.xml
```XML
<depend>rclcpp</depend>
<depend>eigen3_cmake_module</depend>
<depend>Eigen3</depend>
```
### CMakeLists.txt
```CMake
find_package(Eigen3 REQUIRED)
find_package(rclcpp REQUIRED)

add_executable(trajopt_demo src/trajopt_demo.cpp)
ament_target_dependencies(trajopt_demo rclcpp Eigen3)
```