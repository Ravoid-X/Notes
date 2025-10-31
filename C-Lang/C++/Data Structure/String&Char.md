# string
`#include <string>`
## 初始化
```
string s1;
string s2 = "Hello, World!";
string s3("Hello, C++!");
string s4 = s2;
string s5(s2);
string s6(s2, 0, 5) // 从 s2 的第 0 个位置开始，取 5 个字符
string s7(10, '-');
```
## 容量
```
s.size();      //与 s.length(); 相同
s.capacity();  //不重新分配内存的情况下，字符串可以容纳的字符数，通常大于或等于 size()
s.empty();
s.resize(n, c); //改变大小为 n。如果 n > size()，用 c 填充；如果 n < size()，截断
reserve(n)   //为字符串预留空间（扩容），不会改变容量大小
s.clear();
```
## 访问
```
s[0];      // 不进行边界检查
s.at(0);   // 进行边界检查，会抛出 out_of_range 异常。
s.front();
s.back();
```
## 添加
```
s = "Hello";
s += " C++";     
s.append("!");    
s.push_back(' '); 
s.append(3, 'A');  // "Hello C++! AAA"
s.insert(6, "C++ ")  // "Hello C++ C++! AAA"
```
## 删除
```
s = "Hello C++ World";
s.erase(6, 4); // "Hello World"
s.pop_back();
```
## 替换
```
s = "I love Python!";
s.replace(7, 6, "C++"); // "I love C++!"
```
## 查找
```
find(str, pos=0); // 从位置 pos 开始，查找子串 str 第一次出现的位置
rfind(str, pos=npos); // 从末尾开始反向查找，查找子串 str 最后一次出现的位置
find_first_of(chars, pos=0); // 查找 chars 中任意一个字符第一次出现的位置
find_last_of(chars, pos=npos); // 查找 chars 中任意一个字符最后一次出现的位置
find_first_not_of(chars, pos=0); //查找不属于 chars 中任意一个字符的第一个字符
find_last_not_of(chars, pos=npos); //反向查找不属于 chars 中任意一个字符的第一个字符
```
## 比较
```
compare(str); // 与另一个字符串 str 进行字典序比较。
返回 0：两个字符串相等。
返回 < 0：当前字符串小于 str。
```
## 分割
```
substr(pos=0, count=npos); // 获取从 pos 开始，长度为 count 的子串
```
```
#include <sstream>
vector<string> tokens;
string token;
istringstream tokenStream(s);
// getline 会从流中读取字符，直到遇到分隔符 delimiter
while (getline(tokenStream, token, delimiter)) {
    tokens.push_back(token);
}
```
## 转换
```
int i = stoi(s);
long l = stol(s);
long long = stoll(s);
double d = stod(s);
float f = stof(s);
string s = to_string(i);
```
# char
## 定义
```
char ch = 'A';
char str[] = "Hello";
```
## 遍历
```
for (int i = 0; str[i] != '\0'; ++i) {
    cout << str[i] << " ";
}
```
## 相关函数
```
#include <cctype>
isalpha();  //判断字符是否为字母字符
isdigit();  //判断字符是否为数字字符
tolower();  //将字符转换为小写字母
toupper();  //将字符转换为大写字母
```
## 内存空间
```
char s1[] = {'A', 'A', 'A'}; //没有'\n'，3 个字节
char s2[] = "AAA";           //有'\n'，4 个字节
```
## char a,char a[],char *a,char *[],char **a 
### char a
定义了一个存储空间，存储的是 char 类型的变量
### char a[]
字符数组，数组中的每一个元素是一个 char 类型的数据
### char *a
1. 字符串的本质（在计算机眼中）是其第一个字符的地址，所以和 char a[] 没有本质区别
2. 对于 char s[] 和 char* a，可以有 a=s，但不能有 s=a。
3. 创建数组的时候 s 的地址不为空已经确定，但是 a 能改变，不能把可变的地址传给不变的常量。
### char *a[]
1. * 的优先级是低于 [] 的，因此要先看 a[] 再看 *
2. 一个 char 数组，数组中的每一个元素都是指针，这些指针指向 char 类型
3. `char *a[ ] = {"China","French","America","German"}`
### char **a
1. 两个 ** 代表相同的优先级，因此从右往左看，即 char*(*a)
2. 和 char *a[] 一样