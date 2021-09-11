# SpringCloud 微服务笔记



## nacos服务注册

有不清楚的可以去查官方文档

1. 添加pom依赖

   ```java
   <dependency>
       <groupId>com.alibaba.cloud</groupId>
       <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
       <version>${latest.version}</version>
   </dependency>
   ```

   2.yml配置

   ```java
   server:
     port: 9001
   
   spring:
     application:
       name: nacos-payment-provider
     cloud:
       nacos:
         discovery:
           server-addr: 10.221.11.133:8848
   
   management:
     endpoints:
       web:
         exposure:
           include: '*'
   ```

   

3. 通过@EnableDiscoveryClient开启服务注册发现功能

   ```java
   @EnableDiscoveryClient
   @SpringBootApplication
   public class PaymentMain9001
   {
       public static void main(String[] args) {
               SpringApplication.run(PaymentMain9001.class, args);
       }
   }
   
   ```

   

注意给`RestTemplate`实例添加 `@LoadBalanced` 注解，开启 `@LoadBalanced` 与 [Ribbon](https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-ribbon.html) 的集成：

```java
   @LoadBalanced
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
```

## nacos配置中心

1. 添加依赖

   

   ```java
   <dependency>
       <groupId>com.alibaba.cloud</groupId>
       <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
       <version>${latest.version}</version>
   </dependency>
   ```

   2. 在 `bootstrap.properties` 中配置 Nacos server 的地址和应用名，bootstrap优先级高于application

   3. 然后如nacos网页上配置，在 Nacos Spring Cloud 中，`dataId` 的完整格式如下：

      ```plain
      ${prefix}-${spring.profiles.active}.${file-extension}
      ```

      - `prefix` 默认为 `spring.application.name` 的值，也可以通过配置项 `spring.cloud.nacos.config.prefix`来配置。
      - `spring.profiles.active` 即为当前环境对应的 profile，详情可以参考 [Spring Boot文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html#boot-features-profiles)。 **注意：当 `spring.profiles.active` 为空时，对应的连接符 `-` 也将不存在，dataId 的拼接格式变成 `${prefix}.${file-extension}`**，注意最好不要为空
      - `file-exetension` 为配置内容的数据格式，可以通过配置项 `spring.cloud.nacos.config.file-extension` 来配置。目前只支持 `properties` 和 `yaml` 类型。
      - 注意运行报错的原因可能有：配置时的文件后缀应统一为yaml；还有如果存在group属性的可能会报错，删除即可

   4. ```java
      @RestController
      @RefreshScope //支持Nacos的动态刷新功能。
      public class ConfigClientController
      {
          @Value("${config.info}")
          private String configInfo;
      
          @GetMapping("/config/info")
          public String getConfigInfo() {
              return configInfo;
          }
      }
      ```

      

   ## Gateway

   三大核心概念：

   1. Route 路由 路由是构建网关的基本模块，它由ID，目标URI，一系列的断言和过滤组成，如果断言为true则匹配该路由

   2. Predicate 断言 开发人员可以匹配HTTP请求中的所有内容（例如请求头或请求参数），如果请求与断言相匹配则进行路由
   3. Filter 过滤 指的是Spring框架中GatewayFilter的实例，使用过滤器，可以在请求被路由前或路由后对请求进行修改

   1.添加依赖

   ```java
   <!--gateway-->
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-gateway</artifactId>
           </dependency>
   ```

   2.yml文件配置

   ```java
   server:
     port: 9527
   
   spring:
     application:
       name: cloud-gateway
     cloud:
       gateway:
         discovery:
           locator:
             enabled: true #开启从注册中心动态创建路由的功能，利用微服务名进行路由
         routes:
           - id: payment_routh #payment_route    #路由的ID，没有固定规则但要求唯一，建议配合服务名
             #uri: http://localhost:8001          #匹配后提供服务的路由地址
             uri: lb://cloud-payment-service #匹配后提供服务的路由地址
             predicates:
               - Path=/payment/get/**         # 断言，路径相匹配的进行路由
   
           - id: payment_routh2 #payment_route    #路由的ID，没有固定规则但要求唯一，建议配合服务名
             #uri: http://localhost:8001          #匹配后提供服务的路由地址
             uri: lb://cloud-payment-service #匹配后提供服务的路由地址
             predicates:
               - Path=/payment/lb/**         # 断言，路径相匹配的进行路由
               #- After=2020-02-21T15:51:37.485+08:00[Asia/Shanghai]
               #- Cookie=username,zzyy
               #- Header=X-Request-Id, \d+  # 请求头要有X-Request-Id属性并且值为整数的正则表达式
         nacos:
           discovery:
             server-addr: 10.221.11.133:8848
   
   ```

   另外一种路由配置方式

   ```java
   @Configuration
   public class GateWayConfig
   {
       @Bean
       public RouteLocator customRouteLocator(RouteLocatorBuilder routeLocatorBuilder)
       {
           RouteLocatorBuilder.Builder routes = routeLocatorBuilder.routes();
   
           routes.route("path_route_atguigu",
                   r -> r.path("/guonei")
                           .uri("http://news.baidu.com/guonei")).build();
   
           return routes.build();
       }
   }
   ```

   

   开启动态路由方式

   yml文件中discover.locator.enabled = true

   uri需要写 lb:// 后面添加微服务名

   

   常用predicate断言 可以去看官方文档

   ![image-20210911110646314](C:\Users\chen\AppData\Roaming\Typora\typora-user-images\image-20210911110646314.png)

Filter 声明周期，pre或者post 种类两种 GatewayFilter 和GlobalFilter

一般自定义全局过滤器

首先创建一个类filter.***Filter 并实现GlobalFilter、Ordered接口

```java
@Component
@Slf4j
public class MyLogGateWayFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("*******come in MyLogGateWayFilter: " + new Date());
        String uname = exchange.getRequest().getQueryParams().getFirst("uname");
        if (uname == null) {
            log.info("*******用户名为null，非法用户");
            exchange.getResponse().setStatusCode(HttpStatus.NOT_ACCEPTABLE);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

    //返回加载过滤器的顺序，值越小越优先
    @Override
    public int getOrder() {
        return 0;
    }
}

```

这个例子中访问时必须要携带uname 才可以访问成功

## Feign

Feign是一个声明式的Web服务客户端，只需要创建一个接口并且在接口上添加注解即可

1. 添加依赖

   ```java
     <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-feign</artifactId>
           </dependency>
   ```

   

2. 主启动类添加`@EnableFeignClients` 

   ```java
   @SpringBootApplication
   @EnableFeignClients
   public class Application {
       
       public static void main(String[] args) {
           SpringApplication.run(Application.class, args);
       }
   }
   ```

   

3. 定义接口，接口@FeignClient注解指定服务名来绑定服务，然后再使用Spring MVC的注解来绑定具体该服务提供的REST接口。服务不区分大小写

   ```java
   @FeignClient(value = "hello-service-provider")//微服务名
   public interface HelloServiceFeign {
   
       @RequestMapping(value = "/demo/getHost", method = RequestMethod.GET)
       public String getHost(String name);
   
       @RequestMapping(value = "/demo/postPerson", method = RequestMethod.POST, produces = "application/json; charset=UTF-8")
       public Person postPerson(String name);
   }
   ```

4. 使用方式

   ```java
   @RestController
   public class RestClientController {
   
       @Autowired
       private HelloServiceFeign client;
   
       /**
        * @param name
        * @return Person
        * @Description: 测试服务提供者post接口
        * @create date 2018年5月19日上午9:44:08
        */
       @RequestMapping(value = "/client/postPerson", method = RequestMethod.POST, produces = "application/json; charset=UTF-8")
       public Person postPerson(String name) {
           return client.postPerson(name);
       }
   
       /**
        * @param name
        * @return String
        * @Description: 测试服务提供者get接口
        * @create date 2018年5月19日上午9:46:34
        */
       @RequestMapping(value = "/client/getHost", method = RequestMethod.GET)
       public String getHost(String name) {
           return client.getHost(name);
       }
   }
   ```

   

