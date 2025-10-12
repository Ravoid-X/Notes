## Set
基于红黑树实现，具有自动排序的功能
``#include <set>``
1. 创建
```
set<int> st;
```
2. 添加
```
st.insert(4);     //插入元素
set<int>::iterator it = st.begin();
st.insert(it, 2); //任意位置插入
```
3. 删除
```
st.pop_back(4); // 删除值为 4 的元素
set<int>::iterator it = st.begin();
st.erase(it);   //任意位置删除
st.erase(iterator first, iterator last); //删除区间
st.clear();     //清空
```
4. 查找
```
set<int>::iterator it;
it = st.find(2);
cout << *it << endl; // 输出：2
st.count(key); //键 key 的元素个数
```
5. lower_bound(val)
返回数组中大于等于 val 的第一个元素的地址，若均小于 val 则返回尾后地址。
6. upper_bound
返回数组中大于 val 的第一个元素的地址，若均小于等于 val 则返回尾后地址
7. 遍历
```
for(auto it = s.begin(); it != s.end(); ++it) {
    std::cout << *it << " ";
}
```
## unordered_set
基于哈希表，数据插入和查找的时间复杂度很低，几乎是常数时间，而代价是消耗比较多的内存，无自动排序功能。
``#include <unordered_set>``