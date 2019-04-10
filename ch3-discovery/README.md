# MicroServices - Part 3 : Spring Cloud Service Registry and Discovery
在微服务世界里，服务注册和发现扮演着十分重要的角色。为什么这么说，因为同一微服务通常会有多个实例在运行，并且云环境中的实例随时可能启停，我们不能硬编码域名或地址，需要一种自动化的机制来帮助调用。Spring Cloud的服务注册与发现特性就提供了这样一种机制。在本章中，我们使用SpringCloud Netflix Eureka实现服务的注册与发现。

本章主要内容包括：
- 什么是服务注册与发现
- SpringCloud Netflix Eureka服务注册
- 注册成为Eureka客户端
- 使用Eureka客户端发现服务

## 什么是服务注册与发现
假设我们有两个微服务catalog-service和inventory-service，前者负责查询产品，后者负责查询库存；在查询产品时，也需要调用库存REST接口。那么问题来了，如果存在两个或多个inventory-service的实例，应该调用哪一个呢？一般而言，我们会使用负载均衡器作为中间件，catalog-service访问负载均衡器而非直接调用inventory-service。

这种方式仍然面临挑战：如果增加新的实例呢？如果实例挂掉了呢？频繁手工调整负载均衡器既无聊又容易出错，所以我们需要一种自动化的方式，并且使用一个逻辑ID来标识和访问服务。

Netflix Eureka Server用来扮演服务注册中心的角色；微服务作为Eureka客户端，启动后自动向Eureka注册，并定期更新状态；同时，微服务也可通过逻辑ID来调用REST接口。借助Spring Cloud，通过简单步骤即可创建服务注册中心，使用负载均衡的RestTemplate发现服务。

## SpringCloud Netflix Eureka服务注册
首先创建服务注册中心，其实就是一个SpringBoot应用。进入[Spring Initializr](https://start.spring.io/)，artifact栏输入service-registry，Dependencies中加入Eureka Server，创建工程。
在入口函数处增加@EnableEurekaServer的注解：
```java
@EnableEurekaServer
@SpringBootApplication
public class ServiceRegistryApplication {

	public static void main(String[] args) {
		SpringApplication.run(ServiceRegistryApplication.class, args);
	}

}
```
然后配置application.properties文件：
```
spring.application.name=service-registry
server.port=8761

eureka.client.registerWithEureka=false
eureka.client.fetchRegistry=false
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
```
默认情况下，Eureka Server也是Eureka客户端，需要至少一个服务URL来定位peer，避免出现单点失效；简单起见，我们使用Standalone模式，只用一个服务节点，所以上述配置中有两个false项。

启动工程，进入[http://localhost:8761](http://localhost:8761)即可查看UI。*写工程时我尝试使用bootstrap和配置服务，但是Eureka服务器总是尝试连接peer，也就是false项没生效。哪位搞定了麻烦给我说一声。*

*Eureka文档请移步[Github Wiki](https://github.com/Netflix/eureka/wiki)，请注意Eureka 2.0版本已经放弃了开发（可能只是放弃了开源）。*

*Eureka Standalone运行一段时间后会出现如下警告：*
```
EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.
```
*原因在于Renews threshold和Renews (last min)，Eureka服务默认每个客户端30秒发送一次心跳信息，所以每一个新客户端连接时，Renews threshold都+2；然后统计每分钟收到的心跳信息，体现在Renews (last min)中。如果Renews (last min)小于Renews threshold的85%，则会启动Self Preservation模式，出现上述警告信息。问题在于Standalone的服务器要算自己，Renews threshold+1，而不向自己发送心跳信息，所以Renews (last min)永远比Renews threshold小1，在客户端比较少时，肯定触发Self Preservation模式。*

*Self Preservation模式下，Eureka服务器不再过期客户端，也就是说，即使不再收到客户端的心跳信息，也会保留客户端的信息。这种模式用于应对大规模的心跳丢失，发生这种情况最可能的原因是Eureka服务器自己的网络故障。为什么要这样呢，举个例子，A和B都连着Eureka服务器，A要调用B接口，需要从Eureka中发现服务；突然B和Eureka的网络断了，但A和B之间的连接还是通的，此时的Self Preservation模式就能保证A仍然能通过Eureka发现B，然后调用B的服务。*

*对Self Preservation模式的说明请移步[Github Wiki](https://github.com/Netflix/eureka/wiki/Server-Self-Preservation-Mode)。*

## 注册成为Eureka客户端
接下来我们修改catalog-service工程，注册成为Eureka的客户端。

向catalog-service的pom.xml中增加依赖：
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>
    spring-cloud-starter-netflix-eureka-client
  </artifactId>
</dependency>
```
然后修改Git上对应的catalog-service.properties，增加注册配置项：
```
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```
注意保证此处的注册地址与Eureka服务器的配置地址保持一致。

运行工程，再次进入[http://localhost:8761](http://localhost:8761)，在UI上可看到catalog-service被分配了一个逻辑ID：CATALOG-SERVICE。

## 使用Eureka客户端发现服务
我们创建一个新的工程inventory-service，用于提供库存查询的接口。

进入[Spring Initializr](https://start.spring.io/)，artifact栏输入inventory-service，Dependencies中加入Reactive Web、JPA、MySQL、Actuator、Lombok、Config Client、Vault Configuration和Eureka Discovery，创建工程。

创建JPA实体InventoryItem.java
```java
@Data
@Entity
@Table(name = "inventory")
public class InventoryItem {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    @Column(name = "product_code", nullable = false, unique = true)
    private String productCode;
    @Column(name = "quantity")
    private Integer availableQuantity = 0;
}
```
创建JPA仓库InventoryItemRepository.java
```java
public interface InventoryItemRepository extends JpaRepository<InventoryItem, Long> {

    Optional<InventoryItem> findByProductCode(String productCode);

}
```
简单起见，直接创建控制层InventoryController.java
```java
@RestController
@Slf4j
public class InventoryController {
    private final InventoryItemRepository inventoryItemRepository;

    @Autowired
    public InventoryController(InventoryItemRepository inventoryItemRepository) {
        this.inventoryItemRepository = inventoryItemRepository;
    }

    @GetMapping("/api/inventory/{productCode}")
    public ResponseEntity<InventoryItem> findInventoryByProductCode(@PathVariable("productCode") String productCode) {
        log.info("Finding inventory for product code :"+productCode);
        Optional<InventoryItem> inventoryItem = inventoryItemRepository.findByProductCode(productCode);
        if(inventoryItem.isPresent()) {
            return new ResponseEntity(inventoryItem, HttpStatus.OK);
        } else {
            return new ResponseEntity(HttpStatus.NOT_FOUND);
        }
    }
}
```
为方便演示，在数据库中加入一些数据，在src/main/resources中加入data.sql文件，程序启动时JPA会自动加载此文件：
```sql
DELETE FROM inventory;
insert into inventory(id, product_code, quantity) VALUES
(1, 'P001', 250),
(2, 'P002', 132),
(3, 'P003', 0);
```
创建bootstrap.properties文件使用配置服务，内容如下：
```
spring.application.name=inventory-service
server.port=8282

management.endpoints.web.exposure.include=*

spring.cloud.config.uri=http://localhost:8888

spring.cloud.vault.host=0.0.0.0
spring.cloud.vault.port=8200
spring.cloud.vault.scheme=http
spring.cloud.vault.kv.enabled=true
spring.cloud.vault.authentication=token
spring.cloud.vault.token=s.cIwe6aLokLJrNLn1qj0MvXWb
```
将inventory-service.properties文件上传至配置服务的git目录中，内容如下：
```
logging.level.com.example=debug

spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/inventory?useSSL=false
#spring.datasource.username=root
#spring.datasource.password=222222

spring.datasource.initialization-mode=always
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```
inventory-service使用本地MySQL数据库，使用的数据库是inventory库，需要提前创建，注意用户名和密码匹配；JPA设置的模式是update，会自动创建表。和之前一样，在Vault中写入数据库用户名和密码：
```sh
vault kv put secret/inventory-service spring.datasource.username=root spring.datasource.password=222222
```
接下来修改catalog-service工程，通过逻辑ID和负载均衡的方式，调用inventory-service的接口。

修改CatalogServiceApplication.java
```java
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
上述代码声明了一个RestTemplate的Bean，并通过@LoadBalanced注解将这个Bean变成了使用Ribbon LoadBalancer的负载均衡访问器，结合服务发现机制，可使用服务逻辑ID调用接口。

*Ribbon是一个伞形项目，包含多个独立的模块，请参考[Github文档](https://github.com/Netflix/ribbon)；如文档所言，目前项目处于维护状态，未来Netflix考虑更多使用gRPC；这对开发者也是一个参考。*

创建ProductInventoryResponse.java
```java
@Data
public class ProductInventoryResponse {
    private String productCode;
    private Integer availableQuantity = 0;
}
```
上述代码用于接收从接口返回的数据。*这段代码和inventory-service的InventoryItem.java代码基本一致，这里就提出一个问题，同样的代码需要写多次吗？如何确保字段正确性？或许用ProtoBuf更为合适。结合上面的Ribbon说明，个人倾向使用gRPC。*

创建InventoryServiceClient.java：
```java
@Service
@Slf4j
public class InventoryServiceClient {
	private final RestTemplate restTemplate;

	@Autowired
	public InventoryServiceClient(RestTemplate restTemplate) {
		this.restTemplate = restTemplate;
	}

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
}
```
上述代码使用负载均衡的RestTemplate调用inventory-service提供的接口，请注意地址使用的是服务的逻辑ID。

修改ProductService.java：
```java
@Service
@Transactional
@Slf4j
public class ProductService {
	private final ProductRepository productRepository;
	private final InventoryServiceClient inventoryServiceClient;

	@Autowired
	public ProductService(ProductRepository productRepository, InventoryServiceClient inventoryServiceClient) {
		this.productRepository = productRepository;
		this.inventoryServiceClient = inventoryServiceClient;
	}

	public List<Product> findAllProducts() {
		return productRepository.findAll();
	}

	public Optional<Product> findProductByCode(String code) {
		Optional<Product> productOptional = productRepository.findByCode(code);
        if (productOptional.isPresent()) {
            log.info("Fetching inventory level for product_code: " + code);
            Optional<ProductInventoryResponse> itemResponseEntity =
                    this.inventoryServiceClient.getProductInventoryByCode(code);
            if (itemResponseEntity.isPresent()) {
                Integer quantity = itemResponseEntity.get().getAvailableQuantity();
                productOptional.get().setInStock(quantity > 0);
            }
        }
        return productOptional;
	}
}
```
此时查询产品[http://localhost:8181/api/products/P001](http://localhost:8181/api/products/P001)，就能看到调用了inventory-service的接口。可尝试手动增加inventory-service实例（注意端口不要冲突），观察负载均衡现象。

## 总结
通过Spring Cloud提供的服务注册与发现机制，妈妈再也不用担心我的调用了。:smile:
