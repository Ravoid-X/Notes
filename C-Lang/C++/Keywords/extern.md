置于变量或者函数前，以表示变量或者函数的定义在别的文件中，提示编译器遇到此变量和函数时在其他模块中寻找其定义。
## 作用
1. 在多个文件中共享同一个变量
2. 在一个文件中引用另一个文件中定义的函数
## 用于变量
```
// 文件1: main.cpp
extern int shared_var;  // 声明一个外部整型变量
int main() {
    shared_var = 10;  // 使用外部变量
    return 0;
}
// 文件2: shared.cpp
int shared_var = 0;  // 定义一个全局整型变量
```
## 用于函数
```
// 文件1: main.cpp
extern void print_message();  // 声明一个外部函数
int main() {
    print_message();  // 调用外部函数
    return 0;
}
// 文件2: print.cpp
#include <iostream>
void print_message() {  
    cout << "Hello, World!" << endl;
}
```
### extern “C”
1. C++ 支持函数重载，编译器会对函数名进行改编，以区分具有同名不同参函数。
2. C 语言不支持函数重载。如果要在 C++ 中调用 C，或者在 C 中调用 C++，就需要用到 extern “C”。
```
// 文件1: main.cpp
extern "C" void print_message();  // 使用 extern "C" 声明一个外部函数
int main() {
    print_message();  // 调用外部函数
    return 0;
}
// 文件2: print.c
#include <stdio.h>
void print_message() {  // 定义一个函数
    printf("Hello, World!\n");
}
```