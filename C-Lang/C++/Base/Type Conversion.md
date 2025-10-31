## 隐式转换
由编译器自动执行的，不需要程序员显式指定
### 算术转换
在混合类型的算术表达式中，较小的类型会提升为较大的类型
```
int i = 5;
double d = 3.14;
double result = i + d; // i 被隐式转换为 double (5.0)
```
1. 编译器会寻找一种“最安全”的转换，即避免信息丢失
2. 将 double(3.14) 转为 int(3) 会丢失 .14 的精度；将 int(5) 转为 double(5.0) 不会丢失信息
3. 编译器创建了一个临时的 double 值 5.0，再进行计算
### 赋值转换
```
int i;
double d = 10.5;
i = d; // i 值为 10
```
赋值操作中规则是目标类型优先，编译器被强制将右侧的值转换为左侧的类型，即使这会导致信息丢失。
### 构造函数转换
```
class MyString {
public:
    // 这是一个“转换构造函数”
    MyString(const char* s) {
        cout << "Constructor called!" << endl;
    }
};
void printString(MyString s) { //... }
// "Hello" (const char*) 被隐式转换为 MyString
printString("Hello");
```
1. 编译器查找是否存在一种方法可以将 const char* 变为 MyString
2. 找到了构造函数 MyString(const char* s)，恰好接受一个 const char* 参数
3. 编译器自动地、隐式地调用了这个构造函数，用 "Hello" 作为参数，创建了一个临时的 MyString 对象
4. explicit 关键字来阻止这种隐式转换
## 显式转换
当隐式转换不可行或想明确表达意图（即使会丢失数据）时使用
### C 风格转换
```
double d = 3.14;
int i = (int)d; // C 风格
int j = int(d); // 函数风格 (功能上与 C 风格相同)
```
1. C 风格转换是“万能”的，会按顺序尝试 C++ 的多种转换（static_cast, const_cast, reinterpret_cast），直到一种能工作。
2. C 风格转换在 C++ 中应被完全避免，它隐藏了转换的真实意图和风险。C++ 提供了四个意图明确的转换符
### static_cast (静态转换)
在编译时进行类型检查，用于合理或相关的类型转换
1. 数值转换（显式截断）
```
double d = 3.14;
int i = static_cast<int>(d); // i = 3
```
2. void* 指针转换：void* 丢失了类型信息，static_cast 是 C++ 中将 void* 指针转回其原始类型的标准方式。
```
int i = 10;
void* p = &i;
int* pi = static_cast<int*>(p);
cout << *pi; // 输出 10
```
3. 类继承转换（下行转换）：编译器不做任何运行时检查
```
class Base { public: int b_val; };
class Derived : public Base { public: int d_val; };
Base* b = new Derived(); // OK，隐式上行转换
// 假如 Base* b = new Base(); 是未定义行为
Derived* d_ptr = static_cast<Derived*>(b);
d_ptr->d_val = 100; // OK
```
### dynamic_cast
在运行时进行类型检查，专门用于多态类的安全下行转换。
```
class Base { 
public: 
    virtual void foo() {} // 必须是多态基类
    // virtual ~Base() {} // 虚析构函数是更好的选择
};
class Derived : public Base {};
class Another : public Base {};
```
1. 指针转换
```
Base* b_ptr = new Derived();
// 尝试转换为 Derived* (成功)
Derived* d_ptr = dynamic_cast<Derived*>(b_ptr); 
if (d_ptr != nullptr) {
    // 成功, d_ptr 是一个有效的 Derived*
}
// 尝试转换为 Another* (失败)
Another* a_ptr = dynamic_cast<Another*>(b_ptr); 
if (a_ptr == nullptr) {
    // 失败, b_ptr 指向的不是 Another, a_ptr 被设为 nullptr
}
```
2. 引用转换
```
Derived d_obj;
Base& b_ref = d_obj; // b_ref 引用一个 Derived 对象
try {
    // 尝试转换为 Derived& (成功)
    Derived& d_ref = dynamic_cast<Derived&>(b_ref);
    // 尝试转换为 Another& (失败)
    Another& a_ref = dynamic_cast<Another&>(b_ref); 
} catch (bad_cast& e) {
    // 失败!
    cout << "转换失败: " << e.what() << endl;
}
```
### const_cast
唯一能添加或（通常是）移除 const 或 volatile 属性的转换
```
// 假设这是一个不规范的 C 库函数，承诺不修改，但忘了加 const
void legacy_c_function(char* str) { /* 它只是读取 str */ }
const char* my_str = "Hello";
// 编译错误：不能将 const char* 传递给 char*
legacy_c_function(my_str); 
// 强制转换：
legacy_c_function(const_cast<char*>(my_str));
```
1. `const_cast<char*>` 告诉编译器：暂时移除这个指针的 const 限制
2. const_cast 并不改变 my_str 指向的原始数据，只是返回一个新的、非 const 的指针指向相同的地址
### reinterpret_cast
1. 告诉编译器：把这块内存中的二进制位模式当做另一种类型来解释
2. 不做任何有意义的转换，只是原始的位重解释，十分危险
3. 不相关指针转换
```
int i = 65; // 'A' 的 ASCII 码
int* pi = &i;
// 将 int* 重新解释为 char*
char* p_c = reinterpret_cast<char*>(pi);
```
（1）假设是小端机器，内存地址：0x1000 0x1001 0x1002 0x1003 内存内容：[0x41] [0x00] [0x00] [0x00]\
（2）p_c 是一个指针，指向 0x1000，告诉编译器从这里开始只读取 1 个字节作为一个 char，即 0x41，也就是字符 'A'\
（3）如果是大端机器，*p_c 会读取 0x00（空字符 \0）\
（4）reinterpret_cast 几乎总是导致不可移植的代码
4. 指针与整数互转
```
int i = 10;
int* pi = &i;
// 1. 将指针转换为整数
// uintptr_t 是一个足够大来存储指针地址的无符号整数类型
uintptr_t addr = reinterpret_cast<uintptr_t>(pi);
cout << "Address is: 0x" << hex << addr; 
// 输出: Address is: 0x7ffee1234560 (一个内存地址)
// 我们可以存储 addr, 将它写入日志...
// 2. 将整数转回指针
int* pi_back = reinterpret_cast<int*>(addr);
cout << *pi_back; // 输出 10 (如果 i 仍然有效)
```
（1）`reinterpret_cast<uintptr_t>(pi)` 告诉编译器：“不要把这个看作指针，把这个地址本身的位模式当作一个整数值”\
（2）`reinterpret_cast<int*>(addr)` 做相反的操作。告诉编译器：“把这个整数值当作一个内存地址，并创建一个指向该地址的 int* 指针”\
（3）这种技术在底层编程中很有用：在自定义内存分配器中对齐指针地址；将指针存储在不能使用 void* 的地方（例如某些 C API）；与硬件寄存器交互，这些寄存器具有固定的、已知的内存地址\
（4）风险：如果转回去的整数是无效的地址，或者对象已经被销毁，使用 pi_back 将导致未定义行为