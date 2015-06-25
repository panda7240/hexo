
layout: photo
title: LOOM服务治理
date: 2015-06-18 20:53:38
categories: 服务治理
tags:
- 原创
- SOA
- Java
- Zookeeper
- Finagle
- Zipkin
photos:
- http://siye1982.github.io/img/blog/loom/loom.png
---

## 概述

在介绍LOOM之前, 说一下我对一个互联网公司多语言开发环境的感受. 现在互联网高速发展的时代,产品快速迭代会帮助我们快速产出产品原型,快速获取用户反馈,快速抢占市场. 我们需要使用适合的语言去做一些特定领域的事情. 在小型开发团队中,大概10人左右的开放团队中, 开发语言的选型不用过多的考虑管理成本,只要对产品的快速迭代有帮助的都可以使用.随着团队逐渐壮大, 我们需要更多的考虑不同开发团队之间的沟通成本, 招聘新人的培训成本, 由人员更替产生的交接成本等. 这个时候多语言开发环境带来的弊端逐渐显现出来, 我们需要在这中间找到一个平衡点. 
<!--more-->
在公司发展到一定规模, 肯定会产生很多业务系统, 这些业务系统需要互相之间进行调用来完成协作. Thrift帮助我们在异构语言直接的沟通建立了桥梁. 只是简单的RPC调用在服务比较少的阶段还能够接受,但是随着服务逐渐的增多,调用关系越来越复杂, 我们迫切的需要一个框架可以对所有服务进行统一的管理,监控,预警. 这是LOOM诞生的直接需求. “LOOM,织布机”,寓意可以将各个网状服务进行很好的治理.

LOOM的大部分需求直接采用的阿里巴巴的服务治理框架—DUBBO, 这里向DUBBO的所有开发人员致敬, DUBBO的出现带动了SOA概念真正的落地, 相信国内很多互联网公司都在用DUBBO进行服务治理. LOOM里面关于RPC相关的工作,我们直接使用了Twitter的finagle-thrift, 它可以帮助我们非常简单的写出高性能的异步的thrift调用. 另外我们可以直接通过zipkin来进行异构语言之间的调用链信息的收集.



## loom服务治理需求

### 一期需求

在大规模服务化之前，应用可能只是通过Thrift，简单的暴露和引用远程服务，通过配置服务的URL地址进行调用，通过HAProxy等软件进行负载均衡。服务治理平台主要的需求可以通过下面几点来体现:
- 当服务越来越多时，服务URL配置管理变得非常困难，负载均衡器的单点压力也越来越大。此时需要一个服务注册中心，动态的注册和发现服务，使服务的位置透明。并通过在消费方获取服务提供方地址列表，实现软负载均衡和Failover，降低对HAProxy负载均衡器的依赖，也能减少部分成本。

- 当进一步发展，服务间依赖关系变得错踪复杂，甚至分不清哪个应用要在哪个应用之前启动，架构师都不能完整的描述应用的架构关系。这时，需要自动画出应用间的依赖关系图，以帮助架构师理清理关系。

- 接着，服务的调用量越来越大，服务的容量问题就暴露出来，这个服务需要多少机器支撑？什么时候该加机器？为了解决这些问题.
  (1). 要将服务现在每天的调用量，响应时间，都统计出来，作为容量规划的参考指标。
  (2). 要可以动态调整权重, 开启,禁用服务，在线上，将某台机器的权重一直加大，并在加大的过程中记录响应时间的变化，直到响应时间到达阈值，记录此时的访问量，再以此访问量乘以机器数反推总容量。
	

### 二期需求
一期项目中, 我们完成了服务提供者, 消费者从启动开始总调用量的统计. 其实我们可以收集更详细的信息, 可以细化到哪个应用中的哪个方法的什么时候被调用,调用时长是多少.通过这些信息,我们可以通过实时计算出某台机器的某个方法响应时间变慢了等操作.这些操作会分几个部分:
- 原始数据的收集
	方法名,类名,参数类型,参数个数,开始时间,	结束时间,	执行时间,	ip,	port, pid,服务名
            
- 基于原始数据的统计, 每个方法从服务启动开始的调用次数.
            
- 界面展示:
  (1). 针对某个服务, 某个机器上的方法调用列表如下:
       方法名.
       每分钟生产力,成功/失败.
       调用成功次数/失败次数.
  (2). 汇总页
       统计服务方法相应时间分别在5, 200ms, 500ms, 1000ms, 1500ms, 30000ms 方法列表
       eg: 服务响应时长在 [0,200) 之内的
           方法名
           ip:port
           pid
           所属服务名
           执行次数
                
                

## 注册中心Zookeeper数据结构
![zookeeper数据结构](http://siye1982.github.io/img/blog/loom/zookeeper.png)
 
- serviceName 是定位服务的唯一标识, 各个语言程序以该名称为基准, 该节点的value值将存储该服务的描述信息, 也需要进行encode操作.
	
- serviceName下面会有四个静态节点

  (1). providers节点
  所有服务提供者的url记录在该节点下
  - URL样例: 
  ```
  provider://192.168.1.103:12307/loom-test-providerA?application=loom-test-providerA&category=providers&owner=panda&pid=4078&timestamp=1420708343991&version=1.0
  ```

  - 参数说明
	  - application: 服务所在程序的名称, 只是在loom-admin中进行分组管理.
	  - category : 标识是提供者, 带有该标识的统一注册到providers节点下.
	  - owner: 服务所有者.
	  - pid: 服务所在进程的pid.
	  - timestamp: 服务启动注册上zookeeper的时间点.
	  - version: 服务的版本, 用来进行灰度发布时版本不兼容也能发布的需求.
	
  (2). consumers 节点
  所有服务消费者的url注册到该节点下.
  - URL样例:
  ```
  consumer://192.168.1.107/loom-test-consumerA?application=loom-test-providerA&category=consumers&timeout=30000&pid=4078&timestamp=1420708343991&version=1.0
  ```

  - 参数说明
	  - application: 消费者所在程序的名称, 只是在loom-admin中进行分组管理.
	  - category : 标识是消费者, 带有该标识的统一注册到consumers节点下.
	  - timeout: 消费者调用服务多长时间认为是超时,该参数根据各自客户端需求来配置, 也可以没有..
	  - pid: 消费者所在进程的pid.
	  - timestamp: 消费者启动注册上zookeeper的时间点.
	  - version: 消费者消费服务的版本, 用来进行灰度发布时版本不兼容也能发布的需求.
	  
  注: consumers下的临时节点对loom的作用只是可以观察到有多少个消费者在消费这个服务.

  (3). configurators 节点
  该节点会记录每个服务提供者的权重, 是静态节点, 即使服务丢失, 重新启动也不会丢失之前的设置.
  - URL样例: 
  ```
  override://192.168.1.103:12307/loom-test-providerA?application=loom-test-providerA&category=configurators&pid=4078&dynamic=false&weight =100&version=1.0
  ```

  - 参数说明:
	  - application: 消费者所在程序的名称, 只是在loom-admin中进行分组管理.
	  - category : 标识是路由信息, 带有该标识的统一注册到configurators节点下.
	  - disabled: 是否禁用该消费者.
	  - pid: 消费者所在进程的pid.
	  - dynamic: 是否是动态临时节点,当注册方退出时，数据依然保存在注册中心，必填。
	  - weight: 该服务提供者的权重值.
	  - version: 消费者消费服务的版本, 用来进行灰度发布时版本不兼容也能发布的需求.
      - enabled: 覆盖规则是否生效，可不填，缺省生效:true. override节点是有覆盖操作的,当服务提供者启动时需要根据ip,port, version,作为判断与provider下的节点信息进行合并后提供给消费者(只是提供给消费者,而不重写provider节点).
	  
  (4). routers 节点
  该节点会记录该服务消费者的路由信息, 目前只是记录是否禁用该消费者, 将来可能会针对该服务调用者设置服务的黑白名单等路由规则.
  - URL样例: 
  ```
  route://192.168.1.107/loom-test-consumerA?application=loom-test-providerA&category=routers&disabled=false&pid=4078&dynamic=false&version=*
  ```
  - 参数说明:
	  - application: 消费者所在程序的名称, 只是在loom-admin中进行分组管理.
	  - category : 标识是路由信息, 带有该标识的统一注册到routers节点下.
	  - disabled: 是否禁用该消费者.
	  - pid: 消费者所在进程的pid.
	  - dynamic: 是否是动态临时节点.
	  - version: 消费者消费服务的版本, 用来进行灰度发布时版本不兼容也能发布的需求.

  (5). 备注
  所有url为了避免中文乱码,关键字符, 统一encode编码.



## loom核心功能

### 服务注册
服务启动时只需要设置启动端口, zookeeper地址, 即可通过loom将服务资源信息注册到zookeeper中, 相同服务的提供者统一注册到
```
/loom/serviceName/providers
```
节点下, 注册节点必须为动态临时节点.

### 负载均衡
- 服务消费者,根据服务名称到/loom/serviceName/providers下订阅服务提供者列表. 根据weight实现负载均衡算法, 我们需要实现默认的负载均衡算法为Random算法, 并可以根据权重的调整对各个提供者的命中率进行调整.
- 服务消费者需要注册到
```
/loom/serviceName/consumers
```
节点下, 注册节点为动态临时节点.

### 服务治理
- 我们通过loom-admin对某个服务提供者进行权重的设置, 用来控制某个服务的访问量.
- 可以在loom-admin中禁用启用某个服务消费者.
- 可以统计使用的服务列表, 消费服务的程序列表.



## loom 测试用例

### 功能测试

- 权重负载均衡测试:
    场景描述:
	(1) 四台服务提供者机器
	(2) 权重都为100时, 发起10万次请求, 每台服务器都在25000左右, 误差不超过100次
	(3) 发起10万次请求, 四台服务器权重和命中次数分别为: 400:53223 ; 200:26864 ; 100:13269 ; 50:6644, 误差不超过100次

- 服务提供者禁用, 启用测试:
	如果禁用某个提供者, 那么根据剩下的所有提供者的权重重新计算负载均衡值.

- 服务提供者和消费者version测试:
	在消费者的配置中指定需要消费的服务版本, 那么运行程序时,则是获取该版本的服务提供者, 并且只根据相同版本的权重进行负载均衡.

- 服务消费者的禁用,启用测试:
	如果禁用之后, 消费者不应该再访问该提供者, 并且权重会根据现有提供者进行调整.

- 某台服务提供者下线, 消费者会自动更新提供者列表,然后并且更新

- 服务消费者的禁用, 启用.
	
- 多注册中心验证
	
- 增加计数统计,只统计每分钟的生产率和调用失败率,该指标只是可以让管理员直观的看到服务启动之后生产力
	

### 性能测试(注: 未经说明的都是基于scala-2.9.2版本进行的压测)

- A、B方法堵塞测试
  场景描述：
   (1) 同一个接口中存在两个方法，A和B。A方法有延迟，B方法正常。
   (2) 对该接口（返回值是future层）添加aop代理
   (3) 在aop中对返回值进行处理。
   (4) 调用A方法时正常，并发调用B方法时，线程在10个左右的时候，总会出现个别线程的返回值在等待A方法执行完毕。
  问题原因：
    在对future坐Aop代理时，方法返回值是future类型。此类型是不堵塞线程的。在aop代理类中认为此线程已经结束，实际上又用该线程等待future的返回值。等调用B的时候，有可能用重复使用该线程，导致堵塞。
  解决办法：
   (1) 不要在future层做代理。应该保持finagle的干净性。如果有需求，建议在实现类中做代理。
   (2) 如果非要在future中加入代理，请使用future的回调函数addEventListener。
  修改后，此现象消失。


- 单服务,单机器压力测试
  场景描述:
   (1) 通过loom对外提供thrift服务.
   (2) 测试服务中的一个方法, 该方法只打印方法的入参, 如果打印成功, 则可以确认为方法调用成功.
   (3) 压测机器基本配置, cpu: 2G 4核, memory: 16G, 服务JVM启动内存: 256M
   (4) 模拟500个独立用户, 压测两个小时.
  结果:
    每秒相应6000多次, 单次请求耗时:60ms
 
 
- 单服务,单机器增加newrelic后的压力测试
  场景描述:
   (1) 通过loom对外提供thrift服务, newrelic监控其中一个执行打印日志的方法.
   (2) 测试服务中的一个方法, 该方法只打印方法的入参, 如果打印成功, 则可以确认为方法调用成功.
   (3) 压测机器基本配置, cpu: 2G 4核, memory: 16G, 服务JVM启动内存: 256M
   (4) 模拟500个独立用户, 压测两个小时.
  结果:
    每秒相应6000多次, 单次请求耗时:80ms 


- 单服务,2台机器提供一个服务
  场景描述:
    (1) 增加zipkin监控
    (2) jvm内存上限为256M
    (3) jvm内存上限为1024M
  结果: 不同的jvm设置,都会造成内存溢出, 通过分析堆日志,是zipkin线程引起的.下图为,通过分析内存溢出时的堆栈信息得出的结果图.之后又进行了一次针对zipkin的压测, jvm 启动内存为1G,具体数据如下:
  - 模拟1个用户, 2000tps, 不会出现full gc
  - 模拟5个用户, 7500tps, 不会出现full gc
  - 模拟10个用户, 300tps, 频繁出现full gc, 系统大部分时间都在出来full gc
  - 模拟500个用户, 服务瞬间full gc,不会接受其他请求.
   	      
![zipkin内存溢出](http://siye1982.github.io/img/blog/loom/zipkinerror.jpg)   	     
	  
    
    
- 单服务, 2台服务器提供这个服务
  场景描述:
    (1) 增加newrelic监控
    (2) 在server端启动finagle服务时,添加了
	```
	hostConnectionMaxLifeTime(new Duration(300 * Duration.NanosPerSecond()))
    ```
    每个链接最大存活时间为5分钟, 这是每次loadrunner到五分钟时都会停止.
    (3) 压测16小时.
    (4) 客户端线程池大小为300
    (5) 压测机器为8核cpu
    (6) loadrunner模拟500独立用户访问
    
  结果: 请求rpm 60万/分钟, 每台服务器接收请求为30万, cpu大约在10% jvm内存在256M.
  如果是长连接请求, 每个链接会有最大存活时间, 到时,服务端会关闭,所以服务端不要设置链接生命周期.
  长时间压测时,会出现处理请求的数量逐渐下降.cpu在10%左右.
  cpu一直空闲,而处理请求的数量在逐渐下降,可能和调用的方法为空有关, 下面在方法中做一些简单的计算.在三个小时内请求数比较稳定, 之后会逐渐下降.
  下图为loadrunner压测趋势结果
![16小时压测](http://siye1982.github.io/img/blog/loom/16hourtest.png)      
     
              
- 多服务压力测试
  场景描述:
    (1) 服务调用链为: start=>a=>b=>c,d=>a, 具体如下图:
![压测调用关系图](http://siye1982.github.io/img/blog/loom/invoklink.jpg) 
    (2) 机器配置8核,32G内存,两台
    (3) 增加newrelic监控
    (4) loadrunner模拟500独立用户访问
    (5) 客户端线程池大小设置为8个(因为是8核cpu)
    之所以从300变为8个,是因为开启线程池大小为300个时,调用的服务执行的是空方法时,接受请求的访问量会越来越大.这时查看服务器的cpu和内存消耗都不太高,这是怀疑可能是方法执行太快,8核cpu处理300个线程,这样就会出现在cpu内部频繁出现线程切换, 从而降低机器的整体吞吐量. 我们为了降低cpu上下文切换次数,将线程总数限制在8个. 后来为了验证是否是这个问题,又写了一个服务,该服务方法只做了简单的100次加计算,消费该服务的线程设为8, 进行500用户压测2个小时,机器整体吞吐量维持在 5000/s.但是在这样的条件下,2个小时之后又出现了吞吐量逐渐下降,但是并不多.因为是8个线程,整体吞吐量并没有上去.我继续调整线程数, 通常多线程可以设为cpu核数的4倍,这时设为32个线程. 压测两个小时,这时吞吐量在10000/s,整体性能比较平稳.下图为多服务压测时,服务上cpu每秒上下文切换次数. 因为吞吐量比较大, cpu频繁进行上下文切换, 具体是什么引起的上下文切换,还没有具体跟踪.现在初步怀疑是因为finagle进行通信时频繁调用系统的方法引起的.
![压测调用关系图](http://siye1982.github.io/img/blog/loom/cpucontext.jpg) 
            

- 压测64小时      
  结果: 这次压测,发现前3个小时,可以维持在10000/s 的生产力, 之后61个小时逐渐下降,到48小时时维持在3300/s的生产力.这期间我一直观察服务器的负载.cpu队列并没有显示出大量的任务堆积,也就是说服务器状态良好,没有满负荷运转. 这时开始怀疑是压测入口的loadrunner机器出的问题.开始验证,我通过其他机器对loom集群添加额外的压力,通过newrelic监控发现服务器的生产力又上去了. 从而推断,生产力持续下降是因为loadrunner输出的压力持续下降引起的. 这时找压测人员进行问题排查,发现运行loadrunner的机器负载过高,并且满负载的时间点正好是loom集群生产力下降的时间点.
                
         
- 多服务压测, 使cpu达到100%
  观察zookeeper是否还能获取服务的提供者和消费者的心跳.
        
- 基于scala-2.10的性能测试:
  场景描述: 1000/s 的访问压力
  机器配置8核, 4G内存.
  服务器所在的jvm状况如下图:
      
![压测调用关系图](http://siye1982.github.io/img/blog/loom/scala2.10-jconsole1.jpg)    ![压测调用关系图](http://siye1982.github.io/img/blog/loom/scala2.10-jconsole2.jpg)
  对象回收正常, 老年代执行了唯一一次回收, 还是我手动进行的GC. 
  下图是loadrunner压测8小时统计图
![压测调用关系图](http://siye1982.github.io/img/blog/loom/scala-2.10.8hour.test.png) 
  下图是loadrunner压测14小时统计图
![压测调用关系图](http://siye1982.github.io/img/blog/loom/scala-2.10.14hour.test.jpg)
  下两图为loadrunner压测48小时统计图
![压测调用关系图](http://siye1982.github.io/img/blog/loom/scala-2.10.48hour-1.jpg)
![压测调用关系图](http://siye1982.github.io/img/blog/loom/scala-2.10.48hour-2.jpg)
结果: 在每秒1000访问量的压力下 48小时连续测试, scala-2.10版的loom表现正常, 响应时间, 内存回收都表现正常. 
         



## loom接入说明
   前提：服务提供者和服务消费着都必须是基于finagle生成的代码，并且依赖finagle和thrift相关的包。
   
- 在pom文件中加入如下代码：
	```
	 <dependency>       
	    <groupId>com.xxx</groupId>
	    <artifactId>loom</artifactId>
	    <version>1.1.5-SNAPSHOT</version>
	 </dependency>
	```
	```
	 <!-- 如果是基于scala-2.10则引入-->
	 <dependency>       
	    <groupId>com.xxx</groupId>
	    <artifactId>loom</artifactId>
	    <version>1.1.5-2.10-SNAPSHOT</version>
	 </dependency>
	```

- 添加配置文件
```
<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
	  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	  xmlns:context="http://www.springframework.org/schema/context"
	  xmlns:loom="http://xxx.com/schema/loom"
	  default-autowire="byName" default-lazy-init="true"
	  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
	                      http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd
	                      http://xxx.com/schema/loom http://xxx.com/schema/loom/loom.xsd">

	 <!--id是此应用的应用名称，请修改为自己系统的名称, 可以将该应用的负责人填写在owner属性中-->
	 <loom:application id="XXXXXXXXX" owner="xxxx" />

	 <!-- 如果配置该节点则是开启了简单的统计功能,可以统计某一个服务提供者和消费者的每分钟成功调用测试, 失败的调用次数,建议此功能打开 -->
	 <loom:count id="count" ip="#{configManager.getRedisConfig('trade_public_redis','14.1').ip}"
	                 port="#{configManager.getRedisConfig('trade_public_redis','14.1').port}"
	                 pwd="#{configManager.getRedisConfig('trade_public_redis','14.1').pwd}"/>


	  <!-- 如果配置该节点则是开启了zipkin监控功能,默认是注视掉的,如果需要请打开,采样率默认是0.01  -->
	  <!-- <loom:monitor id="monitor01" url="#{configManager.getConfigValue('trade_public_zipkin','zipkinUrl')}"
	                    port="#{configManager.getConfigValue('trade_public_zipkin','zipkinPort')}"
	                    samplerate="0.01"/> -->

	   <!--zookeeper集群地址, 如果address为空则默认不实例化该注册中心,用以测试环境脱离注册中心时使用,必须配置-->
	   <loom:registry id="zk01" protocol="zookeeper"
	                 address="#{configManager.getConfigValue('trade_public_zookeeper','zk01')}"
	                 timeout="20000"/> 

	  <!-- 如果想作为一个task不提供服务, 但是需要注册到zookeeper中, 则可以是不配置ref和api,端口,threads, 建议port设为0000, 这时服务名一定不能是其他真实服务的名称,可以有一个约定标识, 比如:xxx-task --> 
	  <!--服务提供者配置, 如果上面的registry中的address设置为空,则该服务将脱离注册中心,不会注册到注册中心,可以直接通过端口对外提供服务, -->
	  <!-- threads 建议是cpu核数的四倍, 服务端提供服务时,服务超时时间, 默认30秒  --> 
	  <loom:service id="XXXXX"
	                ref=“xXXImpl"
	                api="com.xxx.finagle.thrift.XXXX.XXXServ"
	                port="12301"
	                threads="32"
	                version="1.0"
	                owner="XXXX"
	                registry="zk01"
	                timeout="30000"
	                remark="这里填写对该服务的描述"/>

	  <!--消费其他服务配置,这里surl为具体的服务地址(比如: finagle://192.168.10.5:12306),如果填写具体地址则会脱离注册中心, 该场景是为了简化测试时脱离注册中心 -->
	  <loom:reference id="xxxxReference" sid="XXXXXXX"
	                  api="com.xxx.finagle.thrift.XXXX.XXXServ" version="1.0"
	                  registry="zk01" surl="" timeout="30000"/>
	</beans>
```
在资源文件夹下添加loomContext.xml并在启动时加载，文件内容如上
添加此配置后工程启动时会自动将服务注册到注册中心，并且自动启动相关服务，如果原来代码中有启动服务的，请关闭。
       
   
   
- 配置文件说明：

	loom:application 
		id:应用程序的唯一id
		owner: 应用负责人, 可以是简单的字符

	loom:registry，注册中心，可以配置多个。
		id：注册中心的id
		protocol：注册中心类型，目前只支持“zookeeper”
		address:注册中心地址列表
		timeout：注册中心请求超时时间

	loom:count, 开启服务提供者和调用者的每分钟生产力统计,该统计信息需要redis进行发送.
		id: count的id,唯一标识,每一个应用只能配置一个count
		ip: 用来发送统计信息的服务器ip,可以认为是redis的ip
		port: redis的端口
		pwd: redis的密码

	loom:monitor, 开启finagle的zipkin跟踪.
		id: monitor的id, 唯一标识, 每个应用只配置一个zipkin的跟踪
		url: zipkin的地址
		port: zipkin的端口
		samplerate: 采样率.如果不填,默认为0.01

	loom:service 对外暴漏的服务信息，可以配置多个，配置多个时端口号不能重复
		id：服务的唯一id
		ref：提供服务的实现类名称，第一个字母小写。此实现类必须继承api里面配置的服务iface接口。并且要在spring容器中注入
			如果不设置该设置,或则设置为空,则会认为是类似task的应用,不会启动finagle服务,只是将该应用注册到zookeeper中
		api：提供服务的服务类名，此类是finagle生成的thrift接口文件全路径。
		port：对外提供的服务的端口号
		threads：提供服务的线程池大小
		version：服务的版本号
		owner：该服务的负责人
		registry:该服务注册的注册中心id
		remark：对该服务的简单描述
		timeout: 服务端提供服务时,服务超时时间, 默认30秒

	loom:reference 调用外部服务配置。可以配置多个
		id：消费者在spring容器中的唯一id
		sid：目标服务的服务id
		api：消费服务的服务类名，此类是finagle生成的代码
		version：调用目标服务的版本号
		timeout：调用服务的超时时间(在Await.result中也可以进行设置), 默认是30秒
		registry：此消费者要去哪个注册中心获取服务提供者列表
		surl：服务提供者地址。如果配置此值，此消费者直接消费该地址得服务, 如果设置了该配置,则不会从zookeeper获取服务提供者地址,而是会直接访问这个地址的服务
		owner: 该消费者负责人,默认为空, 直接取application中的owner显示, 该属性是配合agent使用,直接使用loom的用户不需要关系和填写.

- 注意: 
  从之前的通过lvs, HAProxy等软负载去消费服务的方式, 迁移到通过loom来消费服务的方式的过程, 建议分为两个过程
    - 指定surl地址为之前的软负载地址, 运行一段时间.
    - 在第一个步骤稳定运行一段时间之后, 再将surl置空, 通过loom来消费服务, 如果生产环境有问题, 请立即将surl再指回软负载地址. 

	
- 对外提供服务
  新建实现类，实现接口 loom:service节点中api配置的值的.iface 例如：
  ```
  @Service
  public class MemberServiceImpl implements MemberServ.Iface
  ```
  此类名必须和loom:service节点里面的ref值保持一致。（ref值第一个字母要小写）

		
- 调用外部服务
在需要调用的类的上方添加如下代码，注入thrift接口。
  ```
  @Resource(name = "orderServReference")
  OrderServ.ServiceIface orderServReference;
  ```
name值必须是 loom:reference节点的id。尖括号里面是要调用的服务类名

使用方法如下：
可以通过orderServReference直接获取thrift接口中的方法.

  同步调用：
  ```
  String res = Await.result(orderServReference.方法名(参数));
  ```

  同步调用可以设置超时时间:
  ```
  String res = Await.result(orderServReference.方法名(参数),new Duration(10 * Duration.NanosPerSecond()));//十秒超时
  ```

  异步调用：
	```
	orderServReference.方法名(参数).addEventListener(new FutureEventListener<String>() {//这里的泛型为返回值类型
		//如果成功,res为该方法的返回值, 可以拿到该值做后续的业务处理
		@Override
		public void onSuccess(String res) {
		}

		//如果失败
		@Override
		public void onFailure(Throwable throwable) {
		}
	});
	```
	
- 记录某个服务提供者方法的每分钟的评价调用时长
  我们在spring配置文件中又一个service结点,其中的ref属性中的业务逻辑实现类.在该类中的方法上添加@InvokeTimeTrace 则loom可以和eagleye配合收集该方法每分钟的调用次数,平均时长等.
	

	
## loom-admin使用说明
loom-admin项目是loom中的子项目, 用来对注册中心中的服务提供者, 消费者进行管理. 下面是针对该管理中心的核心功能介绍

- 首页应用关系展示	
![应用调用关系图](http://siye1982.github.io/img/blog/loom/relationships.jpg)	

	
- 服务提供者管理

![提供者管理](http://siye1982.github.io/img/blog/loom/providers.png)

当某个服务集群启动时, 会将各自服务提供者的地址注册到zookeeper上, 这时我们可以在提供者维度中看到所有服务的提供者.如上图:

  (1) 在大促前期, 如果我们需要估算生产服务器的负载容量, 我们可以通过对某一个服务提供者进行倍权操作,并随时观察该服务器的负载情况和服务的提供情况,当出现性能瓶颈时,这时将是该服务提供者的最大容量,我们可以根据该容量反推出我们需要多少服务器来应对大促情况下的高并发请求.

  (2) 当发现某一个服务器的负载明显高于其他服务提供者,则可以将该服务提供者进行半权操作, 以此来降低该服务器的服务请求.降低提供服务出错的几率.

  (3) 当要对服务进行发版或者发现严重问题时, 需要先在管理界面上将该服务提供者进行禁用, 然后再停止服务, 这样可以避免因为服务已经停止,还有链接访问该服务并造成出错的可能.

  (4) 可以直接对某一个字段进行实时检索.


- 服务消费者管理

![提供者管理](http://siye1982.github.io/img/blog/loom/consumers.png)

我们可以在消费者维度查看某个服务都被哪些应用消费了,并且可以定位到具体的ip,应用名. 当发现某个服务消费者出现了问题,可以通过管理界面来屏蔽某个具体的消费者.

- 另外我们为了更方便的进行查询各个服务直接的关系,还提供了其他维度的查询,比如: 应用维度; 机器维度; 服务维度. 
    
- 针对某一个生产者实例或者消费者实例后面的总调用详情,可以链接进入具体的方法调用详情,如下图:

![调用详情入口](http://siye1982.github.io/img/blog/loom/detailEntry.png)
    
![方法级别的调用详情](http://siye1982.github.io/img/blog/loom/invokeDetail.png)


- 可以总览消费者或者生产者中所有方法执行时长的统计, 如下图:

![方法耗时总览](http://siye1982.github.io/img/blog/loom/invokeTimeout.jpg)
    


## loom常见问题汇总

- Java提供服务, ruby开启zipkin功能,则无法调用成功, 关闭则可以调用成功.
类似异常如下:
    ```
    WARNING: Unhandled exception in connection with /192.168.1.12:12343 , shutting down connection
    java.lang.NoSuchMethodError: org.apache.commons.codec.binary.Base64.encodeAsString([B)Ljava/lang/String;
            at com.twitter.util.Base64StringEncoder$class.encode(StringEncoder.scala:17)
            at com.twitter.util.Base64StringEncoder$.encode(StringEncoder.scala:25)
            at com.twitter.finagle.zipkin.thrift.RawZipkinTracer$$anonfun$createLogEntries$1.apply(RawZipkinTracer.scala:110)
            at com.twitter.finagle.zipkin.thrift.RawZipkinTracer$$anonfun$createLogEntries$1.apply(RawZipkinTracer.scala:106)
            at com.twitter.util.Try$.apply(Try.scala:13)
            at com.twitter.util.Future$.apply(Future.scala:79)
            at com.twitter.finagle.zipkin.thrift.RawZipkinTracer.createLogEntries(RawZipkinTracer.scala:106)
            at com.twitter.finagle.zipkin.thrift.RawZipkinTracer.logSpan(RawZipkinTracer.scala:118)
            at com.twitter.finagle.zipkin.thrift.RawZipkinTracer.mutate(RawZipkinTracer.scala:142)
            at com.twitter.finagle.zipkin.thrift.RawZipkinTracer.annotate(RawZipkinTracer.scala:234)
            at com.twitter.finagle.zipkin.thrift.RawZipkinTracer.record(RawZipkinTracer.scala:153)
            at com.twitter.finagle.zipkin.thrift.SamplingTracer.record(ZipkinTracer.scala:85)
            at com.twitter.finagle.tracing.Trace$$anonfun$uncheckedRecord$1.apply(Trace.scala:207)
            at com.twitter.finagle.tracing.Trace$$anonfun$uncheckedRecord$1.apply(Trace.scala:207)
            at scala.collection.LinearSeqOptimized$class.foreach(LinearSeqOptimized.scala:59)
            at scala.collection.immutable.List.foreach(List.scala:76)
            at com.twitter.finagle.tracing.Trace$.uncheckedRecord(Trace.scala:207)
            at com.twitter.finagle.tracing.Trace$.record(Trace.scala:248)
            at com.twitter.finagle.thrift.ThriftServerTracingFilter$$anonfun$apply$2$$anonfun$apply$4.apply(ThriftServerFramedCodec.scala:237)
            at com.twitter.finagle.thrift.ThriftServerTracingFilter$$anonfun$apply$2$$anonfun$apply$4.apply(ThriftServerFramedCodec.scala:236)
            at com.twitter.util.Future$$anonfun$map$1$$anonfun$apply$4.apply(Future.scala:823)
            at com.twitter.util.Try$.apply(Try.scala:13)
            at com.twitter.util.Future$.apply(Future.scala:79)
            at com.twitter.util.Future$$anonfun$map$1.apply(Future.scala:823)
            at com.twitter.util.Future$$anonfun$map$1.apply(Future.scala:823)
            at com.twitter.util.Future$$anonfun$flatMap$1.apply(Future.scala:786)
            at com.twitter.util.Future$$anonfun$flatMap$1.apply(Future.scala:785)
            at com.twitter.util.Promise$Transformer.liftedTree1$1(Promise.scala:95)
            at com.twitter.util.Promise$Transformer.k(Promise.scala:95)
            at com.twitter.util.Promise$Transformer.apply(Promise.scala:104)
            at com.twitter.util.Promise$Transformer.apply(Promise.scala:86)
            at com.twitter.util.Promise$$anon$2.run(Promise.scala:326)
            at com.twitter.concurrent.LocalScheduler$Activation.run(Scheduler.scala:186)
    		at com.twitter.concurrent.LocalScheduler$Activation.submit(Scheduler.scala:157)
            at com.twitter.concurrent.LocalScheduler.submit(Scheduler.scala:212)
    ```
    
可能存在java中的codec包版本不兼容问题, 解决办法:
在java服务提供方程序中的pom.xml中引入:
    ```
    <dependency>
        <groupId>commons-codec</groupId>
        <artifactId>commons-codec</artifactId>
        <version>1.8</version>
    </dependency>
    ```
    
- loom-admin 每隔6秒报类似decode错误时,类似如下:
```
Failed to retry subscribe {admin://192.168.1.100/h5wap-test?category=providers,consumers,routers,configurators&check=false&classifier=*&enabled=*&group=*&interface=h5wap-test&version=*=[com.xxx.loom.admin.model.RegistryServerSync@5fc625c3]}, waiting for again, cause: Failed to subscribe admin://192.168.1.100/h5wap-test?category=providers,consumers,routers,configurators&check=false&classifier=*&enabled=*&group=*&interface=h5wap-test&version=* to zookeeper zookeeper://192.168.10.66:2181?application=loom-admin&backup=192.168.10.57:2181,192.168.10.58:2181&timeout=20000, cause: URLDecoder: Illegal hex characters in escape (%) pattern - For input string: "co" #####[LOOM]#####  - [com.xxx.loom.registry.zookeeper.ZookeeperRegistry]
com.xxx.loom.core.rpc.RpcException: Failed to subscribe admin://192.168.1.100/h5wap-test?category=providers,consumers,routers,configurators&check=false&classifier=*&enabled=*&group=*&interface=h5wap-test&version=* to zookeeper zookeeper://192.168.10.66:2181?application=loom-admin&backup=192.168.10.57:2181,192.168.10.58:2181&timeout=20000, cause: URLDecoder: Illegal hex characters in escape (%) pattern - For input string: "co"
```

请检查相应服务节点下的所有节点url进行encode时是否正确.
    
- 如果服务消费者,报类似: "服务消费者, 通过服务名: [promotion-serv]  中的 thrift api [com.xxx.finagle.thrift.promotion.PromotionServ]  的代理调用失败, 请确认服务提供者是否启动成功 !!!!"
    这可能是没有服务提供者.
    也可能是网络问题,导致消费者不能访问到提供者.
    
- finagle在代理中使用aop的问题
   场景描述：
    （1）同一个接口中存在两个方法，A和B。A方法有延迟，B方法正常。
    （2）对该接口（返回值是future层）添加aop代理
    （3）在aop中对返回值进行处理。
    （4）调用A方法时正常，并发调用B方法时，线程在10个左右的时候，总会出现个别线程的返回值在等待A方法执行完毕。

   问题原因：
     在对future坐Aop代理时，方法返回值是future类型。此类型是不堵塞线程的。在aop代理类中认为此线程已经结束，实际上又用该线程等待future的返回值。等调用B的时候，有可能用重复使用该线程，导致堵塞。
   
   解决办法：
    （1）不要在future层做代理。应该保持finagle的干净性。如果有需求，建议在实现类中做代理。
    （2）如果非要在future中加入代理，请使用future的回调函数addEventListener。
   修改后，此现象消失。
        
        
- spring版本问题：
   loom中使用的3.2.7，我们大多数应用使用的是3.2.1， 还有少部分使用的更早的版本；
   消息系统使用的是3.1.0，接入时，启动spring, 报错
   ```
   <loom:registry id="zk01" protocol="zookeeper" address="192.168.101.194:2181" timeout="20000"/>  
   ```
   中timeout没有getter, setter升级spring版本后解决

- service名称匹配问题
    server端注册service和客户端引用service的名称要一致：
  
  
- redis 版本冲突问题
  如果出现了redis的commons-pool问题, 请升级redis的jedis的jar版本.
  loom中默认是支持redis-2.1.0 及以上版本.
  如果你使用的是spring-data-redis ,请升级到如下版本
  ```
    <dependency>
        <groupId>org.springframework.data</groupId>
        <artifactId>spring-data-redis</artifactId>
        <version>1.4.2.RELEASE</version>
    </dependency>
  ```
  升级完之后, 配置参数少了 maxActive, maxWait 变成 maxWaitMillis
  如果你使用的是jedis, 请升级到redis-2.1.0及以上版本.
    

- 通过loom调用服务, 参数不能传null的问题
    请升级到1.1.3版本
    
- 服务消费端调用服务时, 出现了Unknown异常
    请升级到1.1.3版本
    因为finagle中的服务调用是Funture, 异步调用, 在1.1.3之前的版本, 在调用服务的代理中, 我会主动将target置为null, 如果服务调用过长, 异步返回时获取不到target,会引起这个问题
    
- 服务启动不报错, 也没有打印成功启动日志
  该服务的service id 名称跟spring容器中其他类实例的名称冲突, 导致服务实例被覆盖.
   
  类似日志如下:
  ```
   Overriding bean definition for bean 'expressWayBillStatService': replacing [Generic bean: class [com.xxx.loom.config.ServiceBean]; 
   scope=; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; 
   factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null] with 
   [Generic bean: class [com.xxx.express.stat.service.impl.ExpressWayBillStatServiceImpl]; 
   scope=; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; 
   factoryMethodName=null; initMethodName=null; destroyMethodName=null; 
   defined in file [/home/webuser/workarea/bill/target/classes/spring/ApplicationContext-dao.xml]]
  ```

- finagle-thrift和finagle-zipkin版本不一致的问题
  项目用的finagle-thrift_2.9.2-6.10.0.jar和finagle-zipkin_2.9.2-6.8.1.jar包.
  loom中引用的是finagle-thrift_2.9.2-6.20.0.jar和finagle-zipkin_2.9.2-6.20.0.jar包
  在配置了loom后该项目去掉了对finagle-thrift_2.9.2-6.10.0.jar包的依赖,而没有去掉对finagle-zipkin_2.9.2-6.8.1.jar的依赖
  所以打包后出现finagle-thrift_2.9.2-6.20.0.jar和finagle-zipkin_2.9.2-6.8.1.jar版本不一致的问题,导致下面异常.
	```
	SEVERE: A server service  threw an exception
	scala.MatchError: ServiceName(freightInsuranceServ) (of class com.twitter.finagle.tracing.Annotation$ServiceName)
	    at com.twitter.finagle.zipkin.thrift.RawZipkinTracer.record(RawZipkinTracer.scala:142)
	    at com.twitter.finagle.zipkin.thrift.SamplingTracer.record(ZipkinTracer.scala:85)
	    at com.twitter.finagle.tracing.Trace$$anonfun$uncheckedRecord$1.apply(Trace.scala:234)
	    at com.twitter.finagle.tracing.Trace$$anonfun$uncheckedRecord$1.apply(Trace.scala:234)
	    at scala.collection.LinearSeqOptimized$class.foreach(LinearSeqOptimized.scala:59)
	    at scala.collection.immutable.List.foreach(List.scala:76)
	```
  
  解决办法:
  在该系统中删掉对finagle-zipkin_2.9.2-6.8.1.jar的依赖,全部用loom中对这两个包的依赖.
    
    
- spring配置文件中, 通过imago获取服务名时, 在loo:service节点中的id属性使用el表达式报错的问题.
  具体异常,类似如下:
  ```
Caused by: org.xml.sax.SAXParseException; lineNumber: 39; columnNumber: 22; cvc-datatype-valid.1.2.1: '#{configManager.getConfigValue('xxx_public_service_name','trade_subject_serv_2.0')}' 不是 'NCName' 的有效值。
 	at com.sun.org.apache.xerces.internal.util.ErrorHandlerWrapper.createSAXParseException(ErrorHandlerWrapper.java:198)
 	at com.sun.org.apache.xerces.internal.util.ErrorHandlerWrapper.error(ErrorHandlerWrapper.java:134)
 	at com.sun.org.apache.xerces.internal.impl.XMLErrorReporter.reportError(XMLErrorReporter.java:437)
 	at com.sun.org.apache.xerces.internal.impl.XMLErrorReporter.reportError(XMLErrorReporter.java:368)
 	at com.sun.org.apache.xerces.internal.impl.XMLErrorReporter.reportError(XMLErrorReporter.java:325)
 	at com.sun.org.apache.xerces.internal.impl.xs.XMLSchemaValidator$XSIErrorReporter.reportError(XMLSchemaValidator.java:458)
 	at com.sun.org.apache.xerces.internal.impl.xs.XMLSchemaValidator.reportSchemaError(XMLSchemaValidator.java:3237)
 	at com.sun.org.apache.xerces.internal.impl.xs.XMLSchemaValidator.processOneAttribute(XMLSchemaValidator.java:2832)
 	at com.sun.org.apache.xerces.internal.impl.xs.XMLSchemaValidator.processAttributes(XMLSchemaValidator.java:2769)
 	at com.sun.org.apache.xerces.internal.impl.xs.XMLSchemaValidator.handleStartElement(XMLSchemaValidator.java:2056)
 	at com.sun.org.apache.xerces.internal.impl.xs.XMLSchemaValidator.emptyElement(XMLSchemaValidator.java:766)
 	at com.sun.org.apache.xerces.internal.impl.XMLNSDocumentScannerImpl.scanStartElement(XMLNSDocumentScannerImpl.java:355)
  ```
  
  如果有类似异常, 是说xml验证时, ID类型的属性不支持特殊字符(el表达式是以"#"开头). 
  请升级到1.1.5-SNAPSHOT及以上版本
     
     
- 服务端抛出类似  InvocationTargetException 的异常.
  在利用 Method 对象的 invoke 方法调用目标对象的方法时, 若在目标对象的方法内部抛出异常, 会抛出 InvocationTargetException 异常, 该异常包装了目标对象的方法内部抛出异常.
  这可能是服务端的业务方法往上抛出了异常, 这个异常被loom框架捕获,但是没有将异常堆栈详情打印出来, 在1.1.6-SNAPSHOT 及以上版本会针对该异常做出特殊处理, 会将targetException的详细信息输出.
   
- 用thrift短连接调用finagle出现客户端接收不到返回值的bug
  描述:服务端用loom提供服务,客户端用thrift短连接调用,调用一段时间(随机)会出现客户端读取不到返回值的情况
  
  原因:loom里面用到的commons-codec包和commons-httpclient里面用到包版本不一致造成的.
    
  解决办法:将commons-httpclient里面的包排除掉commons-codec解决问题.



## loom版本变更列表

	1.2.1-SNAPSHOT
	1. 统一方法调用统计信息格式为JSON格式
	2. 修复统计信息中的bug
    
    1.2.0-SNAPSHOT
    1. 增加每个方法的调用总次数,一分钟生产力, 平均调用时长的统计功能.
    2. 增加消费者维度,调用远程thrift接口的时长统计, 并将超过200ms的调用打印warn日志,可以结合eagleye进行实时预警.
    3. 增加服务提供者,每个api接口的调用时长统计, 并将超过200ms的调用打印warn日志,可以结合eagleye进行实时预警.
    4. 给application节点增加owner属性, 可以简单标识应该的负责人, 同时可以在loom-admin中消费者维度看到每个消费者的负责人.
    5. 在reference结点上添加owner属性(只是为了配合agent使用,直接只用loom的用户不需要关心).
    6. 在消费者启动逻辑中添加是否是agent判断,如果为agent,则不管是否启动成功都进行注册(直接使用loom的用户不需要关系该细节)
    7. 首页关系图展示, 添加直属关系检索
    8. 增加服务端提供服务时,服务超时时间, 默认30秒
        
    1.1.6-SNAPSHOT
    1. 修复loom:reference节点中timeout无效的bug
    2. 修复loom服务端业务方法抛出异常,只显示InvocationTargetException,并没有详细信息的问题
    
    1.1.5-SNAPSHOT
    1. 修复loom:service 的id不能使用el表达式的问题
    
    1.1.4-SNAPSHOT
    1. 修复通过redis获取心跳时, 当redis不稳定时,不能自动回复的问题
    
    1.1.3
    1. 修复服务调用时间过长抛出Unknown错误的问题
    
    1.1.2
    1. 修复消费者调用服务方法时, 不能传递为null的参数.
    
    1.1.1
    1. 应用kill时主动删除zookeeper上的providers下的节点, 提高其他watch该节点的响应.
    2. 增加服务从启动开始成功调用的总次数和失败总次数.
    3. 如果调用失败, 记录具体请求哪个provider请求失败.
    4. 修复消费者列表,提供者列表详情×显示X的bug.
---


文章题图: 来自冒险岛数据库系统

<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>