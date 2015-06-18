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

![imago架构图](http://siye1982.github.io/img/blog/imago/imago-architecture.png)

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


### 首页

![首页](http://siye1982.github.io/img/blog/imago/imago-index.png)

1. 集中配置管理主要提供如下功能点:
	* 应用管理
	* 配置项管理(具体的配置项数据在该节点设置)
	* 用户管理
	* 资源管理(菜单和功能url的动态管理)
	* 用户组管理(进行用户的分组功能,可以给用户组分配具体的功能权限和数据权限)
	* 日志查看(终于数据的增删改的详细记录)

2. 首页关系图可以查看某个ip上具体的pid引用了什么公共应用, 也可以查看哪些公共应用被什么程序引用了.

### 应用管理

![App管理](http://siye1982.github.io/img/blog/imago/imago-app-manage.png)

1. 可以通过应用类型, 应用key, 应用value进行简单检索

2. 点击应用key上的超链接, 将会进入该应用下的配置项列表.

3. 在进行数据迁移时, 只需要将mysql数据进行迁移, 然后点击”同步全量数据到Zookeeper”按钮,会将mysql中所有数据同步到zookeeper节点上,如果已经有数据在zookeeper上,并且已经有程序watch了这些节点, 那么该操作将***会触发大量的watch操作***. 所有,该按钮建议只在初始化时使用.

4. 可以针对某个应用进行配置项的导入, 导入文件可是遵循 *.properties属性文件格式.

5. 可以将某个应用下的配置项导出为 *.properties属性文件.

6. 可以只针对某个应用从mysql中同步数据到Zookeeper.

7. 删除某个应用, 该操作是逻辑删除, 同时会逻辑删除该应用下所有的配置项, 同时也会物理删除zookeepr上对应的数据.

![App新增,修改](http://siye1982.github.io/img/blog/imago/imago-app-update.png)

进行应用的新增和修改时,需要设置应用的类型:

* 公共应用

	该类型的应用,会是多个项目中使用, 比如mysql,redis,kafka等配置放在公共应用中. 同时首页上会显示公共应用被哪些程序使用的关系图.

* 普通应用

	普通应用是区分公共应用而存在的, 一般应用下的独有的一下配置项放在普通应用下.


### 配置项管理

![配置项管理](http://siye1982.github.io/img/blog/imago/imago-config-manage.png) 

1. 点击某个配置项Key上的链接,可以查看该配置项的所有历史版本.

2. 可以针对配置项Key,Value进行简单检索.

3. 配置项的添加功能, 只能通过具体的应用页面引导进来才能操作, 直接通过菜单上的配置项管理进入没有新增配置项功能按钮.

![配置项新增,修改](http://siye1982.github.io/img/blog/imago/imago-config-update.png) 

在进行配置项的新增或修改时, 需要注意配置项的value值有模板设定, 默认下是走的文本, 但是为了便于对一些公共资源的管理,加入了类似mysql,redis,kafka等配置项的模板.

![获取配置代码](http://siye1982.github.io/img/blog/imago/imago-config-code.png) 

每一个配置项后面都可以通过配置代码按钮获取到某一个配置项在各个语言中的配置代码.

![获取配置项在zk中的数据](http://siye1982.github.io/img/blog/imago/imago-config-getzk.png) 

可以通过该方法获取该配置项在zk中具体的数据, 不用再去zookeepr上通过命令行查询.

![获取配置项在zk中的数据](http://siye1982.github.io/img/blog/imago/imago-config-version.png) 

可以获取到某一配置项的所有版本变更记录, 并可以通过某一个版本进行恢复.


### 资源管理(功能权限管理)

![获取配置代码](http://siye1982.github.io/img/blog/imago/imago-resource-manage.png) 

在用户组管理中我们可以进行资源管理,资源管理主要可以通过树形菜单配置来控制哪些用户组里面的用户可以操作哪些功能url.用户登录后看到的左侧菜单也是通过该功能进行动态配置.


### 数据权限管理

![获取配置代码](http://siye1982.github.io/img/blog/imago/imago-data-manage.png) 

该功能可以配置哪些用户可以对哪些数据进行哪些操作,这里主要集中控制对app数据的控制. 如果涉及到公共应用的如果用户只有查询权限,那么他们会针对该公共应用下的配置项中的密码选项是不可见的. 这个数据操作权限由App延伸到配置项.

### 日志管理

![获取配置代码](http://siye1982.github.io/img/blog/imago/imago-log.png) 

对系统中比较关键的数据操作都会进行详细的日志记录, 便于进行用户行为的跟踪.

## 部署

### 初始化Imago-admin

当首次启动imago-admin时需要初始化zookeeper中的数据库配置,进入到zookeeper命令行执行以下命令:
```
 create /imago ""
 create /imago/trade_public_mysql ""
 create /imago/trade_public_mysql/1.1 {"ip":"xxx.xxx.xxx.xxx","port":"3231","type":"master","dbs":[{"name":"xxx","user":"trade_user","pwd":"xxx"}]}
```
### Imago-java-client 配置说明

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


















































