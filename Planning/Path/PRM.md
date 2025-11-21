## 概述
1. 概率路图法（Probabilistic Roadmap Method）是一种基于采样的经典路径规划算法，特别适用于高维空间（如3D环境或多自由度机械臂）和复杂静态环境
2. 核心思想是将连续的构型空间转化为离散的图，它通过在空间中随机撒点（采样），并尝试连接这些点，构建出一个路线图。
3. 一旦路线图构建完成，寻找路径就变成了在图中搜索最短路径的问题（如使用 Dijkstra 或 A*）。
4. 算法通常分为 学习/构建 和 查询 阶段
## 关键概念
### 自由空间 $C_{free}$ 
机器人可以移动且不与障碍物碰撞的空间
### 局部规划器
一个简单的函数，用于判断两点之间的连线是否穿越障碍物（通常使用直线插值检测）
### 概率完备性
如果存在一条解路径，随着采样点数量 $N \to \infty$，PRM 找到该路径的概率趋向于 1
## 算法步骤
### 采样
随机在地图范围内生成 $N$ 个点，对于每个采样点 $q_{rand}$
1. 检查它是否位于障碍物内（碰撞检测）
2. 如果在 $C_{free}$ 中，将其加入顶点集 $V$
### 近邻搜索
对于图中的每一个节点 $q_i$
1. 寻找它周围距离小于半径 $r$ 的所有邻居节点 $\{n_1, n_2, ...\}$
2. 或者寻找最近的 $k$ 个邻居
### 构建边
对于每一对邻居 $(q_i, q_j)$
1. 调用局部规划器（通常是碰撞检测函数）
2. 如果 $q_i$ 到 $q_j$ 的直线路径上没有障碍物，则在它们之间添加一条边 $E_{ij}$，权重通常为欧几里得距离
### 查询与搜索
1. 将实际的 起点 ($S$) 和 终点 ($G$) 加入图中
2. 尝试将 $S$ 和 $G$ 连接到图中距离最近且无碰撞的现有节点
3. 使用图搜索算法（如 A* 或 Dijkstra）在图上寻找从 $S$ 到 $G$ 的最短路径
## 代码示例
### 结构
```Plaintext
prm_planner_cpp/
├── CMakeLists.txt
├── package.xml
└── src
    └── simple_prm_3d.cpp
```
### prm_3d.cpp
```C++
#include <rclcpp/rclcpp.hpp>
#include <visualization_msgs/msg/marker.hpp>
#include <visualization_msgs/msg/marker_array.hpp>
#include <geometry_msgs/msg/point.hpp>
#include <cmath>
#include <vector>
#include <random>
#include <limits>
#include <queue>

struct Node {
    int id;
    double x, y, z;
    std::vector<int> neighbors; // 存储邻居节点的ID
    std::vector<double> costs;  // 存储到邻居的距离
};

class SimplePRM3D : public rclcpp::Node {
public:
    SimplePRM3D() : Node("simple_prm_3d") {
        // 初始化发布者
        marker_pub_ = this->create_publisher<visualization_msgs::msg::MarkerArray>("prm_viz", 10);
        // 参数设置
        num_samples_ = 500;      // 采样点数量
        connection_radius_ = 2.0; // 连接半径
        map_size_ = 10.0;        // 地图范围 (-5 到 5)
        // 设置起点和终点
        start_node_ = { -1, -4.0, -4.0, -4.0, {}, {} };
        goal_node_ = { -1, 4.0, 4.0, 4.0, {}, {} };
        RCLCPP_INFO(this->get_logger(), "Starting PRM Planning...");
        // 1. 采样
        sampleConfigurationSpace();
        // 2. 把起点终点加入并连接
        addStartAndGoal();
        // 3. 搜索路径
        std::vector<int> path_ids = performDijkstra();
        // 4. 可视化
        publishVisualization(path_ids);
    }
private:
    rclcpp::Publisher<visualization_msgs::msg::MarkerArray>::SharedPtr marker_pub_;
    std::vector<Node> nodes_;
    int num_samples_;
    double connection_radius_;
    double map_size_;
    Node start_node_;
    Node goal_node_;
    // 模拟碰撞检测：假设场景中心有一个半径为 2.0 的球体障碍物
    bool isStateValid(double x, double y, double z) {
        double dist = std::sqrt(x*x + y*y + z*z);
        if (dist < 2.0) return false; 
        if (std::abs(x) > map_size_/2 || std::abs(y) > map_size_/2 || std::abs(z) > map_size_/2) return false;
        return true;
    }
    // 局部规划器：检查两点连线是否经过障碍物 (离散化检测)
    bool checkConnection(const Node& n1, const Node& n2) {
        double dist = std::sqrt(pow(n1.x - n2.x, 2) + pow(n1.y - n2.y, 2) + pow(n1.z - n2.z, 2));
        int steps = dist / 0.1; // 每 0.1m 检查一次
        for (int i = 0; i <= steps; i++) {
            double t = (double)i / steps;
            double ix = n1.x + (n2.x - n1.x) * t;
            double iy = n1.y + (n2.y - n1.y) * t;
            double iz = n1.z + (n2.z - n1.z) * t;
            if (!isStateValid(ix, iy, iz)) return false;
        }
        return true;
    }
    // 第一阶段：构建路图
    void sampleConfigurationSpace() {
        std::random_device rd;
        std::mt19937 gen(rd());
        std::uniform_real_distribution<> dis(-map_size_/2.0, map_size_/2.0);
        int valid_count = 0;
        while (valid_count < num_samples_) {
            double x = dis(gen);
            double y = dis(gen);
            double z = dis(gen);
            if (isStateValid(x, y, z)) {
                Node new_node;
                new_node.id = valid_count;
                new_node.x = x;
                new_node.y = y;
                new_node.z = z;
                nodes_.push_back(new_node);
                valid_count++;
            }
        }
        // 构建边 (暴力搜索，复杂度 O(N^2)，实际应用应用 KD-Tree 优化)
        for (size_t i = 0; i < nodes_.size(); i++) {
            for (size_t j = i + 1; j < nodes_.size(); j++) {
                double dist = std::sqrt(pow(nodes_[i].x - nodes_[j].x, 2) + 
                                        pow(nodes_[i].y - nodes_[j].y, 2) + 
                                        pow(nodes_[i].z - nodes_[j].z, 2));
                if (dist <= connection_radius_) {
                    if (checkConnection(nodes_[i], nodes_[j])) {
                        // 无向图，双向添加
                        nodes_[i].neighbors.push_back(nodes_[j].id);
                        nodes_[i].costs.push_back(dist);
                        nodes_[j].neighbors.push_back(nodes_[i].id);
                        nodes_[j].costs.push_back(dist);
                    }
                }
            }
        }
        RCLCPP_INFO(this->get_logger(), "Roadmap constructed with %zu nodes.", nodes_.size());
    }
    // 将起点和终点接入最近的节点
    void addStartAndGoal() {
        // 处理起点
        if(!isStateValid(start_node_.x, start_node_.y, start_node_.z)) {
            RCLCPP_ERROR(this->get_logger(), "Start is invalid!"); return;
        }
        start_node_.id = nodes_.size(); 
        connectNodeToGraph(start_node_);
        nodes_.push_back(start_node_);
        // 处理终点
        if(!isStateValid(goal_node_.x, goal_node_.y, goal_node_.z)) {
            RCLCPP_ERROR(this->get_logger(), "Goal is invalid!"); return;
        }
        goal_node_.id = nodes_.size();
        connectNodeToGraph(goal_node_);
        nodes_.push_back(goal_node_);
    }
    void connectNodeToGraph(Node& node) {
        // 寻找最近的 K 个邻居或者半径内的邻居进行连接，简化为连接半径内的所有节点
        for (auto& other : nodes_) {
            double dist = std::sqrt(pow(node.x - other.x, 2) + 
                                     pow(node.y - other.y, 2) + 
                                     pow(node.z - other.z, 2));
            if (dist <= connection_radius_ && checkConnection(node, other)) {
                node.neighbors.push_back(other.id);
                node.costs.push_back(dist);
                other.neighbors.push_back(node.id);
                other.costs.push_back(dist);
            }
        }
    }
    // 第二阶段：Dijkstra 搜索
    std::vector<int> performDijkstra() {
        int start_id = start_node_.id;
        int goal_id = goal_node_.id;
        size_t n = nodes_.size();
        std::vector<double> dist(n, std::numeric_limits<double>::max());
        std::vector<int> parent(n, -1);
        std::priority_queue<std::pair<double, int>, std::vector<std::pair<double, int>>, std::greater<std::pair<double, int>>> pq;

        dist[start_id] = 0.0;
        pq.push({0.0, start_id});
        while(!pq.empty()) {
            double d = pq.top().first;
            int u = pq.top().second;
            pq.pop();
            if (d > dist[u]) continue;
            if (u == goal_id) break;
            for (size_t i = 0; i < nodes_[u].neighbors.size(); i++) {
                int v = nodes_[u].neighbors[i];
                double weight = nodes_[u].costs[i];
                if (dist[u] + weight < dist[v]) {
                    dist[v] = dist[u] + weight;
                    parent[v] = u;
                    pq.push({dist[v], v});
                }
            }
        }
        // 重建路径
        std::vector<int> path;
        if (dist[goal_id] == std::numeric_limits<double>::max()) {
            RCLCPP_WARN(this->get_logger(), "No path found!");
            return path;
        }
        for (int v = goal_id; v != -1; v = parent[v]) {
            path.push_back(v);
        }
        std::reverse(path.begin(), path.end());
        RCLCPP_INFO(this->get_logger(), "Path found with %zu steps.", path.size());
        return path;
    }
    // --- 可视化部分 ---
    void publishVisualization(const std::vector<int>& path_ids) {
        visualization_msgs::msg::MarkerArray marker_array;
        rclcpp::Time now = this->now();
        // 1. 障碍物 (红色球体)
        visualization_msgs::msg::Marker obs_marker;
        obs_marker.header.frame_id = "map";
        obs_marker.header.stamp = now;
        obs_marker.ns = "obstacle";
        obs_marker.id = 0;
        obs_marker.type = visualization_msgs::msg::Marker::SPHERE;
        obs_marker.action = visualization_msgs::msg::Marker::ADD;
        obs_marker.pose.position.x = 0; obs_marker.pose.position.y = 0; obs_marker.pose.position.z = 0;
        obs_marker.scale.x = 4.0; obs_marker.scale.y = 4.0; obs_marker.scale.z = 4.0; // 直径 = 2*半径
        obs_marker.color.a = 0.5; obs_marker.color.r = 1.0;
        marker_array.markers.push_back(obs_marker);
        // 2. 采样点 (蓝色小点)
        visualization_msgs::msg::Marker nodes_marker;
        nodes_marker.header.frame_id = "map";
        nodes_marker.header.stamp = now;
        nodes_marker.ns = "nodes";
        nodes_marker.id = 1;
        nodes_marker.type = visualization_msgs::msg::Marker::SPHERE_LIST;
        nodes_marker.action = visualization_msgs::msg::Marker::ADD;
        nodes_marker.scale.x = 0.1; nodes_marker.scale.y = 0.1; nodes_marker.scale.z = 0.1;
        nodes_marker.color.a = 1.0; nodes_marker.color.b = 1.0;
        for(const auto& node : nodes_) {
            geometry_msgs::msg::Point p;
            p.x = node.x; p.y = node.y; p.z = node.z;
            nodes_marker.points.push_back(p);
        }
        marker_array.markers.push_back(nodes_marker);
        // 3. 路图连线 (灰色细线)
        visualization_msgs::msg::Marker edges_marker;
        edges_marker.header.frame_id = "map";
        edges_marker.header.stamp = now;
        edges_marker.ns = "edges";
        edges_marker.id = 2;
        edges_marker.type = visualization_msgs::msg::Marker::LINE_LIST;
        edges_marker.action = visualization_msgs::msg::Marker::ADD;
        edges_marker.scale.x = 0.02; 
        edges_marker.color.a = 0.4; edges_marker.color.r = 0.8; edges_marker.color.g = 0.8; edges_marker.color.b = 0.8;
        for(const auto& node : nodes_) {
            for(int neighbor_id : node.neighbors) {
                // 为避免重复画线，只画 id 小到大的
                if (node.id < neighbor_id) {
                    geometry_msgs::msg::Point p1, p2;
                    p1.x = node.x; p1.y = node.y; p1.z = node.z;
                    p2.x = nodes_[neighbor_id].x; p2.y = nodes_[neighbor_id].y; p2.z = nodes_[neighbor_id].z;
                    edges_marker.points.push_back(p1);
                    edges_marker.points.push_back(p2);
                }
            }
        }
        marker_array.markers.push_back(edges_marker);
        // 4. 最终路径 (绿色粗线)
        if (!path_ids.empty()) {
            visualization_msgs::msg::Marker path_marker;
            path_marker.header.frame_id = "map";
            path_marker.header.stamp = now;
            path_marker.ns = "path";
            path_marker.id = 3;
            path_marker.type = visualization_msgs::msg::Marker::LINE_STRIP;
            path_marker.action = visualization_msgs::msg::Marker::ADD;
            path_marker.scale.x = 0.15; 
            path_marker.color.a = 1.0; path_marker.color.g = 1.0;
            for(int id : path_ids) {
                geometry_msgs::msg::Point p;
                p.x = nodes_[id].x; p.y = nodes_[id].y; p.z = nodes_[id].z;
                path_marker.points.push_back(p);
            }
            marker_array.markers.push_back(path_marker);
        }
        // 持续发布以便 RViz 接收
        // 实际项目中只需发布一次，或者在定时器中发布
        timer_ = this->create_wall_timer(std::chrono::seconds(1), 
            [this, marker_array]() {
                marker_pub_->publish(marker_array);
            });
    }
    rclcpp::TimerBase::SharedPtr timer_;
};
int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<SimplePRM3D>());
    rclcpp::shutdown();
    return 0;
}
```
### CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.8)
project(prm_planner_cpp)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(visualization_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)

add_executable(simple_prm_3d src/simple_prm_3d.cpp)
ament_target_dependencies(simple_prm_3d rclcpp visualization_msgs geometry_msgs)

install(TARGETS simple_prm_3d
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```
### package.xml
```XML
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
    <name>prm_planner_cpp</name>
    <version>0.0.0</version>
    <description>Simple 3D PRM Planner Demo</description>
    <maintainer email="user@todo.todo">user</maintainer>
    <license>TODO</license>

    <buildtool_depend>ament_cmake</buildtool_depend>

    <depend>rclcpp</depend>
    <depend>visualization_msgs</depend>
    <depend>geometry_msgs</depend>

    <test_depend>ament_lint_auto</test_depend>
    <test_depend>ament_lint_common</test_depend>

    <export>
        <build_type>ament_cmake</build_type>
    </export>
</package>
```