## 概述
1. 竞争条件：多个线程同时访问和修改同一个共享资源，可能会导致数据不一致、程序崩溃等问题。
2. 临界区：只有一个线程能够访问该共享资源的代码区域。锁用于保护临界区。
## 实际使用
1. 很少直接调用锁的 lock() 和 unlock() 方法，因为若在加锁和解锁之间发生异常，可能会导致锁无法被释放，从而造成死锁。
2. C++ 标准库推荐使用 RAII (Resource Acquisition Is Initialization) 风格的锁管理器，如 lock_guard、unique_lock 和 shared_lock。
3. 这些管理器在构造时自动加锁，在析构时自动解锁，是更安全、更现代的方法。
# 分类
## 互斥锁
最基础的锁类型，一个线程获取了锁，其他线程就必须等待。
### 潜在问题
1. 如果一个线程在锁定后，因为异常或逻辑错误而没有执行 unlock()，会造成死锁。
2. 同一个线程对同一个 mutex 对象连续调用两次 lock()，会造成死锁。
### 实现
`std::mutex`
### 示例
```
mutex mtx; // 全局互斥锁实例
long long shared_counter = 0;

void increment() {
    for (int i = 0; i < 100000; ++i) {
        mtx.lock(); // 加锁
        // --- 临界区开始 ---
        shared_counter++;
        // --- 临界区结束 ---
        mtx.unlock(); // 解锁
    }
}

int main() {
    vector<thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.push_back(thread(increment));
    }
    for (auto& th : threads) {
        th.join();
    }
    // 应该是 1,000,000
    cout << "Final value: " << shared_counter << endl; 
    return 0;
}
```
## 自旋锁
非阻塞锁。当一个线程尝试获取自旋锁失败时，它不会被挂起，而是在一个循环中忙等待，不断地检查锁是否被释放。
### 特点
1. 优点：对于锁持有时间极短的情况，忙等待的开销可能比线程上下文切换（挂起和唤醒）的开销要小，因此性能更高。
2. 缺点：如果锁持有时间较长，自旋锁会持续消耗 CPU 资源，造成浪费。在单核 CPU 上，自旋锁没有任何优势，甚至会导致系统无法正常工作。
### 实现
`std::atomic_flag` 或 `std::atomic<bool>`
### 示例
```
class Spinlock {
private:
    atomic_flag flag = ATOMIC_FLAG_INIT;

public:
    void lock() {
        // test_and_set: 如果为 false 就设为 true 并返回 false
        // 如果为 true，就直接返回 true
        // 当循环退出时，说明 flag 之前是 false，现在被为 true，成功获取锁
        while (flag.test_and_set(memory_order_acquire)) {
            // 忙等待
        }
    }
    void unlock() {
        // 将 flag 清除为 false，释放锁
        flag.clear(memory_order_release);
    }
};

Spinlock spin;
long long counter = 0;

void spin_increment() {
    for (int i = 0; i < 100000; ++i) {
        spin.lock();
        counter++;
        spin.unlock();
    }
}

int main() {
    vector<thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(spin_increment);
    }
    for (auto& th : threads) {
        th.join();
    }
    // 预计结果 1000000 = 10 * 100000
    cout << "Final counter with spinlock: " << counter << endl;
    return 0;
}
```
## 读写锁
1. 允许多个线程同时进行读操作，但只允许一个线程进行写操作。
2. 读-读共享，写-写互斥，读-写互斥。
### 实现
`std::shared_mutex`
### 示例
```
shared_mutex shared_mtx;
int shared_data = 0;

void reader(int id) {
    shared_lock<shared_mutex> lock(shared_mtx); // 获取共享锁 (用于读)
    cout << "Reader " << id << " reads data: " << shared_data << endl;
    this_thread::sleep_for(chrono::milliseconds(500)); // 模拟读操作
}

void writer(int id) {
    unique_lock<shared_mutex> lock(shared_mtx); // 获取独占锁 (用于写)
    shared_data++;
    cout << "Writer " << id << " writes data: " << shared_data << endl;
    this_thread::sleep_for(chrono::milliseconds(1000)); // 模拟写操作
}

int main() {
    thread t_writer(writer, 1);
    this_thread::sleep_for(chrono::milliseconds(100)); // 确保写者先启动
    vector<thread> readers;
    for (int i = 0; i < 5; ++i) {
        readers.emplace_back(reader, i + 1);
    }
    t_writer.join();
    //几乎可以同时打印信息
    for (auto& t : readers) {
        t.join();
    }
    return 0;
}
```
## 递归锁
1. 允许同一个线程多次获取同一个锁而不会造成死锁。
2. 内部会维护一个计数器，lock() 操作会使计数器加 1，unlock() 操作会使其减 1。
3. 只有当计数器归零时，其他线程才能获取该锁。
### 实现
`std::recursive_mutex`
### 示例
```
recursive_mutex rec_mtx;
int value = 0;

void recursive_function(int count) {
    // 使用 lock_guard 保证安全
    lock_guard<recursive_mutex> lock(rec_mtx); 
    cout << "Thread " << this_thread::get_id() << " locked. Count: " << count << endl;
    value++;
    // 递归调用，会再次尝试加锁
    if (count > 0) {
        recursive_function(count - 1); 
    }
    cout << "Thread " << this_thread::get_id() << " unlocking. Count: " << count << endl;
}

int main() {
    thread t1(recursive_function, 3);
    t1.join();
    // 如果是 mutex，t1 线程会在第二次 lock 时立即死锁。
    cout << "Final value: " << value << endl; // 应该是 4
    return 0;
}
```
# RAII 锁管理器
## std::lock_guard
1. 互斥锁的封装，在构造时自动对传入的互斥体加锁，在析构时自动解锁。
2. 没有 lock() 和 unlock() 成员函数，一旦创建就锁定，直到销毁。、
### 示例
1. 在上面递归锁的示例中，lock_guard的作用域是整个函数，再举一个例子
```
//其他线程至少等待 50 毫秒才能获取锁，即使真正需要锁的操作只是一瞬间。
mutex mtx;
long long shared_counter = 0;
void process_data_bad() {
    // lock_guard 在函数开始时创建，锁定了整个函数
    lock_guard<mutex> guard(mtx);
    // 步骤 1: 准备数据 (不需要锁)
    int data_to_process = 1;
    // 步骤 2: 更新共享资源 (这是唯一需要锁的部分)
    shared_counter += data_to_process;
    // 步骤 3: 耗时的 I/O 操作 (不需要锁)
    this_thread::sleep_for(chrono::milliseconds(50));
}
```
2. 改进方法，使用{}
```
void process_data_good() {
    // 步骤 1: 准备数据 (在锁外部执行，不影响其他线程)
    int data_to_process = 1;
    // 使用花括号创建一个新的作用域
    {   // lock_guard 在这个作用域开始时创建，获取锁
        lock_guard<mutex> guard(mtx);
        // 步骤 2: 更新共享资源 (这是临界区)
        shared_counter += data_to_process;
    } // guard 在这个花括号结束时就被析构，锁立刻被释放
    // 步骤 3: 耗时的 I/O 操作 (在锁外部执行，不会阻塞其他线程)
    this_thread::sleep_for(chrono::milliseconds(50));
}
```
## unique_lock 和 shared_lock
在读写锁部分展示