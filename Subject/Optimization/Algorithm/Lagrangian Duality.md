## 概述
1. 一种将一个（可能非凸的）约束优化问题（称为原始问题）转换为一个（始终是凸的）新问题（称为对偶问题）的方法
2. 通过求解对偶问题，可以获得原始问题最优解的一个下界（最小化问题），并且在特定条件下（如强对偶性成立），对偶问题的解可以用来直接找到或逼近原始问题的解
## 原始问题
$$\begin{align*}
\min_{x} \quad & f(x) \\
\text{s.t.} \quad & g_i(x) \le 0, \quad i = 1, \dots, m \\
& h_j(x) = 0, \quad j = 1, \dots, p
\end{align*}$$
## 拉格朗日函数
$$L(x, \lambda, \nu) = f(x) + \sum_{i=1}^m \lambda_i g_i(x) + \sum_{j=1}^p \nu_j h_j(x)$$
## 拉格朗日对偶函数
固定拉格朗日乘子 $\lambda$ 和 $\nu$（视为常量），然后尝试在所有可能的 $x$ 上最小化拉格朗日函数 $L(x, \lambda, \nu)$
### 数学公式
拉格朗日对偶函数 $g(\lambda, \nu)$ 被定义为 $L$ 关于 $x$ 的下确界
$$g(\lambda, \nu) = \inf_{x} L(x, \lambda, \nu) = \inf_{x} \left( f(x) + \sum_{i=1}^m \lambda_i g_i(x) + \sum_{j=1}^p \nu_j h_j(x) \right)$$
### 特性
#### 凹函数
1. 无论原始问题 $f(x)$ 和 $g_i(x)$ 是什么（即使它们是非凸的），对偶函数 $g(\lambda, \nu)$ 始终是一个凹函数
2. 因为它是一系列关于 $(\lambda, \nu)$ 的仿射函数（$L$ 是 $\lambda, \nu$ 的线性函数）的逐点下确界
#### 提供 $p^*$ 的下界
对于任何 $\lambda \ge 0$ 和任何 $\nu$，对偶函数的值 $g(\lambda, \nu)$ 总是原始问题最优解 $p^*$ 的一个下界
#### 下界证明
假设 $x^*$ 是原始问题的最优解（即 $f(x^*) = p^*$），并且 $x^*$ 满足所有约束（$g_i(x^*) \le 0$ 且 $h_j(x^*) = 0$）
$$\begin{align*}
g(\lambda, \nu) &= \inf_{x} L(x, \lambda, \nu) \\
&\le L(x^*, \lambda, \nu) \quad (\text{因为 } x^* \text{ 只是 } x \text{ 的一个特定取值}) \\
&= f(x^*) + \sum_{i=1}^m \underbrace{\lambda_i}_{\ge 0} \underbrace{g_i(x^*)}_{\le 0} + \sum_{j=1}^p \nu_j \underbrace{h_j(x^*)}_{= 0} \\
&= p^* + (\text{小于等于 0 的项}) + 0 \\
&\le p^*
\end{align*}$$
因此， $g(\lambda, \nu) \le p^*$ 恒成立
## 对偶问题
已经知道 $g(\lambda, \nu)$ 是 $p^*$ 的下界，要找到最好的 $\lambda$ 和 $\nu$，使得下界 $g(\lambda, \nu)$ 尽可能大
### 标准形式
$$\begin{align*}
\max_{\lambda, \nu} \quad & g(\lambda, \nu) \\
\text{s.t.} \quad & \lambda_i \ge 0, \quad i = 1, \dots, m
\end{align*}$$
这是一个凸优化问题，称对偶问题的最优解（如果存在）为 $d^*$
## 对偶性
### 弱对偶性
始终成立，已知 $g(\lambda, \nu) \le p^*$，因为 $d^*$ 是 $g(\lambda, \nu)
$ 的最大值，所以必然有
$$d^* \le p^*$$
这个差值 $p^* - d^*$ 被称为对偶间隙
### 强对偶性
1. 不一定成立，但希望成立。指的是对偶间隙为零的情况
$$d^* = p^*$$
意味着可以通过求解那个（总是凸的）对偶问题来找到原始问题的最优值
2. Slater 条件： 如果原始问题是凸优化问题（即 $f(x)$ 和 $g_i(x)$ 是凸函数， $h_j(x)$ 是仿射函数），并且存在至少一个严格满足不等式约束的可行点 $x$（即存在 $x$ 使得 $g_i(x) < 0$ 且 $h_j(x) = 0$），那么强对偶性成立