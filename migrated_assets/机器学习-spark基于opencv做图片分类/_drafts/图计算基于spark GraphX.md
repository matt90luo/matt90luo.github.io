---
title: 图计算基于spark graphX
tags:
---
pregel 分布式下图计算的计算框架 支持的图算法有BFS 最短路径 PageRank 这是一种BSP模型
计算-通信-同步模型
- 输入输出为有向图
- 分成超步
- 以节点为中心计算，超步内每个节点执行自己的任务，执行节点的顺序不确定
- 两个超步之间是通信阶段



## Random-walk computation of similarities between nodes of a graph, with application to collaborative recommendation
1. computing similarities between nodes of a weighted and undirected graph
2. The procedure used to compute similarities is based on a Markov-chain model
3. commute time 概念理解
4. ECDT is abbriviation of Euclidean Commute Time Distance
### 重要点

1. We show that all the introduced quantities can be expressed in closed form in terms of the pseudoinverse of the Laplacian matrix of the graph. This generalizes results obtained by Klein & Randic [40], derived for the ECTD only, based on the electrical equivalence. Since the pseudoinverse of the Laplacian matrix plays a key role and has a nice interpretation in terms of random walk on a graph, we prove some of its properties.
2. We show that the pseudoinverse of the Laplacian matrix of the graph is a valid kernel (a Gram matrix; see for instance [57]). It therefore defines a kernel on a graph and can be interpreted as a similarity measure.
3. We show that the node space of the graph can be projected into a Eu- clidean subspace that approximately preserves the ECTD. This subspace is optimal in the following sense: it keeps as much variance of the projected data as possible (in terms of the ECTD). It is therefore an equivalent of principal components analysis (PCA) and classical multidimensional scal- ing (MDS), in terms of the ECTD. This provides a nice interpretation to the “Fiedler vector”, widely used in graph partitioning.We also show that the ECTD PCA can be viewed as a special regularized Laplacian kernel, as introduced by Smola & Kondor [61].
4. From an experimental point of view, we show that these quantities can be used in the context of collaborative recommendation [18, 30, 33]. Indeed

在工程上对于超大规模的稀疏矩阵计算其伪逆矩阵非常困难


val graph = GraphLoader.edgeListFile(spark.sparkContext, "hdfs:///tmp/manley/graphrank/edge")
    val ranks = graph.pageRank(0.1).vertices
在 16ex 4core *10G 下 耗时 40min
vertices = 170W
0.1 调整为0.01 1h17min
调整为0.001 1h08min


    





## 问题背景
对于任意两个集合如果他们之间的交集不为空, 那么这两个集合可以合并为一个新的集合,
同时定义一个超集合S, 里面任意两个集合之间的交集都是空。那么现在对于待选集合s, 对于其中的每一个元素按照有交集就合并的方式
要找到最终的超集合S

### 方法
在spark环境下核心方法如下
``` scala
def setMerge[A](x: scala.collection.mutable.ListBuffer[Set[A]], y:scala.collection.mutable.ListBuffer[Set[A]]) ={
    for(it <- y){
      if(x.isEmpty) {
        x.append(it)
      } else{
        val trues = x.filter( ts => (ts & it).nonEmpty )
        val merges = if(trues.nonEmpty) trues.reduce(_ ++ _) ++ it else it
        x --= trues
        x += merges
      }
    }
    x
  }
```
相当于模拟把一个一个的元素加入一个已经确定的是集合内任意两两元素的交集为空, 这个方法是ok的, 但是应用在真实数据集上的时候, 出现了所有数据都集中在一起, 没有起到分组的效果
事实上，这种情况也应该预料到, 两个集合只要交集不为空就可以合并，此条件非常容易达到


## 推荐系统基于图的模型
1. 使用二分图表示用户商品的交互关系
2. 定点的相关性取决于下面三个因素
- 两个定点之间的路径数
- 两个定点之间的路劲长度
- 两个定点之间的路劲经过的顶底
3. 相关性高的的一对顶点具有如下性质
- 两个定点之间有很多路径相连
- 连接两个顶点之间的路径长度都比较短
- 连接两个顶点之间的路径不会经过出度比较大的顶点


## 从pagerank算法到personalRank算法
虽然搜索引擎已经发展了很多年，但是其核心却没有太大变化。从本质上说，搜索引擎是一个资料检索系统，搜索引擎拥有一个资料库（具体到这里就是互联网页面），用户提交一个检索条件（例如关键词），搜索引擎返回符合查询条件的资料列表。理论上检索条件可以非常复杂，为了简单起见，我们不妨设检索条件是一至多个以空格分隔的词，而其表达的语义是同时含有这些词的资料（等价于布尔代数的逻辑与）。例如，提交“张洋 博客”，意思就是“给我既含有‘张洋’又含有‘博客’词语的页面”，以下是Google对这条关键词的搜索结果：
1. 搜索引擎的核心难题
上面阐述了一个非常简单的搜索引擎工作框架，虽然现代搜索引擎的具体细节原理要复杂的多，但其本质却与这个简单的模型并无二异。实际Google在上述两点上相比其前辈并无高明之处。其最大的成功是解决了第三个、也是最为困难的问题：如何对查询结果排序。
我们知道Web页面数量非常巨大，所以一个检索的结果条目数量也非常多，例如上面“张洋 博客”的检索返回了超过260万条结果。用户不可能从如此众多的结果中一一查找对自己有用的信息，所以，一个好的搜索引擎必须想办法将“质量”较高的页面排在前面。其实直观上也可以感觉出，在使用搜索引擎时，我们并不太关心页面是否够全（上百万的结果，全不全有什么区别？而且实际上搜索引擎都是取top，并不会真的返回全部结果。），而很关心前一两页是否都是质量较高的页面，是否能满足我们的实际需求。
因此，对搜索结果按重要性合理的排序就成为搜索引擎的最大核心，也是Google最终成功的突破点。
2. 早期搜索引擎的做法
- 不评价
这个看起来可能有点搞笑，但实际上早期很多搜索引擎（甚至包括现在的很多专业领域搜索引擎）根本不评价结果重要性，而是直接按照某自然顺序（例如时间顺序或编号顺序）返回结果。这在结果集比较少的情况下还说得过去，但是一旦结果集变大，用户叫苦不迭，试想让你从几万条质量参差不齐的页面中寻找需要的内容，简直就是一场灾难，这也注定这种方法不可能用于现代的通用搜索引擎。
- 基于检索词的评价
后来，一些搜索引擎引入了基于检索关键词去评价搜索结构重要性的方法，实际上，这类方法如TF-IDF算法在现代搜索引擎中仍在使用，但其已经不是评价质量的唯一指标。完整描述TF-IDF比较繁琐，本文这里用一种更简单的抽象模型描述这种方法。
基于检索词评价的思想非常朴素：和检索词匹配度越高的页面重要性越高。“匹配度”就是要定义的具体度量。一个最直接的想法是关键词出现次数越多的页面匹配度越高。还是搜索“张洋 博客”的例子：假设A页面出现“张洋”5次，“博客”10次；B页面出现“张洋”2次，“博客”8次。于是A页面的匹配度为5 + 10 = 15，B页面为2 + 8 = 10，于是认为A页面的重要性高于B页面。很多朋友可能意识到这里的不合理性：内容较长的网页往往更可能比内容较短的网页关键词出现的次数多。因此，我们可以修改一下算法，用关键词出现次数除以页面总词数，也就是通过关键词占比作为匹配度，这样可以克服上面提到的不合理。
早期一些搜索引擎确实是基于类似的算法评价网页重要性的。这种评价算法看似依据充分、实现直观简单，但却非常容易受到一种叫“Term Spam”的攻击。
- Term Spam 攻击v
其实从搜索引擎出现的那天起，spammer和搜索引擎反作弊的斗法就没有停止过。Spammer是这样一群人——试图通过搜索引擎算法的漏洞来提高目标页面（通常是一些广告页面或垃圾页面）的重要性，使目标页面在搜索结果中排名靠前。
现在假设Google单纯使用关键词占比评价页面重要性，而我想让我的博客在搜索结果中排名更靠前（最好排第一）。那么我可以这么做：在页面中加入一个隐藏的html元素（例如一个div），然后其内容是“张洋”重复一万次。这样，搜索引擎在计算“张洋 博客”的搜索结果时，我的博客关键词占比就会非常大，从而做到排名靠前的效果。更进一步，我甚至可以干扰别的关键词搜索结果，例如我知道现在欧洲杯很火热，我就在我博客的隐藏div里加一万个“欧洲杯”，当有用户搜索欧洲杯时，我的博客就能出现在搜索结果较靠前的位置。这种行为就叫做“Term Spam”。

3. PageRank算法

转移矩阵
$$\begin{matrix} 0 & 1/2 & 0 & 1/2\\ 1/3 & 0 & 0 & 1/2\\ 1/3 & 1/2 & 0 & 0\\ 1/3 & 0 & 1 & 0 \end{matrix}$$

PageRank 算法假设Web页面之间是强连通的, 事实上 Web并不是强连通的 所谓的Dead ends 就是存在这样的节点(页面) 它没有出度
处理Dead Ends的方法如下
- 迭代拿掉图中的Dead Ends节点及Dead Ends节点相关的边（之所以迭代拿掉是因为当目前的Dead Ends被拿掉后，可能会出现一批新的Dead Ends）
直到每个节点都有出度, 对于dead ends的rank 需要以拿掉dead ends逆向顺序反推dead ends的rank

spider traps 以及平滑处理
- 一个节点(页面)有出度但是仅链向自身 它称作spider trap, 如果对这个图进行计算 会发现spider trap 的rank趋近1 其他趋近0

为了克服这种由于矩阵稀疏性和Spider Traps带来的问题，需要对PageRank计算方法进行一个平滑处理，具体做法是加入“心灵转移（teleporting）”。所谓心灵转移，就是我们认为在任何一个页面浏览的用户都有可能以一个极小的概率瞬间转移到另外一个随机页面。当然，这两个页面可能不存在超链接，因此不可能真的直接转移过去，心灵转移只是为了算法需要而强加的一种纯数学意义的概率数字。
$$\large {v}'=(1-\beta)Mv+e\frac{\beta}{N}$$
其中β往往被设置为一个比较小的参数（0.2或更小），e为N维单位向量，加入e的原因是这个公式的前半部分是向量，因此必须将β/N转为向量才能相加。这样，整个计算就变得平滑，因为每次迭代的结果除了依赖转移矩阵外，还依赖一个小概率的心灵转移。


Topic-Sensitive PageRank
1. 确定话题分类
2. 网页topic归属
3. 分topic向量计算
4. 确定用户topic倾向


针对PageRank的Spam攻击与反作弊
上文说过，Spammer和搜索引擎反作弊工程师的斗法从来就没停止过。实际上，只要是算法，就一定有spam方法，不存在无懈可击的排名算法。下面看一下针对PageRank的spam。
Link Spam
回到文章开头的例子，如果我想让我的博客在搜索“张洋 博客”时排名靠前，显然在PageRank算法下靠Term Spam是无法实现的。不过既然我明白了PageRank主要靠内链数计算页面权重，那么我是不是可以考虑建立很多空架子网站，让这些网站都链接到我博客首页，这样是不是可以提高我博客首页的PageRank？很不幸，这种方法行不通。再看下PageRank算法，一个页面会将权重均匀散播给被链接网站，所以除了内链数外，上游页面的权重也很重要。而我那些空架子网站本身就没啥权重，所以来自它们的内链并不能起到提高我博客首页PageRank的作用，这样只是自娱自乐而已。

首先明确将页面分为几个类型：

1、目标页

目标页是spammer要提高rank的页面，这里就是我的博客首页。

2、支持页

支持页是spammer能完全控制的页面，例如spammer自己建立的站点中页面，这里就是我上文所谓的空架子页面。

3、可达页

可达页是spammer无法完全控制，但是可以有接口供spammer发布链接的页面，例如天涯社区、新浪博客等等这种用户可发帖的社区或博客站。

4、不可达页

这是那些spammer完全无法发布链接的网站，例如政府网站、百度首页等等。

作为一个spammer，我能利用的资源就是支持页和可达页。上面说过，单纯通过支持页是没有办法spam的，因此我要做的第一件事情就是尽量找一些rank较高的可达页去加上对我博客首页的链接。例如我可以去天涯、猫扑等地方回个这样的贴：“楼主的帖子很不错！精彩内容：http://codinglabs.org”。我想大家一定在各大社区没少见这种帖子，这就是有人在做spam。

然后，再通过大量的支持页放大rank，具体做法是让每个支持页和目标页互链，且每个支持页只有一条链接。

这样一个结构叫做Spam Farm，其拓扑图如下：



其中T是目标页，A是可达页，S是支持页。下面计算一下link spam的效果。

设T的总rank为y，则y由三部分组成：

1、可达页的rank贡献，设为x。

2、心灵转移的贡献，为β/n。其中n为全部网页的数量，β为转移参数。

3、支持页的贡献：

设有m个支持页，因为每个支持页只和T有链接，所以可以算出每个支持页的rank为：

\(\large \frac{(1-\beta)y}{m}+\frac{\beta}{n}\)

则支持页贡献的全部rank为：

\(\large m(1-\beta)(\frac{(1-\beta)y}{m}+\frac{\beta}{n})\)

因此可以得到：

\(\large y=m(1-\beta)(\frac{(1-\beta)y}{m}+\frac{\beta}{n})+x+\frac{\beta}{n}\)

由于相对β，n非常巨大，所以可以认为β/n近似于0。 简化后的方程为：

\(\large y=m(1-\beta)(\frac{(1-\beta)y}{m})+x\)

解方程得：

\(\large y=x\frac{1}{2\beta-\beta^2}\)

假设β为0.2，则1/(2β-β^2) = 2.77则这个spam farm可以将x约放大2.7倍。因此如果起到不错的spam效果。

Link Spam反作弊
针对spammer的link spam行为，搜索引擎的反作弊工程师需要想办法检测这种行为，一般来说有两类方法检测link spam。

网络拓扑分析
一种方法是通过对网页的图拓扑结构分析找出可能存在的spam farm。但是随着Web规模越来越大，这种方法非常困难，因为图的特定结构查找是时间复杂度非常高的一个算法，不可能完全靠这种方法反作弊。

TrustRank
更可能的一种反作弊方法是叫做一种TrustRank的方法。

说起来TrustRank其实数学本质上就是Topic-Sensitive Rank，只不过这里定义了一个“可信网页”的虚拟topic。所谓可信网页就是上文说到的不可达页，或者说没法spam的页面。例如政府网站（被黑了的不算）、新浪、网易门户首页等等。一般是通过人力或者其它什么方式选择出一个“可信网页”集合，组成一个topic，然后通过上文的Topic-Sensitive算法对这个topic进行rank计算，结果叫做TrustRank。

TrustRank的思想很直观：如果一个页面的普通rank远高于可信网页的topic rank，则很可能这个页面被spam了。
设一个页面普通rank为P，TrustRank为T，则定义网页的Spam Mass为：(P – T)/P。
Spam Mass越大，说明此页面为spam目标页的可能性越大。



这是用一天加购数据
---day info---
20191105
20191104
getOrderUsefulInfo read hdfs directly
map size: 64703
(rs count ,4326)
11324.06user 43.47system 3:06:34elapsed 101%CPU (0avgtext+0avgdata 3277772maxresident)k

这是用一天详情点击
---day info---
20191106
20191105
getOrderUsefulInfo read hdfs directly
map size: 62873
(rs count ,506)
14514.78user 57.30system 3:59:06elapsed 101%CPU (0avgtext+0avgdata 5000824maxresident)k
33472inputs+21360outputs (4major+1354318minor)pagefaults 0swaps
[ilambda@bi-sp32-west4-b manley]$



