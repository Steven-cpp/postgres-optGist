
# Integrating RLR-Tree into PSql

这是本项目的记录文档，包括相关知识的学习笔记、对于该项目各周期的进度记录，以及讨论的汇总。

## I. Paper Reading

在与Prof Gao讨论之后，我的 Master Project 就是要将 RLR Tree 实现到 PostgresSQL 中。首先需要深入理解 RTree，以及 RLR Tree 的实现思路。

### 1. R-Tree

R-Tree 是 RLR Tree 的基础，它是于 1984 年由 Guttman 提出来的，主要用于**空间搜索**。以下的笔记参考的材料有：

- [Introduction to R-Tree](https://www.geeksforgeeks.org/introduction-to-r-tree/)
- [The R-Tree: A dynamic index structure for spatial searching](https://hpi.de/rabl/teaching/winter-term-2019-20/foundations-of-database-systems/the-r-tree-a-dynamic-index-structure-for-spatial-searching.html)
- [R-Tree: algorithm for efficient indexing of spatial data](https://bartoszsypytkowski.com/r-tree/)

R-Tree 是用于对高维数据和地理数据 (例如坐标和矩形) 进行有效地存取，它的特点是只有一个根结点，而且子节点指向的内容完全包含在父节点的范围中。而只有叶子结点才真正包含指向的对象的内容，这里的数据对象指的是一个闭区间的 $n$ 维矩形。一个典型的 R-Tree 示意图如下：

![image-20220923193722920](https://cdn.jsdelivr.net/gh/Steven-cpp/myPhotoSet@master/img/image-20220923193722920.png)

<div style='font-size: 14px; 
            color: rgba(117, 117, 117, 1); 
            line-height: 20px;     
    				max-width: 80%;
    				min-height: 43px;
    				display: inline-block;
   	 				padding: 10px;
    				margin: 0 4em;
    				border-bottom: 1px solid #eee;' > 
图1: R-Tree的示意图. 图a显示了一个三层的R-Tree, 它每个结点的最大指针数为3, 从而每个结点的可用指针数都不能小于3/2(即2). 而且, 只有叶子结点指向的才是实际的数据对象, 而且子结点完全包含在父结点中, 这一点从图b中可以见得.</div>
R-Tree 是自下而上构建的，叶子结点逐渐聚合为更大的矩形，直到形成树的根节点。

**搜索目标对象**

这里的目标对象指的就是图1中的实线矩形，搜索算法会自顶向下地遍历每个结点，检查它是否完全包含目标矩形。如果是，就选中它的子节点继续遍历。该算法的问题是一个结点下需要搜索多个子树，如果树的高度特别高，时间就会很长，难以度量最差的表现。

**更新 R-Tree**

CondenseTree: 在删除结点时触发。当数据对象被删掉后，该算法检查对应的叶子结点是否仍有 $m/2$ 个可用指针，其中 $m$ 为每层的最大结点数。如果小于该阈值，则会删除该叶子结点，以及父结点中的指针，并将叶子结点中的所有指针保存至临时的数组 $Q$ 中。同时，再对父结点进行类似的检查，最后将 $Q$ 中的元素插入到 R-Tree 中。

AdjustTree: 在插入结点时触发。如果插入后，当前结点的指针数 > $m$，那么就需要对该结点进行分割。在分割的时候需要确保分割后的区域应当是最小化的，正如下图所示：

![image-20220924152147338](https://cdn.jsdelivr.net/gh/Steven-cpp/myPhotoSet@master/img/image-20220924152147338.png)

<div style='font-size: 14px; 
            color: rgba(117, 117, 117, 1); 
            text-align: center; 
            line-height: 20px;     
    				min-height: 43px;
   	 				padding: 10px;
    				margin: 0 1em;
    				border-bottom: 1px solid #eee;' > 
图2: 对结点进行split操作</div>

**结点切分**

切分结点的方法有两种：

1. **线性复杂度切分**

   从 $Q$ 中选取距离最远的两个点分别作为新分组的头元素，然后将剩余的点随机分配至新分组中

2. **平方复杂度切分**

   从 $Q$ 中选取所能张成的最大区域的两个区域作为新分组的头元素

### 2. Reinforcement Learning Based R-Tree

将该篇论文的要点整理如下：

| Title             | A Reinforcement Learning Based R-Tree for Spatial Data Indexing in Dynamic Environments |
| ----------------- | ------------------------------------------------------------ |
| Author            | TuGu, GaoCong @ NTU                                          |
| Year              | 2021                                                         |
| Prerequisite      | [R-Tree, 1984](https://www.google.com/url?sa=t&source=web&rct=j&url=http://www-db.deis.unibo.it/courses/SI-LS/papers/Gut84.pdf&ved=2ahUKEwjIo4Tigpz6AhU0TmwGHetMAnYQFnoECBYQAQ&usg=AOvVaw39B_K-orDTFqVkCujGjYVz), [Recursive Model Index, 2018@MIT](file:///Users/shiqi/Downloads/DBM02_RMI%20Learned%20Index.pdf), |
| Motivation        | 1. 使用 learned indices 替换传统的索引结构 (e.g B-Tree) 往往能够取得不错的性能表现；<br />2. 但是这需要完全替换原有的结构和查询算法，遇到了很多实现上的困难；<br />3. 本文想在<u>不改变索引结构</u>的情况下，采用基于 RL 的方法，提高空间查找的效率。 |
| Current Challenge | 1. 现有 R-tree 的各种 insert 和 split 操作得到的索引树在查询的速度上，都没有显著的优势；<br />2. 将 ChooseSubTree 和 Split 操作形式化为两个连续的 MDP 是相当困难的，如何定义每个过程的状态、动作和奖励信号呢？<br />3. 难以使用 RL 来找到最优的过程，因为当前的 good action 可能会由于之前的 bad action 而得到惩罚值。 |
| Related Work      | 1. Learned Index<br />- data and query limited;<br />- not accurate;<br />- <u>cannot handle updates, or need to periodic rebuild</u>.<br />- replace index structure and query algorithm<br />2. Heuristic Strategies used in R-Tree<br />- no single index outperforms the others |
| Method            | 通过基于 RL 的模型，确定如何建立 R-Tree<br />具体地，这是通过将 insert 和 split 操作形式化为两个连续的 MDP，再使用 RL 来最优化。这就需要定义 MDP 的 state, action, reward signal, transition.<br />**1. State**<br />对每个结点的子节点进行遍历，选取前 $k$ 个插入后面积增加量最少的子节点。并计算$\Delta Area$, $\Delta Peri$, $\Delta Ovlp$, $OR(R)$ 并以相应的最大值正则化，连接后作为该结点的状态向量；<br />**2. Action**<br />类似的，选取这 $k$ 个子节点构成其动作空间，在训练时遍历其中的所有结点；而测试时，给出 Q-Value 最大的结点作为 action.<br />**3. Reward Signal**<br />设计 1 个 reference tree (RT)，将所要插入的对象同时插入到 RT 和 RLR-Tree 中，以两者的*结点访问率 (node access rate)* 的差作为激励信号。 |
| Baseline          |                                                              |
| Highlight         |                                                              |
| Future Challenge  |                                                              |
| Relevant Work     | 1. [The "AI+R"-tree: An Instance-optimized R-tree](https://arxiv.org/pdf/2207.00550v1.pdf): 将原有的数据库查找操作变为多标签分类任务；<br />2. |



## II. Psql Learning



### 1. Index Archetecture

在 PostgreSQL 8.4.1 中支持的索引有：B-Tree 索引、Hash 索引、GiST 索引和 GIN 索引。

[PostgreSQL: BTree-implementation](https://www.postgresql.org/docs/current/btree-implementation.html)

> 🔍**如何实现一个索引？**
>
> 1. 把树的结构写出来，确定它所有接口的 API；
> 2. 链接到数据库的操作中。
>    - 索引如何存储？

#### 1）B-Tree

[Postgres Indexes Under the Hood](https://rcoh.me/posts/postgres-indexes-under-the-hood/#:~:text=Indexes%20in%20Postgres&text=These%20indexes%20are%20implemented%20internally,implementer%20of%20the%20data%20structure.)

**Branching Factor 的选取**

就是一个结点最多能容纳的数据元素的个数

B-Trees are extremely shallow data structures. Because the branching factor is typically in the thousands, they can store millions of elements in only 2-3 layers. When used in a database, this means only 2-3 disk seeks are required to find any given item, greatly improving performance over the dozens of seeks required for a comparable on-disk binary search tree or similar data structure.

Typical branching factors will be between a few hundred to a few thousand items per page.

**Specification**

1. Postgres nodes have a fixed amount of bytes

   If you have variable-size data, each node in your index will actually have a different number of children

2. Highr key allows concurrency

   The “high-key” pointer allows readers to detect that this split has occurred: If you’re looking for a value greater than the high key, you must follow the right-link! The right link allows the reader to traverse directly to the newly split node where the key now resides.

#### 2）GiST Index

[Implementation of GiST indexing for Postgres](https://github.com/postgres/postgres/tree/master/src/backend/access/gist)

[【参考材料1】The GiST Indexing Project](http://gist.cs.berkeley.edu/)

GiST (Generalized Search Tree) 称为通用搜索树，它为各种类型的索引树 (R-trees, B+-trees, hB-trees, TV-trees, Ch-Trees 等) 都提供了一个统一的接口，允许用户在任意数据类型上进行索引。除此之外，GiST 还具有数据和 *查询的可拓展性*。

> 📕 **查询的可拓展性**
>
> 这里指用于可以在 GiST 中自定义查询谓词。以前的搜索树在其处理的数据方面是可扩展的。例如，POSTGRES支持可扩展的B+树和R树。这意味着你可以使用POSTGRES在任何你想要的数据类型上建立一个B+树或R树。但是 B+ 树只支持范围谓词（<, = >），而 R 树只支持 $[n, d]$ 范围查询（包含、包含、相等）。因此，如果你用 POSTGRES B+ 树来索引，比如说，一堆电影，你只能提出类似 "查找所有 < T2 的电影 "的查询。虽然这个查询可能有意义（例如，小于可能意味着价格不那么贵、评分不那么高），但这样的写法并不显然。相反，你想问的是关于电影的特定查询，比如 "找到所有有爆炸场面的电影"，"找到所有有吴京的电影"，或者 "找到所有有摩托车追逐的电影"。这样的查询在 B+ 树、R 树或者除了 GiST 之外的任何其他已知结构中都无法直接支持。
>
> 相比之下，你可以通过编程让 GiST 支持任何查询谓词，包括上面提到的 `爆炸场面` 和其他谓词。要让 GiST 启动和运行，只需要实现 4 个用户定义的方法，这些方法定义了树中键的行为。当然，这些方法会是非常复杂的，来支持复杂的查询。但对于所有的标准查询（如 B- 树、R- 树等），就不需要这些了。简而言之，GiST 结合了新的可扩展性、通用性、代码重用和一个漂亮的简洁界面。

由于 B-Tree 处理的是数值型、R-Tree 是 Bounding Box，这种统一性就意味着 GiST 的 key 是独特的。它的 Key 是由用户自定义的类的成员，并且可以通过判断它的某些属性来使得键的指针能够指向所有的 item，即支持类似于小于操作的属性。

**Key 的 Class 的实现**

以下给出了用于键的用户自定义的 class 需要实现的 4 个接口：

1. **Consistent:** This method lets the tree search correctly. Given a key **p** on a tree page, and user query **q**, the Consistent method should return **NO** if it is certain that both **p** and **q** cannot be true for a given data item. Otherwise it should return **MAYBE**.

   > ? **p** 为 true 的含义是什么
   >
   > -> 就好比自定义数据对象的比较操作

2. **Union:** This method consolidates information in the tree. Given a set **S** of entries, this method returns a new key **p** which is true for all the data items below **S**. A simple way to implement **Union** is to return a predicate equivalent to the disjunction of the keys in **S**, i.e. "**p1** or **p2** or **p3** or...".

3. **Penalty:** Given a choice of inserting a new data item in a subtree rooted by entry **<p, ptr>**, return a number representing how bad it would be to do that. Items will get inserted down the path of least **Penalty** in the tree.

4. **PickSplit:** As in a B-tree, pages in a GiST occasionally need to be split upon insertion of a new data item. This routine is responsible for deciding which items go to the new page, and which ones stay on the old page.

There are some optional additional methods that can enhance performance. These are described in [the original paper](http://s2k-ftp.cs.berkeley.edu/gist/gist.ps) on the data structure.

而对于索引项的增删改查，GiST 已经内置实现了，但这恰恰是本项目需要修改的地方。本项目应当是通过使用与索引项管理相关的 7 种方法，实现：

1. 索引的创建 `gistbuild`；
2. 索引项的插入 `gistdoinsert`;
3. 索引的查询 `gistnext`.



## III. Implementation

首先，我要了解 R-Tree 是如何进行增删的，我找到了[Delete a Node from BST](https://practice.geeksforgeeks.org/problems/delete-a-node-from-bst/1?utm_source=gfg&utm_medium=article&utm_campaign=bottom_sticky_on_article)， 可以在有空的时候练一练。不过我的重点还是应该在看论文，了解这个模型的架构。因为对于这些增删改查的操作，这篇论文是使用了基于 RL 的方法，不要求先学懂传统的增删的方法。

- implement and integrate into DBMSs
- Generalized Search Tree (GiST), a “template” index structure supporting an extensible set of queries and datatypes.
- Why generalized search tree can support extensible queries?

### 0. Extending Python with C++

[Python docs: Extending Python with C++](https://docs.python.org/3/extending/extending.html)



### 1. Project Structure

RLR-Tree 代码仓库中包含了 6 个 Python 文件和 2 个 C 文件，定义了 R-Tree 的结构及接口、从给定的数据集中构建树的过程、KNN 查询测试方法、范围查询测试方法、RL 选择子树策略的实现、RL 分裂结点策略的实现等过程。下面将每一个文件的作用及依赖关系给出。

**数据结构定义**

1. `RTree.cpp`

   实现了 `RTree.h` 中 RTree 的 insert, split, rangeQuery 等等操作

2. `RTree.py`

   依赖于 [1]，对输入项稍加处理后，直接调用 [1] 中 C++ 对于 RTree 的实现

**核心算法定义**

3. `model_ChooseSubtree.py`

   依赖于 [2]，定义了选择子树的 RL 算法

4. `model_Split.py`

   依赖于 [2]，定义了分裂结点的 RL 算法

5. `combined_model.py`

   依赖于 [2]，定义了交替训练选择子树和分裂结点的算法

**测试过程定义**

6. `RTree_RRstar_test_cpp_KNN.py`

   依赖于 [2]，定义了 R-Tree 和 RRStar 使用 KNN 查询的测试过程

7. `RTree_RRstar_test_cpp.py`

   依赖于 [2]，定义了 R-Tree 和 RRStar 使用范围查询的测试过程

8. `main.cpp`

   依赖于 [1]，定义了读取数据集及测试 baseline 的方法

现在，需要确定的是：

- [ ] 能否把文件 [5] 迁移到 Gist 上，也就是基于 [5] 修改 Gist 中ChooseSubtree 和 Split 的 API。也就是修改 `gistdoInsert`；
- [ ] 训练与推断过程 (Python 实现的) 如何迁移到 PSQL (C++ 实现的)上面。是在这两个之间建一个接口，还是使用 PSql 的框架重新实现一遍。

### 2. Gist

在确定完当前的工作后，我看了 Gist 的实现代码，找到了其中要修改的文件之一 `gistsplit.c`。它有 700 多行，而且从注释上看，它与硬盘中的 page 紧密相关，我对其中 picksplit, column 的含义都一无所知，完全看不懂它在干什么。因此，还是有必要先看懂 Gist 的理论基础 [Concurrency and Recovery in Generalized Search Trees](https://dsf.berkeley.edu/papers/sigmod97-gist.pdf)，再看代码实现。

> 🔍 **如何高效阅读源码**
>
> 高效地阅读源码要求我们**自顶向下**地看这个项目，先了解业务流程，再理清执行流程，最后再深入到代码的每一行中。具体地，在需要阅读一个较大项目 (e.g 由多个文件组成，总代码行数 > 5k) 前，需要先充分了解这个代码的业务逻辑，即**要解决什么问题、有哪些功能、数据怎么交互的**。接下来，把代码跑起来，各种功能都用一下，了解他的执行逻辑（这里可以画出代码执行的流程图）。最后，再开始看源码，这样会容易上手很多。

#### 1) GiST 的实现

[Indexes in PostgreSQL](https://habr.com/en/companies/postgrespro/articles/444742/)

GiST 的作者在[Generalized Search Trees for Database Systems](https://pages.cs.wisc.edu/~nil/764/Relat/8_vldb95-gist.pdf)介绍了 GiST 提出的背景、特点、与 B+树 和 R 树的不同、数据结构、实现方法、性能分析，同时作者还回顾了数据库中索引树的基本思想并强调了某些细节。这篇文章非常适合入门，对于后续理解索引树中的并行及 R 树的代码非常重要。

由于传统的索引树如 B+ 树、R 树，只能提供内置的 predicate (如比较数字的大小、范围查询)，并且需要存储数据的 key 满足相应的条件，因此可延展性不够好。于是就有伯克利的学者提出更具延展性的索引树 GiST (Generalized Search Tree)。它支持用户自定义 predicate，只需要实现作者指定的 6 个方法即可。

这六个方法包括与查询相关的 predicate 定义的 4 个方法，以及与树结构调整相关的 2 个方法。对于本项目，应当重点看后者的两个方法:

1. $Penalty(E_1, E_2):$ 给定两个结点 $E_1, E_2$，返回将 $E_2$ 插入到以 $E_1$ 为根的子树中的 cost。例如在 R-Tree 中，这里的 cost 指的就是插入后 $E_2$ 后，$E_1$ 包围盒的增量；
2. $PickSplit(P):$ 给定一个包含 $M+1$ 个结点的集合 $P$，返回将 $P$ 划分为两个集合 $(P_1, P_2)$ 的最佳方案。

在作者提出的 `ChooseSubTree(3)` 和 `Split(3)` 算法中，使用到的外部函数有且仅有上述两个方法。

****

🚩 **目标 1: ** 将上述两个函数，以文件 [5] 中的方法实现即可。

看 PostgeSQL 中 [RTree 的代码](https://github.com/postgres/postgres/tree/master/src/backend/access/gist)，它的 ChooseSubTree() 是不是仅仅基于 penalty() 这个外部方法。如果是的话，基于文件 [5] 实现 penalty 即可。

它调用的是 `gistState->penaltyFn[]`，而对 penalty 的定义是在 `RelationData* index` 中的。通过 `gistStateInit()` 函数，将对每个 key 的 penaltyFn 地址赋值到对应的 `penaltyFn[]` 数组中。

现在我下载了 `libgist` 这个仓库，它是 GiST 的 C++ 实现，但是还没有融入到 PostgreSQL 中，虽然这个里面有 example。所以，我还是决定直接对 PostgreSQL 进行 Debug，使用 [VSCode build PSql 的源码](https://blog.sivaram.co.in/2022/09/25/debugging-postgres-on-m1-mac)、[了解 GiST 在 PSql 的用法](https://habr.com/en/company/postgrespro/blog/444742/)、[调试指令](https://blog.sivaram.co.in/2022/09/25/debugging-postgres-on-m1-mac)，来深入地了解 PSql 的运行逻辑，从而对其进行修改。除此之外，这个源码中还有相当多涉及硬盘分页的知识，我还要深入的学习索引与物理内存、外存的对应关系。这个应当与并行的设计高度相关，所以我还要深入理解 [Concurrency and Recovery in Generalized Search Trees](file:///Users/shiqi/Downloads/gist_concurrency.pdf).

🚩 **目标 1.0: 理解 GiST 的项目架构**



🚩 **目标 1.1: 理解 GiST 的并行 理清 PSql 运行逻辑 **

[如何扩展 Psql 代码](https://docs.postgresql.tw/internals/writing-a-procedural-language-handler)

- 创建临时 context

- 创建空的 GiST 索引树

- `gistInsert()` -> `gistDoInsert(formTuple())` --> `gistChoose()` --(find the specific page)--> `gistplacetopage()`

  在分裂时，先将当前的 full node 复制一份，然后将要插入的 tuple 与复制出来的 node 组合起来 (join) 得到 `itvec`，接着再使用 `gistSplit()` 对 itvec 进行切分操作。

- 在 page 中，都是使用 offset 进行读写操作的

- `gistChoose()`

  该函数是用来选择 penalty 最低的结点，但它是能看到多个插入操作的。那么如果当前它找到的最好的结点与前一个结点同样好 (具有相同的 penalty 的值)，那么该算法就会偏向于选择 old best，但也有一定概率选择 new best。

  该索引也可能是基于多个属性构建的，因此还需要遍历每个属性，调用 `gistPenalty()` 计算出<u>插入至该属性所属元素位置的 penalty</u>. 最终，该函数是通过 `giststate->penaltyFn[attno]` 来调用用户自定义的 penalty 函数。

今天我花了 4 个小时，大概理清了 GiST 是如何实现插入操作的，这就囊括了 search, split, delete 这三种操作。对 GiST 的实现逻辑已经有了基本的认识与了解。下一步我要找到它是如何计算 penalty 的，与哪些函数相关。然后再去 build 这个项目

> **❓执行流程**
>
> 1. **`itup` 是否只是硬盘中的 page，需要加载到缓存中？**
>
>    
>
> 2.  **`oldoffnum` 是谁生成的？**
>
>    应该指的是不需要的结点，可以被替换。
>
>    ✅ 指的是通过 `gistchoose()` 方法找到的 具有最低 penalty 的 page offset。目标 entry 应当直接覆盖这片区域，如果该区域有效。
>
> 3. **`F_FOLLOW_RIGHT` 标志位什么时候被 set?**
>
>    $ 在 `placetoPage()` 开始之前，如果该标志位为 True，那么代表当前结点的分裂尚未完成，应当停止插入操作。
>
>    $ 在论文中，该标志位是在查询时，为了能够与插入同时进行而设立的。如果在查询时，途经结点由于并行的插入操作而分裂，那么查询也能通过 right link 找到目标结点。
>
>    ✅ 顺着这条线出发，如果该标志位为 True，说明有在该结点同时发生的 insert 操作，并且产生了分裂。否则，该标志位应当为 False。那么对于同一个结点的 insert 操作，当前操作确实应当被阻塞。
>
> 4. **Page 存储的是什么？**
>
>    -> 相当于树结构中的每个结点，包含多对 entry?
>
>    $ page 中每个 offset 位置的元素就是一个 tuple
>
>    $ 每页 page 包含 `PageMaxOffsetNumber` 个 tuple
>
> 5. **为什么被插入的 page 不一定为 leaf node?**
>
>    $ GiST 是将所有 tuple 都放到了叶子结点中，那么插入的元素也都应当在叶子结点。
>
>    $ 在 `placeToPage()` 中要求判断输入的 page 是否为叶子结点，也就是说明可以不在叶子结点插入元素。
>
> 5. **为什么先删除旧结点，再写入新结点，而不是直接覆盖旧结点？**
>
>    $ 源码中的逻辑是判断当前结点是否被删除，如果已经被删除，就直接 fill 相应大小的区域；如果只是被标记为删除，就删除这个 tuple，再 fill 相应大小的区域。
>
>    -> 因为 data_type 不同，所以 tuple 的大小不同吗？
>
> 7. **什么是在插入时的 cache-locality?**
>
>    $ 如果要插入多个元素，并且这些元素的目标位置是分裂前的结点。由于 GiST 的key 是可以重合的，因此分裂后的两个结点<u>可能?</u>都可以插入。但是为了保证 cache-locality，作者选择只插入左边的结点，而不管右边的结点。即使这样能够使得插入操作更均匀的分布。
>
>    ✅ 如果选择了右边的结点，就需要对右边的结点进行遍历。这样就会需要 context switch，将新的 page 读取到内存中。



> **🔍 术语理解**
>
> - relation
>
>   在 DBMS 中指的就是数据库中的一张表
>
> - page VS block
>
>   page 是在内存中的一个一个读取单元，而每个 page 又是由多个 block 组成的。当 DBMS 在执行查询操作的时候，会先将需要的 page 读入到内存中，然后再遍历其中的 block。通过分级读取，就能大大提高查询的效率。page 就应当是结点的概念
>
> - NSN (node sequence number)
>
>   全局变量，只有在被分裂产生新结点时，才会被更新。
>
>   -> 被分裂的结点和新结点的 NSN 都需要更新
>
> - 多级索引
>
>   单个索引定义在多个 col 上。在索引定义中，越靠前的 col 重要性越高。

🚩 **目标 1.2: 找到 penalty 相关的函数**



🚩 **目标 1.3: 调试 PSql 源码 深入理解 PSql 的运行逻辑**

我跟着教程，使用 VS Code build 了 PSql 的源码并且了解了如何进行调试。居然可以直接执行编译好的程序，然后 VS Code 就会智能地将当前的执行过程与源代码进行对应，这让我很意外。

我意识到，可能所有我之前的调试都是这样默默进行的。对于 C++，也是编译好了可执行文件，然后运行该程序，将该程序的 process_id attach 到调试器中，调试器调用系统的 API 去监视这一程序的运行资源，并且将机器指令与源代码进行对应，定位到当前执行的行数。调试在 Linux 下具体是如何实现的可以看: [Linux 下调试程序的系统调用](https://www.linuxjournal.com/article/6100)



🚩 **目标 1.3.1: 理解 PSql 索引的工作机制**

[Introduction of indexes in PostgreSQL](https://habr.com/en/company/postgrespro/blog/441962/)

Access method: 索引的类型。需要实现以下任务：

1. build 索引并将数据映射到 page 中的算法；
2. 支持的查询信息，给出适用的 clolumn 以及谓词，方便 optimizer 选择；
3. 计算缓存使用开销；
4. 操作并行访问需要的锁；
5. 记录 write-ahead log (WAL) 日志

Indexing engine: 调用索引，并得到返回的 TID。同时根据事务隔离的等级，检查 TID 是否在当前是可见的

Optimizer: 对执行的查询语句进行优化，选择合适的索引，使得查询的开销最小

Row version: 单独存储的行向量的副本，以支持对 table 的并行访问

使用索引查询的逻辑为：用户输入查询语句，optimizer 对该语句进行优化，选择合适的索引及索引方法。Index engine 调用相应的索引，并使用指定的方法搜索索引，得到所有满足条件的 TID。接下来，index engine 根据这些 TID 访问对应的 entry，获得其 row version. 最后检查 TID 是否在当前可见，并返回这些可见的 row version.

> **🔍 Index Scan vs Bitmap Scan**
>
> 在实际查询索引树的时候，有两种方式: index scan, bitmap heap scan. 前者就是按照索引树的结构读取 page，顺序访问 block，返回满足条件的 TID，直到返回所有满足条件的条目，接着 index engine 就会通过这些 TID 索引对应的 page，并读取 block 中的条目。
>
> 该方法的问题是，当返回的 TID 很多时，多个 TID 很可能在同一个 page 中，因此就会重复读取一个 table page 多次，效率比较低。而[仅仅根据 TID，index engine 并不知道如何最有效地 fetch rows](https://dba.stackexchange.com/questions/119386/understanding-bitmap-heap-scan-and-bitmap-index-scan). Bitmap Scan 就可以做到这一点。
>
> *Lossy* Bitmap Scan 首先为每个条件建立了一个 `<bit> array` 对应每个 table page。在第一次 index scan 的过程中，如果当前 TID 满足条件，那么就把该 TID 所属 page 对应的 bit array 设为 1。最后在 fetch rows 的过程中，只要将给定条件的 bit-array AND 到一起，就只需要顺序读取值为 1 所对应的 page，可以保证每个 page 只读取 1 次。但是，在每次读取的时候，都需要 recheck condition，因为这里的 `1` 仅代表这个 page 至少有 1 个满足条件的元素。
>
> 而 *exact* Bitmap scan 是精确到每个 block，就不需要再 recheck condition 了，尽管在 optimizer plan 的时候仍然会有这一项，但不一定会执行。
>
> ps: 一开始我在理解的时候没有区分 table page 和 index page，这是两个独立的存储结构，得到了 TID 还要再去索引 table page. 这是造成 index scan 效率低的原因。

Covering Index: 查询的数据就包含在 index 中，不需要再索引 table page. 但是索引中并没有存储 visibility 的信息，所以还是需要访问 table page。这显然是件麻烦事，为了解决这个问题，PSql 维护了 `visibility map` 来判断当前 page 是否可见。

Visibility Map: 标记了刚刚被改变的页面，从而无论事务的开始时间和隔离等级，该页面对所有事务都是可见的。

#### 2) PSql 语法

在 retrieve 函数返回的自定义的数据类型时，要声明该对象的类型，例如使用 `from xxx as name(type1, type2, type3)`

```sql
select level, a 
from gist_print('airports_coordinates_idx') as t(level int, valid bool, a box)
    where level = 1;
```

我在这里要了解的重点是多维数据的存取方式，例如我们想创建一张存取二维数据点的表 `points`

1. 开启 PG server (关闭同理)

   ```bash
   $HOME/postgres/pg14/bin/pg_ctl -D $HOME/postgres/pgdata -l logfile start
   ```

2. 登陆 PG 并打开 demo 数据库

   ```bash
   $HOME/postgres/pg14/bin/psql -p 5432 demo
   ```

1. 获取调试进程 ID

   ```sql
   SELECT pg_backend_pid();
   ```

1. 创建表

   ```sql
   create table points(p point);
   ```

2. 插入坐标值

   ```sql
   insert into points(p) values
     (point '(1,1)'), (point '(3,2)'), (point '(6,3)'),
     (point '(5,5)'), (point '(7,8)'), (point '(8,6)');
     
   insert into points(p) values (point '(2,2)'), (point '(4,4)'), (point '(1,5)');
   ```

3. 创建 GiST 索引

   ```sql
   create index on points using gist(p);
   ```

4. 查询包含在某个 bb 的点

   ```sql
   select * from points where p <@ box '(2,1),(7,4)';
   ```

5. 排序

   `p <-> (4, 7)`: 返回距离点 `(4, 7)` 最近的 2 个点

   ```sql
   select * from points order by p <-> point '(4,7)' limit 2;
   ```

   > 🔍 **如何实现排序**
   >
   > 是否需要对查询范围内的每个点计算一遍距离，然后排序后选择前两个？
   >
   > $ Search like this is known as k-NN — k-nearest neighbor search.
   >
   > $ KNN 算法中就是对样本内所有的点计算距离
   >
   > -> (粗剪枝) 以该点为中心，画一个 bb，通过二分的方式，使得 bb 内包含的点 > 要求的个数， < 要求个数+50。从而大多数计算距离 op 变为了包含 op，仅需计算 bb 内所有点到该点的距离即可。可以缩小检查范围。





😞 feeling down, frustrated by the failure of seeking job, limitied by my STP and foreigner status. Nothing to do with my social interaction. Never pay attention to the dishonest rhetoric, sweet but cheap.

After reviewing the profiles of chinese NTUer working for real big foreign companies (e.g Wise, Barclay, Visa), I found that they invariably had a bachelor degree of NTU, neither just had a master degree. Undergraduate is far superior than master, I ever heard such argument, but now, I fully understand it. Not only do they have specific internship programs, but they have the right to commit full-time internship during semester. Now I feel excluded, and really don't know what to do.

But after talking with Alan, I feel better



#### 3) GiST 中并行的实现

在一个完整的数据库系统

Concurrency control techniques are used to ensure that the *Isolation* (or non-interference) property of concurrently executing transactions is maintained. The target of concurrency control is to make concurrent transactions look like serialized transactions. 

**Why B+ tree?**

B+ trees have a higher branching factor: B+ trees typically have a higher branching factor than B trees (B-Tree can also have orders of 3, 4, 5, etc.), which means that they can store more keys per node. This can lead to fewer nodes in the tree, which can result in <u>faster search times</u> and <u>less overhead for tree maintenance operations</u>.

![B+Tree, all leaf nodes are in the same level](https://raw.githubusercontent.com/Steven-cpp/myPhotoSet/main/image-20230330102759908.png)

And for B-Tree, the key is seperated across the entire tree, the internal node is also consists of keys. So, for the range query, B+ tree is much faster than B tree. Supposing one node of B-tree is consistent with the given range, then we still need to iterate through all its substrees, and push all the visited nodes into the result container. While for the B+ tree, it only needs to find the corresponding range, and then get the start and end of the linked_list, then iterate through it, which is much faster.



- section 2 contains a brief description of the basic GiST structure

  1. **Why do we need GiST?**

     由于传统的索引树如 B+ 树、R 树，只能提供内置的 predicate (如比较数字的大小、范围查询)，并且需要存储数据的 key 满足相应的条件，因此可延展性不够好。而且从 B 树的实现上来看，对于并行访问、事务隔离、异常恢复的支持使得代码变得异常的复杂，并且占据了代码的主要部分。真正实现一个 DBMS 中的索引树相当复杂。

     于是就有伯克利的学者提出更具延展性的索引树 GiST (Generalized Search Tree)。它支持用户自定义 predicate，只需要实现作者指定的 6 个方法即可。而且不需要考虑并行访问、事务隔离等特性。

  2. **Why it can support extensible queries?**

     Because the predicates of GiST is user-difined, more specifically, the `consistent()` method. When searching according to one given predicate,  it just invokes this method to determine whether the given predicate is consistent with the current node. So, there is no restrictions on the queries. It can be range comparison like B-Tree, rectangle box containment like R-Tree, and so on. As long as you implement the `consistent()`, it can then support the corresponding queries.

  3. **What's the structure of GiST?**

     The GiST is a banlanced tree, with extensible data types and queries, and it commonly has a large fanout. The leaf node contains $(key, RdI)$ pairs, with the record identifier pointing to the page the target data lies. While the internal node contains $(predicate, childPointer)$ pairs, where the predicate implies all the data items reachable from the subtree, the `childPoniter` points.

     And this exactly captures the essence of a tree-based index structure: a hierarchy of predicates, in which each predicate holds true for all keys stored under it in the hierarchy.

  4. **How GiST search works?**

     

  5. **How GiST insert works?**

     

  6. **What is the difference between B-Tree and GiST**

     Aside from the extensible queries that only GiST supports, their tree structure is also a little bit different. Overlaps between predicate at the same level is allowed in the GiST, and the union of all these predicates may have "holes". While for the B-Tree, the value range distributed in each level is unique and contagious.

- section 3 extends this structure for concurrent access

  1. **What does concurrent access mean?**

     That means multiple tree operations or transactions can be performed simultaneously. For example, the search and insert can be executed concurrently, still with the correct results.

  2. **How to make the original structure support concurrent access?**

     For the original GiST structure, the concurrent access may fail some search operations. Supposing the insert and search are executed concurrently, and the target insertion node is full. Then, the node needs to be split, and some of the predicates will be moved into its right siblings. However, if such predicate is useful for the search operation, then the search will miss target. We need to find a way that can determine whether this node has been split, and when to stop following the right links.

     To resolve such dilemma, we can <u>add node identifier</u>, and links between the node and its right siblings after splitting. The GiST will manage a global monotonically increasing counter. Each operation will increment and then record it. The resulting new node will be assigned to the current value. So if the search operation finds that its sequence value is less than the value of the current node, then it knows that this node has been split, and it should follow the right link of this node.

  3. **What's the implementaion difficulties?**

     Emm, this method needs to extend the binary pairs to triple pairs. Every node needs to add one item called sequence value, it may reduce the fanout of the tree. Thus, it's hard to keep the original performance.

     What's more, while managing the global counter variable. The incrementing and recording invoked by tree operation should be atomic, or it may still miss targets.

- section 4 outlines our design of the hybrid locking mechanism.

  1. **What's the purpose of hybid locking?**

     

  2. **What problem it is going to solve?**

     

  3. **How does hybid locking works?**

     

  4. **What may be the implementaion difficulties?**

     

  5. **What does repeatable read mean?**

     It means that a transaction sees a consistent snapshot of the data at the time it starts. More specifically, if a random search operation run twice in the transaction, then it should return the same result.

  6. **What kind of problem can be solved with key-range locking?**

     

  7. **What is two-phase locking?**

     The two-phase lock protocol limits the transaction to acquire all the necessary locks first, and then can only release locks. The two phases are the growing phase and the shrinking phase. During the growing phase, a transaction acquires locks on the data it needs to access or modify. During the shrinking phase, the transaction releases the locks it no longer needs.

     The two-phase lock protocol helps to ensure that transactions do not hold onto locks for longer than necessary, which can lead to lock contention and other concurrency issues. It also helps to ensure that transactions do not try to acquire locks that they have already released, which can lead to data inconsistencies and other issues.

     

- section 5: key search

  1. **How it enables concurrency?**

- section 6: key insert

  1. **When will the operation propogated upwards?**

     

  2. **What will happen before the entry $(key, RID)$ really added into the GiST?**

- section 7: key delete

  In one transaction, just mark the node as deleted. But only after the transaction is committed, the parent node are shrinked. For the node marked as deleted, we can simply reuse that node, no need to remove that entry physically. During the insertion process, when it finds a suitable place, it checks whether the entry is set deleted. If so, overwriting the entry is jsut fine.

- section 9: Logging and recovery

- section 10: discusses a variety of implementation issues

- section 11: discusses some of the implicationsof the structure of an access method for concurrency control techniques and explains why most of the prior work on B-trees cannot be directly applied in the GiST context

- section 12 concludes this paper with a summary.

#### 4) Page 结构

[PG 中的 Page结构](https://www.jianshu.com/c/f69542bf8bae)

在计算 penalty 的过程中，比较 tricky 的一点是插入后的结点与其它 $k-1$ 个兄弟结点的重合面积。我需要探索能否通过 `GISTENTRY` 来找到该结点的兄弟结点、子结点、父结点。

```c
typedef struct GISTENTRY
{
	Datum			key;
	Relation	rel;
	Page			page;
	OffsetNumber offset;
	bool			leafkey; // whether it's a leaf node
}
```

我现在只知道 `entry->key` 是 R-Tree 的键值，形如 `(a, b)` 的数据点。而对于其它的数据项:

- `rel`:
- `page`:
- `offset`: 对应 page 中的 block

遍历结点是在 `gistdoinsert(): gist.c` 中发生的。

1. 从关系 `r` 的 buffer 的指定位置 (`blkno`) 获取磁盘中对应的 page；

   `maxoff`: 页面上最高的 offset -> 页面上包含的 tuple 数

   `freeSpace`: 该页面仍可以分配的空间

   -> 索引树的每个结点都是一个 page，其中的每个 tuple 都通过 offset 来索引。最后一项的 offset 为 `PageGetMaxOffsetNumber(page)`. 从而我们只需要遍历父结点的所有 tuple，就能遍历到所有的兄弟结点，进而计算出重叠面积。

2. `indexTuple` 就是键值；

3. 如何将 indexTuple 转换为 GistEntry

根据 page 和 offset 生成所在位置的 tuple，并获取其 key 值（叶子结点）的范式:

````c
ItemId iid = PageGetItemId(p, off);
IndexTuple broTuple = (IndexTuple) PageGetItem(p, iid);
bool isNull = false;
Datum datum = index_getattr(broTuple, 1, giststate->leafTupdesc, &isNull); // for index based on 1 key
````



#### 5) Planner

[How Planner Works?](https://www.postgresql.org/docs/current/planner-optimizer.html)

> **🔍 Planner 的开销计算与 `penaltyFn()` 的关系**
>
> 在执行查询语句之前，PG 还会对查询的流程进行优化，尽力选择出开销最小的执行方式，提交给执行器。例如，一个 key 上有 2 个索引，在对这个 key 查询时，planner 就要估算 3 种查找方式的开销: 使用索引 1，使用索引 2，或者直接遍历整张表。但这里的 <u>开销</u> 是针对查询过程，而不是插入操作。只要对有索引的属性进行了插入操作，那么就必须调用索引树的 insert 方法进行插入。而插入的开销是索引树自己定义的，与 Planner 无关。

因此，目前 Planner 与本工作无关，无须继续了解。



### 3. Data Structure



### 4. Operators

the Split operation may be propagated upwards



### 5. Integrate penaltyFn

通过前面的学习和摸索，我已经了解到 PSql 是通过在非叶子结点使用 `gistChoose()` 调用 `gistState->penaltyFn[N_KEYS]` 来计算 penalty，进而选择出开销最小的子树。但是对于 `penaltyFn` 我并没有在源码中找到它的 prototype。我能找到的就是它是从 RelData 中读取的，与单个属性绑定的函数地址。

```bash
# 1. 登陆并打开指定的数据库
$HOME/postgres/pg14/bin/psql -p 5432 sample
# 2. 获取调试进程ID
select pg_backend_pid();
```



#### 1) Function Manager (Fmgr)

PG 的函数都是通过 Fmgr 管理的，为了理解 penaltyFn 的定义方式，进一步使得我自定义的函数能够正确的被 Fmgr 管理，我要充分理解 Fmgr 的数据结构、接口与工作原理。

Fmgr 数据结构的核心是表示**查找函数的结果**，以及**传递的参数**。它定义了以下基本的数据结构：

- `FmgrInfo`: 从 `pg_proc` 中查找到的目标函数信息，包括目标函数的地址、OID、参数的个数等；

- `FunctionCallInfoBaseData`: 调用函数需要提供的信息，包括 FmgrInfo, 函数校验码 `fncollation`, 传入的参数 `args[]`. 每个参数都要求是非空的 `Datum` 类型。

  > `Datum`: 整型大小的变量，可以是整数，也可以是指针。在定义函数的时候，要求把返回值均统一为该类型。

能够被 Fmgr 调用的函数通常为如下的形式：

```c++
Datum function_name(PG_FUNCTION_ARGS)
{
	...
}
```

在了解了上述的接口之后，我们便可以理解 `gistpenalty(6)` 的执行逻辑，它是传入了 `giststate->penaltyFn[attno]` 指定的当前属性对应的 penalty 函数，接着传入了该函数需要的三个参数: `GISTENTRY *orig`, `GISTENTRY *add` 以及结果参数 `float penalty`.

```c
	if (giststate->penaltyFn[attno].fn_strict == false ||
		(isNullOrig == false && isNullAdd == false))
	{
		FunctionCall3Coll(&giststate->penaltyFn[attno],
						  giststate->supportCollation[attno],
						  PointerGetDatum(orig),
						  PointerGetDatum(add),
						  PointerGetDatum(&penalty));
		/* disallow negative or NaN penalty */
		if (isnan(penalty) || penalty < 0.0)
			penalty = 0.0;
	}
```

那么，我只需要对该函数调用进行修改，替换为自己的 penalty 函数即可。这里可以初步确定我所要实现的 `penalty()` 的 prototype:

```c
Datum penaltyRL(GISTENTERY* orig, GISTENTRY* add, float &penalty){
  ...
  penalty = calculatedPenalty;
  return DATUM_TRUE;
}
```

**🚩目标 1:** 是要理解各个参数的含义，如何通过 `GISTENTRY` 数据结构计算出状态向量。

#### 2) GistEntry

为了实现 `penaltyRL()`，我又回溯了一遍该项目的源码，从而进一步确定了该函数的实现逻辑：

1. **计算出状态向量 (`GetSortedInsertStates: RTree.cpp`)**

   状态向量包括 4 个分量: 面积增量、周长增量、重合面积增量、结点使用率。在 RLR-Tree 的实现中，树的结点类型 `RTree` 定义了 `Area()` 和 `Perimeter()` 方法。而其父类 `Rectangle` 提供了 `Set()` 和 `Include()` ，返回与其他 rectangle 进行操作后的新 rectangle。从而能计算出面积与周长的增量。

   重合面积指的是，新结点插入至当前结点后，当前结点与同级的 $k$ 个兄弟结点的重叠面积，需要累加 $k - 1$ 个 rectangle 的重叠面积。

   结点使用率的定义式为: `node->entry_num / TreeNode::maximum_entry`.

2. **获取最优 Action (`Test3: combined_model.py`)**

   最优的 action 通过调用训练得到的 agent，传入上一步得到的计算向量得到。这里的网络是单层的线性神经网络，会返回传入的每个 action 的评分，返回评分最高的 action.

   ```python
   action = self.insertion_network.chooseAction(states)
   ```

   可以将神经网络的输出取反，作为返回的 penalty 值.

**🚩目标 1.1:** 通过被插入结点和新结点的 `GISTENTRY` 数据结构，计算出面积、周长与重合面积的增量，以及结点的使用率。而该数据结构的定义如下：

```c
typedef struct GISTENTRY
{
	Datum			key;
	Relation	rel;
	Page			page;
	OffsetNumber offset;
	bool			leafkey; // whether it's a leaf node
}
```

对于 R-Tree， `GistEntry.key` 应当包含了区间信息，但是我并不知道它的数据类型。在查阅了 [PG 中 R-Tree 的实现](https://gis.stackexchange.com/questions/263370/postgis-r-tree-why-does-it-exist-when-postgresql-already-implements-its-r-tree) 后，我找到了 `NDBOX`，定义了 1 个 $n$ 维的数组，对应各个维度的长度：

```c
typedef struct NDBOX
{
	int32		vl_len_;
	unsigned int header;

	/*
	 * The lower left coordinates for each dimension come first, followed by
	 * upper right coordinates unless the point flag is set.
	 */
	double		x[FLEXIBLE_ARRAY_MEMBER];
} NDBOX;
```

例如点 $A(0, 1)$ 和 $B(3, 5)$ 分别作为对角构成的大小为 $3 \times 4$ 的 $2$  维包围盒线性表示为 $[0, 1, 3, 5]$. 从而如果 `ll` 的位置为 `i`，那么其对应的 `ur` 的索引应当为 `i + dim = i + 2`. 作者也提供了 `LL_CORD(2)` 与 `UR_CORD(2)` 的接口，可以直接使用，得到 LL 位置对应的 UR 坐标分量。

我也找到了用于 R-Tree 的 `penalty` 函数，以面积增量作为 penalty：

```c
/*
** The GiST Penalty method for boxes
** As in the R-tree paper, we use change in area as our penalty metric
*/
Datum
g_cube_penalty(PG_FUNCTION_ARGS)
{
	GISTENTRY  *origentry = (GISTENTRY *) PG_GETARG_POINTER(0);
	GISTENTRY  *newentry = (GISTENTRY *) PG_GETARG_POINTER(1);
	float	   *result = (float *) PG_GETARG_POINTER(2);
	NDBOX	   *ud;
	double		tmp1, tmp2;

	ud = cube_union_v0(DatumGetNDBOXP(origentry->key),
					   DatumGetNDBOXP(newentry->key));
	rt_cube_size(ud, &tmp1);
	rt_cube_size(DatumGetNDBOXP(origentry->key), &tmp2);
	*result = (float) (tmp1 - tmp2);

	PG_RETURN_FLOAT8(*result);
}
```

**🚩目标 1.2:** 以该函数为基础计算出其他三个分量，分别为周长增量、重合面积增量、结点使用率。

遍历 `maxoff` 个子结点，按照面积变化的增量选出前 $k$ 个增量最少的子结点，然后计算这些子结点的其它 3 个属性。从而，每个子结点都会有 1 个 4 维的状态向量，输入到神经网络中，就能输出 penalty 值。

所以，我们只需要为 `gist_penalty(6)` 加上 page 这个参数即可

​	**✅ 目标 1.2.1:** 在此之前，我要先能运行到这个函数处，确保之前的假设都是正确的。这需要插入较多的数据，使得树的高度 >= 3. 我下载了 [Airline Sample Databse](https://postgrespro.com/docs/postgrespro/11/demodb-bookings-installation)，并按照它的指示创建了给定的数据库:

```bash
$HOME/postgres/pg14/bin/psql -f /Users/shiqi/Downloads/demo-small-en-20170815.sql -U postgres
```

其中 `-U postgres` 指的是以 `postgres` 角色完成运行该文件。我们可以在 PG 命令行中通过 `\du` 查看系统中的角色和权限:

```bash
sample=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 shiqi     | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

可见，我们的系统中只存在 shiqi 这一唯一角色。所以，要将上述的指令修改为：

```bash
$HOME/postgres/pg14/bin/psql -f /Users/shiqi/Downloads/demo-small-en-20170815.sql -U shiqi -d sample
```

于是，我们通过角色 shiqi 进入了 sample 数据库，并创建了 `demo` 数据库。现在我们进入该数据库，并查看其中所有的 table:

```bash
demo=# \dt
             List of relations
  Schema  |      Name       | Type  | Owner 
----------+-----------------+-------+-------
 bookings | aircrafts_data  | table | shiqi
 bookings | airports_data   | table | shiqi
 bookings | boarding_passes | table | shiqi
 bookings | bookings        | table | shiqi
 bookings | flights         | table | shiqi
 bookings | seats           | table | shiqi
 bookings | ticket_flights  | table | shiqi
 bookings | tickets         | table | shiqi
(8 rows)
```

我们的目标是在一个 `point` 数据类型的 key 上建立 GiST 索引树，并使其层数 > 3. 使用 `\d <tablename>` 来查看该表的详细信息:

```bash
demo=# \d airports_data
                Table "bookings.airports_data"
    Column    |     Type     | Collation | Nullable | Default 
--------------+--------------+-----------+----------+---------
 airport_code | character(3) |           | not null | 
 airport_name | jsonb        |           | not null | 
 city         | jsonb        |           | not null | 
 coordinates  | point        |           | not null | 
 timezone     | text         |           | not null | 
Indexes:
    "airports_data_pkey" PRIMARY KEY, btree (airport_code)
    "airports_data_coordinates_idx" gist (coordinates) WITH (fillfactor='15')
Referenced by:
    TABLE "flights" CONSTRAINT "flights_arrival_airport_fkey" FOREIGN KEY (arrival_airport) REFERENCES airports_data(airport_code)
    TABLE "flights" CONSTRAINT "flights_departure_airport_fkey" FOREIGN KEY (departure_airport) REFERENCES airports_data(airport_code)
```

于是，我选定了 `airports_data` 中的 `coordinates` 属性。

```sql
INSERT INTO airports_data VALUES('SQA', '{"en": "Seventeens Airport"}', '{"en": "Singapore"}', (point '(250.2, 125.1)'), 'Asia/China');
```

然而在插入该元组时，程序还是直接跳到了叶子结点，可能是因为元组数量较少，而一个结点能容纳的元组数量却很多。因此，两层就可以装下所有的 90 多条数据结点，不需要内部结点，叶子结点就可以完成 GiST 树的构建。

为了增加层数，我选择调整 `FILLFACOTOR: gitst_private.h`，这个比例控制了结点的满载量。默认情况下，GiST 将这一常数设为 $90\%$，即仅当当前结点的容载量 > 90% 时，才会分裂结点。于是，我对该常数进行调整，设为 $15\%$，进一步观察 GiST 树的层数。

```c
#define GIST_DEFAULT_FILLFACTOR		15
```

但是，这样做的问题是仅仅在源代码中进行修改，是否真正影响了 build 后的程序呢？对这一问题需要进行以下探究：

- [x] 源代码与编译后程序的关系，是否有共享的源文件？

  -> 参考 CSAPP

  并没有。在编译后，可执行文件就与源代码脱离了关系，仅依赖于 build 中的文件。源代码的修改并不影响可执行文件。在查阅了 Psql 的文档后，发现可以通过在创建索引时指定 `fillfactor` 的方式来调整该参数。

  ```bash
  CREATE INDEX xxx ON xxx WITH fillfactor=15;
  ```

  ```sql
  # 1. 查看该table的属性以及索引信息
  $ \d airports_data
  # 2. 删除名称为xxx的索引
  $ DROP INDEX airports_data_coordinates_idx;
  # 3. 在cooidinate上创建索引，并指定fillfactor
  $ CREATE INDEX ON airports_data USING gist(coordinates) WITH (fillfactor=15);
  ```


然而，即使指定了 `fillfactor`，在插入结点后，树的层数似乎仍然为 2. 看来，我还是要装上 `gevel` 这个工具，看每层的使用情况。但是，在安装该扩展 `make` 的时候，还是产生了文件找不到的问题。我原本以为需要先了解 makeFile 的原理，才能解决这一问题。但是在与其它可用的 Makefile 文件对比后，我发现 `gevel` 的 MakeFile 基本只有 1 处不同，没有定义 `top_builddir`。在增加该变量的定义，并引入 `Makefile.global` 后，按照 [在PostgreSQL里安装gevel拓展](https://www.lixf.cc/2022/01/08/install-gevel-in-postgres/) 中排除了部分编译错误后，代码编译成功，并顺利在 `regression` 数据库中创建了这些函数:

```shell
contrib % $HOME/postgres/pg14/bin/psql regression < gevel/gevel.sql
SET
BEGIN
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
COMMIT
```

同时，在这一实验中，我也基本理解了 `make`, `makefile`, `cmake`, `make install` 的区别与意义，如下图所示：

![image-20230227204552185](https://raw.githubusercontent.com/Steven-cpp/myPhotoSet/main/image-20230227204552185.png)

- `make`: 就是根据 `makeFile` 将源代码编译为二进制文件，例如将 `main.c` 编译为 `main.o`。在此阶段，还会完成链接操作，例如对于头文件引用的解析，这可能会生成 `dylib` 文件。

  > 🔍 **链接阶段如何处理头文件 & dylib 的意义**
  >
  > 【TODO】看完 CSAPP Linking 后完成
  >
  > 

- `cmake`: 它是生成 build system 的工具，而 `make` 是 build system，通过输入的 `CMakeLists.txt` 指导编译器如何 build 你的代码。例如我们可以通过 cmake 生成特定平台的 makeFile，从而使得源代码具有跨平台的特性。

- `make install`: 将编译好的二进制文件复制到目标安装位置，例如 `/bin` 目录.

从而，编译整个项目的过程可以被解释为：先通过 `./configure` 定义安装位置及操作平台类型，生成 MakeFile 文件；接着 `make` 根据当前目录下的 makeFile，编译项目的源码，得到可执行的二进制文件，并进行链接操作，各个子目录下都产生了 `.o`, `.dylib` 的二进制文件；最后，通过 `make install` 将二进制文件复制到安装目录中，完成软件的安装。

​	**✅ 目标 1.2.2: 使用 gevel 调试 GiST 树，了解页面的数据结构分配** 

在 `airports_data` 表中，我们首先试着以默认 `fillfacotr = 90` 建立 GiST 索引树：

```sql
CREATE INDEX ON airports_data USING gist(coordinates);
```

然后，我们用 `gevel` 来查看其页面数据的分配:

```sql
>>> select * from gist_stat('airports_data_coordinates_idx');
gist_stat               
---------------------------------------
 Number of levels:                  1 +
 Number of pages:                   1 +
 Number of leaf pages:              1 +
 Number of tuples:                 104+
 Number of invalid tuples:  0         +
 Number of leaf tuples:   104         +
 Total size of tuples:     4588 bytes +
 Total size of leaf tuples: 4588 bytes+
 Total size of index:       8192 bytes+
```

接着，我们删除该索引，以 `fillfactor=15` 重新建立 GiST 索引树：

```sql
>>> DROP INDEX airports_data_coordinates_idx;
DROP INDEX
>>> CREATE INDEX ON airports_data USING gist(coordinates) WITH (fillfactor=15);
CREATE INDEX
```

然后，再查看其页面数据分配：

```sql
>>> select * from gist_stat('airports_data_coordinates_idx');
               gist_stat               
---------------------------------------
 Number of levels:                  1 +
 Number of pages:                   1 +
 Number of leaf pages:              1 +
 Number of tuples:                 104+
 Number of invalid tuples:  0         +
 Number of leaf tuples:   104         +
 Total size of tuples:     4588 bytes +
 Total size of leaf tuples: 4588 bytes+
 Total size of index:       8192 bytes+
```

结果还是不变，看来是数据集太小了，即使是最低的 fillfactor，也能在 level 1 装下所有的元素 (我们称根结点所在的高度为 level 0). 所以，我们要选用更大的数据集。但是，更大的数据集并没有增广机场的坐标，自始至终只有 104 个机场的坐标，因此我只能考虑通过脚本插入随机的坐标，自行增广数据。

![image-20230125203939439](https://raw.githubusercontent.com/Steven-cpp/myPhotoSet/main/image-20230125203939439.png)

ChatGPT 推荐使用 `psycopg2` 来连接 PG 数据库并插入数据，明天再来试试。

​	**✅ 目标 1.2.3: 使用 psycopg2 连接 PG 数据库，写脚本增广数据**

在增加了 101 条数据后，表中有了 205 行，此时的 GiST 树已经多了一层，我们的目标已经达成，可以进行下一步的工作:

```bash
demo=# select * from gist_stat('airports_data_coordinates_idx');
               gist_stat                
----------------------------------------
 Number of levels:                  2  +
 Number of pages:                   3  +
 Number of leaf pages:              2  +
 Number of tuples:                 209 +
 Number of invalid tuples:  0          +
 Number of leaf tuples:   207          +
 Total size of tuples:     9232 bytes  +
 Total size of leaf tuples: 9132 bytes +
 Total size of index:       24576 bytes+
```



#### 3) gist_insert

由于我要确认什么情况下需要计算 penalty 以及对哪些目标结点计算，根据 `gist_insert()` 的源码，可以发现：

1. **何时计算 penalty**

   只要插入需要经过 internal node，就一定会调用 `gist_choose()`，该操作需要返回最优的 tuple，这是通过计算每一个 tuple 的 penalty 实现的。也就是返回将该点插入到 tuple 中的面积增量.

2. **对哪些结点计算 penalty**

   internal node 中的每一个 tuple.

所以，我需要更改的是 `gist_choose()` 函数，根据原有的 penalty_fn，对每一个 tuple 计算出 $\Delta S$，然后选出前 $2$ 个 tuple，计算出两者的状态向量，选出最优的 action，然后返回其 offsetNumber。

源代码中还考虑到了建立在多个属性上的索引的情况，本实现暂不考虑。即本实现默认 `key = 0`.

### 6. ONNX Runtime

为了将 Pytorch 模型部署到 C 代码中，这里考虑通过 ONNX Runtime 加载预训练好的神经网络，作为 action module。

[参考: 使用 ONNX Runtime 将 Pytorch 模型部署至 C++ 应用中](https://medium.com/@freshtechyy/deploying-pytorch-model-into-a-c-application-using-onnx-runtime-f9967406564b)

Open Neural Network eXchange（ONNX）是一种用于存储预训练模型的文件格式。它允许各种 AI 框架以相同的格式存取。而 ONNX Runtime 是一个跨平台的机器学习模型加速器，它具有灵活的接口，可以集成特定的硬件库。ONNX Runtime能够使用多种硬件执行神经网络模型，如 CPU、CUDA 和 TensorRT 等。它还可以兼容多种模型，如 PyTorch、Tensorflow/Keras、TFLite、scikit-learn等。

我们可以利用 ONNX Runtime (ORT) 将预先训练好的 PyTorch 模型部署到 C++ 应用程序中进行推断，如下图所示：

![image-20230221144131233](https://raw.githubusercontent.com/Steven-cpp/myPhotoSet/main/image-20230221144131233.png)

#### 1）模型导出为 ONNX 文件格式

由于 ORT 只能接受 `.onnx` 的文件格式，因此需要先将 PyTorch 中预训练得到的 `.pth` 模型文件转换为该格式。我们可以在 Pytorch 中实现这一转换：

```python
# 1. Define the model
net = Model()
net.cuda()

# 2. Load pretrained model file
PATH = 'model.pth'
net.load_state_dict(torch.load(PATH))
net.eval()

# 3. Give shape-matched input, init the model, and then export
x = torch.randn((1, 3, 32, 32)).cuda()
torch.onnx.export(net,       # model being run       
                  x,         # model input
                  "model.onnx",  # where to save the model
                  export_params=True, # store the trained weights
                  opset_version=11,   # the ONNX version
                  do_constant_folding=True,
                  input_names= ['input'], # set model input names    
                  output_names=['output'], # set model output names
)
```

#### 2）使用 ORT 进行推断

接下来，就可以在 PG 中使用导出的 Pytorch 模型了。这包括了两个步骤：

1. **初始化**

   创建 ORT 的运行环境，ORT Session，并设定其参数，包括模型输入/输出的个数、类型、形状以及名字。

   ```c
   OrtEnv* env;
   OrtCreateEnv(ORT_LOGGING_LEVEL_WARNING, "test", &env);
   OrtSessionOptions* session_options;
   OrtCreateSessionOptions(&session_options);
   OrtSession* session;
   OrtCreateSession(env, "model.onnx", session_options, &session);
   ```

2. **推断**

   创建输入、输出向量，并分配内存，接着通过 `OrtRun()` 传入上述的所有参数，即可完成推断过程。

   ```c
   // Create an input tensor object
       OrtMemoryInfo* input_tensor_info;
       OrtCreateCpuMemoryInfo(OrtDeviceAllocator, OrtMemTypeCPU, &input_tensor_info);
       OrtValue* input_tensor;
       OrtCreateTensorWithDataAsOrtValue(input_tensor_info, input_data, sizeof(float) * 1 * 3 * 224 * 224, input_shape, 4, ONNX_TENSOR_ELEMENT_DATA_TYPE_FLOAT, &input_tensor);
   
       // Pass the input tensor through the ONNX model to get the output tensor
       char* input_name = OrtSessionGetInputName(session, 0, input_tensor_info);
       char* output_name = OrtSessionGetOutputName(session, 0, input_tensor_info);
       OrtValue* output_tensor;
       OrtRunOptions* run_options;
       OrtCreateRunOptions(&run_options);
       OrtRun(session, run_options, input_name, &input_tensor, 1, &output_name, 1, &output_tensor);
   ```

3. **处理输出向量**

   由于上一步创建的输入、输出向量都是 `OrtValue` 的类型，不便于读取，还需要通过 `OrtGetTensorMutableData()` 将其转为基本数据类型（例如 `float`）：

   ```c
   // Process the output tensor to obtain the predicted class
   float* output_data;
   OrtGetTensorMutableData(output_tensor, (void**)&output_data);
   int predicted_class = 0;
   float max_score = -1;
   for (int i = 0; i < 5; i++) { // assuming there are 5 classes
     if (output_data[i] > max_score) {
       predicted_class = i;
       max_score = output_data[i];
     }
   }
   ```

   上面给出了一个 5-分类问题的示例代码。

基于上述几个步骤，我们就可以定义得到 PG 中的 Action Module.

#### 3）安装 ORT 至 PG 项目中

现在的问题就是如何将 ORT 的框架整合至本项目中，并且完成编译。经过一系列毫无方向的探索，我想不如从理论出发，看从 C++ 项目的通用编译过程上，如何实现这样的一个操作。于是，我想到可以有 2 种方式：

1. **与 PG 源码一起编译**

   可以通过将 ORT 的源码 copy 至 PG 的运行目录，然后结合 ORT 的 `cmakeLists.txt` 修改 PG 的各种 `makeFile`。但由于 `cmakeLists` 与 `makeFile` 语法不同，因此难以进行对应。于是，我只好考虑另一种方法。

2. **将 ORT 作为链接库**

   直接下载编译好的 ORT，然后只要在 `make` 时指定链接到对应的 lib 即可。可以参考 [Compiling and Linking with Installed Libraries](http://vis.cs.brown.edu/resources/doc/gfxtools-docs/using-installed-libs.html).

































 
