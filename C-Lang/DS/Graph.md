## 定义
G 是由两个集合 V 和 E 组成，记为 G = （V，E），其中 V 代表顶点，E 代表边，如下图。
<img src="../../Pic/C-Lang/DS/graph-difinition.png" style="width:400px;padding:10px;"/>

## 概念
1. 简单图：不存在顶点到其自身的边，且同⼀条边没有重复出现。
2. 无向/有向图：代表边的顶点偶对是无序/有序的。
3. 完全图：图中的每两个顶点之间，都存在一条边。
4. 端点和邻接点：若存在一条边（i，j），则称顶点 i 和顶点 j 为该边的两个端点，并称它们互为邻接点。在有向图中顶点 i 为起点。顶点 j 为终点。
5. 顶点的度：顶点所具有的边的数目。在有向图中，顶点 v 的度就分为入度和出度。
6. 回路/环：一条路径上的开始点与结束点为同一个顶点，则称此路为回路或环。\
（1）欧拉环路：经过图中各边一次且恰好一次的环路。如{C，A，B，A，D，C，D，B，C} \
（2）哈密尔顿环路：经过图中的各顶点一次且恰好一次的环路。如{C，A，B，D，C}
<img src="../../Pic/C-Lang/DS/graph-loop.png" style="width:400px;padding:10px;"/>

7. 无向图中若从顶点 i 到顶点 j 有路径，则称这两个顶点是连通的。如果图 G 中任意两个顶点都连通，则称 G 为连通图，否则称为非连通图。无向图 G 中的极大连通子图称为 G 的连通分量。
8. 有向图中若从顶点 i 到顶点 j 有路径，则称从顶点 i 到顶点 j 是连通的。若图中的任意两个顶点都连通，即从顶点 i 到顶点 j 和从顶点 j 到顶点 i 都存在路径，则称其为强连通图。有向图 G 中的极大强连通子图称为 G 的强连通分量。
9. 当一个图接近完全图的时候，称之为稠密图；相反，当一个图含有较少的边数，则称之为稀疏图。
通常认为边小于 nlogn（n 是顶点的个数）为稀疏图。
10. 边可以对应一个数，称为权。边上带有权的图称为带权图，也称之为网。
11. 有向图中，有向边是由两个顶点组成的有序对，通常用尖括号表示，有向边又称为弧。弧的始点称为弧尾（起始点），弧的终点称为弧头（终端点）。

## 存储
1. 邻接矩阵\
两个数组来表示，一维数组存储图中的顶点信息，二维数组（我们将这个数组称之为邻接矩阵）存储图中的边的信息。\
<img src="../../Pic/C-Lang/DS/graph-adj-matrix.png" style="width:400px;padding:10px;"/>

```
#define MaxVertices 100	//假设包含的最大结点数
#define MaxWeight -1	//假设两点不邻接的正无穷值
//定义结点
struct AdjMarix {
	int Vertices[MaxVertices];	//存储结点信息
	int Edge[MaxVertices][MaxVertices] = { 0 };	//存储每条边的权值
	int numV;	//当前顶点的个数
	int numE;	//当前边的个数
};
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
//遍历图
void ShowGraph(AdjMarix *G) {
	for (int i = 0; i < G->numV; i++) {
		for (int j = 0; j < G->numV; j++) {
			cout << G->Edge[i][j] << " ";
		}
		cout << endl;
	}
}
```
2. 邻接表\
空间浪费问题。尤其是边数相对比较少的稀疏图。可把数组与链表结合在一起来存储，即邻接表。数组来存储所有结点信息，然后每个结点有指向其他结点的指针。\
邻接表是用来存储出度的指针，而逆邻接表就是用来存储入度的指针。\
<img src="../../Pic/C-Lang/DS/graph-adj-table.png" style="width:400px;padding:10px;"/>

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
//创建图
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
//遍历图
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
3. 十字链表\
将邻接表和逆邻接表结合，直接用一个数组表示。\
<img src="../../Pic/C-Lang/DS/graph-adj-cross-link-list1.png" style="width:400px;padding:10px;"/>
tailVex 表示该弧的弧尾顶点在顶点数组中的位置，headVex 表示该弧的弧头顶点在顶点数组中的位置。hLink 则表示指向弧头相同的下一条弧，tLink 则表示指向弧尾相同的下一条弧。我们这里定义的是没有箭头的是弧尾，有箭头的是弧头。
<img src="../../Pic/C-Lang/DS/graph-adj-cross-link-list2.png" style="width:400px;padding:10px;"/>
headLink 把所有弧头相同的边连了起来，比如指向结点 V0 的边有两条，分别是 V1 -> V0 和 V3 -> V0，所以从 V0 的 firstIn 连出去，先指向第一条边 1 -> 0 ，再将 1 -> 0 的 headLink 指向下一条也是指向 V0 的边。同样，V1 和 V3 的 firstIn 也是一样，V2 没有连是因为没有指向 V2 的边。
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
//创建图
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
//遍历图
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
4. 邻接多重表\
十字链表对于边的改动十分麻烦，就有了邻接多重表。它在十字链表的边集定义上做了一些改动，同时在顶点数组中只用保留一个指针。
<img src="../../Pic/C-Lang/DS/graph-adj-multi-table1.png" style="width:400px;padding:10px;"/>
其中 iVex 和 jVex 是与某条边依附的两个顶点在顶点表中的下标。iLink 指向依附顶点 iVex 的下一条边，jLink 指向依附顶点 jVex 的下一条边。
<img src="../../Pic/C-Lang/DS/graph-adj-multi-table2.png" style="width:400px;padding:10px;"/>
这每条边中每个结点后面的指针其实就是指向下一个包含该结点的边。
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
//创建图
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
//遍历图
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