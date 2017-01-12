---
layout: post
title: "搭建大数据全栈集群"
date: 2017-01-12 10:24:44 +0800
comments: true
categories: 
---


###前言
从2015年开始搭建数据分析系统以来，已经过了一年多时间，有必要对之前搭建的经历做一个总结。
本问主要包含以下几个系统/框架

- Hadoop (HDFS/YARN)
- Hive
- HBase
- ZooKeeper
- Spark

其中hadoop的HDFS做为Hive和Hbase的存储基础设施，也保存历史日志数据。
YARN则做为spark的任务执行调度，
引入Hbase则是为了补充Hive不方便做部分内容更新的问题。
ZooKeeper做为Hbase的支撑，也为未来其他以来zk的系统做支撑。
Spark的作用有很多，目前搭建的目的主要是做为数据解析处理的工具。

<!-- more -->

###准备
####硬件
四台服务器，其中一台为跳板机和工作机，其他三台为集群机；三台中的一台做为hadoop的name node
####软件

- JRE
- Scala (跳板机)
- SSH Private Key/Public Key
- 各个软件的最新版
- 本地DNS服务器

另外需要给三台集群机器配置免密码的ssh　public key, 并且创建相同的用户（我用了默认的ubuntu用户），因为大部分hadoop/spark的启动命令都是通过ssh直接发送的。
建议是通过：`ssh-keygen -t rsa`创建key pair，并将public key的内容拷贝到各台机器的`~/.ssh/authorized_keys`文件中。

DNS服务器的目的是由于在云主机的环境里，每次重启IP地址都会发生变化，而配置文件中不少地方都会需要指定主机名或IP地址，尤其是Hbase等，对这块尤其挑剔，配置成主机名经常造成莫名其妙的错误，一个比较简单的方案是用一个网络内的机器做DNS服务器（通过DNSMask等软件），并将其IP设置为所有主机的首选域名解析地址。
比如这样的配置 /etc/network/interfaces：

```
dns-nameservers 10.19.100.100
```

###Step 1: 配置执行路径
在每台机器上的/etc/profile文件都进行如下配置：

```
HADOOP_PREFIX=/home/ubuntu/hadoop-2.7.2
HADOOP_HOME=/home/ubuntu/hadoop-2.7.2
HADOOP_CONF_DIR=/home/ubuntu/hadoop-2.7.2/etc/hadoop
HBASE_HOME=/home/ubuntu/hbase-1.1.4
HIVE_HOME=/home/ubuntu/apache-hive-2.0.0-bin
JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk-amd64
SPARK_HOME=/home/ubuntu/spark-1.6.1-bin-hadoop2.6
ZOOKEEPER_HOME=/home/ubuntu/zookeeper-3.4.8

PATH=/home/ubuntu/spark-1.6.1-bin-hadoop2.6 /bin:/home/ubuntu/hbase-1.1.4/bin:/home/ubuntu/apache-hive-2.0.0-bin/hcatalog/bin:/home/ubuntu/apache-hive-2.0.0-bin/bin:/home/ubuntu/hadoop-2.7.2/bin:$PATH
```


###Step　2: 配置Ｈadoop
hadoop的几个核心配置文件是：

- $HADOOP_HOME/etc/hadoop/core-site.xml
- $HADOOP_HOME/etc/hadoop/hdfs-site.xml
- $HADOOP_HOME/etc/hadoop/yarn-site.xml
- $HADOOP_HOME/etc/hadoop/mapred-site.xml

分别代表hadoop的几个核心组件：核心、hdfs、yarn、mapreduce
大部分的配置使用默认就可以，需要配置的主要有如下几个：

- HDFS Name Node文件保存路径
- HDFS Data Node 文件保持路径，这也是HDFS上的文件实际存储的路径
- fs.defaultFS  默认文件系统，通常指定name node的地址：hdfs://master.local:8020
- Resource Host:　资源管理器的url
除此之外，由于任务运行需要分配到不同的机器，而运行任务需要一个用户身份，就是代理用户，需要配置代理身份的用户名和组名。

以下是实际配置：
core-site.xml

```
  <property>
     <name>fs.defaultFS</name>
     <value>hdfs://master.local:8020</value>
  </property>
  <property>
    <name>hadoop.proxyuser.ubuntu.hosts</name>
    <value>*</value>
  </property>
  <property>
    <name>hadoop.proxyuser.ubuntu.groups</name>
    <value>*</value>
  </property>
```
hdfs-site.xml

```
   <property>
      <name>dfs.namenode.name.dir</name>
      <value>file:///data/hdfs/namenode</value>
   </property>
   <property>
      <name>dfs.datanode.data.dir</name>
      <value>file:///data/hdfs/datanode</value>
   </property>
  <property>
    <name>dfs.namenode.datanode.registration.ip-hostname-check</name>
    <value>false</value>
  </property>

```
yarn-site.xml
```
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>master.local</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <-- 禁止系统在任务内存消耗超过一定比率时强制杀掉进程，减少任务直接中断的概率，也可以不配置-->
  <property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
  </property>

```
mapred-site.xml

```
<!-- 指定由yarn来管理mapreduce的任务-->
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <!-- 更新任务历史的通讯地址-->
  <property>
    <name>mapreduce.jobhistory.address</name>
    <value>master.local:10020</value>
  </property>
    <!-- 查看任务历史的网页地址，这两个配置都指向到name node-->
  <property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>master.local:19888</value>
  </property>
```
除了xml配置，还需要给启动脚本配置两个文件，帮助启动脚本确定应该在哪台服务器启动哪些服务器，主要区分为:

- master
- slave
master文件指定name node和resource manager的地址（都是同一个服务器）
slave文件则指定data node 和node manager的地址
这两个文件都一行一个值。

以上配置应该在三台集群机都是一样的。

之后就可以启动hadoop了（具体来说是启动hdfs和yarn）。

```
sbin/start-hdfs.sh
sbin/start-yarn.sh
```
第一次启动之后，需要对文件系统进行格式化：
```
hdfs namenode -format
```

###step3 配置Hive
Hive其实有两部分：一部分是SQL解析引擎，一部分是表结构等相关信息的存储功能。
在hive的安装目录下的hive其实是第一部分，另外一部分需要保存元信息，需要数据库，比如postgresql或mysql，这里用mysql。这另一部分可以分为：server和meta store。这两个部分在直接用hive时用出不大，但是当hive需要跟其他部分集成时特别重要，比如跟spark集成时，spark需要从hive中读取数据，则需要通过hiveserver访问Hive资源。

如果不做其他配置，启动hive会碰到类似这样的错误：
```
Exception in thread "main" java.lang.RuntimeException: Hive metastore database is not initialized. Please use schematool (e.g. ./schematool -initSchema -dbType ...) to create the schema. If needed, don't forget to include the option to auto-create the underlying database in your JDBC connection string (e.g. ?createDatabaseIfNotExist=true for mysql)
```
错误信息基本说明了问题所在：

- schema 还未初始化
- 数据库链接未配置
基于如下配置：
```
    <property>
        <name>datanucleus.autoCreateTables</name>
        <value>true</value>
    </property>
    <property>
        <name>datanucleus.autoStartMechanism</name>
        <value>SchemaTable</value>
    </property>
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://127.0.0.1:3306/metastore?createDatabaseIfNotExist=true</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>username</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>password</value>
    </property>
    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://master.local:9083</value>
    </property>
```
执行如下命令:
```
$HIVE_HOME/bin/schematool -dbType mysql -initSchema
```
可以将hive初始化。
hive的客户接口除了hive命令行之外，还有hiveserver2自带的beeline，beeline的功能更多，比如用户管理，权限管理等等。

同时为今后与其他系统集成，在集群的master机上运行：
```
$HIVE_HOME/bin/hive --service hiveserver2
$HIVE_HOME/bin/hive --service metastore
```

###step4 配置Spark
spark跟hive类似，其实是hadoop集群的客户端，或者叫driver驱动器，也即由spark向yarn集群提交任务，具体执行是在yarn管理的node里运行jvm执行(spark的scala也是jvm语言)。加上spark会自动识别系统里的hadoop配置文件（比如resource manager的地址），所以spark需要配置的东西其实不多。
通常只需将hive-site.xml配置文件拷贝到spark的conf目录下，这样在使用HiveContext时，能自动找到hive表的schema。

###step5 配置ZooKeeper
zookeeper不是必须的，整个系统中只有hbase用到了zk，而hbase中自带有zk，可以直接启动hbase的zk。为了保持zk的独立性，也为了今后其他系统接入方便，特别将zk独立出来。

由于zk没有主从概念，所以三台机器的配置都是类似的，唯一的区别是需要为每个节点分配一个唯一的节点标识，并且需要在配置文件中指定其他节点的地址。
下面是一个节点的配置：

```
dataDir=/data/zookeeper
server.1=master.local:2888:3888
server.2=slave1.local:2888:3888
server.3=slave2.local:2888:3888
```
其中的两个端口分别表示：zk节点内部通信用的端口、主节点选举时用的通讯端口。
在配置中指定的文件目录（这里为:/data/zookeeper），需要一个文件myid，里面的内容为节点标识，比如第一个节点的内容就是１

配置完成之后在每个节点执行：
```
bin/zkServer.sh start
```
###Step6 配置Hbase
Hbase基于HDFS文件系统，弥补了直接在HDFS上或在HIve表里管理数据的一些缺点，可以方便地进行增删查改的操作，而且Hbase基于列的存储，也决定了他可以方便地调整数据结构。如果配合hive表，则可以实现sql接口的hbase表操作，将Hive表视为hbase的一种“视图”。
Hbase集群有两种角色，一种是master，一种是regionserver，类似hadoop的data node。
需要配置的主要内容有：

- 文件存储目录（HDFS系统而言）
- ZooKeeper配置
下面是一个例子：
```
 <property>
    <name>hbase.rootdir</name>
    <value>hdfs://master.local:8020/hbase</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>>/data/zookeeper</value>
  </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>master.local,slave1.local,slave2.local</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
  </property>
```

###step 7 HBase集成Hive
主要参考：　https://cwiki.apache.org/confluence/display/Hive/HBaseIntegration
主要注意两点：

- Hive的Hbase Handler
- Hive的配置

Hive提供了Hbase存储的处理类：org.apache.hadoop.hive.hbase.HBaseStorageHandler
可以视为Hive底层的一种存储引擎。
在原来hive配置的基础上增加如下配置：
```
  <property>
    <name>hive.aux.jars.path</name>
    <value>file:///home/ubuntu/apache-hive-2.0.0-bin/lib/hbase-client-1.1.1.jar,file:///home/ubuntu/apache-hive-2.0.0-bin/lib/hbase-server-1.1.1.jar,file:///home/ubuntu/download/apache-hive-2.1.0-bin/lib/hive-hbase-handler-2.1.0.jar,file:///home/ubuntu/apache-hive-2.0.0-bin/lib/hbase-protocol-1.1.1.jar,file:///home/ubuntu/apache-hive-2.0.0-bin/lib/zookeeper-3.4.6.jar,file:///home/ubuntu/elasticsearch-hadoop-2.3.2/dist/elasticsearch-hadoop-2.3.2.jar</value>
    <description>The location of the plugin jars that contain implementations of user defined functions and serdes.</description>
  </property>
<property>
  <name>hbase.zookeeper.quorum</name>
  <value>master.local,slave1.local,slave2.local</value>
  <description>A comma separated list (with no spaces) of the IP addresses of all ZooKeeper servers in the cluster.</description>
</property>
<property>
  <name>hbase.zookeeper.property.clientPort</name>
  <value>2181</value>
  <description>The Zookeeper client port. The MapR default clientPort is 5181.</description>
</property>
<property>
  <name>hive.server2.thrift.port</name>
  <value>10000</value>
</property>
<property>
  <name>hive.server2.thrift.bind.host</name>
  <value>0.0.0.0</value>
</property>
```
在hive创建基于hbase的表时，需要通过如下命令：
```
CREATE TABLE hbase_table_1(value map<string,int>, row_key int) 
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES (
"hbase.columns.mapping" = "cf:col_prefix.*,:key"
)
TBLPROPERTIES("hbase.table.name" = "hbase_table_1");
```
事先必须先创建好Hbase表。
