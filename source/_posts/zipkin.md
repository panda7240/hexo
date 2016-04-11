layout: photo
title: 基于Zipkin的Thrift服务RPC调用链跟踪
tags:
	- 原创
	- SOA
	- Finagle
	- Zipkin
photos:
date: 2016-04-07 08:50:21
categories: RPC跟踪
---

# 概述
我们现在所处的生产环境是一个集Nodejs, Go, Java, Ruby, Scala等多种语言程序的混合场景.Twitter的Finagle框架, 是一个基于Thrift协议的RPC框架,其中Zipkin是针对Finagle框架的一个基于Thrift协议的RPC调用链跟踪的工具，可搜集各服务调用数据，并提供分析查询展示功能。帮助识别分布式RPC调用链里，哪一个调用比较耗时，性能有问题，以及是否有异常等，使得诊断分布式系统性能成为可能。
<!--more-->


# 基本概念
## Trace
一次服务调用追踪链路，由一组Span组成。需在web总入口处生成TraceID，并确保在当前请求上下文里能访问到
## Annotation
表示某个时间点发生的Event

* Event类型
 
	```
	cs：Client Send 请求
	sr：Server Receive到请求
	ss：Server 处理完成、并Send Response
	cr：Client Receive 到响应
	```
* 什么时候生成
	客户端发送Request、接受到Response、服务器端接受到Request、发送 Response时生成。Annotation属于某个Span，需把新生成的Annotation添加到当前上下文里Span的annotations数组里

* thrift数据结构

	```
	/**
	 * Associates an event that explains latency with a timestamp.
	 *
	 * Unlike log statements, annotations are often codes: for example "sr".
	 */
	struct Annotation {
	  /**
	   * Microseconds from epoch.
	   *
	   * This value should use the most precise value possible. For example,
	   * gettimeofday or syncing nanoTime against a tick of currentTimeMillis.
	   */
	  1: i64 timestamp
	  /**
	   * Usually a short tag indicating an event, like "sr" or "finagle.retry".
	   */
	  2: string value
	  /**
	   * The host that recorded the value, primarily for query by service name.
	   */
	  3: optional Endpoint host
	  // don't reuse 4: optional i32 OBSOLETE_duration         // how long did the operation take? microseconds
	}
	```


	
## BinaryAnnotation
存放用户自定义信息，比如：sessionID、userID、userIP、异常等
	
* 什么时候生成?
	在任意需要记录自定义跟踪信息时都可生成。比如：异常、SessionID等。如Annotation一样，BinaryAnnotation也属于某个Span。需把新生成的BinaryAnnotation，添加到当前上下文里Span的binary_annotations数组.
 		
* Thrift数据结构
	
	```
		/**
	 * Binary annotations are tags applied to a Span to give it context. For
	 * example, a binary annotation of "http.uri" could the path to a resource in a
	 * RPC call.
	 *
	 * Binary annotations of type STRING are always queryable, though more a
	 * historical implementation detail than a structural concern.
	 *
	 * Binary annotations can repeat, and vary on the host. Similar to Annotation,
	 * the host indicates who logged the event. This allows you to tell the
	 * difference between the client and server side of the same key. For example,
	 * the key "http.uri" might be different on the client and server side due to
	 * rewriting, like "/api/v1/myresource" vs "/myresource. Via the host field,
	 * you can see the different points of view, which often help in debugging.
	 */
	 struct BinaryAnnotation {
	  /**
	   * Name used to lookup spans, such as "http.uri" or "finagle.version".
	   */
	  1: string key,
	  /**
	   * Serialized thrift bytes, in TBinaryProtocol format.
	   *
	   * For legacy reasons, byte order is big-endian. See THRIFT-3217.
	   */
	  2: binary value,
	  /**
	   * The thrift type of value, most often STRING.
	   *
	   * annotation_type shouldn't vary for the same key.
	   */
	  3: AnnotationType annotation_type,
	  /**
	   * The host that recorded value, allowing query by service name or address.
	   *
	   * There are two exceptions: when key is "ca" or "sa", this is the source or
	   * destination of an RPC. This exception allows zipkin to display network
	   * context of uninstrumented services, such as browsers or databases.
	   */
	  4: optional Endpoint host
	}
	```

* AnnotationType 结构
 
	```
	/**
	* A subset of thrift base types, except BYTES.
	*/
	enum AnnotationType {
		BOOL,  BYTES,I16,  I32,  I64,  DOUBLE,  STRING
	}
	```
	 
## Span
表示一次完整RPC调用，是由一组Annotation和BinaryAnnotation组成。是追踪服务调用的基本结构，多span形成树形结构组合成一次Trace追踪记录。Span是有父子关系的，比如：Client A、Client A -> B、B ->C、C -> D、分别会产生4个Span。Client A接收到请求会时生成一个Span A、Client A -> B发请求时会再生成一个Span A-B，并且Span A是 Span A-B的父节点

* 什么时候生成
	* 服务接受到 Request时，若当前Request没有关联任何Span，便生成一个Span，包括：Span ID、TraceID
	* 向下游服务发送Request时，需生成一个Span，并把新生成的Span的父节点设置成上一步生成的Span

* Thrift结构
		
	```
	/**
	* A trace is a series of spans (often RPC calls) which form a latency tree.
	*
	* Spans are usually created by instrumentation in RPC clients or servers, but
	* can also represent in-process activity. Annotations in spans are similar to
	* log statements, and are sometimes created directly by application developers
	* to indicate events of interest, such as a cache miss.
	*
	* The root span is where parent_id = Nil; it usually has the longest duration
	* in the trace.
	*
	* Span identifiers are packed into i64s, but should be treated opaquely.
	* String encoding is fixed-width lower-hex, to avoid signed interpretation.
	*/
	struct Span {
	/**
	* Unique 8-byte identifier for a trace, set on all spans within it.
	*/
	1: i64 trace_id
	/**
	* Span name in lowercase, rpc method for example. Conventionally, when the
	* span name isn't known, name = "unknown".
	*/
	3: string name,
	/**
	* Unique 8-byte identifier of this span within a trace. A span is uniquely
	* identified in storage by (trace_id, id).
	*/
	4: i64 id,
	/**
	* The parent's Span.id; absent if this the root span in a trace.
	*/
	5: optional i64 parent_id,
	/**
	* Associates events that explain latency with a timestamp. Unlike log
	* statements, annotations are often codes: for example SERVER_RECV("sr").
	* Annotations are sorted ascending by timestamp.
	*/
	6: list<Annotation> annotations,
	/**
	* Tags a span with context, usually to support query or aggregation. For
	* example, a binary annotation key could be "http.uri".
	*/
	8: list<BinaryAnnotation> binary_annotations
	/**
	* True is a request to store this span even if it overrides sampling policy.
	*/
	9: optional bool debug = 0
	/**
	* Epoch microseconds of the start of this span, absent if this an incomplete
	* span.
	*
	* This value should be set directly by instrumentation, using the most
	* precise value possible. For example, gettimeofday or syncing nanoTime
	* against a tick of currentTimeMillis.
	*
	* For compatibilty with instrumentation that precede this field, collectors
	* or span stores can derive this via Annotation.timestamp.
	* For example, SERVER_RECV.timestamp or CLIENT_SEND.timestamp.
	*
	* Timestamp is nullable for input only. Spans without a timestamp cannot be
	* presented in a timeline: Span stores should not output spans missing a
	* timestamp.
	*
	* There are two known edge-cases where this could be absent: both cases
	* exist when a collector receives a span in parts and a binary annotation
	* precedes a timestamp. This is possible when..
	*  - The span is in-flight (ex not yet received a timestamp)
	*  - The span's start event was lost
	*/
	10: optional i64 timestamp,
	/**
	* Measurement in microseconds of the critical path, if known.
	*
	* This value should be set directly, as opposed to implicitly via annotation
	* timestamps. Doing so encourages precision decoupled from problems of
	* clocks, such as skew or NTP updates causing time to move backwards.
	*
	* For compatibility with instrumentation that precede this field, collectors
	* or span stores can derive this by subtracting Annotation.timestamp.
	* For example, SERVER_SEND.timestamp - SERVER_RECV.timestamp.
	*
	* If this field is persisted as unset, zipkin will continue to work, except
	* duration query support will be implementation-specific. Similarly, setting
	* this field non-atomically is implementation-specific.
	*
	* This field is i64 vs i32 to support spans longer than 35 minutes.
	*/
	11: optional i64 duration
	}
	```
	
	
# 服务之间需传递的信息
Trace的基本信息需在上下游服务之间传递，如下信息是必须的：

* Trace ID：起始(根)服务生成的TraceID
* Span ID：调用下游服务时所生成的Span ID
* Parent Span ID：父Span ID
* Is Sampled：是否需要采样
* Flags：告诉下游服务，是否是debug Reqeust
	
# Trace Tree组成
一个完整Trace 由一组Span组成，这一组Span必须具有相同的TraceID；Span具有父子关系，处于子节点的Span必须有parent_id，Span由一组 Annotation和BinaryAnnotation组成。整个Trace Tree通过Trace Id、Span ID、parent Span ID串起来的。

# 其他要求
* Web入口处，需把SessionID、UserID(若登陆)、用户IP等信息记录到BinaryAnnotation里
* 关键子子调用也需用zipkin追踪，比如：订单调用了Mysql，也许把个调用的耗时情况记录到 Annotation里
* 关键出错日志或者异常也许记录到BinaryAnnotation里

经过上述三条，用户任何访问所引起的后台服务间调用，完全可以串起来，并形成一颗调用树。通过调用树，哪个调用耗时多久，是否有异常等都可清晰定位到。

# 完整示例
testService(Web服务) -> OrderServ(Thrift) -> StockServ & PayServ(Thrift)。一共有四个服务，testService 调用 OrderServ、OrderServ同时调用 StockServ和PayServ。需生成的Trace信息如下：
 
* testService收到Http Reqeust时，需在入口处生成TraceID、SpanID，以及一个Span对象，假若叫Span1。
* testService向OrderServ发送 Thrift Request时，需新生成一Span2，并把parent ID设置成Span1的spanID。同事需修改Thrift Header，把Span2的spanID、parent ID、TraceID 传递给下游服务。也需生成"cs" Annotation，关联到span2上；当接受到OrderServ的Response时，再生成"cr" Annotation，也关联到span2上。
* OrderServ接受到Thrift Request后，从Thrift Header里解析到TraceID、parent ID、 Span ID(span2)、并保留到上下文里。同时生成"sr"Annotaition，并关联到span2上；当处理完成发送response时，再生成"ss"Annotation，并关联到span2上。
* OrderServ向StockServ发送 Thrift Request时，需新生成一Span3，并把parentID设置成上一步(Span2)的span ID。Annotation处理如上。
* Order Serv向PayServ发送请求时，新生成一Span4，并把parentID设置Span2的span ID。Annotation处理如上。

# 总结客户端要做哪些事情？

## Thrift协议扩展（参照Finagle）

* Thrift Server端向Thrift协议Header里增加TraceID、SpanID、ParentID、Flags、Is Sampled
* Thrift Client从Thrift Request里解析出TraceID、SpanID、ParentID、flags、Is Sampled

## Trace数据生成
* Web入口处，生成TraceID、SpanID、以及Span对象，并把sessionID、userID、userIP记录到BinaryAnnotation里.
* 调用下游服务时生成Span对象.
* Thrift客户端Send Request时，生成"cs" Annotation.
* Thrift客户端Receive Response时，生成"cr"Annotation.
* Thrift服务器端Receive Reqeust时，生成"sr" Annotation.
* Thrift服务器端Send Response时，生成"ss"Annotation.
* 异常、关键系统调用数据也要记录到Annotation里.

## Trace数据传递与收集
* 数据刷新不能影响业务性能，刷新失败更不能影响正常业务逻辑.
* 上下文传递
	* 入口处生成的TraceID、SpanID需要在当前Request上下文里读取到
	* Ruby 、java、NodeJS上下文传递要做到业务代码透明， Golang单独处理
* 客户端生成Trace数据，并写入到本地log文件
* 由logstash(类似工具)统一搜集日志，并写入kafka
* 由kafka consumer从kafka读取数据，并索引到ES里


# Zipkin跟踪数据收集格式定义
我们针对跟踪数据的收集的统一接入点, 为kafka.所有应用产生的zipkin数据统一发送到Kafka集群中.具体的channel为: EAGLEYE_ZIPKIN_CHANNEL.

## Span(记录每次调用信息)
		
**JSON串范例(格式化)：**
	
```
span{
    "app": "app", //所属应用
    "flag": "flag", //cscr标明是调用端，srss标明是被调用端。标识, 一个span数据 ,会有基本的cr, cs, sr, ss四个点. 但是获取数据时,一般只能两两获取, 所以, 一个span通常会被分割为cr, cs和 sr, ss分别发送
    "ip": "192.168.10.100", //ip地址
    "mname": "mname", //方法名
    "pid": "pid", //进程id
    "port": 1000, //端口, 如果是cscr客户端, 则为0
    "psid": "psid", //parent span id
    "sid": "sid", //spanId
    "sname": "sname", //服务名
    "etime": 1449072000000, //结束时间点, 单位到ms,如果是client端,则是cr时间点, 如果是server端,则是ss时间点
    "stime": 1449039924512, //开始时间点, 单位到ms, 如果是client端,则是cs时间点, 如果是server端,则是sr时间点
    "duration": 32075488, //结束时间点减去开始时间点的, 调用的区间时长, 单位ms
    "tid": "tid", //traceId
    "timestamp": 1449039924548 //产生的时间点
}
```

**备注: span信息的json,添加'span'头信息, 在通过kafka做收集时, 通过该头信息将span信息路由到独立的es的index中. 每一条span记录其实是半个span信息, 要么是client端产生的, 要么是server端产生的.**

## BinaryAnnotation(可以用来传递用户session_id的信息, 也可以传递其他业务信息)
* JSON串范例(格式化)

	```
	{
		"app": "app", //所属应用
		"ip": "ip", //ip地址,冗余信息
		"key": "key", //key, 可以设为存储用户session的key, 如果是用来传递用户session信息的, 可以统一约定为: session_id
		"mname": "mname",  //方法名
		"pid": "10000", //进程id,冗余信息
		"sid": "sid", //spanId
		"sname": "sname", //服务名
		"tid": "tid", //traceId
		"timestamp": 1449038780194, //产生的时间戳, 长整型, 精确到毫秒
		"type": "type", //类型,用来区分是记录异常的还是业务流程的等等, 默认是'common'即可
		"value": "value" //如果是传递用户session信息 ,可以直接写在该字段中.
	}
	```		
	
* 备注:这里将BinaryAnnotation独立记录, 并发送到kafka消息队列, 其通过sid关联具体的span信息.


----

# java使用zipkin经验
* 客户端调用服务端失败（服务端没启动），客户端的annotations有cs、cr
* 客户端超时（等待响应超时），服务端的annotations有sr、ss，客户端的annotations有cs、cr
* 服务端超时，服务端的annotations有timeout和sr，没有ss、没有binary_annotation,客户端的annotations有cs、cr
* 客户端无法记录binary_annotation


# 实际效果

收集了Zipkin产生的RPC调用链信息,并给服务治理框架提供跟踪信息检索(双击某行有问题的Span记录可以查看某次调用链详情,了解具体的网络情况和业务执行情况)
    ![rpc的span信息](http://siye1982.github.io/img/blog/loom/rpc_span.png)
    ![rpc调用链详情](http://siye1982.github.io/img/blog/loom/rpc_link.png)














<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>

