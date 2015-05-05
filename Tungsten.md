# 让电火花更加高速地流转在钨丝之上

In a previous blog post, we looked back and surveyed performance improvements made to Spark in the past year. In this post, we look forward and share with you the next chapter, which we are callingProject Tungsten. 2014 witnessed Spark setting the world record in large-scale sorting and saw major improvements across the entire engine from Python to SQL to machine learning. Performance optimization, however, is a never ending process.

在之前的博文中,我们回顾和调查了Spark在过去的一年中的性能改进。在这篇文章中,我们期待着与你分享下一章,我们取名为钨计划(Project Tungsten, 下同)。2014年中, 我们见证了Spark在大规模排序测试中创造的新的世界记录,看到整个Spark引擎中多项重大改进, 从Python到SQL再到机器学习。然而,性能优化,是一个永无止境的过程。

Project Tungsten will be the largest change to Spark’s execution engine since the project’s inception. It focuses on substantially improving the efficiency of memory and CPU for Spark applications, to push performance closer to the limits of modern hardware. This effort includes three initiatives:

钨计划将是Spark执行引擎有史以来最重大的一次革新。它将主要关注如何大幅提高火花应用程序的内存和CPU的效率, 从而使Spark的性能能够更加接近现代硬件性能的极限。这种工作将包括三个部分:

1. Memory Management and Binary Processing: leveraging application semantics to manage memory explicitly and eliminate the overhead of JVM object model and garbage collection
2. Cache-aware computation: algorithms and data structures to exploit memory hierarchy
3. Code generation: using code generation to exploit modern compilers and CPUs


 

1. 内存管理和二进制处理:利用应用程序语义显式地管理内存和消除JVM对象模型的overhead和垃圾收集的开销
2. 基于缓存计算:利用内存层次结构创造新的算法和数据结构
3. 代码生成:利用代码生成技术更好的利用现代编译器和CPU


The focus on CPU efficiency is motivated by the fact that Spark workloads are increasingly bottlenecked by CPU and memory use rather than IO and network communication. This trend is shown by recent research on the performance of big data workloads (Ousterhout et al) and we’ve arrived at similar findings as part of our ongoing tuning and optimization efforts for Databricks Cloudcustomers.

之所以专注于CPU效率是由于我们发现越来越多的应用场景中,Spark的瓶颈在于CPU和内存,而不是IO和网络通信。这一趋势同样在最近对大数据的工作负载性能的研究(参见[Kay Ousterhout等人的论文][1])中发现, 并且我们在对Databricks Cloud持续调优和优化工作中也发现了类似结论。

Why is CPU the new bottleneck? There are many reasons for this. One is that hardware configurations offer increasingly large aggregate IO bandwidth, such as 10Gbps links in networks and high bandwidth SSD’s or striped HDD arrays for storage. From a software perspective, Spark’s optimizer now allows many workloads to avoid significant disk IO by pruning input data that is not needed in a given job. In Spark’s shuffle subsystem, serialization and hashing (which are CPU bound) have been shown to be key bottlenecks, rather than raw network throughput of underlying hardware. All these trends mean that Spark today is often constrained by CPU efficiency and memory pressure rather than IO.

为什么CPU会成为新的瓶颈?这有很多原因。一是硬件配置提供越来越大的IO带宽,如网络设备中更多的采用10 Gbps链接网络,存储设备中应用高带宽SSD或条纹硬盘阵列。从软件的角度来看,Spark中的优化器现在允许各种工作负载通过修剪不需要输入数据来避免无谓的磁盘IO和网络IO。在Spark的shuffle子系统中,序列化和hash分布(等依赖CPU的计算)已被证明是关键的瓶颈,而不是原始的底层硬件的网络吞吐量。所有这些都意味着Spark如今更容易受到来自CPU和内存的压力而不是IO问题。

1. Memory Management and Binary Processing
Applications on the JVM typically rely on the JVM’s garbage collector to manage memory. The JVM is an impressive engineering feat, designed as a general runtime for many workloads. However, as Spark applications push the boundary of performance, the overhead of JVM objects and GC becomes non-negligible.

1. 内存管理和二进制处理

Applications on the JVM typically rely on the JVM’s garbage collector to manage memory. The JVM is an impressive engineering feat, designed as a general runtime for many workloads. However, as Spark applications push the boundary of performance, the overhead of JVM objects and GC becomes non-negligible.

在JVM上运行的应用程序通常依赖于JVM的垃圾收集器来管理内存。JVM是一个伟大的产品,被设计作为一种通用的运行环境支持多种应用场景。然而,当Spark应用程序性能达到某些临界点时,JVM对象的创建和GC成为无法忽略的开销。

Java objects have a large inherent memory overhead. Consider a simple string “abcd” that would take 4 bytes to store using UTF-8 encoding. JVM’s native String implementation, however, stores this differently to facilitate more common workloads. It encodes each character using 2 bytes with UTF-16 encoding, and each String object also contains a 12 byte header and 8 byte hash code, as illustrated by the following output from the the Java Object Layout tool.

Java对象有非常大的潜在内存开销。假设一个简单的字符串“abcd”,一般情况下使用utf-8编码只需要4个字节来存储。然而在JVM的原生字符串实现中,为了适配多种工作场景,它将每个字符用2字节utf-16编码,并且每个字符串对象还包含一个12个字节的对象header信息和8字节Hash码,如以下输出所示布局工具的Java对象。

java.lang.String object internals:

OFFSET|SIZE | TYPE DESCRIPTION | VALUE
----|------|----|----
0 | 4 | (object header) | ...
4 | 4 | (object header) | ...
8 | 4 | (object header) | ...
12 | 4 |char[] String.value | []
16 | 4 | int String.hash | 0
20 | 4 | int String.hash32 | 0
    
Instance size: 24 bytes (reported by Instrumentation API)
A simple 4 byte string becomes over 48 bytes in total in the JVM object model!

实例大小: 24 bytes (通过Instrumentation API得出)
一个简单的四个字符的字符串在JVM的对象模型中总计占用了超过48个字节!

The other problem with the JVM object model is the overhead of garbage collection. At a high level, generational garbage collection divides objects into two categories: ones that have a high rate of allocation/deallocation (the young generation) ones that are kept around (the old generation). Garbage collectors exploit the transient nature of young generation objects to manage them efficiently. This works well when GC can reliably estimate the life cycle of objects, but falls short if the estimation is off (i.e. some transient objects spill into the old generation). Since this approach is ultimately based on heuristics and estimation, eeking out performance can require the “black magic” of GC tuning, with dozens of parameters to give the JVM more information about the life cycle of objects.

JVM对象模型的另一个问题是内存回收的开销。从较高的层次看,分代式的内存回收机制将JVM中的对象分为两类:一种是需要进行高速率分配/回收的(年轻代),另一种是需要长时间保存的(老年代)。内存回收器利用年轻代的对象的瞬态来有效地管理它们。当内存回收器可以准确地估计到对象的生命周期时,这样的机制没有问题.但由于这种方法始终还是基于启发式的估计,所以当对象生命周期长于估计值时(即一些瞬态对象泄漏到年老代)这种方法就会显出不足。寻求性能的提升需要了解内存回收器中的各项“黑魔法”,包含几十个参数用于提示JVM更多关于对象的生命周期的信息。

Spark, however, is not just a general-purpose application. Spark understands how data flows through various stages of computation and the scope of jobs and tasks. As a result, Spark knows much more information than the JVM garbage collector about the life cycle of memory blocks, and thus should be able to manage memory more efficiently than the JVM.

然而Spark不仅仅是一个通用的应用程序。Spark理解数据流在通过不同计算阶段的过程中每个Job和Task的生效范围。因此,Spark比JVM内存回收器更加了解内存块的生命周期,从而能够比JVM更有效地管理内存。

To tackle both object overhead and GC’s inefficiency, we are introducing an explicit memory manager to convert most Spark operations to operate directly against binary data rather than Java objects. This builds on sun.misc.Unsafe, an advanced functionality provided by the JVM that exposes C-style memory access (e.g. explicit allocation, deallocation, pointer arithmetics). Furthermore, Unsafe methods are intrinsic, meaning each method call is compiled by JIT into a single machine instruction.

为了解决对象的开销和GC的效率低下,我们引入一个显式的内存管理器将大多数Spark操作转换为直接对二进制数据而不是Java对象进行操作。这是构建在`sun.misc.Unsafe`上的一个由JVM提供的高级功能,使用C语言的风格进行内存访问(例如显式分配、回收、指针算法)。此外,`Unsafe`的方法是`intrinsic`的,这意味着每个方法调用将由JIT编译成一条机器指令。

In certain areas, Spark has already started using explicitly managed memory. Last year, Databricks contributed a new Netty-based network transport that explicitly manages all network buffers using a jemalloc like memory manager. That was critical in scaling up Spark’s shuffle operation and winning the Sort Benchmark.

在某些领域,Spark已经开始使用显式地管理内存。去年,Databricks贡献了一个新的Netty-based网络传输程序,使用jemalloc风格的内存管理器显式地管理所有网络缓冲区。这一点对于Spark中的shuffle操作的扩展性以及赢得`Sort Benchmark`起到了关键作用。

The first pieces of this will appear in Spark 1.4, which includes a hash table that operates directly against binary data with memory explicitly managed by Spark. Compared with the standard JavaHashMap, this new implementation much less indirection overhead and is invisible to the garbage collector.

上述这些改进中的一部分将会从Spark1.4.0版本开始发布,其中包括一个新的哈希表实现。在新的实现中,用户可以直接操作二进制数据,并且该实现使用Spark显式内存管理机制进行内存的管理。与标准JavaHashMap相比,这种新的实现更少中间环节开销,并且无需通过内存回收器进行管理(即不会因为这部分内存而产生GC)。

![Spark Binary Map][2]

This is still work-in-progress, but initial performance results are encouraging. As shown above, we compare the throughput of aggregation operations using different hash map: one with our new hash map’s heap mode, one with offheap, and one with java.util.HashMap. The new hash table supports over 1 million aggregation operations per second in a single thread, about 2X the throughput of java.util.HashMap. More importantly, without tuning any parameters, it has almost no performance degradation as memory utilization increases, while the JVM default one eventually thrashes due to GC.

这些改进仍然在进行中,但如上图所示,最初的性能结果令人鼓舞。我们使用三种不同的HashMap实现来对比聚合操作的吞吐量:一个是我们的新的实现中的Heap模式,一个是offheap模式,和一个java.util.HashMap。新的哈希表支持每秒超过100万聚合操作在一个线程,约两倍于java.util.HashMap的吞吐量。更重要的是,在没有任何参数调优的情况下,随着内存利用率增加它几乎没有性能下降,而最终`java.util.HashMap`由于GC最终崩溃。

In Spark 1.4, this hash map will be used for aggregations for DataFrames and SQL, and in 1.5 we will have data structures ready for most other operations, such as sorting and joins. This will in many cases eliminating the need to tune GC to achieve high performance.

在Spark1.4版本中, 这个新的hashmap会用于DataFrame和SQL中的聚合操作,而在1.5版本中我们会提供一个新的数据结构用于支持剩余大部分操作,例如排序和Join操作.这样在大多数情况下无需GC的参数调优便可获得一个较高的性能.

2. Cache-aware Computation
Before we explain cache-aware computation, let’s revisit “in-memory” computation. Spark is widely known as an in-memory computation engine. What that term really means is that Spark can leverage the memory resources on a cluster efficiently, processing data at a rate much higher than disk-based solutions. However, Spark can also process data orders magnitude larger than the available memory, transparently spill to disk and perform external operations such as sorting and hashing.

在我们解释什么是缓存计算之前,让我们重温“内存中”计算。Spark是一个广泛的被称为"内存中"的计算引擎。这个词的实际意思是,Spark可以利用集群中的内存资源,在处理数据速度上远高于基于磁盘的解决方案。然而,火花还可以处理数据量大于可用内存的场景,通过透明地将数据溢出保存到磁盘并执行外部操作,如排序和散列。

Similarly, cache-aware computation improves the speed of data processing through more effective use of L1/ L2/L3 CPU caches, as they are orders of magnitude faster than main memory. When profiling Spark user applications, we’ve found that a large fraction of the CPU time is spent waiting for data to be fetched from main memory. As part of Project Tungsten, we are designing cache-friendly algorithms and data structures so Spark applications will spend less time waiting to fetch data from memory and more time doing useful work.

同样,支持缓存计算提高了数据处理的速度通过更有效地利用CPU上L1,L2和L3的缓存,因为它们比物理内存在速度上快多个数量级。剖析Spark用户应用程序时,我们发现大部分的CPU时间是从物理内存等待获取数据。作为钨丝计划的一部分,我们正在设计缓存友好的算法和数据结构,所以Spark应用程序将花费更少的时间等待从内存获取数据,从而用更多的时间做有用的工作。

Consider sorting of records as an example. A standard sorting procedure would store an array of pointers to records and use quicksort to swap pointers until all records are sorted. Sorting in general has good cache hit rate due to the sequential scan access pattern. Sorting a list of pointers, however, has a poor cache hit rate because each comparison operation requires dereferencing two pointers that point to randomly located records in memory.

我们以记录排序作为一个例子。标准排序程序将存储数组的指针记录和使用`quicksort`交换指针,直到所有的记录进行排序。排序通常由于其顺序扫描访问模式一般有很好的缓存命中率。然而在排序列表指针时因为每个比较操作需要对两个指向内存随机位置的指针进行解引用操作,从而缓存命中率会相当低。

So how do we improve the cache locality of sorting? A very simple approach is to store the sort key of each record side by side with the pointer. For example, if the sort key is a 64-bit integer, then we use 128-bit (64-bit pointer and 64-bit key) to store each record in the pointers array. This way, each quicksort comparison operation only looks up the pointer-key pairs in a linear fashion and requires no random memory lookup. Hopefully the above illustration gives you some idea about how we can redesign basic operations to achieve higher cache locality.

那么,我们如何提高排序操作中的缓存本地性呢?一个非常简单的方法来存储每个记录的排序关键字以及它的指针。例如,如果排序键是一个64位整数,然后我们使用128位(64位指针和64位的密钥)来存储指针数组中的每条记录。这样,每个`quicksort`比较操作只需要以线性方式查找pointer-key对,不需要随机内存查找。希望上面的说明能够在"重新设计基本操作,以获得更高的缓存位置"方面给你带来启发。

![cache aware layout][3]

How does this apply to Spark? Most distributed data processing can be boiled down to a small list of operations such as aggregations, sorting, and join. By improving the efficiency of these operations, we can improve the efficiency of Spark applications as a whole. We have already built a version of sort that is cache-aware that is 3X faster than the previous version. This new sort will be used in sort-based shuffle, high cardinality aggregations, and sort-merge join operator. By the end of this year, most Spark’s lowest level algorithms will be upgraded to be cache-aware, increasing the efficiency of all applications from machine learning to SQL.

如何将这种方法应用于Spark?多数分布式数据处理问题可以归结为一系列的操作如聚合、排序和连接。通过改善这些操作的效率,我们可以整体上提高Spark应用程序的效率。我们已经在一个新版本的排序操作中支持缓存,比之前的版本快3倍。这个新类将用于事洗牌,高基数聚合、分类合并连接操作符。到今年年底,大多数Spark的基础算子将被升级支持缓存,从而提升所有Spark应用程序的效率。

 3. Code Generation

About a year ago Spark introduced code generation for expression evaluation in SQL and DataFrames. Expression evaluation is the process of computing the value of an expression (say “age > 35 && age < 40”) on a particular record. At runtime, Spark dynamically generates bytecode for evaluating these expressions, rather than stepping through a slower interpreter for each row. Compared with interpretation, code generation reduces the boxing of primitive data types and, more importantly, avoids expensive polymorphic function dispatches.

大约一年前Spark首次引入了代码生成机制用于SQL和DataFrames表达式求值。表达式求值就是在每一个特定的记录中计算表达式的值的过程(例如: `age > 35 && age < 40`). 在运行时,Spark动态生成字节码来计算这些表达式,而不是步进式对每一行数据使用解释器进行处理。与使用解释器相比,代码生成减少原始数据类型的构建,更重要的是,避免了昂贵的多态函数分派。

In an earlier blog post, we demonstrated that code generation could speed up many TPC-DS queries by almost an order of magnitude. We are now broadening the code generation coverage to most built-in expressions. In addition, we plan to increase the level of code generation from record-at-a-time expression evaluation to vectorized expression evaluation, leveraging JIT’s capabilities to exploit better instruction pipelining in modern CPUs so we can process multiple records at once.

在早些时候的一篇博客文章中,我们证明了代码生成可以加速很多TPC-DS查询近一个数量级。我们现在扩大代码生成覆盖大部分内置的表达式。此外,我们计划将代码生成的执行方式从每条记录使用表达式计算提升到使用矢量表达式求值,利用JIT的能力更好地利用现代CPU中的指令流水线从而让我们可以一次处理多个记录。

We’re also applying code generation in areas beyond expression evaluations to optimize the CPU efficiency of internal components. One area that we are very excited about applying code generation is to speed up the conversion of data from in-memory binary format to wire-protocol for shuffle. As mentioned earlier, shuffle is often bottlenecked by data serialization rather than the underlying network. With code generation, we can increase the throughput of serialization, and in turn increase shuffle network throughput.

除了表达式计算,我们计划将代码生成应用于优化CPU内部组件的效率。其中一个令我们非常兴奋的方向是将代码生成在shuffle中,从而加快将数据从内存中二进制格式的转换为shuffle中指定的格式。如前所述,shuffle中的瓶颈经常出现在数据序列化过程,而不是底层的网络瓶颈。通过代码生成机制,我们可以增加序列化的吞吐量,进而增加shuffle过程中整体的网络吞吐量。

![shuffle with code generation][4]

The above chart compares the performance of shuffling 8 million complex rows in one thread using the Kryo serializer and a code generated custom serializer. The code generated serializer exploits the fact that all rows in a single shuffle have the same schema and generates specialized code for that. This made the generated version over 2X faster to shuffle than the Kryo version.

上面的图是一个shuffle性能的对比测试:用一个线程对800万行数据分别使用Kryo序列化工具和代码生成方式的序列化工具。代码生成方式的序列化工具利用了一次shuffle操作中所有行的数据有相同的schema这一特性,自动生成特定的代码用于序列化工作。这使得序列化数据生成的速度比Kyro的方式快2倍。

Tungsten and Beyond
Project Tungsten is a broad initiative that will influence the design of Spark’s core engine over the next several releases. The first pieces will land in Spark 1.4, which includes explicitly managed memory for aggregation operations in Spark’s DataFrame API as well as customized serializers. Expanded coverage of binary memory management and cache-aware data structures will appear in Spark 1.5. Several parts of project Tungsten leverage the DataFrame model. We will also retrofit the improvements onto Spark’s RDD API whenever possible.

钨丝计划是一个宏大的设计,将影响未来几个版本Spark的核心引擎的改变。第一部分工作将在Spark 1.4版本中,其中包括在DataFrame的API聚合操作中使用显式地管理内存以及定制的序列化工具。全面使用二进制内存管理以及缓存优化数据结构将出现在1.5版本的Spark中。钨丝计划中的众多改进将首先应用于DataFrame模型。我们也将尽可能将这些改进加入到Spark的RDD API中。

There are also a handful of longer term possibilities for Tungsten. In particular, we plan to investigate compilation to LLVM or OpenCL, so Spark applications can leverage SSE/SIMD instructions out of modern CPUs and the wide parallelism in GPUs to speed up operations in machine learning and graph computation.

在钨丝计划中, 还有一些改进机会需要进行长期的考量。特别是,我们计划研究调查通过编译LLVM或OpenCL,从而使得Spark应用程序可以:
1. 无需依赖现代的cpu即可使用SSE/SIMD指令集
2. 在机器学习和图计算中利用GPU较高的并行度能力提升速度

The goal of Spark has always been to offer a single platform where users can get the best distributed algorithms for any data processing task. Performance is a key part of that goal, and Project Tungsten aims to let Spark applications run at the speed offered by bare metal. Stay tuned for the Databricks blog for longer term articles on the components of Project Tungsten as they ship. We’ll also be reporting details about the project at the upcoming Spark Summit in San Francisco in June.

Spark的目标一直是提供一个统一的平台,使用户可以得到最好的分布式算法处理任何数据。性能是一个关键的目标,钨丝计划旨在让电火花更加高速地流转在钨丝之上。请继续关注的Databricks后续关于钨丝计划中各类组件发布的文章。我们也会在即将到来的6月旧金山Spark峰会中介绍这个计划的详细内容。

* Note: hand-drawing diagrams inspired by our friends at Confluent (Martin Kleppmann)


  [1]: https://www.eecs.berkeley.edu/~keo/publications/nsdi15-final147.pdf
  [2]: http://img.ptcms.csdn.net/article/201504/30/5541bca729b1f_middle.jpg?_=45118
  [3]: http://img.ptcms.csdn.net/article/201504/30/5541b37c06a97.jpg
  [4]: http://img.ptcms.csdn.net/article/201504/30/5541b3d157ca6.jpg
