# Flink CDC Connector

## 概念

​        在 Flink 1.11 开始引入了 CDC 机制，CDC 的全称是 Change Data Capture，用于捕捉数据库表的增删改查操作，是目前非常成熟的同步数据库变更方案。Flink CDC Connectors 是 Apache Flink 的一组源连接器，是可以从 MySQL、PostgreSQL 数据直接读取全量数据和增量数据的 Source Connectors，另外支持解析 Kafka 中 debezium-json 和 canal-json 格式的 Change Log，通过Flink 进行计算或者直接写入到其他外部数据存储系统(比如 Elasticsearch)，或者将 Changelog Json 格式的 Flink 数据写入到 Kafka。Flink SQL CDC 内置了 Debezium 引擎，利用其抽取日志获取变更的能力，将 changelog 转换为 Flink SQL 认识的 RowData 数据。

​        Debezium是已经成熟的CDC（Changelog Data Capture，变更数据捕获）的工具，可以把来自 MySQL、PostgreSQL、Oracle、Microsoft SQL Server 和许多其他数据库的更改实时流式传输到 Kafka 中。 它通过重用 Kafka 和 Kafka Connect 来实现持久性、可靠性和容错质量。每个连接器监控单个上游数据库服务器，捕获所有更改并将它们记录在一个或多个 Kafka topic中（通常每个数据库表一个主题）。Kafka 确保所有这些数据变化事件都被复制并完全有序，并允许多个客户端独立使用这些相同的数据变化事件，而对上游系统的影响很小。此外，客户端可以随时停止消费，当他们重新启动时，他们会从上次中断的地方恢复。

## 特性

- 支持读取数据库快照并继续读取binlog，即使发生故障也只处理一次。
- DataStream API 的 CDC 连接器，用户可以在单个作业中使用对多个数据库和表的更改，而无需部署 Debezium 和 Kafka。
- Table/SQL API 的 CDC 连接器，用户可以使用 SQL DDL 创建 CDC 源来监视单个表上的更改。

## 性能

| 目标端       | 性能  |
| ------------ | ----- |
| KAFKA        | 10w+  |
| MySQL-insert | 1w+   |
| MySQL-update | 6000+ |
| MySQL-delete | 1.2w+ |

## 针对各种failure的处理（保证数据一致性）

- Debezium Source 对 MySQL 进行 Snapshot 时发生异常

在 Flink Task 启动后，首先会进行 MySQL 全表扫描，也就是做 Snapshot，这里有个需要注意的地方就是，在 Snapshot 阶段，在扫描全表数据时，没有可用于恢复的位点，所以无法在全表扫描阶段去执行 Checkpoint。为了不执行 Checkpoint，MySQL 的 CDC 源表会让执行中的 Checkpoint 一直等待（通过持有 checkpoint 锁实现），甚至 Checkpoint 超时（如果表超级大，扫描耗时非常长）。这块可以从 DebeziumChangeConsumer 的代码中看到：

在做 Snapshot 阶段，可能会碰到源库 MySQL 异常或者 Flink 任务本身异常，那我们分别分析下异常后如何恢复：

1. **若遇到源库 MySQL 异常**，Flink Task 发现无法连接数据库异常退出，重新启动 Flink Task（或者 retry），因为没有做 snapshot 没做 checkpoint，那么会重新再做一次 Snapshot，这些全量数据最后发送到目的 MySQL，由于下游 MySQL 实现了写幂等，因此最终保持一致性。
2. **若遇到 Flink 任务异常**，重新启动（或者 retry），同上面情况一样，重新做一次 Snapshot，最终也能保持一致性。
3. **若遇到目标库 MySQL 异常**，同场景一一致，Flink Task 无法往目标数据库写入异常退出，在需要重新启动或 retry 后，重新做一次 Snapshot，全量数据最后发送到目的 MySQL，由于目的下游 M 有 SQL 实现了写幂等，最终保持一致性。



- Snapshot 完成后读取 binlog 时发生异常

在全量数据完成同步后，开始进行增量获取，此时 Flink 会进行定时 Checkpoint，将读取 binlog 的位移信息和 schema 信息存入到 StateBackend，若此时发生异常，那我们分析下异常后如何恢复：

1. **若源 MySQL 异常**，Flink Task 发现无法连接数据库异常退出，重新启动 Flink Task（或者 retry），将会从最近一次 Checkpoint 的数据进行恢复，由于可以读取到 mysql binlog 位移信息，实现继续同步，不会丢失数据，最终也能保持一致性。
2. **若 Flink 任务异常**，重新启动或 retry 后，同场景 1 一致，继续读取 binlog，能保持一致性。
3. **若目的 MySQL 异常**，jdbc connector 无法往目标数据库写入，cdc connector 读取到的 binlog 位移信息也不再更新，两个操作是一个原子性操作，在 Flink Task 恢复后，从最近一次 Checkpoint 进行恢复，最终保持一致性。

## 优点

- 开箱即用，简单易上手
- 减少维护的组件，简化实时链路，减轻部署成本
- 减小端到端延迟
- Flink 自身支持 Exactly Once 的读取和计算
- 数据不落地，减少存储成本
- 支持全量和增量流式读取
- binlog 采集位点可回溯

## 缺点

- 使用正则匹配原表后（多个源端表），到目标表无法进行一对一的映射。需要逐个匹配。
- CDC source 端定义时，需要指定所有字段，目前不支持省略字段定义。
- CDC 到 KAFKA 时无法按照主键进行自动分区分发、无法指定分区键分发数据。到 KAFKA 的数据格式指定（JSON，AVRO JSON等）。
- 目标端支持需求：DB2、ADB/GreenPlum、Oracle 暂不支持。不支持 DDL同步，不支持表的创建。
- 任务管理和监控的 REST API 不完善。

## 总结

分布式系统中端到端一致性需要各个组件参与实现，Flink SQL CDC + JDBC Connector 可以通过如下方法保证端到端的一致性：

- 源端是数据库的 binlog 日志，全量同步做 Snapshot 异常后可以再次做 Snapshot，增量同步时，Flink SQL CDC 中会记录读取的日志位移信息，也可以 replay
- Flink SQL CDC 作为 Source 组件，是通过 Flink Checkpoint 机制，周期性持久化存储数据库日志文件消费位移和状态等信息（StateBackend 将 checkpoint 持久化），记录消费位移和写入目标库是一个原子操作，保证发生 failure 时不丢数据，实现 Exactly Once
- JDBC Sink Connecotr 是通过写入时保证 Upsert 语义，从而保证下游的写入幂等性，实现 Exactly Once

## 附：版本说明

### 支持数据库的版本说明

| Connector                                                    | Database   | Database Version                               | Flink Version |
| ------------------------------------------------------------ | ---------- | ---------------------------------------------- | ------------- |
| [MySQL CDC](https://github.com/ververica/flink-cdc-connectors/wiki/MySQL-CDC-Connector) | MySQL      | Database: 5.7, 8.0.x JDBC Driver: 8.0.16       | 1.11+         |
| [Postgres CDC](https://github.com/ververica/flink-cdc-connectors/wiki/Postgres-CDC-Connector) | PostgreSQL | Database: 9.6, 10, 11, 12 JDBC Driver: 42.2.12 | 1.11+         |

### CDC对应Flink各个版本

| Flink CDC Connector Version | Flink Version |
| --------------------------- | ------------- |
| 1.0.0                       | 1.11.*        |
| 1.1.0                       | 1.11.*        |
| 1.2.0                       | 1.12.*        |
| 1.3.0                       | 1.12.*        |
| 1.4.0                       | 1.13.*        |





