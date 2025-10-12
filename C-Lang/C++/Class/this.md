## 概述 
1. 隐含于每一个非静态成员函数中的特殊指针。指向调用该成员函数的那个对象。
2. 当对一个对象调用成员函数时，编译程序先将对象的地址赋给 this 指针，然后调用成员函数，每次成员函数存取数据成员时，都隐式使用 this 指针。
3. this 是个右值，所以不能取得 this 的地址（不能 &this），除了在成员函数里。
4. this 在成员函数的开始执行前构造，在执行结束后清除。
5. 
## 作用
### 解决成员变量与函数参数同名
```
class Example {
private:
    int value;
public:
    void setValue(int value) {
        this->value = value; // 使用 this 区分成员变量和参数
    }
}
```
### 支持方法链调用
```
class Example {
private:
    int value;
public:
    Example& setValue(int v) {
        value = v;
        return *this; // 返回当前对象的引用
    }
    Example& increment() {
        value++;
        return *this; // 返回当前对象的引用
    }
    void display() {
        cout << "Value: " << value << endl;
    }
};

int main() {
    Example obj;
    obj.setValue(10).increment().display(); // 链式调用
    return 0;
}
Value: 11 //输出
```
## delete this
### 成员函数中
1. 在类对象的内存空间中，只有数据成员和虚函数表指针，类的成员函数单独放在代码段中。
2. 当调用 delete this 时，类对象的内存空间被释放。之后进行的其他任何函数调用，只要不涉及到 this 指针的内容，都能够正常运行。
### 析构函数中
1. 导致堆栈溢出，delete 的本质是“为将被释放的内存调用一个或多个析构函数，然后，释放内存”。
2. delete this会去调用本对象的析构函数，而析构函数中又调用delete this，形成无限递归，造成堆栈溢出。