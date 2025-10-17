## 概述
1. 一种通用的编程工具，允许编写通用代码以处理多种数据类型或数据结构，而不需要为每种特定类型编写重复的代码。
2. 可以实现代码的复用和泛化，提高代码的灵活性和可维护性。
## 函数模板
```
template <class 形参名，...> 返回类型 函数名(参数列表){
    函数体
}
```
```
template<class T>
T Add(const T& x,const T& y){
    return x+y;
}
int main(){
    int a=1,b=2;
    int c=Add(a,b);
}
```
1. class可以用typename 关见字代替
2. 根据传入参数自动推导出 T 的类型，并在编译过程中自动替换
## 类模板
定义通用的类，从而可以处理不同类型的数据，可以实现通用的数据结构或算法，如vector、list 等可以存储不同数据类型的容器。
```
template<class 形参名，class 形参名，…> class 类名{ ... };
```
### 创建
```
template<class T> class A{public: T a; T b; T hy(T c, T &d);};
A<int> a;
```
```
template<class T，int N> class A{public:\\....private:
    T* val[N];
};
A<int,10> a;
```
在使用时要指定 T 为某个具体的数据类型，以供编译器进行实例化
### 类外定义
```
template<模板形参列表> 函数返回类型 类名<模板形参名>::函数名(参数列表){函数体};
template<class T1,class T2> void A<T1,T2>::h(){};
```

## 特化
对特定类型或值的情况，为通用模板提供特定的实现
### 全特化
全特化中所有的模板参数都被具体化，不留下泛化的部分
```
template <class T>
bool Is_Equal(const T& x, const T& y){
    return x==y;
}
//针对char*类型的全特化
template<> 
bool Is_Equal<char*>(char* x, char* y){
    return strcmp(x,y)==0;
}
```
### 偏特化
偏特化中只有部分的模板参数被具体化，而其他部分仍然保持泛化
```
//偏特化的类模板
template<>class Example<int, T2>{
    Example (){
        cout <<"调用了偏特化的模板"<< endl;
    }
}
int main(){
    Example<int,char> e;
} 
```