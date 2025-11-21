## 概述
1. 启发式搜索算法，广泛用于在图中寻找从起点到终点的最短路径
2. 最佳优先搜索算法，会优先探索看起来最接近终点的节点
## 核心概念
### 公式
A* 算法的核心在于其评估函数，为每个节点（在 3D 中为体素）计算一个“代价值” $f(n)$，并始终优先探索 $f(n)$ 最小的节点
$$f(n) = g(n) + h(n)$$
1. $g(n)$: 从起点到当前节点 $n$ 的实际累计代价。这是已经付出的、确定的代价
2. $h(n)$: 从当前节点 $n$ 到终点的估计代价。这是一个基于启发式的猜测值，用于指导搜索方向
### 数据结构
A* 算法依赖两个核心列表来管理搜索过程
1. Open List (开放列表)\
（1）作用: 存储所有已被发现但尚未被访问（即尚未被扩展）的节点\
（2）实现: 通常使用最小优先队列，根据节点的 $f(n)$ 值进行排序，$f$ 值最小的节点总是在队列顶部，会被最先取出进行处理
2. Closed List (关闭列表)\
（1）作用: 存储所有已经被访问过的节点\
（2）目的: 防止算法重新处理同一个节点，避免无限循环和冗余计算\
（3）实现: 通常使用哈希集或布尔数组，以实现 $O(1)$ 的快速查找
## 扩展到 3D 的变化
### 空间表示
1. 不再是 2D 栅格地图，而是 3D 体素网格
2. 在 ROS 中，这通常由 OctoMap (octomap_msgs) 或 3D 代价地图 (nav2_costmap_3d，虽然目前 nav2 仍以 2D 为主，但 3D 导航是发展方向) 来表示
### 邻居搜索
1. 在 2D 中有 4/8 邻域，在 3D 中有 6/18/26 领域
2. 选择 26 邻域可以产生更平滑、更自然的对角线路径，但计算成本也更高
### 启发式函数 $h(n)$
$h(n)$ 必须是可接受的，即它绝不能高估到终点的实际距离，否则可能会错过最优路径
1. 3D 曼哈顿距离：适用于 6 邻域搜索
$$h(n) = |\text{curr}.x - \text{goal}.x| + |\text{curr}.y - \text{goal}.y| + |\text{curr}.z - \text{goal}.z|$$
2. 3D 欧几里得距离：最直观的直线距离，适用于 26 邻域，是最常用的启发式函数
$$h(n) = \sqrt{(\text{curr}.x - \text{goal}.x)^2 + (\text{curr}.y - \text{goal}.y)^2 + (\text{curr}.z - \text{goal}.z)^2}$$
3. 3D 切比雪夫距离：适用于 26 邻域
$$h(n) = \max(|\text{curr}.x - \text{goal}.x|, |\text{curr}.y - \text{goal}.y|, |\text{curr}.z - \text{goal}.z|)$$
## 算法步骤
### 初始化
1. 创建一个 Open List (最小优先队列) 和一个 Closed List (哈希集或布尔数组)
2. 将起点放入 Open List
3. 设置起点的 $g(n)$ 值为 0，$h(n)$ 值为计算到终点的启发式距离，$f(n) = g(n) + h(n)$
4. 设置起点的父节点为 nullptr
### 循环搜索
当 Open List 不为空时，执行循环：
1. 从 Open List 中取出 $f(n)$ 值最小的节点，记为当前节点
2. 将当前节点从 Open List 移除，并将其添加到 Closed List 中
### 检查终点
1. 如果当前节点是终点，则搜索成功，不是则扩展邻居
2. 通过从终点开始，沿着每个节点的 parent 指针回溯到起点，重构并返回路径
### 扩展邻居
遍历当前节点的所有邻居
#### 过滤邻居
1. 如果邻居是障碍物，则忽略它
2. 如果邻居已经在 Closed List 中，则忽略它（已经处理过了）
#### 计算新路径代价
计算从起点经过当前节点到达该邻居的 $g$ 值
$$g_{\text{new}} = \text{CurrentNode}.g + \text{distance(CurrentNode, Neighbor)}$$
#### 处理邻居
1. 如果邻居不在 Open List 中：\
（1）设置邻居的 $g = g_{\text{new}}$\
（2）计算邻居的 $h = \text{heuristic(Neighbor, Goal)}$\
（3）计算邻居的 $f = g + h$\
（4）设置邻居的 parent 为当前节点\
（5）将邻居添加到 Open List 中
2. 如果邻居已经在 Open List 中：检查刚刚找到的这条路径是否更优，如果 $g_{\text{new}} < \text{Neighbor}.g$\
（1）找到了一个到达该邻居的更短路径\
（2）更新该邻居的 $g = g_{\text{new}}$\
（3）更新该邻居的 $f = g + h$\
（4）更新该邻居的 parent 为当前节点\
（5）在优先队列中，这可能需要一个 "decrease key" 操作，或者干脆重新插入一个带更低 $f$ 值的新副本
### 无解
如果循环结束（即 Open List 变为空），但仍未找到终点，则说明从起点到终点没有可达路径
## 代码示例
在 ROS2 中，A* 算法通常被实现为 Nav2 规划器插件，会实现nav2_core::GlobalPlanner 接口
### 结构
```Plaintext
astar_3d/
├── CMakeLists.txt          # 构建配置文件
├── include/
│   └── astar_3d.hpp        # 头文件：声明类和数据结构
├── src/
│   └── astar_3d.cpp        # 源文件：实现算法逻辑
└── main.cpp                # 入口文件：测试与调用
```
### astar_3d.hpp
定义了数据结构和 AStar3D 类的接口，将 Node 结构体和 CompareNode 放在私有域或作为辅助结构，保持对外接口的整洁
```C++
#ifndef ASTAR_3D_HPP
#define ASTAR_3D_HPP
#include <vector>
#include <cmath>
#include <limits>

struct Point3D {
    int x, y, z;
    bool operator==(const Point3D& other) const {
        return x == other.x && y == other.y && z == other.z;
    }
    bool operator!=(const Point3D& other) const {
        return !(*this == other);
    }
};
class AStar3D {
public:
    // 构造函数，初始化地图尺寸
    AStar3D(int width, int height, int depth);
    // 设置障碍物
    void setObstacle(int x, int y, int z);
    // 执行路径规划，返回路径点列表，如果未找到则为空
    std::vector<Point3D> findPath(Point3D start, Point3D goal);
private:
    // 内部使用的节点结构
    struct Node {
        Point3D point;
        int id;              // 1D 数组索引
        int parent_id = -1;  // 父节点索引
        double g = 0.0;
        double h = 0.0;
        double f = 0.0;
        bool closed = false;
        void reset() {
            parent_id = -1;
            g = std::numeric_limits<double>::infinity();
            h = 0.0;
            f = std::numeric_limits<double>::infinity();
            closed = false;
        }
    };
    // 优先队列比较器
    struct CompareNode {
        bool operator()(const Node* a, const Node* b) const {
            return a->f > b->f; // 最小堆
        }
    };
    // 成员变量
    int width_, height_, depth_;
    std::vector<int> grid_map_; // 0: 空闲, 1: 障碍
    std::vector<Node> nodes_;   // 节点内存池
    // 辅助函数
    int getIndex(int x, int y, int z) const;
    bool isValid(int x, int y, int z) const;
    bool isObstacle(int x, int y, int z) const;
    double calculateHeuristic(const Point3D& a, const Point3D& b) const;
    std::vector<Point3D> reconstructPath(Node* current);
};
#endif // ASTAR_3D_HPP
```
### astar_3d.cpp
```C++
#include "astar_3d.hpp"
#include <queue>
#include <algorithm>
#include <iostream>

AStar3D::AStar3D(int width, int height, int depth): width_(width), height_(height), depth_(depth) {
    int total_size = width * height * depth;
    grid_map_.resize(total_size, 0);
    nodes_.resize(total_size);
    // 初始化节点池
    for (int z = 0; z < depth; ++z) {
        for (int y = 0; y < height; ++y) {
            for (int x = 0; x < width; ++x) {
                int idx = getIndex(x, y, z);
                nodes_[idx].point = {x, y, z};
                nodes_[idx].id = idx;
                nodes_[idx].reset();
            }
        }
    }
}
void AStar3D::setObstacle(int x, int y, int z) {
    if (isValid(x, y, z)) {
        grid_map_[getIndex(x, y, z)] = 1;
    }
}
int AStar3D::getIndex(int x, int y, int z) const {
    return z * width_ * height_ + y * width_ + x;
}
bool AStar3D::isValid(int x, int y, int z) const {
    return x >= 0 && x < width_ && 
           y >= 0 && y < height_ && 
           z >= 0 && z < depth_;
}

bool AStar3D::isObstacle(int x, int y, int z) const {
    return grid_map_[getIndex(x, y, z)] == 1;
}
double AStar3D::calculateHeuristic(const Point3D& a, const Point3D& b) const {
    // 欧几里得距离
    return std::sqrt(std::pow(a.x - b.x, 2) + 
                     std::pow(a.y - b.y, 2) + 
                     std::pow(a.z - b.z, 2));
}
std::vector<Point3D> AStar3D::reconstructPath(Node* current) {
    std::vector<Point3D> path;
    while (current->parent_id != -1) {
        path.push_back(current->point);
        current = &nodes_[current->parent_id];
    }
    path.push_back(current->point); // 加入起点
    std::reverse(path.begin(), path.end());
    return path;
}
std::vector<Point3D> AStar3D::findPath(Point3D start, Point3D goal) {
    // 1. 重置状态
    for (auto& node : nodes_) node.reset();
    // 2. 校验
    if (!isValid(start.x, start.y, start.z) || !isValid(goal.x, goal.y, goal.z)) {
        std::cerr << "[AStar3D] Error: 起点或终点越界" << std::endl;
        return {};
    }
    if (isObstacle(start.x, start.y, start.z) || isObstacle(goal.x, goal.y, goal.z)) {
        std::cerr << "[AStar3D] Error: 起点或终点位于障碍物内" << std::endl;
        return {};
    }
    // 3. 初始化起点
    int start_idx = getIndex(start.x, start.y, start.z);
    Node* start_node = &nodes_[start_idx];
    start_node->g = 0;
    start_node->h = calculateHeuristic(start, goal);
    start_node->f = start_node->g + start_node->h;
    // 4. Open List
    std::priority_queue<Node*, std::vector<Node*>, CompareNode> open_list;
    open_list.push(start_node);
    // 5. 主循环
    while (!open_list.empty()) {
        Node* current = open_list.top();
        open_list.pop();
        if (current->closed) continue;
        current->closed = true;
        if (current->point == goal) {
            return reconstructPath(current);
        }
        // 26 邻域扩展
        for (int dz = -1; dz <= 1; ++dz) {
            for (int dy = -1; dy <= 1; ++dy) {
                for (int dx = -1; dx <= 1; ++dx) {
                    if (dx == 0 && dy == 0 && dz == 0) continue;
                    int nx = current->point.x + dx;
                    int ny = current->point.y + dy;
                    int nz = current->point.z + dz;
                    if (!isValid(nx, ny, nz) || isObstacle(nx, ny, nz)) continue;
                    int neighbor_idx = getIndex(nx, ny, nz);
                    Node* neighbor = &nodes_[neighbor_idx];
                    if (neighbor->closed) continue;
                    double move_cost = std::sqrt(dx*dx + dy*dy + dz*dz);
                    double new_g = current->g + move_cost;
                    if (new_g < neighbor->g) {
                        neighbor->g = new_g;
                        neighbor->h = calculateHeuristic(neighbor->point, goal);
                        neighbor->f = neighbor->g + neighbor->h;
                        neighbor->parent_id = current->id;
                        open_list.push(neighbor);
                    }
                }
            }
        }
    }
    return {};
}
```
### main.cpp
```C++
#include <iostream>
#include "astar_3d.hpp"

int main() {
    // 1. 创建 10x10x10 的环境
    int w = 10, h = 10, d = 10;
    AStar3D planner(w, h, d);
    std::cout << "地图已初始化: " << w << "x" << h << "x" << d << std::endl;
    // 2. 设置障碍物：在 x=5 处造一面墙，中间留孔
    std::cout << "正在生成障碍物..." << std::endl;
    for (int y = 0; y < h; ++y) {
        for (int z = 0; z < d; ++z) {
            // (5, 5, 5) 是通过的洞
            if (y == 5 && z == 5) continue;
            planner.setObstacle(5, y, z);
        }
    }
    // 3. 定义起点和终点
    Point3D start = {0, 0, 0};
    Point3D goal = {9, 9, 9};
    // 4. 规划
    std::cout << "开始规划路径..." << std::endl;
    auto path = planner.findPath(start, goal);
    // 5. 输出结果
    if (path.empty()) {
        std::cout << "失败: 未找到路径。" << std::endl;
    } else {
        std::cout << "成功: 路径长度 " << path.size() << std::endl;
        std::cout << "路径点序列:" << std::endl;
        for (size_t i = 0; i < path.size(); ++i) {
            const auto& p = path[i];
            std::cout << "(" << p.x << "," << p.y << "," << p.z << ")";
            if (i < path.size() - 1) std::cout << " -> ";
            if ((i + 1) % 5 == 0) std::cout << std::endl;
        }
        std::cout << std::endl;
    }
    return 0;
}
```
### CMakeLists.txt
```CMake
cmake_minimum_required(VERSION 3.15)
project(AStar3DProject)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
include_directories(include)
add_executable(astar_3d_node 
    main.cpp 
    src/astar_3d.cpp
)
```