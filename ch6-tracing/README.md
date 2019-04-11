# MicroServices - Part 6 : Distributed Tracing with Spring Cloud Sleuth and Zipkin
微服务架构的面临的一个挑战是如何Debug。一个简单的调用可能触发一系列的连锁调用。如果没有辅助工具，Debug基本可以GG了。另外还有一个需求是希望知道这一系列的调用都各自用了多长时间。Spring Cloud Sleuth提供了分布式追踪能力，同时我们可以将追踪数据导入到Zipkin来进行可视化。

本章主要内容包括：
- 追踪分布式服务调用
- 使用Spring Cloud Sleuth实现追踪
- 使用Zipkin服务器

*讲一个小插曲：有次测试环境出了非常诡异的问题，某些功能大部分时间测试都是正常的，少数几次调用会出错；大家排查了半天，最后发现是某程序猿把他开发环境的服务注册中心地址写成了测试环境的，导致少数调用会走到他的开发环境中~~~ 1只草泥马飘过，再论追踪的重要性。*

## 追踪分布式服务调用
在前面的例子中，UI可能通过网关调用产品查询，接着触发库存查询。假设调用报错了，或者返回的结果不正确，我们肯定想通过log看哪里出了问题。那么问题来了，网关、产品服务、库存服务的log相互独立，如何才能关联起来呢？（这样才能知道哪些产品查询的log和库存查询的log有关）

一种解决方案是手动增加CORRELATION_ID，所有的log都包含这个ID，并且这个ID也放在header中，这样就能通过ID来辨别哪些log是有关联的。

我们可以通过日志框架的[MDC特征](https://logback.qos.ch/manual/mdc.html)实现上述的方案。一般来讲，我们通过拦截器来检查是否存在CORRELATION_ID，如果不存在则创建一个，并在MDC中设置；如果存在则直接设置。如此即可实现日志关联。

但是，为何要发明轮子呢。Spring Cloud Sleuth就为我们提供了这样的机制。

## 使用Spring Cloud Sleuth实现追踪
首先需要熟悉一些术语：Span、Trace和Annotaion。请移步文档[http://cloud.spring.io/spring-cloud-static/Greenwich.SR1/single/spring-cloud.html#_terminology](http://cloud.spring.io/spring-cloud-static/Greenwich.SR1/single/spring-cloud.html#_terminology)。

在catalog-service和inventory-service的pom.xml文件中都增加如下内容：
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```
启动服务，调用[http://localhost:8181/api/products/P001](http://localhost:8181/api/products/P001)，观察catalog和inventory服务产生的日志：

**catalog**
```
2019-04-11 10:43:01.538  INFO [catalog-service,c407efff1439ccb7,c407efff1439ccb7,false] 11964 ---
```
**inventory**
```
2019-04-11 10:43:02.931  INFO [inventory-service,c407efff1439ccb7,dc855f672c37e5f3,false] 11528 ---
```
上述日志方括号的内容为[appname,traceId,spanId,exportable]，可以看到两者有着同样的traceId，这样就能把日志关联起来。最后的false是因为我们把日志没有导出，马上就说这事。

## 使用Zipkin服务器
创建Zipkin服务器有多种方式，这里我们采用Docker的方式。执行以下命令：
```sh
docker run -d -p 9411:9411 --name zipkin openzipkin/zipkin:2.12.6
```
在catalog-service和inventory-service的pom.xml文件中都增加如下内容：
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```
在catalog-service和inventory-service的Git配置文件中都增加如下内容：
```
spring.zipkin.base-url=http://localhost:9411/
spring.sleuth.sampler.probability=1
```
spring.sleuth.sampler.probability用于设置追踪信息上传的百分比，这里我们为了方便设置成100%，实际环境不需要这么高。

进入[Zipkin UI](http://localhost:9411/zipkin/)，调用一次[http://localhost:8181/api/products/P001](http://localhost:8181/api/products/P001)，点击查找，就能看到刚才的调用情况：

![Zipkin查找](https://github.com/sumuzhou/microservice-spring-cloud/blob/master/ch6-tracing/20190411105837.png "Zipkin查找")

点击可进入查看调用链的详情：

![Zipkin调用链](https://github.com/sumuzhou/microservice-spring-cloud/blob/master/ch6-tracing/20190411105923.png "Zipkin调用链")

点击单次调用可查看详情：

![Zipkin单次调用](https://github.com/sumuzhou/microservice-spring-cloud/blob/master/ch6-tracing/20190411105953.png "Zipkin单次调用")

这些信息在Debug时对我们很有帮助，再也不用盯着日志老眼昏花了。
