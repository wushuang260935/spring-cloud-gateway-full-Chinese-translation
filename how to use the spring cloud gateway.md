# 创建一个网关

这个指南带领你学习怎么使用spring cloud gateway

# 你需要的东西


> 大概15分钟

> 一个编辑器或者IDE

> JDK1.8以上

> gradle 4以上或者maven3.2以上

>你也可以把源码直接引入到你的IDE中

# 怎么完成这个指南

就像其他spring指南一样，你可以从每一个小步骤开始。或者你也可以绕过你熟悉的部分。下面我这里用maven来完成这个指南。

# 使用maven构建网关

## pom.xml如下:

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.springframework</groupId>
    <artifactId>mall-gateway</artifactId>
    <version>0.1.0</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.4.RELEASE</version>
    </parent>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>spring-boot-starter-web</artifactId>
                    <groupId>org.springframework.boot</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

## 创建一个简单的路由

spring cloud gateway 使用路由来处理请求到下游服务中(也就是你下游的目的项目)。在这个指南中我们会把所哟䣌request路由到HTTPBin中。路由可以通过很多种方式配置，但是在这里我们通过gateway的API来配置。

首先我们在application.java中创建一个RouteLocator的Bean。

```
@Bean
public RouteLocator myRoutes(RouteLocatorBuilder builder) {
    return builder.routes().build();
}
```

上面的myRoutes方法需要一个routeLocatorBuilder参数，它能够很容易地创建一个路由。它允许你向路由里面添加“判断器”和“过滤器”。除了修改request/response信息之外，你也可以在其他你设置的条件下进行路由处理。

下面让我们开始创建一个路由，假设现在有一个到达我们网关的请求，这个请求URI包含/get.然后我们通过myRoutes创建一个路由,这个路由会请求到:https://httpbin.org/get。

我们会添加一个过滤器到这个路由的配置中。在请求到达下游服务(这里也就是httpbin.org)之前，这个过滤器会添加一个hello=world到header中。

```
@Bean
public RouteLocator myRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route(p -> p
            .path("/get")
            .filters(f -> f.addRequestHeader("Hello", "World"))
            .uri("http://httpbin.org:80"))
        .build();
}
```

想验证上面的代码，只需要运行application.java.理论上是8080端口。然后在浏览器中发起一个请求:http://localhost:8080/get.然后你应该会收到一下的报文:

```
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Connection": "close",
    "Forwarded": "proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:56207\"",
    "Hello": "World",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.54.0",
    "X-Forwarded-Host": "localhost:8080"
  },
  "origin": "0:0:0:0:0:0:0:1, 73.68.251.70",
  "url": "http://localhost:8080/get"
}
```

上面的报文中，header有hello:world属性

##使用闭环者(Hystrix)

下面来个有趣点的。假设下游服务非常慢，那么我们的客户端就可能会把路由打包到断路器中。你可以使用gateway Hystrix来做这个断路器。只需要实例化一个简单的过滤器就可以生成Hystrix。让我们又建一个路由来理解Hystrix.

下面的例子中，我们利用httpbin的"延迟api"。请求到这个"延迟api"需要几秒钟。那么现实场景中，如果这种延迟很长，我们可能需要把这个路径打包。看下面的例子，我们又向RouteLocator添加了一个路由。

```
@Bean
public RouteLocator myRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route(p -> p
            .path("/get")
            .filters(f -> f.addRequestHeader("Hello", "World"))
            .uri("http://httpbin.org:80"))
        .route(p -> p
            .host("*.hystrix.com")
            .filters(f -> f.hystrix(config -> config.setName("mycmd")))
            .uri("http://httpbin.org:80")).
        build();
}
```

这个例子和上一个例子不同的是，这个使用的是主机判断器了，而不是子路径判断器。这就意味着只要主机是hystrix.com.我们会路由这个请求到httpbin那里,并且打包这个请求到HystrixCommand里面.打包操作是通过断路者过滤器完成的,通过Configuration对象，我们可以配置这个断路者过滤器。只是这里我们仅仅给它加了一个名字.

测试一下，这里最好下载一个http插件，因为我们要修改host属性为hystrix.com，OK了之后，请求localhost:8080/delay/3,你就会发现504错误.

>type=Gateway Timeout, status=504

正如你看到的hystrix 在等待从httpbin的响应过程中超时,其实这个时候，我们也可以提供一个回滚操作以避免用户看到504错误，在生产模式中，我们也许会返回一些缓存中的数据。但是在下面的例子中，我们只返回一个有body到响应：
> 我们设置一个回滚uri，以防止超时,在application.java中设置一个RequestMapping

```
@RequestMapping("/fallback")
public Mono<String> fallback() {
    return Mono.just("fallback");
}
```

>再修改我们的过滤器，设置到我们的回滚路径上

```
@Bean
public RouteLocator myRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route(p -> p
            .path("/get")
            .filters(f -> f.addRequestHeader("Hello", "World"))
            .uri("http://httpbin.org:80"))
        .route(p -> p
            .host("*.hystrix.com")
            .filters(f -> f.hystrix(config -> config
                .setName("mycmd")
                .setFallbackUri("forward:/fallback")))
            .uri("http://httpbin.org:80"))
        .build();
}
```

# 写测试

作为一个优秀的开发人员，为了保证网关所做的事情是我们希望它做的，我们应该写测试。大多数情况下，我们应该限制对外部资源的依赖。特别是单元测试。所以依赖httpbin不好，不过有一个解决办法，就是把URI做成配置，这样我们就能很容易地修改他们。下面的例子中，我们新建了一个配置类。

```
@ConfigurationProperties
class UriConfiguration {

    private String httpbin = "http://httpbin.org:80";

    public String getHttpbin() {
        return httpbin;
    }

    public void setHttpbin(String httpbin) {
        this.httpbin = httpbin;
    }
}
```

然后application.java需要添加注解

>@EnableConfigurationProperties(UriConfiguration.class)
 
然后我们就可以在myRoute中使用这个配置了。 