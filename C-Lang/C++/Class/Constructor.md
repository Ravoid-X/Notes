## 特殊成员函数
共有 6 个：构造函数；析构函数；复制构造函数(拷贝构造函数)；赋值运算符(拷贝运算符)；移动构造函数(c++11引入)；移动赋值运算符(c++11引入)
```
class Widget{
public:
    Widget();//构造函数
    ~Widget();//析构函数
    Widget(const Widget& rhs);//复制构造函数(拷贝构造函数)
    Widget& operator=(const Widget& rhs);//赋值运算符(拷贝运算符)
    Widget(Widget&& rhs);//移动构造函数
    Widget& operator=(Widget&& rhs);//移动赋值运算符
private：
    int width；
    int height；
}
```
## 构造函数
1. 在每次创建类的新对象时执行，帮助创建对象的实例，并对实例进行初始化。
2. 名称与类完全相同，不会返回任何类型，也不会返回 void。
3. 构造函数可以重载，不可以被继承。
4. 构造函数可以带有参数，默认不带参数。
### 调用情况
```
Widget widget; //栈对象
Widget *w = new Widget();  //堆对象
Widget w(0，0);  
```
### 无参&带参
```
class Widget{
public:
    Widget();
    Widget(int,int);
private：
    int width；
    int height；
}

Widget::Widget(){
    width = 0;
    height = 0;
}
Widget::Widget(int w,int h){
    width = w;
    height = h;
}

int main()
{
    Widget w1;           //此处会调用 无参数 的构造函数
    Widget w2(0，0);     //此处会调用 有参数 的构造函数
    return 0;
}
```
### 参数初始化表
```
class Widget{
public:
    Widget():width(0),height(0){}; //无参
    Widget(int w,int h):width(w),height(h){};
private：
    int width；
    int height;
}
Widget::Widget(int w,int h):width(w),height(h){};
```


## 拷贝构造函数
1. 使用一个已经存在的同类对象来创建一个新的对象。
2. 如果不使用引用，参数将通过值传递。编译器需要先复制一份实参，这会调用拷贝构造函数，从而导致无限递归，最终栈溢出。
### 调用
1. 使用一个对象初始化另一个对象
```
Widget w1; 
Widget w2 = w1; // 初始化，调用的不是拷贝赋值运算符。
Widget w3(w1);  // 更推荐的写法
//栈上的三个对象 w3, w2, w1 以相反的顺序被析构
```
2. 函数参数是类的对象（值传递）
```
void func(Widget w){...}{}
Widget w1; 
func(w1); //执行结束时，w1 被析构
```
3. 函数按值返回一个对象
```
Widget func(){
    Widget w1;
    return w1;
}
//理论上会调用拷贝构造函数，现代 C++ 编译器会进行 返回值优化 (Return Value Optimization, RVO)
//跳过创建临时对象的步骤，直接在调用方的栈上构造最终的对象，从而避免了不必要的拷贝构造函数调用。
//可以添加编译选项 -fno-elide-constructors 来关闭 RVO
```

## 移动构造函数
从一个对象中移动（转移）资源到另一个对象，而不是进行复制操作。




