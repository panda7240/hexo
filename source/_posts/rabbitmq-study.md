layout: photo
title: Rabbitmq学习笔记(未完)
date: 2015-07-10 10:07:32
categories: 学习笔记
tags:
- Rabbitmq
- MQ
- AMQP
- 分布式
photos:
- http://siye1982.github.io/img/blog/rabbitmq/rabbitmq-banner.png
---

## 写在前面

我们在进行分布式系统开发时, 为了提升性能, 系统之间松耦合对接, 我们需要在一些业务场景中使用MQ. 在使用MQ时不仅需要考虑MQ API的易用性, 可靠性, 也需要考虑MQ本身是否提供简单,快捷的可视化管理监控界面. 基于这几方面的考虑, 和具体的业务场景, Rabbitmq这个消息中间件进入了我们的选型范围. 本文主要结合[Rabbitmq](http://www.rabbitmq.com)官方文档,和互联网上其他Rabbitmq使用者的经验(涉及到引用地方会标明详细出处),记录学习Rabbitmq的过程.  
<!--more-->

## Rabbitmq关键字

### Producer(生产者)

![生产者](http://siye1982.github.io/img/blog/rabbitmq/producer.png)

### Queue(消息队列)

![消息队列](http://siye1982.github.io/img/blog/rabbitmq/queue.png)

### Temporary queues(临时队列)

临时队列在创建时,队列名称由rabbitmq自动创建(也可以通过uuid指定队列名称). 该队列是非持久的,唯一的,当与队列断开连接时,该队列自动删除.

### Exchanges(交换,中继器)

![交换,中继器](http://siye1982.github.io/img/blog/rabbitmq/exchanges.png)

消息生产者会将消息发送给交换器, 然后根据设定的规则将消息路由到指定的queue中. 这里会有几种模式: direct(点对点直接发送)、topic(组播)、fanout(广播). 这里topic模式可以通过设置变成direct或fanout模式.

### Consumer(消费者)

![消费者](http://siye1982.github.io/img/blog/rabbitmq/consumer.png)


### Bindings(绑定规则)

![绑定规则](http://siye1982.github.io/img/blog/rabbitmq/bindings.png)


<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>