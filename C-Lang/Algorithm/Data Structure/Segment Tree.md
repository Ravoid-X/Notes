## 引出
区间更新问题，如果使用朴素的数组遍历会超时，线段树将数组划分为一系列区间，每个节点代表一个区间。通过懒惰标记（Lazy Propagation）技术，可以将区间更新操作的时间复杂度优化到 O(logn)。
## 区间加乘与单点求值
### 问题
<img src="../../../pic/C-Lang/Algorithm/Data Structure/seg_tree_exp1.png" style="width:600px;padding:10px;"/>

### 解题思路
1. 乘法与加法共存，节点值表示为 (a * mul + add)\
（1）乘法：mul *= x，add *= x\
（2）加法：mul不变，add += x
### 程序
```
struct Node {
    int sum; //该节点区间的和
    ll add; 
    ll mul; 
};
class SegmentTree {
private:
    vector<Node> tree;
    vector<ll> a;
    int n;
    void pushdown(int node, int L, int R) {
        if (L == R || (tree[node].mul == 1 && tree[node].add == 0)) {
            return;
        }
        int M = L + (R - L) / 2;
        int left = 2 * node;
        int right = 2 * node + 1;
        ll mul = tree[node].mul;
        ll add = tree[node].add;
        tree[node].mul = 1;
        tree[node].add = 0;
        tree[left].sum = (tree[left].sum * mul + (M - L + 1) * add) % P;
        tree[left].mul = (tree[left].mul * mul) % P;
        tree[left].add = (tree[left].add * mul + add) % P;
        tree[right].sum = (tree[right].sum * mul + (R - M) * add) % P;
        tree[right].mul = (tree[right].mul * mul) % P;
        tree[right].add = (tree[right].add * mul + add) % P;
    }
    void build(int node, int L, int R) {
        tree[node].mul = 1;
        tree[node].add = 0;
        if (L == R) {
            tree[node].sum = (a[L] % P + P) % P;
            return;
        }
        int M = L + (R - L) / 2;
        build(2 * node, L, M);
        build(2 * node + 1, M + 1, R);
        tree[node].sum = (tree[2 * node].sum + tree[2 * node + 1].sum) % P;
    }
    void update_mul(int node, int L, int R, int l, int r, ll x) {
        if (l <= L && R <= r) {
            tree[node].sum = (tree[node].sum * x) % P;
            tree[node].mul = (tree[node].mul * x) % P;
            tree[node].add = (tree[node].add * x) % P;
            return;
        }
        pushdown(node, L, R);
        int M = L + (R - L) / 2;
        if (l <= M) update_mul(2 * node, L, M, l, r, x);
        if (r > M) update_mul(2 * node + 1, M + 1, R, l, r, x);
        tree[node].sum = (tree[2 * node].sum + tree[2 * node + 1].sum) % P;
    }
    void update_add(int node, int L, int R, int l, int r, ll x) {
        if (l <= L && R <= r) {
            tree[node].sum = (tree[node].sum + (ll)(R - L + 1) * x) % P;
            tree[node].add = (tree[node].add + x) % P;
            return;
        }
        pushdown(node, L, R);
        int M = L + (R - L) / 2;
        if (l <= M) update_add(2 * node, L, M, l, r, x);
        if (r > M) update_add(2 * node + 1, M + 1, R, l, r, x);
        tree[node].sum = (tree[2 * node].sum + tree[2 * node + 1].sum) % P;
    }
    ll query(int node, int L, int R, int target) {
        if (L == R) {
            return tree[node].sum;
        }
        pushdown(node, L, R);
        int M = L + (R - L) / 2;
        if (target <= M) {
            return query(2 * node, L, M, target);
        } else {
            return query(2 * node + 1, M + 1, R, target);
        }
    }
public:
    SegmentTree(const vector<ll>& array) {
        a=array;
        n = a.size() - 1; 
        tree.resize(4 * (n + 1));
        build(1, 1, n);
    }
    void update_mul(int l, int r, ll x) {
        update_mul(1, 1, n, l, r, (x % P + P) % P);
    }
    void update_add(int l, int r, ll x) {
        update_add(1, 1, n, l, r, (x % P + P) % P);
    }
    ll query(int target) {
        return query(1, 1, n, target);
    }
};
vector<ll> a(n+1,0);
```