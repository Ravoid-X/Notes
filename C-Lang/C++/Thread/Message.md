## 概述
1. 是一个用于在线程之间安全地传递数据（“消息”）的中间缓冲区。
2. 作为生产者和消费者之间的桥梁，它必须是线程安全的。
## 优势
1. 解耦：生产者和消费者之间没有直接的依赖关系，它们只与消息队列交互。
2. 异步处理：生产者线程可以将一个耗时的任务（例如，写文件、复杂计算）封装成消息放入队列，然后立即返回，继续执行其他工作。
3. 缓冲与削峰填谷：当生产者的生产速度和消费者的处理速度不匹配时，消息队列可以起到缓冲作用。
4. 线程池的任务分发：实现线程池的核心组件。生产者将任务放入队列，池中的多个消费者竞争性地从队列中获取任务并执行。
## 实现
1. queue：作为底层的数据结构，用于存储消息，先进先出。
2. mutex：互斥锁，用于保护 queue，对它的访问（push, pop等）都必须在锁的保护下进行。
3. condition_variable：条件变量，这是实现高效等待和唤醒的关键。\
（1）当消费者发现队列为空时，不应该“忙等待”。应该使用条件变量进入睡眠/等待状态，并释放互斥锁，让其他线程可以工作。\
（2）当生产者向队列中放入一个新消息后，它需要通知正在等待的消费者
## 示例
### message.h
```
#pragma once
#include <iostream>
#include <string>
#include <queue>
#include <thread>
#include <mutex>

struct Message {
   int id;
   string content;
};

class MessageQueue {
public:
   MessageQueue() = default;
   void pushMessage(const Message& msg);
   // 从 UI 进程接收消息
   void receiveMessageFromUI(int id, const string& content);
   // 尝试从队列获取一条消息（非阻塞）
   bool tryGetMessage(Message& msg_out);
   bool isEmpty() const;
private:
   // 使用 mutable 关键字允许在 const 成员函数中锁定互斥锁
   mutable mutex m_mutex; 
   queue<Message> m_queue;
};
```
### message.cpp
```
#include "message.h"
#include <chrono>
void MessageQueue::pushMessage(const Message& msg) {
   unique_lock<mutex> lock(m_mutex);
   m_queue.push(msg);
}

void MessageQueue::receiveMessageFromUI(int id, const string& content) {
   Message msg = {id, content};
   unique_lock<mutex> lock(m_mutex);
   m_queue.push(msg);
}

bool MessageQueue::tryGetMessage(Message& msg_out) {
   unique_lock<mutex> lock(m_mutex);
   if (m_queue.empty()) {
      return false;
   }
   msg_out = m_queue.front();
   m_queue.pop();
   return true;
}

bool MessageQueue::isEmpty() const {
   unique_lock<mutex> lock(m_mutex);
   return m_queue.empty();
}
```
### main.cpp
```
#include "message.h"
#include <vector>
#include <atomic>

int main() {
   State state;
   MessageQueue messageQueue;

   while (true) {
      Message received_msg;
      if (messageQueue.tryGetMessage(received_msg)) {
         state.handleMessage(received_msg);
      } 
      // 队列为空，短暂休眠，防止CPU占用过高
      this_thread::sleep_for(chrono::milliseconds(50));
   }
   return 0;
}
```
### 编译
必须加上 `-pthread`