---
title: 'MacOS配置Hadoop以及Hive,Hbase'
tags:
---
本篇教程介绍了如何在MacOS上实现Hadoop伪集群
## 安装Homebrew
Homebrew是Mac系统下优秀的包管理软件，打开终端运行以下命令:
``` shell
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"hexo
```
Homebrew常用命令如下：
1. brew upgrade : 更新所有软件
2. brew upgrade [软件名] ： 更新指定软件
3. brew cleanup -n：查看哪些软件包要被清除。
4. brew cleanup [软件名]：清除指定软件包的所有老版本。
5. brew cleanup：清除所有软件包的所有老版本。
6. brew install [软件名] : 安装指定软件
7. brew prune：清理无用的symlink，且清理与之相关的位于/Applications和~/Applications中的无用App链接。
8. brew link 软件名：将指定软件的安装文件symlink到Homebrew上。
9. brew list：列出已安装的软件。
10. brew update：升级Homebrew软件
11. brew doctor：检测系统中与Homebrew有关的潜在问题。

MacOS 已经自带ssh，注意要打开远程登录，System Preference -> Sharing -> Remote Login, 然后执行ssh localhost 成功登录表示没有问题


## Hadoop安装
首先执行 brew install hadoop 安装最新版本的hadoop, 笔者使用的是zsh，以下所有配置会写入 ~/.zshrc, 如果用其它的shell 根据实际情况调整
1. 配置JAVA_HOME HADOOP_HOME 添加对应path
``` shell
export JAVA_HOME = /Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home
export HADOOP_HOME=/usr/local/Cellar/hadoop/3.0.0/libexec
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH   
```
2. 修改 core-site.xml 文件
hadoop的配置文件位于:/usr/local/Cellar/hadoop/3.0.0/libexec/etc/hadoop, 修改 core-site.xml
``` xml
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>file:/Users/mluo/hadoop_tmp</value>
        <description>Abase for other temporary directories.</description>
    </property>
            
    <property>
        <name>fs.default.name</name>
        <value>hdfs://localhost:8020</value>
    </property>
</configuration>
```

3. 修改 hdfs-site.xml 文件  
``` xml
<configuration>
        <property>
             <name>dfs.replication</name>
             <value>1</value>
        </property>
        <property>
             <name>dfs.namenode.name.dir</name>
             <value>file:/Users/mluo/hadoop_tmp/dfs/name</value>
        </property>
        <property>
             <name>dfs.datanode.data.dir</name>
             <value>file:/Users/mluo/hadoop_tmp/dfs/data</value>
        </property>
</configuration>
```

Hadoop 的运行方式是由配置文件决定的（运行 Hadoop 时会读取配置文件），因此如果需要从伪分布式模式切换回非分布式模式，需要删除 core-site.xml 中的配置项。
此外，伪分布式虽然只需要配置 fs.default.name 和 dfs.replication 就可以运行（官方教程如此），不过若没有配置 hadoop.tmp.dir 参数，则默认使用的临时目录为 /tmp/hadoo-hadoop，而这个目录在重启时有可能被系统清理掉，导致必须重新执行 format 才行。所以我们进行了设置，同时也指定 dfs.namenode.name.dir 和 dfs.datanode.data.dir，否则在接下来的步骤中可能会出错。所以不建议将 hadoop.tmp.dir 配置在
/tmp 目录下

开始执行命令:
1. hdfs namenode -format : Namenode初始化
2. start-dfs.sh :开启 NameNode 和 DataNode 守护进程。
执行 jps 命令 查看是否有如下进程 NameNode SecondaryNameNode DataNode 同时：去访问Web 界面 http://localhost:50070 查看 NameNode 和 Datanode 信息，还可以在线查看 HDFS 中的文件。



### 配置YARN

start-dfs.sh 启动 Hadoop，仅仅是启动了 MapReduce 环境，我们可以启动 YARN ，让 YARN 来负责资源管理与任务调度。
1. 修改 mapred-site.xml
``` xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```
2. 修改yarn-site.xml
``` xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

执行 start-yarn.sh 启动YARN服务 然后执行 mapred --daemon start， 通过 jps 查看，可以看到多了 NodeManager 和 ResourceManager 两个后台进程， 通过 Web 界面查看任务的运行情况：http://localhost:8088/cluster

注意
不启动 YARN 需重命名 mapred-site.xml
如果不想启动 YARN，务必把配置文件 mapred-site.xml 重命名，改成 mapred-site.xml.template，需要用时改回来就行。否则在该配置文件存在，而未开启 YARN 的情况下，运行程序会提示 “Retrying connect to server: 0.0.0.0/0.0.0.0:8032” 的错误，这也是为何该配置文件初始文件名为 mapred-site.xml.template。


## Hive配置
Hive是一个基于Hadoop文件系统HDFS上的数据仓库架构, 它为数据仓库管理提供: 数据ETL(抽取, 转换和加载)工具, 数据仓库管理和大型数据集的查询和分析功能. 定义了类SQL语言Hive QL. 其优势在于极大的可扩展性, 良好的容错性和低约束的数据输入格式

Hive中每个表都有一个对应的存储目录.
Hive有三种模式可以连接到RDBMS中:
1. Single User Mode, 利用此模式连接到一个In-Memory数据库, 一般用于单元测试
2. Multi User Mode, 通过网络连接到一个数据库中(常用)
3. Remote Server Mode, 用于非Java客户端访问元数据库, 在服务器端启动MetaStoreServer, 在客户端利用Thrift协议通过MetaStoreServer访问元数据库

### hive安装
1. brew install hive
2. 配置 HIVE_HOME, 
``` shell
export HIVE_HOME=/usr/local/Cellar/hive/2.3.1/libexec
```
3. 修改文件 hive-env.sh 如果不存在 mv hive-env.sh.templat hive-env.sh
``` shell
export HADOOP_HEAPSIZE=1024

HADOOP_HOME=/usr/local/Cellar/hadoop/3.0.0
export HIVE_CONF_DIR=/usr/local/Cellar/hive/2.3.1/libexec/conf 
export HIVE_AUX_JARS_PATH=/usr/local/Cellar/hive/2.3.1/libexec/lib 
```
4. 修改libexec下conf文件夹中hive-site.xml
``` xml
<property>
    <name>hive.querylog.location</name>
    <value>/usr/local/Cellar/hive/1.0.0/log</value>
    <description>Location of Hive run time structured log file</description>
</property>
<property>
    <name>hive.exec.scratchdir</name>
    <value>/tmp/hive-${user.name}</value>
    <description>HDFS root scratch dir for Hive jobs which gets created with write all (733) permission. For each connecting user, an HDFS scratch dir: $     {hive.exec.scratchdir}/&lt;username&gt; is created, with ${hive.scratch.dir.permission}.</description>
</property>
  <property>
    <name>hive.exec.local.scratchdir</name>
    <value>/tmp/${user.name}</value>
    <description>Local scratch space for Hive jobs</description>
  </property>
<property>
    <name>hive.downloaded.resources.dir</name>
    <value>/tmp/${user.name}_resources</value>
    <description>Temporary local directory for added resources in the remote file system.</description>
  </property>
<property>
    <name>hive.server2.logging.operation.log.location</name>
    <value>/tmp/${user.name}/operation_logs</value>
    <description>Top level directory where operation logs are stored if logging functionality is enabled</description>
  </property>
```

5. brew install mysql 安装 mysql
``` shell
$ mysql.server start  #启动mysql
$ mysql -uroot  #root用户进入mysql
mysql> CREATE DATABASE metastore;
mysql> USE metastore;
mysql> CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive';
mysql> GRANT SELECT,INSERT,UPDATE,DELETE,ALTER,CREATE ON metastore.* TO 'hive'@'localhost';
```
至此需要执行命令 schematool -dbType mysql -initSchema 但是会报错无法完成hive基于mysql的schemainitial 可以使用下面的方法
进入mysql shell 执行
``` shell
source /usr/local/Cellar/hive/3.1.1/libexec/scripts/metastore/upgrade/mysql/hive-schema-3.1.0.mysql.sql
```
最后执行hive 可以执行相应命令不出错 说明hive配置已经成功 
另外 要完全删除mysql保留的信息 需要删除 /usr/local/var/mysql 下的所有内容

## Hbase配置
进行hbase的伪分布式部署，首先 brew install hbase 安装最新版本的hbase
  
1. 修改 hbase-env.sh
``` shell
HBASE_MANAGE_ZK = false
```
2. 修改hbase-site.xml
``` xml
<configuration>                                                                      
  <property>                                                                         
    <name>hbase.rootdir</name>                                                       
    <value>hdfs://localhost:8020/hbase</value>                                       
  </property>                                                                        
··                                                                                   
  <property>                                                                         
    <name>hbase.cluster.distributed</name>                                           
  <value>true</value>                                                                
  </property>                                                                        
                                                                                     
  <property>                                                                         
        <name>hbase.zookeeper.quorum</name>                                          
        <value>localhost</value>                                                     
  </property>                                                                        
                                                                                     
  <property>                                                                         
        <name>zookeeper.znode.parent</name>                                          
        <value>/hbase-unsecure</value>                                               
  </property>                                                                        
                                                                                     
  <property>                                                                         
    <name>hbase.zookeeper.property.clientPort</name>                                 
    <value>2181</value>                                                              
  </property>                                                                        
  <property>                                                                         
    <name>hbase.zookeeper.property.dataDir</name>                                    
    <value>/usr/local/var/zookeeper</value>                                          
  </property>                                                                        
  <property>                                                                         
    <name>hbase.zookeeper.dns.interface</name>                                       
    <value>localhost</value>                                                         
  </property>                                                                        
  <property>                                                                         
    <name>hbase.regionserver.dns.interface</name>                                    
    <value>localhost</value>                                                         
  </property>                                                                        
  <property>                                                                         
    <name>hbase.master.dns.interface</name>                                          
    <value>localhost</value>                                                         
  </property>                                                                        
                                                                                     
</configuration>  
```
hbase.rootdir路径一定要跟hadoop中core-site.xml中fs.default.name相同

3. 先启动hdfs，再启动hbase 启动成功HBase会在HDFS下创建/hbase目录 hdfs dfs -ls /hbase
执行 start-hbase.sh 启动hbase
注意: hbase服务依赖于已经启动好的zookeeper服务(因为hbase在单机上以伪分布式运行 所以要配置 zookeeper.znode.parent) 需要提前在本地用brew安装好zookeeper并执行 brew services start zookeeper 启动zookeeper服务
遇到的一个问题是: The node /hbase-unsecure is not in ZooKeeper. It should have been written by the master. Check the value configured in 'zookeeper.znode.parent'. There could be a mismatch with the one configured in the master.
jps命令不能查看到 HMaster 进程
hbase 服务并不能认为启动成功





## JanusGraph图形数据库搭建
JanusGraph通过原生支持TinkerPop来提供图数据库(OLTP)和图分析系统 (OLAP)的能力。下图包括了TinkerPop的几个组成部分。最上层是Gremlin Server，为用户提供了访问图数据库的通道，通过脚本可以连上Gremlin Server，使用Gremlin查询/遍历语言来操作JanusGraph等支持Gremlin的图数据库，Gremlin语言支持Cypher语法；再下层是TinkerPop提供的用于表示属性图的核心API，包括图、定点和边的各种操作接口；Graph Computer是TinkPop的图计算框架，与Spark、Giraph或Hadoop等大数据平台配合并通过调用JanusGraph提供的OLAP I/O接口实现图分析和处理能力。JanusGraph等图数据库和图分析系统通过实现Provider API来对接TinkPop框架，定制所需的功能。
1. 可插拔的索引后端 支持多种搜索引擎作为索引后端 索引后端和存储后端并存，但两者相互独立，且索引后端不是必须的。目前已支持使用索引后端进行地理位置、数字范围和全文检索等查询类型。所支持的查询引擎包括
 ES, Apache Solr, Apache Lucence
2. 可集成的大数据分析能力 支持Apache Spark, Apache Giraph, Apache Hadoop
3. 便捷的可视化工具


###  JanusGraph本地配置过程








