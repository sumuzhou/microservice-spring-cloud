# MicroServices - Part 5 : Spring Cloud Zuul Proxy as API Gateway
在前面的章节中，我们借助Eureka实现了通过逻辑ID调用服务，并且可在多个实例间实现负载均衡。但是，对于一个外部访问者，比如UI，他既不该知道我们的微服务架构，也不该知道我们服务实例的运行状况；对他而言，我们的微服务作为一个整体的应用呈现。如何支持外部访问？此时我们可以提供统一的API网关，然后根据某种方式，比如URL模式，将请求转发给对应的服务。在本章中，我们使用Spring Cloud Zuul Proxy创建API网关。

本章主要内容包括：
- API网关的优缺点
- 基于Spring Cloud Zuul Proxy实现API网关

## API网关的优缺点
使用API网关存在以下利弊：

**优点**
- 简化客户端调用接口的方式
- 避免将微服务内部结构暴露给客户端
- 避免客户端对微服务细节的依赖，可以重构微服务而不影响客户端
- 提供集中化处理的平台，比如安全、监控、限流等

**缺点**
- 需要考虑高可用以避免单点失效
- 可能会受到微服务API设计的影响

## 基于Spring Cloud Zuul Proxy实现API网关
Spring Cloud提供了Zuul代理，类似Nginx，可用于创建API网关。

进入[Spring Initializr](https://start.spring.io/)，artifact栏输入api-gateway，Dependencies中加入Zuul、Config Client和Eureka Discovery，创建工程。

在入库函数处加上@EnableZuulProxy启用代理：
```java
@EnableZuulProxy
@SpringBootApplication
public class ApiGatewayApplication {

	public static void main(String[] args) {
		SpringApplication.run(ApiGatewayApplication.class, args);
	}

}
```
将applicaiton.properties改成bootstrap.properties：
```
spring.application.name=api-gateway
server.port=8080

spring.cloud.config.uri=http://localhost:8888
```
然后将api-gateway.properties上传至Git：
```
logging.level.com.example=debug

eureka.client.service-url.defaultZone=http://localhost:8761/eureka/

zuul.ignored-services=*

zuul.routes.catalogservice.path=/catalog/**
zuul.routes.catalogservice.serviceId=catalog-service
```
上述配置中，zuul.ignored-services默认忽略所有的服务，除了下面手动定义的；zuul.routes定义了catalog服务，说明了其逻辑ID是catalog-service，并指定前缀路径是/catalog。如此所有前缀为/catalog的请求都会被转发至逻辑ID为catalog-service的地址上。网关借助Eureka服务发现机制，将剥离前缀后的请求转发过去。

保持catalog服务、注册服务和配置服务为运行状态，启动网关服务，访问[http://localhost:8080/catalog/api/products](http://localhost:8080/catalog/api/products)，即可看到产品信息。
