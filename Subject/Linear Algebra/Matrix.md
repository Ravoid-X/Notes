## 定义 （pending）
<img src="../../Pic/Subject/Linear Algebra/matrix-definition.png" style="width:250px;padding:10px;"/> 

## 类型
1. 实/复矩阵：储存元素是实数还是负数。
2. 零矩阵：所有元素为零，记作 $O$
3. 方阵：行数与列数相等的矩阵。
4. 行矩阵：只有一行的矩阵，也称为行向量。
5. 对角矩阵：除对角线外的元素都为0的方阵。
6. 单位矩阵：对角线上元素都为1的对角矩阵，用 $E$ 或 $I$ 表示。
7. 同型矩阵：两个矩阵行列相等。
8. 行阶梯型：可画出一条阶梯线，线的下方全为0

## 基本运算
1. 线性运算：必须是同型矩阵才能进行，逐元素操作\
（1）设 $A={({a}_{ij})}_{m\times n}$，$B={({b}_{ij})}_{m\times n}$，则有 $αA+βB={({αa}_{ij}+{βb}_{ij})}_{m\times n}$\
（2）交换律：A + B = B + A\
（3）结合律：（ A + B ）+ C = A + ( B + C )\
（4）α ( A + B ) = α A + α B
2. 乘法\
（1）设 $A={({a}_{ij})}_{m\times s}$，$B={({b}_{ij})}_{s\times n}$，则 $C={({c}_{ij})}_{m\times n}$，其中\
<img src="../../Pic/Subject/Linear Algebra/matrix-multi1.png" style="width:250px;padding:10px;"/> \
（2）结合律：( c A ) B = c ( A B ) = A ( c B )\
( A B ) C = A ( B C )\
（3）无交换律：A B 不一定等于 B A\
（4）无消去律：A B = 0 不一定 A = 0 或 B = 0\
&empA ≠ 0 时，A B = A C 不一定 B = C\
（5）分块对角矩阵\
<img src="../../Pic/Subject/Linear Algebra/matrix-multi2.png" style="width:250px;padding:10px;"/>\
（6）方阵幂性质 \
<img src="../../Pic/Subject/Linear Algebra/matrix-multi3.png" style="width:250px;padding:10px;"/>

## 转置矩阵
把矩阵 $A$ 的行换成同序数的列得到的新矩阵，叫做 $A$ 的转置矩阵，记作 ${A}^{T}$
1. 对称矩阵：如果 $A$ 为方阵 而且 ${A}^{T}=A$，则称 $A$ 为对称矩阵。
2. 反对称矩阵：如果 $A$ 为方阵 而且 $A=-{A}^{T}$，则称 $A$ 为反对称矩阵。\
（1）$A$ 的主对角线上元素全为0\
（2）若A是奇数阶，则 | $A$ | = 0
3. 分块矩阵\
<img src="../../Pic/Subject/Linear Algebra/matrix-transpose.png" style="width:250px;padding:10px;"/>

4. 性质\
（1）${({A}^{T})}^{T}=A$\
（2）${(AB)}^{T}={B}^{T}{A}^{T}$\
（3）$\left|{{A}^{T}}\right|=\left|{A}\right|$\
（4）${({A}^{-1})}^{T}={({A}^{T})}^{-1}$\
（5）${({A}^{T})}^{*}={({A}^{*})}^{T}$

## 伴随矩阵
1. 定义：行列式 $\left|{A}\right|$ 的各个元素的代数余子式 ${A}_{ij}$ 构成的矩阵成为 $A$ 的伴随矩阵\
<img src="../../Pic/Subject/Linear Algebra/matrix-adjoint1.png" style="width:400px;padding:10px;"/>\
${A}_{ij}$ 为 $\left|{A}\right|$ 中个元素的代数余子式

2. 性质\
（1）${A}{A}^{*}={A}^{*}{A}={\left|{A}\right|}{E}$\
（2）${(AB)}^{*}={B}^{*}{A}^{*}={\left|{A}\right|}^{n-2}{A}$，( n ≥ 3 )\
（3）$\left|{{A}^{*}}\right|={\left|{A}\right|}^{n-1}$\
（4）${(cA)}^{*}={c}^{n-1}{A}^{*}$\
（5）${({A}^{k})}^{*}={({A}^{*})}^{k}$\
（6）\
<img src="../../Pic/Subject/Linear Algebra/matrix-adjoint2.png" style="width:400px;padding:10px;"/>
3. 分块对角阵\
<img src="../../Pic/Subject/Linear Algebra/matrix-adjoint3.png" style="width:400px;padding:10px;"/>

## 逆矩阵
1. 定义\
对于方阵 $A$，若存在 $B$，使得 ${AB}={BA}={E}$，则称 $A$ 是可逆的，并称 $B$ 为 $A$ 的逆矩阵，记作 ${A}^{-1}$
2. 当 $\left|{A}\right|=0$ 时，称其为奇异矩阵，否则为非奇异矩阵，如果 $A$ 可逆，则$\left|{A}\right|≠0$，即可逆矩阵为非奇异矩阵。
3. 性质\
（1）${({A}^{-1})}^{-1}={A}$\
（2）${(kA)}^{-1}=\frac{1}{k}{A}^{-1}$，（ k ≠ 0 ）\
（3）${(AB)}^{-1}={B}^{-1}{A}^{-1}$\
（4）${A}^{*}=\left|{A}\right|{A}^{-1}$\
（5）${({A}^{*})}^{-1}={({A}^{-1})}^{*}$\
（6）$\left|{{A}^{-1}}\right|={\left|{A}\right|}^{-1}$\
（7）${A}^{0}={E}$，${({A}^{-1})}^{k}={({A}^{k})}^{-1}={A}^{-k}$\
（8）左消去律：A B = A C 可得 B = C\
&emsp;&emsp;右消去律：B A = C A 可得 B = C

## 正定矩阵
1. 给定一个大小为 $n\times n$ 的实对称矩阵 A ，若对于任意长度为 n 的非零向量 ${x}$ ，有 ${x}^TA{x}>0$ 恒成立，则矩阵 A 是一个正定矩阵。
2. 给定一个大小为 $n\times n$ 的实对称矩阵 A ，若对于任意长度为 n 的非零向量 ${x}$ ，有 ${x}^TA{x}≥0$ 恒成立，则矩阵 A 是一个半正定矩阵。

## 正交矩阵
若 $A$ 满足 ${A}{A}^{T}={A}^{T}{A}={E}$ 或 ${A}^{T}={A}^{-1}$，则称其为正交矩阵

## 相似矩阵
如果$B={M}^{-1}AM$, 则 $A$ 与 $B$ 相似

## 相关概念
### 迹（trace）
矩阵对角元的和。因此只有方阵才有迹\
<img src="../../Pic/Subject/Linear Algebra/matrix-trace.png" style="width:300px;padding:10px;"/>

### 秩（rank）
列秩是 $A$ 的线性无关的纵列的极大数目。类似地，行秩是 A 的线性无关的横行的极大数目。列秩和行秩总是相等的，通常表示为 $r(A)$
### 范数
1. 1-范数：矩阵的列向量的和的最大值\
<img src="../../Pic/Subject/Linear Algebra/matrix-norm1.png" style="width:400px;padding:10px;"/>

2. 2-范数：${A}^{T}A$ 矩阵的最大特征值的开平方\
<img src="../../Pic/Subject/Linear Algebra/matrix-norm2.png" style="width:400px;padding:10px;"/>

### 矩阵多项式
<img src="../../Pic/Subject/Linear Algebra/matrix-polynomial.png" style="width:400px;padding:10px;"/>
<img src="../../Pic/Subject/Linear Algebra/matrix-polynomial2.png" style="width:400px;padding:10px;"/>