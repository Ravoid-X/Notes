## 二维随机变量（pending）
若随机变量来自同一个样本空间，它们之间可能存在某些联系，取两个随机变量 $X,Y$构成 $(X,Y)$，称为二维随机变量。
1. 联合分布函数\
$F(x)=P(X≤x,Y≤y)$\
2. 联合概率密度\
$F(x,y)=\int ^y _{-∞}\int ^x _{-∞}f(u,v)dudv$
3. 边缘分布函数\
$F_X(x)=P(X≤x)$
4. 边缘概率密度\
$f_X(x)=\int ^{+∞} _{-∞}f(x,y)dy$\
$f_Y(y)=\int ^{+∞} _{-∞}f(x,y)dx$
## 条件分布
1. 离散型\
$P(X=x_i|Y=y_i)=\frac {P(X=x_i,Y=y_i)} {P(Y=y_i)}$
2. 连续型概率密度\
$f_{X|Y}(x|y)=\frac {f(x,y)} {f_Y(y)}$
3. 连续性分布函数\
$F_{X|Y}(x|y)=P(X≤x|Y=y)=\int ^x _{-∞}f_{X|Y}(x|y)dx$
## 分布
1. Z=X+Y\
$$F_Z(z)=P(Z≤z)=\int\int _{x+y≤z}f(x,y)dxdy\\
=\int ^z _{-∞} [\int ^{+∞} _{-∞} f(u-y,y)dy]du\\
$$
求导可知\
$f_Z(z)=\int ^{+∞} _{-∞} f(z-y,y)dy$\
变量相互独立时，可以拆分为：\
$$f_Z(z)=\int ^{+∞} _{-∞} f_X(x)f_Y(z-x)dx\\
=\int ^{+∞} _{-∞} f_X(z-y)f_Y(y)dy$$
2. Z=XY\
$f_Z(z)=\int ^{+∞} _{-∞} \frac 1 {|x|} f(x,\frac z x)dx$\
若变量相互独立\
$f_Z(z)=\int ^{+∞} _{-∞} \frac 1 {|x|} f_X(x)f_Y(\frac z x)dx$
3. Z=max(X,Y)，$X,Y$ 相互独立\
$F_Z(z)=F_X(z)F_Y(z)$ 
4. Z=min(X,Y)，$X,Y$ 相互独立\
$F_Z(z)=1-(1-F_X(z))(1-F_Y(z))$ 
5. 正态分布\
$f(x,y)=\frac 1 {2\pi σ_1σ_2\sqrt {1-ρ^2}}exp\{-\frac 1 {2(1-ρ^2)}[\frac {(x-\mu_1)^2} {{σ_1}^2}+\frac {(x-\mu _2)^2} {{σ_2}^2}-2ρ\frac {(x-\mu_1)(x-\mu_2)} {σ_1σ_2}]\}$\
其中 $ρ$ 为 $X,Y$ 相关系数，边缘分布函数是各自变量的一维正态分布