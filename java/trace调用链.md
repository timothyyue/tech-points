###基于Thrift服务RPC调用链跟踪

#### 概述
我们现在所处的生产环境是一个集Java, PHP等多种语言程序的混合场景.owl-trace框架的一个基于Thrift协议的RPC调用链跟踪的工具，可搜集各服务调用数据，并提供分析查询展示功能。帮助识别分布式RPC调用链里，哪一个调用比较耗时，性能有问题，以及是否有异常等，使得诊断分布式系统性能成为可能。


#### 基本概念
#### Trace
一次服务调用追踪链路，由一组Span组成。需在web总入口处生成TraceID，并确保在当前请求上下文里能访问到。

#### Annotation
表示某个时间发生的Event。

* Event类型

```
cs：Client Send 请求
sr：Server Receive到请求
ss：Server 处理完成、并Send Response
cr：Client Receive 到响应
```

* 什么时候生成

客户端发送Request、接受到Response、服务器端接受到Request、发送Response时生成。Annotation属于某个Span，需把新生成的Annotation添加到当前上下文里Span的annotations数组里

* Thrift数据结构

 
```
struct Annotation {
 /**
   * 为当前时间的毫秒(currentTimeMillis).
   */
  1: i64 timestamp
  /**
   * Usually a short tag indicating an event, like "sr".
   * 四个值分别为：CS, CR, SS, SR
   */
  2: string value
}
```

#### BinaryAnnotation

存放用户自定义信息，比如：方法内资源请求的耗时、异常等

* 什么时候生成

在任意需要记录自定义跟踪信息时都可生成。比如：异常、SessionID等。如Annotation一样，BinaryAnnotation也属于某个Span。需把新生成的BinaryAnnotation，添加到当前上下文里Span的binary_annotations数组.

* Thrift数据结构

```
struct BinaryAnnotation {
  /**
   * Name used to lookup spans, such as "mysql.elapsed".
   */
  1: string key,
  /**
   * key's value.
   */
  2: string value,
  /**
  * custom type.
  */
  3: string type
}
```
#### Span
表示一次完整RPC调用，是由一组Annotation和BinaryAnnotation组成。是追踪服务调用的基本结构，多span形成树形结构组合成一次Trace追踪记录。Span是有父子关系的，比如：Client A、Client A -> B、B ->C、C -> D、分别会产生4个Span。Client A接收到请求会时生成一个Span A、Client A -> B发请求时会再生成一个Span A-B，并且Span A是 Span A-B的父节点

* 什么时候生成


	*  服务接受到 Request时，若当前Request没有关联任何Span，便生成一个Span，包括：Span ID、TraceID
	* 向下游服务发送Request时，需生成一个Span，并把新生成的Span的父节点设置成上一步生成的Span

	
* Thrift结构

```
struct Span {
/**
*  for a trace, set on all spans within it.
*/
 1: string trace_id
/**
*  service name
*/
 2: string serviceId,
/**
* Span name in lowercase, rpc method for example. Conventionally, when the
* span name isn't known, name = "unknown".
*/
 3: string spanName,
/**
* of this span within a trace. A span is uniquely
* identified in storage by (trace_id, id).
*/
 4: string id,
/**
* The parent's Span.id; absent if this the root span in a trace.
*/
 5: optional string parent_id,
/**
* Associates events that explain latency with a timestamp. Unlike log
* statements, annotations are often codes: for example SERVER_RECV("sr").
* Annotations are sorted ascending by timestamp.
*/
 6: list<Annotation> annotations,
/**
* local server address.
*/
 7: string ip,
/**
* Tags a span with context, usually to support query or aggregation. For
* example, a binary annotation key could be "http.uri".
*/
 8: list<BinaryAnnotation> binary_annotations
/**
* is save trace log, or not.
*/
 9: optional bool sample
}
```
### 服务之间需传递的信息
Trace的基本信息需在上下游服务之间传递，如下信息是必须的：

* Trace ID：起始(根)服务生成的TraceID
* Span ID：调用下游服务时所生成的Span ID
* Parent Span ID：父Span ID
* Is Sampled：是否需要采样

### Trace Tree组成

一个完整Trace 由一组Span组成，这一组Span必须具有相同的TraceID；Span具有父子关系，处于子节点的Span必须有parent_id，Span由一组 Annotation和BinaryAnnotation组成。整个Trace Tree通过Trace Id、Span ID、parent Span ID串起来的。

### 其他要求

* Web入口处，需把HTTP_X_OWL_RID、IS_SAMPLE等信息放进request header中。
* 关键子子调用也需追踪，比如：订单调用了Mysql，也需把个调用的耗时情况记录到 BinaryAnnotation里
* 关键出错日志或者异常也许记录到BinaryAnnotation里（可以不用，有logger采集错误日志）

### 完整示例

testA(Web服务) -> testB(Thrift) -> testC & testD(Thrift)。一共有四个服务，testA 调用 testB, testB同时调用 testC和testD。需生成的Trace信息如下：

* testA收到Http Reqeust时，需在入口处获取或者自己生成TraceID、SpanID，以及一个Span对象，假若叫Span1。
* testA向testB发送 Thrift Request时，需新生成一Span2，并把parent ID设置成Span1的spanID。同事需修改Thrift 上下文，把Span2的spanID、parent ID、TraceID 传递给下游服务。也需生成"cs" Annotation，关联到span2上；当接受到testB的Response时，再生成"cr" Annotation，也关联到span2上。
* testB接受到Thrift Request后，从Thrift 上下文里解析到TraceID、parent ID、 Span ID(span2)、并保留到上下文里。同时生成"sr"Annotaition，并关联到span2上；当处理完成发送response时，再生成"ss"Annotation，并关联到span2上。
* testB向testC发送 Thrift Request时，需新生成一Span3，并把parentID设置成上一步(Span2)的span ID。Annotation处理如上。
* testB向testC发送请求时，新生成一Span4，并把parentID设置Span2的span ID。Annotation处理如上。

### Trace数据生成

* Web入口处，生成TraceID、SpanID、以及Span对象.
*  调用下游服务时生成Span对象.
* Thrift客户端Send Request时，生成"cs" Annotation.
* Thrift客户端Receive Response时，生成"cr"Annotation.
* Thrift服务器端Receive Reqeust时，生成"sr" Annotation.
* Thrift服务器端Send Response时，生成"ss"Annotation.
*  异常、关键系统调用数据要记录到BinaryAnnotation里.


### Trace数据传递与收集

* 数据刷新不能影响业务性能，刷新失败更不能影响正常业务逻辑.
* 生成Trace数据，写入到本地log文件
* 由logstash(类似工具)统一搜集日志，并写入kafka
* 由kafka consumer从kafka读取数据，进行清洗和数据分析

### 跟踪数据收集格式定义

我们针对跟踪数据的收集的统一接入点, 为kafka.所有应用产生的trace数据统一发送到Kafka集群中.具体的topic为: trace2.


### Span(记录每次调用信息)
```
span{
	"annotations":[{"timestamp":1515469573421,"value":"cs"},{"timestamp":1515469573627,"value":"cr"}],
	"binaryAnnotations":[],
	"id":"1070e98c-2d44-41e8-b78b-572f021d90ad",
	"ip":"192.168.53.40",
	"parentId":"db4a9578-53f4-4d4b-aaa6-f2a806da83ed",
	"sample":true,
	"serviceId":"B",
	"spanName":"get",
	"traceId":"uuid-0aa97856-7da6-4494-a889-1b1b4e507f80"
}
```

**备注: span信息的json,添加'span'头信息, 在通过kafka做收集时, 通过该头信息将span信息路由到独立的es的index中. 每一条span记录其实是半个span信息, 要么是client端产生的, 要么是server端产生的.**

**BinaryAnnotation(可以用来传递用户的信息, 也可以传递其他业务信息)**


### java使用trace调用链经验

* 客户端调用服务端失败（服务端没启动），客户端的annotations有cs、cr
* 客户端超时（等待响应超时），服务端的annotations有sr、ss，客户端的annotations有cs、cr
* 服务端超时，服务端的annotations有sr、ss, 客户端的annotations有cs、cr


### 实际效果
收集了trace产生的RPC调用链信息,并给服务治理框架提供跟踪信息检索(点击详情Span记录可以查看某次调用链详情,了解具体的网络情况和业务执行情况)

* trace列表页
![](/Users/timothy/Downloads/rpc_list.png)

* trace详情页
![](/Users/timothy/Downloads/rpc_detail.png)

* 错误链路追踪
![](/Users/timothy/Downloads/error_rpc.png)








