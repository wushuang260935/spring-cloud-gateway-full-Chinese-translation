# spring cloud gateway(网关)

spring cloud gateway提供了一个API网关.这个网关以spring生态系统为基础.他们是:Spring5,Spring boot2和Project reactor。
spring cloud gateway的目标是提供一个简单的,高效的方式。这个方式能够路由到我们写的服务API上 ,spring cloud gateway还能为开发
人员提供横切关注点(cross cutting concern  有点类似与AOP的那种概念) 例如，我们可以利用spring cloud gateway做安全过滤(security)，
监控(monitoring),统计(metrics),跳转(resiliency).

## 如何添加spring cloud gateway 到项目

想要添加spring cloud gateway到你的项目上,就要用org.springframework.cloud的starter集合(groupid)。artificat id:spring-cloud-start-gateway.
但如果你引进了这个starter。却因为某种原因不想要网管生效就可以这样设置:spring.cloud.gateway.enabled=false.

> 由于spring cloud gateway建立在spring boot2.0,spring webflux和spring reactor上。所以当你使用spring cloud gateway的时候，里面的有些东西你可能不熟悉。
（比如spring data和spring security)。为了你能更好地理解spring cloud gateway，我们建议你先熟悉他们再来看spring cloud gateway的文档。

> spring cloud gateway需要运行在netty 框架上(netty框架由spring boot和spring webflux提供).而不是运行在传统的servlet容器中，或者打成war包运行。

## 相关术语解释

路由(Route)：
    路由是gateway的基本模块。它由一个ID，一个请求URI，一个判断集(collection of predicates),一个过滤集(collection of filters).
如果判断集都返回true。那么这个路由就会请求成功。

判断(predicate)：
  “判断”是jdk8中Predicate的一个实际运用。入参是springframework的ServerWebExchange类型.使用这个类型可以允许开发人员传递HTTP请求的任意部分.
比如:http请求的heads,http请求的parameters.

过滤器(filter):
  "过滤器"是GatewayFilter接口的实例对象。“过滤器”由具体的工厂类负责实例化。

## 工作原理

![网关请求流程图](https://raw.githubusercontent.com/spring-cloud/spring-cloud-gateway/master/docs/src/main/asciidoc/images/spring_cloud_gateway_diagram.png)

客户端向spring cloud gateway发送请求。如果网关的handler mapping 发现这个请求和一个路由匹配,这个请求就会被发送到网关的web handler.handler再把请求发送给filter chain中。上图中filter之间之所以用虚线连接，是因为filter有可能先于代理请求发送过来之前就开始逻辑执行(这个就是前面讲的"判断")。当“判断”逻辑执行完成之后，才生成代理请求。代理请求发送给filter。最后才是过滤处理.

> 如果URI没有端口，网关会默认给他80（http）或者443（https）

## 路由判断工厂

网关把URI路由成spring webflux的handlermapping的一部分。网关包含了很多内嵌的路由判断工厂（也就是说这些工厂是内部类).不同的工厂会去处理不同的http属性。多个路由判断工厂可以通过逻辑"and"组合到一起使用，下面开始介绍具体的路由判断工厂

### 后置路由判断工厂

后置路由判断工厂只需要一个日期时间类型的参数。这个工厂生成的“判断”可以匹配那些发生在日期时间参数之后的请求。如下方在application.yml中配置所示:这个“判断”可以匹配任何“请求时间”晚于2017年1月20号17点42分47秒的请求。

spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]


### 区间路由判断工厂

和上一个类似，区间路由判断工厂也需要两个日期时间类型参数。左边的参数必须小于右边的参数。这个工厂生成的“判断”可以匹配任何请求时间介于这两个参数之间的请求。application.yml配置如下方所示:


spring:
  cloud:
    gateway:
      routes:
      - id: between_route
        uri: https://example.org
        predicates:
        - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
        
### 前置路由判断工厂

前置路由判断工厂也只需要一个日期时间类型参数,同样的，“判断”可以匹配任何请求时间发生在参数之前的请求。application.yml如下所示:
spring:
  cloud:
    gateway:
      routes:
      - id: before_route
        uri: https://example.org
        predicates:
        - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
        
### cookie路由判断工厂
        
这个工厂需要两个参数，一个是cookie名称，另一个是正则表达式.也就是说，这个工厂生成的“判断”可以匹配到含有以下cookie的请求:cookie中的name=第一个参数,cookie中的value通过第二个参数的验证。application.yml如下所示:只要请求中的cookie的name=chocolate并且cookie的value能通过正则表达式"ch.p"验证.那么这个请求就会被匹配到。

spring:
  cloud:
    gateway:
      routes:
      - id: cookie_route
        uri: https://example.org
        predicates:
        - Cookie=chocolate, ch.p
        
### 头部路由判断工厂

和上一个路由工厂类似，头部路由判断工厂也需要两个参数.一个头部属性和一个正则表达式，也就是说，如果一个请求中header有一个属性=第一个参数,并且这个属性的值能够通过第二个参数的验证，那么这个请求就会被匹配到。application.yml如下所示:
请求的header中包含X-Request-Id属性，并且这个属性的值能通过"\d+"的验证，那么这个请求就会被匹配到.

spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: https://example.org
        predicates:
        - Header=X-Request-Id, \d+
        
### Host(主机)路由判断工厂

主机路由判断工厂需要一个参数:一连串域名，域名之间用逗号隔开,域名必须有.号和后缀,同理,如果请求中的host和参数中的仁一个域名相同，这个请求就会被匹配到,application.yml如下所示:需要注意的是,www.somehost.org.sub.somehost.org,

spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Host=**.somehost.org,**.anotherhost.org
