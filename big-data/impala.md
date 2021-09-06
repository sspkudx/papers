# Impala: A Modern, Open-Source SQL Engine for Hadoop

## 前言

impala是2014年开源的MPP SQL引擎，阅读这篇论文是想深入对比impala和doris的区别（虽然doris的查询引擎大部分用的是impala）

该论文发表于2014年
## 一.摘要&简介

 - Impala是一个开源的、**集成于Hadoop的MPP（massive parallel process） SQL引擎**，Impala为Hadoop上的查询分析提供了**低延迟**和**高并发**。SQL on Hadoop
 - 单纯是一个计算引擎，和底层的存储解耦合
 - 能读取大多数数据格式，如**Parquet**（性能最好的列存）、Avro（二进制行格式）、RCFile（一种遗留的列状格式）
 - 为了达到最高效果，**需要使用HDFS作为底层存储**
 - 在多用户查询负载下，相比其他MPP引擎速度更为显著（对高并发查询支撑更为优秀）
 - **由于HDFS存储的局限性，impala不支持update和delete**，基本只支持批量导入


## 二.架构

<img src="https://img-blog.csdnimg.cn/20210621171806764.png" width=80%>

Impala由三种进程组成
 - impalad
 - statestored
 - catalogd

接下来一一介绍
### 2.1 Impalad（解析SQL、协调资源并计算）
Impalad表示为impalad守护进程（Impala daemon），作用如下：
1. 接受来自client的查询
2. 让 查询并行，编排调度query的执行顺序在多个节点上
3. 把中间查询结果返回给coordinator

<img src=https://img-blog.csdnimg.cn/20210513160129459.png width=40%>

观察上图发现，一个Impalad又分为三个角色，分别是**编译者、协调者、查询者**
作用分别是：解析SQL；规划、协调计算资源；计算

| 角色              | 作用               |
| ----------------- | ------------------ |
| query compiler    | 解析SQL            |
| query coordinator | 规划、协调计算资源 |
| query executor    | 计算               |

### 2.2 statestored（监控、广播）
**在大型分布式系统下，一般工作节点都会同步信息、汇报状态给管理节点**（主动汇报心跳等信息），statestored也就是这样一个管理节点的角色，不过在impala中都用进程来代替了。

statestored，状态管理进程，**负责检测所有impala daemon的健康状态**（心跳检测，并不断将findings传递给守护进程，**一个集群只需要一个**
原因是statestore服务不需要HA，如果服务异常或者主机损坏，可以直接重新添加一个statestore服务即可

出现问题时提供一致性帮助，向协调员**广播元数据**

**发布-订阅模式**，将元数据更改分发给订阅者，将集群元数据发送给所有的impala进程
如果一个impala守护进程由于硬件、软件、网络原因离线了，statestore通知所有的impala守护进程，避免后续unreachable的visit

### 2.3 catalogd（元数据同步）
**catalog进程负责元数据同步**

catalog是管理元数据的，从第三方平台（hive metastore、hdfs namenode）拉取元数据，将元数据整合为impala兼容的catalog结构，这个特性让我们可以相对较快的加入新metadata

**Impala cata服务通过statestore广播机制，将元数据给impalad**


下图展示了一个SQL在impala上是如何查询的
<img src="https://img-blog.csdnimg.cn/20210621171806764.png" width=80%>
步骤：
 1. 发送SQL给Impalad
 2. Impalad中的query planner解析
 3. Impalad的query coordinator协调、分发子任务给各个执行节点
 4. Impalad的query executor计算
 5. executor返回结果给coordinator，子结果整合得到最终结果
 6. coordinator返回结果给调用方

其中还涉及到心跳汇报、元数据同步等过程，论文中没列出

## 三.Front-end
**前端作用**：前端将SQL转为查询计划，交给后端执行
过程：①查询解析②语义分析③查询规划/优化

可执行查询计划分为两个阶段，**(1)单节点规划 (2)并行化和碎片化处理**
（1）单点规划：
第一阶段将语法树分解为多个plan node（计划节点），包括：HDFS/HBase Scan、hash join、cross join、union、sort、top-n
（2）单点转分布式执行
第二阶段将单节点作为输入，生成分布式执行计划。
一般的目的是最小化数据块移动（因为远端读很费时间）并最大化扫描局部性，通过必要的交换节点、本地聚合节点来实现分布式计划。
<img src="https://img-blog.csdnimg.cn/20210630110836560.png" width=80%>


## 四.Back-end
back-end是用c++写的，使用runtime code generation，与java引擎相比，使用较小的内存开销。
执行模型是带有Exchange操作的**volcano模型**，整个执行过程**管道化** pipeline-able
需要大量内存的操作，如hash join、sorting、analytic function，在内存不够时溢写到磁盘。

### 4.1 Runtime Code Generation
使用LLVM 运行时代码生成，提升5倍以上的性能。LLVM is modular and reusable.LLVM 允许像impala这样的应用 在运行的进程中执行 **just-in-time**（JIT）编译。可以充分利用现代优化器的优点，并为许多体系结构生成机器码。

> 如果没有运行时代码生成，为了处理编译时不知道的运行时信息，那么不得不需要不高效的函数。
> 例如，只处理整数类型的行解析函数比能处理所有类型的通用函数要快，但是，要扫描的文件的格式在编译时是未知的，因此需要一个通用函数

大量运行时开销来源是虚函数，如果运行时对象实例是已知的，那么我们可以使用代码生成将虚函数调用替换为直接调用正确的函数，然后可以内联该函数。


### 4.2 I/O Management
对于所有Hadoop上的SQL系统来说，如何高效从HDFS检索数据是一个挑战。
为了以接近硬件速度从内存or磁盘读取数据，impala使用了HDFS一个特性，**short-circuit local reads**，“使短路-本地读”，当从本地磁盘读时，绕过DataNode协议。
Impala的IO管理系统比较优秀，impala读吞吐量比其他系统高4~8倍

### 4.3 Storage Formats
Impala支持大多数数据格式，如Avro、RC、Parquet，这些格式可以与不同的压缩算法组合，如snappy、gzip、bz2.
**Impala最推荐parquet**，parquet是Apache一种先进的、列式存储格式，提供高压缩和高扫描效率.parquet针对大数据块进行了优化，内置了对嵌套数据的支持，受Google Dremel的启发，parquet按列存嵌套数据，用最少的信息对它们进行扩充，以便扫描时从列数据重新组装嵌套结构。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210630123320975.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zOTY2NjczNg==,size_16,color_FFFFFF,t_70)



## 五.资源/工作负载 管理
YARN资源调度框架是**集中式架构**，对资源的请求由中央资源管理器分配。这种体系结构的优点是允许在完全了解集群状态下做出决策，但也对**资源的获取造成了严重的延迟**，由于Impala的工作负载为每秒数千个查询，这样请求资源和响应周期太长了。
解决这个问题分两个方面，①实现了一个互补但独立的准入控制机制，允许用户控制他们的工作负载，不需要昂贵的集中决策，相当于“放权”。②设计了一个中间服务，位于impala和YARN之间，该服务叫做Llam，目的是纠正一些阻抗不匹配，它实现了资源缓存、组调度和增量分配变化。
对于没有达到Llama缓存的资源请求，任然将实际的调度推给Yarn。

### 6.1 Llama and YARN
Llama是一个独立的守护进程，所有的Impala守护进程向它发送每个查询的资源请求，每个资源请求都与一个资源池相关，资源池定义了查询可以使用集群可用资源的公平份额。

如果资源池的资源在Llama资源缓存里可用，Llama会立刻将其返回给查询，这种策略让资源竞争程度较低时能够快速绕过Yarn资源分配算法，否则，将请求转发给Yarn。


### 6.2 Admission Control 准入控制
Impala还有一个内置的准入控制机制来抑制传入请求，请求被分配给一个资源池，然后定义每个最大资源池最大并发量，最大内存使用量。这个admission controller设计为快速且分散的，因此运行任何Impala进程传入请求，无需向中央处理器发出同步请求，准入控制机制可以理解为一个简单的节流机制。

## 六.测评与结论
### 6.1测评
下面实验都是运行在21台相同配置机器的集群上的
其中，压缩都采用Snappy压缩
 - Impala使用Parquet格式上的Impala
 - ORC格式上的hive
 - RCFile上的presto
 - Parquet上的spark-SQL

下图是单用户查询性能，比其他引擎平均快6.7倍：

<img src="https://img-blog.csdnimg.cn/20210621172407985.png" width=40%>

下图是多用户并发查询性能，Impala性能优势更为明显，快6.7~18倍，吞吐量也要大很多：

<img src="https://img-blog.csdnimg.cn/20210621173034871.png" width=70%>

### 6.2 未来工作
虽然现在impala比较领先，还需要以下工作：

 - 增加嵌套关系模式，增加复杂列类型，如：struct、array、map
 - 增加 join、agg、sort的性能，更广泛地使用**运行时代码生成**（runtime code generation，将查询中需要物化视图的数据转为内存中柱状格式，以便使用CPU-SIMD指令
 - 自动数据转换，通常数据是以XML、Json这样面向行的格式新加入到系统，需要自动转为Parquet等面向列的格式以提高性能
 - 多租户下的资源管理问题，与现有的YARN继承，难以适应低延迟、高吞吐的工作负载，需待解决
 - 积极拓展impala对云端数据的访问，如Amazon S3

### 6.3 结论
本文介绍了cloudera impala，一个开源的SQL引擎，Impala在分析性场景对比传统的RDBMS有独特的优势，比如多文件格式、处理性能，这意味着更为广泛的计算任务不需要移动数据，在单一系统即可完成。

#### Impala快的原因：
1.使用了运行时代码生成的技术，对比java来说减少了内存的压力，可以根据地层类型进行调优
2.处理进程时无需每次启动
使用服务常驻的方式避免频繁的启动
3.impala所有的聚合操作都是在本地执行预聚合，再通过网络进行最终的聚合，预聚合减少网络传输的压力
4.使用parquert和kudu
5.避免数据读写磁盘，减少写入磁盘