## 概述
1. 粒子群优化算法 (Particle Swarm Optimization, PSO) 是一种基于群体智能的进化计算技术，灵感来源于鸟群觅食的行为
2. 每个解被称为一个粒子，所有粒子在搜索空间中飞行，每个粒子都由其位置向量、速度向量和适应度值决定
## 原理
PSO 的核心在于粒子速度和位置的更新公式
### 数学模型
假设在一个 $D$ 维搜索空间中，有 $N$ 个粒子。
对于第 $i$ 个粒子 ($i = 1, ..., N$)：
1. 位置: $X_i = (x_{i1}, x_{i2}, ..., x_{iD})$
2. 速度: $V_i = (v_{i1}, v_{i2}, ..., v_{iD})$
3. 个体历史最优 (PBest): $P_i = (p_{i1}, p_{i2}, ..., p_{iD})$
4. 全局历史最优 (GBest): $P_g = (p_{g1}, p_{g2}, ..., p_{gD})$
### 速度更新公式
$$v_{id}^{t+1} = \underbrace{w \cdot v_{id}^t}_{\text{惯性部分}} + \underbrace{c_1 \cdot r_1 \cdot (p_{id}^t - x_{id}^t)}_{\text{认知部分 (自我认知)}} + \underbrace{c_2 \cdot r_2 \cdot (p_{gd}^t - x_{id}^t)}_{\text{社会部分 (群体协作)}}$$
### 位置更新公式
$$x_{id}^{t+1} = x_{id}^t + v_{id}^{t+1}$$
### 参数解释
1. $w$ (Inertia Weight, 惯性权重): 控制粒子保持前一时刻运动状态的能力\
（1）$w$ 较大：全局搜索能力强（容易跳出局部最优）\
（2）$w$ 较小：局部开发能力强（收敛精度高），通常采用线性递减策略
2. $c_1, c_2$ (Acceleration Coefficients, 学习因子)\
（1）$c_1$ (自我认知): 调节粒子向自身历史最好位置飞行的步长\
（2）$c_2$ (社会经验): 调节粒子向群体最好位置飞行的步长。
3. $r_1, r_2$: $[0, 1]$ 之间的随机数，增加搜索的随机性
## 算法步骤
1. 初始化: 随机初始化粒子群的位置 $X$ 和速度 $V$
2. 评估: 计算每个粒子的适应度值
3. 更新 PBest: 如果当前粒子的适应度优于其历史最优值，则更新 $P_i$
4. 更新 GBest: 如果当前粒子的适应度优于全局历史最优值，则更新 $P_g$
5. 更新状态: 根据公式更新粒子的速度和位置
6. 边界处理: 检查粒子是否飞出边界，若越界需进行限制（如反弹或截断）
7. 终止判断: 达到最大迭代次数或满足误差要求则停止，否则返回步骤 2
## 代码示例
寻找二维平面上目标点 $(10.0, 10.0)$，节点会发布当前的全局最优解，并通过日志输出迭代过程
### pso_node.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <geometry_msgs/msg/point.hpp>
#include <vector>
#include <random>
#include <cmath>
#include <algorithm>
#include <iostream>

// 定义粒子结构体
struct Particle {
    std::vector<double> position;
    std::vector<double> velocity;
    std::vector<double> pbest_position;
    double pbest_value;
    double current_value;
    Particle(int dim) {
        position.resize(dim);
        velocity.resize(dim);
        pbest_position.resize(dim);
        pbest_value = std::numeric_limits<double>::max();
    }
};

class PSOOptimizer : public rclcpp::Node {
public:
    PSOOptimizer() : Node("pso_optimizer"), gen_(rd_()) {
        // 声明参数
        this->declare_parameter("num_particles", 30);
        this->declare_parameter("iterations", 100);
        this->declare_parameter("target_x", 10.0);
        this->declare_parameter("target_y", 10.0);
        // 获取参数
        num_particles_ = this->get_parameter("num_particles").as_int();
        max_iterations_ = this->get_parameter("iterations").as_int();
        target_x_ = this->get_parameter("target_x").as_double();
        target_y_ = this->get_parameter("target_y").as_double();
        // PSO 超参数
        w_ = 0.7;   // 惯性权重
        c1_ = 1.5;  // 认知系数
        c2_ = 1.5;  // 社会系数        
        // 初始化发布者 (发布当前的全局最优位置)
        gbest_publisher_ = this->create_publisher<geometry_msgs::msg::Point>("pso_best_position", 10);
        // 初始化粒子群
        initialize_particles();
        // 创建定时器进行迭代 (模拟实时优化过程)
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(100), // 10Hz
            std::bind(&PSOOptimizer::optimize_step, this)
        );
        RCLCPP_INFO(this->get_logger(), "PSO Node Started. Target: (%.2f, %.2f)", target_x_, target_y_);
    }
private:
    // 随机数生成器
    std::random_device rd_;
    std::mt19937 gen_;
    // 参数
    int num_particles_;
    int max_iterations_;
    double target_x_, target_y_;
    double w_, c1_, c2_;
    int current_iter_ = 0;
    // 粒子群数据
    std::vector<Particle> particles_;
    std::vector<double> gbest_position_;
    double gbest_value_ = std::numeric_limits<double>::max();
    // ROS 通信
    rclcpp::Publisher<geometry_msgs::msg::Point>::SharedPtr gbest_publisher_;
    rclcpp::TimerBase::SharedPtr timer_;
    // 辅助函数：生成随机浮点数 [min, max]
    double random_double(double min, double max) {
        std::uniform_real_distribution<> dis(min, max);
        return dis(gen_);
    }
    // 适应度函数 (目标函数)：计算与目标的欧氏距离
    double calculate_fitness(const std::vector<double>& pos) {
        double dx = pos[0] - target_x_;
        double dy = pos[1] - target_y_;
        return std::sqrt(dx * dx + dy * dy);
    }
    void initialize_particles() {
        gbest_position_.resize(2);        
        for (int i = 0; i < num_particles_; ++i) {
            Particle p(2); // 2D 空间            
            // 随机初始化位置 [-20, 20] 和速度 [-1, 1]
            p.position[0] = random_double(-20.0, 20.0);
            p.position[1] = random_double(-20.0, 20.0);
            p.velocity[0] = random_double(-1.0, 1.0);
            p.velocity[1] = random_double(-1.0, 1.0);            
            // 初始评估
            p.current_value = calculate_fitness(p.position);
            p.pbest_position = p.position;
            p.pbest_value = p.current_value;
            // 更新全局最优
            if (p.current_value < gbest_value_) {
                gbest_value_ = p.current_value;
                gbest_position_ = p.position;
            }
            particles_.push_back(p);
        }
    }
    void optimize_step() {
        if (current_iter_ >= max_iterations_) {
            timer_->cancel();
            RCLCPP_INFO(this->get_logger(), "Optimization Finished!");
            RCLCPP_INFO(this->get_logger(), "Final Best Position: [%.4f, %.4f], Error: %.6f", 
                        gbest_position_[0], gbest_position_[1], gbest_value_);
            return;
        }
        for (auto& p : particles_) {
            for (int d = 0; d < 2; ++d) {
                double r1 = random_double(0.0, 1.0);
                double r2 = random_double(0.0, 1.0);
                // 1. 更新速度
                // v = w*v + c1*r1*(pbest - x) + c2*r2*(gbest - x)
                p.velocity[d] = w_ * p.velocity[d] +
                                c1_ * r1 * (p.pbest_position[d] - p.position[d]) +
                                c2_ * r2 * (gbest_position_[d] - p.position[d]);

                // 速度限幅 (可选，防止发散)
                p.velocity[d] = std::max(-2.0, std::min(2.0, p.velocity[d]));
                // 2. 更新位置
                p.position[d] += p.velocity[d];
            }
            // 3. 评估新位置
            p.current_value = calculate_fitness(p.position);
            // 4. 更新个体最优 PBest
            if (p.current_value < p.pbest_value) {
                p.pbest_value = p.current_value;
                p.pbest_position = p.position;
            }
            // 5. 更新全局最优 GBest
            if (p.current_value < gbest_value_) {
                gbest_value_ = p.current_value;
                gbest_position_ = p.position;
            }
        }
        // 发布当前最优解
        auto msg = geometry_msgs::msg::Point();
        msg.x = gbest_position_[0];
        msg.y = gbest_position_[1];
        msg.z = 0.0;
        gbest_publisher_->publish(msg);
        if (current_iter_ % 10 == 0) {
             RCLCPP_INFO(this->get_logger(), "Iter %d: Best Error = %.6f", current_iter_, gbest_value_);
        }        
        current_iter_++;
    }
};
int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<PSOOptimizer>());
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(geometry_msgs REQUIRED)

add_executable(pso_node src/pso_node.cpp)
ament_target_dependencies(pso_node rclcpp geometry_msgs)

install(TARGETS
  pso_node
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```
### package.xml
```XML
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
    <name>pso_cpp</name>
    <version>0.0.1</version>
    <description>Particle Swarm Optimization example in ROS 2</description>
    <maintainer email="user@example.com">User</maintainer>
    <license>Apache-2.0</license>

    <buildtool_depend>ament_cmake</buildtool_depend>

    <depend>rclcpp</depend>
    <depend>geometry_msgs</depend>

    <test_depend>ament_lint_auto</test_depend>
    <test_depend>ament_lint_common</test_depend>

    <export>
        <build_type>ament_cmake</build_type>
    </export>
</package>
```