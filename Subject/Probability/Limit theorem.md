## 切比雪夫不等式（Chebyshev）(pending)
1. 示性函数：对于随机事件 $A$，引入函数如下
$$I_A=\begin{cases}
1，\,\,\,\,\,A发生\\
0，\,\,\,\,\,A不发生\\
\end{cases}$$
2. 马尔可夫不等式\
对于非负的随机变量 $X$ 和定值 $a$ ，考虑随机事件 $A=\{X≥a\}$，其示性函数如下所示\
<img src="../../Pic/Subject/Probability/limit-markov.png" style="width:300px;padding:10px;"/>\
可知$I_{X≥a}(X)≤\frac x a$，对该式取数学期望得马尔可夫不等式\
$I_{X≥a}=P(X≥a)≤\frac {E(X)} a$

3. 切比雪夫不等式\
于随机变量 $X$，记 $\mu=E(X)$，考虑随机事件 $A=\{|X-\mu|≥a\}$，其示性函数如下所示\
<img src="../../Pic/Subject/Probability/limit-chebyshev.png" style="width:300px;padding:10px;"/>\
可知$I_{|X-\mu|≥a}(X)≤\frac {(X-\mu)^2} {a^2}$，对该式取数学期望得切比雪夫不等式\
$I_{|X-\mu|≥a}=P(|X-\mu|≥a)≤\frac {D(X)} {a^2}$

## 大数定律
1. 定义\
对于一系列随机变量 $\{X_n\}$，设每个随机变量都有期望。任取 $\epsilon>0$，若恒有 $lim_{n\to\infty}P(|\bar{X_n}-E(\bar{X_n})|<\epsilon)=1$，称 $\{X_n\}$，服从（弱）大数定律。\
其中 $\bar{X_n}=\frac 1 n\sum ^n _{i=1}X_i$
2. 马尔可夫大数定律\
$P(|\bar{X_n}-E(\bar{X_n})|<\epsilon)≥1-\frac {D(\bar{X_n})} {\epsilon^2}=1-\frac 1 {\epsilon^2n^2}D(\sum ^n _{i=1}X_i)$\
$lim_{n\to\infty}\frac 1 {n^2}D(\sum ^n _{i=1}X_i)=0$
3. 切比雪夫大数定律\
在马尔可夫大数定律的基础上，如果 $\{X_n\}$ 两两不相关，则\
$\frac 1 {n^2}D(\sum ^n _{i=1}X_i)=\frac 1 {n^2}\sum ^n _{i=1}D(X_i)$\
如果 $D(X_i)$ 有共同的上界 $c$,则\
$\frac 1 {n^2}\sum ^n _{i=1}D(X_i)≤\frac c n$\
$P(|\bar{X_n}-E(\bar{X_n})|<\epsilon)≥1-\frac c {\epsilon^2n}$
4. 独立同分布大数定律\
在切比雪夫大数定律的基础上，进一步限制 $\{X_n\}$ 独立同分布
5. 伯努利大数定律\
   
6. 辛钦大数定律\

## 中心极限定理
1. 定义\
若 $\{X_n\}$ 满足一定的条件，当 $n$ 足够大时，$\bar{X_n}$ 近似服从正态分布
2. 独立同分布中心极限定理\
如果 $\{X_n\}$ 独立同分布，且 $E(X)=\mu,D(X)=\sigma^2>0$，则近似服从正态分布 $N(\mu,\frac {\sigma^2} n)$\
$lim_{n\to\infty}P(\frac {\bar{X_n}-\mu} {\sigma/\sqrt n}<a)=\Phi(a)=\int ^a _{-\infty}\frac 1 {\sqrt {2\pi}}e^{\frac {-t^2} 2}dt$
3. 二项分布中心极限定理\

4. Lyapunov中心极限定理\