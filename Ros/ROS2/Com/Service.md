## 概念
一种基于请求/响应模型的通信机制，用于实现远程过程调用，即一个节点（客户端）可以调用另一个节点（服务器）上的一个函数，并等待它返回结果
### 服务器
1. 提供服务的节点，发布一个服务名称，然后等待请求的到来
2. 当它收到一个请求时，会执行一个回调函数，计算出一个响应，并将响应发回给客户端
### 客户端
1. 使用服务的节点，向一个已知的服务名称发送一个请求数据
2. 发送后，会阻塞或异步等待，直到收到来自服务器的响应数据
### 服务定义
1. Service 使用 .srv 文件来定义其数据结构
2. 被一个 `---` 分成两部分，上部分是请求的数据结构，下部分响应的数据结构
### 示例
```
# --- 请求 (Request) ---
int64 a
int64 b
---
# --- 响应 (Response) ---
int64 sum
```
## 原理
### 同步与阻塞
客户端 call() (或 async_send_request()) 后，它必须等待服务器的响应。默认情况下，客户端的调用线程会被阻塞，直到响应到达或超时
### “一对一” 事务
虽然一个 Service 可以被多个客户端调用，并且也可以有多个服务器提供（实现负载均衡，但不常见），但一次事务总是一对一的
## 示例
### 创建接口包
1. 确保有一个接口包（例如 my_robot_interfaces），在包中创建 srv/ 目录
2. 创建 my_robot_interfaces/srv/AddTwoInts.srv 文件并填入内容
```
int64 a
int64 b
---
int64 sum
```
3. 修改 CMakeLists.txt，添加 .srv 文件
```CMake
# ...
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/HardwareStatus.msg"  # (如果你有)
  "srv/AddTwoInts.srv"      # <-- 添加这一行
)
```
5. 修改 package.xml：在<buildtool_depend>ament_cmake</buildtool_depend> 之后添加
```XML
<build_depend>rosidl_default_generators</build_depend>
<exec_depend>rosidl_default_runtime</exec_depend>
<member_of_group>rosidl_interface_packages</member_of_group>
```
6. 在工作空间根目录 colcon build 一次，以生成 .hpp 头文件，供下一步使用
### 服务器
1. 创建一个新包 my_service_server
```Bash
cd ~/ros2_ws/src
ros2 pkg create my_service_server --build-type ament_cmake --dependencies rclcpp my_robot_interfaces
```
1. 创建 my_service_server/src/server_node.cpp
```C++
#include "rclcpp/rclcpp.hpp"
// 1. 包含自动生成的 .hpp 服务头文件
#include "my_robot_interfaces/srv/add_two_ints.hpp"
#include <memory>

// 5. 定义服务回调函数
// 它的签名是固定的：
// void function_name(
//   const std::shared_ptr<RequestType> request,
//   std::shared_ptr<ResponseType>      response)
//
void handle_add_two_ints(const std::shared_ptr<my_robot_interfaces::srv::AddTwoInts::Request> request,
                std::shared_ptr<my_robot_interfaces::srv::AddTwoInts::Response> response){
    // 6. 从 'request' 对象中读取数据
    response->sum = request->a + request->b;
    // 7. 打印日志 (最佳实践)
    RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "收到请求: a=%ld, b=%ld", request->a, request->b);
    RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "发送响应: sum=%ld", response->sum);
}

int main(int argc, char **argv){
    // 2. 初始化 ROS2
    rclcpp::init(argc, argv);
    // 3. 创建一个节点
    std::shared_ptr<rclcpp::Node> node = rclcpp::Node::make_shared("add_two_ints_server");
    // 4. 创建服务
    rclcpp::Service<my_robot_interfaces::srv::AddTwoInts>::SharedPtr service =
        node->create_service<my_robot_interfaces::srv::AddTwoInts>("add_two_ints", &handle_add_two_ints);

    RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "服务已启动，等待请求...");
    // 8. 启动 rclcpp::spin()，使节点保持活动状态以接收请求
    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```
3. 配置 my_service_server/CMakeLists.txt
```CMake
add_executable(server_node src/server_node.cpp)
ament_target_dependencies(server_node rclcpp my_robot_interfaces)

install(TARGETS
  server_node
  DESTINATION lib/${PROJECT_NAME}
)
```
### 客户端
其他步骤等同
```C++
#include "rclcpp/rclcpp.hpp"
// 1. 包含服务头文件
#include "my_robot_interfaces/srv/add_two_ints.hpp"
#include <chrono>
#include <cstdlib> // for atoll
#include <memory>
using namespace std::chrono_literals;

int main(int argc, char **argv){
    // 2. 初始化 ROS2
    rclcpp::init(argc, argv);
    if (argc != 3) {
        RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "用法: ros2 run my_service_client client_node <int_a> <int_b>");
        return 1;
    }
    // 3. 创建一个节点，用于发送请求
    std::shared_ptr<rclcpp::Node> node = rclcpp::Node::make_shared("add_two_ints_client");
    // 4. 创建客户端
    auto client = node->create_client<my_robot_interfaces::srv::AddTwoInts>("add_two_ints");
    // 5. 创建请求对象
    auto request = std::make_shared<my_robot_interfaces::srv::AddTwoInts::Request>();
    request->a = std::atoll(argv[1]);
    request->b = std::atoll(argv[2]);
    // 6. 关键：等待服务上线
    while (!client->wait_for_service(1s)) {
        if (!rclcpp::ok()) {
            RCLCPP_ERROR(rclcpp::get_logger("rclcpp"), "客户端在等待服务时被中断...");
            return 1;
        }
        RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "服务未上线，正在等待...");
    }
    // 7. 发送异步请求
    // async_send_request 返回一个 "Future"
    auto result_future = client->async_send_request(request);
    RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "已发送请求 a=%ld, b=%ld", request->a, request->b);
    // 8. 关键：等待响应
    if (rclcpp::spin_until_future_complete(node, result_future) ==
        rclcpp::FutureReturnCode::SUCCESS)    {
        // 9. 获取结果
        auto result = result_future.get();
        RCLCPP_INFO(rclcpp::get_logger("rclcpp"), "收到响应: sum=%ld", result->sum);
    } else {
        RCLCPP_ERROR(rclcpp::get_logger("rclcpp"), "服务调用失败");
    }
    // 10. 关闭
    rclcpp::shutdown();
    return 0;
}
```
### 构建与运行
1. 构建
```Bash
cd ~/ros2_ws
colcon build --packages-up-to my_service_client my_service_server
```
2. 运行
终端 1 (激活环境并运行 Server)
```Bash
source install/setup.bash
ros2 run my_service_server server_node
# 终端会显示: "服务已启动，等待请求..."
```
终端 2 (激活环境并运行 Client)
```Bash
source install/setup.bash
# 传入两个数字作为参数
ros2 run my_service_client client_node 5 10
```