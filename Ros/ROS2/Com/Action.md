## 核心概念
用于处理可抢占的、有反馈的、长时间运行的任务
### 客户端
1. 向服务器发送一个目标，例如：“请移动到坐标 (X, Y)”，可以（可选）接收反馈
2. 最终会收到一个结果（“已到达”）或中止通知（“路径被堵死”），也可以随时发送一个取消请求
### 服务器
1. 接收一个目标，并决定接受还是拒绝，如果接受便开始执行任务
2. 在执行期间，它会定期发布反馈（例如：“当前在 (X', Y')，距离目标还剩5米”）。
3. 任务完成后，它发送一个最终结果。会监听取消请求，并停止任务。
### .action 定义文件
与 .srv 类似，被两个 --- 分隔符分成了三部分，以下是经典的斐波那契示例
```
# --- 第一部分：目标 (Goal) ---
# 客户端发送的请求
int32 order
---
# --- 第二部分：结果 (Result) ---
# 服务器在任务完成后发送的最终响应
int32[] sequence
---
# --- 第三部分：反馈 (Feedback) ---
# 服务器在任务执行期间发送的周期性更新
int32[] partial_sequence
```
## 原理
### 完整的状态机
Action 的核心是一个目标句柄的生命周期，服务器和客户端都会跟踪这个句柄的状态
1. PENDING (待定): 客户端已发送，服务器尚未响应
2. ACCEPTED (已接受): 服务器已接受该目标
3. EXECUTING (执行中): 服务器正在处理任务
4. CANCELING (取消中): 客户端请求取消
5. SUCCEEDED (已成功): 任务成功完成，结果已发送
6. ABORTED (已中止): 服务器在执行中遇到错误，任务失败
7. CANCELED (已取消): 任务被成功取消
### 底层实现
Action 本身并不是一个新的底层协议。ROS2 巧妙地使用 Topic 和 Service 将Action 组合了起来\
当创建一个名为 /my_action 的 Action 时，ROS2 会自动创建五个“隐藏”的 Topic 和 Service
1. /my_action/_action/send_goal (Service): 客户端用它来发送新目标
2. /my_action/_action/cancel_goal (Service): 客户端用它来请求取消
3. /my_action/_action/get_result (Service): 客户端用它来查询已完成目标的结果
4. /my_action/_action/feedback (Topic): 服务器用它来广播反馈
5. /my_action/_action/status (Topic): 服务器用它来广播所有目标的状态（即上面的状态机）
>rclcpp_action 库将这五个通信通道封装成了一个易于使用的API
## 示例
### 创建接口包
1. 确保有一个接口包，在包中创建 action/ 目录
2. 创建 my_robot_interfaces/action/Fibonacci.action 文件并填入三部分内容
3. 修改 CMakeLists.txt，添加 .action 文件
```CMake
# ...
find_package(rosidl_default_generators REQUIRED)
find_package(ament_cmake REQUIRED)

# 找到action的C++支持
find_package(ros2_action_cflags REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/HardwareStatus.msg"  # (如果有)
  "srv/AddTwoInts.srv"      # (如果有)
  "action/Fibonacci.action" # <-- 添加这一行
)
```
4. 修改 package.xml
```XML
<build_depend>rosidl_default_generators</build_depend>
<exec_depend>rosidl_default_runtime</exec_depend>
<build_depend>action_msgs</build_depend> 
<exec_depend>action_msgs</exec_depend>
<member_of_group>rosidl_interface_packages</member_of_group>
```
5. 在工作空间根目录 colcon build 一次，以生成 .hpp 头文件
### 服务器
代码是最复杂的，因为它必须管理状态和回调
1. 创建新包 my_action_server (依赖 rclcpp, rclcpp_action, my_robot_interfaces)
2. 创建 my_action_server/src/fibonacci_server.cpp
```C++
#include "rclcpp/rclcpp.hpp"
#include "rclcpp_action/rclcpp_action.hpp"
#include "my_robot_interfaces/action/fibonacci.hpp"
#include <memory>
#include <thread>

class FibonacciActionServer : public rclcpp::Node{
public:
    // 为 Action 类型定义别名，方便使用
    using Fibonacci = my_robot_interfaces::action::Fibonacci;
    using GoalHandleFibonacci = rclcpp_action::ServerGoalHandle<Fibonacci>;

    explicit FibonacciActionServer():Node("fibonacci_action_server"){
        // 3. 创建 Action Server
        this->action_server_ = rclcpp_action::create_server<Fibonacci>(this,"fibonacci",
        std::bind(&FibonacciActionServer::handle_goal, this, std::placeholders::_1, std::placeholders::_2),
        std::bind(&FibonacciActionServer::handle_cancel, this, std::placeholders::_1),
        std::bind(&FibonacciActionServer::handle_accepted, this, std::placeholders::_1));
        RCLCPP_INFO(this->get_logger(), "Fibonacci Action Server 已启动.");
    }

private:
    rclcpp_action::Server<Fibonacci>::SharedPtr action_server_;
    // 4. 回调1: handle_goal (处理新目标)，决定是接受还是拒绝
    rclcpp_action::GoalResponse handle_goal(const rclcpp_action::GoalUUID & uuid,
        std::shared_ptr<const Fibonacci::Goal> goal) {
        RCLCPP_INFO(this->get_logger(), "收到新目标，阶数: %d", goal->order);
        (void)uuid;
        // 可以拒绝目标，例如：
        if (goal->order > 20) {
            RCLCPP_WARN(this->get_logger(), "目标被拒绝 (阶数 > 20).");
            return rclcpp_action::GoalResponse::REJECT;
        }
        RCLCPP_INFO(this->get_logger(), "目标已接受.");
        return rclcpp_action::GoalResponse::ACCEPT_AND_EXECUTE;
    }
    // 5. 回调2: handle_cancel (处理取消请求)
    rclcpp_action::CancelResponse handle_cancel(const std::shared_ptr<GoalHandleFibonacci> goal_handle){
        RCLCPP_INFO(this->get_logger(), "收到对目标 #%d 的取消请求", goal_handle->get_goal()->order);
        (void)goal_handle;
        return rclcpp_action::CancelResponse::ACCEPT;
    }
    // 6. 回调3: handle_accepted (执行被接受的目标)
    // 关键：这个函数不应该阻塞，它应该启动一个线程去执行
    void handle_accepted(const std::shared_ptr<GoalHandleFibonacci> goal_handle)    {
        RCLCPP_INFO(this->get_logger(), "开始执行目标 (阶数: %d)", goal_handle->get_goal()->order);
        // 创建一个新线程来执行任务，避免阻塞 rclcpp::spin()
        std::thread{std::bind(&FibonacciActionServer::execute, this, std::placeholders::_1),
        goal_handle}.detach();
    }
    // 7. 核心执行函数 (在单独的线程中运行)
    void execute(const std::shared_ptr<GoalHandleFibonacci> goal_handle)    {
        const auto goal = goal_handle->get_goal();
        auto feedback = std::make_shared<Fibonacci::Feedback>();
        auto result = std::make_shared<Fibonacci::Result>();

        feedback->partial_sequence.push_back(0);
        feedback->partial_sequence.push_back(1);
        
        rclcpp::Rate loop_rate(1); // 模拟一个较慢的任务，每秒1Hz

        for (int i = 1; (i < goal->order) && rclcpp::ok(); ++i){
            // 8. 检查是否被取消
            if (goal_handle->is_canceling()) {
                result->sequence = feedback->partial_sequence;
                goal_handle->canceled(result); // 设置为 CANCELED 状态
                RCLCPP_INFO(this->get_logger(), "目标已被取消");
                return;
            }
            // 9. 执行任务
            feedback->partial_sequence.push_back(
            feedback->partial_sequence[i] + feedback->partial_sequence[i - 1]);
            // 10. 发布反馈
            goal_handle->publish_feedback(feedback);
            RCLCPP_INFO(this->get_logger(), "发布反馈: %d", (int)feedback->partial_sequence.size());

            loop_rate.sleep();
        }
        // 11. 任务完成，设置最终结果
        if (rclcpp::ok()) {
            result->sequence = feedback->partial_sequence;
            goal_handle->succeed(result); // 设置为 SUCCEEDED 状态
            RCLCPP_INFO(this->get_logger(), "目标已成功");
        }
    }
};

int main(int argc, char ** argv){
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<FibonacciActionServer>());
    rclcpp::shutdown();
    return 0;
}
```
### 客户端
1. 创建新包 my_action_client (依赖 rclcpp, rclcpp_action, my_robot_interfaces)
2. 创建 my_action_client/src/fibonacci_client.cpp
```C++
#include "rclcpp/rclcpp.hpp"
#include "rclcpp_action/rclcpp_action.hpp"
#include "my_robot_interfaces/action/fibonacci.hpp"
#include <chrono>
using namespace std.chrono_literals;

class FibonacciActionClient : public rclcpp::Node{
public:
    using Fibonacci = my_robot_interfaces::action::Fibonacci;
    using GoalHandleFibonacci = rclcpp_action::ClientGoalHandle<Fibonacci>;

    explicit FibonacciActionClient():Node("fibonacci_action_client"){
        // 1. 创建 Action Client
        this->client_ptr_ = rclcpp_action::create_client<Fibonacci>(this,"fibonacci");
        RCLCPP_INFO(this->get_logger(), "Action Client 已创建.");
    }
    // 2. 发送目标的函数
    void send_goal(int order)    {
        RCLCPP_INFO(this->get_logger(), "等待 Action Server 上线...");
        if (!this->client_ptr_->wait_for_action_server(10s)) {
            RCLCPP_ERROR(this->get_logger(), "Action Server 未上线.");
            return;
        }
        // 3. 创建一个目标
        auto goal_msg = Fibonacci::Goal();
        goal_msg.order = order;
        RCLCPP_INFO(this->get_logger(), "发送目标 (阶数: %d)", order);
        // 4. 设置回调函数
        // 客户端必须注册回调来处理反馈和最终结果
        auto send_goal_options = rclcpp_action::Client<Fibonacci>::SendGoalOptions();
        // 4a. 注册反馈回调
        send_goal_options.feedback_callback =
        std::bind(&FibonacciActionClient::feedback_callback, this, std::placeholders::_1, std::placeholders::_2);
        // 4b. 注册结果回调
        send_goal_options.result_callback =
        std::bind(&FibonacciActionClient::result_callback, this, std::placeholders::_1);
        // 5. 异步发送目标，这会立即返回一个 "Goal Handle Future"
        auto goal_handle_future = this->client_ptr_->async_send_goal(goal_msg, send_goal_options);
        RCLCPP_INFO(this->get_logger(), "等待服务器接受目标...");
        // 可以等待 'goal_handle_future' 来确认服务器是否接受了目标
        if (rclcpp::spin_until_future_complete(this->get_node_base_interface(), goal_handle_future) !=
            rclcpp::FutureReturnCode::SUCCESS)        {
            RCLCPP_ERROR(this->get_logger(), "目标发送失败");
            return;
        }
        auto goal_handle = goal_handle_future.get();
        if (!goal_handle) {
            RCLCPP_ERROR(this->get_logger(), "目标被服务器拒绝");
        } else {
            RCLCPP_INFO(this->get_logger(), "目标被接受，等待结果...");
        }
    }
private:
    rclcpp_action::Client<Fibonacci>::SharedPtr client_ptr_;
    // 6. 反馈回调 (当服务器发布反馈时调用)
    void feedback_callback(GoalHandleFibonacci::SharedPtr,
        const std::shared_ptr<const Fibonacci::Feedback> feedback){
        RCLCPP_INFO(this->get_logger(), "收到反馈: 序列长度 %d", (int)feedback->partial_sequence.size());
    }
    // 7. 结果回调 (当Action完成时调用)
    void result_callback(const GoalHandleFibonacci::WrappedResult & result){
        // 8. 检查最终状态
        switch (result.code){
        case rclcpp_action::ResultCode::SUCCEEDED:
            RCLCPP_INFO(this->get_logger(), "结果: 成功! 最终序列长度: %d", (int)result.result->sequence.size());
            break;
        case rclcpp_action::ResultCode::ABORTED:
            RCLCPP_ERROR(this->get_logger(), "结果: 任务中止");
            break;
        case rclcpp_action::ResultCode::CANCELED:
            RCLCPP_WARN(this->get_logger(), "结果: 任务取消");
            break;
        default:
            RCLCPP_ERROR(this->get_logger(), "结果: 未知错误");
            break;
        }
        // 示例完成后退出 spin
        rclcpp::shutdown();
    }
};

int main(int argc, char ** argv){
    rclcpp::init(argc, argv);
    auto action_client = std::make_shared<FibonacciActionClient>();
    // 发送一个目标
    action_client->send_goal(10);
    // 客户端也需要 spin 来接收回调
    rclcpp::spin(action_client);
    rclcpp::shutdown();
    return 0;
}
```
### 构建和运行
1. 构建
```Bash
cd ~/ros2_ws
colcon build --packages-up-to my_action_client my_action_server
```
2. 运行\
终端 1 (激活环境并运行 Server)
```Bash
source install/setup.bash
ros2 run my_action_server fibonacci_server
# 终端会显示: "Fibonacci Action Server 已启动."
```
终端 2 (激活环境并运行 Client)
```Bash
source install/setup.bash
ros2 run my_action_client fibonacci_client
```