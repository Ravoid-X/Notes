## 概述
1. 核心目标是在一组线性等式和不等式约束下，寻找一个多变量二次函数的最优值
2. 一个 QP 问题的解被定义和验证是基于两个核心的数学理论：KKT 条件和拉格朗日对偶
## 数学定义
> 二次规划中的“规划”一词，并非指计算机编程，而是指数学规划，要与二次方程区分
### 数学表达
QP 问题通常用其规范（或标准）形式来表述。设决策变量为 $n$ 维向量 $\mathbf{x} \in \mathbb{R}^n$，标准的最小化 QP 问题定义如下
$$\begin{aligned}
& \underset{\mathbf{x}}{\text{minimize}}
& & \frac{1}{2} \mathbf{x}^T \mathbf{P} \mathbf{x} + \mathbf{q}^T \mathbf{x} \\
& \text{subject to}
& & \mathbf{G} \mathbf{x} \leq \mathbf{h} \\
& & & \mathbf{A} \mathbf{x} = \mathbf{b}
\end{aligned}$$
### 解释
1. $\mathbf{P} \in \mathbb{R}^{n \times n}$ 是一个对称矩阵，定义了目标函数的二次项。该矩阵是目标函数关于 $\mathbf{x}$ 的海森矩阵
2. $\mathbf{q} \in \mathbb{R}^n$ 是一个向量，定义了目标函数的线性项
3. $\mathbf{G} \in \mathbb{R}^{m \times n}$ 和 $\mathbf{h} \in \mathbb{R}^m$ 分别定义了 $m$ 个线性不等式约束
4. $\mathbf{A} \in \mathbb{R}^{p \times n}$ 和 $\mathbf{b} \in \mathbb{R}^p$ 分别定义了 $p$ 个线性等式约束
5. $\mathbf{G} \mathbf{x} \leq \mathbf{h}$ 表示向量的逐元素比较
## 特性区分
### 可分离 & 不可分离
1. 可分离：QP 的海森矩阵 $\mathbf{P}$ 是对角矩阵。目标函数是简单的各变量平方和，例如 $\sum a_i x_i^2$
2. 不可分离：QP 的 $\mathbf{P}$ 矩阵有非零的非对角线元素，这意味着目标函数中存在交叉项，例如 $x_1 x_2$
## 示例
### 问题
$$\begin{aligned}
& \underset{x_1, x_2}{\text{minimize}}
& & f(x_1, x_2) = x_1^2 + x_2^2 - 4x_1 - 6x_2 \\
& \text{subject to}
& & g_1: x_1 + x_2 \leq 2 \\
& & & g_2: 2x_1 + x_2 \leq 3
\end{aligned}$$
### KKT 系统构建
#### 海森矩阵
$\mathbf{P} = \begin{bmatrix} 2 & 0 \\ 0 & 2 \end{bmatrix}$，是正定的，因此问题是严格凸的，KKT 条件是充要的
#### 拉格朗日函数
$\mathcal{L} = (x_1^2 + x_2^2 - 4x_1 - 6x_2) + \lambda_1(x_1 + x_2 - 2) + \lambda_2(2x_1 + x_2 - 3)$
#### KKT 条件
1. 平稳性:
$$\partial\mathcal{L}/\partial x_1 = 2x_1 - 4 + \lambda_1 + 2\lambda_2 = 0$$
$$\partial\mathcal{L}/\partial x_2 = 2x_2 - 6 + \lambda_1 + \lambda_2 = 0$$
2. 原始可行性
$$x_1 + x_2 \le 2$$
$$2x_1 + x_2 \le 3$$
3. 对偶可行性
$$\lambda_1 \ge 0$$
$$\lambda_2 \ge 0$$
4. 互补松弛性
$$\lambda_1(x_1 + x_2 - 2) = 0$$
$$\lambda_2(2x_1 + x_2 - 3) = 0$$
### 四种情况
基于互补松弛性，必须检查 4 种可能的有效集组合
#### 两个约束都未激活（$\lambda_1 = 0, \lambda_2 = 0$）
1. 根据平稳性：$x_1 = 2$，$x_2 = 3$
2. 检查原始可行性发现不符合，因此不是 QP 的解
#### $g_1$ 激活，$g_2$ 未激活（$\lambda_1 \ge 0, \lambda_2 = 0$）
1. 有三个方程求解三个未知数\
（1）$2x_1 - 4 + \lambda_1 = 0$\
（2）$2x_2 - 6 + \lambda_1 = 0$\
（3）$x_1 + x_2 - 2 = 0$ (来自 $g_1$ 激活)
2. 得 $x_1 = 0.5$，$x_2 = 1.5$，$\lambda_1 = 3$
3. 满足原始和对偶可行性，因此是 QP 的解
#### $g_2$ 激活，$g_1$ 未激活（$\lambda_1 = 0, \lambda_2 \ge 0$）
违反原始可行性
#### 两个约束都激活（$\lambda_1 \ge 0, \lambda_2 \ge 0$）
违反对偶可行性
### 最终解
在四种可能性中，只有情形 2 成功地满足了全部 KKT 条件。由于问题是凸的，KKT 条件是充分的，因此已经找到了全局最优解
$$\mathbf{x}^* = (x_1, x_2) = (0.5, 1.5) \quad \text{且} \quad (\lambda_1, \lambda_2) = (3, 0)$$