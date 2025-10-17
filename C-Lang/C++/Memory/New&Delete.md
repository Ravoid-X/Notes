## operator new 和 operator delete
C++ 语言标准库的库函数，原型分别如下
```
void *operator new(size_t);     //allocate an object
void *operator delete(void *);    //free an object

void *operator new[](size_t);     //allocate an array
void *operator delete[](void *);    //free an array
```
## new
用于单个对象
### 操作
1. 在堆上分配足够容纳该对象的内存空间。
2. 调用该对象的构造函数在这块内存上初始化对象
### 示例
```
int* p_int = new int(10);
```
## new[]
### 操作
1. 在堆上分配能容纳指定数量对象的大块内存。
2. （关键点） 在这块内存的开始位置（通常是前几个字节），额外存储数组的大小。
3. 循环调用默认构造函数，在内存块上依次初始化数组中的每一个对象。
### 示例
```
int* p_int_array = new int[5]; // 分配 5 个int的内存
MyClass* p_obj_array = new MyClass[3];
```
## delete
用于对象数组
### 操作
1. 调用该对象的析构函数清理资源。
2. 将该对象占用的内存返还给堆。
### 示例
```
delete p_int;   // 释放p_int指向的内存
p_int = nullptr; // 防止悬空指针
```
## delete[]
### 操作
1. 读取内存块头部的数组大小信息，确定有多少个对象需要被销毁。
2. 逆序循环调用数组中每一个对象的析构函数（从最后一个元素到第一个）。
3. 将整个内存块（包括存储数组大小的头部）返还给堆。
### 示例
```
delete[] p_int_array;
p_int_array = nullptr;
```
## 混用后果
### 用 delete 释放 new[] 分配的数组
1. delete 不知道这是一个数组，不会去读取头部的数组大小信息。
2. 只有数组的第一个元素 (p_array[0]) 的析构函数被调用了。
3. p_array[1] 和 p_array[2] 的析构函数完全被跳过，导致它们管理的资源（如文件句柄、网络连接、其他内存）全部泄漏。
4. delete 尝试释放内存，但由于 new[] 可能分配了额外的头部空间，delete 释放的内存大小或地址可能是错误的，这会破坏堆的结构，极易导致程序在后续操作中崩溃。