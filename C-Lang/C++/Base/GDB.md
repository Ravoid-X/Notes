## 概述
1. Linux 和类 Unix 系统下标配的、功能强大的命令行调试工具，主要用于 C、C++、Go、Rust 等编译型语言
2. 可以诊断段错误、内存泄漏、逻辑错误等问题
## 准备工作
### 编译代码
为了让 GDB 能够获取到源代码行号、变量名等信息，必须在编译时加入 -g 标志、
### 示例
```
g++ -g -o my_program my_program.cpp
```
不要在生产环境的最终发布版本中使用，会增大文件体积
## 启动/退出 GDB
### 启动并加载程序
最常用，会启动 GDB 并加载 my_program，但不会立即运行它，会看到 GDB 的提示符 (gdb)
```
gdb ./my_program
```
### 附加到已运行的进程
如果有一个正在运行的程序（例如陷入了死循环）并且想要调试
1. 找到进程 ID (PID)
```
ps aux | grep my_program
```
2. 使用 GDB 附加到它
```
gdb -p <PID>
```
### 退出 GDB
`quit`，缩写 `q`
## 核心命令
### 断点
1. `break <line_num>`：缩写 `b <line_num>`，在当前文件的第 line_num 行设置断点。
2. `break <filename>`：缩写 `<line_num>	b file.cpp:10`，在 file.cpp 文件的第 10 行设置断点。
3. `break <function_name>`：缩写 `b main`，在 main 函数的入口处设置断点。
4. `info breakpoints`：缩写 `i b`，查看当前设置的所有断点及其编号 (Num)。
5. `delete <Num>`：`d <Num>`，删除编号为 Num 的断点。
6. `clear <line_num>`：清除指定行的断点。
7. `disable <Num>`：暂时禁用编号为 Num 的断点（不是删除）。
8. `enable <Num>`：重新启用编号为 Num 的断点。
### 运行与执行
1. `run`：缩写 `r`，启动程序。如果程序需要命令行参数，跟在 run 后面，如 r arg1 arg2
2. `continue`：缩写 `c`，继续执行，直到遇到下一个断点或程序结束
3. `kill`：终止当前正在调试的程序（但 GDB 不会退出）
### 单步调试
1. `next`：缩写 `n`，单步跳过，执行下一行代码。如果该行是函数调用，它会执行完整个函数，停在函数调用的下一行（不会进入函数内部）。
2. `step`：缩写 `s`，单步进入，执行下一行代码。如果该行是函数调用，它会进入该函数并停在函数体的第一行。
3. `finish`：缩写 `fin`，完成当前函数。执行当前函数剩余的代码，并停在函数返回后的下一行。
4. `until <line_num>`：缩写 `u <line_num>`，运行到指定行。在当前循环中很有用，可以跳过循环的剩余部分。
### 查看数据
程序停下来时，查看变量的值
1. `print <var_name>`：`p <var_name>`，打印变量的值。例如 p i，p my_vector。
2. `print *ptr`：`p *ptr`，打印指针 ptr 所指向的内容。
3. `print /x <var_name>`：按十六进制格式打印变量。
4. `display <var_name>`：自动显示。设置后，每次程序暂停时，都会自动打印该变量的值。
5. `info display`：缩写 `i display`，查看所有自动显示的变量。
6. `undisplay <Num>`：取消编号为 Num 的自动显示。
7. `watch <var_name>`：监视变量。当 var_name 的值发生改变时，程序会立即暂停（这是一个“硬件断点”，非常有用）。
### 堆栈与上下文
1. `backtrace`：缩写 `bt`，查看调用堆栈。显示函数调用的层级关系，#0 是当前函数，#1 是调用 #0 的函数，以此类推。
2. `frame <Num>`：缩写 `f <Num>`，切换堆栈帧。允许你切换到编号为 Num 的堆栈帧（例如 f 1），去查看那个函数上下文中的局部变量。
3. `list`：缩写 `l`，查看源代码。显示当前暂停位置附近的 10 行代码。
## 其他语法
### 条件断点
`break <line> if <condition>`
### TUI 模式
1. 启动 GDB 时使用 gdb -tui，或者在 GDB 内部按 Ctrl + X 然后按 A。
2. 这会打开一个分屏界面，上面显示源代码和当前行，下面是 GDB 命令行，体验大大提升。
### 修改变量
(gdb) set var i = 4 可以在调试时手动修改变量的值，测试特定分支
## 示例
```
// buggy.cpp
#include <iostream>

int square(int n) {
    return n * n;
}
int main() {
    int total = 0;
    int i;
    for (i = 0; i < 5; i++) {
        // 假设这里有个 bug，我们错误地多加了 1
        total += square(i + 1);
    }
    // 预期结果是 1*1 + 2*2 + 3*3 + 4*4 + 5*5 = 55
    // 但我们想看 0*0 + ... + 4*4 = 30
    cout << "Total: " << total << endl;
    return 0;
}
```
### 编译
```
g++ -g -o buggy buggy.cpp
```
### 启动
```
gdb ./buggy
(gdb)
```
### 设置断点
在 main 函数开始处和循环内部设置断点
```
(gdb) b main
Breakpoint 1 at 0x1165: file buggy.cpp, line 9.
(gdb) b 13
Breakpoint 2 at 0x1188: file buggy.cpp, line 13.
```
### 运行程序
程序在 main 的开头停下了
```
(gdb) run
Starting program: /path/to/buggy 

Breakpoint 1, main () at buggy.cpp:9
9           int total = 0;
```
### 单步执行 (n) 
用 next (n) 走到循环开始前，程序停在了设置的第 2 个断点处（13 行），这是循环的第一次
```
(gdb) n
11          for (i = 0; i < 5; i++) {
(gdb) n
13              total += square(i + 1);
```
### 查看变量 (p) 
查看此时 i 和 total 的值
```
(gdb) p i
$1 = 0
(gdb) p total
$2 = 0
```
### 单步进入 (s)
1. 看 square 函数内部发生了什么，使用 step (s)
2. 进入了 square 函数，GDB 告诉我们传入的参数 n 是 1
3. i 是 0，但传给 square 的是 1，这就是 bug 所在 ( i + 1 )
```
(gdb) s
square (n=1) at buggy.cpp:4
4           return n * n;
```
### 完成函数 (fin)
已经知道 square 内部没问题了，用 finish (fin) 执行完它并返回 main
```
(gdb) fin
Run till exit from #0  square (n=1) at buggy.cpp:4
0x000055555555518f in main () at buggy.cpp:13
13              total += square(i + 1);
Value returned is $3 = 1
```
### 查看堆栈 (bt)
确认回到了 main 函数
```
(gdb) bt
#0  main () at buggy.cpp:13
```
### 继续执行 (c) 
让程序继续运行，会停在循环的下一次（断点 2 处）
```
(gdb) c
Continuing.

Breakpoint 2, main () at buggy.cpp:13
13              total += square(i + 1);
```
### 再次查看变量
```
(gdb) p i
$4 = 1
(gdb) p total
$5 = 1
```
### 退出
```
(gdb) quit
A debugging session is active.

    Inferior 1 [process 12345] will be killed.

Quit anyway? (y or n) y
```