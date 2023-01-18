# Chapter4 分布式数据库HBase

> shenhao  王洲烽

## 4.0 产生的背景

&emsp;&emsp;在介绍HBase之前，我们首先来思考一下Hadoop的局限性？  
> ps：奇怪的冷知识又增加了，勤思考，多动脑，不仅预防老年痴呆，而且还能升职加薪
>
> HBase给我冲冲冲！！！！！

<center><img src="https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.0.png" style="zoom:80%;" /></center>

### 4.0.1 Hadoop的局限

&emsp;&emsp;Hadoop可以通过HDFS来存储**结构化**、**半结构**甚至**非结构化**的数据，它是传统数据库的补充，是海量数据存储的最佳方法。它针对大文件的存储、批量访问和流式访问都做了优化，同时也通过多副本解决了容灾问题。  
&emsp;&emsp;但是Hadoop的缺陷在于它只能执行**批处理**，并且只能以**顺序的方式访问数据**。这意味着，即使是最简单的工作，也必须搜索整个数据库，**无法实现对数据的随机访问**。反观传统的关系型数据库，其主要特点就在于随机访问，但它们却不能用于海量数据的存储。  
&emsp;&emsp;在这种情况下，必须有一种新的解决方案存储海量数据和访问单个时间单元（随机访问）中的任意数据点，HBase因此孕育而生！

> 补充：Cassandra、ouchDB、Dynamo和MongoDB也能存储海量数据并支持随机访问。

### 4.0.2 HBase VS 传统数据库

&emsp;&emsp;首先，我们来了解一下数据结构的分类：  
- **结构化数据**：即以关系型数据库表形式管理的数据；
- **半结构化数据**：具有非关系模型、基本固定结构模式的数据，例如日志文件、XML文档、JSON文档、Email等；
- **非结构化数据**：没有固定模式的数据，如Word、PDF、PPT、Excel、各种格式的图片、视频等。

&emsp;&emsp;为了存储不同的数据结构，也诞生了众多类型的数据库：
- **关系型数据库**：关系型数据库模型，是把复杂的数据结构归结为简单的二元关系（即二维表格形式）。
  - 代表软件：**MySQL**

- **键值存储数据库**：键值数据库是一种非关系数据库，它使用简单的键值方法来存储数据。键值数据库将数据存储为键值对集合，其中键作为唯一标识符。
  - 代表软件：**Redis**

- **列存储数据库**：列式存储（column-based）是相对于传统关系型数据库的行式存储（Row-basedstorage）来说的。简单来说，两者的区别就是对表中数据的存储形式的差异。
  - 代表软件：**HBase**

- **面向文档数据库**：此类数据库可存放并获取文档，可以是XML、JSON、BSON等格式，这些文档具备可描述性（self-describing），呈现分层的树状结构（hierarchical tree data structure），可以包含映射表、集合和纯量值。数据库中的文档彼此相似，但不必完全相同。文档数据库所存放的文档，就相当于键值数据库所存放的“值”。文档数据库可视为其值可查的键值数据库。
  - 代表软件：**MongoDB**

- **图形数据库**：图形数据库顾名思义，就是一种存储图形关系的数据库。图形数据库是[NoSQL数据库](https://blog.csdn.net/qq_14927217/article/details/73835463)的一种类型，可以用于存储实体之间的关系信息。最常见例子就是社会网络中人与人之间的关系。
  - 代表软件：**Neo4J**、ArangoDB、OrientDB、FlockDB、GraphDB、InfiniteGraph、Titan和Cayley等
  
- **搜索引擎数据库**：搜索引擎数据库是一类专门用于数据内容搜索的非关系数据库。搜索引擎数据库使用索引对数据中的相似特征进行归类，并提高搜索能力。搜索引擎数据库经过优化，以处理可能内容很长的半结构化或非结构化数据，它们通常提供专业的方法，例如全文搜索、复杂搜索表达式和搜索结果排名等。 
  - 代表软件：**Solr**、**Elasticsearch**等

------
&emsp;&emsp;**补充了那么多冷知识，那我们来看一下为什么传统数据库不能适应如今大数据的时代呢？**

<center><img src="https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch2.0.12.png" style="zoom: 67%;" /></center>

&emsp;&emsp;随着Web 2.0应用的不断发展，传统的关系数据库已经无法满足需求，无论在**数据高并发**方面，还是在**高可拓展性**和**高可用性方面**，传统的关系型数据库都显得力不从心，其完善的事务机制和高效的查询机制也成为“鸡肋”。因此，包括HBase在内的非关系型数据库的逐渐崭露头角！

&emsp;&emsp;HBase与传统的关系型数据库的区别主要在于：  
- **数据类型**：关系型数据库数据类型较为丰富，数据类型有`int`、`date`、`long`等。Hbase数据类型简单，每个数据都被存储为未经解释的字符串，用户需要自己编写程序把字符串解析成不同的数据类型。
- **数据操作**：关系型数据库存在增删改查，还有我们比较熟悉的联表操作，效率较低。Hbase不会把数据进行充分的规范化。很多数据是存在一张表里，避免了低效率的连接操作。
- **存储模式**：关系型数据库采用行模式存储，Hbase是基于列存储。
- **数据索引**：关系型数据库可以对不同的列构建复杂的索引结构。Hbase支持对行键的索引。
- **数据维护**：更新操作时，关系型数据库会把数据替换掉，Hbase会保留旧的版本数据一段时间，到了一定期限才会在后台清理数据。
- **可伸缩性**：关系型数据库很难实现水平扩展，Hbase采用分布式集群存储，水平扩展性较好

&emsp;&emsp;但是相比于关系型数据库，HBase也有自身的局限，由于其不支持事务，因此无法实现跨行的原子性。

## 4.1 概述

### 4.1.1 HBase 简介

&emsp;&emsp;HBase是构建在Hadoop文件系统之上的一个**高可靠、高性能、面向列、可伸缩**的**分布式数据库**，主要用来存储**非结构化**和**半结构化**的松散数据。它是谷歌 BigTable 的开源实现，可以通过水平扩展的方式，利用廉价计算机集群处理由超过10亿行数据和数百万列元素组成的数据表。  
&emsp;&emsp;HBase旨在提供对大量结构化数据的快速随机访问。它利用Hadoop文件系统（HDFS）提供的容错功能，同时作为Hadoop生态系统的一部分，提供对Hadoop文件系统中的数据的随机**实时**读写访问。  
&emsp;&emsp;在Hadoop生态系统中，HBase利用Hadoop MapReduce来处理HBase中的海量数据，实现高性能计算；利用ZooKeeper作为协同服务，实现稳定服务和失败恢复；使用HDFS作为高可靠的底层存储，利用廉价集群提供海量数据存储能力。（当然，HBase也可以直接使用本地文件系统而不用HDFS作为底层数据存储方式。）Sqoop为HBase提供了高效、便捷的RDBMS数据导入功能，Pig和Hive为HBase提供了高层语言支持。

<center><img src="https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.1.1.png" alt="Hadoop生态系统" style="zoom:50%;" /></center>

### 4.1.2 HBase 访问接口

HBase提供了众多的访问方式，详见下表。


| 类型                | 特点                                                 | 场合                                          |
| :------------------ | ---------------------------------------------------- | --------------------------------------------- |
| **Native Java API** | 最常规和高效的访问方式                               | 适合Hadoop MapReduce作业并行批处理HBase表数据 |
| **HBase Shell**     | HBase的命令行工具，最简单的接口                      | 适合HBase管理使用                             |
| **Thrift Gateway**  | 利用Thrift序列化技术，支持C++、PHP、Python等多种语言 | 适合其他异构系统在线访问HBase表数据           |
| **REST Gateway**    | 解除了语言限制                                       | 支持REST风格的Http API访问HBase               |
| **Pig**             | 使用Pig Latin流式编程语言来处理HBase中的数据         | 适合做数据统计                                |
| **Hive**            | 简单                                                 | 当需要以类似SQL语言方式来访问HBase的时候  |

## 4.2 HBase 数据模型

&emsp;&emsp;数据模型是一个数据库产品的核心，本节介绍HBase列族数据模型，包括表、行键、列族、列限定符、单元格和时间戳等概念，并阐述HBase数据库的概念视图和物理视图的差别等。

### 4.2.1 数据模型概述

- HBase是一个稀疏、多维度、排序的映射表，这张表的**索引**是**行键、列族、列限定符和时间戳**。
- 每个值是一个未经解释的字符串，没有数据类型。
- 用户在表中存储数据，每一行都有一个可排序的行键和任意多的列。
- 表在水平方向由一个或者多个列族组成，一个列族中可以包含任意多个列，**同一个列族里面的数据存储在一起**。
- 列族支持动态扩展，可以很轻松地添加一个列族或列，无需预先定义列的数量以及类型，所有列均以字符串形式存储。因此对于整个映射表的每行数据而言，有些列的值是空的，所以说HBase是稀疏的。
- HBase中执行**更新**操作时，并不会删除数据旧的版本，而是**生成一个新的版本**，旧有的版本仍然保留（这是和HDFS只允许追加不允许修改的特性相关的）。

### 4.2.2 数据模型的相关概念

- **表**：HBase采用表来组织数据，表由行和列组成，列划分为若干个列族。
- **行**：每个HBase表都由若干行组成，每个行由行键（row key）来标识。
- **列族**：一个HBase表被分组成许多“列族”（Column Family）的集合，它是基本的访问控制单元。表中的每个列都归属于某个列族，数据可以被存放到列族的某个列下面（列族需要先创建好）。在创建完列族以后，就可以使用同一个列族当中的列。列名都以列族作为前缀。例如，`courses:history`和`courses:math`这两个列都属于`courses`这个列族。
- **列限定符**：列族里的数据通过列限定符（或列）来定位。
- **单元格**：在HBase表中，通过行、列族和列限定符确定一个“单元格”（cell），单元格中存储的数据没有数据类型，总被视为字节数组`byte[]`。
- **时间戳**：每个单元格都保存着同一份数据的多个版本，这些版本采用时间戳进行索引。

&emsp;&emsp;下面用一个示例为阐释HBase的数据模型，下图为一张用来存储学生信息的HBase表：

<center><img src="https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.2.2.png" style="zoom:50%;" /></center>

&emsp;&emsp;学号作为行键来唯一标识每个学生，表中设计了列族`Info`来保存学生相关信息，列族`Info`中包含3个列——`name`、`major`和`email`，分别用来保存学生的姓名、专业和电子邮件信息。  
&emsp;&emsp;学号为`201505003`的学生存在两个版本的电子邮件地址，时间戳分别为`ts1=1174184619081`和`ts2=1174184620720`，时间戳较大的版本的数据是最新的数据。

### 4.2.3 数据坐标

&emsp;&emsp;HBase使用坐标来定位表中的数据，也就是说，每个值都通过坐标来访问。HBase中需要根据行键、列族、列限定符和时间戳来确定一个单元格，因此，可以视为一个“四维坐标”，即 `[行键, 列族, 列限定符, 时间戳]`。  
&emsp;&emsp;如果把所有坐标看成一个整体，视为“键”，把四维坐标对应的单元格中的数据视为“值”，那么，HBase可以看成一个键值数据库。

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.2.3.png)

### 4.2.4 概念视图

&emsp;&emsp;在HBase的概念视图中，一个表可以视为一个稀疏、多维的映射关系。下表是一个存储网页的HBase表的片段：

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.2.4.png)

- 行键是一个反向`URL`，由于HBase是按照行键的字典序来排序存储数据的，采用这种方式可以让来自同一个网站的数据内容都存在相邻的位置，在按照行键的值进行水平分区时，就可以尽量把来自同一个网站的数据分到同一个分区（Region）中。
- 列族`content`用来存储网页内容。
- 列族`anchor`包含了任何引用这个页面的锚链接文本。
- 时间戳代表不同时间的版本

&emsp;&emsp;可以用“四维坐标”定位单元格中的数据，比如`["com.cnn.www","anchor","cnnsi.com","t5"]`对应的单元格里存储的数据为`CNN`。  
&emsp;&emsp;可以看出，在一个HBase表的概念视图中，每个行都包含相同的列族，尽管行不需要在每个列族里存储数据。从这个角度来说，HBase表是一个稀疏的映射关系，即里面存在很多空的单元格。

### 4.2.5 物理视图

&emsp;&emsp;从概念视图层面，HBase中的每个表是许多行组成的，但是在物理存储层面，它采用基于列的存储方式，而不是像传统关系数据库那样采用基于行的存储方式，这也是HBase和传统关系数据库的重要区别。  
&emsp;&emsp;上一节的概念视图在物理存储的时候，会存成下图的两个小片段。也就是说，这个HBase表会按照`contents`和`anchor`这两个列族分别存放，属于同一个列族的数据保存在一起，同时，和每个列族一起存放的还包括行键和时间戳。  

> **注意：**在概念视图中，我们可以发现，有些列是空的。但是在物理视图中， 这些空的列不会被存储。如果请求这些空白的单元格的时候，会返回`null`值。

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.2.5.png)

### 4.2.6 面向列的存储

&emsp;&emsp;我们通过一个简单例子，看看列式存储与行式存储方式的具体差别。

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.2.6.png)

1. **传统行式数据库**  
- 数据是按行存储的
- 没有索引的查询使用大量I/O。在从磁盘中读取数据时，需要从磁盘中顺序扫描每个元组的完整内容，然后从每个元组中筛选出查询所需要的属性
- 建立索引和物理视图需要花费大量时间和资源
- 面对查询的需求，数据库必须被大量膨胀才能满足性能要求

2. **列式数据库**  
- 数据按列存储，每一列单独存放
- 数据即是索引
- 只访问查询涉及的列，大量降低系统IO
- 每一列由一个线索来处理，查询采用并发处理方式
- 数据类型一致，数据特征相似，采用高效压缩方式
- 缺陷：执行链接操作时，需要昂贵的元组重构代价

> ps：乍一看HBase这玩意是挺好的，但是他内部是如何实现工作的呢，且听下面分说

<center><img src="https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.2_emoji.png" style="zoom:80%;" /></center>

## 4.3 HBase的实现原理

&emsp;&emsp;本节从三个角度介绍HBase的实现原理：HBase功能组件、表和Region、Region的定位。

### 4.3.1 HBase功能组件

&emsp;&emsp;**HBase的实现包括三个主要的功能组件** ： 库函数（用于连接到每个客户端）、一个Master主服务器、许多个Region服务器。

其中：
- 主服务器Master：负责管理和维护HBase表的分区信息，维护Region服务器列表，分配Region，负载均衡。
- Region服务器：负责存储和维护分配给自己的Region，处理来自客户端的读写请求。
- 客户端并不是直接从Master主服务器上读取数据，而是在获得Region的存储位置信息后，直接从Region服务器上读取数据。  
- 客户端并不依赖Master，而是通过Zookeeper获得Region位置信息，大多数客户端甚至从来不和Master通信，这种设计方式使得Master负载很小 。

### 4.3.2 表和Region

&emsp;&emsp;在一个HBase中，存储了很多的表。对于每个HBase表而言，表中的行是根据行键的值的字典序进行维护的，表中包含的行的数量可能非常庞大，无法存储在一台机器上，需要分布存储到多台机器上。因此，需要根据行键的值对表中的行进行分区。每个行区间构成一个分区，被称为Region。Region包含了位于某个值域区间内的所有数据，是负载均衡和数据分发的基本单位。

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.3.2_1.png)

**生成Region的过程：**
1. 开始只有一个Region，随着数据不断的插入，Region越来越大。当达到一定的阈值时，开始分裂成两个新的Region
2. 随着表中行的数量继续增加，就会分裂出越来越多的Region
3. Region拆分操作非常快，因为拆分之后的Region读取的仍然是原存储文件，直到“合并”过程把存储文件异步地写到独立的文件之后，才会读取新文件

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.3.2_2.png)

其中：
- 每个Region默认大小是100MB到200MB（2006年以前的硬件配置），每个Region的最佳大小取决于单台服务器的有效处理能力，目前每个Region最佳大小建议1GB-2GB（2013年以后的硬件配置)
- 同一个Region不会被分拆到多个Region服务器
- 每个Region服务器负责管理一个Region集合，通常在每个Region服务器会存储10-1000个Region

### 4.3.3 Region定位

&emsp;&emsp;一个HBase的表可能非常庞大，会被分裂成很多个Region，这些Region可被分发到不同Region服务器上。因此，本小节将会介绍Region的定位机制。  
&emsp;&emsp;每个Region都有一个`RegionID`来标识它的唯一性，这样，一个Region标识符就可以表示成`表名＋开始主键+RegionID`。  
&emsp;&emsp;为了定位每个Region所在的位置，可以构建一张映射表，每行包含两项内容（表示Region和Region服务器之间的对应关系，从而就可以知道某个Region被保存在哪个Region服务器中）：**Region标识符**、**Region服务器标识**。
&emsp;&emsp;这个映射表包含了关于Region的元数据（即Region和Region服务器之间的对应关系）。因此，也被称为“元数据表”，又名`.META.`表。  
&emsp;&emsp;当一个HBase表中的Region数量非常庞大的时候，`.META.`表的条目就会非常多 ，一个服务器保存不下，也需要分区存储到不同的服务器上，因此，`.META.`表也会被分裂成多个Region。  
&emsp;&emsp;这时，为了定位这些Region，就需要再构建一个新的映射表，记录所有元数据的具体位置，这个新的映射表就是“根数据表”，又名`-ROOT-`表。该表的特点有：  

- `-ROOT-`表是不能被分割的，永远只存在一个Region用于存放`-ROOT-`表。
- 这个用来存放`-ROOT-`表的唯一个Region，它的名字是在程序中被写死的，Master主服务器永远知道它的位置。

&emsp;&emsp;综上所述，HBase使用类似B+树的三层结构来保存Region位置信息，详见下图：

<center><img src="https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.3.3.png" style="zoom:50%;" /></center>

&emsp;&emsp;三层结构的特点：
- 为了加快访问速度，`.META.`表的全部Region都会被保存在内存中
- 客户端访问数据时采用的是三级寻址
- 为了加速寻址，客户端会缓存位置信息。同时，需要解决缓存失效问题
- 寻址过程客户端只需要询问Zookeeper服务器，不需要连接Master服务器。因此，主服务器的负载相对就小了很多。

## 4.4 HBase运行机制

&emsp;&emsp;在本小节中，我们将介绍HBase运行机制，包括HBase系统架构以及Region服务器、`Store`和`HLog`的工作原理。

### 4.4.1 HBase系统架构

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.4.1.png)

- **客户端**：客户端包含访问HBase的接口，同时在缓存中维护着已经访问过的`Region`位置信息，用来加快后续数据访问过程。
- **Zookeeper服务器**：Zookeeper可以帮助选举出一个Master作为集群的总管，并保证在任何时刻总有唯一一个Master在运行，这就避免了Master的“单点失效”问题。同时，Zookeeper也是一个很好的集群管理工具，被大量用于分布式计算，提供配置维护、域名服务、分布式同步、组服务等。

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.4.1_zookeeper.png)

- **Master服务器**：主服务器Master主要负责表和Region的管理工作：
  - 管理用户对表的增加、删除、修改、查询等操作
  - 实现不同Region服务器之间的负载均衡
  - 在Region分裂或合并后，负责重新调整Region的分布
  - 对发生故障失效的Region服务器上的Region进行迁移

- **Region服务器**：Region服务器是HBase中最核心的模块，负责维护分配给自己的Region，并响应用户的读写请求。

### 4.4.2 Region服务器的工作原理

&emsp;&emsp;Region服务器的工作原理分为三个阶段：用户读写数据过程、 缓存的刷新、`StoreFile`的合并。

1. 用户读写数据过程 
    - 用户写入数据时，被分配到相应Region服务器去执行
    - 用户数据首先被写入到`MemStore`和`Hlog`中
    - 只有当操作写入`Hlog`之后，调用`commit()` 方法才会将其返回给客户端
    - 当用户读取数据时， Region服务器会首先访问`MemStore`缓存，如果找不到，再到磁盘的`StoreFile`中寻找

2. 缓存的刷新
    - 系统会周期性地把`MemStore`缓存里的内容刷写到磁盘的`StoreFile`文件中，清空缓存，并在`Hlog`里面写入一个标记
    - 每次刷写都生成一个新的`StoreFile`文件，因此，每个`Store`包含多个`StoreFile`文件
    - 每个Region服务器都有一个自己的`HLog`文件，每次启动都检查该文件，确认最近一次执行缓存刷新操作之后是否发生新的写入操作；如果发现更新，则先写入`MemStore`，再刷写到`StoreFile`，最后删除旧的`Hlog`文件，开始为用户提供服务

3. `StoreFile`的合并
    - 每次刷写都生成一个新的`StoreFile`，数量太多，影响查找速度
    - 调用`Store.compact()`把多个`StoreFile`合并成一个
    - 合并操作比较耗费资源，只有数量达到一定阈值后才会启动合并

### 4.4.3 Store工作原理

- `Store`是Region服务器的核心
- 多个`StoreFile`合并成一个`StoreFile` 
- 单个`StoreFile`过大时，又触发分裂操作，1个父Region被分裂成两个子Region 

&emsp;&emsp;`StoreFile`的合并和分裂过程见下图：

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.4.3.png)

### 4.4.4 HLog工作原理

&emsp;&emsp;在分布式环境下，必须考虑到系统出错的情形，比如当Region服务器发生故障时，`MemStore`缓存中的数据（还没有写入文件）会全部丢失。因此，HBase用`HLog`来保证系统发生故障时能够恢复到正确的状态，`HLog`具有以下特点：  

- HBase系统为每个Region服务器配置了一个`HLog`文件，它是一种预写式日志（Write Ahead Log）。
- 用户更新数据必须首先写入日志后，才能写入`MemStore`缓存，并且，直到`MemStore`缓存内容对应的日志已经写入磁盘后，该缓存内容才能被刷写到磁盘。
- Zookeeper会实时监测每个Region服务器的状态，当某个Region服务器发生故障时，Zookeeper会通知Master。
- Master首先会处理该故障Region服务器上面遗留的`HLog`文件，这个遗留的`HLog`文件中包含了来自多个Region对象的日志记录。
- 系统会根据每条日志记录所属的Region对象对`HLog`数据进行拆分，分别放到相应Region对象的目录下，然后，再将失效的Region重新分配到可用的Region服务器中，并把与该Region对象相关的`HLog`日志记录也发送给相应的Region服务器。
- Region服务器领取到分配给自己的Region对象以及与之相关的`HLog`日志记录以后，会重新做一遍日志记录中的各种操作，把日志记录中的数据写入到`MemStore`缓存中，然后，刷新到磁盘的`StoreFile`文件中，完成数据恢复。
- 共用日志的优点是提高对表的写操作性能；其缺点是恢复时需要分拆日志。

### 4.4.5 HBase性能优化

- **行键（Row Key）**： 行键是按照字典序存储。因此，设计行键时，要充分利用这个排序特点，将经常一起读取的数据存储到一块，将最近可能会被访问的数据放在一块。例如：如果最近写入HBase表中的数据是最可能被访问的，可以考虑将时间戳作为行键的一部分 。
- **InMemory**：创建表的时候，可将表放到Region服务器的缓存中，保证在读取的时候被`cache`命中。
- **Max Version**：创建表的时候，可设置表中数据的最大版本，如果只需要保存最新版本的数据，那么可以设置`setMaxVersions(1)`。
- **Time To Live**：创建表的时候，可设置表中数据的存储生命期，过期数据将自动被删除。

## 4.5 HBase 编程实战

### 4.5.1 实验一：HBase的安装部署和使用

#### 4.5.1.1 实验准备

**实验环境：**Linux Ubuntu 22.04     
**前提条件：**  
1. 完成Java运行环境部署（详见第2章Java安装）
2. 完成Hadoop 3.3.1的单点部署（详见第2章安装单机版Hadoop）

#### 4.5.1.2 实验内容

&emsp;&emsp;基于上述前提条件，安装HBase和学习HBase Shell的使用。

#### 4.5.1.3 实验步骤

##### 1.解压安装包

&emsp;&emsp;通过官网下载地址（✅**官网下载地址**：[HBase下载](https://hbase.apache.org/downloads.html)），下载hbase 2.3.5的安装包到本地指定目录，如/data/hadoop/下。运行下面的命令，解压安装包至`/opt`目录下：
```shell
sudo tar -zxvf /data/hadoop/hbase-2.4.8-bin.tar.gz -C /opt/
```

##### 2.更改文件夹名和所属用户

&emsp;&emsp;安装包解压成功后，在`/opt`目录下将会产生`hbase-2.4.8`目录，如下图所示：

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.1.png)

&emsp;&emsp;将`hbase-2.4.8`目录更名为`hbase`，命令如下：
```shell
sudo mv /opt/hbase-2.4.8/ /opt/hbase
```
&emsp;&emsp;改变hbase目录所属用户为`datawhale`，用户组为`datawhale`，命令如下：
```shell
sudo chown -R datawhale:datawhale /opt/hbase/
```

##### 3.设置HBASE_HOME环境变量

&emsp;&emsp;将`HBASE_HOME`环境变量设置为`/opt/hbase`，作为工作目录，打开系统环境变量配置文件，命令如下：
```shell
sudo vim /etc/profile
```

&emsp;&emsp;在文件末尾，添加如下内容：
```shell
# hbase
export HBASE_HOME=/opt/hbase
export PATH=$PATH:$HBASE_HOME/bin
```

&emsp;&emsp;使用`Shift+:`，输入`wq`后回车，保存退出。运行下面命令使环境变量生效：
```shell
source /etc/profile
```

##### 4.修改hbase-site.xml配置文件

&emsp;&emsp;打开`hbase-site.xml`文件，命令如下：
```shell
sudo vim /opt/hbase/conf/hbase-site.xml
```

&emsp;&emsp;添加下面配置到`<configuration>与</configuration>`标签之间，添加内容如下：
```html
<configuration>
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/opt/hadoop/zookeeper</value>
    </property>
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://master:9000/hbase</value>
    </property>
    <property>
        <name>hbase.unsafe.stream.capability.enforce</name>
        <value>false</value>
    </property>
</configuration>
```
&emsp;&emsp;使用`Shift+:`，输入`wq`后回车，保存并退出。

##### 5.修改hbase-env.sh配置文件

&emsp;&emsp;打开`hbase-env.sh`文件，命令如下：
```shell
sudo vim /opt/hbase/conf/hbase-env.sh
```

&emsp;&emsp;配置`JAVA_HOME`，`HBASE_CLASSPATH`，`HBASE_MANAGES_ZK`。`hbase-env.sh`中本来就存在这些变量的配置，大家只需要删除前面的`#`并修改配置内容即可(`#`代表注释)，或者直接在文件中增加以下内容：
```shell
export JAVA_HOME=/opt/java/ # JDK的安装目录
export HBASE_CLASSPATH=/opt/hbase/conf # 本机HBase安装目录下的conf目录
export HBASE_MANAGES_ZK=true # 由HBase自己管理zookeeper，不需要单独的zookeeper
```
&emsp;&emsp;使用`Shift+:`，输入`wq`后回车，保存并退出。

##### 6.启动hadoop

&emsp;&emsp;进入`hadoop`目录下的`sbin`目录，命令如下：
```shell
cd /opt/hadoop/sbin/
```

&emsp;&emsp;启动Hadoop集群，命令如下：
```shell
./start-all.sh
```

&emsp;&emsp;执行`jps`命令检查`hadoop`是否启动成功，出现了6个进程，则表示正常启动，可以得到如下类似结果：
```log
2261 Jps
1317 DataNode
2086 NodeManager
1788 ResourceManager
1550 SecondaryNameNode
1199 NameNode
```

##### 7.启动HBase

&emsp;&emsp;启动HBase，命令如下：
```shell
cd /opt/hadoop/sbin/
start-hbase.sh
```

&emsp;&emsp;执行`jps`命令检查HBase是否启动成功，新增了3个进程（`HQuorumPeer`、`HMaster`和`HRegionServer`），则表示正常启动，可得到如下类似结果：

```shell
2261 Jps
1317 DataNode
2086 NodeManager
1788 ResourceManager
1550 SecondaryNameNode
1199 NameNode
2215 HQuorumPeer
2284 HMaster
2444 HRegionServer
```

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.2.png)

**Tips：**

- 如果HMaster启动后瞬间消失，请查看`/opt/hbase/logs`日志文件。  一般为 hadoop 的 core-site.xml 和 hbase的hbase-site.xml 中的hdfs的路径不匹配，修改一致即可
- 如果出现`connection failed`，注意虚拟机环回IP的通信问题以及防火墙是否关闭！！！  

1. 关闭防火墙  
⚠ 如果熟悉Linux操作，可以不用关闭防火墙，只需要开放9000端口就行。但是为了避免这类问题，建议直接关闭防火墙，命令如下：  
```shell
systemctl disable firewalld.service
```

2. 修改`/etc/hosts`文件  
    &emsp;&emsp;打开`/etc/hosts`文件，命令如下：  
    ```shell
    sudo vim /etc/hosts
    ```

    &emsp;&emsp;在文件中添加以下内容：  
    ```
    127.0.1.1 虚拟机的名称
    ```
    &emsp;&emsp;使用`Shift+:`，输入`wq`后回车，保存并退出。  

3. 修改`/opt/hbase/conf/regionservers`文件  
    &emsp;&emsp;打开`/opt/hbase/conf/regionservers`文件，命令如下：  
    ```shell
    vim /opt/hbase/conf/regionservers
    ```

    &emsp;&emsp;在文件中添加以下内容：  
    ```shell
    <虚拟机的名称>
    ```

✅**官方HBase安装指南**：[HBase伪集群分布安装](https://hbase.apache.org/book.html#quickstart_pseudo)

✅**林子雨HBase2.2版本安装指南**：[HBase2.2安装](http://dblab.xmu.edu.cn/blog/2442-2/)

> ps：代码敲多了，感觉女人都没意思了哈哈哈，手动狗头QAQ  
> 谈恋爱哪有学习敲代码香啊，给我继续敲！！代码使我变强，对象使我牵挂  
> 不慌，继续学！！  

<center><img src="https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.2_emoji.png" style="zoom:80%;" /></center>

##### 8.启动HBase Shell

&emsp;&emsp;启动`HBase Shell`，命令如下：  
```shell
cd /opt/hbase/bin
hbase shell
```

&emsp;&emsp;启动成功后，进入HBase命令行模式，显示如下：  
```log
datawhale@localhost:~$ hbase shell
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/hbase/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
HBase Shell; enter 'help' for list of supported commands.
Type "exit" to leave the HBase Shell
Version 1.2.6, rUnknown, Mon May 29 02:25:32 CDT 2017
 
hbase(main):001:0>
```

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.3.png)

##### 9.创建表（`create`命令）

&emsp;&emsp;在hbase的命令行模式下，创建一个`student`表，`info`和`addr`为该表的两个列族，创建语句如下：  
```sql
create 'student','info','addr'
```

&emsp;&emsp;执行语句后，执行结果如下：  
```log
hbase(main):009:0> create 'student','info','addr'
0 row(s) in 2.2840 seconds

=> Hbase::Table - student
```

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.4.png)

- 有时会报错ServerNotRunningYetException: Server is not running yet，解决办法：


第一步：先看当前用户是否对hbase有权限，一般使用hbase目录的所有者操作hbase

```
sudo chmod 777 /opt/hbase/*
```

第二步：看hadoop是否处于安全模式，若处于则关闭安全模式

```
hdfs dfsadmin -safemode get     #查看是否处于安全模式

hdfs dfsadmin -safemode leave   #关闭安全模式
```

如果还无法解决，则是`slf4j-log4j12-1.7.30.jar`在Hadoop和hbase都存在 同时启动出现占用情况导致服务无法访问的问题

```
cd /opt/hbase/lib/client-facing-thirdparty/
ls
rm slf4j-log4j12-1.7.30.jar
```

然后再hbase的 `hbase-env.sh`中将 `export HBASE_DISABLE_HADOOP_CLASSPATH_LOOKUP="true"` 注释去掉 可以把 引号也删除，若配置文件没有这个信息则直接插入即可，该配置表示，启动时告诉HBase是否应该包含Hadoop的lib， 默认值为false，表示包含Hadoop的lib。



- 有时会报错Master is initializing，出现该问题的原因是因为重新安装hbase时，旧的元数据信息没有删除，使用zookeeper删除后hbase元数据后重启hbase即可。解决方法：

```
cd /opt/hbase/bin

hbase zkcli      #启动zookeeper
```

查看目录 / ,可以看到hbase

```
ls /
```

再查看/hbase目录中的内容，可发现有mete-region-server文件

```
ls /hbase
```

执行删除命令（rmr命令无效的话使用delete命令）

```
rmr /hbase/meta-region-server
```

quit退出后，关闭hbase

```
hbase-daemons.sh stop master
stop-hbase.sh
```

重启hbase后问题解决。

##### 10.添加数据（`put`命令）

&emsp;&emsp;HBase中的`put`命令，用于向表中添加数据，下面我们向`student`表中添加数据，分别执行以下语句：  
```sql
put 'student','1','info:name','zeno'
put 'student','1','info:age','22'
put 'student','1','addr:city','hefei'
put 'student','2','info:sex','man'
```

&emsp;&emsp;执行结果如下：  
![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.5.png)

##### 11.查看表内容（`scan`命令）

&emsp;&emsp;HBase中的`scan`命令，用于扫描表内容，下面我们看看`student`表有哪些数据，执行以下语句：
```sql
scan 'student'
```

&emsp;&emsp;执行结果如下：  
```log
hbase(main):014:0> scan 'student'
ROW                   COLUMN+CELL
1                    column=addr:city, timestamp=1531207679298, value=hefei
1                    column=info:age, timestamp=1531207651174, value=22
1                    column=info:name, timestamp=1531207642229, value=zeno
2                    column=info:sex, timestamp=1531207752067, value=man
2 row(s) in 0.0200 seconds
```

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.6.png)

##### 12.查询数据（`get`命令）

&emsp;&emsp;HBase中的`get`命令用于查询数据，下面我们查询一下`student`表中`rowkey`为1的一条数据，执行以下语句：
```sql
get 'student','1'
```

&emsp;&emsp;执行结果如下：  
```log
hbase(main):015:0> get 'student','1'
COLUMN                CELL
addr:city            timestamp=1531207679298, value=hefei
info:age             timestamp=1531207651174, value=22
info:name            timestamp=1531207642229, value=zeno
3 row(s) in 0.0480 seconds
```

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.7.png)

##### 13.修改内容（`put`命令）

&emsp;&emsp;严格来说，在HBase中没有修改数据的概念，只有覆盖数据，也是使用`put`命令，我们先插入一条数据，执行语句如下：  
```sql
put 'student','1','info:age','18'
```

&emsp;&emsp;执行结果如下：  
```log
hbase(main):016:0> put 'student','1','info:age','18'
0 row(s) in 0.0170 seconds
```

&emsp;&emsp;再使用`get`命令查询一下修改结果：
```log
hbase(main):017:0> get 'student','1'
COLUMN                CELL
addr:city            timestamp=1531207679298, value=hefei
info:age             timestamp=1531207651183, value=18
info:name            timestamp=1531207642229, value=zeno
3 row(s) in 0.0520 seconds
```

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.8.png)

##### 14.添加列族（`alter`命令）

&emsp;&emsp;使用`alter`命令添加`nation`列族（**这里可以指定NAME => 'nation' or 'NAME' => 'nation'**），语句如下：  
```sql
alter 'student', 'nation'
```

&emsp;&emsp;查看`student`表信息，语句如下：  
```sql
describe 'student'
```

&emsp;&emsp;执行结果如下：  

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.9.png)

##### 15.删除列族（`alter`命令）

&emsp;&emsp;使用`alter`命令删除`nation`列族，并使用`describe`命令查看`student`表信息，语句如下：  
```sql
alter 'student', NAME => 'nation', METHOD => 'delete'
```

&emsp;&emsp;执行结果如下：

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.10.png)

##### 16.删除表（`disable`命令）

&emsp;&emsp;HBase中的表不能直接删除，需要禁用（`disable`命令）后，才能删除（`drop`命令），下面我们删除`student`表，首先禁用`student`表，语句如下：
```sql
disable 'student'
```
&emsp;&emsp;执行结果如下：  
```log
hbase(main):018:0> disable 'student'
0 row(s) in 2.2950 seconds
```

&emsp;&emsp;接着，删除`student`表，语句如下：  
```sql
drop 'student'
```
&emsp;&emsp;执行结果如下：
```log
hbase(main):019:0> drop 'student'
0 row(s) in 2.2770 seconds
```

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex1.11.png)

### 4.5.2 实验二：常用的HBase操作

#### 4.5.2.1 实验准备

**实验环境：**Linux Ubuntu 22.04   
**前提条件：**  

1. 完成Java运行环境部署（详见第2章Java安装）
2. 完成Hadoop 3.3.1的单点部署（详见第2章安装单机版Hadoop）

#### 4.5.2.2 实验内容

&emsp;&emsp;基于上述前提条件， 使用`HBase Shell`命令完成以下指定功能：
1. 将关系型数据库中的表和数据，转换为适合于HBase存储的表并插入数据
2. 使用`list`命令，列出HBase所有的表的相关信息，如表名、创建时间等
3. 使用`scan`命令，打印`Course`表的所有记录数据（按照截图，创建`Course`表及列族，并添加数据）
4. 使用`alter`命令，向已经创建好的表添加和删除指定的列族或列
5. 使用`truncate`命令，清空指定的表的所有记录数据
6. 使用`count`命令，统计表的行数

#### 4.5.2.3 实验步骤

##### 1.现有以下关系型数据库中的表和数据，要求将其转换为适合于HBase存储的表并插入数据

> ⚠：请根据给定的表，自行完成插入数据的操作，不提供代码。

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.2_stu.png)

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.2_stu_answer.png)

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.2_course.png)

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.2_course_answer.png)

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.2_SC.png)

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.2_SC_answer.png)


##### 2.编程实现以下指定功能，并用Hadoop提供的HBase Shell命令完成任务

1. 使用`list`命令，列出HBase所有的表的相关信息，如表名、创建时间等

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.1.1.png)

2. 使用`scan`命令，打印`Course`表的所有记录数据

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.1.2.png)

3. 向已经创建好的表添加和删除指定的列族或列  
- 使用`alter`命令，添加列族

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.1.3_add.png)

- 使用`alter '表名','列族',METHOD=>'delete'`语句，删除列族 

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.1.3_del.png)

4. 使用`truncate`命令，清空指定的表的所有记录数据

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.1.4.png)

5. 使用`count`命令，统计表的行数

![](https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4_ex2.1.5.png)

## 4.6 本章小结

&emsp;&emsp;在本章的学习中，主要介绍了HBase的由来及其和传统数据库的区别，其次讲解了HBase访问接口、数据模型、实现原理和运行机制，并在最后基于`HBase Shell`命令，学习HBase编程实践方面的知识。  
&emsp;&emsp;在下一章中，我们将学习到分布式并行编程模型MapReduce， 💥💥💥 **硬核+高能预警**！！💥💥💥

> ps：MapReduce，Hadoop生态系统的又一大基石，兄弟们，上高地的时候到了，准备冲冲冲

<center><img src="https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.6.1.png" style="zoom:80%;" /></center>

<center><img src="https://cdn.jsdelivr.net/gh/shenhao-stu/Big-Data/doc_imgs/ch4.6.2.png" style="zoom:80%;" /></center>
