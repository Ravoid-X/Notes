## Vector
``include <vector>``
1. 创建
```
vector<int> a;           //构造一个空的vector,
vector<int> a(10);       //定义了10个整型元素的向量，但没有给出初值
vector<int> a(10,1);     //定义了10个整型元素的向量，且给出每个元素的初值为1
vector<int> a(b);        //用b向量来创建a向量，整体复制性赋值， 拷贝构造
vector<int> v3=a ;       //移动构造
vector<int> a(b.begin(),b.begin+3);   //定义了a值为b中第0个到第2个（共3个）元素
int b[7]={1,2,3,4,5,9,8};
vector<int> a(b,b+6);    //从数组中获得初值，b[0]~b[5]
```
2. 插入
```
a.push_back(1);   //尾部插入
vec.insert(vec.begin()+i,a);  //在第i+1个元素前面插入
```
3. 查找
```
vector<int>::iterator iter;
iter=find(v.begin(),v.end(),element);
std::distance(v.begin(),iter);
```
4. 删除
```
vec.erase(vec.begin()+2);  //删除第3个元素
```
5. 去重
```
vector<int>::iterator iter;
sort(v.begin(), v.end());
iter = unique(v.begin(), v.end());
if (iter != v.end()) {
    v.erase(iter, v.end());
}
```
5. 交集
```
vector<int> v;
sort(v1.begin(),v1.end());   
sort(v2.begin(),v2.end());   
set_intersection(v1.begin(),v1.end(),v2.begin(),v2.end(),back_inserter(v));
```
6. 并集
```
vector<int> v;
sort(v1.begin(),v1.end());   
sort(v2.begin(),v2.end());   
set_union(v1.begin(),v1.end(),v2.begin(),v2.end(),back_inserter(v));
```
## List
``#include <list> ``
随机存取效率底下，可以高效执行任意地方的插入和删除操作。
1. 创建
```
list<int> a; // 定义一个int类型的列表a
list<int> a(10); // 定义一个int类型的列表a，并设置初始大小为10
list<int> a(10, 1); // 定义一个int类型的列表a，并设置初始大小为10且初始值都为1
list<int> b(a); // 定义并用列表a初始化列表b
deque<int> b(a.begin(), ++a.end());// 将列表a中的第1个元素作为列表b的初始值
int n[] = { 1, 2, 3, 4, 5 };
list<int> a(n, n + 5);
```
2. 容量
```
lst.max_size(); //容器最大容量
lst.resize();   //更改容器大小
lst.empty();    //容器判空
```
3. 添加
```
lst.push_front(const T& x); //头部添加
lst.push_back(const T& x);  //末尾添加
lst.insert(iterator it, const T& x);  //任意位置插入
lst.insert(iterator it, int n, const T& x); //任意位置插入 n 个相同元素
lst.insert(iterator it, iterator first, iterator last); //插入另一个向量的 [first,last] 间的数据
```
4. 查找
```
list<int>::iterator it = find(lst.begin(), lst.end(), 10); 
lst.front();
lst.back();
```
5. 删除
```
lst.pop_front();  //头部删除
lst.pop_back();   //末尾删除
lst.erase(iterator it);  //任意位置删除
lst.erase(iterator first, iterator last); //区间删除
``` 