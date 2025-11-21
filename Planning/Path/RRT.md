## 概述
1. RRT（Rapidly-exploring Random Trees，快速探索随机树），尤其擅长处理高维度和复杂约束空间，因此在 3D 环境中应用非常普遍
2. 一种基于采样的增量式路径规划算法，不试图搜索整个空间，而是快速地在未探索区域建立一个搜索树，从而高效地找到一条（通常不是最优的）可行路径
## 核心思想
### Voronoi 偏向
RRT 算法的快速探索特性来源于它的扩展方式，这被称为 Voronoi 偏向
1. 在空间中，未探索区域越大，随机采样点落在该区域的概率就越大
2. 树上的节点会试图向这些随机点延伸
3. 结果树会迅速填满空白空间，而不是在一个角落里打转
### 关键术语
1. $C_{free}$: 无障碍物的自由空间。$C_{obs}$: 障碍物空间
2. $q_{start}$ / $q_{goal}$: 起点和终点
3. $q_{rand}$: 随机采样点
4. $q_{near}$: 树中距离 $q_{rand}$ 最近的节点
5. $q_{new}$: 从 $q_{near}$ 向 $q_{rand}$ 延伸一步后产生的新节点
## 算法步骤
RRT的算法流程非常简洁，在 3D 空间中计算量主要集中在最近邻搜索和碰撞检测上
### 随机采样 — 生成 $q_{rand}$
1. 在三维空间 $C_{space}$ 中随机一个点，作为树生长的指引方向。
2. 为了防止完全随机采样导致算法效率过低，引入一个概率 $P_{bias}$（通常 0.05 - 0.1）\
（1）生成一个 0 到 1 的随机数 $r$\
（2）如果 $r < P_{bias}$，直接令 $q_{rand} = q_{goal}$（强制向终点生长），否则在空间内全随机采样\
（3）如果 $r$ 太大，树会只顾着往终点冲，一旦终点方向有墙（局部极小值），算法就会卡死在墙边，丧失绕路的能力
### 寻找最近邻 — 确定 $q_{near}$
1. 找到当前树 $T$ 中，距离 $q_{rand}$ 最近的节点 $q_{near}$
2. 在 3D 空间通常使用欧几里得距离，需要遍历树中所有的节点 $V$，找到：
$$ \\ q\_{near} = \arg\min\_{v \in V} dist(v, q\_{rand})$$
3. 最耗时的步骤之一，使用 KD-Tree 数据结构进行优化，可以将搜索复杂度降低到接近 $O(\log N)$
4. 在 C++ 中通常使用 FLANN 库或 PCL 里的 KD-Tree 实现
### 扩展 — 计算 $q_{new}$
1. 沿着向量 $\vec{v} = q_{rand} - q_{near}$ 的方向，截取固定步长 $\delta$ (step size)，生成新节点 $q_{new}$
2. 首先计算方向向量 $\vec{d} = q_{rand} - q_{near}$ 和距离 $L = \| \vec{d} \|$
3. 判断逻辑：\
（1）$L \le \delta$: 目标点很近，直接一步到位，$q_{new} = q_{rand}$\
（2）$L > \delta$: 目标点太远，需要截断
$$q_{new} = q_{near} + \delta \cdot \frac{q_{rand} - q_{near}}{\| q_{rand} - q_{near} \|}$$
4. $\delta$ 太小会生长极慢，计算量爆炸；太大容易穿过狭窄的障碍物缝隙
### 碰撞检测
1. 判断从 $q_{near}$ 到 $q_{new}$ 的连线是否安全，仅仅检测 $q_{new}$ 这一个点是不够的，必须检测整个线段
2. 将线段离散化为多个点进行检测，假设检测分辨率为 $res$（例如 0.05米）
$$ \\ N\_{steps} = \frac{dist(q\_{near}, q\_{new})}{res}
$$
3. 循环 $i$ 从 1 到 $N_{steps}$
$$ \\ p\_{check} = q\_{near} + i \cdot res \cdot \vec{unit\_vector}
$$
4. 如果 Map.isOccupied(p_check) 为真，则判定碰撞，舍弃 $q_{new}$，本次循环结束
### 更新树
1. 如果 $q_{new}$ 通过了碰撞检测，它就正式成为树的一部分
2. 数据操作\
（1）节点列表：Nodes.push_back(q_new)\
（2）父子关系：至关重要，必须记录 $q_{new}$ 是从谁长出来的，q_new.parent = &q_near\
（3）边：在可视化时，画一条线连接 $q_{near}$ 和 $q_{new}$
### 收敛判定
每次生成 $q_{new}$ 后，计算它与终点 $q_{goal}$ 的距离
$$ \\ dist(q\_{new}, q\_{goal}) < \text{Threshold} $$
如果小于阈值（例如 0.5米），则认为路径找到了。通常为了确保最后一段连通，还会额外做一次从 $q_{new}$ 直连 $q_{goal}$ 的碰撞检测
### 路径回溯
1. 初始化列表 Path = [q_goal]
2. 当前节点 curr = q_new
3. 循环 while (curr != q_start):\
Path.insert(0, curr) (插在最前面)\
curr = curr.parent
4. Path.insert(0, q_start)
5. 输出：得到从 Start 到 Goal 的有序点集
## 代码示例
工业界标准做法是使用 OMPL，以下仅为手写演示代码
### 结构
```Plaintext
rrt_octomap_planner/
├── CMakeLists.txt
├── package.xml
├── include/
│   └── rrt_octomap_planner/
│       └── rrt_planner.hpp
└── src/
    └── rrt_node.cpp
```
### package.xml
```XML
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
    <name>rrt_octomap_planner</name>
    <version>0.0.1</version>
    <description>Hand-written RRT planner with Octomap collision checking</description>
    <maintainer email="user@todo.todo">User</maintainer>
    <license>MIT</license>

    <buildtool_depend>ament_cmake</buildtool_depend>

    <depend>rclcpp</depend>
    <depend>geometry_msgs</depend>
    <depend>visualization_msgs</depend>
    <depend>octomap_msgs</depend>
    <depend>liboctomap-dev</depend> <test_depend>ament_lint_auto</test_depend>
    <test_depend>ament_lint_common</test_depend>

    <export>
        <build_type>ament_cmake</build_type>
    </export>
</package>
```
### CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.8)
project(rrt_octomap_planner)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(visualization_msgs REQUIRED)
find_package(octomap_msgs REQUIRED)
find_package(octomap REQUIRED) # 查找系统 Octomap

include_directories(include)

add_executable(rrt_planner_node src/rrt_node.cpp)
ament_target_dependencies(rrt_planner_node
  rclcpp
  geometry_msgs
  visualization_msgs
  octomap_msgs
)

# 单独链接 octomap 库 (因为它是非 ROS 库)
target_link_libraries(rrt_planner_node ${OCTOMAP_LIBRARIES})
install(TARGETS rrt_planner_node
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```
### rrt_planner.hpp
```C++
#ifndef RRT_PLANNER_HPP
#define RRT_PLANNER_HPP

#include <vector>
#include <cmath>
#include <random>
#include <memory>
#include <iostream>
#include <octomap/octomap.h>
#include <octomap/OcTree.h>

struct Point3D {
    double x, y, z;
};
struct Node {
    Point3D point;
    int parent_idx; // -1 表示根节点

    Node(Point3D p, int parent) : point(p), parent_idx(parent) {}
};
class RRTPlanner {
public:
    RRTPlanner() : gen_(rd_()) {}
    // 设置 Octomap 指针 (由 ROS 回调函数更新)
    void setMap(std::shared_ptr<octomap::OcTree> map) {
        map_ = map;
    }
    // 设置参数
    void setParams(Point3D start, Point3D goal, double step_size, int max_iter, double goal_thresh, double x_min, double x_max, double y_min, double y_max, double z_min, double z_max) {
        start_ = start;
        goal_ = goal;
        step_size_ = step_size;
        max_iter_ = max_iter;
        goal_threshold_ = goal_thresh;
        bounds_min_ = {x_min, y_min, z_min};
        bounds_max_ = {x_max, y_max, z_max};
    }
    // RRT 主算法
    bool plan(std::vector<Point3D>& out_path, std::vector<Node>& out_tree) {
        if (!map_) {
            std::cerr << "[RRT] Error: No Octomap received yet!" << std::endl;
            return false;
        }
        nodes_.clear();
        nodes_.emplace_back(start_, -1);
        for (int i = 0; i < max_iter_; ++i) {
            // 1. Sampling with Goal Bias
            Point3D q_rand = sample();
            // 2. Nearest Neighbor
            int near_idx = getNearestIdx(q_rand);
            Point3D q_near = nodes_[near_idx].point;
            // 3. Steer
            Point3D q_new = steer(q_near, q_rand);
            // 4. Collision Check (Edge check)
            if (!checkCollision(q_near, q_new)) {
                // 5. Add to Tree
                nodes_.emplace_back(q_new, near_idx);
                // 6. Check Goal
                if (dist(q_new, goal_) < goal_threshold_) {
                    if (!checkCollision(q_new, goal_)) {
                        nodes_.emplace_back(goal_, nodes_.size() - 1);
                        out_tree = nodes_;
                        out_path = reconstructPath();
                        return true;
                    }
                }
            }
        }
        out_tree = nodes_; // 即使失败也返回树用于调试
        return false;
    }
private:
    std::shared_ptr<octomap::OcTree> map_;
    std::vector<Node> nodes_;
    Point3D start_, goal_;
    Point3D bounds_min_, bounds_max_;
    double step_size_;
    int max_iter_;
    double goal_threshold_;
    std::random_device rd_;
    std::mt19937 gen_;
    // 1. 采样函数
    Point3D sample() {
        std::uniform_real_distribution<> dis(0.0, 1.0);
        // 10% 概率直接选目标点
        if (dis(gen_) < 0.1) return goal_;
        std::uniform_real_distribution<> x_dis(bounds_min_.x, bounds_max_.x);
        std::uniform_real_distribution<> y_dis(bounds_min_.y, bounds_max_.y);
        std::uniform_real_distribution<> z_dis(bounds_min_.z, bounds_max_.z);
        return {x_dis(gen_), y_dis(gen_), z_dis(gen_)};
    }
    // 2. 寻找最近邻 (暴力搜索 O(N))
    int getNearestIdx(const Point3D& q_rand) {
        int nearest = -1;
        double min_dist = std::numeric_limits<double>::max();
        for (size_t i = 0; i < nodes_.size(); ++i) {
            double d = dist(nodes_[i].point, q_rand);
            if (d < min_dist) {
                min_dist = d;
                nearest = i;
            }
        }
        return nearest;
    }
    // 3. 扩展
    Point3D steer(const Point3D& from, const Point3D& to) {
        double d = dist(from, to);
        if (d < step_size_) return to;
        double scale = step_size_ / d;
        return {
            from.x + (to.x - from.x) * scale,
            from.y + (to.y - from.y) * scale,
            from.z + (to.z - from.z) * scale
        };
    }
    // 4. 碰撞检测 (核心：基于 Octomap)
    bool checkCollision(const Point3D& p1, const Point3D& p2) {
        // 使用八叉树的分辨率进行步进检测
        double resolution = map_->getResolution();
        double d = dist(p1, p2);
        int steps = std::ceil(d / resolution);
        for (int i = 0; i <= steps; ++i) {
            double t = (double)i / steps;
            double x = p1.x + (p2.x - p1.x) * t;
            double y = p1.y + (p2.y - p1.y) * t;
            double z = p1.z + (p2.z - p1.z) * t;
            octomap::OcTreeNode* node = map_->search(x, y, z);
            if (node) {
                // 如果节点存在且被占据，则碰撞
                if (map_->isNodeOccupied(node)) return true;
            } else {
                // 节点为 NULL 表示未知区域，保守策略视为障碍，乐观策略视为自由
                // 这里假设未知区域不安全
                return true; 
            }
        }
        return false;
    }
    // 辅助：欧氏距离
    double dist(const Point3D& a, const Point3D& b) {
        return std::sqrt(std::pow(a.x - b.x, 2) + std::pow(a.y - b.y, 2) + std::pow(a.z - b.z, 2));
    }
    // 回溯路径
    std::vector<Point3D> reconstructPath() {
        std::vector<Point3D> path;
        int curr = nodes_.size() - 1;
        while (curr != -1) {
            path.push_back(nodes_[curr].point);
            curr = nodes_[curr].parent_idx;
        }
        std::reverse(path.begin(), path.end());
        return path;
    }
};
#endif
```
### rrt_node.cpp
负责 ROS 通信：接收地图，调用 RRT，发布可视化
```C++
#include <rclcpp/rclcpp.hpp>
#include <octomap_msgs/msg/octomap.hpp>
#include <octomap_msgs/conversions.h>
#include <visualization_msgs/msg/marker.hpp>
#include <visualization_msgs/msg/marker_array.hpp>
#include <nav_msgs/msg/path.hpp>
#include "rrt_octomap_planner/rrt_planner.hpp"

using std::placeholders::_1;

class RRTNode : public rclcpp::Node {
public:
    RRTNode() : Node("rrt_octomap_node") {
        // 订阅 Octomap
        octomap_sub_ = this->create_subscription<octomap_msgs::msg::Octomap>(
            "/octomap_full", 1, std::bind(&RRTNode::mapCallback, this, _1));
        // 发布者
        path_pub_ = this->create_publisher<nav_msgs::msg::Path>("rrt_path", 10);
        tree_pub_ = this->create_publisher<visualization_msgs::msg::Marker>("rrt_tree", 10);
        // 初始化规划器
        planner_ = std::make_shared<RRTPlanner>();
        RCLCPP_INFO(this->get_logger(), "RRT Node Started. Waiting for Octomap...");
    }
private:
    void mapCallback(const octomap_msgs::msg::Octomap::SharedPtr msg) {
        RCLCPP_INFO(this->get_logger(), "Octomap received!");
        // 1. 转换 ROS 消息 -> Octomap 对象
        // 注意：这里需要 delete，建议使用智能指针封装，但 conversions.h 返回原生指针
        octomap::AbstractOcTree* tree = octomap_msgs::msgToMap(*msg);
        octomap::OcTree* octree = dynamic_cast<octomap::OcTree*>(tree);
        if (!octree) {
            RCLCPP_ERROR(this->get_logger(), "Failed to cast to OcTree");
            return;
        }
        // 2. 配置规划器 (示例：从原点飞到 (5, 5, 2))
        std::shared_ptr<octomap::OcTree> map_ptr(octree); // 接管指针所有权
        planner_->setMap(map_ptr);
        Point3D start = {0.0, 0.0, 1.0};
        Point3D goal = {5.0, 5.0, 2.0};
        // 范围: x(-10,10), y(-10,10), z(0, 5)
        planner_->setParams(start, goal, 0.5, 5000, 0.5, -10, 10, -10, 10, 0, 5);
        // 3. 执行规划
        std::vector<Point3D> path;
        std::vector<Node> tree_nodes;
        auto start_time = this->now();
        bool success = planner_->plan(path, tree_nodes);
        auto duration = (this->now() - start_time).seconds();
        // 4. 可视化
        publishTree(tree_nodes); // 总是画树，方便调试
        if (success) {
            RCLCPP_INFO(this->get_logger(), "Path Found! Size: %zu, Time: %.3fs", path.size(), duration);
            publishPath(path);
        } else {
            RCLCPP_WARN(this->get_logger(), "RRT Failed to find path after %d iterations", 5000);
        }
    }
    void publishPath(const std::vector<Point3D>& path) {
        nav_msgs::msg::Path path_msg;
        path_msg.header.frame_id = "map"; // 假设 Octomap 也是 map 坐标系
        path_msg.header.stamp = this->now();
        for (const auto& p : path) {
            geometry_msgs::msg::PoseStamped pose;
            pose.pose.position.x = p.x;
            pose.pose.position.y = p.y;
            pose.pose.position.z = p.z;
            pose.pose.orientation.w = 1.0;
            path_msg.poses.push_back(pose);
        }
        path_pub_->publish(path_msg);
    }
    void publishTree(const std::vector<Node>& nodes) {
        visualization_msgs::msg::Marker marker;
        marker.header.frame_id = "map";
        marker.header.stamp = this->now();
        marker.ns = "rrt_tree";
        marker.id = 0;
        marker.type = visualization_msgs::msg::Marker::LINE_LIST;
        marker.action = visualization_msgs::msg::Marker::ADD;
        marker.scale.x = 0.02; // 线宽
        marker.color.r = 0.0; marker.color.g = 1.0; marker.color.b = 0.0; marker.color.a = 0.5;
        for (const auto& node : nodes) {
            if (node.parent_idx != -1) {
                const auto& parent = nodes[node.parent_idx];
                geometry_msgs::msg::Point p1, p2;
                p1.x = node.point.x; p1.y = node.point.y; p1.z = node.point.z;
                p2.x = parent.point.x; p2.y = parent.point.y; p2.z = parent.point.z;
                marker.points.push_back(p1);
                marker.points.push_back(p2);
            }
        }
        tree_pub_->publish(marker);
    }
    rclcpp::Subscription<octomap_msgs::msg::Octomap>::SharedPtr octomap_sub_;
    rclcpp::Publisher<nav_msgs::msg::Path>::SharedPtr path_pub_;
    rclcpp::Publisher<visualization_msgs::msg::Marker>::SharedPtr tree_pub_;
    std::shared_ptr<RRTPlanner> planner_;
};

int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<RRTNode>());
    rclcpp::shutdown();
    return 0;
}
```