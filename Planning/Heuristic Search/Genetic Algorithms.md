## 概述
1. 遗传算法 (Genetic Algorithms, GA) 是一种元启发式搜索算法，灵感来源于达尔文的生物进化论
2. 不保证找到全局最优解，但在搜索空间巨大、非线性、非凸的复杂问题中，能以较高的效率找到近似最优解
## 原理
将问题的解空间映射为“种群”，每一个解被称为“个体”或“染色体”。通过模拟自然界的进化过程，让种群逐代演变，最终收敛到适应度最高的解
### 核心概念
1. 基因: 解的最小特征单元（例如路径规划中的一个坐标点）
2. 染色体/个体: 一组基因的集合，代表问题的一个潜在解（例如一条完整的路径）
3. 种群: 多个个体的集合
4. 适应度函数: 核心评价标准，用于量化一个个体“有多好”。适应度越高，生存概率越大
## 算法步骤
标准的 GA 流程是一个循环迭代的过程
### 初始化
1. 随机生成 $N$ 个个体组成初始种群。
2. 在机器人领域，通常是随机生成一组轨迹或参数
### 评估
1. 算种群中每个个体的适应度
2. 例如：$Fitness = \frac{1}{\text{距离目标的误差} + \text{碰撞惩罚}}$
### 选择
1. 根据适应度选择父代。由“轮盘赌选择法”或“锦标赛选择法”实现
2. 适应度高的个体有更大几率被选中遗传给下一代
### 交叉
1. 模拟生物交配。选取两个父代，交换部分基因片段，生成新的子代
2. 目的是融合父代的优良特征，探索解空间的新区域
### 变异
1. 以极小的概率随机改变个体中的某些基因（例如给坐标加一个随机噪声）
2. 目的是维持种群多样性，防止算法过早陷入局部最优
### 终止
达到最大迭代次数或适应度达到阈值，输出当前最优解
## 代码示例
在 2D 平面上，找到一组坐标 $(x, y)$，使其无限接近目标点 $(10.0, 10.0)$
### ga_node.cpp
```C++
/**
 * Genetic Algorithm Example for ROS 2
 * Scene: Find optimal coordinates (x, y) to reach a target (10, 10).
 */

#include <rclcpp/rclcpp.hpp>
#include <std_msgs/msg/string.hpp>
#include <geometry_msgs/msg/point.hpp>
#include <vector>
#include <random>
#include <algorithm>
#include <cmath>
#include <sstream>

// --- GA Parameters ---
const int POPULATION_SIZE = 20;
const double MUTATION_RATE = 0.1;
const double CROSSOVER_RATE = 0.7;
const double TARGET_X = 10.0;
const double TARGET_Y = 10.0;
const double GENE_MIN = 0.0;
const double GENE_MAX = 20.0; // Search space boundary

// --- Data Structures ---
struct Individual {
    double x;
    double y;
    double fitness;
};

class GeneticAlgorithmNode : public rclcpp::Node {
public:
    GeneticAlgorithmNode() : Node("genetic_algorithm_node"), generation_count_(0) {
        // Publisher to show the best result of current generation
        best_sol_pub_ = this->create_publisher<geometry_msgs::msg::Point>("ga_best_solution", 10);        
        // Initialize random engine
        std::random_device rd;
        rng_ = std::mt19937(rd());
        // 1. Initialization
        initialize_population();
        // Create a timer to run one generation per second (1 Hz) for visualization
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(500),
            std::bind(&GeneticAlgorithmNode::run_generation_loop, this)
        );
        RCLCPP_INFO(this->get_logger(), "GA Node Started. Target: (%.2f, %.2f)", TARGET_X, TARGET_Y);
    }
private:
    std::vector<Individual> population_;
    std::mt19937 rng_;
    rclcpp::Publisher<geometry_msgs::msg::Point>::SharedPtr best_sol_pub_;
    rclcpp::TimerBase::SharedPtr timer_;
    int generation_count_;
    // --- Helper: Random Double ---
    double random_double(double min, double max) {
        std::uniform_real_distribution<double> dist(min, max);
        return dist(rng_);
    }
    // --- Step 1: Initialize Population ---
    void initialize_population() {
        population_.clear();
        for (int i = 0; i < POPULATION_SIZE; ++i) {
            Individual ind;
            ind.x = random_double(GENE_MIN, GENE_MAX);
            ind.y = random_double(GENE_MIN, GENE_MAX);
            ind.fitness = 0.0;
            population_.push_back(ind);
        }
    }
    // --- Step 2: Calculate Fitness ---
    void evaluate_fitness() {
        for (auto &ind : population_) {
            double dist = std::sqrt(std::pow(ind.x - TARGET_X, 2) + std::pow(ind.y - TARGET_Y, 2));
            // Prevent division by zero, add small epsilon
            ind.fitness = 1.0 / (dist + 1e-6);
        }
    }
    // --- Step 3: Selection (Tournament Selection) ---
    Individual tournament_selection() {
        int k = 3; 
        Individual best = population_[std::uniform_int_distribution<int>(0, POPULATION_SIZE - 1)(rng_)];        
        for (int i = 0; i < k - 1; ++i) {
            Individual candidate = population_[std::uniform_int_distribution<int>(0, POPULATION_SIZE - 1)(rng_)];
            if (candidate.fitness > best.fitness) {
                best = candidate;
            }
        }
        return best;
    }
    // --- Step 4: Crossover (Arithmetic Crossover) ---
    Individual crossover(const Individual &p1, const Individual &p2) {
        Individual child;
        if (random_double(0.0, 1.0) < CROSSOVER_RATE) {
            // Linear interpolation (common for real-valued GA)
            double alpha = random_double(0.0, 1.0);
            child.x = alpha * p1.x + (1.0 - alpha) * p2.x;
            child.y = alpha * p1.y + (1.0 - alpha) * p2.y;
        } else {
            // Just copy one parent
            child = p1; 
        }
        return child;
    }
    // --- Step 5: Mutation (Gaussian Mutation) ---
    void mutate(Individual &ind) {
        if (random_double(0.0, 1.0) < MUTATION_RATE) {
            // Add noise between -1.0 and 1.0
            ind.x += random_double(-1.0, 1.0);
            ind.y += random_double(-1.0, 1.0);
            // Clamp values to boundaries
            ind.x = std::clamp(ind.x, GENE_MIN, GENE_MAX);
            ind.y = std::clamp(ind.y, GENE_MIN, GENE_MAX);
        }
    }
    void run_generation_loop() {
        // A. Evaluate
        evaluate_fitness();
        // Find and publish best for this generation (for logging)
        auto best_it = std::max_element(population_.begin(), population_.end(), 
            [](const Individual &a, const Individual &b) { return a.fitness < b.fitness; });        
        Individual best_current = *best_it;        
        // Publish ROS msg
        geometry_msgs::msg::Point msg;
        msg.x = best_current.x;
        msg.y = best_current.y;
        msg.z = 0.0;
        best_sol_pub_->publish(msg);
        double distance = std::sqrt(std::pow(best_current.x - TARGET_X, 2) + std::pow(best_current.y - TARGET_Y, 2));
        RCLCPP_INFO(this->get_logger(), "Gen %d | Best: (%.2f, %.2f) | Dist: %.4f", 
            generation_count_, best_current.x, best_current.y, distance);
        // Check termination (optional)
        if (distance < 0.1) {
            RCLCPP_INFO(this->get_logger(), "Target Reached! Stopping.");
            timer_->cancel();
            return;
        }
        // B. Create New Generation
        std::vector<Individual> new_population;        
        // Elitism
        new_population.push_back(best_current);
        while (new_population.size() < POPULATION_SIZE) {
            // Selection
            Individual p1 = tournament_selection();
            Individual p2 = tournament_selection();            
            // Crossover
            Individual child = crossover(p1, p2);            
            // Mutation
            mutate(child);            
            new_population.push_back(child);
        }
        population_ = new_population;
        generation_count_++;
    }
};
int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    auto node = std::make_shared<GeneticAlgorithmNode>();
    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
find_package(rclcpp REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(std_msgs REQUIRED)

add_executable(ga_node src/ga_node.cpp)
ament_target_dependencies(ga_node rclcpp geometry_msgs std_msgs)

install(TARGETS
  ga_node
  DESTINATION lib/${PROJECT_NAME}
)
```