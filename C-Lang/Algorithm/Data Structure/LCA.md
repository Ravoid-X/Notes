## LCA (最近公共祖先)
在一棵树中，对于任意两个节点 u 和 v，找到它们深度最深的共同祖先
1. 定义 $up[u][j]$：表示节点 u 的第 $2 ^j$ 个祖先
2. 预处理：遍历得到节点的深度和父节点，递推 $up[u][j]=up[up[u][j-1]][j-1]$
3. 查询\
（1）保证节点在同意深度，利用倍增加速跳跃\
（2）同步跳跃 u 和 v,从 j = logN 开始向下循环。如果 $up[u][j]$ 不等于 $up[v][j]$，说明跳了 $2 ^j$ 步后 LCA 还在更上方。可以安全地跳这一步：$u=up[u][j]，v=up[v][j]$\
（3）如果等于，说明已经跳到了 LCA 或其祖先，所以这一步不能跳。\
（4）循环结束后，u 和 v 就在 LCA 的正下方，返回 up[u][0]即可。
### 程序
```
void dfs(int u, int p, int d) {
    depth[u] = d;
    up[u][0] = p;
    for (int v : adj[u]) {
        if (v != p) dfs(v, u, d + 1);
    }
}
void build_lca() {
    // 假设根节点是 1，没有父节点，深度为 0
    dfs(1, 0, 0); 
    // 根节点的父节点设为0或自己，避免越界
    up[1][0] = 1;
    if (n>0 && up[0][0] == 0) up[0][0] = 0; // 虚拟节点0的父节点是自己
    for (int j = 1; j < LOGN; ++j) {
        for (int i=1; i<=n;++i) up[i][j] = up[up[i][j-1]][j-1];
    }
}
int lca(int u, int v) {
    // 1. 拉平深度，让 v 成为更深的节点
    if (depth[u] > depth[v]) std::swap(u, v);
    // 将 v 跳到和 u 同样的高度
    for (int j = LOGN - 1; j >= 0; --j) {
        if (depth[v] - (1 << j) >= depth[u]) v = up[v][j];
    }
    // 2. 如果此时 u == v, 说明 u 是 v 的祖先
    if (u == v) return u;
    // 3. 同步向上跳，直到成为 LCA 的子节点
    for (int j = LOGN - 1; j >= 0; --j) {
        if (up[u][j] != up[v][j]) {
            u = up[u][j];
            v = up[v][j];
        }
    }
    return up[u][0];
}
```