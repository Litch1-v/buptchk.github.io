---
layout: post
title: 数据结构整理之图
tags: 数据结构

---
趁着期末考试赶紧把不懂的地方补一补，感觉图是比较难的一章

1. 一些名词解释

   度：一个顶点的度  说的是与这个顶点相连的边的数目

   连通分量：对**无向图**来说，其中的极大连通子图

   强连通图：对**有向图**来说，从vi到vj和从vj到vi都存在路径

   生成树：极小连通子图，含有图中全部顶点，但只有足以构成一棵树的n-1条边 

   **特点**：多一条边就一定会生成环 ，对一个图来说，少于n-1条边则不连通，对于n-1条边则一定有环，等于n-1条边不一定是生成树

2. 图的存储结构

   1. 邻接矩阵

      **对于无向图**：顶点vi的度是邻接矩阵很重第i行（或第i列 都可以）的元素之和

      **对于有向图**：第i行之和记录vi的出度，若vi是没有通向vj的**直接**路线，则邻接矩阵中的gij计为∞，第i列之和记录vi的入度

   2. 邻接表

      对每一个顶点都建立一个单链表，**对于无向图**第i个单链表中的结点表示与vi**直接相连**的顶点，**对于有向图**为从vi指出相连的顶点（出度），（想求入度就必须遍历）

   3. 十字链表（有向图）有两种结点，弧结点和顶点结点

      | 弧的尾域的位置（顶点结点） | 头域的位置 | 指向弧头相同的下一条弧 | 指向弧尾相同的下一条弧 | 信息 |
      | -------------------------- | ---------- | ---------------------- | ---------------------- | ---- |
      | tailvex                    | headvex    | hlink                  | tlink                  | info |

      | 数据 | 以该顶点为头的弧 | 以该顶点为尾的弧 |
      | ---- | ---------------- | ---------------- |
      | data | firstin          | firstout         |

   4. 邻接多重表（无向图）弧结点和顶点结点

      | 是否被搜索过的标志 | 这条弧依赖的顶点位置 | 下一条依赖ivex的弧 | 这条弧依赖的另一个顶点的位置 | 下一条依赖jvex的边 | 信息 |
      | ------------------ | -------------------- | ------------------ | ---------------------------- | ------------------ | ---- |
      | mark               | ivex                 | ilink              | jvex                         | jink               | info |

      | 信息 | 第一条依赖于该顶点的边 |
      | ---- | ---------------------- |
      | data | firstedge              |

      **四种存储类型的比较**：当边稀疏（远小于C2n），或和边相关的信息较多时 邻接表比邻接矩阵节省空间

      ​					邻接表方便查找任一顶点的邻接点，但是不好判定是否相连

      ​					十字链表容易求顶点的出度和入度

      ​					构建邻接矩阵的时间复杂度：O（n²+e*n）

      ​					邻接表，信息不为编号时：O（n*e）

      ​					十字邻接表：O（n*e）

3. 深度优先和广度优先算法：

   时间复杂度分析：1. 广度，深度优先算法：每个节点需要遍历一次，有n个节点总时间为O（n），对一个节点，需要查找他的链表，即遍历弧，所以所有弧遍历完的时间为O（e），所以时间复杂度为O（n+e）

   ```c
   void DfS(GraphAdjList GL, int i)//对GL图从i顶点开始进行深度优先遍历//
   {
       EdgeNode *p; //边表节点,存储内容：EdgeNode *next 指向下一个邻接点，以及adjvex，该顶点对应的下标//
       visited[i]=TRUE;//i节以及被访问//
       printf("%c",GL->adjList[i].data)；
       p=GL->adjList[i].firstedge;//adjList是一个头结点表，这里把第i个头结点的邻接节点赋给p//
       while(p){
           if(!visited[p->adjvex])//如果这个邻接节点未被访问，那就调用DFS访问//
               DFS(GL,p->adjvex);//把这个邻接点当成头结点，调用DFS递归遍历//
           p=p->next;//继续遍历i顶点的下一个邻接节点//
       }
   }
   ```

   ```c
   void BFSTraverse(GraphAdjList GL)
   {
       int i;
       EdgeNode *p;
       Queue Q;
       for(i=0;i<GL->numVertexes;i++)
       visited[i]=FALSE;
       InitQueue(&Q);//以上都是初始化操作，需要一个辅助队列//
       for(i=0;i<GL->numVertexes;i++)//对GL图中的每一个结点循环，下面取第i个结点来看//
       {
           if(!visited[i])//如果没有访问过i，那就开始访问//
           {
               visited[i]=TRUE;//i现在已经被访问了//
               printf("%c",GL->adjList[i].data);
               EnQueue(&Q,i);//将这个顶点入列//
               while(!QueueEmpty(Q))//只要队列中有元素，就弹一个出来，开始搜索他的邻接点//
               {
                   DeQueue(&Q,&i);//弹出一个顶点，对这个顶点开始遍历他的邻接点//
                   p=GL->adjList[i].firstedge;//找到这个顶点的邻接点//
                   while(p)
                   {
                       if(!visited[p->adjvex])
                       {
                           visited[p->adjvex]=TRUE;
                           printf("%c",GL->adjList[p->adjvex].data);
                           EnQueue(&Q,p->adjvex);//所有没有访问过的邻接点进队列,接下来就会对这些邻接点进行逐一访问他们的邻接点//
                       }
                       p=p->next;//遍历i顶点的所有邻接点//
                   }
               }
           }
       }
   }
   ```

   

4. 最小生成树

   1.解决的问题：n个城市，需要n-1条边相连，各条边相连有不同的代价，求最小的代价。（通信网问题）

   MST性质:![MSTæ§è´¨çè¯æ - æµæµªè -](http://img20.ph.126.net/jf5EhWzNY6pFr4sUFHxEHg==/3174756262321549001.png) 

   把顶点集合分为两部分，假设已经有一颗生成树，则V和U-V中一定会有一条连通路，则肯定是U和U-V相连中最短的一条路径。

   2. 普里姆算法

      先找到权值最小的边，之后把已经相连的看成整体，不断利用MST性质找权值最小的边，时间复杂度为O（n²）

      `https://blog.csdn.net/jnu_simba/article/details/8869876`

   3. 克鲁斯卡尔算法

      从边出发，寻找权值最小的弧，如果这条弧连接的是两个不同的连通分量，就选中，否则找次小的

      `https://blog.csdn.net/jnu_simba/article/details/8870481`

5. 关节点和重连通分量

   关节点：删去关节点，图被分为多个连通分量 重连通图：没有关节点的图

   判断方法：使用深度优先生成树，若生成树的根有多棵子数，则顶点为关键点2. 若某个节点的v某颗子树中的所有节点均没有指向v的祖先的回边，则v为关节点。算法的时间复杂度仍然为O(n+e)

6. 拓扑排序

   解决问题:工程能否进行的问题（是否存在有向环）

   判断方法：所有节点是否都在拓扑序列中，找拓扑序列的方法：在图中把没有前驱的节点删掉，如果能一直删到没有节点，则不存在有向环，若中途存在所有的节点都有前驱的情况，则存在有向环，时间复杂度：O（n+e）

7. 关键路径

   解决问题：工程最少需要几天能够完成得问题（在没有有向环存在的条件下）

   `https://blog.csdn.net/wang379275614/article/details/13990163`（算法关键：四个特征量）

8. 最短路径

   1. 从某个源点到其余各顶点的最短路径（**迪杰斯特拉算法**）

      假设S为已经求得的最短路径的终点的集合，则下一条最短路径（设其终点为x）或者是弧（v,x）或者是中间只经过S中的顶点而最后达到顶点x的路径。时间复杂度为O（n²）

      ```c
      #define MAXVEX 9
      #define INFINITY 65535
      typedef int Pathmatirx[MAXVEX];//用于存储最短路径下标的数组//
      typedef int ShortPathTable[MAXVEX];//用于存储最短路径长度的数组//
      void ShortestPath_Dijkstra(MGraph G,int v0,Pathmatirx *P,ShortPathTable *D)
      {
          int v,w,k,min;
          int final[MAXVEX];
          for(v=0;v<G.numVertexes;v++)
          {
              final[v]=0;
              (*D)[v]=G.matirx[v0][v];
              (*P)[v]=0;
          }
          (*D)[v0]=0;
          final[v0]=1;
          for(v=1;v<G.numVertexes;v++)
          {
              min=INFINITY;
              for(w=0;w<G.numVertexes;w++)
              {
                  if(!final[w]&&(*D)[w]<min)
                  {
                      k=w;
                      min=(*D)[w];
                  }
              }
              final[k]=1;
              for(w=0;w<G,numVertexes;w++)
              {
                  if(!final[w]&&(*D)[w]<min)
                  {
                      k=w;
                      min=(*D)[w];
                  }
              }
              final[k]=1;
              for(w=0;w<G.numVertexes;w++)
              {
                  if(!final[w]&&(min+G.matirx[k][w]<(*D)[w]))
                  {
                      (*D)[w]= min+G.matirx[k][w];
                      (*P)[w]=k;
                  }
              }
          }
      }
      ```

      

   2. 每一对顶点之间的最短路径

      1. 重复执行迪杰斯特拉算法n次，时间复杂度为O（n3）
      2. 弗洛伊德算法