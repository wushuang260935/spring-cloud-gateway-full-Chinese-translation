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

### 回滚头网关过滤器工厂

这个工厂允许你添加“断路器执行异常”的细节到请求头中。这里也不详细说明。

### 前缀网关过滤器工厂

这个工厂只需要一个prefix参数,如下图所示:通过这个过滤器的所有请求hui加上prefix参数作为前缀，例如:/hello就会变成/mypath/hello

spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - PrefixPath=/mypath
     
### 保留主机头网关过滤器工厂

这个工厂不需要参数,当收到请求后，保留主机头网关过滤器会设置一个request attribute(请求属性).这个属性会告诉路由过滤器尽量发送原始host header，而不是由http客户端的host header.

spring:
  cloud:
    gateway:
      routes:
      - id: preserve_host_route
        uri: https://example.org
        filters:
        - PreserveHostHeader
        
### 请求频率限制器过滤器工厂

请求频率限制器过滤器工厂使用RateLimiter实例来决定当前请求是否可以继续。如果不能，那么就会返回http429(too many request).请求频率限制器路由器有一个可选参数,KeyResolver(这是一个复杂数据类型，继承自KeyResolver接口)。默认的接口实例是PrincipalNameKeyResolver.这个实例会从serverWebExchange检索Principal并且调用Principal.getName()方法.
默认情况下,如果keyResolver没找到key，请求就会被拒绝，	实际上，是否使用key是可以配置的：spring.cloud.gateway.filter.request-rate-limiter.deny-empty-key (true or false) 和 spring.cloud.gateway.filter.request-rate-limiter.empty-key-status-code properties.

#### Redis 频率限制器

使用Redis 频率限制器需要spring-boot-starter-data-redis-reactive.

### 重定向网关过滤器工厂

这个工厂需要一个status和一个url参数。status限定填写http 300系列,url限定填写合法url.application.yml如下所示:

spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - RedirectTo=302, https://acme.org
        
### 通过hop头过滤器移除hop网关过滤器工厂

这个工厂会移除转发请求的部分头部属性。默认被移除的头部信息来自于IETF清单,他们是:

Connection

Keep-Alive

Proxy-Authenticate

Proxy-Authorization

TE

Trailer

Transfer-Encoding

Upgrade
        
如果你想修改这个清单，请设置spring.cloud.gateway.filter.remove-non-proxy-headers.headers属性

### 移除请求头网关过滤器工厂

这个工厂需要name参数,他是即将被移除的header属性名.application.yml如下所示:需要注意的是，移除操作是在downstream(响应后逻辑处理)之前

spring:
  cloud:
    gateway:
      routes:
      - id: removerequestheader_route
        uri: https://example.org
        filters:
        - RemoveRequestHeader=X-Request-Foo

### 移除响应头网关过滤器工厂

这个工厂需要一个name参数，他是即将被移除的header属性名，application.yml如下所示:需要注意的是，移除操作发生在响应到达网关客户端之前。如果你想配置一次所有地方都可以使用的话，就可以使用spring.cloud.gateway.default-filters属性.

### 重写路径网关过滤器工厂

这个工厂需要一个正则表达式参数和一个replacement参数.使用java正则表达式筛选出需要被重写的路径:application.yml如下所示:对于这个/foo/bar请求。过滤器将会把这个url变成/bar。然后再去downstream(响应后逻辑处理).注意这个$\属性，yaml格式的转义字符也是\.所以他是为了$字符串

spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://example.org
        predicates:
        - Path=/foo/**
        filters:
        - RewritePath=/foo/(?<segment>.*), /$\{segment}

### 重写响应头网关过滤器工厂

这个工厂需要name参数，正则表达式参数，replacement参数，这里应该使用java正则表达式。application.yml如下所示:header中 name=X-Response-Foo的属性(后面的，表示支持多个name），regexp="password=[^&]+"，replacement="password=***"。如果这个请求属性是这样的:X-Response-Foo:user=admin&password=123456,那么这个过滤器就会把他变成X-Response-Foo:user=admin&password=***

### 保存绘画网关过滤器工厂

在进行downstream之前，这个工厂会强制做session.save()操作.当使用spring session的时候会非常有用。特别是在做转发请求之前，需要保证session状态已经被保存的时候。还有就是，有懒数据加载的时候。

spring:
  cloud:
    gateway:
      routes:
      - id: save_session
        uri: https://example.org
        predicates:
        - Path=/foo/**
        filters:
        - SaveSession
        
如果你想整合spring session到spring security中。在某些细节处理上，这个过滤器就很重要了。

### 安全头部网关过滤器工厂

这个工厂添加一定数量的头部属性到response中，这些属性来自一个博客推送:https://blog.appcanary.com/2017/http-security-headers.html。属性列表如下:

X-Xss-Protection:1; mode=block

Strict-Transport-Security:max-age=631138519

X-Frame-Options:DENY

X-Content-Type-Options:nosniff

Referrer-Policy:no-referrer

Content-Security-Policy:default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src https:; style-src 'self' https: 'unsafe-inline'

X-Download-Options:noopen

X-Permitted-Cross-Domain-Policies:none

如果想修改这些属性，请设置spring.cloud.gateway.filter.secure-headers

### 设置路径网关过滤器工厂

这个工厂需要一个路径template参数.这个模板提供一种简单地修改请求路径的方式:允许模板分隔符。有了模板分隔符，那么不需要修改的地方，就可以使用模板分隔符表示，而某些确定的地方就被修改了。application.yml如下所示:如果uri是：/foo/bar，/foo/scde,/foo/wer，那么久会被修改成,bar,/scde,/wer

spring:
  cloud:
    gateway:
      routes:
      - id: setpath_route
        uri: https://example.org
        predicates:
        - Path=/foo/{segment}
        filters:
        - SetPath=/{segment}
        
### 设置响应头网关过滤器工厂

这个工厂需要name参数和value参数。他会把新的name:value替换旧的name:value。application.yml如下所示:加入原来的X-Response-Foo:1234,那么就会替换为X-Response-Foo:Bar

spring:
  cloud:
    gateway:
      routes:
      - id: setresponseheader_route
        uri: https://example.org
        filters:
        - SetResponseHeader=X-Response-Foo, Bar
        
### 设置状态网关过滤器工厂

这个工厂需要一个status参数，它是一个Spring的HttpStatus。默认是404或者枚举类型NOT_FOUND.使用这个过滤器，所有的请求响应状态都会设置成status.application.yml如下所示:所有的响应都是401

spring:
  cloud:
    gateway:
      routes:
      - id: setstatusstring_route
        uri: https://example.org
        filters:
        - SetStatus=BAD_REQUEST
      - id: setstatusint_route
        uri: https://example.org
        filters:
        - SetStatus=401
        
### 剥离前缀网关过滤器工厂

这个工厂需要一个parts参数，他是一个数字，路径的第几个部分会被剥离。这个操作发生在请求downstream之前。application.yml如下所示：从左到右下标为2的路径bar将会被去掉。运来的路径:https://nameservice/name/bar/foo,过滤器操作过后的路径:https://nameservice/name/foo.

spring:
  cloud:
    gateway:
      routes:
      - id: nameRoot
        uri: https://nameservice
        predicates:
        - Path=/name/**
        filters:
        - StripPrefix=2        

### 重试网关过滤器工厂

这个工厂需要四个参数:

>retries:尝试重试的次数

>statuses:需要被重试的http状态码，(org.springframework.http.HttpStatus) 

>methods:需要被重试的http方法.(org.springframework.http.HttpMethod)

>series:org.springframework.http.HttpStatus.Series

spring:
  cloud:
    gateway:
      routes:
      - id: retry_test
        uri: http://localhost:8080/flakey
        predicates:
        - Host=*.retry.com
        filters:
        - name: Retry
          args:
            retries: 3
            statuses: BAD_GATEWAY
            
>这个过滤器并不支持有body的请求。

>使用这个过滤器的时候，如果某个请求是转发请求，请注意这个请求的返回类型。

### 请求大小网关过滤器工厂

这个工厂需要一个RequestSize参数(可选的,没设置默认是5M）。当请求大小超过设置的阈值的时候。这个工厂的过滤器能够限制请求到达downstream服务。

spring:
  cloud:
    gateway:
      routes:
      - id: request_size_route
      uri: http://localhost:8080/upload
      predicates:
      - Path=/upload
      filters:
      - name: RequestSize
        args:
          maxSize: 5000000
          
如果请求大小超过阈值，那么过滤器就会设置header的errorMessage:413 Payload Too Large.    

### 修改请求体网关过滤器工厂

这个工厂还处于测试状态中，API有可能会有变化,这个过滤器用来修改请求体。修改操作发生在downstream之前。

>这个过滤器只能用java DSL配置

`@Bean`
`public RouteLocator routes(RouteLocatorBuilder builder) {`
`    return builder.routes()`
`        .route("rewrite_request_obj", r -> r.host("*.rewriterequestobj.org")`
`            .filters(f -> f.prefixPath("/httpbin")`
`                .modifyRequestBody(String.class, Hello.class, MediaType.APPLICATION_JSON_VALUE,`
`                    (exchange, s) -> return Mono.just(new Hello(s.toUpperCase())))).uri(uri))`
`        .build();`
`}`

`static class Hello {`
`    String message;`

`    public Hello() { }`

`    public Hello(String message) {`
`        this.message = message;`
`    }`

`    public String getMessage() {`
`        return message;`
`    }`

`    public void setMessage(String message) {`
`        this.message = message;`
`    }`
`}`

### 修改响应体网关过滤器工厂

这个工厂还处于测试状态中，API有可能会有变化,这个过滤器用来修改请求体。修改操作发生在响应发送给客户端之前。

>这个过滤器只能用java DSL配置

`@Bean`
`public RouteLocator routes(RouteLocatorBuilder builder) {`
`    return builder.routes()`
`        .route("rewrite_response_upper", r -> r.host("*.rewriteresponseupper.org")`
`            .filters(f -> f.prefixPath("/httpbin")`
`        		.modifyResponseBody(String.class, String.class,`
`        		    (exchange, s) -> Mono.just(s.toUpperCase()))).uri(uri)`
`        .build();`
`}`

### 默认过滤器

如果你想添加到适用于所有路由的过滤器，你可以使用spring.cloud.gateway.default-filters属性，这个属性需要一定数量的过滤器.application.yml如下所示:

spring:
  cloud:
    gateway:
      default-filters:
      - AddResponseHeader=X-Response-Default-Foo, Default-Bar
      - PrefixPath=/httpbin
      
## 全局过滤器

全局过滤器接口拥有和网关过滤器相同的签名，他们是一群特殊的过滤器，因为他们适用于所有路由。

### 全局过滤器和网关过滤器的关联顺序

当一个请求被匹配成功了之后，Filtering web handler 将会添加所有的全局过滤器实例到过滤器链中。而且还会添加与路由有关的网关过滤器实例到过滤器链中。这些过滤器是通过org.springframework.core.Ordered接口来排序的。你可以通过getOrder()方法设置排序顺序，也可以使用@Order注解.

由于spring cloud网关通过区分pre和post标签来过滤逻辑执行。pre是请求前执行，post是请求后执行。过滤器会把优先级最高的打上pre标签，优先级最低的打上Post标签。

`@Bean`
`@Order(-1)`
`public GlobalFilter a() {`
`    return (exchange, chain) -> {`
`        log.info("first pre filter");`
`        return chain.filter(exchange).then(Mono.fromRunnable(() -> {`
`            log.info("third post filter");`
`        }));`
    };`
`}`

`@Bean`
`@Order(0)`
`public GlobalFilter b() {`
`    return (exchange, chain) -> {`
`        log.info("second pre filter");`
`        return chain.filter(exchange).then(Mono.fromRunnable(() -> {`
`            log.info("second post filter");`
`        }));`
`    };`
`}`

`@Bean`
`@Order(1)`
`public GlobalFilter c() {`
`    return (exchange, chain) -> {`
`        log.info("third pre filter");`
`        return chain.filter(exchange).then(Mono.fromRunnable(() -> {`
`            log.info("first post filter");`
`        }));`
`    };`
`}`

### 转发路由过滤器

这个过滤器会在exchange属性：ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR中寻找uri,如果发现uri中有forward请求。（如:forward:/localhost),那么过滤器会使用dispatcherhandler处理请求。然后就用转发URL覆盖请求URL。并且把请求URL加到ServerWebExchangeUtils.GATEWAY_ORIGINAL_REQUEST_URL_ATTR属性列表中。

### 负载均衡客户端过滤器

这个过滤器会在exchange属性:ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR中找寻uri,如果这个uri是lb请求(ie lb://myservice)。那么过滤器会用LoadBalancerClientl来解析这个列子中的myservice。解析出这个负载均衡地址的真实主机和端口。然后用主机和端口替换myservice。不过过滤器还是会保存一下原来的uri。如例子中的:lb://myservice.把这个地址加到ServerWebExchangeUtils.GATEWAY_ORIGINAL_REQUEST_URL_ATTR属性中去。

这个过滤器除了在ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR中找寻uri之外，还会在ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR中找lb的uri。

spring:
  cloud:
    gateway:
      routes:
      - id: myRoute
        uri: lb://service
        predicates:
        - Path=/service/**

> 默认情况下， 过滤器如果在负载均衡器中没找某一个服务实例。那么会返回503.但是如果你想返回404.可以设置spring.cloud.gateway.loadbalancer.use404=true

>负载均衡器中返回的ServiceInstance有一个isSecure属性。这个属性的值会覆盖请求中的scheme的值.

### netty路由过滤器

如果ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR属性中有http或者https策略,那么这个过滤器就会运行。这个过滤器使用netty的HttpClient来创建downstream代理请求。代理请求发送过后返回的响应，将会被放到ServerWebExchangeUtils.CLIENT_RESPONSE_ATTR属性中。后面的过滤器会用到他们。

### netty写响应过滤器

如果在ServerWebExchangeUtils.CLIENT_RESPONSE_ATTR属性中发现了NettyResponse。这个过滤器将会运行。但是运行是有条件的，它会等到其他所有的过滤器执行完成，并且写完发给网关客户端的代理响应过后，再运行。这样能保证所有的响应都会检测到。

### 路由到请求URL过滤器

如果ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR属性中有Route对象的话，这个路由器就会运行。他会把当前请求的URI更新成Route对象里面的URI。然后把新的URI放到ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR属性中。如果这个URI有一个策略前缀,如:lb,ws.这些前缀就会被剥离并放到ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR属性中。以供后面的过滤器链使用。

### Websocket路由过滤器

如果ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR属性中有ws或者wss策略的话，这个过滤器就会运行。这个过滤器使用Spring Web Socket转发请求 

spring:
  cloud:
    gateway:
      routes:
      # SockJS route
      - id: websocket_sockjs_route
        uri: http://localhost:3001
        predicates:
        - Path=/websocket/info/**
      # Normwal Websocket route
      - id: websocket_route
        uri: ws://localhost:3001
        predicates:
        - Path=/websocket/**
        
### 网关度量过滤器

要使用这个过滤器，请添加spring-boot-starter-actuator pom依赖。然后只要spring.cloud.gateway.metrics.enabled不等于false,这个过滤器就会一直运行。

### 路由过后做交换

网关路由完了一个ServerWebExchange后。他就会添加一个gatewayAlreadyRouted 属性到exchange属性中。这样就标志着这个exchange被路由过了。有了这个标志，其他的路由过滤器就会被跳过。

实际上，跳过过滤器，还有其他方法来标志这个exchange已经被路由了。并且提供了方法来检查这个exchange是否被路由过。

>ServerWebExchangeUtils.isAlreadyRouted需要一个ServerWebExchange参数来检查这个exchange是否被路由过。

>ServerWebExchangeUtils.setAlreadyRouted标记ServerWebExchange对象被路由了。

## TLS/SSL

网关可以监听https请求，但请求必须这样配置才能被监听到:

server:
  ssl:
    enabled: true
    key-alias: scg
    key-store-password: scg1234
    key-store: classpath:scg-keystore.p12
    key-store-type: PKCS12
    
如果网关中有后管的请求，那么可以做一下设置，让网关信任所有的请求。

spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          useInsecureTrustManager: true
          
生产环境中不能使用useInsecureTrustManager。因为网关上会配置一系列的可信任证书。

spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          trustedX509Certificates:
          - cert1.pem
          - cert2.pem
          
### TLS握手

网关保存了客户连接池。这些连接池是用来路由到后管的。当他们使用https通信的时候，TLS就会握手，而且握手有过期时间。过期时间的配置如下:

spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          handshake-timeout-millis: 10000
          close-notify-flush-timeout-millis: 3000
          close-notify-read-timeout-millis: 0
          
## 配置

spring cloud 网关的配置是一群RouteDefinitionLocator的集合。默认情况下，PropertiesRouteDefinitionLocator 使用spring boot的@ConfigurationProperties机制加载属性。application.yml如下所示:

spring:
  cloud:
    gateway:
      routes:
      - id: setstatus_route
        uri: https://example.org
        filters:
        - name: SetStatus
          args:
            status: 401
      - id: setstatusshortcut_route
        uri: https://example.org
        filters:
        - SetStatus=401
        
某些情况下，使用配置文件就够了，但是生产环境的某些情况下使用额外的资源配置会更好,比如数据库.RouteDefinitionLocator被用来加载Redis,MongoDB,等的属性配置                           