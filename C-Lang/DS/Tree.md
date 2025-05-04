## 定义
由n（n>=1）个有限结点组成一个具有层次关系的集合。
## 概念
1. 根结点：没有父结点的结点。
2. 兄弟节点：拥有共同父节点的节点互称为兄弟节点。
3. 度：节点的子树数目就是节点的度。
4. 叶子节点：度为零的节点就是叶子节点。
5. 节点的层次：从根节点开始，根节点为第一层，根的子节点为第二层。
6.  节点深度：对任意节点x，x节点的深度表示为根节点到x节点的路径长度。
7.  节点高度：对任意节点x，叶子节点到x节点的路径长度就是节点x的高度。
8.  树的深度：一棵树中节点的最大深度就是树的深度，也称为高度。
9.  祖先：对任意节点x，从根节点到节点x的所有节点都是x的祖先（节点x也是自己的祖先）。
10. 后代：对任意节点x，从节点x到叶子节点的所有节点都是x的后代（节点x也是自己的后代）。
11. 森林：m颗互不相交的树构成的集合就是森林。
## 有序分类
1. 有序树：每个结点的各子树看成是从左到右有次序的(即不能互换)。
2. 无序树：每个结点的各子树从左到右是没有次序的(即可以互换)
## 二叉树
树的任意节点至多包含两棵子树。
1. 完全二叉树：其中所有的层都是完全填满的，除了最后一层。在最后一层中，所有的节点都尽可能地向左对齐。
2. 满二叉树：所有的非叶子节点都有两个子节点。
3. 平衡二叉树：每个节点的左右子树的高度差不超过1。
4. 二叉搜索树：所有左子树的节点的值都小于该节点的值；所有右子树的节点的值都大于该节点的值。
## 构建
```
struct Node{
    int value;
    Node * left;
    Node * right;
    Node(int value):value(value),left(NULL),right(NULL){}
};
void inertNode(Node *node,int value){
    if(value<=node->value){
        if(!node->left){
            node->left=new Node(value);
        }
        else{
            inertNode(node->left,value);
        }
    }
    else{
        if(!node->right){
            node->right=new Node(value);
        }
        else{
            inertNode(node->right,value);
        }
    }
}
```
## 遍历
1. 前序\
深度优先的遍历方法，它首先访问根节点，然后递归地访问左子树，最后访问右子树。
```
void preOrder(Node *node){
    if(node){
        std::cout<<node->value;
        preOrder(node->left);
        preOrder(node->right);
    }

}
void preOrder(Node *node){
    if(node==NULL){
        return;
    }
    std::stack<Node *> nstack;
    nstack.push(node);
    while(!nstack.empty()){
        Node *temp=nstack.top();
        std::cout<<temp->value;
        nstack.pop();
        if(temp->right){
            nstack.push(temp->right);
        }
        if(temp->left){
            nstack.push(temp->left);
        }
    }

}
```
2. 中序\
首先递归地访问左子树，然后访问根节点，最后访问右子树。
```
void inOrder(Node *node){
    if(node){
        inOrder(node->left);
        std::cout<<node->value;
        inOrder(node->right);
    }
}
void inOrder(Node *node){
    if(node==NULL){
        return;
    }
    std::stack<Node *> nstack;
    Node *temp=node;
    while(temp||!nstack.empty()){
        if(temp){
            nstack.push(temp);
            temp=temp->left;
        }
        else{
            temp=nstack.top();
            std::cout<<temp->value;
            nstack.pop();
            temp=temp->right;
        }
    }
}
```
3. 后序\
先递归地访问左子树，然后访问右子树，最后访问根节点。
```
void posOrder(Node *node){
    if(node){
        posOrder(node->left);
        posOrder(node->right);
        std::cout<<node->value;
    }
}
//后序遍历非递归实现
void posOrder1(Node *node){
    if(node==NULL){
        return;
    }
    std::stack<Node *> nstack1, nstack2;
    nstack1.push(node);
    while (!nstack1.empty()){
        Node *temp=nstack1.top();
        nstack1.pop();
        nstack2.push(temp);
        if(temp->left)
            nstack1.push(temp->left);
        if(temp->right)
           nstack1.push(temp->right);
    }
    while(!nstack2.empty())
    {
        std::cout<<nstack2.top()->value;
        nstack2.pop();
    }
}
```
4. 层序遍历
广度优先,从根节点开始，逐层访问树中的每个节点。
```
void broadOrder(Node *node){
    if(!node){
        return;
    }
    std::queue<Node *> qnodes;
    qnodes.push(node);
    while(!qnodes.empty()){
        Node * temp=qnodes.front();
        std::cout<<temp->value;
        qnodes.pop();
        if(temp->left){
            qnodes.push(temp->left);
        }
        if(temp->right){
            qnodes.push(temp->right);
        }

    }
}
```
## 增删改查
1. 增
```
void broadOrder(Node *node){
    if(!node){
        return;
    }
    std::queue<Node *> qnodes;
    qnodes.push(node);
    while(!qnodes.empty()){
        Node * temp=qnodes.front();
        std::cout<<temp->value;
        qnodes.pop();
        if(temp->left){
            qnodes.push(temp->left);
        }
        if(temp->right){
            qnodes.push(temp->right);
        }

    }
}
```
2. 删
```
void Remove(BST* &tree, int data)
{
	if (tree != NULL)
	{
		if (data < tree->data)
			Remove(tree->leftChild, data);
		else if (data > tree->data)
			Remove(tree->rightChild, data);
		else if (tree->leftChild != NULL)
		{
			tree->data = FindMax(tree->leftChild);
			Remove(tree->leftChild, tree->data);
		}
		else if (tree->rightChild != NULL)
		{
			tree->data = FindMin(tree->rightChild);
			Remove(tree->rightChild, tree->data);
		}
		else
		{
			BST* node = tree;
			tree = tree->leftChild ? tree->leftChild : tree->rightChild;
			delete node;
			node = NULL;
		}

	}
}
```
3. 查
```
bool isExist(BST* tree, int data)
{
	if (tree == NULL)
		return false;
	if (tree->data == data)
		return true;
	else if (tree->data > data)
		isExist(tree->leftChild, data);
	else
		isExist(tree->rightChild, data);
}
```
4. 改
```
void change(BST* &tree, int data，int new_data)
{
	if (tree == NULL)
		return ;
	if (tree->data == data){
        tree->data=new_data;
        return;
    }		
	else if (tree->data > data)
		isExist(tree->leftChild, data,new_data);
	else
		isExist(tree->rightChild, data,new_data);
}
```