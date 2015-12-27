layout: photo
title: ElasticSearch 1.7.2 升级到 2.X 的流程
tags:
  - Elasticsearch
  - 原创
  - 分布式
photos:
date: 2015-12-23 16:35:16
categories: Elasticsearch
---

# ElasticSearch 1.7.2 升级到 2.X 的流程

之前在做Elasticsearch 1.4.4 到 1.7.2 升级的过程中, 直接替换ES程序即可, 升级很平滑.但是这次从1.7.2版本升级到2.1.0版本就不那么顺利了. 下面会记录升级整个过程中碰到的问题.

<!--more-->

## 升级步骤

### ES集群升级

这次升级官方给出了很多[Breaking changes](https://www.elastic.co/guide/en/elasticsearch/reference/master/breaking-changes.html), 但是我们可以借助ES一个插件来帮助我们分析, 现有ES集群需要做出哪些改动才可以顺利升级到2.X版本.

* 插件安装: `./bin/plugin -i elastic/elasticsearch-migration`
	查看原集群中哪些配置和2.x版本冲突, 我们这里碰到的问题是, **在mapping配置里面, 如果是同一个索引模板中,不同的type,里面如果字段名字有一样的, 但是字段模块不一致的需要修改, 即使名字一样,有的设置了分词, 有的设置了不分词也是不可以的. 2.x版本的索引压缩有很大改进.** 

* [下载最新版本](https://download.elasticsearch.org/elasticsearch/release/org/elasticsearch/distribution/zip/elasticsearch/2.1.1/elasticsearch-2.1.1.zip)

* 将原版下的 $ES_HOME/config 下的文件全部copy到新版本中

* 将$ES_HOME/bin/elasticsearch.in.sh elasticsearch 两个文件copy到新版本中相应的路径中, 然后修改该文件中elasticsearch.x.x.jar为新版本的jar

* 将 `$ES_HOME/plugins` 文件夹copy到新版 $ES_HOME目录下

* 针对IK分词, 需要安装最新版本
	* 获取最新版本并编译, copy到新版$ES_HOME/plugins/ik下
	
		```
		git clone https://github.com/medcl/elasticsearch-analysis-ik
		cd elasticsearch-analysis-ik
		mvn clean
		mvn compile
		mvn package
		copy & unzip file  #{project_path}/elasticsearch-analysis-ik/target/releases/elasticsearch-analysis-ik-xxx.zip to your elasticsearch's folder: plugins/ik
		```

	* 将ik项目的config/ik中的内容copy到your-es-root/config/ik中
	
* 更新 head 插件
	`./bin/plugin install mobz/elasticsearch-head`
	
* 安装sql插件
	`./bin/plugin install https://github.com/NLPchina/elasticsearch-sql/releases/download/2.1.0/elasticsearch-sql-2.1.0.zip`
	如果碰见版本不一致的,需要修改插件内部的plugin配置文件, 将其中es版本修改为匹配的.
	安装该插件后必须要重启
	
* 安装kopf插件
	'./elasticsearch/bin/plugin install lmenezes/elasticsearch-kopf/{branch|version}'
	
* elasticsearch.yml 配置文件中的改动
	* 去除 `index.analysis.analyzer.default.type : "ik"`
	* 添加主机host `network.host: 192.168.10.235`

	如果配置中有配置自动发现的,注释掉, 并改为单播模式
	
	```
	#ping 其它节点的超时时间
	#discovery.zen.ping_timeout: 30s
	#要选出可用master, 最少需要几个master节点
	#discovery.zen.minimum_master_nodes: 2
	discovery.zen.ping.unicast.hosts: "192.168.10.235:9309,192.168.10.236:9309,192.168.10.237:9309"
	```

* 将$ES_HOME/bin/elasticsearch.in.sh 文件中添加如下内容
	
	```
	 ES_HEAP_SIZE=8g
	 ES_GC_LOG_FILE="/eagleye/data/esdata/logs/esgc.log"
   ```
  	**如果使用的是jdk1.8将如下需要做如下改动**
   
  ```
   # Add gc options. ES_GC_OPTS is unsupported, for internal testing
	if [ "x$ES_GC_OPTS" = "x" ]; then
	#  ES_GC_OPTS="$ES_GC_OPTS -XX:+UseParNewGC"
	#  ES_GC_OPTS="$ES_GC_OPTS -XX:+UseConcMarkSweepGC"
	#  ES_GC_OPTS="$ES_GC_OPTS -XX:CMSInitiatingOccupancyFraction=75"
	#  ES_GC_OPTS="$ES_GC_OPTS -XX:+UseCMSInitiatingOccupancyOnly"
	  ES_GC_OPTS="$ES_GC_OPTS -XX:+UseG1GC"
	fi
  ```
	
* 复制原集群的索引元数据到新集群中(重要)
	元数据在master节点的data目录下, 将该目录下的所有元数据copy到新版本集群中相应位置即可
	
* 2.1.1版本已经将config目录下的index template移除, **如果是新建索引在原先config/template下的索引模板将不起作用**, 如果需要给某个索引配置模板可以参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html#indices-templates)

* 关闭自动分片

	```
	curl -XPUT http://192.168.1.2:9200/_cluster/settings -d 
	'{ 
		"transient" : { 
			"cluster.routing.allocation.enable" : "none" 
	  	} 
	}'
	```
* 同步flush操作
	如果不允许任何丢失, 需要执行该操作, 如果可以容忍在短时间内的数据丢失, 可以忽略这一步骤
	
	```
	
	curl -XPOST http://192.168.1.2:9200/_flush/synced
	```
* 需要同时停止所有集群的es服务, 然后挨个重启
	
	```
	curl -XPOST http://192.168.1.3:9200/_cluster/nodes/_local/_shutdown
	```



### ES的客户端升级

这里主要描述java客户端的改动.

* pom中的es引用变更为

	```
	<dependency>
       <groupId>org.elasticsearch</groupId>
       <artifactId>elasticsearch</artifactId>
       <version>2.1.1</version>
   </dependency>
	```

	**注意: 该版本依赖google的guava-18.0版本, 我们项目中依赖的guava-15.0导致,spring启动NoSuchClass异常, 需要同时升级guava版本**

* 创建客户端的改动,下面是具体变更之后的内容,官方文档可以看这具体看[这里](https://www.elastic.co/guide/en/elasticsearch/reference/current/breaking_20_java_api_changes.html#_query_filter_refactoring)
	
	* ESClient变更
	
		```
		 public ESClient(String clusterName, String esNodes) {
			//Settings settings = ImmutableSettings.settingsBuilder()
	        Settings settings = Settings.builder()
	                .put("cluster.name", clusterName)
	                .put("client.transport.sniff", false)
	                .build();
	        this.client = TransportClient.builder().settings(settings).build();//new TransportClient(settings);
	        String[] esNodeList = esNodes.split(",");
	        for (String serv : esNodeList) {
	            String[] node = serv.split(":");
	            if (node.length == 2) {
			//this.client.addTransportAddress(new InetSocketTransportAddress(node[0], Integer.valueOf(node[1])));
	                this.client.addTransportAddress(new InetSocketTransportAddress(new InetSocketAddress(node[0], Integer.valueOf(node[1]))));
	            }
	        }
	    }
		
		```
	* 依赖的jackson版本需要跟着变更
	
		```
		<fasterxml.jackson.version>2.6.2</fasterxml.jackson.version>
		```
	* 查询代码需要将所有Filter变更为Query
	* 2.1.1版本中tribe节点无法加入集群中,具体看[Github上针对这个问题的issue](https://github.com/elastic/elasticsearch/issues/15373)

	
	


	
<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>

