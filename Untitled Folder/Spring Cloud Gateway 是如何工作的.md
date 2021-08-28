# 网关中间件

所谓的API网关，就是指系统的统一入口，它封装了应用程序的内部结构，为客户端提供统一服务，一些与业务本身功能无关的公共逻辑可以在这里实现，诸如认证、鉴权、监控、路由转发等。

![Spring Cloud 图片](../img/diagram-microservices.svg)

| 中间件       | Nginx               | Kong                                                 | Netflix Zuul                           | Spring Cloud Gateway              | shenyu                       |
| ------------ | ------------------- | ---------------------------------------------------- | -------------------------------------- | --------------------------------- | ---------------------------- |
| 主要开发语言 | C                   | Lua                                                  | Java                                   | Java                              | Java                         |
| 依赖关系     | 无                  | 基于 Nginx_Lua模块                                   | 无                                     | 无                                | 无                           |
| 支持协议     | HTTP                | HTTP, GRPC                                           | HTTP,                                  | HTTP                              | HTTP, WebSocket, Dubbo, GRPC |
| 扩展         | 基于 Lua 脚本       | 基于 Lua 脚本                                        | Java 写过滤器                          | Java 写过滤器、断言               | Java 写插件                  |
| 编程模型     | 多进程 + io多路复用 | 基于 Nginx                                           | Zuul 1 采用 Servlet, Zuul 2 采用 Netty | Spring WebFlux（Netty Reactor）   | Netty Reactor                |
| 配置页面     | 无                  | 丰富                                                 | 无                                     | 无                                | 丰富                         |
| 负载均衡     | 写死的              | 支持 Consul(间接可以支持使用 Consul 的 Spring Cloud) | Spring Cloud 相关                      | Spring Cloud 相关                 | 通过各种插件实现             |
| GitHub       | nginx/nginx         | Kong/kong                                            | Netflix/zuul                           | spring-cloud/spring-cloud-gateway | apache/incubator-shenyu      |

# Spring Cloud Gateway 使用和一些实现细节

官网地址：https://docs.spring.io/spring-cloud-gateway/docs/2.2.8.RELEASE/reference/html/

默认已经提供的功能：

- http 请求转发和负责均衡
- websocket 的请求转发和负载均衡
- 限流



Spring Boot 项目中引入依赖，具体的版本号视情况而定。

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

如果需要开启负载均衡，需要引入对应的依赖，比如使用 Nacos 则需要引入

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```



## 日常使用

Spring Cloud Gateway 相关配置均在 `spring.cloud.gateway` 下，需要配置均在这里

![](../img/scg.png)

### 一、全局跨域配置

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':     // 全部请求
            allow-credentials: true
            allowed-origins: "*"
            allowed-headers: "*"
            allowed-methods: "*"
            max-age: 3600
```

各个参数可以定制化



### 二、负载均衡失效的配置

如果请求时，配置了负载均衡，且无法找对对应的服务实例，默然返回 503，通过 loadbalancer.use404 可以将其改为 404 返回

```yaml
spring:
  cloud:
    gateway:
      loadbalancer:
        use404: true
```



### 三、各种谓词路由的配置

#### 1.  时间谓词路由

这个主要控制某个时间范围走指定的路由

**指定时间点之前**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: before_route
        uri: https://example.org
        predicates:
        - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
```

**指定时间短范围内**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: between_route
        uri: https://example.org
        predicates:
        - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
```

**指定时间点之后**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
```



#### 2. Cookie 谓词路由

cookie 中指定 key 的 值符合指定正则

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: cookie_route
        uri: https://example.org
        predicates:
        - Cookie=chocolate, ch.p
```

此路由匹配具有名为 Chocolate 的 cookie 的请求，该 cookie 的值与 ch.p 正则表达式匹配。 



#### 3. Header 谓词路由

和 Cookie 谓词路由功能一样，只不过这次是从 headers 里面判断

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: https://example.org
        predicates:
        - Header=X-Request-Id, \d+
```



以下内容太多，看官网吧：https://docs.spring.io/spring-cloud-gateway/docs/2.2.8.RELEASE/reference/html/

#### 4. Host 谓词路由

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Host=**.somehost.org,**.anotherhost.org
```

#### 5. 请求方法谓词路由

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: https://example.org
        predicates:
        - Method=GET,POST
```

#### 6. 路径参数谓词路由

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: path_route
        uri: https://example.org
        predicates:
        - Path=/engine/**,/blue/{segment}
```

可以通过来过去占位命名变量值

```java
Map<String, String> uriVariables = ServerWebExchangeUtils.getPathPredicateVariables(exchange);

String segment = uriVariables.get("segment");
```

#### 7. 查询参数谓词路由

请求参数中有 key 为 green 的请求参数

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: lb:skyrim-engine-sync
        predicates:
        - Query=green
```

查询参数中有 key 为 name 的变量，且值符合 gree. 正则

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: https://example.org
        predicates:
        - Query=name, gree.
```

#### 8. RemoteAddr 地址谓词路由

可以做白名单

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: https://example.org
        predicates:
        - RemoteAddr=192.168.1.1/24
```

#### 9. 权重谓词路由

第一个参数是所在组，另一个是权重

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: weight_high
        uri: https://weighthigh.org
        predicates:
        - Weight=group1, 8
      - id: weight_low
        uri: https://weightlow.org
        predicates:
        - Weight=group1, 2
```

#### 如何实现一个谓词

默认提供的谓词实现都在 `org.springframework.cloud.gateway.handler.predicate` 包下，通过如果想自定义实现一个谓词，只需继承`AbstractRoutePredicateFactory`, 即可，看一下 时间谓词路由 Before 是怎么实现的

```java
package org.springframework.cloud.gateway.handler.predicate;

import java.time.ZonedDateTime;
import java.util.Collections;
import java.util.List;
import java.util.function.Predicate;

import org.springframework.web.server.ServerWebExchange;


public class BeforeRoutePredicateFactory extends AbstractRoutePredicateFactory<BeforeRoutePredicateFactory.Config> {

	public static final String DATETIME_KEY = "datetime";

	public BeforeRoutePredicateFactory() {
		super(Config.class);
	}

	@Override
	public List<String> shortcutFieldOrder() 
    // 这里返回的 list 变量名需要和配置文件中一一对应，顺序和变量名都得对应上
    // - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
    // 比如这样顺序决定时间和变量名的对应关系，这里的情况则为
    // datetime 对应 2017-01-20T17:42:47.789-07:00[America/Denver]，
    // 其中 datatime 又需要对应上 Config.class 中的属性名，这样才能通过反射
    // 将 2017-01-20T17:42:47.789-07:00[America/Denver] 映射到 Config.class 的 datetime 属性上
    // 所以这里需要注意下顺序和变量名，否则可能会出现 Config.class 无法取到值的情况
		return Collections.singletonList(DATETIME_KEY);
	}

	@Override
	public Predicate<ServerWebExchange> apply(Config config) {
		return new GatewayPredicate() {
      
      // 这里返回 boolean 来确定是否能命中断言
			@Override
			public boolean test(ServerWebExchange serverWebExchange) {
				final ZonedDateTime now = ZonedDateTime.now();
				return now.isBefore(config.getDatetime());
			}
		};
	}

	public static class Config {

		private ZonedDateTime datetime;

		public ZonedDateTime getDatetime() {
			return datetime;
		}

		public void setDatetime(ZonedDateTime datetime) {
			this.datetime = datetime;
		}
	}
}
```

通过上面的代码可以确定出，只要 test 方法即可

**注意** 实现谓词时，需要以 XxxxRoutePredicateFactory 命名，其中 Xxxx 就是以后配置时的前缀了



### 四、过滤器的配置

过滤器分两种：GlobalFilter 针对全局路由使用；GatewayFilter  针对指定的路由的使用

#### GatewayFilter

通过为 route 配置 filters 来显示的生效

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        filters:
        - AddRequestHeader=X-Request-red, blue
```



如何自定义个 GatewayFilter，在日常的开发中，有些接口是需要登录，有些不需要登录，这里以验证为例，看一下如何定制 GatewayFilter

定制 GatewayFilter 需要实现的是 `AbstractGatewayFilterFactory`

```java
// 实现 AbstractGatewayFilterFactory 这里也需要一个 Config 对象
// 和实现谓词基本一致
@Component
@Slf4j
public class AuthGatewayFilterFactory extends AbstractGatewayFilterFactory<Config> {
	
  private String message = "{\n"
      + " \"code\": 401,\n"
      + " \"errorMessage\": \"用户身份信息失效，请重新登录\""
      + "}";

  public AuthGatewayFilterFactory() {
    super(Config.class);
  }

  @Override
  public List<String> shortcutFieldOrder() {
    // 这里也是注意顺序和名称
    return Arrays.asList("executor");
  }

  @Override
  public GatewayFilter apply(Config config) {
    return (exchange, chain) -> {
      boolean valid = 这里应该是验证逻辑，如果验证通过返回 true;
      if (valid) {
        // 如果验证通过了，就继续走过滤链
        return chain.filter(exchange);
      } else {
				// 验证没通过，直接返回 401
        ServerHttpResponse response = exchange.getResponse();
        //设置 headers
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON_UTF8);
        //设置body
        DataBuffer bodyDataBuffer = response.bufferFactory().wrap(message.getBytes());
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        return response.writeWith(Mono.just(bodyDataBuffer));
      }
    };
  }

  @ToString
  public static class Config {

    @Getter
    @Setter
    private String executor;

  }

}
```

为指定的路由配置该过滤器

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        filters:
        - Auth=JWT
```

看到这里应该可以看出，这里也是感觉名字前缀来配置的



#### GlobalFilter

GlobalFilter 对全部的路由都有效，不要额外进行配置，注入就能用。

**自定义 GlobalFilter** 直接实现 GlobalFilter 即可

```java
@Component
@Slf4j
public class TimeStatisticalFilter implements GlobalFilter, Ordered {

  private static final String START_TIME = "startTime";
  
  @Override
  public int getOrder() {
    // 指定此过滤器位于NettyWriteResponseFilter之后
    // 即待处理完响应体后接着处理响应头
    return Ordered.LOWEST_PRECEDENCE;
  }

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    exchange.getAttributes().put(START_TIME, System.currentTimeMillis());

    return chain.filter(exchange).then(Mono.fromRunnable(() -> {
      Long startTime = exchange.getAttribute(START_TIME);
      
      if (startTime != null) {
        long executeTime = (System.currentTimeMillis() - startTime);
        log.info(exchange.getRequest().getURI().getRawPath() + " : " + executeTime + "ms");
      }
    }));
  }
}
```



## 注意事项

exchange 本身中对 request、response 不能直接修改，如果需要修改，需要生成一个新的 exchange 对象进行修改，调用链本身有顺序，如果要自定义 Filter 注意优先级的设置



## 常见过滤器的优先级和功能

每个版本的 Spring Cloud Gateway 可能不一样，具体看 `org.springframework.cloud.gateway.config.GatewayAutoConfiguration` 里面相关配置

| 名称                             | 优先级                    | 是否启用                                                     | 请求阶段                                                     | 响应阶段                         |
| -------------------------------- | ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------- |
| RemoveCachedBodyFilter           | HIGHEST_PRECEDENCE        | 是                                                           |                                                              | 如果发现有 RequestBody 就去除    |
| AdaptCachedBodyGlobalFilter      | HIGHEST_PRECEDENCE + 1000 | 是                                                           | 把 requestBody 缓存到 cachedRequestBody Attribute 中         |                                  |
| DefaultValue                     |                           |                                                              | 什么都没干，还抛出了一个异常                                 |                                  |
| ForwardPathFilter                | 0                         | 是                                                           | set the path in the request URI if the {@link Route} URI has the scheme |                                  |
| GatewayMetricsFilter             | 0                         | 是                                                           | 记录下发起时间                                               | 统计耗时                         |
| NettyWriteResponseFilter         | -1                        | 是                                                           |                                                              | 将最终的 exchange 响应写回客户端 |
| WebClientWriteResponseFilter     | -1                        | 否，代码中无任何开启的方式                                   |                                                              | 与 NettyWriteResponseFilter 类似 |
| RouteToRequestUrlFilter          | 10000                     | 是                                                           | 将原始请求地址和路由配置的地址进行替换，将替换成的新地址放在 GATEWAY_REQUEST_URL_ATTR  属性中 |                                  |
| ReactiveLoadBalancerClientFilter | 10150                     | 是                                                           | 如果是 lb 则根据服务发现找到应的实例将实例地址设置成当前请求的 host |                                  |
| NoLoadBalancerClientFilter       | 10150                     | 当 ReactorLoadBalancer 不存在且 spring.cloud.gateway.loadbalancer 属性存在 |                                                              |                                  |
| WebsocketRoutingFilter           | LOWEST_PRECEDENCE - 1     | WebSocket  的请求转发                                        |                                                              |                                  |
| ForwardRoutingFilter             | LOWEST_PRECEDENCE         |                                                              | 如果 shema 中含有 forward 则转发                             |                                  |
| NettyRoutingFilter               | LOWEST_PRECEDENCE         | 是                                                           | 如果 shema 中为 http 则转发并写入 response                   |                                  |
| WebClientHttpRoutingFilter       | LOWEST_PRECEDENCE         | 否，代码中无任何开启的方式                                   | 和 NettyRoutingFilter 功能一样，转发请求的方式改为了 WebClient |                                  |

## 一些常见的操作

### 1. 修改 response headers

```java
ServerHttpResponse response = exchange.getResponse();
response.getHeaders().setContentType(MediaType.APPLICATION_JSON_UTF8);  // 这句如果出现异常，直接 catch 即可，不影响修改
```

### 2. 修改 response 状态码

```java
ServerHttpResponse response = exchange.getResponse();
response.setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
```

### 3. 修改 response 示例

```java
ServerHttpResponse response = exchange.getResponse();

//设置 headers
response.getHeaders().setContentType(MediaType.APPLICATION_JSON_UTF8);

//设置body
DataBuffer bodyDataBuffer = response.bufferFactory().wrap("{}".getBytes());
response.setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
return response.writeWith(Mono.just(bodyDataBuffer));
```

### 4. 设置或者添加属性

```java
exchange.getAttributes().put(key, value);
exchange.getAttribute(key)
```

### 5. 统计请求时间示例

```java
public class TimeStatisticalFilter implements GlobalFilter, Ordered {

  private static final String START_TIME = "startTime";

  @Override
  public int getOrder() {
    // 这里可以通过设置不同的优先级来统计不同的阶段的时间
    return Ordered.HIGHEST_PRECEDENCE;
  }

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    exchange.getAttributes().put(START_TIME, System.currentTimeMillis());
    return chain.filter(exchange).then(Mono.fromRunnable(() -> {
      Long startTime = exchange.getAttribute(START_TIME);
      if (startTime != null) {
        long executeTime = (System.currentTimeMillis() - startTime);
        log.info(exchange.getRequest().getURI().getRawPath() + " : " + executeTime + "ms");
      }
    }));
  }
}
```

### 6. 读取 Request Body

有一些情况，我们可能要读取 Request Body，比如要对 Request Body 加解密或者其他的判断，如果只是读取操作，可以使用 ReadBodyRoutePredicateFactory 来实现，ReadBodyRoutePredicateFactory 配置有两个参数需要配置：1. inClass 用来配置将body 转换的类型；2. Predicate 判断什么情况下可以转。

先配置一个永真的 Predicate 来确定执行这个谓词

```java
@Configuration
public class Config {

  @Bean
  public Predicate bodyPredicate(){
    return new Predicate() {
      @Override
      public boolean test(Object o) {
        return true;
      }
    };
  }
}
```

增加配置 route 配置，对需要读取 Request Body 的路由进行配置，这里配置将 Request Body 转换成 String，也方便后面使用的直接进行其他转换操作，例如 JSON。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: common
          uri: lb://hhhh
          predicates:
            - Path=/hhhh
            - name: ReadBody
              args:
                inClass: '#{T(String)}'
                predicate: '#{@bodyPredicate}'
```

在后面的操作中，可以直接一下语句来获取 Request Body 来进行其他操作 

```java
String requestBody = exchange.getAttribute("cachedRequestBodyObject");
```

### 7. 删除重复的 headers

```java
@Component
public class RemoveDuplicateResponseHeaderFilter implements GlobalFilter, Ordered {

  @Override
  public int getOrder() {
    // 指定此过滤器位于NettyWriteResponseFilter之后
    // 即待处理完响应体后接着处理响应头
    return NettyWriteResponseFilter.WRITE_RESPONSE_FILTER_ORDER + 1;
  }

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    return chain.filter(exchange).then(Mono.defer(() -> {
      exchange.getResponse().getHeaders().entrySet().stream()
          .filter(kv -> (kv.getValue() != null && kv.getValue().size() > 1))
          .forEach(kv -> {
            kv.setValue(new ArrayList<String>() {{
              add(kv.getValue().get(0));
            }});
          });
      return chain.filter(exchange);
    }));
  }
}
```
