## 概念
1. DDS 是一个由 OMG (Object Management Group, 也是 UML 和 CORBA 的制定者) 标准化的中间件协议。
2. 它并非专为 ROS 设计，而是在工业、军事、航空航天、金融等领域广泛使用的、高性能的工业级标准。
## 数据中心
1. 去中心化，没有服务器，这是 DDS 最核心的原理，也是它与 MQTT 或 AMQP 等其他中间件的根本区别
2. DDS 创造了一个叫做 “全局数据空间” 的抽象概念，网络上的所有节点（应用程序）都在共同查看这个全局数据空间
3. 生产者和消费者在时间（无需同时在线）、空间（无需知道对方 IP）和流（无需同步收发）上完全解耦，DDS 关心的是状态而不是事件
## 实体模型
### Domain (域)
1. 一个逻辑隔离的通信网络。可以将其视为一个虚拟局域网，只有在同一个 Domain ID 下的 DDS 实体才能相互发现和通信
2. 在 ROS 2 中，这由环境变量 ROS_DOMAIN_ID 控制（默认为 0）。这是实现多机器人系统在同一局域网下互不干扰的基础
### DomainParticipant (域参与者)
1. 应用程序（即 ROS 2 节点）进入 DDS Domain 的入口
2. rclcpp::Node 实例在底层就对应一个 DomainParticipant，负责创建下面的 Publisher 和 Subscriber
### Topic (话题)
1. 全局数据空间中数据的唯一标识符，由名称和数据类型唯一定义
2. .msg 和 .srv 文件在编译时，会通过 rosidl 工具链自动转换成 DDS 能理解的 IDL 文件，然后再由 IDL 编译器生成特定语言（C++ 或 Python）的数据结构和序列化/反序列化代码
### Publisher (发布者) & DataWriter (数据写入器)
1. Publisher 是一个组织容器，它负责管理一个或多个 DataWriter，DataWriter 才是真正将数据写入全局数据空间的实体
2. 当创建一个 rclcpp::Publisher 时，DDS 底层会创建一个 Publisher 实体和一个与之关联的 DataWriter
### Subscriber (订阅者) & DataReader (数据读取器)
1. Subscriber 是管理 DataReader 的容器，DataReader 负责从全局数据空间读取数据
2. rclcpp::Subscription 对应底层的 Subscriber 和 DataReader
## 自动发现
使用一个名为 RTPS (Real-Time Publish-Subscribe) 协议的子协议（特别是 SPDP - Simple Participant Discovery Protocol）来进行自动发现，是 ROS2 图能自动形成的原理。工作流程如下
### 启动
1. 当一个 DomainParticipant（ROS2 节点）启动时，它会开始在预定义的多播地址和端口上广播自己的存在，也会在这些端口上监听，以发现其他 Participant
2. 网络上所有已存在的其他 Participant 都会收到这个消息，并将其记录在案
### 发现
1. 假设一个 Subscriber (DataReader) 启动了，广播：“我是 S1，我在 Domain 0，我想订阅 Topic T (类型为 D)，我要求的 QoS 是 Q_S”
2. 过了一会儿，一个 Publisher (DataWriter) 启动了，广播：“我是 P1，我在 Domain 0，我可以发布 Topic T (类型为 D)，我提供的 QoS 是 Q_P”
### 匹配
P1 和 S1 互相听到了对方的广播。它们会进行检查 Domain ID、Topic 名称、Topic 类型和 QoS 策略是否兼容
### 建立连接
1. 如果所有检查都通过，P1 和 S1 就会交换它们用于数据传输的单播地址（IP 和端口）
2. 此后，P1 会绕开多播，直接通过一个高效的（通常是 UDP）单播连接将数据发送给 S1