---
title: 微服务调用组件Dubbo实战
date: 2022-05-13 16:02:06
categories: 
- Java后端
tags:
- 微服务
---

### 1. Spring Cloud整合Dubbo

#### 1. provider端配置

引入依赖

```yaml
<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-dubbo</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>
```

application.yml

```yaml
sever:
  port: 8001
dubbo:
  scan:
    # 指定 Dubbo 服务实现类的扫描基准包
    base-packages: com.feng.dubbo.provider.service
  #  application:
  #    name: ${spring.application.name}
  protocol:
    # dubbo 协议
    name: dubbo
    # dubbo 协议端口（ -1 表示自增端口，从 20880 开始）
    port: -1
#  registry:
#    #挂载到 Spring Cloud 注册中心  高版本可选
#    address: spring-cloud://127.0.0.1:8848

spring:
  application:
    name: spring-cloud-dubbo-provider-user
  main:
    # Spring Boot2.1及更高的版本需要设定
    allow-bean-definition-overriding: true
  cloud:
    nacos:
      # Nacos 服务发现与注册配置
      discovery:
        server-addr: 127.0.0.1:8848
```

服务实现类上配置`@DubboService`暴露服务

```java
@DubboService
public class UserServiceImpl implements UserService {

    @Override
    public List<String> list() {
        return Arrays.asList("111", "222");
    }

    @Override
    public Integer getById(Integer id) {
        return id;
    }
}
```

#### 2. consumer端配置

引入依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.feng</groupId>
        <artifactId>dubbo-provider-demo</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-dubbo</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>
```

application.yml

```yaml
dubbo:
  scan:
    # 指定需要订阅的服务提供方，默认值*，会订阅所有服务，不建议使用
    subscribed‐services: spring-cloud-dubbo-provider-user
  #  application:
  #    name: ${spring.application.name}
  protocol:
    # dubbo 协议
    name: dubbo
    # dubbo 协议端口（ -1 表示自增端口，从 20880 开始）
    port: -1
#  registry:
#    #挂载到 Spring Cloud 注册中心  高版本可选
#    address: spring-cloud://127.0.0.1:8848

spring:
  application:
    name: spring-cloud-dubbo-consumer-user
  main:
    # Spring Boot2.1及更高的版本需要设定
    allow-bean-definition-overriding: true
  cloud:
    nacos:
      # Nacos 服务发现与注册配置
      discovery:
        server-addr: 127.0.0.1:8848
```

当应用使用属性dubbo.cloud.subscribed­services为默认值时，日志中将会输出 警告：

<img src="https://feng.mynatapp.cc/blog/image-20220513165825482.png" alt="image-20220513165825482" style="zoom:50%;" />

服务消费方通过`@DubboReference`引入服务

```java
@RestController
@RequestMapping("/user")
public class UserConstroller {

    @DubboReference
    private UserService userService;

    @RequestMapping("/info/{id}")
    public Integer info(@PathVariable("id") Integer id) {
        return userService.getById(id);
    }

    @RequestMapping("/list")
    public List<String> list() {
        return userService.list();
    }
}
```

#### 3. 从Open Feign迁移到Dubbo

Dubbo Spring Cloud 提供了方案，即 `@DubboTransported` 注解，支持在类，方法，属性上使用。能够帮助服务消费端的 Spring Cloud Open Feign 接口以 及 @LoadBalanced RestTemplate Bean 底层走 Dubbo 调用（可切换 Dubbo 支持的协 议），而服务提供方则只需在原有 @RestController 类上追加 Dubbo @Servce 注解（需要 抽取接口）即可，换言之，在不调整 Feign 接口以及 RestTemplate URL 的前提下，实现无缝迁移。 

修改服务提供者

1）添加`@DubboService`注解

2）服务端引入依赖

```xml
<dependency> 
    <groupId>org.springframework.cloud</groupId> 
    <artifactId>spring‐cloud‐starter‐openfeign</artifactId> 
</dependency>
```

3）feign的实现，Application启动类上添加@EnableFeignClients

4）feign接口添加 `@DubboTransported` 注解

```java
@FeignClient(value = "spring‐cloud‐dubbo‐provider‐user‐feign",path = "/user") @DubboTransported(protocol = "dubbo")
public interface UserDubboFeignService {
```

5）调用对象添加 `@DubboTransported` 注解

```java
@Autowired 
@DubboTransported 
private UserFeignService userFeignService;

@Bean 
@LoadBalanced 
@DubboTransported 
public RestTemplate restTemplate() { 
  return new RestTemplate(); 
}
```

