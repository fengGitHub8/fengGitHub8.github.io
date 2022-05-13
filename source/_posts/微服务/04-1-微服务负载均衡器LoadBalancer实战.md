---
title: 微服务负载均衡器LoadBalancer实战
date: 2022-05-13 10:34:57
categories: 
- Java后端
tags:
- 微服务
---

### 1. 什么是Spring Cloud LoadBalancer

Spring Cloud LoadBalancer是Spring Cloud官方自己提供的客户端负载均衡器, 用来替代 Ribbon。

Spring官方提供了两种负载均衡的客户端：

`RestTemplate`

RestTemplate是Spring提供的用于访问Rest服务的客户端，RestTemplate提供了多种便捷访问 远程Http服务的方法，能够大大提高客户端的编写效率。默认情况下，RestTemplate默认依赖 jdk的HTTP连接工具。

`WebClient`

WebClient是从Spring WebFlux 5.0版本开始提供的一个非阻塞的基于响应式编程的进行Http请 求的客户端工具。它的响应式编程的基于Reactor的。WebClient中提供了标准Http请求方式对 应的get、post、put、delete等方法，可以用来发起相应的请求。

### 2. RestTemplate整合LoadBalancer

**1）引入依赖**

```tex
<dependencies>
  <!-- loadbalancer -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
  </dependency>

  <!--提供了RestTemplate支持-->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!--nacos服务注册与发现 移除ribbon支持-->
  <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <exclusions>
      <exclusion>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
      </exclusion>
    </exclusions>
  </dependency>
</dependencies>
```

<span style="color:red">注意： nacos-discovery中引入了ribbon，需要移除ribbon的包 如果不移除，也可以在yml中配置不使用ribbon</span>

原理：默认情况下，如果同时拥有RibbonLoadBalancerClient和 BlockingLoadBalancerClient，为了保持向后兼容性，将使用RibbonLoadBalancerClient。要 覆盖它，可以设置spring.cloud.loadbalancer.ribbon.enabled属性为false。

<img src="https://feng.mynatapp.cc/blog/image-20220513105713997.png" alt="image-20220513105713997" style="zoom:50%;" />

2）使用@LoadBalanced注解配置RestTemplate

```java
@Configuration
public class RestConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

3）使用

```java
@RestController
public class UserController {

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping(value = "/findOrderByUserId/{id}")
    public List<?> findOrderByUserId(@PathVariable("id") Integer id) {
        String url = "http://mall‐order/order/findOrderByUserId/" + id;
        List<?> result = restTemplate.getForObject(url, List.class);
        return result;
    }
}
```

### 3. WebClient整合LoadBalancer

1）引入依赖

```tex
<dependencies>
  <!-- loadbalancer -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
  </dependency>

  <!--webflux-->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
  </dependency>

  <!--提供了RestTemplate支持-->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!--nacos服务注册与发现 移除ribbon支持-->
  <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <exclusions>
      <exclusion>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
      </exclusion>
    </exclusions>
  </dependency>
</dependencies>
```

2）配置WebClient作为负载均衡器的client

```java
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }

    @Bean
    public WebClient webClient() {
        return webClientBuilder().build();
    }
}
```

3）使用

```java
@Autowired
private WebClient webClient;

@RequestMapping(value = "/findOrderByUserId2/{id}")
public Mono<?> findOrderByUserIdWithWebClient(@PathVariable("id") Integer id) {
		String url = "http://mall‐order/order/findOrderByUserId/" + id;
  	//基于WebClient
  	return webClient.get().uri(url).retrieve().bodyToMono(List.class);
}
```

原理： 底层会使用ReactiveLoadBalancer

<img src="https://feng.mynatapp.cc/blog/image-20220513141912797.png" alt="image-20220513141912797" style="zoom:50%;" />

引入webFlux

```java
@Autowired
private ReactorLoadBalancerExchangeFilterFunction lbFunction;

@RequestMapping(value = "/findOrderByUserId3/{id}")
public Mono<?> findOrderByUserIdWithWebFlux(@PathVariable("id") Integer id) {
		String url = "http://mall‐order/order/findOrderByUserId/" + id;
		//基于WebClient+webFlux
  	return WebClient.builder().filter(lbFunction).build().get().uri(url).retrieve().bodyToMono(List.class);
}
```

