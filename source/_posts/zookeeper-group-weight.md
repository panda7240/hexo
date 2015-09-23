title: Zookeeper配置中的Group和Weight
date: 2015-09-21 15:52:25
categories: Zookeeper
tags: 
- 原创
- Zookeeper
- 分布式
---

## 简单理解

Zookeeper的集群有一个法则，整个集群的数量要过半存活，集群就是稳定的。在这法则之外，官方文档中还提供了Group和Weight的配置。Zookeeper的Group配置的意思就是对整个大的zk集群进行分组.

例如，我们有1、2、3、4、5、6、7七个节点。我们做如下配置：
<!--more-->
```
group.1=1:2:3
group.2=4:5:6
group.2=7
```
将七台机器分为三个组，这时，只要三个组中的两个是稳定的，整个集群的状态就是稳定的。即有2n+1个组，只要有n+1个组是稳定状态，整个集群就是稳定的。也就是过半原则。

Zookeeper的Weight配置要和Group配置一起使用。官方文档中介绍Weight时提到此值影响集群节点Leader的选取，经过实际测试和翻阅zk的投票源码，Weight等于0的节点不参与投票，没有被选举权。而不影响投票的权重. 源码片段如下:

```
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch,long curId, long curZxid, long curEpoch){
	if (self.getQuorumVerifier().getWeight(newId) == 0){
		return false;
	}
	
	return ((newEpoch > curEpoch) || 
			 (newEpoch == curEpoch && newZxid > curZxid) || 
			 (newZxid == curZxid && newId > curId));
}
```
   
Weight可以调节一个组内单个节点的权重，默认每个节点的权重是1（如果配置是0不参与leader的选举）.每个组有一个法定数的概念，法定人数等于组内所有节点的权重之和.此时判断一个组是否稳定，是要判断存活的节点权重之和是否大于该组法定数的权重.

接上面的例子，我们做如下配置:

```
weight.1=3
weight.2=1
weight.3=1
weight.4=1
weight.5=1
weight.6=1
weight.7=1
```    
此时Group1的法定数是：3+1+1=5，只要节点权重之和过半该组就是稳定的。也就是说，该组中，节点1（权重是3）只要存活,该组就是稳定的. 经过以上配置,停掉节点2，3，4，5，6整个集群仍然是稳定的. 此时Group1和Group3是稳定状态.

