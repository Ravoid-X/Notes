`#include <atomic>`
## 概述
1. 原子变量是一种特殊的数据类型，支持各种数据类型，如整数、布尔值、指针等。用于执行原子操作。
2. 原子操作是不可分割的操作，可以确保在多线程环境中线程安全地执行。
3. 提供了一种线程安全的方式来访问和修改共享数据，而无需使用显式的互斥锁。
## 使用
### 创建
```
atomic<int> atomicInt(0);
atomic<bool> atomicBool(true);
```
### 读取
```
int value = atomicInt.load();
bool flag = atomicBool.load();
```
### 修改
```
atomicInt.store(42);
atomicBool.store(false);
```
### 加法举例
```
atomic<int> atomicValue(0);
int increment = 5;
int result = atomicValue.fetch_add(increment);
```
## 常见应用场景
### 计数器
在多线程环境中,可以使用 fetch_add 和 fetch_sub 来安全地增减计数器的值。
```
atomic<int> counter(0);
counter.fetch_add(1);
// 线程2减少计数器
counter.fetch_sub(1);
```
### 控制标志
用于控制线程的启动和停止，可以使用 load 和 store 来读取和修改标志的状态。
```
atomic<bool> flag(true);
if (flag.load()) {
    // 执行操作
}
flag.store(false);
```
### 链表和数据结构
此例中两个线程同时增加一个原子计数器的值，而不需要显式的互斥锁
```
atomic<int> atomicCounter(0);
void incrementCounter() {
    for (int i = 0; i < 10000; ++i) {
        atomicCounter.fetch_add(1);
    }
}
thread t1(incrementCounter);
thread t2(incrementCounter);
t1.join();
t2.join();
return 0;
```