## 作用
防止同一个头文件的内容在一次编译过程中被重复包含（include）多次
## 示例
1. 假设有 3 个文件，main.cpp，a.h，b.h
2. main.cpp 引用 了 a.h 和 b.h，b.h 引用了 a.h
3. 当编译器开始编译 main.cpp，会黏贴两份 a.h 的 代码
4. 编译器会报 重复定义 或 重声明 错误。
## 语法
```
#ifndef XXX_H 
#define XXX_H 
... (头文件的所有内容) ... 
#endif
```