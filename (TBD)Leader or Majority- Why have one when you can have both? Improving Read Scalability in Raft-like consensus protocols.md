说实话这篇论文没看懂。。。。。以后再看

# Abstract

所有的读写都通过leader来保证强一致性（线性一致性）。因为Raft算法保证复制到quorum，那为什么不利用这个特点来形成不需要将读通过Leader的强一致性读呢？

# 1 Introduction

因此，将quorum read和leader read结合起来可以提高Raft的读扩展性。

第二节提供了共识算法的背景。

第三节我们提供了CockroachDB的Raft实现以及一致性读的实现细节。

第四节讨论了Raft中通过quorum read来扩展读的设计

第五节衡量了结果。

第六节做了总结。

# 2 Background

# 3 CockroachDB

## 3.1 Overview

![image-20220311114842379](/Users/chenzhihao/Documents/chaosCode/I-read-paper/image/image-20220311114842379.png)

### Leaders and Leases.

lease：在lease的窗口时间内，leader被认为是不变的。减少了leader再次发动一轮心跳来证明自己确实是leader。lease可以在follower上。带有lease的follower可以进行本地的一致性读，但是必须将写提交给leader。

要注意到这个lease和raft优化中的read lease大部分是相似的，只不过raft的read lease只针对读，raft read lease是不会和写接触的。而这里的lease会将写转发给leader处理，相当于转发给leader一条 entry。对于读请求，lease holder也需要和leader进行rpc通信，用于获取最新的commitIndex。

### Processing read/write requests.

## 3.2 Bottlenecks to read performance

单点上的写请求带来整个系统性能上的降低。

# 4 Designing for Read Scalability

两种读取的变体。第一种就是简单的quorum read，但是返回的value可能不是线性的。第二种返回的一定是强一致读，但是开销会大一点。

## 4.1 Quorum Read Approaches

### Quorum Reads

读取quorum的最新committed value，但是这个value可能不是线性的，因为leader commit value要比follower快。

### Strongly Consistent Quorum Reads

timestamp：任何对某个key的更新都会被持久化为一个write intent，intent包含提议的时间戳、value、关于事务的元信息。当事务完成时intents才会被清空。

如果节点收到一个强一致性quorum read，并且这个read是对于一个正在更新的key，那么节点只会返回key的timestmap，不会返回数据。只返回ts不返回数据说明对应的key正在write intent。这样以来，强一致性quorum read至少可以得知最新的正在commit的timestamp是多少。并且只要这个timestamp对应的value完成提交，读取这个value就是线性一致的。gateway便用这个timestamp去读取数据，如果存在，那么读到的数据就是线性的。如果读不到timestamp，那么说明系统中还有待处理的更新，那么本次请求应当视为失败。一个备用方法是重试请求。

## 4.2 Combining Lease-holder and Quorum Reads

* 从lease holder中读
* 强一致quorum读（不包含lease holder），注意holder不一定是leader。

# 5 Evaluation

# 6 Conclusion

# 7 Discussion Topics

随着写的比例增加，基于quorum的收益减少。并且，随着数据竞争和写比例增加，强一致quorum read可能导致更多的重试，quorum read更加可能返回一个落后于leader的commit value。