``# include <iostream>``
## Input
1. ``cin>>``
```
//遇到空白（包括回车、空格等）结束输入，并把非int字符留在缓存区（包括回车，空格等）
int m;
cin >> m; 
```
2. ``get()``
```
//回收缓存区的第一个字符（任意字符），通常用来接收由cin产生的空白字符
cin.get();
```
3. ``getline()``
```
//输入一行带空白字符的字符串，遇到回车结束输入
string str;
getline(cin, str);
```
4. 多行带空白字符的字符串
```
string str1;
vector<string> vec;
while (getline(cin, str1)) {
    if (str1.size() != 0) { //当前行只输入回车
        vec.push_back(str1);
    }
    else {
        break;
    }
}
```
## Output
1. 使用'\n'而不是endl，因为 '\n' 只是输出一个换行符，而 endl 则执行输出换行符和刷新输出流（flush）两个操作
2. '\n'只会在程序结束或达到缓冲区存储上限时才会打印，若程序崩溃，有可能看不到输出