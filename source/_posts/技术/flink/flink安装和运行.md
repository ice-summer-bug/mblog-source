---
layout: post
title: Flink 安装和运行
categories: flink
tags: flink
date: 2018-11-19 22:00:00
description: apache flink 安装运行
---

# `Flink` 安装和运行
## 单点安装

### 下载 flink 安装包

在 [官网下载地址](https://flink.apache.org/downloads.html "官网下载地址") 选择适当版本的flink 安装包，这里我选择 `Apache Flink 1.6.2 only`；下载安装包到 `/path/to/flink-1.6.2` 中

### 解压运行 flink

```bash
cd /path/to/
tar -xvf flink-1.6.2.tgz
cd flink-1.6.2
sh ./bin/start-cluster.sh
```

到这里我们就已经把 `单机版flink` 成功的运行起来了，我们可以访问 `localhost:8081` 看到 `flink` 管理页面

![图片](/assets/picture/flink.single.png "单机版flink运行")

## `flink` 集群搭建准备工作

### JAVA 环境配置

***这里不赘述如何配置 JAVA 环境，只需要注意使用 JAVA 1.8+ 即可***

### 集群机器互信

不管是 `standalone` 部署模式还是依赖 `hadoop yarn` 搭建集群，都需要在集群机器之间设置互信，实现 `ssh` 相互免密登录

1. 生成`公钥/秘钥`

```bash
ssh-keygen -t rsa  #一直回车即可
```
生成了 `~/.ssh/id_rsa.pub` 和 `~/.ssh/id_rsa`

2. 公钥认证

将机器A 上面的 `~/.ssh/id_rsa.pub` 追加到机器B 的公钥认证文件 `~/.ssh/authorized_keys` 里面去；
再将机器B 上面的 `~/.ssh/id_rsa.pub` 追加到机器A 的公钥认证文件 `~/.ssh/authorized_keys` 里面去

这样我们就可以在机器A、B之间互相免密码登陆了

## `flink on yarn` 集群搭建

在 `10.0.0.1`，`10.0.0.2`，`10.0.0.3` 三台机器上尝试搭建 `flink` 集群

### `hadoop yarn` 安装配置

#### 下载解压 `hadoop` 安装包

这里我用的 `cdh` 版本的 `hadoop`，可以在[这里](http://archive-primary.cloudera.com/cdh5/cdh/5/ "hadoop cdh")下载，然后解压

```bash
tar -xvf hadoop-2.6.0-cdh5.11.0.tar.gz
```

#### 配置 `hadoop`

在 `/path/to/hadoop-2.6.0-cdh5.11.0/etc/hadoop/` 文件夹下配置如下七个文件 `hadoop-env.sh`，`yarn-env.sh`，`slaves`，`core-site.xml`，`hdfs-site.xml`，`mapred-site.xml`，`yarn-site.xml`

##### 在 `hadoop-env.sh`

```bash
export JAVA_HOME=/path/to/java/home
```

##### 在 `yarn-env.sh` 中配置 `JAVA_HOME`

```bash
export JAVA_HOME=/path/to/java/home
```

##### 在 `slaves` 中配置 slave 节点的ip 或者host

```bash
10.0.0.2
10.0.0.3
```

##### 修改 `core-site.xml`

配置 `hadoop` 集群文件系统主机和端口、`hadoop` 临时目录

```xml
<configuration>
  <property>
    <!-- hadoop文件系统主机和端口 -->
    <name>fs.defaultFS</name>
    <value>hdfs://10.0.0.1:9000/</value>
  </property>
  <property>
    <!-- 配置 hadoop 临时目录 -->
    <name>hadoop.tmp.dir</name>
    <value>file:/path/to/hadoop-2.6.0-cdh5.11.0/tmp</value>
  </property>
</configuration>
```

##### 修改 `hdfs-site.xml`

配置 `hadoop` 集群文件系统主机和端口、`hadoop` 临时目录

```xml
<configuration>
  <property>
    <!-- hadoop文件系统主机和端口 -->
    <name>fs.defaultFS</name>
    <value>hdfs://10.0.0.1:9000/</value>
  </property>
  <property>
    <!-- 配置 hadoop 临时目录 -->
    <name>hadoop.tmp.dir</name>
    <value>file:/path/to/hadoop-2.6.0-cdh5.11.0/tmp</value>
  </property>
</configuration>
```

## `standalone` 部署模式集群搭建

举例说明，搭建

### 下载、解压安装包

### 修改 `flink-conf.yaml` 文件配置

修改 `/path/to/flink-1.6.2/conf` 文件中的 `flink-conf.yaml` 文件，参数说明如下：

```yaml
# java安装路径，如果没有指定则默认使用系统的$JAVA_HOME环境变量。建议设置此值，因为之前我曾经在standalone模式中启动flink集群，报找不到JAVA_HOME的错误。config.sh中（Please specify JAVA_HOME. Either in Flink config ./conf/flink-conf.yaml or as system-wide JAVA_HOME.）
env.java.home: /path/to/java/home

# 定制JVM选项，在Flink启动脚本中执行。需要单独执行JobManager和TaskManager的选项。
env.java.opts: -DXms=1024MB

# 执行jobManager的JVM选项。在Yarn Client环境下此参数无效。
env.java.opts.jobmanager:

# 执行taskManager的JVM选项。在Yarn Client环境下此参数无效。
env.java.opts.taskmanager:

# Jobmanager的IP地址，即master地址。默认是localhost，此参数在HA环境下或者Yarn下无效，仅在local和无HA的standalone集群中有效。
jobmanager.rpc.address: localhost

# JobMamanger的端口，默认是6123。
jobmanager.rpc.port: 6123

# JobManager的堆大小（单位是MB）。当长时间运行operator非常多的程序时，需要增加此值。具体设置多少只能通过测试不断调整。
jobmanager.heap.mb: 1024

# 每一个TaskManager的堆大小（单位是MB），由于每个taskmanager要运行operator的各种函数（Map、Reduce、CoGroup等，包含sorting、hashing、caching），因此这个值应该尽可能的大。如果集群仅仅跑Flink的程序，建议此值等于机器的内存大小减去1、2G，剩余的1、2GB用于操作系统。如果是Yarn模式，这个值通过指定tm参数来分配给container，同样要减去操作系统可以容忍的大小（1、2GB）。
taskmanager.heap.mb: 8196

# 每个TaskManager的并行度。一个slot对应一个core，默认值是1.一个并行度对应一个线程。总的内存大小要且分给不同的线程使用。
taskmanager.numberOfTaskSlots: 5

# 每个operator的默认并行度。默认是1.如果程序中对operator设置了setParallelism，或者提交程序时指定了-p参数，则会覆盖此参数。如果只有一个Job运行时，此值可以设置为taskManager的数量 * 每个taskManager的slots数量。即NumTaskManagers  * NumSlotsPerTaskManager 。
parallelism.default: 3

# 设置默认的文件系统模式。默认值是file:///即本地文件系统根目录。如果指定了hdfs://localhost:9000/，则程序中指定的文件/user/USERNAME/in.txt，即指向了hdfs://localhost:9000/user/USERNAME/in.txt。这个值仅仅当没有其他schema被指定时生效。一般hadoop中core-site.xml中都会配置fs.default.name。
fs.default-scheme: hdfs://localhost:9000/

# HDFS的配置路径。例如：/home/flink/hadoop/hadoop-2.6.0/etc/hadoop。如果配置了这个值，用户程序中就可以简写hdfs路径如：hdfs:///path/to/files。而不用写成：hdfs://address:port/path/to/files这种格式。配置此参数后，Flink就可以找到此路径下的core-site.xml和hdfs-site.xml了。建议配置此参数。
fs.hdfs.hadoopconf: /home/flink/hadoop/hadoop-2.6.0/etc/hadoop

# flink 服务的 http 端口
rest.port: 8081
```

## flink on Yarn

### 启动 yarn session 运行 flink job

```bash
bin/yarn-session.sh -n 4 -jm 1024 -tm 4096 -s 32
```

#### `yarn-session.sh` 使用说明

```shell
Usage:
   Required
     -n,--container <arg>   Number of YARN container to allocate (=Number of Task Managers)
   Optional
     -D <property=value>             use value for given property
     -d,--detached                   If present, runs the job in detached mode
     -h,--help                       Help for the Yarn session CLI.
     -id,--applicationId <arg>       Attach to running YARN session
     -j,--jar <arg>                  Path to Flink jar file
     -jm,--jobManagerMemory <arg>    Memory for JobManager Container with optional unit (default: MB)
     -m,--jobmanager <arg>           Address of the JobManager (master) to which to connect. Use this flag to connect to a different JobManager than the one specified in the configuration.
     -n,--container <arg>            Number of YARN container to allocate (=Number of Task Managers)
     -nl,--nodeLabel <arg>           Specify YARN node label for the YARN application
     -nm,--name <arg>                Set a custom name for the application on YARN
     -q,--query                      Display available YARN resources (memory, cores)
     -qu,--queue <arg>               Specify YARN queue.
     -s,--slots <arg>                Number of slots per TaskManager
     -sae,--shutdownOnAttachedExit   If the job is submitted in attached mode, perform a best-effort cluster shutdown when the CLI is terminated abruptly, e.g., in response to a user interrupt, such
                                     as typing Ctrl + C.
     -st,--streaming                 Start Flink in streaming mode
     -t,--ship <arg>                 Ship files in the specified directory (t for transfer)
     -tm,--taskManagerMemory <arg>   Memory per TaskManager Container with optional unit (default: MB)
     -yd,--yarndetached              If present, runs the job in detached mode (deprecated; use non-YARN specific option instead)
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths for high availability mode
```

#### `yarn` 会话管理

```bash
$yarn application --help
usage: application
 -appStates <States>             Works with -list to filter applications
                                 based on input comma-separated list of
                                 application states. The valid application
                                 state can be one of the following:
                                 ALL,NEW,NEW_SAVING,SUBMITTED,ACCEPTED,RUN
                                 NING,FINISHED,FAILED,KILLED
 -appTypes <Types>               Works with -list to filter applications
                                 based on input comma-separated list of
                                 application types.
 -help                           Displays help for all commands.
 -kill <Application ID>          Kills the application.
 -list                           List applications. Supports optional use
                                 of -appTypes to filter applications based
                                 on application type, and -appStates to
                                 filter applications based on application
                                 state.
 -movetoqueue <Application ID>   Moves the application to a different
                                 queue.
 -queue <Queue Name>             Works with the movetoqueue command to
                                 specify which queue to move an
                                 application to.
 -status <Application ID>        Prints the status of the application.
```

### 直接提交 flink job 到 yarn 集群

```bash
bin/flink run -m yarn-cluster -yn 2 -yjm 1024 -ytm 1024 ./examples/batch/WordCount.jar
```

#### `yarn` 日常维护

##### 查看 `yarn session` 中的 flink job

```
./bin/flink list -m yarn-cluster -yid <Yarn Application Id> -r  
```

##### `yarn` 日志查看

```bash
yarn logs -applicationId <applicationId>
```

##### `yarn application` 下线

```bash
yarn application -kill <applicationId>
```

##### 触发 `savepoints` 取消 flink job

```bash
./bin/flink cancel -s [savepointDirectory] <jobID>
```

执行上述指令将得到如下提示

```
Cancelling job <jobID> with savepoint to <savepointDirectory>.
Cancelled job <jobID>. Savepoint stored in <savepointDirectory>/<savepointID>.
```

##### 从 `savepoints` 恢复启动 flink job

```bash
./bin/flink run -s <savepointDirectory>/<savepointID> -m yarn-cluster -yn 2 -yjm 1024 -ytm 1024 ./examples/batch/WordCount.jar
```
***这里 `-s` 参数值是执行`cancel` 指令的时候得到的 savepoint 保存地址<savepointDirectory\>/<savepointID\>***


#### `flink run` 指令

```bash
$./bin/flink run --help

Action "run" compiles and runs a program.

  Syntax: run [OPTIONS] <jar-file> <arguments>
  "run" action options:
     -c,--class <classname>               当 flink job 的 jar包中没有指定 mainfest 时，通过这个                                   
                                          参数来指定包含 main() 方法，或者 getPlan() 方法的主类。
     -C,--classpath <url>                 新增 flink 类加载器的
                                          Adds a URL to each user code
                                          classloader  on all nodes in the
                                          cluster. The paths must specify a
                                          protocol (e.g. file://) and be
                                          accessible on all nodes (e.g. by means
                                          of a NFS share). You can use this
                                          option multiple times for specifying
                                          more than one URL. The protocol must
                                          be supported by the {@link
                                          java.net.URLClassLoader}.
     -d,--detached                        以后台模式运行 flink job
     -n,--allowNonRestoredState           Allow to skip savepoint state that
                                          cannot be restored. You need to allow
                                          this if you removed an operator from
                                          your program that was part of the
                                          program when the savepoint was
                                          triggered.
     -p,--parallelism <parallelism>       The parallelism with which to run the
                                          program. Optional flag to override the
                                          default value specified in the
                                          configuration.
     -q,--sysoutLogging                   If present, suppress logging output to
                                          standard out.
     -s,--fromSavepoint <savepointPath>   Path to a savepoint to restore the job
                                          from (for example
                                          hdfs:///flink/savepoint-1537).
     -sae,--shutdownOnAttachedExit        If the job is submitted in attached
                                          mode, perform a best-effort cluster
                                          shutdown when the CLI is terminated
                                          abruptly, e.g., in response to a user
                                          interrupt, such as typing Ctrl + C.
  Options for yarn-cluster mode:
     -d,--detached                        If present, runs the job in detached
                                          mode
     -m,--jobmanager <arg>                Address of the JobManager (master) to
                                          which to connect. Use this flag to
                                          connect to a different JobManager than
                                          the one specified in the
                                          configuration.
     -sae,--shutdownOnAttachedExit        If the job is submitted in attached
                                          mode, perform a best-effort cluster
                                          shutdown when the CLI is terminated
                                          abruptly, e.g., in response to a user
                                          interrupt, such as typing Ctrl + C.
     -yD <property=value>                 use value for given property
     -yd,--yarndetached                   If present, runs the job in detached
                                          mode (deprecated; use non-YARN
                                          specific option instead)
     -yh,--yarnhelp                       Help for the Yarn session CLI.
     -yid,--yarnapplicationId <arg>       Attach to running YARN session
     -yj,--yarnjar <arg>                  Path to Flink jar file
     -yjm,--yarnjobManagerMemory <arg>    JobManager Container 的内存大小，默认单位是 MB
     -yn,--yarncontainer <arg>            指定分配的 yarn container 的数量，等同于 flink Task Managers 的数量
     -ynl,--yarnnodeLabel <arg>           Specify YARN node label for the YARN
                                          application
     -ynm,--yarnname <arg>                Set a custom name for the application
                                          on YARN
     -yq,--yarnquery                      Display available YARN resources
                                          (memory, cores)
     -yqu,--yarnqueue <arg>               Specify YARN queue.
     -ys,--yarnslots <arg>                Number of slots per TaskManager
     -yst,--yarnstreaming                 Start Flink in streaming mode
     -yt,--yarnship <arg>                 Ship files in the specified directory
                                          (t for transfer)
     -ytm,--yarntaskManagerMemory <arg>   单个 taskManager 的内存大小，默认单位是 MB
     -yz,--yarnzookeeperNamespace <arg>   Namespace to create the Zookeeper
                                          sub-paths for high availability mode
     -z,--zookeeperNamespace <arg>        Namespace to create the Zookeeper
                                          sub-paths for high availability mode

  Options for default mode:
     -m,--jobmanager <arg>           Address of the JobManager (master) to which
                                     to connect. Use this flag to connect to a
                                     different JobManager than the one specified
                                     in the configuration.
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths
                                     for high availability mode
```
