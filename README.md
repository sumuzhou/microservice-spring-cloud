# microservice-spring-cloud
[![996.icu](https://img.shields.io/badge/link-996.icu-red.svg)](https://996.icu)
[![LICENSE](https://img.shields.io/badge/license-Anti%20996-blue.svg)](https://github.com/996icu/996.ICU/blob/master/LICENSE)
基于Spring Cloud构建微服务的简单教程

#### 难得闲暇，写写文章

## 写在前面
本文绝大部分材料来源于[SivaLabs的文章](https://sivalabs.in/2018/03/microservices-using-springboot-spring-cloud-part-1-overview/)，感谢作者的分享，主要在下述方面有所区别：
- 翻译为中文，并对内容做了少许调整
- 可能原文时间较早，提供的工程并不能部署起来，按照文章逻辑重新创建了工程
- 针对创建工程中遇到的问题，增加了一些延伸内容

有鉴于此，在这里特别说明版本：
- Spring Boot: 2.1.3
- Spring Cloud: Greenwich.SR1
- Vault: 1.1.0
- Zipkin: 2.12.6
- MySQL: 5.7

## 环境准备
运行工程需要Vault、Zipkin、MySQL和Git服务，Git服务可由Github提供，其余可由Docker创建
#### TODO：提供一个docker-compose.yml文件，启动所有服务

## 写在后面
微服务不是包治百病的灵丹妙药，在采用微服务架构之前考虑清楚下述问题：
- 我的团队规模/能力是否足以支撑微服务架构，特别是运维团队
- 我的整个系统是否有足够的理由要拆分成独立服务，比如独立的迭代周期、明确的责任边界、差异较大的伸缩能力要求
- 我想要的系统是AP还是CP的
