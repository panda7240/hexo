title: HAProxy与Nginx关于动态配置的测试
date: 2015-11-03 09:33:52
categories: 服务治理
tags: 
- 转载
- HAProxy
- Nginx
- 分布式
---

## 写在前面

目前公司在做基于thrift的服务治理, 目前有针对Java的服务治理框架, 其他语言也想接入到该框架中.现在有两种方案来进行接入, 分别如下:

* 各个语言开发客户端来接入到现有框架中, 这就需要各个需要拿到服务地址列表进行后续的连接维护和负载均衡.
* 非Java语言通过接入HAProxy或Nginx来接入框架, 这就需要我们可以动态加载HAProxy的配置.

<!--more-->

针对第二个方案,公司运维的同事做了如下的调研, 我现在将这位同事做的调研过程转载这里, 并声明本文大部分内容不是本人原创.	

## 整理的测试过程

### Nginx和HAProxy在Tcp方面代理的情况。
下面是对这两个软负载进行简单的对比

* 从性能上面来说没有明显的区别.
* Nginx 的tcp 的代理是有第3方模块配置，也有官方配置。
* http 的目前有些开源模块可以支持动态加载，但是tcp 的目前支持的效果不是很好，稳定性也不能保证。
* 官方 在 tcp 和http 都有非常好的模块来支持了，但是进一步了解这些功能只开放给商业版的，
 
所以Nginx 在tcp 的这块的功能暂时不继续往下研究了，等这个月底，看看 openresty 的作者在北京的开源大会中会不会提及他的动态模块了。

因此 本次的测试主要是HAProxy.之前HAProxy 使用上面只是做了简单的代理而已，在重视度上面不够高，所以不够深入。所以目前线上的版本是HAProxy1.4.，有点老的版本了。由于要做动态的配置，所以1.4 是不支持的。我大致的看了下HAProxy官方文档。包含的稳定版 1.5 和1.6的功能。（1.6找到了可以动态切换ip的方式）发现一些可以提升性能的参数,比如:

* cpu-map 类似Nginx 的 worker_cpu_affinity   ，可以捆绑cpu核，增加cpu亲缘性
* peers 功能， 可以做成集群master，主从模式，在 session会话保持同步中，其他HAProxy 可以将session的内容从master复制并且保留， 在master异常后，接管master的服务，这个功能不知道我们会不会用到 。但是感觉以后会有帮助，所以先给大家说说而已，粗略的说明下，就不细讲了。
* 支持配置变更后自动发邮件
* 支持更加复杂的健康检查功能
* HAProxy 1.6.+ 开始引入了dns 功能, 下面是 dns 配置的一个例子，也是我们使用动态切换配置的核心功能： 

  ```
	resolvers mydns
	nameserver dns1 10.0.0.1:53
	nameserver dns2 10.0.0.2:53
	resolve_retries       3
	timeout retry         1s
	hold valid           10s
	```

目前测试了3种方案：
	
* 通过api 动态添加和删除节点。
	* 优点：简单，稳定，高效
 	* 缺点：不支持多进程下（没有Nginx中的全局变量，无法统一管理），如果要使用需要启动很多的HAProxy。看上去不优雅.

* reload 配置文件：
	* 优点：简单，轻松管理所有服务
  	* 缺点:
  		* 平滑reload （-sf 模式），代码建立的长连接会导致老进程无法释放，频繁reload 会造成如下的情景（如果代码的在长连接中有一定的设计和规范，这个情况应该可以避免)
  		![haproxy1](http://siye1982.github.io/img/blog/haproxy/haproxy1.png)
  		
		* 释放长连接的reload  （-sf模式），会导致长连接全部断开，如果代码没有重试机制的话，可能当前的请求就直接报错了,而且1.4以下的版本，这样reload 在大并发下会出现比较长的卡顿，影响服务，1.5+的版本对这个功能优化了， 也仍然有一定的卡顿，下面的文章是国外的团队的解决方案（此方案涉及网络层方面，我对这块不是很懂，而且双11项目太多，我尽力不够，也无法使测试和其他人一起配置验证效果，因此可以等我们开会后，确认是否测试此功能）：
		  http://www.codeceo.com/article/HAProxy-reload.html
               
大家看完上述的方案后， 如果可以接受， 用 释放长连接的方案也是可行的。但是需要在代码层做统一的规范，需要良好的设计， 而且国外很多大型架构都有这样的影子，做的非常好的。如下面：
https://github.com/QubitProducts/bamboo

当然我们还是需要依据业务的场景选择。下面是架构方案:
![haproxy1](http://siye1982.github.io/img/blog/haproxy/haproxy2.png)

----

* dns动态解析ip
	* 优点：动态解析过程中长连接不会被断开， HAProxy有内部缓存和保护机制，性能稳定.
	* 缺点：需要设计好 整个HAProxy的 配置文件，如果域名添加过少，新增域名的时候需要reload配置文件，会产生波动。所以要预先生成整个配置文件，从业务和架构层面减少reload 的可能性.

* 下面是一个简单的设计过程：
	* 每个服务，设置10个域名，代理10个ip(ip有可能是相同的).如果从zookeeper只获取到2个节点，就分别解析成10个域名，5/5对应节点，如果是 3个节点，就3/3/3对应节点，剩下一个随机分配.依次的算法结构， 
	* 问题：是机器的权重不能完全的平均，不过就实际业务权重只要差距不大，是没有影响的。如果节点超过10个了，那么就需要reload 配置文件了，这个操作建议在低峰做（如果长连接设计的好，高峰期问题也不大）
  	* 下面是的操作测试的数据
		* HAProxy 会缓存dns 的服务，如果dns全部出现异常后，HAProxy会使用之前的缓存。直到dns恢复.
		* 当其中一个dns解析的域名可用，但是端口不可用用的时候， HAProxy也会检查出有问题，
然后使用第2个dns去解析。如果都无法使用，使用之前的缓存
		* 其中一个dns 解析的ip变换了， HAProxy 发现变动的ip，进行动态切换。
    	* 官方默认是 10s 去检查一次dns是否有变动，检查时间不到，使用内部缓存，性能会很好，时间可以根据我们的业务需求进行改变。
     	* 另外说明一个情况， 由于多进程下的HAProxy，并不是共享内存的管理机制（Nginx有这个优点），
所以HAProxy去检查dns变化的时间不是同时发生的，有一定的时间差距， 我可以在测试的时候给大家演示一边。 这样会导致时间有误差,如下图:

		![haproxy1](http://siye1982.github.io/img/blog/haproxy/haproxy3.png)
		
		![haproxy1](http://siye1982.github.io/img/blog/haproxy/haproxy4.png)

		切换一个ip 的时候 是这样,如下图:
		
		![haproxy1](http://siye1982.github.io/img/blog/haproxy/haproxy5.png)

		也就是说dns 轮询是每个进程去同时会检查2次，为什么是2次呢，目前还没有搞清楚原因
 
  		* 验证长连接，在dns 切换的过程中是否会出现断开现象。
        验证结果：不会断开长连接
  		* 验证：如果dns 挂了， host配置是否会生效。
验证5  的功能 （ 如果dns 挂了， host配置是否会生效）
			* dns存活的情况下， 如果配置上了hosts. 如果dns和hosts的同一个域名，但是ip对应的不一致，会出现这样的现象: HAProxy启动的时候会先根据hosts文件进行配置，然后进入内存管理后，在dns进行查询，切换ip（出现这种现象的原因： linux默认是先从hosts走，可以修改这个配置， 我修改配置后，默认就先从dns服务走了， 文件是 /etc/nsswitch.conf）。如果dns和hosts的同一个域名，ip也一致，没有任何变化, 如下图: 
			![haproxy1](http://siye1982.github.io/img/blog/haproxy/haproxy6.png)
			* 如果dns挂了，HAProxy直接就使用了hosts文件，没有任何的日志显示，貌似直接就用了hosts去了（不怎么友好）. 但是要生效hosts必须reload （-st方式） 配置文件，否则HAProxy依然会使用之前dns解析的ip进行服务,日志会显示如下pause mode ，这个时候会出现短暂的卡顿，长连接也会断开,如下图:
     		![haproxy1](http://siye1982.github.io/img/blog/haproxy/haproxy7.png)
			* 如果 hosts已经生效了， 这个时候dns恢复了， 发现一个bug：HAProxy进程显示已经切换了到了dns。但是访问的仍然是hosts的文件，这个地方可以和官方提个bug。
			解决办法： ```-st 模式reload HAProxy```, 如下图:
			![haproxy1](http://siye1982.github.io/img/blog/haproxy/haproxy8.png)
			
			
----

### 总结
HAProxy dns动态解析的功能，基本上能满足我们的需求，但是需要在配置文件上面进行良好的设计，如果设计的不好，使用过程中reload的几率就会加大。

在dns出现异常的时候， hosts也可以做为备用功能进行切换，可以做到双保险。



<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>

