# Optimizing R-Tree with RL strategy

这是我选的 Master Project，想要将 [RLR-Tree](https://arxiv.org/abs/2103.04541) 的索引树更新策略在 PostgreSQL (PG) 中的 R-Tree 中实现，从而能大大提高索引树的查询表现。

![image-20230130171817852](https://raw.githubusercontent.com/Steven-cpp/myPhotoSet/main/image-20230130171817852.png)

根据原作者在包含超过 1 亿多个空间对象的数据集中的实验，RLR-Tree 的相对 I/O 开销相比于 R-Tree 能够减少 $70\%$ 以上。

## Prerequisite Knowledge

该项目的实现需要深入理解 R-Tree、RLR-Tree，以及 PG 中 R-Tree 的实现，具体地，在阅读 PG 源码前需要以下前置知识:

- **索引树的结构与算法**

  包括 R-Tree 的结构与增删改算法、RLR-Tree 的增删改算法、PG 中 GiST 索引的架构与增删改算法。

- **索引树在硬盘中的存储结构**

  在 PG 的 GiST 实现中，索引树的每个<u>结点</u>都在一个 page 中，而每个 page 含有多个 offset，每个 offset 就对应该结点中的<u>数据项</u> (indexTuple)。

## Build Instruction

本项目是在 PG 的源码基础之上修改的，因此 build 该项目的过程就类似于 build PG 的源码。下面给出了我在 M1 Mac 上，使用 VSCode build 本项目源码的过程。

### Dependencies

在开始之前，需要确保以下软件已被安装:

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

   ```
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
   ```

   然后可以看到类似于下面的输出:

   > Success. You can now start the database server using:
   >
   > /Users/me/postgres/pg14/bin/pg_ctl -D /Users/me/postgres/pgdata -l logfile start

7. 现在我们就可以启动 PG 了

   ```bash
   $HOME/postgres/pg14/bin/pg_ctl -D $HOME/postgres/pgdata -l logfile start
   ```

   > waiting for server to start…. done
   > server started

8. 最后，创建并打开一个数据库

   ```bash
   $HOME/postgres/pg14/bin/createdb -p 5432 sample
   $HOME/postgres/pg14/bin/psql -p 5432 sample
   ```

   我们现在应该登录到了 PG 中，我们可以尝试创建一个 `sample` 表并尝试一些查询

   ```sql
   create table hello(id int, message text);
   insert into hello values(1, 'Hello world!');
   select * from hello;
   ```

## Development Progress

























