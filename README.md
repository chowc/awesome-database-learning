SQL 语句就是 Operator 树。https://www.infoq.cn/article/0rSVq2VIfUE0YLedLe5o

绑定器的作用就是将语法树通过和数据库的元数据(metadata)结合，为它附上语义(semantic)。比如语句里有 SELECT…FROM student，绑定器会去查询元数据确认 student 表是否存在;如果存在，是否有 class 和 id 两个属性;对于属性的后续操作是否符合规则-比如，对于 SUBSTR()这个方法，输入表达式必须是字符串类型等的一系列检查。

假如一个 SQL 语句包含 10 个表的联合，这 10 个表可以相互两两联合形成中间表(intermediate result)，这些中间表还需要再一次进行两两联合，然后再继续。并且，每一次联合有两种选择(table1 join table2 或者 table2 join table1)，而且联合对应的物理操作符又有好几个(HashJoin 或者 MergeSortJoin 注：在讲 Join operator 的时候会深入讲解)。这样一来，一个复杂的查询语句对应上百万个执行树就不难理解了。

- iterator model(迭代模式)或者叫 Volcano model(火山模式): operator 一次返回一个结果给上层；
- materialization: 一次性返回所有结果；

并不是所有的操作都适用于流模型，比如处理 order by 语句的 SortOperator，如果要对全部输入进行排序，必须等到所有输入都得到后才能进行排序，因此执行过程会堵塞(block)。-> 将下层返回的数据缓存到内存中，读取完成后再进行操作。如果内存放不下全量数据，就需要 split to disk。

- 知识点向量模式(vectorization model)，或者叫批处理模式(batch model)：一次返回批量数据；-> 减少函数调用；适合向量运算 SIMD；比较适用于数据量很大的 OLAP 查询语句。

数据库内核杂谈（四）：执行模式
原文链接： https://www.infoq.cn/article/spfiSuFZENC6UtrftSDD

### 基于时间戳进行加锁控制：控制并发、事务有序性

对于事务 Ti 要读取数据 A read(A):

如果 TS(Ti) < W-timestamp(A)，说明 A 被一个 TS 比 Ti 更大的事务改写过，但 Ti 只能读取比自身 TS 小的数据。因此 Ti 的读取请求会被拒绝，Ti 会被回滚。

如果 TS(Ti) > W-timestamp(A)，说明 A 最近一次被修改小于 TS(Ti)，因此读取成功，并且，R-timestamp(A)被改写为 TS(Ti)。

对于事务 Ti 要修改数据 A write(A):

如果 TS(Ti) < R-timestamp(A)，说明 A 已经被一个更大 TS 的事务读取了，Ti 对 A 的修改就没有意义了，因此 Ti 的修改请求会被拒绝，Ti 会被回滚。

如果 TS(Ti) < W-timestamp(A)，说明 A 已经被一个更大 TS 的事务修改了，Ti 对 A 的修改也没有意义了，因此 Ti 的修改请求会被拒绝，Ti 会被回滚。

其他情况下，Ti 的修改会被接受，同时 W-timestamp(A)会被改写为 TS(Ti)。

对于写操作，可以进一步优化：当 Ti 要写 A 的时候，如果 A 已经被更大的事务 Tj 写了，此时 Ti 可以直接跳过这个写操作，而不需要回滚，因为此时 Ti 的写操作已经不会被其他任何事务看到了（新事务只能读到 Tj 或者比 Tj 更大的事务的修改；） -> 托马斯的修改规则

https://www.infoq.cn/article/KyZjpzySYHUYDJa2e1fS

### 将语句拆分为 pipeline，提高执行效率

首先，文章定义了一个新的概念来辅助介绍这种执行模式，pipeline-breaker。Pipeline-breaker 的定义是，在执行语句中的某一个算子，如果它的执行逻辑需要把一个待处理的 tuple 从 CPU 寄存器中去除，那这个算子就被定义成一个 pipeline-breaker。如果一个算子需要等待子算子把所有的 tuple 都送给它，才能处理数据，那这个算子就被定义成 full-pipeline-breaker。（当然，实际情况中，可能某个 tuple 已经大到 CPU 寄存器无法存下，文章这边做了个小假设， 假设有足够的寄存器）。

有了这个定义后，下一个问题就是，如何才能一直将数据放在寄存器中呢？上面介绍的传统火山模型显然是做不到这一点：因为火山模型算子之间通过方法调用来传递 tuple，方法调用牵涉到方法栈的更新，数据早就被移出寄存器了。本文提到的方法就是，通过不断地把数据 push 给下一个要处理的算子，直到遇到 pipeline-breaker。但这个解释不直观。我的理解就是，对于某一个 tuple 数据，把要对其进行处理的算子（直到遇到 pipeline-breaker）排成队，依次对数据进行操作，在这个 pipeline 处理过程中，数据始终是存放在 CPU 寄存器中，因此，执行一个 pipeline 是非常高效的。而根据 pipeline-breaker，一个执行计划就会被 pipeline-breaker 算子分解成几个 pipeline，数据在这个维度下，总是从一个 pipeline，被处理完，进入另一个 pipeline。

https://www.infoq.cn/article/cywqRf9UsBG8DBZCiidX

*Usually, latches are used to guarantee physical consistency of data, while locks are used to assure logical consistency of data.*

latch 是编程上的锁，如 synchronized，lock 是概念上的锁，如分布式锁。

With hierarhical locking, the intention locks(IX, IS, and SIX) are generally obtained on the higher levels of the hierarchy (e.g. table), and the S and X locks are obtained on the lower levels(e.g. record). The nonintention mode locks(S and X), when obtained on an object at a certain level of the hierarchy, implicitly grant locks of the corresponding mode on the lower level objects of that higher level object. The intention mode locks, on the other hand, only give the privilege of requesting the corresponding intention or nonintention mode locks on the lower level objects. For example, SIX on a table *implicitly* grants S on all the records of that table, and it allows X to be requested explicitly on the records. 

---
# Awesome Database Learning

A list of learning materials to understand databases internals, including but not limited to:

- papers
- blogs
- courses
- talks

Please submit a pull request if there is any material that you think should be included in this collection.

## Table of Contents

<!-- vim-markdown-toc GFM -->

* [Recommended Courses, Books and Talks](#recommended-courses-books-and-talks)
    * [Courses](#courses)
    * [Books](#books)
    * [Talks](#talks)
    * [Blogs](#blogs)
* [SQL & Relation Algebra](#sql--relation-algebra)
* [Query Optimizer](#query-optimizer)
    * [Planner Models](#planner-models)
    * [Subquery Optimization](#subquery-optimization)
    * [Join Order Optimization](#join-order-optimization)
    * [Functional Dependency & Physical Properties](#functional-dependency--physical-properties)
    * [Cost Model](#cost-model)
    * [Statistics](#statistics)
* [Query Execution](#query-execution)
    * [Execution Framework](#execution-framework)
    * [Vectorization vs Compilization](#vectorization-vs-compilization)
    * [Join](#join)
    * [Hash Table](#hash-table)
    * [Bloom Filter](#bloom-filter)
* [DDL](#ddl)
* [Relational Model](#relational-model)
    * [Codd's Rules](#codds-rules)
    * [Relational Data Model](#relational-data-model)
    * [Relational Algebra](#relational-algebra)
    * [ER to Relational Model](#er-to-relational-model)
    * [SQL - Overview](#sql---overview)
* [Transaction](#transaction)
    * [Isolation Levels](#isolation-levels)
    * [Concurrency Control](#concurrency-control)
* [Network](#network)
* [Storage](#storage)
    * [NoSQL Systems](#nosql-systems)
    * [Buffer Management](#buffer-management)
    * [Disk IO](#disk-io)
    * [B-Tree](#b-tree)
    * [LSM-Tree](#lsm-tree)
    * [Learned Indexes Structures](#learned-indexes-structures)
* [Serializing & RPC](#serializing--rpc)
* [Data Partitioning](#data-partitioning)
* [Replication & Consistency](#replication--consistency)
* [Consensus](#consensus)
* [Scheduling](#scheduling)
* [Benchmark & Testing](#benchmark--testing)
* [HTAP](#htap)
* [TLA+](#tla)

<!-- vim-markdown-toc -->

## Recommended Courses, Books and Talks

### Courses

- [x] CMU [Database Systems (15-445/645)](https://15445.courses.cs.cmu.edu/fall2019/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
- CMU [Advanced Database Systems (15-721)](https://15721.courses.cs.cmu.edu/spring2020/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
- UC Berkeley [Introduction to Database Systems](https://cs186berkeley.net/calendar/)
- Stanford [Database System Implementation](https://web.stanford.edu/class/cs346/2015/)
- Cornell [Introduction to Database Systems](http://www.databaselecture.com/) by Prof. Trummer
- [x] [Let's Build a Simple Database](https://cstack.github.io/db_tutorial/), thanks to [cstack](https://github.com/cstack)

### Books

- Stanford [Database Systems: The Complete Book](http://infolab.stanford.edu/~ullman/dscb.html)
- [x] [Designing Data-Intensive Applications](http://shop.oreilly.com/product/0636920032175.do), [中文翻译](https://github.com/Vonng/ddia)
- [Database Internals](https://www.oreilly.com/library/view/database-internals/9781492040330/)
- [Foundations of Databases](http://webdam.inria.fr/Alice/)
- [Readings in Database Systems, 5th Edition](http://www.redbook.io/)
- [Database Design and Implementation: Second Edition (Data-Centric Systems and Applications)](https://www.amazon.com/dp/3030338355)
- [Principles of Distributed Database Systems, 4th ed](https://www.amazon.com/dp/3030262529)
- [x] [Inside SQLite](https://books.google.com/books/about/Inside_SQLite.html?id=QoxUx8GOjKMC)
- [Architecture of a Database System](https://dsf.berkeley.edu/papers/fntdb07-architecture.pdf) -> Section 4

1. 70s 操作系统对多线程的支持有限或者实现不够高效，因此数据库大多采取 process-per worker / workers on process pool 或者自己实现用户层的线程；这些都沿袭至今：
   1. postgres: process-per worker;
   2. oracle 默认 process-per worker，或者是 process pool;
2. NUMA hardware architectures are an interesting middle ground between shared-nothing and shared-memory systems. They are much easier to program than shared-nothing clusters, and also scale to more processors than shared-memory systems by avoiding shared points of contention such as shared-memory buses.
3. Often the memory of large shared memory multi-processors is divided into sections and each section is associated with a small subset of the processors in the system. Each combined subset of memory and CPUs is often referred to as a pod. Each processor can access local pod memory slightly faster than remote pod memory. This use of the NUMA design pattern has allowed shared memory systems to scale to very large numbers of processors. As a consequence, NUMA shared memory multi-processors are now very common whereas NUMA clusters have never achieved any significant market share.
4. shared-memory、shard-disk、share-nothing，第一个几乎所有 DMBS 都支持，后面两个则出现分化；
5. Oracle avoids moving rows in heap files by allowing rows to span pages. So, when a row is updated to a longer value that no longer fits on the original page, rather than being forced to move the row, they store what fits in the original page and the remainder can span to the next.
6. 将过滤条件放到数据访问层，避免了 pin/unpin page 和 tuple copy；
7. Storage Management
   1. raw disk: bypass the file system, need to handle different "disk interface"; the rising of "virtual disk";
   2. create a big file and put related data in close offset;
8. In the standard usage model, the system administrator creates a file system on each disk or logical volume in the DBMS. The DBMS then allocates a single large file in each of these file systems and controls placement of data within that file via low-level interfaces like the mmap suite. The
DBMS essentially treats each disk or logical volume as a linear array of (nearly) contiguous database pages.
9. Index 的并发：The key insight in these schemes is that modifications to the tree’s physical structure (e.g., splitting pages) can be made in a non-transactional manner as long as all concurrent transactions continue to find the correct data at the leaves.
10. 对 Index B-Tree 修改的回滚：The main idea is that structural index changes need not be undone when the associated transaction is aborted; such changes can often have no effect on the database tuples seen by other transactions. For example, if a B+-tree page is split during an inserting transaction that subsequently aborts, there is no pressing need to undo the split during the abort processing. This raises the challenge of labeling some log records redo-only. During any undo processing of the log, the redo-only changes can be left in place. ARIES provides an elegant mechanism for these scenarios, called nested top actions, that allows the recovery process to “jump over” log records for physical structure modifications during recovery without any special-case code.
11. 
- [Relational Database Index Design and the Optimizers](https://www.amazon.com/Relational-Optimizers-Lahdenmaki-published-Wiley-Blackwell/dp/B00EKYLFSI)
- [Transactional Information Systems: Theory, Algorithms, and the Practice of Concurrency Control](https://www.sciencedirect.com/book/9781558605084/transactional-information-systems)

### Talks

- [Data Structures and Algorithms for Big Databases](https://people.csail.mit.edu/bradley/BenderKuszmaul-tutorial-xldb12.pdf)
- [A Journey From A Quick HackTo A High-Reliability Database Engine](https://www.sqlite.org/talks/wroclaw-20090310.pdf)

### Blogs

- [x] [How does a relational database work](http://coding-geek.com/how-databases-work) ⭐

> only PostgreSQL is not using an UNDO. It uses instead a garbage collector daemon that removes the old versions of data. This is linked to the implementation of the data versioning in PostgreSQL.

- [The Internals of PostgreSQL](http://www.interdb.jp/pg/index.html)
- [Books propose](https://cakebytheoceanluo.github.io/2020/03/10/books/)

## SQL & Relation Algebra

Courses:

- CMU [Database Systems (15-445/645)](https://15445.courses.cs.cmu.edu/fall2019/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [Course Introduction and the Relational Model](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#aug-26-2019)
    - [Advanced SQL](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#aug-28-2019)

- UC Berkeley [Introduction to Database Systems](https://cs186berkeley.net/calendar/)
    - Introduction + SQL I
    - SQL II
    - Relational Algebra

## Query Optimizer

Courses:

- CMU [Database Systems (15-445/645)](https://15445.courses.cs.cmu.edu/fall2019/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [Query Planning & Optimization I](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#oct-14-2019)
    - [Query Planning & Optimization II](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#oct-21-2019)

Blogs:

- [x] [数据库内核杂谈](https://www.infoq.cn/theme/46), thanks to [顾仲贤](https://www.infoq.cn/profile/1780661/publish)
    - [数据库内核杂谈（七）：数据库优化器（上）](https://www.infoq.cn/article/GhhQlV10HWLFQjTTxRtA)
    - [数据库内核杂谈（八）：数据库优化器（下）](https://www.infoq.cn/article/JCJyMrGDQHl8osMFQ7ZR)
- [x] [SQL优化器原理 - 查询优化器综述](https://yq.aliyun.com/articles/610128), thanks to [勿烦](https://yq.aliyun.com/users/kyni3qcv656rk?spm=a2c4e.11153940.0.0.6adc1a8etfb0vx)

### Planner Models

Blogs:

- [x] [数据库内核杂谈](https://www.infoq.cn/theme/46), thanks to [顾仲贤](https://www.infoq.cn/profile/1780661/publish)
    - [数据库内核杂谈（九）：开源优化器 ORCA](https://www.infoq.cn/article/5o16eHOZ5zk6FzPSJpT2)
- [x] [SQL 查询优化原理与 Volcano Optimizer 介绍](https://zhuanlan.zhihu.com/p/48735419), thanks to [张茄子](https://www.zhihu.com/people/chase-zh) SQL 优化本质上是对关系代数的优化
- [x] [Cascades Optimizer](https://zhuanlan.zhihu.com/p/73545345), thanks to [hellocode](https://www.zhihu.com/people/hellocode-ming)

Papers:

- [x] 1979, [Access Path Selection in a Relational Database Management System](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.71.3735&rep=rep1&type=pdf), SIGMOD

1. Indexes are implemented as B-trees <3>, whose leaves are pages containing sets of (**key, identifiers of tuples which contain that key**).
2. If the tuples are inserted into segment pages in the index ordering, and if this physical proximity corresponding to index key value is maintained, we say that the index is clustered. A clustered index has the property that not only each index page, but also each data page containing a tuple from that relation will be touched only once in a scan on that index.
3. A "sargable(search argument able) predicate" is one of the form (or which can be put into the form) “column comparison-operator value”. SARGS are expressed as a boolean expression of such predicates in disjunctive normal form.

- 1979, [Query Processing in Main Memory Database Management Systems](http://15721.courses.cs.cmu.edu/spring2016/papers/p239-lehman.pdf), VLDB
- 1987, [Query Optimization by Simulated Annealing](http://ftp.cs.wisc.edu/pub/techreports/1987/TR693.pdf), SIGMOD
- 1988, [Grammar-like Functional Rules for Representing Query Optimization Alternatives](https://people.eecs.berkeley.edu/~brewer/cs262/23-lohman88.pdf), SIGMOD
- 1993, [The Volcano Optimizer Generator- Extensibility and Efficient Search](https://pdfs.semanticscholar.org/a817/a3e74d1663d9eb35b4baf3161ab16f57df85.pdf), ICDE
- 1995, [The Cascades Framework for Query Optimization](https://pdfs.semanticscholar.org/360e/cdfc79850873162ee4185bed8f334da30031.pdf), IEEE Data engineering Bulltin
- 1998, [An Overview of Query Optimization in Relational Systems](https://web.stanford.edu/class/cs345d-01/rl/chaudhuri98.pdf), PODS
- 2001, [LEO – DB2’s LEarning Optimizer](http://15721.courses.cs.cmu.edu/spring2016/papers/stillger-vldb2001.pdf), VLDB
- 2004, [Robust Query Processing through Progressive Optimization](https://www.cse.iitb.ac.in/infolab/Data/Courses/CS632/2006/Papers/sigmod04-markl.pdf), SIGMOD
- 2014, [Orca: A Modular Query Optimizer Architecture for Big Data](http://15721.courses.cs.cmu.edu/spring2016/papers/p337-soliman.pdf), SIGMOD
- 2016, [Parallelizing Query Optimization on Shared-Nothing Architectures](http://www.vldb.org/pvldb/vol9/p660-trummer.pdf), VLDB
- 2016, [The MemSQL Query Optimizer: A modern optimizer for real-time analytics in a distributed database](http://www.vldb.org/pvldb/vol9/p1401-chen.pdf), VLDB

### Subquery Optimization

Blogs:

- [SQL 子查询的优化](https://zhuanlan.zhihu.com/p/60380557), thanks to [Eric Fu](https://www.zhihu.com/people/fuyufjh)
- [Calcite 子查询处理 - I (RemoveSubQuery)](https://zhuanlan.zhihu.com/p/62338250), thanks to [一只无情的小猫咪](https://www.zhihu.com/people/loop_recur)
- [Calcite 子查询处理 - II (Decorrelate)](https://zhuanlan.zhihu.com/p/66227661), thanks to [一只无情的小猫咪](https://www.zhihu.com/people/loop_recur)

Papers:

- 2001, [Orthogonal Optimization of Subqueries and Aggregation](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.563.8492&rep=rep1&type=pdf), SIGMOD
- 2009, [Enhanced subquery optimizations in Oracle](https://www.researchgate.net/publication/220538535_Enhanced_Subquery_Optimizations_in_Oracle), VLDB
- 2015, [Unnesting Arbitrary Queries](http://www.btw-2015.de/res/proceedings/Hauptband/Wiss/Neumann-Unnesting_Arbitrary_Querie.pdf), BTW

### Join Order Optimization

Papers:

- 2006, [Analysis of Two Existing and One New Dynamic Programming Algorithm for the Generation of Optimal Bushy Join Trees without Cross Products](http://www.vldb.org/conf/2006/p930-moerkotte.pdf), VLDB
- 2015, [How Good Are Query Optimizers, Really?](http://www.vldb.org/pvldb/vol9/p204-leis.pdf), VLDB
- 2018, [Adaptive Optimization of Very Large Join Queries](https://db.in.tum.de/~radke/papers/hugejoins.pdf), SIGMOD

### Functional Dependency & Physical Properties

Thesis:

- 2000, [Exploiting Functional Dependence in Query Optimization](https://cs.uwaterloo.ca/research/tr/2000/11/CS-2000-11.thesis.pdf)

Papers:

- 1996, [Fundamental Techniques for Order Optimization](https://cs.uwaterloo.ca/~gweddell/cs798/p57-simmen.pdf), SIGMOD
- 2004, [An Efficient Framework for Order Optimization](https://www.researchgate.net/publication/4084912_An_efficient_framework_for_order_optimization), ICDE
- 2010, [Incorporating Partitioning and Parallel Plans into the SCOPE Optimizer](http://www.cs.albany.edu/~jhh/courses/readings/zhou10.pdf), ICDE

### Cost Model

Papers:

- 1996, [Modelling Costs for a MM-DBMS](), in Real-Time Databases
- 2014, [Approximation Schemes for Many-Objective Query Optimization](https://infoscience.epfl.ch/record/219202/files/p1299-trummer.pdf), SIGMOD
- 2015, [Multi-Objective Parametric Query Optimization](http://www.vldb.org/pvldb/vol8/p221-trummer.pdf), VLDB

### Statistics

Papers:

- 1984, [Accurate Estimation of the Number of Tuples Satisfying a Condition](https://dl.acm.org/doi/pdf/10.1145/971697.602294), SIGMOD
- 1993, [Optimal Histograms for Limiting Worst-Case Error Propagation in the Size of Join Results](https://dl.acm.org/doi/pdf/10.1145/169725.169708), ACM Trans. on Database Systems
- 1993, [Universality of Serial Histograms](https://pdfs.semanticscholar.org/deeb/d2fa377a41de49e5556ea948191a741faa1e.pdf), VLDB
- 1995, [Balancing Histogram Optimality and Practicality for Query Result Size Estimation](https://dl.acm.org/doi/pdf/10.1145/223784.223841), SIGMOD
- 1996, [Improved Histograms for Selectivity Estimation of Range Predicates](https://dl.acm.org/doi/pdf/10.1145/233269.233342), SIGMOD
- 1997, [SEEKing the truth about ad hoc join costs](https://dl.acm.org/doi/pdf/10.1007/s007780050043?download=true), VLDB
- 2000, [Towards Estimation Error Guarantees for Distinct Values](https://dl.acm.org/doi/pdf/10.1145/335168.335230), SIGMOD/PODS
- 2001, [Distinct Sampling for Highly-Accurate Answers to Distinct Values Queries and Event Reports](http://vldb.org/conf/2001/P541.pdf), VLDB
- 2003, [The History of Histograms](http://www.vldb.org/conf/2003/papers/S02P01.pdf), VLDB
- 2005, [An Improved Data Stream Summary: The Count-Min Sketch and its Applications](http://dimacs.rutgers.edu/~graham/pubs/papers/cm-full.pdf), Journal of Algorithms
- 2007, [New Estimation Algorithms for Streaming Data: Count-min Can Do More](http://webdocs.cs.ualberta.ca/~drafiei/papers/cmm.pdf)
- 2009, [Preventing Bad Plans by Bounding the Impact of Cardinality Estimation Errors](https://dl.acm.org/doi/pdf/10.14778/1687627.1687738), VLDB
- 2010, [Histograms Reloaded: The Merits of Bucket Diversity](https://dl.acm.org/doi/pdf/10.1145/1807167.1807239), SIGMOD
- 2014, [Exploiting Ordered Dictionaries to Efficiently Construct Histograms with Q-Error Guarantees in SAP HANA](https://dl.acm.org/doi/pdf/10.1145/2588555.2595629), SIGMOD
- 2017, [Adaptive Statistics in Oracle 12c](http://www.vldb.org/pvldb/vol10/p1813-zait.pdf), VLDB
- 2019, [Pessimistic Cardinality Estimation: Tighter Upper Bounds for Intermediate Join Cardinalities](https://dl.acm.org/doi/pdf/10.1145/3299869.3319894), SIGMOD
- 2019, [Deep Unsupervised Cardinality Estimation](http://www.vldb.org/pvldb/vol13/p279-yang.pdf), VLDB
- 2020, [NeuroCard: One Cardinality Estimator for All Tables](https://vldb.org/pvldb/vol14/p61-yang.pdf), VLDB

Books:
- [Synopses for Massive Data: Samples, Histograms, Wavelets, Sketches](https://db.cs.berkeley.edu/cs286/papers/synopses-fntdb2012.pdf)

## Query Execution

Courses:

- CMU [Database Systems (15-445/645)](https://15445.courses.cs.cmu.edu/fall2019/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [Query Execution I](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#oct-07-2019)
    - [Query Execution II](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#oct-09-2019)

### Execution Framework

Papers:

- 1994, [Volcano-An Extensible and Parallel Query Evaluation System](https://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf), IEEE Transactions on Knowledge and Data EngineeringFebruary
- 2014, [Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age](https://15721.courses.cs.cmu.edu/spring2019/papers/14-scheduling/p743-leis.pdf), SIGMOD

### Vectorization vs Compilization

Blogs:

- [Overhead of a Generalized Query Execution Engine](https://github.com/pivotal/blog/blob/master/content/post/codegen-gpdb-qx.md), from [The Pivotal Engineering Journal](https://github.com/pivotal/blog), thanks to the Pivotal Engineering team

Papers:

- 2005, [MonetDB/X100: Hyper-Pipelining Query Execution](http://cidrdb.org/cidr2005/papers/P19.pdf), CIDR
- 2011, [Efficiently Compiling Efficient Query Plans for Modern Hardware](https://www.vldb.org/pvldb/vol4/p539-neumann.pdf), VLDB
- 2017, [Relaxed Operator Fusion for In-Memory Databases: Making Compilation, Vectorization, and Prefetching Work Together At Last](http://www.vldb.org/pvldb/vol11/p1-menon.pdf), VLDB
- 2018, [Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask](http://www.vldb.org/pvldb/vol11/p2209-kersten.pdf), VLDB
- 2018, [Adaptive Execution of Compiled Queries](https://db.in.tum.de/~leis/papers/adaptiveexecution.pdf), ICDE

### Join

Papers:

- 2013, [Multi-Core, Main-Memory Joins: Sort vs. Hash Revisited](http://www.vldb.org/pvldb/vol7/p85-balkesen.pdf), VLDB
- 2017, [Looking Ahead Makes Query Plans Robust](http://www.vldb.org/pvldb/vol10/p889-zhu.pdf), VLDB

### Hash Table

Courses:

- CMU [Database Systems (15-445/645)](https://15445.courses.cs.cmu.edu/fall2019/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [Hash Tables](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#sep-16-2019)

Blogs:

- [Fibonacci Hashing: The Optimization that the World Forgot (or: a Better Alternative to Integer Modulo)](https://probablydance.com/2018/06/16/fibonacci-hashing-the-optimization-that-the-world-forgot-or-a-better-alternative-to-integer-modulo/), thanks to [Malte Skarupke](https://probablydance.com/)
- [All hash table sizes you will ever need](https://databasearchitects.blogspot.com/2020/01/all-hash-table-sizes-you-will-ever-need.html), thanks to [Database Architects - Thomas Neumann](https://databasearchitects.blogspot.com/)

### Bloom Filter

Papers:
- 2018, [SuRF: Practical Range Query Filtering with Fast Succinct Tries](https://www.cs.cmu.edu/~pavlo/papers/mod601-zhangA-hm.pdf), SIGMOD


## DDL

- 2013, [Online, Asynchronous Schema Change in F1](https://research.google.com/pubs/archive/41376.pdf), VLDB

## Relational Model

Blogs: 

- [What is a Relational Database?](https://www.calebcurry.com/what-is-a-relational-database/), thanks to [Caleb Curry](https://www.calebcurry.com/author/calebcurry_rlrc3d/)
- [What is a Relational Database?](https://careerkarma.com/blog/relational-database/),thank to [JAMES GALLAGHER](https://careerkarma.com/blog/author/jamesgallagher/)

### Codd's Rules

Blogs:

- [Codd’s Rules for Relational Database Systems](https://www.oreilly.com/library/view/sql-in-a/9780596155322/ch01s01s01.html), thanks to [Kevin Kline](https://en.wikipedia.org/wiki/Kevin_Kline)

### Relational Data Model

Blogs:

- [Relational model](https://en.wikipedia.org/wiki/Relational_model), thanks to [Wikipedia](https://en.wikipedia.org/wiki/Wikipedia)

### Relational Algebra

Blogs:

- [Introduction of Relational Algebra in DBMS](https://www.geeksforgeeks.org/introduction-of-relational-algebra-in-dbms/#:~:text=Relational%20Algebra%20is%20procedural%20query,column%20data%20from%20a%20relation.), thanks to [GeeksforGeeks](https://www.linkedin.com/company/geeksforgeeks?originalSubdomain=in)

### ER to Relational Model

Blogs:

- [ER Model to Relational Model](https://www.tutorialspoint.com/dbms/er_model_to_relational_model.htm), thanks to [tutorialspoint](https://www.tutorialspoint.com/index.htm1)

### SQL - Overview

Blogs:

- [An Overview of SQL Text Functions](https://learnsql.com/blog/), thanks to [Zahin Rahman](https://learnsql.com/authors/zahin-rahman/)

## Transaction

### Isolation Levels

Blogs:

- [x] [一致性模型](https://www.jianshu.com/p/3673e612cce2), thanks to [siddontang](https://www.jianshu.com/u/1yJ3ge) :star:

Papers:

- 1995, [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf), SIGMOD
- 2000, [Generalized Isolation Level Definitions](http://pmg.csail.mit.edu/papers/icde00.pdf), Proceedings of 16th International Conference on Data Engineering

### Concurrency Control

Courses:

- CMU [Database Systems (15-445/645)](https://15445.courses.cs.cmu.edu/fall2019/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [Concurrency Control Theory](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#oct-23-2019)
    - [Two-Phase Locking Concurrency Control](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#oct-28-2019)
    - [Timestamp Ordering Concurrency Control](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#oct-30-2019)
    - [Multi-Version Concurrency Control](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#nov-04-2019)

- CMU [Advanced Database Systems (15-721)](https://15721.courses.cs.cmu.edu/spring2020/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [Multi-Version Concurrency Control (Design Decisions)](https://15721.courses.cs.cmu.edu/spring2020/schedule.html#jan-22-2020)
    - [Multi-Version Concurrency Control (Protocols)](https://15721.courses.cs.cmu.edu/spring2020/schedule.html#jan-27-2020)
    - [Multi-Version Concurrency Control (Garbage Collection)](https://15721.courses.cs.cmu.edu/spring2020/schedule.html#jan-29-2020)

Papers:

- 1976, [The Notions of Consistency and Predicate Locks in a Database System](http://jimgray.azurewebsites.net/papers/on%20the%20notions%20of%20consistency%20and%20predicate%20locks%20in%20a%20database%20system%20cacm.pdf), Communications of the ACM
- 1981, [Concurrency Control in Distributed Database Systems](https://people.eecs.berkeley.edu/~brewer/cs262/concurrency-distributed-databases.pdf), ACM Computing Surveys
- 1981, [On Optimistic Methods for Concurrency Control](https://www.eecs.harvard.edu/~htk/publication/1981-tods-kung-robinson.pdf), ACM Transactions on Database Systems
- 1983, [Multiversion Concurrency Control - Theory and Algorithms](https://sites.fas.harvard.edu/~cs265/papers/bernstein-1983.pdf), ACM Transactions on Database Systems
- 2012, [Serializable Snapshot Isolation in PostgreSQL](http://www.vldb.org/pvldb/vol4/p783-jung.pdf), VLDB
- 2012, [Calvin: Fast Distributed Transactions for Partitioned Database Systems](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf), SIGMOD
- 2014, [MaaT: effective and scalable coordination of distributed transactions in the cloud](http://www.nawab.me/Uploads/MaaT_VLDB2014.pdf), VLDB
- 2014, [Staring into the Abyss: An Evaluation of Concurrency Control with One Thousand Cores](http://www.vldb.org/pvldb/vol8/p209-yu.pdf), VLDB
- 2014, [An Evaluation of the Advantages and Disadvantages of Deterministic Database Systems](http://www.vldb.org/pvldb/vol7/p821-ren.pdf), VLDB
- 2015, [Fast Serializable Multi-Version Concurrency Control for Main-Memory Database Systems](https://db.in.tum.de/~muehlbau/papers/mvcc.pdf), SIGMOD
- 2017, [An Empirical Evaluation of In-Memory Multi-Version Concurrency Control](http://www.vldb.org/pvldb/vol10/p781-Wu.pdf), VLDB
- 2017, [An Evaluation of Distributed Concurrency Control](https://www.vldb.org/pvldb/vol10/p553-harding.pdf), VLDB
- 2019, [Scalable Garbage Collection for In-Memory MVCC Systems](https://db.in.tum.de/~boettcher/p128-boettcher.pdf), VLDB

## Network

Courses:

- CMU [Advanced Database Systems (15-721)](https://15721.courses.cs.cmu.edu/spring2020/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [Networking Protocols](https://15721.courses.cs.cmu.edu/spring2020/schedule.html#feb-19-2020)

Papers:

- 2016, [The End of Slow Networks: It's Time for a Redesign](http://www.vldb.org/pvldb/vol9/p528-binnig.pdf), VLDB
- 2016, [Accelerating Relational Databases by Leveraging Remote Memory and RDMA](https://15721.courses.cs.cmu.edu/spring2020/papers/11-networking/li-sigmod2016.pdf), SIGMOD
- 2017, [Don't Hold My Data Hostage: A Case for Client Protocol Redesign](http://www.vldb.org/pvldb/vol10/p1022-muehleisen.pdf), VLDB

## Storage

### NoSQL Systems

Papers:

- 2006, [Bigtable: A Distributed Storage System for Structured Data](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf), OSDI
- [x] 2007, [Dynamo: Amazon’s Highly Available Key-value Store](https://sites.cs.ucsb.edu/~agrawal/fall2009/dynamo.pdf), SOSP

1. 使用 gossip 在节点间同步 membership；

- 2008, [PNUTS: Yahoo!’s Hosted Data Serving Platform](https://sites.cs.ucsb.edu/~agrawal/fall2009/PNUTS.pdf), VLDB
- 2010, [Cassandra - A Decentralized Structured Storage System](https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf), SOSP

1. A node outage rarely signifies a permanent departure and therefore should not result in re-balancing of the partition assignment or repair of the
unreachable replicas;

- 2019, [PNUTS to Sherpa: Lessons from Yahoo!’s Cloud Database ](http://www.vldb.org/pvldb/vol12/p2300-cooper.pdf), VLDB

### Buffer Management

Courses:

- CMU [Database Systems (15-445/645)](https://15445.courses.cs.cmu.edu/fall2019/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [Buffer Pools](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#sep-11-2019)

Papers:

- 1987, [The 5 Minute Rule for Trading Memory for Disc Accesses and the 5 Byte Rule for Trading Memory for CPU Time](https://www.hpl.hp.com/techreports/tandem/TR-86.1.pdf), SIGMOD
- 2008, [The Five Minute Rule 20 Years Later and How Flash Memory Changes the Rules](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.227.3846&rep=rep1&type=pdf), ACM Queue
- 2018, [Managing Non-Volatile Memory in Database Systems](https://db.in.tum.de/people/sites/vanrenen/papers/HyMem.pdf?lang=de), SIGMOD
- 2018, [LeanStore: In-Memory Data Management Beyond Main Memory](https://db.in.tum.de/~leis/papers/leanstore.pdf), ICDE
- 2020, [Umbra: A Disk-Based System with In-Memory Performance](http://cidrdb.org/cidr2020/papers/p29-neumann-cidr20.pdf), CIDR

### Disk IO

Blogs:

- [x] [On Disk IO, Part 1: Flavors of IO](https://medium.com/databasss/on-disk-io-part-1-flavours-of-io-8e1ace1de017), thanks to [Alex](https://twitter.com/ifesdjeen)

Sector is the smallest unit of data transfer for block device, it is not possible to transfer less than one sector worth of data. However, often it is possible to fetch multiple adjacent segments at a time. 

Because Direct IO involves direct access to backing store, bypassing intermediate buffers in Page Cache, it is required that all operations are aligned to sector boundary.
In other words, every operation has to have a starting offset of a multiple of 512 and a buffer size has to be a multiple of 512 as well. When using Page Cache, because writes first go to memory, alignment is not important: when actual block device write is performed, Kernel will make sure to split the page into parts of the right size and perform aligned writes towards hardware.
Whether or not O_DIRECT flag is used, it is always a good idea to make sure your reads and writes are block aligned. Crossing segment boundary will cause multiple sectors to be loaded from (or written back on) disk as shown on images above. Using the block size or a value that fits neatly inside of a block guarantees block-aligned I/O requests, and prevents extraneous work inside the kernel.

*`O_NONBLOCK` is generally ignored for regular files, because block device operations are considered non-blocking (unlike socket operations, for example). Filesystem IO delays are not taken into account by the system. Possibly this decision was made because there’s a more or less hard time bound on operation completion.
For same reason, something you would usually use in Network context, like select and epoll, does not allow monitoring and/or checking status of regular files.*


- [x] [On Disk IO, Part 2: More Flavours of IO](https://medium.com/databasss/on-disk-io-part-2-more-flavours-of-io-c945db3edb13?), thanks to [Alex](https://twitter.com/ifesdjeen)

With mmap a file can be mapped to a memory segment privately or in shared mode. Private mapping allows reading from the file, but any write would trigger copy-on-write of the page in question in order to leave the original page intact and keep the changes private, so none of the changes will get reflected on the file itself. In shared mode, the file mapping is shared with other processes so they can see updates to the mapped memory segment. Additionally, changes are carried through to the underlying file

Memory Mapping is done through the page cache, the same way as the Standard IO operations such as read and write and is using a demand paging.

mmap is a very useful tool for working with IO: It avoids creating an extraneous copy of the buffer in memory (unlike Standard IO, where the data has to be copied into the user-space buffers before the system call is made). Besides, it avoids a system call (and subsequent context switch) overhead for triggering actual IO operation, except when Page Faults occur. From a developers perspective, issuing a random read using an mmapped file looks just like a normal pointer operation and doesn’t involve lseek calls.

mlock 将 page pin 在 page cache：Another useful call is mlock. It allows you to force pages to be held in memory. This means that once the page is loaded into memory, all subsequent operations will be served from the page cache. It has to be used with caution, since calling it on every page will simply exhaust the system resources.

**Vectored IO**

- [x] [On Disk IO, Part 3: LSM Trees](https://medium.com/databasss/on-disk-io-part-3-lsm-trees-8b2da218496f), thanks to [Alex](https://twitter.com/ifesdjeen)
- [x] [On Disk IO, Part 4: B-Trees and RUM Conjecture](https://medium.com/databasss/on-disk-storage-part-4-b-trees-30791060741), thanks to [Alex](https://twitter.com/ifesdjeen)
- [x] [On Disk IO, Part 5: Access Patterns in LSM Trees](https://medium.com/databasss/on-disk-io-access-patterns-in-lsm-trees-2ba8dffc05f9), thanks to [Alex](https://twitter.com/ifesdjeen)

(For SSD)It’s better to keep the write operations page-aligned in order to avoid additional write multiplication. And last, keeping the data with similar lifecycle together (e.g. the data that would be both written and discarded at the same time) will be beneficial for performance. Most of these points are points speak favour of immutable LSM-like Storage, rather than systems that allows in-place updates: writes are batched and SSTables are written sequentially, files are immutable and, when deleted, the whole file is invalidated at once.

A key goal of log-structured systems is sequentialising writes. However, if the FTL is shared by two log- structured applications (or even a single application with multiple append streams), the incoming data into the FTL is likely to look random or disjoint. 
It’s often advised to use a separate physical device for Write Ahead Log to make sure both memory table flushes and WAL writes are sequential. There are many other reasons to do so, too: to avoid IO saturation, for better failover, more predictable latencies.

- [x] [Ensuring data reaches disk(LWN)](https://lwn.net/Articles/457667/)

I/O operations performed against files opened with O_DIRECT bypass the kernel's page cache, writing directly to the storage. Recall that the storage may itself store the data in a write-back cache, so *`fsync()` is still required for files opened with `O_DIRECT` in order to save the data to stable storage*. The `O_DIRECT` flag is only relevant for the system I/O API.

Synchronous I/O is any I/O (system I/O with or without O_DIRECT, or stream I/O) performed to a file descriptor that was opened using the O_SYNC or O_DSYNC flags. These are the synchronous modes, as defined by POSIX:

O_SYNC: File data and all file metadata are written synchronously to disk.
O_DSYNC: Only file data and metadata needed to access the file data are written synchronously to disk.
O_RSYNC: Not implemented
The data and associated metadata for write calls to such file descriptors end up immediately on stable storage. Note the careful wording, there. Metadata that is not required for retrieving the data of the file may not be written immediately. That metadata may include the file's access time, creation time, and/or modification time.

It is also worth pointing out the subtleties of opening a file descriptor with O_SYNC or O_DSYNC, and then associating that file descriptor with a libc file stream. Remember that fwrite()s to the file pointer are buffered by the C library. It is not until an fflush() call is issued that the data is known to be written to disk. In essence, associating a file stream with a synchronous file descriptor means that an fsync() call is not needed on the file descriptor after the fflush(). The fflush() call, however, is still necessary. -> flush 将 page cache 写到 disk，同时将 disk cache -> disk stable storage. 

Similarly, if you encounter a system failure (such as power loss, ENOSPC or an I/O error) while overwriting a file, it can result in the loss of existing data. To avoid this problem, it is common practice (and advisable) to write the updated data to a temporary file, ensure that it is safe on stable storage, then rename the temporary file to the original file name (thus replacing the contents). This ensures an atomic update of the file, so that other readers get one copy of the data or another.


- [x] [Read, write & space amplification - pick 2](http://smalldatum.blogspot.com/2015/11/read-write-space-amplification-pick-2_23.html), thanks to [Mark Callaghan](https://twitter.com/markcallaghandb)

Endurance (write amp) and capacity (space amp) matter when using flash. IOPs (read amp for point and range queries, write amp for writes) matters when using disk.

Papers:

- 2016, [Design Tradeoffs of Data Access Methods](http://scholar.harvard.edu/files/stratos/files/rum-tutorial.pdf?m=1461167186), SIGMOD
- 2016, [Designing Access Methods: The RUM Conjecture](https://stratos.seas.harvard.edu/files/stratos/files/rum.pdf), EDBT

### B-Tree

Blogs:

- [x] [B树、B+树索引算法原理（上）](https://www.codedump.info/post/20200609-btree-1/) thanks to [codedump](https://www.codedump.info/)
- [x] [B树、B+树索引算法原理（下）](https://www.codedump.info/post/20200615-btree-2/)

Courses:

- CMU [Database Systems (15-445/645)](https://15445.courses.cs.cmu.edu/fall2019/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [Trees Indexes I](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#sep-18-2019)
    - [Trees Indexes II](https://15445.courses.cs.cmu.edu/fall2019/schedule.html#sep-23-2019)

- CMU [Advanced Database Systems (15-721)](https://15721.courses.cs.cmu.edu/spring2020/schedule.html), thanks to [Andy Pavlo](http://www.cs.cmu.edu/~pavlo/)
    - [OLTP Indexes (B+Tree Data Structures)](https://15721.courses.cs.cmu.edu/spring2020/schedule.html#feb-03-2020)

Papers:

- 1979, [The Ubiquitous B-Tree](http://carlosproal.com/ir/papers/p121-comer.pdf)

### LSM-Tree

Papers:

- 1996, [The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf),
- 2014, [A Comparison of Fractal Trees to Log-Structured Merge (LSM) Trees](http://www.pandademo.com/wp-content/uploads/2017/12/A-Comparison-of-Fractal-Trees-to-Log-Structured-Merge-LSM-Trees.pdf)
- 2017, [WiscKey: Separating Keys from Values in SSD-conscious Storage](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf), TOS
- 2019, [LSM-based Storage Techniques: A Survey](https://arxiv.org/pdf/1812.07527.pdf)

### Learned Indexes Structures

Papers:

- 2018, [The Case for Learned Index Structures](https://www.cl.cam.ac.uk/~ey204/teaching/ACS/R244_2018_2019/papers/Kraska_SIGMOD_2018.pdf)
- 2019, [Learning Multi-dimensional Indexes](https://arxiv.org/pdf/1912.01668.pdf)
- 2020, [XIndex: A Scalable Learned Index for Multicore Data Storage](https://dl.acm.org/doi/pdf/10.1145/3332466.3374547)
- 2020, [RadixSpline: A Single-Pass Learned Index](https://dl.acm.org/doi/10.1145/3401071.3401659), [Source Code](https://github.com/learnedsystems/RadixSpline), aiDM@SIGMOD
- 2020, [The PGM-index: a fully-dynamic compressed learned index with provable worst-case bounds](http://www.vldb.org/pvldb/vol13/p1162-ferragina.pdf), [Source Code](https://github.com/gvinciguerra/PGM-index), VLDB
- 2020, [From WiscKey to Bourbon: A Learned Index for Log-Structured Merge Trees](http://pages.cs.wisc.edu/~yifann/bourbon-osdi20.pdf)

## Serializing & RPC

- [Protocol Buffers Developer Guide](https://developers.google.com/protocol-buffers/docs/overview)
- [gRPC Documntation](https://www.grpc.io/docs/quickstart/go/)

## Data Partitioning

Blogs:

- [TiDB Internal (I) - Data Storage](https://pingcap.com/blog/2017-07-11-tidbinternal1/)
- [x] [Partitioning Behavior of DynamoDB](https://dzone.com/articles/partitioning-behavior-of-dynamodb), thanks to [Parth Modi](https://dzone.com/users/3098371/parthmodi.html)

Papers:

- [x] 2007, [Dynamo: Amazon’s Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf), SOSP

## Replication & Consistency

Blogs:

- [Tick or Tock? Keeping Time and Order in Distributed Databases](https://dzone.com/articles/tick-or-tock-keeping-time-and-order-in-distributed-1), thanks to [Liu Tang](https://dzone.com/users/3186309/siddontang.html)

Papers:

- 2012, [Consistency Tradeoffs in Modern Distributed Database System Design](http://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf)
- 2020, [Strong and Efficient Consistency with Consistency-Aware Durability ](http://pages.cs.wisc.edu/~ag/cad.pdf), FAST 2020

## Consensus

Technical report:

- University of Cambridge [Distributed consensus revised](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-935.pdf), a great paper about Consenssus especially Paxos and Paxos-Related algorithms, by Heidi Howard

Papers:

- 2014, [Ark: A Real-World Consensus Implementation](https://arxiv.org/pdf/1407.4765.pdf), CoRR

## Scheduling

Blogs:

- [Building a Large-scale Distributed Storage System Based on Raft](https://www.cncf.io/blog/2019/11/04/building-a-large-scale-distributed-storage-system-based-on-raft/), by Ed Huang

Papers:

- 2016, [Automated Demand-driven Resource Scaling in Relational Database-as-a-Service](http://www.audentia-gestion.fr/MICROSOFT/p883-das.pdf), SIGMOD
- 2019, [Autoscaling Tiered Cloud Storage in Anna](https://dl.acm.org/doi/pdf/10.14778/3311880.3311881), VLDB
- 2020, [Adaptive HTAP through Elastic Resource Scheduling](https://dl.acm.org/doi/pdf/10.1145/3318464.3389783), SIGMOD
- 2020, [MorphoSys: Automatic Physical Design Metamorphosis for Distributed Database Systems](http://www.vldb.org/pvldb/vol13/p3573-abebe.pdf), VLDB

## Benchmark & Testing

Blogs:

- [Use go-ycsb to benchmark different databases (1)](https://medium.com/@siddontang/use-go-ycsb-to-benchmark-different-databases-8850f6edb3a7), thanks to [siddontang](https://medium.com/@siddontang)
- [Chaos Tools and Techniques for Testing the TiDB Distributed NewSQL Database](https://dzone.com/articles/chaos-tools-and-techniques-for-testing-the-tidb-di-1), thanks to [Liu Tang](https://dzone.com/users/3186309/siddontang.html)
- [Creating Custom Sysbench Scripts](https://www.percona.com/blog/2019/04/25/creating-custom-sysbench-scripts/), thanks to [Matthew Boehm](https://www.percona.com/blog/author/matthew-boehm/)


Papers:

- 2010, [Benchmarking Cloud Serving Systems with YCSB](https://courses.cs.duke.edu/fall13/compsci590.4/838-CloudPapers/ycsb.pdf), SOCC

## HTAP

Papers:

- 2020, [TiDB: A Raft-based HTAP Database](http://www.vldb.org/pvldb/vol13/p3072-huang.pdf), VLDB
- 2020, [F1 Lightning: HTAP as a Service](http://www.vldb.org/pvldb/vol13/p3313-yang.pdf), VLDB

## TLA+

Talks:
- [The TLA+ Video Course](http://lamport.azurewebsites.net/video/videos.html)

## OLAP

- [Dremel](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36632.pdf)
- [The-striping-and-assembly-algorithms-from-the-Dremel-paper](https://github.com/julienledem/redelm/wiki/The-striping-and-assembly-algorithms-from-the-Dremel-paper)
- [Dremel made simple with parquet](https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet)

1. Organizing by column allows for better compression, as data is more homogenous. The space savings are very noticeable at the scale of a Hadoop cluster.

I/O will be reduced as we can efficiently scan only a subset of the columns while reading the data. Better compression also reduces the bandwidth required to read the input.

As we store data of the same type in each column, we can use encodings better suited to the modern processors’ pipeline by making instruction branching more predictable.
