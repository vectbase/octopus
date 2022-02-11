### PrestoSQL(Trino)

参考张振提供的presto文档部署trino-server-353以及client在测试集群slave03上

关于prestoSQL和别的组件对接，以kafka为例，在server文件夹下etc/catalog配置x.properties文件（非常好玩！值得一试！），参考：https://trino.io/docs/current/connector/kafka-tutorial.html

### PrestoSQL(Trino) plugin

参考: https://www.jianshu.com/p/888186c38827

将ranger的presto插件服务部署到presto server的节点上

```
tar xvf ranger-2.1.0-presto-plugin.tar.gz
cd ranger-2.1.0-presto-plugin
```

修改install.properties中相关配置

```
POLICY_MGR_URL=http://slave03:6080
REPOSITORY_NAME=prestodev
COMPONENT_INSTALL_DIR_NAME=../../trino-server-353

# Enable audit logs to Solr
XAAUDIT.SOLR.ENABLE=true
XAAUDIT.SOLR.URL=http://slave01:8983/solr/ranger_audits
XAAUDIT.SOLR.USER=NONE
XAAUDIT.SOLR.PASSWORD=NONE
XAAUDIT.SOLR.ZOOKEEPER=master:2181,slave01:2181,slave02:2181/solr
XAAUDIT.SOLR.SOLR_URL=http://slave01:8983/solr/ranger_audits

# Solr Audit Provider
XAAUDIT.SOLR.IS_ENABLED=true
XAAUDIT.SOLR.MAX_QUEUE_SIZE=1
XAAUDIT.SOLR.MAX_FLUSH_INTERVAL_MS=1000
XAAUDIT.SOLR.SOLR_URL=http://slave01:8983/solr/ranger_audits

#
# Custom component user
# CUSTOM_COMPONENT_USER=<custom-user>
# keep blank if component user is default
CUSTOM_USER=presto


#
# Custom component group
# CUSTOM_COMPONENT_GROUP=<custom-group>
# keep blank if component group is default
CUSTOM_GROUP=hadoop

#虽然文档中没有提及，不设置的话，enable-presto-plugin.sh脚本执行出错
XAAUDIT.SUMMARY.ENABLE=false
```

然后用root执行：
```shell
# ./enable-presto-plugin.sh
// 省略中间输出
Ranger Plugin for presto has been enabled. Please restart presto to ensure that changes are effective.
```

重启sparkSQL:
```shell
/home/servers/trino-server-353/bin/launcher restart
```