---
layout: post
title:  "图的算法"
date:   2019-11-9
categories: 算法和数据结构
tags: DataStructure Graph Dijkstra
mathjax: true
author: wzx
---

- 目录
{:toc}

最短路径与最小生成树算法



## 最短路径

### *Dijkstra*算法
> 戴克斯爪，听起来很专业的读法

*Dijkstra* 利用**贪心**的思想，求出所有顶点对某一源点的[单源最短路径](https://baike.baidu.com/item/%E5%8D%95%E6%BA%90%E6%9C%80%E7%9F%AD%E8%B7%AF%E5%BE%84)。将顶点分为两个集合，一个为已找到最短路径的顶点的集合 $U$ ，另一个为尚未确定最短路径的顶点的集合 $V$ 。重复做以下步骤
1. 从 $V$ 取出路径长度最短的顶点 $v_i$
2. 遍历 $V$ 中的其他顶点 $v_j$，**比较 $v_j$ 原来的路径长度与经过 $v_i$ 这个中间点之后的长度**，取更小的路径更新 $v_j$ 的路径长度

之所以取出 $V$ 中路径长度最短的顶点 $v_i$ ，是因为集合 $V$ 中没有能让 $v_i$ 变小的中间点**，所以这个 $v_i$ 的最短路径就是当前的路径

很明显，如果存在**负权值**，[某些路径将会计算错误](https://blog.csdn.net/sugarbliss/article/details/86498323)。*Dijkstra*算法不支持负权值的情况

我们看下面这个例子

![]({{ site.url }}/assets/img/2019-11-9-1.png){:height="150"}

要求所有顶点到 $v_0$ 的最短距离

![]({{ site.url }}/assets/img/2019-11-9-2.png){:height="250"}

如图所示
1. 取出拥有最短路径的 $v_2$ ，通过 $v_2$ 更新了 $v_3$ ，$v_2$ 求解完毕
2. 取出拥有最短路径的 $v_3$ ，通过 $v_2$ 更新了 $v_1$ 和 $v_4$，$v_3$ 求解完毕
3. 取出拥有最短路径的 $v_1$ ，通过 $v_1$ 更新了 $v_2$ 和 $v_4$，$v_1$ 求解完毕
3. 取出拥有最短路径的 $v_4$ ，通过 $v_4$ 更新了 $v_3$，$v_4$ 求解完毕

利用[最小堆]({% post_url 2019-10-25-min-heap %})实现
```c++
void dijkstra(Graph& graph, int s, Dist dist[]) {
    // 初始化
	dist = new Dist[graph.verticesNum()];
	for (int i = 0; i < graph.verticesNum(); i++) {
		graph.setVisited(i, false);
		dist[i].length = INT_MAX;
		dist[i].pre = s;
	}

	// 最小堆
	priority_queue<Dist, vector<Dist>, cmp> heap;
	// 放入源点
	dist[s].length = 0;
	heap.push(dist[s]);

	for (int i = 0; i < graph.verticesNum(); i++) {
		Dist d;

		bool found = false;
		while (!heap.empty()) {
			// 找到拥有最短路径的点，取出
			d = heap.top();
			heap.pop();
			// 防止访问旧顶点
			if (!graph.isVisited(d.index)) {
				found = true;
				break;
			}
		}
		// 没有与源点连通的顶点了
		if (!found) {
			break;
		}

		// 更新
		int fromVertex = d.index;
		graph.setVisited(fromVertex, true);
		for (Edge* edge = graph.firstEdge(fromVertex); edge != NULL; edge = graph.nextEdge(edge)) {
			// 相邻点到源点的长度小于两边之和
			int toVertex = graph.ToVertex(edge);
			if (dist[toVertex].length > (dist[fromVertex].length + edge->weight)) {
				dist[toVertex].length = dist[fromVertex].length + edge->weight;
				dist[toVertex].pre = fromVertex;
				heap.push(dist[toVertex]);
				// 这里没有删除旧顶点，因为旧顶点与新顶点拥有相同的索引
				// 但是新顶点总在旧顶点之前取出(比它小)，该索引被标记为visited
				// 所以旧顶点不会被访问到，节省了删除顶点的时间
			}
		}
	}
}
```

有最小堆的性质可知，最短路径顶点删除的时间复杂度为 $O(logn)$ ，入堆的时间复杂度为 $O(logn)$，所以当顶点数为 $E$ ，边数为 $V$ 时，算法的时间复杂度为 $O((E+V)logV)$

### *Floyd*算法
弗洛伊德算法利用动态规划的想法，先保存两点间的直接距离，再用其他顶点作为中间点更新两点的距离。很明显需要通过两个循环嵌套来遍历所有的顶点对，再嵌套一个循环用来遍历所有的中间点。很明显，时间复杂度为 $O(E^3)$

想法比较简单，这里直接给出代码

```c++
void floyd(Graph& graph, Dist** &dist) {
	// 初始化
	dist = new Dist*[graph.verticesNum()];
	for (int i = 0; i < graph.verticesNum(); i++) {
		dist[i] = new Dist[graph.verticesNum()];
	}
	for (int i = 0; i < graph.verticesNum(); i++) {
		for (int j = 0; j < graph.verticesNum(); j++) {
			if (i == j) {
				dist[i][j].length = 0;
				dist[i][j].pre = i;
			} else {
				dist[i][j].length = numeric_limits<int>::max();
				dist[i][j].pre = -1;
			}
		}
	}

	// 更新直接路径
	for (int vex = 0; vex < graph.verticesNum(); vex++) {
		for (Edge* edge = graph.firstEdge(vex); edge != NULL; edge = graph.nextEdge(edge)) {
			dist[vex][graph.toVertex(edge)].length = graph.Weight(edge);
			dist[vex][graph.toVertex(edge)].pre = vex;
		}
	}

	// 用每个顶点作为中间点更新，原两点间的路径
	for (int v = 0; v < graph.verticesNum(); v++) {
		for (int i = 0; i < graph.verticesNum(); i++) {
			for (int j = 0; j < graph.verticesNum(); j++) {
				// 因为将不连通的距离设置为了int最大值
				// 这里是为了防止溢出
				if (dist[i][v].length == numeric_limits<int>::max() ||
					dist[v][j].length == numeric_limits<int>::max()) {
					continue;
				}
				if (dist[i][j].length > dist[i][v].length + dist[v][j].length) {
					dist[i][j].length = dist[i][v].length + dist[v][j].length;
					dist[i][j].pre = dist[v][j].pre;
				}
			}
		}
	}
}
```

## 最小生成树(*minimum-cost spanning* *MST*)
最小生成树指拥有图中所有顶点，并且树的权值总和最小的树
![]({{ site.url }}/assets/img/2019-11-9-3.png){:height="150"}
### *Prim*算法
普里姆算法与 *Dijkstra*算法很类似，是贪心的算法。理解起来很简单，每次往 *MST* 中加入**与MST距离最短的顶点**，直至所有顶点加入。这里的距离最短是指与 *MST* 中某一点的距离最短即可，这就体现了贪心的思想。与*Dijkstra*算法类似地，如果采用最小堆，时间复杂度为 $O((E+V)logV)$

比较简单，这里直接给出代码

```c++
void prim(Graph& graph, int s, Edge* &mst) {
	Dist *dist = new Dist[graph.verticesNum()];
	priority_queue<Dist, vector<Dist>, cmp> heap;
    // 最小生成树数组计数
    int tag = 0;
    mst = new Edge[graph.verticesNum() - 1];

	// 初始化
	for (int i = 0; i < graph.verticesNum(); i++) {
		graph.setVisited(i, false);
		dist[i].index = i;
		dist[i].length = numeric_limits<int>::max();
		dist[i].pre = s;
	}

	dist[s].length = 0;
	graph.setVisited(s, true);
	// 当前放入最小生成树的顶点索引
	int v = s;
	// 已经从s顶点出发了，所以只要加入v-1个点
	for (int i = 0; i < graph.verticesNum() - 1; i++) {
		// 更新其余点到 MST 的距离
		for (Edge* edge = graph.firstEdge(v); edge != NULL; edge = graph.nextEdge(edge)) {
			if (!graph.isVisited(graph.toVertex(edge)) &&
				dist[graph.toVertex(edge)].length > edge->weight) {

				dist[graph.toVertex(edge)].length = edge->weight;
				dist[graph.toVertex(edge)].pre = v;
				// 这里使用了与 dijkstra 中一样的trick
				heap.push(dist[graph.toVertex(edge)]);
			}
		}

		// 找到与MST距离最小的顶点
		bool found = false;
		Dist d;
		while (!heap.empty()) {
			d = heap.top();
			heap.pop();
			// 防止访问旧顶点
			if (!graph.isVisited(d.index)) {
				found = true;
				break;
			}
		}

		// 非连通，有不可达顶点
		if (!found) {
			return;
		}

		v = d.index;
		graph.setVisited(v, true);

		// 放入最小生成树
		mst[tag].adjVex = v;
		mst[tag++].weight = d.length;
	}
}
```

### *Kruskal*算法
克鲁斯克尔算法就与 *Prim*算法如出一辙，但是不同的是，他每次往 *MST* 中加入**全局最小的边**，当然要判断加入边会不会产生回路，这就需要[并查集]({% post_url 2019-11-2-forest %}#并查集union-find)来解决了。如果使用最小堆，时间复杂度为 $O(ElogE)$

比较简单，这里直接给出代码

```c++
void kruskal(Graph& graph, int s, Edge* &mst) {
	UnionFindSet unionFindSet(graph.verticesNum());
	priority_queue<Edge, vector<Edge>, cmp> heap;
	// 最小生成树数组计数
	int tag = 0;
	mst = new Edge[graph.verticesNum() - 1];

	// 所有边放入最小堆
	for (int i = 0; i < graph.verticesNum(); i++) {
		for (Edge* edge = graph.firstEdge(i); edge != NULL; edge = graph.nextEdge(edge)) {
			// 防止重边
			if (graph.fromVertex(edge) < graph.toVertex(edge)) {
				heap.push(*edge);
			}
		}
	}

	// 当放入V-1个边时，说明求解完毕
	while (tag < graph.verticesNum() - 1) {
		// 非连通，有不可达顶点
		if (heap.empty()) {
			return;
		}

		Edge edge = heap.top();
		heap.pop();
		int from = graph.fromVertex(&edge);
		int to = graph.toVertex(&edge);
		// 边的两个顶点不在一个等价类，就不会产生回路的情况
		if (unionFindSet.find(from) != unionFindSet.find(to)) {
			unionFindSet.aUnion(from, to);
			mst[tag++] = edge;
		}
	}
}
```

## REFERENCE
[1][最短路径](https://www.coursera.org/learn/shuju-jiegou-suanfa/lecture/IkT4p/zui-duan-lu-jing) - 北京大学[EB/OL]. Coursera.  
[2][最小生成树](https://www.coursera.org/learn/shuju-jiegou-suanfa/lecture/uAuks/zui-xiao-sheng-cheng-shu) - 北京大学[EB/OL]. Coursera.
