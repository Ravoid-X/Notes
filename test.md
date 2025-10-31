<img src="../../pic" style="width:600px;padding:10px;"/>
```
class SegmentTreeAdd_PointQuery {
private:
    struct Node {
        ll sum;     // 区间和 (对于叶子节点，这就是它的值)
        ll add_tag; // 区间加法懒标记
        int len;     // 区间长度
        Node() : sum(0), add_tag(0), len(0) {}
    };

    vector<Node> tree;
    vector<ll> arr;
    int n;

    // 向上合并
    void pushup(int u) {
        tree[u].sum = tree[u * 2].sum + tree[u * 2 + 1].sum;
    }

    // 向下应用懒标记
    void pushdown(int u) {
        if (tree[u].add_tag == 0) {
            return;
        }
        ll tag = tree[u].add_tag;

        // 应用到左子节点 (u*2)
        tree[u * 2].sum += tag * tree[u * 2].len;
        tree[u * 2].add_tag += tag;

        // 应用到右子节点 (u*2 + 1)
        tree[u * 2 + 1].sum += tag * tree[u * 2 + 1].len;
        tree[u * 2 + 1].add_tag += tag;

        tree[u].add_tag = 0;
    }

    // 构建
    void build(int u, int l, int r) {
        tree[u].len = r - l + 1;
        if (l == r) {
            tree[u].sum = arr[l];
            return;
        }
        int mid = (l + r) / 2;
        build(u * 2, l, mid);
        build(u * 2 + 1, mid + 1, r);
        pushup(u);
    }

    // 区间加法 (内部实现)
    void update_internal(int u, int l, int r, int L, int R, ll val) {
        if (l > R || r < L) {
            return;
        }
        if (l >= L && r <= R) {
            tree[u].sum += val * tree[u].len;
            tree[u].add_tag += val;
            return;
        }
        pushdown(u); // 先下推
        int mid = (l + r) / 2;
        update_internal(u * 2, l, mid, L, R, val);
        update_internal(u * 2 + 1, mid + 1, r, L, R, val);
        pushup(u); // 再合并
    }
    // --- (核心) 单点查询 (内部实现) ---
    // u: 当前节点, [l, r]: 当前节点代表的区间
    // idx: 目标查询的索引
    ll query_internal(int u, int l, int r, int idx) {
        // 1. 找到了叶子节点
        if (l == r) {
            // 此时 l == r == idx
            // 它的 sum 就是我们要的值
            return tree[u].sum;
        }
        // 2. (关键) 向下传递懒标记
        // 确保子节点的数据在被访问前是“最新”的
        pushdown(u);
        // 3. 递归地在左子树或右子树中查找
        int mid = (l + r) / 2;
        if (idx <= mid) {
            // 目标在左子树
            return query_internal(u * 2, l, mid, idx);
        } else {
            // 目标在右子树
            return query_internal(u * 2 + 1, mid + 1, r, idx);
        }
    }
public:
    SegmentTreeAdd_PointQuery(const vector<ll>& initial_array) {
        n = initial_array.size();
        arr = initial_array;
        tree.resize(4 * n);
        build(1, 0, n - 1);
    }
    // 公开的区间加法接口 (0-based)
    void range_add(int L, int R, ll val) {
        update_internal(1, 0, n - 1, L, R, val);
    }
    // 公开的单点查询接口 (0-based)
    ll point_query(int idx) {
        if (idx < 0 || idx >= n) {
            cerr << "查询索引越界!" << endl;
            return -1; // 或抛出异常
        }
        return query_internal(1, 0, n - 1, idx);
    }
};
```