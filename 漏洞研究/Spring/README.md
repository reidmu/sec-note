# 基础知识
## Spring
Spring一般指的是Spring Framework，一个轻量级Java应用程序开源框架，提供了简易的开发方式。 Spring 框架是一个轻量级的控制反转(IoC)和面向切面(AOP)的容器框架，大大简化了软件开发复杂性，它现在是 Java 后端框架家族里面最强大的一个。
并且，Spring 现在能与所有主流开发框架集成，相当于一个万能框架，Spring 让 JAVA 开发变得更简单。

请求流程如下：
- 用户发送请求给服务器
- 服务器收到请求，使用DispatchServlet处理
- Dispatch使用HandleMapping检查url是否有对应的Controller，如果有，执行
- 如果Controller返回字符串，ViewResolver将字符串转换成相应的视图对象
- DispatchServlet将视图对象中的数据，输出给服务器
- 服务器将数据输出给客户端

## Spring MVC
Spring MVC 是一个 MVC 开源框架，用来代替 Struts。它是 Spring 项目里面的一个重要组成部分，能与 Spring IOC 容器紧密结合，以及拥有松耦合、方便配置、代码分离等特点，让 JAVA 程序员开发 WEB 项目变得更加容易。 

## Spring Boot
- [SpringBoot主要知识](https://github.com/ZXZxin/SpringBoot)

Spring Boot 是 Spring 开源组织下的一个子项目，也是 Spring 组件一站式解决方案，主要是为了简化使用 Spring 框架的难度，简省繁重的配置。
Spring Boot提供了各种组件的启动器（starters），开发者只要能配置好对应组件参数，Spring Boot 就会自动配置，让开发者能快速搭建依赖于 Spring 组件的 Java 项目。

## Spring Cloud
- [spring-gateway-demo](https://github.com/wdahlenburg/spring-gateway-demo)

Spring Cloud 是一系列框架的有序集合，是目前最火热的微服务框架首选，它利用 Spring Boot 的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用 Spring Boot 的开发风格做到一键启动和部署。
框架分析。


# 相关漏洞
- [Spring漏洞总结](https://si1ent.xyz/2021/06/28/Spring%E6%BC%8F%E6%B4%9E%E5%90%88%E9%9B%86/)
- [技术分享|Spring Boot环境下的渗透总结 @四叶草安全](https://mp.weixin.qq.com/s/lbMH68OeInb-cVMiubCpKw)
- [Spring Boot 相关漏洞学习资料，利用方法和技巧合集，黑盒安全评估 check list @LandGrey](https://github.com/LandGrey/SpringBootVulExploit)
- [Spring Boot信息泄露](https://blog.csdn.net/weixin_45039616/article/details/106637978)
- [Springboot heapdump信息泄露以及MAT分析](https://www.cnblogs.com/snowie/p/15561081.html)
  - [命令行工具heapdump_tool](https://github.com/wyzxxz/heapdump_tool)

## CVE-2016-4977 Spring Security OAuth SpEL RCE
- 漏洞描述：Spring Security OAuth 是为 Spring 框架提供安全认证支持的一个模块。在其使用 whitelabel views 来处理错误时，由于使用了Springs Expression Language (SpEL)，攻击者在被授权的情况下可以通过构造恶意参数来远程执行命令。
- 影响范围：
  - 2.0.0 - 2.0.91.0.0 - 1.0.5
- 漏洞原理：Spring的一个错误页面，对用户传的参数的递归解析，从而导致SpEL注入；最终获得服务器权限。
- 漏洞条件：用户名、密码

## CVE-2017-4971 Spring WebFlow SpEL RCE
- 漏洞描述：Spring WebFlow 是一个适用于开发基于流程的应用程序的框架（如购物逻辑），可以将流程的定义和实现流程行为的类和视图分离开来。在其 2.4.x 版本中，如果我们控制了数据绑定时的field，将导致一个SpEL表达式注入漏洞，最终造成任意命令执行。
- 影响范围：
  - Spring WebFlow 2.4.0 - 2.4.4
- 漏洞原理：如控制了数据绑定时的field，将导致一个SpEL表达式注入漏洞，最终造成任意命令执行。
- 漏洞条件：用户名、密码

## CVE-2017-8046 Spring Data REST SpEL RCE
- 漏洞描述：Spring Data REST是一个构建在Spring Data之上，为了帮助开发者更加容易地开发REST风格的Web服务。在REST API的Patch方法中（实现RFC6902），path的值被传入setValue，导致执行了SpEL表达式，触发远程命令执行漏洞。恶意攻击者使用精心构造的JSON数据包，向spring-data-rest服务器提交恶意PATCH请求，可以执行任意的Java代码。
- 影响范围：
  - Spring Data REST 2.5.12, 2.6.7, 3.0 RC3之前的版本
  - Spring Boot 2.0.0M4之前的版本
  - Spring Data release trains Kay-RC3之前的版本
- 漏洞原理：path的值被传入setValue，导致执行了SpEL表达式，触发远程命令执行漏洞。
- 漏洞条件：无

## CVE-2018-1273 Spring Data SpEL RCE
- 漏洞描述：Spring Data是一个用于简化数据库访问，并支持云服务的开源框架，Spring Data Commons是Spring Data下所有子项目共享的基础框架。Spring Data Commons 在2.0.5及以前版本中，存在一处SpEL表达式注入漏洞，攻击者可以注入恶意SpEL表达式以执行任意命令。
- 影响范围：
  - Spring Data Commons 1.13 - 1.13.10 (Ingalls SR10)
  - Spring Data REST 2.6 - 2.6.10 (Ingalls SR10)
  - Spring Data Commons 2.0 to 2.0.5 (Kay SR5)
  - Spring Data REST 3.0 - 3.0.5 (Kay SR5)及更早的版本也会受到影响
- 漏洞原理：Spring Data Commons 在2.0.5及以前版本中，存在一处SpEL表达式注入漏洞，攻击者可以注入恶意SpEL表达式以执行任意命令。
- 漏洞条件：无

## CVE-2022-22947 Spring Cloud Gateway SpEL RCE
- 漏洞描述：Spring Cloud Gateway 是基于 Spring Framework 和 Spring Boot 构建的 API 网关，它旨在为微服务架构提供一种简单、有效、统一的 API 路由管理方式。
该漏洞为当Spring Cloud Gateway启用和暴露 Gateway Actuator 端点时，使用 Spring Cloud Gateway 的应用程序可受到代码注入攻击。攻击者可以发送特制的恶意请求，从而远程执行任意代码。
- 影响范围：
  - Spring Cloud Gateway 3.1.x < 3.1.1
  - Spring Cloud Gateway 3.0.x < 3.0.7
  - 其他旧的、不受支持的SpringCloudGateway版本
- 漏洞原理：Spring Cloud GateWay支持通过Actuator端点对网关进行监控和交互，其中还可以删除和创建特定的路由。`filter`参数可控，导致可以将Filter的配置属性传入`normalize`中,最后进入`getValue`执行SPEL表达式造成SPEL表达式注入。
- 漏洞条件：Spring Cloud Gateway启用、暴露 Gateway Actuator 端点

## CVE-2022-22963 Spring Cloud Function SpEL表达式注入
- 漏洞描述：Spring Cloud Function 是基于Spring Boot 的函数计算框架（FaaS），该项目提供了一个通用的模型，用于在各种平台上部署基于函数的软件，包括像 Amazon AWS Lambda 这样的 FaaS（函数即服务，function as a service）平台。它抽象出所有传输细节和基础架构，允许开发人员保留所有熟悉的工具和流程，并专注于业务逻辑。 在版本3.0.0到当前最新版本3.2.2(commit dc5128b)，默认配置下，都存在Spring Cloud Function SpEL表达式注入漏洞。
- 影响范围：
  - 3.x<=v3.2.2
- 漏洞条件
  - 需要修改配置+任意路由
    - 在`application.properties`设置中添加`spring.cloud.function.definition:functionRoute`
  - 默认配置+特定路由
    - 保持`application.properties`默认配置，特定路由`/functionRouter`存在SpEL表达式注入
