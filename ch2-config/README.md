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
```
spring.config.name=config-server
server.port=8888
spring.cloud.config.server.git.uri=https://github.com/sumuzhou/microservice-spring-cloud-config.git
spring.cloud.config.server.git.clone-on-start=true
```
然后只需在入口的ConfigServerApplication加上@EnableConfigServer注解即可
```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServerApplication.class, args);
	}

}
```

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

为方便演示，在数据库中加入一些数据，在src/main/resources中加入data.sql文件，程序启动时JPA会自动加载此文件：
```sql
DELETE FROM products;

insert into products(id, code, name, description, price) VALUES
(1, 'P001', 'Product 1', 'Product 1 description', 25),
(2, 'P002', 'Product 2', 'Product 2 description', 32),
(3, 'P003', 'Product 3', 'Product 3 description', 50);
```
接下来到了重点部分，如何在catalog-service中使用配置服务。常规的SpringBoot项目使用application.properties文件进行配置，在这里我们改成bootstrap.properties文件，内容如下：
```
spring.application.name=catalog-service
server.port=8181

management.endpoints.web.exposure.include=*

spring.cloud.config.uri=http://localhost:8888
```
请注意上述的spring.application.name，默认机制下我们的应用会到配置服务中去取同样名称的properties文件，所以确保两者一致。

然后我们只需将catalog-service.properties文件上传至配置服务的git目录中，内容如下：
```
logging.level.com.example=debug

spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/catalog?useSSL=false
#spring.datasource.username=root
#spring.datasource.password=222222

spring.datasource.initialization-mode=always
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```
catalog-service使用本地MySQL数据库，可以通过Docker提供，使用的数据库是catalog库，需要提前创建，注意用户名和密码匹配；JPA设置的模式是update，会自动创建表。注意username和password注释掉了，在后面的Vault章节会讲到如何保存使用此类敏感数据。

*创建SpringCloud应用一般使用bootstrap进行配置，但并不代表application不能并存；事实上，bootstrap创建的Context是application创建的Context的父类，意味着：1. 先读取bootstrap再读取application；2. application会覆盖bootstrap中同名内容。详细说明可参考[SpringCloud Reference](https://cloud.spring.io/spring-cloud-static/Greenwich.RELEASE/multi/multi__spring_cloud_context_application_context_services.html)。实际开发中，在本地保留一份application的配置可简化开发流程，避免对配置服务的依赖。*

# 使用Vault保存敏感数据
Vault是一个用于保存/访问敏感数据的工具，详细的说明请参考[官网](https://learn.hashicorp.com/vault/#getting-started)。这里我们使用Vault的dev模式简单说明如何在项目中使用。

通过Docker可快速启动Vault服务：
```sh
docker run --cap-add=IPC_LOCK -d -p8200:8200 --name=vault vault:1.1.0
```
别急，还没有弄完。执行docker logs vault，会看到类似如下的输出：
```
You may need to set the following environment variable:

    $ export VAULT_ADDR='http://0.0.0.0:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: AMTCPEhG8IxyCJYgjXphbHHMuopc/0NOR6Zks9YjPeI=
Root Token: s.cIwe6aLokLJrNLn1qj0MvXWb
```
通过docker exec -it vault sh进入到容器中，执行如下命令：
```sh
export VAULT_ADDR='http://0.0.0.0:8200'
echo 's.lQSXNLRbbZbDQ0qwQpA9qbNc' > ~/.vault-token
vault kv put secret/catalog-service spring.datasource.username=root spring.datasource.password=222222
```
上述的echo将生成的Root Token写入到~/.vault-token文件中，否则会报missing client token的错误，详见[issues-657](https://github.com/hashicorp/vault/issues/657#issuecomment-237364100)；然后通过kv put命令将对应的配置项写入到Vault中，写入的路径以secret/开头，后面的名称和spring.application.name对应。

最后修改catalog-service的bootstrap.properties，加入如下配置：
```
spring.cloud.vault.host=0.0.0.0
spring.cloud.vault.port=8200
spring.cloud.vault.scheme=http
spring.cloud.vault.kv.enabled=true
spring.cloud.vault.authentication=token
spring.cloud.vault.token=s.cIwe6aLokLJrNLn1qj0MvXWb
```
其中spring.cloud.vault.token就是Vault生成的Root Token。

## 总结
依次启动config-server和catalog-service，访问[http://localhost:8181/api/products](http://localhost:8181/api/products)，即可看到产品信息。
