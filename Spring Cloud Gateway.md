# Spring Cloud Gateway 文档

2.2.5 RELEASE

这个项目在 Spring 生态系统的顶层创建一个 API 网关，项目中包含 Spring 5、SpringBoot 2 和 Project Reactor（响应式编程）。

1.  如何引入 Spring Cloud Gateway

   在你的项目中导入 group ID 为 `org.springframework.cloud`，artifact ID 为 `spring-cloud-starter-gateway` 的 spring starter，如果你的项目使用 Maven 构建，那么引入方式如下：

   ```java
   <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
   </dependency>
   ```

   如果你引入了这个包，但并不想 spring gateway 生效，可以采用如下配置:  `spring.cloud.gateway.enabled=false`

   > Spring Cloud Gateway 内嵌了一个 Netty 网络框架，它不能与传统的 Servlet 容器兼容，例如基于 Servlet 实现的 HttpServletRequest 无法在由以 Spring Cloud Gateway 为基础的项目中使用。

2. Spring Cloud Gateway 中的三个关键字

   - Route （路由）：路由是网关的基础块，它由四个部分组成：**id** **uri** **predicates** **filters**，如果所有的 **predicate** 都匹配，则路由生效。
   - Predicate（断言）：Java 8 的一个 Function Predicate，输入的类型为 ServerWebExchange，它可以让你对 Http Request 中的任意一项进行断言。
   - Filter （过滤器）：它是 Spring 框架中 GatewayFilter 的实现，通过内置或自定义的 filters，你可以修改 request 或 response。

3. Spring Cloud Gateway 的工作方式

   

   ![](https://docs.spring.io/spring-cloud-gateway/docs/2.2.5.RELEASE/reference/html/images/spring_cloud_gateway_diagram.png)

   当有一个请求从客户端到网关时，首先经过 Gateway Handler Mapping 判断该请求是和某一 route 匹配，如果是则将请求转到 Gateway Web Handler，该 Handler 将请求通		过一系列指定的 Filter，图中的 Filter 被虚线隔开是因为 Filter 不仅可以在请求处理前执行也可以在请求处理后执行，pre filter 会在请求被处理前执行，post filter 在请求被处理后执行。

   >  Route 中的 URI 如果没有指定端口，那么对于 http 请求会添加默认端口 80，对于 https 请求则是 443 。

4. 配置 Predicate 和 Filter 的两种方式

   1. Shortcut Configuration

      该配置方式中的参数需依照规定顺序用逗号依次排列（每一种断言或者过滤器所需要的参数不一定一样）

      ```yaml
      spring:
        cloud:
          gateway:
            routes:
            - id: after_route
              uri: https://example.org
              predicates:
              - Cookie=mycookie,mycookievalue
      ```

   2. Fully Expanded Arguments

      ```yaml
      spring:
        cloud:
          gateway:
            routes:
            - id: after_route
              uri: https://example.org
              predicates:
              - name: Cookie
                args:
                  name: mycookie
                  regexp: mycookievalue
      ```

5. 路由中断言的种类（断言工厂）

   	- **After**： 接受一个时间作为参数，请求产生的时间需在指定的时间之后

   - **Before**： 接受一个时间作为参数，请求产生的时间需在指定的时间之前

   - **Between**： 接受一个时间作为参数，请求产生的时间需在指定的时间之间

   - **Cookie**： 接受一个 cookie name 和 一个正则表达式作为参数，将表达式与该 name 所对应的 cookie value 相比较

   - **Header**： 接受一个 header name 和 一个正则表达式作为参数，将表达式与该 name 所对应的 header value 相比较

   - **Host**： 接受一个模式列表作为参数，与 **Host** header 相比较

     Example: 

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

     除了上述例子中的模式之外，Host 断言还可以接受 uri 模版，例如：Host={sub}.somehost.org，并且其中的变量值可以通过 ServerWebExchange.getAttributes() 来获取。

   - **Method**：接受一个方法列表作为参数，与 request method 相比较

   - **Path**：接受一个模式列表作为参数，与 request path 相比较，除此之外，该断言也可以接受 uri 模版的形式，与 Host 断言同理

   - **Query**：接受一个 param name 和一个正则表达式作为参数，将表达式和该 name 所对应的 param value 相比较

     *Example 1*:

     ```yaml
     spring:
       cloud:
         gateway:
           routes:
           - id: query_route
             uri: https://example.org
             predicates:
             - Query=color
     ```

     上述实例表示 request param 中有 name 为 color 的参数则断言为 true

     *Example 2*:

     ```yaml
     spring:
       cloud:
         gateway:
           routes:
           - id: query_route
             uri: https://example.org
             predicates:
             - Query=color,gree.
     ```

     上述实例表示 request param 中有 name 为 color 的参数并且其值为 gree. 则断言为 true，由于后者为正则表达式，所以 green 或者 greet 都可以匹配

   - **RemoteAddr**：接受一个 ip 地址作为参数，可以为 ipv4 或者 ipv6 并且可以携带子网掩码（subnet mask），将该地址与 RemoteAddr header相比较

     Example:

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

   - **Weight**：接受一个 group name 和一个 weight (int) 作为参数，同一个 group name 下的请求将按照权重进行分发

     Example:

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

     上述实例表示 20% 的请求将发送到 weightlow.org，80% 的请求将发送到 weighthigh.org

     > 这里看起来并不像是一个断言，更像是一个功能，但是这个功能起到了筛选的作用，这也许就是这个功能放到断言目录下的原因吧。

6. 路由中过滤器的种类（过滤器工厂）

   ​	路由中的过滤器允许修改 Request 和 Response，Spring Cloud Gateway 中已经内建了很多过滤器。

   - AddRequestHeader

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

     上述实例表示添加一个 `X-Request-red:blue` 的 Request Header。

     除此之外，该过滤器还可以使用 Host 断言和 Path 断言中的路径模版携带的参数，如下：

     ```yaml
     spring:
       cloud:
         gateway:
           routes:
           - id: add_request_header_route
             uri: https://example.org
             predicates:
             - Path=/red/{segment}
             filters:
             - AddRequestHeader=X-Request-Red, Blue-{segment}
     ```

   - AddRequestParameter

     ```yaml
     spring:
       cloud:
         gateway:
           routes:
           - id: add_request_parameter_route
             uri: https://example.org
             filters:
             - AddRequestParameter=color, blue
     ```

     上述实例表示添加一个 color=blue 的 querying string。

     同样的，该过滤器也可以使用路径模版中的参数，参考 AddRequestHeader 中的例子

   - DedupeResponseHeader

     ```yaml
     spring:
       cloud:
         gateway:
           routes:
           - id: dedupe_response_header_route
             uri: https://example.org
             filters:
             - DedupeResponseHeader=Access-Control-Allow-Origin
     ```

     该过滤器接受两个参数，name、strategy，name 是一个 header name  的数组，strategy 是一个枚举值，允许的值有 `RETAIN_FIRST` (默认值)，`RETAIN_LAST` 和 `RETAIN_UNIQUE`

     这个过滤器可以删除掉某个 Response Header 中的多余项，例如该实例中可以使得 Access-Control-Allow-Origin 仅保留一项，防止 Gateway 和 底层服务都对该 header 进行操作所导致的 ResponseHeader 错误问题。

     [^Access-Control-Allow-Origin: *,http://example.org]: 该 header 就是一个明显的错误，被设置了两次，保留一个即可。

   - Hystrix  

     该过滤器后续会被移除，官方建议使用 CircuitBreaker 作为替代品。

   - CircuitBreaker

     

   - 

   

7. 

8. 

