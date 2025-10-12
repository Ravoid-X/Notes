友元不能继承
## 友元函数
```
friend void friendFunction(const MyClass& obj);
```
1. 在类中声明的一个非成员函数。可以访问该类的私有成员，包括私有变量和私有函数。
2. 声明通常位于类内，但其实现则位于类外部。
3. 在 public 或 private 中进行声明都可以。

## 友元类
```
class MyClass;

class FriendClass {
public:
    void accessPrivateData(const MyClass& obj);
};

class MyClass {
private:
    int privateData;
public:
    MyClass(int data) : privateData(data) {}
    // 声明友元类
    friend class FriendClass;
    int getPrivateData() const { return privateData; }
};
```
1. 在另一个类中声明为友元。被声明为友元的类可以访问另一个类的私有成员，类似于友元函数的概念，但是是对整个类的授权。
2. 通常用于需要多个类之间紧密协作的情况。