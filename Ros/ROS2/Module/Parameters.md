## 特性
在 ROS1 中，所有参数都存储在 roscore 上的一个参数服务器中。节点通常在启动时拉取一次参数，运行时很少更改，ROS2 彻底改变了这一点
1. 节点归属：每个节点都有自己的参数服务器。参数不再是全局的，而是属于特定的节点。
2. 分布式：参数的状态和逻辑存在于各个节点内部，没有中心化的 roscore 来管理它们。
3. 动态性：可以在运行时被外部（如命令行工具或其他节点）安全地读取和更改。
4. 强类型：参数具有明确的类型（如 bool, int, double, string 及其数组）。
5. 自省与通知：节点可以响应对其参数的更改请求（接受或拒绝）；系统会通过一个全局话题（/parameter_events）广播所有参数的变更事件
## 参数服务
当一个 rclcpp::Node 被创建时，它通过 rcl 层会自动创建一组服务，这些服务默认对网络可见
### 服务列表
这些服务都以节点名称为前缀（例如节点名是 my_controller）
1. /<node_name>/set_parameters (类型: rcl_interfaces/srv/SetParameters)：允许外部请求更改该节点的一个或多个参数。
2. /<node_name>/get_parameters (类型: rcl_interfaces/srv/GetParameters)：允许外部读取该节点的一个或多个参数的值。
3. /<node_name>/list_parameters (类型: rcl_interfaces/srv/ListParameters)：允许外部查询该节点当前声明了哪些参数。
4. /<node_name>/describe_parameters (类型: rcl_interfaces/srv/DescribeParameters)：获取参数的“元数据”（描述、类型、约束范围等）
### 工作流程
以 ros2 param set 为例
1. 在终端运行 ros2 param set /my_controller kp 1.5
2. ros2 命令行工具（本身也是一个临时的 ROS2 节点）会成为一个服务客户端
3. 它在网络上查找名为 /my_controller/set_parameters 的服务服务端
4. my_controller 节点（作为服务端）接收到请求（"请将参数 'kp' 设置为 1.5"）
5. my_controller 节点内部的参数逻辑开始工作
6. 如果设置成功，节点返回一个成功的响应；如果失败（例如被回调拒绝），则返回失败响应
7. ros2 param set 命令在终端打印出结果
## 参数回调
当 set_parameters 服务被调用时，节点不会立即接受新值，而是触发一个参数设置回调（如果用户注册了的话）。这个回调函数在 rclcpp 和 rclpy 中都可以定义，职责如下
1. 接收请求更改的参数列表（例如 [{"name": "kp", "value": 1.5}, {"name": "ki", "value": -0.1}]）
2. 逐个审查这些更改，决定是接受还是拒绝这个更改
3. 返回一个结果对象，告诉参数服务：“接受了 'kp' 的更改，但我拒绝了 'ki' 的更改，因为它不能为负数。”
## 全局通知
当一个节点最终（在通过回调验证后）接受了一个参数更改时，会做两件事
1. 更新自己内部的参数值
2. 在全局的 topics:/parameter_events（类型: rcl_interfaces/msg/ParameterEvent）上发布一条消息
### 消息内容
1. 发生更改的节点的全名
2. 被更改的参数名称
3. 参数的新值
## 示例
```C++
#include "rclcpp/rclcpp.hpp"
#include "rcl_interfaces/msg/set_parameters_result.hpp"

using std::placeholders::_1;

class MyParamNode : public rclcpp::Node{
public:
    MyParamNode() : Node("my_param_node"){
        // 1. 声明参数并设置默认值
        // declare_parameter 会返回参数的当前值（如果已在外部设置）或默认值
        this->declare_parameter<double>("kp", 1.0);
        this->declare_parameter<int>("max_iterations", 100);
        this->declare_parameter<bool>("use_ekf", true);
        // 2. (可选) 声明一个带“描述符”的参数 (用于约束)
        rcl_interfaces::msg::ParameterDescriptor kp_desc;
        kp_desc.description = "Proportional gain for the controller";
        rcl_interfaces::msg::FloatingPointRange kp_range;
        kp_range.set__from_value(0.0).set__to_value(100.0); // 必须在 0.0 到 100.0 之间
        kp_desc.floating_point_range.push_back(kp_range);
        this->declare_parameter<double>("kp_with_desc", 2.0, kp_desc);
        // 3. 注册“参数设置回调” (最关键的部分)
        set_param_callback_handle_ = this->add_on_set_parameters_callback(
            std::bind(&MyParamNode::parameters_callback, this, _1));
        // 4. 在需要时获取参数值
        // 最好不要在构造函数中获取，因为此时参数可能还未被外部（如 launch）设置
        // 这里只是演示 API
        double kp_value = this->get_parameter("kp").as_double();
        RCLCPP_INFO(this->get_logger(), "Initial kp value: %f", kp_value);
    }
private:
    // 5. 参数回调函数的实现
    rcl_interfaces::msg::SetParametersResult 
    parameters_callback(const std::vector<rclcpp::Parameter>& parameters)
    {
        rcl_interfaces::msg::SetParametersResult result;
        result.successful = true; // 默认假设成功

        for (const auto& param : parameters)
        {
            if (param.get_name() == "max_iterations")
            {
                if (param.get_type() == rclcpp::ParameterType::PARAMETER_INTEGER)
                {
                    if (param.as_int() <= 0) {
                        RCLCPP_WARN(this->get_logger(), "Rejecting max_iterations <= 0");
                        result.successful = false;
                        result.reason = "max_iterations must be positive";
                    } else {
                        RCLCPP_INFO(this->get_logger(), "Accepting new max_iterations: %ld", param.as_int());
                    }
                }
            }
            // 可以在这里检查 "kp_with_desc"，但描述符的范围检查目前更多是“建议性”的，真正的强制约束在回调中实现最可靠)
        }
        return result;
    }
    OnSetParametersCallbackHandle::SharedPtr set_param_callback_handle_;
};
```
## 命令行
假设节点（例如 my_param_node）正在运行
### 列出节点的所有参数
```Bash
ros2 param list /my_param_node
```
### 获取特定参数的值
```Bash
ros2 param get /my_param_node kp
```
### 运行时设置参数
```Bash
# 尝试一个有效值
ros2 param set /my_param_node max_iterations 50
```
>节点日志会输出： Accepting new max_iterations: 50
```Bash
# 尝试一个会被回调拒绝的无效值
ros2 param set /my_param_node max_iterations -5
```
>节点日志会输出：Rejecting max_iterations <= 0 ros2 param set 命令会报告失败： Setting parameter failed: [parameters_callback] failed to set parameter 'max_iterations': max_iterations must be positive
### 查看参数描述
```Bash
ros2 param describe /my_param_node kp_with_desc
```