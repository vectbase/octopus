# Uber RSS配置文档

https://github.com/uber/RemoteShuffleService

## 1.如何搭建

打包RSS服务端，得到 target/remote-shuffle-service-0.0.9-server.jar

```bash
mvn clean package -Pserver -DskipTests
```



打包RSS客户端，得到 target/remote-shuffle-service-0.0.9-client.jar

```bash
mvn clean package -Pclient -DskipTests
```



## 2.如何运行RSS（Standalone）

### 2.1 运行RSS Server

在环境中选择一台机器，例如：server1,上传Server jar包。用java执行server jar包：

```bash
java -Dlog4j.configuration=log4j-rss-prod.properties -cp /home/xinghao/RSS/remote-shuffle-service-0.0.9-server.jar com.uber.rss.StreamServer -port 12222 -serviceRegistry standalone -dataCenter dc1
```



### 2.2 运行RSS Client

上传client jar包到HDFS中，例如路径： hdfs:///file/path/remote-shuffle-service-0.0.9-client.jar

```bash
hadoop fs -put /home/xinghao/RSS/remote-shuffle-service-0.0.9-client.jar /shared
```



添加配置文件到Spark-default中：

```bash
spark.jars=hdfs:///file/path/remote-shuffle-service-0.0.9-client.jar
spark.executor.extraClassPath=remote-shuffle-service-0.0.9-client.jar
spark.shuffle.manager=org.apache.spark.shuffle.RssShuffleManager
spark.shuffle.rss.serviceRegistry.type=standalone
spark.shuffle.rss.serviceRegistry.server=server1:12222
spark.shuffle.rss.dataCenter=dc1
```



执行Spark Application即可。



## 3.如何运行RSS（Zookeeper）

### 3.1运行RSS Server

在环境中选择一台机器，例如：server1，上传Server jar包。用java执行server jar包：

```bash
java -Dlog4j.configuration=log4j-rss-prod.properties -cp /home/xinghao/RSS/remote-shuffle-service-0.0.9-server.jar com.uber.rss.StreamServer -port 12222 -serviceRegistry zookeeper -zooKeeperServers zkServer1:2181 zkServer2:2181 -dataCenter dc1
```



### 3.2运行RSS Client

上传client jar包到HDFS中，例如路径： hdfs:///file/path/remote-shuffle-service-0.0.9-client.jar

```bash
hadoop fs -put /home/xinghao/RSS/remote-shuffle-service-0.0.9-client.jar /shared
```



添加配置文件到spark-default中：

```bash
spark.jars=hdfs:///file/path/remote-shuffle-service-0.0.9-client.jar
spark.executor.extraClassPath=remote-shuffle-service-0.0.9-client.jar
spark.shuffle.manager=org.apache.spark.shuffle.RssShuffleManager
spark.shuffle.rss.serviceRegistry.type=zookeeper
spark.shuffle.rss.serviceRegistry.zookeeper.servers=zkServer1:2181,zkServer2:2181
spark.shuffle.rss.dataCenter=dc1
```



执行Spark Application即可。



## 4.测试

```bash
spark-sql> show databases;
spark-sql> use tpcds_bin_partitioned_orc_5;
spark-sql> select avg(i_current_price) as avg_price,i_manufact from item group by i_manufact;
```

备份文件存储在hdfs，shuffle文件存在RSS服务器上。