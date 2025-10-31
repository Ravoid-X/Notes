## 定义
其中的各对象按线性顺序排列，顺序是由各个对象里的指针决定。
## 创建节点
```
struct ListNode{
	int data;
	ListNode *next;
}listnode;
```
## 创捷链表
```
ListNode* ListCreate(int n){
	ListNode *head,*p1,*end;
	head=new ListNode;
	end=head;
	for(int i=0;i<n;i++){
		p1=new ListNode;
		p1->data=i;
		end->next=p1;
		end=p1;
	}
	end->next=NULL; //链表的尾部next指向空
	return head;
}
```
## 遍历链表
```
void print(ListNode *head){
	ListNode *temp;
	temp=head->next;
	while(temp!=NULL){
		cout<<temp->data<<' ';
		temp=temp->next;
	}
}
```
## 修改节点
```
void ListChange(ListNode *head,int n,int temp){
	ListNode *p=head;
	for(int i=0;i<n&&p!=NULL;i++){
		p=p->next;
	}
	if(p!=NULL){
		p->data=temp;
	}
	else cout<<"找不到节点/(ㄒoㄒ)/~~"<<endl;
}
```
## 插入节点
```
ListNode* ListInsert(ListNode *head,int n,int data){
	ListNode *p=head;
	ListNode *in=new ListNode;
	for(int i=0;i<n&&p!=NULL;i++){
		p=p->next;
	}
	if(p!=NULL){
		in->data=data;
		in->next=p->next;
		p->next=in;
	}
	else cout<<"找不到/(ㄒoㄒ)/~~"<<endl;
}
```
## 删除节点
```
void ListDelete(ListNode *head,int n){
	ListNode *p=head,*p1=head;
	for(int i=0;i<n&&p!=NULL;i++){
		p1=p;
		p=p->next;
	}
	if(p!=NULL){
		p1->next=p->next;
		delete p;
	}
	else cout<<"找不到节点/(ㄒoㄒ)/~~"<<endl;
}
```