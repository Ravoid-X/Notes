# 定义
&emsp;&emsp;有标注的训练数据集中学习出一个模型进行预测， 常见任务包括分类（离散）与回归（连续）
# 线性回归
&emsp;&emsp;利用回归方程(函数)对一个或多个自变量(特征值)和因变量(目标值)之间关系进行建模的一种分析方式
## 房屋价格预测
<img src="../../Pic/AI/Base/House_sell_model.png" style="width:520px;padding:10px;"/>

使用plt画出数据集分布
````
import numpy as np
import matplotlib.pyplot as plt
plt.style.use('./deeplearning.mplstyle')

x_train = np.array([1.0, 2.0])
y_train = np.array([300.0, 500.0])
plt.scatter(x_train, y_train, marker='x', c='r')
plt.title("Housing Prices")
plt.ylabel('Price (in 1000s of dollars)')
plt.xlabel('Size (1000 sqft)')
plt.show()
````
## 单元线性回归模型
<img src="../../Pic/AI/Base/single_linear_regression_model.png" style="width:520px;padding:10px;"/>

($x^{(i)}$，$y^{(i)}$）表示一对输入输出数据，w、b为模型参数，$\widehat{y}$为模型预测值
$$ \widehat{y}=f_{w,b}(x^{(i)}) = wx^{(i)} + b$$
## 代价函数
$$J(w,b) = \frac{1}{2m} \sum\limits_{i = 0}^{m-1} (f_{w,b}(x^{(i)}) - y^{(i)})^2$$
需要调整w、b使J最小，如下图所示
<img src="../../Pic/AI/Base/goal_of_regression.png" style="width:520px;padding:10px;"/>

## 梯度下降
&emsp;&emsp;公式被定义为
$$\begin{align*} \text{repeat}&\text{ until convergence:} \; \lbrace \newline
\;  w &= w -  \alpha \frac{\partial J(w,b)}{\partial w}   \; \newline 
 b &= b -  \alpha \frac{\partial J(w,b)}{\partial b}  \newline \rbrace
\end{align*}$$
$$
\begin{align*}
\frac{\partial J(w,b)}{\partial w}  &= \frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{w,b}(x^{(i)}) - y^{(i)})x^{(i)} \\
  \frac{\partial J(w,b)}{\partial b}  &= \frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{w,b}(x^{(i)}) - y^{(i)}) \\
\end{align*}
$$
<img src="../../Pic/AI/Base/slopes.png" style="width:520px;padding:10px;"/>

## 学习率
控制调整网络参数的速度，学习率大能更快收敛但可能达不到局部最优点
<img src="../../Pic/AI/Base/learningrate.png" style="width:520px;padding:10px;"/>

<img src="../../Pic/AI/Base/alpha_too_big.png" style="width:520px;padding:10px;"/>

# 多元线性回归
## 输入x的每一行是一个例子
$$\mathbf{X} = 
\begin{pmatrix}
 x^{(0)}_0 & x^{(0)}_1 & \cdots & x^{(0)}_{n-1} \\ 
 x^{(1)}_0 & x^{(1)}_1 & \cdots & x^{(1)}_{n-1} \\
 \cdots \\
 x^{(m-1)}_0 & x^{(m-1)}_1 & \cdots & x^{(m-1)}_{n-1} 
\end{pmatrix}
$$
## 模型
$$ f_{\mathbf{w},b}(\mathbf{x}) = \mathbf{w} \cdot \mathbf{x} + b  $$ 
<img src="../../Pic/AI/Base/multi_linear_regression_model.png" style="width:700px;padding:10px;"/>

## 代价函数
$$J(\mathbf{w},b) = \frac{1}{2m} \sum\limits_{i = 0}^{m-1} (f_{\mathbf{w},b}(\mathbf{x}^{(i)}) - y^{(i)})^2 $$ 
$$ f_{\mathbf{w},b}(\mathbf{x}^{(i)}) = \mathbf{w} \cdot \mathbf{x}^{(i)} + b  $$
$$\begin{align*} \lbrace \newline\;
& w_j = w_j -  \alpha \frac{\partial J(\mathbf{w},b)}{\partial w_j} ; & \text{for j = 0..n-1}\newline
&b\ \ = b -  \alpha \frac{\partial J(\mathbf{w},b)}{\partial b}  \newline \rbrace
\end{align*}$$
## 特征缩放
<img src="../../Pic/AI/Base/feature_scale.png" style="width:700px;padding:10px;"/>

### w的更新速度不相同原因:
1. 学习率共享
2. 微分项有$x^{(i)}$，尺度不同
$$
\begin{align*}
\frac{\partial J(w,b)}{\partial w}  &= \frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{w,b}(x^{(i)}) - y^{(i)})x^{(i)} \\
\end{align*}
$$
### 为解决此问题，有三种技术
1. 除以最大绝对值，缩放到[-1,1]
2. 均值归一化
$$x_i= \dfrac{x_i - \mu_i}{max - min} $$
3. z-score归一化
$$x^{(i)}_j = \dfrac{x^{(i)}_j - \mu_j}{\sigma_j}$$
$$\begin{align*}
\mu_j &= \frac{1}{m} \sum_{i=0}^{m-1} x^{(i)}_j \\
\sigma^2_j &= \frac{1}{m} \sum_{i=0}^{m-1} (x^{(i)}_j - \mu_j)^2 
\end{align*}
$$
>注意 : 对于新的数据，也应该进行归一化

如下图，特征归一化后，梯度下降对每个参数更新进展相同
<img src="../../Pic/AI/Base/contours.png" style="width:360px;padding:10px;"/>

## 特征工程
1. 选择多项式来拟合非线性
2. 变换或组合特征生成与输出更相关的新特征
<img src="../../Pic/AI/Base/feature_eng.png" style="width:800px;padding:10px;"/>

# 逻辑回归
在线性回归的基础上，来解决分类问题。
## sigmoid
<img src="../../Pic/AI/Base/logistic_regression.png" style="width:500px;padding:10px;"/>

$f_{w,b}(x)$表示为1的概率
## 代价函数
<img src="../../Pic/AI/Base/logistic_error.png" style="width:500px;padding:10px;"/>

使用平方误差成本函数作为代价函数，是非凸函数，无法平滑进行梯度下降
> **Loss** 单样本预测误差函数  
**Cost** 训练集的损失和

### 定义新的损失函数 
$$\begin{equation*}
  loss(f_{\mathbf{w},b}(\mathbf{x}^{(i)}), y^{(i)}) = \begin{cases}
    - \log\left(f_{\mathbf{w},b}\left( \mathbf{x}^{(i)} \right) \right) & \text{if $y^{(i)}=1$}\\
    -\log \left( 1 - f_{\mathbf{w},b}\left( \mathbf{x}^{(i)} \right) \right) & \text{if $y^{(i)}=0$}
  \end{cases}
\end{equation*}$$
简化为
$$loss(f_{\mathbf{w},b}(\mathbf{x}^{(i)}), y^{(i)}) = (-y^{(i)} \log\left(f_{\mathbf{w},b}\left( \mathbf{x}^{(i)} \right) \right) - \left( 1 - y^{(i)}\right) \log \left( 1 - f_{\mathbf{w},b}\left( \mathbf{x}^{(i)} \right) \right)$$
<img src="../../Pic/AI/Base/logistic_loss_a.png" style="width:500px;padding:10px;"/>

<img src="../../Pic/AI/Base/logistic_loss_b.png" style="width:500px;padding:10px;"/>

### 新的代价函数
<img src="../../Pic/AI/Base/logistic_cost.jpg" style="width:500px;padding:10px;"/>

<img src="../../Pic/AI/Base/logistic_gradient_descent.png" style="width:500px;padding:10px;"/>

$$\begin{align*}
\frac{\partial J(\mathbf{w},b)}{\partial w_j}  &= \frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{\mathbf{w},b}(\mathbf{x}^{(i)}) - y^{(i)})x_{j}^{(i)}  \\
\frac{\partial J(\mathbf{w},b)}{\partial b}  &= \frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{\mathbf{w},b}(\mathbf{x}^{(i)}) - y^{(i)})  
\end{align*}$$
# 过拟合
<img src="../../Pic/AI/Base/fit_model.jpg" style="width:500px;padding:10px;"/>

## 解决方法
1. 加入更多数据
<img src="../../Pic/AI/Base/overfit_a.png" style="width:500px;padding:10px;"/>

2. 选择部分特征
缺点：无法利用所有数据信息
<img src="../../Pic/AI/Base/overfit_b.png" style="width:500px;padding:10px;"/>

3. 正则化
保留所有特征，只防止一些特征影响过大
<img src="../../Pic/AI/Base/overfit_c.png" style="width:500px;padding:10px;"/>

## 线性回归正则化
正则化项使w影响更小，有助于减少过拟合\
一般不对b进行正则化，没有效果
$$J(\mathbf{w},b) = \frac{1}{2m} \sum\limits_{i = 0}^{m-1} (f_{\mathbf{w},b}(\mathbf{x}^{(i)}) - y^{(i)})^2  + \frac{\lambda}{2m}  \sum_{j=0}^{n-1} w_j^2$$ 
<img src="../../Pic/AI/Base/linear_cost_regularized.png" style="width:500px;padding:10px;"/>

<img src="../../Pic/AI/Base/linear_gradient_regularized.png" style="width:500px;padding:10px;"/>

## 逻辑回归正则化
<img src="../../Pic/AI/Base/logistic_cost_regularized.png" style="width:500px;padding:10px;"/>

<img src="../../Pic/AI/Base/logistic_gradient_regularized.png" style="width:500px;padding:10px;"/>