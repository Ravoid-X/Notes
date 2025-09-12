## Map
``#include <map> ``
map的底层是红黑树，因此 map 内部所有的数据都是有序的，查询、插入、删除操作的时间复杂度都是O(logn)。
1. 创建
```
std:map<int, string> stu;
```
2. 插入
```
// 用insert函數插入pair，已有关键字时，insert无法插入
stu.insert(pair<int, string>(000, "stu_zero"));
// 用insert函数插入value_type数据
stu.insert(map<int, string>::value_type(001, "stu_one"));
// 用"array"方式插入，可以覆盖
stu[123] = "stu_first";
stu[456] = "stu_second";
```
3. 查找
```
map<int,string>::iterator iter; 
iter = stu.find("123");
cout<<"Find, the value is"<<iter->second<<endl;
stu.find(key) == stu.end(); //如果key不存在，则find返回map::end
stu.count(key);   //与上等价
```
4. 删除
```
stu.erase(iter);  //迭代器刪除
//用关键字刪除，刪除了會返回1，否則返回0
int n = stu.erase("123"); 
```
## unordered_map
``#include<unordered_map>``
unordered_map内部元素是无序的。unordered_map的底层是一个防冗余的哈希表（开链法避免地址冲突）