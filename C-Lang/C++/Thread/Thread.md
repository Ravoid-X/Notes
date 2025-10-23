## 概述
1. C++11 前的库中没有提供和线程相关的类或者接口，编程时 Windows 上需调用 CreateThread 创建线程，Linux下需调用 clone 或者 pthread 线程库的接口函数 pthread_create 来创建线程。
2. 是直接调用系统相关的 API 函数，编写的代码无法跨平台编译运行
3. main 函数所在的线程称为主线程，其余创建的线程称为子线程；在 Linux 环境下线程的本质仍然是进程
# `#include <thread>`
## 创建线程
### 普通函数
```
void task() {
   cout << "Hello from thread!" << endl;
}

thread t(task); 
t.join(); // 等待线程 t 执行完毕
return 0;
```
### Lambda 表达式
```
thread t([]() {
   cout << "Hello from lambda thread!" << endl;
});
t.join();
```
### 函数对象
当一个类重载了函数调用运算符 () 时，就可以像调用函数一样“调用”这个类的对象，称为函数对象。
```
class Task {
public:
   void operator()() const {
      cout << "Hello from functor thread!" << endl;
   }
};

Task my_task;
thread t(my_task);
t.join();
```
### 成员函数
```
class MyClass {
public:
   void run() {
      cout << "Hello from member function thread!" << endl;
   }
};

MyClass obj;
// 第一个参数是成员函数指针，第二个参数是对象实例的地址
thread t(&MyClass::run, &obj);
t.join();
```
## 有参数情况
### 值传递
```
void print_message(const string& message) {
   cout << message << endl;
}

string msg = "Hello with arguments!";
thread t(print_message, msg); // msg 被复制到线程 t
t.join();
```
### 引用传递
1. 对于只移动类型（如 unique_ptr），需要使用 move
2. 通过引用传递，必须使用 ref 或 cref，cref具有常量性
```
void update_value(int& value) {
    value = 100;
}
int val = 10;
thread t(update_value, ref(val)); // 传递 val 的引用
t.join();
```
## 生命周期
### join()
1. 会阻塞当前调用它的线程（例如 main 线程），直到线程 t 执行完成。
2. 确保了在程序继续执行或退出前，所有由线程使用的资源都已处理完毕。
### detach()
1. 会将线程 t 从 thread 对象中分离出去，使其成为一个“守护线程”在后台独立运行。
2. 如果主线程退出，所有分离的线程也会被强制终止，可能导致资源未被正确释放。
## 其他操作
### 硬件信息查询
1. 返回一个 unsigned int，表示硬件支持的并发线程数
2. 通常等于 CPU 的核心数或超线程数
3. 在某些系统或环境下，可能返回 0，表示无法检测到此信息
```
thread::hardware_concurrency();
```
### 线程标识
```
this_thread::get_id(); // 当前正在执行此代码的线程的 ID
thread_object_name.get_id(); //该对象所管理的那个线程的 ID
```
```
void foo() {
    cout << "Hello from foo. My thread ID is: " << this_thread::get_id() << endl;
}

thread t1(foo);
thread t2(foo);
// 打印主线程的 ID
cout << "Main thread ID is: " << this_thread::get_id() << endl;
// 打印 t1 和 t2 对象管理的线程的 ID
cout << "Thread t1's ID is: " << t1.get_id() << endl;
cout << "Thread t2's ID is: " << t2.get_id() << endl;
t1.join();
t2.join();
```
### 线程调度
```
//让当前线程休眠一段时间
this_thread::sleep_for(duration); 
//让当前线程阻塞，直到一个指定的时间点
this_thread::sleep_until(time_point) 
//提示调度器可以将 CPU 时间片让给其他线程
this_thread::yield(); 
```
### 对象管理
```
//返回一个 bool 值，判断一个线程对象是否是可结合的
//线程对象创建并关联到一个正在执行的线程后，还没有被 join() 或 detach() 过即 可结合的
thread_object_name.joinable(); 
```