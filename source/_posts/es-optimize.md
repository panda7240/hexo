layout: photo
title: Elasticsearch使用积累(持续更新...)
tags:
  - Elasticsearch
  - 原创
  - 分布式
photos:
date: 2015-09-17 11:15:01
categories: Elasticsearch
---

## 写在前面
接触Elasticsearch已经有几年的时间, 从一开始在自己的普通台式机上搭建单实例来简单记录测试环境的日志, 到现在生产上用9个Elasticsearch实例搭成的集群来记录商城后台服务的全量日志. 使用场景没变,都是用了记录日志, 并做实时检索和统计. 但是随着每天记录的数据量快速递增,QPS快速递增,我们经常会碰到一些性能问题,这些问题一直在驱动我们对Elasticsearch集群进行持续优化, 这里是有必要记录一下我们在使用ES(后续文章中用ES代替Elasticsearch)的过程中积累的经验.
<!--more-->


## 常用插件

1. [Head](https://github.com/mobz/elasticsearch-head)
	查看分片情况,操作简单api
2. [Bigdesk](https://github.com/lukas-vlcek/bigdesk)
   监控所在机器的CPU,IO,JVM等指标,简单分片概览
3. [KOPF](https://github.com/lmenezes/elasticsearch-kopf) 
   查看集群gc回收磁盘性能, 分片情况, 简单操作api, 感觉该插件较Head更实用一些
4. [Sql](https://github.com/NLPchina/elasticsearch-sql)
	可以通过sql进行聚合检索, 可以将sql语句翻译成ES的JSON检索语句
	
	
## ES集群优雅停止,启动

在一开始使用ES的时候, 都是通过 `kill <pid>` (不是Kill -9)来关闭ES实例. 但是每回重启后, 都会发现有很长时间的分片同步(即使没有手动删除数据等操作). 后来发现ES默认是开启自动分片均衡的. 那么如果想要我们在停止,启动某个ES实例后, 可以快速将集群状态变更为Green. 我们最好可以采用如下步骤进行:

1. **暂停集群的分片自动均衡(在集群中任一台ES实例上执行都可以,只需要执行一次)**

	```
	curl -XPUT http://127.0.0.1:9200/_cluster/settings -d' 
	{ 
		"transient" : { 
			"cluster.routing.allocation.enable" : "none" 
		} 
	}'
	```
2. **优雅停止你要升级或检测的ES实例(不到万不得已,绝对不要用Kill -9)**

	```
	curl -XPOST http://127.0.0.1:9200/_cluster/nodes/_local/_shutdown
	```

3. **升级重启该节点，并确认该节点重新加入到了集群中**
4. **重复上面的2,3步，操作其他集群内其他ES实例**
5. **重新启动集群的分片自动均衡(在集群中任一ES实例中执行一次即可)**

	```
	curl -XPUT http://127.0.0.1:9200/_cluster/settings -d' 
	{ 
		"transient" : { 
			"cluster.routing.allocation.enable" : "all" 
		} 
	}'
	```


## ES集群JVM调优积累

在最开始, 我们使用的ES实例,没有涉及到很大的数据写入和检索(每天几百万数据). 当时我们配置ES的JVM(Xms=Xmx=8G)的垃圾回收器主要是CMS,具体配置如下:

```
# reduce the per-thread stack size
JAVA_OPTS="$JAVA_OPTS -Xss256k"

JAVA_OPTS="$JAVA_OPTS -XX:+UseParNewGC"
JAVA_OPTS="$JAVA_OPTS -XX:+UseConcMarkSweepGC"

JAVA_OPTS="$JAVA_OPTS -XX:CMSInitiatingOccupancyFraction=75"
JAVA_OPTS="$JAVA_OPTS -XX:+UseCMSInitiatingOccupancyOnly"
```

这个在小数据量,小内存时运行良好, 基本很少会出现Full gc.

后来我们的数据量为了一天1亿多, 为了加快检索, 我们将Xms,Xmx同时调整为32G. 并按天进行索引,后来发现随着索引数增加,写入数据量增加, Full GC已经影响到了我们的写入和检索(这个时候我们的ES集群是三个实例).我们决定将G1作为垃圾回收期,但是[官方并不推荐使用G1](https://www.elastic.co/guide/en/elasticsearch/guide/current/_don_8217_t_touch_these_settings.html#_garbage_collector), 我们还是想尝试一下换成G1, 并借此机会把JDK从1.7升级到1.8(升级很平滑), G1的具体配置如下: 

```
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC "
#init_globals()末尾打印日志
JAVA_OPTS="$JAVA_OPTS -XX:+PrintFlagsFinal "
#打印gc引用
JAVA_OPTS="$JAVA_OPTS -XX:+PrintReferenceGC "
#输出虚拟机中GC的详细情况.
JAVA_OPTS="$JAVA_OPTS -verbose:gc "
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails "
#Enables printing of time stamps at every GC. By default, this option is disabled.
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCTimeStamps "
#Enables printing of information about adaptive generation sizing. By default, this option is disabled.
JAVA_OPTS="$JAVA_OPTS -XX:+PrintAdaptiveSizePolicy "
# unlocks diagnostic JVM options
JAVA_OPTS="$JAVA_OPTS -XX:+UnlockDiagnosticVMOptions "
#to measure where the time is spent
JAVA_OPTS="$JAVA_OPTS -XX:+G1SummarizeConcMark "
#设置触发标记周期的 Java 堆占用率阈值。默认占用率是整个 Java 堆的 45%。
#JAVA_OPTS="$JAVA_OPTS -XX:InitiatingHeapOccupancyPercent=45 "
```

> TODO 之后会在这补充替换为G1后,ES集群的表现

## ES集群健康说明
在Elasticsearch集群中可以监控统计很多信息，其中最重要的就是：集群健康(cluster health)。它的 status 有 `green`、`yellow`、`red` 三种；

可以通过如下命令获取:

```
GET http://127.0.0.1:9200/_cluster/health
```

在一个没有索引的空集群中，它将返回如下信息：

```
{
   "cluster_name":          "elasticsearch",
   "status":                "green", <1>
   "timed_out":             false,
   "number_of_nodes":       1,
   "number_of_data_nodes":  1,
   "active_primary_shards": 0,
   "active_shards":         0,
   "relocating_shards":     0,
   "initializing_shards":   0,
   "unassigned_shards":     0
}
```

`status` 是我们最应该关注的字段。
`status` 可以告诉我们当前集群是否处于一个可用的状态。三种颜色分别代表：

**状态说明**
green	: 所有主分片和从分片都可用
yellow: 	所有主分片可用，但存在不可用的从分片
red:	存在不可用的主要分片


## ES配置文件详解

```
cluster.name: elasticsearch
#配置es的集群名称，默认是elasticsearch，es会自动发现在同一网段下的es，如果在同一网段下有多个集群，就可以用这个属性来区分不同的集群。

node.name: "Franz Kafka"
#节点名，默认随机指定一个name列表中名字，该列表在es的jar包中config文件夹里name.txt文件中，其中有很多作者添加的有趣名字。

node.master: true
#指定该节点是否有资格被选举成为node，默认是true，es是默认集群中的第一台机器为master，如果这台机挂了就会重新选举master。

node.data: true
#指定该节点是否存储索引数据，默认为true。

index.number_of_shards: 5
#设置默认索引分片个数，默认为5片。

index.number_of_replicas: 1
#设置默认索引副本个数，默认为1个副本。

path.conf: /path/to/conf
#设置配置文件的存储路径，默认是es根目录下的config文件夹。

path.data: /path/to/data
#设置索引数据的存储路径，默认是es根目录下的data文件夹，可以设置多个存储路径，用逗号隔开，例：
#path.data: /path/to/data1,/path/to/data2

path.work: /path/to/work
#设置临时文件的存储路径，默认是es根目录下的work文件夹。

path.logs: /path/to/logs
#设置日志文件的存储路径，默认是es根目录下的logs文件夹

path.plugins: /path/to/plugins
#设置插件的存放路径，默认是es根目录下的plugins文件夹

bootstrap.mlockall: true
#设置为true来锁住内存。因为当jvm开始swapping时es的效率会降低，所以要保证它不swap，可以把#ES_MIN_MEM和ES_MAX_MEM两个环境变量设置成同一个值，并且保证机器有足够的内存分配给es。同时也要#允许elasticsearch的进程可以锁住内存，linux下可以通过`ulimit -l unlimited`命令。

network.bind_host: 192.168.0.1
#设置绑定的ip地址，可以是ipv4或ipv6的，默认为0.0.0.0。


network.publish_host: 192.168.0.1
#设置其它节点和该节点交互的ip地址，如果不设置它会自动判断，值必须是个真实的ip地址。

network.host: 192.168.0.1
#这个参数是用来同时设置bind_host和publish_host上面两个参数。

transport.tcp.port: 9300
#设置节点间交互的tcp端口，默认是9300。

transport.tcp.compress: true
#设置是否压缩tcp传输时的数据，默认为false，不压缩。

http.port: 9200
#设置对外服务的http端口，默认为9200。

http.max_content_length: 100mb
#设置内容的最大容量，默认100mb

http.enabled: false
#是否使用http协议对外提供服务，默认为true，开启。

gateway.type: local
#gateway的类型，默认为local即为本地文件系统，可以设置为本地文件系统，分布式文件系统，hadoop的#HDFS，和amazon的s3服务器，其它文件系统的设置方法下次再详细说。

gateway.recover_after_nodes: 1
#设置集群中N个节点启动时进行数据恢复，默认为1。

gateway.recover_after_time: 5m
#设置初始化数据恢复进程的超时时间，默认是5分钟。

gateway.expected_nodes: 2
#设置这个集群中节点的数量，默认为2，一旦这N个节点启动，就会立即进行数据恢复。

cluster.routing.allocation.node_initial_primaries_recoveries: 4
#初始化数据恢复时，并发恢复线程的个数，默认为4。

cluster.routing.allocation.node_concurrent_recoveries: 2
#添加删除节点或负载均衡时并发恢复线程的个数，默认为4。

indices.recovery.max_size_per_sec: 0
#设置数据恢复时限制的带宽，如入100mb，默认为0，即无限制。

indices.recovery.concurrent_streams: 5
#设置这个参数来限制从其它分片恢复数据时最大同时打开并发流的个数，默认为5。

discovery.zen.minimum_master_nodes: 1
#设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。默认为1，对于大的集群来说，可以设置大一点的值（2-4）

discovery.zen.ping.timeout: 3s
#设置集群中自动发现其它节点时ping连接超时时间，默认为3秒，对于比较差的网络环境可以高点的值来防止自动发现时出错。

discovery.zen.ping.multicast.enabled: false
#设置是否打开多播发现节点，默认是true。

discovery.zen.ping.unicast.hosts: ["host1", "host2:port", "host3[portX-portY]"]
#设置集群中master节点的初始列表，可以通过这些节点来自动发现新加入集群的节点。
```

## ES使用中碰到的问题

1. **rejected execution (queue capacity 50) on org.elasticsearch.action.support.replication.TransportShar** 

	该问题需要在配置文件中添加Thread pool参数.
	
	```
	threadpool:
    #这是批量插入时需要做的配置
     bulk:
        type: fixed
        size: 100
        queue_size: 10000

    #检索时需要做的配置
    search:
        tyep: fixed
        size: 100
        queue_size: 10000
	```
	整个数据将会被处理它的节点载入内存中，所以如果请求量很大的话，留给其他请求的内存空间将会很少。bulk应该有一个最佳的限度。超过这个限制后，性能不但不会提升反而可能会造成宕机。

	最佳的容量并不是一个确定的数值，它取决于你的硬件，你的文档大小以及复杂性，你的索引以及搜索的负载。幸运的是，这个平衡点 很容易确定：

	试着去批量索引越来越多的文档。当性能开始下降的时候，就说明你的数据量太大了。一般比较好初始数量级是1000到5000个文档，或者你的文档很大，你就可以试着减小队列。 有的时候看看批量请求的物理大小是很有帮助的。1000个1KB的文档和1000个1MB的文档的差距将会是天差地别的。比较好的初始批量容量是5-15MB。
	
	
## 其他

1. **批量删除索引**	
	
	```
	#可用批量删除名字以xxx-log-2015.06开头的索引
	curl -XDELETE http://127.0.0.1:9201/xxx-log-2015.06*
	```
2. **保证最大限度的使用内存而不引起OutOfMemory**
	设置es的缓存类型为Soft Reference，它的主要特点是据有较强的引用功能。只有当内存不够的时候，才进行回收这类内存，因此在内存足够的时候，它们通常不被回收。另外，这些引 用对象还能保证在Java抛出OutOfMemory 异常之前，被设置为null。它可以用于实现一些常用图片的缓存，实现Cache的功能.在es的配置文件中做如下修改:

	```
	index.cache.field.type: soft
	```
3. **通过修改Template,缩小索引大小**
	在ES的Template中做如下调整
	
	```
	"_all" : { "enabled" : false }, //可以减小索引数量
	"pid": {
        //not_analyzed可以不分词, no:不索引, 去除index则会根据分词器进行分词并索引
	   	"index": "not_analyzed",
	    "type": "string"
	 },
	```
4. **加快调整分片和副本时的恢复进度**
	副本配置和分片配置不一样，是可以随时调整的。有些较大的索引，甚至可以在做 optimize 前，先把副本全部取消掉，等 optimize 完后，再重新开启副本，节约单个 segment 的重复归并消耗。
5. **ES的版本升级非常平滑,向前兼容做的不错**	
6. **index设置合理的刷新时间**
	建立的索引，不会立马查到，这是为什么elasticsearch为near-real-time的原因
需要配置index.refresh_interval参数，默认是1s。在template中设置如下:

	```
	{
  "logstash" : {
    "order" : 0,
    "template" : "eagleye-log*",
    "settings" : {
      "index.refresh_interval" : "2s"
    },
    "mappings" : {
      "log" : {
        "_all" : { "enabled" : false },
        ... ...
	```
	
	
	
<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>

