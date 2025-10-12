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
7. 排序
```
#include <algorithm>
sort(v.begin(), v.end()); //从小到大
sort(v.begin(), v.end(),greater<int>()); //从大到小
sort(v.begin(), v.end(),[](const vector<int>&a,const vector<int>&b){return a[1]<b[1];}); //二维
bool cmp(vector<int>&a,vector<int>&b){
    if(a[0]!=b[0]) return a[0]<b[0];
    else return a[1]>b[1];
 }//第一列升序排序，如何第一列相同时，按照第二列降序排序
sort(v.begin(),v.end(),cmp);
```
## arr，&arr[0]，&arr
```
int arr[10]={0};
printf("%p\n",arr);//首元素的地址
printf("%p\n",arr+1);

printf("%p\n",&arr[0]);//首元素的地址
printf("%p\n",&arr[0]+1);

printf("%p\n",&arr);//整个数组元素的地址
printf("%p\n",&arr+1); 
```
1. 首个打印结果一样，都是数组首元素的地址。&arr 应该是整个元素的地址，打印出来也是首元素的地址。
2. 地址做加法运算后有不同，arr+1 和&arr[0]+1 都只移动 4 个字节，但是 &arr+1 移动了 40 个字节。
3.  arr 本身不是地址而是指代整个数组，会隐式转成指针。