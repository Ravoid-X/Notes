## 类定义  
<img src="../../../pic/C-Lang/C++/Class/classes-define.png" style="width:400px;padding:10px;"/>

```
class Line
{   
    public:
        void setLength( double len );
        double getLength( void );
        Line();                     // 构造函数
        Line( const Line &obj);     // 拷贝构造函数
        Line& operator=(const Line& obj); //拷贝赋值运算符
        friend void print( Line line ); //友元函数
        friend class LineTwo;        //友元类
        ~Line();                    // 析构函数声明
    private:
    double length;
    char *p;
};
```
## 访问修饰符
### public
在类的外部是可访问的，一般用于定义函数
### private(默认)
在类的外部是不可访问的，一般用于定义数据。只有类内部的成员函数和友元函数可以访问私有成员
### protected
类似 private，但成员在派生类（即子类）中是可访问的

## 成员函数
1. 类内定义：函数默认为内联，，即便没有使用 inline 标识符
2. 类外定义：需使用使用范围解析运算符 ::
```
double getLength( void ){
    return length;
}
```