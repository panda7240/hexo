layout: photo
title: 通过Packetbeat抓取Finagle协议数据包(自定义Thrift协议)总结
tags:
  - Elasticserch
  - Beats
  - 分布式
  - 原创
photos:
  - http://siye1982.github.io/img/blog/beats/beats-petals.png
date: 2016-04-30 09:05:59
categories: 全链路监控
---




# 写在前面
最近一年多一直在做服务治理相关的开发工作. 起初服务监控采用了成本比较低的方式来实现(提供者,消费者自己按分钟维度上报健康数据到Redis,但是这种方式只是在Java的服务提供者和消费者做到了很好的实现, 其他语言目前只能上报很少部分的监控数据). 因为公司的开发语言是多样的, 其中包括: Nodejs, Ruby, Golang, Java, Scala等, 那么将来要对监控数据的模型拓展, 需求变更等, 将很难快速推广实现. 随着公司业务的高速发展, 以及将来所有服务部署Docker化, 服务的监控预警已经是服务治理工作中的重中之重. 服务监控最好可以同时监控基础服务(Mysql, Redis等),业务服务. 我们的业务服务是采用的Twitter的Finagle-thrift实现多语言之间的RPC调用. Balabala说了这么多, 就是我们现在要做全链路监控, 做监控首先第一步是需要可以收集到这些网络调用的原始数据, 这个时候ElasticStack中的Beats项目进入了我们的视线, Beats项目中的Packetbeat子项目可以抓取到像Mysql, Redis, Thrift等协议的数据包. 但是,我们业务使用的通信协议是Finagle-thrift, 这里面为了满足一些拓展(比如:用于RPC调用链跟踪的Zipkin),Finagle-thrift在原生Thrift上做了二次封装, 接下来需要让Packetbeat对Finagle-thrift协议支持. 下面我将分析过程整理如下, 方便以后温习回顾. 

<!--more-->


# Packetbeat项目介绍

更详细的请参考 [Medcl](http://log.medcl.net/item/2015/12/packetbeat-zi-ding-yi-xie-yi-di-kai-fa-jiao-cheng/)的一个教程

整个Beats项目都是用的Golang语言开发, Golang这几天也是现学现卖, 我在整个调试过程中没有找到可以比较方便进行Debug的方式, 只能通过fmt.Println进行各种调试信息的输出, 这个过程比较痛苦. 这里我顺便记录一下怎么配置Go的环境, 有几个概念比较懵,在此记录一下.

## 安装GO
在 [这里](https://golang.org/dl/) 获取对应的操作系统的GO安装bao

## GOPATH
* 安装好Go后需要设置环境变量,如下:

	```
	#这是Go的安装路径
	export GOROOT=/usr/local/go
 	export GOBIN=$GOROOT/bin
 	
 	#这里可以理解为Go项目的工作空间, 这里允许有多个目录,注意用":"分割
 	#当有多个GOPATH时,执行 go get命令的内容默认会放在第一个目录下
 	export GOPATH=/work/goworkspace
	```

* GOPATH的的几个目录约定
	* src 放置Go项目的源码
	* pkg Go项目中使用的第三方包
	* bin 编译后生成的可执行文件, 可以把此目录加入到 PATH 变量中	

	
## 获取项目

```
 #创建相应目录
 mkdir -p $GOPATH/src/github.com/elastic/ 
 cd $GOPATH/src/github.com/elastic
	
 #签出源码
 git clone https://github.com/elastic/beats.git
 cd beats
	
 #修改官方仓库为upstream源，设置自己的仓库为origin源
 git remote rename origin upstream
 git remote add origin git@github.com:medcl/packetbeat.git
	
 #获取上游最新的代码，如果是刚fork的话可不用管
 git pull upstream master
	
 #签出一个名为finagle的分支，用于开发这个功能
 git checkout -b finagle
	
 #切换到packetbeat模块
 cd packetbeat
	
 #获取依赖信息
 (mkdir -p $GOPATH/src/golang.org/x/&&cd $GOPATH/src/golang.org/x &&git clone  https://github.com/golang/tools.git )
 go get github.com/tools/godep
	
 #编译
 make

```

## yml配置文件说明

```
interfaces:
  #如果提供者消费者在本机,直接写成lo0 
  device: en0
protocols:
	# 自定义协议名
  finaglethrift:
    ports: [20880, 9090, 9091, 9099, 9098]
    # 自定义Thrift的Transport type一定要是frame的方式, 否则解析不出来
    transport_type: framed
    protocol_type: binary
    # idl文件一定要有
    idl_files: ["test_cfg/result.thrift","test_cfg/order.thrift","test_cfg/hello.thrift"]


output:
  elasticsearch:
    hosts: ["192.168.10.235:9200"]
  kafka:
    hosts: ["192.168.5.159:9092"]
    topic: "packetbeat_test_qqq"
shipper:
logging:
  files:
    path: /tmp/mybeat

```

## 需要修改哪些文件
* 新增协议目录, packetbeat启动时会自动扫描protos目录下的协议包

	![](http://siye1982.github.io/img/blog/beats/14619840915684.jpg)
	因为是要对Thrift协议进行拓展, 所以之前很多代码是可以复用的, 直接将原来的thrift目录在当前目录下复制一份, 直接改名为finaglethrift

* 文件修改
	为了便于区分, 我们将原来所有文件名中的thrift变更为finaglethrift, 变更之后我们只需要修改finaglethrift.go文件即可.
	* 将包名从thrift变更为finaglethrift

		![](http://siye1982.github.io/img/blog/beats/14619845095547.jpg)

	* 修改协议注册名,这里的名称直接匹配yml配置文件中的协议名

		![](http://siye1982.github.io/img/blog/beats/14619846122124.jpg)

  
  * 协议解析的具体方法修改, 主要业务抓包分析将在这个方法中完成,我们本次改动也是针对这个方法的修改

 ![](http://siye1982.github.io/img/blog/beats/14619847474701.jpg)


# 原生Thrift简单分析

## 通讯协议格式
* TCompactProtocol
* TBinaryProtocol(我们主要采用这种格式进行通讯)
	TBinaryProtocol下通信方式采用TFramedTransport，即以帧的方式对数据进行传输
	
	注意: 服务端, 服务端需要采用Framed的方式进行通信, packetbeat采用Framed的方式进行抓包分析, 如果thrift的传输方式不是这种方式, packetbeat将解析不出
	
* TJSONProtocol


## 核心模型
* TTransport, 这是一个基类,我们使用的传输方式是Framed, 那么直接使用的TFramedTransport将继承TTransport. TFramedTransport会将数据写入到一个buf中, 等全部写完之后会调用flush方法,首先计算出buf中的数据长度,将4个字节的帧长度和数据内容进行封装进行发送. 针对解析方怎么判断是否解析完,都是通过发送的data中头四个字节判断.具体如下图:
	![](http://siye1982.github.io/img/blog/beats/2016-04-30_11-17-27.png)

	具体封装源代码:
	
	```
	@Override
  public void flush() throws TTransportException {
	    byte[] buf = writeBuffer_.get();
	    int len = writeBuffer_.len();
	    writeBuffer_.reset();
	    # 封装成 4个字节 + 帧内容
	    encodeFrameSize(len, i32buf);
	    transport_.write(i32buf, 0, 4);
	    transport_.write(buf, 0, len);
	    transport_.flush();
  }
	```

* TProtocol, 协议接口, 我们主要是采用TBinaryProtocol的协议类进行通信, 其中实现了接口中的操作协议的方法. TBinaryProtocol需要为消息体封装一个Header, 其中还定义了Thrift中的读写模式(这里很重要,如果模式不匹配将无法正常解析),主要分为: 严谨的读写, 普通读写. 因为我们主要针对严谨读写模式进行抓包分析, 下面将重点解析一下在严谨读写模式下的消息体内容都是什么, 具体如下图:

	![2016-04-30_11-39-09](http://siye1982.github.io/img/blog/beats/2016-04-30_11-39-09-1.jpg)


在TBinaryProtocol中有对消息体的读取和写入操作, 具体代码如下:

	
	```
	/**
   * Reading methods.
   */

  public TMessage readMessageBegin() throws TException {
	    int size = readI32();
	    if (size < 0) {
	      int version = size & VERSION_MASK;
	      if (version != VERSION_1) {
	        throw new TProtocolException(TProtocolException.BAD_VERSION, "Bad version in readMessageBegin");
	      }
	      return new TMessage(readString(), (byte)(size & 0x000000ff), readI32());
	    } else {
	      if (strictRead_) {
	        throw new TProtocolException(TProtocolException.BAD_VERSION, "Missing version in readMessageBegin, old client?");
	      }
	      return new TMessage(readStringBody(size), readByte(), readI32());
	    }
  }	
	
	public void writeMessageBegin(TMessage message) throws TException {
	    if (strictWrite_) {
	      int version = VERSION_1 | message.type;
	      writeI32(version);
	      writeString(message.name);
	      writeI32(message.seqid);
	    } else {
	      writeString(message.name);
	      writeByte(message.type);
	      writeI32(message.seqid);
	    }
  }
	
	
	/**
	 * Message type constants in the Thrift protocol.
	 *
	 */
	public final class TMessageType {
		  public static final byte CALL  = 1;
		  public static final byte REPLY = 2;
		  public static final byte EXCEPTION = 3;
		  public static final byte ONEWAY = 4;
	} 

	```


* TMessage, 服务提供者,消费者在进行RPC通信时都会讲传递的数据封装成TMessage, 主要包含三部分
	* 名称
	* 序号
	* 类型


# Finagle-thrift协议分析
因为Finagle-thrift是在Thrift协议之上做了封装, 我们主要对着两个协议中具体的数据进行比对.

## 测试数据IDL
为了让测试具有代表性, 构建的IDL文件中既有简单的没有入参,返回值的finaglePing方法, 也有有入参,复杂返回值的detail方法

```
	
include "result.thrift"
	
/*订单*/
struct Order {
    1:i32 userId
	/*买家*/
	2:string userName,
	/*订单ID*/
	3:string orderId,
}
	
struct OrderResult {
    1:result.Result result,
	2:optional Order order
}
	
service OrderServ{
	/*订单详情*/
	OrderResult detail(1:i32 userId, 2:string userName, 3:string orderId)
	void finaglePing()
}	
	
	
	
	
/************************复杂返回值Result的定义***************************/
	
struct FailDesc {
	1:string name,
	2:string failCode,
	3:string desc
}
	
struct Result {
	
	1:i32 code,
	
	2:optional list<FailDesc> failDescList
}
	
struct StringResult {
	1:Result result,
	
	2:optional string value,
	
	3:optional string extend
}	
	
	
```

## 一次RPC调用的差异
### 原生Thrift调用
我们针对finaglePing方法通过原生Thrift进行一次RPC调用,并在Client端TcpDump出产生的数据包
![2016-04-30_12-33-13](http://siye1982.github.io/img/blog/beats/2016-04-30_12-33-13.jpg)
从图上可以看出,包含了3次握手, 1次Client与Server的业务请求交互, 4次挥手关闭连接.

下面我们看Client发送请求时的具体数据包内容如下图:
![2016-04-30_12-39-12](http://siye1982.github.io/img/blog/beats/2016-04-30_12-39-12.jpg)
这里包含数据长度, Thrift是否是严谨读写,消息类型, 消息内容等信息.
		
### Fiangle-thrift调用及分析
我们针对finaglePing方法同样通过Fiangle-thrift方式进行一次RPC调用,并在Client端TcpDump出产生的数据包
![2016-04-30_12-45-41](http://siye1982.github.io/img/blog/beats/2016-04-30_12-45-41.jpg)
从上图看出, 一次RPC调用包含了, 3次握手, 1次fiangle确认协议的请求交互, 1次Client与Server的业务请求交互, 4次挥手关闭连接.
		
关于Client发送的请求和原生Thrift还不太一样, 在创建完连接之后, 需要发送
一次带有`__can__finagle__trace__v3__`信息的请求已确认是否是Finagle-thrift协议, 确认成功之后才会进行真正的业务交互, 这次确认是一次标准的Thrift通信,具体如下图:
![2016-04-30_12-52-52](http://siye1982.github.io/img/blog/beats/2016-04-30_12-52-52.png)
		
		
下面是在确认Fiangle标识之后进行的真正的业务通信,具体如下图:
		
![2016-04-30_12-57-36](http://siye1982.github.io/img/blog/beats/2016-04-30_12-57-36.png)

我们上面这张图中可以看出在标准的Thrift协议数据之前Finagle-thrift自己又加了很多自己的数据,具体加了什么, 我们来看一下Fiangle的源码, 具体如下:
		
	```
	/**
	 * ThriftClientFramedCodec implements a framed thrift transport that
	 * supports upgrading in order to provide TraceContexts across
	 * requests.
	 */
	object ThriftClientFramedCodec {
	  /**
	   * Create a [[com.twitter.finagle.thrift.ThriftClientFramedCodecFactory]].
	   * Passing a ClientId will propagate that information to the server iff the server is a finagle
	   * server.
	   */
	  def apply(clientId: Option[ClientId] = None) =
	    new ThriftClientFramedCodecFactory(clientId)
	
	  def get() = apply()
	}
	
	class ThriftClientFramedCodecFactory(
	    clientId: Option[ClientId],
	    _useCallerSeqIds: Boolean,
	    _protocolFactory: TProtocolFactory)
	  extends CodecFactory[ThriftClientRequest, Array[Byte]]#Client {
	
	  def this(clientId: Option[ClientId]) = this(clientId, false, Protocols.binaryFactory())
	
	  def this(clientId: ClientId) = this(Some(clientId))
	
	  // Fix this after the API/ABI freeze (use case class builder)
	  def useCallerSeqIds(x: Boolean): ThriftClientFramedCodecFactory =
	    new ThriftClientFramedCodecFactory(clientId, x, _protocolFactory)
	
	  /**
	   * Use the given protocolFactory in stead of the default `TBinaryProtocol.Factory`
	   */
	  def protocolFactory(pf: TProtocolFactory) =
	    new ThriftClientFramedCodecFactory(clientId, _useCallerSeqIds, pf)
	
	  /**
	   * Create a [[com.twitter.finagle.thrift.ThriftClientFramedCodec]]
	   * with a default TBinaryProtocol.
	   */
	  def apply(config: ClientCodecConfig) =
	    new ThriftClientFramedCodec(_protocolFactory, config, clientId, _useCallerSeqIds)
	}
	
	class ThriftClientFramedCodec(
	  protocolFactory: TProtocolFactory,
	  config: ClientCodecConfig,
	  clientId: Option[ClientId] = None,
	  useCallerSeqIds: Boolean = false
	) extends Codec[ThriftClientRequest, Array[Byte]] {
	
	  private[this] val preparer = ThriftClientPreparer(
	    protocolFactory, config.serviceName,
	    clientId, useCallerSeqIds)
	
	  def pipelineFactory: ChannelPipelineFactory =
	    ThriftClientFramedPipelineFactory
	
	  override def prepareConnFactory(
	    underlying: ServiceFactory[ThriftClientRequest, Array[Byte]]
	  ) = preparer.prepare(underlying)
	
	  override val protocolLibraryName: String = "thrift"
	}		
	```
		
Scala源码看起来太费劲, 既然知道了原理, 为了可以解析出具体的Fiangle-thrift中的东西, 我只需要设置FrameSize和data的offset的位置, 获取到原生的Thrift协议中的Framed数据即可, 然后复用Packetbeat自带的针对Thrift协议包的抓取与组合逻辑.
		
通过比对两个业务包我知道中间Fiangle-thrift自己添加的信息字节大小固定为129个字节,这只是Client在发送请求时才会添加这些附加信息, Server端返回值则是在原生Thrift协议中添加了1个字节, 其中我还需要排除创建连接之后发送的Finagle-thrift协议确认请求. 
		
我们完成了普通Finagle-thrift协议的解析,接下来还要针对附带Zipkin信息的Finagle-thrift协议的解析, Zipkin是参考Google的Dapper完成的可以对RPC调用链进行跟踪的框架, 这已经是业内针对分布式系统之间RPC调用链跟踪的通用解决方案. Zipkin无非就是在RPC调用时多传输了TraceId, SpanId, ParentSpanId, IsSample等信息, 通过下面的Zipkin源码可以确定这些信息的大小也是固定字节,并且大小为4个字节. Zipkin关于这块的源码如下:
		
	```
	/**
     * The wire format is (big-endian):
     *     ''spanId:8 parentId:8 traceId:8 flags:8''
     */
    def tryUnmarshal(body: Buf): Try[TraceId] = {
	      if (body.length != 32)
	        return Throw(new IllegalArgumentException("Expected 32 bytes"))
	
	      val bytes = local.get()
	      body.write(bytes, 0)
	
	      val span64 = ByteArrays.get64be(bytes, 0)
	      val parent64 = ByteArrays.get64be(bytes, 8)
	      val trace64 = ByteArrays.get64be(bytes, 16)
	      val flags64 = ByteArrays.get64be(bytes, 24)
	
	      val flags = Flags(flags64)
	      val sampled = if (flags.isFlagSet(Flags.SamplingKnown)) {
	        if (flags.isFlagSet(Flags.Sampled)) someTrue else someFalse
	      } else None
	
	      val traceId = TraceId(
	        if (trace64 == parent64) None else Some(SpanId(trace64)),
	        if (parent64 == span64) None else Some(SpanId(parent64)),
	        SpanId(span64),
	        sampled,
	        flags)
	
	      Return(traceId)
    }
	```

		
下面是Fiangle-thrift针对Zipkin关闭和开启的一个抓包对比图:
![2016-04-30_13-46-49](http://siye1982.github.io/img/blog/beats/2016-04-30_13-46-49.png)

根据上面的分析逻辑,我就可以在Packetbeat中的messageParser方法中通过一些字节特征修正FrameSize和data的offset来把数据包变成原生的Thrift协议数据包, 具体代码如下:
		
	```
	func (thrift *Thrift) messageParser(s *ThriftStream, dir uint8) (bool, bool) {
		var ok, complete bool
		var m = s.message
		
		dataStr := string(s.data)
		//dataLen := len(s.data)
		
		
		if (!strings.Contains(dataStr, "__can__finagle__trace__v3__")) {
			if dir == uint8(1) {// client -> server
				var thriftFlag []byte
				thriftFlag,_ = hex.DecodeString("8001")
				if bytes.Equal(s.data[133:135] , thriftFlag) {// 普通finagle
					m.FrameSize = common.Bytes_Ntohl(s.data[:4]) - 129
					s.parseOffset = 4 + 129
				}else { // 附带zipkin
					m.FrameSize = common.Bytes_Ntohl(s.data[:4]) - 129 - 4
					s.parseOffset = 4 + 129 + 4
					//fmt.Println("&&&&&&&&&&&&&&&&&&&&&&&&&& len: [", dataLen, "]  data: [", s.data, "] flag: [", s.data[137:139],"]")
				}
		
			}else { // server -> client
				m.FrameSize = common.Bytes_Ntohl(s.data[:4]) - 1
				s.parseOffset = 4 + 1
			}
		}
		
		... ...
	}
	```

# 参考

[Thrift Tutorial](http://thrift-tutorial.readthedocs.io/en/latest/thrift-stack.html)

[Packetbeat协议扩展开发教程](http://log.medcl.net/item/2015/12/packetbeat-zi-ding-yi-xie-yi-di-kai-fa-jiao-cheng/)

[由浅入深了解Thrift](http://houjixin.blog.163.com/blog/static/3562841020150165145988/)






<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>

