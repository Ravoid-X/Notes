# B
```
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

// B-Tree 节点
class BTreeNode {
public:
    int *keys;      // 键数组
    int t;          // 最小度 (Minimum degree)
    BTreeNode **C;  // 子指针数组
    int n;          // 当前键的数量
    bool leaf;      // 是否是叶节点

    BTreeNode(int _t, bool _leaf) : t(_t), leaf(_leaf), n(0) {
        keys = new int[2 * t - 1];
        C = new BTreeNode *[2 * t];
    }

    ~BTreeNode() {
        delete[] keys;
        delete[] C;
    }

    // 查找键 k 在当前节点中的位置
    int findKey(int k) {
        int idx = 0;
        while (idx < n && keys[idx] < k)
            ++idx;
        return idx;
    }

    // 从非叶节点删除键 k
    void removeFromNonLeaf(int idx);

    // 从叶节点删除键 k
    void removeFromLeaf(int idx);

    // 获取前驱
    int getPred(int idx);

    // 获取后继
    int getSucc(int idx);

    // 填充子节点 C[idx] (使其键数 >= t)
    void fill(int idx);

    // 从 C[idx-1] 借一个键
    void borrowFromPrev(int idx);

    // 从 C[idx+1] 借一个键
    void borrowFromNext(int idx);

    // 合并 C[idx] 和 C[idx+1]
    void merge(int idx);

    // 遍历以当前节点为根的子树
    void traverse() {
        int i;
        for (i = 0; i < n; i++) {
            if (!leaf)
                C[i]->traverse();
            cout << " " << keys[i];
        }

        // 打印最后一个子树
        if (!leaf)
            C[i]->traverse();
    }

    // 搜索键 k
    BTreeNode *search(int k) {
        int i = 0;
        while (i < n && k > keys[i])
            i++;

        if (keys[i] == k)
            return this;

        if (leaf)
            return nullptr;

        return C[i]->search(k);
    }

    // 在非满节点中插入
    void insertNonFull(int k);

    // 分裂子节点 C[i] (它必须是满的)
    void splitChild(int i, BTreeNode *y);

    // 删除键 k
    void remove(int k);
};

class BTree {
public:
    BTreeNode *root;
    int t; // 最小度

    BTree(int _t) : root(nullptr), t(_t) {}

    ~BTree() {
        // (需要递归删除所有节点)
    }

    void traverse() {
        if (root != nullptr)
            root->traverse();
        cout << endl;
    }

    BTreeNode *search(int k) {
        return (root == nullptr) ? nullptr : root->search(k);
    }

    // 插入键 k
    void insert(int k);

    // 删除键 k
    void remove(int k);
};

// BTreeNode 方法实现

void BTreeNode::insertNonFull(int k) {
    int i = n - 1; // 从最右边的键开始

    if (leaf) {
        // 如果是叶节点，找到 k 的位置并插入
        while (i >= 0 && keys[i] > k) {
            keys[i + 1] = keys[i];
            i--;
        }
        keys[i + 1] = k;
        n = n + 1;
    } else {
        // 如果是内部节点
        // 找到 k 应该插入的子节点
        while (i >= 0 && keys[i] > k)
            i--;

        // C[i+1] 是目标子节点
        if (C[i + 1]->n == 2 * t - 1) {
            // 如果子节点满了，先分裂
            splitChild(i + 1, C[i + 1]);

            // 分裂后，中间键上升，决定 k 插入到哪个新节点
            if (keys[i + 1] < k)
                i++;
        }
        C[i + 1]->insertNonFull(k);
    }
}

void BTreeNode::splitChild(int i, BTreeNode *y) {
    // y 是 C[i]，并且是满的
    // z 是新节点，将存储 y 的后 t-1 个键
    BTreeNode *z = new BTreeNode(y->t, y->leaf);
    z->n = t - 1;

    // 复制 y 的后 t-1 个键到 z
    for (int j = 0; j < t - 1; j++)
        z->keys[j] = y->keys[j + t];

    // 如果 y 不是叶节点，复制 y 的后 t 个子节点到 z
    if (!y->leaf) {
        for (int j = 0; j < t; j++)
            z->C[j] = y->C[j + t];
    }

    // y 的键数量减少
    y->n = t - 1;

    // 为新子节点 z 腾出空间
    for (int j = n; j >= i + 1; j--)
        C[j + 1] = C[j];

    // 将 z 链接到 C[i+1]
    C[i + 1] = z;

    // 为 y 的中间键腾出空间
    for (int j = n - 1; j >= i; j--)
        keys[j + 1] = keys[j];

    // 将 y 的中间键提升到当前节点
    keys[i] = y->keys[t - 1];

    // 键数量增加
    n = n + 1;
}

// BTree 方法实现

void BTree::insert(int k) {
    if (root == nullptr) {
        // 树为空
        root = new BTreeNode(t, true);
        root->keys[0] = k;
        root->n = 1;
    } else {
        // 如果根节点满了
        if (root->n == 2 * t - 1) {
            // 创建新根
            BTreeNode *s = new BTreeNode(t, false);
            // 旧根成为新根的子节点
            s->C[0] = root;
            // 分裂旧根
            s->splitChild(0, root);

            // 新根现在有2个子节点，决定 k 插入哪个
            int i = 0;
            if (s->keys[0] < k)
                i++;
            s->C[i]->insertNonFull(k);

            // 更新根
            root = s;
        } else {
            // 如果根未满
            root->insertNonFull(k);
        }
    }
}

// ----------------- B树删除 -----------------

void BTreeNode::remove(int k) {
    int idx = findKey(k);

    if (idx < n && keys[idx] == k) {
        // Case 1: k 在当前节点
        if (leaf)
            removeFromLeaf(idx);
        else
            removeFromNonLeaf(idx);
    } else {
        // Case 2: k 不在当前节点
        if (leaf) {
            // k 不在树中
            cout << "Key " << k << " does not exist." << endl;
            return;
        }

        // k 应该在 C[idx] 子树中
        bool flag = (idx == n); // k 是否在最后一个子树中

        if (C[idx]->n < t) {
            // 如果子节点键太少 (t-1)，先填充它
            fill(idx);
        }

        // 在 fill 之后，C[idx] 可能与 C[idx-1] 合并了
        // 如果 k 在最后一个子树中 (flag=true) 并且 C[idx] 被合并到了 C[idx-1]
        if (flag && idx > n)
            C[idx - 1]->remove(k);
        else
            C[idx]->remove(k);
    }
}

void BTreeNode::removeFromLeaf(int idx) {
    // 从 idx 处左移所有键
    for (int i = idx + 1; i < n; ++i)
        keys[i - 1] = keys[i];
    n--;
}

void BTreeNode::removeFromNonLeaf(int idx) {
    int k = keys[idx];

    // Case 2a: 左子节点 C[idx] 至少有 t 个键
    // 找到 k 的前驱 pred，替换 k，然后递归删除 pred
    if (C[idx]->n >= t) {
        int pred = getPred(idx);
        keys[idx] = pred;
        C[idx]->remove(pred);
    }
    // Case 2b: 右子节点 C[idx+1] 至少有 t 个键
    // 找到 k 的后继 succ，替换 k，然后递归删除 succ
    else if (C[idx + 1]->n >= t) {
        int succ = getSucc(idx);
        keys[idx] = succ;
        C[idx + 1]->remove(succ);
    }
    // Case 2c: 左右子节点都只有 t-1 个键
    // 合并 C[idx], k, 和 C[idx+1]
    else {
        merge(idx);
        // k 已经下降到 C[idx] 中，从 C[idx] 中删除 k
        C[idx]->remove(k);
    }
}

int BTreeNode::getPred(int idx) {
    // 前驱是 C[idx] 子树中的最大键
    BTreeNode *cur = C[idx];
    while (!cur->leaf)
        cur = cur->C[cur->n];
    return cur->keys[cur->n - 1];
}

int BTreeNode::getSucc(int idx) {
    // 后继是 C[idx+1] 子树中的最小键
    BTreeNode *cur = C[idx + 1];
    while (!cur->leaf)
        cur = cur->C[0];
    return cur->keys[0];
}

void BTreeNode::fill(int idx) {
    // 如果 C[idx-1] (前一个兄弟) 有富余的键
    if (idx != 0 && C[idx - 1]->n >= t)
        borrowFromPrev(idx);
    // 如果 C[idx+1] (后一个兄弟) 有富余的键
    else if (idx != n && C[idx + 1]->n >= t)
        borrowFromNext(idx);
    // 否则，合并 C[idx] 和它的一个兄弟
    else {
        if (idx != n)
            merge(idx); // 合并 C[idx] 和 C[idx+1]
        else
            merge(idx - 1); // 合并 C[idx-1] 和 C[idx]
    }
}

void BTreeNode::borrowFromPrev(int idx) {
    BTreeNode *child = C[idx];
    BTreeNode *sibling = C[idx - 1];

    // child 中所有键右移
    for (int i = child->n - 1; i >= 0; --i)
        child->keys[i + 1] = child->keys[i];

    // 如果 child 不是叶节点，子指针也右移
    if (!child->leaf) {
        for (int i = child->n; i >= 0; --i)
            child->C[i + 1] = child->C[i];
    }

    // 父节点的 keys[idx-1] 下降到 child
    child->keys[0] = keys[idx - 1];

    // sibling 的最大键上升到父节点
    keys[idx - 1] = sibling->keys[sibling->n - 1];

    // 如果 sibling 不是叶节点，将其最后一个子节点移给 child
    if (!child->leaf)
        child->C[0] = sibling->C[sibling->n];

    child->n += 1;
    sibling->n -= 1;
}

void BTreeNode::borrowFromNext(int idx) {
    BTreeNode *child = C[idx];
    BTreeNode *sibling = C[idx + 1];

    // 父节点的 keys[idx] 下降
    child->keys[child->n] = keys[idx];

    // sibling 的最小键上升
    keys[idx] = sibling->keys[0];

    // 如果不是叶节点，移动子指针
    if (!child->leaf)
        child->C[child->n + 1] = sibling->C[0];

    // sibling 的键左移
    for (int i = 1; i < sibling->n; ++i)
        sibling->keys[i - 1] = sibling->keys[i];

    // sibling 的子指针左移
    if (!sibling->leaf) {
        for (int i = 1; i <= sibling->n; ++i)
            sibling->C[i - 1] = sibling->C[i];
    }

    child->n += 1;
    sibling->n -= 1;
}

void BTreeNode::merge(int idx) {
    BTreeNode *child = C[idx];
    BTreeNode *sibling = C[idx + 1];

    // 1. 将父节点的 keys[idx] 下降到 child
    child->keys[t - 1] = keys[idx];

    // 2. 将 sibling 的所有键 (t-1个) 复制到 child
    for (int i = 0; i < sibling->n; ++i)
        child->keys[i + t] = sibling->keys[i];

    // 3. 如果不是叶节点，复制 sibling 的子指针 (t个)
    if (!child->leaf) {
        for (int i = 0; i <= sibling->n; ++i)
            child->C[i + t] = sibling->C[i];
    }

    // 4. 更新父节点：删除 keys[idx] 和 C[idx+1]
    for (int i = idx + 1; i < n; ++i)
        keys[i - 1] = keys[i];
    for (int i = idx + 2; i <= n; ++i)
        C[i - 1] = C[i];

    // 5. 更新节点计数
    child->n += sibling->n + 1;
    n--;

    // 6. 释放 sibling
    delete sibling;
}

void BTree::remove(int k) {
    if (!root) {
        cout << "The tree is empty." << endl;
        return;
    }

    root->remove(k);

    // 如果根节点在合并后变为空
    if (root->n == 0) {
        BTreeNode *tmp = root;
        if (root->leaf)
            root = nullptr;
        else
            root = root->C[0]; // 唯一的子节点成为新根
        delete tmp;
    }
}
```
# B+
```
#include <iostream>
#include <vector>
#include <algorithm> // for std::copy, std::copy_backward
#include <queue>     // for printTree (BFS)
#include <cmath>     // for ceil
#include <iomanip>   // for std::setw

// 定义B+树的阶 (Order)
// ORDER = m (最大子节点指针数)
const int ORDER = 4;

// 最大键数
const int MAX_KEYS = ORDER - 1; // 3

// 内部节点最小键数
const int MIN_KEYS_INTERNAL = (int)ceil(ORDER / 2.0) - 1; // ceil(4/2)-1 = 1

// 叶节点最小键数
// (注意：叶节点和内部节点的最小键数定义可能不同)
const int MIN_KEYS_LEAF = (int)ceil((ORDER - 1) / 2.0); // ceil(3/2) = 2

// B+树节点
struct BPlusTreeNode {
    bool isLeaf;
    int size;               // 当前键的数量
    int *keys;              // 键数组 (大小: MAX_KEYS)
    BPlusTreeNode *parent;  // 父节点指针

    // 对于内部节点
    BPlusTreeNode **children; // 子节点指针数组 (大小: ORDER)

    // 对于叶节点
    int *data;              // 数据数组 (大小: MAX_KEYS)
    BPlusTreeNode *next;    // 指向下一个叶节点的指针

    BPlusTreeNode() {
        // 为最大容量分配内存
        keys = new int[MAX_KEYS];
        children = new BPlusTreeNode *[ORDER];
        data = new int[MAX_KEYS];

        isLeaf = false;
        size = 0;
        parent = nullptr;
        next = nullptr;
        
        // 初始化子节点指针
        for (int i = 0; i < ORDER; i++) {
            children[i] = nullptr;
        }
    }

    ~BPlusTreeNode() {
        delete[] keys;
        delete[] children;
        delete[] data;
    }
};

class BPlusTree {
public:
    BPlusTreeNode *root;

    BPlusTree() {
        // 树从一个空的叶节点开始
        root = new BPlusTreeNode();
        root->isLeaf = true;
    }

    ~BPlusTree() {
        recursiveDestroy(root);
    }

    // 递归删除所有节点
    void recursiveDestroy(BPlusTreeNode* node) {
        if (node == nullptr) return;
        if (!node->isLeaf) {
            for (int i = 0; i <= node->size; i++) {
                recursiveDestroy(node->children[i]);
            }
        }
        delete node;
    }

    // --- 1. 搜索 ---

    /**
     * @brief 查找包含指定键的叶节点
     * @param key 要查找的键
     * @return 应该包含该键的叶节点
     */
    BPlusTreeNode* findLeaf(int key) {
        BPlusTreeNode *curr = root;
        while (!curr->isLeaf) {
            // 在内部节点中，找到第一个大于 'key' 的键
            // 子树 curr->children[i] 包含的键 <= curr->keys[i]
            //
            // 找到第一个 curr->keys[i] > key
            // int i = 0;
            // while (i < curr->size && key >= curr->keys[i]) {
            //     i++;
            // }
            // curr = curr->children[i];

            // 使用 std::upper_bound (更高效)
            // 找到第一个 > key 的键的位置
            int* it = std::upper_bound(curr->keys, curr->keys + curr->size, key);
            int index = std::distance(curr->keys, it);

            // 如果 key 在 keys[index-1] 处，它仍然在 children[index]
            // 如果 key >= keys[size-1], index 将是 size, 导航到 children[size]
            
            // 修正：upper_bound 找到第一个 > key 的。
            // 我们要找第一个 >= key 的。
            // 例子: keys=[10, 20], 找 15. upper_bound(15) -> 20 (index 1). C[1]
            // 例子: keys=[10, 20], 找 10. upper_bound(10) -> 20 (index 1). C[1]
            // 例子: keys=[10, 20], 找 5.  upper_bound(5)  -> 10 (index 0). C[0]
            // 例子: keys=[10, 20], 找 30. upper_bound(30) -> end (index 2). C[2]
            
            // 逻辑应该是：找到第一个 key[i] > key
            int i = 0;
            while(i < curr->size && key >= curr->keys[i]) {
                i++;
            }
            curr = curr->children[i];
        }
        return curr;
    }

    /**
     * @brief 搜索一个键
     * @return 成功返回 true, 失败返回 false
     */
    bool search(int key) {
        if (root == nullptr) return false;
        
        BPlusTreeNode *leaf = findLeaf(key);
        
        // 在叶节点中二分查找 (或线性查找)
        int* it = std::lower_bound(leaf->keys, leaf->keys + leaf->size, key);
        int index = std::distance(leaf->keys, it);

        if (index < leaf->size && leaf->keys[index] == key) {
            std::cout << "Found key " << key << " with data " << leaf->data[index] << std::endl;
            return true;
        } else {
            std::cout << "Key " << key << " not found." << std::endl;
            return false;
        }
    }

    /**
     * @brief 范围搜索
     */
    void rangeSearch(int start_key, int end_key) {
        if (root == nullptr) return;

        BPlusTreeNode *leaf = findLeaf(start_key);
        std::cout << "Range search from " << start_key << " to " << end_key << ":" << std::endl;
        bool found = false;

        while (leaf != nullptr) {
            for (int i = 0; i < leaf->size; i++) {
                if (leaf->keys[i] >= start_key && leaf->keys[i] <= end_key) {
                    std::cout << "  Key: " << leaf->keys[i] << ", Data: " << leaf->data[i] << std::endl;
                    found = true;
                }
                
                if (leaf->keys[i] > end_key) {
                    // 我们已经超出了范围
                    if (!found) std::cout << "  No keys found in range." << std::endl;
                    return;
                }
            }
            // 移动到下一个叶节点
            leaf = leaf->next;
        }
        if (!found) std::cout << "  No keys found in range." << std::endl;
    }


    // --- 2. 插入 ---

    /**
     * @brief 插入主函数
     */
    void insert(int key, int data) {
        if (root == nullptr) {
            root = new BPlusTreeNode();
            root->isLeaf = true;
        }

        BPlusTreeNode *leaf = findLeaf(key);
        insertIntoLeaf(leaf, key, data);
    }

    /**
     * @brief 步骤 2.1: 在叶节点中插入
     */
    void insertIntoLeaf(BPlusTreeNode* leaf, int key, int data) {
        // 找到插入位置
        int i = leaf->size - 1;
        while (i >= 0 && leaf->keys[i] > key) {
            leaf->keys[i + 1] = leaf->keys[i];
            leaf->data[i + 1] = leaf->data[i];
            i--;
        }
        // 插入
        leaf->keys[i + 1] = key;
        leaf->data[i + 1] = data;
        leaf->size++;

        // 检查上溢 (Overflow)
        if (leaf->size > MAX_KEYS) {
            splitLeaf(leaf);
        }
    }

    /**
     * @brief 步骤 2.2: 分裂叶节点
     */
    void splitLeaf(BPlusTreeNode* leaf) {
        // 创建新叶节点
        BPlusTreeNode *newLeaf = new BPlusTreeNode();
        newLeaf->isLeaf = true;
        newLeaf->parent = leaf->parent;

        // 找到分裂点
        int splitPoint = (int)ceil(ORDER / 2.0); // e.g., ORDER=4 -> size=4 -> split=2

        // 将后半部分数据移动到新叶节点
        for (int i = splitPoint; i < leaf->size; i++) {
            newLeaf->keys[i - splitPoint] = leaf->keys[i];
            newLeaf->data[i - splitPoint] = leaf->data[i];
            newLeaf->size++;
        }

        // 更新原叶节点的大小
        leaf->size = splitPoint;

        // 更新叶节点链表
        newLeaf->next = leaf->next;
        leaf->next = newLeaf;

        // 将新叶节点的第一个键 "复制" (Copy Up) 到父节点
        int keyToPush = newLeaf->keys[0];
        insertIntoParent(leaf, keyToPush, newLeaf);
    }

    /**
     * @brief 步骤 2.3: 插入到父节点 (递归)
     */
    void insertIntoParent(BPlusTreeNode* left, int key, BPlusTreeNode* right) {
        if (left == root) {
            // 如果分裂的是根节点，创建新根
            BPlusTreeNode *newRoot = new BPlusTreeNode();
            newRoot->keys[0] = key;
            newRoot->children[0] = left;
            newRoot->children[1] = right;
            newRoot->size = 1;
            newRoot->isLeaf = false;
            root = newRoot;
            left->parent = newRoot;
            right->parent = newRoot;
            return;
        }

        BPlusTreeNode *parent = left->parent;
        
        // 找到插入位置
        int i = parent->size - 1;
        while (i >= 0 && parent->keys[i] > key) {
            parent->keys[i + 1] = parent->keys[i];
            parent->children[i + 2] = parent->children[i + 1];
            i--;
        }
        // 插入
        parent->keys[i + 1] = key;
        parent->children[i + 2] = right;
        parent->size++;

        // 检查上溢 (Overflow)
        if (parent->size > MAX_KEYS) {
            splitInternal(parent);
        }
    }

    /**
     * @brief 步骤 2.4: 分裂内部节点
     */
    void splitInternal(BPlusTreeNode* node) {
        BPlusTreeNode *newInternal = new BPlusTreeNode();
        newInternal->isLeaf = false;
        newInternal->parent = node->parent;
        
        // 内部节点分裂: 中间键被 "提升" (Push Up) 
        // e.g., ORDER=4, MAX_KEYS=3. size=4. keys=[k0,k1,k2,k3]. children=[c0,c1,c2,c3,c4]
        // 分裂点: (MAX_KEYS / 2) = 3 / 2 = 1.
        // key[1] (k1) 被提升
        int splitKeyIndex = MAX_KEYS / 2; // e.g., 3/2 = 1
        int keyToPush = node->keys[splitKeyIndex];

        // 复制分裂点 *之后* 的键到新节点
        int newIndex = 0;
        for (int i = splitKeyIndex + 1; i < node->size; i++) {
            newInternal->keys[newIndex++] = node->keys[i];
        }

        // 复制分裂点 *之后* 的子节点到新节点
        newIndex = 0;
        for (int i = splitKeyIndex + 1; i <= node->size; i++) {
            newInternal->children[newIndex] = node->children[i];
            if (newInternal->children[newIndex] != nullptr) {
                newInternal->children[newIndex]->parent = newInternal;
            }
            newIndex++;
        }

        // 更新原节点大小
        node->size = splitKeyIndex;
        newInternal->size = (MAX_KEYS + 1) - (splitKeyIndex + 1); // e.g. 4 - (1+1) = 2
        
        // 递归插入
        insertIntoParent(node, keyToPush, newInternal);
    }


    // --- 3. 删除 ---

    /**
     * @brief 删除主函数
     */
    void remove(int key) {
        if (root == nullptr) return;

        BPlusTreeNode* leaf = findLeaf(key);
        
        // 查找键在叶节点中的索引
        int keyIndex = findKeyIndexInNode(leaf, key);
        if (keyIndex == -1) {
            std::cout << "Key " << key << " not found for removal." << std::endl;
            return;
        }

        // 步骤 3.1: 从叶节点中删除
        // 向左移动键和数据
        for (int i = keyIndex; i < leaf->size - 1; i++) {
            leaf->keys[i] = leaf->keys[i + 1];
            leaf->data[i] = leaf->data[i + 1];
        }
        leaf->size--;

        // 步骤 3.2: 检查下溢 (Underflow)
        if (leaf == root) {
            // 如果根是叶节点，它可以是任意大小 (包括 0)
            return;
        }

        if (leaf->size < MIN_KEYS_LEAF) {
            handleLeafUnderflow(leaf);
        }
    }

    /**
     * @brief 在节点中查找键的索引
     */
    int findKeyIndexInNode(BPlusTreeNode* node, int key) {
        for (int i = 0; i < node->size; i++) {
            if (node->keys[i] == key) return i;
        }
        return -1;
    }

    /**
     * @brief 获取节点在其父节点中的子节点索引
     */
    int getSiblingIndex(BPlusTreeNode* node) {
        if (node->parent == nullptr) return -1;
        BPlusTreeNode* parent = node->parent;
        for (int i = 0; i <= parent->size; i++) {
            if (parent->children[i] == node) return i;
        }
        return -1; // 不应发生
    }

    /**
     * @brief 步骤 3.3: 处理叶节点下溢 (借键或合并)
     */
    void handleLeafUnderflow(BPlusTreeNode* leaf) {
        BPlusTreeNode* parent = leaf->parent;
        int siblingIndex = getSiblingIndex(leaf);
        int parentKeyIndex = -1; // 父节点中分隔键的索引

        BPlusTreeNode* leftSibling = nullptr;
        if (siblingIndex > 0) {
            leftSibling = parent->children[siblingIndex - 1];
            parentKeyIndex = siblingIndex - 1;
        }

        BPlusTreeNode* rightSibling = nullptr;
        if (siblingIndex < parent->size) {
            rightSibling = parent->children[siblingIndex + 1];
            if (leftSibling == nullptr) parentKeyIndex = siblingIndex;
        }

        // --- 尝试从左兄弟借 (Borrow) ---
        if (leftSibling && leftSibling->size > MIN_KEYS_LEAF) {
            // 1. 从左兄弟获取最后一个键/数据
            int borrowedKey = leftSibling->keys[leftSibling->size - 1];
            int borrowedData = leftSibling->data[leftSibling->size - 1];
            leftSibling->size--;

            // 2. 在 'leaf' 中为新键/数据腾出空间
            std::copy_backward(leaf->keys, leaf->keys + leaf->size, leaf->keys + leaf->size + 1);
            std::copy_backward(leaf->data, leaf->data + leaf->size, leaf->data + leaf->size + 1);
            
            // 3. 插入新键/数据
            leaf->keys[0] = borrowedKey;
            leaf->data[0] = borrowedData;
            leaf->size++;
            
            // 4. 更新父节点的键 (用 'leaf' 的新首键)
            parent->keys[parentKeyIndex] = leaf->keys[0];
        }
        // --- 尝试从右兄弟借 (Borrow) ---
        else if (rightSibling && rightSibling->size > MIN_KEYS_LEAF) {
            // 1. 从右兄弟获取第一个键/数据
            int borrowedKey = rightSibling->keys[0];
            int borrowedData = rightSibling->data[0];

            // 2. 向左移动右兄弟的键/数据
            std::copy(rightSibling->keys + 1, rightSibling->keys + rightSibling->size, rightSibling->keys);
            std::copy(rightSibling->data + 1, rightSibling->data + rightSibling->size, rightSibling->data);
            rightSibling->size--;

            // 3. 将新键/数据添加到 'leaf' 的末尾
            leaf->keys[leaf->size] = borrowedKey;
            leaf->data[leaf->size] = borrowedData;
            leaf->size++;
            
            // 4. 更新父节点的键 (用 'rightSibling' 的新首键)
            parent->keys[parentKeyIndex] = rightSibling->keys[0];
        }
        // --- 与左兄弟合并 (Merge) ---
        else if (leftSibling) {
            // 1. 将 'leaf' 的所有键/数据复制到 'leftSibling'
            for (int i = 0; i < leaf->size; i++) {
                leftSibling->keys[leftSibling->size] = leaf->keys[i];
                leftSibling->data[leftSibling->size] = leaf->data[i];
                leftSibling->size++;
            }
            // 2. 更新叶链表
            leftSibling->next = leaf->next;
            
            // 3. 从父节点删除键和 'leaf' 指针
            // parentKeyIndex 是分隔键
            removeEntryFromInternalNode(parent, parentKeyIndex, siblingIndex);
            
            delete leaf;
        }
        // --- 与右兄弟合并 (Merge) ---
        else if (rightSibling) {
            // 1. 将 'rightSibling' 的所有键/数据复制到 'leaf'
            for (int i = 0; i < rightSibling->size; i++) {
                leaf->keys[leaf->size] = rightSibling->keys[i];
                leaf->data[leaf->size] = rightSibling->data[i];
                leaf->size++;
            }
            // 2. 更新叶链表
            leaf->next = rightSibling->next;
            
            // 3. 从父节点删除键和 'rightSibling' 指针
            // parentKeyIndex 是分隔键
            removeEntryFromInternalNode(parent, parentKeyIndex, siblingIndex + 1);

            delete rightSibling;
        }
    }

    /**
     * @brief 从内部节点中删除一个条目 (键和子指针)
     * @param node 内部节点
     * @param keyIndex 要删除的键的索引
     * @param childIndex 要删除的子指针的索引
     */
    void removeEntryFromInternalNode(BPlusTreeNode* node, int keyIndex, int childIndex) {
        // 删除键
        std::copy(node->keys + keyIndex + 1, node->keys + node->size, node->keys + keyIndex);
        
        // 删除子指针
        std::copy(node->children + childIndex + 1, node->children + node->size + 1, node->children + childIndex);

        node->size--;

        // 步骤 3.4: 检查内部节点的下溢 (递归)
        if (node == root) {
            if (root->size == 0 && !root->isLeaf) {
                // 根节点是空的内部节点，树的高度降低
                BPlusTreeNode *oldRoot = root;
                root = root->children[0];
                root->parent = nullptr;
                delete oldRoot;
            }
            return;
        }

        if (node->size < MIN_KEYS_INTERNAL) {
            handleInternalUnderflow(node);
        }
    }

    /**
     * @brief 步骤 3.5: 处理内部节点下溢 (借键或合并)
     */
    void handleInternalUnderflow(BPlusTreeNode* node) {
        BPlusTreeNode* parent = node->parent;
        int siblingIndex = getSiblingIndex(node);
        int parentKeyIndex = -1;

        BPlusTreeNode* leftSibling = nullptr;
        if (siblingIndex > 0) {
            leftSibling = parent->children[siblingIndex - 1];
            parentKeyIndex = siblingIndex - 1;
        }

        BPlusTreeNode* rightSibling = nullptr;
        if (siblingIndex < parent->size) {
            rightSibling = parent->children[siblingIndex + 1];
            if (leftSibling == nullptr) parentKeyIndex = siblingIndex;
        }

        // --- 尝试从左兄弟借 (Borrow) ---
        if (leftSibling && leftSibling->size > MIN_KEYS_INTERNAL) {
            // 1. 父节点的键下降
            // 2. 左兄弟的最大键上升
            int parentKey = parent->keys[parentKeyIndex];
            int borrowedKey = leftSibling->keys[leftSibling->size - 1];
            BPlusTreeNode* borrowedChild = leftSibling->children[leftSibling->size];
            leftSibling->size--;

            // 3. 在 'node' 中为新键/子节点腾出空间
            std::copy_backward(node->keys, node->keys + node->size, node->keys + node->size + 1);
            std::copy_backward(node->children, node->children + node->size + 1, node->children + node->size + 2);
            
            // 4. 插入下降的父键和借来的子节点
            node->keys[0] = parentKey;
            node->children[0] = borrowedChild;
            if (borrowedChild) borrowedChild->parent = node;
            node->size++;

            // 5. 更新父节点的键
            parent->keys[parentKeyIndex] = borrowedKey;
        }
        // --- 尝试从右兄弟借 (Borrow) ---
        else if (rightSibling && rightSibling->size > MIN_KEYS_INTERNAL) {
            // 1. 父节点的键下降
            // 2. 右兄弟的最小键上升
            int parentKey = parent->keys[parentKeyIndex];
            int borrowedKey = rightSibling->keys[0];
            BPlusTreeNode* borrowedChild = rightSibling->children[0];
            
            // 3. 向左移动右兄弟的键/子节点
            std::copy(rightSibling->keys + 1, rightSibling->keys + rightSibling->size, rightSibling->keys);
            std::copy(rightSibling->children + 1, rightSibling->children + rightSibling->size + 1, rightSibling->children);
            rightSibling->size--;

            // 4. 将下降的父键和借来的子节点插入 'node'
            node->keys[node->size] = parentKey;
            node->children[node->size + 1] = borrowedChild;
            if (borrowedChild) borrowedChild->parent = node;
            node->size++;
            
            // 5. 更新父节点的键
            parent->keys[parentKeyIndex] = borrowedKey;
        }
        // --- 与左兄弟合并 (Merge) ---
        else if (leftSibling) {
            // 1. 将父节点的键下降到 'leftSibling'
            leftSibling->keys[leftSibling->size] = parent->keys[parentKeyIndex];
            leftSibling->size++;
            
            // 2. 将 'node' 的所有键/子节点复制到 'leftSibling'
            for (int i = 0; i < node->size; i++) {
                leftSibling->keys[leftSibling->size] = node->keys[i];
                leftSibling->children[leftSibling->size] = node->children[i];
                if (node->children[i]) node->children[i]->parent = leftSibling;
                leftSibling->size++;
            }
            leftSibling->children[leftSibling->size] = node->children[node->size];
            if (node->children[node->size]) node->children[node->size]->parent = leftSibling;
            
            // 3. 从父节点删除键和 'node' 指针
            removeEntryFromInternalNode(parent, parentKeyIndex, siblingIndex);
            
            delete node;
        }
        // --- 与右兄弟合并 (Merge) ---
        else if (rightSibling) {
            // 1. 将父节点的键下降到 'node'
            node->keys[node->size] = parent->keys[parentKeyIndex];
            node->size++;
            
            // 2. 将 'rightSibling' 的所有键/子节点复制到 'node'
            for (int i = 0; i < rightSibling->size; i++) {
                node->keys[node->size] = rightSibling->keys[i];
                node->children[node->size] = rightSibling->children[i];
                if (rightSibling->children[i]) rightSibling->children[i]->parent = node;
                node->size++;
            }
            node->children[node->size] = rightSibling->children[rightSibling->size];
            if (rightSibling->children[rightSibling->size]) rightSibling->children[rightSibling->size]->parent = node;
            
            // 3. 从父节点删除键和 'rightSibling' 指针
            removeEntryFromInternalNode(parent, parentKeyIndex, siblingIndex + 1);
            
            delete rightSibling;
        }
    }

    // --- 4. 打印 ---

    /**
     * @brief 打印树 (层序遍历 BFS)
     */
    void printTree() {
        if (root == nullptr || (root->size == 0 && root->isLeaf)) {
            std::cout << "Tree is empty." << std::endl;
            return;
        }
        
        std::queue<std::pair<BPlusTreeNode*, int>> q;
        q.push({root, 0});
        int currentLevel = 0;
        std::cout << "--- B+ Tree (Order=" << ORDER << ") ---" << std::endl;
        
        while (!q.empty()) {
            BPlusTreeNode* node = q.front().first;
            int level = q.front().second;
            q.pop();

            if (level != currentLevel) {
                std::cout << std::endl;
                currentLevel = level;
            }
            
            std::cout << "L" << level << ": [";
            for (int i = 0; i < node->size; i++) {
                std::cout << node->keys[i];
                if (i < node->size - 1) std::cout << ",";
            }
            std::cout << "] ";

            if (!node->isLeaf) {
                for (int i = 0; i <= node->size; i++) {
                    if(node->children[i]) q.push({node->children[i], level + 1});
                }
            }
        }
        std::cout << std::endl;
        printLeafList();
        std::cout << "-------------------------" << std::endl;
    }
    
    /**
     * @brief 打印叶节点链表
     */
    void printLeafList() {
        if (root == nullptr) return;
        
        BPlusTreeNode* leaf = root;
        while (!leaf->isLeaf) {
            leaf = leaf->children[0];
        }
        
        std::cout << "Leaf List: ";
        while (leaf != nullptr) {
            std::cout << "[";
            for (int i = 0; i < leaf->size; i++) {
                std::cout << leaf->keys[i];
                if (i < leaf->size - 1) std::cout << ",";
            }
            std::cout << "] -> ";
            leaf = leaf->next;
        }
        std::cout << "NULL" << std::endl;
    }
};


// --- 主函数：测试 ---
int main() {
    BPlusTree tree;

    std::cout << "--- 插入测试 (ORDER=4) ---" << std::endl;
    // 插入会导致分裂
    tree.insert(10, 100);
    tree.insert(20, 200);
    tree.insert(30, 300); // 叶节点满
    tree.printTree();
    
    tree.insert(40, 400); // 叶节点分裂, 根节点分裂
    std::cout << "\nAfter inserting 40 (Leaf & Root split):" << std::endl;
    tree.printTree();
    
    tree.insert(5, 50);
    tree.insert(15, 150);
    tree.insert(25, 250); // 叶节点分裂
    std::cout << "\nAfter inserting 5, 15, 25 (Leaf split):" << std::endl;
    tree.printTree();
    
    tree.insert(35, 350);
    tree.insert(50, 500); // 叶节点分裂, 内部节点分裂, 根节点分裂
    std::cout << "\nAfter inserting 35, 50 (Internal & Root split):" << std::endl;
    tree.printTree();
    
    tree.insert(1, 10);
    tree.insert(12, 120);
    tree.insert(18, 180);
    tree.insert(22, 220);
    tree.insert(28, 280);
    tree.insert(38, 380);
    tree.insert(45, 450);
    tree.insert(55, 550);
    tree.insert(60, 600);
    std::cout << "\nAfter inserting more values:" << std::endl;
    tree.printTree();

    std::cout << "\n--- 搜索测试 ---" << std::endl;
    tree.search(25);
    tree.search(30);
    tree.search(99); // 不存在

    std::cout << "\n--- 范围搜索测试 ---" << std::endl;
    tree.rangeSearch(15, 45);
    
    
    std::cout << "\n--- 删除测试 ---" << std::endl;

    std::cout << "\nRemoving 38 (Leaf borrow from left):" << std::endl;
    tree.remove(38);
    tree.printTree();

    std::cout << "\nRemoving 40 (Leaf merge with left):" << std::endl;
    tree.remove(40);
    tree.printTree();
    
    std::cout << "\nRemoving 35 (Internal underflow, borrow from right):" << std::endl;
    tree.remove(35);
    tree.printTree();

    std::cout << "\nRemoving 50, 55, 60 (Cascade Merge):" << std::endl;
    tree.remove(50);
    tree.remove(55);
    tree.remove(60);
    std::cout << "(After 50, 55, 60 removed)" << std::endl;
    tree.printTree(); // 树高应该降低

    std::cout << "\nRemoving 1, 5, 10, 12, 15, 18..." << std::endl;
    tree.remove(1);
    tree.remove(5);
    tree.remove(10);
    tree.remove(12);
    tree.remove(15);
    tree.remove(18);
    tree.printTree();

    std::cout << "\nRemoving 20, 22, 25..." << std::endl;
    tree.remove(20);
    tree.remove(22);
    tree.remove(25);
    tree.printTree();

    std::cout << "\nRemoving 28, 30, 45 (Final removals):" << std::endl;
    tree.remove(28);
    tree.remove(30);
    tree.remove(45);
    tree.printTree(); // 树应该为空
    return 0;
}
```