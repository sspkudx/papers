# Mesa: Near Real-Time, Scalable Data Warehousing

这篇论文发表于2013，阅读原因是想看看apache doris的存储系统的设计

## 一.摘要&简介
Mesa是谷歌存放广告的一个数据仓库，它有以下特点

 - **原子性更新**（可能一条用户的动作会影响很多相关数据的更新，这些更新需要原子性、一次性完成）
 - **高度可拓展**（方便应对后续源源不断的数据增加）
 - **高可用**（不能有单点故障，就算整个数据中心挂了，数据仓库也不能宕机）
 - **查询性能高**
 - **强一致性和正确性**
 - **近乎实时更新且吞吐量大**（1s100w行更新的吞吐量）、支持连续更新
 - **在线数据和元数据转化**（用户常常修改表的schema，比如更改数据粒度，这些更改不能影响正常的查询和更新操作）
 - 具有事务处理系统的**ACID**能力

**Mesa是分布式、可复制、高可用的结构化数据处理、存储、查询系统**
Mesa使用了Colossus（Google下一代文件系统）、BigTable和MapReduce。

 - 为了实现存储可拓展和高可用，数据是**水平分区**并备份的。 
 - 为了实现更新时的一致性和可重复查询，底层数据是**多版本**的。
 - 为了实现更新的可拓展，**数据是按批更新**的，分配一个版本号，并**定期合并到mesa中**
 - 为了实现跨数据中心的数据一致性，采用了paxos

## 二.Mesa存储系统
mesa里面的数据是多维度的，**维度列（称为key列），指标列（称为value列）**，**维度列**用于分组和排序, **指标列**可通过聚合函数SUM, COUNT, MIN, MAX, REPLACE, HLL_UNION, BITMAP_UNION等累加起来
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210712163129709.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTY2NjczNg==,size_16,color_FFFFFF,t_70)


由于许多维度属性是分层的(甚至可能有多个层次结构，例如，日期维度可以按日/月/年级别或财政周/季度/年级别组织数据)，**单个事实可以基于这些维度层次结构聚集在多个物化视图中**，以支持使用钻取和上滚进行数据分析。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210618120319470.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTY2NjczNg==,size_16,color_FFFFFF,t_70)
上图展示了三个mesa表，它们根据不同的维度聚合，底层的事实表是同一个
**注意：C表是物化视图的表**
### 2.1 数据模型
Mesa使用表table来维护数据，每个表都有对应的schema来指定表的结构，schema**指定了键空间K和值空间V，还指定了聚合函数F**，聚合函数用来聚集选中的值，**必须满足结合律 F(F(v1,v2),v3)=F(v1,F(v2,v3))**，**不一定满足交换律**，比如，F(v0,v1)=v0，这个聚合函数用来替换V

### 2.2 更新和查询
为了提高更新的吞吐量，mesa采用批量更新

 - Mesa的更新：指定版本号n和一组表单行（表名、键、值），每次更新最多包含一个聚合值
 - Mesa的查询：查询由版本号n和键空间上的谓词P组成，匹配版本0~n所有满足的**键**，响应的键是更新该键的**所用值**的总和
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210619054533901.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTY2NjczNg==,size_16,color_FFFFFF,t_70)
对于表A和表B，维护更新的一致性。
对于物化视图表的更新，mesa是自动进行的，因为可以基于已更新的维度表更新。Mesa限定物化视图的表采用相同的聚合函数（比如都是Sum()）

**Mesa严格按照版本号顺序进行更新，在进行下一个更新前，总是将一个更新完全合并**。从而保证原子性，用户无法看到部分更新。
按照顺序操作也运行mesa对于错误的操作**使用逆操作进行回滚**。顺序更新确保逆操作不会先于正面操作先完成。

### 2.3数据版本管理
versioned data在mesa中非常重要，**聚合的数据会小很多**，但是对于所有的版本数据，都应用一遍聚合并存储下来，代价也太高了。
如何解决这个问题？**mesa只预先聚合了特定版本的数据，并使用增量derta的形式存储**。
**版本号v1和v2(v1<= v2)之间的增量表示为[v1，v2]**，单例增量如对版本Vk的增量采用[Vk，Vk]表示。
一个derta[v1，v2]和另一个derta[v2+1，v3]可以聚合生成[v1，v3]

> derta[v1,v2]+derta[v2+1,v3]=derta[v1,v3]

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210619062433154.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTY2NjczNg==,size_16,color_FFFFFF,t_70)
mesa新生成的版本B'，会定期删除低版本的B，累计增量和低于B'的所有单例增量

 - 单例增量singleton [Vk，Vk]
 - 累计增量cumulative [V,U] 其中V<U

> 对于查询版本n，可以表示为derta[0,n]=[0,base]+[base+1,base+1]+[base+2,base+2]+...+[n,n]
> 也可以采用累计增量 derta[0,n]=[0,base]+[base+1,V1]+[V1+1,V2]+...+[Vk,n]

目前mesa在生产中采用的是两级增量策略的变体

### 2.4物理数据和索引格式
mesa的增量derta是根据derta compaction policy创建和删除的。**derta一旦创建后就是不可变的**。存储需要存储效率高并支持对于特定键的快速查找，每个mesa表都有一个or多个表索引。
**行被组织为行块，每个行块被转置后按照列压缩**（同一类数据方便采取相同的压缩算法）。
由于读和查询是主要需求，所以在选取压缩算法时，我们强调压缩比和**读/解压时间**高于写/压缩率时间，压缩速度可以慢一点，解压和读取一定要快。

## 三.Mesa架构
### 3.1单数据中心实例
mesa使用了BigTable和Colossus

 - 所有持续化的元数据存储在Bigtable中
 - 所有数据文件存储在Colossus中

mesa由两个子系统组成，①更新和维护系统（也就是存储）②查询系统
两个子系统**解耦**，允许它们独立伸缩

##### 3.1.1 更新/维护子系统

 - 保证数据是正确的、最新的、并为查询做优化
 - 在后台执行各种操作：加载更新、table compaction、schema change，这一组框架称为controller/worker
 - controller作用：确定任务、管理所有表的元数据、工作调度器、worker队列管理者（只负责安排工作和更新元数据）
 - worker作用：空闲的worker轮询controller来请求相关任务，来完成更新、compaction、schema change、表校验和等任务

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210619111446385.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTY2NjczNg==,size_16,color_FFFFFF,t_70)
controller/worker框架仅用于更新和维护工作，这些组件可以重启并不影响用户。
此外控制器是按照表分片的，**允许框架拓展**
控制器是无状态的，所有状态信息再bigtable中保持一致


##### 3.1.2 查询子系统（查询引擎）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210619111939749.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTY2NjczNg==,size_16,color_FFFFFF,t_70)
查询服务器用来接收用户的查询、查找表元数据、确定所需数据的位置、动态聚合。
**Mesa提供了一个有限的查询引擎，支持条件过滤和分组聚合，上层可以对接更高级的查询引擎**，如MySQL、F1和Dremel，它们提供更丰富的SQL功能，如join

### 3.2多数据中心部署
##### 3.2.1 一致性更新机制
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210620081137994.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTY2NjczNg==,size_16,color_FFFFFF,t_70)
如上所示，Mesa的committer负责协调所有的mesa实例更新，一次一个版本
committer为每次更新分配一个新版本号，并将与更新相关的元数据发布到版本数据库，committer自己是无状态的，mesa实例运行在多个数据中心来保证HA
mesa的controller监听版本数据库的变化，检测是否需要更新、并分配工作给update workers
新查询总是针对已提交的版本进行，更新是批量应用的
**Mesa不需要在query和update之间设定任何lock**
**所有更新的数据都是mesa实例异步并入的，只有元数据基于paxos同步复制**，这就是为什么查询和更新吞吐量都很大的原因

##### 3.2.2 新Mesa实例
peer-to-peer load mechanism 点对点加载机制
在controller/worker框架中，还有一种其他的worker，**专门负责将表从一个mesa实例复制到另一个**，然后使用update worker更新，再提供给外部查询

## 四.优化
### 4.1 查询服务器（query server）的性能优化

 - **derta pruning 增量剪枝**：如果查询中的过滤器不在此范围内，可以删除derta，因为用不上。比如按照最近时间序列的数据查询，不在查找范围内的时间增量直接剪枝
 - **scan-to-seek优化**：比如对一个具有索引A和B的表，查找B=2，根据最左匹配原则，过滤器B=2不会用到索引，需要扫描每一行。**scan-to-seek会把它转化为构建一个前缀**，用到A的索引，做到跳跃查找。比如表A的第一个值是1，那么查询期间会使用索引搜索所有A=1，B=2的行，跳过所以A=1，B<2的行，如果A的下一个值是4，查询会直接跳到A=4，B=2。这种优化会显著加速查询速度
 - **resume key**：对于每个子任务，mesa附带一个resume key，如果查询服务器挂了，可以根据resume key去恢复查询，而不需要重新执行整个查询

### 4.2 并行化worker操作

 - controller/worker 框架由一个controller控制，它协调多个worker工作

 - mesa使用mapreduce框架来并行执行不同类型的worker，其中MapReduce操作跨越多个mapper和reducer是难点，为了实现这种并行化，当写入任何derta时，mesa worker会对每第s行进行采样，算法没看懂。。大概说的是worker会并行处理任务


### 4.3 Mesa中的schema change
用户常常会修改schema change，例如：增/删列，增/删索引，增/删表（特别是 roll-up table），schema change必须是在线完成，查询和更新在此期间都不能阻塞。
有两种schema change方式
 - naive change：简单但代价高昂。生成新schema之后再将查询切换到新版本

 - linked schema change：有优化，相比naive节省了50%磁盘空间，适用于大多数场景。old/new schema version作为更新和压缩，新schema让查询立即可见，查询时对新列适用默认值，并为其他表构建新的derta

### 4.4 减轻数据损坏的问题
对于数万台机器，简单的文件校验和不足以抵御故障了，因为可能连cpu都会发生短暂的故障
尽管所有的mesa实例存储的是一样的数据，但是增量数据derta version不一样，mesa利用这种多样性，**通过在线、离线数据验证组合来防止机器和人为故障**

 - pre-instance verification：在线检查发生于每次更新和查询。当写入更新derta时，mesa执行 行顺序、键范围和聚合值校验，dera是顺序存储的
 - global offline check：mesa定期执行全局离线校验，对所有mesa实例的表的索引进行全局校验和。当索引的校验和不匹配时，mesa会告警
 - mesa还会有一个轻量级的离线进程，进行全局聚合的校验，聚合是在云数据上进行的，因此效率较高
 - 当表损坏时，去离mesa实例最近物理地址的数据中心恢复，加载未损坏的副本


## 五.经验教训
 - 随着数据量和查询的增大，**去中心化架构和云计算范式**结合非常有效
 - 数据从本地磁盘读比远端读取当然更快，但是可以切片分散在远端存储，**通过大量的并行操作来抵消远端读网络开销造成的性能下降**
 - 模块化、分层设计非常重要，mesa复用了colossus和Bigtable


## 六.结论
本论文，我们展示了一个end-to-end的设计、跨数据中心进行地理复制、近乎实时、可拓展的叫做Mesa的数据仓库。

Mesa支持在线更新，提供强一致性、多版本数据和事务正确性保障，通过引入数据多版本来保证更新原子性，controller/worker框架允许水平拓展worker来提供需要的算力。

Mesa是一个云数据仓库，运行在云端动态提供的机器上，不依赖本地磁盘，支持PB级别数据量和大查询和更新负载。特别的，Mesa支持高吞吐的更新、低延迟的点查询，批量提取查询负载。