## 概述
1. CHOMP (Covariant Hamiltonian Optimization for Motion Planning) 是一种基于优化的运动规划算法
2. 从一条初始轨迹（可能存在碰撞）开始，通过梯度下降法，不断“拉扯”和“平滑”这条轨迹，直到它避开障碍物并保持平滑。
## 核心思想
### 轨迹即函数
1. 将机器人的轨迹表示为一组参数 $\xi$（例如一系列的时间点上的关节角）。目标是找到一个 $\xi$，使得代价函数 $U(\xi)$ 最小
2. 总代价函数通常由两部分组成
$$U(\xi) = F_{smooth}(\xi) + \lambda F_{obs}(\xi)$$
$F_{smooth}(\xi)$（平滑代价）： 惩罚轨迹的加速度或速度，确保轨迹动态可行且平滑\
$F_{obs}(\xi)$（障碍物代价）： 惩罚轨迹进入障碍物区域的行为
### Covariant
1. 标准的梯度下降法是 $\xi_{new} = \xi_{old} - \eta \nabla U$。
但在轨迹空间中，标准的梯度下降会导致轨迹变得极其 jagged（锯齿状/不平滑）
2. CHOMP 利用黎曼流形的概念，不再使用欧几里得梯度的方向，而是通过一个逆平滑算子（矩阵 $A^{-1}$） 来扭曲梯度方向
$$\xi_{new} = \xi_{old} - \frac{1}{\eta} A^{-1} \nabla U(\xi)$$
3. $A$ 是一个表示导数（如加速度）的带状矩阵，$A^{-1}$ 本质上起到了低通滤波器的作用
4. 意味着即使障碍物的梯度很尖锐（比如突然靠近一个尖角），更新量的作用也会被平滑地分布到整个轨迹上，从而自然地保持轨迹的平滑性
### Hamiltonian
1. 算法的推导过程使用了哈密顿蒙特卡洛（HMC）的一些思想，将优化问题构建为动力学系统的能量最小化问题。
2. 在实际的 CHOMP 实现中，主要体现在利用辛几何保持梯度的物理意义
### 有向距离场 (Signed Distance Field，SDF)
为了快速计算 $F_{obs}$ 及其梯度，CHOMP 依赖于有向距离场
1. SDF 告诉机器人当前位置距离最近障碍物有多远
2. 如果距离为负，说明在障碍物内部
3. SDF 的梯度 $\nabla d(x)$ 直接给出了远离障碍物的最快方向
## 算法步骤
### 初始化
1. 生成一条从起点 $q_{start}$ 到终点 $q_{goal}$ 的初始轨迹，通常是一条直线（线性插值），哪怕这条直线穿过了墙壁也没关系。
2. 预计算平滑矩阵 $A$ 及其逆矩阵 $A^{-1}$（这部分只与轨迹的时间步数有关，与环境无关）
### 构建距离场
在工作空间中建立体素网格，计算每个网格点到最近障碍物的距离及梯度
### 优化循环
1. 计算障碍物梯度：遍历轨迹上的每个点，利用正向运动学计算机器人身体各关键点在工作空间的位置 $x$
2. 查询 SDF：获取这些位置的距离值 $d(x)$ 和距离梯度 $\nabla d(x)$
3. 计算泛函梯度 $\nabla F_{obs}$：将工作空间的排斥力通过雅可比矩阵映射回关节空间
$$\nabla F_{obs} = \sum J^T \cdot v \cdot \|\nabla d(x)\|$$
> $v$ 是流速向量，用于确定推力大小
4. 投影更新：应用协变梯度更新规则
$$\xi_{t+1} = \xi_t - \frac{1}{\eta} A^{-1} (\nabla F_{smooth} + \lambda \nabla F_{obs})$$
>实际实现中，$\nabla F_{smooth} = A\xi$，所以公式常化简为对误差的修正
5. 边界约束： 确保起点和终点固定不变
### 收敛判定
当代价变化小于阈值或达到最大迭代次数时停止
## 代码示例
编写一个简化的 2D 平面轨迹规划器，不依赖复杂的 MoveIt，而是直接展示 CHOMP 的核心矩阵运算和梯度下降逻辑
### CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.8)
project(chomp_simple_example)

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(visualization_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(Eigen3 REQUIRED)

add_executable(simple_chomp_node src/simple_chomp_node.cpp)
ament_target_dependencies(simple_chomp_node rclcpp visualization_msgs geometry_msgs)
target_include_directories(simple_chomp_node PUBLIC ${EIGEN3_INCLUDE_DIRS})

install(TARGETS simple_chomp_node DESTINATION lib/${PROJECT_NAME})
ament_package()
```
### package.xml
```XML
<depend>rclcpp</depend>
<depend>visualization_msgs</depend>
<depend>geometry_msgs</depend>
<depend>libeigen-dev</depend>
```
### chomp_node.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <visualization_msgs/msg/marker.hpp>
#include <geometry_msgs/msg/point.hpp>
#include <Eigen/Dense>
#include <vector>
#include <cmath>
#include <chrono>
using namespace std::chrono_literals;

class SimpleChompNode : public rclcpp::Node {
public:
    SimpleChompNode() : Node("simple_chomp_node") {
        publisher_ = this->create_publisher<visualization_msgs::msg::Marker>("chomp_path", 10);
        obstacle_pub_ = this->create_publisher<visualization_msgs::msg::Marker>("obstacle", 10);
        // 初始化参数
        num_points_ = 50;       // 轨迹点数量
        dt_ = 1.0;              // 时间步长 (归一化处理)
        learning_rate_ = 0.05;  // 学习率 (eta)
        obstacle_weight_ = 10.0; // 障碍物权重 (lambda)        
        // 障碍物定义 (圆心 x, y, 半径 r)
        obs_x_ = 5.0;
        obs_y_ = 5.0;
        obs_r_ = 2.0;
        // 1. 初始化轨迹 (起点 0,0 -> 终点 10,10)
        initTrajectory(0.0, 0.0, 10.0, 10.0);
        // 2. 预计算平滑矩阵 A 及其逆 A_inv
        computeSmoothnessMatrix();
        // 定时器循环执行优化和可视化
        timer_ = this->create_wall_timer(50ms, std::bind(&SimpleChompNode::controlLoop, this));        
        RCLCPP_INFO(this->get_logger(), "CHOMP Node Started. Check Rviz topic /chomp_path");
    }

private:
    // 轨迹数据：N x 2 矩阵 (x, y)
    Eigen::MatrixXd trajectory_;    
    // CHOMP 矩阵
    Eigen::MatrixXd A_;      // 平滑代价矩阵
    Eigen::MatrixXd A_inv_;  // 协变矩阵 (逆平滑)    
    int num_points_;
    double dt_;
    double learning_rate_;
    double obstacle_weight_;
    double obs_x_, obs_y_, obs_r_;
    rclcpp::Publisher<visualization_msgs::msg::Marker>::SharedPtr publisher_;
    rclcpp::Publisher<visualization_msgs::msg::Marker>::SharedPtr obstacle_pub_;
    rclcpp::TimerBase::SharedPtr timer_;
    void initTrajectory(double start_x, double start_y, double end_x, double end_y) {
        trajectory_ = Eigen::MatrixXd::Zero(num_points_, 2);
        for (int i = 0; i < num_points_; ++i) {
            double t = (double)i / (num_points_ - 1);
            trajectory_(i, 0) = start_x + t * (end_x - start_x);
            trajectory_(i, 1) = start_y + t * (end_y - start_y);
        }
    }
    // 计算有限差分矩阵 A (代表加速度平方的积分)
    void computeSmoothnessMatrix() {
        A_ = Eigen::MatrixXd::Zero(num_points_, num_points_);        
        // 简单的二阶差分矩阵 (近似加速度)
        // [ 1 -2  1  0 ... ]
        // [ 0  1 -2  1 ... ]
        // 注意：CHOMP 中通常将首尾固定，这里简化处理内部点
        for (int i = 1; i < num_points_ - 1; ++i) {
            A_(i, i) = 2.0;
            A_(i, i - 1) = -1.0;
            A_(i, i + 1) = -1.0;
        }        
        // 边界条件处理：保证起点和终点不被拉动
        // 在标准 CHOMP 中，会将矩阵分块处理。这里为了简化，
        // 我们将首尾行的对角线设为非常大，使其梯度极小，从而固定端点。
        A_(0, 0) = 10000.0; 
        A_(num_points_-1, num_points_-1) = 10000.0;
        // 计算逆矩阵 (协变矩阵)
        // 在实际高维应用中应使用 Cholesky 分解求解线性方程，而不是直接求逆
        A_inv_ = A_.inverse();
    }
    // 计算障碍物代价的梯度
    Eigen::MatrixXd calculateObstacleGradient() {
        Eigen::MatrixXd grad = Eigen::MatrixXd::Zero(num_points_, 2);
        for (int i = 1; i < num_points_ - 1; ++i) {
            double x = trajectory_(i, 0);
            double y = trajectory_(i, 1);
            // 简单的 SDF 计算：距离圆心的距离
            double dist_sq = std::pow(x - obs_x_, 2) + std::pow(y - obs_y_, 2);
            double dist = std::sqrt(dist_sq);
            // 障碍物排斥场：如果距离小于 (半径 + 安全余量)，产生梯度
            double safe_margin = 1.0; 
            double cost_threshold = obs_r_ + safe_margin;
            if (dist < cost_threshold && dist > 0.001) {
                // 简单的代价函数 c(d) = 0.5 * (threshold - d)^2
                // 梯度方向：指向远离障碍物的方向
                // 链式法则: dC/dx = dC/dd * dd/dx                
                double d_cost_d_dist = -1.0 * (cost_threshold - dist); // 越近，负值越大                
                double d_dist_d_x = (x - obs_x_) / dist;
                double d_dist_d_y = (y - obs_y_) / dist;
                grad(i, 0) = d_cost_d_dist * d_dist_d_x;
                grad(i, 1) = d_cost_d_dist * d_dist_d_y;
            }
        }
        return grad;
    }
    void controlLoop() {
        // 1. 计算总梯度
        // 平滑梯度部分： A * xi (我们希望 xi 接近直线/平滑)
        Eigen::MatrixXd smooth_grad = A_ * trajectory_;        
        // 障碍物梯度部分
        Eigen::MatrixXd obs_grad = calculateObstacleGradient();
        // 总梯度 (Cost Gradient)
        Eigen::MatrixXd total_grad = smooth_grad + obstacle_weight_ * obs_grad;
        // 2. 更新步骤 (Covariant Update)
        // xi_new = xi - (1/eta) * A_inv * total_grad
        trajectory_ = trajectory_ - learning_rate_ * (A_inv_ * total_grad);
        // 3. 强制固定起点和终点 (防止数值误差漂移)
        trajectory_(0, 0) = 0.0; trajectory_(0, 1) = 0.0;
        trajectory_(num_points_-1, 0) = 10.0; trajectory_(num_points_-1, 1) = 10.0;
        // 4. 可视化
        publishVis();
    }
    void publishVis() {
        // 发布轨迹
        visualization_msgs::msg::Marker path_msg;
        path_msg.header.frame_id = "map";
        path_msg.header.stamp = this->now();
        path_msg.id = 0;
        path_msg.type = visualization_msgs::msg::Marker::LINE_STRIP;
        path_msg.action = visualization_msgs::msg::Marker::ADD;
        path_msg.scale.x = 0.1; 
        path_msg.color.a = 1.0; path_msg.color.g = 1.0;
        for (int i = 0; i < num_points_; ++i) {
            geometry_msgs::msg::Point p;
            p.x = trajectory_(i, 0);
            p.y = trajectory_(i, 1);
            p.z = 0.0;
            path_msg.points.push_back(p);
        }
        publisher_->publish(path_msg);
        // 发布障碍物
        visualization_msgs::msg::Marker obs_msg;
        obs_msg.header.frame_id = "map";
        obs_msg.header.stamp = this->now();
        obs_msg.id = 1;
        obs_msg.type = visualization_msgs::msg::Marker::SPHERE;
        obs_msg.action = visualization_msgs::msg::Marker::ADD;
        obs_msg.pose.position.x = obs_x_;
        obs_msg.pose.position.y = obs_y_;
        obs_msg.scale.x = obs_r_ * 2; obs_msg.scale.y = obs_r_ * 2; obs_msg.scale.z = 0.1;
        obs_msg.color.a = 0.5; obs_msg.color.r = 1.0; // Red
        obstacle_pub_->publish(obs_msg);
    }
};
int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<SimpleChompNode>());
    rclcpp::shutdown();
    return 0;
}
```