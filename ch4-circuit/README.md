# MicroServices - Part 4 : Spring Cloud Circuit Breaker using Netflix Hystrix
在微服务的环境中，服务之间相互调用可以说是日常。哪怕只有一个服务不可用，也可能会引发级联反应，导致大面积服务不可用。不可用有两种含义：1. 服务直接报错；2. 服务长时间未响应。为应对这种情况，Netflix创造了Hystrix库，通过Circuit Breaker模式来保证服务的可用性。Spring Cloud提供了对Hystrix Circuit Breaker的支持，通过很Spring的简单方式即可实现断路保护。

本章主要内容包括：
- 使用@HystrixCommand实现断路保护
- 使用Hystrix Dashboard监控断路

*更多关于Circuit Breaker，请移步[MartinFowler的文章](https://martinfowler.com/bliki/CircuitBreaker.html)。*

*Hystrix项目也进入到了[维护模式](https://github.com/Netflix/Hystrix)，对于原因，Netflix提到了两点：1. 现在更关注基于应用实时性能的自适应方式，而非传统的预设置方式；2. 考虑使用其它开源项目，比如[resilience4j](https://github.com/resilience4j/resilience4j)。Netflix也推荐大家这样做。We are beginning to recommend others do the same.*

## 使用@HystrixCommand实现断路保护
在前面的章节中，catalog-service调用了inventory-service的接口来获取库存信息。如果inventory服务不可用该怎么办呢？对应不可用的两种含义，我们需要加入两种机制：超时和回退。超时机制保证了我们不会受到长时间未响应的影响，回退机制保证了服务可用性不会发生级联反应。

在catalog-service的pom.xml中加入依赖：
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>
    spring-cloud-starter-netflix-hystrix
  </artifactId>
</dependency>
```
在入口函数处加上@EnableCircuitBreaker注解，启用断路保护：
```java
@EnableCircuitBreaker
@SpringBootApplication
public class CatalogServiceApplication {

	@Bean
  @LoadBalanced
  public RestTemplate restTemplate() {
    return new RestTemplate();
  }

	public static void main(String[] args) {
		SpringApplication.run(CatalogServiceApplication.class, args);
	}

}
```
修改InventoryServiceClient.java，通过@HystrixCommand注解启用超时和回退机制：
```java
@Service
@Slf4j
public class InventoryServiceClient {
	private final RestTemplate restTemplate;

	@Autowired
	public InventoryServiceClient(RestTemplate restTemplate) {
		this.restTemplate = restTemplate;
	}

	@HystrixCommand(commandKey = "inventory-by-productcode", fallbackMethod = "getDefaultProductInventoryByCode")
	public Optional<ProductInventoryResponse> getProductInventoryByCode(String productCode) {
		ResponseEntity<ProductInventoryResponse> itemResponseEntity = restTemplate.getForEntity(
				"http://inventory-service/api/inventory/{code}", ProductInventoryResponse.class, productCode);
		if (itemResponseEntity.getStatusCode() == HttpStatus.OK) {
			return Optional.ofNullable(itemResponseEntity.getBody());
		} else {
			log.error("Unable to get inventory level for product_code: " + productCode + ", StatusCode: "
					+ itemResponseEntity.getStatusCode());
			return Optional.empty();
		}
	}

	@SuppressWarnings("unused")
	Optional<ProductInventoryResponse> getDefaultProductInventoryByCode(String productCode) {
		log.info("Returning default ProductInventoryByCode for productCode: " + productCode);
		ProductInventoryResponse response = new ProductInventoryResponse();
		response.setProductCode(productCode);
		response.setAvailableQuantity(50);
		return Optional.ofNullable(response);
	}
}
```
@HystrixCommand注解是一个Decorator，将方法调用包装成一个Hystrix的命令。commandKey指定了命令的名称，fallbackMethod指定了回退机制。注意回退方法必须定义在同一个类中，并且签名一致。除了直接在注解中设置参数，还可以通过配置文件设置。

在Git的catalog-service.properties文件中加入如下内容：
```
hystrix.command.inventory-by-productcode.execution.isolation.thread.timeoutInMilliseconds=2000
hystrix.command.inventory-by-productcode.circuitBreaker.errorThresholdPercentage=60
```
前者指定了超时机制，后者指定了断路开关打开的阈值。也就是说，如果一段时间内超过60%的调用出错，后续一段时间的调用直接走回退逻辑。

## 使用Hystrix Dashboard监控断路
配合Spring Boot Actuator，Hystrix命令的状态可以以事件流的形式展示出来，链接地址[http://localhost:8181/actuator/hystrix.stream]( http://localhost:8181/actuator/hystrix.stream)。说实话展示方式有点反人类，所以Spring Cloud另外提供了一个很nice的Dashboard来帮助监控Hystrix命令。

进入[Spring Initializr](https://start.spring.io/)，artifact栏输入hystrix-dashboard，Dependencies中加入Hystrix Dashboard，创建工程。

在入口函数处加上@EnableHystrixDashboard注解启用Dashboard：
```java
@EnableHystrixDashboard
@SpringBootApplication
public class HystrixDashboardApplication {

	public static void main(String[] args) {
		SpringApplication.run(HystrixDashboardApplication.class, args);
	}

}
```
在application.properties文件中加入如下内容：
```
spring.config.name=hystrix-dashboard
server.port=8788
```
启动工程，从[http://localhost:8788/hystrix](http://localhost:8788/hystrix )进入到Dashboard，在中间地址栏输入[http://localhost:8181/actuator/hystrix.stream](http://localhost:8181/actuator/hystrix.stream)，然后点击Monitor Stream即可。

![Dashboard截图](https://github.com/sumuzhou/microservice-spring-cloud/blob/add_ch4/ch4-circuit/20190409145010.png "Dashboard截图")

图中左上方波形图旁边的数字，上面代表成功调用次数，中间代表断路保护次数，下面代表失败调用次数。停止inventory-service服务，然后频繁调用catalog-service接口，开始的几次调用会等待一段时间（也就是超时时间），后面的调用会直接返回，此时在监控页面可发现断路保护已打开，调用走的都是回退逻辑。

Spring Cloud提供的Dashboard也存在问题，一次只能看一个服务的Hystrix命令运行状况。我们可以使用Turbine提供一个统一的视图，查看所有服务的状况。欲了解详情请移步[http://cloud.spring.io/spring-cloud-static/Greenwich.SR1/single/spring-cloud.html#_turbine]( http://cloud.spring.io/spring-cloud-static/Greenwich.SR1/single/spring-cloud.html#_turbine)。
