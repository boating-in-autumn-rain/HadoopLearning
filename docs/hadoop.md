---
typora-root-url: pictures\hadoop_pictures
---

# 概述

## 1.Hadoop1.x、2.x、3.X区别

<img src="hadoop123区别.png" alt="hadoop123区别" style="zoom: 80%;" />

**3.X：**
Java运行环境升级为1.8；
3.0支持单active namenode+多standby namenode部署方式进一步提升了可用性。
MapReduce本地优化，性能提升了30%。
良好的可视化界面，方便操作。

## 2.hdfs架构

**Namenode，datanode，client，secondary namenode。**
1）：namenode(nn)：就是master，他是一个主管，管理者。
		1、管理HDFS的名称空间
		2、配置副本策略
		3、管理数据块映射信息
		4、处理客户端读写请求。
2）：Datanode：就是slave。Namenode下达命令，Datanode执行实际的操作。
		1、存储实际的数据块
		2、执行数据块的读/写操作。
3）：client：就是客户端。
		1、文件切分。文件上传HDFS时，Client将文件切分成一个一个的block，然后进行上传
		2、与namenode交互，获取文件的位置信息。
		3、与datanode交互，读取或者写入数据。
		4、Client提供一些命令来管理HDFS，比如namenode格式化。
		5、Client可以通过一些命令来访问HDFS，比如对HDFS增删改查操作。
4）：secondary namenode(2)：并非namenode的热备。当namenode挂掉的时候，它并不能马上替换namenode并提供服务。
		1、辅助namenode，分担其工作量，比如定期合并Fsimage和Edits，并推送给namenode；
		2、在紧急情况下，可辅助恢复namenode。

## 3.NameNode宕机

NameNode故障后，可以采用如下两种方法恢复数据。
**1）将SecondaryNameNode中数据拷贝到NameNode存储数据的目录。**
（1）kill -9 NameNode进程
（2）删除NameNode存储的数据
（3）拷贝SecondaryNameNode中数据拷贝到原NameNode存储数据目录
（4）重新启动NameNode

## 4.MR过程

### **1.分为map和reduce。四次排序**

**Map：**
**（1）Read阶段：**由程序内的InputFormat（默认实现类是TextInputFormat）来读取外部数据，它会调用RecordReader的read方法来读取，返回k，v键值对。
**（2）Map阶段：**该节点将key/value交给用户编写map()函数处理，并产生一系列新的key/value。
**（3）Collect收集阶段：**它会将生成的key/value分区（调用Partitioner），并写入一个环形内存缓冲区中。
**（4）Spill溢写阶段：**缓冲区默认100M，可以通过MR.SORT.MB配置，达到80%就开始反向溢写，当环形缓冲区满后，MapReduce会将数据写到本地磁盘上，生成一个临时文件。
		Spill组件会从环形缓冲区溢出文件，这过程会按照定义的partitioner分区（默认是hashpartition），并且进行排序（**底层主要是快排**），如果有combiner也会执行combiner。
**（5）Combine阶段：**小文件执行合并，形成且分区且分区内有序的大文件（**使用的是归并排序**）。每轮合并io.sort.factor（默认10）个文件，并将产生的文件重新加入待合并列表中，对文件排序后，重复以上过程，直到最终得到一个大文件。

**Reduce：**
（1）ReduceTask从各个MapTask上远程拷贝一片数据，并针对某一片数据，如果其大小超过一定阈值，则写到磁盘上，磁盘上文件数据达到一定阈值，进行一次**归并排序**以生成更大的文件。否则直接放到内存中。
（2）当所有文件拷贝完毕后，Reduce Task统一对内存和磁盘上所有数据进行一次**归并排序**。通过GroupingComparator分辨同一组数据，把他们发送给reduce方法。
（3）调用context（）方法，会让OutPutFormat调用RecodeWriter的write（）方法将处理结果写入到HDFS中。

为什么要排序：MR在reduce阶段需要分组，将key相同的放在一起进行规约

**reduce任务什么时候开始？**

只要有map任务完成，就可以开始reduce任务

### 2.shuffle的环形缓冲区

使用环形数据结构是为了更有效地使用内存空间，在内存中放置尽可能多的数据。

​		底层就是一个字节数组：数组前面记录关于K-V的索引位置，数组后面记录K-V的数据。首尾相接构成一个环形的缓冲区，中间是赤道。用于数据spil溢出处理。
​    	缓冲区采用典型单生产者消费者模型。MapOutputBuffer的collect方法和MapOutputBuffer.Buffer的write方法作为生产者，spillThread线程是消费者，其间同步是通过可重入互斥锁spillLock和spillLock上的两个条件变量(spillDone和spillReady)实现的。
​    	Kvoffsets，主要由三个变量控制，kvstart、kvend、kvindex。开始时kvstart=kvend，kvindex指向待写入位置，当写入一条数据后，kvindex向后移动一位，当kvoffsets内存使用率超过io.sort.spill.percent(默认80%)后，数据开始溢出到磁盘。

![环形缓冲区](hadoop\环形缓冲区.jpg)

### 3.MR发生OOM

**1、Mapper/Reducer阶段JVM内存溢出（一般都是堆）**
		JVM堆内存溢出：堆内存不足

​		栈内存溢出：MR代码中有递归调用

**2.MRAppMaster内存不足**
		如果作业的输入的数据很大，导致产生了大量的Mapper和Reducer数量，致使MRAppMaster的压力很大，最终导致MRAppMaster内存不足，

**3.非JVM内存溢出**
		自己申请使用操作系统的内存，没有控制好，出现了内存泄露，导致的内存溢出。

### 4.关闭正在执行的MR任务

hadoop job -list    
hadoop job -kill jobid来杀死



## ----------分割线-------------

## 3.MR的shuffle机制

（1）MapTask收集我们的map()方法输出的kv对，放到内存缓冲区中
（2）从内存缓冲区不断溢出本地磁盘文件，可能会溢出多个文件
（3）多个溢出文件会被合并成大的溢出文件
（4）在溢出过程及合并的过程中，都要调用Partitioner进行分区和针对key进行**（快排）排序**
（5）ReduceTask根据自己的分区号，去各个MapTask机器上取相应的结果分区数据
（6）ReduceTask会取到同一个分区的来自不同MapTask的结果文件，ReduceTask会将这些文件再进行合并**（归并排序）**
（7）合并成大文件后，Shuffle的过程也就结束了，后面进入ReduceTask的逻辑运算过程（从文件中取出一个一个的键值对Group，调用用户自定义的reduce()方法）
注意：
（1）Shuffle中的缓冲区大小会影响到MapReduce程序的执行效率，原则上说，缓冲区越大，磁盘io的次数越少，执行速度就越快。
（2）缓冲区的大小可以通过参数调整，参数：io.sort.mb默认100M。

## 4.Yarn工作原理与架构

**工作原理**

![yarn工作原理](hadoop\yarn工作原理.png)

**架构**

**YARN主要ResourceManager、NodeManager、ApplicationMaster和Container等组件构成。**

<img src="hadoop\yarn架构.png" alt="yarn架构" style="zoom:67%;" />



## 5.YARN的资源调度器

Hadoop作业调度器主要有三种：**FIFO、Capacity Scheduler和Fair Scheduler。**
Hadoop3.1.3默认的资源调度器是Capacity Scheduler。2.7.2也是容量。
1、YARN是对应用进行资源分配，那应用具体指的什么？
（1）Mapreduce采用的模型是一个用户作业对应一个应用。
（2）Spark采用的是每个工作流或用户的对话对应一个应用。效率比第一种高。
（3）多个用户共享一个长期运行的应用。
**调度器（2）**
**1、FIFO先入先出调度器**
		先入先出队列，先为第一个应用请求资源，第一个满足后再依次为下一个提供服务。不适合共享集群。
**2、容量调度器(会出现饿死)**
		容量调度器（默认）允许多个组织共享整个hadoop集群，每个组织被配置一个专门的队列，每个队列分配整个集群资源的一部分，队列内部采用FIFO队列。
    	队列作业太多导致资源不够用，容量调度器可能会将空余的资源分配给队列中的作业。这叫“弹性队列”。但是，容量调度器不会强制队列释放资源，当一个队列资源不够时，必须等其他队列资源释放后，才能获取资源。
**3、公平调度器Fair**
    	强调多用户公平的使用资源，并且会动态调整应用程序的资源分配。比如：当一个大job提交时，只有这一个job在运行时，此时这个应用将获得所有的集群资源；当第二个job提交时，公平调度器会从第一个job中分配一半给第二个job，可能会存在延时，因为要等待第一个job的资源释放。

## 6.动态增删节点

**服役新数据节点**
（1）在hadoop104主机上再克隆一台hadoop105主机
（2）修改IP地址和主机名称
（3）删除原来HDFS文件系统留存的文件（/opt/module/hadoop-3.1.3/data和log）
（4）source一下配置文件source /etc/profile
（5）直接启动DataNode，即可关联到集群
**退役旧数据节点**
**添加白名单：**添加到白名单的主机节点，都允许访问NameNode，不在白名单的主机节点，都会被退出。
**黑名单退役：**在黑名单上面的主机都会被强制退出。

## 7.HDFS 副本放置策略

副本节点选择：
第一个副本在Client所处的节点上。如果客户端在集群外，随机选一个。
第二个副本与第一个副本位于相同机架，随机节点。
第三个副本位于不同机架，随机节点。

## 8.client和HDFS读写

**读流程**

<img src="hadoop\hdfs读流程.png" alt="hdfs读流程" style="zoom: 67%;" />

**写流程**

<img src="hadoop\hdfs写流程.png" alt="hdfs写流程" style="zoom:67%;" />



**延迟太高，怎么解决**

延迟高，从网络，IO，小文件分析下就行了。

## 9.调优

**很多牛逼的电脑，大CPU大内存，网卡超烂，如何优化？** 
强制本地化，选高压缩的序列化格式，核心就是尽量减少网络IO



## 10.数据倾斜

数据倾斜的现象：
数据频率倾斜：某一个区域的数据量要远远大于其他区域。
数据大小倾斜：部分记录的大小远远大于平均值。

减少数据倾斜的方法：
**1、抽样和范围分区：**可以通过对原始数据进行抽样得到的结果集来预设分区边界值。
**2、自定义分区：**基于输出键的背景知识进行自定义分区。例如，如果Map输出键的单词来源于一本书中。且其中的某几个专业词汇较多。那么就可以自定义分区将这些专业词汇发送给固定的一部分reduce实例。而将其他的都发送给剩余的Reduce实例。
**3、Combine：**使用Combine可以大量地减少数据倾斜。在可能的情况下，Combine的目的就是聚合并精简数据。
**4、采用Map join，尽量避免Reduce join。**

## 11.Hadoop的XML配置文件

core-site.xml：默认的访问端口hadoop102:8020，数据存放位置hadoop.tmp.dir
hdfs-site.xml：NN2存放位置hadoop104:9868，
mapred-site.xml：指定在yarn上运行。
yarn-site.xml：resourcemanager位置hadoop103，

## 12.有哪两大服务

 1）提供海量数据的存储服务。HDFS
 2）提供分析海量数据框架及运行平台。MapReduce和Yarn



## 14.mapreduce中间combine

1）combiner是MR程序中Mapper和Reducer之外的一种组件。
2）combiner的意义就是对每一个maptask的输出进行局部汇总，以减小网络传输量。
3）combiner能够应用的前提是不能影响最终的业务逻辑，而且，combiner的输出kv应该和reducer的输入kv类型要对应起来。

## 15.hdfs小文件

HDFS上每个文件都要在NameNode上建立一个索引，索引大小约为150byte，当小文件比较多的时候，就会产生很多的索引文件
		1.会大量占用NameNode的内存空间，
		2.索引文件过大使得索引速度变慢。
**解决办法：**
		1.在数据采集的时候，就将小文件或小批数据合成大文件再上传HDFS。
		2.在业务处理之前，在HDFS上使用MapReduce程序对小文件进行合并。
		3.在MapReduce处理时，可采用CombineTextInputFormat提高效率。
**解决方案：**
1.Hadoop Archive
		是一个高效地将小文件放入HDFS块中的文件存档工具，他能够将多个小文件打包成一个HAR文件，这样就减少了namenode的内存使用。
2.Sequence File
		Sequence File由一系列的二进制key/value组成，如果key为文件名，value为文件内容，则可以将大批小文件合并成一个大文件。
3.CombineFileInputFormat
    	CombineFileInputFormat是一种新的InputFormat，用于将多个文件合并成一个单独的Split，另外，他会考虑数据的存储位置。
4.开启JVM重用
		对于大量小文件Job，可以开启JVM重用会减少45%运行时间。
		JVM重用原理：一个Map运行在一个JVM上，开启重用的话，该Map在JVM上运行完毕后，JVM继续运行其他Map。
		具体设置：mapreduce.job.jvm.numtasks值在10-20之间。

## 16.DataNode挂了流程

1、DataNode进程死亡造成DataNode无法与NameNode通信。
2、NameNode不会立即把该节点判定为死亡，要经过一段时间，这段时间暂称作超时时长。
3、HDFS默认的超时时长为10分钟+30秒。
4、如果要定义超时时间为TimeOut，则超时时长的计算公式为：
TimeOut = 2 * dfs.namenode.heartbeat.recheck.interval + 10 * dfs.heartbeat.interval
默认的：dfs.namenode.heartbeat.recheck.interval  5分钟
默认的：dfs.heartbeat.interval  3秒



## 18.hdfs存储数据类型

HDFS适合存储半结构化和非结构化数据，若有严格的结构化数据存储场景，也可以考虑采用Hbase的方案。

## 19.combine和merge区别

例如有两个键值对  <“a”,1>  和  <“a”,1>。
如果合并，会得到<“a”,2>。
如果归并，会得到<“a”,<1,1>>。

## 20.hdfs 默认副本数

默认的副本数为3。
-setrep：设置HDFS中文件的副本数量
[atguigu@hadoop102 hadoop-3.1.3]$ hadoop fs -setrep 10 
这里设置的副本数只是记录在NameNode的元数据中，是否真的会有这么多副本，还得看DataNode的数量。因为目前只有3台设备，最多也就3个副本，只有节点数的增加到10台时，副本数才能达到10。

## 21.hdfs上传

-put：等同于copyFromLocal

## 22.hdfs为什么128M

​		在HDFS中存储数据是以块（block）的形式存放在DataNode中的，块（block）的大小可以通过设置dfs.blocksize来实现；
​		在Hadoop2.x的版本中，文件块的默认大小是128M，老版本中默认是64M；

寻址时间：HDFS中找到目标文件块（block）所需要的时间。
原理：
文件块越大，寻址时间越短，但磁盘传输时间越长；
文件块越小，寻址时间越长，但磁盘传输时间越短。

**一、为什么HDFS中块（block）不能设置太大，也不能设置太小？**

**1.如果块设置过大，**

​		一方面，从磁盘传输数据的时间会明显大于寻址时间，导致程序在处理这块数据时，变得非常慢；
​		另一方面，mapreduce中的map任务通常一次只处理一个块中的数据，如果块过大运行速度也会很慢。

**2.如果块设置过小，**
		一方面存放大量小文件会占用NameNode中大量内存来存储元数据，而NameNode的内存是有限的，不可取；
		另一方面文件块过小，寻址时间增大，导致程序一直在找block的开始位置。

​		因而，块适当设置大一些，减少寻址时间，那么传输一个由多个块组成的文件的时间主要取决于磁盘的传输速率。

**二、 HDFS中块（block）的大小为什么设置为128M？**

1. HDFS中平均寻址时间大概为10ms；

2. 经过前人的大量测试发现，寻址时间为传输时间的1%时，为最佳状态；
    所以最佳传输时间为10ms/0.01=1000ms=1s

3. 目前磁盘的传输速率普遍为100MB/s；
    计算出最佳block大小：100MB/s x 1s = 100MB
    所以我们设定block大小为128MB。

ps：实际在工业生产中，磁盘传输速率为200MB/s时，一般设定block大小为256MB
       磁盘传输速率为400MB/s时，一般设定block大小为512MB

## 23.如何确定map数量

​		在进行map计算之前，MapReduce框架会根据输入文件计算输入数据分片，每个数据分片针对一个map任务，数据分片存储的并非数据本身，而是一个分片长度和一个记录数据的位置的数组。
**影响map个数的主要因素有：**

**1.文件的大小。**当块为128m时，如果输入文件为128m，会被划分为1个split；当块为256m，会被划分为2个split。
**2.文件的个数。**FileInputFormat按照文件分割split，并且只会分割大文件，即那些大小超过HDFS块的大小的文件。如果HDFS中dfs.block.size设置为128m，而输入的目录中文件有100个，则划分后的split个数至少为100个。
**3.splitSize的大小。**分片是按照splitszie的大小进行分割的，一个split的大小在没有设置的情况下，默认等于hdfs block的大小。

**MapTask的数量计算原则为：**
**（1）默认map个数**
如果不进行任何设置，默认的map个数是和blcok_size相关的。
default_num = total_size / block_size;
**（2）自定义设置分片的minSize、maxSize**
splitSize=Math.max(minSize, Math.min(maxSize, blockSize)

**查看map个数：**
1.在客户端提交任务时可在日志中查看
2.如果开启了Hadoop的History Server，可在UI界面中看到MapTask的数量。

## 25.Hadoop指令 

Hadoop fs 
-ls：显示信息
-Mkdir：创建文件夹
-put：本地上传文件到hdfs
-get：hdfs下载到本地
-cat：查看文件内容
-mv：在hdfs中移动内容

## 26.Hadoop需要启动的进程

**1）NameNode**它是hadoop中的主服务器，管理文件系统名称空间和对集群中存储的文件的访问，保存有metadate。
**2）SecondaryNameNode**它不是namenode的冗余守护进程，而是提供周期检查点和清理任务。帮助NN合并editslog，减少NN启动时间。
**3）DataNode**它负责管理连接到节点的存储（一个集群中可以有多个节点）。每个存储数据的节点运行一个datanode守护进程。
**4）ResourceManager**（JobTracker）JobTracker负责调度DataNode上的工作。每个DataNode有一个TaskTracker，它们执行实际工作。
**5）NodeManager**（TaskTracker）执行任务

**6）DFSZKFailoverController**高可用时它负责监控NN的状态，并及时的把状态信息写入ZK。它通过一个独立线程周期性的调用NN上的一个特定接口来获取NN的健康状态。FC也有选择谁作为Active NN的权利，因为最多只有两个节点，目前选择策略还比较简单（先到先得，轮换）。
**7）JournalNode** 高可用情况下存放namenode的editlog文件.

## 27.Hadoop和spark区别

1.Hadoop 每次 shuffle 操作后，必须写到磁盘，而 Spark在shuffle后不一定落盘，可以缓存到内存中		

2.Hadoop 的 shuffle 操作一定连着完整的 MapReduce 操作，冗余繁琐。而 Spark 基于 RDD 提供了丰富的算子操作，且 reduce 操作产生 shuffle 数据，可以缓存在内存中。

3.JVM 的优化: Hadoop 每次 MapReduce 操作，启动一个 Task 便会启动一次 JVM，基于进程的操作。而 Spark 每次操作是基于线程的，只在启动 Executor 是启动一次 JVM，内存的 Task 操作是在线程复用的。

## 28.压缩算法

| 压缩格式   | hadoop自带？ | 算法    | 文件扩展名 | 是否可切分 | 换成压缩格式后，原来的程序是否需要修改 |
| ---------- | ------------ | ------- | ---------- | ---------- | -------------------------------------- |
| DEFLATE    | 是，直接使用 | DEFLATE | .deflate   | 否         | 和文本处理一样，不需要修改             |
| Gzip       | 是，直接使用 | DEFLATE | .gz        | 否         | 和文本处理一样，不需要修改             |
| bzip2      | 是，直接使用 | bzip2   | .bz2       | **是**     | 和文本处理一样，不需要修改             |
| **LZO**    | 否，需要安装 | LZO     | .lzo       | **是**     | 需要建索引，还需要指定输入格式         |
| **Snappy** | 否，需要安装 | Snappy  | .snappy    | 否         | 和文本处理一样，不需要修改             |

| 压缩算法 | 原始文件大小 | 压缩文件大小 | 压缩速度 | 解压速度 |
| -------- | ------------ | ------------ | -------- | -------- |
| gzip     | 8.3GB        | 1.8GB        | 17.5MB/s | 58MB/s   |
| bzip2    | 8.3GB        | 1.1GB        | 2.4MB/s  | 9.5MB/s  |
| LZO      | 8.3GB        | 2.9GB        | 49.3MB/s | 74.6MB/s |

## 29.container

​		资源分配的体现就要用到一个抽象概念“容器”表示，将内存、 CPU、磁盘、网络等资源封装在一起，这样可以起到限定资源边界的作用。Container可看做一个可序列化Java对象

## 30.Hadoop HA的实现，脑裂问题

​		高可用最关键的策略是消除单点故障。
​		HA严格来说应该分成各个组件的HA机制：HDFS的HA和YARN的HA。
**HDFS的HA：**
​		HDFS的高可用主要是namenode的高可用，namenode的HA主要包括：
​		**共享日志存储、主备切换。**
**1、共享日志存储：**active namenode向共享文件系统写入日志文件，standby从共享文件系统读取日志与active保持同步。(**这里面涉及一个预防脑裂的隔离操作，内置fencing机制**)。基本原理就是用2N+1台JournalNode存储EditLog，每次写数据操作有大多数(>=N+1)返回成功时即认为该次写成功，数据不会丢失了。当然这个算法所能容忍的是最多有N台机器挂掉，如果多于N台挂掉，这个算法就失效了。这个原理是基于Paxos算法。
**2、主备切换：**每一个namenode运行着轻量级的ZKFC即ZKFailoverController。ZKFC主要包括两部分HealthMonitor和ActiveStandbyElector。HealthMonitor启动内部线程定时调用NameNode的HAServiceProtocol RPC接口，监控Namenode的健康状态并向ZKFC反馈；如果检测到namenode健康状态发生变化，ZKFC调用**ActiveStandbyElector**与zookeeper集群交互完成自动的主备选举，之后回调ZKFC的相应方法与namenode进行RPC通信，完成主备切换。

**Namenode中HA的脑裂问题：**
在namenode高可用下，为了预防出现多个active namenode(脑裂问题)，采用隔离(fencing)机制进行预防(实际就是用数值来隔离)：
**（1）共享存储fencing**：Quorum Journal内置了fencing机制，通过epoch number来解决隔离问题，数字大的可写，原来因为“假死”的namenode，再次去写JN的时候，比较epoch发现小于JN当中(主备切换后JN的epoch+1)，所以写不进去。
**（2）客户端fencing：**同一时间只允许一个namenode响应客户端请求(利用限时重连的方式，失败重连)。
**（3）Datanode的fencing：**同一时间只允许一个namenode向datanode下发指令(本质还是NN会向DN发送序列号和状态，如果NN中的序列号大于或等于DN中的序列号，才可以下发指令，主备切换后，序列号会变大，所以原来“假死”的NN不能向DN发送指令了，因为序列号小于DN的序列号了)。

















