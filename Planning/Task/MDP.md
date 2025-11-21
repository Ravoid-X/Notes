## 概述
1. 马尔可夫决策过程（Markov Decision Process, MDP）是解决不确定性环境下任务规划和决策问题的核心数学框架
2. 在机器人领域，它常用于路径规划、高层任务调度以及强化学习的基础
3. 与传统搜索算法不同，MDP 能够处理动作执行的不确定性（例如：机器人想向前走，但有10%概率滑倒偏向左边）
## 原理
MDP 描述了一个智能体通过与环境交互来最大化累积奖励的过程
### 数学模型
一个 MDP 由一个五元组 $(S, A, P, R, \gamma)$ 定义
1. $S$ (State Space): 状态集合。代表机器人在环境中的所有可能状态（例如：机器人在网格地图的坐标 $(x, y)$）
2. $A$ (Action Space): 动作集合。机器人可以执行的动作（例如：上、下、左、右）
3. $P$ (Transition Probability): 状态转移概率 $P(s' | s, a)$。表示在状态 $s$ 执行动作 $a$ 后，转移到状态 $s'$ 的概率。这是MDP处理不确定性的核心
4. $R$ (Reward Function): 奖励函数 $R(s, a, s')$。智能体在状态 $s$ 执行 $a$ 到达 $s'$ 后获得的即时反馈（到达目标给正分，撞墙给负分，每走一步扣分以鼓励最短路径）
5. $\gamma$ (Discount Factor): 折扣因子 $\gamma \in [0, 1]$。用于平衡当前奖励与未来奖励的重要性
6. 目标是找到一个最优策略 $\pi^*(s)$。策略是一个从状态到动作的映射：告诉机器人在状态 $s$ 应该做什么动作
### 贝尔曼方程
MDP 的求解依赖于价值函数 $V(s)$，它表示从状态 $s$ 开始，一直遵循某种策略能获得的期望累积奖励。最优价值函数 $V^*(s)$ 必须满足贝尔曼最优方程
$$V^*(s) = \max_{a \in A} \sum_{s' \in S} P(s' | s, a) \left[ R(s, a, s') + \gamma V^*(s') \right] $$
一个状态的“价值”，等于执行某个最佳动作后，预期获得的“即时奖励”加上“下一个状态的折扣价值”的加权和
## 算法步骤
虽然有多种求解方法（如策略迭代、Q-Learning），但价值迭代在任务规划中最为直观且常用
### 初始化
将所有状态的价值 $V(s)$ 初始化为 0（或任意随机值）
### 迭代更新
对每一个状态 $s$，利用贝尔曼方程更新其价值
$$V_{k+1}(s) \leftarrow \max\_{a} \sum\_{s'} P(s'|s,a) [R(s,a,s') + \gamma V\_k(s')]$$
### 收敛检查
检查 $\max |V_{k+1}(s) - V_k(s)|$ 是否小于一个极小的阈值 $\epsilon$。如果是，停止迭代
### 提取策略
价值稳定后，通过贪婪法提取最优策略
$$ \\ \pi^*(s) = \arg\max\_{a} \sum\_{s'} P(s'|s,a) [R(s,a,s') + \gamma V^*(s')]$$ 
## 代码示例
### 场景
1. 模拟在一个简单的 Grid World (网格世界) 中使用 MDP 进行规划
2. 目标：(2, 2) 是目标点，奖励 +100；障碍：(1, 1) 是陷阱，奖励 -100
3. 不确定性：为了演示 MDP 特性，假设动作有 80% 成功率，10% 向左偏，10% 向右偏（代码中简化为确定性以便于理解核心逻辑，但预留了概率接口）
### mdp_node.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <vector>
#include <cmath>
#include <algorithm>
#include <iostream>
#include <iomanip>

// 定义简单的网格状态
struct State {
    int x;
    int y;    
    bool operator==(const State& other) const {
        return x == other.x && y == other.y;
    }
};
// 定义动作
enum Action { UP, DOWN, LEFT, RIGHT, STOP };

class MDPPlannerNode : public rclcpp::Node {
public:
    MDPPlannerNode() : Node("mdp_planner_node") {
        // 初始化参数
        grid_width_ = 3;
        grid_height_ = 3;
        gamma_ = 0.9; // 折扣因子
        epsilon_ = 1e-4; // 收敛阈值        
        // 初始化价值地图 (3x3 matrix flatten to vector)
        values_.resize(grid_width_ * grid_height_, 0.0);        
        // 设定特殊奖励
        // 目标点 (2,2)
        setReward(2, 2, 100.0);
        // 陷阱点 (1,1)
        setReward(1, 1, -100.0);
        RCLCPP_INFO(this->get_logger(), "Starting Value Iteration...");
        runValueIteration();
        printPolicy();
    }
private:
    int grid_width_;
    int grid_height_;
    double gamma_;
    double epsilon_;
    std::vector<double> values_;    
    // 简化的奖励地图存储，仅用于示例
    struct RewardEntry {
        int x, y;
        double value;
    };
    std::vector<RewardEntry> special_rewards_;
    // 获取状态索引
    int getIndex(int x, int y) {
        return y * grid_width_ + x;
    }
    // 设置奖励
    void setReward(int x, int y, double val) {
        special_rewards_.push_back({x, y, val});
    }
    // 获取某状态的即时奖励 R(s)
    // 在简化模型中，奖励通常绑定在进入的状态上
    double getReward(int x, int y) {
        for (const auto& r : special_rewards_) {
            if (r.x == x && r.y == y) return r.value;
        }
        return -1.0; // 每走一步的成本，鼓励最短路径
    }
    // 状态转移逻辑 (模拟 Physics)
    // 返回: 下一个状态坐标
    State transition(int x, int y, Action action) {
        int nx = x;
        int ny = y;
        switch (action) {
            case UP:    ny++; break;
            case DOWN:  ny--; break;
            case LEFT:  nx--; break;
            case RIGHT: nx++; break;
            case STOP:  break;
        }
        // 边界检查：撞墙保持原地
        if (nx < 0 || nx >= grid_width_ || ny < 0 || ny >= grid_height_) {
            return {x, y};
        }        
        // 障碍物检查逻辑可在此添加        
        return {nx, ny};
    }
    // 价值迭代核心算法
    void runValueIteration() {
        int iteration = 0;
        while (true) {
            double max_delta = 0.0;
            std::vector<double> new_values = values_;
            for (int y = 0; y < grid_height_; ++y) {
                for (int x = 0; x < grid_width_; ++x) {                    
                    // 跳过目标状态和陷阱状态（作为终结状态，价值固定或不更新）
                    // 这里为了简单，我们假设到达目标后价值即为奖励本身，不再迭代
                    bool is_terminal = false;
                    for(auto& r : special_rewards_) {
                        if(r.x == x && r.y == y) {
                            new_values[getIndex(x,y)] = r.value; // 保持终结状态价值
                            is_terminal = true;
                        }
                    }
                    if(is_terminal) continue;
                    double max_q = -1e9;
                    // 遍历所有可能的动作
                    std::vector<Action> actions = {UP, DOWN, LEFT, RIGHT};
                    for (auto action : actions) {                        
                        // 计算 Q(s, a)
                        // 贝尔曼方程: sum(P(s'|s,a) * [R + gamma * V(s')])
                        // 此处演示确定性转移：P=1.0 (若要改为概率性，需在此处对多个可能的next_state求和)                        
                        State next_s = transition(x, y, action);
                        double reward = getReward(next_s.x, next_s.y);
                        double next_val = values_[getIndex(next_s.x, next_s.y)];                        
                        // 标准 Bellman Update (Deterministic)
                        double q_value = reward + gamma_ * next_val;                        
                        /* // 如果是概率性 MDP (例如：80%成功，10%左滑，10%右滑)，代码如下：
                        // double q_value = 0.0;
                        // for (auto possible_outcome : outcomes) {
                        //    q_value += possible_outcome.prob * (possible_outcome.reward + gamma_ * values_[...]);
                        // }
                        */
                        if (q_value > max_q) {
                            max_q = q_value;
                        }
                    }
                    new_values[getIndex(x, y)] = max_q;
                    max_delta = std::max(max_delta, std::abs(max_q - values_[getIndex(x, y)]));
                }
            }
            values_ = new_values;
            iteration++;            
            // 打印进度
            if (iteration % 10 == 0) {
               RCLCPP_INFO(this->get_logger(), "Iteration %d, Delta: %f", iteration, max_delta);
            }
            if (max_delta < epsilon_) {
                RCLCPP_INFO(this->get_logger(), "Converged after %d iterations.", iteration);
                break;
            }
        }
    }
    // 输出最终策略
    void printPolicy() {
        std::cout << "\n--- Final Policy & Values ---\n";
        for (int y = grid_height_ - 1; y >= 0; --y) { // 打印从上到下
            for (int x = 0; x < grid_width_; ++x) {                
                // 再次寻找最佳动作
                double best_val = -1e9;
                std::string best_act = " . ";                
                // 检查是否为特殊点
                bool is_special = false;
                 for(auto& r : special_rewards_) {
                    if(r.x == x && r.y == y) {
                        if(r.value > 0) best_act = "[G]"; // Goal
                        else best_act = "[X]"; // Trap
                        is_special = true;
                    }
                }
                if (!is_special) {
                    std::vector<Action> actions = {UP, DOWN, LEFT, RIGHT};
                    for (auto action : actions) {
                        State next_s = transition(x, y, action);
                        double val = getReward(next_s.x, next_s.y) + gamma_ * values_[getIndex(next_s.x, next_s.y)];
                        if (val > best_val) {
                            best_val = val;
                            switch (action) {
                                case UP: best_act = " ^ "; break;
                                case DOWN: best_act = " v "; break;
                                case LEFT: best_act = " < "; break;
                                case RIGHT: best_act = " > "; break;
                                default: break;
                            }
                        }
                    }
                }                
                std::cout << std::fixed << std::setprecision(1) << values_[getIndex(x,y)] << best_act << "\t";
            }
            std::cout << std::endl;
        }
    }
};
int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    auto node = std::make_shared<MDPPlannerNode>();
    // 由于本例是计算型任务，计算完即可退出，或者使用 spin 等待服务回调
    // rclcpp::spin(node); 
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.8)
project(mdp_planner)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)

add_executable(mdp_node src/mdp_node.cpp)
ament_target_dependencies(mdp_node rclcpp)

install(TARGETS
    mdp_node
    DESTINATION lib/${PROJECT_NAME})

ament_package()
```
### package.xml
```XML
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
    <name>mdp_planner</name>
    <version>0.0.0</version>
    <description>MDP Algorithm Example for ROS 2</description>
    <maintainer email="user@todo.todo">user</maintainer>
    <license>TODO</license>

    <buildtool_depend>ament_cmake</buildtool_depend>
    <depend>rclcpp</depend>

    <test_depend>ament_lint_auto</test_depend>
    <test_depend>ament_lint_common</test_depend>

    <export>
        <build_type>ament_cmake</build_type>
    </export>
</package>
```