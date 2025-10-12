## 引入命名空间
```
using namespace std;//std::cin>>x; 可略写为 cin>>x;
```
## 引入单个声明
```
using std::string;//std::string s = "123"; 可略写为 string s = "123";
```
## 重定义
```
using ULL = unsigned long long;	
using func = void(*)(int, int);
using mapInt = std::map<int, Val>;
```
### 改变派生类对父类成员的访问控制
```
class Base
{
protected:
    int bn1;
    int bn2;
};
 
class Derived: private Base//私有继承
{	
public:
    using Base::bn1;	//在当前作用域中引入了父类的保护成员, 在当前作用域中可以访问
};
class DerivedAgain: public Derived//在Derived的子类中仍然可以访问bn1
{	

};
```