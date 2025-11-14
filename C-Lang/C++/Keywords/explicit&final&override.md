## final
主要有两个目的：防止继承和防止虚函数被重写
### 用于类
1. 禁止其他类从该类继承，确保某个类是继承体系中的终点
2. 防止不适当或不安全的继承，例如当一个类的设计并未考虑作为基类使用时
```C++
class Base final {
    // ...
};
class Derived : public Base { // 编译错误！
    // ...
};
```
### 用于虚函数
1. 禁止派生类重写该虚函数，确保其在继承体系的某个特定级别上具有最终实现
2. 安全和设计：锁定核心算法或行为，防止派生类意外或错误地修改它
3. 效率：尽管不强制，但可以给编译器一个提示，允许编译器进行某些优化（例如，将虚函数调用转换为直接调用，因为知道它不会被重写）
```C++
class Base {
public:
    virtual void algorithm() const final {
        // 这是 algorithm() 的最终实现
    }
};
class Derived : public Base {
public:
    // 编译错误！
    virtual void algorithm() const override { 
        // ...
    }
};
```
## explicit
用于构造函数和 C++20 中用于转换操作符，主要目的是禁止隐式类型转换
### 用于构造函数
1. 默认情况下，如果一个构造函数只需要一个参数（或者有多个参数但除了第一个参数外都有默认值），编译器可以将其用作隐式转换函数。
2. explicit 禁止编译器使用该构造函数进行隐式类型转换。消除意料之外的、可能导致错误的类型转换行为
```C++
class Duration {
public:
    Duration(int seconds) : s_(seconds) {} 
    explicit Duration(double seconds) : s_(static_cast<int>(seconds)) {} 
private:
    int s_;
};
void print_duration(Duration d) { /* ... */ }

int main() {
    // 对于构造函数 (1) (无 explicit)
    print_duration(10); // OK！ int 10 被隐式转换为 Duration 对象
    Duration d1 = 20;   // OK！ int 20 被隐式转换为 Duration 对象
    // 对于构造函数 (2) (有 explicit)
    print_duration(10.5); // 编译错误！禁止隐式转换
    Duration d2 = 30.0;   // 编译错误！禁止隐式转换
    // 只能使用显式构造
    print_duration(Duration(10.5)); // OK！ 显式构造
    Duration d3(30.0);              // OK！ 显式构造
    Duration d4 = static_cast<Duration>(40.0); // OK！ 显式转换
}
```
### 用于转换操作符
1. 在 C++20 中，explicit 还可以用于转换操作符（例如 operator int() 或 operator bool()）
2. 禁止编译器在非布尔上下文中使用转换操作符进行隐式转换
```C++
class MyBool {
public:
    // 显式布尔转换 (C++11/14/17 引入了特殊规则，但 C++20 标准化了 explicit)
    explicit operator bool() const { return true; } 
    // 显式 int 转换 (C++20)
    explicit operator int() const { return 5; }
};

int main() {
    MyBool mb;
    if (mb) { /* ... */ } // OK: 在布尔上下文中使用显式转换
    int a = mb;           // 编译错误！禁止隐式 int 转换
    int b = static_cast<int>(mb); // OK: 显式转换
}
```
## override
1. 用于派生类中，紧跟在成员函数的声明之后，明确告诉编译器：此函数旨在重写基类中的一个虚函数。
2. 编译器会执行严格的检查，确保以下条件都被满足\
（1）基类中必须存在同名函数\
（2）基类中的函数必须是 virtual\
（3）派生类中函数的签名（包括参数类型、参数个数、const 限定符等）必须与基类中被重写的虚函数完全一致
3. 将原本可能在运行时表现出来的多态性问题，提前到编译时检查出来，提高了代码的可靠性。