# The Google File System 

Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung 

Google∗ 

注：文本中的块=chunk。

## ABSTRACT

在这篇论文中，我们展示了文件系统被设计用于支持分布式应用的接口拓展， 讨论我们设计的许多方面，并且对micro-benchmarks和现实世界的使用做测量报告。

## General Terms

Design, reliability, performance, measurement

## Keywords

Fault tolerance, scalability, data storage, clustered storage 

## 1.INTRODUCTION 

1. 组件故障是常态而不是例外。
2. 按照传统标准，文件很大。
3. 绝大部分文件的改动是通过追加新数据而不是重写已有数据。
4. 通过协同设计应用程序和文件系统API，整个系统增加了灵活性。举个例子，我们放宽了GFS的一致性模型来简化文件系统，而不会给应用程序带来繁重的负担。我们也介绍了一种原子追加操作所以多个客户端可以并发向文件追加而不用应用程序进行额外的同步。

## 2.DESIGN OVERVIEW 

###  2.1 Assumptions 

* 系统由很多便宜的经常出错的组件构成。
* 系统存储着少量的大文件。我们预计有几百万个文件，每个文件通常是100MB或者更大。几个GB的文件是常见的，而且应该被有效管理。系统支持小文件，但是我们不需要为它们优化。
* 工作负载主要包含两种类型的read：大型流式read和小型随机read。
* 工作负载还包含很多大型顺序write追加数据到文件。
* 系统必须为多个客户端并发追加到相同文件提供高效的实现。
* 高持续带宽比低延时更加重要。

###  2.2 Interface 

GFS提供相似的文件系统接口，不提供标准API类似POSIX的实现。我们支持常见的操作如create、delete、open、close、read和write文件。

GFS提供snapshot（3.4节）和record append（3.3节）操作。snapshot以低成本创建了一个文件或者目录树的拷贝。record append允许多个客户端并发的向同一个文件中追加数据，并且保证每个客户端追加的原子性。这在实现多路合并结果和生产者消费者队列中很有用，客户端可以不用额外的锁来同时追加。

### 2.3 Architecture

 文件被划分成固定长度的chunks。每一个chunk都在创建时被master分配全局不变的64bit chunk handle。Chunkservers把chunks当作Linux文件存储在本地磁盘中。每一个chunk都被多个chunkservers复制。

master维护所有的文件系统元数据。包括命名空间、访问控制信息、从文件到chunk的映射和chunks的当前位置。它也控制系统范围内的活动就像chunk lease管理、孤儿chunks的垃圾回收、chunkservers之间的chunk迁移。master周期性的用HeartBeat信息和每一个chunkserver进行通信，发送指令和收集状态。

client和chunkserver都不缓存文件数据。client进行缓存几乎不能带来收益，因为绝大部分应用流规模较大，很难被缓存。不缓存文件数据简化了client和整个系统处理，避免缓存一致性带来的问题（当然clients需要缓存元数据）。chunkservers不需要缓存文件数据是因为chunks被当做本地文件来存储，因此Linux缓冲区已经在内存中缓存了频繁被访问的数据。

### 2.4 Single Master 

使用单master简化了我们的设计，使得master可以使用全局知识来完成复杂的chunk placement和replication decisions。当然，我们必须减少master在reads、writes中的参与，否则master会成为瓶颈。clients从不通过master read或者write文件数据。相反的，client询问master应该联系哪一个chunkserver。client会在一段时间内缓存这个信息，并且后续的操作都直接和chunkservers交互。

参考图1，让我们来解释一个简单的read做了哪些交互。首先因为使用的是固定长度的chunk，client将应用程序制定的文件名和字节偏移转换成文件内的块索引。然后client将文件名和块索引作为请求内容发送给master。master回复对应的chunk handle和副本的位置。client使用文件名和块索引作为key缓存了master的回复。 

client发送请求到其中一个replica，绝大多数情况下是最近的一个。该请求明确chunk handle和该chunk下的字节范围。之后的相同chunk的请求不再需要client-master交互，除非缓存信息失效或者文件被重新打开。

### 2.5 Chunk Size 

chunk size=64MB。Lazy space allocation可以避免由于内部碎片带来的空间浪费。

![1623763664644](../image/1623763664644.png)

###  2.6 Metadata

master存储三种主要的元数据类型：文件和块命名，从文件到块的映射，每一个块的副本们的位置。所有的元数据都保存在master的**内存**中。头两种类型也通过operation log记录改变，并被持久化在master的本地磁盘中，同时在remote machines上复制。master不会持久化的存储块位置，它会在启动时询问每一个chunkserver中的chunks，还有每一个chunkserver加入集群时也会询问。

#### 2.6.1 In-Memory Data Structures

 master在后台周期性的扫描整个状态，这样的周期扫描可以用于实现chunk垃圾回收、chunkserver出现故障时的re-replication、用于平衡负载的chunk迁移、跨chunservers的磁盘空间使用。4.3和4.4节会讨论这些活动。

master为每个64MB的chunk维护少于64B的元数据。

#### 2.6.2 ChunkLocations

master不需要持久化记录给定一个chunk，哪个chunkserver持有该chunk的副本。master在启动时通过和chunkservers的poll获取该信息。当chunkservers加入离开集群、改名、故障、重启时，减少了保持master和chunkservers之间同步的问题。

从另一个角度来理解这样的设计，我们要明白chunkserver对chunks是否在自己的磁盘中有最终话语权。因此在master上持久化保存这些信息没意义，chunkserver上发生的任何错误都会导致chunks消失。

#### 2.6.3 Operation Log

operation log包含了关键元数据改变的历史记录。这是GFS的核心。它不仅是元数据的唯一持久化记录，而且它作为定义并发操作的逻辑时间线服务。

我们将它复制到多台远程机器上并且只有当把对应的log record刷到本地磁盘和远程磁盘后才会回应客户操作。

master通过重新执行operation log来恢复文件系统状态。为了减少启动时间，我们必须保持log处于小的状态。当log增长超过一定大小master会checkpoint状态，因此master可以从最近的checkpoint开始重新执行operation log来恢复。

因为创建checkpoint需要一段时间，master的内部状态被构建成当一个新的checkpoint被创建时不会延迟incoming mutations。master切换到新的log文件并且用一个单独的线程创建checkpoint。新的checkpoint包含切换前的所有mutations。对于一个几百万个文件的集群来说创建新的checkpoint大概需要1分钟。当创建完成后，被写到本地和远端机器上。

### 2.7 Consistency Model

 GFS有一个宽松的一致性模型来支持我们的分布式应用。

#### 2.7.1 Guarantees by GFS 

文件命名空间的mutatoins是原子的。他们被master专门处理。

![1623849344183](../image/1623849344183.png)

defined是consistent的子集。

一个文件区域是consistent当所有客户端不管读到哪个replicas都能看到相同的数据。一个区域是defined的，当发生file data mutation后如果它还是consistent的并且所有客户端都能完整看见mutation。当mutation没有被并发的writers干扰，成功执行时，受影响的区域就是defined（同时也是consistent）。并发成功mutations保持区域undefined但是consistent：所有的客户端会看到相同的数据，但是不会反映出写了哪个mutation。通常相同的数据由来自多个mutations的混合片段组成。一个失败的mutation使得区域inconsistent（因此也是undefined）：不同的客户端可能在不同的时间点看到不同的数据。



## 3. SYSTEM INTERACTIONS 

我们在设计系统时减少master对所有operations的参与。在此背景下，我们现在描述client，master和chunkservers如何交互来实现数据改变，原子记录追加和快照。

### 3.1 Leases and Mutation Order 

![1623810321608](../image/1623810321608.png)

### 3.2 Data Flow 

### 3.3 Atomic Record Appends 

## 4. MASTER OPERATION

### 4.1 Namespace Management and Locking

read、write lock

### 4.2 Replica Placement

machine rack

### 4.3 Creation, Re-replication, Rebalancing  

### 4.4 Garbage Collection 

### 4.5 Stale Replica Detection  

 chunk version number  

## 5.FAULT TOLERANCE AND DIAGNOSIS 

### 5.1 High Availability 

### 5.2 Data Integrity 

 checksums

### 5.3 Diagnostic Tools 

![1623824038345](../image/1623824038345.png)