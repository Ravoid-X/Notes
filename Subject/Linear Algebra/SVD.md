## 定义
奇异值分解(Singular Value Decomposition)为特征分解的扩展。不要求分解的矩阵为方阵。假设 $A$ 为 $m\times n$ 的矩阵，定义 $A$ 的SVD为：\
&emsp;&emsp;&emsp;&emsp;$A=U\sum {V}^{T}$\
其中 $U$ 为 $m\times m$ 的矩阵，$\sum$ 为 $m\times n$ 的矩阵，除了主对角线上的元素（称为奇异值）以外全为0，$U$ 为 $n\times n$ 的矩阵。
## 计算
1. V\
（1）${A}^{T}A$ 是方阵，对其进行特征值分解\
（2）特征向量（一般称为右奇异向量）组成的 $n\times n$ 矩阵即为V
2. U\
（1）$A{A}^{T}$ 是方阵，对其进行特征值分解\
（2）特征向量（一般称为左奇异向量）组成的 $m\times m$ 矩阵即为V
3. 奇异值
<img src="../../Pic/Subject/Linear Algebra/svd-sum-matrix.png" style="width:600px;padding:10px;"/>

4. 第二种求奇异值方式\
特征值矩阵等于奇异值矩阵的平方，通过求出 ${A}^{T}A$ 的特征值取平方根来求奇异值
## 实例
<img src="../../Pic/Subject/Linear Algebra/svd-example.png" style="width:600px;padding:10px;"/>