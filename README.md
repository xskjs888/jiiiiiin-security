### 参考：

> [Spring Security 技术栈开发企业级认证与授权](https://coding.imooc.com/class/134.html)

> [Spring Security Tutorial 《Spring Security 教程》](https://waylau.gitbooks.io/spring-security-tutorial/content/)
>
> [使用 Spring Security 保护 Web 应用的安全](https://www.ibm.com/developerworks/cn/java/j-lo-springsecurity/)

> [社区 Spring Security 从入门到进阶系列教程](http://www.spring4all.com/article/428)

> [spring boot 要如何学习？](https://www.zhihu.com/question/53729800/answer/311948415)

> [SpringBoot 整合 Security（一）实现用户认证并判断返回 json 还是 view](https://www.jianshu.com/p/18875c2995f1)

> [Spring Security (Authentication & Authorisation from MySQL) in Spring Boot App | Tech Primers](https://youtu.be/egXtoL5Kg08)
>
> [Spring Security using Spring Data JPA + MySQL + Spring Boot](https://www.youtube.com/watch?v=IyzC1kkHZ-I&frags=pl%2Cwn)
>
> [Java开发中用到的，lombok是什么？](https://www.zhihu.com/question/42348457)
>
> [Java 8 中的 Streams API 详解](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/index.html)

# 关键点

## 代码结构



- ![代码结构](https://ws2.sinaimg.cn/large/006tNbRwgy1fue02z4h20j31kw0rc0y8.jpg)

## 自定义配置

- ![属性配置](https://ws1.sinaimg.cn/large/006tNbRwgy1fuilgv8orxj30rx0emgm6.jpg)

## spring security 相关：

### 参考文档：

  > https://spring.io/projects/spring-security 
 https://docs.spring.io/spring-security/site/docs/4.2.7.RELEASE/reference/htmlsingle/

### 配置规划

  ![](https://ws2.sinaimg.cn/large/0069RVTdgy1fuqepk8hyhj30zg0h3jtv.jpg)

### 验证码

  ![](https://ws1.sinaimg.cn/large/0069RVTdgy1fupfn8ecd6j30yv0fiwfo.jpg)

### 记住我

  ![](https://ws1.sinaimg.cn/large/0069RVTdgy1fuoes59unqj30zo0fuabd.jpg)

### [session 管理](https://coding.imooc.com/lesson/134.html#mid=6992)

#### session 超时处理
      配置：
    
      ```properties
      server:
        port: 8080
        session:
          # 设置session过期时间，单位是秒，默认30分钟，但是在springboot中（嵌入式）,最少的时间是1分钟，可以查看TomcatEmbeddedServletContainerFactory#configureSession方法`sessionTimeout = Math.max(TimeUnit.SECONDS.toMinutes(sessionTimeout), 1L);`
          timeout: 10
      ```
    
      如何自定义 session 失效导致的需要授权登录提示？
    
    - session 并发控制
    
    ```java
      .and()
      // 开启session管理配置：
      .sessionManagement()
          // 设置session过期之后处理的接口
          // .invalidSessionUrl(INVALID_SESSION_URL)
          // 设置session过期之后处理策略
          .invalidSessionStrategy(invalidSessionStrategy)
          // 设置单个用户session存在系统的数量，1标识系统中同一个用户只能存在一个session（登录用户）
          .maximumSessions(1)
          .maxSessionsPreventsLogin(securityProperties.getBrowser().getSession().isMaxSessionsPreventsLogin())
          // 设置不能剔除上一个登录用户，当session数量等于上面配置的最大数量
          // .maxSessionsPreventsLogin(true)
          // 用来做session被“剔除”之后的记录
          .expiredSessionStrategy(sessionInformationExpiredStrategy)
    ```

#### session 集群管理

![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv47i9tml2j30xg0ewab0.jpg)
​    
+ 依赖：

```xml
<!--做集群环境下的session管理-->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session</artifactId>
</dependency>
```

spring session 用来做集群环境下的 session“独立”存储支持；
spring session 支持的存储方式，参考`StoreType`；

 - 以 redis 为例：
	
	**使用 redis 的两个原因：**
	
	1. 在插入的是可以设置超时时间（数据的），即在系统设置的 session 超时时间到期之后，redis 将会自动帮我们清理数据
	2. 如果使用 jdbc 访问数据库的方式，因为每一次请求都会访问 session（ss 框架过滤器链），势必会对数据库造成很大压力，而且需要手动清理过期的 session 记录；
	
	+ 配置：
	
	```properties
	spring:
	  session:
	    # 这里的值对应的就是`StoreType`中的某一个存储类型
	    store-type: redis
	```
	
	+ 问题：
	
	1. `Caused by: org.springframework.core.serializer.support.SerializationFailedException: Failed to serialize object using DefaultSerializer; nested exception is java.lang.IllegalArgumentException: DefaultSerializer requires a Serializable payload but received an object of type [cn.jiiiiiin.security.core.validate.code.image.ImageCode]`
	
	开启 spring session -》 redis 之后，我们的 session 将会被存储到 redis 中，但是我们存放到 session 中的图形验证码没有实现`Serializable`接口，导致不可以被序列号到 redis 中，故报出此错误；
	
	+ 周边：[Intellij IDEA 自动生成 serialVersionUID](https://blog.csdn.net/tiantiandjava/article/details/8781776)
	
	+ 测试：启动两个节点，在其中一个登录，之后访问了一个节点的`user/me`，可以获取到上一个节点登录的用户信息；
	
		查看 redis 存储：
		![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv4agf6zn1j31hu0b0434.jpg)
	
		之后可以测试一下 spring security 并发控制等是否正常（开启集群之后，spring security 会监控 redis 中的 session 登录记录）

### 身份认证(核心)

#### 基础概念
  ![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuia8dpyrej30dv0bvdg4.jpg)

#### 核心流程(Spring Security Filter Chain)

  ![](https://ws2.sinaimg.cn/large/0069RVTdgy1fuoesrwcz1j31kw0jvgp1.jpg)

#### 参考：
> [Spring Security 核心过滤器链分析](https://juejin.im/post/5a434de6f265da43333eae7d)

项目启动时候输出的spring security filter chain: `Creating filter chain`

  ```java
     o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: OrRequestMatcher [requestMatchers=[Ant [pattern='/css/**'], Ant [pattern='/js/**'], Ant [pattern='/images/**'], Ant [pattern='/webjars/**'], Ant [pattern='/**/favicon.ico'], Ant [pattern='/error']]], []
            2018-08-24 10:46:59.538  INFO 5296 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: org.springframework.security.web.util.matcher.AnyRequestMatcher@1, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@64e247e, org.springframework.security.web.context.SecurityContextPersistenceFilter@1be3a294, org.springframework.security.web.header.HeaderWriterFilter@7d49fe37, org.springframework.security.web.authentication.logout.LogoutFilter@73633230, org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter@356ab368, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@729d1428, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@4425b6ed, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@66df362c, org.springframework.security.web.session.SessionManagementFilter@231c521e, org.springframework.security.web.access.ExceptionTranslationFilter@5af1b221, org.springframework.security.web.access.intercept.FilterSecurityInterceptor@5c30decf]
            2018-08-24 10:46:59.558  INFO 5296 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: org.springframework.boot.actuate.autoconfigure.ManagementWebSecurityAutoConfiguration$LazyEndpointPathRequestMatcher@7ca8f5d7, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@2dc319cf, org.springframework.security.web.context.SecurityContextPersistenceFilter@4f5df012, org.springframework.security.web.header.HeaderWriterFilter@3ad9fea, org.springframework.web.filter.CorsFilter@1690929, org.springframework.security.web.authentication.logout.LogoutFilter@53b9952f, org.springframework.security.web.authentication.www.BasicAuthenticationFilter@46df794e, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@38f3dbbf, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@574e4184, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@34d26a68, org.springframework.security.web.session.SessionManagementFilter@556e4588, org.springframework.security.web.access.ExceptionTranslationFilter@574413bd, org.springframework.security.web.access.intercept.FilterSecurityInterceptor@7da9b32c]
            2018-08-24 10:46:59.561  INFO 5296 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: OrRequestMatcher [requestMatchers=[Ant [pattern='/**']]], [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@7945986a, org.springframework.security.web.context.SecurityContextPersistenceFilter@4144d4a, org.springframework.security.web.header.HeaderWriterFilter@30a1b2ad, org.springframework.security.web.authentication.logout.LogoutFilter@37b44e8e, org.springframework.security.web.authentication.www.BasicAuthenticationFilter@11ec2b2f, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@20276412, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@1dae9e61, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@442151d1, org.springframework.security.web.session.SessionManagementFilter@436d2bb9, org.springframework.security.web.access.ExceptionTranslationFilter@20608ef4, org.springframework.security.web.access.intercept.FilterSecurityInterceptor@7c31e01f]
            2018-08-24 10:47:00.175  INFO 5296 --- [           m
  ```

####  个性化用户认证流程

+ 参考: https://coding.imooc.com/lesson/134.html#mid=6872

	浏览器相关 security 配置参考: bean::BrowserSecurityConfig、MyUserDetailsService


![](https://ws1.sinaimg.cn/large/006tNbRwgy1fujs30pbkcj31kw10atc6.jpg)

上图就是框架认证流程的核心处理流程

自定义UserDetailsService用来通过用户名获取UserDetails用户标识对象；


+ 如果需要自定义:

##### 自定义登录页面

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fuiiuwx7wxj31kw0s5wjo.jpg)
为了区分渠道，需要像上图那样去重新定义 security 框架的处理逻辑；

+ 关键在于 spring security 的授权配置：

```java
protected void configure(HttpSecurity http) throws Exception {
http
//...
.and()
// formLogin() 指定身份认证的方式
// 下面这样配置就改变了默认的httpBasic认证方式，而提供一个登录页面
.formLogin()
// 自定义登录页面
//                .signInUrl("/signIn.html")
.signInUrl("/signIn")
// 自定义登录交易请求接口，会被UsernamePasswordAuthenticationFilter所识别作为requiresAuthenticationRequestMatcher
.loginProcessingUrl("/signIn")
.and()
}
```

+ 获取请求的用户名和密码关键在`UsernamePasswordAuthenticationFilter#attemptAuthentication` 解析用户名密码哪里，目前默认是获取 request 中的 req params，也就是表单参数，如果要支持 json 请求，需要继承该类，手动实现，可以参考：

  ​    https://docs.spring.io/spring-security/site/docs/4.2.7.RELEASE/reference/htmlsingle/#form-login-filter
  
+ 自定义登录成功处理流程
  如果前端使用 ajax 异步请求一个需要授权登录的交易，那么默认的 spring security 的处理方式是在登录完成之后会重定向到上一个交易，如果要干预或者说，登录成功之后需要返回给前端登录用户的消息，那么就必须要进行这里的自定义；
  关键在于`AuthenticationSuccessHandler`
  ```java
  .and()
  // formLogin() 指定身份认证的方式
  // 下面这样配置就改变了默认的httpBasic认证方式，而提供一个登录页面
  //...
  // 配置自定义认证成功处理器
  .successHandler(jAuthenticationSuccessHandler)
  ```
#### 自定义登录失败处理流程

#### spring security 退出登录相关配置

  - 框架处理逻辑:

    ![框架处理逻辑](https://ws1.sinaimg.cn/large/0069RVTdgy1fv4g8yzc2yj30is0eowez.jpg)

    退出成功之后，会重定向到`http://localhost:8080/authentication/require?logout`登录页，通过`logout`参数标记是通过退出这个行为标记；

## spring social相关记录

### OAuth2协议介绍

  ![](https://ws2.sinaimg.cn/large/0069RVTdgy1fuqn54pzzmj30wd0ilaby.jpg)

+ 这里的第二步其实流程会像下面这样进行：

  关键点就是第三方应用引导用户到授权服务提供商去进行**授权页面**的显示和请求授权码，如果用户在服务提供商页面同意授权，那么服务提供商将下发一个**授权码**给第三方应用，
  之后第三方应用才能走上面的第三步流程：**申请令牌**，凭借获得的授权码，provider 验证授权码之后才会下发真正的授权令牌（token）；
  申请令牌的过程应该是在第三方应用的后台去请求 provider server；

  ![](https://ws4.sinaimg.cn/large/0069RVTdgy1fuqnerfq5uj30w10g3q4o.jpg)

  上图也是下面的授权模式中，使用最多的授权码模式的交互流程；

  ![](https://ws1.sinaimg.cn/large/0069RVTdgy1fuqn6o8vm2j30ox0bm3z0.jpg)

  授权码模式和密码模式、客户端模式不同的地方就在于，其他两种模式获取授权码是直接在第三方应用上完成的（第三方应用直接去请求授权码，而不是 provider“主动”下发）；

  授权码模式需要第三方应用有自己的服务器，去通过授权码获取 token，而没有 server 的应用一般使用简化模式

  授权码模式获得的 token 应该是存储于 server 端，安全性相对较高，简化模式，token 是直接返回给第三方应用的页面；

  以上是一个 Oauth 的授权流程；

### spring social 的授权登录流程：

  ![](https://ws2.sinaimg.cn/large/0069RVTdgy1fuqnpxoqo6j30wg0hr0v0.jpg)

  spring social 的作用就是帮助我们封装了这套流程到 ss 过滤器链的`SocialAuthenticationFilter`中；

  ![](https://ws4.sinaimg.cn/large/0069RVTdgy1fuqoup6qazj31kw0jqn0l.jpg)

  主要涉及的类：

  ![](https://ws2.sinaimg.cn/large/0069RVTdgy1fuqp901tqxj30y20izwgv.jpg)

### 接如 qq PC端的实现步骤

+ 参考：https://coding.imooc.com/lesson/134.html#mid=6890

+ 作为第三方比如接 qq 的实现步骤：

  1.先实现`Api`接口，因为每一家服务供应商提供的用户信息数据结构都是不一样的，就需要根据不同供应商去实现该接口
  参考：cn.jiiiiiin.security.core.social.qq.api.QQImpl 2.构建出 service provider（一般需要两个组件，OAuth2Operations 这个可以使用默认实现，如果供应商实现的 OAuth 比较标准、Api 就使用上面第一步得到的组件）
  参考:`cn.jiiiiiin.security.core.social.qq.connet.QQServiceProvider` 3.构建一个`Connection Factory`(一般需要两个自定义组件，就是上面两步得到的）
  参考：`cn.jiiiiiin.security.core.social.qq.connet.QQConnectionFactory` 4.得到`Connection`，就可以拿到用户信息
  通过`Connection Factory`构建 5.得到用户信息之后通过`UserConnectionRepository`就可以构建出 ss 框架自定义的`UserConnection`表中的数据，将 service provider

  ![](https://ws4.sinaimg.cn/large/0069RVTdgy1fuxbmqywxfj31kw0vntfu.jpg)

  OAuth2AuthenticationService 主要是用来进行 auth 流程的执行【`OAuth2AuthenticationService#getAuthToken`方法】，其会调用上面的 connection--> oauth2template 来执行 auth 流程；

  ```java
  public SocialAuthenticationToken getAuthToken(HttpServletRequest request, HttpServletResponse response) throws SocialAuthenticationRedirectException {
      // code就是授权服务提供商在成功授权之后给的授权码[Authorization Code]
      // 参考：http://wiki.connect.qq.com/%E4%BD%BF%E7%94%A8authorization_code%E8%8E%B7%E5%8F%96access_token
      // 判断是否有授权码，即auth流程第三步返回
  			String code = request.getParameter("code");
  			if (!StringUtils.hasText(code)) {
                  // 认为是流程的第一步
  				OAuth2Parameters params =  new OAuth2Parameters();
  				params.setRedirectUri(buildReturnToUrl(request));
  				setScope(request, params);
  				params.add("state", generateState(connectionFactory, request));
  				addCustomParameters(params);
                  // 如果是第一步就会组织参数，抛出异常，让social将请求重定向到qq的授权地址
  				throw new SocialAuthenticationRedirectException(getConnectionFactory().getOAuthOperations().buildAuthenticateUrl(params));
  			} else if (StringUtils.hasText(code)) {
  				try {
                      // 如果有授权码
  					String returnToUrl = buildReturnToUrl(request);
                      // 那授权码去换取令牌，第4、5步
                      // 这里很可能出现异常，因为第三方返回的响应数据，social解析不出来的时候
                      // 发送获取授权码的默认实现：extractAccessGrant(getRestTemplate().postForObject(accessTokenUrl, parameters, Map.class));
                      // 标明框架期望得到的是一个json数据【RestTemplate】
                      // 如果返回的数据内容类型非`application/json`或者数据不是json格式，就会出问题，qq返回的就不是这样的数据结构
                      // access_token=FE04************************CCE2&expires_in=7776000&refresh_token=88E4************************BE14
                      //http://wiki.connect.qq.com/%E4%BD%BF%E7%94%A8authorization_code%E8%8E%B7%E5%8F%96access_token
                      // 解决方式就是自己实现template，添加相应的数据格式解析器，参考：cn.jiiiiiin.security.core.social.qq.connet.QQOAuth2Template#createRestTemplate
                      // restTemplate.getMessageConverters().add(new StringHttpMessageConverter(Charset.forName("UTF-8")));
  					AccessGrant accessGrant = getConnectionFactory().getOAuthOperations().exchangeForAccess(code, returnToUrl, null);
  					// TODO avoid API call if possible (auth using token would be fine)
  					Connection<S> connection = getConnectionFactory().createConnection(accessGrant);
  					return new SocialAuthenticationToken(connection, null);
  				} catch (RestClientException e) {
  					logger.debug("failed to exchange for access", e);
  					return null;
  				}
  			} else {
  				return null;
  			}
  		}
  ```

+ 授权完成之后  服务提供商的  用户信息会封装到`Conncetion`中；

+ 之后再进入 ss 身份认证的流程，`SocialAuthenticationToken`会包含着 Connection 一起丢给`AuthenticationManager`进行身份认证；

+ `SocialAuthenticationProvider`会调用`JdbcUsersConnectionRepository`来  通过 connection 中的授权用户信息去数据库[UserConnection 表]中查询一个`UserId`，之后调用`SocialUserDetailsService`去查询正在的业务系统的用户信息`SocialUserDetails`，重新构建 Authtication Token 标记为认证成功， 之后防止到 ss 的 context 中， 最后防止到 session 中，标识授权登录完成；

  - 绑定解绑 spring social 服务，`ConnectController`:

    1. `/connect::GET`提供查询当前登录用户查询第三方授权绑定信息集合的接口；

       需要自定义响应视图，参考`CustomConnectionStatusView`

    2. `/connect/[provideId]::POST`提供绑定当前登录用户到对第三方授权服务提供商的接口；

       需要自定义响应视图，参考`CustomConnectionStatusView`

### 本地修改host文件测试方式：

![](https://ws1.sinaimg.cn/large/0069RVTdgy1fuso8tlo2fj311w0hc74n.jpg)
上面的错误是因为没有在或者配置错误，qq 的开放平台配置的第三方应用的回调域，即流程的第三部，在服务提供商页面授权完毕之后，其会回调我们配置的这个域名；

而实际在第一步导向到 qq 服务器的时候，spring social 已经为我们处理了`redirec_url`这个参数：
`https://graph.qq.com/oauth2.0/show?which=error&display=pc&error=100010&client_id=100550231&response_type=code&redirect_uri=http%3A%2F%2Flocalhost%3A8080%2Fauth%2Fqq&state=c1a93ae2-8da1-4a1a-bc78-5ec5bc8b6b5d`

开发中这个域名还是本地 localhost 就和配置在开放平台的不匹配就报错了；

`redirect_uri=http%3A%2F%2Flocalhost%3A8080%2Fauth%2Fqq`这个是我们的登录授权页面发起 qq 渠道授权的时候 href 的接口，即我们自身的接口；

解决方式，让这里的 redirect_uri 和平台注册时候填写的回调域保持同一个域名；

可以使用修改本机的 host 文件完成；
![](https://ws4.sinaimg.cn/large/0069RVTdgy1fuswrpang1j30qq0k8jrt.jpg)

或者部署到测试服务器，进行正式的联调；


## 自定义改造

### 基于 Token 的“会话保持” - Spring Security OAuth 实现服务提供商端介绍

> [SpringBoot + Spring Security OAuth2 基本使用](https://blog.csdn.net/u013435893/article/details/79735097)

### 使用 cookie 导致的一些问题:

![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv4jvkv8f9j30te0gxaaj.jpg)

### Spring Security OAuth介绍

![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv4kb5juokj30u20hh3zw.jpg)

其中绿色部分，spring security 已经帮我们做了  默认实现；

### 实现一个标准的 OAuth2 协议中 Provider 角色的主要功能

  - 添加了`@EnableAuthorizationServer`注解之后，项目就可以当做一个授权服务提供商，给第三方应用提供 oauth 授权服务`，参考`CustomAuthorizationServerConfig`

  - 添加了`@EnableResourceServer`注解标明当前应用是一个“资源服务器”提供商`

  - 流程和参数参考:https://tools.ietf.org/html/rfc6749#section-4.1

  - spring security 默认的授权页面：

  ![](https://ws2.sinaimg.cn/large/0069RVTdgy1fv5e6jnxg1j30p10aedgr.jpg)

  - 用户名密码模式
    ![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv5epvqbcrj31120g3jtw.jpg)

  - 授权码模式
    ![](https://ws2.sinaimg.cn/large/0069RVTdgy1fv5er1a31uj30zs0etdhn.jpg)

    第一步：发送获取授权码请求（模拟第三方调用）
    http://localhost:8080/oauth/authorize?response_type=code&client_id=fa0d197a-7a7b-4103-b9f6-6fad9d44feb2&redirect_uri=http://www.baidu.com&scope=all

    该 url 具有如下信息：

    1.那个第三方应用请求授权：通过`client_id`标识

    2.那个用户提供授权：通过 basic 页面输入的用户名密码标识

    3.给什么权限：通过 `scope` 标识

+ 问题：

    1. 需要添加`ROLE_USER`角色

    ```java
     /**
     *
     * @param userId 业务系统用户唯一标识
     * @return
     */
    private SocialUserDetails buildUser(String userId) {
        // `ROLE_USER`权限提供给，应用作为oauth授权服务提供商时候，第三方在获取授权码的时候，返回用户必须要有这个角色
        return new SocialUser(userId, passwordEncoder.encode("a11111"),
                true, true, true, true,
                AuthorityUtils.commaSeparatedStringToAuthorityList("admin,ROLE_USER"));
    }
    ```

    2.简单配置第三方 clientid：

    ```properties
      security:
        oauth2:
          client:
            client-id: immoc
            client-secret: immocsecret
    ```

    设置之后获取授权码的链接就变为：
    `http://localhost:8080/oauth/authorize?response_type=code&client_id=immoc&redirect_uri=http://www.baidu.com&scope=all`

    使用 postman 添加授权头：

    ![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv5o2y0e2pj31kw0syabx.jpg)

    ![](https://ws4.sinaimg.cn/large/0069RVTdgy1fv5lqnreh3j31kw0kojs3.jpg)

    ![](https://ws1.sinaimg.cn/large/0069RVTdgy1fv5oa4j0atj31ce0y6gnf.jpg)

    通过 token 方式进行身份认证：

    ![](https://ws3.sinaimg.cn/large/0069RVTdgy1fv5qea3mqyj31kw0uttcf.jpg)

### spring security oauth 核心源码：

+ 参考： https://coding.imooc.com/lesson/134.html#mid=7233

![](https://ws1.sinaimg.cn/large/0069RVTdgy1fv5qj3zttsj30yn0iamz6.jpg)
​    

- TokenEndpoint 相当于一个控制器，其定义了：`@RequestMapping(value = "/oauth/token", method=RequestMethod.GET)`这样的一系列接口，来处理 oauth 服务提供商提供给第三方的一些接口，比如获取令牌的请求、授权码的请求；
其中`http://{{host}}/oauth/token`获取授权令牌的时候，该接口是根据`grant_type`来判断需要执行的是要走“授权码”还是“用户名密码”模式的流程；

- ClientDetailsService 类似之前的 UserDetailsService，用来读取第三方应用的消息的，上面发送获取授权 token 的时候在请求头中附带的：`请求的Header中有一个Authorization参数，该参数的值是Basic + （clientId:secret Base64值）`，这个接口会读取这个头参数，生成第三方的应用信息，返回调用端`ClientDetails`对象；

- ClientDetails 封装了第三方应用的消息，类似 UserDetails；

- TokenRequest 封装了请求体中的参数信息，由 TokenEndpoint 创建，同时持有了 ClientDetails 对象；

- TokenGranter 由 TokenEndpoint 调用根据 TokenRequest 中的 GrantType 来选取一个实现类去做令牌生成的逻辑，其跟着会创建 OAuth2Request（是 TokenRequest+ClientDetails 的整合对象），另外一个创建 Authentication，其封装了认证用户的授权信息，这个里面的用户授权信息是通过 UserDetailsService 读取出来的；

- OAuth2Authentication 由 OAuth2Request 和 Authentication 组合而成，标识了，哪一个第三方应用，需要哪一个用户给予授权，使用哪种授权模式等信息持有者

- AuthorizationServerTokenServices 认证服务器令牌服务，在流程创建了 OAuth2Authentication 之后将 OAuth2Authentication 交给当前类

- TokenStore 订制令牌的存取
- TokenEnhancer 令牌增强器，可以改造生成的令牌

- OAuth2AccessToken 由 tokenServices#createAccessToken 创建

### 自定义 token 登录：

+ 参考：https://coding.imooc.com/lesson/134.html#mid=7224
  
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fv6jkj7vcgj30zq0hn406.jpg)
​    
使用 spring security oauth 的授权模式改造成上面的  流程，来响应登录请求，返回 token 令牌，替代 cookie 的 jsessionid 模式；

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fv6jrwvd86j30zo0kedi4.jpg)

设置`请求的Header中有一个Authorization参数，该参数的值是Basic + （clientId:secret Base64值）`标识第三方应用请求获取授权令牌；

`clientid`参数是在请求头的`Authorization参数`中解析得到；

需要改造我们的`AuthenticationSuccessHandler`来完成后续的生成 Token 流程，依赖 spring security oauth 默认的创建逻辑；

参考:

  - BasicAuthenticationFilter#doFilterInternal 解析`clientid`参数是在请求头的`Authorization参数`中的 clientid；

  ```java
  @Override
  		protected void doFilterInternal(HttpServletRequest request,
  				HttpServletResponse response, FilterChain chain)
  						throws IOException, ServletException {
  			final boolean debug = this.logger.isDebugEnabled();

  			String header = request.getHeader("Authorization");

  			if (header == null || !header.startsWith("Basic ")) {
  				chain.doFilter(request, response);
  				return;
  			}

  			try {
  				String[] tokens = extractAndDecodeHeader(header, request);
  				assert tokens.length == 2;

  				String username = tokens[0];
  ```

#### 验证码改造

  ![](https://ws3.sinaimg.cn/large/006tNbRwgy1fv6tqe05yuj30xn0g5t9d.jpg)

  因为使用 token 模式进行认证是没有 session 的，故之前将  验证码存储在 session 中的做法就并不可行，故思路就会改成上面的方式：

  1.在请求验证码的时候，传递一个`deviceId`标识客户端

  2.将验证码和标识存储到类 redis 存储中；

  3.在校验的时候获取存储的数据进行校验

  - 模拟客户端短信验证码登录

  ```bash
   jiiiiiin@jiiiiiins-MacBook-Pro  ~/Documents/StudioProjects/ynrcc/MobileBank   tag/306curl -X POST \ST \
  http://127.0.0.1:8080/authentication/mobile \
  -H 'Authorization: Basic aW1tb2M6aW1tb2NzZWNyZXQ=' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'Postman-Token: c36660c7-8389-45fd-b01b-e6355ad196a2' \
  -H 'deviceId: 007' \
  -d 'mobile=13588888888&smsCode=6255'
  {"access_token":"95ee3517-027c-44d9-8449-b0f3f908f8ae","token_type":"bearer","refresh_token":"00cb5fb8-c253-4d4b-9a50-c179809ed435","expires_in":43199}%
  ```

  这种模拟就  不会受到 session 的影响；

#### token 模式社交登录：

客户端的社交登录，一般依赖于第三方服务提供商的客户端 sdk 来  走第一步流程；

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fv6v0fbc6fj30wb0jkgns.jpg)

上图是简化模式，在 sdk 导向用户完成授权之后返回的直接就是`openid`和`accessToken`；

而客户端在进行授权认证的是需要向自身的服务提供商获取`accessToken`，通过上面第三方拿到的`openid`来作为标识；

也就是说  服务器需要提供一个接口，通过`openid`来换取自定义的`accessToken`令牌，即  身份认证（登录）；

需要自定义：

```java
+ OpenIdAuthenticationToken
封装认证信息
 * <p>
 * 使用第三方授权服务提供商获取的`openid`来进行认证（登录）

+ OpenIdAuthenticationFilter
  请求参数过滤器
  拦截app token认证模式的根据`openId`的认证请求
 * <p>
 * 交给{@link AuthenticationManager}完成身份认证

+ OpenIdAuthenticationProvider
  验证{@link OpenIdAuthenticationToken} 完成身份认证（登录）

+ OpenIdAuthenticationSecurityConfig
  openid授权认证配置类，将自定义组件配置到spring security安全模块中；
```



#### token 社交登录（标准流程）

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fv7wv3r3daj30vs0h7acl.jpg)

需要app将授权码转发给应用，由应用去获取授权用户数据，在返回应用的授权令牌给app；

可以通过依赖browser项目模拟前3步，获取到服务提供商（如微信）的授权码，之后再依赖app模块，直接发送第4步（沿用browser浏览器上面的链接）

![image-20180913151524838](https://ws2.sinaimg.cn/large/006tNbRwgy1fv7xowbncyj30nm00ymxw.jpg)

![image-20180913151632693](https://ws2.sinaimg.cn/large/006tNbRwgy1fv7xpy64c8j30pu0bl76s.jpg)

因为我们在配置social的时候没有指定获取用户信息之后的处理器，故直接请求是发不通的（302，spring默认的成功处理做跳转，在浏览器环境其实是正确的，如果需要跳转），因为我们没有在获取用户信息成功之后，走我们自己的成功处理器`CustomAuthenticationSuccessHandler`,需要CustomSpringSocialConfigurer#setSocialAuthenticationFilterPostProcessor(socialAuthenticationFilterPostProcessor) 设置一个后处理器来完成；

在CustomSpringSocialConfigurer中完成配置，为此需要定义一个SocialAuthenticationFilterPostProcessor后处理器；



```java

@Component
public class AppSocialAuthenticationFilterPostProcessor implements SocialAuthenticationFilterPostProcessor {

    @Autowired
    private AuthenticationSuccessHandler customAuthenticationSuccessHandler;

    @Override
    public void process(SocialAuthenticationFilter socialAuthenticationFilter) {
        // 修改spring social的默认授权后处理
        socialAuthenticationFilter.setAuthenticationSuccessHandler(customAuthenticationSuccessHandler);
    }
}
```



![image-20180913171738358](https://ws4.sinaimg.cn/large/006tNbRwgy1fv817yw1ksj31kw0pjwnf.jpg)



这个url需要在设置为brower模式，跳转到微信扫码授权页面，就停掉服务，之后由微信重定向回来的那个url，来模拟客户端调用sdk获取到标准模式的授权码，那也就是说，客户端就调相同的接口`[filter-processes-url]/[providerId]`

#### token(app)模式自定义注册

```java
 /**
     * 社交登录配置类，供浏览器或app模块引入设计登录配置用。
     *
     * @return
     * @see SpringSocialConfigurer 可以参考默认实现
     */
    @Bean
    public SpringSocialConfigurer socialSecurityConfig() {
        final String filterProcessesUrl = securityProperties.getSocial().getFilterProcessesUrl();
        final CustomSpringSocialConfigurer configurer = new CustomSpringSocialConfigurer(filterProcessesUrl);
        // 配置自定义注册页面接口，当第三方授权获取user detail在业务系统找不到的时候默认调整到该页面
        configurer.signupUrl(securityProperties.getBrowser().getSignUpUrl());
        // 注入后处理器，以便app模式（标准）下授权登录能够完成
        configurer.setSocialAuthenticationFilterPostProcessor(socialAuthenticationFilterPostProcessor);
        return configurer;
    }
```

新增`setSocialAuthenticationFilterPostProcessor`设置，通过`socialAuthenticationFilterPostProcessor`自定义后处理器，来控制获取到第三方用户信息，但是没有用户标识（`UserConnnect`表中没有记录），让`socialAuthenticationFilterPostProcessor`帮我们将授权和客户端标识通过`AppSingUpUtils`(app环境下替换providerSignInUtils，避免由于没有session导致读不到社交用户信息的问题)将客户端和授权信息缓存到本地，在返回给客户端json格式的用户信息；



```java
public class SpringSocialConfigurerPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        // socialSecurityConfig就是需要进行后处理的bean
        if (StringUtils.equals(beanName, "socialSecurityConfig")) {
            // 实现`CustomSpringSocialConfigurer`
            SpringSocialConfigurer springSocialConfigurer = (SpringSocialConfigurer) bean;
            // 覆盖默认的针对浏览器的处理接口
            springSocialConfigurer.signupUrl(SecurityConstants.DEFAULT_SOCIAL_USER_INFO_URL);
            return springSocialConfigurer;
        }
        return bean;
    }
}

```

app模块组件，自定义`BeanPostProcessor`，覆盖注册处理接口，在app环境进行容器初始化的时候；



```java
 /**
     * 需要注册时跳到这里，返回401和用户信息给前端
     *
     * @param request
     * @return
     * @see AppSingUpUtils
     * @see cn.jiiiiiin.security.app.component.authentication.social.SpringSocialConfigurerPostProcessor
     * @see SocialConfig#socialSecurityConfig()
     */
    @GetMapping(SecurityConstants.DEFAULT_SOCIAL_USER_INFO_URL)
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public SocialUserInfo getSocialUserInfo(HttpServletRequest request) {
        Connection<?> connection = providerSignInUtils.getConnectionFromSession(new ServletWebRequest(request));
        appSingUpUtils.saveConnectionData(new ServletWebRequest(request), connection.createData());
        return buildSocialUserInfo(connection);
    }
```

实现处理接口；

还需要对当前接口配置授权；

```java
 /**
     * 系统授权注册接口，提供给第三方授权之后，为查询到业务系统的userid，即没有记录时候渲染的注册页面使用
     * <p>
     * 如果注册页面需要获取第三方授权用户信息，可以使用 {@link cn.jiiiiiin.security.browser.controller.BrowserSecurityController#getSocialUserInfo(HttpServletRequest)}
     * <p>
     * app模块需要使用{@link cn.jiiiiiin.security.app.AppSecurityController#getSocialUserInfo(HttpServletRequest)}
     *
     * @param user
     * @param request
     * @see cn.jiiiiiin.security.core.social.SocialConfig#socialSecurityConfig
     * <p>
     * 如果期望让用户进行第三方授权登录之后，自动帮用户创建业务系统的用户记录，完成登录，而无需跳转到下面这个接口进行注册，请看：
     * @see org.springframework.social.security.SocialAuthenticationProvider#toUserId 去获取userIds的方法，在{@link org.springframework.social.connect.jdbc.JdbcUsersConnectionRepository#findUserIdsWithConnection}中通过注入{@link ConnectionSignUp}完成
     */
    @PostMapping("/user/auth/register")
    public User register(User user, HttpServletRequest request, HttpServletResponse response) {
        // TODO 待写注册或者绑定逻辑（绑定需要查询应用用户的userid，通过授权用户信息，授权用户信息在providerSignInUtils中可以获取）
        // 不管是注册用户还是绑定用户，都会拿到一个用户唯一标识。
        String userId = user.getUsername();
        // 真正让业务系统用户信息和spring social持有的授权用户信息进行绑定，记录`UserConnection`表
        providerSignInUtils.doPostSignUp(userId, new ServletWebRequest(request));
        // app端关联
        appSingUpUtils.doPostSignUp(new ServletWebRequest(request), userId);
        return user;
    }
```

编写注册和绑定`UserConnection`表；

至此app模式的第三方授权注册、绑定，就可以实现了；



#### 刷新令牌

![image-20180918154338442](https://ws4.sinaimg.cn/large/006tNbRwgy1fvdqltok70j31kw0ndatc.jpg)

在客户端进行资源访问的时候，如果后台返回令牌过期**"expires_in": 3599,**，那么就可以静默的发送上面的接口，进行token令牌的刷新；



#### token(app) 自定义Token生成策略



+ 参考：https://coding.imooc.com/lesson/134.html#mid=7236



##### Token创建流程

![image-20180917170645562](https://ws1.sinaimg.cn/large/006tNbRwgy1fvcndv0n8gj31440nan0q.jpg)

##### 基本的Token参数配置

```java
/**
 * 认证服务器配置
 *
 * @author zhailiang
 * @see org.springframework.security.oauth2.provider.endpoint.TokenEndpoint
 */
@Configuration
@EnableAuthorizationServer
public class CustomAuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    /**
     * 当去做认证的时候使用的`userDetailsService`
     */
    @Autowired
    private UserDetailsService userDetailsService;

    /**
     * 当去做认证的时候使用的`authenticationManager`
     */
    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private SecurityProperties securityProperties;

    /**
     * 认证及token配置
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        // 当继承了`AuthorizationServerConfigurerAdapter`之后就需要自己配置下面的认证组件
        endpoints
                .authenticationManager(authenticationManager)
                .userDetailsService(userDetailsService);
    }

    /**
     * 客户端配置
     * <p>
     * 当复写了该方法，默认的
     * <p>
     * security:
     * oauth2:
     * client:
     * client-id: immoc
     * client-secret: immocsecret
     * 配置将会失效
     * <p>
     * 需要自己根据配置应用支持的第三方应用client-id等应用信息
     *
     * @param clients 那些应用允许来进行token认证
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        // clients.jdbc() 如果要实现qq那样提供授权信息给第三方最好使用这种模式，而下面的模式主要是针对`token`客户端登录的
        InMemoryClientDetailsServiceBuilder builder = clients.inMemory();
        if (!securityProperties.getOauth2().getClients().isEmpty()) {
            for (OAuth2ClientProperties client : securityProperties.getOauth2().getClients()) {
                // 指定支持的第三方应用信息
                builder.withClient(client.getClientId())
                        .secret(client.getClientSecret())
                        // 针对当前第三方应用所支持的授权模式，即http://{{host}}/oauth/token#grant_type
                        .authorizedGrantTypes("refresh_token", "authorization_code", "password")
                        // 配置令牌的过期时间限
                        .accessTokenValiditySeconds(client.getAccessTokenValidateSeconds())
                        //
                        .refreshTokenValiditySeconds(2592000)
                        // 针对当前第三方应用所支持的权限，即http://{{host}}/oauth/token#scope
                        // 说明应用需要的权限，发送请求的scope参数需要在此范围之内，不传就使用默认（即配置的值）
                        .scopes("all");
            }
        }
    }

}

```



上面这种方式创建的令牌是存储到内存，重启应用就会失效；

建议使用redis完成令牌的存储：`RedisTokenStore`

```java

@Configuration
public class TokenStoreConfig {

    /**
     * 使用redis存储token的配置，只有在imooc.security.oauth2.tokenStore配置为redis时生效
     *
     * @author zhailiang
     */
    @Configuration
    @ConditionalOnProperty(prefix = "jiiiiiin.security.oauth2", name = "tokenStore", havingValue = "redis")
    public static class RedisConfig {

        /**
         * 链接工厂
         */
        @Autowired
        private RedisConnectionFactory redisConnectionFactory;

        /**
         * @return
         */
        @Bean
        public TokenStore redisTokenStore() {
            return new RedisTokenStore(redisConnectionFactory);
        }

    }

    
@Configuration
@EnableAuthorizationServer
public class CustomAuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    /**
     * 认证及token配置
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        // 当继承了`AuthorizationServerConfigurerAdapter`之后就需要自己配置下面的认证组件
        endpoints
                .tokenStore(tokenStore)
                .authenticationManager(authenticationManager)
                .userDetailsService(userDetailsService);
```

声明使用`TokenStore`为redis；

这样token模式的存储也和应用分离，就可以实现多应用节点部署，无需前置（F5/Nginx）进行session粘黏；![image-20180917145147262](https://ws3.sinaimg.cn/large/006tNbRwgy1fvcjhjyo0xj30x60je7jg.jpg)

z'h

##### 使用JWT（Json Web Token）替换默认的Token



参考: 

[什么是JWT](什么是JWT)

[JSON Web Token 入门教程](http://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)

+ 自包含：包含有意义的消息（自身包含用户信息）相比ss原始的UUID座位Token，原始的令牌需要依赖一个存储，去查询附加信息；
+ 密签：使用的是签名机制来防串改
+ 可扩展：可以自定义放进token的信息

![image-20180917162836370](https://ws3.sinaimg.cn/large/006tNbRwgy1fvcma73pucj31hg0k4k1o.jpg)



集成：

```java
/**
 *
 */
package cn.jiiiiiin.security.app.server;

import cn.jiiiiiin.security.core.properties.SecurityProperties;
import lombok.val;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.security.oauth2.provider.token.TokenEnhancer;
import org.springframework.security.oauth2.provider.token.TokenStore;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
import org.springframework.security.oauth2.provider.token.store.JwtTokenStore;
import org.springframework.security.oauth2.provider.token.store.redis.RedisTokenStore;

/**
 * 负责令牌的存取
 *
 * @author zhailiang
 * @author jiiiiiin
 * @see CustomAuthorizationServerConfig
 */
@Configuration
public class TokenStoreConfig {

    /**
     * 使用redis存储token的配置，只有在imooc.security.oauth2.tokenStore配置为redis时生效
     *
     * @author zhailiang
     */
    @Configuration
    @ConditionalOnProperty(prefix = "jiiiiiin.security.oauth2", name = "tokenStore", havingValue = "redis")
    public static class RedisConfig {

        /**
         * 链接工厂
         */
        @Autowired
        private RedisConnectionFactory redisConnectionFactory;

        /**
         * @return
         */
        @Bean
        public TokenStore redisTokenStore() {
            return new RedisTokenStore(redisConnectionFactory);
        }

    }

    /**
     * 使用jwt时的配置，默认生效
     *
     * @author zhailiang
     */
    @Configuration
    @ConditionalOnProperty(prefix = "jiiiiiin.security.oauth2", name = "tokenStore", havingValue = "jwt", matchIfMissing = true)
    public static class JwtConfig {

        @Autowired
        private SecurityProperties securityProperties;

        /**
         * @return
         * @see TokenStore 处理token的存储
         */
        @Bean
        public TokenStore jwtTokenStore() {
            return new JwtTokenStore(jwtAccessTokenConverter());
        }

        /**
         * @return
         * @see JwtAccessTokenConverter 处理token的生成
         */
        @Bean
        public JwtAccessTokenConverter jwtAccessTokenConverter() {
            val converter = new JwtAccessTokenConverter();
            // 指定密签秘钥
            converter.setSigningKey(securityProperties.getOauth2().getJwtSigningKey());
            return converter;
        }

        /**
         * 用于扩展和解析JWT的信息
         * <p>
         * 业务系统可以自行配置自己的{@link TokenEnhancer}
         *
         * @return
         */
        @Bean
        @ConditionalOnBean(TokenEnhancer.class)
        public TokenEnhancer jwtTokenEnhancer() {
            return new TokenJwtEnhancer();
        }

    }


}

/**
 *
 */
package cn.jiiiiiin.security.app.server;

import lombok.val;
import org.springframework.security.oauth2.common.DefaultOAuth2AccessToken;
import org.springframework.security.oauth2.common.OAuth2AccessToken;
import org.springframework.security.oauth2.provider.OAuth2Authentication;
import org.springframework.security.oauth2.provider.token.TokenEnhancer;

import java.util.HashMap;
import java.util.Map;

/**
 * 扩展和解析JWT的信息
 *
 * 将自定义的信息加入到token中返回给客户端
 *
 * @author zhailiang
 * @author jiiiiiin
 */
public class TokenJwtEnhancer implements TokenEnhancer {

    /* (non-Javadoc)
     * @see org.springframework.security.oauth2.provider.token.TokenEnhancer#enhance(org.springframework.security.oauth2.common.OAuth2AccessToken, org.springframework.security.oauth2.provider.OAuth2Authentication)
     */
    @Override
    public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
        // 自定义token元信息
        final Map<String, Object> info = new HashMap<>();
        // TODO 测试
        info.put("company", "jiiiiiin");
        // 设置附加信息
        ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(info);
        return accessToken;
    }

}


/**
 *
 */
package cn.jiiiiiin.security.app.server;

import cn.jiiiiiin.security.core.properties.OAuth2ClientProperties;
import cn.jiiiiiin.security.core.properties.SecurityProperties;
import lombok.val;
import org.apache.commons.lang3.ArrayUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.oauth2.config.annotation.builders.InMemoryClientDetailsServiceBuilder;
import org.springframework.security.oauth2.config.annotation.configurers.ClientDetailsServiceConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configuration.AuthorizationServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerEndpointsConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerSecurityConfigurer;
import org.springframework.security.oauth2.provider.OAuth2Authentication;
import org.springframework.security.oauth2.provider.token.TokenEnhancer;
import org.springframework.security.oauth2.provider.token.TokenEnhancerChain;
import org.springframework.security.oauth2.provider.token.TokenStore;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;

import java.util.ArrayList;
import java.util.List;

/**
 * 认证服务器配置
 * <p>
 * <p>
 * ![](https://ws4.sinaimg.cn/large/0069RVTdgy1fuqnerfq5uj30w10g3q4o.jpg)
 * <p>
 * 上图也是下面的授权模式中，使用最多的授权码模式的交互流程；
 * <p>
 * ![](https://ws1.sinaimg.cn/large/0069RVTdgy1fuqn6o8vm2j30ox0bm3z0.jpg)
 *
 * <p>
 * https://oauth.net/2/
 * <p>
 * 4.1.  Authorization Code Grant
 * <p>
 * The authorization code grant type is used to obtain both access
 * tokens and refresh tokens and is optimized for confidential clients.
 * Since this is a redirection-based flow, the client must be capable of
 * interacting with the resource owner's user-agent (typically a web
 * browser) and capable of receiving incoming requests (via redirection)
 * from the authorization server.
 * <p>
 * +----------+
 * | Resource |
 * |   Owner  |
 * |          |
 * +----------+
 * ^
 * |
 * (B)
 * +----|-----+          Client Identifier      +---------------+
 * |         -+----(A)-- & Redirection URI ---->|               |
 * |  User-   |                                 | Authorization |
 * |  Agent  -+----(B)-- User authenticates --->|     Server    |
 * |          |                                 |               |
 * |         -+----(C)-- Authorization Code ---<|               |
 * +-|----|---+                                 +---------------+
 * |    |                                         ^      v
 * (A)  (C)                                        |      |
 * |    |                                         |      |
 * ^    v                                         |      |
 * +---------+                                      |      |
 * |         |>---(D)-- Authorization Code ---------'      |
 * |  Client |          & Redirection URI                  |
 * |         |                                             |
 * |         |<---(E)----- Access Token -------------------'
 * +---------+       (w/ Optional Refresh Token)
 * <p>
 * Note: The lines illustrating steps (A), (B), and (C) are broken into
 * two parts as they pass through the user-agent.
 * <p>
 * Figure 3: Authorization Code Flow
 *
 * <p>
 * 添加了`@EnableAuthorizationServer`注解之后，项目就可以当做一个授权服务提供商，给第三方应用提供oauth授权服务
 * <p>
 * `/oauth/authorize::GET`接口来提供oauth流程第一步，提供第三方服务获取“授权码”
 * <p>
 * 所需参数：
 * response_type
 * REQUIRED.  Value MUST be set to "code".
 * <p>
 * client_id
 * REQUIRED.  The client identifier as described in Section 2.2.
 * <p>
 * redirect_uri
 * OPTIONAL.  As described in Section 3.1.2.
 * <p>
 * <p>
 * `/oauth/authorize::GET`接口来提供oauth流程第一步，提供第三方服务获取“授权码”
 * <p>
 * `/oauth/token::POST`接口来提供oauth流程第而步，第三方服务通过“授权码”来获取授权令牌
 * <p>
 * 关于自定义生成Token
 * <p>
 * 当继承了{@link AuthorizationServerConfigurerAdapter}之后默认就不会生成默认的`clientId`和`secret`
 *
 * @author zhailiang
 * @see org.springframework.security.oauth2.provider.endpoint.TokenEndpoint
 */
@Configuration
@EnableAuthorizationServer
public class CustomAuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    /**
     * 当去做认证的时候使用的`userDetailsService`
     */
    @Autowired
    private UserDetailsService userDetailsService;

    /**
     * 当去做认证的时候使用的`authenticationManager`
     */
    @Autowired
    private AuthenticationManager authenticationManager;

    /**
     * 负责令牌的存取
     */
    @Autowired
    private TokenStore tokenStore;

    /**
     * 自有{@link TokenStore}使用jwt进行存储时候生效
     */
    @Autowired(required = false)
    private JwtAccessTokenConverter jwtAccessTokenConverter;

    @Autowired(required = false)
    private TokenEnhancer jwtTokenEnhancer;

    @Autowired
    private SecurityProperties securityProperties;

    /**
     * 认证及token配置
     * 定义token增强器来自定义token的生成策略，覆盖{@link org.springframework.security.oauth2.provider.token.DefaultTokenServices}默认的UUID生成策略
     *
     * @see org.springframework.security.oauth2.provider.token.DefaultTokenServices#createAccessToken(OAuth2Authentication)
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        // 当继承了`AuthorizationServerConfigurerAdapter`之后就需要自己配置下面的认证组件
        endpoints
                .tokenStore(tokenStore)
                .authenticationManager(authenticationManager)
                .userDetailsService(userDetailsService);
        // jwtAccessTokenConverter将token生成策略改成jwt
        if (jwtAccessTokenConverter != null) {
            val enhancerChain = new TokenEnhancerChain();
            List<TokenEnhancer> enhancers = new ArrayList<>();
            if (jwtTokenEnhancer != null) {
                // jwtTokenEnhancer向jwt token中订制自定义数据
                enhancers.add(jwtTokenEnhancer);
            }
            enhancers.add(jwtAccessTokenConverter);
            enhancerChain.setTokenEnhancers(enhancers);
            endpoints.tokenEnhancer(enhancerChain).accessTokenConverter(jwtAccessTokenConverter);
        }
    }




```





测试：

![image-20180917163047090](https://ws2.sinaimg.cn/large/006tNbRwgy1fvcmcgomy4j31kw0qwqi6.jpg)

![image-20180917163132472](https://ws3.sinaimg.cn/large/006tNbRwgy1fvcmdbaipzj31kw16ldon.jpg)

通过https://jwt.io/进行解析`access_token`得到的payload就是我们创建的token；

**如果使用jwt方式，那么框架会根据jwt的payload重新组装一个 {@link Authentication}对象，直接获取 {@link UserDetails}将得不到内容**



扩展和解析JWT的信息



![image-20180917165238883](https://ws3.sinaimg.cn/large/006tNbRwgy1fvcmz6uofmj31kw0lgwl7.jpg)

##### 



#### 基于JWT实现SSO（单点登录）



> [基于Jwt实现sso](https://coding.imooc.com/lesson/134.html#mid=7238)

##### sso流程

![image-20180918160642584](https://ws3.sinaimg.cn/large/006tNbRwgy1fvdr9ru6mtj30qo0gbdi2.jpg)

用户只用在应用A进行授权时候在认证服务器端登录，在应用B进行授权时候就不用登录；

![image-20180918161151717](https://ws1.sinaimg.cn/large/006tNbRwgy1fvdrf1le1oj30y50gc43g.jpg)

如果应用B是一个前后端分离的应用，在拿到JWT之后，再去对应的资源服务器进行请求就可以了。


## 其他

### RESTFul API

![RESTFul API](https://ws3.sinaimg.cn/large/006tNbRwgy1fufeoc5gxdj31kw0yswnl.jpg)

#### 异步处理 REST 服务

+ 使用 Callable 来处理

  ![](<(https://ws4.sinaimg.cn/large/006tNbRwgy1fuhcqkylckj31kw0rm0v2.jpg)>)

  如果使用异步方式处理请求，那么请求就不占用容器的【主线程】的资源数，使得容器可以处理更多请求而不被阻塞，提升服务器的吞吐量；
  对于浏览器来说，还是一个【正常】的请求，因为耗时还是 1.x 秒得到响应；

  特点：
  子线程必须是在主线程中开启

  - 有一种情况：

  ![](https://ws1.sinaimg.cn/large/006tNbRwgy1fuhdliloxxj30x40ezmxz.jpg)

  即接收请求的服务器是一个前置服务器，真正进行耗时操作的是外围一台服务器， 应用 1 只负责接收消息，之后将消息放到消息队列，而真正处理业务是在应用 2 中去监控消息队列，取出并处理，处理完毕之后将结果返给消息队列，应用 1 在实时的去取结果；
  而真正的请求的响应式在应用 1 从队列中取出处理结果之后返回给前端的；

  针对这种场景，上面的处理方式就不能满足需求，需要使用 DeferredResult 来进行处理，参考`@RequestMapping("/async/order2")处理逻辑

  还需要注意：

  ```java
      /**
       * 针对异步接口的拦截器配置需要通过下面的接口进行注册（相应的拦截器也需要重写）
       * 否则常规的拦截器是拦截不到异步的接口
       *
       * @param configurer
       */
      @Override
      public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
          super.configureAsyncSupport(configurer);
  //        configurer.registerDeferredResultInterceptors()
  //        configurer.registerCallableInterceptors()
      }
  ```


### [kaptcha 集成](https://www.jianshu.com/p/1f2f7c47e812)

### 使用 Swagger 自动生成文档

  根据代码自动生成文档，提供给前端开发人员识别接口；

  [集成](https://mvnrepository.com/artifact/io.springfox/springfox-swagger2)

  ![](https://ws1.sinaimg.cn/large/006tNbRwgy1fui7xqrglgj31kw0ocab6.jpg)

  ```java
      @SpringBootApplication
      @RestController
      @EnableSwagger2
      public class App {}

    // 常用注解：@ApiParam、@ApiOperation

      @GetMapping("/{id:\\d+}")
      @JsonView(User.UserDetailView.class)
      @ApiOperation(value = "用户查询服务")
      public User getUserInfo(@ApiParam(value = "用户id") @PathVariable String id, @PathVariable(name = "id") Long idddd) {

    // 常用注解：@ApiModelProperty
    public class UserQryCondition {

        private String username;
        @ApiModelProperty(value = "年龄起始值")
        private int age;
        // 年龄区间
        @ApiModelProperty(value = "年龄终止值")
  ```

### springboot aop、拦截器、过滤器相关:

  ![springboot aop、拦截器、过滤器相关](https://ws3.sinaimg.cn/large/006tNbRwgy1fuh5paluivj30n90glgm0.jpg)

  `@ControllerAdvice`就是如控制器异常处理类上面声明的注解；

  如果控制器中抛出的异常，在外围还是抛出的话，那就会进行层层传递；

  如果 filter 还没有处理异常，就会抛到容器（如：tomcat）最终显示到前端；

### springboot 默认错误处理控制器 `org.springframework.boot.autoconfigure.web.BasicErrorController`

  ```java
  // /error是默认错误视图的路径
  @Controller
  @RequestMapping("${server.error.path:${error.path:/error}}")
  public class BasicErrorController extends AbstractErrorController {

      // 表示请求参数头中的Accept被认定为html的时候进入该接口进行错误处理
      @RequestMapping(produces = "text/html")

      // 否则进入：
      @RequestMapping
      @ResponseBody
      public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
  ```

  手动处理 404 网页版：需要在`/jiiiiiin-security/jiiiiiin-security-demo/src/main/resources/resources/error/404.html`创建对应状态码的页面；
  这样对应状态码的网页版的错误页面就可以被订制；

## 问题集合：

### spring security 常见错误：

#### signInUrl 配置：

  ```java
    .signInUrl(LOGIN_URL)
  ```

  这里的配置如果需要使用控制器来做渠道控制（渲染登录页面或者返回 json 提示），那么接口的名称不要和被渲染的页面名称一致，比如接口【signIn】而页面名称【signIn.html】这样会导致视图解析器报错

#### 默认开启了 csrf，但是没有做合理配置，登录的时候就报：
    ```xml
    There was an unexpected error (type=Forbidden, status=403).
    Could not verify the provided CSRF token because your session was not found.
    ```
    调试期间可以先关闭这个特性：
    ```java
      .and()
            .csrf().disable();
    ```

**需要先创建对应数据库，启动 redis 服务**

### [redis 安装](https://www.youtube.com/watch?v=JGvbEk4jtrU)

### `Cannot determine embedded database driver class for database type NONE`：

  因为在`jiiiiiin-security-core`中声明了 jdbc 依赖，但是没有配置数据库链接配置则会报错：

  ```xml
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-jdbc</artifactId>
          </dependency>
  ```

  解决，配置 jdbc 链接配置信息：

  ```properties
  spring:
    datasource:
      driver-class-name: com.mysql.jdbc.Driver
  #    http://database.51cto.com/art/201005/199278.htm
      url: jdbc:mysql://127.0.0.1:3306/jiiiiiin-security-demo?&useUnicode=true&characterEncoding=utf8&autoReconnect=true&failOverReadOnly=false
      username: root
      password: root
  ```

### `Caused by: java.lang.IllegalArgumentException: No Spring Session store is configured: set the 'spring.session.store-type' property`
  因为`jiiiiiin-security-browser`中声明了 spring session 的依赖，但是没有配置相关依赖配置故报错：
  解决先关闭依赖配置：
  ```properties
  spring:
    session:
      store-type: none
  ```
- 项目启动访问需要“登录信息”，因为 spring security 默认开启认证，可以先关闭默认配置：

  ```properties
  security:
    basic:
      enabled: false
  ```

### 怎么将 demo 项目打出可执行 jar：

  解决：

  ```xml
       <plugins>
          <!--将项目打包成一个可执行的jar https://docs.spring.io/spring-boot/docs/1.5.15.RELEASE/reference/htmlsingle/#getting-started-first-application-executable-jar-->
          <plugin>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-maven-plugin</artifactId>
              <executions>
                  <execution>
                      <goals>
                          <goal>repackage</goal>
                      </goals>
                  </execution>
              </executions>
          </plugin>

      </plugins>
  ```

### `java.lang.AssertionError: No value at JSON path "$.length()": java.lang.IllegalArgumentException: json can not be null or empty`

  因为测试的方法没有返回正确的 json 数据；

  - postman

    - [Postman 高级应用（1）：生成 cURL 和多语言代码](https://blog.csdn.net/qq598535550/article/details/80889843)
