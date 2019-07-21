# 大数据 —— `Flume`

[toc]

---

## `Flume`简介

`Flume`是`Cloudera`提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统，`Flume`支持在日志系统中定制各类数据发送方，用于收集数据，它提供了从`console`（控制台）、`RPC（Thrift-RPC）`、`text`（文件）、`tail（UNIX tail）`、`syslog`（`syslog`日志系统），支持`TCP`和`UDP`2种模式），`exec`（命令执行）等数据源上收集数据的能力；同时，`Flume`提供对数据进行简单处理，并写到各种数据接受方（可定制，如`HDFS`，`HBase`等）的能力。

## `Flume`结构

`Agent`主要由:`source`，`channel`，`sink`三个组件组成：
**（1）、`Source`：**
从数据发生器接收数据,并将接收的数据以`Flume`的`event`格式传递给一个或者多个通道`channel`，`Flume`提供多种数据接收的方式,比如`Avro`，`Thrift`，`twitter1%`等。
**（2）、`Channel`：**
`channel`是一种短暂的存储容器，它将从`source`处接收到的`event`格式的数据缓存起来，直到它们被`sinks`消费掉，它在`source`和`sink`间起着一共桥梁的作用，`channal`是一个完整的事务，这一点保证了数据在收发的时候的一致性。并且它可以和任意数量的`source`和`sink`链接。支持的类型有：`JDBC channel`，`File System channel`， `Memort channel`等。
**（3）、`sink`：**
`sink`将数据存储到集中存储器比如`Hbase`和`HDFS`，它从`channals`消费数据(`events`)并将其传递给目标地。目标地可能是另一个`sink`，也可能`HDFS`，`HBase`。

## `Flume`安装

（1）、将下载的`flume`包，解压到指定目录中。
（2）、将`flume/conf/`目录中的`flume-env.sh.template`文件重命名为`flume-env.sh`或者拷贝一份再重命名，然后配置`flume-env.sh`文件，将其中的`JAVA_HOME`变量设置为自己的JDK安装目录。
（3）、配置环境变量，修改`~/.bashrc`文件（可选，如果不配置，那么每次执行命令都要进入`flume/bin`目录，很不方便）：
```shell
export FLUME_HOME=(flume安装位置)
export PATH=$PATH:$FLUME_HOME/bin
```
（4）、输入`flume-ng version`命令，出现版本信息即安装成功。

## `Flume`常用命令

> * `help`：帮助命令，`flume-ng help`。
> * `agent`：启动一个`agent`。
> * `--conf，-c`：指定配置文件，一般用`flume/conf/`。
> * `--name，-n`：给当前`agent`起名字，要和自定义配置文件中起的名字一致，一般就用`a1`。
> * `--conf-file，-f`：指定自己写的`conf`文件。
> * `-Dflume.root.logger=INFO,console`：将运行日志输出到控制台。

完整的运行命令：`flume-ng agent --conf flume/conf/ --name a1 --conf-file data/flume/flume-2-kafka.conf -Dflume.root.logger=INFO,console`
或者：`flume-ng agent -c flume/conf/ -n a1 -f data/flume/flume-2-kafka.conf -Dflume.root.logger=INFO,console`

## `Flume`实例

### 1. 监控一个文件,实时采集新增的数据输出到控制台

`agent`选型：`exec source` + `memory channel` + `logger sink`。

利用`tail -F`命令监控文件。

编写配置文件`flume-file-console.conf`：

```shell
# 此处agent命名为a1，那么执行文件的时候命令也要写a1
#Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /home/au/software/data/flume/test.log # 此处为需要监控的文件
# 命令从-c后的字符串读取
a1.sources.r1.shell = /bin/sh -c

# Describe the sink
a1.sinks.k1.type = logger

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1


```
启动`agent`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f flume-file-console.conf -Dflume.root.logger=INFO,console`

测试，往`test.log`文件写入内容（`echo "hello" >> test.log`），控制台会打印该内容：
```shell
# 控制台打印的内容：
2019-07-21 09:25:50,680 (SinkRunner-PollingRunner-DefaultSinkProcessor) 
[INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] 
Event: { headers:{} body: 68 65 6C 6C 6F                                  hello }

```


### 2. 从指定网络端口采集数据单行数据输出到控制台

`agent`选型：`netcat source` + `memory channel` + `logger sink`。

编写配置文件`flume-telnet-logger.conf`：

```shell
#Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost # 绑定ip
a1.sources.r1.port = 44444 # 监听的端口

#Describe the sink
a1.sinks.k1.type = logger

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1

```

启动`agent`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f flume-telnet-logger.conf -Dflume.root.logger=INFO,console`

测试：`telnet localhost 44444`，然后输入任意字符即可。

```shell
# 控制台打印的内容：
2019-07-21 09:38:34,496 (SinkRunner-PollingRunner-DefaultSinkProcessor)
[INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)]
Event: { headers:{} body: 68 65 6C 6C 6F 0D                               hello. }

```


### 3. 监控一个文件,实时采集新增的数据输出到`Kafka`

`agent`选型：`exec source` + `memory channel` + `KafkaSink sink`。

编写配置文件`flume-2-kafka.conf`：

```shell
a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /home/au/software/data/flume/test.log # 此处为需要监控的文件

a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
a1.channels.c1.byteCapacityBufferPercentage = 20
a1.channels.c1.byteCapacity = 800000

a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.k1.kafka.bootstrap.servers = hadoopPD:9092 # kafka服务器地址
a1.sinks.k1.kafka.topic = ct
a1.sinks.k1.kafka.flumeBatchSize = 20
a1.sinks.k1.kafka.producer.acks = 1

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1

```

启动`agent`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f flume-2-kafka.conf`

接下来就是`Kafka`的消费了。


### 4. 监听TCP的端口，实时采集新增的数据输出到控制台

`agent`选型：`syslogtcp source` + `memory channel` + `logger sink`。

编写配置文件`flume-syslogtcp-logger.conf`：

```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1
# Describe/configure the source
a1.sources.r1.type = syslogtcp
a1.sources.r1.port = 5140 #指定端口
a1.sources.r1.host = 0.0.0.0 #指定ip
# Describe the sink
a1.sinks.k1.type = logger
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

启动`agent`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f flume-syslogtcp-logger.conf -Dflume.root.logger=INFO,console`

测试产生`syslog`：`echo "hello world" | nc localhost 5140`

```shell
# 控制台打印的内容：
2019-07-21 09:56:58,495 (SinkRunner-PollingRunner-DefaultSinkProcessor) 
[INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] 
Event: { headers:{Severity=0, Facility=0, flume.syslog.status=Invalid} 
body: 68 65 6C 6C 6F 20 77 6F 72 6C 64                hello world }

```


### 5. 监控一个文件,实时采集新增的数据输出到`hdfs`

`agent`选型：`exec source` + `memory channel` + `hdfs sink`。

编写配置文件`flume-file-hdfs.conf`：

```shell
# Name the components on this agent
a1.sinks = k1
a1.sources = r1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /home/au/software/data/flume/test.log
a1.sources.r1.shell = /bin/bash -c

# Describe the sink
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = hdfs://hadoopPD:9000/flume/%Y%m%d%H # 改为自己的hdfs集群
a1.sinks.k1.hdfs.filePrefix = log
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 1
a1.sinks.k1.hdfs.roundUnit = hour
a1.sinks.k1.hdfs.useLocalTimeStamp = true
a1.sinks.k1.hdfs.batchSize = 1000
a1.sinks.k1.hdfs.fileType = DataStream
a1.sinks.k1.hdfs.rollInterval = 600
a1.sinks.k1.hdfs.rollSize = 134217700
a1.sinks.k1.hdfs.rollCount = 0
a1.sinks.k1.hdfs.minBlockReplicas = 1

#Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

启动`agent`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f flume-file-hdfs.conf`

测试，往`test.log`文件写入内容：`echo "hello" >> test.log`
查询`hdfs`文件系统的数据：`hdfs dfs -ls /flume`


### 6. 将A端服务器日志，实时采集到B端服务器

`agent`选型：
`A`：`exec source` + `memory channel` + `avro sink`
`B`：`avro source` + `memory channel` + `logger sink`

`A`端服务器编写配置文件`flume-file-avro.conf`：

```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1

a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /home/au/software/data/flume/test.log
a1.sources.r1.shell = /bin/sh -c

a1.sinks.k1.type = avro
a1.sinks.k1.hostname = localhost
a1.sinks.k1.port = 44444

a1.channels.c1.type = memory

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

`B`端服务器编写配置文件`flume-avro-logger.conf`：

```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1

a1.sources.r1.type = avro
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 44444

a1.sinks.k1.type = logger

a1.channels.c1.type = memory

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

启动`agent`：
`A`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f flume-file-avro.conf -Dflume.root.logger=INFO,console`
`B`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f flume-avro-logger.conf -Dflume.root.logger=INFO,console`

测试：在`A`服务器输出数据给`test.log`，在`B`服务器控制台可看到结果。


### 7. 故障转移（`failover`）

`Sinks`组允许用户将多个`Sink`分到一个组中。`Sink Processor`可用于在组内的所有`Sink`上提供负载平衡功能，或在发生暂时性故障时实现从一个`Sink`到另一个`Sink`的故障转移。

需要实现的效果：
一台机器一直发送数据给其中一个`sink`，当这个`sink`不可用的时候，自动发送到下一个`sink`。

**步骤：**
**1. 在`server1`创建`Flume_Sink_Processors.conf`配置文件：**
```shell
#这个是配置failover的关键，需要有一个sink group
a1.sinkgroups = g1
a1.sinkgroups.g1.sinks = k1 k2
#处理的类型是failover
a1.sinkgroups.g1.processor.type = failover
#优先级，数字越大优先级越高，每个sink的优先级必须不相同
a1.sinkgroups.g1.processor.priority.k1 = 5
a1.sinkgroups.g1.processor.priority.k2 = 10
#设置为10秒，当然可以根据你的实际状况更改成更快或者很慢
a1.sinkgroups.g1.processor.maxpenalty = 10000
  
# Describe/configure the source
a1.sources.r1.type = syslogtcp
a1.sources.r1.port = 5140
a1.sources.r1.channels = c1 c2
a1.sources.r1.selector.type = replicating

# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.channel = c1
a1.sinks.k1.hostname = server2
a1.sinks.k1.port = 5555
 
a1.sinks.k2.type = avro
a1.sinks.k2.channel = c2
a1.sinks.k2.hostname = server3
a1.sinks.k2.port = 5555
  
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
  
a1.channels.c2.type = memory
a1.channels.c2.capacity = 1000
a1.channels.c2.transactionCapacity = 100
```

**2. 在`server2`创建`Flume_Sink_Processors_avro.conf`配置文件：**
```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1
  
# Describe/configure the source
a1.sources.r1.type = avro
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 5555
  
# Describe the sink
a1.sinks.k1.type = logger
  
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
  
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

**3. 在`server3`创建`Flume_Sink_Processors_avro.conf`配置文件：**
```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1
  
# Describe/configure the source
a1.sources.r1.type = avro
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 5555
  
# Describe the sink
a1.sinks.k1.type = logger
  
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
  
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

**4. 启动`agent`：**
分别在`server1`，`server2`，`server3`启动上述`agent`：
`server1`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f flume-file-avro.conf -Dflume.root.logger=INFO,console`
`server2`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f Flume_Sink_Processors_avro.conf -Dflume.root.logger=INFO,console`
`server3`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f Flume_Sink_Processors_avro.conf -Dflume.root.logger=INFO,console`

**5. 在`server1`机器上，测试产生`log`：**
`echo "test" | nc localhost 5140`

**6. 因为server3的优先级高于server2，所以在server3的sink窗口，可以看到`server1`的`log`信息，而server2没有。**

**7. 我们停止掉`server3`机器上的`sink(ctrl+c)`，再次输出测试数据（`echo "test" | nc localhost 5140`），在server2的sink窗口，可以看到`server1`的`log`信息。**

**8. 再在`server3`的`sink`窗口中，启动`sink`：**
`server3`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f Flume_Sink_Processors_avro.conf -Dflume.root.logger=INFO,console`

**9. 因为优先级的关系，`log`消息会再次落到`server3`上：**

### 8. 负载均衡（`load balance`）

`load balance`和`failover`不同的地方是，`load balance`有两个配置，一个是轮询，一个是随机。两种情况下如果被选择的`sink`不可用，就会自动尝试发送到下一个可用的`sink`上面。

**步骤：**
**1. 在`server1`创建`Load_balancing_Sink_Processors.conf`配置文件：**
```shell
a1.sources = r1
a1.sinks = k1 k2
a1.channels = c1 c2
  
#这个是配置Load balancing的关键，需要有一个sink group
a1.sinkgroups = g1
a1.sinkgroups.g1.sinks = k1 k2
a1.sinkgroups.g1.processor.type = load_balance
a1.sinkgroups.g1.processor.backoff = true
a1.sinkgroups.g1.processor.selector = round_robin
  
# Describe/configure the source
a1.sources.r1.type = syslogtcp
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 5140
a1.sources.r1.channels = c1
  
  
# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.channel = c1
a1.sinks.k1.hostname = localhost
a1.sinks.k1.port = 5555
  
a1.sinks.k2.type = avro
a1.sinks.k2.channel = c2
a1.sinks.k2.hostname = localhost
a1.sinks.k2.port = 6666
  
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

a1.channels.c2.type = memory
```
**2. 在`server2`创建`Load_balancing_Sink_Processors_avro.conf`配置文件：**

```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1
  
# Describe/configure the source
a1.sources.r1.type = avro
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 5555
  
# Describe the sink
a1.sinks.k1.type = logger
  
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
  
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```
**3. 在`server3`创建`Load_balancing_Sink_Processors_avro.conf`配置文件：**
```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1
  
# Describe/configure the source
a1.sources.r1.type = avro
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 5555
  
# Describe the sink
a1.sinks.k1.type = logger
  
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
  
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

**4. 启动`agent`：**
分别在`server1`，`server2`，`server3`启动上述`agent`：
`server1`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f Load_balancing_Sink_Processors.conf -Dflume.root.logger=INFO,console`
`server2`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f Load_balancing_Sink_Processors_avro.conf -Dflume.root.logger=INFO,console`
`server3`：`flume-ng agent -c ~/software/flume/conf/ -n a1 -f Load_balancing_Sink_Processors_avro.conf -Dflume.root.logger=INFO,console`

**5. 在`server1`机器上，测试产生`log`：**
`echo "test" | nc localhost 5140`

**6. 可以看到`server2`和`server3`接连的的接受`log`信息，说明轮询模式起到了作用。**