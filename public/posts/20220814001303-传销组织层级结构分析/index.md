# 传销组织层级结构分析


前几天一个网友发了一个传销组织的数据，他想求每个层级的人数。

{{< figure src="https://tujiabing-1258622425.cos.ap-chengdu.myqcloud.com/image_2022-07-31-22-43-14.png" caption="<span class=\"figure-number\">Figure 1: </span>图1" >}}

这两天正好学了`networkx`，我们来看如果用网络分析解决这个问题。

{{< figure src="https://tujiabing-1258622425.cos.ap-chengdu.myqcloud.com/image_2022-07-31-22-44-58.png" caption="<span class=\"figure-number\">Figure 2: </span>图2" >}}

这是整个推荐关系的可视化网络图。其中正红色的点为根节点。

下面我们一步一步来解决。


## 读取数据并创建网络 {#读取数据并创建网络}

我们使用`pandas`读取 excel 数据，并用`nx.from_pandas_edgelist(df,source,target,edge_attr,create_using)`函数来创建一个图=G=。

这个函数是根据边数据来创建图，其中：

-   source:df中表示边起始的列名（推荐人）。
-   target:df中表示边目标的列名（被推荐人）。
-   edge\\_attr:df中表示边属性的列名（如权重，颜色，大小等）。
-   create\\_using:表示创建什么类型的图，无向图，有向图等。这里我们使用有向图=DiGraph=,因为推荐关系是有方向的。

<!--listend-->

{{< highlight python >}}
import networkx as nx
import pandas as pd

# 读取数据创建图
df = pd.read_excel('~/传销原始数据.xlsx')
df = df.loc[:, ['推荐人ID', '被推荐人ID']]
df.columns = ['source', 'target']
G = nx.from_pandas_edgelist(df, 'source', 'target', create_using=nx.DiGraph())
{{< /highlight >}}

当然这样创建的图=G=只是一个类的实例化，并不一张真正可视化的图。如果你想可视化它，可以使用=pyvis=包进行，它可以生成一个可交互的网络图。

{{< highlight python >}}
from pyvis.network import Network
nt = Network('650px', '1250px', directed=True)
nt.from_nx(G)
nt.show('test.html')
{{< /highlight >}}

这将会生成如图 2 所示的网络图。


## 找到根节点 {#找到根节点}

考虑到这个网络是一个传销组织，那么正常情况下应该是有个唯一的根节点，整个组织类似树状结构。

我们先得找到这个根节点，怎么找呢？

这就需要先引一个图论中的概念**度**，度的意思就是一个节点的相邻节点的数量。

{{< figure src="https://tujiabing-1258622425.cos.ap-chengdu.myqcloud.com/image_2022-07-31-23-00-30.png" caption="<span class=\"figure-number\">Figure 3: </span>图3" >}}

如图 3 所示，如果不考虑边的方向，那点节点 1 有4个相邻节点（有边相连），那么节点 1 的度就是 4 。

即`degree=4`。

但是这是一个有向图，就会分成`in_degree`和 `out_degree` 两种度。

那么我们要找到根结点，只需要去找`in_degree==0`的节点就是根节点，同理`out_degree==0`的节点为末级节点。

因此，我们写代码：

{{< highlight python >}}
top_nodes = [n for n, d in G.in_degree() if d == 0]
print('root node:', top_nodes)
{{< /highlight >}}

可以计算出根结点为：

{{< highlight python >}}
[0]
{{< /highlight >}}

{{< figure src="https://tujiabing-1258622425.cos.ap-chengdu.myqcloud.com/image_2022-07-31-23-05-56.png" caption="<span class=\"figure-number\">Figure 4: </span>图4" >}}

如图 4 所示，我们可以找到图中的根节点，它就是这个网络的头目。


## 计算网络层级关系 {#计算网络层级关系}

为了计算网络层级关系，这里我们需要引入一个概念\*距离\*，也就是两个节点之间的最短路径长度。

{{< figure src="https://tujiabing-1258622425.cos.ap-chengdu.myqcloud.com/image_2022-07-31-23-13-15.png" caption="<span class=\"figure-number\">Figure 5: </span>图5" >}}

对于节点 1 到节点 4 的距离为 3 ，因为两条路径可以从节点 1 到达节点 3 ：

-   [1, 2, 3, 4]
-   [1, 2, 5, 4]

这两条路径最短的距离就是 3 。

在`networkx`库中有个函数`nx.shortest_path_length(G,source,target)`可以求出节点`source`和 `target` 之间的距离。

如果省略`target`参数，就可以求出`source`下所有节点与`source`之间的距离。

因此，我们只需要用=nx.shortest_path_length(G, 0)=就可以求出=根节点0=下的所有节点的距离，也就是\*网络层级\*。

{{< highlight python >}}
level = nx.shortest_path_length(G, 0)
nx.set_node_attributes(G, level, 'level')
{{< /highlight >}}

`level` 的值是下面这样的=节点:距离=的字典，可以看到一共 32 个层级。

{{< highlight python >}}
{0:0,2576:1,..., 5659: 32}
{{< /highlight >}}

我们求出了所有子节点到根节点的距离`level`列表，用`set_node_attributes()`函数给每个节点添加一个层级属性。

下面，我们只需要将`level`列表，统计出 1-32 层级中分别有哪些节点即可。

{{< highlight python >}}
# 显示层级
data = {}
for n, l in level.items():
    if l in data.keys():
        nodes = data[l]
        nodes.append(n)
        data[l] = nodes
    else:
        data[l] = [n]
# 打印前10层节点
for l, n in data.items():
    if l < 10:
        print(l, n)
{{< /highlight >}}

这里我们展示前 10 层级对应的哪些节点：

{{< figure src="https://tujiabing-1258622425.cos.ap-chengdu.myqcloud.com/image_2022-07-31-23-26-39.png" >}}

当然，我们将上面`print(l,n)`替换成`print(l,len(n))`，就可以看到每一层级对应的节点数量。

{{< figure src="https://tujiabing-1258622425.cos.ap-chengdu.myqcloud.com/image_2022-07-31-23-29-06.png" caption="<span class=\"figure-number\">Figure 6: </span>图6" >}}


## 下线前10的节点 {#下线前10的节点}

我们知道=度=表示了相邻节点数量，那么度值最大的 10 个，也就是下线数最大的 10 个。

{{< highlight python >}}
degrees = G.out_degree()
top_degree_nodes = sorted(degrees, key=lambda x: x[1], reverse=True)[:10]
print(top_degree_nodes)
{{< /highlight >}}

计算结果：

{{< highlight python >}}
[(2828, 264), (0, 115), (2700, 86), (2833, 65), (2999, 55), (2560, 53), (2574, 42), (3021, 37), (2651, 36), (2834, 31)]
{{< /highlight >}}

可以看到下线最多的节点是`节点2828`有 264 个下线，第二是`根节点0`有 115 个下线。

{{< figure src="https://tujiabing-1258622425.cos.ap-chengdu.myqcloud.com/image_2022-07-31-23-33-59.png" caption="<span class=\"figure-number\">Figure 7: </span>图7" >}}

这些节点表现在图中就是像水母一样的中心节点。


## 最大介绍top10 {#最大介绍top10}

除了通过=度=来衡量一个节点是否为关键节点外，我们还可以通过介数来衡量。

如图 8 所示，根节点 0 传递到节点1,节点2,节点3....

其中节点2,节点 5 的度非常小，分别为 2 和1,但是如果少了他们的话，后面整个网络就断了。

介数就是表示网络中群体与群体之间的中间人角色，现实生活中如果度数大的是黄牛，那么这个介数的中间人就是给黄牛提供渠道的关键人物。

{{< figure src="https://tujiabing-1258622425.cos.ap-chengdu.myqcloud.com/image_2022-07-31-23-37-04.png" caption="<span class=\"figure-number\">Figure 8: </span>图8" >}}

我们在图中将前 10 大中介点标记成了绿色，方便查看。


## 完整代码 {#完整代码}

以上分析的完整代码：

{{< highlight python >}}
import networkx as nx
from pyvis.network import Network
import pandas as pd

# 读取数据创建图
df = pd.read_excel('~/传销原始数据.xlsx')
df = df.loc[:, ['推荐人ID', '被推荐人ID']]
df.columns = ['source', 'target']
G = nx.from_pandas_edgelist(df, 'source', 'target', create_using=nx.DiGraph())

# 求根节点
top_nodes = [n for n, d in G.in_degree() if d == 0]
print('root node:', top_nodes)

# 节点层级
level = nx.shortest_path_length(G, 0)
nx.set_node_attributes(G, level, 'level')

# 显示层级
data = {}
for n, l in level.items():
    if l in data.keys():
        nodes = data[l]
        nodes.append(n)
        data[l] = nodes
    else:
        data[l] = [n]
# 打印前10层节点
for l, n in data.items():
    if l < 10:
        print(l, len(n))

# 打印前10大度节点
degrees = G.out_degree()
top_degree_nodes = sorted(degrees, key=lambda x: x[1], reverse=True)[:10]
print(top_degree_nodes)

# 给节点添加属性
for node in G.nodes:
    G.nodes[node]['title'] = str(node)
    level = G.nodes[node]['level']
    # 给节点添加大小属于
    G.nodes[node]['value'] = 32 - level
    # 第一、二、三层节点添加颜色
    if level == 0:
        G.nodes[node]['color'] = 'red'
    elif level == 1:
        G.nodes[node]['color'] = 'fuchsia'
    elif level == 2:
        G.nodes[node]['color'] = 'purple'
# 中介点
center = nx.betweenness_centrality(G)
center_tops = sorted(center.items(), key=lambda x: x[1], reverse=True)[:10]
# 给前10大中介点添加颜色
for node in center_tops:
    G.nodes[node[0]]['color'] = 'teal'

nx.write_gexf(G, 'test.gexf')
nt = Network('650px', '1250px', directed=True)
nt.from_nx(G)
nt.show('test.html')
{{< /highlight >}}


## 结语 {#结语}

networkx是复杂网络分析的利器，搭配上可视化库 pyvis ，可以简单几行代码完成分析和可视化。

