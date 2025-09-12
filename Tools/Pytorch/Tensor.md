## 张量初始化
### 固定尺寸和值
```
auto b = torch::zeros({3,4});
b = torch::ones({3,4});
b= torch::eye(4);
b = torch::full({3,4},10);
b = torch::tensor({33,22,11});
```
### 固定尺寸，随机值
```
auto r = torch::rand({3, 4});
r = torch::randn({3, 4});
r = torch::randint(0, 4,{3,3});
```
randn 取正态分布 N(0,1) 的随机值，randint 取 [min,max) 的随机整型数值
### 从c++的其他数据类型转换
```
int aa[10] = {3,4,6};
std::vector<float> aaaa = {3,4,6};
auto aaaaa = torch::from_blob(aa,{3},torch::kFloat);
auto aaa = torch::from_blob(aaaa.data(),{3},torch::kFloat);
```
### 已有张量
```
auto b = torch::zeros({3,4});
auto d = torch::Tensor(b);
d = torch::zeros_like(b);
d = torch::ones_like(b);
d = torch::rand_like(b,torch::kFloat);
d = b.clone();
```
1. auto a = torch::Tensor(b) 等价于 auto a = b，两者初始化的张量 a 均随原张量 b 变化，如果 b 只是张量变形，a 却不会跟着变形，称为浅拷贝。
2. clone() 是拷贝成一个新的张量，原张量 b 的变化不会影响 a，这被称作深拷贝。
## 张量变形
```
auto b = torch::full({10},3);
b.view({1, 2,-1});
b = b.view({1, 2,-1});
auto c = b.transpose(0,1);
auto d = b.reshape({1,1,-1});
auto e = b.permute({1,0,2});
```
1. view(a,b,c) 操作会返回一个新的张量，它与原始张量共享底层数据。a、b、c表示维度长度，-1 是占位符，pytorch 会自动计算大小。
2. transpose(dim0, dim1) 会交换张量的 dim0 和 dim1 这两个维度。
3. 当底层数据是连续的时，reshape 会返回一个视图（像 view 一样）；如果数据不连续，会创建一个新的数据副本。
4. b 的第 0 维是 1，第 1 维是 2，第 2 维是 5，permute({1, 0, 2}) 的意思是：将原张量的第 1 维作为新张量的第 0 维，将原张量的第 0 维作为新张量的第 1 维，将原张量的第 2 维作为新张量的第 2 维。
## 张量截取
### 使用索引
```
auto b = torch::rand({10,3,28,28});
b[0].sizes();//第 0 张照片
b[0][0].sizes();//第 0 张照片的第 0 个通道
b[0][0][0].sizes();//第 0 张照片的第 0 个通道的第 0 行像素 dim为 1
b[0][0][0][0].sizes();//第 0 张照片的第 0 个通道的第 0 行的第 0 个像素 dim为 0
```
### 其他操作
```
b.index_select(0,torch::tensor({0, 3, 3})).sizes();//选择第 0 维的 0，3，3 组成新张量 [3,3,28,28]
b.index_select(1,torch::tensor({0,2})).sizes(); //选择第 1 维的第 0 和第 2 的组成新张量 [10, 2, 28, 28]
b.index_select(2,torch::arange(0,8)).sizes(); //选择 10 张图片每个通道的前 8 列的所有像素 [10, 3, 8, 28]
b.narrow(1,0,2).sizes();//选择第 1 维，从 0 开始，截取长度为2的部分张量 [10, 2, 28, 28]
b.select(3,2).sizes();//选择第 3 维度的第 2 个张量，即所有图片的第 2 行组成的张量 [10, 3, 28]
```
### index
index需要单独说明用途。在pytorch中，通过掩码Mask对张量进行筛选是容易的直接Tensor[Mask]即可。但是c++中无法直接这样使用，需要index函数实现，
```
auto c = torch::randn({3,4});
auto mask = torch::zeros({3,4});
mask[0][0] = 1;
std::cout<<c;
std::cout<<c.index({mask.to(torch::kBool)});
```
## 张量间操作
### 拼接和堆叠
```
auto b = torch::ones({3,4});
auto c = torch::zeros({3,4});
auto cat = torch::cat({b,c},1);//在第 1 维 拼接，输出张量 [3,8]
auto stack = torch::stack({b,c},1);//在第 1 维添加一个新的维度，输出 [3,2,4]
```
### 四则运算
```
auto b = torch::rand({3,4});
auto c = torch::rand({3,4});
std::cout<<b<<c<<b*c<<b/c<<b.mm(c.t());
```
直接用 * 和 /，也可以用 .mul 和 .div。矩阵乘法用 .mm，加入批次就是 .bmm。