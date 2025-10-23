## Struct
### 实例
```
struct Room {
		int floor;
		int No;
};
struct Student {
	int age;
	int score;
	Student(int a,int s){
		age=a;
		score=s;
	}
};
```
### 特性
1. struct 中的成员变量和成员函数也有访问权限，在 class 中，默认的访问权限是 private，而在 struct 中默认访问权限是 public，

## 内存对齐
### 举例
```
struct A {
    char c1;
    char c2;
    int i;
    double d;
};
struct B {
    char c1;
    int i;
    char c2;
    double d;
};
```
<img src="../../../pic/C-Lang/C++/Data Structure/struct_size.png" style="width:450px;padding:10px;"/>

### 概述
1. 提高内存访问速度的策略，CPU 在访问未对齐的内存可能需要经过两次的内存访问，而经过内存对齐一次就可以了。
2. 对齐的情况读取\
<img src="../../../pic/C-Lang/C++/Data Structure/struct_alignment1.png" style="width:450px;padding:10px;"/>

3. 不对齐的情况读取\
<img src="../../../pic/C-Lang/C++/Data Structure/struct_alignment2.png" style="width:450px;padding:10px;"/>

### 对齐原则
1. 在 x86 下，GCC 默认按4字节对齐，但是可以使用__attribute__选项改变对齐规则， vs studio 上用 #pragma pack (n) （即偏移量是n的整数倍）方式改变
2. 第一个成员的偏移量为 0，其余成员的偏移量是 其实际长度 的整数倍，如果不是，则在前一个成员后面补充字节。
3. 所有数据成员各自内存对齐后，结构体本身还要进行一次内存对齐，保证整个结构体占用内存大小是结构体内最大数据成员的最小整数倍。
## union 
### 举例
```
testunion {
	char c;
	int i;
};
```
### 特性
1. 各成员共享一段内存空间, 一个 union 变量的长度等于各成员中最长的长度，以达到节省空间的目的。
2. 把多个成员同时装入一个 union 变量内, 而是指该 union 变量可被赋予任一成员值,但每次只能赋一种值, 赋入新值则冲去旧值。
3. sizeof(union) 是最长的数据成员的长度