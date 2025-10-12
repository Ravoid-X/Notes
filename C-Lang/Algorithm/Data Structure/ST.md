## ST ( Sparse Table ) 表
专门用于解决 RMQ (Range Minimum/Maximum Query) 区域最值查询问题。
1. 定义 $st[i][j]$：表示从数组下标 i 开始，长度为 $2 ^j$ 的区间内的最值。
2. 查询\
（1）根据 $2^k \le len < 2^{k+1}$，求得 k\
（2）区间 [L, R] 可以被 $[L,L+2^k-1]$ 和 $[R-2^k+1,R]$ 完全覆盖\
（3）$query(L, R) = min(st[L][k], st[R - (1 << k) + 1][k])$
### 程序
```
void build_st() {
    // Base Case: j = 0, 区间长度为 1
    for (int i = 0; i < n; ++i) {
        st[i][0] = arr[i];
    }
    // 递推
    for (int j = 1; (1 << j) <= n; ++j) {
        for (int i = 0; i + (1 << j) <= n; ++i) {
            st[i][j] = min(st[i][j - 1], st[i + (1 << (j - 1))][j - 1]);
        }
    }
}
int query(int l, int r) {
    int k = log2(r - l + 1);
    return min(st[l][k], st[r - (1 << k) + 1][k]);
}
```