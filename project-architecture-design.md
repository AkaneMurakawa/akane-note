# 项目架构设计

## 前言

分布式需面临的问题

1. 服务通信
2. 服务管理
3. 服务挂了怎么办，熔断机制

### 降级、限流、熔断

限流：限制访问的流量，顶不住就挡一部分出去但是不能说不行

降级：当OrderProvider服务挂掉后，返回默认值。 实现部分可用，服务降级

熔断：至少不要影响别的系统

隔离：你本身就独立的，但是你会调用其他的系统，你快不行了就别拖累兄弟们了

## 技术架构

前端：前后端分离

后端：Java、Spring、SpringBoot

NoSQL：Redis集群、哨兵，主从同步、读写分离；MongoDB；ElasticSearch；

分布式锁：Redis、Zookeeper 、MySQL悲观锁for update

分布式事务：SEATA（[常用的分布式事务解决方案](https://juejin.im/post/5aa3c7736fb9a028bb189bca#heading-0)）

分布式任务调度平台：XXL-JOB

分布式文件系统：FastDFS

SQL审核平台：Archery

数据库：MySQL、Druid

ORM：Mybatis，Mybatis-Plus、tkMybatis

缓存：Memercache、Redis、Guava-Cache、阿里Jetcache

开源工具：Druid、Lombok、Goolge Guava、Hutool、Dinger、Justauth、Hibernator-Validator、Mapstruct、FastJson

依赖管理：Maven、Nexus

版本控制：Git、GitLab

文档：YApi、Swagger

容器化：Docker、Portatiner

运维发布平台：SPUG

网关\(Gateway\)：Spring Cloud Gateway\(代替Zuul\)、Spring Cloud Zuul 路由网关 路由 代理 过滤。\(SLB：Server loadBalance\)

HA 高可用：Nginx、lvs

客户端负载均衡的工具：Ribbon，@LoadBalanced

注册中心：Alibaba Nacos、Eureka、Consul

消息中间件：RocketMQ、RabbitMQ

远程调用：Dubbo、Feign

熔断降级：Hystrix

流量控制、降级：Alibaba Sentinel

### 分布式框架

* Spring Cloud NetFlix 一站式解决方案（Eureka、Ribbon、Feign、Hystrix、Zuul）

api网关：zuul组件

Feign：HttpClient，Http通信方式，同步，阻塞

服务注册发现：Eureka

熔断机制： Hystrix

* Apache Dubbo zookeeper 半自动，需要整合别人的

API网关： 没有，找第三方组件

Dubbo：RPC

Zookeeper

熔断机制：没有，借助Hystrix

* Spring Cloud Alibaba --一站式解决方案 更简单

新概念： 服务网格 server Mesh

istio

## 权限设计

单点登录、权限设计 Shiro + CAS、JWT

## REST风格

OData：开放数据协议（Open Data Protocol，简称OData）是由[微软](https://baike.baidu.com/item/微软)于2007年发起的开放协议 \[1\] 。当时的开发代号叫“Project Astoria”。于2009年更名为Open Data Protocol \[3\] 。

## API版本

API版本命名风格：/api/v1/test

## 服务设计

健康检查

服务监控

消息幂等、超时重试设计、业务幂等、消息id幂等

并发处理、分布式锁

日志记录

设计模式

数据一致性

业务一致性-分布式事务处理

数据传输与路由

## 微服务器化

全球唯一ID

分布式锁

幂等性

服务拆分、服务治理、服务降级、熔断等等

DBUnit

状态机

搜索

分布式session

日志聚合、调用链

服务化框架

容器化

## 书籍

1.《Redis开发与运维》豆瓣评分8.9 作者: 付磊 / 张益军

2.《高性能MySQL》第三版 豆瓣评分9.3 作者: 施瓦茨 \(Baron Schwartz\) / 扎伊采夫 \(Peter Zaitsev\) / 特卡琴科 \(Vadim Tkachenko\)

3.《Spring Cloud微服务：入门、实战与进阶》豆瓣评分7.9 作者: 尹吉欢

