## 随机样本
1. 总体：试验的全部可能观察值
2. 个体：每一个可能的观察值
3. 容量：总体中包含的个体数
4. 样本：若 $X_n$ 是具有同一分布函数F的、相互独立的随机变量，则称 $X_n$ 为从分布函数 $F$（或总体 $F$ 或总体 $X$）得到的容量为 $n$ 的简单随机样本，其观察值 $x_n$ 称为样本值，又称为 $X$ 的 $n$ 个独立的观察值
5. 统计量：假设有一个整体 $X$，从中选取一组样本 $X_1,X_2,……,X_n$，由这些随机变量组成的函数 $g(X_1,X_2,……,X_n)$ 亦是一个随机变量，称为统计量
## 统计图
1. 箱线图：通过5个数字来描述数据的分布的标准方式，包括最小值，第一四分位数（Q1），中位数，第三四分位数（Q3），最大值。能够明确的展示离群点的信息，同时能够让我们了解数据是否对称，如何分组，及峰度。\
2. 解释\
<img src="../../Pic/Subject/Probability/sample-boxplot.png" style="width:400px;padding:10px;"/>\
（1）第一个四分位数（Q1 / 25百分位数）：最小数（不是“最小值”）和数据集的中位数之间的中间数\
（2）第三四分位数（Q3 / 75th Percentile）：数据集的中位数和最大值之间的中间值（不是“最大值”）；\
（3）四分位间距（IQR）：第25至第75个百分点的距离；\
（4）晶须（蓝色显示）\
（5）离群值（显示为绿色圆圈）\
（6）最大值：Q3 + 1.5 * IQR\
（7）最小值：Q1 -1.5 * IQR

## $\chi^2$分布
1. 假设 $X_1,X_2,……,X_n$ 都是来自标准正态分布 $N(0,1)$ 的样本，那么统计量\
$\chi^2=X_1^2+X_2^2+……+X_n^2$\
服从自由度为 $n$ 的 $\chi^2$ 分布，简记 $\chi^2\sim \chi^2(n)$
2. 概率密度函数：
$$f(x)=\begin{cases}
\frac {x^{\frac n 2-1}e^{-\frac x 2}} {2^{\frac n 2}\Gamma(\frac n 2)}\hspace{3em}x>0\\
0\hspace{6em}other case\\
\end{cases}$$
3. $E(\chi^2)=n,D(\chi^2)=2n$
4. 可加性：\
$\chi^2_1+\chi^2_2\sim\chi^2(n_1+n_2)$
5. 上 $\alpha$ 分位数：\
$\chi^2_\alpha(n)\approx\frac 1 2(z_\alpha+\sqrt{2n-1})^2$
## $t$分布
1. 若两随机变量分别服从 $X\sim N(0,1),Y\sim \chi^2(n)$，则随机变量\
$t=\frac X {\sqrt {\frac Y n}}$\
服从自由度为 $n$ 的 $t$ 分布，简记 $t\sim t(n)$
2. 概率密度函数：\
$h(t)=\frac {\Gamma(\frac {n+1} 2)} {\sqrt {\pi n}\Gamma(\frac n 2)}(1+\frac {t^2} n)^{-\frac {n+1} 2}$\
当 $n\to \infty$ 时，上式就趋于标准正态分布。在实用时，当 $n>45$ 就可近似。
3. 上 $\alpha$ 分位数因轴对称具有性质\
$t_{1-\alpha}(n)=-t_\alpha(n)$\
当 $n>45$，取 $t_\alpha(n)\approx z_\alpha$
## $F$分布
1. 若两个相互独立的随机变量服从 $X\sim \chi^2(n_1),Y\sim \chi^2(n_2)$，则随机变量\
$t=\frac {\frac X {n_1}} {\frac Y {n_2}}$\
服从自由度为 $(n_1,n_2)$ 的 $F$ 分布，简记 $F\sim F(n_1,n_2)$
2. 概率密度函数：\
$$\psi(x)=\begin{cases}
\frac{\Gamma(\frac{n_1|n_2}2)(\frac{n_1}{n_2})^{\frac{n_1}2}x^{\frac{n_1}2-1}}{\Gamma(\frac{n_1}2)\Gamma(\frac{n_2}2)(1+\frac{n_1x}{n_2})^{\frac {n_1+n_2}2}}\hspace{2em} x>0\\
0\hspace{10em} other case\\
\end{cases}$$
3. $\frac 1 F\sim F(n_1,n_2)$
4. 上 $\alpha$ 分位数：$F_{1-\alpha}(n_1,n_2)=\frac 1 {F_\alpha(n_2,n_1)}$