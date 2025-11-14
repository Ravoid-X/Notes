## 概述
1. C++11 引入的一项强大特性，它提供了一种简洁的方式来定义匿名函数对象（或称为 闭包）
2. 常用于需要一个小型、局部函数，并且不希望为其显式命名的场景，例如作为算法（如 std::sort, std::for_each）的谓词或操作
## 语法结构
$$\text{[capture-list] (parameter-list) } \rightarrow \text{return-type } \{ \text{body} \}$$
### [capture-list]：捕获列表
捕获外部作用域的变量，供 lambda 体内使用，必须有
### (parameter-list)：参数列表
类似于普通函数的参数列表，可选
### -> return-type：尾随返回类型
明确指定 lambda 的返回类型，可选
### {body}：函数体
lambda 表达式执行的代码，必须有
## 捕获列表
定义了 lambda 表达式如何访问其定义所在作用域的局部变量
### 值捕获
[var] 或 [=]，捕获变量的副本，lambda 内部修改不会影响外部变量
### 引用捕获
[&var] 或 [&]，捕获变量的引用，lambda 内部修改会影响外部变量
### 混合捕获
[&, var1, =]，可以混合使用默认捕获方式和特定变量的捕获方式
## 示例
```C++
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    vector<int> nums = {10, 5, 20, 15};
    int threshold = 12;
    // 示例 1: 使用 lambda 作为 sort 的比较器 (值捕获)
    // 捕获列表为空 []，因为不需要访问外部变量
    sort(nums.begin(), nums.end(), [](int a, int b) {
        return a > b; // 降序排序
    });
    // 示例 2: 使用 lambda 和 for_each 统计大于 threshold 的元素 (值捕获 =)
    int count = 0;
    // [=] 表示按值捕获所有需要的外部变量 (这里是 threshold 和 count)
    // [&count, threshold] 表示 count 按引用捕获，threshold 按值捕获
    for_each(nums.begin(), nums.end(), [&count, threshold](int n) {
        if (n > threshold) {
            count++; // 修改 count，因为它被引用捕获
        }
    });
    // 示例 3: 带返回类型和 mutable 的 lambda
    int x = 10;
    // [x] 值捕获 x，mutable 允许修改捕获的副本
    auto add_and_modify = [x](int y) mutable -> int {
        x += 1; // 修改的是 lambda 内部的 x 副本
        return x + y;
    };
    return 0;
}
```