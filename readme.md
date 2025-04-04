# Optimizing R-Tree with RL strategy

这是我选的 Master Project，想要将 [RLR-Tree](https://arxiv.org/abs/2103.04541) 的索引树更新策略在 PostgreSQL (PG) 中的 R-Tree 中实现，从而大大提高索引树的查询表现。

![image-20230130172101269](https://raw.githubusercontent.com/Steven-cpp/myPhotoSet/main/image-20230130172101269.png)

根据原作者在包含超过 1 亿多个空间对象的数据集中的实验，RLR-Tree 的相对 I/O 开销相比于 R-Tree 减少了 $70\%$ 以上。

## Prerequisite Knowledge

该项目的实现需要深入理解 R-Tree、RLR-Tree，以及 PG 中 R-Tree 的实现，具体地，在阅读 PG 源码前需要以下前置知识:

- **索引树的结构与算法**

  包括 R-Tree 的结构与增删改算法、RLR-Tree 的增删改算法、PG 中 GiST 索引的架构与增删改算法。

- **索引树在硬盘中的存储结构**

  在 PG 的 GiST 实现中，索引树的每个<u>结点</u>都在一个 page 中，而每个 page 含有多个 offset，每个 offset 就对应该结点中的一个<u>数据项</u> (indexTuple)。

## Build Instruction

本项目是在 PG 的源码基础之上修改的，因此 build 该项目的过程就类似于 build PG 的源码。下面给出了我在 M1 Mac 上，使用 VSCode build 本项目源码的过程。

### Dependencies

在开始之前，需要确保安装了以下工具:

1. Git
2. GCC compiler
3. make
4. VS Code

### Build and install opt-gist

1. 首先，克隆本项目的源码

   ```bash
   git clone https://github.com/Steven-cpp/postgres-optGist.git
   ```

2. 之后，cd 进入文件夹 `postgres-optGist`

   ```bash
   cd postgres-optGist
   ```

3. 为了配置如何 build 本项目，可以通过运行 `./configure` 文件来指定

   ```bash
   ./configure --prefix=$HOME/postgres/pg14 --enable-cassert \
   --enable-debug  CFLAGS="-ggdb -O0 -fno-omit-frame-pointer" CPPFLAGS="-g -O0"
   ```
   这些配置包括:
   - prefix - PostGres 的安装目录
   - enable-debug - 以 Debug 模式 build 本项目
   - CFLAGS - C compiler flags
     - -O0 - 不允许编译器自优化
     - -g - 产生 Debug 符号表
   - CPPFLAGS - C++ compiler flags

4. `configure` 脚本将做一些检查并配置 build 文件，输出结果类似于这样:

   > config.status: linking src/include/port/darwin.h to src/include/pg_config_os.h config.status: linking src/makefiles/Makefile.darwin to src/Makefile.port

   随后，`Makefile.global` 会在 `src` 目录下被创建。

5. 运行 `make && make install` 指令 buid 该项目，并将可执行程序安装至指定的目录中 (`$HOME/postgres/pg14`)

   ```bash
   make -j4 && make install
   ```

6. 为数据库初始化 `PGDATA` 文件夹，实际创建的数据库会存储在该文件夹下

   ```bash
   $HOME/postgres/pg14/bin/initdb -D $HOME/postgres/pgdata
   
   Success. You can now start the database server using:
   /Users/me/postgres/pg14/bin/pg_ctl -D /Users/me/postgres/pgdata -l logfile start
   ```
   
7. 现在我们就可以启动 PG 了

   ```bash
   $HOME/postgres/pg14/bin/pg_ctl -D $HOME/postgres/pgdata -l logfile start
   
   waiting for server to start…. done
   server started
   ```

8. 最后，创建并打开一个数据库

   ```bash
   $HOME/postgres/pg14/bin/createdb -p 5432 sample
   $HOME/postgres/pg14/bin/psql -p 5432 sample
   ```
   我们现在应该登录到了 PG 中，可以尝试创建一个 `sample` 表并尝试一些查询

   ```sql
   create table hello(id int, message text);
   insert into hello values(1, 'Hello world!');
   select * from hello;
   ```

## Development Progress

为了实现 RLR-Tree，其实只需要重写 `gist_penalty()` 和 `pick_split()` 这两个方法即可。虽然可以参考 RLR-Tree 的源码，但仍然有两大难点:

1. **数据结构完全不同**

   RLR-Tree 是用 C++ 实现的，定义了树结点类 `RTree`，提供了 `Area()` 和 `Perimeter()` 方法。并且在类成员变量中维护了 `sortedChildNode[]` 数组，来记录 top-K 结点。

   而 PG 则是用 C 实现的，对于数据类型、内存、异常处理的管理非常精细，涉及到操作系统级别的细节，例如 buffer, page, block；绝大部分的数据类型、函数都是通过 pointer 传送，在读取时转换为相应的数据类型，这就给理解源代码及调试程序造成了相当大的困难。

   两者实现树的数据结构完全不一样，要求我必须要能够先看懂并理解 PG 中 GiST 的源码，然后再通过给定的接口函数，实现 RLR-Tree 的逻辑。这有可能涉及到代码的重构。

2. **神经网络如何加载**

   RLR-Tree 是以 C++ 实现 R-Tree 的各种操作，然后在 Python 中使用这一数据结构，并构建神经网络进行训练。那么如何在 C 中实现这一点呢？ 

   💡在 C 中，可以使用 `ONNX Runtime` 加载 Pytorch 训练好的模型。从而，只需计算出输入向量，传入到 ONNX Runtime 中推理即可。


### Implement gist_penalty()

为了实现 `gist_penalty()`，我又回溯了一遍 RLR-Tree 的源码，进一步确定了该函数的实现逻辑：

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

目前，计算状态向量的代码已经基本完成 [见 gistutil.c: 776](https://github.com/postgres/postgres/commit/8a02e8c04e4f8c702d4ee908188d4f97bed43f8d#diff-8eafa3bd09e22563db7e3689a6c419ce700ff90f0486c582fe1701d71571a110)。在 top-K 结点的处理上，先采用最为简单的方法进行实现，在计算其它三个分量前，先尝试对所有 tuple 插入该结点，并计算出面积增量，如下图所示。

![image-20230130204224428](https://raw.githubusercontent.com/Steven-cpp/myPhotoSet/main/image-20230130204224428.png)

接着，对数组进行排序，选择前 $k$ 个最小的 tuple 计算分量。如下图所示。然后对数组进行排序，选出增量最小的前 $k$ 个 tuple。再遍历这 $k$ 个 tuple，计算其余的 3 个分量。而且，为了规避判断条件的复杂性，这里取 $k = 2$ 。之后，再对该算法的性能进行评估，如果没有提升的话，再考虑针对耗时的操作进行优化。

### Integrate ONNX Runtime

我们可以利用 ONNX Runtime (ORT) 将预先训练好的 PyTorch 模型部署到 C/C++ 应用程序中进行推断，如下图所示：

![image-20230221144131233](https://raw.githubusercontent.com/Steven-cpp/myPhotoSet/main/image-20230221144131233.png)

但最大的问题就是如何将 ORT 的框架整合至本项目中，并且完成编译。由于官网对 C 的 build instruction 非常有限，特别是在 Mac 平台上。我只能参考 [onnxruntime-c-inference](https://github.com/microsoft/onnxruntime-inference-examples/blob/main/c_cxx/README.md)的构建过程， 这里是在 makefile 中指定，在编译时 link ORT 的 lib。

于是，可以尝试修改 PG 的各种 `makeFile` 以及 `cMakeList`，将该 ORT link 到 PostgreSQL 的代码中。









