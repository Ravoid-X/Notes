# static
## 局部变量
1. 在局部变量之前加上 static，就被定义为局部静态变量
2. 存储在静态存储区，只能被初始化一次（默认 0）
3. 在程序执行期间，对应的存储空间不会释放
4. 作用域不变
## 全局变量
1. 定义为全局静态变量，存储在静态存储区，只能被初始化一次（默认 0）
2. 全局变量的作用域是整个源程序，在各个源文件中都是有效
3. 全局静态变量作用域只在定义该变量的源文件内
## 函数
作用域变为在定义该变量的源文件内
## 类
### 成员变量
1. 一样遵从 public，protected，private 访问规则
2. 实际使其成为类的全局变量，会被类的所有对象共享，包括派生类的对象。
3. 必须在类外进行初始化，而不能在构造函数内初始化，除非用 const 修饰。
### 成员函数
1. 所有对象共享该函数，不含this指针。
2. 可以独立访问，即无须创建任何对象实例就可以访问。
3. 非静态成员函数可以任意地访问静态成员函数和静态数据成员
4. 静态成员函数不能访问非静态成员函数和非静态数据成员，只可以相互访问。
5. 不能同时用 const 和 static，C++ 编译器在实现 const 成员函数的时为了确保该函数不能修改类的实例的状态，会在函数中添加一个隐式的参数 const this*。但当一个成员为 static 的时候，该函数是没有 this 指针的。

# volatile
1. 告诉编译器：这个变量的值可能在任何时候被程序（在当前执行流程中）无法感知的外部因素所改变。
2. 阻止编译器对该变量的读/写操作进行优化，确保每次访问该变量时，程序都会强制重新从内存地址读取最新的值，而不是使用 CPU 寄存器中缓存的旧值。
## 没有 volatile
### 示例
```C++
bool device_ready = false;
// 模拟的硬件中断或另一个线程，会在某个时刻改变 device_ready 的值
void simulate_external_change() {
    // 假设这个函数由硬件中断触发
    device_ready = true;
}
void wait_for_device() {
    cout << "Waiting for device..." << endl;
    while (!device_ready) {
        // 循环等待...
    }
}
```
### 编译器的分析 (带优化，例如 -O2)
1. 查看 wait_for_device 函数中的 while (!device_ready) 循环
2. 发现 device_ready 在循环开始时为 false
3. 发现在 while 循环内部，没有任何代码修改 device_ready
4. 编译器假设 device_ready 的值永远不会改变，因此 !device_ready 将永远为 true
5. 优化结果： 编译器将这个循环变成一个死循环 (jmp $)，因为它断定 device_ready 永远不会变为 true
### 结果
1. simulate_external_change() 被硬件触发，它修改的是内存中 device_ready 的值。
2. 但 wait_for_device 函数却卡在了一个只检查 CPU 寄存器中旧值的死循环里
## 有 volatile
### 示例
```C++
volatile bool device_ready = false;

void simulate_external_change() {
    device_ready = true;
}
void wait_for_device() {
    cout << "Waiting for device..." << endl;
    while (!device_ready) {
        // 循环等待...
    }
}
```
### volatile 操作
当 device_ready 被声明为 volatile 时，向编译器发出了两个强制命令
1. 禁止缓存： 不得将此变量的值缓存到寄存器中。每次在代码中访问该变量时，都必须生成一条指令，直接从其内存地址重新读取该值。
2. 禁止重排/消除： 不得对 volatile 变量的访问进行重排序（相对于其他 volatile 访问）或将其消除。如果代码写了 *ptr = 1; *ptr = 2;，编译器不能优化掉第一次写入，必须执行两次写入。