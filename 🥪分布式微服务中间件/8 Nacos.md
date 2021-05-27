# Nacos
欢迎来到 Nacos 的世界！

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

## 文档
快速上手: https://nacos.io/zh-cn/docs/quick-start.html
SpringBoot快速开始: https://nacos.io/zh-cn/docs/quick-start-spring-boot.html
版本说明: https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E
Dubbo-Nacos: https://nacos.io/zh-cn/docs/use-nacos-with-dubbo.html

## 快速启动
以MODE="standalone"单即模式启动(以cmd方式启动需要修改脚本)
访问: http://localhost:8848/nacos 
账号密码: nacos/nacos

## SpringBoot快速开始
https://github.com/nacos-group/nacos-examples/blob/master/nacos-spring-boot-example/nacos-spring-boot-discovery-example/pom.xml

### 配置集(Data ID)
在系统中,一个配置文件通常就是一个配置集,一个配置集可以包含了系统的各种配置信息,例如，一个配置集可能包含了数据源、线程池、日志级别等配置项。每个配置集都可以定义一个有意义的名称,就是配置集的ID即Data ID。

### 配置项
配置集中包含的一个个配置内容就是配置项。它代表一个具体的可配置的参数 与其值域,通常以key=value
的形式存在。例如我们常配置系统的日志输出级别( logLevel=INFO | WARN lERROR )就是一个配置项。

### 配置分组(Group)
配置分组是对配置集进行分组,通过一个有意义的字符串(如Buy或Trade )来表示,不同的配置分组下可以有相同的配置集( DataID )。
当您在Nacos.上创建一个配置时 ,如果未填写配置分组的名称,则配置分组的名称默认采用DEFAULT_GROUP。
配置分组的常见场景: 可用于区分不同的项目或应用,例如:学生管理系统的配置集可以定义一个group为: STUDENT_GROUP。

### 命名空间(Namespace)
命名空间( namespace )可用于进行不同环境的配置隔离。例如可以隔离开发环境、测试环境和生产环境。
因为它们的配置可能各不相同,或者是隔离不同的用户,不同的开发人员使用同一个nacos管理各自的配置,可通过namespace隔离。不同的命名空间下,可以存在相同名称的配置分组(Group)或配置集。


### Nacos 保护阈值
当**服务A健康实例数/总实例数 < 保护阈**的时候，说明健康实例不多了，这个时候保护阈值会被触发(状态true)
