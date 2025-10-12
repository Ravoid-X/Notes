## 概念
1. 异常处理机制允许程序在出现错误时采取适当的行动，而不需要完全依赖于错误代码或中断程序的正常流。
2. 异常通常分成两类\
（1）逻辑错误：如数组越界、空指针解引用等\
（2）运行时错误：如文件未找到、网络连接中断、内存分配失败等
## 流程
### throw
1. 用于抛出一个异常，将错误信息传递给异常处理机制。
2. 抛出的异常对象可以是任何类型的对象，通常情况下是异常类的实例。
```
throw runtime_error("An error occurred");
```
### try
1. 包含程序中可能会抛出异常的代码。
2. 程序会在执行 try 块中的代码时监控异常的发生。如果发生了异常，控制流将立即跳转到相应的 catch 块。
### catch
1. 用于捕获并处理从 try 块中抛出的异常。
2. 一个 try 块可以有多个 catch 块，它们用于捕获不同类型的异常。
```
try {
    throw runtime_error("An error occurred");
} catch (const runtime_error& e) {
    cout << "Runtime error: " << e.what() << endl;
} catch (const exception& e) {
    cout << "Some other exception: " << e.what() << endl;
}
```
### 传播
当一个异常被抛出后，它会沿着调用栈向外传播，直到遇到匹配的 catch 块为止。如果没有找到匹配的 catch 块，程序将终止并打印出异常信息。

## 标准异常类
1. std::exception：所有标准异常的基类。
2. std::runtime_error：运行时错误，通常用于程序运行过程中检测到的错误。
3. std::logic_error：逻辑错误，表示代码错误，如非法参数。
4. std::out_of_range：数组下标越界等错误。
5. std::invalid_argument：无效的函数参数。