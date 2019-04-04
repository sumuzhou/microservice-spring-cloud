# MicroServices - Part 2 : Configuration Management with Spring Cloud Config and Vault
本章主要内容包括：
- Spring Cloud Config & Vault的需求
- 创建配置服务
- 创建catalog-service微服务
- 使用Vault保存敏感数据

## Spring Cloud Config & Vault的需求
对于单个服务，SpringBoot提供了丰富的配置选择；但是在微服务的语境下，通常存在多个服务，每个服务又可能有多个运行实例，此时配置项的管理就变成了一项繁杂且容易出错的工作。如何应对？集中化管理配置。

Spring Cloud Config Server为所有的微服务提供配置项，支持使用git、svn、数据库和Consul作为存储。配置服务创建好后，我们还需要在微服务的配置文件中指明配置服务地址。

有些配置项，比如数据库用户名和密码，不适合明文存储。我们可以用安全存储工具，比如Vault。SpringCloud可提供了对Vault的支持。

## 创建配置服务
终于到了写代码环节，激动人心。我们要基于Git创建配置服务，就是一个常规的SpringBoot工程。

进入[Spring Initializr](https://start.spring.io/)，artifact栏输入config-server，Dependencies中加入Config Server，创建工程。在application.properties文件中写入对应的git地址，这里以我的github地址为例：

    spring.config.name=config-server
    server.port=8888
    spring.cloud.config.server.git.uri=https://github.com/sumuzhou/microservice-spring-cloud-config.git
    spring.cloud.config.server.git.clone-on-start=true

然后只需在入口的ConfigServerApplication加上@EnableConfigServer注解即可。

## 创建catalog-service微服务
创建第一个实现业务逻辑的微服务Catalog Service，提供产品查询服务。进入[Spring Initializr](https://start.spring.io/)，artifact栏输入catalog-service，Dependencies中加入Reactive Web、JPA、MySQL、Actuator、Lombok、Config Client和Vault Configuration，创建工程。

创建JPA实体Product.java
```java
@Data
@Entity
@Table(name = "products")
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    @Column(nullable = false, unique = true)
    private String code;
    @Column(nullable = false)
    private String name;
    private String description;
    private double price;
    @Transient
    private boolean inStock;
}
```
创建JPA仓库ProductRepository.java
```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    Optional<Product> findByCode(String code);
}
```
创建服务层ProductService.java
```java
@Service
@Transactional
@Slf4j
public class ProductService {
	private final ProductRepository productRepository;

	@Autowired
	public ProductService(ProductRepository productRepository) {
		this.productRepository = productRepository;
	}

	public List<Product> findAllProducts() {
		return productRepository.findAll();
	}

	public Optional<Product> findProductByCode(String code) {
		Optional<Product> productOptional = productRepository.findByCode(code);
        return productOptional;
	}
}
```
创建控制层ProductController.java
```java
@RestController
@RequestMapping("/api/products")
@Slf4j
public class ProductController {
    private final ProductService productService;
    @Autowired
    public ProductController(ProductService productService) {
        this.productService = productService;
    }
    @GetMapping("")
    public List<Product> allProducts() {
    	log.debug("in all products");
        return productService.findAllProducts();
    }
    @GetMapping("/{code}")
    public Product productByCode(@PathVariable String code) {
        return productService.findProductByCode(code)
                .orElseThrow(() -> new ProductNotFoundException("Product with code ["+code+"] doesn't exist"));
    }
}
```
自定义异常类ProductNotFoundException.java
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException() {
    }
    public ProductNotFoundException(String message) {
        super(message);
    }
    public ProductNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
    public ProductNotFoundException(Throwable cause) {
        super(cause);
    }
}
```
@Data注解使用Lombok库自动生成getter、setter方法，@Slf4j注解使用Lombok库自动生成log对象；如何在IDE中支持Lombok请看[这里](https://projectlombok.org/setup/eclipse)。

为方便演示，在数据库中加入一些数据，在src/main/resources中加入data.sql文件：
```sql
DELETE FROM products;

insert into products(id, code, name, description, price) VALUES
(1, 'P001', 'Product 1', 'Product 1 description', 25),
(2, 'P002', 'Product 2', 'Product 2 description', 32),
(3, 'P003', 'Product 3', 'Product 3 description', 50);
```
