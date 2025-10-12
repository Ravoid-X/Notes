## define
```
#define  宏名  字符串
```
1. 在程序的预编译期，预处理器会解析源代码文本，执行一个替换源程序的动作，把宏引用的地方替换成定义处的文本。叫做宏的展开。
2. 不作正确性检查.
3. 不是语句，不要在行末加分号，否则会连分号一块置换。
4. 作用域为自 #define 那一行起到源程序结束。如果要终止其作用域可以使用 #undef，格式为：#undef  标识符。
5. 如果要替换的文本一行写不完，可以分成多行，在每一行的结尾处加上续行符号“\”（除了最后一行）即可
### 有参数
```
#define 宏名(参数1, 参数2, ..., 参数 n) 要替换的文本
#define LESS(a,b) (a>b?b:a)
```
1. 函数的代码只占用一份内存空间，而宏会重复占用多份内存
2. 函数调用时有时间和空间方面的额外开销，但宏没有
3. 函数的参数有类型信息，但宏的参数没有类型信息
4. 函数可以调试，宏不能调试
### 特殊宏符号
```
#define CONCATENATE(a, b) a ## b
#define TO_STRING(x) #x
#define PRINT_EXPR(expr) std::cout << #expr << " = " << expr << std::endl

int xy = CONCATENATE(1, 2);  // 实际上是 int xy = 12;
const char* str = TO_STRING(Hello World);  // 实际上是const char* str = "Hello World";

```
1. 在宏的替换文本中，可以使用一些特殊的符号，起到特定的解释作用
2. ##：用于连接两个标记。
3. #：用于字符串化一个宏参数。

## typedef
### 定义类型的别名
```
char* pa, pb; // 只声明了一个指向字符变量的指针和一个字符变量；

typedef char* PCHAR; // 一般用大写
PCHAR pa, pb; // // 同时声明了两个指向字符变量的指针
```
### 定义与平台无关的类型
比如定义一个叫 REAL 的浮点类型
```
typedef long double REAL;  // 平台一上，表示最高精度
typedef double REAL;  // 在不支持 long double 的平台二上
typedef float REAL;   // 在 不支持 double 的平台三上
```
1. 当跨平台时，只要修改 typedef 本身就行
2. 标准库就广泛使用了这个技巧，比如size_t
### 定义简单别名
1. 原声明：int *(*a[5])(int, char*);
2. 变量名为 a，直接用新别名 pFun 替换：typedef int *(*pFun)(int, char*); 
3. 原声明的最简化版：pFun a[5];
## 备注
### const 
```
typedef char* PSTR;
int mystrcmp(const PSTR, const PSTR);
```
1. const PSTR 并不是 const char*，实际上相当于char* const
2. const给予了整个指针本身以常量性，也就是形成了常量指针char* const
### 语法
```
typedef static int INT2; //不可行
```
1. typedef 在语法上是一个存储类的关键字（如auto、extern等一样），虽然它并不真正影响对象的存储特性。
2. 上面例子编译将失败，会提示“指定了一个以上的存储类”。