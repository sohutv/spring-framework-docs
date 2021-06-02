This section describes Spring Framework’s support for the RSocket protocol.

这个部分描述了Spring框架对于RSocket协议的支持。

# Overview

# 概述

RSocket is an application protocol for multiplexed, duplex communication over TCP, WebSocket, and other byte stream transports, using one of the following interaction models:

RSocket 是一种应用协议，用于通过 TCP、WebSocket 和其他字节流传输协议进行多路复用、双工通信，使用以下交互模型之一：

  - `Request-Response` — send one message and receive one back.

  - `请求-响应` — 发送一个消息然后等待回复。

  - `Request-Stream` — send one message and receive a stream of messages back.

  - `请求-响应流` — 发送一条消息并接收返回的消息流。

  - `Channel` — send streams of messages in both directions.

  - `Channel` — 双向发送消息流。

  - `Fire-and-Forget` — send a one-way message.

  - `Fire-and-Forget` — 发送单向消息。

Once the initial connection is made, the "client" vs "server" distinction is lost as both sides become symmetrical and each side can initiate one of the above interactions. This is why in the protocol calls the participating sides "requester" and "responder" while the above interactions are called "request streams" or simply "requests".

一旦建立了初始连接，“客户端”与“服务器”的区别就会消失，因为双方变得对称并且每一方都可以发起上述交互之一。 这就是为什么在协议中将参与方称为“请求者”和“响应者”，而将上述交互称为“请求流”或简称为“请求”。

These are the key features and benefits of the RSocket protocol:

这些是 RSocket 协议的主要特性和优点：

  - [Reactive Streams](https://www.reactive-streams.org/) semantics across network boundary — for streaming requests such as `Request-Stream` and `Channel`, back pressure signals travel between requester and responder, allowing a requester to slow down a responder at the source, hence reducing reliance on network layer congestion control, and the need for buffering at the network level or at any level.

  - Request throttling — this feature is named "Leasing" after the `LEASE` frame that can be sent from each end to limit the total number of requests allowed by other end for a given time. Leases are renewed periodically.

  - Session resumption — this is designed for loss of connectivity and requires some state to be maintained. The state management is transparent for applications, and works well in combination with back pressure which can stop a producer when possible and reduce the amount of state required.

  - [Reactive Streams](https://www.reactive-streams.org/) 跨网络边界的语义 — 对于诸如`Request-Stream`和`Channel`之类的流请求，回压信号在请求者和响应者之间传播，允许请求者在源头减慢响应者的速度，从而减少对网络层拥塞控制的依赖，以及在网络级别或任何级别进行缓冲的需要。

  - 请求限制 — 此功能叫做“租赁”，任何一端都可以发送`LEASE`帧，以限制另一端在给定时间内允许的请求总数。 租约定期更新。

  - 会话恢复 — 这是为失去连接而设计的，需要维护一些状态。 状态管理对应用程序是透明的，并且与回压结合使用效果很好，可以在可能的情况下停止生产者并减少所需的状态量。

  - Fragmentation and re-assembly of large messages.

  - 大消息的碎片化和重组。

  - Keepalive (heartbeats).

  - 保活（心跳）。

RSocket has [implementations](https://github.com/rsocket) in multiple languages. The [Java library](https://github.com/rsocket/rsocket-java) is built on [Project Reactor](https://projectreactor.io/), and [Reactor Netty](https://github.com/reactor/reactor-netty) for the transport. That means signals from Reactive Streams Publishers in your application propagate transparently through RSocket across the network.

RSocket 有多种语言的[实现](https://github.com/rsocket)。 [Java 库](https://github.com/rsocket/rsocket-java) 建立在 [Project Reactor](https://projectreactor.io/) 和 [Reactor Netty](https://github.com/reactor/reactor-netty) 用于传输。 这意味着来自应用程序中的 Reactive Streams Publisher 的信号通过 RSocket 透明地在网络上传播。

## The Protocol

## 协议

One of the benefits of RSocket is that it has well defined behavior on the wire and an easy to read [specification](https://rsocket.io/docs/Protocol) along with some protocol [extensions](https://github.com/rsocket/rsocket/tree/master/Extensions). Therefore it is a good idea to read the spec, independent of language implementations and higher level framework APIs. This section provides a succinct overview to establish some context.

RSocket 的好处之一是它在网络上具有明确定义的行为，并且易于阅读 [规范](https://rsocket.io/docs/Protocol) 以及一些协议 [扩展](https://github.com/rsocket/rsocket/tree/master/Extensions)。因此，阅读规范是一个好主意，独立于语言实现和更高级别的框架 API。 本节提供了一个简洁的概述来建立一些上下文。

**Connecting**

**连接**

Initially a client connects to a server via some low level streaming transport such as TCP or WebSocket and sends a `SETUP` frame to the server to set parameters for the connection.

最初，客户端通过一些低级流传输（例如 TCP 或 WebSocket）连接到服务器，并向服务器发送`SETUP`帧以设置连接参数。

The server may reject the `SETUP` frame, but generally after it is sent (for the client) and received (for the server), both sides can begin to make requests, unless `SETUP` indicates use of leasing semantics to limit the number of requests, in which case both sides must wait for a `LEASE` frame from the other end to permit making requests.

服务器可能会拒绝`SETUP`帧，但一般在发送（对于客户端）和接收（对于服务器）之后，双方就可以开始发出请求，除非`SETUP`指示使用租用语义来限制请求数量，在这种情况下，双方必须等待来自另一端的“LEASE”帧以允许发出请求。

**Making Requests**

**发出请求**

Once a connection is established, both sides may initiate a request through one of the frames `REQUEST_RESPONSE`, `REQUEST_STREAM`, `REQUEST_CHANNEL`, or `REQUEST_FNF`. Each of those frames carries one message from the requester to the responder.

一旦建立连接，双方可以通过`REQUEST_RESPONSE`、`REQUEST_STREAM`、`REQUEST_CHANNEL`或`REQUEST_FNF`帧之一发起请求。 这些帧中的每一个都将携带一条消息从请求者传送到响应者。

The responder may then return `PAYLOAD` frames with response messages, and in the case of `REQUEST_CHANNEL` the requester may also send `PAYLOAD` frames with more request messages.

响应者可以返回带有响应消息的`PAYLOAD`帧，并且在`REQUEST_CHANNEL`帧的情况下，请求者也可以发送带有更多请求消息的`PAYLOAD`帧。

When a request involves a stream of messages such as `Request-Stream` and `Channel`, the responder must respect demand signals from the requester. Demand is expressed as a number of messages. Initial demand is specified in `REQUEST_STREAM` and `REQUEST_CHANNEL` frames. Subsequent demand is signaled via `REQUEST_N` frames.

当请求涉及诸如`Request-Stream`和`Channel`之类的消息流时，响应者必须尊重来自请求者的需求信号。 需求表示为多个消息。 初始需求在`REQUEST_STREAM`和`REQUEST_CHANNEL`帧中指定。 随后的需求通过`REQUEST_N`帧发出信号。

Each side may also send metadata notifications, via the `METADATA_PUSH` frame, that do not pertain to any individual request but rather to the connection as a whole.

每一端也可以通过`METADATA_PUSH`帧来发送元数据通知，这些通知与任何单独的请求无关，而是与整个连接有关。

**Message Format**

**消息格式**

RSocket messages contain data and metadata. Metadata can be used to send a route, a security token, etc. Data and metadata can be formatted differently. Mime types for each are declared in the `SETUP` frame and apply to all requests on a given connection.

While all messages can have metadata, typically metadata such as a route are per-request and therefore only included in the first message on a request, i.e. with one of the frames `REQUEST_RESPONSE`, `REQUEST_STREAM`, `REQUEST_CHANNEL`, or `REQUEST_FNF`.

RSocket 消息包含数据和元数据。 元数据可用于发送路由、安全令牌等。数据和元数据的格式可以不同。 每个 Mime 类型都在 `SETUP` 帧中声明，并应用于给定连接上的所有请求。

虽然所有消息都可以具有元数据，但通常元数据（例如路由）是针对每个请求的，因此仅包含在请求的第一条消息中，即`REQUEST_RESPONSE`、`REQUEST_STREAM`、`REQUEST_CHANNEL`或`REQUEST_FNF`帧之一

Protocol extensions define common metadata formats for use in applications:

协议扩展定义了用于应用程序的通用元数据格式：

  - [Composite Metadata](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md)-- multiple, independently formatted metadata entries.

  - [Composite Metadata](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md)-- 多个独立格式化的元数据条目。

  - [Routing](https://github.com/rsocket/rsocket/blob/master/Extensions/Routing.md) — the route for a request.

  - [Routing](https://github.com/rsocket/rsocket/blob/master/Extensions/Routing.md) — 请求路由。

## Java Implementation

## Java实现

The [Java implementation](https://github.com/rsocket/rsocket-java) for RSocket is built on [Project Reactor](https://projectreactor.io/). The transports for TCP and WebSocket are built on [Reactor Netty](https://github.com/reactor/reactor-netty). As a Reactive Streams library, Reactor simplifies the job of implementing the protocol. For applications it is a natural fit to use `Flux` and `Mono` with declarative operators and transparent back pressure support.

The API in RSocket Java is intentionally minimal and basic. It focuses on protocol features and leaves the application programming model (e.g. RPC codegen vs other) as a higher level, independent concern.

The main contract [io.rsocket.RSocket](https://github.com/rsocket/rsocket-java/blob/master/rsocket-core/src/main/java/io/rsocket/RSocket.java) models the four request interaction types with `Mono` representing a promise for a single message, `Flux` a stream of messages, and `io.rsocket.Payload` the actual message with access to data and metadata as byte buffers. The `RSocket` contract is used symmetrically. For requesting, the application is given an `RSocket` to perform requests with. For responding, the application implements `RSocket` to handle requests.

This is not meant to be a thorough introduction. For the most part, Spring applications will not have to use its API directly. However it may be important to see or experiment with RSocket independent of Spring. The RSocket Java repository contains a number of [sample apps](https://github.com/rsocket/rsocket-java/tree/master/rsocket-examples) that demonstrate its API and protocol features.

RSocket 的 [Java 实现](https://github.com/rsocket/rsocket-java) 建立在 [Project Reactor](https://projectreactor.io/) 上。 TCP 和 WebSocket 的传输建立在 [Reactor Netty](https://github.com/reactor/reactor-netty) 上。作为一个 Reactive Streams 库，Reactor 简化了实现协议的工作。对于应用程序，使用带有声明性运算符和透明回压支持的 `Flux` 和 `Mono` 是很自然的选择。

RSocket Java 中的 API 是有意进行最小化和基本化的。它专注于协议特性，并将应用程序编程模型（例如 RPC 代码生成与其他）作为更高级别的独立关注点。

主合约 [io.rsocket.RSocket](https://github.com/rsocket/rsocket-java/blob/master/rsocket-core/src/main/java/io/rsocket/RSocket.java) 模拟了四个请求交互类型，其中 `Mono` 表示单个消息的promise，`Flux` 是消息流，`io.rsocket.Payload` 是实际消息，可以访问数据和元数据作。 `RSocket` 合约是对称使用的。对于请求，应用程序会获得一个`RSocket`来执行请求。对于响应，应用程序实现了 `RSocket` 来处理请求。

这部分仅仅是一个简单的介绍。大多数情况下，Spring 应用程序不必直接使用其 API。然而，查看或试验独立于 Spring 的 RSocket 可能很重要。 RSocket Java 存储库包含许多演示其 API 和协议功能的 [示例应用程序](https://github.com/rsocket/rsocket-java/tree/master/rsocket-examples)。

## Spring Support

The `spring-messaging` module contains the following:

`spring-messaging`模块包括如下内容：

  - [RSocketRequester](#rsocket-requester) — fluent API to make requests through an `io.rsocket.RSocket` with data and metadata encoding/decoding.

  - [RSocketRequester](#rsocket-requester) - 流式API，通过带有数据和元数据编码/解码的 `io.rsocket.RSocket` 发出请求。

  - [Annotated Responders](#rsocket-annot-responders) — `@MessageMapping` annotated handler methods for responding.

  - [Annotated Responders](#rsocket-annot-responders) - `@MessageMapping` 注解处理方法用于响应

The `spring-web` module contains `Encoder` and `Decoder` implementations such as Jackson CBOR/JSON, and Protobuf that RSocket applications will likely need. It also contains the `PathPatternParser` that can be plugged in for efficient route matching.

Spring Boot 2.2 supports standing up an RSocket server over TCP or WebSocket, including the option to expose RSocket over WebSocket in a WebFlux server. There is also client support and auto-configuration for an `RSocketRequester.Builder` and `RSocketStrategies`. See the [RSocket section](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-rsocket) in the Spring Boot reference for more details.

`spring-web` 模块包含 `Encoder` 和 `Decoder` 实现，例如 Jackson CBOR/JSON 和 RSocket 应用程序可能需要的 Protobuf。 它还包含可以插入以进行高效路由匹配的“PathPatternParser”。

Spring Boot 2.2 支持通过 TCP 或 WebSocket 建立 RSocket 服务器，包括在 WebFlux 服务器中通过 WebSocket 公开 RSocket 的选项。 `RSocketRequester.Builder`和`RSocketStrategies`也有客户端支持和自动配置。 有关更多详细信息，请参阅 Spring Boot 参考中的 [RSocket 部分](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-rsocket)。

Spring Security 5.2 provides RSocket support.

Spring Security 5.2 提供了RSocket支持。

Spring Integration 5.2 provides inbound and outbound gateways to interact with RSocket clients and servers. See the Spring Integration Reference Manual for more details.

Spring Integration 5.2 提供了入站和出站网关与 RSocket 客户端和服务器交互的相关内容。 有关更多详细信息，请参阅 Spring 集成参考手册。

Spring Cloud Gateway supports RSocket connections.

Spring Cloud Gateway 支持RSocket连接。

# RSocketRequester

`RSocketRequester` provides a fluent API to perform RSocket requests, accepting and returning objects for data and metadata instead of low level data buffers. It can be used symmetrically, to make requests from clients and to make requests from servers.

`RSocketRequester` 提供了一个流式 API 来执行 RSocket 请求，接受和返回数据和元数据的对象，而不是低级别的数据缓冲区。 它可以对称地使用，从客户端发出请求和从服务器发出请求。

## Client Requester

To obtain an `RSocketRequester` on the client side is to connect to a server which involves sending an RSocket `SETUP` frame with connection settings. `RSocketRequester` provides a builder that helps to prepare an `io.rsocket.core.RSocketConnector` including connection settings for the `SETUP` frame.

This is the most basic way to connect with default settings:

在客户端获得一个 `RSocketRequester` 就是连接到服务器，这涉及发送带有连接设置的 RSocket `SETUP` 帧。 `RSocketRequester` 提供了一个构建器来帮助准备一个 `io.rsocket.core.RSocketConnector`，包括 `SETUP` 帧的连接设置。

这是连接默认设置的最基本方法：

**Java.**

``` java
RSocketRequester requester = RSocketRequester.builder().tcp("localhost", 7000);

URI url = URI.create("https://example.org:8080/rsocket");
RSocketRequester requester = RSocketRequester.builder().webSocket(url);
```

**Kotlin.**

``` kotlin
val requester = RSocketRequester.builder().tcp("localhost", 7000)

URI url = URI.create("https://example.org:8080/rsocket");
val requester = RSocketRequester.builder().webSocket(url)
```

The above does not connect immediately. When requests are made, a shared connection is established transparently and used.

以上不会立即连接。 发出请求时，会透明地建立并使用共享连接。

### Connection Setup

### 连接设置

`RSocketRequester.Builder` provides the following to customize the initial `SETUP` frame:

  - `dataMimeType(MimeType)` — set the mime type for data on the connection.

  - `metadataMimeType(MimeType)` — set the mime type for metadata on the connection.

  - `setupData(Object)` — data to include in the `SETUP`.

  - `setupRoute(String, Object…​)` — route in the metadata to include in the `SETUP`.

  - `setupMetadata(Object, MimeType)` — other metadata to include in the `SETUP`.

`RSocketRequester.Builder` 提供以下内容来自定义初始的 `SETUP` 框架：

   - `dataMimeType(MimeType)` — 设置连接数据的 MIME 类型。

   - `metadataMimeType(MimeType)` — 为连接上的元数据设置 MIME 类型。

   - `setupData(Object)` — 包含在 `SETUP` 中的数据。

   - `setupRoute(String, Object... )` — 元数据中的路由以包含在 `SETUP` 中。

   - `setupMetadata(Object, MimeType)` — 要包含在 `SETUP` 中的其他元数据。

For data, the default mime type is derived from the first configured `Decoder`. For metadata, the default mime type is [composite metadata](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md) which allows multiple metadata value and mime type pairs per request. Typically both don’t need to be changed.

Data and metadata in the `SETUP` frame is optional. On the server side, [@ConnectMapping](#rsocket-annot-connectmapping) methods can be used to handle the start of a connection and the content of the `SETUP` frame. Metadata may be used for connection level security.

对于数据，默认 mime 类型源自第一个配置的`解码器`。 对于元数据，默认的 mime 类型是 [composite metadata](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md)，它允许每个请求有多个元数据值和 mime 类型对。 通常两者都不需要更改。

`SETUP` 帧中的数据和元数据是可选的。 在服务器端，可以使用 [@ConnectMapping](#rsocket-annot-connectmapping) 方法来处理连接的开始和 `SETUP` 帧的内容。 元数据可用于连接级别的安全。

### Strategies

### 策略

`RSocketRequester.Builder` accepts `RSocketStrategies` to configure the requester. You’ll need to use this to provide encoders and decoders for (de)-serialization of data and metadata values. By default only the basic codecs from `spring-core` for `String`, `byte[]`, and `ByteBuffer` are registered. Adding `spring-web` provides access to more that can be registered as follows:


`RSocketRequester.Builder` 接受 `RSocketStrategies` 来配置请求者。 你需要使用它来为数据和元数据值的（反）序列化提供编码器和解码器。 默认情况下，仅注册了来自 `spring-core` 的用于 `String`、`byte[]` 和 `ByteBuffer` 的基本编解码器。 添加 `spring-web` 可以访问更多可以注册的内容，如下所示：

**Java.**

``` java
RSocketStrategies strategies = RSocketStrategies.builder()
    .encoders(encoders -> encoders.add(new Jackson2CborEncoder()))
    .decoders(decoders -> decoders.add(new Jackson2CborDecoder()))
    .build();

RSocketRequester requester = RSocketRequester.builder()
    .rsocketStrategies(strategies)
    .tcp("localhost", 7000);
```

**Kotlin.**

``` kotlin
val strategies = RSocketStrategies.builder()
        .encoders { it.add(Jackson2CborEncoder()) }
        .decoders { it.add(Jackson2CborDecoder()) }
        .build()

val requester = RSocketRequester.builder()
        .rsocketStrategies(strategies)
        .tcp("localhost", 7000)
```

`RSocketStrategies` is designed for re-use. In some scenarios, e.g. client and server in the same application, it may be preferable to declare it in Spring configuration.

`RSocketStrategies` 是为重复使用而设计的。 在某些情况下，例如 客户端和服务器在同一个应用程序中，最好在 Spring 配置中声明它。

### Client Responders

`RSocketRequester.Builder` can be used to configure responders to requests from the server.

You can use annotated handlers for client-side responding based on the same infrastructure that’s used on a server, but registered programmatically as follows:

`RSocketRequester.Builder` 可用于配置响应来自服务器的请求。

你可以基于在服务器上使用相同的结构使用带注解的handler进行客户端响应，但以编程方式注册如下：

**Java.**

``` java
RSocketStrategies strategies = RSocketStrategies.builder()
    .routeMatcher(new PathPatternRouteMatcher())  
    .build();

SocketAcceptor responder =
    RSocketMessageHandler.responder(strategies, new ClientHandler()); 

RSocketRequester requester = RSocketRequester.builder()
    .rsocketConnector(connector -> connector.acceptor(responder)) 
    .tcp("localhost", 7000);
```

  - Use `PathPatternRouteMatcher`, if `spring-web` is present, for efficient route matching.

  - Create a responder from a class with `@MessageMaping` and/or `@ConnectMapping` methods.

  - Register the responder.

  - 如果存在`spring-web`，则使用`PathPatternRouteMatcher` 以实现高效的路由匹配。

  - 从具有`@MessageMaping` 和/或`@ConnectMapping` 方法的类创建响应者。

  - 注册响应者。

**Kotlin.**

``` kotlin
val strategies = RSocketStrategies.builder()
        .routeMatcher(PathPatternRouteMatcher())  
        .build()

val responder =
    RSocketMessageHandler.responder(strategies, new ClientHandler()); 

val requester = RSocketRequester.builder()
        .rsocketConnector { it.acceptor(responder) } 
        .tcp("localhost", 7000)
```

  - Use `PathPatternRouteMatcher`, if `spring-web` is present, for efficient route matching.

  - Create a responder from a class with `@MessageMaping` and/or `@ConnectMapping` methods.

  - Register the responder.

  - 如果存在`spring-web`，则使用`PathPatternRouteMatcher` 以实现高效的路由匹配。

  - 从具有`@MessageMaping` 和/或`@ConnectMapping` 方法的类创建响应者。

  - 注册响应者。

Note the above is only a shortcut designed for programmatic registration of client responders. For alternative scenarios, where client responders are in Spring configuration, you can still declare `RSocketMessageHandler` as a Spring bean and then apply as follows:

请注意，以上只是为客户端响应程序的编程注册而设计的快捷方式。 对于客户端响应程序在 Spring 配置中的替代方案，你仍然可以将 `RSocketMessageHandler` 声明为 Spring bean，然后按如下方式使用：

**Java.**

``` java
ApplicationContext context = ... ;
RSocketMessageHandler handler = context.getBean(RSocketMessageHandler.class);

RSocketRequester requester = RSocketRequester.builder()
    .rsocketConnector(connector -> connector.acceptor(handler.responder()))
    .tcp("localhost", 7000);
```

**Kotlin.**

``` kotlin
import org.springframework.beans.factory.getBean

val context: ApplicationContext = ...
val handler = context.getBean<RSocketMessageHandler>()

val requester = RSocketRequester.builder()
        .rsocketConnector { it.acceptor(handler.responder()) }
        .tcp("localhost", 7000)
```

For the above you may also need to use `setHandlerPredicate` in `RSocketMessageHandler` to switch to a different strategy for detecting client responders, e.g. based on a custom annotation such as `@RSocketClientResponder` vs the default `@Controller`. This is necessary in scenarios with client and server, or multiple clients in the same application.

See also [Annotated Responders](#rsocket-annot-responders), for more on the programming model.

对于上述情况，你可能还需要在 `RSocketMessageHandler` 中使用 `setHandlerPredicate` 来切换到检测客户端响应者的不同策略，例如基于自定义注解，像`@RSocketClientResponder` 与默认的 `@Controller`。 这在客户端和服务器，或同一应用程序中的多个客户端的场景中是必要的。

有关编程模型的更多信息，另请参见 [Annotated Responders](#rsocket-annot-responders)。

### Advanced

`RSocketRequesterBuilder` provides a callback to expose the underlying `io.rsocket.core.RSocketConnector` for further configuration options for keepalive intervals, session resumption, interceptors, and more. You can configure options at that level as follows:

`RSocketRequesterBuilder` 提供了一个回调来公开底层的 `io.rsocket.core.RSocketConnector`，以提供保活间隔、会话恢复、拦截器等的进一步配置选项。 你可以在该级别配置选项，如下所示：

**Java.**

``` java
RSocketRequester requester = RSocketRequester.builder()
    .rsocketConnector(connector -> {
        // ...
    })
    .tcp("localhost", 7000);
```

**Kotlin.**

``` kotlin
val requester = RSocketRequester.builder()
        .rsocketConnector {
            //...
        }
        .tcp("localhost", 7000)
```

## Server Requester

To make requests from a server to connected clients is a matter of obtaining the requester for the connected client from the server.

In [Annotated Responders](#rsocket-annot-responders), `@ConnectMapping` and `@MessageMapping` methods support an `RSocketRequester` argument. Use it to access the requester for the connection. Keep in mind that `@ConnectMapping` methods are essentially handlers of the `SETUP` frame which must be handled before requests can begin. Therefore, requests at the very start must be decoupled from handling. For example:

从服务器向连接的客户端发出请求需要获取连接客户端的请求者。

在 [Annotated Responders](#rsocket-annot-responders) 中，`@ConnectMapping` 和 `@MessageMapping` 方法支持 `RSocketRequester` 参数。 使用它来访问连接的请求者。 请记住，`@ConnectMapping` 方法本质上是 `SETUP` 帧的处理程序，必须在请求开始之前处理。 因此，一开始的请求必须与处理分离。 例如：

**Java.**

``` java
@ConnectMapping
Mono<Void> handle(RSocketRequester requester) {
    requester.route("status").data("5")
        .retrieveFlux(StatusReport.class)
        .subscribe(bar -> { 
            // ...
        });
    return ... 
}
```

  - Start the request asynchronously, independent from handling.

  - 异步开始请求，与处理相独立

  - Perform handling and return completion `Mono<Void>`.

  - 进行处理并返回`Mono<Void>`

**Kotlin.**

``` kotlin
@ConnectMapping
suspend fun handle(requester: RSocketRequester) {
    GlobalScope.launch {
        requester.route("status").data("5").retrieveFlow<StatusReport>().collect { 
            // ...
        }
    }
    /// ... 
}
```

  - Start the request asynchronously, independent from handling.

  - 异步开始请求，与处理相独立

  - Perform handling in the suspending function.

  - 进行处理并返回`Mono<Void>`

## Requests

Once you have a [client](#rsocket-requester-client) or [server](#rsocket-requester-server) requester, you can make requests as follows:

一旦你有一个 [client](#rsocket-requester-client) 或 [server](#rsocket-requester-server) 请求者，你可以发出如下请求：

**Java.**

``` java
ViewBox viewBox = ... ;

Flux<AirportLocation> locations = requester.route("locate.radars.within") 
        .data(viewBox) 
        .retrieveFlux(AirportLocation.class); 
```

  - Specify a route to include in the metadata of the request message.

  - Provide data for the request message.

  - Declare the expected response.

  - 指定要包含在请求消息的元数据中的路由。

  - 为请求消息提供数据。

  - 声明预期的响应。 

**Kotlin.**

``` kotlin
val viewBox: ViewBox = ...

val locations = requester.route("locate.radars.within") 
        .data(viewBox) 
        .retrieveFlow<AirportLocation>() 
```

  - Specify a route to include in the metadata of the request message.

  - Provide data for the request message.

  - Declare the expected response.

  - 指定要包含在请求消息的元数据中的路由。

  - 为请求消息提供数据。

  - 声明预期的响应。 

The interaction type is determined implicitly from the cardinality of the input and output. The above example is a `Request-Stream` because one value is sent and a stream of values is received. For the most part you don’t need to think about this as long as the choice of input and output matches an RSocket interaction type and the types of input and output expected by the responder. The only example of an invalid combination is many-to-one.

The `data(Object)` method also accepts any Reactive Streams `Publisher`, including `Flux` and `Mono`, as well as any other producer of value(s) that is registered in the `ReactiveAdapterRegistry`. For a multi-value `Publisher` such as `Flux` which produces the same types of values, consider using one of the overloaded `data` methods to avoid having type checks and `Encoder` lookup on every element:

交互类型由输入和输出的基数隐式确定。 上面的例子是一个`Request-Stream`，因为一个值被发送，一个值的流被接收。 大多数情况下，只要输入和输出的选择与 RSocket 交互类型以及响应者期望的输入和输出类型相匹配，你就不需要考虑这一点。 无效组合的唯一示例是多对一。

`data(Object)` 方法还接受任何 Reactive Streams `Publisher`，包括 `Flux` 和 `Mono`，以及在 `ReactiveAdapterRegistry` 中注册的任何其他值的生产者。 对于产生相同类型值的多值 `Publisher`，例如 `Flux`，考虑使用重载的 `data` 方法之一，以避免对每个元素进行类型检查和 `Encoder` 查找：

``` java
data(Object producer, Class<?> elementClass);
data(Object producer, ParameterizedTypeReference<?> elementTypeRef);
```

The `data(Object)` step is optional. Skip it for requests that don’t send data:

`data(Object)` 步骤是可选的。 对于不发送数据的请求，跳过它：

**Java.**

``` java
Mono<AirportLocation> location = requester.route("find.radar.EWR"))
    .retrieveMono(AirportLocation.class);
```

**Kotlin.**

``` kotlin
import org.springframework.messaging.rsocket.retrieveAndAwait

val location = requester.route("find.radar.EWR")
    .retrieveAndAwait<AirportLocation>()
```

Extra metadata values can be added if using [composite metadata](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md) (the default) and if the values are supported by a registered `Encoder`. For example:

如果使用 [composite metadata](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md)（默认值）并且这些值受注册的`Encoder` 支持，则可以添加额外的元数据值. 例如：

**Java.**

``` java
String securityToken = ... ;
ViewBox viewBox = ... ;
MimeType mimeType = MimeType.valueOf("message/x.rsocket.authentication.bearer.v0");

Flux<AirportLocation> locations = requester.route("locate.radars.within")
        .metadata(securityToken, mimeType)
        .data(viewBox)
        .retrieveFlux(AirportLocation.class);
```

**Kotlin.**

``` kotlin
import org.springframework.messaging.rsocket.retrieveFlow

val requester: RSocketRequester = ...

val securityToken: String = ...
val viewBox: ViewBox = ...
val mimeType = MimeType.valueOf("message/x.rsocket.authentication.bearer.v0")

val locations = requester.route("locate.radars.within")
        .metadata(securityToken, mimeType)
        .data(viewBox)
        .retrieveFlow<AirportLocation>()
```

For `Fire-and-Forget` use the `send()` method that returns `Mono<Void>`. Note that the `Mono` indicates only that the message was successfully sent, and not that it was handled.

For `Metadata-Push` use the `sendMetadata()` method with a `Mono<Void>` return value.

对于`Fire-and-Forget`，使用返回`Mono<Void>`的`send()`方法。 请注意，`Mono` 仅表示消息已成功发送，而不表示已被处理。

对于 `Metadata-Push`，使用带有 `Mono<Void>` 返回值的 `sendMetadata()` 方法。

# Annotated Responders

RSocket responders can be implemented as `@MessageMapping` and `@ConnectMapping` methods. `@MessageMapping` methods handle individual requests while `@ConnectMapping` methods handle connection-level events (setup and metadata push). Annotated responders are supported symmetrically, for responding from the server side and for responding from the client side.

RSocket 响应者可以实现为 `@MessageMapping` 和 `@ConnectMapping` 方法。 `@MessageMapping` 方法处理单个请求，而 `@ConnectMapping` 方法处理连接级别的事件（设置和元数据推送）。 当然也支持带注解的响应者，用于从服务器端响应和从客户端响应。

## Server Responders

To use annotated responders on the server side, add `RSocketMessageHandler` to your Spring configuration to detect `@Controller` beans with `@MessageMapping` and `@ConnectMapping` methods:

要在服务器端使用带注解的响应者，需要将 `RSocketMessageHandler` 添加到你的 Spring 配置中，以使用 `@MessageMapping` 和 `@ConnectMapping` 方法检测 `@Controller` bean：

**Java.**

``` java
@Configuration
static class ServerConfig {

    @Bean
    public RSocketMessageHandler rsocketMessageHandler() {
        RSocketMessageHandler handler = new RSocketMessageHandler();
        handler.routeMatcher(new PathPatternRouteMatcher());
        return handler;
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
class ServerConfig {

    @Bean
    fun rsocketMessageHandler() = RSocketMessageHandler().apply {
        routeMatcher = PathPatternRouteMatcher()
    }
}
```

Then start an RSocket server through the Java RSocket API and plug the `RSocketMessageHandler` for the responder as follows:

然后通过Java RSocket API启动一个RSocket服务器，并为响应者配置`RSocketMessageHandler`，如下所示：

**Java.**

``` java
ApplicationContext context = ... ;
RSocketMessageHandler handler = context.getBean(RSocketMessageHandler.class);

CloseableChannel server =
    RSocketServer.create(handler.responder())
        .bind(TcpServerTransport.create("localhost", 7000))
        .block();
```

**Kotlin.**

``` kotlin
import org.springframework.beans.factory.getBean

val context: ApplicationContext = ...
val handler = context.getBean<RSocketMessageHandler>()

val server = RSocketServer.create(handler.responder())
        .bind(TcpServerTransport.create("localhost", 7000))
        .awaitSingle()
```

`RSocketMessageHandler` supports [composite](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md) and [routing](https://github.com/rsocket/rsocket/blob/master/Extensions/Routing.md) metadata by default. You can set its [MetadataExtractor](#rsocket-metadata-extractor) if you need to switch to a different mime type or register additional metadata mime types.

You’ll need to set the `Encoder` and `Decoder` instances required for metadata and data formats to support. You’ll likely need the `spring-web` module for codec implementations.

By default `SimpleRouteMatcher` is used for matching routes via `AntPathMatcher`. We recommend plugging in the `PathPatternRouteMatcher` from `spring-web` for efficient route matching. RSocket routes can be hierarchical but are not URL paths. Both route matchers are configured to use "." as separator by default and there is no URL decoding as with HTTP URLs.

`RSocketMessageHandler` can be configured via `RSocketStrategies` which may be useful if you need to share configuration between a client and a server in the same process:

`RSocketMessageHandler` 默认支持 [composite](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md) 和 [routing](https://github.com/rsocket/rsocket/blob/master/Extensions/Routing.md) 元数据。如果你需要使用不同的 mime 类型或注册其他元数据 mime 类型，你可以设置其 [MetadataExtractor](#rsocket-metadata-extractor)。

你需要设置支持元数据和数据格式所需的 `Encoder` 和 `Decoder` 实例。你可能需要用于编解码器实现的 `spring-web` 模块。

默认情况下，`SimpleRouteMatcher` 用于通过 `AntPathMatcher` 匹配路由。我们建议配置来自 `spring-web` 的 `PathPatternRouteMatcher` 以实现高效的路由匹配。 RSocket 路由可以是分层的，但不是 URL 路径。两个路由匹配器在默认情况下都配置为使用“.”作为分隔符，并且没有像 HTTP URL 那样的 URL 解码。

`RSocketMessageHandler` 可以通过 `RSocketStrategies` 进行配置，如果你需要在同一进程中的客户端和服务器之间共享配置：

**Java.**

``` java
@Configuration
static class ServerConfig {

    @Bean
    public RSocketMessageHandler rsocketMessageHandler() {
        RSocketMessageHandler handler = new RSocketMessageHandler();
        handler.setRSocketStrategies(rsocketStrategies());
        return handler;
    }

    @Bean
    public RSocketStrategies rsocketStrategies() {
        return RSocketStrategies.builder()
            .encoders(encoders -> encoders.add(new Jackson2CborEncoder()))
            .decoders(decoders -> decoders.add(new Jackson2CborDecoder()))
            .routeMatcher(new PathPatternRouteMatcher())
            .build();
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
class ServerConfig {

    @Bean
    fun rsocketMessageHandler() = RSocketMessageHandler().apply {
        rSocketStrategies = rsocketStrategies()
    }

    @Bean
    fun rsocketStrategies() = RSocketStrategies.builder()
            .encoders { it.add(Jackson2CborEncoder()) }
            .decoders { it.add(Jackson2CborDecoder()) }
            .routeMatcher(PathPatternRouteMatcher())
            .build()
}
```

## Client Responders

Annotated responders on the client side need to be configured in the `RSocketRequester.Builder`. For details, see [Client Responders](#rsocket-requester-client-responder).

客户端的带注解的响应者需要在 `RSocketRequester.Builder` 中进行配置。 详情请参见[Client Responders](#rsocket-requester-client-responder)。


## @MessageMapping

Once [server](#rsocket-annot-responders-server) or [client](#rsocket-annot-responders-client) responder configuration is in place, `@MessageMapping` methods can be used as follows:

服务端[server](#rsocket-annot-responders-server) 或者客户端[client](#rsocket-annot-responders-client) 配置好之后， `@MessageMapping` 方法可按照如下方法使用：

**Java.**

``` java
@Controller
public class RadarsController {

    @MessageMapping("locate.radars.within")
    public Flux<AirportLocation> radars(MapRequest request) {
        // ...
    }
}
```

**Kotlin.**

``` kotlin
@Controller
class RadarsController {

    @MessageMapping("locate.radars.within")
    fun radars(request: MapRequest): Flow<AirportLocation> {
        // ...
    }
}
```

The above `@MessageMapping` method responds to a Request-Stream interaction having the route "locate.radars.within". It supports a flexible method signature with the option to use the following method arguments:

上述`@MessageMapping` 方法处理具有路由“locate.radars.within”的Request-Stream交互请求。 它支持灵活的方法签名，可以选择使用以下方法参数： 

<table>
<colgroup>
<col style="width: 25%" />
<col style="width: 75%" />
</colgroup>
<thead>
<tr class="header">
<th>Method Argument</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>@Payload</code></p></td>
<td><p>The payload of the request. This can be a concrete value of asynchronous types like <code>Mono</code> or <code>Flux</code>.</p>
<p><strong>Note:</strong> Use of the annotation is optional. A method argument that is not a simple type and is not any of the other supported arguments, is assumed to be the expected payload.</p></td>
</tr>
<tr class="even">
<td><p><code>RSocketRequester</code></p></td>
<td><p>Requester for making requests to the remote end.</p></td>
</tr>
<tr class="odd">
<td><p><code>@DestinationVariable</code></p></td>
<td><p>Value extracted from the route based on variables in the mapping pattern, e.g. <code>@MessageMapping(&quot;find.radar.{id}&quot;)</code>.</p></td>
</tr>
<tr class="even">
<td><p><code>@Header</code></p></td>
<td><p>Metadata value registered for extraction as described in <a href="#rsocket-metadata-extractor">section_title</a>.</p></td>
</tr>
<tr class="odd">
<td><p><code>@Headers Map&lt;String, Object&gt;</code></p></td>
<td><p>All metadata values registered for extraction as described in <a href="#rsocket-metadata-extractor">section_title</a>.</p></td>
</tr>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 25%" />
<col style="width: 75%" />
</colgroup>
<thead>
<tr class="header">
<th>Method Argument</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>@Payload</code></p></td>
<td><p>请求负载。 可以是异步类型，如 <code>Mono</code> 或者 <code>Flux</code>.</p>
<p><strong>备注:</strong> 使用注解是可选的。 如果一个方法参数不是简单类型并且不是其他任何支持的参数方法, 都被假定为预期的负载。</p></td>
</tr>
<tr class="even">
<td><p><code>RSocketRequester</code></p></td>
<td><p>向远端发出请求的请求者。</p></td>
</tr>
<tr class="odd">
<td><p><code>@DestinationVariable</code></p></td>
<td><p>基于映射模式中的变量从路由中提取的值，例如：<code>@MessageMapping(&quot;find.radar.{id}&quot;)</code>.</p></td>
</tr>
<tr class="even">
<td><p><code>@Header</code></p></td>
<td><p>为提取而注册的元数据值，详细描述： <a href="#rsocket-metadata-extractor">MetadataExtractor</a>.</p></td>
</tr>
<tr class="odd">
<td><p><code>@Headers Map&lt;String, Object&gt;</code></p></td>
<td><p>为提取而注册的元数据值，详细描述：<a href="#rsocket-metadata-extractor">MetadataExtractor</a>.</p></td>
</tr>
</tbody>
</table>

The return value is expected to be one or more Objects to be serialized as response payloads. That can be asynchronous types like `Mono` or `Flux`, a concrete value, or either `void` or a no-value asynchronous type such as `Mono<Void>`.

The RSocket interaction type that an `@MessageMapping` method supports is determined from the cardinality of the input (i.e. `@Payload` argument) and of the output, where cardinality means the following:

预期返回值是一个或多个要序列化为响应负载的对象。 这可以是异步类型，如`Mono` 或`Flux`，一个具体值，或者`void` 或无值异步类型，如`Mono<Void>`。

`@MessageMapping` 方法支持的 RSocket 交互类型由输入（即 `@Payload` 参数）和输出的基数确定，其中基数的含义如下：

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th>Cardinality</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p>1</p></td>
<td><p>Either an explicit value, or a single-value asynchronous type such as <code>Mono&lt;T&gt;</code>.</p></td>
</tr>
<tr class="even">
<td><p>Many</p></td>
<td><p>A multi-value asynchronous type such as <code>Flux&lt;T&gt;</code>.</p></td>
</tr>
<tr class="odd">
<td><p>0</p></td>
<td><p>For input this means the method does not have an <code>@Payload</code> argument.</p>
<p>For output this is <code>void</code> or a no-value asynchronous type such as <code>Mono&lt;Void&gt;</code>.</p></td>
</tr>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th>基数</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p>1</p></td>
<td><p>或者是一个明确的值，或者是一个单值的异步类型，如 <code>Mono&lt;T&gt;</code>.</p></td>
</tr>
<tr class="even">
<td><p>Many</p></td>
<td><p>多值异步类型，如 <code>Flux&lt;T&gt;</code>.</p></td>
</tr>
<tr class="odd">
<td><p>0</p></td>
<td><p>对于输入，这表示这个方法没有 <code>@Payload</code> 参数。</p>
<p>对于输出 <code>void</code> or 一个不带有值的异步类型，如 <code>Mono&lt;Void&gt;</code>.</p></td>
</tr>
</tbody>
</table>

The table below shows all input and output cardinality combinations and the corresponding interaction type(s):

下表显示了所有输入和输出基数组合以及相应的交互类型：

| Input Cardinality | Output Cardinality | Interaction Types                 |
| ----------------- | ------------------ | --------------------------------- |
| 0, 1              | 0                  | Fire-and-Forget, Request-Response |
| 0, 1              | 1                  | Request-Response                  |
| 0, 1              | Many               | Request-Stream                    |
| Many              | 0, 1, Many         | Request-Channel                   |

## @ConnectMapping

`@ConnectMapping` handles the `SETUP` frame at the start of an RSocket connection, and any subsequent metadata push notifications through the `METADATA_PUSH` frame, i.e. `metadataPush(Payload)` in `io.rsocket.RSocket`.

`@ConnectMapping` methods support the same arguments as [@MessageMapping](#rsocket-annot-messagemapping) but based on metadata and data from the `SETUP` and `METADATA_PUSH` frames. `@ConnectMapping` can have a pattern to narrow handling to specific connections that have a route in the metadata, or if no patterns are declared then all connections match.

`@ConnectMapping` methods cannot return data and must be declared with `void` or `Mono<Void>` as the return value. If handling returns an error for a new connection then the connection is rejected. Handling must not be held up to make requests to the `RSocketRequester` for the connection. See [Server Requester](#rsocket-requester-server) for details.

`@ConnectMapping` 处理 RSocket 连接开始时的 `SETUP` 帧，以及通过 `METADATA_PUSH` 帧的任何后续元数据推送通知，即 `io.rsocket.RSocket` 中的 `metadataPush(Payload)`。

`@ConnectMapping` 方法支持与 [@MessageMapping](#rsocket-annot-messagemapping) 相同的参数，但基于来自 `SETUP` 和 `METADATA_PUSH` 帧的元数据和数据。 `@ConnectMapping` 可以有一个模式，将处理范围缩小到在元数据中有路由的特定连接，或者如果没有声明任何模式，则所有连接都匹配。

`@ConnectMapping` 方法不能返回数据，必须使用 `void` 或 `Mono<Void>` 作为返回值声明。 如果新连接处理错误，则拒绝连接。 处理不能阻止向 `RSocketRequester` 发出连接请求。 有关详细信息，请参阅 [Server Requester](#rsocket-requester-server)。

# MetadataExtractor

Responders must interpret metadata. [Composite metadata](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md) allows independently formatted metadata values (e.g. for routing, security, tracing) each with its own mime type. Applications need a way to configure metadata mime types to support, and a way to access extracted values.

`MetadataExtractor` is a contract to take serialized metadata and return decoded name-value pairs that can then be accessed like headers by name, for example via `@Header` in annotated handler methods.

`DefaultMetadataExtractor` can be given `Decoder` instances to decode metadata. Out of the box it has built-in support for ["message/x.rsocket.routing.v0"](https://github.com/rsocket/rsocket/blob/master/Extensions/Routing.md) which it decodes to `String` and saves under the "route" key. For any other mime type you’ll need to provide a `Decoder` and register the mime type as follows:

响应者必须理解元数据。 [Composite metadata](https://github.com/rsocket/rsocket/blob/master/Extensions/CompositeMetadata.md) 允许独立格式化的元数据值（例如用于路由、安全、跟踪），每个值都有自己的 mime 类型。 应用程序需要一种方法来配置要支持的元数据 mime 类型，以及一种访问提取值的方法。

`MetadataExtractor` 是一个契约，用于获取序列化的元数据并返回解码的名称-值对，然后可以像首部一样按名称访问，例如通过带注释的处理程序方法中的 `@Header`。

`DefaultMetadataExtractor` 可以被注入到 `Decoder` 实例来解码元数据。 它内置了对 ["message/x.rsocket.routing.v0"](https://github.com/rsocket/rsocket/blob/master/Extensions/Routing.md) 解码的支持，它解码为`String`类型并保存在“route”键下。 对于任何其他 mime 类型，你需要提供一个 `Decoder` 并按如下方式注册 mime 类型：

**Java.**

``` java
DefaultMetadataExtractor extractor = new DefaultMetadataExtractor(metadataDecoders);
extractor.metadataToExtract(fooMimeType, Foo.class, "foo");
```

**Kotlin.**

``` kotlin
import org.springframework.messaging.rsocket.metadataToExtract

val extractor = DefaultMetadataExtractor(metadataDecoders)
extractor.metadataToExtract<Foo>(fooMimeType, "foo")
```

Composite metadata works well to combine independent metadata values. However the requester might not support composite metadata, or may choose not to use it. For this, `DefaultMetadataExtractor` may needs custom logic to map the decoded value to the output map. Here is an example where JSON is used for metadata:

复合元数据可以很好地组合独立的元数据值。 然而，请求者可能不支持复合元数据，或者可能选择不使用它。 为此，`DefaultMetadataExtractor` 可能需要自定义逻辑来将解码值映射到输出映射。 以下是将 JSON 用于元数据的示例：

**Java.**

``` java
DefaultMetadataExtractor extractor = new DefaultMetadataExtractor(metadataDecoders);
extractor.metadataToExtract(
    MimeType.valueOf("application/vnd.myapp.metadata+json"),
    new ParameterizedTypeReference<Map<String,String>>() {},
    (jsonMap, outputMap) -> {
        outputMap.putAll(jsonMap);
    });
```

**Kotlin.**

``` kotlin
import org.springframework.messaging.rsocket.metadataToExtract

val extractor = DefaultMetadataExtractor(metadataDecoders)
extractor.metadataToExtract<Map<String, String>>(MimeType.valueOf("application/vnd.myapp.metadata+json")) { jsonMap, outputMap ->
    outputMap.putAll(jsonMap)
}
```

When configuring `MetadataExtractor` through `RSocketStrategies`, you can let `RSocketStrategies.Builder` create the extractor with the configured decoders, and simply use a callback to customize registrations as follows:

当通过 `RSocketStrategies` 配置 `MetadataExtractor` 时，你可以让 `RSocketStrategies.Builder` 使用配置的解码器创建提取器，只需使用回调来自定义注册，如下所示：

**Java.**

``` java
RSocketStrategies strategies = RSocketStrategies.builder()
    .metadataExtractorRegistry(registry -> {
        registry.metadataToExtract(fooMimeType, Foo.class, "foo");
        // ...
    })
    .build();
```

**Kotlin.**

``` kotlin
import org.springframework.messaging.rsocket.metadataToExtract

val strategies = RSocketStrategies.builder()
        .metadataExtractorRegistry { registry: MetadataExtractorRegistry ->
            registry.metadataToExtract<Foo>(fooMimeType, "foo")
            // ...
        }
        .build()
```
