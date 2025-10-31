## 概述（pending）
为了解决万向锁问题而生的一种更高级、更稳健的数学工具。是现代 3D 空间中表示姿态和旋转的最优越的数学工具。
## 万向锁


## 数学定义
是对复数 ($a + bi$) 的扩展，有 4 个部分（1 个实部 $w$，3 个虚部 $x\mathbf{i}, y\mathbf{j}, z\mathbf{k}$），可以完美描述 3D 旋转。
### 基本形式
一个四元数 $\mathbf{q}$ 可以表示为
$$\mathbf{q} = w + x\mathbf{i} + y\mathbf{j} + z\mathbf{k}$$
$w$: 实部。$(x, y, z)$: 虚部，可以看作一个 3D 向量 $\mathbf{v}$。$\mathbf{i}, \mathbf{j}, \mathbf{k}$: 虚数单位\
所以，$\mathbf{q}$ 也可以写成 $(w, \mathbf{v})$ 的形式
## 核心规则 (哈密顿法则)
### 乘法规则
$$\mathbf{i}^2 = \mathbf{j}^2 = \mathbf{k}^2 = \mathbf{i}\mathbf{j}\mathbf{k} = -1$$
可以推导出
$$\mathbf{ij} = \mathbf{k}$$
$$\mathbf{jk} = \mathbf{i}$$
$$\mathbf{ki} = \mathbf{j}$$
### 关键特性
不满足乘法交换律，3D 空间中的旋转本身就是不满足交换律的
$$\mathbf{ji} = -\mathbf{k}$$
$$\mathbf{kj} = -\mathbf{i}$$
$$\mathbf{ik} = -\mathbf{j}$$
## 表示 3D 旋转
一个单位四元数（模长为 1）可以表示一个 3D 空间中的旋转
### 轴-角表示法
1. 一个 3D 旋转可以被定义为：绕一个单位向量轴 $\mathbf{u} = (u_x, u_y, u_z)$ 旋转 $\theta$ 角度
2. 一个表示此旋转的四元数 $\mathbf{q}$ 可以被构造为
$$\mathbf{q} = (\cos(\frac{\theta}{2}), \sin(\frac{\theta}{2})\mathbf{u})$$
3. 展开分量
$$w = \cos(\frac{\theta}{2})$$
$$x = u_x \cdot \sin(\frac{\theta}{2})$$
$$y = u_y \cdot \sin(\frac{\theta}{2})$$
$$z = u_z \cdot \sin(\frac{\theta}{2})$$
### $\theta/2$ 解释
在四元数“双重覆盖”的数学特性中，旋转 $\mathbf{p}' = \mathbf{q} \mathbf{p} \mathbf{q}^{-1}$ 这个“三明治”乘积公式会导致旋转被应用了两次，因此需要预先除以 2 来抵消这个效应
### 示例

## 核心运算
### 旋转向量
有一个四元数 $\mathbf{q}$，还有一个 3D 向量 $\mathbf{p} = (p_x, p_y, p_z)$。想知道 $\mathbf{p}$ 被 $\mathbf{q}$ 旋转后得到的新点 $\mathbf{p}'$ 在哪里。
1. “提升”向量：将 $\mathbf{p}$ 转换成一个四元数 $\mathbf{p}_{quat}$
$$\mathbf{p}_{quat} = 0 + p_x\mathbf{i} + p_y\mathbf{j} + p_z\mathbf{k}$$
2. 共轭/逆： 计算 $\mathbf{q}$ 的共轭 $\mathbf{q}^*$
$$\mathbf{q} = w + x\mathbf{i} + y\mathbf{j} + z\mathbf{k}$$
$$\mathbf{q}^* = w - x\mathbf{i} - y\mathbf{j} - z\mathbf{k}$$
3. 重要特性：对于单位四元数，它的逆 $\mathbf{q}^{-1}$ 就等于它的共轭 $\mathbf{q}^*$。使得计算极大简化
4. “三明治”乘积：就是施加旋转的公式
$$\mathbf{p}'_{quat} = \mathbf{q} \cdot \mathbf{p}_{quat} \cdot \mathbf{q}^{-1}$$
5. $\mathbf{p}'_{quat}$ 将会是一个新的四元数，实部为 0，虚部 $(p'_x, p'_y, p'_z)$ 就是旋转后的新向量 $\mathbf{p}'$
### 组合旋转 (链式旋转)
1. 举例旋转 $\mathbf{q}_1$ (例如：绕 Z 轴转 30 度)，然后旋转 $\mathbf{q}_2$ (例如：绕 X 轴转 45 度)
2. 组合后的总旋转 $\mathbf{q}_{total}$ 是：
$$\mathbf{q}_{total} = \mathbf{q}_2 \cdot \mathbf{q}_1$$
3. 和矩阵一样，是右乘，需要一次四元数乘法，计算量远小于 $3 \times 3$ 矩阵乘法
### 平滑插值 (SLERP)
1. 问题：假设知道姿态 A (由 $\mathbf{q}_A$ 表示) 和姿态 B (由 $\mathbf{q}_B$ 表示)，想在这两个姿态之间平滑地插值
2. 欧拉角插值： 灾难。如果对 (Yaw, Pitch, Roll) 三个值分别进行线性插值，由于万向锁的存在，物体会以非预期的、不均匀的、诡异的路径（例如绕远路）进行旋转。
3. 四元数插值 (SLERP)：SLERP (Spherical Linear Interpolation，球面线性插值) 是可以在 4D 超球面上找到从 $\mathbf{q}_A$ 到 $\mathbf{q}_B$ 的最短路径