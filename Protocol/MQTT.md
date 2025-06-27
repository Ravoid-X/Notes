# 简介
&emsp;&emsp;MQTT（Message Queuing Telemetry Transport，消息队列遥测传输协议）是基于 Publish/Subscribe 模式的"轻量级"物联网通信协议，构建于TCP/IP协议上，具有简单易实现、支持 QoS、报文小等特点.作为一种低开销、低带宽占用的即时通讯协议，在物联网、小型设备、移动应用等方面有较广泛的应用。
# Qos消息服务质量机制
MQTT中QoS是客户端与服务器之间的。消息在发布者和broker之间定义了Qos,在订阅者和broker之间定义了Qos，最终这条消息的Qos是取这两者之间的最低值
## Qos0
“至多一次”，消息发布完全依赖底层TCP/IP网络，会发生消息丢失或重复\
这一级别可用于如下情况，传感器数据，丢失一次记录影响不大\
<img src="../Pic/Protocol/Qos0.jpg" style="width:400px;padding:10px;"/>

## Qos1
“至少一次”，确保消息到达，但消息重复可能会发生\
<img src="../Pic/Protocol/Qos1.jpg" style="width:400px;padding:10px;"/>

## Qos2
“只有一次”，确保消息到达一次\
这一级别可用于如下情况，在计费系统中，消息重复或丢失会导致不正确的结果。还可以用于即时通讯类的APP的推送，确保用户收到且只会收到一次\
<img src="../Pic/Protocol/Qos2.jpg" style="width:400px;padding:10px;"/>

# 数据包结构
一个MQTT数据包由：固定头（Fixed header）、可变头（Variable header）、消息体（payload）三部分构成
## 固定头
<img src="../Pic/Protocol/MQTT-fixed-header.jpg" style="width:500px;padding:10px;"/>

包含首字节(字节1)和剩余消息报文长度(从第二个字节开始，长度为1-4字节)，剩余长度是当前包中剩余内容长度的字节数，包括变量头和有效负载中的数据
1. 数据包类型\
<img src="../Pic/Protocol/MQTT-.jpg" style="width:400px;padding:10px;"/>

2. 标志位\
<img src="../Pic/Protocol/MQTT.jpg.jpg" style="width:400px;padding:10px;"/>

## 可变头

## 消息体
