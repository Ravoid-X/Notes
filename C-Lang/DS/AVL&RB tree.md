## 平衡二叉树
因为树不断插入的时候，有可能退化成链表，如节点正好从大到小插入。
1. 特性：对于任何一颗子树的root根结点而言，它的左子树任何节点的key一定比root小，而右子树任何节点的key 一定比root大；每个节点的左右子节点的高度之差的绝对值最多为1。
2. 出现如下情况时，AVL树会通过节点的旋转进行进行平衡，调整的过程称之为左旋（逆时针）和右旋。
<img src="../../Pic/C-Lang/DS/avltree-rotate.png" style="width:400px;padding:10px;"/>

3. 旋转支点（pivot）：失去平衡这部分树，在自平衡之后的根节点。子树失衡有四大场景。
4. 左左结构失衡\
插入的元素在子树root的左侧不平衡元素的左侧，左侧的不平衡元素为 pivot, 进行右旋。如果 pivot 有右子树，则作为原 root 的左子树。
<img src="../../Pic/C-Lang/DS/avltree-ll.jpg" style="width:400px;padding:10px;"/>

5. 右右结构失衡\
插入的元素在子树root右侧的不平衡子树的右侧，右侧的不平衡元素为 pivot，进行左旋。如果 pivot 有左子树，则作为原 root 的右子树.
<img src="../../Pic/C-Lang/DS/avltree-rr.png
" style="width:400px;padding:10px;"/>

6. 左右结构失衡（左旋+右旋）\
插入的元素在左侧的不平衡元素的右侧
<img src="../../Pic/C-Lang/DS/avltree-lr.png
" style="width:400px;padding:10px;"/>

7. 右左结构失衡（右旋+左旋）\
插入的元素在右侧的不平衡元素的左侧
<img src="../../Pic/C-Lang/DS/avltree-rl.png
" style="width:400px;padding:10px;"/>

## 平衡二叉树的删除
1. 叶子节点\
直接删除，并从父亲节点开始往上看，判断是否失衡。
2. 节点只有左子树或只有右子树\
将节点删除，以左子树或右子树进行代替，并进行相应的平衡判断
3. 左右子树都有\
找到其前驱或者后驱节点将其替换，再判断是否失衡。

## 红黑树
1. AVL树必须保证左右子树平衡，在插入的时候很容易出现不平衡的情况，需要进行旋转。需要花大量时间在调整上，故AVL树一般用于查询场景，而不是增加删除频繁的场景。
2. RB树在查询速率和平衡调整中寻找平衡，放宽了树的平衡条件，从而可以用于 增加删除 频繁的场景。
3. 特性：节点非黑即红；根节点是黑色；叶子节点是黑色（指为空的叶子节点）；每个红色节点的两个子节点都为黑色；从任一节点到其每个叶子的所有路径，都包含相同数目的黑色节点。
4. 一般在插入红黑树节点的时候，会将这个节点设置为红色，因为红色破坏原则的可能性最小。
5. 当原则不满足时，可进行三种操作：变色、左旋、右旋（与AVL左右旋类似）。

## 红黑树节点插入
1. 空树：直接插入作为根节点，设置为黑色。
2. Key已经存在：更新节点的值，颜色不改变。
3. 父节点为黑色：直接插入无需做自平衡。
<img src="../../Pic/C-Lang/DS/rbtree-father-black.png
" style="width:400px;padding:10px;"/>

4. 父亲和叔叔为红色
<img src="../../Pic/C-Lang/DS/rbtree-father-red.png
" style="width:400px;padding:10px;"/>

5. 父亲为红色，叔叔为黑色，插在父亲的左节点\
（1）LL失衡
<img src="../../Pic/C-Lang/DS/rbtree-left-ll.png
" style="width:400px;padding:10px;"/>
（2）LR失衡<img src="../../Pic/C-Lang/DS/rbtree-left-lr.png
" style="width:400px;padding:10px;"/>

6. 父亲为红色，叔叔为黑点，并且父亲是祖父的右节点
（1）RR失衡
<img src="../../Pic/C-Lang/DS/rbtree-right-rr.png
" style="width:400px;padding:10px;"/>
（2）LR失衡<img src="../../Pic/C-Lang/DS/rbtree-right-rl.png
" style="width:400px;padding:10px;"/>