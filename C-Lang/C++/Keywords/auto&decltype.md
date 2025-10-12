## auto
让编译器自动推导变量的类型
```
auto a = 1; // 类型为int  
auto b = 5.20; // 类型为double  
auto person = Person(); // 类型为 Person,并且自动初始化成员变量变量
```
1. 必须马上初始化，否则无法推导出类型
```
auto e; // 报错
```
2. 一行定义多个变量时，各个变量的推导不能产生二义性，否则编译失败
```
auto d = 0, f = 1.0; // 报错，0 和 1.0 类型不同
```
3. 不能用于函数参数
```
void func(auto value) {} // 报错
```
4. 在类中不能用于非静态成员变量
```
class A {
    auto a = 1; // 报错
    static auto b = 1; // 报错，这里与 auto 无关，因为类内初始化必须和 const 一起修饰
    static const auto c = 1; // 正常
};
```
5. 不能定义数组，可以定义指针
```
int a[10] = {0};
auto b = a; // 正常
auto c[10] = a; // 报错
```
6. 无法推导出模板参数
```
vector<auto> f = d; // 报错
```
7. 不影响代码代码可读性的情况下可以使用 auto
## decltype
用于推导表达式类型，可以不初始化
```
decltype(exp) varName=value;
```
### 推导规则
1. 如果 exp 是不被括号包围的表达式，类成员访问表达式，或是单独的变量，decltype(exp) 类型与 exp 一致。
```
class A{
public:
    static int total;
    string name;
    int age;
    float scores;
}
 
int A::total=0;
 
int main()
{
int n=0;
const int &r=n;
A a;
decltype(n) x=n;    //n为Int，x被推导为Int
decltype(r) y=n;    //r为const int &，y被推导为const int &
decltype(A::total)  z=0;  ///total是类A的一个int 类型的成员变量，z被推导为int
decltype(A.name) url="www.baidu.com";//url为stringleix
return 0;
}
```
2. 如果 exp 是函数调用，则 decltype(exp) 类型与函数返回值的类型一致。
```
int& func1(int ,char);//返回值为int&
int&& func2(void);//返回值为int&&
int func3(double);//返回值为int
 
const int& func4(int,int,int);//返回值为const int&
const int&& func5(void);//返回值为const int&&
 
int n=50;
decltype(func1(100,'A')) a=n;//a的类型为int&
decltype(func2()) b=0;//b的类型为int&&
decltype(func3(10.5)) c=0;//c的类型为int
 
decltype(func4(1,2,3)) x=n;//x的类型为const int&
decltype(func5()) y=0;//y的类型为const int&&
```
3. 如果 exp 是一个左值，或被括号包围，decltype(exp) 类型是 exp 的引用，假设 exp 的类型为 T，则 decltype(exp) 的类型为 T&
```
class A{
public:
   int x;
}
 
int main()
{
const A obj;
decltype(obj.x) a=0;//a的类型为int
decltype((obj.x)) b=a;//b的类型为int&
 
int n=0,m=0;
decltype(m+n) c=0;//n+m得到一个右值，c的类型为int
decltype(n=n+m) d=c;//n=n+m得到一个左值，d的类型为int &
return 0;
}
```