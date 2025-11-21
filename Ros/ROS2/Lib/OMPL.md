## 概述
目前最流行的基于采样的运动规划库，通常是 MoveIt 2 的核心规划后端，也可以独立使用来处理自定义的规划任务（例如非机械臂的路径规划）
## 核心概念
### 自定义项
本身是纯粹的规划算法库，不包含碰撞检测库（如 FCL）和可视化工具，依赖用户定义以下内容
1. 状态空间：机器人的配置空间（例如：二维平面 $R^2$，三维空间 $R^3$，或者机械臂的关节空间 $R^n$）
2. 状态有效性检查器：一个函数，输入一个状态，返回 true (无碰撞/合法) 或 false (碰撞/非法)
3. 规划器：算法本身（如 RRT, RRT*, PRM, RRT-Connect 等）
### ROS2 中的架构
1. MoveIt 2 方式: ROS 2 -> MoveIt 2 (封装了碰撞检测) -> OMPL Plugin -> 轨迹
2. Standalone 方式 (本示例): ROS 2 Node -> 直接调用 OMPL API -> 自定义碰撞检测 -> 发布 Path 消息 -> Rviz2 显示
## 示例
### ompl_planner_node.cpp
使用 OMPL 的 RRT-Connect 算法，在一个 2D 平面上规划从起点到终点的路径，并绕过中间的一个圆形障碍物
```C++
#include <rclcpp/rclcpp.hpp>
#include <visualization_msgs/msg/marker.hpp>
#include <geometry_msgs/msg/point.hpp>
#include <ompl/base/SpaceInformation.h>
#include <ompl/base/spaces/RealVectorStateSpace.h>
#include <ompl/geometric/SimpleSetup.h>
#include <ompl/geometric/planners/rrt/RRTConnect.h>
#include <memory>
#include <cmath>

namespace ob = ompl::base;
namespace og = ompl::geometric;

class OmplPlannerNode : public rclcpp::Node{
public:
    OmplPlannerNode() : Node("ompl_planner_node"){
        // 创建发布者用于在 Rviz 中显示路径
        marker_pub_ = this->create_publisher<visualization_msgs::msg::Marker>("planned_path", 10);
        // 延时一秒执行规划，确保 Rviz 有时间连接
        timer_ = this->create_wall_timer(
            std::chrono::seconds(1), 
            std::bind(&OmplPlannerNode::plan, this));
    }

private:
    rclcpp::Publisher<visualization_msgs::msg::Marker>::SharedPtr marker_pub_;
    rclcpp::TimerBase::SharedPtr timer_;
    // --- 1. 定义状态有效性检查器 (碰撞检测) ---
    // 如果状态位于中心 (0,0) 半径为 2.0 的圆内，则视为无效（碰撞）
    bool isStateValid(const ob::State *state){
        const auto *pos = state->as<ob::RealVectorStateSpace::StateType>();
        double x = pos->values[0];
        double y = pos->values[1];
        if (sqrt(x*x + y*y) < 2.0) {
            return false; 
        }
        return true;
    }
    void plan() {
        this->timer_->cancel(); // 只执行一次
        RCLCPP_INFO(this->get_logger(), "Starting OMPL planning...");
        // --- 2. 构建状态空间 (2D 平面) ---
        auto space = std::make_shared<ob::RealVectorStateSpace>(2);
        // 设置边界 (-5 到 5)
        ob::RealVectorBounds bounds(2);
        bounds.setLow(-5.0);
        bounds.setHigh(5.0);
        space->setBounds(bounds);
        // --- 3. 实例化 SpaceInformation 并设置碰撞检测 ---
        auto si = std::make_shared<ob::SpaceInformation>(space);
        // 使用 bind 绑定成员函数或 lambda，这里为了简单使用了 lambda 包装
        si->setStateValidityChecker([this](const ob::State *state) { 
            return this->isStateValid(state); 
        });
        // --- 4. 定义起点和终点 ---
        ob::ScopedState<> start(space);
        start->values[0] = -4.0;
        start->values[1] = -4.0;
        ob::ScopedState<> goal(space);
        goal->values[0] = 4.0;
        goal->values[1] = 4.0;
        // --- 5. 创建 ProblemDefinition ---
        auto pdef = std::make_shared<ob::ProblemDefinition>(si);
        pdef->setStartAndGoalStates(start, goal);
        // --- 6. 选择规划算法 (RRT-Connect) ---
        auto planner = std::make_shared<og::RRTConnect>(si);
        planner->setProblemDefinition(pdef);
        planner->setup();
        // --- 7. 求解 ---
        ob::PlannerStatus solved = planner->solve(1.0); // 给予 1秒 的计算时间
        if (solved){
            RCLCPP_INFO(this->get_logger(), "Found solution!");
            // 获取路径
            auto path = pdef->getSolutionPath()->as<og::PathGeometric>();
            // 平滑路径 (可选，OMPL 自带简化器)
            og::PathSimplifier simplifier(si);
            simplifier.simplifyMax(*path);
            // 插值，使路径点更密集，方便可视化
            path->interpolate(50);
            publishPath(path);
        }
        else{
            RCLCPP_WARN(this->get_logger(), "No solution found.");
        }
    }
    // 辅助函数：将 OMPL 路径转换为 ROS Marker 并发布
    void publishPath(og::PathGeometric* path){
        visualization_msgs::msg::Marker marker;
        marker.header.frame_id = "map";
        marker.header.stamp = this->get_clock()->now();
        marker.ns = "ompl";
        marker.id = 0;
        marker.type = visualization_msgs::msg::Marker::LINE_STRIP;
        marker.action = visualization_msgs::msg::Marker::ADD;
        marker.scale.x = 0.05; // 线宽
        marker.color.a = 1.0;
        marker.color.r = 0.0;
        marker.color.g = 1.0;
        marker.color.b = 0.0;
        for (std::size_t i = 0; i < path->getStateCount(); ++i){
            const auto *state = path->getState(i)->as<ob::RealVectorStateSpace::StateType>();
            geometry_msgs::msg::Point p;
            p.x = state->values[0];
            p.y = state->values[1];
            p.z = 0.0;
            marker.points.push_back(p);
        }
        marker_pub_->publish(marker);
        RCLCPP_INFO(this->get_logger(), "Path published to Rviz.");
    }
};

int main(int argc, char **argv){
    rclcpp::init(argc, argv);
    auto node = std::make_shared<OmplPlannerNode>();
    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.8)
project(ompl_example_cpp)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(visualization_msgs REQUIRED)
# 查找 OMPL
# 方法1: 使用 pkg-config (ROS 2 推荐方式)
find_package(PkgConfig REQUIRED)
pkg_check_modules(OMPL REQUIRED ompl)
# 方法2: 如果 pkg-config 找不到，可以尝试直接 find_package(ompl)
# find_package(ompl REQUIRED)

add_executable(ompl_planner_node src/ompl_planner_node.cpp)
target_include_directories(ompl_planner_node PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
  ${OMPL_INCLUDE_DIRS}  # 包含 OMPL 头文件路径
)
ament_target_dependencies(ompl_planner_node
  rclcpp
  visualization_msgs
)
# 链接 OMPL 库
target_link_libraries(ompl_planner_node
  ${OMPL_LIBRARIES}
)
install(TARGETS ompl_planner_node
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```
### package.xml
在 package.xml 中添加依赖，虽然对编译不是必须的，但对包管理很重要
```XML
<depend>ompl</depend>
```