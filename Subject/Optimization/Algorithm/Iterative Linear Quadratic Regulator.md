## 概述（pending）
$$J(x) \approx J(x_k) + \nabla J(x_k)^T (x - x_k) + \frac{1}{2} (x - x_k)^T H(x_k) (x - x_k)$$
$$\nabla J(x_k) + H(x_k) (x_{k+1} - x_k) = 0$$
$$
\underbrace{H(x_k)}{\text{Hessian}} \underbrace{(x{k+1} - x_k)}{\text{搜索步长 } p_k} = - \underbrace{\nabla J(x_k)}{\text{梯度 } g_k} $$
$$x_{k+1} = x_k + p_k = x_k - H_k^{-1} g_k$$