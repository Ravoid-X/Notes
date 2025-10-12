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