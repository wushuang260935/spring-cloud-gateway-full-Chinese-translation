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

网关把URI路由成spring webflux的handlermapping的一部分。网关包含了很多内嵌的路由判断工厂（也就是说这些工厂是内部类).不同的工厂会去处理不同的http属性。多个路由判断工厂可以通过逻辑"and"联合