## 定义
1. 程序在堆上申请了内存，但在使用完毕后，未能按时或没有释放它，导致这块内存无法被操作系统重新分配给其他程序或当前程序的其他部分使用。
2. 这块“丢失了地址”且“无法被再次使用”的内存，就是泄漏的内存。
## 发生原因
### 忘记 delete
```
void memory_leak_simple() {
    int* p = new int(5);
    // 忘记调用 delete p;
} // 函数结束，指针 p 被销毁，但它指向的内存块留在了堆上
```
### 指针被覆盖
在释放内存前，将指针指向了另一块内存或 nullptr
```
void memory_leak_overwrite() {
    int* p1 = new int[10];
    int* p2 = new int(100);
    p1 = p2; // 严重错误！p1 原来指向的数组地址丢失了
             // 现在 p1 和 p2 指向同一个 int(100)
    delete p2;
    delete[] p1; // 错误，因为和 p2 是同一块内存
                 // 并且最初的数组已经彻底泄漏了
}
```
### 函数提前返回
在 new 和 delete 之间，函数因为某些条件判断而提前 return
```
bool process_data() {
    MyObject* obj = new MyObject();
    if (!obj->is_valid()) {
        // 在这里直接返回，没有 delete obj
        return false; // 内存泄漏发生！
    }
    delete obj;
    return true;
}
```
### 抛出异常（最隐蔽）
如果在 new 之后、delete 之前，代码块中抛出了一个异常，而这个异常没有被捕获并妥善处理，那么 delete 语句将永远不会被执行。
```
void memory_leak_exception() {
    MyObject* obj = new MyObject(); 
    try {
        risky_operation(); // 抛出了异常
    } catch (const SpecificException& e) {
        // 捕获了某个特定的异常，但不是 risky_operation 抛出的
        // 异常会继续向上传播
    }
    delete obj; // 这行代码因为异常而永远无法到达
}
```
## 危害
### 性能下降
可用内存逐渐减少，导致操作系统需要更频繁地进行内存交换（使用虚拟内存/Swap），大大降低程序乃至整个系统的运行速度。
### 内存耗尽
对于需要长时间运行的程序（如服务器、后台服务），即使每次泄漏的内存很小，日积月累也会耗尽所有可用内存，最终导致程序因无法申请新内存而崩溃。
### 程序不稳定
内存耗尽也可能导致其他程序申请内存失败，使得整个系统环境变得不稳定。