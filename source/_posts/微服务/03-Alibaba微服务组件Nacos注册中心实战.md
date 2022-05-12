---
title: Alibaba微服务组件Nacos注册中心实战
date: 2022-05-11 10:58:24
categories: 
- Java后端
tags:
- 微服务
---

### 1. 什么是 Nacos

官方文档： https://nacos.io/zh-cn/docs/what-is-nacos.html Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态 服务发现、服务配置、服务元数据及流量管理。 

Nacos 的关键特性包括:

- 服务发现和服务健康监测 
- 动态配置服务 
- 动态 DNS 服务 
- 服务及其元数据管理

#### 1.1 Nacos 架构

![nacos_arch.jpg](https://feng.mynatapp.cc/blog/1561217892717-1418fb9b-7faa-4324-87b9-f1740329f564.jpeg)

NamingService：命名服务，注册中心核心接口 

ConfigService：配置服务，配置中心核心接口 

OpenAPI文档：https://nacos.io/zh-cn/docs/open-api.html

nacos版本： v1.1.4 升级到v1.4.1

#### 1.2 Nacos Server部署

下载源码编译

源码下载地址：https://github.com/alibaba/nacos/

```tex
cd nacos/ 
# 编译1.4.1 增加 ‐Drat.skip=true 参数 ，跳过licensing 检查
mvn -Prelease-nacos -Dmaven.test.skip=true clean install -U
cd nacos/distribution/target/
```

下载安装包

下载地址：https://github.com/alibaba/Nacos/releases

##### 1.2.1 单机模式

官方文档： https://nacos.io/zh-cn/docs/deployment.html 

解压，进入nacos目录

<img src="https://feng.mynatapp.cc/blog/image-20220511140404789.png" alt="image-20220511140404789" style="zoom:50%;"/>

单机启动nacos，执行命令

```tex
sh bin/startup.sh -m standalone
```

也可以修改默认启动方式

<img src="https://feng.mynatapp.cc/blog/image-20220511145533167.png" alt="image-20220511145533167" style="zoom:50%;"/>

访问nocas的管理端：http://192.168.3.14:8848/nacos ，默认的用户名密码是 nocas/nocas

##### 1.2.2 集群模式

官网文档： https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html

集群部署架构图

<img src="https://feng.mynatapp.cc/blog/image-20220511150219719.png" alt="image-20220511150219719" style="zoom:50%;"/>

1）单机搭建伪集群，复制nacos安装包，修改为nacos8850，nacos8854，nacos8856

<img src="https://feng.mynatapp.cc/blog/image-20220511162435600.png" alt="image-20220511162435600" style="zoom:50%;"/>

2）以nacos8850为例，进入nacos8850目录

2.1）修改conf\application.properties的配置，使用外置数据源

```properties
spring.datasource.platform=mysql

### Count of DB:
db.num=1

### Connect URL of DB:
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=root
db.password=123456
```

2.2）将conf\cluster.conf.example改为cluster.conf,添加节点配置

```tex
# ip:port
192.168.10.110:8850
192.168.10.110:8854
192.168.10.110:8856
```

3）创建mysql数据库,sql文件位置：conf\nacos­-mysql.sql

4）修改启动脚本（bin\startup.sh）的jvm参数

![image-20220511162929569](https://feng.mynatapp.cc/blog/image-20220511162929569.png)

nacos8854，nacos8856 按同样的方式配置。

5）修改端口号，分别启动nacos8850，nacos8854，nacos8856

以nacos8850为例，进入nacos8850目录，启动nacos

```tex
sh bin/startup.sh
```

![image-20220511163145895](https://feng.mynatapp.cc/blog/image-20220511163145895.png)

```tex
# 查看输出日志
tail -f ./logs/start.out
```

6) 测试

登录 http://localhost:8850/nacos ，用户名和密码都是nacos

![image-20220511163413406](https://feng.mynatapp.cc/blog/image-20220511163413406.png)

7）官方推荐，nginx反向代理

![image-20220511163522112](https://feng.mynatapp.cc/blog/image-20220511163522112.png)

访问： http://localhost:8847/nacos

#### 1.3 prometheus+grafana监控Nacos

https://nacos.io/zh-cn/docs/monitor-guide.html

Nacos 0.8.0版本完善了监控系统，支持通过暴露metrics数据接入第三方监控系统监控Nacos运 行状态。

1. nacos暴露metrics数据

```properties
management.endpoints.web.exposure.include=*
```

测试：http://localhost:8848/nacos/actuator/prometheus

![image-20220511164629102](https://feng.mynatapp.cc/blog/image-20220511164629102.png)

2. prometheus采集Nacos metrics数据

3. grafana展示metrics数据

### 2. Nacos注册中心

#### 2.1 注册中心演变及其设计思想

![image-20220511170508079](https://feng.mynatapp.cc/blog/image-20220511170508079.png)

![image-20220511170534111](https://feng.mynatapp.cc/blog/image-20220511170534111.png)

#### 2.2 Nacos注册中心架构

![image-20220511170604520](https://feng.mynatapp.cc/blog/image-20220511170604520.png)

#### 2.3 核心功能

`服务注册`：Nacos Client会通过发送REST请求的方式向Nacos Server注册自己的服务，提供自 身的元数据，比如ip地址、端口等信息。Nacos Server接收到注册请求后，就会把这些元数据信 息存储在一个双层的内存Map中。 

`服务心跳`：在服务注册后，Nacos Client会维护一个定时心跳来持续通知Nacos Server，说明服 务一直处于可用状态，防止被剔除。默认5s发送一次心跳。 

`服务同步`：Nacos Server集群之间会互相同步服务实例，用来保证服务信息的一致性。 

`服务发现`：服务消费者（Nacos Client）在调用服务提供者的服务时，会发送一个REST请求给 Nacos Server，获取上面注册的服务清单，并且缓存在Nacos Client本地，同时会在Nacos Client本地开启一个定时任务定时拉取服务端最新的注册表信息更新到本地缓存 

`服务健康检查`：Nacos Server会开启一个定时任务用来检查注册服务实例的健康情况，对于超过 15s没有收到客户端心跳的实例会将它的healthy属性置为false(客户端服务发现时不会发现)，如 果某个实例超过30秒没有收到心跳，直接剔除该实例(被剔除的实例如果恢复发送心跳则会重新 注册)

#### 2.4 服务注册表结构

![image-20220511170802268](https://feng.mynatapp.cc/blog/image-20220511170802268.png)

#### 2.5 服务领域模型

![image-20220511170832255](https://feng.mynatapp.cc/blog/image-20220511170832255.png)

#### 2.6 服务实例数据

![image-20220511170916488](https://feng.mynatapp.cc/blog/image-20220511170916488.png)

### 3. Spring Cloud Alibaba Nacos快速开始

#### 3.1 Spring Cloud Alibaba版本选型

**组件版本关系**

![image-20220511171027442](https://feng.mynatapp.cc/blog/image-20220511171027442.png)

**毕业版本依赖关系（推荐使用）**

![image-20220511171137516](https://feng.mynatapp.cc/blog/image-20220511171137516.png)

#### 3.2 搭建Nacos­client服务

1）引入依赖

父Pom中支持spring cloud&spring cloud alibaba, 引入依赖

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>Hoxton.SR8</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-alibaba-dependencies</artifactId>
      <version>2.2.4.RELEASE</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

当前项目pom中引入依赖

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

2）application.properties中配置

```properties
server.port=8002
# 微服务名称
spring.application.name=nacos-service-test
#配置 Nacos server 的地址
spring.cloud.nacos.discovery.server‐addr=localhost:8848
```

更多配置：https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-discovery

3）启动springboot应用，nacos管理端界面查看是否成功注册

