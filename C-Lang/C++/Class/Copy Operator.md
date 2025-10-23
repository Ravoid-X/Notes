## 示例
```
class Widget{
public:
    Widget& operator=(const Widget& rhs);//赋值运算符
    Widget& operator=(Widget&& rhs);//移动赋值运算符
private：
    int width；
    int height；
    int *p；
}
Widget a, b;
a = b; // 调用拷贝赋值运算符
b = Widget(); // 调用移动赋值运算符( Widget() 是右值)
```
## 赋值运算符
1. 将一个对象的内容拷贝到另一个已经存在的对象中，是深拷贝。
2. 如果没有显式定义，编译器会自动生成。默认的运算符会执行成员级别的逐个拷贝。
3. 对于指针成员，这只会拷贝指针的地址，会导致悬挂指针等问题。
### 代码
```
Widget& Widget::operator=(const Widget& rhs) { 
    // 防止自我赋值 
    // 如果没有检查，后续释放内存再访问就会导致程序崩溃。
    if (this == &rhs) {
        return *this;
    }
    // 释放当前对象（*this）已经拥有的资源
    delete[] p;
    // 为当前对象分配新的资源
    width = rhs.width;
    height = rhs.height;
    p = new int[width];
    // 将源对象的数据复制到新分配的资源中
    for (int i = 0; i < width; ++i) {
        p[i] = rhs.p[i];
    }
    // 返回对当前对象的引用，以支持链式赋值 (a = b = c)
    return *this;
}
```
## 移动赋值运算符
1. 从一个对象中转移资源到另一个对象，而不是进行复制操作。
2. 通常与右值引用一起使用，以实现高效的资源转移，提高性能。
### 代码
```
Widget& Widget::operator=(Widget&& rhs) { 
    // 防止自我赋值 (a = std::move(a);)
    if (this == &rhs) {
        return *this;
    }
    // 释放当前对象（*this）已经拥有的资源
    delete[] p;
    // 直接将 rhs 的成员变量赋值过来
    // 不是数据复制，仅仅是指针地址的复制，速度极快。
    width = rhs.width;
    height = rhs.height;
    p = rhs.p;
    // 将源对象（rhs）置于“空”状态
    rhs.width = 0;
    rhs.height = 0;
    rhs.p = nullptr;
    // 返回对当前对象的引用
    return *this;
}
```