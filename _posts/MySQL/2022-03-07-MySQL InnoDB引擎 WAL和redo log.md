---
title: "MySQL WAL策略和checkpoint"
subtitle: ""
layout: post
author: "Geooo"
header-img: ""
header-style: text
tags:
  - MySQL
  
---

## InnoDB 体系架构


### InnoDB 关键特性


在说 WAL 之前，有必要简单介绍下 InnoDB 存储引擎的体系架构，方便我们理解下文，并且 **redo log** 也是 **InnoDB 存储引擎所特有的**。

如下图，InnoDB 存储引擎由**内存池**和一些**后台线程**组成：

![](https://gitee.com/veal98/images/raw/master/img/20210629165703.png )

### 内存池 buffer pool
InnoDB 存储引擎是基于磁盘存储的，并将其的记录按照**页**的方式进行管理，因此可以将其视为 **基于磁盘的数据库系统 (Disk-base DataBase)**，因为磁盘IO读取的速度特别慢，与CPU的速度不匹配，通常使用**缓冲池**技术来提高数据库的整体性能。

具体来说，缓冲池其实是一块内存区域，在CPU和磁盘之间加入内存访问，通过内存的速度来弥补磁盘速度较慢对数据库的影响。

InnoDB中 **读取页** 操作的具体步骤是这样的：
- 首先将从磁盘中读取到的页放入缓冲池中
- 下次再读到相同页的时候，首先判断该页是否在缓冲池中，若在缓冲池中，称该页在缓冲池中被命中，直接读取该页。否则从磁盘中读取B+树。

InnoDB中 **修改页** 具体步骤是这样的：
- 首先修改缓存池  **buffer pool** 中的页，然后再以一定的频率刷新到磁盘上。
（如果是**普通索引**数据页不在缓存池中，则会先写入 **change buffer**中，等到数据页加载到缓存池中的时候就会进行 **merge** 操作）

**脏页：** 加载到缓存池 buffer pool中的数据页被修改，但是还没刷到磁盘上，我们就成缓冲池中的这些页叫做 **"脏页"**，即缓冲池中页的版本要比磁盘数据的新。

### 后台线程

后台线程最大的作用就是来完成 **"将磁盘读到的页存放在缓冲池中"** 以及 **"将缓冲池中的数据页以一定的频率刷新到磁盘上"**。
> 《MySQL 技术内幕：InnoDB 存储引擎 - 第 2 版》 对后台线程解释：后台线程主要作用是刷新内存中的数据，保证内存的数据是最新的数据；此外将已经修改的内存数据刷新到磁盘文件中，同时保证当数据库发生异常的时候，InnoDB能够恢复到正常运行状态。

另外，InnoDB 存储引擎是多线程的模型，也就是说它拥有多个不同的后台线程，负责处理不同的任务。这里简单列举下几种不同的后台线程：

- **Master Thread**：主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性
- **IO Thread**：在 InnoDB 存储引擎中大量使用了 AIO（Async IO）来处理写 IO 请求，这样可以极大提高数据库的性能。IO Thread 的工作主要是负责这些 IO 请求的回调（call back）处理
- **Purge Thread**：回收已经使用并分配的 undo 页
- **Page Cleaner Thread**：将之前版本中脏页的刷新操作都放入到单独的线程中来完成。其目的是为了减轻原 Master Thread 的工作及对于用户查询线程的阻塞，进一步提高 InnoDB 存储引擎的性能

### redo log 与 WAL策略

MySQL为了避免数据丢失，采取了 WAL机制，当前事务数据库系统（并非 MySQL 所独有）普遍都采用了 WAL（Write Ahead Log，预写日志）策略：即当事务提交时，先写重做日志（redo log），再修改页（先修改缓冲池，再刷新到磁盘）；当由于发生宕机而导致数据丢失时，通过 redo log 来完成数据的恢复。这也是事务 ACID 中 D（Durability 持久性）的要求。

有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。

- redo log file 不能设置得太大，如果设置得很大，在恢复时可能需要很长的时间
- redo log file 又不能设置得太小了，否则可能导致一个事务的日志需要多次切换重做日志文件

### CheckPoint 技术

因此 Checkpoint 技术的目的就是解决上述问题：

- 缓冲池不够用时，将脏页刷新到磁盘
- redo log 不可用时，将脏页刷新到磁盘
- 缩短数据库的恢复时间

所谓 CheckPoint 技术简单来说其实就是在 redo log file 中找到一个位置，将这个位置前的页都刷新到磁盘中去，这个位置就称为 CheckPoint（检查点）。

**1）缩短数据库的恢复时间**：当数据库发生宕机时，数据库不需要重做所有的日志，因为 Checkpoint 之前的页都已经刷新回磁盘。故数据库只需对 Checkpoint 后的 redo log 进行恢复就行了。这显然大大缩短了恢复的时间。

**2）缓冲池不够用时，将脏页刷新到磁盘**：所谓缓冲池不够用的意思就是缓冲池的空间无法存放新读取到的页，这个时候 InnoDB 引擎会怎么办呢？LRU 算法。 InnoDB 存储引擎对传统的 LRU 算法做了一些优化，用其来管理缓冲池这块空间。

总的思路还是传统 LRU 那套，具体的优化细节这里就不再赘述了：即最频繁使用的页在 LRU 列表（LRU List）的前端，最少使用的页在 LRU 列表的尾端；当缓冲池的空间无法存放新读取到的页时，将首先释放 LRU 列表中尾端的页。这个被释放出来（溢出）的页，如果是脏页，那么就需要强制执行 CheckPoint，将脏页刷新到磁盘中去。

**3）redo log 不可用时，将脏页刷新到磁盘**：

所谓 redo log 不可用就是所有的 redo log file 都写满了。但事实上，其实 redo log 中的数据并不是时时刻刻都是有用的，那些已经不再需要的部分就称为 ”可以被重用的部分“，即当数据库发生宕机时，数据库恢复操作不需要这部分的 redo log，因此这部分就可以被覆盖重用（或者说被擦除）。

![](https://gitee.com/veal98/images/raw/master/img/20210629222559.png )

**write pos 和 CheckPoint 之间的就是 redo log file 上还空着的部分，可以用来记录新的操作。**

### 有了 bin log 为什么还需要 redo log？

binlog 日志只能用于归档，因此 binlog 也被称为归档日志，显然如果 MySQL 只依靠 binlog 等这四种日志是没有 crash-safe 能力的，所以为了弥补这种先天的不足，得益于 MySQL 可插拔的存储引擎架构，InnoDB 开发了另外一套日志系统 — 也就是 redo log 来实现 crash-safe 能力。
(crash safe 数据页加载到内存的时候，InnoDB会直接update buffer pool 的数据页，不会刷新磁盘，所以，bin log是不知道)
