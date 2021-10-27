# springcloud-oauth
springcloud projects for oauth

## oauth2认证流程

<img src="https://img-blog.csdnimg.cn/20210306221832431.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zODM2MTM0Nw==,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom: 67%;" />

资源拥有者：一般指用户

客户端：需要登陆的网站或者应用

认证服务器：该微服务用于认证功能

资源服务器：该微服务包含需要访问的接口

## oauth2颁发token授权方式

1. **授权码模式**
2. **密码模式**
3. 隐藏模式
4. 客户端模式

## 微服务统一认证

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210105111029928.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlbGxvQ2VkYXI=,size_16,color_FFFFFF,t_70#pic_center)

## 认证服务器搭建

引入的maven包

需要注意的是spring cloud 的版本要与springboot的版本对应，具体的版本可以去springcloud官网去查

```java
 <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR2</spring-cloud.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <version>2.1.3.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-oauth2</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>4.6.3</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.9.0</version>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

yml文件

```java
server:
  port: 9401
spring:
  application:
    name: oauth-server
  cloud:
    nacos:
      discovery:
        server-addr: 10.221.11.133:8848
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://10.221.18.16:16033/integration?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&transformedBitIsBoolean=true&allowPublicKeyRetrieval=true&serverTimezone=Asia/Shanghai
    username: root
    password: 4rfv3edc!
management:
  endpoints:
    web:
      exposure:
        include: '*'
security:
  oauth2:
    resource:
      id: oauth
```

`configure(ClientDetailsServiceConfigurer clients)`
⽤来配置客户端详情服务（ClientDetailsService），客户端详情信息在这⾥进⾏初始化，能够把客户端详情信息写死在这⾥或者是通过数据库来存储调取详情信息
`configure(AuthorizationServerEndpointsConfigurer endpoints)`
⽤来配置令牌（token）的访问端点和令牌服务(token services)
`configure(AuthorizationServerSecurityConfigurer security)
⽤来配置令牌端点的安全约束`

```java
package com.security.oauth.oauthserver.config;

import com.security.oauth.oauthserver.common.JwtTokenEnhancer;
import com.security.oauth.oauthserver.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.oauth2.config.annotation.builders.JdbcClientDetailsServiceBuilder;
import org.springframework.security.oauth2.config.annotation.configurers.ClientDetailsServiceConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configuration.AuthorizationServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerEndpointsConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerSecurityConfigurer;
import org.springframework.security.oauth2.provider.token.*;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;


import javax.sql.DataSource;
import java.util.ArrayList;
import java.util.List;

/**
 * 当前类为Oauth2 server的配置类（需要继承特定的⽗类 AuthorizationServerConfigurerAdapter）
 * @Author liuruchen
 */
@Configuration
@EnableAuthorizationServer //开启认证服务功能
public class OauthServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private UserService userService;

    @Autowired
    @Qualifier("jwtTokenStore")
    private TokenStore tokenStore;

    @Autowired
    private JwtAccessTokenConverter jwtAccessTokenConverter;

    @Autowired
    private JwtTokenEnhancer jwtTokenEnhancer;

    @Autowired
    private DataSource dataSource;


    /**
     * 这里配置token令牌管理相关（token此时就是⼀个字符串，当下的token需要在服务器端存储
     * 那么存储在哪⾥呢？都是在这里配置）
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        TokenEnhancerChain enhancerChain = new TokenEnhancerChain();
        List<TokenEnhancer> delegates = new ArrayList<>();
        delegates.add(jwtTokenEnhancer); //配置JWT的内容增强器
        delegates.add(jwtAccessTokenConverter);
        enhancerChain.setTokenEnhancers(delegates);
        endpoints.authenticationManager(authenticationManager)//指定认证管理器，随后注⼊一个到当前类使⽤即可
                .userDetailsService(userService)
                .tokenStore(tokenStore) //配置令牌存储策略
                .accessTokenConverter(jwtAccessTokenConverter)
                .tokenEnhancer(enhancerChain);
    }
    /**
     * 客户端详情配置，比如client_id，secret；当前这个服务就如同 QQ 平台，客户端需要qq平台进⾏登录
     * 授权认证等，提前需要到QQ平台注册，QQ平台会给 客户端 颁发client_id等必要参数，表明客户端是谁
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {

/*        clients.inMemory()//存储到内存中
                .withClient("admin")//配置client_id
                .secret(passwordEncoder.encode("admin123456"))//配置client_secret
                .resourceIds("oauth")//指定客户端所能访问的资源id清单，此处的资源id是需要在具体的资源服务器上配置
                .accessTokenValiditySeconds(3600)//配置访问token的有效期
                .refreshTokenValiditySeconds(864000)//配置刷新token的有效期
                .redirectUris("http://www.baidu.com")//配置redirect_uri，用于授权成功后跳转
                .autoApprove(true) //自动授权配置
                .scopes("all")//配置申请的权限范围
                .authorizedGrantTypes("authorization_code","password","refresh_token");//配置grant_type，表示授权类型*/
        //存储到数据库中
        JdbcClientDetailsServiceBuilder jcsb = clients.jdbc(dataSource);
        jcsb.passwordEncoder(passwordEncoder);
    }

    /**
     * 认证服务器最终是以 api 接口的方式对外提供服务（校验合法性并⽣成令牌、校验令牌等）
     * 那么，以api接口方式对外的话，就涉及到接口的访问权限，我们需要在这里进行必要的配置
     */
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        //相当于打开endpoints访问接口的开关，这样的话后期我们能够访问该接口
        //允许客户端表单认证
        security.allowFormAuthenticationForClients();
        //开启端口/oauth/token_key的访问权限（允许）
        security.checkTokenAccess("isAuthenticated()");
        // 开启端⼝/oauth/check_token的访问权限（允许）
        security.tokenKeyAccess("isAuthenticated()");
    }

}

```



jwt配置

```java
@Configuration
public class JwtTokenStoreConfig {

    @Bean
    public TokenStore jwtTokenStore() {
        return new JwtTokenStore(jwtAccessTokenConverter());
    }
    @Bean
    public JwtTokenEnhancer jwtTokenEnhancer() {
        return new JwtTokenEnhancer();
    }

    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter accessTokenConverter = new JwtAccessTokenConverter();
        accessTokenConverter.setSigningKey("test_key");//配置JWT使用的秘钥
        return accessTokenConverter;
    }
}
```

jwt增强

```java
/**
 * Jwt内容增强器
 * Created by macro on 2019/10/8.
 */
public class JwtTokenEnhancer implements TokenEnhancer {
    @Override
    public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
        Map<String, Object> info = new HashMap<>();
        info.put("enhance", "enhance info");
        ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(info);
        return accessToken;
    }
}
```

这里通过数据库配置client，client_secret 字段不能直接是 secret 的原始值，需要经过加密。因为是用的 `BCryptPasswordEncoder`，所以最终插入的值应该是经过 `BCryptPasswordEncoder.encode()`之后的值。采用了jwt存储token。下面是默认的数据库配置，表名默认为oauth_client_details,如果需要自定义，可以实现`ClientDetails`接口，新建表结构。

```mysql
create table oauth_client_details (
    client_id VARCHAR(256) PRIMARY KEY,
    resource_ids VARCHAR(256),
    client_secret VARCHAR(256),
    scope VARCHAR(256),
    authorized_grant_types VARCHAR(256),
    web_server_redirect_uri VARCHAR(256),
    authorities VARCHAR(256),
    access_token_validity INTEGER,
    refresh_token_validity INTEGER,
    additional_information VARCHAR(4096),
    autoapprove VARCHAR(256)
);
INSERT INTO oauth_client_details
    (client_id, client_secret, scope, authorized_grant_types,
    web_server_redirect_uri, authorities, access_token_validity,
    refresh_token_validity, additional_information, autoapprove)
VALUES
    ('user-client', '$2a$10$o2l5kA7z.Caekp72h5kU7uqdTDrlamLq.57M1F6ulJln9tRtOJufq', 'all',
    'authorization_code,refresh_token,password', null, null, 3600, 36000, null, true);

INSERT INTO oauth_client_details
    (client_id, client_secret, scope, authorized_grant_types,
    web_server_redirect_uri, authorities, access_token_validity,
    refresh_token_validity, additional_information, autoapprove)
VALUES
    ('order-client', '$2a$10$GoIOhjqFKVyrabUNcie8d.ADX.qZSxpYbO6YK4L2gsNzlCIxEUDlW', 'all',
    'authorization_code,refresh_token,password', null, null, 3600, 36000, null, true);
```

spring security配置

```java
package com.security.oauth.oauthserver.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

/**
 * 该配置类，主要处理用户名和密码的校验等事宜
 */
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    /**
     * 密码编码对象
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    /**
     * 注册⼀个认证管理器对象到容器
     */
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    /**
     * 设置哪些可以直接访问，哪些需要认证后访问
     * @param http
     * @throws Exception
     */
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.csrf()
                .disable()
                .authorizeRequests()
                .antMatchers("/oauth/**", "/login/**", "/logout/**")
                .permitAll()
                .anyRequest()
                .authenticated()
                .and()
                .formLogin()
                .permitAll();
    }
}

```

资源配置

主要是需要在资源服务器配置，这里可以加也可以不用加，注意，如果配置了资源服务，需要设置resource_id,并且在客户端的resource_ids中添加该resource_id才能正常访问该服务器的接口

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Value("${security.oauth2.resource.id}")
    private String resource_id;

    /**
     * 场景：⼀个服务中可能有很多资源（API接口）
     * 某⼀些API接口，需要先认证，才能访问
     * 某⼀些API接口，压根就不需要认证，本来就是对外开放的接口
     * 我们就需要对不同特点的接口区分对待（在当前configure⽅法中完成），设置是否需要经过认证
     */
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest()
                .authenticated()
                .and()
                .requestMatchers()
                .antMatchers("/user/**");
    }

    /**
     * 指定资源id
     *
     * @param resources
     * @throws Exception
     */
    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        resources.resourceId(resource_id);
    }
}

```

## 客户端搭建

pom依赖

```java
 <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-oauth2</artifactId>
            <version>2.2.5.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.9.1</version>
        </dependency>
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>3.14.2</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>
```

yml文件

```java
server:
  port: 9501
  servlet:
    session:
      cookie:
        name: OAUTH2-CLIENT-SESSIONID #防止Cookie冲突，冲突会导致登录验证不通过
oauth2-server-url: http://localhost:9401
spring:
  application:
    name: oauth-client
security:
  oauth2: #与oauth2-server对应的配置
    client:
      client-id: admin
      client-secret: 123456
      user-authorization-uri: ${oauth2-server-url}/oauth/authorize
      access-token-uri: ${oauth2-server-url}/oauth/token
    resource:
      jwt:
        key-uri: ${oauth2-server-url}/oauth/token_key
        key-value: test_key
```

```java
@Configuration
@EnableResourceServer
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Autowired
    private TokenStore jwtTokenStore;


    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        resources.tokenStore(jwtTokenStore).resourceId("oauth_client");
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().antMatchers("/login").permitAll();
    }
}
```

## 授权码访问流程

1. 获取授权码

访问[localhost:9401/oauth/authorize?client_id=admin&response_type=code&redirect_uri=http://www.baidu.com](http://localhost:9401/oauth/authorize?client_id=admin&response_type=code&redirect_uri=http://www.baidu.com)

授权后

https://www.baidu.com/?code=v9vhfc

复制该code

2. 打开postman

使用post请求http://localhost:9401/oauth/token

Authorization中选择basic Auth 输入客户端账号密码 body中添加grant_type 为authorization_code， code为刚才复制的code，redirect_uri这里随意，scope=all

3. 获取token后

在访问接口时，在header中添加Authorization 格式如下 bearer 空格eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsib2F1dGgiLCJvYXV0aF9jbGllbnQiXSwidXNlcl9uYW1lIjoibWFjcm8iLCJzY29wZSI6WyJhbGwiXSwiZXhwIjoxNjMyMjc0ODM3LCJhdXRob3JpdGllcyI6WyJhZG1pbiJdLCJqdGkiOiJlNzc4ODVmMC1mZjA4LTQxYzMtYWQ5OC1mMTM2ZDM2ODgzOGUiLCJjbGllbnRfaWQiOiJhZG1pbiIsImVuaGFuY2UiOiJlbmhhbmNlIGluZm8ifQ.Ko39YBK2lmup1PVuuCQ-R-ZBZLBlUbJpzX2iZ8FGgdg

## Feign调用微服务接口

需要增加拦截器

```java
@Configuration
public class FeignOauth2RequestInterceptor implements RequestInterceptor {

    private final String AUTHORIZATION_HEADER = "Authorization";
    private final String BEARER_TOKEN_TYPE = "Bearer";

    @Override
    public void apply(RequestTemplate requestTemplate) {
        SecurityContext securityContext = SecurityContextHolder.getContext();
        Authentication authentication = securityContext.getAuthentication();
        if (authentication != null && authentication.getDetails() instanceof OAuth2AuthenticationDetails) {
            OAuth2AuthenticationDetails details = (OAuth2AuthenticationDetails) authentication.getDetails();
            requestTemplate.header(AUTHORIZATION_HEADER, String.format("%s %s", BEARER_TOKEN_TYPE, details.getTokenValue()));
        }

    }
}

```

## 客户端权限问题

如果需要对接口增加权限判断，ResourceServerConfig要加注解@EnableGlobalMethodSecurity(prePostEnabled=true),并且在接口上添加@PreAuthorize("hasAuthority('admin')")，注意除了客户端模式，其它模式均是使用的是用户表的authorities，客户端模式是客户端表的authorities。
