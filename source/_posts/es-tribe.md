layout: photo
title: Elasticsearch独立master,tribe节点
tags:
  - Elasticsearch
  - 原创
  - 分布式
photos:
date: 2015-11-26 10:29:36
categories: Elasticsearch
---


## 场景描述

之前的ES机器是三台服务器九个实例, 之所以每台服务器上部署三个实例, 是因为机器不够, 另外一个是每个机器有三块硬盘, 24核cpu, 196G内存, 所以一个物理机上起三个实例, 可以最大限度的发挥ES集群的优势. 
<!--more-->
集群中每个节点都可以是data和master节点, 并且和master通信的超时时间为3s.

```
discovery.zen.ping.timeout: 3s
```
当我们的数据节点因为写入压力过大时, 可能会使节点之间的心跳通信超过这个时间, 那么可能会引起重新选举master的可能. 这次将新增三个实例分布到这三台服务器上, 做master节点.下面是master节点的主要配置:

```
cluster.name: eagleye_es
node.name: "eagleye_es_xx_master"

node.master: true
node.data: false

#ping 其它节点的超时时间
discovery.zen.ping_timeout: 30s

#心跳timeout设为2分钟，超过6次心跳没有回应，则认为该节点脱离master，每隔20s发送一次心跳。
discovery.zen.fd.ping_timeout: 120s
discovery.zen.fd.ping_retries: 6
discovery.zen.fd.ping_interval: 20s

#要选出可用master, 最少需要几个master节点
discovery.zen.minimum_master_nodes: 2

path.logs: /var/log/es_master
#不使用交换区
bootstrap.mlockall: true

transport.tcp.port: 8309
transport.tcp.compress: true
http.port: 8209
```

由于我们的业务场景是给业务人员提供实时的日志查询, 那么近几日的日志将会是频繁查询的对象, 那么一个月以前的日志可能少有人进行查询. 配合这样业务场景, 我们将来打算进行集群数据的冷热分离. 这次变更正好分出独立的查询节点, 并通过tribe的方式去进行. 这次增加两个tribe节点用来做查询节点, 并为之后的冷热分离做准备, tribe节点主要配置如下:

```
node.name: "eagleye_es_xx_tribe"
node.data: false
node.master: false
path.logs: /var/log/es_tribe
bootstrap.mlockall: true

index.search.slowlog.threshold.query.warn: 5s
index.search.slowlog.threshold.query.info: 1s
index.search.slowlog.threshold.query.debug: 500ms
index.search.slowlog.threshold.query.trace: 500ms

index.search.slowlog.threshold.fetch.warn: 5s
index.search.slowlog.threshold.fetch.info: 1s
index.search.slowlog.threshold.fetch.debug: 500ms
index.search.slowlog.threshold.fetch.trace: 500ms

index.indexing.slowlog.threshold.index.warn: 5s
index.indexing.slowlog.threshold.index.info: 2s
index.indexing.slowlog.threshold.index.debug: 500ms
index.indexing.slowlog.threshold.index.trace: 500ms

transport.tcp.port: 8308
transport.tcp.compress: true
http.port: 8208

tribe:
  hot:
    cluster.name: eagleye_es
  blocks:
    write: true
    metadata: true
  on_conflict: prefer_hot

threadpool:
  search:
    tyep: fixed
    size: 24
    #用来保存请求的队列
    queue_size: 100
```

做到这个时候, 还需要将原有的9个节点修改为data节点, 主要配置变更如下:

```
node.data: true
node.master: false
```

最后我们在查询程序中, 就不能指定集群的名字了, 而是直接通过tribe节点进行检索

```
 private static void init() {
        Settings settings = ImmutableSettings.settingsBuilder()
//                .put("cluster.name", AppConstants.EAGLEYE_ES_CLUSTER)  // 由于使用了 节点 , 就不能加 cluster.name 属性了
                .put("client.transport.sniff", false)
                .build();
        client = new TransportClient(settings);
        String esNodes = PropertiesUtil.getProperty(AppConstants.EAGLEYE_IMAGO_KEY, AppConstants.EAGLEYE_ES_TRIBES_KEY);
//        String esNodes = PropertiesUtil.getProperty(AppConstants.EAGLEYE_IMAGO_KEY, AppConstants.EAGLEYE_ES_NODES_KEY);
        String[] esNodeList = esNodes.split(",");
        for (String serv : esNodeList) {
            String[] node = serv.split(":");
            if (node.length == 2) {

                client.addTransportAddress(new InetSocketTransportAddress(node[0], Integer.valueOf(node[1])));
            }
        }
    }
```

## 现在的ES集群结构

![ES集群架构图](http://siye1982.github.io/img/blog/es_tribe/architecture.png)

## 碰到的问题

* 开始master节点中并没有配置分片数量, 导致新索引按照默认的5个进行分片.
	后来我们在数据节点对应的template中的mapping中添加了分片信息. 主要配置修改如下:
	
	```
	{
  "eagleye_bigindex": {

    "order": 0,
    "template": "eagleye_bigindex*",
    "settings": {
      "index.refresh_interval": "2s"
      "number_of_shards": 9,
 		 "number_of_replicas": 0, 
    },

    "mappings": {
      "log": {
        "_all": {
          "enabled": false
        },
        
        ...
        ...
	```

<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>

