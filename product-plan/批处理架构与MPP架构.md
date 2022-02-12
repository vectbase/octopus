# 批处理架构与MPP架构

## 对比

|                                     | **批处理架构**  | **MPP架构**                  |
| ----------------------------------- | --------------- | ---------------------------- |
| executor与数据的关系                | 解耦            | 耦合                         |
| 单个executor的数据视角 （数据分片） | 可看到全部数据  | 只能看到本executor对应的数据 |
| 参与计算的executor                  | 部分executor    | 全部executor                 |
| task与executor的关系                | 解耦            | 耦合                         |
| task执行方式                        | 分阶段异步执行  | 同时执行                     |
| task调度                            | 细粒度          | 组粒度                       |
| 容错性                              | 高（推测执行）  | 低（木桶效应）               |
| 扩展性                              | 高              | 低                           |
| 中间结果集                          | 落盘            | 不落盘                       |
| 延时                                | 高（分钟~小时） | 低（秒）                     |
| 并发度                              | 高              | 低                           |

注：executor相当于worker进程。

以上特征的因果关系总结如下（以批处理架构为例，MPP架构同理）：

1. 数据视角：

   executor与数据解耦 => 每个executor可看到全部数据 => 部分executor参与计算 => 决定调度方式

2. 调度视角：

   task与executor解耦 => task异步执行 => 好处是细粒度调度 => 容错性高 => 扩展性高

   task与executor解耦 => task异步执行 => 坏处是中间结果集落盘 => 延时高

   task与executor解耦 => 好处是并发度高

3. 批处理执行慢是因为task是分阶段异步执行，而MPP是同时执行。

4. 如果能做到细粒度调度和task同时执行，就兼具了两种架构的优势。                                                                

## 参考

- [MPP大规模并行处理架构详解](https://www.cnblogs.com/itlz/p/14998858.html)

- [MPP 的进化：深入理解 Batch 和 MPP 优缺点](https://toutiao.io/posts/2a9ayg/preview)

- [一篇文章掌握 Sql-On-Hadoop 核心技术](https://www.infoq.cn/article/an-article-mastering-sql-on-hadoop-core-technology)

- [Apache HAWQ(sigmod 2014)论文笔记](https://zhuanlan.zhihu.com/p/23167162)

- HAWQ论文：https://github.com/changleicn/publications/raw/master/hawq-sigmod-2014.pdf

- [Presto实现原理和美团的使用实践](https://tech.meituan.com/2014/06/16/presto.html) 其中presto生成的plan的执行流程

- AnalyticDB（并没有详细讲MPP+DAG）：

  [云栖干货回顾 |“顶级玩家”集结！分布式数据库专场精华解读_能力](https://www.sohu.com/a/347392198_612370)

  [阿里如何实现海量数据实时分析技术-AnalyticDB - BarryW - 博客园](https://www.cnblogs.com/barrywxx/p/10141153.html)

  [Why AnalyticDB for MySQL is the Best Bet for Building a Real-time Data Warehouse](https://www.alibabacloud.com/blog/why-analyticdb-for-mysql-is-the-best-bet-for-building-a-real-time-data-warehouse_596492)

  

