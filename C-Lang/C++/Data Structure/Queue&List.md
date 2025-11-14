## queue
`#include <queue>`
先进先出的数据结构，允许在一端添加元素（称为队尾），并在另一端移除元素（称为队首）
### 创建
```C++
queue<Type> q;
priority_queue<Type> q; //自动排序，降序
```
### 容量
```C++
q.size();   //返回容器大小
q.empty();    //容器判空
```
### 添加
```C++
q.push(x); //尾部添加
```
### 查找
```C++
q.front();
q.back();
```
### 删除
```C++
q.pop_front();  //头部删除
``` 
### 结构体排序
```C++
struct cmp{
    bool operator()(node a,node b){
	    return a.c<b.c;
    }
}; //升序
priority_queue<node,vector<node>,cmp> a;
struct{
    XXXX
    bool operator>(const edge& other) const{        
        return w > other.w; 
    }//升序
    bool operator<(const edge& other) const{        
        return w < other.w; 
    }//降序
}
priority_queue<node, vector<node>, greater<node>> pq;//升序
priority_queue<node> pq;//默认降序
```
## deque
``#include <deque> ``

## List
`#include <list>`
随机存取效率底下，可以高效执行任意地方的插入和删除操作。
### 创建
```C++
list<int> a; // 定义一个int类型的列表a
list<int> a(10); // 定义一个int类型的列表a，并设置初始大小为10
list<int> a(10, 1); // 定义一个int类型的列表a，并设置初始大小为10且初始值都为1
list<int> b(a); // 定义并用列表a初始化列表b
deque<int> b(a.begin(), ++a.end());// 将列表a中的第1个元素作为列表b的初始值
int n[] = { 1, 2, 3, 4, 5 };
list<int> a(n, n + 5);
```
### 容量
```C++
lst.max_size(); //容器最大容量
lst.resize();   //更改容器大小
lst.empty();    //容器判空
```
### 添加
```C++
lst.push_front(const T& x); //头部添加
lst.push_back(const T& x);  //末尾添加
lst.insert(iterator it, const T& x);  //任意位置插入
lst.insert(iterator it, int n, const T& x); //任意位置插入 n 个相同元素
lst.insert(iterator it, iterator first, iterator last); //插入另一个向量的 [first,last] 间的数据
```
### 查找
```C++
list<int>::iterator it = find(lst.begin(), lst.end(), 10); 
lst.front();
lst.back();
```
### 删除
```C++
lst.pop_front();  //头部删除
lst.pop_back();   //末尾删除
lst.erase(iterator it);  //任意位置删除
lst.erase(iterator first, iterator last); //区间删除
``` 