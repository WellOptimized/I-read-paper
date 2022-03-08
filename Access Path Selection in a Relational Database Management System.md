# Query Optimization

Selinger等人在System R上的基础论文将查询优化的问题分解成三个不同的子问题来实现实际的查询优化：成本估计、用于定义搜索空间的等价关系式？、基于成本的搜索。

优化器使用I/O以及CPU开销来估计查询的开销。为了计算这些具体开销，优化器依赖于预先计算好的关系式的开销，以及一组用于决定查询结果基数的启发式（比如启发式是基于某个估计出来的谓词选择性）。作为实践部分，需要仔细考虑这些启发式：什么时候它们是合理的？遇到什么输入它们是不合理的？如何改进它们？

通过使用这些成本估计，优化器使用了一种动态编程算法来为一个查询构建计划。优化器定义了一个物理运算符集合来执行一个给定的逻辑运算（使用完整的段和索引来查找一个元组）。使用这个集合，优化器迭代的构建一个关于运算符的left-deep树（被驱动表需要全表扫描，是存放在join buffer，所以越小越好；而驱动表可以用索引，所以可以是大表），树使用成本启发式来减少运行运算符所需的工作量，并且只需要关心上游消费者所要求的部分运算符计算顺序。这避免考虑所有可能的运算符顺序。

优化器得到的计划并不是数学上的最优结果，只是尽可能好的结果。

# 1. Introduction

第二节将描述优化器在SQL语句处理中的位置，第三节将描述存储部件，第四节将描述单个表查询中的开销公式，第五节讨论两个或者更多的表的join以及对应的开销。嵌套查询在第六节。

# 2. Processing of an SQL statement

SQL语句的处理分为四阶段：解析、优化、代码生成、执行。

Relations=数据库表。元组存储在页中，页被组织进段中。一个段可能有多个表，但是没有表是跨段的。两张不同的表中的元组可能出现在同一个页中。

# 3. The Research Storage System

# 4. Costs for single relation access paths

COST = PAGE FETCHES + W * (RSI CALLS).RSI CALLS是从RSS（Research Storage System）中返回的预测元组个数。

对于每一张表T：

- NCARD(T), the cardinality of relation T.

- TCARD(T), the number of pages in the segment that hold tuples of relation T.
- P(T), the fraction of data pages in the segment that hold tuples of relation T. P(T) = TCARD(T) / (no. of non-empty pages in the segment).

对于每一个张表上的每一个索引T：

- ICARD(I), number of distinct keys in index I.
- NINDX(I), the number of pages in index I.

这些数据都存放在catalog中，利用这些数据可以计算选择因子F。

选择因子F。选择因子可以粗糙的等价于符合谓词的元组占比。

论文给出了一张选择因子的表，里面是各种谓词的选择因子计算方式。从计算方式中可以看出，我们倾向于选择基数更大的列作为索引，这样可以将RSI CALLS？减少。

对于单表来说，最低访问路径通过计算每一条可行的访问路径的开销来获得（表中的每一个索引都可以算作一条路径，再加上全表扫描）。如果对元组的顺序有要求，那么最低访问路径开销就是min（按感兴趣顺序要求的最低路径开销，不按感兴趣顺序要求的最低路径+排序开销）。

单表访问路径的开销 COST FORMULAS：

# 5. Access path selection for joins

join的方法中只有两种方法是最优的以及接近最优的。

第一种是nested loops。外表中取一个元组，扫描内表。

第二种是merging scans。就跟merge sort差不多，将外表和内表按照join column order排序。

n路join中可以看成是多个2路join，第二个2路join不需要等待第一个2路join完成才开始。n路join中如果下一趟2路join没有要求，那么这一趟的结果是不会进行排序的。

from列表中有n个表的话，就有n！个join顺序。

join solution包括表的join顺序，每一个join的方法，一个表是如何被访问的计划。当然如果一个表在join前需要被sort，那么sort也要包括进计划。

构建搜索树。首先，访问每一个表最好的方法是以感兴趣的顺序和无序的方式。然后，join的最好方法是启发式。

被存储的solution最多有2**n种？。

## Computation of costs

C-outer(path1)扫描外表的开销，N扫描外表的到的元组个数

N = (product of the cardinalities of all relations T of the join so far) * (product of the selectivity factors of al1 applicable predicates).

C-nested-loop-join(pathl,path2)= C-outer(path1) + N * C-inner(path2)

C-merge(pathl,path2)= C-outer(path1) + N * C-inner(path2)

如果内表被排序进一个临时表的话，C-inner(sorted list) = TEMPPAGES/N + W*RSICARD，这个式子假设在merge的过程中，内部表的每一页只取一次，所以除数为N。

然后就发现了嵌套循环和merge的开销是相同的，有时候merge开销更少是因为在排序之后，内部表的页可以少取几次。

C-sort(path)包括将数据按特定的访问路径取出来，排序，并把结果插入到一个临时表中。

## Example of tree

1. 单表的搜索树
2. 两表join的搜索树，只有谓词中有连接关系的才需要先进行join。分别做NLJ和SMJ。
3. 比较相同表相同order的NLJ和SMJ的cost大小，选择其中较小的表order方法，再对第三个表进行相同order的NLJ和SMJ cost大小比较。

到此为止可以先总结一下：最开始我们通过catalog中的统计信息估算出谓词的选择因子，然后就可以计算出单表下的几条访问路径开销，这些访问路径会应用单表下的本地谓词，这些访问路径开销由磁盘和CPU共同组成，并且单表下的访问结果是按照某个colomn order排序或者无序的。得到单表访问结果以后就可以join第二张表，join分成NLJ和SMJ，我们需要比较相同表相同order的NLJ和SMJ结果，取开销小的，相同的操作join第三张表。

# 6. Nested Queries

子查询结果如果是一个集合的话，只能被顺序访问。依赖于高层查询的子查询叫做相关子查询。非相关子查询从里到外被计算。

# 7. Conclusion

等价类：表集合+interesting order 都相同的就属于一个类。

将一个不属于interesting order的表集合转换成order需要多一步sort的开销。

动态规划的思想，合并相同等价类的最优解，再加入新的表。