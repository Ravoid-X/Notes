## 概述
1. 差分进化算法（Differential Evolution, DE）是一种基于群体的随机优化算法，属于进化算法的一个分支
2. 由于其不需要通过梯度下降求解，因此特别适合处理非线性、不可微、多模态（即有多个局部最优解）的复杂优化问题
3. 在机器人领域（ROS2），DE 常用于机械臂逆运动学求解、路径规划的轨迹优化（如优化贝塞尔曲线控制点）以及 PID 参数自整定
## 原理
利用群体中个体之间的距离（差分向量）来指导搜索方向
### 核心直觉
1. DE 假设群体中不同个体之间的差异包含了有关搜索空间拓扑结构的有用信息
2. 在搜索初期，群体分散，差向量较大，通过变异产生的步长也大，利于全局探索
3. 在搜索后期，群体趋于收敛，差向量变小，步长自动缩小，利于局部开发（精细搜索）
### 关键参数
1. $NP$ (Population Size): 种群大小
2. $F$ (Scaling Factor): 缩放因子（通常在 [0, 2] 之间），控制差分向量放大的倍数，即步长
3. $CR$ (Crossover Rate): 交叉概率（[0, 1]），控制新生成的变异向量在多大程度上替换原向量
## 算法步骤
假设要寻找向量 $x$ 使目标函数 $f(x)$ 最小化
### 初始化
在搜索空间 $[min, max]$ 内随机生成 $NP$ 个个体，组成第一代种群
$$x_{i, 0} = x_{min} + rand(0, 1) \cdot (x_{max} - x_{min}), \quad i=1, \dots, NP$$
### 变异
对于第 $G$ 代的每一个目标个体 $\mathbf{x}_{i, G}$，生成一个变异向量 $\mathbf{v}_{i, G}$。最经典的策略是 DE/rand/1
$$v_{i, G+1} = x_{r1, G} + F \cdot (x_{r2, G} - x_{r3, G})$$
1. $r1, r2, r3$ 是从种群中随机选择的互不相同的索引，且都不等于 $i$
2. $F$ 是缩放因子，通常在 $[0, 2]$ 之间，控制差分向量放大的程度，决定了算法的探索能力
### 交叉
为了增加多样性，将变异向量 $v_i$ 与原目标个体 $x_i$ 进行离散交叉，生成试验向量 $u_i$。对于第 $j$ 个维度 ($j = 1...D$)
$$u_{i, j, G} = \begin{cases} v_{i, j, G} & \text{if } rand_j(0, 1) \le CR \text{ or } j = j_{rand} \\ x_{i, j, G} & \text{otherwise} \end{cases}$$
1. $CR \in [0, 1]$ 是交叉概率
2. $j_{rand}$ 是随机选的一个维度索引，保证试验向量至少有一个维度来自变异向量，避免试验向量完全复制原个体
### 选择
采用贪婪策略，比较试验个体 $u_i$ 和原个体 $x_i$ 的适应度值
$$x_{i, G+1} = \begin{cases} u_{i, G} & \text{if } f(u_{i, G}) \le f(x_{i, G}) \\ x_{i, G} & \text{otherwise} \end{cases}$$
>DE 只有在后代比父代更优秀（或相等）时才更新，这保证了种群的最优解不会退化
### 参数选择
1. NP (种群大小): 一般取 $5D \sim 10D$。太小易陷入局部最优，太大计算慢
2. F (缩放因子): 常用 0.5 到 0.9。$F$ 越大，跳出局部最优能力越强，但收敛越慢
3. CR (交叉概率): 常用 0.1 到 0.9。$CR$ 越大，收敛越快，但也更容易早熟收敛
## 代码示例
### 场景
1. 寻找一个目标点 $(x, y)$，使得函数 $f(x, y) = (x - 2.0)^2 + (y + 3.0)^2$ 最小。显然，最优解是 $(2.0, -3.0)$，最小值为 0。
2. 该代码并未阻塞主线程，而是利用 ROS2 的 Timer 进行迭代，每次 Timer 回调进化一代，并发布当前的全局最优解
### de_node.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <std_msgs/msg/float32_multi_array.hpp>
#include <vector>
#include <random>
#include <algorithm>
#include <iostream>
#include <iomanip>

struct Agent {
    std::vector<double> position;
    double cost;
};

class DifferentialEvolutionNode : public rclcpp::Node {
public:
    DifferentialEvolutionNode() : Node("de_optimizer_node") {
        // 1. 声明和获取参数
        this->declare_parameter("population_size", 20);
        this->declare_parameter("dim", 2);
        this->declare_parameter("max_generations", 100);
        this->declare_parameter("F", 0.5);  // 缩放因子
        this->declare_parameter("CR", 0.7); // 交叉概率
        this->declare_parameter("bounds_min", -10.0);
        this->declare_parameter("bounds_max", 10.0);
        pop_size_ = this->get_parameter("population_size").as_int();
        dim_ = this->get_parameter("dim").as_int();
        max_gens_ = this->get_parameter("max_generations").as_int();
        F_ = this->get_parameter("F").as_double();
        CR_ = this->get_parameter("CR").as_double();
        bound_min_ = this->get_parameter("bounds_min").as_double();
        bound_max_ = this->get_parameter("bounds_max").as_double();
        // 2. 初始化随机数生成器
        std::random_device rd;
        gen_ = std::mt19937(rd());
        // 3. 初始化种群
        initialize_population();
        // 4. 设置发布者和定时器 (用于迭代)
        best_sol_pub_ = this->create_publisher<std_msgs::msg::Float32MultiArray>("de/best_solution", 10);        
        // 0.1秒执行一代，模拟实时优化过程
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(100),
            std::bind(&DifferentialEvolutionNode::optimize_step, this));
        RCLCPP_INFO(this->get_logger(), "DE Optimizer started. Target: Minimize (x-2)^2 + (y+3)^2");
    }
private:
    // --- 目标函数 (Rosenbrock 或简单的 Sphere 函数) ---
    double cost_function(const std::vector<double>& pos) {
        double x = pos[0];
        double y = pos[1];
        return (x - 2.0) * (x - 2.0) + (y + 3.0) * (y + 3.0);
    }
    // --- 初始化种群 ---
    void initialize_population() {
        std::uniform_real_distribution<> dis(bound_min_, bound_max_);
        population_.resize(pop_size_);
        for (int i = 0; i < pop_size_; ++i) {
            population_[i].position.resize(dim_);
            for (int j = 0; j < dim_; ++j) {
                population_[i].position[j] = dis(gen_);
            }
            population_[i].cost = cost_function(population_[i].position);
        }        
        // 找到初始最优
        update_global_best();
    }
    // --- 找到当前种群中最优个体 ---
    void update_global_best() {
        for (const auto& agent : population_) {
            if (agent.cost < global_best_cost_) {
                global_best_cost_ = agent.cost;
                global_best_pos_ = agent.position;
            }
        }
    }
    // --- 核心进化迭代步 (Timer Callback) ---
    void optimize_step() {
        if (current_gen_ >= max_gens_) {
            timer_->cancel();
            RCLCPP_INFO(this->get_logger(), "Optimization Finished!");
            RCLCPP_INFO(this->get_logger(), "Best Position: [%f, %f], Cost: %f", 
                        global_best_pos_[0], global_best_pos_[1], global_best_cost_);
            return;
        }
        std::uniform_int_distribution<> int_dis(0, pop_size_ - 1);
        std::uniform_real_distribution<> real_dis(0.0, 1.0);
        std::uniform_int_distribution<> dim_dis(0, dim_ - 1);
        std::vector<Agent> new_population = population_;
        for (int i = 0; i < pop_size_; ++i) {
            // 1. 变异：选择 r1, r2, r3，且互不相同且不等于 i
            int r1, r2, r3;
            do { r1 = int_dis(gen_); } while (r1 == i);
            do { r2 = int_dis(gen_); } while (r2 == i || r2 == r1);
            do { r3 = int_dis(gen_); } while (r3 == i || r3 == r2 || r3 == r1);
            std::vector<double> mutant_vector(dim_);
            std::vector<double> trial_vector(dim_);
            // 2. 生成变异向量 & 3. 交叉
            int j_rand = dim_dis(gen_); // 保证至少有一维发生变异
            for (int j = 0; j < dim_; ++j) {
                // 计算变异值 v = x_r1 + F * (x_r2 - x_r3)
                double v_val = population_[r1].position[j] + F_ * (population_[r2].position[j] - population_[r3].position[j]);                
                // 边界处理 (Clamp)
                v_val = std::max(bound_min_, std::min(v_val, bound_max_));
                // 交叉操作
                if (real_dis(gen_) <= CR_ || j == j_rand) {
                    trial_vector[j] = v_val;
                } else {
                    trial_vector[j] = population_[i].position[j];
                }
            }
            // 4. 选择 (Greedy Selection)
            double trial_cost = cost_function(trial_vector);
            if (trial_cost <= population_[i].cost) {
                new_population[i].position = trial_vector;
                new_population[i].cost = trial_cost;
            } else {
                // 保持原样 (new_population 初始化时已复制原种群)
            }
        }
        population_ = new_population;
        update_global_best();
        publish_status();
        current_gen_++;
    }
    void publish_status() {
        auto msg = std_msgs::msg::Float32MultiArray();
        // 格式: [gen, best_x, best_y, best_cost]
        msg.data.push_back(static_cast<float>(current_gen_));
        for(double val : global_best_pos_) msg.data.push_back(static_cast<float>(val));
        msg.data.push_back(static_cast<float>(global_best_cost_));        
        best_sol_pub_->publish(msg);        
        // 打印日志以观察收敛过程
        if (current_gen_ % 10 == 0) {
             RCLCPP_INFO(this->get_logger(), "Gen %d: Best Cost = %.6f at [%.4f, %.4f]", 
                 current_gen_, global_best_cost_, global_best_pos_[0], global_best_pos_[1]);
        }
    }
    // 成员变量
    rclcpp::TimerBase::SharedPtr timer_;
    rclcpp::Publisher<std_msgs::msg::Float32MultiArray>::SharedPtr best_sol_pub_;    
    // DE 参数
    int pop_size_;
    int dim_;
    int max_gens_;
    int current_gen_ = 0;
    double F_;
    double CR_;
    double bound_min_;
    double bound_max_;
    std::mt19937 gen_;
    std::vector<Agent> population_;
    std::vector<double> global_best_pos_;
    double global_best_cost_ = std::numeric_limits<double>::max();
};
int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<DifferentialEvolutionNode>());
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

add_executable(de_node src/de_node.cpp)
ament_target_dependencies(de_node rclcpp std_msgs)
install(TARGETS de_node DESTINATION lib/${PROJECT_NAME})
```