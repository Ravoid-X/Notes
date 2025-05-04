# 类定义  
<img src="../../Pic/C-Lang/C++/cpp-classes-define.png" style="width:400px;padding:10px;"/>

成员函数可以定义在类定义内部，或者单独使用范围解析运算符 :: 来定义\
类定义内部的类成员函数是文件内部作用域(默认为inline)，类定义外部的有全局作用域\
inline：为了解决频繁调用的小函数大量消耗栈空间（栈内存），仅仅是一个对编译器的建议，最终不一定内联
# 访问修饰符
1. public\
   在中类的外部是可访问的，一般用于定义函数\
   继承时访问属性在派生类中不变
2. private(默认)\
   在类的外部是不可访问的，一般用于定义数据。只有类内部的成员函数和友元函数可以访问私有成员\
   继承时public和protected访问属性在派生类中变为private
3. protected\
   类似private，但成员在派生类（即子类）中是可访问的\
   继承时public访问属性在派生类中变为protected
# 类&对象
1. 定义
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
2. 构造函数\
    类的一种特殊的成员函数，在每次创建类的新对象时执行\
    名称与类完全相同，不会返回任何类型，也不会返回 void，默认不带参数\
    如果需要，构造函数也可以带有参数。这样在创建对象时就会给对象赋初始值
    ```
    Line(double len){
        length = len;
    }
    等同于
    Line( double len): length(len){
    }
    ```
    如下可以初始化多个字段
    ```
    C::C( double a, double b, double c): X(a), Y(b), Z(c){
    }
    ```
3. 析构函数\
    类的一种特殊的成员函数，在每次删除对象时执行\
    名称与类完全相同的，在前面加（~）作为前缀，返回任何值，也不能带有任何参数\
    析构函数有助于在跳出程序（比如关闭文件、释放内存等）前释放资源
4. 拷贝构造函数\
    将一个类的对象拷贝给同一个类的另一个对象，类中没有定义时编译器会自行定义一个，常用于：\
        同一类中之前创建的对象来初始化新创建的对象\
        如果函数的形参是类的对象，调用函数时，将使用实参初始化形参，发生拷贝构造\
        如果函数的返回值是类的对象，函数执行完成返回时，将使用return的对象初始化一个临时的无名对象传递给主调函数，发生拷贝构造。
    ````
    class Clock {
        private:
            int H, M, S;
        public:
            Clock(int h = 0, int m = 0, int s = 0) { 
                H = h, M = m, S = s;
            };
            ~Clock() {};
            Clock(Clock &p) { // 由于一般不修改p，所以一般写为Clock(const Clock &p)
                H = p.H;
                M = p.M;
                S = p.S;
            }
    };
    Clock fun(Clock c) {
        return c;
    }

    int main() {
        Clock c1(8, 0, 0); // 调用构造函数
        Clock c2(c1); // 等价于Clock c2 =c1;，这里调用拷贝构造函数，而不是=
        fun(c2); // c2作为实参传入时，调用一次拷贝构造函数。返回Clock对象时，又调用一次拷贝构造函数
        Clock c3;
        c3 = c2;	// c2和c3都是已存在的对象。此时不调用拷贝构造函数，为赋值函数
        return 0;
    }
    ````
5. 假如类中存在指针变量，拷贝时两个指针指向同一个地址容易造成同一块内存资源释放多次，崩溃或者内存泄漏，为浅拷贝；而深拷贝则会为每一个对象分配自己的内存空间，需要自己写拷贝函数,如下
   ````
    test(const test &p) {
		int a = strlen(p.Str);//改为sizeof就是错的
		Str = new char[a + 1];
		strcpy_s(Str, a + 1, p.Str);
	}
    Line::Line(const Line &obj)
    {
        p = new int;
        *p = *obj.p;
    }
   ````
6. 拷贝赋值运算符\
   需要拷贝构造函数时通常也需要自定义拷贝赋值运算符，反之亦然
   返回类型为引用类型，返回*this（如果返回值类型，将调用拷贝构造函数）
   形参为const 引用类型；(const 说明不改变=右侧的值，引用传参不会调用拷贝)
   要考虑自赋值的操作，即判同测试，因为赋值之前要释放‘=’左侧变量的内存空间，避免把自身的内存给释放掉；
   避免内存泄漏，当不是自赋值时，先释放掉原来的内内存空间。
   ````
    Mystring& Mystring::operator=(const Mystring& other)
    {
        if (this == &other)
            return *this;
        if (m_data != NULL)
            delete[] m_data;
        int length = strlen(other.m_data) + 1;
        m_data = new char[length];
        strcpy_s(m_data,length,other.m_data);
        return *this;
    }
   ````
7. 友元函数\
   定义在类外部，但有权访问类的所有private和protected成员。尽管其原型出现在类的定义中，但并不是任何类的成员函数；可以是一个函数，称为友元函数；也可以是一个类，称为友元类，此种情况下，整个类及其所有成员都是友元
8. 内联函数\
   inline：为了解决频繁调用的小函数大量消耗栈空间，在编译时，编译器会把该函数的代码副本放置在每个调用该函数的地方，只是一个对编译器的建议，最终不一定内联\
   对内联函数进行任何修改，都需要重新编译函数的所有客户端\
   如果已定义的函数多于一行，编译器会忽略 inline 限定符
9.  this 指针\
   每一个对象都能通过 this 指针来访问自己的地址,友元函数没有this指针，因为友元不是类的成员
11. static\
   静态变量在类的所有对象中共享,如果不存在其他初始化语句，创建第一个对象时初始化为零\
   不能在类的定义中初始化，可以在类的外部通过使用 :: 来重新声明静态变量并初始化\
   静态成员函数在类对象不存在的情况下也能被调用，只要使用类名加 :: 就可以访问\
   静态成员函数没有 this 指针，只能访问静态成员（包括静态成员变量和静态成员函数）
# 继承定义 
依据另一个类来定义一个类，更容易创建和维护一个应用程序。这样做，也达到了重用代码功能和提高执行效率的效果，已有的类称为基类，新建的类称为派生类
````
class Animal {
    eat();
    sleep();
};
class Dog : public Animal {
    bark();
};
````
1. 多继承
   一个类可以派生自多个类，可以从多个基类继承数据和函数
   ````
   class Rectangle: public Shape, private PaintCost{
        public:
            int getArea();
   };
   ````
# 多态
当类之间存在层次结构并且是通过继承关联，调用成员函数时，会根据调用函数的对象的类型来执行不同的函数\
1. 虚函数\
   在基类中使用virtual声明的函数，在派生类中重新定义时，让编译器不静态链接到该函数
2. 纯虚函数\
   可能想要在基类中定义虚函数，以便在派生类中重新定义时更好地适用于对象，但在基类中又不能对虚函数给出有意义的实现
   ````
   virtual int area() = 0;
   ````
# 抽象
1. 数据抽象
   仅向用户暴露接口而把具体的实现细节隐藏起来的机制
2. 数据封装
   把数据和操作数据的函数捆绑在一起的机制
3. 接口（抽象类）
   至少有一个函数被声明为纯虚函数的类；可用于实例化对象的类被称为具体类
# 重载
1. 定义\
允许为在同一作用域中的函数和运算符指定多个定义，分别称为函数重载和运算符重载；编译器通过对比使用的参数类型与定义中的参数类型，决定选用最合适的定义，过程称为重载决策
2. 函数重载\
在同一个作用域内，可以声明几个功能类似的同名函数，但是这些同名函数的形式参数（指参数的个数、类型或者顺序）必须不同，不能仅通过返回类型的不同来重载函数
````
class printData
{
   public:
      void print(int i) {
        cout << "整数为: " << i << endl;
      }
      void print(double  f) {
        cout << "浮点数为: " << f << endl;
      }
      void print(char c[]) {
        cout << "字符串为: " << c << endl;
      }
};
````
3. 运算符重载\
可以重定义或重载大部分C++内置运算符，不可以改变语法结构、操作数个数、优先级、结合性\
重载运算符是有特殊名称的函数，函数名由关键字operator和其后的运算符构成的\
与其他函数一样，重载运算符有一个返回类型和一个参数列表\
<img src="../../Pic/C-Lang/C++/overload-type.jpg" style="width:480px;padding:10px;"/>

````
class Symbol{
    public:
        friend Symbol operator +(Symbol& a, Symbol& b)
        {
            Symbol s;
            s.Length = a.Length + b.Length;
            s.Width = a.Width + b.Width;
            s.Height = a.Height + b.Height;
            return s;
        }
        Symbol operator -(Symbol& a)//成员函数默认第一个形参省略，用this指针代替
        {
            Symbol s;
            s.Length = this->Length - a.Length;
            s.Width = this->Width - a.Width;
            s.Height = this->Height - a.Height;
            return s;
        }
        int Length;
        int Width;
        int Height;
};
````