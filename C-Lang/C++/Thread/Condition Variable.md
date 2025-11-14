## 概述
1. 同步原语，用于阻塞一个或多个线程，直到另一个线程修改了一个共享变量（即条件），并通知了 condition_variable。
2. 通常用于解决线程间的等待/通知问题，例如生产者-消费者模型
## 主要成员函数
### wait(lock)
1. 阻塞当前线程，原子性地释放互斥量（通过 lock），并将其加入等待队列。
2. 当收到通知或发生虚假唤醒时，线程被唤醒，并再次原子性地获得互斥量。
3. 此版本不推荐，因为没有检查条件，容易产生虚假唤醒问题
### wait(lock, predicate)
1. 阻塞当前线程，但会先检查一个谓词（predicate，一个可调用对象，返回 bool）。
2. 如果谓词为 true，则不等待，直接返回。如果为 false，则执行等待操作（释放锁并等待通知）。
3. 被唤醒后，会再次检查谓词。这是推荐的用法，可以避免虚假唤醒问题
### notify_one()
唤醒一个正在等待的线程
### notify_all()
唤醒所有正在等待的线程
## 示例
```C++
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <string>
#include <chrono>
using namespace std;

// 1. 互斥量和条件变量
mutex mtx;
condition_variable cv;
// 2. 共享数据（条件）
queue<int> data_queue;
bool done = false; // 退出标志
// 生产者函数
void producer() {
    for (int i = 1; i <= 5; ++i) {
        this_thread::sleep_for(chrono::milliseconds(500)); 
        {   // 生产者需要锁定互斥量来修改共享数据
            lock_guard<mutex> lock(mtx);
            data_queue.push(i);
            cout << "Producer produced: " << i << endl;
        } 
        // 生产完数据后，通知一个等待的消费者线程
        cv.notify_one(); 
    }
    // 设置退出标志并再次通知，确保消费者能退出循环
    {
        lock_guard<mutex> lock(mtx);
        done = true;
    }
    // 通知所有等待线程（尽管只有一个消费者，使用 notify_all 更安全）
    cv.notify_all(); 
}
// 消费者函数
void consumer() {
    int consumed_count = 0;
    while (!done || !data_queue.empty()) {
        unique_lock<mutex> lock(mtx);
        // wait 会原子性地：
        // 1. 检查 lambda 表达式（谓词）。如果为 true，则不等待，继续执行。
        // 2. 如果为 false，则释放互斥量 `lock` 并阻塞线程。
        // 3. 当被 notify_one/all 唤醒或虚假唤醒时，它会再次原子性地获得互斥量。
        // 4. 再次检查 lambda 表达式。如果为 true，返回；否则，继续等待。
        cv.wait(lock, [&]{ 
            return !data_queue.empty() || done; 
        });
        // 退出循环的条件：生产者完成 AND 队列为空
        if (done && data_queue.empty()) {
            cout << "Consumer finished, consumed " << consumed_count << " items." << endl;
            break;
        }
        // 此时，lock 仍处于锁定状态，可以安全地访问共享数据
        if (!data_queue.empty()) {
            int data = data_queue.front();
            data_queue.pop();
            consumed_count++;
            cout << "Consumer consumed: " << data << endl;
        }
    }
}
int main() {
    thread t1(producer);
    thread t2(consumer);
    t1.join();
    t2.join();
    cout << "All threads finished." << endl;
    return 0;
}
```
### 解释
1. 谓词（Lambda 表达式 [&]{ return !data_queue.empty() || done;}）：定义了线程应该继续执行的条件。
2. 如果谓词为 true（队列有数据或已完成），线程不会阻塞，继续执行后续代码
3. 如果谓词为 false，会原子性地执行两个操作：释放 lock 持有的互斥量；使线程进入阻塞状态，等待被通知
4. 谓词机制确保了即使线程被虚假唤醒，如果条件仍不满足，线程会再次自动进入等待状态