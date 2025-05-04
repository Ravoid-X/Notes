## 数学期望
1. 一维\
若离散型 $rvs$ 且 $\sum ^∞ _{i=1}x_iP_i$ 绝对收敛，则称其为 $X$ 的数学期望\
若连续型且 $\int ^∞ _{i=1}xf(x)dx$ 绝对收敛，则称其为 $X$ 的数学期望\
2. 性质\
（1）$E(aX+c)=aE(X)+c$\
（2）$E(X±Y)=E(X)±E(Y)$\
（3）若 $X,Y$独立，$E(XY)=E(X)E(Y)$
3. 二维\
若离散型 $rvs$，$g(X,Y)$ 是 $X,Y$的函数，且 $\sum ^∞ _{i=1}\sum ^∞ _{j=1}g(x_i,y_i)P_{ij}$ 绝对收敛，则称其为数学期望\
若连续型且 $\int ^∞ _{-∞}\int ^∞ _{-∞}g(x,y)f(x,y)dxdy$ 绝对收敛，则称其为数学期望\
将 $g(x,y)$ 替换为 $xy$ 就是E(XY)
## 方差
1. 一维定义：$Var(X)$ 或 $D(X)$\
$Var(X)=E[(X-E(X)^2]$
2. 标准差\
$\sigma(X)=\sqrt {Var(x)}$
3. 性质\
（1）$E(X^2)=D(X)+(E(X))^2$\
（2）$D(aX+b)=a^2D(X)$\
（3）$D(X±Y)=D(X)+D(Y)+2Cov(X,Y))$\
（4）若相互独立，$D(aX+bY)=a^2D(X)+b^2D(Y)$
## 矩
1. $k$ 阶原点矩：$E(X^k)$
2. $k$ 阶中心矩：$E((X-E(X))^k)$
3. $k+l$ 阶混合矩：$E(X^kY^l)$
4. $k+l$ 阶混合中心矩：$E((X-E(X))^k(X-E(Y))^l)$
## 协方差
1. 协方差：描述变量间关联程度\
$Cov(X,Y)=E[(X-E(X))(Y-E(Y))]=E(XY)-E(X)E(Y)$
2. 性质\
（1）$Cov(X,Y)=Cov(Y,X)$\
（2）$Cov(X,X)=D(X)$\
（3）$Cov(X,c)=0$\
（4）$Cov(aX+b,Y)=aCov(X,Y)$\
（5）$Cov(X_1+X_2,Y)=Cov(X_1,Y)+Cov(X_2,Y)$
3. 相关系数：为 $0$ 则不相关，描述变量间线性相关性\
$ρ_{XY}=\frac {Cov(X,Y)} {\s qrt {D(X)}\sqrt {D(Y)}}$\
若 $Y=aX+b$，则 $a>0,ρ_{XY}=1;a<0,ρ_{XY}=-1$\
<img src="../../Pic/Subject/Probability/numerical-table.png" style="width:800px;padding:10px;"/>
