## 目标
将所有与中间件无关的、但对 ROS2 客户端（C++, Python 等）至关重要的“核心逻辑和状态管理”提取到一个单一的、共享的、用 C 语言实现的库中
## 概念
rcl 是一个 C 库，没有类或对象。它使用 C 语言的标准模式：不透明的句柄或结构体来管理状态。
### 创建 Node
当在 rclcpp 中创建一个 Node 时，rclcpp 会在内部
1. 调用 rcl_node_init() 来初始化一个 rcl_node_t 结构体
2. rcl_node_t 内部会保存这个节点的状态，例如它的名字、命名空间、以及它所拥有的发布者、订阅者列表
3. rclcpp 的 Node 类只是对这个 C 语言的 rcl_node_t 句柄进行包装，提供了 C++ 的面向对象接口（如方法、析构函数等）
### rcl_..._t 句柄
1. Context (上下文)：rcl_context_t，最重要的句柄，代表了整个 ROS2 的初始化状态和全局资源。rcl_init() 的核心就是初始化这个上下文
2. Node (节点): rcl_node_t
3. Publisher (发布者): rcl_publisher_t
4. Subscription (订阅者): rcl_subscription_t
5. Timer (定时器): rcl_timer_t
6. Wait Set (等待集): rcl_wait_set_t
7. Guard Condition (守卫条件): rcl_guard_condition_t （用于手动唤醒执行器）
## 周期管理
rcl 负责管理所有 ROS 实体的生命周期，以创建一个发布者 (rcl_publisher_init) 为例，rcl 在其内部会执行以下操作
1. 分配句柄：分配一个 rcl_publisher_t 结构体的内存
2. 参数校验：检查传入的 QoS 策略是否有效
3. 调用 RMW：调用 rmw_create_publisher()，请求 RMW 层（及底层的 DDS）在网络上实际创建一个发布者
4. 保存 RMW 句柄：RMW 层会返回一个 rmw_publisher_handle_t。rcl 将这个 RMW 句柄保存在自己的 rcl_publisher_t 结构体中
5. 状态保存：rcl 还会保存 QoS 设置、主题名称等信息
> 销毁 (rcl_publisher_fini) 的过程则相反，先调用 rmw_destroy_publisher()，然后再释放 rcl_publisher_t 的内存
## 核心逻辑实现
rcl 实现了那些与通信协议无关，但对 ROS2 至关重要的功能。最好的例子就是参数
1. rclcpp::Node 提供的强大的参数功能（设置、获取、描述、回调）的所有逻辑，都在 rcl 的 rcl_params 库中实现
2. rcl 内部实现了一个参数树的状态机
3. 当一个节点设置一个参数时，rclcpp 会调用 rcl 的函数
4. rcl 会处理这个请求，然后使用常规的 rcl_publish_t 和 rcl_service_t（由 rcl 自己创建和管理）来发布参数事件 (/parameter_events) 和提供“参数服务” (/set_parameters, /get_parameters)
5. rclpy 节点也能使用参数，因为它调用的也是同一套 rcl C API，因此保证了行为的绝对一致
## 等待集
rcl 最复杂、最精妙的部分，也是理解 spin() 和执行器的关键
1. ROS 节点需要同时处理多种事件，如订阅 A 上有新消息、定时器 T 到时了、服务 S 接到了一个新请求等等
2. RMW 层提供了一个原始的 rmw_wait() 函数，可以等待一组底层的 DDS 实体（rmw_subscription_t, rmw_service_t 等）
3. rcl 在此之上构建了一个更高级的概念：rcl_wait_set_t (等待集)
### rcl 职责
1. rcl_wait_set_init()：初始化一个等待集。
2. rcl_wait_set_add_subscription()：将一个 rcl 订阅句柄添加到等待集。
3. rcl_wait_set_add_timer()：将一个 rcl 定时器句柄添加到等待集。
4. rcl_wait_set_add_service()：...
5. rcl_wait()：执行等待，rcl 会在内部收集所有被添加实体的 RMW 句柄，并调用一次 rmw_wait()。这个调用会阻塞，直到任何一个事件准备就绪。
5. 检查状态：rcl_wait() 返回后，rcl 会检查等待集，标记哪些句柄是就绪的（例如，rcl_wait_set_t 的 subscriptions[0] 已就绪）
## 执行器
1. rcl 本身不提供执行器，rclcpp 和 rclpy 中才有执行器的实现（如 SingleThreadedExecutor）
2. 但 rclcpp 中的 Executor 的 spin() 循环，其核心逻辑完全是建立在 rcl 提供的等待集原语之上的
3. 一个 rclcpp::Executor::spin() 的简化逻辑如下
```伪代码
while (rcl_context_is_valid(context)) {
    // 1. (rcl) 创建一个空的等待集
    rcl_wait_set_t wait_set = rcl_get_zero_initialized_wait_set();
    rcl_wait_set_init(&wait_set, ...);
    // 2. (rclcpp) 遍历所有已"添加"到执行器的节点和回调
    //    (rcl) 将它们全部添加到等待集中
    foreach (subscription in executor) {
        rcl_wait_set_add_subscription(&wait_set, subscription->get_subscription_handle());
    }
    foreach (timer in executor) {
        rcl_wait_set_add_timer(&wait_set, timer->get_timer_handle());
    }
    // ... 添加 services, clients, actions ...
    // 3. (rcl) 调用 rcl_wait()，阻塞线程，等待事件发生
    //    这在内部会调用 rmw_wait()
    rcl_wait(&wait_set, timeout);
    // 4. (rclcpp/rcl) 循环检查哪些事件准备好了
    for (int i=0; i < wait_set.size_of_subscriptions; ++i) {
        if (wait_set.subscriptions[i]) {
            // 5. (rcl) 从 RMW 队列中取出数据
            rcl_take(wait_set.subscriptions[i], &msg, ...);
            // 6. (rclcpp) 执行用户提供的 C++ 回调函数
            user_callback(msg);
        }
    }
    for (int i=0; i < wait_set.size_of_timers; ++i) {
        if (wait_set.timers[i]) {
            // 5. (rcl) 调用定时器
            rcl_timer_call(wait_set.timers[i]);
            // 6. (rclcpp) 执行用户的 C++ 定时器回调
            user_timer_callback();
        }
    }
    // ... 处理 services, clients ...
    // 7. (rcl) 清理等待集
    rcl_wait_set_fini(&wait_set);
}
```