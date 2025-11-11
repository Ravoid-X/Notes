## 概述
1. 任务：给定作用于系统的力和力矩，以及系统当前状态（即所有组成部分的位置和速度），计算出系统各部分的加速度。
$$\ddot{q} = \text{FD}(q, \dot{q}, \tau)$$
2. 是物理仿真与模拟的基础，通过在仿真循环中反复求解 $\ddot{q}$，并对其进行数值积分（例如龙格-库塔法），可以预测机器人在给定力矩输入下的完整运动轨迹。
## 仿真循环
正动力学是所有物理仿真的核心引擎，其预测能力通过一个称为“仿真循环”的迭代过程得以实现 
### 流程
这个循环将连续的时间离散化为一系列微小的时间步长 $δt$，并在每个步长内执行以下操作：
1. 状态输入：在时间步 $t$ 开始时，获取系统的当前状态，包括所有广义坐标（如关节角度）的位置 $q(t)$ 和速度 $\dot{q}(t)$。
2. 力输入：确定在当前状态下作用于系统的所有力，包括由控制器计算出的关节力矩 $τ(t)$ 和环境施加的外部力 $F_{ext}(t)$（如重力、接触力等）
3. 正动力学计算：将当前状态 $(q(t), \dot{q}(t))$ 和力 $τ(t)$ 作为输入，调用正动力学算法，求解系统的运动方程，得到该时刻的关节加速度 $\ddot{q}(t)$
4. 数值积分：使用数值积分方法（如欧拉法或更精确的龙格-库塔法），根据计算出的加速度 $\ddot{q}(t)$ 来更新系统的速度和位置，以预测下一个时间步 $t+δt$ 的状态 2。更新速度：$\dot{q}(t + \delta t) \approx \dot{q}(t) + \ddot{q}(t) \cdot \delta t$更新位置：$q(t + \delta t) \approx q(t) + \dot{q}(t) \cdot \delta t + \frac{1}{2} \ddot{q}(t) \cdot \delta t^2$
5. 循环迭代：将新的状态 $(q(t + \delta t), \dot{q}(t + \delta t))$ 作为下一个时间步的输入，重复上述过程
## 显式求逆法 (CRBA + RNEA)
最符合直觉，遵循 $\ddot{q} = \mathbf{M}^{-1} (\tau - c)$ 的数学定义，通过显式地构建方程的各个部分，然后求解
### 算法步骤
1. 计算惯性矩阵 $\mathbf{M}(q)$：使用 CRBA
2. 计算偏置向量 $c(q, \dot{q})$： 使用 RNEA，即 $c = \text{RNEA}(q, \dot{q}, \mathbf{0})$
3. 求解线性系统： 求解 $\mathbf{M}(q) \ddot{q} = \tau - c(q, \dot{q})$ 得到 $\ddot{q}$
### 求解线性系统
由于 $\mathbf{M}$ 是对称正定的，通常使用 Cholesky 分解法（或高斯消元法）来求解此方程组。这些标准数值方法的计算复杂度为 $O(N^3)$
## 复合刚体算法 (Composite-Rigid-Body Algorithm, CRBA)
### 目标
高效计算 $O(n^2)$ 复杂度的 $\mathbf{M}(q)$ 矩阵
### 核心概念
1. 对于连杆 $i$，其对应的“复合刚体”$i$ 被定义为将连杆 $i$ 及其所有下游连杆（$j > i$）“焊接”在一起形成的一个单一刚体
2. $\mathbf{M}$ 矩阵的元素 $M_{ij}$ 可以被证明是这些复合刚体惯性 $\mathbf{I}^C$ 的函数
### 字符定义
1. $p(i)$: 连杆 $i$ 的父连杆索引（例如 $p(3)=2$, $p(1)=0$ 表示基座）
2. $\lambda(i)$: 连杆 $i$ 的子连杆索引（为简化，假设为串联机器人，$\lambda(i) = i+1$）
3. $S_i$: 关节 $i$ 的运动轴（Motion Subspace），是一个 $6 \times 1$ 的空间向量。\
（1）对于转动关节（绕 $z$ 轴）：$S_i = \begin{bmatrix} 0 & 0 & 1 & 0 & 0 & 0 \end{bmatrix}^T$\
（2）对于移动关节（沿 $z$ 轴）：$S_i = \begin{bmatrix} 0 & 0 & 0 & 0 & 0 & 1 \end{bmatrix}^T$
4. ${^i X_j}$: $6 \times 6$ 空间坐标变换矩阵，将空间向量从坐标系 $\{j\}$ 变换到坐标系 $\{i\}$
5. $I_i$: 连杆 $i$ 自身的 $6 \times 6$ 空间惯性张量，在连杆 $i$ 的本地坐标系 $\{i\}$ 中表示。它由连杆质量 $m_i$、质心位置 $c_i$ 和绕质心的 $3 \times 3$ 惯性张量 $I_{c_i}$ 构成
$$I_i = \begin{bmatrix} I_{c_i} + m_i S(c_i)^T S(c_i) & m_i S(c_i) \\ -m_i S(c_i) & m_i \mathbf{1}_{3 \times 3} \end{bmatrix}$$
6. $I_i^C$: 复合刚体 $B_i$（连杆 $i$ 到 $n$）的 $6 \times 6$ 空间惯性张量，在坐标系 $\{i\}$ 中表示
### 反向递归
#### 目标
计算出 $I_1^C, I_2^C, ..., I_n^C$
#### 初始化
在连杆 $n$ 之外没有惯性，定义一个虚拟的 $I_{n+1}^C$
$$I_{n+1}^C = \mathbf{0}_{6 \times 6}$$
#### 反向递归
复合刚体 $B_i$ 的惯性 $I_i^C$ 等于 连杆 $i$ 自身的惯性 $I_i$ 加上 下游复合刚体 $B_{i+1}$ 的惯性 $I_{i+1}^C$（但 $I_{i+1}^C$ 需要先变换到坐标系 $\{i\}$ 中）
$$I_i^C = I_i + {^i X_{\lambda(i)}}^T I_{\lambda(i)}^C {^i X_{\lambda(i)}}$$
${^i X_{\lambda(i)}}^T (\cdot) {^i X_{\lambda(i)}}$ 是将空间惯性张量从坐标系 $\{i+1\}$ 变换到 $\{i\}$ 的标准操作（空间平行轴定理）
### 正向遍历
#### 目标
利用 $I_i^C$ 来填充 $M$ 矩阵，$M$ 是对称的，所以只需要计算上三角（或下三角）和对角线
#### 初始化
$M = \mathbf{0}_{n \times n}$ (一个 $n \times n$ 的零矩阵)
#### 正向遍历
在每次循环中，计算 $M$ 矩阵的第 $i$ 列（以及利用对称性计算第 $i$ 行）
1. 计算空间力 $F$：一个 $6 \times 1$ 的空间力向量
$$F = I_i^C S_i$$
$F$ 是在关节 $i$ 处施加单位加速度 $\ddot{q}_i = 1$（同时所有其他关节加速度为 0）时，在坐标系 $\{i\}$ 处产生的空间力（考虑了 $i$ 之后所有连杆的惯性 $I_i^C$）

2. 计算对角线元素 $M_{ii}$
$$M_{ii} = S_i^T F$$
$M_{ii}$ 是 $F$ 在关节 $i$ 运动轴 $S_i$ 上的投影，即在关节 $i$ 处产生 $\ddot{q}_i = 1$ 所需的力矩/力

3. 向上游传播力 $F$ 并计算非对角线元素：从 $k = i$ 开始，沿着运动链向基座回溯\
（1）while $p(k) \neq 0$ (即 $k$ 不是基座)\
（2）$j = p(k)$ (获取父关节 $j$)\
（3）$F = {^j X_k}^T F$ (将力 $F$ 从坐标系 $\{k\}$ 变换到父坐标系 $\{j\}$)\
（4）$H_{ji} = S_j^T F$ (计算 $H_{ji}$：这个力 $F$ 在父关节 $j$ 轴上的投影)\
（5）$H_{ij} = H_{ji}$ (利用对称性填充 $H_{ij}$)\
（6）$k = j$ (继续向上游回溯)
## 铰接体算法（Articulated-Body Algorithm, ABA）
### 概述
1. 能够以 $O(N)$ 的计算复杂度直接求解正动力学问题。这是理论上的最优解，因为算法必须至少访问每个连杆一次
2. 不显式地构造 $\mathbf{M}$ 矩阵，对惯性矩阵的逆进行递归分解实现隐式求逆。
3. $O(N)$ 性能的关键在于“铰接体惯量”这一核心概念
### 铰接体惯量
1. 考虑一个连杆 $i$ 作为其所有下游子连杆（即其子树）的“句柄”。这个由连杆 $i$ 及其可相对运动的子树组成的系统，被称为铰接体 $A_i$
2.  $A_i$ 的内部连杆可以运动，但施加在“句柄”连杆 $i$ 上的空间力 $\mathbf{f}_i$ 与其产生的空间加速度 $\mathbf{a}_i$ 之间，仍然存在一个线性关系
$$\mathbf{f}_i = \mathbf{IA}_i \mathbf{a}_i + \mathbf{pA}_i$$
$\mathbf{IA}_i$ 是 $A_i$ 的铰接体惯量，$\mathbf{pA}_i$ 是铰接体偏置力，它包含了该子系统内部所有的速度、重力效应和下游施加的力矩

3. ABA 算法的本质就是递归地计算这些 $\mathbf{IA}_i$ 和 $\mathbf{pA}_i$ 项，从叶节点开始，向内传播到根节点
## 计算流程
### 从根到叶（运动学）
#### 目标
计算每个连杆 $i$ 的速度 $\mathbf{v}_i$、速度引起的偏置加速度 $\mathbf{c}_i$（包含科里奥利力和离心力效应）以及刚体偏置力 $\mathbf{p}_i$
#### 递归
1. $\mathbf{v}_J = \mathbf{S}_i \dot{q}_i$ (关节速度)
2. $\mathbf{v}_i = \mathbf{v}_{\lambda(i)} + \mathbf{v}_J$ (传播连杆速度) 
3. $\mathbf{c}_i = \mathbf{c}_{\lambda(i)} + \mathbf{v}_i \times \mathbf{v}_J$ (偏置加速度，即 $\ddot{q}=0$ 时的加速度) 
4. $\mathbf{p}_i = \mathbf{v}_i \times^* \mathbf{I}_i \mathbf{v}_i - \mathbf{f}^{\text{ext}}_i$ (刚体偏置力，包含陀螺力和外部力)
### 从叶到根（铰接体惯量）
#### 目标
算法的核心。从叶节点向内传播，计算每个连杆的铰接体惯量 $\mathbf{IA}_i$ 和铰接体偏置力 $\mathbf{pA}_i$
#### 初始化 (叶节点 $i$)
1. $\mathbf{IA}_i = \mathbf{I}_i$ (叶节点的铰接体惯量等于其刚体惯量)
2. $\mathbf{pA}_i = \mathbf{p}_i$ (叶节点的铰接体偏置力等于其刚体偏置力)
#### 递归
1. 计算关节 $i$ 的中间项（这些项代表了关节 $i$ 的自由度对 $\mathbf{IA}_i$ 和 $\mathbf{pA}_i$ 的影响）\
（1）$\mathbf{h}_i = \mathbf{IA}_i \mathbf{S}_i$ (投影的铰接体惯量)\
（2）$d_i = \mathbf{S}_i^T \mathbf{h}_i$ (关节 $i$ 处的标量“表观惯量”)\
（3）$u_i = \tau_i - \mathbf{S}_i^T \mathbf{pA}_i$ (扣除偏置力后，作用在关节 $i$ 上的“净”力矩)\
（4）$\mathbf{Ia}_i = \mathbf{IA}_i - \mathbf{h}_i \mathbf{h}_i^T / d_i$ (考虑了关节 $i$ 自由度后的“剩余”铰接体惯量)\
（5）$\mathbf{pa}_i = \mathbf{pA}_i + \mathbf{Ia}_i \mathbf{c}_i + \mathbf{h}_i u_i / d_i$ (对应的“剩余”铰接体偏置力)
2. 传播到父节点 $\lambda(i)$: 将计算出的“剩余”惯量和偏置力（$\mathbf{Ia}_i, \mathbf{pa}_i$）变换坐标系后，累加到父连杆 $\lambda(i)$ 上\
（1）$\mathbf{IA}_{\lambda(i)} = \mathbf{IA}_{\lambda(i)} + {^{\lambda(i)}\mathbf{X}_i^*} \mathbf{Ia}_i {^i\mathbf{X}_{\lambda(i)}}$\
（2）$\mathbf{pA}_{\lambda(i)} = \mathbf{pA}_{\lambda(i)} + {^{\lambda(i)}\mathbf{X}_i^*} \mathbf{pa}_i$
### 从根到叶（加速度）
#### 目标
从根节点开始向外递归地求解每个关节的加速度 $\ddot{q}_i$
#### 初始化
 $\mathbf{a}_0 = -\mathbf{a}_g$ (设置基座加速度以包含重力) 32
#### 递归
1. $\mathbf{a}'_i = {^i\mathbf{X}_{\lambda(i)}} \mathbf{a}_{\lambda(i)} + \mathbf{c}_i$ (计算连杆 $i$ 的偏置加速度，即如果 $\ddot{q}_i = 0$ 时它会有的加速度)
2. $\ddot{q}_i = (u_i - \mathbf{h}_i^T \mathbf{a}'_i) / d_i$ (求解 $\ddot{q}_i$。这是 ABA 的最终输出。注意 $u_i, \mathbf{h}_i, d_i$ 均在 Pass 2 中计算得到) 
3. 32$\mathbf{a}_i = \mathbf{a}'_i + \mathbf{S}_i \ddot{q}_i$ (计算连杆 $i$ 的最终空间加速度，并将其作为 $\mathbf{a}_{\lambda(i)}$ 传递给其子连杆)
## 比较
### 拓扑结构
1. $O(N^3)$ 和 $O(N^2)$ 的复杂度分析通常是针对串联链。对于分叉链（如人形机器人），$\mathbf{M}$ 矩阵会呈现出稀疏性。
2. 利用这种稀疏性， $O(N^3)$ 算法在分叉链上的运行速度可能远快于在同等 $N$ 值的串联链上，从而缩小了与 ABA 的性能差距。
### 实际的交叉点
1. 渐近复杂度描述的是 $N \to \infty$ 时的趋势。在实际工程中， $N$ 值通常较小。
2. 如前所述，对于 $N \le 18$ 的系统， $O(N^2)$ 的 CRBA 步骤可能比 $O(N^3)$ 的求解步骤更耗时。
3. 对于常见的 6-DOF（例如 UR5 33）或 7-DOF（例如 Kuka）工业臂， $N$ 值很小，ABA 的 $O(N)$ 优势并不明显，两种方法的绝对计算时间可能相差无几。
4. ABA 的 $O(N)$ 优势主要体现在高自由度系统上，例如人形机器人（$N > 30$）16 或蛇形机器人（$N=16$ 或更高）。
## 应用
### ABA
1. 物理引擎（如 Bullet , MuJoCo, Drake ）、计算机动画 、轨迹模拟
2. 只需要 $\ddot{\boldsymbol{q}}$ 的最终值，以便对状态 $(\boldsymbol{q}, \dot{\boldsymbol{q}})$ 进行数值积分。速度是最高优先级
### CRBA+RNEA
1. 高级控制算法，如“计算力矩控制”或“反馈线性化”
2. 这些控制律本身需要显式的 $\mathbf{M}(\boldsymbol{q})$ 和 $\mathbf{b}(\boldsymbol{q}, \dot{\boldsymbol{q}})$ 项