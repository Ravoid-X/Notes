## 邻接矩阵
### 定义结点
```
#define MaxVertices 100	//假设包含的最大结点数
#define MaxWeight -1	//假设两点不邻接的正无穷值
struct AdjMarix {
	int Vertices[MaxVertices];	//存储结点信息
	int Edge[MaxVertices][MaxVertices] = { 0 };	//存储每条边的权值
	int numV;	//当前顶点的个数
	int numE;	//当前边的个数
};
```
### 创建
```
void CreatGraph(AdjMarix *G) {
	int vi, vj, w;
	cout << "请输入顶点数量：" << endl;
	cin >> G->numV;
	cout << "请输入顶点信息：" << endl;
	//输入结点的编号并初始化
	for (int i = 0; i < G->numV; i++) {
		cin >> vi;
		G->Vertices[i] = vi;
		G->Edge[i][i] = MaxWeight;
	}
	cout << "请输入边的数量：" << endl;
	cin >> G->numE;
	cout << "请输入边的信息：" << endl;
	for (int i = 0; i < G->numE; i++) {
		cin >> vi >> vj >> w;
		G->Edge[vi - 1][vj - 1] = w;
		//G->Edge[vj-1][vi-1]=w; 无向图需要再加上这一句
	}
}
```
### 遍历
```
void ShowGraph(AdjMarix *G) {
	for (int i = 0; i < G->numV; i++) {
		for (int j = 0; j < G->numV; j++) {
			cout << G->Edge[i][j] << " ";
		}
		cout << endl;
	}
}
```
## 邻接表
### 定义
```
#define MaxVertices 100
//定义结点
struct VertexNode {
	int data;	//结点编号
	int weight = 0;	//指向下一个结点的边的权值
	VertexNode *next = NULL;
};
//定义邻接表
struct GraphAdjList {
	VertexNode *AdjList[MaxVertices];	//存储所有结点
	int numV, numE;
};
```
### 创建
```
void CreatGraph(GraphAdjList &G) {
	int vi, vj, w;
	cout << "请输入顶点数：" << endl;
	cin >> G.numV;
	cout << "请输入顶点信息：" << endl;
	for (int i = 0; i < G.numV; i++) {
		cin >> vi;
		VertexNode *new_node = new VertexNode;
		new_node->data = vi;
		G.AdjList[i] = new_node;
	}
	cout << "请输入边的数量：" << endl;
	cin >> G.numE;
	cout << "请输入边的信息：" << endl;
	for (int i = 0; i < G.numE; i++) {
		cin >> vi >> vj >> w;
		//找到邻接表中对应结点的位置，往其中链表插入对应边
		for (int j = 0; j < G.numV; j++) {
			if (vi == G.AdjList[j]->data) {
				VertexNode *temp = G.AdjList[j];
				//这里用的是尾插法
				while (temp->next != NULL) {
					temp = temp->next;
				}
				VertexNode *newEdge = new VertexNode;
				newEdge->data = vj;
				newEdge->weight = w;
				temp->next = newEdge;
				break;
			}
		}
	}
}
```
### 遍历
```
void showGraph(GraphAdjList &G) {
	for (int i = 0; i < G.numV; i++) {
		VertexNode *temp = G.AdjList[i]->next;
		int vi = G.AdjList[i]->data;
		cout << "顶点" << vi << "的边有：" << endl;
		if (temp == NULL) {
			cout << "无" << endl;
		}
		while (temp != NULL) {
			cout << vi << "->" << temp->data << " 权值=" << temp->weight << endl;
			temp = temp->next;
		}
	}
}
```
## 邻接数组
```
for(unsigned short i=0;i<m;++i){
	unsigned short u,v;
	cin>>u>>v;
	adj[u].push_back(v);
	adj[v].push_back(u);
}
```
## 十字链表
### 定义
```
//边集定义
struct ArcBox {
	int headvex, tailvex;	//对应弧头和弧尾的下标
	ArcBox *hlink, * tlink;	//分别指向弧头相同和弧尾相同的下标
};
//顶点定义
struct VexNode {
	int data;
	ArcBox *firstin, *firstout;
};
//定义图
struct OLGraph {
	VexNode xlist[100];
	int vexnum, arcnum;
};
//找到顶点在数组中的位置
int Location(OLGraph *G, int key) {
	//遍历每个顶点
	for (int i = 0; i < G->vexnum; i++) {
		if (key == G->xlist[i].data) {
			return i;
		}
	}
}
```
### 创建
```
void CreatGraph(OLGraph *G) {
	int vi, vj, xi, xj;

	cout << "请输入顶点数：" << endl;
	cin >> G->vexnum;
	cout << "请输入顶点信息：" << endl;
	for (int i = 0; i < G->vexnum; i++) {
		cin >> G->xlist[i].data;
		G->xlist[i].firstin = NULL;
		G->xlist[i].firstout = NULL;
	}
	cout << "请输入弧数：" << endl;
	cin >> G->arcnum;
	cout << "请输入弧的信息：" << endl;
	for (int i = 0; i < G->arcnum; i++) {
		cin >> vi >> vj;
		xi = Location(G, vi);
		xj = Location(G, vj);
		ArcBox *new_node = new ArcBox;
		new_node->headvex = xj;		//headvex是存有箭头的弧头
		new_node->tailvex = xi;		//tailvex是存没有箭头的弧尾
		new_node->hlink = G->xlist[xj].firstin;		//将hlink指向弧头也是xj的边
		new_node->tlink = G->xlist[xi].firstout;	//将tlink指向弧尾也是xi的边
		G->xlist[xj].firstin = G->xlist[xi].firstout = new_node;	//更新firstin和firstout
	}
}
```
### 遍历
```
void ShowGraph(OLGraph *G) {
	for (int i = 0; i < G->vexnum; i++) {
		int vi = G->xlist[i].data;
		cout << "与结点" << vi << "相连的结点有：" << endl;

		ArcBox *temp = G->xlist[i].firstout;
		cout << "【出度】" << endl;
		if (temp == NULL) {
			cout << "暂无";
		}
		while (temp != NULL) {
			cout << vi << "->" << G->xlist[temp->headvex].data << "   ";
			temp = temp->tlink;
		}

		temp = G->xlist[i].firstin;
		cout << endl << "【入度】" << endl;
		if (temp == NULL) {
			cout << "暂无";
		}
		while (temp != NULL) {
			cout << G->xlist[temp->tailvex].data << "->" << vi << "   ";
			temp = temp->hlink;
		}
		cout << endl;
	}
}
```
## 邻接多重表
### 定义
```
//定义边集
struct ArcNode {
	int ivex, jvex;
	ArcNode *vi, * vj;
};
//定义顶点
struct VexNode {
	int data;
	ArcNode *firstEdge;
};
//定义图
struct Graph {
	VexNode Dvex[100];
	int vexnum, arcnum;
};
//找到顶点在数组中的下标
int Location(Graph *G, int key) {
	for (int i = 0; i < G->vexnum; i++) {
		if (G->Dvex[i].data == key) {
			return i;
		}
	}
}
```
### 创建
```
void CreatGraph(Graph *G) {
	int vi, vj;

	cout << "请输入顶点数：" << endl;
	cin >> G->vexnum;
	cout << "请输入顶点信息：" << endl;
	for (int i = 0; i < G->vexnum; i++) {
		cin >> G->Dvex[i].data;
		G->Dvex[i].firstEdge = NULL;
	}
	cout << "请输入边数：" << endl;
	cin >> G->arcnum;
	cout << "请输入边的信息：" << endl;
	for (int i = 0; i < G->arcnum; i++) {
		cin >> vi >> vj;
		ArcNode *new_edge = new ArcNode;
		int xi = Location(G, vi);
		int xj = Location(G, vj);
		new_edge->ivex = xi;	//记录弧尾即不带箭头的一边
		new_edge->jvex = xj;	//记录弧头即带箭头的一边
		new_edge->vi = G->Dvex[xi].firstEdge;	//指向包含vi的边
		new_edge->vj = G->Dvex[xj].firstEdge;	//指向包含vj的边
		G->Dvex[xi].firstEdge = G->Dvex[xj].firstEdge = new_edge;	//更新firstEdge
	}
}
```
### 遍历
```
void showGraph(Graph *G) {
	for (int i = 0; i < G->vexnum; i++) {
		int vi = G->Dvex[i].data;
		ArcNode *temp = G->Dvex[i].firstEdge;
		cout << "与结点" << vi << "相连的结点有：" << endl;
		if (temp == NULL) {
			cout << "暂无";
		}
		while (temp != NULL) {
			int vj, flag = 0;
			if (G->Dvex[temp->ivex].data == vi) {
				vj = G->Dvex[temp->jvex].data;
				flag = 1;
			} else {
				vj = G->Dvex[temp->ivex].data;
			}
			cout << "结点" << vj << "  ";
			if (flag == 1) {
				temp = temp->vi;
			} else {
				temp = temp->vj;
			}
		}
		cout << endl;
	}
}
```