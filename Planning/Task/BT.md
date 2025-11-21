## 概述
行为树（Behavior Tree, BT）是目前机器人任务规划领域的主流方法，特别是在 ROS2 生态中（如 Nav2 导航栈），它已经基本取代了传统的状态机
## 原理
一种树状的数据结构，用于模拟决策过程，通过 Tick（滴答）机制，从根节点向叶子节点传递执行信号，并根据子节点的返回状态决定下一步的流向
### 状态与Tick
每个节点在被 Tick 唤醒后，必须返回以下三种状态之一
1. SUCCESS (成功): 任务完成
2. FAILURE (失败): 任务无法完成
3. RUNNING (运行中): 任务正在进行（这是BT处理ROS异步行为的关键，如导航到某点）
### 节点
BT 的 逻辑由非叶子节点（控制流）和叶子节点（执行/感知）组成
### 黑板
一个键值对存储区，允许不同节点之间交换数据（例如：感知节点将目标坐标写入黑板，移动节点从黑板读取坐标）
## 节点类型
BT 的 逻辑由非叶子节点（控制流）和叶子节点（执行/感知）组成
### Sequence (序列)	
1. 符号为 ->，AND 逻辑
2. 依次执行子节点。若子节点返回 SUCCESS，则执行下一个；若遇到 FAILURE，立即返回 FAILURE；若遇到 RUNNING，返回 RUNNING。
### Fallback (选择/回退)
1. 符号为 ?，OR 逻辑
2. 依次执行子节点。若子节点返回 FAILURE，则尝试下一个；若遇到 SUCCESS，立即返回 SUCCESS；若遇到 RUNNING，返回 RUNNING。
### Action (动作)
1. 图标为 矩形，叶子节点
2. 执行具体命令（如：移动机器人、打开抓手）
### Condition (条件)
1. 图标为 椭圆，叶子节点
2. 检查系统状态（如：电量是否>20%？），通常立即返回 SUCCESS 或 FAILURE
### Decorator (装饰)
1. 图标为 菱形
2. 修改子节点的结果（如：Inverter 取反，Retry 重试，Repeat 重复）
## 算法步骤
假设有一个简单的机器人任务：“如果电量低则回家充电，否则进行巡逻”
### 树结构设计
Root (Fallback/Selector) ?\
&emsp;&emsp;Sequence -> (充电流程)\
&emsp;&emsp;&emsp;&emsp;Condition: IsBatteryLow (电量低?)\
&emsp;&emsp;&emsp;&emsp;Action: GoHome (回家)\
&emsp;&emsp;Action: Patrol (巡逻)
## 遍历步骤 (每一帧/Tick)
### Tick Root (?)
根节点开始工作。
### Root调用第一个子节点 (Sequence ->)
1. Sequence调用 IsBatteryLow\
（1）电量足: 返回 FAILURE\
（2）Sequence 逻辑: 收到 FAILURE，Sequence 立即向父节点返回 FAILURE
2. Root 收到 FAILURE: 因为 Root 是 Fallback 节点，它继续尝试下一个子节点
3. Root 调用 Patrol：Action，执行巡逻逻辑，返回 RUNNING
4. Root 收到 RUNNING: 整个树向系统返回 RUNNING
### 下一帧
重复上述过程
### 情况变化 (电量低)
1. Sequence调用 IsBatteryLow: 返回 SUCCESS
2. Sequence收到 SUCCESS: 继续调用下一个子节点 GoHome
3. Sequence调用 GoHome: 返回 RUNNING（正在前往）
4. Sequence收到 RUNNING: 向父节点返回 RUNNING
5. Root收到 RUNNING: 停止后续分支（Patrol不会被执行），向系统返回 RUNNING
## 代码示例
### bt_main.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <behaviortree_cpp/bt_factory.h>
#include <behaviortree_cpp/action_node.h>
#include <behaviortree_cpp/condition_node.h>
using namespace BT;

// 1. 定义自定义节点 (Custom Nodes)
// 条件节点：检查电池状态 
// 简单的同步节点，不涉及耗时操作
class IsBatteryLow : public ConditionNode{
public:
    IsBatteryLow(const std::string& name, const NodeConfiguration& config)
        : ConditionNode(name, config) {}
    // 必须定义的静态方法，用于声明端口（这里不需要输入输出）
    static PortsList providedPorts() { return {}; }
    // 执行逻辑
    NodeStatus tick() override {
        // 模拟：这里我们硬编码返回 SUCCESS (假设电量低)
        // 在实际ROS中，这里会读取订阅到的BatteryState话题
        std::cout << "[Condition] Checking Battery... Low!" << std::endl;
        return NodeStatus::SUCCESS; 
    }
};
// 动作节点：回家充电
// SyncActionNode 适用于瞬间完成或不需要异步等待的动作
// 对于真正的机器人移动，通常使用 StatefulActionNode 或 CoroActionNode
class GoHome : public SyncActionNode{
public:
    GoHome(const std::string& name, const NodeConfiguration& config)
        : SyncActionNode(name, config) {}
    static PortsList providedPorts() { return {}; }
    NodeStatus tick() override {
        std::cout << "[Action] Going Home to Recharge..." << std::endl;
        // 模拟动作执行成功
        return NodeStatus::SUCCESS;
    }
};
// 动作节点：巡逻
class Patrol : public SyncActionNode{
public:
    Patrol(const std::string& name, const NodeConfiguration& config)
        : SyncActionNode(name, config) {}
    static PortsList providedPorts() { return {}; }
    NodeStatus tick() override{
        std::cout << "[Action] Patrolling the area..." << std::endl;
        return NodeStatus::SUCCESS;
    }
};
// 2. XML 行为树描述
// 这里使用Fallback (Selector)，如果Sequence失败（电量不低），则执行Patrol
static const char* xml_text = R"(
 <root main_tree_to_execute = "MainTree">
     <BehaviorTree ID="MainTree">
        <Fallback name="root_fallback">
            <Sequence name="recharge_sequence">
                <IsBatteryLow name="battery_check"/>
                <GoHome name="go_home_action"/>
            </Sequence>
            <Patrol name="patrol_action"/>
        </Fallback>
     </BehaviorTree>
 </root>
 )";
class BTNode : public rclcpp::Node{
public:
    BTNode() : Node("bt_runner"){
        // 注册节点
        factory.registerNodeType<IsBatteryLow>("IsBatteryLow");
        factory.registerNodeType<GoHome>("GoHome");
        factory.registerNodeType<Patrol>("Patrol");
        // 从XML创建树
        tree = factory.createTreeFromText(xml_text);
        // 创建定时器，每500ms Tick一次树
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(1000),
            std::bind(&BTNode::update_tree, this));            
        RCLCPP_INFO(this->get_logger(), "Behavior Tree Started.");
    }
private:
    void update_tree()    {
        // 核心：Tick整个树
        NodeStatus status = tree.tickOnce();        
        std::cout << "--- Tree Status: " << toStr(status) << " ---\n" << std::endl;
    }
    BehaviorTreeFactory factory;
    Tree tree;
    rclcpp::TimerBase::SharedPtr timer_;
};
int main(int argc, char **argv){
    rclcpp::init(argc, argv);
    auto node = std::make_shared<BTNode>();
    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.8)
project(bt_example)
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
# 查找 BT 库
find_package(behaviortree_cpp REQUIRED)
add_executable(bt_main src/bt_main.cpp)
ament_target_dependencies(bt_main rclcpp behaviortree_cpp)
# 确保C++17标准（BT.CPP v4需要）
target_compile_features(bt_main PUBLIC cxx_std_17)
install(TARGETS bt_main
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```