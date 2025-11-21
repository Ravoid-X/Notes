## 概述
1. POMDP (Partially Observable Markov Decision Process，部分可观测马尔可夫决策过程) 是解决“机器人不知道确切状态，只能通过有噪声的传感器猜测，并据此做决策”这一问题的数学框架
2. 相比于传统的规划算法（假设已知世界状态），POMDP 引入了“信念（Belief）”的概念，让机器人在“探索（收集信息）”和“利用（执行任务）”之间进行权衡
## 原理
### 数学定义
POMDP 定义为一个七元组 $(S, A, T, R, \Omega, O, \gamma)$
1. $S$ (States): 真实世界的隐藏状态集合（机器人无法直接看到）。
2. $A$ (Actions): 机器人可执行的动作集合。
3. $T$ (Transitions): 状态转移概率 $T(s' | s, a) = P(s_{t+1}=s' | s_t=s, a_t=a)$。
4. $R$ (Reward): 奖励函数 $R(s, a)$。
5. $\Omega$ (Observations): 机器人能感知到的观测值集合。
6. $O$ (Observation Function): 观测概率 $O(o | s', a) = P(o_{t+1}=o | s_{t+1}=s', a_t=a)$。即：在执行动作 $a$ 并到达状态 $s'$ 后，观察到 $o$ 的概率
### 信念状态
1. 因为不知道真实状态 $s$，机器人维护一个信念 $b$，$b(s)$ 表示机器人认为当前处于状态 $s$ 的概率
2. $b$ 是一个概率分布，$\sum_{s \in S} b(s) = 1$
3. 信念更新：当机器人执行动作 $a$，并获得观测 $o$ 后，利用贝叶斯公式更新信念
$$b'(s') = \eta O(o | s', a) \sum_{s \in S} T(s' | s, a) b(s)$$
 $\eta$ 是归一化常数，确保概率和为 1
## 算法步骤
解决 POMDP 实际上是在信念空间上解决一个 MDP
### 初始化
设定初始信念 $b_0$（例如：均匀分布，代表完全无知）
### 策略规划
1. 这是最难的一步，目标是找到策略 $\pi(b) \to a$
2. 由于信念空间是连续的，求精确解通常不可行（PSPACE-complete 难题）
3. 常用近似算法: PBVI (Point-Based Value Iteration), POMCP (基于蒙特卡洛树搜索), DESPOT
### 动作选择
根据当前信念 $b$，查询策略得到最佳动作 $a^*$
### 执行与观测
1. 机器人执行 $a^*$
2. 环境发生变化（状态从 $s$ 变为 $s'$，不仅机器人不知道，求解器也不知道）
3. 机器人获得观测值 $o$ 和奖励 $r$
### 贝叶斯更新
根据 $a^*$ 和 $o$，计算新的信念 $b'$
### 循环
令 $b \leftarrow b'$，回到动作选择
## 代码示例
### 场景描述
1. 环境：机器人面前有左、右两扇门
2. 隐藏状态 ($S$)：目标（Target）在左门 (Left) 或右门 (Right)
3. 动作 ($A$)：\
（1）LISTEN: 使用传感器探测（即使目标在左边，也有 15% 概率听错成右边）\
（2）OPEN_LEFT: 打开左门\
（3）OPEN_RIGHT: 打开右门
4. 奖励 ($R$)：\
（1）打开有目标的门：+10（任务成功）\
（2）打开空的门：-100（任务失败/撞墙）\
（3）探测 (Listen)：-1（时间成本）
5. 目标：机器人需要通过 LISTEN 更新信念，直到确信度足够高才开门
### 
```C++
// pomdp_node.cpp
#include <rclcpp/rclcpp.hpp>
#include <vector>
#include <string>
#include <cmath>
#include <iostream>
#include <random>

// 定义状态、动作和观测
enum State { TARGET_LEFT = 0, TARGET_RIGHT = 1 };
enum Action { LISTEN = 0, OPEN_LEFT = 1, OPEN_RIGHT = 2 };
enum Observation { HEAR_LEFT = 0, HEAR_RIGHT = 1, NO_OBS = 2 };

class POMDPNode : public rclcpp::Node {
public:
    POMDPNode() : Node("simple_pomdp_planner") {
        // 1. 初始化信念：50% 左，50% 右 (完全不确定)
        belief_ = {0.5, 0.5};        
        // 初始化真实世界模拟 (仅用于生成观测，规划器不知道这个值)
        std::random_device rd;
        rng_ = std::mt19937(rd());
        // 随机放置目标
        true_state_ = (std::uniform_int_distribution<>(0, 1)(rng_) == 0) ? TARGET_LEFT : TARGET_RIGHT;
        RCLCPP_INFO(this->get_logger(), "Simulated Truth: Target is on the %s", 
                    (true_state_ == TARGET_LEFT ? "LEFT" : "RIGHT"));
        // 设置定时器进行规划循环
        timer_ = this->create_wall_timer(
            std::chrono::seconds(1), std::bind(&POMDPNode::planningLoop, this));
    }
private:
    // --- POMDP 模型参数 ---
    const double LISTEN_COST = -1.0;
    const double OPEN_REWARD = 10.0;
    const double WRONG_PENALTY = -100.0;
    const double SENSOR_ACCURACY = 0.85; // 传感器准确率 85%
    // 当前信念状态 [P(Left), P(Right)]
    std::vector<double> belief_;    
    // 模拟器状态
    State true_state_;
    std::mt19937 rng_;
    rclcpp::TimerBase::SharedPtr timer_;
    bool mission_complete_ = false;
    // --- 核心逻辑：规划循环 ---
    void planningLoop() {
        if (mission_complete_) return;
        RCLCPP_INFO(this->get_logger(), "--------------------------------");
        RCLCPP_INFO(this->get_logger(), "Current Belief: Left=%.2f, Right=%.2f", belief_[0], belief_[1]);
        // 1. 选择动作 (策略层)
        Action action = selectBestAction();
        // 2. 执行动作并获取观测 (模拟与环境交互)
        Observation obs = simulateEnvironment(action);
        // 3. 处理终端状态
        if (action == OPEN_LEFT || action == OPEN_RIGHT) {
            if ((action == OPEN_LEFT && true_state_ == TARGET_LEFT) ||
                (action == OPEN_RIGHT && true_state_ == TARGET_RIGHT)) {
                RCLCPP_INFO(this->get_logger(), "ACTION: Open %s -> SUCCESS! Reward: %.1f", 
                            (action == OPEN_LEFT ? "Left" : "Right"), OPEN_REWARD);
            } else {
                RCLCPP_INFO(this->get_logger(), "ACTION: Open %s -> FAIL! Reward: %.1f", 
                            (action == OPEN_LEFT ? "Left" : "Right"), WRONG_PENALTY);
            }
            mission_complete_ = true;
            return; // 任务结束，不再更新信念
        }
        // 4. 如果是信息收集动作 (LISTEN)，执行贝叶斯信念更新
        if (action == LISTEN) {
            RCLCPP_INFO(this->get_logger(), "ACTION: Listen -> Observation: %s", 
                        (obs == HEAR_LEFT ? "Hear Left" : "Hear Right"));
            updateBelief(action, obs);
        }
    }
    // --- 策略：简单的贪婪策略 ---
    // 在实际 POMDP 中，这里会使用值迭代或树搜索 (POMCP)
    Action selectBestAction() {
        double confidence_threshold = 0.90; // 只有确信度 > 90% 才开门

        if (belief_[0] > confidence_threshold) {
            return OPEN_LEFT;
        } else if (belief_[1] > confidence_threshold) {
            return OPEN_RIGHT;
        } else {
            // 否则，信息不足，继续探测
            return LISTEN;
        }
    }
    // --- 贝叶斯信念更新 ---
    void updateBelief(Action action, Observation obs) {
        std::vector<double> new_belief(2);
        double normalization_const = 0.0;
        // 遍历所有可能的“下一个状态” s_next
        for (int s_next = 0; s_next < 2; ++s_next) {
            double likelihood = getObservationProb(obs, (State)s_next, action);            
            // 计算先验概率 (Prediction): sum(T * b)
            // 在此简单例子中，状态是静态的 (Target不移动)，所以 T(s'|s) = 1 if s'=s else 0
            double prior = belief_[s_next]; 
            new_belief[s_next] = likelihood * prior;
            normalization_const += new_belief[s_next];
        }
        // 归一化
        if (normalization_const > 0.0) {
            for (int i = 0; i < 2; ++i) {
                new_belief[i] /= normalization_const;
            }
        }        
        belief_ = new_belief;
    }
    // --- 观测模型 P(o | s', a) ---
    double getObservationProb(Observation obs, State s, Action a) {
        if (a != LISTEN) return 0.0; // 开门动作没有有意义的观测
        if (s == TARGET_LEFT) {
            if (obs == HEAR_LEFT) return SENSOR_ACCURACY;      // 0.85
            if (obs == HEAR_RIGHT) return 1.0 - SENSOR_ACCURACY; // 0.15
        } else { // TARGET_RIGHT
            if (obs == HEAR_LEFT) return 1.0 - SENSOR_ACCURACY; // 0.15
            if (obs == HEAR_RIGHT) return SENSOR_ACCURACY;      // 0.85
        }
        return 0.5; 
    }
    // --- 模拟真实环境生成观测 ---
    Observation simulateEnvironment(Action action) {
        if (action != LISTEN) return NO_OBS;
        // 生成基于真实状态的随机噪声观测
        std::uniform_real_distribution<> dist(0.0, 1.0);
        double roll = dist(rng_);
        if (true_state_ == TARGET_LEFT) {
            return (roll < SENSOR_ACCURACY) ? HEAR_LEFT : HEAR_RIGHT;
        } else {
            return (roll < SENSOR_ACCURACY) ? HEAR_RIGHT : HEAR_LEFT;
        }
    }
};
int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<POMDPNode>());
    rclcpp::shutdown();
    return 0;
}
```