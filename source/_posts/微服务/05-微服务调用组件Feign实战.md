---
title: 微服务调用组件Feign实战
date: 2022-05-13 14:41:52
categories: 
- Java后端
tags:
- 微服务
---

### JAVA 项目中如何实现接口调用？

#### 1）Httpclient

HttpClient 是 Apache Jakarta Common 下的子项目，用来提供高效的、最新的、功能丰富的支持 Http 协 议的客户端编程工具包，并且它支持 HTTP 协议最新版本和建议。HttpClient 相比传统 JDK 自带的 URLConnection，提升了易用性和灵活性，使客户端发送 HTTP 请求变得容易，提高了开发的效率。 

#### 2）Okhttp 

一个处理网络请求的开源项目，是安卓端最火的轻量级框架，由 Square 公司贡献，用于替代 HttpUrlConnection 和 Apache HttpClient。OkHttp 拥有简洁的 API、高效的性能，并支持多种协议 （HTTP/2 和 SPDY）。 

#### 3）HttpURLConnection 

HttpURLConnection 是 Java 的标准类，它继承自 URLConnection，可用于向指定网站发送 GET 请求、 POST 请求。HttpURLConnection 使用比较复杂，不像 HttpClient 那样容易使用。

#### 4）RestTemplate 

RestTemplate 是 Spring 提供的用于访问 Rest 服务的客户端，RestTemplate 提供了多种便捷访问远程 HTTP 服务的方法，能够大大提高客户端的编写效率。 上面介绍的是最常见的几种调用接口的方法，我们下面要介绍的方法比上面的更简单、方便，它就是 Feign。

### 1. 什么是Feign

Feign是Netflix开发的声明式、模板化的HTTP客户端，其灵感来自Retrofit、JAXRS-2.0以及WebSocket。 Feign可帮助我们更加便捷、优雅地调用HTTP API。 

Feign支持多种注解，例如Feign自带的注解或者JAX-RS注解等。<span style="color:red">Spring Cloud openfeign对Feign进行了 增强，使其支持Spring MVC注解，另外还整合了Ribbon和Eureka，从而使得Feign的使用更加方便</span>

#### 1.1 优势

Feign可以做到<span style="color:red">使用 HTTP 请求远程服务时就像调用本地方法一样的体验</span>，开发者完全感知不到这是远程方 法，更感知不到这是个 HTTP 请求。它像 Dubbo 一样，consumer 直接调用接口方法调用 provider，而不 需要通过常规的 Http Client 构造请求再解析返回数据。它解决了让开发者调用远程接口就跟调用本地方法 一样，无需关注与远程的交互细节，更无需关注分布式环境开发。

#### 1.2 Feign的设计架构

<img src="https://feng.mynatapp.cc/blog/image-20220513145142603.png" alt="image-20220513145142603" style="zoom:50%;" />

#### 1.3 Ribbon&Feign对比

**Ribbon+RestTemplate进行微服务调用**

```java
@Bean 
@LoadBalanced 
public RestTemplate restTemplate() { 
  return new RestTemplate(); 
}

//调用方式 
String url = "http://mall‐order/order/findOrderByUserId/"+id; 
R result = restTemplate.getForObject(url,R.class);
```

**Feign进行微服务调用**

```java
@FeignClient(value = "mall‐order",path = "/order")
public interface OrderFeignService { 
  @RequestMapping("/findOrderByUserId/{userId}") 
  public R findOrderByUserId(@PathVariable("userId") Integer userId); 
}

@Autowired 
OrderFeignService orderFeignService; 
//feign调用 
R result = orderFeignService.findOrderByUserId(id);
```

#### 1.4 Feign单独使用

引入依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.netflix.feign</groupId>
        <artifactId>feign-core</artifactId>
        <version>8.18.0</version>
    </dependency>

    <dependency>
        <groupId>com.netflix.feign</groupId>
        <artifactId>feign-jackson</artifactId>
        <version>8.18.0</version>
    </dependency>
</dependencies>
```

编写接口

```java
public interface RemoteService {
    @Headers({"Content‐Type: application/json", "Accept: application/json"})
    @RequestLine("GET /order/findOrderByUserId/{userId}")
    List<?> findOrderByUserId(@Param("userId") Integer userId);
}
```

调用

```java
public class FeignDemo {

    public static void main(String[] args) {
        // 基于json
        // encoder指定对象编码方式，decoder指定对象解码方式
        RemoteService service = Feign.builder()
                .encoder(new JacksonEncoder())
                .decoder(new JacksonDecoder())
                .options(new Request.Options(1000, 3500))
                .retryer(new Retryer.Default(5000, 5000, 3))
                .target(RemoteService.class, "http://localhost:8020/");

        System.out.println(service.findOrderByUserId(1));

    }
}
```

### 2. Spring Cloud Alibaba快速整合Feign

1）引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2）编写调用接口+@FeignClient注解

```java
@FeignClient(value = "mall‐order", path = "/order")
public interface OrderFeignService {

    @RequestMapping("/findOrderByUserId/{userId}")
    List<?> findOrderByUserId(@PathVariable("userId") Integer userId);
}
```

3）调用端在启动类上添加`@EnableFeignClients`注解

```java
@SpringBootApplication
@EnableFeignClients
public class FeignLearnDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(FeignLearnDemoApplication.class, args);
    }

}
```

4）发起调用，像调用本地方式一样调用远程服务

```java
@RestController
@RequestMapping("/user")
public class UserController {
    @Autowired
    private OrderFeignService orderFeignService;

    @RequestMapping(value = "/findOrderByUserId/{id}")
    public List<?> findOrderByUserId(@PathVariable("id") Integer id) {
        //feign调用 
        return orderFeignService.findOrderByUserId(id);
    }
}
```

<span style="color:red">提示： Feign 的继承特性可以让服务的接口定义单独抽出来，作为公共的依赖，以方便使用。</span>

### 3. Spring Cloud Feign的自定义配置及使用

Feign 提供了很多的扩展机制，让用户可以更加灵活的使用。

#### 3.1 日志配置

有时候我们遇到 Bug，比如接口调用失败、参数没收到等问题，或者想看看调用性能，就需要配置 Feign 的 日志了，以此让 Feign 把请求信息输出来。

##### 1）定义一个配置类，指定日志级别

```java
/**
 * <p>
 * 注意： 此处配置@Configuration注解就会全局生效，如果想指定对应微服务生效，就不能配置
 * </p>
 *
 * @author pyf 2022-05-13
 */
public class FeignConfig {
    /**
     * 日志级别
     *
     * @return
     */
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

通过源码可以看到日志等级有 4 种，分别是：

- `NONE`【性能最佳，适用于生产】：不记录任何日志（默认值）。 
- `BASIC`【适用于生产环境追踪问题】：仅记录请求方法、URL、响应状态代码以及执行时间。 
- `HEADERS`：记录BASIC级别的基础上，记录请求和响应的header。 
- `FULL`【比较适用于开发及测试环境定位问题】：记录请求和响应的header、body和元数据。

##### 2) 局部配置，让调用的微服务生效，在@FeignClient 注解中指定使用的配置类

<img src="https://feng.mynatapp.cc/blog/image-20220513151512993.png" alt="image-20220513151512993" style="zoom:50%;" />

##### 3) 在yml配置文件中执行 Client 的日志级别才能正常输出日志，格式是"logging.level.feign接口包路径=debug"

```yaml
logging:
  level:
    com.feng.feign.config: debug
```

测试：BASIC级别日志

<img src="https://feng.mynatapp.cc/blog/image-20220513151843037.png" alt="image-20220513151843037" style="zoom:50%;" />

补充：局部配置可以在yml中配置

对应属性配置类：

org.springframework.cloud.openfeign.FeignClientProperties.FeignClientConfiguration

```yaml
feign:
  client:
    config:
      mall‐order:
      #对应微服务
        loggerLevel: FULL
```

#### 3.2 契约配置

Spring Cloud 在 Feign 的基础上做了扩展，可以让 Feign 支持 Spring MVC 的注解来调用。原生的 Feign 是不支持 Spring MVC 注解的，如果你想在 Spring Cloud 中使用原生的注解方式来定义客户端也是 可以的，通过配置契约来改变这个配置，Spring Cloud 中默认的是 SpringMvcContract。

##### 1）修改契约配置，支持Feign原生的注解

```java
/**
 * 修改契约配置，支持Feign原生的注解
 *
 * @return
 */
@Bean
public Contract feignContract() {
    return new Contract.Default();
}
```

<span style="color:red">注意：修改契约配置后，OrderFeignService 不再支持springmvc的注解，需要使用Feign原生的注解</span>

##### 2）OrderFeignService 中配置使用Feign原生的注解

```java
@FeignClient(value = "mall‐order", path = "/order")
public interface OrderFeignService {

  	// 原生注解@RequestLine
    @RequestLine("GET /findOrderByUserId/{userId}")
    List<?> findOrderByUserId(@PathVariable("userId") Integer userId);
}
```

##### 3）补充，也可以通过yml配置契约

```yaml
feign:
  client:
    config:
      mall‐order:
        # 对应微服务
        loggerLevel: FULL
        # 指定Feign原生注解契约配置
        contract: feign.Contract.Default 
```

#### 3.3 通过拦截器实现认证

通常我们调用的接口都是有权限控制的，很多时候可能认证的值是通过参数去传递的，还有就是通过请求头 去传递认证信息，比如 Basic 认证方式。

##### Feign 中我们可以直接配置 Basic 认证

```java
@Configuration // 全局配置 
public class FeignConfig { 
  @Bean 
  public BasicAuthRequestInterceptor basicAuthRequestInterceptor() { 
    return new BasicAuthRequestInterceptor("fox", "123456"); 
  } 
}
```

<span style="color:red">扩展点： feign.RequestInterceptor</span>

每次 feign 发起http调用之前，会去执行拦截器中的逻辑。

```java
public interface RequestInterceptor {
    /**
     * Called for every request. Add data using methods on the supplied {@link RequestTemplate}
     */
    void apply(RequestTemplate template);
}
```

使用场景

1. 统一添加 header 信息；

2. 对 body 中的信息做修改或替换；

##### 自定义拦截器实现认证逻辑

```java
public class FeignAuthRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 业务逻辑
        String accessToken = UUID.randomUUID().toString();
        template.header("Authorization", accessToken);
    }
}

@Configuration // 全局配置 
public class FeignConfig { 
    @Bean 
    public Logger.Level feignLoggerLevel() { 
      	return Logger.Level.FULL; 
    } 

    /** 
    * 自定义拦截器 
    * @return 
    */ 
    @Bean 
    public FeignAuthRequestInterceptor feignAuthRequestInterceptor(){ 
      	return new FeignAuthRequestInterceptor(); 
    } 
}
```

测试

<img src="https://feng.mynatapp.cc/blog/image-20220513153343954.png" alt="image-20220513153343954" style="zoom:50%;" />

补充：可以在yml中配置

```yaml
feign:
  client:
    config:
      # 对应微服务
      mall‐order:
        # 配置拦截器
        requestInterceptors[0]:  com.feng.feign.interceptor.FeignAuthRequestInterceptor
```

mall-order端可以通过 @RequestHeader获取请求参数

#### 3.4 超时时间配置

通过 Options 可以配置连接超时时间和读取超时时间，Options 的第一个参数是连接的超时时间（ms）， 默认值是 2s；第二个是请求处理的超时时间（ms），默认值是 5s。

##### 全局配置

```java
@Bean
public Request.Options options() {
  	return new Request.Options(5000, 10000);
}
```

##### yml中配置

```yaml
feign:
  client:
    config:
      # 对应微服务
      mall‐order:
        # 连接超时时间，默认2s
        connectTimeout: 5000
        # 请求处理超时时间，默认5s
        readTimeout: 10000
```

<span style="color:red">补充说明： Feign的底层用的是Ribbon，但超时时间以Feign配置为准</span>

测试超时情况：

<img src="https://feng.mynatapp.cc/blog/image-20220513154028051.png" alt="image-20220513154028051" style="zoom:50%;" />

返回结果

<img src="https://feng.mynatapp.cc/blog/image-20220513154048230.png" alt="image-20220513154048230" style="zoom:50%;" />

#### 3.5 客户端组件配置

Feign 中默认使用 JDK 原生的 URLConnection 发送 HTTP 请求，我们可以集成别的组件来替换掉 URLConnection，比如 Apache HttpClient，OkHttp。

Feign发起调用真正执行逻辑：<span style="color:red">feign.Client#execute （扩展点）</span>

<img src="https://feng.mynatapp.cc/blog/image-20220513154153284.png" alt="image-20220513154153284" style="zoom:50%;" />

##### 3.5.1 配置Apache HttpClient

引入依赖

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.7</version>
</dependency>

<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>10.1.0</version>
</dependency>
```

然后修改yml配置，将 Feign 的 Apache HttpClient启用 ：

```yaml
feign:
  #feign 使用 Apache HttpClient 可以忽略，默认开启
  httpclient:
    enabled: true
```

关于配置可参考源码： <span style="color:red">org.springframework.cloud.openfeign.FeignAutoConfiguration</span>

<img src="https://feng.mynatapp.cc/blog/image-20220513154521452.png" alt="image-20220513154521452" style="zoom:50%;" />

测试：调用会进入feign.httpclient.ApacheHttpClient#execute

##### 3.5.2 配置 OkHttp

引入依赖

```xml
<dependency>
  	<groupId>io.github.openfeign</groupId>
  	<artifactId>feign-okhttp</artifactId>
</dependency>
```

然后修改yml配置，将 Feign 的 HttpClient 禁用，启用 OkHttp，配置如下：

```yaml
feign:
  #feign 使用 okhttp 
  httpclient: 
    enabled: false 
  okhttp:
    enabled: true
```

关于配置可参考源码： <span style="color:red">org.springframework.cloud.openfeign.FeignAutoConfiguration</span>

<img src="https://feng.mynatapp.cc/blog/image-20220513154846113.png" alt="image-20220513154846113" style="zoom:50%;" />

测试：调用会进入feign.okhttp.OkHttpClient#execute

#### 3.6 GZIP 压缩配置

开启压缩可以有效节约网络资源，提升接口性能，我们可以配置 GZIP 来压缩数据：

```yaml
feign:
  # 配置 GZIP 来压缩数据
  compression:
    request:
      enabled: true 
      # 配置压缩的类型 
      mime‐types: text/xml,application/xml,application/json 
      # 最小压缩值 
      min‐request‐size: 2048 
      response:
        enabled: true
```

<span style="color:red">注意：只有当 Feign 的 Http Client 不是 okhttp3 的时候，压缩才会生效，配置源码在 FeignAcceptGzipEncodingAutoConfiguration</span>

<img src="https://feng.mynatapp.cc/blog/image-20220513155113195.png" alt="image-20220513155113195" style="zoom:50%;" />

核心代码就是 @ConditionalOnMissingBean（type="okhttp3.OkHttpClient"），表示 Spring BeanFactory 中不包含指定的 bean 时条件匹配，也就是没有启用 okhttp3 时才会进行压缩配置。

#### 3.7 编码器解码器配置

Feign 中提供了自定义的编码解码器设置，同时也提供了多种编码器的实现，比如 Gson、Jaxb、Jackson。 我们可以用不同的编码解码器来处理数据的传输。如果你想传输 XML 格式的数据，可以自定义 XML 编码解 码器来实现获取使用官方提供的 Jaxb。

<span style="color:red">扩展点：Encoder & Decoder</span>

```java
public interface Decoder {
    Object decode(Response response, Type type) throws IOException, DecodeException, FeignException;
}

public interface Encoder {
    void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException;
}
```

#### Java配置方式

配置编码解码器只需要在 Feign 的配置类中注册 Decoder 和 Encoder 这两个类即可:

```java
    @Bean
    public Decoder decoder() {
        return new JacksonDecoder();
    }

    @Bean
    public Encoder encoder() {
        return new JacksonEncoder();
    }
```

#### yml配置方式

```yaml
feign:
  client:
    config:
      # 对应微服务
      mall‐order:
        # 配置编解码器
        encoder: feign.jackson.JacksonEncoder
        decoder: feign.jackson.JacksonDecoder
```

