# clickhouse and s3

## 版本

- clickhouse v21.4.4.30-stable
- minio RELEASE.2021-04-06T23-11-00Z

## 机器环境

主机名：ubuntu0，IP：192.168.56.40

主机名：centos0.local，IP：192.168.56.30

主机名：centos4.local，IP：192.168.56.34

## minio

安装文档：

- https://docs.min.io/docs/minio-quickstart-guide.html
- https://docs.min.io/docs/minio-client-quickstart-guide.html

启动minio server：

```
$ pwd
/export/minio
$ cat minio-server.sh
export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=minioadmin
export MINIO_KMS_SECRET_KEY=my-minio-encryption-key:w32MemBMibtmqaeTBjWpjiUfWrq0eGrJsL6p/8et/r0=
nohup ./minio server --console-address ":9998" --address ":9999" ./data > minio-server.log 2>&1 &
$ ./minio-server.sh
```

其中`MINIO_KMS_SECRET_KEY`参数是可选的，用于配置信息的加密，它是通过命令`cat /dev/urandom | head -c 32 | base64 -`产生的。

配置minio client：

```
$ cat ~/.mc/config.json
{
	"version": "10",
	"aliases": {
		"local": {
			"url": "http://ubuntu0:9999",
			"accessKey": "minioadmin",
			"secretKey": "minioadmin",
			"api": "S3v4",
			"path": "auto"
		}
}
```

上传一个文件到minio：

```
$ cat localdata/data.csv
1,2,3
3,2,1
6,6,6
0,0,0
$ mc cp localdata/data.csv local/bucket1
$ mc ls local/bucket1
[2021-04-19 16:52:35 CST]    24B data.csv
```

## clickhouse + s3

### remote_url_allow_hosts

如果使用s3 table function和table engine，需要在clickhouse-server上配置`remote_url_allow_hosts`：

```
<remote_url_allow_hosts>
    <host>ubuntu0</host>
    <!--<host_regexp></host_regexp>-->
</remote_url_allow_hosts>
```

### table function

在clickhouse-client上查询数据：

```
ubuntu0 :) SELECT * FROM s3('http://ubuntu0:9999/bucket1/data.csv', 'minioadmin', 'minioadmin', 'CSV', 'column1 UInt32, column2 UInt32, column3 UInt32');
┌─column1─┬─column2─┬─column3─┐
│       1 │       2 │       3 │
│       3 │       2 │       1 │
│       6 │       6 │       6 │
│       0 │       0 │       0 │
└─────────┴─────────┴─────────┘

4 rows in set. Elapsed: 0.004 sec.
```

注意，如果不配置上面那个`remote_url_allow_hosts`，就会报：

```
Code: 491. DB::Exception: Received from localhost:9000. DB::Exception: URL "http://ubuntu0:9999/bucket1/data.csv" is not allowed in config.xml.
```

### table engine

先在minio上创建一个文件：

```
$ mc cp local/bucket1/data.csv local/bucket1/data2.csv
```

创建table engine，并查询：

```
ubuntu0 :) CREATE TABLE s3_engine_table (column1 UInt32, column2 UInt32, column3 UInt32) ENGINE=S3('http://ubuntu0:9999/bucket1/data2.csv', 'minioadmin', 'minioadmin', 'CSV', 'none');
ubuntu0 :) select * from s3_engine_table;
┌─column1─┬─column2─┬─column3─┐
│       1 │       2 │       3 │
│       3 │       2 │       1 │
│       6 │       6 │       6 │
│       0 │       0 │       0 │
└─────────┴─────────┴─────────┘

4 rows in set. Elapsed: 0.003 sec.
```

注意，如果不事先创建文件，则报以下错误：

```
Code: 499. DB::Exception: Received from localhost:9000. DB::Exception: <?xml version="1.0" encoding="UTF-8"?>
<Error><Code>NoSuchKey</Code><Message>The specified key does not exist.</Message><Key>data2.csv</Key><BucketName>bucket1</BucketName><Resource>/bucket1/data2.csv</Resource><RequestId>16776D095F0BE185</RequestId><HostId>062bce5a-789d-498a-8147-8e68dff75f99</HostId></Error>: While executing S3.
```

### storage policy

配置s3 storage policy：

```
<yandex>
    <storage_configuration>
        <disks>
            <s3>
                <type>s3</type>
                <endpoint>http://ubuntu0:9999/bucket1/key1/</endpoint>
                <access_key_id>minioadmin</access_key_id>
                <secret_access_key>minioadmin</secret_access_key>
            </s3>
        </disks>
        <policies>
            <s3poc>
                <volumes>
                    <main>
                        <disk>s3</disk>
                    </main>
                </volumes>
            </s3poc>
        </policies>
    </storage_configuration>
</yandex>
```

注意：endpoint的格式为：`${address}/${bucket}/${key}/`，最后的`/`一定要加。

创建表时使用以上policy，并执行插入和查询操作：

```
ubuntu0 :) CREATE TABLE demo
(
    `name` String,
    `id` Int32,
    `dt` String
)
ENGINE = MergeTree
PARTITION BY dt
ORDER BY id
SETTINGS storage_policy = 's3poc';
ubuntu0 :) insert into demo values('a', 1, '20200202');
ubuntu0 :) select * from demo;
┌─name─┬─id─┬─dt───────┐
│ a    │  1 │ 20200202 │
└──────┴────┴──────────┘

1 rows in set. Elapsed: 0.005 sec.
```

### ReplicatedMergeTree

#### 数据同步(1 shard 2 replica)

在/etc/metrika.xml里配置一个shard两个replica：

```
<yandex>
    <clickhouse_remote_servers>
        <cluster_two_replicas>
            <shard>
                <replica>
                    <host>ubuntu0</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>centos0.local</host>
                    <port>9000</port>
                </replica>
            </shard>
        </cluster_two_replicas>
    </clickhouse_remote_servers>
</yandex>
```

两个replica上的宏定义分别为：

```
# ubuntu0
    <macros>
        <shard>01</shard>
        <replica>ubuntu0</replica>
    </macros>
    
# centos0.local
    <macros>
        <shard>01</shard>
        <replica>centos0</replica>
    </macros>
```

在两个ck server实例上分别执行以下建表语句：

```
ubuntu0 :) CREATE TABLE demo_replicated
(
    `name` String,
    `id` Int32, 
    `dt` String
) 
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/demo_replicated', '{replica}')  
ORDER BY id
PARTITION BY(dt)
SETTINGS storage_policy='s3poc';
```

在其中一个ck server实例上插入数据：

```
centos0.local :) insert into demo_replicated values('a', 10, '20200101');
```

在另一个ck server实例上可以查询到数据，说明数据同步成功：

```
ubuntu0 :) select * from demo_replicated;
┌─name─┬─id─┬─dt───────┐
│ a    │ 10 │ 20200101 │
└──────┴────┴──────────┘

1 rows in set. Elapsed: 0.004 sec.
```

#### zero-copy

##### 1.开启zero-copy

在/etc/clickhouse-server/config.xml中配置：

```
    <merge_tree>
        <allow_s3_zero_copy_replication>1</allow_s3_zero_copy_replication>
    </merge_tree>
```

在demo_replicated的一个replica上插入数据，当另一个replica同步完数据之后，可以在zk上看到两个replica共用一个路径：

```
/clickhouse/tables/01/default/demo_replicated2/zero_copy_s3/shared/5f574679f0fdcc89e6d07a3290c4dfc0_0_0_0/key1_femqzqqufbodwbrhphmirllhckcrabkw/{utuntu0, centos0}
```

用`mc ls local/bucket1/key1/ | sort`命令查看minio上新增的文件数是9个。zk里的路径就对应mino上新增的文件。

需要说明的是，zk路径中`femqzqqufbodwbrhphmirllhckcrabkw`是data part的checksums.txt文件在mino上的key。

##### 2.关闭zero-copy

将`allow_s3_zero_copy_replication`配置改为0。

在demo_replicated的一个replica上插入数据，当另一个replica同步完数据之后，可以在zk上看到两个replica使用单独的路径：

```
/clickhouse/tables/01/default/demo_replicated2/zero_copy_s3/shared/9689420b6747a2e726512fe04e11d1a9_0_0_0/key1_bimqaimdhannskadezkiuvkjqbfqjpze/ubuntu0
/clickhouse/tables/01/default/demo_replicated2/zero_copy_s3/shared/9689420b6747a2e726512fe04e11d1a9_0_0_0/key1_fjyfddlapaspyhbubxgcpobhjklmtday/centos0
```

用`# mc ls local/bucket1/key1/ | sort`命令查看minio上新增的文件数是18个。zk里的路径分别对应mino上新增的文件，每个replica对应9个新增的文件。

#### 分布式表(2 shard 1 replica)

在/etc/metrika.xml里配置两个shard各一个replica：

```
<yandex>
    <clickhouse_remote_servers>
        <cluster_two_shards>
            <shard>
                <replica>
                    <host>ubuntu0</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <host>centos0.local</host>
                    <port>9000</port>
                </replica>
            </shard>
        </cluster_two_shards>
    </clickhouse_remote_servers>
</yandex>
```

 在ubuntu0上创建本地表：

```
ubuntu0 :) CREATE TABLE demo_shard
(
    `name` String,
    `id` Int32,
    `dt` String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/01/{database}/demo_shard', '{replica}')
ORDER BY id
PARTITION BY(dt)
SETTINGS storage_policy='s3poc';
```

在centos0.local上创建本地表：

```
centos0.local :) CREATE TABLE demo_shard
(
    `name` String,
    `id` Int32,
    `dt` String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/02/{database}/demo_shard', '{replica}')
ORDER BY id
PARTITION BY(dt)
SETTINGS storage_policy='s3poc';
```

创建分布式表：

```
ubuntu0 :) CREATE TABLE all_demo_shard ON CLUSTER cluster_two_shards as demo_shard ENGINE = Distributed(cluster_two_shards, default, demo_shard);
```

分别在两个本地表上插入数据，类似这样：

```
ubuntu0 :) insert into demo_shard values('y', 31, '20200401');
```

在分布式表上可以正常查询出来：

```
ubuntu0 :) select * from all_demo_shard;
┌─name─┬─id─┬─dt───────┐
│ y    │ 31 │ 20200401 │
└──────┴────┴──────────┘
┌─name─┬─id─┬─dt───────┐
│ y    │ 30 │ 20200401 │
└──────┴────┴──────────┘
┌─name─┬─id─┬─dt───────┐
│ z    │  2 │ 20200401 │
└──────┴────┴──────────┘

3 rows in set. Elapsed: 0.010 sec.
```

## clickhouse + juicefs + s3

参考：

- 快速开始：https://github.com/juicedata/juicefs/blob/main/docs/en/quick_start_guide.md
- 命令行参数：https://github.com/juicedata/juicefs/blob/main/docs/en/command_reference.md

### 元数据服务

参考：https://github.com/juicedata/juicefs/blob/main/docs/en/databases_for_metadata.md

这里以redis或mysql作为元数据服务（二者选其一即可）。

#### 1.redis

修改redis.conf文件，修改参数允许远程client访问：

```
bind * -::*
protected-mode no
```

启动：

```
$ nohup src/redis-server ./redis.conf > redis-server.log 2>&1 &
```

#### 2.mysql

mysql版本：mariadb-server-5.5.68

在mysql中创建用户和授权：

```
CREATE DATABASE juicefs;
CREATE USER 'juicefs'@'%' IDENTIFIED BY 'juicefs';
GRANT ALL ON juicefs.* TO 'juicefs'@'%' IDENTIFIED BY 'juicefs' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

如果想在本地用juicefs用户登录，则删除匿名用户：

```
USE mysql;
DELETE FROM user WHERE User='';
```

### 初始化文件系统

格式化一个volume，volume名字为vol1（这相当于初始化一个文件系统卷）：

```
# 使用redis作为元数据服务
$ juicefs format \
	--storage minio \
	--bucket http://192.168.56.34:9999/bkt1 \
	--access-key minioadmin \
	--secret-key minioadmin \
	redis://192.168.56.34:6379/1 \
	vol1

# 使用mysql作为元数据服务
$ juicefs format --storage minio \
    --bucket http://192.168.56.34:9999/bkt1 \
    --access-key minioadmin \
    --secret-key minioadmin \
    mysql://juicefs:juicefs@"(192.168.56.34:3306)"/juicefs \
    vol1
```

使用以下命令可以查看volume配置和状态：

```
$ juicefs status redis://192.168.56.34:6379/1
$ juicefs status mysql://juicefs:juicefs@"(192.168.56.34:3306)"/juicefs
```

### 使用posix接口

如果想使用posix接口，则挂载到一个本地目录：

```
$ juicefs mount -d redis://192.168.56.34:6379/1 /mnt/jfs
```

### 使用s3接口

如果想使用s3接口，则需要启动gateway服务：

```
$ export MINIO_ROOT_USER=minioadmin
$ export MINIO_ROOT_PASSWORD=minioadmin
$ nohup juicefs gateway --cache-dir /data0/juicefs/cache --access-log /data0/juicefs/access.log redis://192.168.56.34:6379/1 192.168.56.34:9900 > gateway.log 2>&1 &
```

gateway缓存的文件就在本地的`/data0/juicefs/cache`目录里，其中的文件与在minio里看到的文件是一样的。

通过浏览器可访问gateway的web UI：http://192.168.56.34:9900 。注意在juicefs UI上和在minio UI上看到的bucket的内容不一样：

- 在juicefs上看到的bucket名字是vol1，而在minio上看到的bucket名字是bkt1。
- juicefs把file拆成了object存到了对象存储里，所以在juicefs里看到的是file，而在minio里看到的是object。

通过mc访问gateway：

```
$ mc alias set juicefs http://192.168.56.34:9900 minioadmin minioadmin --api S3v4
$ mc ls juicefs/bkt1
```

### 创建表

配置clickhouse storage policy：

```
<yandex>
    <storage_configuration>
        <disks>
            <juicefs>
                <type>s3</type>
                <endpoint>http://192.168.56.34:9900/vol1/key1/</endpoint>
                <access_key_id>minioadmin</access_key_id>
                <secret_access_key>minioadmin</secret_access_key>
            </juicefs>
        </disks>
        <policies>
            <juicefspoc>
                <volumes>
                    <main>
                        <disk>juicefs</disk>
                    </main>
                </volumes>
            </juicefspoc>
        </policies>
    </storage_configuration>
</yandex>
```

这里的`key1`是自定义的，在web UI上呈现的就是一个名为`key1`的目录。可以给不同的replica设置不同的key名字（比如取replica名字），那么会方便debug。

创建clickhouse table：

```
CREATE TABLE default.demo_juicefspoc
(
    `name` String,
    `id` Int32,
    `dt` String
)
ENGINE = MergeTree
PARTITION BY dt
ORDER BY id
SETTINGS storage_policy = 'juicefspoc';
```

### 导数

建表：

```
CREATE TABLE ssb.lineorder_jfs
(
    `LO_ORDERKEY` UInt32,
    `LO_LINENUMBER` UInt8,
    `LO_CUSTKEY` UInt32 CODEC(T64, LZ4),
    `LO_PARTKEY` UInt32 CODEC(T64, LZ4),
    `LO_SUPPKEY` UInt32 CODEC(T64, LZ4),
    `LO_ORDERDATE` Date CODEC(T64, LZ4),
    `LO_ORDERPRIORITY` LowCardinality(String) CODEC(ZSTD(1)),
    `LO_SHIPPRIORITY` UInt8,
    `LO_QUANTITY` UInt8 CODEC(ZSTD(1)),
    `LO_EXTENDEDPRICE` UInt32 CODEC(T64, LZ4),
    `LO_ORDTOTALPRICE` UInt32 CODEC(T64, LZ4),
    `LO_DISCOUNT` UInt8 CODEC(ZSTD(1)),
    `LO_REVENUE` UInt32 CODEC(T64, LZ4),
    `LO_SUPPLYCOST` UInt32 CODEC(T64, LZ4),
    `LO_TAX` UInt8 CODEC(ZSTD(1)),
    `LO_COMMITDATE` Date CODEC(T64, LZ4),
    `LO_SHIPMODE` LowCardinality(String) CODEC(ZSTD(1))
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(LO_ORDERDATE)
ORDER BY (LO_SUPPKEY, LO_ORDERDATE)
SETTINGS storage_policy = 'juicefspoc';
```

导数：

```
$ clickhouse-client --query "INSERT INTO ssb.lineorder_jfs FORMAT CSV" --format_csv_delimiter="," --max_insert_block_size="300000" < lineorder.tbl
```

查询：

```
select sum(lo_extendedprice*lo_discount) as revenue from ssb.lineorder_jfs;
```

### 负载均衡

使用sidekick：https://blog.min.io/introducing-sidekick-a-high-performance-load-balancer/
