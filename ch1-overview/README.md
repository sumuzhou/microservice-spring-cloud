# MicroServices using Spring Boot & Spring Cloud – Part 1 : Overview
本章主要内容包括：
- 单体应用（Monoliths，随手翻译）
- 微服务是什么
- 微服务的优势
- 微服务的挑战
- 为何使用SpringBoot & SpringCloud
- 工程介绍

## 单体应用
首先需要说明的是，单体应用是一种和微服务相对的部署方式，两者并没有褒贬之分，请注意。

过去我们开发企业应用的时候，仍然使用了模块化、DDD、TDD等方法，只是在最终部署时，整合成为了一个部署单元（比如EAR包或WAR包），这种应用称为单体应用。

单体应用存在一些问题：
- 代码不可避免的混乱（人超多，人总是没有机器可爱，对吧）
- 无法仅扩展应用的部分功能，只能整体扩展
- 升级和重构要搞死人，随便来个简单feature都会要几个团队讨论半天

原作者在这里申明，我也赞同，就是部署单体应用实际上是比部署微服务容易的。

## 微服务
和单体应用相反，微服务就是将系统拆成多个可独立部署的子系统，最终协同部署形成应用。用户能够感知到的只是最终形成的整体应用。拆分的方法一般都是DDD，基于业务领域。在微服务语境下，一般将拆分出来的各独立模块称为服务。
> 想了解更多，请移步至大名鼎鼎的Martin Fowler的文章
https://martinfowler.com/articles/microservices.html

## 微服务的优势
- 单个服务代码量相对较少（毕竟不是三哥）
- 可独立扩展部分的功能
- 拆分团队，使团队更加专注
- 升级和重构都在团队内部进行，相对容易

## 微服务的挑战
- 从一开始就拆分是很容易出问题的，个人建议是在开发过程中逐渐拆分
- 开发分布式应用对程序员的要求比较高，开发时要考虑限流/熔断之类的问题，出错时debug相对复杂；所以更烧钱咯
- 微服务的运维是一项巨大的挑战，DevOps了解下；所以更烧钱咯
- 搭建环境变得非常复杂，有时严重影响了本地测试效率，还好有Docker

## 为何使用SpringBoot & SpringCloud
*尊重原著，进行翻译，并不完全赞同作者的观点。SpringCloud的技术栈限定在Java生态圈，并且对开发有一定的干涉。依个人拙见，未来的开发倾向于多样化，SpringCloud只是构建微服务方式之一，基于Kubernetes来搭建微服务架构会更适合。*

SpringBoot的思路是通过简化配置，让开发人员集中精力在业务逻辑上，从而提升开发效率。犹记得当年学Spring框架，一定要从写web.xml开始，将HelloWorld跑起来都费事。SpringBoot的出现让开发单个应用变得非常简单，新手程序员也能掌握。

SpringCloud是什么？另一个框架？事实上SpringCloud是很多模块的合称，这些模块定义了我们在开发Cloud Native应用时应该遵守的设计模式。和SpringBoot一样，SpringCloud旨在减少架构带来的问题，让开发人员集中精力在业务逻辑上。至于Cloud Native，在这里不展开讲，需要有一个单独的主题版块。

下面列举一些SpringCloud中常用的模块：
- Config Server：用一个中心化的配置服务来管理微服务的配置项，保证一致性
- Service Registry & Discovery：服务的注册与发现机制；微服务的运行结构随时间会发生变化，需要自动化的机制来应对
- Circuit Breaker：断路开关；微服务架构中服务相互依赖，单个服务down掉可能引发级联反应，这个时候就需要断路开关来保护
- Distributed Tracing：微服务中一个调用可能引发一连串的调用，需要借助分布式追踪来协助Debug

除此之外，还有Spring Cloud Data Streams、Spring Cloud Security、Spring Cloud Contract等，详细的列表请访问官网。

## 工程介绍
**实践出真知**
通过一个示例工程，介绍SpringBoot和SpringCloud的用法。工程中的业务逻辑刻意做的很简单，让大家关注点放在SpringBoot和SpringCloud的特性上。工程包含以下微服务：
- catalog-service：提供产品查询服务
- config-server：SpringCloud的配置服务
- hystrix-dashboard：查看Hystrix命令执行情况
- inventory-service：提供库存查询服务
- proxy-gateway：SpringCloud的网关服务
- service-registry：SpringCloud的服务注册
