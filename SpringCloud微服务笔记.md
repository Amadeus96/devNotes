# SpringCloud 微服务笔记



## nacos服务注册

有不清楚的可以去查官方文档，官网是最好的学习文档

1. 添加pom依赖

   ```java
      <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
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

   
   
   ## Hystrix
   
   服务降级：服务器忙，请稍后再试，不让客户端等待并立刻返回一个友好提示，fallback 
   
   服务熔断：类比保险丝达到最大服务访问后，直接拒绝访问，拉闸限电，然后调用服务降级的方法并返回友好提示
   
   服务限流：秒杀高并发操作，严禁一窝蜂的过来拥挤，大家排队，一秒钟N个，有序进行
   
   
   
   解决方式：1. 对方服务超时了，调用者不能一直卡死等待，必须由服务降级
   
   				2. 对方服务宕机了，调用者不能一直卡死等待，必须有服务降级
      				3. 对方服务Ok,调用者自己出故障或有自我要求（自己的等待时间小于服务提供者）
   
   一.服务调用者
   
   1. 添加依赖
   
      ```java
       <dependency>
                  <groupId>org.springframework.cloud</groupId>
                  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
              </dependency>
      ```
   
   2. service层上加入@HystrixCommand注解 一旦调用服务方法失败并抛出了错误信息后，会自动调用@HystrixCommand标注好的fallbackMethod调用类中的指定方法
   
      ```java
         @HystrixCommand(fallbackMethod = "paymentInfo_TimeOutHandler",commandProperties = {
                  @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="5000")//这里是设置了超时时间，超过就返回
          })
          public String paymentInfo_TimeOut(Integer id)
          {
              //int age = 10/0;
              try { TimeUnit.MILLISECONDS.sleep(3000); } catch (InterruptedException e) { e.printStackTrace(); }
              return "线程池:  "+Thread.currentThread().getName()+" id:  "+id+"\t"+"O(∩_∩)O哈哈~"+"  耗时(秒): ";
          }
          public String paymentInfo_TimeOutHandler(Integer id)
          {
              return "线程池:  "+Thread.currentThread().getName()+"  8001系统繁忙或者运行报错，请稍后再试,id:  "+id+"\t"+"o(╥﹏╥)o";
          }
      ```
   
      
   
   3. 主启动类添加新注解@EnableCircuitBreaker

存在问题：每个业务方法对应要给兜底的方法，代码膨胀，统一和自定义分 开

所以可以添加@DefaultProperties(defaultFallback=""),这里指定默认的处理方法，如果业务接口没有指定具体的配置，则执行默认的配置

```java
@RestController
@Slf4j
@DefaultProperties(defaultFallback = "payment_Global_FallbackMethod")
public class OrderHystirxController
{
    @Resource
    private PaymentHystrixService paymentHystrixService;

    @GetMapping("/consumer/payment/hystrix/ok/{id}")
    public String paymentInfo_OK(@PathVariable("id") Integer id)
    {
        String result = paymentHystrixService.paymentInfo_OK(id);
        return result;
    }

    @GetMapping("/consumer/payment/hystrix/timeout/{id}")
    @HystrixCommand(fallbackMethod = "paymentTimeOutFallbackMethod",commandProperties = {
            @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="1500")
    })
    //@HystrixCommand
    public String paymentInfo_TimeOut(@PathVariable("id") Integer id)
    {
        int age = 10/0;
        String result = paymentHystrixService.paymentInfo_TimeOut(id);
        return result;
    }
    public String paymentTimeOutFallbackMethod(@PathVariable("id") Integer id)
    {
        return "我是消费者80,对方支付系统繁忙请10秒钟后再试或者自己运行出错请检查自己,o(╥﹏╥)o";
    }

    // 下面是全局fallback方法
    public String payment_Global_FallbackMethod()
    {
        return "Global异常处理信息，请稍后再试，/(ㄒoㄒ)/~~";
    }
}

```



服务降级，客户端去调用服务端，碰上服务端宕机或者关闭

通赔服务降级

1. 新建一个类实现feign配置接口，fallback返回可以在对应的方法中进行异常处理

   ```java
   @Component
   public class PaymentFallbackService implements PaymentHystrixService
   {
       @Override
       public String paymentInfo_OK(Integer id)
       {
           return "-----PaymentFallbackService fall back-paymentInfo_OK ,o(╥﹏╥)o";
       }
   
       @Override
       public String paymentInfo_TimeOut(Integer id)
       {
           return "-----PaymentFallbackService fall back-paymentInfo_TimeOut ,o(╥﹏╥)o";
       }
   }
   
   ```

2. @FeignClient注解上需要加入fallback参数，参数名对1中对应的类名

   ```java
   @Component
   @FeignClient(value = "CLOUD-PROVIDER-HYSTRIX-PAYMENT" ,fallback = PaymentFallbackService.class)
   public interface PaymentHystrixService
   {
       @GetMapping("/payment/hystrix/ok/{id}")
       public String paymentInfo_OK(@PathVariable("id") Integer id);
   
       @GetMapping("/payment/hystrix/timeout/{id}")
       public String paymentInfo_TimeOut(@PathVariable("id") Integer id);
   }
   
   ```

3. yml配置

   ```java
   
   feign:
     hystrix:
       enabled: true
   ```

   

   全局fallback优先级大于通配服务降级

   写在controller类内部的处理，并加上默认或者指定降级处理。**如果没有继承service的降级处理，他处理的是服务端与客户端异常**

   写在controller类内部的处理，并加上默认或者指定降级处理。**如果存在一个继承service的降级处理，他处理的是客户端自身的异常**

   继承service的降级处理，他处理的是服务端的异常

   **他们俩都能处理服务端异常，但是如果都存在，则service优先处理服务端出现的异常。**



服务熔断

当链路的某个微服务出错不可用或者响应时间太长时，会进行服务的降级，进而熔断该节点的微服务调用，快速返回错误的响应信息，当检测到该节点微服务调用响应正常后，恢复调用链路。

service层

```java
    //=====服务熔断
    @HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback",commandProperties = {
            @HystrixProperty(name = "circuitBreaker.enabled",value = "true"),// 是否开启断路器
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "10"),// 请求次数
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "10000"), // 时间窗口期
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "60"),// 失败率达到多少后跳闸
    })
    public String paymentCircuitBreaker(@PathVariable("id") Integer id)
    {
        if(id < 0)
        {
            throw new RuntimeException("******id 不能负数");
        }
        String serialNumber = IdUtil.simpleUUID();

        return Thread.currentThread().getName()+"\t"+"调用成功，流水号: " + serialNumber;
    }
    public String paymentCircuitBreaker_fallback(@PathVariable("id") Integer id)
    {
        return "id 不能负数，请稍后再试，/(ㄒoㄒ)/~~   id: " +id;
    }
```

controller层

```java
    //====服务熔断
    @GetMapping("/payment/circuit/{id}")
    public String paymentCircuitBreaker(@PathVariable("id") Integer id)
    {
        String result = paymentService.paymentCircuitBreaker(id);
        log.info("****result: "+result);
        return result;
    }
```

默认条件

当满足一定的阈值时，默认10秒内超过20个请求次数

当失败率达到一定的时候，默认10秒内超过50%的请求失败

到达以上阈值，断路器将会开启

当开启的时候，所有请求都不会转发

一段时间之后（默认是5s），这个时候断路器是半开状态，会让其中一个请求进行转发，如果成功，短路器会关闭，若失败，继续开启。重复4和5

熔断后不会调用主逻辑，直接调用fallback

Hytrix图形化监控

1. 添加依赖

   ```java
       <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-actuator</artifactId>
           </dependency>
   ```

2. 启动类添加注解`@EnableHystrixDashboard`

   ```java
   @SpringBootApplication
   @EnableHystrixDashboard
   public class HystrixDashboardMain9001
   {
       public static void main(String[] args) {
               SpringApplication.run(HystrixDashboardMain9001.class, args);
       }
   }
   
   ```

3. 需要在被监控的微服务启动类加入以下代码

   ```java
     /**
        *此配置是为了服务监控而配置，与服务容错本身无关，springcloud升级后的坑
        *ServletRegistrationBean因为springboot的默认路径不是"/hystrix.stream"，
        *只要在自己的项目里配置上下面的servlet就可以了
        */
       @Bean
       public ServletRegistrationBean getServlet() {
           HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
           ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
           registrationBean.setLoadOnStartup(1);
           registrationBean.addUrlMappings("/hystrix.stream");
           registrationBean.setName("HystrixMetricsStreamServlet");
           return registrationBean;
       }
   ```

4.监控网址http://localhost:9001/hystrix

然后在输入需要监控的微服务网址，这里的示例是http://localhost:8001/hystrix.stream

![image-20210913112928513](C:\Users\chen\AppData\Roaming\Typora\typora-user-images\image-20210913112928513.png)

