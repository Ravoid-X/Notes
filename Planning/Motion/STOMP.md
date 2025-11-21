## 概述
1. STOMP（Stochastic Trajectory Optimization for Motion Planning，随机轨迹优化运动规划）特别适用于高维空间（如机械臂）和复杂环境中的路径规划
2. 与基于梯度的算法（如 CHOMP）不同，STOMP 不需要对障碍物距离场进行微分，这使得它可以处理任意形式的代价函数
## 原理
### 核心思想
通过生成大量带有随机噪声的轨迹，利用这些轨迹的代价值作为权重，将当前的均值轨迹拉向低代价（无碰撞、更平滑）的区域
### 特性
1. 无梯度：很多时候，环境的代价地图（Costmap）是离散的或者难以求导的。STOMP 不计算梯度，而是通过概率采样来寻找下降方向
2. 处理复杂约束：可以在代价函数中加入任意约束（如关节力矩限制、末端姿态约束），只要能算出一个数值代价即可
3. 基于概率：它利用变分推断的思想，将路径规划问题转化为概率最大化问题
## 数学模型
STOMP 假设轨迹由 $N$ 个离散的时间点组成
### 轨迹表示
设轨迹为 $\Theta$，对于机械臂，$\Theta$ 是一个 $N \times D$ 的矩阵（$N$ 为时间步，$D$ 为关节自由度）
### 代价函数
1. 总代价 $S(\Theta)$ 由两部分组成
$$S(\Theta) = S_{smooth}(\Theta) + S_{state}(\Theta)$$
2. 平滑代价: 衡量轨迹的加速度或加加速度，通常为二次型
$$S_{smooth}(\Theta) = \frac{1}{2} \Theta^T R \Theta$$
$R$ 是一个带状矩阵（基于有限差分），用于计算加速度的平方和

3. 状态代价: 主要是障碍物代价
$$S_{state}(\Theta) = \sum_{t=0}^{N} q(\theta_t)$$
$q(\theta_t)$ 是在时刻 $t$ 机器人状态 $\theta_t$ 碰到障碍物的惩罚值
### 概率加权与更新
1. 在当前轨迹 $\Theta_{mean}$ 的基础上加上高斯噪声 $\epsilon_k$ 生成 $K$ 条噪声轨迹。
2. 每条轨迹的概率 $P(\Theta_k)$ 与其状态代价成反比
$$P(\Theta_k) \propto \exp \left( -\frac{1}{\lambda} S_{state}(\Theta_k) \right)$$
$\lambda$ 决定了算法对高代价轨迹的敏感程度

3. 更新规则：新的均值轨迹不是简单的加权平均，而是根据噪声的加权和来更新
$$\delta \theta = \sum_{k=1}^{K} w_k \epsilon_k$$
$w_k$ 是归一化后的概率权重
## 算法步骤
### 初始化
1. 生成一条初始轨迹（通常是起点到终点的线性插值）
2. 预计算平滑矩阵 $R$ 及其逆矩阵 $R^{-1}$（也就是协方差矩阵 $\Sigma$），用于生成平滑的随机噪声
### 迭代优化循环
1. 生成噪声: 基于当前的均值轨迹，生成 $K$ 条带噪声的轨迹。噪声不是纯随机的，而是经过平滑处理以保证生成的轨迹物理可行
$$\Theta_k = \Theta_{mean} + \epsilon_k, \quad \epsilon_k \sim \mathcal{N}(0, R^{-1})$$
2. 计算代价: 对每一条噪声轨迹，计算其 $S_{state}$（障碍物代价）
3. 计算权重: 使用 Softmax 形式计算每条轨迹的权重。代价越低，权重越大
4. 更新轨迹: 将加权后的噪声叠加回均值轨迹
5. 平滑处理: 对更新后的轨迹再应用一次平滑滤波器（可选，但在实践中常用）
### 收敛判定
达到最大迭代次数，或轨迹变化量小于阈值
## 代码示例
模拟了一个 2D 平面上的点机器人，需要从起点移动到终点，中间有一个圆形障碍物
### stomp_planner_node.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <visualization_msgs/msg/marker_array.hpp>
#include <geometry_msgs/msg/point.hpp>
#include <Eigen/Dense>
#include <vector>
#include <cmath>
#include <random>
#include <algorithm>

// 使用 Eigen 简化矩阵运算
using MatrixXd = Eigen::MatrixXd;
using VectorXd = Eigen::VectorXd;

class StompPlanner : public rclcpp::Node {
public:
    StompPlanner() : Node("stomp_planner_node") {
        // ROS 发布者，用于在 RViz 中显示轨迹
        viz_pub_ = this->create_publisher<visualization_msgs::msg::MarkerArray>("stomp_trajectories", 10);        
        // 初始化参数
        num_timesteps_ = 50;    // 轨迹点数量
        num_rollouts_ = 20;     // 每次迭代生成的噪声轨迹数
        num_iterations_ = 50;   // 迭代次数
        dt_ = 1.0;        
        // 起点 (0, 0) 终点 (10, 10)
        start_pos_ = VectorXd::Zero(2);
        goal_pos_ = VectorXd::Constant(2, 10.0);        
        // 障碍物：圆形，位置 (5, 5)，半径 2.0
        obstacle_pos_ = VectorXd::Constant(2, 5.0);
        obstacle_radius_ = 2.0;
        // 初始化轨迹（线性插值）
        current_trajectory_ = initialize_trajectory(start_pos_, goal_pos_, num_timesteps_);
        // 预计算平滑矩阵 (这里简化处理，仅使用高斯平滑核作为协方差)
        // 在完整 STOMP 中，这里使用的是有限差分矩阵 A 的逆 (A^T A)^-1
        timer_ = this->create_wall_timer(std::chrono::milliseconds(100), std::bind(&StompPlanner::run_stomp_step, this));
        RCLCPP_INFO(this->get_logger(), "STOMP Planner Initialized.");
    }
private:
    // 参数
    int num_timesteps_;
    int num_rollouts_;
    int num_iterations_;
    int current_iteration_ = 0;
    double dt_;
    VectorXd start_pos_, goal_pos_, obstacle_pos_;
    double obstacle_radius_;
    // 轨迹数据: N x 2 矩阵 (N个点，2维坐标)
    MatrixXd current_trajectory_;     
    rclcpp::Publisher<visualization_msgs::msg::MarkerArray>::SharedPtr viz_pub_;
    rclcpp::TimerBase::SharedPtr timer_;
    // 初始化轨迹为直线
    MatrixXd initialize_trajectory(const VectorXd& start, const VectorXd& goal, int steps) {
        MatrixXd traj(steps, 2);
        for (int i = 0; i < steps; ++i) {
            double alpha = static_cast<double>(i) / (steps - 1);
            traj.row(i) = (1.0 - alpha) * start + alpha * goal;
        }
        return traj;
    }
    // 生成平滑噪声
    // 真实的 STOMP 使用协方差矩阵采样，这里为了代码简洁，使用简单的低通滤波噪声
    MatrixXd generate_smooth_noise(int steps, int dim) {
        MatrixXd noise = MatrixXd::Zero(steps, dim);        
        // 随机数生成器
        static std::random_device rd;
        static std::mt19937 gen(rd());
        std::normal_distribution<> d(0, 1);
        // 1. 生成原始白噪声
        for(int i=1; i<steps-1; ++i) { // 起点和终点不加噪声
            for(int j=0; j<dim; ++j) {
                noise(i, j) = d(gen) * 0.5; // 幅度
            }
        }
        // 2. 简单平滑 (Moving Average) 模拟协方差矩阵的效果
        MatrixXd smoothed = noise;
        for (int k = 0; k < 5; ++k) { // 重复几次以获得更平滑的效果
            for (int i = 1; i < steps - 1; ++i) {
                smoothed.row(i) = 0.25 * smoothed.row(i-1) + 0.5 * smoothed.row(i) + 0.25 * smoothed.row(i+1);
            }
        }
        return smoothed;
    }
    // 计算单个状态的代价 (障碍物距离场)
    double calculate_state_cost(const VectorXd& pos) {
        double dist = (pos - obstacle_pos_).norm();
        if (dist < obstacle_radius_ + 0.5) { // 0.5 是安全边际
            return 100.0 * (obstacle_radius_ + 0.5 - dist); // 线性惩罚
        }
        return 0.0;
    }
    void run_stomp_step() {
        if (current_iteration_ >= num_iterations_) return;
        std::vector<MatrixXd> noisy_trajectories(num_rollouts_);
        std::vector<double> trajectory_costs(num_rollouts_);        
        // 1. 生成噪声轨迹并计算代价
        // 注意：STOMP 这里的代价通常是局部代价，这里简化为整条轨迹的总状态代价
        for (int k = 0; k < num_rollouts_; ++k) {
            MatrixXd noise = generate_smooth_noise(num_timesteps_, 2);
            noisy_trajectories[k] = current_trajectory_ + noise;
            double cost_sum = 0.0;
            for (int t = 0; t < num_timesteps_; ++t) {
                cost_sum += calculate_state_cost(noisy_trajectories[k].row(t));
            }
            trajectory_costs[k] = cost_sum;
        }
        // 2. 计算权重 (Softmax)
        // 寻找最小和最大代价以进行数值稳定的 Softmax
        double min_cost = *std::min_element(trajectory_costs.begin(), trajectory_costs.end());
        double max_cost = *std::max_element(trajectory_costs.begin(), trajectory_costs.end());        
        std::vector<double> weights(num_rollouts_);
        double weight_sum = 0.0;
        double h = 10.0; // 敏感度参数 (Lambda)
        for (int k = 0; k < num_rollouts_; ++k) {
            // 指数加权: exp(-(S - S_min) / h)
            weights[k] = std::exp(-(trajectory_costs[k] - min_cost) / h);
            weight_sum += weights[k];
        }
        // 3. 更新轨迹
        // delta_theta = sum(weight_k * noise_k) / sum(weights)
        MatrixXd delta_update = MatrixXd::Zero(num_timesteps_, 2);
        for (int k = 0; k < num_rollouts_; ++k) {
            weights[k] /= weight_sum; // 归一化
            MatrixXd noise = noisy_trajectories[k] - current_trajectory_;
            delta_update += weights[k] * noise;
        }
        current_trajectory_ += delta_update;
        // 4. 可视化
        publish_visualization(noisy_trajectories);        
        current_iteration_++;
        if (current_iteration_ == num_iterations_) {
            RCLCPP_INFO(this->get_logger(), "Optimization Finished.");
        }
    }
    void publish_visualization(const std::vector<MatrixXd>& rollouts) {
        visualization_msgs::msg::MarkerArray markers;
        int id = 0;
        // 1. 绘制障碍物 (红色球体)
        visualization_msgs::msg::Marker obs_marker;
        obs_marker.header.frame_id = "map";
        obs_marker.header.stamp = this->now();
        obs_marker.ns = "obstacle";
        obs_marker.id = id++;
        obs_marker.type = visualization_msgs::msg::Marker::SPHERE;
        obs_marker.action = visualization_msgs::msg::Marker::ADD;
        obs_marker.pose.position.x = obstacle_pos_(0);
        obs_marker.pose.position.y = obstacle_pos_(1);
        obs_marker.scale.x = obstacle_radius_ * 2;
        obs_marker.scale.y = obstacle_radius_ * 2;
        obs_marker.scale.z = 0.1;
        obs_marker.color.a = 0.5;
        obs_marker.color.r = 1.0;
        markers.markers.push_back(obs_marker);
        // 2. 绘制噪声轨迹 (细绿色线)
        for (const auto& traj : rollouts) {
            visualization_msgs::msg::Marker line;
            line.header.frame_id = "map";
            line.header.stamp = this->now();
            line.ns = "rollouts";
            line.id = id++;
            line.type = visualization_msgs::msg::Marker::LINE_STRIP;
            line.scale.x = 0.02; 
            line.color.a = 0.1;
            line.color.g = 1.0;            
            for (int i = 0; i < traj.rows(); ++i) {
                geometry_msgs::msg::Point p;
                p.x = traj(i, 0);
                p.y = traj(i, 1);
                line.points.push_back(p);
            }
            markers.markers.push_back(line);
        }
        // 3. 绘制当前最优轨迹 (粗蓝色线)
        visualization_msgs::msg::Marker best_line;
        best_line.header.frame_id = "map";
        best_line.header.stamp = this->now();
        best_line.ns = "best_path";
        best_line.id = id++;
        best_line.type = visualization_msgs::msg::Marker::LINE_STRIP;
        best_line.scale.x = 0.1; 
        best_line.color.a = 1.0;
        best_line.color.b = 1.0;        
        for (int i = 0; i < current_trajectory_.rows(); ++i) {
            geometry_msgs::msg::Point p;
            p.x = current_trajectory_(i, 0);
            p.y = current_trajectory_(i, 1);
            best_line.points.push_back(p);
        }
        markers.markers.push_back(best_line);
        viz_pub_->publish(markers);
    }
};
int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<StompPlanner>());
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
find_package(Eigen3 REQUIRED)
find_package(rclcpp REQUIRED)
find_package(visualization_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)

add_executable(stomp_node src/stomp_planner_node.cpp)
ament_target_dependencies(stomp_node rclcpp visualization_msgs geometry_msgs)
target_include_directories(stomp_node PUBLIC ${EIGEN3_INCLUDE_DIRS})

install(TARGETS stomp_node DESTINATION lib/${PROJECT_NAME})
```