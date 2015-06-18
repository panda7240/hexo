title: EAGLEYE监控预警平台
date: 2015-06-18 22:20:11
categories: 监控预警
tags:
- Elasticsearch
- 监控
- Java
- logstash
---


## 概述

    监控预警平台, eagle + eye (鹰眼)的合体词, 寓意可以快速发现问题, 并及时作出响应

        Eagleye整个分为:
        
        1. eagleye-java-client: 针对java程序进行日志收集
        
        2. eagleye-alerter 根据控制台设置的预警规则, 对匹配日志进行实时预警.
        
        3. eagleye-admin
          * 应用管理
          * 用户管理
          * 规则管理
          * 预警设置
          * 日志检索
          * 日志统计
        



## 需求

### 应用App

    描述：应用app是每个系统的唯一标识
    
    一、应用app列表（有分页信息）
        1、应用名称
        2、负责人
        3、相关人员：本组的开发人员
        4、最后更新时间
        5、备注
        
    二、添加：弹出添加窗口
      输入框：
        1、应用名称：此处是英文名称，和loom里面的appId保持一致。必须填写，不能为空。
        2、备注：描述该系统（400字以内）。
      按钮：
        1、保存：保存数据到数据库，并关闭弹窗
        2、取消：关闭弹窗
        
    三、编辑：弹出编辑窗口，自动填充相关信息。
        所有内容都可以修改
        功能同添加功能。
    
    四、删除：删除所选应用，在数据库中逻辑删除，如果已经绑定了预警设置，直接删除与其绑定的预警设置。
    
    五、搜索：根据应用名称和负责人搜索。
    
    
    问题: 1. 负责人, 相关人员是否有必要去维护, 如果某个人离职, 个关系怎么维护.
         2. 是否应该有通过app,跳转到app绑定了哪些用户组, 有哪些规则.
         3. log4j中appender的代码提示.
    
### 用户

    描述：此用户信息有两个功能，一是用户登录用，二是发送报警信息的对象。
   
    一、信息列表（有分页信息）
		1、登录名
		2、用户名
		3、手机
		4、邮箱
		5、用户类型：管理员、读写、只读
		6、最后更新时间
		7、状态：正常、禁用。
		8、操作：修改密码、禁用、启用
	
	二、添加：弹出添加用户的窗口
		账号、用户名、密码，邮箱、手机是必填项，备注是选填项。
		1、保存：保存数据到数据库，并关闭弹窗
		2、取消：关闭弹窗

	三、编辑：弹出编辑页面
		只能修改用户名，手机、邮箱、用户类型、备注。
		1、保存：保存数据到数据库，并关闭弹窗
        2、取消：关闭弹窗
        
	四、删除：删除所选用户，逻辑删除，如果已经绑定了预警功能，则直接删除和改用有关的所有关系，admin账户不能删除。
	
	五、禁用、启用：用户的状态分为正常和和禁用两种状态。禁用时不能登录，不能接受预警信息。admin账户不能禁用。
	
	六、修改密码：点击打开修改密码页面，在修改页面不用输入老的密码，直接输入两次密码，如果一致并提交，就修改成功。
	
	七、搜索：在列表页有一个搜索框，输入可以根据登录名，用户名，邮箱，手机，备注进行模糊搜索用户。

### 用户组（不做）

    描述：每个用户组包含多个用户，方便在绑定预警规则的时候绑定多人。
    
    一、信息列表
        1、组名：用户组的名字（简单明了，不能为空，不能超过30个字）
        2、用户数量：统计此用户组的用户数量
        3、最后更新时间。
        4、备注
        5、包含用户：点击加号展开，所有用户名，用逗号隔开。
        
    二、搜索：根据用户组名搜索。
    
    三、添加：添加用户组。
      输入框：
       1、组名：
       2、备注（400字以内）
      按钮：
       1、保存：保存数据到数据库，并关闭弹窗
       2、取消：关闭弹窗

    四、编辑：和添加功能相同
    
    五、删除：逻辑删除。如果绑定了预警规则，自动解除和预警规则的绑定。
    
    六、编辑用户：弹窗打开所有用户信息，用复选框显示
       按钮：
         1、保存：保存数据到数据库，并关闭弹窗
         2、取消：关闭弹窗
    
### 预警规则
    
    描述：定义触发什么条件的时候开始报警。
    
    一、列表：点击规则时下方根据选中的规则显示绑定的用户组和用户信息。
        1、规则名称
        2、关键字：包含此关键字时开始报警，可以是多个，多个之间用英文的逗号隔开
        3、最后更新时间
        4、操作：（编辑，删除，禁用、启用）
        5、备注：点击加号显示
        
    二、搜索：根据搜索字段：规则名称，关键字。
    
    三、添加：弹窗添加预警规则
        1、预警规则名称：文本框输入，长度不超过50字
        2、关键字：文本框输入，长度不超过400字。
        3、备注：长度400限制。
      按钮：
        1、保存：保存数据到数据库，并关闭弹窗
        2、取消：关闭弹窗
        
    四、编辑：弹窗打开编辑窗口，自动填充相关信息，功能同添加界面。
    
    五、删除：逻辑删除此规则，如果已经绑定了预警设置，直接删除与其绑定的预警设置。
    
    六、禁用、启用：禁用后预警时忽略此预警信息。

### 预警设置

    描述：此功能绑定app和规则的关系，并设置违反此规则的时候给那些用户发送短信或者邮件提醒。
    
    一、列表：
        1、app名称
        2、规则名称
        3、最后更新时间
        4、绑定短信用户：点击弹窗选择接受短信的用户
        5、绑定邮件用户：点击弹窗选择邮件接受预警的用户。
        5、操作：删除
    二、 添加：弹窗打开添加窗口（规则和app关系不能重复）
        1、规则：下拉框，必选，
        2、应用app：下拉框，必选，同一个规则下唯一，支持搜索
        按钮
         1、保存：保存数据到数据库，并关闭弹窗
         2、取消：关闭弹窗
    三、编辑：修改其绑定关系
    
    三、删除：直接删除绑定关系。删除何其关联的发送信息设置。
        
### 预警
    
    一、根据预警规则收到日志后分析
    
    二、发送短信和邮件日志
       
       1、接受者
       2、发送时间
       3、发送类型：（短信，邮件）
       4、发送内容：（长度10000）

### 发送日志记录
### 感知有多少app在使用该系统

### 日志检索

    多维度检索: 
       1. 按照时间区间
       2. 日志内容
       3. 应用

### 日志统计

    1. 按照app的异常数据进行统计排名
    2. 按照具体的异常信息进行统计排名



## 详细设计

### Eagleye 整体架构图

  ![eagleye,整体架构图](http://siye1982.github.io/img/blog/eagleye/architecture.jpg)
  
    eagleye作为所有服务的日志监控预警平台, 需要收集所有后台服务的全量日志.
    
    主要分成如下几个部分:
    1. 各个java服务需要接入的基于log4j appender的日志收集客户端(eagleye-java-client)
    
    2. 我们统一通过Kafka作为日志传输通道, 因为要收集的是所有服务的全量日志,日志量庞大,Kafka非常适合这种分布式收集的场景, 并且可以很容易的进行水平拓展.
    
    3. 我们通过eagleye-alert 从kafka中获取日志, 并根据eagleye-admin中设置的预警规则进行实时预警, eagleye-alert支持水平拓展,并可以做到高可用.
    
    4. 通过logstash进行全量日志的收集并存储到elasticsearch中进行存储, 最终用kibana进行快速的检索和统计.
    
    5. eagleye-admin是eagleye架构的管理控制台, 从这可以进行各种预警规则的配置, 监控各个client, alerter的心跳, es的存储状态, 提供日志检索,统计界面.
       eagleye-admin作为一个管理控制台,并不需要高可以,所以只需要部署一个实例, 即使admin不可用也不会影响所有日志的收集预警功能.
       
    6. 所有服务器的心跳, 预警规则, 都是通过redis进行通信.

### 日志收集,存储架构图

  ![日志收集,存储架构图](http://siye1982.github.io/img/blog/eagleye/collect_storage.jpg)
  
#### 日志可靠性说明
 
    因为将来会涉及到交易流程的日志跟踪, 所以日志的可靠性需要保证.
    
    鉴于之前推动zookeeper的dns设置过程(运维人员需要每台服务器去修改,推动时,不是这台服务器没有设置,就是那台服务器没有设置), 日志收集任务前期在服务端进行日志采集的过程, 并没有引入第三方的收集工具.
    
    在服务端进行日志采集方式, 采用自定义log4j的appdender方式接入采集点.通过kafka作为日志传输通道.
    
    当然, 采用这种方式有一个风险点是,加入redis不可用时,日志将采集不到. 但是会记录到本地log日志中,并不会丢失. 
    
    服务端进行日志采集,也有高可用方案,可以每台部署服务的机器都部署logstash进行收集,即使logstash不可用,日志也不会丢失,并且,logstash启动时会将丢失收集的日志重新采集.
    
    但是logstash部署在服务端所在的机器,对前期日志采集工作量比较大,并不适合我们前期小步迭代推动eagleye项目.所以我们前期约定,在极端情况下,我们允许小概率收集日志失败的可能.
    
    因为采用的是redis的队列方式进行传输,所以并不影响我们将来更替高可用的日志采集方案.
    
  
#### 日志中继说明

    日志中继器的存在,有利于我们水平拓展日志收集,并灵活指定存储媒介.
    
    logstash可以灵活的往es, hdfs, mysql, cvs, local file 等存储媒介, 并可以灵活的对日志内容进行过滤.
    
    logstash相比flume会更轻量一些,并且是elasticsearch的同源产品.
    
#### 日志存储

    我们选择elasticsearch进行存储, 当然也可以随时切换成hdfs进行存储, elasticsearch提供了丰富的RESTFul风格的api,和语言无关.
    
    es的集群搭建成本非常低,每一个es服务会自动发现集群并加入进去.
    
#### 日志检索, 统计

    因为有Kibana的存在, 我们可以很轻易并且快速的从es中检索出我们想要的日志. 可以从多维度进行日志的统计.
    
    关键是这些功能的提供, 我们不需要写任何一行代码.
  
#### 备注

    上图中的channel, relayer, es 可以部署在相同的机器中,也可以进行拆分. 每一层都可以选用其他方案进行替代,并不影响整个系统的运转. 




## 测试

### 功能测试


### 性能测试

#### logstash 性能测试:
    单台服务器的单个logstash实例
    内存:500m
    input consumers : 1
    output workers : 2
    机器内存: 4G, 其中2个G被es消耗, 1个多G被日志mock程序占用
    机器8核,普通pc机
       
    在output配置中
    workers => 2 # 如果这个配置开太大容易内存溢出, 需要根据机器配置进行调整, 在测试环境logstash 内存500m时,2个线程效率最高
    flush_size => 1000 #该配置配合workers进行调整, 如果太大, 也容易内存溢出, 或者增加gc次数.目前这个配置可以达到单台logstash可以20000/s的日志写入
    
    gc观察
    在bin/logstash.lib.sh 文件中 logstash 采用了 如下gc方式
    JAVA_OPTS="$JAVA_OPTS -XX:+UseParNewGC"
    JAVA_OPTS="$JAVA_OPTS -XX:+UseConcMarkSweepGC"
    观察曲线如下图:
   ![logstash采用自己默认gc](http://siye1982.github.io/img/blog/eagleye/logstash-gc0.png)
           
    老年代gc会比较频繁,增加了stop the world的时间,会影响性能.
   
    注释掉之后的曲线老年代很少出现回收现象, 不会造成stop the world 现象     
   ![logstash采用jvm默认gc](http://siye1982.github.io/img/blog/eagleye/logstash-gc1.png)
           
     但是采用jvm默认gc方式后运行了1个小时之后出现了频繁的老年代gc, 如下图:
   ![logstash采用jvm默认gc](http://siye1982.github.io/img/blog/eagleye/logstash-gc2.png) 
          
     简单的测试观察之后, 还是建议使用logstash默认的gc
     JAVA_OPTS="$JAVA_OPTS -XX:+UseParNewGC"
     JAVA_OPTS="$JAVA_OPTS -XX:+UseConcMarkSweepGC"
     
#### eagleye-java-client 性能测试
    
    client往kafka中发送日志可以80000/s, 这取决与网络速度和硬盘速度
    
    运行时通过jmx观察内存,如下图: 没有一次老年代回收
   ![eagleye-java-client gc图](http://siye1982.github.io/img/blog/eagleye/eagleye-java-client-gc.png)

#### eagleye-alert 性能测试

    1. 单服务, 单机器(244)压力测试:
        
        cpu使用如下图:
        
![cpu](http://siye1982.github.io/img/blog/eagleye/单机器cpu.png)
        
        内存使用如下图:
        
![内存](http://siye1982.github.io/img/blog/eagleye/单机器内存.png)
        
        吞吐量:3.15w条message/s
        
![吞吐量](http://siye1982.github.io/img/blog/eagleye/单机器1.png) 
    
        2. 双服务, 双机器(244, 177)压力测试:
        
        177cpu使用如下图:
        
![cpu](http://siye1982.github.io/img/blog/eagleye/双机177cpu.png)
        
        244cpu使用如下图:http://siye1982.github.io/img/blog/eagleye
        
![cpu](http://siye1982.github.io/img/blog/eagleye/双机244cpu.png)
        
        177内存使用如下图:
        
![内存](http://siye1982.github.io/img/blog/eagleye/双机177内存.png)
        
        244内存使用如下图:
        
![内存](http://siye1982.github.io/img/blog/eagleye/双机244内存.png)
        
        177吞吐量:2.32w条message/s
            
        244吞吐量:3.22w条message/s
        
        发现单机处理时最大吞吐量 与双机处理时最大吞吐量相同 , 但是通过下面单机双实例测试可以确认服务器网络io不是瓶颈 , 
        初步认为是单个kafka topic 分区的生产力是有限的 ,    双机处理时吞吐量的差距是由于kafka topic 分区分配不均衡导致(244得到3个分区 , 177得到两个)
        修改kafka分区数到10个后, 发现单个程序实例即使分配到更多(5个或10个)分区, 吞吐量上限仍然是3w/s , 
        进一步测试得到的结果是程序的性能瓶颈在kafka消费者迭代获取数据部分, 且一个程序中运行多个针对单个topic的ConsumerConnector并不会提高程序吞吐量
        
        
        3. 单机(244)双实例
        程序1(吞吐量 1.68w条message/s , 分配到2个分区)
        程序2(吞吐量 2.9w条message/s , 分配到3个分区)
        
        总结:
        alerter程序启动时可以通过配置文件loomContext.xml配置alerterConsumerThreadNum(alerter程序kafka消费者线程数)
        一个kafka消费者线程可以消费多个到0个分区, 但一个分区最多只能被一个消费者线程消费
        新的消费程序加入或旧的消费程序失去连接会引发所有分片和所有消费者线程间的再平衡, 但是如果已经是每个分片都被不同的线程消费, 即分片和消费线程为1对1的关系时, 再加入消费程序, 该消费程序将会分配不到分区 导致该消费程序吞吐量为0
        单个程序实例的吞吐量在3w/s, 可以通过部署多个实例的方式实现水平扩容, 但需要确保新部署的实例会得到topic分区
            如线上服务topic有24个分区, 最多可部署alerter程序实例数为 24/alerterConsumerThreadNum, 即如alerterConsumerThreadNum配置为3, 最多可以部署24/3=8个alerter程序实例
        







## 部署

### kafka topic创建

    ./kafka-topics.sh -create --topic 'EAGLEYE_LOG_CHANNEL' --zookeeper 'spark:2181,witgo:2181' --partitions 10 --replication-factor 1
    ./kafka-topics.sh -create --topic 'EAGLEYE_ALERT_MAIL' --zookeeper 'spark:2181,witgo:2181' --partitions 10 --replication-factor 1
    ./kafka-topics.sh -create --topic 'EAGLEYE_ALERT_SMS' --zookeeper 'spark:2181,witgo:2181' --partitions 10 --replication-factor 1

### logstash 部署步骤

    logstash 需要有jruby1.7.11 版本支持
    
    1. 通过rbenv 对ruby环境安装:
    git clone git://github.com/sstephenson/rbenv.git ~/.rbenv
    # 用来编译安装 ruby
    git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
    # 用来管理 gemset, 可选, 因为有 bundler 也没什么必要
    git clone git://github.com/jamis/rbenv-gemset.git  ~/.rbenv/plugins/rbenv-gemset
    # 通过 gem 命令安装完 gem 后无需手动输入 rbenv rehash 命令, 推荐
    git clone git://github.com/sstephenson/rbenv-gem-rehash.git ~/.rbenv/plugins/rbenv-gem-rehash
    # 通过 rbenv update 命令来更新 rbenv 以及所有插件, 推荐
    git clone https://github.com/rkh/rbenv-update.git ~/.rbenv/plugins/rbenv-update
    然后把下面的代码放到 ~/.bash_profile 里
    export PATH="$HOME/.rbenv/bin:$PATH"
    然后重开一个终端就可以执行 rbenv 了.
    
    rbenv install 1.9.3-p551 # 安装 1.9.3-p551
    rbenv install jruby-1.7.11 # 安装 jruby-1.7.11
    
    rbenv global jruby-1.7.3 # 默认使用 jruby-1.7.3
    
    
    2. 下面从咱们的gitlab上获取已经安装好插件的logstash-1.4.2
    git clone git@git.tuan800-inc.com:trade_architect/logstash.git
    
    3.  因为logstash需要使用kafka, 所以需要在本机配置, 生产环境的kafka的hosts表
    
    172.16.11.43 YJ-18-1
    172.16.11.46 YJ-18-4
    172.16.11.60 YJ-17-4
    172.16.11.67 YJ-17-11
    172.16.11.73 YJ-16-3
    
    4. 按照logstash进行负载均衡存储的es实例
    
    (1). git clone git@git.tuan800-inc.com:trade_architect/elasticsearch.git
    
    (2). 在  elasticsearch-1.4.4/config/elasticsearch.yml 中设置将需要输出的数据, 日志路径.
    
       node.name: "eagleye_es_logstash_XXX"   其中XXX可以标识出该机器
    
       path.data: /home/zhejava/esdata/data   因为该实例不存储数据使用, 只是用来进行检索和存储时当做集群负载均衡器使用,
       path.work: /home/zhejava/esdata/work  存放临时文件的位置
       path.logs: /home/zhejava/esdata/logs  存放日志的位置
    
       transport.tcp.port: 9300 配置开启, 并修改为9301
    
      http.port: 9200 配置开启 , 并修改为9201
    
      node.master: false  开启配置
      node.data: false     开启配置
    
    (3). 在 elasticsearch-1.4.4/bin/elasticsearch 文件中的 ES_HEAP_SIZE=32768m 修改为 8192m 即可
    
    (4). 执行  elasticsearch-1.4.4/bin/elasticsearch -d
    
    5. 进入到 logstash-1.4.2 中执行  sh start.sh 即可, 在该文件中可以配置logstash具体的日志输出路径
    
    如果不想使用已经安装好插件的logstash请参照下面的方式安装.
    
    ----------------------------------------------------------------------------------------------------------------------------
    
    如果按照官方文档的部署可以git clone 最新版的logstash-kafka, 并且在该目录下执行 make tarball, 然后生成 logstash-1.4.2.tar.gz
        
    但是这需要有一个畅通的网络, 因为有一个无形的墙,阻挡我们完成这件事, 我个人尝试vpn出去, 也出现了失败, 网络并不稳定.
    
    所以我还是阐述一下通过手动搭建一个logstash 服务, 并可以从kafka中消费队列, 并存储消息到elasticsearch中.
    
    
    1. 下载官方logstash-1.4.2 版本, 这是最新的release版本.
       wget https://download.elasticsearch.org/logstash/logstash/logstash-1.4.2.tar.gz
       
    2. 下载 logstash 针对kafka的插件 logstash-kafka v0.6.2 (或者可以通过git clone获取最新的版本)
       wget https://github.com/joekiller/logstash-kafka/archive/v0.6.2.tar.gz  
       
    3. 因为logstash-kafka是ruby项目, 所以需要依赖ruby环境
    
    4. 解压logstash包,得到 $logstash_home, 并创建 $logstash_home/vendor/jar/kafka_2.9.2-0.8.1.1/libs/
    
    5. 下载kafka程序包(如果没有, 请去官网下最新0.8.1.1版本, wget https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.1.1/kafka_2.9.2-0.8.1.1.tgz)
        
    6. 从$kafka_home/lib 目录复制所有jar文件到$logstash_home/vendor/jar/kafka_2.9.2-0.8.1.1/libs/下
       
    7. 复制logstash-kafka/lib/logstash里的 inputs 和 outputs 下的 kafka.rb，拷贝到对应的 $logstash_home/lib/logstash 里的 inputs 和 outputs 对应目录下
    
    8. git clone https://github.com/logstash-plugins/logstash-input-kafka.git
       然后进入该目录下运行bundle install, 这时会下载logstash运行时依赖的各种gem (这又是一个大坑, 之前忽略了这一步)
       
    9. 在$logstash_home目录下运行：
       GEM_HOME=vendor/bundle/jruby/1.9 GEM_PATH= java -jar vendor/jar/jruby-complete-1.7.11.jar --1.9 ../logstash-kafka/gembag.rb ../logstash-kafka/logstash-kafka.gemspec
       
        logstash到此已经部署完毕, 这里碰到一个坑: 是logstash从kafka中消费消息是始终不能转换DefaultDecoder对象为json, 最后发现 v0.6.2版本的logstash-kafka中的inputs插件有问题,
        这时, 你需要去github中获取logstash-kafka 项目下的logstash-kafka/lib/logstash/inputs/kafka.rb 作为最新的input插件覆盖之前的.
  
    10, 在$logstash_home目录下创建 logstash.yml 配置文件
       文件内容为如下:
       input {
           kafka {
               zk_connect => "witgo:2181,spark:2181"
               group_id => "eagleye_logstash"
               topic_id => "EAGLEYE_LOG_CHANNEL"
               consumer_threads => 5  
           }
       }
       
       output {
         elasticsearch {
           cluster => "eagleye_es"
           # 我们可以版本搭建一个es服务, 该服务不存储数据和检索, 只是发现es集群,并转发操作请求
           host => "localhost"
           port => "9301"
           protocol => "transport"
           index => "eagleye-log-%{+YYYY.MM.dd}"
           index_type => "log"
           workers => 2 # 如果这个配置开太大容易内存溢出, 需要根据机器配置进行调整, 在测试环境logstash 内存500m时,2个线程效率最高
           flush_size => 1000 #该配置配合workers进行调整, 如果太大, 也容易内存溢出, 或者增加gc次数.目前这个配置可以达到单台logstash可以20000/s的日志写入
           template_overwrite => true
         }
       }
       
     10, 启动logstash 
         bin/logstash agent -f logstash.yml > /home/zhejava/logstash_log/logstash.log &
    
     11, logstash-1.4.2 默认的elasticsearch-logstash.json 模板并没有生效, 里面只是一个通用的样板, 
          我将 logstash-1.4.2/lib/logstash/outputs/elasticsearch/elasticsearch-template.json 修改为如下内容:
          {
            "template" : "eagleye-*", //这个需要和要操作的index名称匹配,很重要,默认是logstash*,而我们的是eagleye*,所以该模板一直不好使,没注意,大坑啊.
            "settings" : {
              "index.refresh_interval" : "5s"
            },
            "mappings" : {
               "log" : {
                      "properties" : {
                          "@timestamp" : {
                              "type" : "date",
                              "format" : "dateOptionalTime"
                          },
                          "timestamp" : {
                              "type" : "date", //日期类型, 在生成json的对象中需要是date类型
                              "format" : "dateOptionalTime" 
                          },
                          "head" : {//涉及到map嵌套,可以这样写
                              "properties" : {
                                  "app" : {
                                      "type" : "string",
                                      "index" : "not_analyzed" //该字段不需要分词索引,统计时不会因为"-"而将一个分开统计
                                  }
                              }
                          }
                      }
                  }
            }
          }
          
        12. logstash时差8小时问题
            
            logstash在按每天输出到elasticsearch时，因为时区使用utc，造成每天8:00才创建当天索引，而8:00以前数据则输出到昨天的索引
            修改logstash/lib/logstash/event.rb 可以解决这个问题
            第226行
            .withZone(org.joda.time.DateTimeZone::UTC)
            修改为
            .withZone(org.joda.time.DateTimeZone.getDefault())


### elasticsearch 部署

    依赖 jdk 1.7
    
    1. git clone git@git.tuan800-inc.com:trade_architect/elasticsearch.git
    
    2. 在  elasticsearch-1.4.4/config/elasticsearch.yml 中设置将需要输出的数据, 日志路径.
    
       node.name: "eagleye_es_XXX"   其中XXX可以标识出该机器
    
    path.data: /home/zhejava/esdata/data   es数据存储位置, 需要是独立的磁盘, 空间越大越好, 到时,公司的全量日志可能会存储在这, 会是高IO操作
    path.work: /home/zhejava/esdata/work  存放临时文件的位置, 建议有独立的磁盘
    path.logs: /home/zhejava/esdata/logs  存放日志的位置, 建议独立的磁盘
    
    3. 执行  elasticsearch-1.4.4/bin/elasticsearch -d
    




### eagleye-java-client 部署

    因为我们的日志收集是对kafka强依赖, 所以需要有一些配置来进行相关的配置才能接入, 具体的配置主要集中在log4j.xml 和spring配置文件中.
    1. pom.xml中添加如下内容:
        <dependency>
            <groupId>com.xxx</groupId>
            <artifactId>eagleye-java-client</artifactId>
            <version>1.1.0-SNAPSHOT</version>
        </dependency>
        
        如果是基于scala-2.10
        <dependency>
            <groupId>com.xxx</groupId>
            <artifactId>eagleye-java-client</artifactId>
            <version>1.1.0-2.10-SNAPSHOT</version>
        </dependency>
    
    2. log4j.xml中需要添加一个自定义的appender节点, 具体如下:
    
        <appender name="eagleye" class="com.xxx.eagleye.client.log4jappender.EagleyeAppender">
            <param name="app" value="eagleye-test" /> <!-- 这里的名称就是eagleye-admin中提到的app , eagleye会按照这个进行约定预警规则的绑定,可以和application name(loom中的)一致 -->
            <layout class="org.apache.log4j.PatternLayout" />
        </appender>
    
        <root>
            <level value="INFO"/>
            <appender-ref ref="console"/>
            <!-- 不要忘记在这里将自定义的eagleye appender 添加到root节点下 -->
    	    <appender-ref ref="eagleye"/> 
        </root>
        
        
        如果你用的是log4j.properties文件, 如下配置:
        log4j.appender.eagleye=com.xxx.eagleye.client.log4jappender.EagleyeAppender
        # 这里的名称就是eagleye-admin中提到的app , eagleye会按照这个进行约定预警规则的绑定,可以和application name(loom中的)一致
        log4j.appender.eagleye.app=eagleye-test 
        log4j.appender.eagleye.layout=org.apache.log4j.PatternLayout
        
        不用忘记在rootLogger中添加eagleye的appender
        log4j.rootLogger=INFO,console,file,eagleye
        
    3. 在spring配置文件中进行如下配置:
        <!-- 以下配置可以引入imago进行集中配置管理 需要显式设置lazy-init="false" 因为整个程序没有通过spring去获取eagleyeChannel对象 -->
        <bean id="eagleyeChannel" class="com.xxx.eagleye.client.Configuration" lazy-init="false">
            <!-- 该redis地址主要是记录client的心跳, 日志生产力,日志总量 -->
            <constructor-arg name="redisIp" value="#{configManager.getRedisConfig('trade_public_redis','14.1').ip}"/> 
            <constructor-arg name="redisPort" value="#{configManager.getRedisConfig('trade_public_redis','14.1').port}"/>
            <constructor-arg name="redisPwd" value="#{configManager.getRedisConfig('trade_public_redis','14.1').pwd}"/>
            <constructor-arg name="kafkaBroker" value="#{configManager.getKafkaConfig('trade_public_kafka','1.1').broker}"/> <!-- 需要用到进行日志收集的kafka配置 -->
            <!--<constructor-arg name="kafkaTopic" value="EAGLEYE_LOG_CHANNEL" />--> <!-- 收集日志的topic名称, 该配置默认不需要配置 -->
            <!--<constructor-arg name="sendFreq" value="3000" />--> <!-- 批量操作时间间隔, 单位 ms, 这些时间间隔往kafka批量推送一次 -->
        </bean>
        
    4. 因为使用的kafka需要用到域名, 需要配置以下kafka的host表
       下面是测试环境的配置.
       192.168.5.158 witgo
       192.168.5.159 spark
       
    5. 如果要发送监控数据, 在代码中可以这样写:
        logger.info(MonitorLog.build().type(LogType.EAGLEYE_MONITOR_LOG_REFUND_SHMJCSWGBSL).value(105D));
       
    注: 如果你想看到client发送的具体日志内容, 请在log4j.xml配置做如下配置:
        <logger name="com.xxx.eagleye.client.log4jappender.EagleyeAppender">
            <level value="debug"/>
        </logger>

### kibana部署和配置

     概述:Kibana 是为 Elasticsearch 设计的开源分析和可视化平台。你可以使用 Kibana 来搜索，查看存储在 Elasticsearch 索引中的数据并与之交互。你可以很容易实现高级的数据分析和可视化，以图标的形式展现出来。

#### 部署:
   
     1.部署一个用于做负载的Elasticsearch,Elasticsearch的部署详见(elasticsearch 部署),部署完成后修改
     2.下载Kibana 4
     3.解压 .zip 或 tar.gz 压缩文件
     4.配置Elasticsearch地址,在config/kibana.yml文件,配置第一步安装的elasticsearch地址.
        elasticsearch_url: "http://localhost:9200"
     5.bin/kibana,启动kibana
     6.访问5601端口.
   
#### 配置
    
     1.配置索引(相当于数据库名),es存入数据是每天创建一个索引,所以kibana要读取数据必须先配置索引,索引可以按天模糊匹配.配置方法如下图.
     配置完索引后,选择时间戳,要选择带"@"符号的timestamp用日志进入es的时间字段.
   
  ![kibana](http://siye1982.github.io/img/blog/eagleye/kibana_index.png)
        
     2.配置discover:配置各种查询数据,进行条件筛选
   
   ![kibana](http://siye1982.github.io/img/blog/eagleye/kib_dis_tool.png) 
   ![kibana](http://siye1982.github.io/img/blog/eagleye/kibana_discover.png)      
   
        (a).点击discover菜单,点击搜索口后面的添加按钮(加号),在左侧Field项下会列出所有的字段信息,鼠标移到字段上面,会出现add按钮,点击添加,该字段会出现在上方的select field项下.
        (b).配置完成后,右边会出现选择的列和数据.
        (c).点击apply应用所选的配值
        (d).点击保存,在页面的右上方,有个保存的图标.填写保存的名字
            
     3.配置visualize:visualize是对discover数据表的各种汇总信息,并以各种个图表显示.
         
   ![kibana](http://siye1982.github.io/img/blog/eagleye/kib_vis_tool.png)  
   ![kibana](http://siye1982.github.io/img/blog/eagleye/kibana_vis.png)   
   
        (a).点击visualize菜单,点击搜索口后面的添加按钮(加号).
        (b).选择样式
        (c).选择基础数据(discover保存的基础数据)
        (d).在左侧选择自己需要的统计数据,Metric选项卡中选择统计结果如(count,avg,sum等).buckets下面的选项可以选择如何分组,相当于group by 后面的字段
        (e).点击apply应用所选的配值,点击右上角保存按钮,然后输入名字保存.
            
     4.配置dashboard:将discover和visualize保存的各种表格和报表集中在一起显示
         
   ![kibana](http://siye1982.github.io/img/blog/eagleye/kib_dash_tool.png)  
   ![kibana](http://siye1982.github.io/img/blog/eagleye/kib_dash.png)   

        (a).点击上图添加按钮,跳转到选择discover和visualize页面.
        (b).选择需要的discover和visualize页面.此时会在下放显示.
        (c).拖动改变位置和大小.
        (d).保存设置好的面板.
        
     5.搜索框简易使用说明
       1.在特定的字段里搜索,例如：line_id:86169
       2.可以用 AND/OR 来组合复杂的搜索，AND/OR必须大写,例如:food AND love
       3.括号的使用：("played upon" OR "every man") AND stage
       4.数值类型的数据可以直接搜索范围：line_id:[30000 TO 80000] AND havoc
       5.搜索所有：*
       6.如果输入 where1 where2 两个单词,实际搜索的是where1 OR where2 如果要搜索where1 where2,请用双引号引起来"where1 where2"
       
### alerter 部署和配置
    
    部署:
    1. git clone git@git.tuan800-inc.com:trade_architect/eagleye.git
    2. cd eagleye/eagleye-alerter 进入在eagleye-alerter程序根目录
    3. 在eagleye-alerter程序根目录执行 git -pull && mvn clean package -Dmaven.test.skip=true -U , 完成后进入文件夹得到 项目压缩包 eagleye-alerter-1.0.0-all.tar.gz
    4. 解压项目包
    5. 执行 nohup ./start.sh  > /data/applogs/eagleye/service.log  2>&1  &

    需要注意: 

    1 需要替换conf/log4j.xml 中的日志文件目录 , 和命令输出日志目录

    2 需要修改run.sh 中的变量IP 为服务器IP , HEAP_DUMP_PATH 为日志目录的文件

    关闭:
    需要手动kill掉程序进程,可以执行以下命令:
    ps -ef | grep 'com.xxx.eagleye.alerter.App' | grep -v 'grep' |  awk '{print $2}' | xargs  kill
    
    配置说明:
    由于alerter程序会和kafka,zookeeper集群进行通信, 所以程序运行前需要配置相关host, 配置如下:
    
    #kakfa cluster
    172.16.11.43 YJ-18-1
    172.16.11.46 YJ-18-4
    172.16.11.60 YJ-17-4
    172.16.11.67 YJ-17-11
    172.16.11.73 YJ-16-3
    
    loomContext.xml 中可以配置日志toipc kafka消费者的消费线程数(alerterConsumerThreadNum),
     alerter程序可以部署的最大实例数为 kafka日志topic分区数/alerterConsumerThreadNum 
     如果部署的程序实例数超过了最大实例数, 新的程序实例将会得消费不到得到topic分区, 不产生吞吐量
     
    由于alerter即是kafka消费者, 又是kafka生产者, 为了避免死循环, 程序在打印描述性日志前都都加了'~', 创建alerter程序的预警规则时, 应当在规则中过滤日志内容第一个字符为'~'的情况

    
    
    
    
        
### eagleye-admin部署     

   [sql脚本](../init.sql)
   
    1.在trde库执行附件脚本，初始化表.   
    2.git clone git@git.tuan800-inc.com:trade_architect/eagleye.git
    3.cd eagleye/
    4.mvn clean; mvn -DskipTests package
    5.在tomcat中部署eagleye-admin/target/eagleye-admin.war包。
    6.初始用户名密码:admin/test
  




## 非java项目对接
     如果非java服务想对接到该预警平台, 请按照如下json格式:
    ``` 
	 {
         "body": "日志内容",
         "head": {
             "app": "eagleye-test",
             "ip": "192.168.1.100",
             "level": "WARN",
             "pid": "3817"
         },
         "timestamp": 1426216784971
     }
     发送到kafka的 名为:EAGLEYE_LOG_CHANNEL 的topic中
     
     如果是直接接入其他非java的日志, 并且还不想格式化上述json,可以直接开启一个logstash实例, 配置文件如下:
     input {
       file {
          path => ["/home/zhejava/logstash/logstash-1.4.2/temp/*.log"]
         # type => "system"
         # start_position => "beginning"
       }
     }
     
     
     filter {
       grok {
         match => { "message" => "" }
         add_field => {
             "head.app" => "logstash-test"
             "head.ip" => "192.168.5.177"
             "head.level" => "WARN"
             #"timestamp" => 1426237376377
             #"head" => "{'app' : 'eagleye-test-b','ip' : '192.168.5.244','level' : 'ERROR','pid' : '9744'}"
         }
     
         remove_field => [ "message" ]
       }
     }
     
     
     # 如果是入kafka请这么写
     output {
       #stdout {
       #   codec => rubydebug
       #   #codec => plain
       #}
     
       kafka {
          #codec => plain {
          #   format => "%{body}"
          #}
          broker_list => "witgo:9092,spark:9092"
          topic_id => "EAGLEYE_LOG_CHANNEL"
     
       }
     }
     
     
     # 如果是入es 请这么写,不推荐直接入es, 这样就不能设置实时预警了
     output {
           elasticsearch {
           cluster => "eagleye_es"
           host => "localhost"
           port => "9301"
           protocol => "transport"
           index => "eagleye-log-%{+YYYY.MM.dd}"
           index_type => "log"
           workers => 2
           flush_size => 1000
           template_overwrite => true
           }
     }
    ```




     

## 其他

### es自定义检索功能说明


	1. elastic search建立索引时, 会使用现有词典进行分词, 如果用户输入的关键词不在当前词库中, 将不会搜索到结果, 遇到这种情况, 建议用户缩小关键词粒度, 并使用多个关键词进行查询
	
	2. 多个关键词可以用空格分开

###xxx es index template功能简介

[elasticsearch template 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/1.3/indices-templates.html)

	1. es 可用通过预先设置的template来指定即将创建索引的setting 和 mapping 设置

	2. 使用方式:

		1. 使用命令添加

		2. 将 template模板 添加到 config/templates/ 目录


