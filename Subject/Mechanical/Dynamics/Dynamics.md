## 概述
1. 研究产生运动的物理原因，可分为正动力学和逆动力学
2. 最经典建立机器人数学模型的方法有两种：拉格朗日法和牛顿-欧拉法
## 正动力学
1. 问题： 已知施加在机器人各关节上的力或力矩 $\tau$，以及机器人当前的状态（位置 $q$ 和速度 $\dot{q}$），求机器人将产生的加速度 $\ddot{q}$ 是多少
2. 主要应用是仿真，这是构建机器人模拟器（如 Gazebo, MuJoCo）的核心。
3. 通过不断积分（$\ddot{q} \rightarrow \dot{q} \rightarrow q$），可以预测机器人在给定的力矩输入下将如何运动
4. 实时仿真一个复杂机器人通常比实时控制它在计算上更具挑战性。
## 逆动力学
1. 问题：已知机器人的期望运动轨迹（每个关节的位置 $q(t)$、速度 $\dot{q}(t)$ 和加速度 $\ddot{q}(t)$），求需要施加在每个关节上的力或力矩 $\tau$ 是多少
2. 最常见的应用是控制，控制器需要实时计算出下一刻需要多大的电机输出（力矩）
## 空间向量代数
### 空间速度
结合了角速度 $\omega$ 和线速度 $v_O$（在原点 $O$ 处的速度）
$$v = \begin{bmatrix} \omega \\ v_O \end{bmatrix} \in \mathbb{R}^6$$
### 空间力
结合了力矩 $n$ 和线性力 $f_O$
$$f = \begin{bmatrix} n \\ f_O \end{bmatrix} \in \mathbb{R}^6$$
### 空间惯性
一个 $6 \times 6$ 矩阵 $\mathbf{I}$，统一了转动惯量、质量和质心位置（一阶矩）
$$\mathbf{I} = \begin{bmatrix} \bar{I}_C + m(c \times)(c \times)^T & m(c \times) \\ m(c \times)^T & m\mathbf{1} \end{bmatrix} \in \mathbb{R}^{6 \times 6}$$
# 拉格朗日公式
通过系统的能量而非直接作用的力来描述运动，提供了一种高度系统化且与坐标系选择无关的建模方法。
## 最小作用量原理
1. 一个物理系统从一个状态到另一个状态的实际运动路径，是使其作用量泛函 $S$ 取得驻值（通常是最小值）的那条路径
2. 系统的全部动态特性被封装在一个称为拉格朗日函数 $L$ 的标量函数中
## 拉格朗日函数 
1. 核心是拉格朗日函数 $L$，定义为系统的总动能 $T$ 与总势能 $U$之差
$$L(q, \dot{q}) = T(q, \dot{q}) - V(q)$$
1. 系统的动能 $T$ 是广义坐标 $q$ 及其导数 $\dot{q}$（广义速度）的函数
2. 势能 $U$（例如重力势能）通常只与广义坐标 $q$ 有关。
## 建模要素
### 广义坐标 $q$
1. 对于 $n$ 自由度的机械臂，这组广义坐标通常就是 $n$ 个关节的角度或位移
2. 好处是自动消除了所有无功约束力，例如，连接连杆 1 和连杆 2 的铰链内部存在复杂的约束力，牛顿法必须显式地处理这些力。
3. 拉格朗日法通过选择关节角度 $q_i$ 作为坐标，使得这些内部约束力在推导过程中自然消失。
### 系统总动能 $T(q, \dot{q})$
1. 机械臂中所有连杆动能的总和，第 $i$ 个连杆的动能 $T_i$ 包括其质心的平动动能和绕质心的转动动能。
2. 连杆 $i$ 的质心线速度 $v_i$ 和角速度 $\omega_i$ 都是关节速度 $\dot{q}$ 的线性函数（通过雅可比矩阵 $J_{v_i}(q)$ 和 $J_{\omega_i}(q)$ 映射） 
3. 总动能 $T = \sum T_i$ 最终总可以表示为关于广义速度 $\dot{q}$ 的一个二次型
$$ T(q, \dot{q}) = \frac{1}{2} \sum_{i=1}^n \sum_{j=1}^n M_{ij}(q) \dot{q}_i \dot{q}_j = \frac{1}{2} \dot{q}^T \mathbf{M}(q) \dot{q} $$
1. $\mathbf{M}(q)$ 是一个 $n \times n$ 的对称矩阵，被称为系统的惯性矩阵 或质量矩阵，取决于机械臂的当前构型。
### 系统总势能 $V(q)$
1. 势能的主要来源是重力势能，总势能 $V(q)$ 是所有连杆重力势能的总和
$$V(q) = \sum_{i=1}^n V_i(q) = - \sum_{i=1}^n m_i \mathbf{g}^T \mathbf{r}_{c_i}(q)$$
1. $m_i$ 是连杆 $i$ 的质量，$\mathbf{g}$ 是惯性系中的重力加速度向量（例如 $[0, 0, -9.81]^T$）
2. $\mathbf{r}_{c_i}(q)$ 是连杆 $i$ 的质心在惯性系中的位置向量。由于 $\mathbf{r}_{c_i}$ 仅是构型 $q$ 的函数，$V(q)$ 也仅是 $q$ 的函数
## 欧拉-拉格朗日方程
### E-L 方程
1. 根据哈密顿的最小作用量原理，系统的运动轨迹是使作用量积分 $S = \int L dt$ 取极小值的路径。
2. 这一原理最终导出了欧拉-拉格朗日方程，它是描述系统动力学的核心方程：
$$\frac{d}{dt} \left( \frac{\partial L}{\partial \dot{q}_i} \right) - \frac{\partial L}{\partial q_i} = \tau_i$$
$q_i$ 是第 $i$ 个广义坐标，$\dot{q}_i$ 是对应的广义速度，而 $τ_i$ 是作用在第 $i$ 个广义坐标上的广义力（对于旋转关节，是力矩；对于移动关节，是力）
### 计算代入
将 $L = T(q, \dot{q}) - V(q)$ 代入 E-L 方程，并利用 $T = \frac{1}{2} \dot{q}^T \mathbf{M}(q) \dot{q}$ 和 $V = V(q)$
1. 计算第一项 $\frac{d}{dt} \left( \frac{\partial L}{\partial \dot{q}} \right)$
$$ \frac{d}{dt} \left( \frac{\partial L}{\partial \dot{q}} \right) = \frac{d}{dt} (\mathbf{M}(q) \dot{q}) = \dot{\mathbf{M}}(q, \dot{q}) \dot{q} + \mathbf{M}(q) \ddot{q} $$
1. 计算第二项 $\frac{\partial L}{\partial q}$
$$\frac{\partial L}{\partial q} = \frac{\partial T}{\partial q} - \frac{\partial V}{\partial q}$$
$$\frac{\partial L}{\partial q} = \frac{1}{2} \dot{q}^T \frac{\partial \mathbf{M}(q)}{\partial q_k} \dot{q}- \frac{\partial V}{\partial q_k}$$
1. 代入 E-L 方程 $\tau = \frac{d}{dt}(\frac{\partial L}{\partial \dot{q}}) - \frac{\partial L}{\partial q}$
$$\tau = (\dot{\mathbf{M}} \dot{q} + \mathbf{M} \ddot{q}) - \left( \frac{\partial T}{\partial q} - \frac{\partial V}{\partial q} \right) $$ 
$$ \tau = \mathbf{M}(q) \ddot{q} + \left( \dot{\mathbf{M}}(q, \dot{q}) \dot{q} - \frac{\partial T(q, \dot{q})}{\partial q} \right) + \frac{\partial V(q)}{\partial q}$$
### 最终形式
$$\mathbf{M}(q) \ddot{q} + \mathbf{C}(q, \dot{q}) \dot{q} + \mathbf{G}(q) = \tau$$
## 动力学方程剖析
### 惯性矩阵 $\mathbf{M}(q)$
建立了关节加速度 $\ddot{q}$ 与关节力矩 $\tau$ 之间的关系，具有三个核心性质
1. 对称性: $\mathbf{M}(q) = \mathbf{M}^T(q)$
2. 正定性: 对于任意非零向量 $x$，都有 $x^T \mathbf{M}(q) x > 0$。这源于其与动能的直接关系（动能 $T = \frac{1}{2} \dot{q}^T \mathbf{M} \dot{q}$ 必须始终为正）
3. 构型相关性: $\mathbf{M}(q)$ 矩阵的元素值取决于当前的关节构型 $q$。这是机器人动力学非线性的主要来源之一。物理上，这意味着机器人在不同姿态下的惯性是不同的。
### 重力向量 $\mathbf{G}(q)$
代表在构型 $q$ 下，为了克服重力影响所需要施加的静态关节力矩
$$ \mathbf{G}(q) = \nabla_q V(q) = \left[ \frac{\partial V}{\partial q_1}, \dots, \frac{\partial V}{\partial q_n} \right]^T $$
### 科里奥利与离心力矩阵 $\mathbf{C}(q, \dot{q})$
1. 捕获了所有与速度相关的非线性力，包括离心力（由 $\dot{q}_i^2$ 产生）和科里奥利力（由 $\dot{q}_i \dot{q}_j, i \neq j$ 产生）
2. $\mathbf{C}$ 矩阵的定义并非唯一，因为满足 $\mathbf{C}(q, \dot{q}) \dot{q} = \dot{\mathbf{M}}\dot{q} - \frac{\partial T}{\partial q}$ 的 $\mathbf{C}$ 可以有多种形式
3. 不过存在一个标准的、在控制上具有特殊意义的定义，它通过第一类克里斯托费尔符号$\Gamma_{ijk}$ 来构建
4. $\mathbf{C}$ 矩阵的第 $(i, j)$ 个元素 $C_{ij}$ 可以定义为
$$C_{ij}(q, \dot{q}) = \sum_{k=1}^n \Gamma_{ijk}(q) \dot{q}_k$$
1. $\Gamma_{ijk}(q)$ 完全由惯性矩阵 $\mathbf{M}(q)$ 及其对 $q$ 的偏导数决定
$$ \Gamma_{ijk}(q) = \frac{1}{2} \left( \frac{\partial M_{ij}}{\partial q_k} + \frac{\partial M_{ik}}{\partial q_j} - \frac{\partial M_{jk}}{\partial q_i} \right) $$
1. 揭示了 $\mathbf{C}$ 矩阵并非一个独立的物理量，它只是 $\mathbf{M}$ 矩阵随构型 $q$ 变化时所产生的速度效应的数学表达
### $\dot{M} - 2C$ 的斜对称性
1. 如果 $\mathbf{C}$ 矩阵是使用上述克里斯托费尔符号定义的，那么矩阵 $\mathbf{N}(q, \dot{q}) = \dot{\mathbf{M}}(q, \dot{q}) - 2\mathbf{C}(q, \dot{q})$ 必定是一个斜对称矩阵
2. 斜对称意味着 $\mathbf{N}^T = - \mathbf{N}$，或者等价地，对于任意向量 $x$，都有 $x^T \mathbf{N} x = 0$
3. 物理意义：证明了科里奥利力和离心力是无功力，虽然会极大地影响系统的动态行为，但它们本身既不产生能量，也不消耗能量，只在系统的不同自由度之间传递能量。
## 计算分析
1. 实践中面临着维度灾难，当系统自由度增加到 6 或更高时，所需进行的符号微分和代数化简的计算量会急剧膨胀，变得不切实际且极易出错。
2. 理解动力学结构的绝佳工具，但本身并不直接导出一个高效的计算算法。，于是引出了更适合计算机实现的数值方法，牛顿-欧拉法。

# 牛顿-欧拉公式
将牛顿第二定律和欧拉方程分别应用于多体链中的每一个连杆，并通过递归计算来系统地求解整个系统的动力学。
## 单个刚体动力学
### 牛顿第二定律（平移运动）
$$\sum F = m a_c$$
### 欧拉方程（旋转运动）
$$\sum \tau_c = I_c \dot{\omega} + \omega \times (I_c \omega)$$
$I_c$ 是刚体关于其重心的惯性张量，$\omega \times (I_c \omega)$ 这是陀螺力矩，源于旋转坐标系的变化
## 开链结构
### 计算问题
1. 直接应用N-E 法会导致一个庞大且复杂的方程组，因为会引入大量并不关心的连杆间约束力/力矩。
2. 为了得到最终的 $n$ 个关节力矩 $\tau$，必须通过繁琐的代数运算来消除所有这些 $n-1$ 组内部约束力。
### 递归牛顿-欧拉算法
1. 通过两次系统性的递归传递，避免了显式求解那些不需要的约束力，从而在 $O(N)$ 的线性时间内完成了计算
2. 是求解逆向动力学的标准算法，由两次传递组成
3. 该算法的输入是期望的运动（$q, \dot{q}, \ddot{q}$）和末端施加的外力/力矩 $F_{tip}$，输出是所需的关节力矩向量 $\tau$
## 正向递归
此过程从基座（连杆 0）开始，逐个连杆向末端执行器（连杆 n）传播
### 目标
计算每个连杆 $i$ 的速度和加速度（通常在各自的连杆坐标系 {i} 中表示）
### 初始化
1. 基座 $i=0$ 的速度为零：$V_0 = 0$
2. 基座 $i=0$ 的加速度：$\dot{V}_0 = -g$
3. RNEA 可以在后续计算中自动地、无缝地将重力项包含在内，而无需单独处理势能
### 递归步骤
对于 $i=1$ 到 $n$，计算连杆 $i$ 的速度与加速度
1. 旋量 ($V_i$): 连杆 $i$ 的速度 $V_i$（一个 6D 向量，包含线速度和角速度）等于从连杆 $i-1$ 传递过来的速度（通过坐标变换），再加上关节 $i$ 自身运动 $\dot{q}_i$ 产生的相对速度
$$V_i = \text{Ad}_{T_{i,i-1}} (V_{i-1}) + \mathcal{A}_i \dot{q}_i$$
$\text{Ad}$ 是 6x6 伴随变换矩阵，$\mathcal{A}_i$ 是关节 $i$ 的螺旋轴

2. 加速度 ($\dot{V}_i$): 连杆 $i$ 的加速度 $\dot{V}_i$（一个 6D 向量）由三部分组成\
（1）从 $i-1$ 传递的加速度: $\text{Ad}_{T_{i,i-1}} (\dot{V}_{i-1})$\
（2）关节 $i$ 的角加速度引起的加速度: $\mathcal{A}_i \ddot{q}_i$\
（3）速度乘积项 (Velocity-Product Term): $\text{ad}_{V_i}(\mathcal{A}_i) \dot{q}_i$
$$ \dot{V}i = \text{Ad}{T_{i,i-1}} (\dot{V}{i-1}) + \text{ad}{V_i}(\mathcal{A}_i) \dot{q}_i + \mathcal{A}_i \ddot{q}_i $$
## 反向递归
此过程从作用在末端执行器上的已知外力（或称扳手，Wrench）开始，逐个连杆向基座传播
### 目标
计算从连杆 $i+1$ 传递到连杆 $i$ 的力/力矩，并最终求解出关节 $i$ 上的力 $\tau_i$ 
### 初始化
1. 末端连杆 $n$ 受到来自环境的外力/力矩 $F_{tip}$（例如，抓取物体）
2. 连杆 $n$ 上的总扳手 $F_n = F_{tip}$
### 递归步骤
对于 $i=n$ 到 $1$，计算扳手 $F_i$ 和关节力矩 $\tau_i$
1. 扳手 ($F_i$): 连杆 $i$ 上感受到的总扳手 $F_i$（一个 6D 向量）等于两部分之和\
（1）它施加给连杆 $i+1$ 的扳手 $F_{i+1}$ 的反作用力（通过坐标变换）\
（2）使连杆 $i$ 自身 产生运动（由 $V_i, \dot{V}_i$ 描述）所需的惯性力/力矩（根据刚体动力学 $F=ma$ 计算得出 $\mathcal{F}_i$）
$$F_i = \text{Ad}^T_{T_{i+1,i}} (F_{i+1}) + \mathcal{F}_i(V_i, \dot{V}_i)$$
2. 关节力矩 ($\tau_i$)：关节 $i$ 的致动器不需要提供全部的 6D $F_i$，只需要提供 $F_i$ 在关节运动轴 $\mathcal{A}_i$ 上的投影
$$\tau_i = F_i^T \mathcal{A}_i$$
3. $F_i$ 中剩余的 5 个分量被关节的物理结构（如轴承、销钉）所约束或支撑了，其效果与拉格朗日法中 自动消除无功约束力 的效果完全等价