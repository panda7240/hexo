title: 集中配置管理(IMAGO)
date: 2015-06-17 10:11:19
categories: 分布式
tags: 
- 配置管理
- Java
- zookeeper
---

## 概述

常规项目会有很多配置文件,比如数据库链接信息, 缓存信息等. 我们通常的做法是通过配置文件进行管理可以达到只需要修改一个地方,整个项目都可以生效的目的.目前我们的开发项目繁多, 程序为了保持高可用,会进行集群部署. 这样我们要对某一个或者某几个项目进行配置文件变更时,需要逐个进行修改.这样大量的人为操作可能会增加程序升级或发布的风险.与此同时修改配置文件之后会涉及到应用的重启.为了减小发布风险,我们需要一个地方对各个系统配置进行统一的管理,这时统一配置管理系统诞生的背景,我们暂且取名为 “IMAGO”没有特殊的含义,只是一个代号.


## 架构

![imago架构图](http://siye1982.github.io/img/blog/imago-architecture.png)

Imago分为两部分：

1. 配置管理控制台(imago-admin)

   配置管理后台提供配置管理, 简单权限管理, 详细配置变更日志管理等, 所有的配置数据最终以Mysql数据库中存储的配置为最终正确配置. 

2. 配置客户端(imago-client)

   上图中的箭头方向表示数据的流向. 当应用启动时, imago-client从Zookeeper获取相关配置项到本地, 并存储一份文件快照, 一份缓存快照, 应用直接从本地缓存中读取相关配置. 当Zookeeper中的配置项发生变动时, imago-client会监听到变动并更新本地文件快照和本地缓存. 在使用过程中如果Zookeeper不可用, 应用还可以从本地文件快照中获取配置进行启动. 应用始终都是从imago-client的本地缓存中获取缓存, 提高效率.


## 存储在Zookeeper中的数据结构

1. 配置管理后台负责配置数据的维护，如增、删、改、查，配置数据存放于zookeeper节点之上，每个APP都有一个唯一的ID，如order的节点path为/imago/order，该节点下又可以有多个配置节点，如/imago/order/key1, /imago/order/key2… 管理后台必须校验节点PATH的唯一性.

2. 应用可以有两种类型

	(1). 公共应用, 该种应用会将一些公共资源的配置放在这种应用下面, 比如mysql的配置, redis的配置等, 这么做的目的是为了将来这些配置ip或者其他信息变更了,我们只需要修改一个地方.我们将来可以通过在imago-admin里面可以看到哪些应用引用了这些公共应用.
	比如,和交易相关的公共配置统一放在appkey为trade_public目录下, 具体参考如下:
	```	
		/imago/trade_public_mysql/1.1
		/imago/trade_public_redis/1.1
	```
	(2). 普通应用, 该种应用下的配置项可以设置一些该应用特有的一些配置.

## 容灾设计

我们可以总结一下imago整个系统完全不可用的条件,下面是可能发生的情况:

1. 数据库不可用。
2. 所有zookeeper均不可用。
3. imago-client主动删除了File Snapshot, Local cache.

在上面2, 3都不可用时才会发生不可用的情况.这在生产环境中出现的几率是极小的.

## 测试用例

### 功能测试(在没有任何异常的情况下进行)

1. 创建测试数据
    ```
		/imago/app1/xxx1
		/imago/app1/xxx2
		/imago/app1/xxx3
    ```
    
2. app启动时client可以顺利从zookeeper加载相关配置到file snapshot 和 local cache. 
    
3. 验证是否可以生成文件快照
	```
		~/.imago/imago_snapshot/app1.properties
    ```
4. 验证本地缓存是否可以取到这些值.
    
5. 当zookeeper其中某一个数据发生变动时, 相应的文件快照和本地缓存是否也相应变更.

### 容灾测试

1. zookeeper集群中其中一台机器不可用, client是否可以正常从zookeeper获取数据.

2. client(非首次)启动时无法从zookeeper获取数据, 则可以从之前的文件快照获取配置数据, 并加载到本地缓存中.

### 配置不可用的情况
  
1. client第一次启动时, zookeeper就不可用, 则无法从中获取数据, 并生成文件快照和本地缓存.

## Imago-admin 功能说明





## 部署

### 初始化imago-admin

当首次启动imago-admin时需要初始化zookeeper中的数据库配置,进入到zookeeper命令行执行以下命令:
```
 create /imago ""
 create /imago/trade_public_mysql ""
 create /imago/trade_public_mysql/1.1 {"ip":"xxx.xxx.xxx.xxx","port":"3231","type":"master","dbs":[{"name":"xxx","user":"trade_user","pwd":"xxx"}]}
```
## imago-java-client 配置说明

具体使用步骤总结如下:

1. 在pom.xml中加入:
  	```
	<dependency>
   	<groupId>com.xxx.xxx</groupId>
		<artifactId>imago-java-client</artifactId>
		<version>1.8.0-SNAPSHOT</version>
		<exclusions>
			<exclusion>
				<groupId>org.jboss.netty</groupId>
				<artifactId>netty</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	```

2.  大家可以在spring配置文件中嵌入:
		
	因为测试环境是复杂的, 为了应对各种情况下, imago-java-client都可用, 我将各种场景列举如下, 大家根据具体情况进行变通:
	(1) 普通配置,  在dns正常, zookeeper服务正常的情况下: 
	```
	<bean id="configManager" class="com.xxx.imago.client.ConfigManager” />
	```
   	(2) 应对测试环境dns不可用的情况, 我们可以自己指定 zookeeper地址
	```
	<bean id="configManager" class="com.xxx.imago.client.ConfigManager”>
		<constructor-arg name="serverList" value="192.168.1.100:2181,192.168.1.101:2181,192.168.1.102:2181"/>
	</bean>
	```
	(3) 应对测试环境zookeeper 不可用情况, 在 ~/.imago/imago_snapshot/ 目录下创建 .closezk文件(windows系统在程序所在盘符的根目录下寻找该路径)该情况, 需要快照文件已经在该路径下.(如果初始化时也不可用, 可以从imago-admin 中下载相应的配置文件放在该路径下.)
	```
	<bean id="configManager" class="com.xxx.imago.client.ConfigManager” />
	```
   

3.   在spring中的配置示例代码如下:
  	(1) 简单引用
	```
		<property name="jdbcUrl" value="#{configManager.getMysqlConfig('trade_public_mysql','1.1','trade').url}"/>
	```
   	(2) 参数是变量,可以支持参数是变量的引用
	```
  		<constructor-arg name="redisIp" value="#{configManager.getRedisConfig('trade_public_redis’,{configManager.getConfigValue('trade_eagleye','redis_key')}).ip}"/>
	```
    
## 关于通过DNS获取zookeeper集群地址

 我们的集中配置管理统一使用的域名为: zk.xxx.com, 通过dns来获取zk集群地址是为了将来如果该地址变更,每个应用可以不用修改zk集群地址.


## 什么样的配置适合放在imago中进行管理
1. 静态配置, 不会频繁变更的配置(比如一些业务上的开关, 阈值).
2. 公共资源配置, 比如: mysql, redis, kafka, mongodb, zookeeper等.
3. 下面这些配置不建议放在imago中进行管理.
	(1) 基本不会变更的配置(比如: c3p0的一些基础配置, redis, kafka,mongodb等除了ip, 端口, 权限之外的一些基础配置)这些配置可以直接写死在应用中.
	(2) 如果接入loom进行服务治理之后,消费服务的地址和端口.

## 备注(重要):
1. 在imago-admin首页, 可以看到哪些应用的实例引用了以trade_public为前缀的公共资源.
2. 从数据库同步数据到zookeeper的功能, 仅限于初次搭建imago进行数据迁移,初始化imago时使用.该功能会触发所有数据库中存储的配置项的使用, 如果在执行时,已经有大量的imago-java-client监听了zk的数据节点, 会触发所有watch事件.


















































