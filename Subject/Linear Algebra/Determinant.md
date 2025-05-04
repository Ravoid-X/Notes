## 性质 （pending）
定义：不同行不同列元素乘积代数和 
1. 行列互换（转置），行列式的值不变
2. 两行（列）互换，行列式变号
3. 行列式的某一行（列）的所有元素都乘以同一数 k，等于用数k乘此行列式
4. 行列式中如果某一行（列）的元素都是两组数之和，那么这个行列式就等于两个行列式之和
5. 一行（列）乘k加到另一行（列），行列式的值不变
6. 两行成比例，行列式的值为0
## 行列式计算
1. 二阶\
<img src="../../Pic/Subject/Linear Algebra/determinant-2d-calcu1.png" style="width:250px;padding:10px;"/> 
<img src="../../Pic/Subject/Linear Algebra/determinant-2d-calcu2.png" style="width:600px;padding:10px;"/> 

2. 三阶\
<img src="../../Pic/Subject/Linear Algebra/determinant-3d-calcu.png" style="width:600px;padding:10px;"/>

3. 逆序数：一个排列中，逆序对的数目 \
   倒序数：将某个正整数各数字逆序排列
4. n阶（公式法）\
<img src="../../Pic/Subject/Linear Algebra/determinant-nd-calcu1.png" style="width:600px;padding:10px;"/>
<img src="../../Pic/Subject/Linear Algebra/determinant-nd-calcu2.png" style="width:600px;padding:10px;"/>

5. n阶（代数余子式）\
<img src="../../Pic/Subject/Linear Algebra/determinant-nd-calcu3.png" style="width:600px;padding:10px;"/>\
（2，2）代数余子式如下所示
<img src="../../Pic/Subject/Linear Algebra/determinant-nd-calcu4.png" style="width:500px;padding:10px;"/>\
矩阵的行列式可以写成\
<img src="../../Pic/Subject/Linear Algebra/determinant-nd-calcu5.png" style="width:300px;padding:10px;"/>


## 特殊行列式
1. 主对角线行列式的值等于主对角线元素的乘积；副对角线行列式的值等于副对角线元素的乘积乘 ${(-1)}^{\frac {n(n-1)} {2}}$
2. 箭型行列式\
<img src="../../Pic/Subject/Linear Algebra/determinant-arrow-calcu1.png" style="width:400px;padding:10px;"/>\
用第一列加上第二列的$-\frac {c} {{a}_{2}}$，第三列的$-\frac {c} {{a}_{3}}$，···\
<img src="../../Pic/Subject/Linear Algebra/determinant-arrow-calcu2.png" style="width:600px;padding:10px;"/>

3. 范德蒙行列\
<img src="../../Pic/Subject/Linear Algebra/determinant-vandermonde-calcu1.png" style="width:600px;padding:10px;"/>
<img src="../../Pic/Subject/Linear Algebra/determinant-vandermonde-calcu2.png" style="width:400px;padding:10px;"/>

4. 两条线型行列式
   
5. 两三角型行列式（b=c）
   
6. 两三角型行列式（b≠c）
   
7. 三对角型行列式
## 几何意义
<img src="../../Pic/Subject/Linear Algebra/determinant-geometric-mean1.png" style="width:300px;padding:10px;"/>
<img src="../../Pic/Subject/Linear Algebra/determinant-geometric-mean2.png" style="width:400px;padding:10px;"/>