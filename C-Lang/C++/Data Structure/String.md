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