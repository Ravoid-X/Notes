## 概述
1. 模拟退火算法 (Simulated Annealing, SA) 是一种启发式搜索算法，主要用于在庞大的搜索空间中寻找全局最优解
2. 灵感来源于金属冶金中的退火过程。金属被加热到高温（原子高速运动，无序），然后缓慢冷却。在冷却过程中，原子逐渐趋于有序，最终形成低能态的晶体结构
## 原理
### 算法映射
1. 温度 ($T$) $\rightarrow$ 控制搜索随机性的参数
2. 能量 ($E$) $\rightarrow$ 目标函数值
3. 状态 $\rightarrow$ 解空间中的当前解
4. 冷却 $\rightarrow$ 随着迭代进行，降低 $T$，减少接受“坏解”的概率
### Metropolis 准则
1. SA 与传统的爬山算法最大的不同在于：它有概率接受比当前更差的解。这使得算法能够跳出局部最优，从而有机会找到全局最优
2. 假设当前解为 $S$，新产生的邻域解为 $S_{new}$，能量差为 $\Delta E = E(S_{new}) - E(S)$
3. 接受新解的概率 $P$ 定义为
$$P = \begin{cases} 
1 & \text{if } \Delta E < 0 \quad (\text{新解更优，直接接受}) \\
e^{-\frac{\Delta E}{T}} & \text{if } \Delta E \ge 0 \quad (\text{新解更差，按概率接受})
\end{cases}$$
>当 $T$ 很高时，$e^{-\frac{\Delta E}{T}}$ 接近 1，算法表现得像随机游走，极易接受坏解，探索能力强

>当 $T$ 很低时，$e^{-\frac{\Delta E}{T}}$ 接近 0，算法趋向于贪婪算法，只接受好解，收敛能力强
## 算法步骤
1. 初始化：设定初始温度 $T_{start}$，终止温度 $T_{end}$，降温系数 $\alpha$ (通常为 0.95~0.99)，以及初始解 $S$
2. 生成新解 在当前解 $S$ 的邻域内随机扰动产生一个新解 $S_{new}$
3. 计算能量差：$\Delta E = E(S_{new}) - E(S)$
4. 判断接受：根据 Metropolis 准则决定是否用 $S_{new}$ 替代 $S$
5. 降温：更新温度 $T = T \times \alpha$
6. 循环：重复步骤 2-5，直到 $T < T_{end}$ 或达到最大迭代次数
## 代码示例
### 场景
1. 假设要在二维平面上找到一个数学函数的最小值（模拟机器人寻找地势最低点）
2. 目标函数：Rastrigin 函数的一个简化版（具有许多局部最小值的函数）
$$f(x, y) = x^2 + y^2 - 10(\cos(2\pi x) + \cos(2\pi y)) + 20$$
### sa_node.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <std_msgs/msg/float64_multi_array.hpp>
#include <geometry_msgs/msg/point.hpp>
#include <cmath>
#include <random>
#include <vector>
#include <algorithm>

// 定义常量 PI
#ifndef M_PI
#define M_PI 3.14159265358979323846
#endif

class SimulatedAnnealingNode : public rclcpp::Node {
public:
    SimulatedAnnealingNode() : Node("simulated_annealing_node") {
        // 1. 声明并获取参数
        this->declare_parameter("initial_temp", 1000.0);
        this->declare_parameter("cooling_rate", 0.99);
        this->declare_parameter("min_temp", 0.01);
        this->declare_parameter("step_size", 0.5); // 邻域扰动范围
        initial_temp_ = this->get_parameter("initial_temp").as_double();
        cooling_rate_ = this->get_parameter("cooling_rate").as_double();
        min_temp_ = this->get_parameter("min_temp").as_double();
        step_size_ = this->get_parameter("step_size").as_double();
        // 初始化发布者 (用于发布当前最优解的位置，便于可视化)
        solution_pub_ = this->create_publisher<geometry_msgs::msg::Point>("best_solution", 10);
        // 使用定时器触发一次性优化任务（为了不阻塞构造函数）
        timer_ = this->create_wall_timer(
            std::chrono::seconds(1),
            std::bind(&SimulatedAnnealingNode::run_optimization, this));
        // 初始化随机数生成器
        std::random_device rd;
        gen_ = std::mt19937(rd());
        dist_ = std::uniform_real_distribution<>(-5.12, 5.12); // 搜索范围 [-5.12, 5.12]
        prob_dist_ = std::uniform_real_distribution<>(0.0, 1.0);
    }
private:
    // 目标函数 (Rastrigin Function 简化版)
    // 全局最小值在 (0, 0)，值为 0
    double objective_function(double x, double y) {
        return (x * x - 10 * cos(2 * M_PI * x)) + (y * y - 10 * cos(2 * M_PI * y)) + 20;
    }
    // 获取邻域解
    std::pair<double, double> get_neighbor(double x, double y) {
        std::uniform_real_distribution<> perturbation(-step_size_, step_size_);
        double new_x = x + perturbation(gen_);
        double new_y = y + perturbation(gen_);        
        // 简单的边界限制
        new_x = std::max(-5.12, std::min(5.12, new_x));
        new_y = std::max(-5.12, std::min(5.12, new_y));        
        return {new_x, new_y};
    }
    void run_optimization() {
        // 取消定时器，只运行一次
        timer_->cancel();
        RCLCPP_INFO(this->get_logger(), "Starting Simulated Annealing...");
        // 初始解
        double current_x = dist_(gen_);
        double current_y = dist_(gen_);
        double current_energy = objective_function(current_x, current_y);
        // 记录全局最优
        double best_x = current_x;
        double best_y = current_y;
        double best_energy = current_energy;
        double temp = initial_temp_;
        int iteration = 0;
        // --- 模拟退火主循环 ---
        while (temp > min_temp_ && rclcpp::ok()) {
            // 1. 生成新解
            auto neighbor = get_neighbor(current_x, current_y);
            double new_x = neighbor.first;
            double new_y = neighbor.second;
            double new_energy = objective_function(new_x, new_y);
            // 2. 计算能量差
            double delta_energy = new_energy - current_energy;
            // 3. Metropolis 准则
            bool accept = false;
            if (delta_energy < 0) {
                accept = true; // 更好的解，直接接受
            } else {
                // 更差的解，按概率接受
                double probability = std::exp(-delta_energy / temp);
                if (prob_dist_(gen_) < probability) {
                    accept = true;
                }
            }
            if (accept) {
                current_x = new_x;
                current_y = new_y;
                current_energy = new_energy;
                // 更新全局最优
                if (current_energy < best_energy) {
                    best_x = current_x;
                    best_y = current_y;
                    best_energy = current_energy;
                }
            }
            // 4. 发布当前数据便于调试/可视化
            if (iteration % 100 == 0) {
                geometry_msgs::msg::Point msg;
                msg.x = best_x;
                msg.y = best_y;
                msg.z = best_energy;
                solution_pub_->publish(msg);                
                // 偶尔打印日志
                RCLCPP_INFO(this->get_logger(), 
                    "Iter: %d, Temp: %.2f, Best E: %.4f, Current E: %.4f", 
                    iteration, temp, best_energy, current_energy);
            }
            // 5. 降温
            temp *= cooling_rate_;
            iteration++;            
            // 添加微小延时，模拟实时过程（可选）
            // std::this_thread::sleep_for(std::chrono::milliseconds(1));
        }
        RCLCPP_INFO(this->get_logger(), "Optimization Finished!");
        RCLCPP_INFO(this->get_logger(), "Final Solution: x=%.4f, y=%.4f, Energy=%.4f", best_x, best_y, best_energy);
    }
    double initial_temp_;
    double cooling_rate_;
    double min_temp_;
    double step_size_;
    rclcpp::Publisher<geometry_msgs::msg::Point>::SharedPtr solution_pub_;
    rclcpp::TimerBase::SharedPtr timer_;    
    std::mt19937 gen_;
    std::uniform_real_distribution<> dist_;
    std::uniform_real_distribution<> prob_dist_;
};
int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    auto node = std::make_shared<SimulatedAnnealingNode>();
    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)

add_executable(sa_node src/sa_node.cpp)
ament_target_dependencies(sa_node rclcpp std_msgs geometry_msgs)

install(TARGETS
    sa_node
    DESTINATION lib/${PROJECT_NAME})

ament_package()
```
### package.xml
```XML
<depend>rclcpp</depend>
<depend>std_msgs</depend>
<depend>geometry_msgs</depend>
```