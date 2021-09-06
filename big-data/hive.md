# Hive - A Warehousing Solution Over a Map-ReduceFramework

## 1.简介

Hadoop是一种开源的MapReduce实现（以下简称MapReduce为MR），MR框架太慢了，并且开发人员需要自定义写MR代码，因此诞生了Hive

 - Hive是搭建在Hadoop上的开源的数据仓库
 - Hive支持HQL（一种类SQL语言），它被编译成Hadoop上的MR任务
 - Hive还包含catalog系统，叫做Hive-metastore，包含schema和staticics，这在数据分析和查询优化很有用处。

## 2.Hive Database
### 2.1 数据模型
数据在Hive中被划分为Tables、Partitions、Bucket，这种数据划分是之后很多存储组件的开山鼻祖。

不同开源组件的数据划分，原始表都是table，后续划分只不过是名称不一样，大体思想是一致的：
|          | Hive      | Doris     | Kudu   |
| -------- | --------- | --------- | ------ |
| 一级划分 | Partition | Partition | Tablet |
| 二级划分 | Bucket    | Bucket    | RowSet |


### 2.2 查询语言
HQL和SQL没啥大区别，SQL支持的，HQL也支持，比如select、project、join、aggregate、union等，不多介绍了

概念补充：sql project就是，select col_1,col_2,col_3 只返回指定列的操作叫做投影（project）

## 3.Hive 架构
<img src="https://img-blog.csdnimg.cn/20210718150120429.png" width=40%>

**注意，Hive的数据是存在HDFS中的，元数据是存在mysql中的，hive本身不存数据，可以把hive简单理解为将类SQL转为MR任务的框架**
[https://blog.csdn.net/qq_31246691/article/details/79467358](https://blog.csdn.net/qq_31246691/article/details/79467358)

Hive的主要组件

 - **External interface**：有CLI命令行、webUI、应用编程接口JDBC（java）、ODBC（C++）
 - **thirft server**：thrift是一个跨语言服务框架，支持客户端不同的语言需求
 - **MetaStore**：metastore是系统级目录
 - **Driver**：driver在编译、优化、执行的过程中管理HiveQL的生命周期。在从CLI、webUI或者thrift server接收到HiveQL语句时，会创建一个session handle，后续用来跟踪语句的执行时间、输出行数等信息。
 - **Compiler**：driver收到HQL会调用compiler，compiler将HQL语句转化为由MapReduce任务组成的DAG（有向无环图），目前（发表论文时）Hive的执行引擎使用的是Hadoop

接下来详细介绍了metastore和compiler这两个组件
### 3.1 MetaStore
metastore是Hive系统级别的目录，记录了存储在Hive中表的元数据，
metastore包含以下内容：

①库；②表，表的元信息有：列及类型、所有者、底层数据的位置、数据格式、数据备份信息、SerDe（序列化器、反序列化器的方法实现类）；③分区信息

注意，metastore需要对随机访问的数据的更新进行元数据的更新，因此不能用HDFS（HDFS适合顺序扫描），metastore使用的是传统的关系型数据库如mysql或者文件系统如（NFS、AFS）

### 3.2 Compiler
对于insert和查询语句，compiler将HQL转化为MR组成的DAG

 1. Parser（解析器）：将查询语句转化为解析树 Semantic
 2. Analyzer（语义分析器）：将解析树转化为基于锁的内部的查询表示，从metastore检索信息 
 3. Logical Plan Generator（逻辑计划生成器）： 将查询转化为一颗由逻辑运算符组成的树
 4. Optimizer （优化器）：对Logical Plan多次重写

	优化器多次重写分为以下几个方面：
	
	 - 将共享join键的多个join合并为一个多路join
	 - 提前对列剪枝，将谓词推到表扫描操作附近
	 - 对于分区表，不去查询用不到的分区
	 - 对于采样查询，修剪不需要的bucket

 5. Physical Plan Generator（物理计划生成器）：转化为MapReduce任务组成的DAG图

#### 论文很精简，以上是主要内容，完成撒花~