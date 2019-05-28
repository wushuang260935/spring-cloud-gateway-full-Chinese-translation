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

主机路由判断工厂需要一个参数:一连串uri，uri之间用逗号隔开,uri必须有.号和后缀,同理,如果请求中的uri和参数中的任一个域名相同，这个请求就会被匹配到,application.yml如下所示:需要注意的是:www.somehost.org.sub.somehost.org,mmi.somehost.org都匹配**.somehost.org.另外注意需要注意的是:"判断"也支持uri中存在变量路径.比如:{type}.somehost.org。还需要注意:"host路由判断"会把“变量路径”和"变量值"放到一个map中。“变量路径”作为key.“变量值”作为value。如:map.put("type","login")。并且，为了方便后面的gatewayFilter factories(网关过滤器工厂)使用这个map.它还会把这个map作为value，ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE作为key.放到ServerWebExchange.getAttributes()中.

spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Host=**.somehost.org,**.anotherhost.org,{type}.somehost.org

### 方式路由判断工厂

方式路由判断工厂需要一个参数:http请求方式.application.yml如下所示:

spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: https://example.org
        predicates:
        - Method=GET
     
### 子路径路由判断工厂

子路径路由判断工厂需要两个参数:一连串Spring Pathmathcer类型的子路径，以及一个选填参数:matchOptionalTrailingSeparator(。。。。分隔符)

spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Path=/foo/{segment},/bar/{segment}
        
和Host路由工厂完全一样，如果有变量路径就会放到map里面，以上就不再赘述了。

### 查询路由判断工厂

查询路由判断工厂需要两个参数:一个param和一个可选的正则表达式."判断”会匹配到请求的参数名等于param并且请求参数值能通过正则表达式验证的请求。

spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: https://example.org
        predicates:
        - Query=foo,ba./d
        
### 远程地址路由判断工厂

这个工厂需要一个list(长度不能小于1).list放入ipv4或者ipv6地址串,application.yml如下所示:

spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: https://example.org
        predicates:
        - RemoteAddr=192.168.1.1
        
### 修改远程地址的解析方式

默认情况下远程地址路由判断工厂使用请求中给我们的远程地址.但是如果我们的网关拿到的是代理服务器转发的请求的话.spring cloud gateway拿到的就不是客户端请求的真实IP,就会导致匹配失败。这种情况下，你可以设置一个自定义RemoteAddressResolver来修改远程地址的解析方式。spring cloud gateway自带一个解析器:XForwardedRemoteAddressResolver,这个解析器是基于 X-Forwarded-For header的.并且它有两个静态构造函数trustAll和maxTrustedIndex,第一个构造那函数生成的解析器总是从 X-Forwarded-For header中拿第一个IP地址.(通常x-forwarded-for中第一个IP地址是真实地址，自行了解http头的x-forwarded-for属性)；但是x-forwarded-for属性是可以被一些客户端修改的，所以他也有缺陷。而另一个构造函数maxTrustedIndex接受一个integer参数(它不能小于1）。这个参数代表的是受信任的ip地址数量。我们从下面的例子来说明:

如果x-forwarded-for属性中有这几个ip:0.0.0.1,0.0.0.2,0.0.0.3.也就表示这个请求解析出了3个ip地址。

而当maxTrustedIndex的值如下时，解析器就会解析到这几个地址:


maxTrustedIndex的参数值                                        解析结果

     0                           IllegalArgumentException 
     
     1                                   0.0.0.3
     
     2                                0.0.0.2,0.0.0.3
     
     3                            0.0.0.1,0.0.0.2,0.0.0.3
     
     4                            0.0.0.1,0.0.0.2,0.0.0.3
     
可以看到他是从右边向左边开始过滤的。

## 网关过滤器工厂

网关路由器中的路由过滤器允许通过某种方式来修改http request。和http response;路由过滤器适合处理一种特定的路由。spring cloud gateway包含了很多内嵌的(内部class)网关过滤器工厂。

### 添加请求头网关过滤器工厂

这个工厂需要两个参数:name和value.application.yml如下所示:这个过滤器将会添加属性名称为X-Request-Foo,属性值为Bar到请求头中.

spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        filters:
        - AddRequestHeader=X-Request-Foo, Bar
        
### 添加请求参数网关过滤器工厂

这个工厂需要两个参数:name和value.application.yml如下所示:这个过滤器将会添加属性名称为foo,属性值为bar到请求参数中.

### 添加返回头网关过滤器工厂

这个工厂需要两个参数:name和value.application.yml如下所示:这个过滤器将会添加属性名称为X-Response-Foo,属性值为Bar到返回头中.

spring:
  cloud:
    gateway:
      routes:
      - id: add_response_header_route
        uri: https://example.org
        filters:
        - AddResponseHeader=X-Response-Foo, Bar
        
### 消除重复响应头网关过滤器工厂

这个工厂需要一个name参数。name参数可以是一连串响应头信息，响应头之间用空格隔开,这个工厂还需要一个可选的strategy参数。application.yml如下所示:有时候spring cloud gateway的CORS logic(跨域请求逻辑处理）和downstream(响应后逻辑处理）都会添加Access-Control-Allow-Credentials响应头和Access-Control-Allow-Origin响应头，这就会导致响应头属性重复。所以这个过滤器可以去掉指定的重复的部分。至于去掉的是哪一个重复的就可以看第二个参数(strategy)了。第二个参数实际只有3个值,RETAIN_FIRST(默认，表示只保留第一个),RETAIN_LAST(表示只保留最后一个),RETAIN_UNIQUE(表示只保留属性值不一样的那一个)

spring:
  cloud:
    gateway:
      routes:
      - id: dedupe_response_header_route
        uri: https://example.org
        filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin,RETAIN_UNIQUE

这个过滤器不只是处理上面例子中的跨域请求的重复问题。它其实可以处理所有响应头属性重复的问题。
        
### 断路器网关过滤器工厂

“断路器”是一个实现了netflix(奈飞)的"破除闭环者"模式的库。"断路器网关过滤器"允许你引入"破除闭环者"到网关路由过程中。防止发生连环故障(主要是spring cloud 服务集群的时候某一个服务挂了，会影响其他有关联的服务，可能导致整个服务瘫痪)，还有一个功能就是:一旦downstream(响应后逻辑处理)失败，还允许你返回应急响应给客户端。

要使用这个工厂，你需要从spring cloud Netflix中加入spring-cloud-starter-netflix-hystrix.这里主要是基于netflix的spring cloud，所以我就不做详细研究了。请查看英文文档了解更多。

       
        