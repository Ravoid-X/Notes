## 协方差矩阵（covariance matrix）
度量两个变量之间线性关系强度和方向的统计量
### 协方差公式
$$\text{Cov}(X,Y) = \frac{\sum_{i=1}^{n}(x_i - \bar{x})(y_i - \bar{y})}{n-1}$$
1. $\text{Cov}(X,Y)$ 接近 0，表示两个变量之间没有明显的线性关系。
2. 符号为正，表示两个变量呈正相关。
### 协方差矩阵
当数据有多个变量时，为了一次性表示所有变量两两协方差，引入协方差矩阵
$$\Sigma = \begin{bmatrix}
Cov(X_1, X_1) & Cov(X_1, X_2) & \cdots & Cov(X_1, X_p) \\
Cov(X_2, X_1) & Cov(X_2, X_2) & \cdots & Cov(X_2, X_p) \\
\vdots & \vdots & \ddots & \vdots \\
Cov(X_p, X_1) & Cov(X_p, X_2) & \cdots & Cov(X_p, X_p)
\end{bmatrix}$$