This part of the reference documentation covers support for reactive-stack WebSocket messaging.

参考文档的这一部分涵盖了对反应式堆栈 WebSocket 消息传递的支持。

# Introduction to WebSocket

# 介绍WebSocket

The WebSocket protocol, [RFC 6455](https://tools.ietf.org/html/rfc6455), provides a standardized way to establish a full-duplex, two-way communication channel between client and server over a single TCP connection. It is a different TCP protocol from HTTP but is designed to work over HTTP, using ports 80 and 443 and allowing re-use of existing firewall rules.

WebSocket 协议 [RFC 6455](https://tools.ietf.org/html/rfc6455) 提供了一种标准化的方法来建立客户端和服务器之间通过单个 TCP 连接的全双工、双向通信通道。 这是一个与HTTP不同的TCP协议，但旨在通过 HTTP 工作，使用端口 80 和 443，并允许重新使用现有的防火墙规则。 

A WebSocket interaction begins with an HTTP request that uses the HTTP `Upgrade` header to upgrade or, in this case, to switch to the WebSocket protocol. The following example shows such an interaction:

WebSocket 交互以 HTTP 请求开始，使用 HTTP `Upgrade` 首部进行升级，或者在这种情况下，切换到 WebSocket 协议。 以下示例显示了这样的交互：

``` yaml
GET /spring-websocket-portfolio/portfolio HTTP/1.1
Host: localhost:8080
Upgrade: websocket 
Connection: Upgrade 
Sec-WebSocket-Key: Uc9l9TMkWGbHFD2qnFHltg==
Sec-WebSocket-Protocol: v10.stomp, v11.stomp
Sec-WebSocket-Version: 13
Origin: http://localhost:8080
```

  - The `Upgrade` header.

  - Using the `Upgrade` connection.

Instead of the usual 200 status code, a server with WebSocket support returns output similar to the following:

并非200响应状态码，支持WebSocket协议的服务端返回类似输出，如下：

``` yaml
HTTP/1.1 101 Switching Protocols 
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: 1qVdfYHU9hPOl4JYYNXF623Gzn0=
Sec-WebSocket-Protocol: v10.stomp
```

  - Protocol switch

    - 协议切换

After a successful handshake, the TCP socket underlying the HTTP upgrade request remains open for both the client and the server to continue to send and receive messages.

A complete introduction of how WebSockets work is beyond the scope of this document. See RFC 6455, the WebSocket chapter of HTML5, or any of the many introductions and tutorials on the Web.

Note that, if a WebSocket server is running behind a web server (e.g. nginx), you likely need to configure it to pass WebSocket upgrade requests on to the WebSocket server. Likewise, if the application runs in a cloud environment, check the instructions of the cloud provider related to WebSocket support.

成功握手后，HTTP 升级请求底层的 TCP 套接字对客户端和客户端都保持打开状态继续发送和接收消息。

对 WebSockets 工作原理的完整介绍超出了本文档的范围。 请参阅 RFC 6455，WebSocket 章节HTML5 或互联网上的介绍和教程。

请注意，如果 WebSocket 服务器在 Web 服务器（例如 nginx）后面运行，则你可能需要将其配置为通过WebSocket 升级请求到 WebSocket 服务器。 同样，如果应用程序在云环境中运行，请检查与 WebSocket 支持相关的云提供商的说明。

## HTTP Versus WebSocket

Even though WebSocket is designed to be HTTP-compatible and starts with an HTTP request, it is important to understand that the two protocols lead to very different architectures and application programming models.

In HTTP and REST, an application is modeled as many URLs. To interact with the application, clients access those URLs, request-response style. Servers route requests to the appropriate handler based on the HTTP URL, method, and headers.

By contrast, in WebSockets, there is usually only one URL for the initial connect. Subsequently, all application messages flow on that same TCP connection. This points to an entirely different asynchronous, event-driven, messaging architecture.

WebSocket is also a low-level transport protocol, which, unlike HTTP, does not prescribe any semantics to the content of messages. That means that there is no way to route or process a message unless the client and the server agree on message semantics.

WebSocket clients and servers can negotiate the use of a higher-level, messaging protocol (for example, STOMP), through the `Sec-WebSocket-Protocol` header on the HTTP handshake request. In the absence of that, they need to come up with their own conventions.

尽管 WebSocket 被设计为与 HTTP 兼容并以 HTTP 请求开始，但是这两种协议导致了非常不同的架构和应用程序编程模型。

在 HTTP 和 REST 中，一个应用程序被建模为多个 URL。为了与应用程序交互，客户端访问这些 URL，请求-响应风格。 服务器根据 HTTP URL、方法和首部将请求路由到适当的处理程序。

相比之下，在 WebSockets 中，通常只有一个 URL 用于初始连接。随后，所有消息在同一个 TCP 连接上流动。 这指向一个完全不同的异步、事件驱动、消息传递的架构。

WebSocket 也是一种低级传输协议，与 HTTP 不同，它不为消息的内容规定任何语义。 这意味着除非客户端和服务器就消息语义达成一致，否则无法路由或处理消息。

WebSocket 客户端和服务器可以协商使用更高级别的消息传递协议（例如，STOMP），通过HTTP 握手请求中的 `Sec-WebSocket-Protocol` 首部。 如果不使用这种模式，他们需要使用他们自己的会话协议。

## When to Use WebSockets

## 什么时候使用WebSockets

WebSockets can make a web page be dynamic and interactive. However, in many cases, a combination of Ajax and HTTP streaming or long polling can provide a simple and effective solution.

For example, news, mail, and social feeds need to update dynamically, but it may be perfectly okay to do so every few minutes. Collaboration, games, and financial apps, on the other hand, need to be much closer to real-time.

Latency alone is not a deciding factor. If the volume of messages is relatively low (for example, monitoring network failures) HTTP streaming or polling can provide an effective solution. It is the combination of low latency, high frequency, and high volume that make the best case for the use of WebSocket.

Keep in mind also that over the Internet, restrictive proxies that are outside of your control may preclude WebSocket interactions, either because they are not configured to pass on the `Upgrade` header or because they close long-lived connections that appear idle. This means that the use of WebSocket for internal applications within the firewall is a more straightforward decision than it is for public facing applications.

WebSockets 可以使网页具有动态性和交互性。但是，在很多情况下，Ajax 和 HTTP 流的组合或长轮询可以提供简单有效的解决方案。

例如，新闻、邮件和社交提要需要动态更新，但每隔几分钟更新可能完全没问题。 另一方面，协作、游戏和金融应用程序需要更接近实时。

延迟本身并不是决定性因素。 如果消息量比较小（比如监控网络失败）HTTP 流或轮询可以提供有效的解决方案。 低延迟、高频率和高容量是使用 WebSocket 的最佳案例。

还要记住，在 Internet 上，不受您控制的限制性代理可能会阻止 WebSocket交互，要么是因为它们未配置为传递 `Upgrade` 标头，要么是因为它们关闭了看起来空闲的长连接。 这意味着在防火墙内对内部应用程序使用 WebSocket 是一种比面向公众的应用程序更直接的决定。

# WebSocket API

[Same as in the Servlet stack](web.xml#websocket-server)

The Spring Framework provides a WebSocket API that you can use to write client- and server-side applications that handle WebSocket messages.

Spring Framework 提供了一个 WebSocket API，你可以使用它来编写客户端和服务器端应用程序来处理WebSocket 消息。

## Server

[Same as in the Servlet stack](web.xml#websocket-server-handler)

To create a WebSocket server, you can first create a `WebSocketHandler`. The following example shows how to do so:

要创建 WebSocket 服务器，你可以先创建一个 `WebSocketHandler`。 以下示例显示了如何执行此操作：

**Java.**

``` java
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.WebSocketSession;

public class MyWebSocketHandler implements WebSocketHandler {

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        // ...
    }
}
```

**Kotlin.**

``` kotlin
import org.springframework.web.reactive.socket.WebSocketHandler
import org.springframework.web.reactive.socket.WebSocketSession

class MyWebSocketHandler : WebSocketHandler {

    override fun handle(session: WebSocketSession): Mono<Void> {
        // ...
    }
}
```

Then you can map it to a URL:

然后你把它映射到一个URL：

**Java.**

``` java
@Configuration
class WebConfig {

    @Bean
    public HandlerMapping handlerMapping() {
        Map<String, WebSocketHandler> map = new HashMap<>();
        map.put("/path", new MyWebSocketHandler());
        int order = -1; // before annotated controllers

        return new SimpleUrlHandlerMapping(map, order);
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
class WebConfig {

    @Bean
    fun handlerMapping(): HandlerMapping {
        val map = mapOf("/path" to MyWebSocketHandler())
        val order = -1 // before annotated controllers

        return SimpleUrlHandlerMapping(map, order)
    }
}
```

If using the [WebFlux Config](web-reactive.xml#webflux-config) there is nothing further to do, or otherwise if not using the WebFlux config you’ll need to declare a `WebSocketHandlerAdapter` as shown below:

如果使用 [WebFlux Config](web-reactive.xml#webflux-config) 这样就可以了。 否则如果不使用 WebFlux ，你需要声明一个 `WebSocketHandlerAdapter`，如下所示：

**Java.**

``` java
@Configuration
class WebConfig {

    // ...

    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
class WebConfig {

    // ...

    @Bean
    fun handlerAdapter() =  WebSocketHandlerAdapter()
}
```

## `WebSocketHandler`

The `handle` method of `WebSocketHandler` takes `WebSocketSession` and returns `Mono<Void>` to indicate when application handling of the session is complete. The session is handled through two streams, one for inbound and one for outbound messages. The following table describes the two methods that handle the streams:

`WebSocketHandler` 的 `handle` 方法接受 `WebSocketSession` 并返回 `Mono<Void>` 以指示会话的应用程序处理何时完成。 会话通过两个流处理，一个用于入站消息，另一个用于出站消息。 下表描述了处理流的两种方法：

| `WebSocketSession` method                      | Description                                                                                                                                         |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Flux<WebSocketMessage> receive()`             | Provides access to the inbound message stream and completes when the connection is closed.                                                          |
| `Mono<Void> send(Publisher<WebSocketMessage>)` | Takes a source for outgoing messages, writes the messages, and returns a `Mono<Void>` that completes when the source completes and writing is done. |

| `WebSocketSession` 方法                        | 描述                                                                                                                                         |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Flux<WebSocketMessage> receive()`             | 提供对入站消息流的访问，并在连接关闭时完成。                                                          |
| `Mono<Void> send(Publisher<WebSocketMessage>)` | 获取传出消息的源，写入消息，并返回一个 `Mono<Void>`，在源完成并写入完成时完成。|

A `WebSocketHandler` must compose the inbound and outbound streams into a unified flow and return a `Mono<Void>` that reflects the completion of that flow. Depending on application requirements, the unified flow completes when:

`WebSocketHandler` 必须将入站和出站流组合成一个统一的流，并返回一个反映该流完成的 `Mono<Void>`。 根据应用程序要求，统一的流将在以下情况下完成：

  - Either the inbound or the outbound message stream completes.

  - 入站或出站消息流完成。

  - The inbound stream completes (that is, the connection closed), while the outbound stream is infinite.

  - 入站流完成（即连接关闭），而出站流是无限的。

  - At a chosen point, through the `close` method of `WebSocketSession`.

  - 在选定的点，通过 `WebSocketSession` 的 `close` 方法。

When inbound and outbound message streams are composed together, there is no need to check if the connection is open, since Reactive Streams signals end activity. The inbound stream receives a completion or error signal, and the outbound stream receives a cancellation signal.

当入站和出站消息流组合在一起时，无需检查连接是否打开，因为 Reactive Streams 会给出指示。 入站流接收完成或错误信号，出站流接收取消信号。

The most basic implementation of a handler is one that handles the inbound stream. The following example shows such an implementation:

一个handler最基本的实现是处理入站流。 下面的示例展示了一个实现：

**Java.**

``` java
class ExampleHandler implements WebSocketHandler {

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        return session.receive()            
                .doOnNext(message -> {
                    // ...                  
                })
                .concatMap(message -> {
                    // ...                  
                })
                .then();                    
    }
}
```

  - Access the stream of inbound messages.

  - 访问入站消息流。

  - Do something with each message.

  - 针对每一个消息做一些事情。

  - Perform nested asynchronous operations that use the message content.

  - 使用消息内容做一些嵌套的异步操作。

  - Return a `Mono<Void>` that completes when receiving completes.

  - 返回一个 `Mono<Void>` 在接收完成的时候完成。

**Kotlin.**

``` kotlin
class ExampleHandler : WebSocketHandler {

    override fun handle(session: WebSocketSession): Mono<Void> {
        return session.receive()            
                .doOnNext {
                    // ...                  
                }
                .concatMap {
                    // ...                  
                }
                .then()                     
    }
}
```

  - Access the stream of inbound messages.

  - 访问入站消息流。

  - Do something with each message.

  - 针对每一个消息做一些事情。

  - Perform nested asynchronous operations that use the message content.

  - 使用消息内容做一些嵌套的异步操作。

  - Return a `Mono<Void>` that completes when receiving completes.

  - 返回一个 `Mono<Void>` 在接收完成的时候完成。

> **Tip**
> 
> For nested, asynchronous operations, you may need to call `message.retain()` on underlying servers that use pooled data buffers (for example, Netty). Otherwise, the data buffer may be released before you have had a chance to read the data. For more background, see [Data Buffers and Codecs](core.xml#databuffers).

The following implementation combines the inbound and outbound streams:

> **提示**
>
> 对于嵌套的异步操作，你需要在使用池化数据缓冲区的底层服务器（例如 Netty）上调用 `message.retain()`。 否则，数据缓冲区可能会在你有机会读取数据之前被释放。 有关更多背景信息，请参阅 [数据缓冲区和编解码器](core.xml#databuffers)。

以下实现结合了入站和出站流： 

**Java.**

``` java
class ExampleHandler implements WebSocketHandler {

    @Override
    public Mono<Void> handle(WebSocketSession session) {

        Flux<WebSocketMessage> output = session.receive()             
                .doOnNext(message -> {
                    // ...
                })
                .concatMap(message -> {
                    // ...
                })
                .map(value -> session.textMessage("Echo " + value)); 

        return session.send(output);                                    
    }
}
```

  - Handle the inbound message stream.

  - Create the outbound message, producing a combined flow.

  - Return a `Mono<Void>` that does not complete while we continue to receive.

**Kotlin.**

``` kotlin
class ExampleHandler : WebSocketHandler {

    override fun handle(session: WebSocketSession): Mono<Void> {

        val output = session.receive()                      
                .doOnNext {
                    // ...
                }
                .concatMap {
                    // ...
                }
                .map { session.textMessage("Echo $it") }    

        return session.send(output)                         
    }
}
```

  - Handle the inbound message stream.

  - Create the outbound message, producing a combined flow.

  - Return a `Mono<Void>` that does not complete while we continue to receive.

Inbound and outbound streams can be independent and be joined only for completion, as the following example shows:

**Java.**

``` java
class ExampleHandler implements WebSocketHandler {

    @Override
    public Mono<Void> handle(WebSocketSession session) {

        Mono<Void> input = session.receive()                              
                .doOnNext(message -> {
                    // ...
                })
                .concatMap(message -> {
                    // ...
                })
                .then();

        Flux<String> source = ... ;
        Mono<Void> output = session.send(source.map(session::textMessage));   

        return Mono.zip(input, output).then();                              
    }
}
```

  - Handle inbound message stream.

  - Send outgoing messages.

  - Join the streams and return a `Mono<Void>` that completes when either stream ends.

**Kotlin.**

``` kotlin
class ExampleHandler : WebSocketHandler {

    override fun handle(session: WebSocketSession): Mono<Void> {

        val input = session.receive()                                   
                .doOnNext {
                    // ...
                }
                .concatMap {
                    // ...
                }
                .then()

        val source: Flux<String> = ...
        val output = session.send(source.map(session::textMessage))     

        return Mono.zip(input, output).then()                           
    }
}
```

  - Handle inbound message stream.

  - Send outgoing messages.

  - Join the streams and return a `Mono<Void>` that completes when either stream ends.

## `DataBuffer`

`DataBuffer` is the representation for a byte buffer in WebFlux. The Spring Core part of the reference has more on that in the section on [Data Buffers and Codecs](core.xml#databuffers). The key point to understand is that on some servers like Netty, byte buffers are pooled and reference counted, and must be released when consumed to avoid memory leaks.

When running on Netty, applications must use `DataBufferUtils.retain(dataBuffer)` if they wish to hold on input data buffers in order to ensure they are not released, and subsequently use `DataBufferUtils.release(dataBuffer)` when the buffers are consumed.

## Handshake

[Same as in the Servlet stack](web.xml#websocket-server-handshake)

`WebSocketHandlerAdapter` delegates to a `WebSocketService`. By default, that is an instance of `HandshakeWebSocketService`, which performs basic checks on the WebSocket request and then uses `RequestUpgradeStrategy` for the server in use. Currently, there is built-in support for Reactor Netty, Tomcat, Jetty, and Undertow.

`HandshakeWebSocketService` exposes a `sessionAttributePredicate` property that allows setting a `Predicate<String>` to extract attributes from the `WebSession` and insert them into the attributes of the `WebSocketSession`.

## Server Configation

[Same as in the Servlet stack](web.xml#websocket-server-runtime-configuration)

The `RequestUpgradeStrategy` for each server exposes configuration specific to the underlying WebSocket server engine. When using the WebFlux Java config you can customize such properties as shown in the corresponding section of the [WebFlux Config](web-reactive.xml#webflux-config-websocket-service), or otherwise if not using the WebFlux config, use the below:

**Java.**

``` java
@Configuration
class WebConfig {

    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter(webSocketService());
    }

    @Bean
    public WebSocketService webSocketService() {
        TomcatRequestUpgradeStrategy strategy = new TomcatRequestUpgradeStrategy();
        strategy.setMaxSessionIdleTimeout(0L);
        return new HandshakeWebSocketService(strategy);
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
class WebConfig {

    @Bean
    fun handlerAdapter() =
            WebSocketHandlerAdapter(webSocketService())

    @Bean
    fun webSocketService(): WebSocketService {
        val strategy = TomcatRequestUpgradeStrategy().apply {
            setMaxSessionIdleTimeout(0L)
        }
        return HandshakeWebSocketService(strategy)
    }
}
```

Check the upgrade strategy for your server to see what options are available. Currently, only Tomcat and Jetty expose such options.

## CORS

[Same as in the Servlet stack](web.xml#websocket-server-allowed-origins)

The easiest way to configure CORS and restrict access to a WebSocket endpoint is to have your `WebSocketHandler` implement `CorsConfigurationSource` and return a `CorsConfiguration` with allowed origins, headers, and other details. If you cannot do that, you can also set the `corsConfigurations` property on the `SimpleUrlHandler` to specify CORS settings by URL pattern. If both are specified, they are combined by using the `combine` method on `CorsConfiguration`.

## Client

Spring WebFlux provides a `WebSocketClient` abstraction with implementations for Reactor Netty, Tomcat, Jetty, Undertow, and standard Java (that is, JSR-356).

> **Note**
> 
> The Tomcat client is effectively an extension of the standard Java one with some extra functionality in the `WebSocketSession` handling to take advantage of the Tomcat-specific API to suspend receiving messages for back pressure.

To start a WebSocket session, you can create an instance of the client and use its `execute` methods:

**Java.**

``` java
WebSocketClient client = new ReactorNettyWebSocketClient();

URI url = new URI("ws://localhost:8080/path");
client.execute(url, session ->
        session.receive()
                .doOnNext(System.out::println)
                .then());
```

**Kotlin.**

``` kotlin
val client = ReactorNettyWebSocketClient()

        val url = URI("ws://localhost:8080/path")
        client.execute(url) { session ->
            session.receive()
                    .doOnNext(::println)
            .then()
        }
```

Some clients, such as Jetty, implement `Lifecycle` and need to be stopped and started before you can use them. All clients have constructor options related to configuration of the underlying WebSocket client.
