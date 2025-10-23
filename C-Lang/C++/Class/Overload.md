## 定义
1. 允许为在同一作用域中的函数和运算符指定多个定义，分别称为函数重载和运算符重载
2. 编译器通过对比使用的参数类型与定义中的参数类型，决定选用最合适的定义，过程称为重载决策
## 函数重载
1. 在同一个作用域内，可以声明几个功能类似的同名函数，这些同名函数的形参列表（参数的个数、类型或者顺序）必须不同。
2. 不能仅通过返回类型的不同来重载函数，无法区分要调用的是谁。
```
void Swap(int* a, int* b)
{
	int temp = *a;
	*a = *b;
	*b = temp;
}
void Swap(double* a, double* b)
{
	double temp = *a;
	*a = *b;
	*b = temp;
}
```

# 运算符重载
## 引出
```
Complex c1(2, 3);
Complex c2(4, 1);
Complex sum = c1.add(c2);
```
1. 如果没有运算符重载，想要对两个自定义对象进行相加需要调用成员函数。
2. 这样的代码虽然正确但不够直观
## 特性
1. 可以重载大部分 C++ 内置运算符，不可以改变语法结构、操作数个数、优先级、结合性。
2. 重载运算符是有特殊名称的函数，函数名由关键字 operator 和其后的运算符构成的
3. 与其他函数一样，重载运算符有一个返回类型和一个参数列表
<img src="../../../pic/C-Lang/C++/Class/overload-type.jpg" style="width:480px;padding:10px;"/>

## 作为成员函数
左操作数隐式地是调用该函数的对象本身（即 *this）
```
class Point {
public:
    int x, y;
    Point(int x = 0, int y = 0) : x(x), y(y) {}
    Point operator+(const Point& rhs) const {
        Point result;
        result.x = this->x + rhs.x;
        result.y = this->y + rhs.y;
        return result;
    }
};
```

## 作为非成员函数
需要接收所有的操作数作为参数
### 左侧操作数不是类的对象 (强制情况)
```
class A {
private:
    int x, y; 
public:
    A(int x, int y) : x(x), y(y) {}
    friend ostream& operator<<(ostream& os, const A2D& a){
		os << "A(" << a.x << ", " << a.y << ")";
		return os; 
	}
};
int main() {
    Vector2D v(3, 4);
    // 调用的是全局的 operator<<(cout, v)
    std::cout << "My vector is: " << v << std::endl; 
}
```
### 支持对称的隐式类型转换(设计更佳)
```
Number n(10);
Number result1 = n + 5; // OK. 解释为 n.operator+(5)
Number result2 = 5 + n; // Error! 编译器尝试解释为 5.operator+(n)
```
```
class Number {
public:
    int value;
    // 提供一个从 int 转换的构造函数 (非 explicit)
    Number(int v = 0) : value(v) {} 
};
// 非成员函数提供了对称性
Number operator+(const Number& lhs, const Number& rhs) {
    return Number(lhs.value + rhs.value);
}
```
### 增强封装性与接口分离
```
//operator+= 通常会修改对象自身的状态
//operator+ 创建并返回一个新对象，可以基于 += 来实现 +。
class DataBlock {
public:
    // += 是核心功能，修改自身状态，适合做成员
    DataBlock& operator+=(const DataBlock& rhs) {
        // ... 执行合并逻辑 ...
        return *this;
    }
};
// + 可以作为非成员函数，它不直接访问私有成员，
// 而是通过公共的 += 成员来实现。它不是 friend。
DataBlock operator+(DataBlock lhs, const DataBlock& rhs) {
    // 参数 lhs 是按值传递的，所以我们修改的是一个副本
    lhs += rhs; // 调用成员函数 DataBlock::operator+=
    return lhs; // 返回修改后的副本
}
```
1. 在这个设计中，DataBlock 类只暴露了核心的 += 操作。
2. 而 + 作为一个方便用户的工具函数，被放在了类外部，它没有也无需访问类的内部细节，从而保护了封装性。