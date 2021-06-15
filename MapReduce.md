<center>MapReduce: Simplified Data Processing on Large Clusters </center>
<center>Jeffrey Dean and Sanjay Ghemawat</center>
<center>jeff@google.com, sanjay@google.com</center>
<center>Google, Inc.</center> 



<center>Abstract</center>
MapReduce是一种编程模型，是用于处理和生成大数据集的相关实现。用户自定义map函数来处理key/value对，并生成中间key/value的集合。用户自定义reduce函数来合并所有中间key相同的value。

使用这种函数形式写出来的程序是天然并行的，并且可以在大规模集群中运行。实时系统负责处理输入数据划分的细节，在机器集合中调度程序的运行，处理机器故障，并且管理机器之间的通信。MapReduce的出现使得没有并行和分布式系统经验的码农可以轻易的使用大规模分布式系统的资源。

一个典型的MapReduce计算可以在几千台机器上处理TB（10^9^B）级别的数据。

## 1 Introduction

Google设计了新的抽象概念来允许我们来简单的表达计算，并且将关于并行、容错、数据分布和负载平衡放在库中。

Section2描述了MapReduce的基本编程模型并且给出了几个例子。Section3描述了基于我们的集群计算环境下MapReduce的接口的实现。section4描述了我们发现的几种有用的编程模型的改进。section5测量了我们对于几种任务的实现的性能。section6探索了Google对于使用MapReduce作为基础来重写索引系统的经验。section7讨论了相关的以及未来的工作。

## 2 Programming Model

Map接受一个input pair，产生中间的key/value对的集合。MapReduce库按中间对的key进行分组，将中间对的value集合传递给Reduce函数。

Reduce函数接受中间对的key和这个key对应的value集合。它将这些value合并成更小的value集合。一般情况下每次Reduce调用只有0或者1个value输出。中间对的value通过iterator传递到Reduce函数，这让我们可以处理比内存空间更大的value集合。

### 2.1 Example

 考虑word count问题。用户可以写出下面的伪代码。

```c++
map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
    	EmitIntermediate(w, "1");

reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result += ParseInt(v);
    Emit(AsString(result));
```

## 3 Implementation

### 3.1 Execution Overview

Map调用是通过在多台机器上自动将输入数据划分成M个splits。这些splits可以并行的被不同机器处理。Reduce调用使用哈希函数（hash（key）mod R）将中间key划分成R片。

图1展示了MapReduce操作的全部流。当用户程序调用MapReduce函数时，下面的动作序列就会发生（图1中的标签数字正好对应下面列表的数字）：

![1623721189730](../image/1623721189730.png)

1. 用户程序中的MapReduce库首先将输入文件分成M片，每一片大约16MB-64MB（用户可以通过一个可选参数控制片的大小）。然后在集群的机器上都开始执行相同的程序。
2. 其中一份程序的拷贝是特殊的——master。剩下的是被master分配的workers。总共需要分配M个map tasks和R个reduce tasks。master选择空闲的worker和一个map task或者一个reduce task。
3. 被分配到map task的worker从输入片中读取对应的内容。它从输入数据中解析出key/value对，并且将每一对都传递给用户定义的Map函数。Map函数产生的中间key/value对被缓存在内存中。
4. 内存中的中间pair被周期性的写到本地磁盘中，pair通过划分函数被划分成R片区域。这些在本地磁盘中缓存的pair的地址被传递到master，master负责将这些地址传输到reduce workers。
5. 当reduce worker被master通知到这些地址时，它使用远程进程调用来读取map workers在本地磁盘中写下的缓存数据。当reduce worker读取了全部的中间数据，它会排序中间keys以便相同的key可以分在同一组。排序是必要的，因为很多不同keys的会映射到相同的reduce task。如果中间数据的数量大到不能在内存中放下，就需要外部排序了。
6. reduce workers在排序好的中间数据中迭代，遇到每一个独特的中间key，它将key和对应的value传递给用户定义的Reduce函数。Reduce函数的输出被追加在这个reduce partition的最后输出文件中。
7. 但所有的map task和reduce task完成后，master唤醒用户程序。这时，MapReduce返回用户代码。

### 3.3 Fault Tolerance

#### Worker Failure 

master周期性的ping每一个worker。如果在一段时间内没有收到worker的回应，master就将该worker标记成failed。任何一个该worker完成的map task都会被重置成idle状态，因此可以被调度到其他workers上去。相似的，任何一个运行中的map task或reduce task失败了都会被重置为idle并且有资格被重新调度。

已经完成的map tasks在失败后需要被重新执行是因为它们的输出是存储在失败机器的本地磁盘上的，因此没办法访问。完成的reduce tasks不需要重新执行因为它们的输出存储在GFS。

当一个map task先被workerA执行再被workerB执行（因为A出错了），所有执行相关reduce task的workers都被通知重新执行。任何一个没有从workerA中读取数据的reduce task都会从workerB中读取数据。

####  Master Failure

让master周期性的将master数据结构写入checkpoints是简单的。如果master task死了，一个新的拷贝master task就会从上一个checkpoint开始。然而，注意到这里只有一个master，它失败可能性比较小；因此当前我们的实现是master出错就abort MapReduce计算。

###  3.4 Locality

网络带宽在我们的计算环境中是相对稀缺的资源。我们通过将数据（被GFS管理）存储在集群机器中的本地磁盘上来节约网络带宽。GFS将每一个文件划分成64MB的块，并且存储块的拷贝（一般来说是3个拷贝）在不同的机器上。如果在某个副本上的map task任务出错了，master会尝试将map task调度到靠近出错副本的副本上去（例如和出错机器在相同交换机的工作机器上去）。所以在大部分集群的workers上运行大规模MapReduce操作时，绝大部分的输入数据的读取是本地的，并且不消耗网络带宽。

###  3.5 Task Granularity 

master必须使用O（M+R）来调度决定并且在内存中保存O（M*R）的状态。

###  3.6 Backup Tasks

MapReduce操作的总时间延长的一个普遍情况是出现了一个落伍者：一台机器花费了长时间来完成最后几个map或者reduce tasks。

当一个MapReduce操作接近完成时，master会调度剩下in-progress的tasks的备用执行。

##  4 Refinements 

###  4.3 Combiner Function 

combiner function和reduce function的唯一区别是MapReduce库如何处理function的输出。reduce function的输出被写到最后的输出文件中，combiner function的输出被写到中间文件中，这个中间文件会被发送到reduce task。

###  4.6 Skipping Bad Records 

我们提供了一种可选的执行模式，但MapReduce库检测到导致崩溃的记录时，跳过这些记录来前进。

在调用用户Map或Reduce操作时，MapReduce库会将参数的序号存储在全局变量中，如果用户代码产生信号，signal handler会发送一个带有序号的包给MapReduce master。当master看到超过一次的特殊record，就应该跳过该record。

