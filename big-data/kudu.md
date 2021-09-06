# Kudu: Storage for Fast Analytics on Fast Data

## 一. kudu简介

 - kudu是开源的、针对结构化数据的存储引擎
 - 支持低延迟随机访问、高效的分析型访问
 - kudu是hadoop生态圈下的，支持多种访问模式，如 Impala、Spark、MapReduce

在hadoop生态中，对于结构化数据存储，通常有两种方式

 1. 对于**静态**数据集，常使用二进制格式如 Parquet（一种静态数据列格式）、Avro存储在HDFS中。**缺点是，这种方式不管是HDFS还是存储格式，都不提供更新单独行、或者高效的随机访问**
 2. **易变数据**通常存储在半结构化存储中，如HBase、Cassandra。缺点是，这些系统允许低延迟的行级别读/写，但对于顺序读，吞吐量远低于静态文件格式

**对静态数据的大批量顺序读分析，和对易变数据的低延迟的行级别随机访问的需求需要统一起来，这就是kudu的背景和定位。**

此外，没有kudu之前，开发人员有另外一种方式：在HBase中动态流式采集、更新数据，再定期把数据导入到parquet中以供后续分析，这种方式的缺点有：

 1. 需要写复杂的代码来管理两套系统间的数据流和同步
 2. 操作员需要在多个独立的系统间管理一致性的备份、安全、监控
 3. 数据在从Hbase流到HDFS静态格式有延迟，这一段时间新到来的数据是不能及时分析的
 4. 对于一些过去数据的更改、对已经导入到静态格式的数据进行隐私相关的删除，这些需求都很难实现

**kudu是一种全新的存储系统，用来填补高吞吐顺序读取系统（HDFS）和低延迟随机访问系统（HBase）之间的空白**，kudu也为行级别更新、删除提供了API，提供了类似parquet的表扫描。

## 二. kudu at high level
### 2.1 table and schema
kudu是结构化数据表的存储系统，kudu集群可以有任意多个表，而每个表，都是由列组成的，每一列都有名称、类型。用户在创建表时，必须定义表的schema，用户也可以随时创建、删除列。
每一列的类型都是显示的定义的，而不是像nosql一样，“everything is byte”，这样有两个优点，①显示类型便于特定类型特定编码，例如整数类型的位压缩 ②显示类型便于在BI上公开元数据。

注意，kudu不提供主键外的其他索引，也不提供唯一性约束，kudu要求每个表都有一个主键。

### 2.2 写/读操作
**写**：通过Insert、Update、Delete，目前不提供多行事务api
**读**：提供scan，从表中检索数据，目前只提供两种类型的谓词，列和常量值之间的比较，和复合主键的范围（[sql中的谓词有哪些](https://blog.csdn.net/qingdatiankong/article/details/77915015)，这里都是between）
读取除了应用谓词，还能为scan指定投影（投影：从表中选取出部分列进行操作）

### 2.3 一致性模型
kudu提供了两种一致性，默认的一致性是快照一致性。
默认情况下，kudu不提供外部一致性保证，HBase也不提供
什么是外部一致性？外部一致性是用户决定的，kudu提供了client之间手动传播时间戳的选项，传播令牌，保证两个客户机所做写入之间的因果关系。
如果传播令牌太复杂，kudu可以使用spanner的*wait-commit*

> 外部一致性：先后执行(执行区间不重叠)的多个事务，它们的在时间线上的顺序由用户确定，数据库无法感知到他们之间的关系，它们之间的一致性问题可以被认为是外部一致性问题。
> 外部一致性问题在lock
> base或mvcc实现的单机数据库中并不存在，而在分布式事务中比较常见的不满足外部一致性的情况是"写后读"，即提交成功后不能立即能够被读到；还有一种更复杂的情况，可以称之为"半写后读"，比如客户端依次执行T1:
> write A; T2: write B; T3 read A and
> B，如果没有全局时钟，那么就可能出现T3读取到最新的B，但是没能读取到最新的A。所以"内部"和"外部"最重要的区别是可见顺序由数据库(内部)确定，还是由用户(外部)确定。
> 作者：郁白
> 链接：[https://www.zhihu.com/question/56073588/answer/251047079](https://www.zhihu.com/question/56073588/answer/251047079)

### 2.4 timestamp
kudu在内部使用timestamps来进行并发控制，kudu不允许用户在写操作时手动设置timestamp，但是允许用户在读操作时设置timestamps，允许用户对过去执行时间点查询。

## 三. kudu at high level
### 3.1 集群角色
类似于Bigtable和GFS，kudu也是master-slave架构
一个master：负责元数据，多个slave负责存储数据

### 3.2 分区
kudu的表是水平切分的，水平分区后的称为tablet。任何行都可以基于其主键映射到唯一的tablet，保证随机访问操作如insert、update只影响单个tablet。

 - **kudu分区就叫tablet，二级分区叫rowSet，tablet由多个rowSet组成；Doris不一样，Doris分区叫partition、二级分区才叫tablet，partition由多个tablet组成**

不同于HBase和Cassandra，（HBase只提供基于key范围的分区，Cassandra只提供基于哈希的分区）kudu支持灵活的分区方式，创建表时，可以给表指定**partition schema**。
主要有 hash-partition（哈希分区）和 range-partition（范围分区）

### 3.3 复制
kudu使用raft来复制它的tablets，使用raft给每个tablet的逻辑操作日志达成一致（如 update、insert、delete）
当一个client想要执行写操作时，它首先会定位到replica leader并发送write-rpc，

 - 如果replica leader不再是领导者并且client的信息过时了，它便会拒绝请求，导致client失效并刷新其元数据缓存，并将请求重新发送给新的leader。
 - 如果replica leader还是leader，它将使用本地锁管理器 （local lock
   manager）来对其他并发的操作进行序列化操作，选择一个timestamp并通过raft对其他follower提出propose。如果大多数replicas接受了write-propose并写入本地write-ahead-log，则write被认为是可持久复制的、并在所有的replica上commit。

**kudu做了一些raft实现上的优化**：

 1. 在leader选举时，我们将raft 持久性元数据放置在活跃的服务器磁盘上，有助于选举快速收敛
 2. 当一个leader联系一个和它日志有分歧的follower时，老raft会一步一步往后退，直到找到第一个相同的日志；**kudu-raft维护了一个commitIndex，一次就能找到最近的一个相同log处**。

**kudu不复制tablet在磁盘上的数据，复制的是operation-log**

### 3.4 kudu master
kudu-master的职责：

 - catalog manager，跟踪存在的表和分片以及它们的schema等元数据。当表创建、变更时，master会协调这些操作
 - cluster maneger，跟踪server是否存活，以及挂掉后数据如何重新分配

为了最大化减少网络开销，client在向master查询后，**client会在本地保留一份metadata cache**，记录最近常访问的tablet的信息。任何时候客户端本地的缓存都有可能过期，如果客户端联系一个不再是leader的服务器来请求write，则请求会被拒绝。
通过master查找tablet在哪个server上我们认为不是瓶颈，270台集群上，99.99%的查询tablet位置在3.2ms完成。
## 四. Tablet store[重点]

分片存储需要达到几个目标

 - 快速的列式扫描：需要快速扫描parquet、ORC等大多文件格式
 - 低延迟随机更新：需要O(lg(n))随机查找复杂度
 - 持久稳定的性能：峰值性能和低谷性能不能差距过大


tablet由多个rowSet组成，有些rowSet仅存在内存中，称为**MemRowSet**；其他的存在内存和磁盘组合中，称为**DiskRowSet**。

**简单来说，kudu数据就是先分区得到tablet，tablet先按照行划分得到行集（RowSet），行集按照列拆分存储。**

 - 任何一行只存在一个rowSet中，因此，不同的rowSet没有交集。
 - 在任意时刻，每个tablet都会有唯一的一个MemRowSet，来记录最近插入的新行，有后台进程会周期性的将内存的数据flush进磁盘，MemRowSet会转变为一个or多个DiskRowSet

### 4.1 MemRowSet
MemRowSet是用**基于乐观锁的内存-并行-B树**实现的，大致基于MassTree，由于是分析性场景，读多写少，故用乐观锁。

 - 不支持从树中删数据，不支持任意位置的更新，只记录删除/更新的日志；推迟到刷盘时才删除
 - 我们将叶节点用指针连接起来，类似于B+树，但又不是B+树
 - 我们没有完全实现“前缀树”，没有需要极端高的随机访问性能
 - 和磁盘行集存储不太一样，MemRowSet是行存的，使用SSE2内存预取指令集在扫描前取数，使用LLVM和JIT编译行投影操作
 - MemRowSet是有序的，使用了order-preserving encoding对主键进行编码


### 4.2 DiskRowSet
在内存行集刷入磁盘时，每32MB划分一下，保证没有过大的磁盘行集。由于MemRowSet是有序的，刷入磁盘的DiskRowSet也是有序的。
**DiskRowSet有两部分，base data（列存） 和 derta store（行存）**
**base data存大块不变的数据，derta store存变更的数据**

##### 4.2.1 base（列存）
base data是按列拆分的。每一列是会按照page大小划分到连续的磁盘块中。

列存储时，使用了各种压缩编码：

 - dictionary-encoding
 - bit-shuffle
 - bit-packed
 - front-encoding
 - 还可选通用压缩，如gzpi、bzip2、LZ4

我们还维护了一个主键索引列，给每一行记录了编码后那一行的主键，基于编码后的主键可以使用**布隆过滤器**来寻找那一行是否存在。

##### 4.2.2 derta（行存）

列式存储下，数据是不可变更的，**update和delete操作通过derta store记录**。derta store也分为内存和磁盘两种，分别叫DertaMemStore和DertaFile

 - DertaMemStore是内存的并行B树，和上述MemRowSet一样
 - DertaFile是二进制的列块

两种derta store都维护了一个从 (row_offset,timestamp) 到 RowChangeList的Map，RowChangeList是对某一行二进制编码后的变更操作

DertaMemStore是存在内存中的，空间有限，故也需要刷盘。**当MemRowSet刷盘时，DertaMemStore也一并进行刷盘。**
刷盘后，MemRowSet转变为DiskRowSet，DertaMemStore转变为DertaFile。


### 4.3 Materialization & Compaction

##### 4.3.1 Lazy-Materialization 
如果给扫描指定了谓词，我们会给列准备**Lazy Materialization**，我猜大概是异步物化视图的意思，把谓词过滤后的列存储起来，如果之后的扫描中有过滤过的物化列，我们就会把谓词过滤后的列对应的行应用在这一批所有数据中，其他列有的数据就可以压根不读了

##### 4.3.2 Derta-Compaction
derta是行存的，base是列存的，随着dertas越来越多，扫描速度会变慢

因此kudu后台进程会周期的将derta数据合并入base中。由于是行变更写入列存中，只会写入变化的列，来减少不必要的IO

##### 4.3.3 RowSet-Compaction
写入DiskRowSet的数据就永远不变了吗？也不是。
kudu也有后台进程周期性的将不同的DiskRowSet合并，过程中会将删除的行移除，将在多个DiskRowSet重复出现的行删除


### 后面是实验、总结和致谢了，核心内容是以上，
### 完结撒花~
