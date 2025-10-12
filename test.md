<img src="../../../pic/C-Lang/C++" style="width:600px;padding:10px;"/>
```
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
        }}
```