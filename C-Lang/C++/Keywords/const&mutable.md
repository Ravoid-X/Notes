# const
## 变量
在定义的时候要进行初始化，并且之后不能再给它赋值，否则会报错。
## 指针
### 指针指向的数据不可变
```
const int *p    //或者
int const *p
```
指针指向的数据是不可变的
### 指针本身不可变
```
int * const p
```
### 结合
```
const int * const p
```
## 函数
### 修饰形参
```
int& fun(const int& a);
```
### 修饰返回值
```
const int& fun(int& a);
```
### 修饰成员函数
```
int& fun(int& a) const{} 
```
## 类
### 成员变量
```
class A{
public:
    A(int a):m_a(a){}  //正常
    A(int a){m_a = a;} //报错   
private:
    const int m_a;
};
```
1. 不能在类内初始化 const 成员变量，因为类的对象没创建前，编译器并不知道 const 成员变量是什么。
2. const 成员变量只能在初始化列表中初始化。
### 成员函数
```
class A {
public:
    int a = 1;
    const int* show(const int* p) const ;
};
const int* A::show(const int* p) const {
    a = 2;
    *p = 2;
    return p;
}
```
1. 在函数声明处添加了 const 关键字，在类外定义的时候依然要带上
2. 限定成员函数不可以修改任何数据成员
### 对象
```
const class object(params);
const class *p = new class(params);
```
const 对象只能访问类中的 const 成员变量和 const 成员函数

# mutable
为了突破const的限制而设置的。被mutable修饰的变量，将永远处于可变的状态，即使在一个const函数中。
```
class ClxTest{
    public:
    　　ClxTest();
    　　~ClxTest();
    　　void Output() const;
    　　int GetOutputTimes() const;
　  private:
　　    mutable int m_iTimes;
};

ClxTest::ClxTest(){
　m_iTimes = 0;
}

ClxTest::~ClxTest(){}

void ClxTest::Output() const{
　cout << "Output for test!" << endl;
　m_iTimes++;
}

int ClxTest::GetOutputTimes() const{
　return m_iTimes;
}

void OutputTest(const ClxTest& lx){
　cout << lx.GetOutputTimes() << endl;
　lx.Output();
　cout << lx.GetOutputTimes() << endl;
}
```
m_iTimes 跟对象的状态无关，为了修改该变量可以使用 mutable