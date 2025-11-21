## 特征
### 概率占据
1. 不仅表示 有障碍物 或 空闲，而是存储占据的概率
2. 这允许地图随着传感器的多次观测动态更新（例如，一次误报不会立即标记为障碍物，需要多次确认）
### 高效存储
如果一个节点下的 8 个子节点状态相同则会被合并为父节点，这极大地节省了内存
### 多分辨率支持
可以方便地查询不同层级（分辨率）的地图数据
## 示例
通常使用 octomap_server 包来接收点云，并将其转换为八叉树地图
### 
```C++
#include <memory>
#include <string>
#include <vector>
#include "rclcpp/rclcpp.hpp"
#include "sensor_msgs/msg/point_cloud2.hpp"
#include "octomap_msgs/msg/octomap.hpp"
#include "octomap_msgs/conversions.h"
#include <pcl/point_cloud.h>
#include <pcl/point_types.h>
#include <pcl_conversions/pcl_conversions.h>
#include <octomap/octomap.h>
#include <octomap/OcTree.h>

using std::placeholders::_1;

class PclOctomapServer : public rclcpp::Node{
public:
    PclOctomapServer() : Node("pcl_octomap_server"){
        // 1. 参数声明
        this->declare_parameter("resolution", 0.05); // 分辨率 5cm
        this->declare_parameter("frame_id", "map"); // 地图坐标系
        this->declare_parameter("sensor_max_range", 5.0); // 传感器最大有效距离
        double res = this->get_parameter("resolution").as_double();
        frame_id_ = this->get_parameter("frame_id").as_string();
        max_range_ = this->get_parameter("sensor_max_range").as_double();
        // 2. 初始化 Octree
        tree_ = std::make_shared<octomap::OcTree>(res);
        // 设置概率更新参数 (这是调优的关键)
        tree_->setProbHit(0.7);   // 击中时概率增加到 0.7
        tree_->setProbMiss(0.4);  // 穿透(空闲)时概率降低到 0.4
        tree_->setClampingThresMin(0.12); // 最小概率阈值
        tree_->setClampingThresMax(0.97); // 最大概率阈值
        // 3. 创建发布者和订阅者
        pub_octomap_ = this->create_publisher<octomap_msgs::msg::Octomap>("octomap_out", 1);        
        sub_pointcloud_ = this->create_subscription<sensor_msgs::msg::PointCloud2>(
            "input_cloud", 10, std::bind(&PclOctomapServer::pointCloudCallback, this, _1));
        RCLCPP_INFO(this->get_logger(), "Octomap Server running. Waiting for PointCloud2 on /input_cloud...");
    }
private:
    void pointCloudCallback(const sensor_msgs::msg::PointCloud2::SharedPtr msg){
        // 步骤 1: 将 ROS 消息转换为 PCL 点云
        pcl::PCLPointCloud2 pcl_pc2;
        pcl_conversions::toPCL(*msg, pcl_pc2);
        pcl::PointCloud<pcl::PointXYZ> pcl_cloud;
        pcl::fromPCLPointCloud2(pcl_pc2, pcl_cloud);
        // 步骤 2: 转换为 Octomap 的 Pointcloud 结构
        octomap::Pointcloud octo_cloud;
        for (const auto& p : pcl_cloud.points) {
            // 过滤掉 NaN (无效点)
            if (!std::isnan(p.x) && !std::isnan(p.y) && !std::isnan(p.z)) {
                octo_cloud.push_back(p.x, p.y, p.z);
            }
        }
        // 步骤 3: 获取传感器原点 (Sensor Origin)
        // 注意：在真实机器人中，你需要使用 TF2 查找 "map" 到 "sensor_frame" 的变换
        // 这里为了简化演示，假设点云已经是 map 坐标系下的，且传感器位置在 (0,0,0)
        // 如果你的点云是在 "lidar_link" 坐标系，这里必须提供 lidar 在 map 中的位置
        octomap::point3d sensor_origin(0.0, 0.0, 0.0); 
        // 步骤 4: 插入点云并执行光线投射 (Raycasting)
        // 这会自动更新占据的体素，并清除传感器和障碍物之间的体素
        if (octo_cloud.size() > 0) {
            tree_->insertPointCloud(octo_cloud, sensor_origin, max_range_, true, true);
        }
        // 步骤 5: 剪枝与发布
        // 只有当且仅当我们需要显示或使用时才发布，不需要每一帧点云都发布一次全图
        // 这里为了演示实时性，每次更新都发布
        tree_->prune();         
        publishMap(msg->header.stamp);
    }
    void publishMap(const rclcpp::Time& stamp)    {
        auto map_msg = std::make_unique<octomap_msgs::msg::Octomap>();
        map_msg->header.frame_id = frame_id_;
        map_msg->header.stamp = stamp;
        // 使用二进制地图 (Binary Map) 节省带宽，只包含 占据/空闲
        // 如果需要显示颜色高度图，binaryMapToMsg 也足够，因为Rviz插件会根据高度渲染颜色
        if (octomap_msgs::binaryMapToMsg(*tree_, *map_msg)) {
            pub_octomap_->publish(std::move(map_msg));
        } else {
            RCLCPP_ERROR(this->get_logger(), "Error serializing Octomap");
        }
    }
    std::shared_ptr<octomap::OcTree> tree_;
    std::string frame_id_;
    double max_range_;
    rclcpp::Publisher<octomap_msgs::msg::Octomap>::SharedPtr pub_octomap_;
    rclcpp::Subscription<sensor_msgs::msg::PointCloud2>::SharedPtr sub_pointcloud_;
};

int main(int argc, char *argv[]){
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<PclOctomapServer>());
    rclcpp::shutdown();
    return 0;
}
```