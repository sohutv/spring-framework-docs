The original web framework included in the Spring Framework, Spring Web MVC, was purpose-built for the Servlet API and Servlet containers. The reactive-stack web framework, Spring WebFlux, was added later in version 5.0. It is fully non-blocking, supports [Reactive Streams](https://www.reactive-streams.org/) back pressure, and runs on such servers as Netty, Undertow, and Servlet 3.1+ containers.

Both web frameworks mirror the names of their source modules ({spring-framework-main-code}/spring-webmvc\[spring-webmvc\] and {spring-framework-main-code}/spring-webflux\[spring-webflux\]) and co-exist side by side in the Spring Framework. Each module is optional. Applications can use one or the other module or, in some cases, both — for example, Spring MVC controllers with the reactive `WebClient`.

# Overview

Why was Spring WebFlux created?

Part of the answer is the need for a non-blocking web stack to handle concurrency with a small number of threads and scale with fewer hardware resources. Servlet 3.1 did provide an API for non-blocking I/O. However, using it leads away from the rest of the Servlet API, where contracts are synchronous (`Filter`, `Servlet`) or blocking (`getParameter`, `getPart`). This was the motivation for a new common API to serve as a foundation across any non-blocking runtime. That is important because of servers (such as Netty) that are well-established in the async, non-blocking space.

The other part of the answer is functional programming. Much as the addition of annotations in Java 5 created opportunities (such as annotated REST controllers or unit tests), the addition of lambda expressions in Java 8 created opportunities for functional APIs in Java. This is a boon for non-blocking applications and continuation-style APIs (as popularized by `CompletableFuture` and [ReactiveX](http://reactivex.io/)) that allow declarative composition of asynchronous logic. At the programming-model level, Java 8 enabled Spring WebFlux to offer functional web endpoints alongside annotated controllers.

## Define “Reactive”

We touched on “non-blocking” and “functional” but what does reactive mean?

The term, “reactive,” refers to programming models that are built around reacting to change — network components reacting to I/O events, UI controllers reacting to mouse events, and others. In that sense, non-blocking is reactive, because, instead of being blocked, we are now in the mode of reacting to notifications as operations complete or data becomes available.

There is also another important mechanism that we on the Spring team associate with “reactive” and that is non-blocking back pressure. In synchronous, imperative code, blocking calls serve as a natural form of back pressure that forces the caller to wait. In non-blocking code, it becomes important to control the rate of events so that a fast producer does not overwhelm its destination.

Reactive Streams is a [small spec](https://github.com/reactive-streams/reactive-streams-jvm/blob/master/README.md#specification) (also [adopted](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html) in Java 9) that defines the interaction between asynchronous components with back pressure. For example a data repository (acting as [Publisher](https://www.reactive-streams.org/reactive-streams-1.0.1-javadoc/org/reactivestreams/Publisher.html)) can produce data that an HTTP server (acting as [Subscriber](https://www.reactive-streams.org/reactive-streams-1.0.1-javadoc/org/reactivestreams/Subscriber.html)) can then write to the response. The main purpose of Reactive Streams is to let the subscriber control how quickly or how slowly the publisher produces data.

> **Note**
> 
> **Common question: what if a publisher cannot slow down?**  
> The purpose of Reactive Streams is only to establish the mechanism and a boundary. If a publisher cannot slow down, it has to decide whether to buffer, drop, or fail.

## Reactive API

Reactive Streams plays an important role for interoperability. It is of interest to libraries and infrastructure components but less useful as an application API, because it is too low-level. Applications need a higher-level and richer, functional API to compose async logic — similar to the Java 8 `Stream` API but not only for collections. This is the role that reactive libraries play.

[Reactor](https://github.com/reactor/reactor) is the reactive library of choice for Spring WebFlux. It provides the [`Mono`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html) and [`Flux`](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html) API types to work on data sequences of 0..1 (`Mono`) and 0..N (`Flux`) through a rich set of operators aligned with the ReactiveX [vocabulary of operators](http://reactivex.io/documentation/operators.html). Reactor is a Reactive Streams library and, therefore, all of its operators support non-blocking back pressure. Reactor has a strong focus on server-side Java. It is developed in close collaboration with Spring.

WebFlux requires Reactor as a core dependency but it is interoperable with other reactive libraries via Reactive Streams. As a general rule, a WebFlux API accepts a plain `Publisher` as input, adapts it to a Reactor type internally, uses that, and returns either a `Flux` or a `Mono` as output. So, you can pass any `Publisher` as input and you can apply operations on the output, but you need to adapt the output for use with another reactive library. Whenever feasible (for example, annotated controllers), WebFlux adapts transparently to the use of RxJava or another reactive library. See [???](#webflux-reactive-libraries) for more details.

> **Note**
> 
> In addition to Reactive APIs, WebFlux can also be used with [Coroutines](languages.xml#coroutines) APIs in Kotlin which provides a more imperative style of programming. The following Kotlin code samples will be provided with Coroutines APIs.

## Programming Models

The `spring-web` module contains the reactive foundation that underlies Spring WebFlux, including HTTP abstractions, Reactive Streams [adapters](#webflux-httphandler) for supported servers, [codecs](#webflux-codecs), and a core [section\_title](#webflux-web-handler-api) comparable to the Servlet API but with non-blocking contracts.

On that foundation, Spring WebFlux provides a choice of two programming models:

  - [section\_title](#webflux-controller): Consistent with Spring MVC and based on the same annotations from the `spring-web` module. Both Spring MVC and WebFlux controllers support reactive (Reactor and RxJava) return types, and, as a result, it is not easy to tell them apart. One notable difference is that WebFlux also supports reactive `@RequestBody` arguments.

  - [section\_title](#webflux-fn): Lambda-based, lightweight, and functional programming model. You can think of this as a small library or a set of utilities that an application can use to route and handle requests. The big difference with annotated controllers is that the application is in charge of request handling from start to finish versus declaring intent through annotations and being called back.

## Applicability

Spring MVC or WebFlux?

A natural question to ask but one that sets up an unsound dichotomy. Actually, both work together to expand the range of available options. The two are designed for continuity and consistency with each other, they are available side by side, and feedback from each side benefits both sides. The following diagram shows how the two relate, what they have in common, and what each supports uniquely:

![spring mvc and webflux venn](images/spring-mvc-and-webflux-venn.png)

We suggest that you consider the following specific points:

  - If you have a Spring MVC application that works fine, there is no need to change. Imperative programming is the easiest way to write, understand, and debug code. You have maximum choice of libraries, since, historically, most are blocking.

  - If you are already shopping for a non-blocking web stack, Spring WebFlux offers the same execution model benefits as others in this space and also provides a choice of servers (Netty, Tomcat, Jetty, Undertow, and Servlet 3.1+ containers), a choice of programming models (annotated controllers and functional web endpoints), and a choice of reactive libraries (Reactor, RxJava, or other).

  - If you are interested in a lightweight, functional web framework for use with Java 8 lambdas or Kotlin, you can use the Spring WebFlux functional web endpoints. That can also be a good choice for smaller applications or microservices with less complex requirements that can benefit from greater transparency and control.

  - In a microservice architecture, you can have a mix of applications with either Spring MVC or Spring WebFlux controllers or with Spring WebFlux functional endpoints. Having support for the same annotation-based programming model in both frameworks makes it easier to re-use knowledge while also selecting the right tool for the right job.

  - A simple way to evaluate an application is to check its dependencies. If you have blocking persistence APIs (JPA, JDBC) or networking APIs to use, Spring MVC is the best choice for common architectures at least. It is technically feasible with both Reactor and RxJava to perform blocking calls on a separate thread but you would not be making the most of a non-blocking web stack.

  - If you have a Spring MVC application with calls to remote services, try the reactive `WebClient`. You can return reactive types (Reactor, RxJava, [or other](#webflux-reactive-libraries)) directly from Spring MVC controller methods. The greater the latency per call or the interdependency among calls, the more dramatic the benefits. Spring MVC controllers can call other reactive components too.

  - If you have a large team, keep in mind the steep learning curve in the shift to non-blocking, functional, and declarative programming. A practical way to start without a full switch is to use the reactive `WebClient`. Beyond that, start small and measure the benefits. We expect that, for a wide range of applications, the shift is unnecessary. If you are unsure what benefits to look for, start by learning about how non-blocking I/O works (for example, concurrency on single-threaded Node.js) and its effects.

## Servers

Spring WebFlux is supported on Tomcat, Jetty, Servlet 3.1+ containers, as well as on non-Servlet runtimes such as Netty and Undertow. All servers are adapted to a low-level, [common API](#webflux-httphandler) so that higher-level [programming models](#webflux-programming-models) can be supported across servers.

Spring WebFlux does not have built-in support to start or stop a server. However, it is easy to [assemble](#webflux-web-handler-api) an application from Spring configuration and [WebFlux infrastructure](#webflux-config) and [run it](#webflux-httphandler) with a few lines of code.

Spring Boot has a WebFlux starter that automates these steps. By default, the starter uses Netty, but it is easy to switch to Tomcat, Jetty, or Undertow by changing your Maven or Gradle dependencies. Spring Boot defaults to Netty, because it is more widely used in the asynchronous, non-blocking space and lets a client and a server share resources.

Tomcat and Jetty can be used with both Spring MVC and WebFlux. Keep in mind, however, that the way they are used is very different. Spring MVC relies on Servlet blocking I/O and lets applications use the Servlet API directly if they need to. Spring WebFlux relies on Servlet 3.1 non-blocking I/O and uses the Servlet API behind a low-level adapter. It is not exposed for direct use.

For Undertow, Spring WebFlux uses Undertow APIs directly without the Servlet API.

## Performance

Performance has many characteristics and meanings. Reactive and non-blocking generally do not make applications run faster. They can, in some cases, (for example, if using the `WebClient` to run remote calls in parallel). On the whole, it requires more work to do things the non-blocking way and that can slightly increase the required processing time.

The key expected benefit of reactive and non-blocking is the ability to scale with a small, fixed number of threads and less memory. That makes applications more resilient under load, because they scale in a more predictable way. In order to observe those benefits, however, you need to have some latency (including a mix of slow and unpredictable network I/O). That is where the reactive stack begins to show its strengths, and the differences can be dramatic.

## Concurrency Model

Both Spring MVC and Spring WebFlux support annotated controllers, but there is a key difference in the concurrency model and the default assumptions for blocking and threads.

In Spring MVC (and servlet applications in general), it is assumed that applications can block the current thread, (for example, for remote calls). For this reason, servlet containers use a large thread pool to absorb potential blocking during request handling.

In Spring WebFlux (and non-blocking servers in general), it is assumed that applications do not block. Therefore, non-blocking servers use a small, fixed-size thread pool (event loop workers) to handle requests.

> **Tip**
> 
> “To scale” and “small number of threads” may sound contradictory but to never block the current thread (and rely on callbacks instead) means that you do not need extra threads, as there are no blocking calls to absorb.

**Invoking a Blocking API.**

What if you do need to use a blocking library? Both Reactor and RxJava provide the `publishOn` operator to continue processing on a different thread. That means there is an easy escape hatch. Keep in mind, however, that blocking APIs are not a good fit for this concurrency model.

**Mutable State.**

In Reactor and RxJava, you declare logic through operators. At runtime, a reactive pipeline is formed where data is processed sequentially, in distinct stages. A key benefit of this is that it frees applications from having to protect mutable state because application code within that pipeline is never invoked concurrently.

**Threading Model.**

What threads should you expect to see on a server running with Spring WebFlux?

  - On a “vanilla” Spring WebFlux server (for example, no data access nor other optional dependencies), you can expect one thread for the server and several others for request processing (typically as many as the number of CPU cores). Servlet containers, however, may start with more threads (for example, 10 on Tomcat), in support of both servlet (blocking) I/O and servlet 3.1 (non-blocking) I/O usage.

  - The reactive `WebClient` operates in event loop style. So you can see a small, fixed number of processing threads related to that (for example, `reactor-http-nio-` with the Reactor Netty connector). However, if Reactor Netty is used for both client and server, the two share event loop resources by default.

  - Reactor and RxJava provide thread pool abstractions, called schedulers, to use with the `publishOn` operator that is used to switch processing to a different thread pool. The schedulers have names that suggest a specific concurrency strategy — for example, “parallel” (for CPU-bound work with a limited number of threads) or “elastic” (for I/O-bound work with a large number of threads). If you see such threads, it means some code is using a specific thread pool `Scheduler` strategy.

  - Data access libraries and other third party dependencies can also create and use threads of their own.

**Configuring.**

The Spring Framework does not provide support for starting and stopping [servers](#webflux-server-choice). To configure the threading model for a server, you need to use server-specific configuration APIs, or, if you use Spring Boot, check the Spring Boot configuration options for each server. You can [configure](web-reactive.xml#webflux-client-builder) the `WebClient` directly. For all other libraries, see their respective documentation.

# Reactive Core

The `spring-web` module contains the following foundational support for reactive web applications:

  - For server request processing there are two levels of support.
    
      - [HttpHandler](#webflux-httphandler): Basic contract for HTTP request handling with non-blocking I/O and Reactive Streams back pressure, along with adapters for Reactor Netty, Undertow, Tomcat, Jetty, and any Servlet 3.1+ container.
    
      - [section\_title](#webflux-web-handler-api): Slightly higher level, general-purpose web API for request handling, on top of which concrete programming models such as annotated controllers and functional endpoints are built.

  - For the client side, there is a basic `ClientHttpConnector` contract to perform HTTP requests with non-blocking I/O and Reactive Streams back pressure, along with adapters for [Reactor Netty](https://github.com/reactor/reactor-netty), reactive [Jetty HttpClient](https://github.com/jetty-project/jetty-reactive-httpclient) and [Apache HttpComponents](https://hc.apache.org/). The higher level [WebClient](web-reactive.xml#webflux-client) used in applications builds on this basic contract.

  - For client and server, [codecs](#webflux-codecs) for serialization and deserialization of HTTP request and response content.

## `HttpHandler`

{api-spring-framework}/http/server/reactive/HttpHandler.html\[HttpHandler\] is a simple contract with a single method to handle a request and a response. It is intentionally minimal, and its main and only purpose is to be a minimal abstraction over different HTTP server APIs.

The following table describes the supported server APIs:

| Server name           | Server API used                                                                    | Reactive Streams support                                            |
| --------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Netty                 | Netty API                                                                          | [Reactor Netty](https://github.com/reactor/reactor-netty)           |
| Undertow              | Undertow API                                                                       | spring-web: Undertow to Reactive Streams bridge                     |
| Tomcat                | Servlet 3.1 non-blocking I/O; Tomcat API to read and write ByteBuffers vs byte\[\] | spring-web: Servlet 3.1 non-blocking I/O to Reactive Streams bridge |
| Jetty                 | Servlet 3.1 non-blocking I/O; Jetty API to write ByteBuffers vs byte\[\]           | spring-web: Servlet 3.1 non-blocking I/O to Reactive Streams bridge |
| Servlet 3.1 container | Servlet 3.1 non-blocking I/O                                                       | spring-web: Servlet 3.1 non-blocking I/O to Reactive Streams bridge |

The following table describes server dependencies (also see [supported versions](https://github.com/spring-projects/spring-framework/wiki/What%27s-New-in-the-Spring-Framework)):

| Server name   | Group id                | Artifact name               |
| ------------- | ----------------------- | --------------------------- |
| Reactor Netty | io.projectreactor.netty | reactor-netty               |
| Undertow      | io.undertow             | undertow-core               |
| Tomcat        | org.apache.tomcat.embed | tomcat-embed-core           |
| Jetty         | org.eclipse.jetty       | jetty-server, jetty-servlet |

The code snippets below show using the `HttpHandler` adapters with each server API:

**Reactor Netty**

**Java.**

``` java
HttpHandler handler = ...
ReactorHttpHandlerAdapter adapter = new ReactorHttpHandlerAdapter(handler);
HttpServer.create().host(host).port(port).handle(adapter).bind().block();
```

**Kotlin.**

``` kotlin
val handler: HttpHandler = ...
val adapter = ReactorHttpHandlerAdapter(handler)
HttpServer.create().host(host).port(port).handle(adapter).bind().block()
```

**Undertow**

**Java.**

``` java
HttpHandler handler = ...
UndertowHttpHandlerAdapter adapter = new UndertowHttpHandlerAdapter(handler);
Undertow server = Undertow.builder().addHttpListener(port, host).setHandler(adapter).build();
server.start();
```

**Kotlin.**

``` kotlin
val handler: HttpHandler = ...
val adapter = UndertowHttpHandlerAdapter(handler)
val server = Undertow.builder().addHttpListener(port, host).setHandler(adapter).build()
server.start()
```

**Tomcat**

**Java.**

``` java
HttpHandler handler = ...
Servlet servlet = new TomcatHttpHandlerAdapter(handler);

Tomcat server = new Tomcat();
File base = new File(System.getProperty("java.io.tmpdir"));
Context rootContext = server.addContext("", base.getAbsolutePath());
Tomcat.addServlet(rootContext, "main", servlet);
rootContext.addServletMappingDecoded("/", "main");
server.setHost(host);
server.setPort(port);
server.start();
```

**Kotlin.**

``` kotlin
val handler: HttpHandler = ...
val servlet = TomcatHttpHandlerAdapter(handler)

val server = Tomcat()
val base = File(System.getProperty("java.io.tmpdir"))
val rootContext = server.addContext("", base.absolutePath)
Tomcat.addServlet(rootContext, "main", servlet)
rootContext.addServletMappingDecoded("/", "main")
server.host = host
server.setPort(port)
server.start()
```

**Jetty**

**Java.**

``` java
HttpHandler handler = ...
Servlet servlet = new JettyHttpHandlerAdapter(handler);

Server server = new Server();
ServletContextHandler contextHandler = new ServletContextHandler(server, "");
contextHandler.addServlet(new ServletHolder(servlet), "/");
contextHandler.start();

ServerConnector connector = new ServerConnector(server);
connector.setHost(host);
connector.setPort(port);
server.addConnector(connector);
server.start();
```

**Kotlin.**

``` kotlin
val handler: HttpHandler = ...
val servlet = JettyHttpHandlerAdapter(handler)

val server = Server()
val contextHandler = ServletContextHandler(server, "")
contextHandler.addServlet(ServletHolder(servlet), "/")
contextHandler.start();

val connector = ServerConnector(server)
connector.host = host
connector.port = port
server.addConnector(connector)
server.start()
```

**Servlet 3.1+ Container**

To deploy as a WAR to any Servlet 3.1+ container, you can extend and include {api-spring-framework}/web/server/adapter/AbstractReactiveWebInitializer.html\[`AbstractReactiveWebInitializer`\] in the WAR. That class wraps an `HttpHandler` with `ServletHttpHandlerAdapter` and registers that as a `Servlet`.

## `WebHandler` API

The `org.springframework.web.server` package builds on the [section\_title](#webflux-httphandler) contract to provide a general-purpose web API for processing requests through a chain of multiple {api-spring-framework}/web/server/WebExceptionHandler.html\[`WebExceptionHandler`\], multiple {api-spring-framework}/web/server/WebFilter.html\[`WebFilter`\], and a single {api-spring-framework}/web/server/WebHandler.html\[`WebHandler`\] component. The chain can be put together with `WebHttpHandlerBuilder` by simply pointing to a Spring `ApplicationContext` where components are [auto-detected](#webflux-web-handler-api-special-beans), and/or by registering components with the builder.

While `HttpHandler` has a simple goal to abstract the use of different HTTP servers, the `WebHandler` API aims to provide a broader set of features commonly used in web applications such as:

  - User session with attributes.

  - Request attributes.

  - Resolved `Locale` or `Principal` for the request.

  - Access to parsed and cached form data.

  - Abstractions for multipart data.

  - and more..

### Special bean types

The table below lists the components that `WebHttpHandlerBuilder` can auto-detect in a Spring ApplicationContext, or that can be registered directly with it:

| Bean name                    | Bean type                    | Count | Description                                                                                                                                                                                    |
| ---------------------------- | ---------------------------- | ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| \<any\>                      | `WebExceptionHandler`        | 0..N  | Provide handling for exceptions from the chain of `WebFilter` instances and the target `WebHandler`. For more details, see [section\_title](#webflux-exception-handler).                       |
| \<any\>                      | `WebFilter`                  | 0..N  | Apply interception style logic to before and after the rest of the filter chain and the target `WebHandler`. For more details, see [section\_title](#webflux-filters).                         |
| `webHandler`                 | `WebHandler`                 | 1     | The handler for the request.                                                                                                                                                                   |
| `webSessionManager`          | `WebSessionManager`          | 0..1  | The manager for `WebSession` instances exposed through a method on `ServerWebExchange`. `DefaultWebSessionManager` by default.                                                                 |
| `serverCodecConfigurer`      | `ServerCodecConfigurer`      | 0..1  | For access to `HttpMessageReader` instances for parsing form data and multipart data that is then exposed through methods on `ServerWebExchange`. `ServerCodecConfigurer.create()` by default. |
| `localeContextResolver`      | `LocaleContextResolver`      | 0..1  | The resolver for `LocaleContext` exposed through a method on `ServerWebExchange`. `AcceptHeaderLocaleContextResolver` by default.                                                              |
| `forwardedHeaderTransformer` | `ForwardedHeaderTransformer` | 0..1  | For processing forwarded type headers, either by extracting and removing them or by removing them only. Not used by default.                                                                   |

### Form Data

`ServerWebExchange` exposes the following method for accessing form data:

**Java.**

``` java
Mono<MultiValueMap<String, String>> getFormData();
```

**Kotlin.**

``` Kotlin
suspend fun getFormData(): MultiValueMap<String, String>
```

The `DefaultServerWebExchange` uses the configured `HttpMessageReader` to parse form data (`application/x-www-form-urlencoded`) into a `MultiValueMap`. By default, `FormHttpMessageReader` is configured for use by the `ServerCodecConfigurer` bean (see the [Web Handler API](#webflux-web-handler-api)).

### Multipart Data

[Web MVC](web.xml#mvc-multipart)

`ServerWebExchange` exposes the following method for accessing multipart data:

**Java.**

``` java
Mono<MultiValueMap<String, Part>> getMultipartData();
```

**Kotlin.**

``` Kotlin
suspend fun getMultipartData(): MultiValueMap<String, Part>
```

The `DefaultServerWebExchange` uses the configured `HttpMessageReader<MultiValueMap<String, Part>>` to parse `multipart/form-data` content into a `MultiValueMap`. By default, this is the `DefaultPartHttpMessageReader`, which does not have any third-party dependencies. Alternatively, the `SynchronossPartHttpMessageReader` can be used, which is based on the [Synchronoss NIO Multipart](https://github.com/synchronoss/nio-multipart) library. Both are configured through the `ServerCodecConfigurer` bean (see the [Web Handler API](#webflux-web-handler-api)).

To parse multipart data in streaming fashion, you can use the `Flux<Part>` returned from an `HttpMessageReader<Part>` instead. For example, in an annotated controller, use of `@RequestPart` implies `Map`-like access to individual parts by name and, hence, requires parsing multipart data in full. By contrast, you can use `@RequestBody` to decode the content to `Flux<Part>` without collecting to a `MultiValueMap`.

### Forwarded Headers

[Web MVC](web.xml#filters-forwarded-headers)

As a request goes through proxies (such as load balancers), the host, port, and scheme may change. That makes it a challenge, from a client perspective, to create links that point to the correct host, port, and scheme.

[RFC 7239](https://tools.ietf.org/html/rfc7239) defines the `Forwarded` HTTP header that proxies can use to provide information about the original request. There are other non-standard headers, too, including `X-Forwarded-Host`, `X-Forwarded-Port`, `X-Forwarded-Proto`, `X-Forwarded-Ssl`, and `X-Forwarded-Prefix`.

`ForwardedHeaderTransformer` is a component that modifies the host, port, and scheme of the request, based on forwarded headers, and then removes those headers. If you declare it as a bean with the name `forwardedHeaderTransformer`, it will be [detected](#webflux-web-handler-api-special-beans) and used.

There are security considerations for forwarded headers, since an application cannot know if the headers were added by a proxy, as intended, or by a malicious client. This is why a proxy at the boundary of trust should be configured to remove untrusted forwarded traffic coming from the outside. You can also configure the `ForwardedHeaderTransformer` with `removeOnly=true`, in which case it removes but does not use the headers.

> **Note**
> 
> In 5.1 `ForwardedHeaderFilter` was deprecated and superceded by `ForwardedHeaderTransformer` so forwarded headers can be processed earlier, before the exchange is created. If the filter is configured anyway, it is taken out of the list of filters, and `ForwardedHeaderTransformer` is used instead.

## Filters

[Web MVC](web.xml#filters)

In the [section\_title](#webflux-web-handler-api), you can use a `WebFilter` to apply interception-style logic before and after the rest of the processing chain of filters and the target `WebHandler`. When using the [section\_title](#webflux-config), registering a `WebFilter` is as simple as declaring it as a Spring bean and (optionally) expressing precedence by using `@Order` on the bean declaration or by implementing `Ordered`.

### CORS

[Web MVC](web.xml#filters-cors)

Spring WebFlux provides fine-grained support for CORS configuration through annotations on controllers. However, when you use it with Spring Security, we advise relying on the built-in `CorsFilter`, which must be ordered ahead of Spring Security’s chain of filters.

See the section on [section\_title](#webflux-cors) and the [section\_title](#webflux-cors-webfilter) for more details.

## Exceptions

[Web MVC](web.xml#mvc-ann-customer-servlet-container-error-page)

In the [section\_title](#webflux-web-handler-api), you can use a `WebExceptionHandler` to handle exceptions from the chain of `WebFilter` instances and the target `WebHandler`. When using the [section\_title](#webflux-config), registering a `WebExceptionHandler` is as simple as declaring it as a Spring bean and (optionally) expressing precedence by using `@Order` on the bean declaration or by implementing `Ordered`.

The following table describes the available `WebExceptionHandler` implementations:

<table>
<colgroup>
<col style="width: 33%" />
<col style="width: 66%" />
</colgroup>
<thead>
<tr class="header">
<th>Exception Handler</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>ResponseStatusExceptionHandler</code></p></td>
<td><p>Provides handling for exceptions of type {api-spring-framework}/web/server/ResponseStatusException.html[<code>ResponseStatusException</code>] by setting the response to the HTTP status code of the exception.</p></td>
</tr>
<tr class="even">
<td><p><code>WebFluxResponseStatusExceptionHandler</code></p></td>
<td><p>Extension of <code>ResponseStatusExceptionHandler</code> that can also determine the HTTP status code of a <code>@ResponseStatus</code> annotation on any exception.</p>
<p>This handler is declared in the <a href="#webflux-config">section_title</a>.</p></td>
</tr>
</tbody>
</table>

## Codecs

[Web MVC](integration.xml#rest-message-conversion)

The `spring-web` and `spring-core` modules provide support for serializing and deserializing byte content to and from higher level objects through non-blocking I/O with Reactive Streams back pressure. The following describes this support:

  - {api-spring-framework}/core/codec/Encoder.html\[`Encoder`\] and {api-spring-framework}/core/codec/Decoder.html\[`Decoder`\] are low level contracts to encode and decode content independent of HTTP.

  - {api-spring-framework}/http/codec/HttpMessageReader.html\[`HttpMessageReader`\] and {api-spring-framework}/http/codec/HttpMessageWriter.html\[`HttpMessageWriter`\] are contracts to encode and decode HTTP message content.

  - An `Encoder` can be wrapped with `EncoderHttpMessageWriter` to adapt it for use in a web application, while a `Decoder` can be wrapped with `DecoderHttpMessageReader`.

  - {api-spring-framework}/core/io/buffer/DataBuffer.html\[`DataBuffer`\] abstracts different byte buffer representations (e.g. Netty `ByteBuf`, `java.nio.ByteBuffer`, etc.) and is what all codecs work on. See [Data Buffers and Codecs](core.xml#databuffers) in the "Spring Core" section for more on this topic.

The `spring-core` module provides `byte[]`, `ByteBuffer`, `DataBuffer`, `Resource`, and `String` encoder and decoder implementations. The `spring-web` module provides Jackson JSON, Jackson Smile, JAXB2, Protocol Buffers and other encoders and decoders along with web-only HTTP message reader and writer implementations for form data, multipart content, server-sent events, and others.

`ClientCodecConfigurer` and `ServerCodecConfigurer` are typically used to configure and customize the codecs to use in an application. See the section on configuring [section\_title](#webflux-config-message-codecs).

### Jackson JSON

JSON and binary JSON ([Smile](https://github.com/FasterXML/smile-format-specification)) are both supported when the Jackson library is present.

The `Jackson2Decoder` works as follows:

  - Jackson’s asynchronous, non-blocking parser is used to aggregate a stream of byte chunks into `TokenBuffer`'s each representing a JSON object.

  - Each `TokenBuffer` is passed to Jackson’s `ObjectMapper` to create a higher level object.

  - When decoding to a single-value publisher (e.g. `Mono`), there is one `TokenBuffer`.

  - When decoding to a multi-value publisher (e.g. `Flux`), each `TokenBuffer` is passed to the `ObjectMapper` as soon as enough bytes are received for a fully formed object. The input content can be a JSON array, or any [line-delimited JSON](https://en.wikipedia.org/wiki/JSON_streaming) format such as NDJSON, JSON Lines, or JSON Text Sequences.

The `Jackson2Encoder` works as follows:

  - For a single value publisher (e.g. `Mono`), simply serialize it through the `ObjectMapper`.

  - For a multi-value publisher with `application/json`, by default collect the values with `Flux#collectToList()` and then serialize the resulting collection.

  - For a multi-value publisher with a streaming media type such as `application/x-ndjson` or `application/stream+x-jackson-smile`, encode, write, and flush each value individually using a [line-delimited JSON](https://en.wikipedia.org/wiki/JSON_streaming) format. Other streaming media types may be registered with the encoder.

  - For SSE the `Jackson2Encoder` is invoked per event and the output is flushed to ensure delivery without delay.

> **Note**
> 
> By default both `Jackson2Encoder` and `Jackson2Decoder` do not support elements of type `String`. Instead the default assumption is that a string or a sequence of strings represent serialized JSON content, to be rendered by the `CharSequenceEncoder`. If what you need is to render a JSON array from `Flux<String>`, use `Flux#collectToList()` and encode a `Mono<List<String>>`.

### Form Data

`FormHttpMessageReader` and `FormHttpMessageWriter` support decoding and encoding `application/x-www-form-urlencoded` content.

On the server side where form content often needs to be accessed from multiple places, `ServerWebExchange` provides a dedicated `getFormData()` method that parses the content through `FormHttpMessageReader` and then caches the result for repeated access. See [section\_title](#webflux-form-data) in the [section\_title](#webflux-web-handler-api) section.

Once `getFormData()` is used, the original raw content can no longer be read from the request body. For this reason, applications are expected to go through `ServerWebExchange` consistently for access to the cached form data versus reading from the raw request body.

### Multipart

`MultipartHttpMessageReader` and `MultipartHttpMessageWriter` support decoding and encoding "multipart/form-data" content. In turn `MultipartHttpMessageReader` delegates to another `HttpMessageReader` for the actual parsing to a `Flux<Part>` and then simply collects the parts into a `MultiValueMap`. By default, the `DefaultPartHttpMessageReader` is used, but this can be changed through the `ServerCodecConfigurer`. For more information about the `DefaultPartHttpMessageReader`, refer to to the {api-spring-framework}/http/codec/multipart/DefaultPartHttpMessageReader.html\[javadoc of `DefaultPartHttpMessageReader`\].

On the server side where multipart form content may need to be accessed from multiple places, `ServerWebExchange` provides a dedicated `getMultipartData()` method that parses the content through `MultipartHttpMessageReader` and then caches the result for repeated access. See [section\_title](#webflux-multipart) in the [section\_title](#webflux-web-handler-api) section.

Once `getMultipartData()` is used, the original raw content can no longer be read from the request body. For this reason applications have to consistently use `getMultipartData()` for repeated, map-like access to parts, or otherwise rely on the `SynchronossPartHttpMessageReader` for a one-time access to `Flux<Part>`.

### Limits

`Decoder` and `HttpMessageReader` implementations that buffer some or all of the input stream can be configured with a limit on the maximum number of bytes to buffer in memory. In some cases buffering occurs because input is aggregated and represented as a single object — for example, a controller method with `@RequestBody byte[]`, `x-www-form-urlencoded` data, and so on. Buffering can also occur with streaming, when splitting the input stream — for example, delimited text, a stream of JSON objects, and so on. For those streaming cases, the limit applies to the number of bytes associated with one object in the stream.

To configure buffer sizes, you can check if a given `Decoder` or `HttpMessageReader` exposes a `maxInMemorySize` property and if so the Javadoc will have details about default values. On the server side, `ServerCodecConfigurer` provides a single place from where to set all codecs, see [section\_title](#webflux-config-message-codecs). On the client side, the limit for all codecs can be changed in [WebClient.Builder](web-reactive.xml#webflux-client-builder-maxinmemorysize).

For [Multipart parsing](#webflux-codecs-multipart) the `maxInMemorySize` property limits the size of non-file parts. For file parts, it determines the threshold at which the part is written to disk. For file parts written to disk, there is an additional `maxDiskUsagePerPart` property to limit the amount of disk space per part. There is also a `maxParts` property to limit the overall number of parts in a multipart request. To configure all three in WebFlux, you’ll need to supply a pre-configured instance of `MultipartHttpMessageReader` to `ServerCodecConfigurer`.

### Streaming

[Web MVC](web.xml#mvc-ann-async-http-streaming)

When streaming to the HTTP response (for example, `text/event-stream`, `application/x-ndjson`), it is important to send data periodically, in order to reliably detect a disconnected client sooner rather than later. Such a send could be a comment-only, empty SSE event or any other "no-op" data that would effectively serve as a heartbeat.

### `DataBuffer`

`DataBuffer` is the representation for a byte buffer in WebFlux. The Spring Core part of this reference has more on that in the section on [Data Buffers and Codecs](core.xml#databuffers). The key point to understand is that on some servers like Netty, byte buffers are pooled and reference counted, and must be released when consumed to avoid memory leaks.

WebFlux applications generally do not need to be concerned with such issues, unless they consume or produce data buffers directly, as opposed to relying on codecs to convert to and from higher level objects, or unless they choose to create custom codecs. For such cases please review the information in [Data Buffers and Codecs](core.xml#databuffers), especially the section on [Using DataBuffer](core.xml#databuffers-using).

## Logging

[Web MVC](web.xml#mvc-logging)

`DEBUG` level logging in Spring WebFlux is designed to be compact, minimal, and human-friendly. It focuses on high value bits of information that are useful over and over again vs others that are useful only when debugging a specific issue.

`TRACE` level logging generally follows the same principles as `DEBUG` (and for example also should not be a firehose) but can be used for debugging any issue. In addition, some log messages may show a different level of detail at `TRACE` vs `DEBUG`.

Good logging comes from the experience of using the logs. If you spot anything that does not meet the stated goals, please let us know.

### Log Id

In WebFlux, a single request can be run over multiple threads and the thread ID is not useful for correlating log messages that belong to a specific request. This is why WebFlux log messages are prefixed with a request-specific ID by default.

On the server side, the log ID is stored in the `ServerWebExchange` attribute ({api-spring-framework}/web/server/ServerWebExchange.html\#LOG\_ID\_ATTRIBUTE\[`LOG_ID_ATTRIBUTE`\]), while a fully formatted prefix based on that ID is available from `ServerWebExchange#getLogPrefix()`. On the `WebClient` side, the log ID is stored in the `ClientRequest` attribute ({api-spring-framework}/web/reactive/function/client/ClientRequest.html\#LOG\_ID\_ATTRIBUTE\[`LOG_ID_ATTRIBUTE`\]) ,while a fully formatted prefix is available from `ClientRequest#logPrefix()`.

### Sensitive Data

[Web MVC](web.xml#mvc-logging-sensitive-data)

`DEBUG` and `TRACE` logging can log sensitive information. This is why form parameters and headers are masked by default and you must explicitly enable their logging in full.

The following example shows how to do so for server-side requests:

**Java.**

``` java
@Configuration
@EnableWebFlux
class MyConfig implements WebFluxConfigurer {

    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        configurer.defaultCodecs().enableLoggingRequestDetails(true);
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class MyConfig : WebFluxConfigurer {

    override fun configureHttpMessageCodecs(configurer: ServerCodecConfigurer) {
        configurer.defaultCodecs().enableLoggingRequestDetails(true)
    }
}
```

The following example shows how to do so for client-side requests:

**Java.**

``` java
Consumer<ClientCodecConfigurer> consumer = configurer ->
        configurer.defaultCodecs().enableLoggingRequestDetails(true);

WebClient webClient = WebClient.builder()
        .exchangeStrategies(strategies -> strategies.codecs(consumer))
        .build();
```

**Kotlin.**

``` kotlin
val consumer: (ClientCodecConfigurer) -> Unit  = { configurer -> configurer.defaultCodecs().enableLoggingRequestDetails(true) }

val webClient = WebClient.builder()
        .exchangeStrategies({ strategies -> strategies.codecs(consumer) })
        .build()
```

### Appenders

Logging libraries such as SLF4J and Log4J 2 provide asynchronous loggers that avoid blocking. While those have their own drawbacks such as potentially dropping messages that could not be queued for logging, they are the best available options currently for use in a reactive, non-blocking application.

### Custom codecs

Applications can register custom codecs for supporting additional media types, or specific behaviors that are not supported by the default codecs.

Some configuration options expressed by developers are enforced on default codecs. Custom codecs might want to get a chance to align with those preferences, like [enforcing buffering limits](#webflux-codecs-limits) or [logging sensitive data](#webflux-logging-sensitive-data).

The following example shows how to do so for client-side requests:

**Java.**

``` java
WebClient webClient = WebClient.builder()
        .codecs(configurer -> {
                CustomDecoder decoder = new CustomDecoder();
                   configurer.customCodecs().registerWithDefaultConfig(decoder);
        })
        .build();
```

**Kotlin.**

``` kotlin
val webClient = WebClient.builder()
        .codecs({ configurer ->
                val decoder = CustomDecoder()
                configurer.customCodecs().registerWithDefaultConfig(decoder)
         })
        .build()
```

# `DispatcherHandler`

[Web MVC](web.xml#mvc-servlet)

Spring WebFlux, similarly to Spring MVC, is designed around the front controller pattern, where a central `WebHandler`, the `DispatcherHandler`, provides a shared algorithm for request processing, while actual work is performed by configurable, delegate components. This model is flexible and supports diverse workflows.

`DispatcherHandler` discovers the delegate components it needs from Spring configuration. It is also designed to be a Spring bean itself and implements `ApplicationContextAware` for access to the context in which it runs. If `DispatcherHandler` is declared with a bean name of `webHandler`, it is, in turn, discovered by {api-spring-framework}/web/server/adapter/WebHttpHandlerBuilder.html\[`WebHttpHandlerBuilder`\], which puts together a request-processing chain, as described in [section\_title](#webflux-web-handler-api).

Spring configuration in a WebFlux application typically contains:

  - `DispatcherHandler` with the bean name `webHandler`

  - `WebFilter` and `WebExceptionHandler` beans

  - [`DispatcherHandler` special beans](#webflux-special-bean-types)

  - Others

The configuration is given to `WebHttpHandlerBuilder` to build the processing chain, as the following example shows:

**Java.**

``` java
ApplicationContext context = ...
HttpHandler handler = WebHttpHandlerBuilder.applicationContext(context).build();
```

**Kotlin.**

``` kotlin
val context: ApplicationContext = ...
val handler = WebHttpHandlerBuilder.applicationContext(context).build()
```

The resulting `HttpHandler` is ready for use with a [server adapter](#webflux-httphandler).

## Special Bean Types

[Web MVC](web.xml#mvc-servlet-special-bean-types)

The `DispatcherHandler` delegates to special beans to process requests and render the appropriate responses. By “special beans,” we mean Spring-managed `Object` instances that implement WebFlux framework contracts. Those usually come with built-in contracts, but you can customize their properties, extend them, or replace them.

The following table lists the special beans detected by the `DispatcherHandler`. Note that there are also some other beans detected at a lower level (see [section\_title](#webflux-web-handler-api-special-beans) in the Web Handler API).

<table>
<colgroup>
<col style="width: 33%" />
<col style="width: 66%" />
</colgroup>
<thead>
<tr class="header">
<th>Bean type</th>
<th>Explanation</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>HandlerMapping</code></p></td>
<td><p>Map a request to a handler. The mapping is based on some criteria, the details of which vary by <code>HandlerMapping</code> implementation — annotated controllers, simple URL pattern mappings, and others.</p>
<p>The main <code>HandlerMapping</code> implementations are <code>RequestMappingHandlerMapping</code> for <code>@RequestMapping</code> annotated methods, <code>RouterFunctionMapping</code> for functional endpoint routes, and <code>SimpleUrlHandlerMapping</code> for explicit registrations of URI path patterns and <code>WebHandler</code> instances.</p></td>
</tr>
<tr class="even">
<td><p><code>HandlerAdapter</code></p></td>
<td><p>Help the <code>DispatcherHandler</code> to invoke a handler mapped to a request regardless of how the handler is actually invoked. For example, invoking an annotated controller requires resolving annotations. The main purpose of a <code>HandlerAdapter</code> is to shield the <code>DispatcherHandler</code> from such details.</p></td>
</tr>
<tr class="odd">
<td><p><code>HandlerResultHandler</code></p></td>
<td><p>Process the result from the handler invocation and finalize the response. See <a href="#webflux-resulthandling">section_title</a>.</p></td>
</tr>
</tbody>
</table>

## WebFlux Config

[Web MVC](web.xml#mvc-servlet-config)

Applications can declare the infrastructure beans (listed under [Web Handler API](#webflux-web-handler-api-special-beans) and [`DispatcherHandler`](#webflux-special-bean-types)) that are required to process requests. However, in most cases, the [section\_title](#webflux-config) is the best starting point. It declares the required beans and provides a higher-level configuration callback API to customize it.

> **Note**
> 
> Spring Boot relies on the WebFlux config to configure Spring WebFlux and also provides many extra convenient options.

## Processing

[Web MVC](web.xml#mvc-servlet-sequence)

`DispatcherHandler` processes requests as follows:

  - Each `HandlerMapping` is asked to find a matching handler, and the first match is used.

  - If a handler is found, it is run through an appropriate `HandlerAdapter`, which exposes the return value from the execution as `HandlerResult`.

  - The `HandlerResult` is given to an appropriate `HandlerResultHandler` to complete processing by writing to the response directly or by using a view to render.

## Result Handling

The return value from the invocation of a handler, through a `HandlerAdapter`, is wrapped as a `HandlerResult`, along with some additional context, and passed to the first `HandlerResultHandler` that claims support for it. The following table shows the available `HandlerResultHandler` implementations, all of which are declared in the [section\_title](#webflux-config):

<table>
<colgroup>
<col style="width: 25%" />
<col style="width: 50%" />
<col style="width: 25%" />
</colgroup>
<thead>
<tr class="header">
<th>Result Handler Type</th>
<th>Return Values</th>
<th>Default Order</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>ResponseEntityResultHandler</code></p></td>
<td><p><code>ResponseEntity</code>, typically from <code>@Controller</code> instances.</p></td>
<td><p>0</p></td>
</tr>
<tr class="even">
<td><p><code>ServerResponseResultHandler</code></p></td>
<td><p><code>ServerResponse</code>, typically from functional endpoints.</p></td>
<td><p>0</p></td>
</tr>
<tr class="odd">
<td><p><code>ResponseBodyResultHandler</code></p></td>
<td><p>Handle return values from <code>@ResponseBody</code> methods or <code>@RestController</code> classes.</p></td>
<td><p>100</p></td>
</tr>
<tr class="even">
<td><p><code>ViewResolutionResultHandler</code></p></td>
<td><p><code>CharSequence</code>, {api-spring-framework}/web/reactive/result/view/View.html[<code>View</code>], {api-spring-framework}/ui/Model.html[Model], <code>Map</code>, {api-spring-framework}/web/reactive/result/view/Rendering.html[Rendering], or any other <code>Object</code> is treated as a model attribute.</p>
<p>See also <a href="#webflux-viewresolution">section_title</a>.</p></td>
<td><p><code>Integer.MAX_VALUE</code></p></td>
</tr>
</tbody>
</table>

## Exceptions

[Web MVC](web.xml#mvc-exceptionhandlers)

The `HandlerResult` returned from a `HandlerAdapter` can expose a function for error handling based on some handler-specific mechanism. This error function is called if:

  - The handler (for example, `@Controller`) invocation fails.

  - The handling of the handler return value through a `HandlerResultHandler` fails.

The error function can change the response (for example, to an error status), as long as an error signal occurs before the reactive type returned from the handler produces any data items.

This is how `@ExceptionHandler` methods in `@Controller` classes are supported. By contrast, support for the same in Spring MVC is built on a `HandlerExceptionResolver`. This generally should not matter. However, keep in mind that, in WebFlux, you cannot use a `@ControllerAdvice` to handle exceptions that occur before a handler is chosen.

See also [section\_title](#webflux-ann-controller-exceptions) in the “Annotated Controller” section or [section\_title](#webflux-exception-handler) in the WebHandler API section.

## View Resolution

[Web MVC](web.xml#mvc-viewresolver)

View resolution enables rendering to a browser with an HTML template and a model without tying you to a specific view technology. In Spring WebFlux, view resolution is supported through a dedicated [HandlerResultHandler](#webflux-resulthandling) that uses `ViewResolver` instances to map a String (representing a logical view name) to a `View` instance. The `View` is then used to render the response.

### Handling

[Web MVC](web.xml#mvc-handling)

The `HandlerResult` passed into `ViewResolutionResultHandler` contains the return value from the handler and the model that contains attributes added during request handling. The return value is processed as one of the following:

  - `String`, `CharSequence`: A logical view name to be resolved to a `View` through the list of configured `ViewResolver` implementations.

  - `void`: Select a default view name based on the request path, minus the leading and trailing slash, and resolve it to a `View`. The same also happens when a view name was not provided (for example, model attribute was returned) or an async return value (for example, `Mono` completed empty).

  - {api-spring-framework}/web/reactive/result/view/Rendering.html\[Rendering\]: API for view resolution scenarios. Explore the options in your IDE with code completion.

  - `Model`, `Map`: Extra model attributes to be added to the model for the request.

  - Any other: Any other return value (except for simple types, as determined by {api-spring-framework}/beans/BeanUtils.html\#isSimpleProperty-java.lang.Class-\[BeanUtils\#isSimpleProperty\]) is treated as a model attribute to be added to the model. The attribute name is derived from the class name by using {api-spring-framework}/core/Conventions.html\[conventions\], unless a handler method `@ModelAttribute` annotation is present.

The model can contain asynchronous, reactive types (for example, from Reactor or RxJava). Prior to rendering, `AbstractView` resolves such model attributes into concrete values and updates the model. Single-value reactive types are resolved to a single value or no value (if empty), while multi-value reactive types (for example, `Flux<T>`) are collected and resolved to `List<T>`.

To configure view resolution is as simple as adding a `ViewResolutionResultHandler` bean to your Spring configuration. [WebFlux Config](#webflux-config-view-resolvers) provides a dedicated configuration API for view resolution.

See [section\_title](#webflux-view) for more on the view technologies integrated with Spring WebFlux.

### Redirecting

[Web MVC](web.xml#mvc-redirecting-redirect-prefix)

The special `redirect:` prefix in a view name lets you perform a redirect. The `UrlBasedViewResolver` (and sub-classes) recognize this as an instruction that a redirect is needed. The rest of the view name is the redirect URL.

The net effect is the same as if the controller had returned a `RedirectView` or `Rendering.redirectTo("abc").build()`, but now the controller itself can operate in terms of logical view names. A view name such as `redirect:/some/resource` is relative to the current application, while a view name such as `redirect:https://example.com/arbitrary/path` redirects to an absolute URL.

### Content Negotiation

[Web MVC](web.xml#mvc-multiple-representations)

`ViewResolutionResultHandler` supports content negotiation. It compares the request media types with the media types supported by each selected `View`. The first `View` that supports the requested media type(s) is used.

In order to support media types such as JSON and XML, Spring WebFlux provides `HttpMessageWriterView`, which is a special `View` that renders through an [HttpMessageWriter](#webflux-codecs). Typically, you would configure these as default views through the [WebFlux Configuration](#webflux-config-view-resolvers). Default views are always selected and used if they match the requested media type.

# Annotated Controllers

[Web MVC](web.xml#mvc-controller)

Spring WebFlux provides an annotation-based programming model, where `@Controller` and `@RestController` components use annotations to express request mappings, request input, handle exceptions, and more. Annotated controllers have flexible method signatures and do not have to extend base classes nor implement specific interfaces.

The following listing shows a basic example:

**Java.**

``` java
@RestController
public class HelloController {

    @GetMapping("/hello")
    public String handle() {
        return "Hello WebFlux";
    }
}
```

**Kotlin.**

``` kotlin
@RestController
class HelloController {

    @GetMapping("/hello")
    fun handle() = "Hello WebFlux"
}
```

In the preceding example, the method returns a `String` to be written to the response body.

## `@Controller`

[Web MVC](web.xml#mvc-ann-controller)

You can define controller beans by using a standard Spring bean definition. The `@Controller` stereotype allows for auto-detection and is aligned with Spring general support for detecting `@Component` classes in the classpath and auto-registering bean definitions for them. It also acts as a stereotype for the annotated class, indicating its role as a web component.

To enable auto-detection of such `@Controller` beans, you can add component scanning to your Java configuration, as the following example shows:

**Java.**

``` java
@Configuration
@ComponentScan("org.example.web") 
public class WebConfig {

    // ...
}
```

  - Scan the `org.example.web` package.

**Kotlin.**

``` kotlin
@Configuration
@ComponentScan("org.example.web") 
class WebConfig {

    // ...
}
```

  - Scan the `org.example.web` package.

`@RestController` is a [composed annotation](core.xml#beans-meta-annotations) that is itself meta-annotated with `@Controller` and `@ResponseBody`, indicating a controller whose every method inherits the type-level `@ResponseBody` annotation and, therefore, writes directly to the response body versus view resolution and rendering with an HTML template.

## Request Mapping

[Web MVC](web.xml#mvc-ann-requestmapping)

The `@RequestMapping` annotation is used to map requests to controllers methods. It has various attributes to match by URL, HTTP method, request parameters, headers, and media types. You can use it at the class level to express shared mappings or at the method level to narrow down to a specific endpoint mapping.

There are also HTTP method specific shortcut variants of `@RequestMapping`:

  - `@GetMapping`

  - `@PostMapping`

  - `@PutMapping`

  - `@DeleteMapping`

  - `@PatchMapping`

The preceding annotations are [section\_title](#webflux-ann-requestmapping-composed) that are provided because, arguably, most controller methods should be mapped to a specific HTTP method versus using `@RequestMapping`, which, by default, matches to all HTTP methods. At the same time, a `@RequestMapping` is still needed at the class level to express shared mappings.

The following example uses type and method level mappings:

**Java.**

``` java
@RestController
@RequestMapping("/persons")
class PersonController {

    @GetMapping("/{id}")
    public Person getPerson(@PathVariable Long id) {
        // ...
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public void add(@RequestBody Person person) {
        // ...
    }
}
```

**Kotlin.**

``` kotlin
@RestController
@RequestMapping("/persons")
class PersonController {

    @GetMapping("/{id}")
    fun getPerson(@PathVariable id: Long): Person {
        // ...
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun add(@RequestBody person: Person) {
        // ...
    }
}
```

### URI Patterns

[Web MVC](web.xml#mvc-ann-requestmapping-uri-templates)

You can map requests by using glob patterns and wildcards:

<table>
<colgroup>
<col style="width: 20%" />
<col style="width: 30%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th>Pattern</th>
<th>Description</th>
<th>Example</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>?</code></p></td>
<td><p>Matches one character</p></td>
<td><p><code>&quot;/pages/t?st.html&quot;</code> matches <code>&quot;/pages/test.html&quot;</code> and <code>&quot;/pages/t3st.html&quot;</code></p></td>
</tr>
<tr class="even">
<td><p><code>*</code></p></td>
<td><p>Matches zero or more characters within a path segment</p></td>
<td><p><code>&quot;/resources/*.png&quot;</code> matches <code>&quot;/resources/file.png&quot;</code></p>
<p><code>&quot;/projects/*/versions&quot;</code> matches <code>&quot;/projects/spring/versions&quot;</code> but does not match <code>&quot;/projects/spring/boot/versions&quot;</code></p></td>
</tr>
<tr class="odd">
<td><p><code>**</code></p></td>
<td><p>Matches zero or more path segments until the end of the path</p></td>
<td><p><code>&quot;/resources/**&quot;</code> matches <code>&quot;/resources/file.png&quot;</code> and <code>&quot;/resources/images/file.png&quot;</code></p>
<p><code>&quot;/resources/**/file.png&quot;</code> is invalid as <code>**</code> is only allowed at the end of the path.</p></td>
</tr>
<tr class="even">
<td><p><code>{name}</code></p></td>
<td><p>Matches a path segment and captures it as a variable named &quot;name&quot;</p></td>
<td><p><code>&quot;/projects/{project}/versions&quot;</code> matches <code>&quot;/projects/spring/versions&quot;</code> and captures <code>project=spring</code></p></td>
</tr>
<tr class="odd">
<td><p><code>{name:[a-z]}+</code></p></td>
<td><p>Matches the regexp <code>&quot;[a-z]&quot;+</code> as a path variable named &quot;name&quot;</p></td>
<td><p><code>&quot;/projects/{project:[a-z]}/versions&quot;+</code> matches <code>&quot;/projects/spring/versions&quot;</code> but not <code>&quot;/projects/spring1/versions&quot;</code></p></td>
</tr>
<tr class="even">
<td><p><code>{*path}</code></p></td>
<td><p>Matches zero or more path segments until the end of the path and captures it as a variable named &quot;path&quot;</p></td>
<td><p><code>&quot;/resources/{*file}&quot;</code> matches <code>&quot;/resources/images/file.png&quot;</code> and captures <code>file=images/file.png</code></p></td>
</tr>
</tbody>
</table>

Captured URI variables can be accessed with `@PathVariable`, as the following example shows:

**Java.**

``` java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
    // ...
}
```

**Kotlin.**

``` kotlin
@GetMapping("/owners/{ownerId}/pets/{petId}")
fun findPet(@PathVariable ownerId: Long, @PathVariable petId: Long): Pet {
    // ...
}
```

You can declare URI variables at the class and method levels, as the following example shows:

**Java.**

``` java
@Controller
@RequestMapping("/owners/{ownerId}") 
public class OwnerController {

    @GetMapping("/pets/{petId}") 
    public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
        // ...
    }
}
```

  - Class-level URI mapping.

  - Method-level URI mapping.

**Kotlin.**

``` kotlin
@Controller
@RequestMapping("/owners/{ownerId}") 
class OwnerController {

    @GetMapping("/pets/{petId}") 
    fun findPet(@PathVariable ownerId: Long, @PathVariable petId: Long): Pet {
        // ...
    }
}
```

  - Class-level URI mapping.

  - Method-level URI mapping.

URI variables are automatically converted to the appropriate type or a `TypeMismatchException` is raised. Simple types (`int`, `long`, `Date`, and so on) are supported by default and you can register support for any other data type. See [section\_title](#webflux-ann-typeconversion) and [section\_title](#webflux-ann-initbinder).

URI variables can be named explicitly (for example, `@PathVariable("customId")`), but you can leave that detail out if the names are the same and you compile your code with debugging information or with the `-parameters` compiler flag on Java 8.

The syntax `{*varName}` declares a URI variable that matches zero or more remaining path segments. For example `/resources/{*path}` matches all files under `/resources/`, and the `"path"` variable captures the complete relative path.

The syntax `{varName:regex}` declares a URI variable with a regular expression that has the syntax: `{varName:regex}`. For example, given a URL of `/spring-web-3.0.5.jar`, the following method extracts the name, version, and file extension:

**Java.**

``` java
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
public void handle(@PathVariable String version, @PathVariable String ext) {
    // ...
}
```

**Kotlin.**

``` kotlin
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
fun handle(@PathVariable version: String, @PathVariable ext: String) {
    // ...
}
```

URI path patterns can also have embedded `${…​}` placeholders that are resolved on startup through `PropertyPlaceHolderConfigurer` against local, system, environment, and other property sources. You ca use this to, for example, parameterize a base URL based on some external configuration.

> **Note**
> 
> Spring WebFlux uses `PathPattern` and the `PathPatternParser` for URI path matching support. Both classes are located in `spring-web` and are expressly designed for use with HTTP URL paths in web applications where a large number of URI path patterns are matched at runtime.

Spring WebFlux does not support suffix pattern matching — unlike Spring MVC, where a mapping such as `/person` also matches to `/person.*`. For URL-based content negotiation, if needed, we recommend using a query parameter, which is simpler, more explicit, and less vulnerable to URL path based exploits.

### Pattern Comparison

[Web MVC](web.xml#mvc-ann-requestmapping-pattern-comparison)

When multiple patterns match a URL, they must be compared to find the best match. This is done with `PathPattern.SPECIFICITY_COMPARATOR`, which looks for patterns that are more specific.

For every pattern, a score is computed, based on the number of URI variables and wildcards, where a URI variable scores lower than a wildcard. A pattern with a lower total score wins. If two patterns have the same score, the longer is chosen.

Catch-all patterns (for example, `**`, `{*varName}`) are excluded from the scoring and are always sorted last instead. If two patterns are both catch-all, the longer is chosen.

### Consumable Media Types

[Web MVC](web.xml#mvc-ann-requestmapping-consumes)

You can narrow the request mapping based on the `Content-Type` of the request, as the following example shows:

**Java.**

``` java
@PostMapping(path = "/pets", consumes = "application/json")
public void addPet(@RequestBody Pet pet) {
    // ...
}
```

**Kotlin.**

``` kotlin
@PostMapping("/pets", consumes = ["application/json"])
fun addPet(@RequestBody pet: Pet) {
    // ...
}
```

The consumes attribute also supports negation expressions — for example, `!text/plain` means any content type other than `text/plain`.

You can declare a shared `consumes` attribute at the class level. Unlike most other request mapping attributes, however, when used at the class level, a method-level `consumes` attribute overrides rather than extends the class-level declaration.

> **Tip**
> 
> `MediaType` provides constants for commonly used media types — for example, `APPLICATION_JSON_VALUE` and `APPLICATION_XML_VALUE`.

### Producible Media Types

[Web MVC](web.xml#mvc-ann-requestmapping-produces)

You can narrow the request mapping based on the `Accept` request header and the list of content types that a controller method produces, as the following example shows:

**Java.**

``` java
@GetMapping(path = "/pets/{petId}", produces = "application/json")
@ResponseBody
public Pet getPet(@PathVariable String petId) {
    // ...
}
```

**Kotlin.**

``` kotlin
@GetMapping("/pets/{petId}", produces = ["application/json"])
@ResponseBody
fun getPet(@PathVariable String petId): Pet {
    // ...
}
```

The media type can specify a character set. Negated expressions are supported — for example, `!text/plain` means any content type other than `text/plain`.

You can declare a shared `produces` attribute at the class level. Unlike most other request mapping attributes, however, when used at the class level, a method-level `produces` attribute overrides rather than extend the class level declaration.

> **Tip**
> 
> `MediaType` provides constants for commonly used media types — e.g. `APPLICATION_JSON_VALUE`, `APPLICATION_XML_VALUE`.

### Parameters and Headers

[Web MVC](web.xml#mvc-ann-requestmapping-params-and-headers)

You can narrow request mappings based on query parameter conditions. You can test for the presence of a query parameter (`myParam`), for its absence (`!myParam`), or for a specific value (`myParam=myValue`). The following examples tests for a parameter with a value:

**Java.**

``` java
@GetMapping(path = "/pets/{petId}", params = "myParam=myValue") 
public void findPet(@PathVariable String petId) {
    // ...
}
```

  - Check that `myParam` equals `myValue`.

**Kotlin.**

``` kotlin
@GetMapping("/pets/{petId}", params = ["myParam=myValue"]) 
fun findPet(@PathVariable petId: String) {
    // ...
}
```

  - Check that `myParam` equals `myValue`.

You can also use the same with request header conditions, as the follwing example shows:

**Java.**

``` java
@GetMapping(path = "/pets", headers = "myHeader=myValue") 
public void findPet(@PathVariable String petId) {
    // ...
}
```

  - Check that `myHeader` equals `myValue`.

**Kotlin.**

``` kotlin
@GetMapping("/pets", headers = ["myHeader=myValue"]) 
fun findPet(@PathVariable petId: String) {
    // ...
}
```

  - Check that `myHeader` equals `myValue`.

### HTTP HEAD, OPTIONS

[Web MVC](web.xml#mvc-ann-requestmapping-head-options)

`@GetMapping` and `@RequestMapping(method=HttpMethod.GET)` support HTTP HEAD transparently for request mapping purposes. Controller methods need not change. A response wrapper, applied in the `HttpHandler` server adapter, ensures a `Content-Length` header is set to the number of bytes written without actually writing to the response.

By default, HTTP OPTIONS is handled by setting the `Allow` response header to the list of HTTP methods listed in all `@RequestMapping` methods with matching URL patterns.

For a `@RequestMapping` without HTTP method declarations, the `Allow` header is set to `GET,HEAD,POST,PUT,PATCH,DELETE,OPTIONS`. Controller methods should always declare the supported HTTP methods (for example, by using the HTTP method specific variants — `@GetMapping`, `@PostMapping`, and others).

You can explicitly map a `@RequestMapping` method to HTTP HEAD and HTTP OPTIONS, but that is not necessary in the common case.

### Custom Annotations

[Web MVC](web.xml#mvc-ann-requestmapping-composed)

Spring WebFlux supports the use of [composed annotations](core.xml#beans-meta-annotations) for request mapping. Those are annotations that are themselves meta-annotated with `@RequestMapping` and composed to redeclare a subset (or all) of the `@RequestMapping` attributes with a narrower, more specific purpose.

`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, and `@PatchMapping` are examples of composed annotations. They are provided, because, arguably, most controller methods should be mapped to a specific HTTP method versus using `@RequestMapping`, which, by default, matches to all HTTP methods. If you need an example of composed annotations, look at how those are declared.

Spring WebFlux also supports custom request mapping attributes with custom request matching logic. This is a more advanced option that requires sub-classing `RequestMappingHandlerMapping` and overriding the `getCustomMethodCondition` method, where you can check the custom attribute and return your own `RequestCondition`.

### Explicit Registrations

[Web MVC](web.xml#mvc-ann-requestmapping-registration)

You can programmatically register Handler methods, which can be used for dynamic registrations or for advanced cases, such as different instances of the same handler under different URLs. The following example shows how to do so:

**Java.**

``` java
@Configuration
public class MyConfig {

    @Autowired
    public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler) 
            throws NoSuchMethodException {

        RequestMappingInfo info = RequestMappingInfo
                .paths("/user/{id}").methods(RequestMethod.GET).build(); 

        Method method = UserHandler.class.getMethod("getUser", Long.class); 

        mapping.registerMapping(info, handler, method); 
    }

}
```

  - Inject target handlers and the handler mapping for controllers.

  - Prepare the request mapping metadata.

  - Get the handler method.

  - Add the registration.

**Kotlin.**

``` kotlin
@Configuration
class MyConfig {

    @Autowired
    fun setHandlerMapping(mapping: RequestMappingHandlerMapping, handler: UserHandler) { 

        val info = RequestMappingInfo.paths("/user/{id}").methods(RequestMethod.GET).build() 

        val method = UserHandler::class.java.getMethod("getUser", Long::class.java) 

        mapping.registerMapping(info, handler, method) 
    }
}
```

  - Inject target handlers and the handler mapping for controllers.

  - Prepare the request mapping metadata.

  - Get the handler method.

  - Add the registration.

## Handler Methods

[Web MVC](web.xml#mvc-ann-methods)

`@RequestMapping` handler methods have a flexible signature and can choose from a range of supported controller method arguments and return values.

### Method Arguments

[Web MVC](web.xml#mvc-ann-arguments)

The following table shows the supported controller method arguments.

Reactive types (Reactor, RxJava, [or other](#webflux-reactive-libraries)) are supported on arguments that require blocking I/O (for example, reading the request body) to be resolved. This is marked in the Description column. Reactive types are not expected on arguments that do not require blocking.

JDK 1.8’s `java.util.Optional` is supported as a method argument in combination with annotations that have a `required` attribute (for example, `@RequestParam`, `@RequestHeader`, and others) and is equivalent to `required=false`.

<table>
<colgroup>
<col style="width: 33%" />
<col style="width: 66%" />
</colgroup>
<thead>
<tr class="header">
<th>Controller method argument</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>ServerWebExchange</code></p></td>
<td><p>Access to the full <code>ServerWebExchange</code> — container for the HTTP request and response, request and session attributes, <code>checkNotModified</code> methods, and others.</p></td>
</tr>
<tr class="even">
<td><p><code>ServerHttpRequest</code>, <code>ServerHttpResponse</code></p></td>
<td><p>Access to the HTTP request or response.</p></td>
</tr>
<tr class="odd">
<td><p><code>WebSession</code></p></td>
<td><p>Access to the session. This does not force the start of a new session unless attributes are added. Supports reactive types.</p></td>
</tr>
<tr class="even">
<td><p><code>java.security.Principal</code></p></td>
<td><p>The currently authenticated user — possibly a specific <code>Principal</code> implementation class if known. Supports reactive types.</p></td>
</tr>
<tr class="odd">
<td><p><code>org.springframework.http.HttpMethod</code></p></td>
<td><p>The HTTP method of the request.</p></td>
</tr>
<tr class="even">
<td><p><code>java.util.Locale</code></p></td>
<td><p>The current request locale, determined by the most specific <code>LocaleResolver</code> available — in effect, the configured <code>LocaleResolver</code>/<code>LocaleContextResolver</code>.</p></td>
</tr>
<tr class="odd">
<td><p><code>java.util.TimeZone</code> + <code>java.time.ZoneId</code></p></td>
<td><p>The time zone associated with the current request, as determined by a <code>LocaleContextResolver</code>.</p></td>
</tr>
<tr class="even">
<td><p><code>@PathVariable</code></p></td>
<td><p>For access to URI template variables. See <a href="#webflux-ann-requestmapping-uri-templates">section_title</a>.</p></td>
</tr>
<tr class="odd">
<td><p><code>@MatrixVariable</code></p></td>
<td><p>For access to name-value pairs in URI path segments. See <a href="#webflux-ann-matrix-variables">section_title</a>.</p></td>
</tr>
<tr class="even">
<td><p><code>@RequestParam</code></p></td>
<td><p>For access to Servlet request parameters. Parameter values are converted to the declared method argument type. See <a href="#webflux-ann-requestparam">section_title</a>.</p>
<p>Note that use of <code>@RequestParam</code> is optional — for example, to set its attributes. See “Any other argument” later in this table.</p></td>
</tr>
<tr class="odd">
<td><p><code>@RequestHeader</code></p></td>
<td><p>For access to request headers. Header values are converted to the declared method argument type. See <a href="#webflux-ann-requestheader">section_title</a>.</p></td>
</tr>
<tr class="even">
<td><p><code>@CookieValue</code></p></td>
<td><p>For access to cookies. Cookie values are converted to the declared method argument type. See <a href="#webflux-ann-cookievalue">section_title</a>.</p></td>
</tr>
<tr class="odd">
<td><p><code>@RequestBody</code></p></td>
<td><p>For access to the HTTP request body. Body content is converted to the declared method argument type by using <code>HttpMessageReader</code> instances. Supports reactive types. See <a href="#webflux-ann-requestbody">section_title</a>.</p></td>
</tr>
<tr class="even">
<td><p><code>HttpEntity&lt;B&gt;</code></p></td>
<td><p>For access to request headers and body. The body is converted with <code>HttpMessageReader</code> instances. Supports reactive types. See <a href="#webflux-ann-httpentity">section_title</a>.</p></td>
</tr>
<tr class="odd">
<td><p><code>@RequestPart</code></p></td>
<td><p>For access to a part in a <code>multipart/form-data</code> request. Supports reactive types. See <a href="#webflux-multipart-forms">section_title</a> and <a href="#webflux-multipart">section_title</a>.</p></td>
</tr>
<tr class="even">
<td><p><code>java.util.Map</code>, <code>org.springframework.ui.Model</code>, and <code>org.springframework.ui.ModelMap</code>.</p></td>
<td><p>For access to the model that is used in HTML controllers and is exposed to templates as part of view rendering.</p></td>
</tr>
<tr class="odd">
<td><p><code>@ModelAttribute</code></p></td>
<td><p>For access to an existing attribute in the model (instantiated if not present) with data binding and validation applied. See <a href="#webflux-ann-modelattrib-method-args">section_title</a> as well as <a href="#webflux-ann-modelattrib-methods">section_title</a> and <a href="#webflux-ann-initbinder">section_title</a>.</p>
<p>Note that use of <code>@ModelAttribute</code> is optional — for example, to set its attributes. See “Any other argument” later in this table.</p></td>
</tr>
<tr class="even">
<td><p><code>Errors</code>, <code>BindingResult</code></p></td>
<td><p>For access to errors from validation and data binding for a command object, i.e. a <code>@ModelAttribute</code> argument. An <code>Errors</code>, or <code>BindingResult</code> argument must be declared immediately after the validated method argument.</p></td>
</tr>
<tr class="odd">
<td><p><code>SessionStatus</code> + class-level <code>@SessionAttributes</code></p></td>
<td><p>For marking form processing complete, which triggers cleanup of session attributes declared through a class-level <code>@SessionAttributes</code> annotation. See <a href="#webflux-ann-sessionattributes">section_title</a> for more details.</p></td>
</tr>
<tr class="even">
<td><p><code>UriComponentsBuilder</code></p></td>
<td><p>For preparing a URL relative to the current request’s host, port, scheme, and context path. See <a href="#webflux-uri-building">section_title</a>.</p></td>
</tr>
<tr class="odd">
<td><p><code>@SessionAttribute</code></p></td>
<td><p>For access to any session attribute — in contrast to model attributes stored in the session as a result of a class-level <code>@SessionAttributes</code> declaration. See <a href="#webflux-ann-sessionattribute">section_title</a> for more details.</p></td>
</tr>
<tr class="even">
<td><p><code>@RequestAttribute</code></p></td>
<td><p>For access to request attributes. See <a href="#webflux-ann-requestattrib">section_title</a> for more details.</p></td>
</tr>
<tr class="odd">
<td><p>Any other argument</p></td>
<td><p>If a method argument is not matched to any of the above, it is, by default, resolved as a <code>@RequestParam</code> if it is a simple type, as determined by {api-spring-framework}/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-[BeanUtils#isSimpleProperty], or as a <code>@ModelAttribute</code>, otherwise.</p></td>
</tr>
</tbody>
</table>

### Return Values

[Web MVC](web.xml#mvc-ann-return-types)

The following table shows the supported controller method return values. Note that reactive types from libraries such as Reactor, RxJava, [or other](#webflux-reactive-libraries) are generally supported for all return values.

<table>
<colgroup>
<col style="width: 33%" />
<col style="width: 66%" />
</colgroup>
<thead>
<tr class="header">
<th>Controller method return value</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>@ResponseBody</code></p></td>
<td><p>The return value is encoded through <code>HttpMessageWriter</code> instances and written to the response. See <a href="#webflux-ann-responsebody">section_title</a>.</p></td>
</tr>
<tr class="even">
<td><p><code>HttpEntity&lt;B&gt;</code>, <code>ResponseEntity&lt;B&gt;</code></p></td>
<td><p>The return value specifies the full response, including HTTP headers, and the body is encoded through <code>HttpMessageWriter</code> instances and written to the response. See <a href="#webflux-ann-responseentity">section_title</a>.</p></td>
</tr>
<tr class="odd">
<td><p><code>HttpHeaders</code></p></td>
<td><p>For returning a response with headers and no body.</p></td>
</tr>
<tr class="even">
<td><p><code>String</code></p></td>
<td><p>A view name to be resolved with <code>ViewResolver</code> instances and used together with the implicit model — determined through command objects and <code>@ModelAttribute</code> methods. The handler method can also programmatically enrich the model by declaring a <code>Model</code> argument (described <a href="#webflux-viewresolution-handling">earlier</a>).</p></td>
</tr>
<tr class="odd">
<td><p><code>View</code></p></td>
<td><p>A <code>View</code> instance to use for rendering together with the implicit model — determined through command objects and <code>@ModelAttribute</code> methods. The handler method can also programmatically enrich the model by declaring a <code>Model</code> argument (described <a href="#webflux-viewresolution-handling">earlier</a>).</p></td>
</tr>
<tr class="even">
<td><p><code>java.util.Map</code>, <code>org.springframework.ui.Model</code></p></td>
<td><p>Attributes to be added to the implicit model, with the view name implicitly determined based on the request path.</p></td>
</tr>
<tr class="odd">
<td><p><code>@ModelAttribute</code></p></td>
<td><p>An attribute to be added to the model, with the view name implicitly determined based on the request path.</p>
<p>Note that <code>@ModelAttribute</code> is optional. See “Any other return value” later in this table.</p></td>
</tr>
<tr class="even">
<td><p><code>Rendering</code></p></td>
<td><p>An API for model and view rendering scenarios.</p></td>
</tr>
<tr class="odd">
<td><p><code>void</code></p></td>
<td><p>A method with a <code>void</code>, possibly asynchronous (for example, <code>Mono&lt;Void&gt;</code>), return type (or a <code>null</code> return value) is considered to have fully handled the response if it also has a <code>ServerHttpResponse</code>, a <code>ServerWebExchange</code> argument, or an <code>@ResponseStatus</code> annotation. The same is also true if the controller has made a positive ETag or <code>lastModified</code> timestamp check. // TODO: See <a href="#webflux-caching-etag-lastmodified">section_title</a> for details.</p>
<p>If none of the above is true, a <code>void</code> return type can also indicate “no response body” for REST controllers or default view name selection for HTML controllers.</p></td>
</tr>
<tr class="even">
<td><p><code>Flux&lt;ServerSentEvent&gt;</code>, <code>Observable&lt;ServerSentEvent&gt;</code>, or other reactive type</p></td>
<td><p>Emit server-sent events. The <code>ServerSentEvent</code> wrapper can be omitted when only data needs to be written (however, <code>text/event-stream</code> must be requested or declared in the mapping through the <code>produces</code> attribute).</p></td>
</tr>
<tr class="odd">
<td><p>Any other return value</p></td>
<td><p>If a return value is not matched to any of the above, it is, by default, treated as a view name, if it is <code>String</code> or <code>void</code> (default view name selection applies), or as a model attribute to be added to the model, unless it is a simple type, as determined by {api-spring-framework}/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-[BeanUtils#isSimpleProperty], in which case it remains unresolved.</p></td>
</tr>
</tbody>
</table>

### Type Conversion

[Web MVC](web.xml#mvc-ann-typeconversion)

Some annotated controller method arguments that represent String-based request input (for example, `@RequestParam`, `@RequestHeader`, `@PathVariable`, `@MatrixVariable`, and `@CookieValue`) can require type conversion if the argument is declared as something other than `String`.

For such cases, type conversion is automatically applied based on the configured converters. By default, simple types (such as `int`, `long`, `Date`, and others) are supported. Type conversion can be customized through a `WebDataBinder` (see [section\_title](#webflux-ann-initbinder)) or by registering `Formatters` with the `FormattingConversionService` (see [Spring Field Formatting](core.xml#format)).

A practical issue in type conversion is the treatment of an empty String source value. Such a value is treated as missing if it becomes `null` as a result of type conversion. This can be the case for `Long`, `UUID`, and other target types. If you want to allow `null` to be injected, either use the `required` flag on the argument annotation, or declare the argument as `@Nullable`.

### Matrix Variables

[Web MVC](web.xml#mvc-ann-matrix-variables)

[RFC 3986](https://tools.ietf.org/html/rfc3986#section-3.3) discusses name-value pairs in path segments. In Spring WebFlux, we refer to those as “matrix variables” based on an [“old post”](https://www.w3.org/DesignIssues/MatrixURIs.html) by Tim Berners-Lee, but they can be also be referred to as URI path parameters.

Matrix variables can appear in any path segment, with each variable separated by a semicolon and multiple values separated by commas — for example, `"/cars;color=red,green;year=2012"`. Multiple values can also be specified through repeated variable names — for example, `"color=red;color=green;color=blue"`.

Unlike Spring MVC, in WebFlux, the presence or absence of matrix variables in a URL does not affect request mappings. In other words, you are not required to use a URI variable to mask variable content. That said, if you want to access matrix variables from a controller method, you need to add a URI variable to the path segment where matrix variables are expected. The following example shows how to do so:

**Java.**

``` java
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
public void findPet(@PathVariable String petId, @MatrixVariable int q) {

    // petId == 42
    // q == 11
}
```

**Kotlin.**

``` kotlin
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
fun findPet(@PathVariable petId: String, @MatrixVariable q: Int) {

    // petId == 42
    // q == 11
}
```

Given that all path segments can contain matrix variables, you may sometimes need to disambiguate which path variable the matrix variable is expected to be in, as the following example shows:

**Java.**

``` java
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable(name="q", pathVar="ownerId") int q1,
        @MatrixVariable(name="q", pathVar="petId") int q2) {

    // q1 == 11
    // q2 == 22
}
```

**Kotlin.**

``` kotlin
@GetMapping("/owners/{ownerId}/pets/{petId}")
fun findPet(
        @MatrixVariable(name = "q", pathVar = "ownerId") q1: Int,
        @MatrixVariable(name = "q", pathVar = "petId") q2: Int) {

    // q1 == 11
    // q2 == 22
}
```

You can define a matrix variable may be defined as optional and specify a default value as the following example shows:

**Java.**

``` java
// GET /pets/42

@GetMapping("/pets/{petId}")
public void findPet(@MatrixVariable(required=false, defaultValue="1") int q) {

    // q == 1
}
```

**Kotlin.**

``` kotlin
// GET /pets/42

@GetMapping("/pets/{petId}")
fun findPet(@MatrixVariable(required = false, defaultValue = "1") q: Int) {

    // q == 1
}
```

To get all matrix variables, use a `MultiValueMap`, as the following example shows:

**Java.**

``` java
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable MultiValueMap<String, String> matrixVars,
        @MatrixVariable(pathVar="petId") MultiValueMap<String, String> petMatrixVars) {

    // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
    // petMatrixVars: ["q" : 22, "s" : 23]
}
```

**Kotlin.**

``` kotlin
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
fun findPet(
        @MatrixVariable matrixVars: MultiValueMap<String, String>,
        @MatrixVariable(pathVar="petId") petMatrixVars: MultiValueMap<String, String>) {

    // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
    // petMatrixVars: ["q" : 22, "s" : 23]
}
```

### `@RequestParam`

[Web MVC](web.xml#mvc-ann-requestparam)

You can use the `@RequestParam` annotation to bind query parameters to a method argument in a controller. The following code snippet shows the usage:

**Java.**

``` java
@Controller
@RequestMapping("/pets")
public class EditPetForm {

    // ...

    @GetMapping
    public String setupForm(@RequestParam("petId") int petId, Model model) { 
        Pet pet = this.clinic.loadPet(petId);
        model.addAttribute("pet", pet);
        return "petForm";
    }

    // ...
}
```

  - Using `@RequestParam`.

**Kotlin.**

``` kotlin
import org.springframework.ui.set

@Controller
@RequestMapping("/pets")
class EditPetForm {

    // ...

    @GetMapping
    fun setupForm(@RequestParam("petId") petId: Int, model: Model): String { 
        val pet = clinic.loadPet(petId)
        model["pet"] = pet
        return "petForm"
    }

    // ...
}
```

  - Using `@RequestParam`.

> **Tip**
> 
> The Servlet API “request parameter” concept conflates query parameters, form data, and multiparts into one. However, in WebFlux, each is accessed individually through `ServerWebExchange`. While `@RequestParam` binds to query parameters only, you can use data binding to apply query parameters, form data, and multiparts to a [command object](#webflux-ann-modelattrib-method-args).

Method parameters that use the `@RequestParam` annotation are required by default, but you can specify that a method parameter is optional by setting the required flag of a `@RequestParam` to `false` or by declaring the argument with a `java.util.Optional` wrapper.

Type conversion is applied automatically if the target method parameter type is not `String`. See [section\_title](#webflux-ann-typeconversion).

When a `@RequestParam` annotation is declared on a `Map<String, String>` or `MultiValueMap<String, String>` argument, the map is populated with all query parameters.

Note that use of `@RequestParam` is optional — for example, to set its attributes. By default, any argument that is a simple value type (as determined by {api-spring-framework}/beans/BeanUtils.html\#isSimpleProperty-java.lang.Class-\[BeanUtils\#isSimpleProperty\]) and is not resolved by any other argument resolver is treated as if it were annotated with `@RequestParam`.

### `@RequestHeader`

[Web MVC](web.xml#mvc-ann-requestheader)

You can use the `@RequestHeader` annotation to bind a request header to a method argument in a controller.

The following example shows a request with headers:

    Host                    localhost:8080
    Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
    Accept-Language         fr,en-gb;q=0.7,en;q=0.3
    Accept-Encoding         gzip,deflate
    Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
    Keep-Alive              300

The following example gets the value of the `Accept-Encoding` and `Keep-Alive` headers:

**Java.**

``` java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, 
        @RequestHeader("Keep-Alive") long keepAlive) { 
    //...
}
```

  - Get the value of the `Accept-Encoging` header.

  - Get the value of the `Keep-Alive` header.

**Kotlin.**

``` kotlin
@GetMapping("/demo")
fun handle(
        @RequestHeader("Accept-Encoding") encoding: String, 
        @RequestHeader("Keep-Alive") keepAlive: Long) { 
    //...
}
```

  - Get the value of the `Accept-Encoging` header.

  - Get the value of the `Keep-Alive` header.

Type conversion is applied automatically if the target method parameter type is not `String`. See [section\_title](#webflux-ann-typeconversion).

When a `@RequestHeader` annotation is used on a `Map<String, String>`, `MultiValueMap<String, String>`, or `HttpHeaders` argument, the map is populated with all header values.

> **Tip**
> 
> Built-in support is available for converting a comma-separated string into an array or collection of strings or other types known to the type conversion system. For example, a method parameter annotated with `@RequestHeader("Accept")` may be of type `String` but also of `String[]` or `List<String>`.

### `@CookieValue`

[Web MVC](web.xml#mvc-ann-cookievalue)

You can use the `@CookieValue` annotation to bind the value of an HTTP cookie to a method argument in a controller.

The following example shows a request with a cookie:

    JSESSIONID=415A4AC178C59DACE0B2C9CA727CDD84

The following code sample demonstrates how to get the cookie value:

**Java.**

``` java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { 
    //...
}
```

  - Get the cookie value.

**Kotlin.**

``` kotlin
@GetMapping("/demo")
fun handle(@CookieValue("JSESSIONID") cookie: String) { 
    //...
}
```

  - Get the cookie value.

Type conversion is applied automatically if the target method parameter type is not `String`. See [section\_title](#webflux-ann-typeconversion).

### `@ModelAttribute`

[Web MVC](web.xml#mvc-ann-modelattrib-method-args)

You can use the `@ModelAttribute` annotation on a method argument to access an attribute from the model or have it instantiated if not present. The model attribute is also overlaid with the values of query parameters and form fields whose names match to field names. This is referred to as data binding, and it saves you from having to deal with parsing and converting individual query parameters and form fields. The following example binds an instance of `Pet`:

**Java.**

``` java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) { } 
```

  - Bind an instance of `Pet`.

**Kotlin.**

``` kotlin
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
fun processSubmit(@ModelAttribute pet: Pet): String { } 
```

  - Bind an instance of `Pet`.

The `Pet` instance in the preceding example is resolved as follows:

  - From the model if already added through [section\_title](#webflux-ann-modelattrib-methods).

  - From the HTTP session through [section\_title](#webflux-ann-sessionattributes).

  - From the invocation of a default constructor.

  - From the invocation of a “primary constructor” with arguments that match query parameters or form fields. Argument names are determined through JavaBeans `@ConstructorProperties` or through runtime-retained parameter names in the bytecode.

After the model attribute instance is obtained, data binding is applied. The `WebExchangeDataBinder` class matches names of query parameters and form fields to field names on the target `Object`. Matching fields are populated after type conversion is applied where necessary. For more on data binding (and validation), see [Validation](core.xml#validation). For more on customizing data binding, see [section\_title](#webflux-ann-initbinder).

Data binding can result in errors. By default, a `WebExchangeBindException` is raised, but, to check for such errors in the controller method, you can add a `BindingResult` argument immediately next to the `@ModelAttribute`, as the following example shows:

**Java.**

``` java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

  - Adding a `BindingResult`.

**Kotlin.**

``` kotlin
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
fun processSubmit(@ModelAttribute("pet") pet: Pet, result: BindingResult): String { 
    if (result.hasErrors()) {
        return "petForm"
    }
    // ...
}
```

  - Adding a `BindingResult`.

You can automatically apply validation after data binding by adding the `javax.validation.Valid` annotation or Spring’s `@Validated` annotation (see also [Bean Validation](core.xml#validation-beanvalidation) and [Spring validation](core.xml#validation)). The following example uses the `@Valid` annotation:

**Java.**

``` java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

  - Using `@Valid` on a model attribute argument.

**Kotlin.**

``` kotlin
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
fun processSubmit(@Valid @ModelAttribute("pet") pet: Pet, result: BindingResult): String { 
    if (result.hasErrors()) {
        return "petForm"
    }
    // ...
}
```

  - Using `@Valid` on a model attribute argument.

Spring WebFlux, unlike Spring MVC, supports reactive types in the model — for example, `Mono<Account>` or `io.reactivex.Single<Account>`. You can declare a `@ModelAttribute` argument with or without a reactive type wrapper, and it will be resolved accordingly, to the actual value if necessary. However, note that, to use a `BindingResult` argument, you must declare the `@ModelAttribute` argument before it without a reactive type wrapper, as shown earlier. Alternatively, you can handle any errors through the reactive type, as the following example shows:

**Java.**

``` java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public Mono<String> processSubmit(@Valid @ModelAttribute("pet") Mono<Pet> petMono) {
    return petMono
        .flatMap(pet -> {
            // ...
        })
        .onErrorResume(ex -> {
            // ...
        });
}
```

**Kotlin.**

``` kotlin
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
fun processSubmit(@Valid @ModelAttribute("pet") petMono: Mono<Pet>): Mono<String> {
    return petMono
            .flatMap { pet ->
                // ...
            }
            .onErrorResume{ ex ->
                // ...
            }
}
```

Note that use of `@ModelAttribute` is optional — for example, to set its attributes. By default, any argument that is not a simple value type( as determined by {api-spring-framework}/beans/BeanUtils.html\#isSimpleProperty-java.lang.Class-\[BeanUtils\#isSimpleProperty\]) and is not resolved by any other argument resolver is treated as if it were annotated with `@ModelAttribute`.

### `@SessionAttributes`

[Web MVC](web.xml#mvc-ann-sessionattributes)

`@SessionAttributes` is used to store model attributes in the `WebSession` between requests. It is a type-level annotation that declares session attributes used by a specific controller. This typically lists the names of model attributes or types of model attributes that should be transparently stored in the session for subsequent requests to access.

Consider the following example:

**Java.**

``` java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {
    // ...
}
```

  - Using the `@SessionAttributes` annotation.

**Kotlin.**

``` kotlin
@Controller
@SessionAttributes("pet") 
class EditPetForm {
    // ...
}
```

  - Using the `@SessionAttributes` annotation.

On the first request, when a model attribute with the name, `pet`, is added to the model, it is automatically promoted to and saved in the `WebSession`. It remains there until another controller method uses a `SessionStatus` method argument to clear the storage, as the following example shows:

**Java.**

``` java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {

    // ...

    @PostMapping("/pets/{id}")
    public String handle(Pet pet, BindingResult errors, SessionStatus status) { 
        if (errors.hasErrors()) {
            // ...
        }
            status.setComplete();
            // ...
        }
    }
}
```

  - Using the `@SessionAttributes` annotation.

  - Using a `SessionStatus` variable.

**Kotlin.**

``` kotlin
@Controller
@SessionAttributes("pet") 
class EditPetForm {

    // ...

    @PostMapping("/pets/{id}")
    fun handle(pet: Pet, errors: BindingResult, status: SessionStatus): String { 
        if (errors.hasErrors()) {
            // ...
        }
        status.setComplete()
        // ...
    }
}
```

  - Using the `@SessionAttributes` annotation.

  - Using a `SessionStatus` variable.

### `@SessionAttribute`

[Web MVC](web.xml#mvc-ann-sessionattribute)

If you need access to pre-existing session attributes that are managed globally (that is, outside the controller — for example, by a filter) and may or may not be present, you can use the `@SessionAttribute` annotation on a method parameter, as the following example shows:

**Java.**

``` java
@GetMapping("/")
public String handle(@SessionAttribute User user) { 
    // ...
}
```

  - Using `@SessionAttribute`.

**Kotlin.**

``` kotlin
@GetMapping("/")
fun handle(@SessionAttribute user: User): String { 
    // ...
}
```

  - Using `@SessionAttribute`.

For use cases that require adding or removing session attributes, consider injecting `WebSession` into the controller method.

For temporary storage of model attributes in the session as part of a controller workflow, consider using `SessionAttributes`, as described in [section\_title](#webflux-ann-sessionattributes).

### `@RequestAttribute`

[Web MVC](web.xml#mvc-ann-requestattrib)

Similarly to `@SessionAttribute`, you can use the `@RequestAttribute` annotation to access pre-existing request attributes created earlier (for example, by a `WebFilter`), as the following example shows:

**Java.**

``` java
@GetMapping("/")
public String handle(@RequestAttribute Client client) { 
    // ...
}
```

  - Using `@RequestAttribute`.

**Kotlin.**

``` kotlin
@GetMapping("/")
fun handle(@RequestAttribute client: Client): String { 
    // ...
}
```

  - Using `@RequestAttribute`.

### Multipart Content

[Web MVC](web.xml#mvc-multipart-forms)

As explained in [section\_title](#webflux-multipart), `ServerWebExchange` provides access to multipart content. The best way to handle a file upload form (for example, from a browser) in a controller is through data binding to a [command object](#webflux-ann-modelattrib-method-args), as the following example shows:

**Java.**

``` java
class MyForm {

    private String name;

    private MultipartFile file;

    // ...

}

@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(MyForm form, BindingResult errors) {
        // ...
    }

}
```

**Kotlin.**

``` kotlin
class MyForm(
        val name: String,
        val file: MultipartFile)

@Controller
class FileUploadController {

    @PostMapping("/form")
    fun handleFormUpload(form: MyForm, errors: BindingResult): String {
        // ...
    }

}
```

You can also submit multipart requests from non-browser clients in a RESTful service scenario. The following example uses a file along with JSON:

    POST /someUrl
    Content-Type: multipart/mixed
    
    --edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
    Content-Disposition: form-data; name="meta-data"
    Content-Type: application/json; charset=UTF-8
    Content-Transfer-Encoding: 8bit
    
    {
        "name": "value"
    }
    --edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
    Content-Disposition: form-data; name="file-data"; filename="file.properties"
    Content-Type: text/xml
    Content-Transfer-Encoding: 8bit
    ... File Data ...

You can access individual parts with `@RequestPart`, as the following example shows:

**Java.**

``` java
@PostMapping("/")
public String handle(@RequestPart("meta-data") Part metadata, 
        @RequestPart("file-data") FilePart file) { 
    // ...
}
```

  - Using `@RequestPart` to get the metadata.

  - Using `@RequestPart` to get the file.

**Kotlin.**

``` kotlin
@PostMapping("/")
fun handle(@RequestPart("meta-data") Part metadata, 
        @RequestPart("file-data") FilePart file): String { 
    // ...
}
```

  - Using `@RequestPart` to get the metadata.

  - Using `@RequestPart` to get the file.

To deserialize the raw part content (for example, to JSON — similar to `@RequestBody`), you can declare a concrete target `Object`, instead of `Part`, as the following example shows:

**Java.**

``` java
@PostMapping("/")
public String handle(@RequestPart("meta-data") MetaData metadata) { 
    // ...
}
```

  - Using `@RequestPart` to get the metadata.

**Kotlin.**

``` kotlin
@PostMapping("/")
fun handle(@RequestPart("meta-data") metadata: MetaData): String { 
    // ...
}
```

  - Using `@RequestPart` to get the metadata.

You can use `@RequestPart` in combination with `javax.validation.Valid` or Spring’s `@Validated` annotation, which causes Standard Bean Validation to be applied. Validation errors lead to a `WebExchangeBindException` that results in a 400 (BAD\_REQUEST) response. The exception contains a `BindingResult` with the error details and can also be handled in the controller method by declaring the argument with an async wrapper and then using error related operators:

**Java.**

``` java
@PostMapping("/")
public String handle(@Valid @RequestPart("meta-data") Mono<MetaData> metadata) {
    // use one of the onError* operators...
}
```

**Kotlin.**

``` kotlin
@PostMapping("/")
fun handle(@Valid @RequestPart("meta-data") metadata: MetaData): String {
    // ...
}
```

To access all multipart data as a `MultiValueMap`, you can use `@RequestBody`, as the following example shows:

**Java.**

``` java
@PostMapping("/")
public String handle(@RequestBody Mono<MultiValueMap<String, Part>> parts) { 
    // ...
}
```

  - Using `@RequestBody`.

**Kotlin.**

``` kotlin
@PostMapping("/")
fun handle(@RequestBody parts: MultiValueMap<String, Part>): String { 
    // ...
}
```

  - Using `@RequestBody`.

To access multipart data sequentially, in streaming fashion, you can use `@RequestBody` with `Flux<Part>` (or `Flow<Part>` in Kotlin) instead, as the following example shows:

**Java.**

``` java
@PostMapping("/")
public String handle(@RequestBody Flux<Part> parts) { 
    // ...
}
```

  - Using `@RequestBody`.

**Kotlin.**

``` kotlin
@PostMapping("/")
fun handle(@RequestBody parts: Flow<Part>): String { 
    // ...
}
```

  - Using `@RequestBody`.

### `@RequestBody`

[Web MVC](web.xml#mvc-ann-requestbody)

You can use the `@RequestBody` annotation to have the request body read and deserialized into an `Object` through an [HttpMessageReader](#webflux-codecs). The following example uses a `@RequestBody` argument:

**Java.**

``` java
@PostMapping("/accounts")
public void handle(@RequestBody Account account) {
    // ...
}
```

**Kotlin.**

``` kotlin
@PostMapping("/accounts")
fun handle(@RequestBody account: Account) {
    // ...
}
```

Unlike Spring MVC, in WebFlux, the `@RequestBody` method argument supports reactive types and fully non-blocking reading and (client-to-server) streaming.

**Java.**

``` java
@PostMapping("/accounts")
public void handle(@RequestBody Mono<Account> account) {
    // ...
}
```

**Kotlin.**

``` kotlin
@PostMapping("/accounts")
fun handle(@RequestBody accounts: Flow<Account>) {
    // ...
}
```

You can use the [section\_title](#webflux-config-message-codecs) option of the [section\_title](#webflux-config) to configure or customize message readers.

You can use `@RequestBody` in combination with `javax.validation.Valid` or Spring’s `@Validated` annotation, which causes Standard Bean Validation to be applied. Validation errors cause a `WebExchangeBindException`, which results in a 400 (BAD\_REQUEST) response. The exception contains a `BindingResult` with error details and can be handled in the controller method by declaring the argument with an async wrapper and then using error related operators:

**Java.**

``` java
@PostMapping("/accounts")
public void handle(@Valid @RequestBody Mono<Account> account) {
    // use one of the onError* operators...
}
```

**Kotlin.**

``` kotlin
@PostMapping("/accounts")
fun handle(@Valid @RequestBody account: Mono<Account>) {
    // ...
}
```

### `HttpEntity`

[Web MVC](web.xml#mvc-ann-httpentity)

`HttpEntity` is more or less identical to using [section\_title](#webflux-ann-requestbody) but is based on a container object that exposes request headers and the body. The following example uses an `HttpEntity`:

**Java.**

``` java
@PostMapping("/accounts")
public void handle(HttpEntity<Account> entity) {
    // ...
}
```

**Kotlin.**

``` kotlin
@PostMapping("/accounts")
fun handle(entity: HttpEntity<Account>) {
    // ...
}
```

### `@ResponseBody`

[Web MVC](web.xml#mvc-ann-responsebody)

You can use the `@ResponseBody` annotation on a method to have the return serialized to the response body through an [HttpMessageWriter](#webflux-codecs). The following example shows how to do so:

**Java.**

``` java
@GetMapping("/accounts/{id}")
@ResponseBody
public Account handle() {
    // ...
}
```

**Kotlin.**

``` kotlin
@GetMapping("/accounts/{id}")
@ResponseBody
fun handle(): Account {
    // ...
}
```

`@ResponseBody` is also supported at the class level, in which case it is inherited by all controller methods. This is the effect of `@RestController`, which is nothing more than a meta-annotation marked with `@Controller` and `@ResponseBody`.

`@ResponseBody` supports reactive types, which means you can return Reactor or RxJava types and have the asynchronous values they produce rendered to the response. For additional details, see [section\_title](#webflux-codecs-streaming) and [JSON rendering](#webflux-codecs-jackson).

You can combine `@ResponseBody` methods with JSON serialization views. See [section\_title](#webflux-ann-jackson) for details.

You can use the [section\_title](#webflux-config-message-codecs) option of the [section\_title](#webflux-config) to configure or customize message writing.

### `ResponseEntity`

[Web MVC](web.xml#mvc-ann-responseentity)

`ResponseEntity` is like [section\_title](#webflux-ann-responsebody) but with status and headers. For example:

**Java.**

``` java
@GetMapping("/something")
public ResponseEntity<String> handle() {
    String body = ... ;
    String etag = ... ;
    return ResponseEntity.ok().eTag(etag).build(body);
}
```

**Kotlin.**

``` kotlin
@GetMapping("/something")
fun handle(): ResponseEntity<String> {
    val body: String = ...
    val etag: String = ...
    return ResponseEntity.ok().eTag(etag).build(body)
}
```

WebFlux supports using a single value [reactive type](#webflux-reactive-libraries) to produce the `ResponseEntity` asynchronously, and/or single and multi-value reactive types for the body. This allows a variety of async responses with `ResponseEntity` as follows:

  - `ResponseEntity<Mono<T>>` or `ResponseEntity<Flux<T>>` make the response status and headers known immediately while the body is provided asynchronously at a later point. Use `Mono` if the body consists of 0..1 values or `Flux` if it can produce multiple values.

  - `Mono<ResponseEntity<T>>` provides all three — response status, headers, and body, asynchronously at a later point. This allows the response status and headers to vary depending on the outcome of asynchronous request handling.

  - `Mono<ResponseEntity<Mono<T>>>` or `Mono<ResponseEntity<Flux<T>>>` are yet another possible, albeit less common alternative. They provide the response status and headers asynchronously first and then the response body, also asynchronously, second.

### Jackson JSON

Spring offers support for the Jackson JSON library.

#### JSON Views

[Web MVC](web.xml#mvc-ann-jackson)

Spring WebFlux provides built-in support for [Jackson’s Serialization Views](https://www.baeldung.com/jackson-json-view-annotation), which allows rendering only a subset of all fields in an `Object`. To use it with `@ResponseBody` or `ResponseEntity` controller methods, you can use Jackson’s `@JsonView` annotation to activate a serialization view class, as the following example shows:

**Java.**

``` java
@RestController
public class UserController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView.class)
    public User getUser() {
        return new User("eric", "7!jd#h23");
    }
}

public class User {

    public interface WithoutPasswordView {};
    public interface WithPasswordView extends WithoutPasswordView {};

    private String username;
    private String password;

    public User() {
    }

    public User(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @JsonView(WithoutPasswordView.class)
    public String getUsername() {
        return this.username;
    }

    @JsonView(WithPasswordView.class)
    public String getPassword() {
        return this.password;
    }
}
```

**Kotlin.**

``` kotlin
@RestController
class UserController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView::class)
    fun getUser(): User {
        return User("eric", "7!jd#h23")
    }
}

class User(
        @JsonView(WithoutPasswordView::class) val username: String,
        @JsonView(WithPasswordView::class) val password: String
) {
    interface WithoutPasswordView
    interface WithPasswordView : WithoutPasswordView
}
```

> **Note**
> 
> `@JsonView` allows an array of view classes but you can only specify only one per controller method. Use a composite interface if you need to activate multiple views.

## `Model`

[Web MVC](web.xml#mvc-ann-modelattrib-methods)

You can use the `@ModelAttribute` annotation:

  - On a [method argument](#webflux-ann-modelattrib-method-args) in `@RequestMapping` methods to create or access an Object from the model and to bind it to the request through a `WebDataBinder`.

  - As a method-level annotation in `@Controller` or `@ControllerAdvice` classes, helping to initialize the model prior to any `@RequestMapping` method invocation.

  - On a `@RequestMapping` method to mark its return value as a model attribute.

This section discusses `@ModelAttribute` methods, or the second item from the preceding list. A controller can have any number of `@ModelAttribute` methods. All such methods are invoked before `@RequestMapping` methods in the same controller. A `@ModelAttribute` method can also be shared across controllers through `@ControllerAdvice`. See the section on [section\_title](#webflux-ann-controller-advice) for more details.

`@ModelAttribute` methods have flexible method signatures. They support many of the same arguments as `@RequestMapping` methods (except for `@ModelAttribute` itself and anything related to the request body).

The following example uses a `@ModelAttribute` method:

**Java.**

``` java
@ModelAttribute
public void populateModel(@RequestParam String number, Model model) {
    model.addAttribute(accountRepository.findAccount(number));
    // add more ...
}
```

**Kotlin.**

``` kotlin
@ModelAttribute
fun populateModel(@RequestParam number: String, model: Model) {
    model.addAttribute(accountRepository.findAccount(number))
    // add more ...
}
```

The following example adds one attribute only:

**Java.**

``` java
@ModelAttribute
public Account addAccount(@RequestParam String number) {
    return accountRepository.findAccount(number);
}
```

**Kotlin.**

``` kotlin
@ModelAttribute
fun addAccount(@RequestParam number: String): Account {
    return accountRepository.findAccount(number);
}
```

> **Note**
> 
> When a name is not explicitly specified, a default name is chosen based on the type, as explained in the javadoc for {api-spring-framework}/core/Conventions.html\[`Conventions`\]. You can always assign an explicit name by using the overloaded `addAttribute` method or through the name attribute on `@ModelAttribute` (for a return value).

Spring WebFlux, unlike Spring MVC, explicitly supports reactive types in the model (for example, `Mono<Account>` or `io.reactivex.Single<Account>`). Such asynchronous model attributes can be transparently resolved (and the model updated) to their actual values at the time of `@RequestMapping` invocation, provided a `@ModelAttribute` argument is declared without a wrapper, as the following example shows:

**Java.**

``` java
@ModelAttribute
public void addAccount(@RequestParam String number) {
    Mono<Account> accountMono = accountRepository.findAccount(number);
    model.addAttribute("account", accountMono);
}

@PostMapping("/accounts")
public String handle(@ModelAttribute Account account, BindingResult errors) {
    // ...
}
```

**Kotlin.**

``` kotlin
import org.springframework.ui.set

@ModelAttribute
fun addAccount(@RequestParam number: String) {
    val accountMono: Mono<Account> = accountRepository.findAccount(number)
    model["account"] = accountMono
}

@PostMapping("/accounts")
fun handle(@ModelAttribute account: Account, errors: BindingResult): String {
    // ...
}
```

In addition, any model attributes that have a reactive type wrapper are resolved to their actual values (and the model updated) just prior to view rendering.

You can also use `@ModelAttribute` as a method-level annotation on `@RequestMapping` methods, in which case the return value of the `@RequestMapping` method is interpreted as a model attribute. This is typically not required, as it is the default behavior in HTML controllers, unless the return value is a `String` that would otherwise be interpreted as a view name. `@ModelAttribute` can also help to customize the model attribute name, as the following example shows:

**Java.**

``` java
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount")
public Account handle() {
    // ...
    return account;
}
```

**Kotlin.**

``` kotlin
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount")
fun handle(): Account {
    // ...
    return account
}
```

## `DataBinder`

[Web MVC](web.xml#mvc-ann-initbinder)

`@Controller` or `@ControllerAdvice` classes can have `@InitBinder` methods, to initialize instances of `WebDataBinder`. Those, in turn, are used to:

  - Bind request parameters (that is, form data or query) to a model object.

  - Convert `String`-based request values (such as request parameters, path variables, headers, cookies, and others) to the target type of controller method arguments.

  - Format model object values as `String` values when rendering HTML forms.

`@InitBinder` methods can register controller-specific `java.beans.PropertyEditor` or Spring `Converter` and `Formatter` components. In addition, you can use the [WebFlux Java configuration](#webflux-config-conversion) to register `Converter` and `Formatter` types in a globally shared `FormattingConversionService`.

`@InitBinder` methods support many of the same arguments that `@RequestMapping` methods do, except for `@ModelAttribute` (command object) arguments. Typically, they are declared with a `WebDataBinder` argument, for registrations, and a `void` return value. The following example uses the `@InitBinder` annotation:

**Java.**

``` java
@Controller
public class FormController {

    @InitBinder 
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }

    // ...
}
```

  - Using the `@InitBinder` annotation.

**Kotlin.**

``` kotlin
@Controller
class FormController {

    @InitBinder 
    fun initBinder(binder: WebDataBinder) {
        val dateFormat = SimpleDateFormat("yyyy-MM-dd")
        dateFormat.isLenient = false
        binder.registerCustomEditor(Date::class.java, CustomDateEditor(dateFormat, false))
    }

    // ...
}
```

Alternatively, when using a `Formatter`-based setup through a shared `FormattingConversionService`, you could re-use the same approach and register controller-specific `Formatter` instances, as the following example shows:

**Java.**

``` java
@Controller
public class FormController {

    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd")); 
    }

    // ...
}
```

  - Adding a custom formatter (a `DateFormatter`, in this case).

**Kotlin.**

``` kotlin
@Controller
class FormController {

    @InitBinder
    fun initBinder(binder: WebDataBinder) {
        binder.addCustomFormatter(DateFormatter("yyyy-MM-dd")) 
    }

    // ...
}
```

  - Adding a custom formatter (a `DateFormatter`, in this case).

## Managing Exceptions

[Web MVC](web.xml#mvc-ann-exceptionhandler)

`@Controller` and [@ControllerAdvice](#webflux-ann-controller-advice) classes can have `@ExceptionHandler` methods to handle exceptions from controller methods. The following example includes such a handler method:

**Java.**

``` java
@Controller
public class SimpleController {

    // ...

    @ExceptionHandler 
    public ResponseEntity<String> handle(IOException ex) {
        // ...
    }
}
```

  - Declaring an `@ExceptionHandler`.

**Kotlin.**

``` kotlin
@Controller
class SimpleController {

    // ...

    @ExceptionHandler 
    fun handle(ex: IOException): ResponseEntity<String> {
        // ...
    }
}
```

  - Declaring an `@ExceptionHandler`.

The exception can match against a top-level exception being propagated (that is, a direct `IOException` being thrown) or against the immediate cause within a top-level wrapper exception (for example, an `IOException` wrapped inside an `IllegalStateException`).

For matching exception types, preferably declare the target exception as a method argument, as shown in the preceding example. Alternatively, the annotation declaration can narrow the exception types to match. We generally recommend being as specific as possible in the argument signature and to declare your primary root exception mappings on a `@ControllerAdvice` prioritized with a corresponding order. See [the MVC section](web.xml#mvc-ann-exceptionhandler) for details.

> **Note**
> 
> An `@ExceptionHandler` method in WebFlux supports the same method arguments and return values as a `@RequestMapping` method, with the exception of request body- and `@ModelAttribute`-related method arguments.

Support for `@ExceptionHandler` methods in Spring WebFlux is provided by the `HandlerAdapter` for `@RequestMapping` methods. See [section\_title](#webflux-dispatcher-handler) for more detail.

### REST API exceptions

[Web MVC](web.xml#mvc-ann-rest-exceptions)

A common requirement for REST services is to include error details in the body of the response. The Spring Framework does not automatically do so, because the representation of error details in the response body is application-specific. However, a `@RestController` can use `@ExceptionHandler` methods with a `ResponseEntity` return value to set the status and the body of the response. Such methods can also be declared in `@ControllerAdvice` classes to apply them globally.

> **Note**
> 
> Note that Spring WebFlux does not have an equivalent for the Spring MVC `ResponseEntityExceptionHandler`, because WebFlux raises only `ResponseStatusException` (or subclasses thereof), and those do not need to be translated to an HTTP status code.

## Controller Advice

[Web MVC](web.xml#mvc-ann-controller-advice)

Typically, the `@ExceptionHandler`, `@InitBinder`, and `@ModelAttribute` methods apply within the `@Controller` class (or class hierarchy) in which they are declared. If you want such methods to apply more globally (across controllers), you can declare them in a class annotated with `@ControllerAdvice` or `@RestControllerAdvice`.

`@ControllerAdvice` is annotated with `@Component`, which means that such classes can be registered as Spring beans through [component scanning](core.xml#beans-java-instantiating-container-scan). `@RestControllerAdvice` is a composed annotation that is annotated with both `@ControllerAdvice` and `@ResponseBody`, which essentially means `@ExceptionHandler` methods are rendered to the response body through message conversion (versus view resolution or template rendering).

On startup, the infrastructure classes for `@RequestMapping` and `@ExceptionHandler` methods detect Spring beans annotated with `@ControllerAdvice` and then apply their methods at runtime. Global `@ExceptionHandler` methods (from a `@ControllerAdvice`) are applied *after* local ones (from the `@Controller`). By contrast, global `@ModelAttribute` and `@InitBinder` methods are applied *before* local ones.

By default, `@ControllerAdvice` methods apply to every request (that is, all controllers), but you can narrow that down to a subset of controllers by using attributes on the annotation, as the following example shows:

**Java.**

``` java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
```

**Kotlin.**

``` kotlin
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = [RestController::class])
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = [ControllerInterface::class, AbstractController::class])
public class ExampleAdvice3 {}
```

The selectors in the preceding example are evaluated at runtime and may negatively impact performance if used extensively. See the {api-spring-framework}/web/bind/annotation/ControllerAdvice.html\[`@ControllerAdvice`\] javadoc for more details.

# Functional Endpoints

[Web MVC](web.xml#webmvc-fn)

Spring WebFlux includes WebFlux.fn, a lightweight functional programming model in which functions are used to route and handle requests and contracts are designed for immutability. It is an alternative to the annotation-based programming model but otherwise runs on the same [web-reactive.xml](web-reactive.xml#webflux-reactive-spring-web) foundation.

## Overview

[Web MVC](web.xml#webmvc-fn-overview)

In WebFlux.fn, an HTTP request is handled with a `HandlerFunction`: a function that takes `ServerRequest` and returns a delayed `ServerResponse` (i.e. `Mono<ServerResponse>`). Both the request and the response object have immutable contracts that offer JDK 8-friendly access to the HTTP request and response. `HandlerFunction` is the equivalent of the body of a `@RequestMapping` method in the annotation-based programming model.

Incoming requests are routed to a handler function with a `RouterFunction`: a function that takes `ServerRequest` and returns a delayed `HandlerFunction` (i.e. `Mono<HandlerFunction>`). When the router function matches, a handler function is returned; otherwise an empty Mono. `RouterFunction` is the equivalent of a `@RequestMapping` annotation, but with the major difference that router functions provide not just data, but also behavior.

`RouterFunctions.route()` provides a router builder that facilitates the creation of routers, as the following example shows:

**Java.**

``` java
import static org.springframework.http.MediaType.APPLICATION_JSON;
import static org.springframework.web.reactive.function.server.RequestPredicates.*;
import static org.springframework.web.reactive.function.server.RouterFunctions.route;

PersonRepository repository = ...
PersonHandler handler = new PersonHandler(repository);

RouterFunction<ServerResponse> route = route()
    .GET("/person/{id}", accept(APPLICATION_JSON), handler::getPerson)
    .GET("/person", accept(APPLICATION_JSON), handler::listPeople)
    .POST("/person", handler::createPerson)
    .build();


public class PersonHandler {

    // ...

    public Mono<ServerResponse> listPeople(ServerRequest request) {
        // ...
    }

    public Mono<ServerResponse> createPerson(ServerRequest request) {
        // ...
    }

    public Mono<ServerResponse> getPerson(ServerRequest request) {
        // ...
    }
}
```

**Kotlin.**

``` kotlin
val repository: PersonRepository = ...
val handler = PersonHandler(repository)

val route = coRouter { 
    accept(APPLICATION_JSON).nest {
        GET("/person/{id}", handler::getPerson)
        GET("/person", handler::listPeople)
    }
    POST("/person", handler::createPerson)
}


class PersonHandler(private val repository: PersonRepository) {

    // ...

    suspend fun listPeople(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun createPerson(request: ServerRequest): ServerResponse {
        // ...
    }

    suspend fun getPerson(request: ServerRequest): ServerResponse {
        // ...
    }
}
```

  - Create router using Coroutines router DSL, a Reactive alternative is also available via `router { }`.

One way to run a `RouterFunction` is to turn it into an `HttpHandler` and install it through one of the built-in [server adapters](web-reactive.xml#webflux-httphandler):

  - `RouterFunctions.toHttpHandler(RouterFunction)`

  - `RouterFunctions.toHttpHandler(RouterFunction, HandlerStrategies)`

Most applications can run through the WebFlux Java configuration, see [section\_title](#webflux-fn-running).

## HandlerFunction

[Web MVC](web.xml#webmvc-fn-handler-functions)

`ServerRequest` and `ServerResponse` are immutable interfaces that offer JDK 8-friendly access to the HTTP request and response. Both request and response provide [Reactive Streams](https://www.reactive-streams.org) back pressure against the body streams. The request body is represented with a Reactor `Flux` or `Mono`. The response body is represented with any Reactive Streams `Publisher`, including `Flux` and `Mono`. For more on that, see [Reactive Libraries](web-reactive.xml#webflux-reactive-libraries).

### ServerRequest

`ServerRequest` provides access to the HTTP method, URI, headers, and query parameters, while access to the body is provided through the `body` methods.

The following example extracts the request body to a `Mono<String>`:

**Java.**

``` java
Mono<String> string = request.bodyToMono(String.class);
```

**Kotlin.**

``` kotlin
val string = request.awaitBody<String>()
```

The following example extracts the body to a `Flux<Person>` (or a `Flow<Person>` in Kotlin), where `Person` objects are decoded from someserialized form, such as JSON or XML:

**Java.**

``` java
Flux<Person> people = request.bodyToFlux(Person.class);
```

**Kotlin.**

``` kotlin
val people = request.bodyToFlow<Person>()
```

The preceding examples are shortcuts that use the more general `ServerRequest.body(BodyExtractor)`, which accepts the `BodyExtractor` functional strategy interface. The utility class `BodyExtractors` provides access to a number of instances. For example, the preceding examples can also be written as follows:

**Java.**

``` java
Mono<String> string = request.body(BodyExtractors.toMono(String.class));
Flux<Person> people = request.body(BodyExtractors.toFlux(Person.class));
```

**Kotlin.**

``` kotlin
   val string = request.body(BodyExtractors.toMono(String::class.java)).awaitSingle()
    val people = request.body(BodyExtractors.toFlux(Person::class.java)).asFlow()
```

The following example shows how to access form data:

**Java.**

``` java
Mono<MultiValueMap<String, String> map = request.formData();
```

**Kotlin.**

``` kotlin
val map = request.awaitFormData()
```

The following example shows how to access multipart data as a map:

**Java.**

``` java
Mono<MultiValueMap<String, Part> map = request.multipartData();
```

**Kotlin.**

``` kotlin
val map = request.awaitMultipartData()
```

The following example shows how to access multiparts, one at a time, in streaming fashion:

**Java.**

``` java
Flux<Part> parts = request.body(BodyExtractors.toParts());
```

**Kotlin.**

``` kotlin
val parts = request.body(BodyExtractors.toParts()).asFlow()
```

### ServerResponse

`ServerResponse` provides access to the HTTP response and, since it is immutable, you can use a `build` method to create it. You can use the builder to set the response status, to add response headers, or to provide a body. The following example creates a 200 (OK) response with JSON content:

**Java.**

``` java
Mono<Person> person = ...
ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).body(person, Person.class);
```

**Kotlin.**

``` kotlin
val person: Person = ...
ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).bodyValue(person)
```

The following example shows how to build a 201 (CREATED) response with a `Location` header and no body:

**Java.**

``` java
URI location = ...
ServerResponse.created(location).build();
```

**Kotlin.**

``` kotlin
val location: URI = ...
ServerResponse.created(location).build()
```

Depending on the codec used, it is possible to pass hint parameters to customize how the body is serialized or deserialized. For example, to specify a [Jackson JSON view](https://www.baeldung.com/jackson-json-view-annotation):

**Java.**

``` java
ServerResponse.ok().hint(Jackson2CodecSupport.JSON_VIEW_HINT, MyJacksonView.class).body(...);
```

**Kotlin.**

``` kotlin
ServerResponse.ok().hint(Jackson2CodecSupport.JSON_VIEW_HINT, MyJacksonView::class.java).body(...)
```

### Handler Classes

We can write a handler function as a lambda, as the following example shows:

**Java.**

``` java
HandlerFunction<ServerResponse> helloWorld =
  request -> ServerResponse.ok().bodyValue("Hello World");
```

**Kotlin.**

``` kotlin
val helloWorld = HandlerFunction<ServerResponse> { ServerResponse.ok().bodyValue("Hello World") }
```

That is convenient, but in an application we need multiple functions, and multiple inline lambda’s can get messy. Therefore, it is useful to group related handler functions together into a handler class, which has a similar role as `@Controller` in an annotation-based application. For example, the following class exposes a reactive `Person` repository:

**Java.**

``` java
import static org.springframework.http.MediaType.APPLICATION_JSON;
import static org.springframework.web.reactive.function.server.ServerResponse.ok;

public class PersonHandler {

    private final PersonRepository repository;

    public PersonHandler(PersonRepository repository) {
        this.repository = repository;
    }

    public Mono<ServerResponse> listPeople(ServerRequest request) { 
        Flux<Person> people = repository.allPeople();
        return ok().contentType(APPLICATION_JSON).body(people, Person.class);
    }

    public Mono<ServerResponse> createPerson(ServerRequest request) { 
        Mono<Person> person = request.bodyToMono(Person.class);
        return ok().build(repository.savePerson(person));
    }

    public Mono<ServerResponse> getPerson(ServerRequest request) { 
        int personId = Integer.valueOf(request.pathVariable("id"));
        return repository.getPerson(personId)
            .flatMap(person -> ok().contentType(APPLICATION_JSON).bodyValue(person))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
}
```

  - `listPeople` is a handler function that returns all `Person` objects found in the repository as JSON.

  - `createPerson` is a handler function that stores a new `Person` contained in the request body. Note that `PersonRepository.savePerson(Person)` returns `Mono<Void>`: an empty `Mono` that emits a completion signal when the person has been read from the request and stored. So we use the `build(Publisher<Void>)` method to send a response when that completion signal is received (that is, when the `Person` has been saved).

  - `getPerson` is a handler function that returns a single person, identified by the `id` path variable. We retrieve that `Person` from the repository and create a JSON response, if it is found. If it is not found, we use `switchIfEmpty(Mono<T>)` to return a 404 Not Found response.

**Kotlin.**

``` kotlin
class PersonHandler(private val repository: PersonRepository) {

    suspend fun listPeople(request: ServerRequest): ServerResponse { 
        val people: Flow<Person> = repository.allPeople()
        return ok().contentType(APPLICATION_JSON).bodyAndAwait(people);
    }

    suspend fun createPerson(request: ServerRequest): ServerResponse { 
        val person = request.awaitBody<Person>()
        repository.savePerson(person)
        return ok().buildAndAwait()
    }

    suspend fun getPerson(request: ServerRequest): ServerResponse { 
        val personId = request.pathVariable("id").toInt()
        return repository.getPerson(personId)?.let { ok().contentType(APPLICATION_JSON).bodyValueAndAwait(it) }
                ?: ServerResponse.notFound().buildAndAwait()

    }
}
```

  - `listPeople` is a handler function that returns all `Person` objects found in the repository as JSON.

  - `createPerson` is a handler function that stores a new `Person` contained in the request body. Note that `PersonRepository.savePerson(Person)` is a suspending function with no return type.

  - `getPerson` is a handler function that returns a single person, identified by the `id` path variable. We retrieve that `Person` from the repository and create a JSON response, if it is found. If it is not found, we return a 404 Not Found response.

### Validation

A functional endpoint can use Spring’s [validation facilities](core.xml#validation) to apply validation to the request body. For example, given a custom Spring [Validator](core.xml#validation) implementation for a `Person`:

**Java.**

``` java
public class PersonHandler {

    private final Validator validator = new PersonValidator(); 

    // ...

    public Mono<ServerResponse> createPerson(ServerRequest request) {
        Mono<Person> person = request.bodyToMono(Person.class).doOnNext(this::validate); 
        return ok().build(repository.savePerson(person));
    }

    private void validate(Person person) {
        Errors errors = new BeanPropertyBindingResult(person, "person");
        validator.validate(person, errors);
        if (errors.hasErrors()) {
            throw new ServerWebInputException(errors.toString()); 
        }
    }
}
```

  - Create `Validator` instance.

  - Apply validation.

  - Raise exception for a 400 response.

**Kotlin.**

``` kotlin
class PersonHandler(private val repository: PersonRepository) {

    private val validator = PersonValidator() 

    // ...

    suspend fun createPerson(request: ServerRequest): ServerResponse {
        val person = request.awaitBody<Person>()
        validate(person) 
        repository.savePerson(person)
        return ok().buildAndAwait()
    }

    private fun validate(person: Person) {
        val errors: Errors = BeanPropertyBindingResult(person, "person");
        validator.validate(person, errors);
        if (errors.hasErrors()) {
            throw ServerWebInputException(errors.toString()) 
        }
    }
}
```

  - Create `Validator` instance.

  - Apply validation.

  - Raise exception for a 400 response.

Handlers can also use the standard bean validation API (JSR-303) by creating and injecting a global `Validator` instance based on `LocalValidatorFactoryBean`. See [Spring Validation](core.xml#validation-beanvalidation).

## `RouterFunction`

[Web MVC](web.xml#webmvc-fn-router-functions)

Router functions are used to route the requests to the corresponding `HandlerFunction`. Typically, you do not write router functions yourself, but rather use a method on the `RouterFunctions` utility class to create one. `RouterFunctions.route()` (no parameters) provides you with a fluent builder for creating a router function, whereas `RouterFunctions.route(RequestPredicate, HandlerFunction)` offers a direct way to create a router.

Generally, it is recommended to use the `route()` builder, as it provides convenient short-cuts for typical mapping scenarios without requiring hard-to-discover static imports. For instance, the router function builder offers the method `GET(String, HandlerFunction)` to create a mapping for GET requests; and `POST(String, HandlerFunction)` for POSTs.

Besides HTTP method-based mapping, the route builder offers a way to introduce additional predicates when mapping to requests. For each HTTP method there is an overloaded variant that takes a `RequestPredicate` as a parameter, though which additional constraints can be expressed.

### Predicates

You can write your own `RequestPredicate`, but the `RequestPredicates` utility class offers commonly used implementations, based on the request path, HTTP method, content-type, and so on. The following example uses a request predicate to create a constraint based on the `Accept` header:

**Java.**

``` java
RouterFunction<ServerResponse> route = RouterFunctions.route()
    .GET("/hello-world", accept(MediaType.TEXT_PLAIN),
        request -> ServerResponse.ok().bodyValue("Hello World")).build();
```

**Kotlin.**

``` kotlin
val route = coRouter {
    GET("/hello-world", accept(TEXT_PLAIN)) {
           ServerResponse.ok().bodyValueAndAwait("Hello World")
       }
}
```

You can compose multiple request predicates together by using:

  - `RequestPredicate.and(RequestPredicate)` — both must match.

  - `RequestPredicate.or(RequestPredicate)` — either can match.

Many of the predicates from `RequestPredicates` are composed. For example, `RequestPredicates.GET(String)` is composed from `RequestPredicates.method(HttpMethod)` and `RequestPredicates.path(String)`. The example shown above also uses two request predicates, as the builder uses `RequestPredicates.GET` internally, and composes that with the `accept` predicate.

### Routes

Router functions are evaluated in order: if the first route does not match, the second is evaluated, and so on. Therefore, it makes sense to declare more specific routes before general ones. This is also important when registering router functions as Spring beans, as will be described later. Note that this behavior is different from the annotation-based programming model, where the "most specific" controller method is picked automatically.

When using the router function builder, all defined routes are composed into one `RouterFunction` that is returned from `build()`. There are also other ways to compose multiple router functions together:

  - `add(RouterFunction)` on the `RouterFunctions.route()` builder

  - `RouterFunction.and(RouterFunction)`

  - `RouterFunction.andRoute(RequestPredicate, HandlerFunction)` — shortcut for `RouterFunction.and()` with nested `RouterFunctions.route()`.

The following example shows the composition of four routes:

**Java.**

``` java
import static org.springframework.http.MediaType.APPLICATION_JSON;
import static org.springframework.web.reactive.function.server.RequestPredicates.*;

PersonRepository repository = ...
PersonHandler handler = new PersonHandler(repository);

RouterFunction<ServerResponse> otherRoute = ...

RouterFunction<ServerResponse> route = route()
    .GET("/person/{id}", accept(APPLICATION_JSON), handler::getPerson) 
    .GET("/person", accept(APPLICATION_JSON), handler::listPeople) 
    .POST("/person", handler::createPerson) 
    .add(otherRoute) 
    .build();
```

  - `GET /person/{id}` with an `Accept` header that matches JSON is routed to `PersonHandler.getPerson`

  - `GET /person` with an `Accept` header that matches JSON is routed to `PersonHandler.listPeople`

  - `POST /person` with no additional predicates is mapped to `PersonHandler.createPerson`, and

  - `otherRoute` is a router function that is created elsewhere, and added to the route built.

**Kotlin.**

``` kotlin
import org.springframework.http.MediaType.APPLICATION_JSON

val repository: PersonRepository = ...
val handler = PersonHandler(repository);

val otherRoute: RouterFunction<ServerResponse> = coRouter {  }

val route = coRouter {
    GET("/person/{id}", accept(APPLICATION_JSON), handler::getPerson) 
    GET("/person", accept(APPLICATION_JSON), handler::listPeople) 
    POST("/person", handler::createPerson) 
}.and(otherRoute) 
```

  - `GET /person/{id}` with an `Accept` header that matches JSON is routed to `PersonHandler.getPerson`

  - `GET /person` with an `Accept` header that matches JSON is routed to `PersonHandler.listPeople`

  - `POST /person` with no additional predicates is mapped to `PersonHandler.createPerson`, and

  - `otherRoute` is a router function that is created elsewhere, and added to the route built.

### Nested Routes

It is common for a group of router functions to have a shared predicate, for instance a shared path. In the example above, the shared predicate would be a path predicate that matches `/person`, used by three of the routes. When using annotations, you would remove this duplication by using a type-level `@RequestMapping` annotation that maps to `/person`. In WebFlux.fn, path predicates can be shared through the `path` method on the router function builder. For instance, the last few lines of the example above can be improved in the following way by using nested routes:

**Java.**

``` java
RouterFunction<ServerResponse> route = route()
    .path("/person", builder -> builder 
        .GET("/{id}", accept(APPLICATION_JSON), handler::getPerson)
        .GET(accept(APPLICATION_JSON), handler::listPeople)
        .POST("/person", handler::createPerson))
    .build();
```

  - Note that second parameter of `path` is a consumer that takes the a router builder.

**Kotlin.**

``` kotlin
val route = coRouter {
    "/person".nest {
        GET("/{id}", accept(APPLICATION_JSON), handler::getPerson)
        GET(accept(APPLICATION_JSON), handler::listPeople)
        POST("/person", handler::createPerson)
    }
}
```

Though path-based nesting is the most common, you can nest on any kind of predicate by using the `nest` method on the builder. The above still contains some duplication in the form of the shared `Accept`-header predicate. We can further improve by using the `nest` method together with `accept`:

**Java.**

``` java
RouterFunction<ServerResponse> route = route()
    .path("/person", b1 -> b1
        .nest(accept(APPLICATION_JSON), b2 -> b2
            .GET("/{id}", handler::getPerson)
            .GET(handler::listPeople))
        .POST("/person", handler::createPerson))
    .build();
```

**Kotlin.**

``` kotlin
val route = coRouter {
    "/person".nest {
        accept(APPLICATION_JSON).nest {
            GET("/{id}", handler::getPerson)
            GET(handler::listPeople)
            POST("/person", handler::createPerson)
        }
    }
}
```

## Running a Server

[Web MVC](web.xml#webmvc-fn-running)

How do you run a router function in an HTTP server? A simple option is to convert a router function to an `HttpHandler` by using one of the following:

  - `RouterFunctions.toHttpHandler(RouterFunction)`

  - `RouterFunctions.toHttpHandler(RouterFunction, HandlerStrategies)`

You can then use the returned `HttpHandler` with a number of server adapters by following [HttpHandler](web-reactive.xml#webflux-httphandler) for server-specific instructions.

A more typical option, also used by Spring Boot, is to run with a [`DispatcherHandler`](web-reactive.xml#webflux-dispatcher-handler)-based setup through the [web-reactive.xml](web-reactive.xml#webflux-config), which uses Spring configuration to declare the components required to process requests. The WebFlux Java configuration declares the following infrastructure components to support functional endpoints:

  - `RouterFunctionMapping`: Detects one or more `RouterFunction<?>` beans in the Spring configuration, [orders them](core.xml#beans-factory-ordered), combines them through `RouterFunction.andOther`, and routes requests to the resulting composed `RouterFunction`.

  - `HandlerFunctionAdapter`: Simple adapter that lets `DispatcherHandler` invoke a `HandlerFunction` that was mapped to a request.

  - `ServerResponseResultHandler`: Handles the result from the invocation of a `HandlerFunction` by invoking the `writeTo` method of the `ServerResponse`.

The preceding components let functional endpoints fit within the `DispatcherHandler` request processing lifecycle and also (potentially) run side by side with annotated controllers, if any are declared. It is also how functional endpoints are enabled by the Spring Boot WebFlux starter.

The following example shows a WebFlux Java configuration (see [DispatcherHandler](web-reactive.xml#webflux-dispatcher-handler) for how to run it):

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Bean
    public RouterFunction<?> routerFunctionA() {
        // ...
    }

    @Bean
    public RouterFunction<?> routerFunctionB() {
        // ...
    }

    // ...

    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        // configure message conversion...
    }

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        // configure CORS...
    }

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        // configure view resolution for HTML rendering...
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    @Bean
    fun routerFunctionA(): RouterFunction<*> {
        // ...
    }

    @Bean
    fun routerFunctionB(): RouterFunction<*> {
        // ...
    }

    // ...

    override fun configureHttpMessageCodecs(configurer: ServerCodecConfigurer) {
        // configure message conversion...
    }

    override fun addCorsMappings(registry: CorsRegistry) {
        // configure CORS...
    }

    override fun configureViewResolvers(registry: ViewResolverRegistry) {
        // configure view resolution for HTML rendering...
    }
}
```

## Filtering Handler Functions

[Web MVC](web.xml#webmvc-fn-handler-filter-function)

You can filter handler functions by using the `before`, `after`, or `filter` methods on the routing function builder. With annotations, you can achieve similar functionality by using `@ControllerAdvice`, a `ServletFilter`, or both. The filter will apply to all routes that are built by the builder. This means that filters defined in nested routes do not apply to "top-level" routes. For instance, consider the following example:

**Java.**

``` java
RouterFunction<ServerResponse> route = route()
    .path("/person", b1 -> b1
        .nest(accept(APPLICATION_JSON), b2 -> b2
            .GET("/{id}", handler::getPerson)
            .GET(handler::listPeople)
            .before(request -> ServerRequest.from(request) 
                .header("X-RequestHeader", "Value")
                .build()))
        .POST("/person", handler::createPerson))
    .after((request, response) -> logResponse(response)) 
    .build();
```

  - The `before` filter that adds a custom request header is only applied to the two GET routes.

  - The `after` filter that logs the response is applied to all routes, including the nested ones.

**Kotlin.**

``` kotlin
val route = router {
    "/person".nest {
        GET("/{id}", handler::getPerson)
        GET("", handler::listPeople)
        before { 
            ServerRequest.from(it)
                    .header("X-RequestHeader", "Value").build()
        }
        POST("/person", handler::createPerson)
        after { _, response -> 
            logResponse(response)
        }
    }
}
```

  - The `before` filter that adds a custom request header is only applied to the two GET routes.

  - The `after` filter that logs the response is applied to all routes, including the nested ones.

The `filter` method on the router builder takes a `HandlerFilterFunction`: a function that takes a `ServerRequest` and `HandlerFunction` and returns a `ServerResponse`. The handler function parameter represents the next element in the chain. This is typically the handler that is routed to, but it can also be another filter if multiple are applied.

Now we can add a simple security filter to our route, assuming that we have a `SecurityManager` that can determine whether a particular path is allowed. The following example shows how to do so:

**Java.**

``` java
SecurityManager securityManager = ...

RouterFunction<ServerResponse> route = route()
    .path("/person", b1 -> b1
        .nest(accept(APPLICATION_JSON), b2 -> b2
            .GET("/{id}", handler::getPerson)
            .GET(handler::listPeople))
        .POST("/person", handler::createPerson))
    .filter((request, next) -> {
        if (securityManager.allowAccessTo(request.path())) {
            return next.handle(request);
        }
        else {
            return ServerResponse.status(UNAUTHORIZED).build();
        }
    })
    .build();
```

**Kotlin.**

``` kotlin
val securityManager: SecurityManager = ...

val route = router {
        ("/person" and accept(APPLICATION_JSON)).nest {
            GET("/{id}", handler::getPerson)
            GET("", handler::listPeople)
            POST("/person", handler::createPerson)
            filter { request, next ->
                if (securityManager.allowAccessTo(request.path())) {
                    next(request)
                }
                else {
                    status(UNAUTHORIZED).build();
                }
            }
        }
    }
```

The preceding example demonstrates that invoking the `next.handle(ServerRequest)` is optional. We only let the handler function be run when access is allowed.

Besides using the `filter` method on the router function builder, it is possible to apply a filter to an existing router function via `RouterFunction.filter(HandlerFilterFunction)`.

> **Note**
> 
> CORS support for functional endpoints is provided through a dedicated [`CorsWebFilter`](#webflux-cors-webfilter).

# URI Links

[Web MVC](web.xml#mvc-uri-building)

This section describes various options available in the Spring Framework to prepare URIs.

本节介绍 Spring Framework 中可用于准备 URI 的各种选项。

## UriComponents

Spring MVC and Spring WebFlux

`UriComponentsBuilder` helps to build URI’s from URI templates with variables, as the following example shows:

`UriComponentsBuilder` 有助于从带有变量的 URI 模板构建 URI，如以下示例所示：

**Java.**

``` java
UriComponents uriComponents = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")  
        .queryParam("q", "{q}")  
        .encode() 
        .build(); 

URI uri = uriComponents.expand("Westin", "123").toUri();  
```

  - Static factory method with a URI template.

  - Add or replace URI components.

  - Request to have the URI template and URI variables encoded.

  - Build a `UriComponents`.

  - Expand variables and obtain the `URI`.

  - 带有 URI 模板的静态工厂方法。

  - 添加或替换 URI 组件。

  - 请求对 URI 模板和 URI 变量进行编码。

  - 构建一个 `UriComponents`。

  - 展开变量并获取`URI`。

**Kotlin.**

``` kotlin
val uriComponents = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")  
        .queryParam("q", "{q}")  
        .encode() 
        .build() 

val uri = uriComponents.expand("Westin", "123").toUri()  
```

  - Static factory method with a URI template.

  - Add or replace URI components.

  - Request to have the URI template and URI variables encoded.

  - Build a `UriComponents`.

  - Expand variables and obtain the `URI`.

  - 带有 URI 模板的静态工厂方法。

  - 添加或替换 URI 组件。

  - 请求对 URI 模板和 URI 变量进行编码。

  - 构建一个 `UriComponents`。

  - 展开变量并获取`URI`。

The preceding example can be consolidated into one chain and shortened with `buildAndExpand`, as the following example shows:

前面的示例可以合并为一个链并简化为使用 `buildAndExpand` ，如下例所示：

**Java.**

``` java
URI uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .encode()
        .buildAndExpand("Westin", "123")
        .toUri();
```

**Kotlin.**

``` kotlin
val uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .encode()
        .buildAndExpand("Westin", "123")
        .toUri()
```

You can shorten it further by going directly to a URI (which implies encoding), as the following example shows:

你可以通过直接转换为 URI（自动隐式编码）来进一步简化它，如以下示例所示：

**Java.**

``` java
URI uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
```

**Kotlin.**

``` kotlin
val uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123")
```

You can shorten it further still with a full URI template, as the following example shows:

你可以使用一个完整的URI模板进一步简化实现，如下示例：

**Java.**

``` java
URI uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}?q={q}")
        .build("Westin", "123");
```

**Kotlin.**

``` kotlin
val uri = UriComponentsBuilder
    .fromUriString("https://example.com/hotels/{hotel}?q={q}")
    .build("Westin", "123")
```

## UriBuilder

Spring MVC and Spring WebFlux

[`UriComponentsBuilder`](#web-uricomponents) implements `UriBuilder`. You can create a `UriBuilder`, in turn, with a `UriBuilderFactory`. Together, `UriBuilderFactory` and `UriBuilder` provide a pluggable mechanism to build URIs from URI templates, based on shared configuration, such as a base URL, encoding preferences, and other details.

You can configure `RestTemplate` and `WebClient` with a `UriBuilderFactory` to customize the preparation of URIs. `DefaultUriBuilderFactory` is a default implementation of `UriBuilderFactory` that uses `UriComponentsBuilder` internally and exposes shared configuration options.

The following example shows how to configure a `RestTemplate`:

[`UriComponentsBuilder`](#web-uricomponents) 实现了 `UriBuilder`。 你可以使用 `UriBuilderFactory`创建一个 `UriBuilder`。 `UriBuilderFactory` 和 `UriBuilder` 一起提供了一种可插拔机制，可以根据共享配置（例如基本 URL、编码首选项和其他信息）从 URI 模板构建 URI。

你可以使用 `UriBuilderFactory` 配置 `RestTemplate` 和 `WebClient` 来为定义 URI 做准备。 `DefaultUriBuilderFactory` 是 `UriBuilderFactory` 的默认实现，它在内部使用 `UriComponentsBuilder` 并暴露共享配置选项。

以下示例显示了如何配置 `RestTemplate`：

**Java.**

``` java
// import org.springframework.web.util.DefaultUriBuilderFactory.EncodingMode;

String baseUrl = "https://example.org";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl);
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VALUES);

RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);
```

**Kotlin.**

``` kotlin
// import org.springframework.web.util.DefaultUriBuilderFactory.EncodingMode

val baseUrl = "https://example.org"
val factory = DefaultUriBuilderFactory(baseUrl)
factory.encodingMode = EncodingMode.TEMPLATE_AND_VALUES

val restTemplate = RestTemplate()
restTemplate.uriTemplateHandler = factory
```

The following example configures a `WebClient`:

**Java.**

``` java
// import org.springframework.web.util.DefaultUriBuilderFactory.EncodingMode;

String baseUrl = "https://example.org";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl);
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VALUES);

WebClient client = WebClient.builder().uriBuilderFactory(factory).build();
```

**Kotlin.**

``` kotlin
// import org.springframework.web.util.DefaultUriBuilderFactory.EncodingMode

val baseUrl = "https://example.org"
val factory = DefaultUriBuilderFactory(baseUrl)
factory.encodingMode = EncodingMode.TEMPLATE_AND_VALUES

val client = WebClient.builder().uriBuilderFactory(factory).build()
```

In addition, you can also use `DefaultUriBuilderFactory` directly. It is similar to using `UriComponentsBuilder` but, instead of static factory methods, it is an actual instance that holds configuration and preferences, as the following example shows:

另外，你也可以直接使用 DefaultUriBuilderFactory 。 它类似于使用“UriComponentsBuilder”，但不是使用静态工厂方法，而是一个包含配置和首选项的实际实例，如以下示例所示：

**Java.**

``` java
String baseUrl = "https://example.com";
DefaultUriBuilderFactory uriBuilderFactory = new DefaultUriBuilderFactory(baseUrl);

URI uri = uriBuilderFactory.uriString("/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
```

**Kotlin.**

``` kotlin
val baseUrl = "https://example.com"
val uriBuilderFactory = DefaultUriBuilderFactory(baseUrl)

val uri = uriBuilderFactory.uriString("/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123")
```

## URI Encoding

Spring MVC and Spring WebFlux

`UriComponentsBuilder` exposes encoding options at two levels:

  - {api-spring-framework}/web/util/UriComponentsBuilder.html\#encode--\[UriComponentsBuilder\#encode()\]: Pre-encodes the URI template first and then strictly encodes URI variables when expanded.

  - {api-spring-framework}/web/util/UriComponents.html\#encode--\[UriComponents\#encode()\]: Encodes URI components *after* URI variables are expanded.

  - [UriComponentsBuilder#encode()](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/web/util/UriComponentsBuilder.html#encode--): 首先对 URI 模板进行预编码，然后在展开时对 URI 变量进行严格编码。

  - [UriComponents#encode()](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/web/util/UriComponents.html#encode--): 在扩展 URI 变量之后对 URI 组件进行编码。

Both options replace non-ASCII and illegal characters with escaped octets. However, the first option also replaces characters with reserved meaning that appear in URI variables.

> **Tip**
> 
> Consider ";", which is legal in a path but has reserved meaning. The first option replaces ";" with "%3B" in URI variables but not in the URI template. By contrast, the second option never replaces ";", since it is a legal character in a path.

For most cases, the first option is likely to give the expected result, because it treats URI variables as opaque data to be fully encoded, while the second option is useful if URI variables do intentionally contain reserved characters. The second option is also useful when not expanding URI variables at all since that will also encode anything that incidentally looks like a URI variable.

The following example uses the first option:

这两个选项都用转义的八位字节替换非 ASCII 和非法字符。 但是，第一个选项也会替换出现在 URI 变量中的具有保留含义的字符。

> **提示**
>
> 考虑“;”，它在路径中是合法的，但是是一个保留字符。 第一个选项使用“%3B”替换在URI变量中的“;” ，但不会替换 URI 模板。 相比之下，第二个选项永远不会替换“;”，因为它是路径中的合法字符。

在大多数情况下，第一个选项可能会给出预期的结果，因为它将 URI 变量视为要完全编码的不透明数据，而如果 URI 变量有意包含保留字符，则第二个选项很有用。 当根本不扩展 URI 变量时，第二个选项也很有用，因为第一个选项也会对任何看起来像 URI 变量的东西进行编码。

以下示例使用第一个选项：

**Java.**

``` java
URI uri = UriComponentsBuilder.fromPath("/hotel list/{city}")
        .queryParam("q", "{q}")
        .encode()
        .buildAndExpand("New York", "foo+bar")
        .toUri();

// Result is "/hotel%20list/New%20York?q=foo%2Bbar"
```

**Kotlin.**

``` kotlin
val uri = UriComponentsBuilder.fromPath("/hotel list/{city}")
        .queryParam("q", "{q}")
        .encode()
        .buildAndExpand("New York", "foo+bar")
        .toUri()

// Result is "/hotel%20list/New%20York?q=foo%2Bbar"
```

You can shorten the preceding example by going directly to the URI (which implies encoding), as the following example shows:

你可以通过直接转换为 URI（自动隐式编码）来进一步简化它，如以下示例所示：

**Java.**

``` java
URI uri = UriComponentsBuilder.fromPath("/hotel list/{city}")
        .queryParam("q", "{q}")
        .build("New York", "foo+bar");
```

**Kotlin.**

``` kotlin
val uri = UriComponentsBuilder.fromPath("/hotel list/{city}")
        .queryParam("q", "{q}")
        .build("New York", "foo+bar")
```

You can shorten it further still with a full URI template, as the following example shows:

你可以使用一个完整的URI模板进一步简化实现，如下示例：

**Java.**

``` java
URI uri = UriComponentsBuilder.fromUriString("/hotel list/{city}?q={q}")
        .build("New York", "foo+bar");
```

**Kotlin.**

``` kotlin
val uri = UriComponentsBuilder.fromUriString("/hotel list/{city}?q={q}")
        .build("New York", "foo+bar")
```

The `WebClient` and the `RestTemplate` expand and encode URI templates internally through the `UriBuilderFactory` strategy. Both can be configured with a custom strategy. as the following example shows:

`WebClient` 和 `RestTemplate` 在内部通过 `UriBuilderFactory` 策略扩展和编码 URI 模板。 两者都可以使用自定义策略进行配置。 如以下示例所示：

**Java.**

``` java
String baseUrl = "https://example.com";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl)
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VALUES);

// Customize the RestTemplate..
RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);

// Customize the WebClient..
WebClient client = WebClient.builder().uriBuilderFactory(factory).build();
```

**Kotlin.**

``` kotlin
val baseUrl = "https://example.com"
val factory = DefaultUriBuilderFactory(baseUrl).apply {
    encodingMode = EncodingMode.TEMPLATE_AND_VALUES
}

// Customize the RestTemplate..
val restTemplate = RestTemplate().apply {
    uriTemplateHandler = factory
}

// Customize the WebClient..
val client = WebClient.builder().uriBuilderFactory(factory).build()
```

The `DefaultUriBuilderFactory` implementation uses `UriComponentsBuilder` internally to expand and encode URI templates. As a factory, it provides a single place to configure the approach to encoding, based on one of the below encoding modes:

  - `TEMPLATE_AND_VALUES`: Uses `UriComponentsBuilder#encode()`, corresponding to the first option in the earlier list, to pre-encode the URI template and strictly encode URI variables when expanded.

  - `VALUES_ONLY`: Does not encode the URI template and, instead, applies strict encoding to URI variables through `UriUtils#encodeUriVariables` prior to expanding them into the template.

  - `URI_COMPONENT`: Uses `UriComponents#encode()`, corresponding to the second option in the earlier list, to encode URI component value *after* URI variables are expanded.

  - `NONE`: No encoding is applied.

The `RestTemplate` is set to `EncodingMode.URI_COMPONENT` for historic reasons and for backwards compatibility. The `WebClient` relies on the default value in `DefaultUriBuilderFactory`, which was changed from `EncodingMode.URI_COMPONENT` in 5.0.x to `EncodingMode.TEMPLATE_AND_VALUES` in 5.1.

`DefaultUriBuilderFactory` 实现在内部使用 `UriComponentsBuilder` 来扩展和编码 URI 模板。作为一个工厂，它提供了一个单一的地方来配置编码方法，基于以下编码模式之一：

  - `TEMPLATE_AND_VALUES`：使用 `UriComponentsBuilder#encode()`，对应于前面列表中的第一个选项，对 URI 模板进行预编码，并在扩展时对 URI 变量进行严格编码。

  - `VALUES_ONLY`：不对 URI 模板进行编码，而是在将 URI 变量扩展到模板之前通过 `UriUtils#encodeUriVariables` 对 URI 变量进行严格编码。

  - `URI_COMPONENT`：使用 `UriComponents#encode()`，对应于前面列表中的第二个选项，在扩展 URI 变量之后对 URI 组件值进行编码。

  - `NONE`：不应用编码。

出于历史原因和向后兼容性，将 `RestTemplate` 设置为 `EncodingMode.URI_COMPONENT`。 `WebClient` 依赖于 `DefaultUriBuilderFactory` 中的默认值，该值从 5.0.x 中的 `EncodingMode.URI_COMPONENT` 更改为 5.1 中的 `EncodingMode.TEMPLATE_AND_VALUES`。

# CORS

[Web MVC](web.xml#mvc-cors)

Spring WebFlux lets you handle CORS (Cross-Origin Resource Sharing). This section describes how to do so.

Spring WebFlux 可让您处理 CORS（跨站资源共享）。 本节介绍如何执行此操作。

## Introduction

## 介绍

[Web MVC](web.xml#mvc-cors-intro)

For security reasons, browsers prohibit AJAX calls to resources outside the current origin. For example, you could have your bank account in one tab and evil.com in another. Scripts from evil.com should not be able to make AJAX requests to your bank API with your credentials — for example, withdrawing money from your account\!

Cross-Origin Resource Sharing (CORS) is a [W3C specification](https://www.w3.org/TR/cors/) implemented by [most browsers](https://caniuse.com/#feat=cors) that lets you specify what kind of cross-domain requests are authorized, rather than using less secure and less powerful workarounds based on IFRAME or JSONP.

出于安全原因，浏览器禁止对当前来源之外的资源进行 AJAX 调用。 例如，你可以在一个页面中登录你的银行帐户，而在另一个页面中访问 evil.com。 来自 evil.com 的脚本不应能够使用你的凭据向你的银行 API 发出 AJAX 请求 — 例如，从你的帐户中取款\！

跨站资源共享 (CORS) 是由 [大多数浏览器](https://caniuse.com/#feat=cors) 实现的 [W3C 规范](https://www.w3.org/TR/cors/) 这让你可以指定授权的跨域请求类型，而不是使用基于 IFRAME 或 JSONP 的安全性较低且功能较弱的解决方法。

## Processing

## 处理

[Web MVC](web.xml#mvc-cors-processing)

The CORS specification distinguishes between preflight, simple, and actual requests. To learn how CORS works, you can read [this article](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), among many others, or see the specification for more details.

Spring WebFlux `HandlerMapping` implementations provide built-in support for CORS. After successfully mapping a request to a handler, a `HandlerMapping` checks the CORS configuration for the given request and handler and takes further actions. Preflight requests are handled directly, while simple and actual CORS requests are intercepted, validated, and have the required CORS response headers set.

In order to enable cross-origin requests (that is, the `Origin` header is present and differs from the host of the request), you need to have some explicitly declared CORS configuration. If no matching CORS configuration is found, preflight requests are rejected. No CORS headers are added to the responses of simple and actual CORS requests and, consequently, browsers reject them.

Each `HandlerMapping` can be {api-spring-framework}/web/reactive/handler/AbstractHandlerMapping.html\#setCorsConfigurations-java.util.Map-\[configured\] individually with URL pattern-based `CorsConfiguration` mappings. In most cases, applications use the WebFlux Java configuration to declare such mappings, which results in a single, global map passed to all `HandlerMapping` implementations.

You can combine global CORS configuration at the `HandlerMapping` level with more fine-grained, handler-level CORS configuration. For example, annotated controllers can use class- or method-level `@CrossOrigin` annotations (other handlers can implement `CorsConfigurationSource`).

The rules for combining global and local configuration are generally additive — for example, all global and all local origins. For those attributes where only a single value can be accepted, such as `allowCredentials` and `maxAge`, the local overrides the global value. See {api-spring-framework}/web/cors/CorsConfiguration.html\#combine-org.springframework.web.cors.CorsConfiguration-\[`CorsConfiguration#combine(CorsConfiguration)`\] for more details.

CORS 规范区分了预检请求、简单请求和实际请求。要了解 CORS 的工作原理，你可以阅读 [这篇文章](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) 等，或查看规范以获取更多详细信息。

Spring WebFlux `HandlerMapping` 实现为 CORS 提供了内置支持。成功将请求映射到处理器后，“HandlerMapping”会检查给定请求和处理程序的 CORS 配置并采取进一步行动。预检请求被直接处理，而简单和实际的 CORS 请求被拦截、验证并设置所需的 CORS 响应首部。

为了启用跨域请求（即，存在 `Origin` 标头并且与请求的主机不同），你需要有一些显式声明的 CORS 配置。如果未找到匹配的 CORS 配置，则拒绝预检请求，并且不会将 CORS 首部添加到简单和实际 CORS 请求的响应中，因此浏览器会拒绝它们。

每个 `HandlerMapping` 可以 [配置 ](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/web/reactive/handler/AbstractHandlerMapping.html#setCorsConfigurations-java.util.Map-) 为单独使用基于 URL 模式的 `CorsConfiguration` 映射。在大多数情况下，应用程序使用 WebFlux Java 配置来声明此类映射，这样的话所有 `HandlerMapping` 实现都会使用一个单一的全局映射。

您可以将“HandlerMapping”级别的全局 CORS 配置与更细粒度的处理器级别的 CORS 配置相结合。例如，带注释的控制器可以使用类或方法级别的 `@CrossOrigin` 注释（其他处理器可以实现 `CorsConfigurationSource`）。

结合全局和局部配置的规则一般是相加的 —— 例如，所有全局和所有局部的源。对于那些只能接受单个值的属性，例如 `allowCredentials` 和 `maxAge`，局部值会覆盖全局值。有关更多详细信息，请参阅 [ CorsConfiguration#combine(CorsConfiguration)](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/web/cors/CorsConfiguration.html#combine-org.springframework.web.cors.CorsConfiguration-)。

> **Tip**
> 
> To learn more from the source or to make advanced customizations, see:
> 
>   - `CorsConfiguration`
> 
>   - `CorsProcessor` and `DefaultCorsProcessor`
> 
>   - `AbstractHandlerMapping`

> **提示**
> 
> 要从源码中了解更多信息或进行高级自定义，请参阅：
> 
>   - `CorsConfiguration`
> 
>   - `CorsProcessor` and `DefaultCorsProcessor`
> 
>   - `AbstractHandlerMapping`

## `@CrossOrigin`

[Web MVC](web.xml#mvc-cors-controller)

The {api-spring-framework}/web/bind/annotation/CrossOrigin.html\[`@CrossOrigin`\] annotation enables cross-origin requests on annotated controller methods, as the following example shows:

[@CrossOrigin](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/web/bind/annotation/CrossOrigin.html)注解在一个基于注解的控制器方法上启用跨站请求访问，如下示例所示：

**Java.**

``` java
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin
    @GetMapping("/{id}")
    public Mono<Account> retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public Mono<Void> remove(@PathVariable Long id) {
        // ...
    }
}
```

**Kotlin.**

``` kotlin
@RestController
@RequestMapping("/account")
class AccountController {

    @CrossOrigin
    @GetMapping("/{id}")
    suspend fun retrieve(@PathVariable id: Long): Account {
        // ...
    }

    @DeleteMapping("/{id}")
    suspend fun remove(@PathVariable id: Long) {
        // ...
    }
}
```

By default, `@CrossOrigin` allows:

  - All origins.

  - All headers.

  - All HTTP methods to which the controller method is mapped.

`allowCredentials` is not enabled by default, since that establishes a trust level that exposes sensitive user-specific information (such as cookies and CSRF tokens) and should be used only where appropriate. When it is enabled either `allowOrigins` must be set to one or more specific domain (but not the special value `"*"`) or alternatively the `allowOriginPatterns` property may be used to match to a dynamic set of origins.

`maxAge` is set to 30 minutes.

`@CrossOrigin` is supported at the class level, too, and inherited by all methods. The following example specifies a certain domain and sets `maxAge` to an hour:

默认情况下，`@CrossOrigin` 允许：

   - 所有源。

   - 所有标题。

   - 控制器方法映射到的所有 HTTP 方法。

`allowCredentials` 默认不启用，因为它建立了一个暴露敏感用户特定信息（例如 cookie 和 CSRF 令牌）的信任级别，并且应该仅在适当的情况下使用。 当它启用时，`allowOrigins` 必须设置为一个或多个特定域（不允许通配符 `"*"`），或者`allowOriginPatterns` 属性可用于匹配一组动态源。

`maxAge` 设置为 30 分钟。

`@CrossOrigin` 在类级别也受支持，并由所有方法继承。 以下示例指定了某个域并将 `maxAge` 设置为一个小时：

**Java.**

``` java
@CrossOrigin(origins = "https://domain2.com", maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @GetMapping("/{id}")
    public Mono<Account> retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public Mono<Void> remove(@PathVariable Long id) {
        // ...
    }
}
```

**Kotlin.**

``` kotlin
@CrossOrigin("https://domain2.com", maxAge = 3600)
@RestController
@RequestMapping("/account")
class AccountController {

    @GetMapping("/{id}")
    suspend fun retrieve(@PathVariable id: Long): Account {
        // ...
    }

    @DeleteMapping("/{id}")
    suspend fun remove(@PathVariable id: Long) {
        // ...
    }
}
```

You can use `@CrossOrigin` at both the class and the method level, as the following example shows:

 `@CrossOrigin` 可以同时在类和方法级别使用，如下示例所示：

**Java.**

``` java
@CrossOrigin(maxAge = 3600) 
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin("https://domain2.com") 
    @GetMapping("/{id}")
    public Mono<Account> retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public Mono<Void> remove(@PathVariable Long id) {
        // ...
    }
}
```

**Kotlin.**

``` kotlin
@CrossOrigin(maxAge = 3600) 
@RestController
@RequestMapping("/account")
class AccountController {

    @CrossOrigin("https://domain2.com") 
    @GetMapping("/{id}")
    suspend fun retrieve(@PathVariable id: Long): Account {
        // ...
    }

    @DeleteMapping("/{id}")
    suspend fun remove(@PathVariable id: Long) {
        // ...
    }
}
```

## Global Configuration

## 全局配置

[Web MVC](web.xml#mvc-cors-global)

In addition to fine-grained, controller method-level configuration, you probably want to define some global CORS configuration, too. You can set URL-based `CorsConfiguration` mappings individually on any `HandlerMapping`. Most applications, however, use the WebFlux Java configuration to do that.

By default global configuration enables the following:

  - All origins.

  - All headers.

  - `GET`, `HEAD`, and `POST` methods.

`allowedCredentials` is not enabled by default, since that establishes a trust level that exposes sensitive user-specific information( such as cookies and CSRF tokens) and should be used only where appropriate. When it is enabled either `allowOrigins` must be set to one or more specific domain (but not the special value `"*"`) or alternatively the `allowOriginPatterns` property may be used to match to a dynamic set of origins.

`maxAge` is set to 30 minutes.

To enable CORS in the WebFlux Java configuration, you can use the `CorsRegistry` callback, as the following example shows:

默认情况下，`@CrossOrigin` 允许：

   - 所有源。

   - 所有标题。

   - 控制器方法映射到的所有 HTTP 方法。

`allowCredentials` 默认不启用，因为它建立了一个暴露敏感用户特定信息（例如 cookie 和 CSRF 令牌）的信任级别，并且应该仅在适当的情况下使用。 当它启用时，`allowOrigins` 必须设置为一个或多个特定域（但不是特殊值 `"*"`），或者`allowOriginPatterns` 属性可用于匹配一组动态源。

`maxAge` 设置为 30 分钟。

`@CrossOrigin` 在类级别也受支持，并由所有方法继承。 以下示例指定了某个域并将 `maxAge` 设置为一个小时：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        registry.addMapping("/api/**")
            .allowedOrigins("https://domain2.com")
            .allowedMethods("PUT", "DELETE")
            .allowedHeaders("header1", "header2", "header3")
            .exposedHeaders("header1", "header2")
            .allowCredentials(true).maxAge(3600);

        // Add more mappings...
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun addCorsMappings(registry: CorsRegistry) {

        registry.addMapping("/api/**")
                .allowedOrigins("https://domain2.com")
                .allowedMethods("PUT", "DELETE")
                .allowedHeaders("header1", "header2", "header3")
                .exposedHeaders("header1", "header2")
                .allowCredentials(true).maxAge(3600)

        // Add more mappings...
    }
}
```

## CORS `WebFilter`

[Web MVC](web.xml#mvc-cors-filter)

You can apply CORS support through the built-in [CorsWebFilter](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/web/cors/reactive/CorsWebFilter.html), which is a good fit with [functional endpoints](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-fn).

可以使用内置的[CorsWebFilter](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/web/cors/reactive/CorsWebFilter.html)来支持CORS，适合使用[functional endpoints](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-fn)。


> **Note**
> 
> If you try to use the `CorsFilter` with Spring Security, keep in mind that Spring Security has [built-in support](https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#cors) for CORS.

To configure the filter, you can declare a `CorsWebFilter` bean and pass a `CorsConfigurationSource` to its constructor, as the following example shows:

> **注意**
>
> 如果将 `CorsFilter` 与 Spring Security 一起使用，请记住 Spring Security 对于CORS具有 [内置支持](https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#cors)。

要配置过滤器，你可以声明一个 `CorsWebFilter` 类型的bean 并将 `CorsConfigurationSource` 传递给其构造函数，如下例所示：

**Java.**

``` java
@Bean
CorsWebFilter corsFilter() {

    CorsConfiguration config = new CorsConfiguration();

    // Possibly...
    // config.applyPermitDefaultValues()

    config.setAllowCredentials(true);
    config.addAllowedOrigin("https://domain1.com");
    config.addAllowedHeader("*");
    config.addAllowedMethod("*");

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);

    return new CorsWebFilter(source);
}
```

**Kotlin.**

``` kotlin
@Bean
fun corsFilter(): CorsWebFilter {

    val config = CorsConfiguration()

    // Possibly...
    // config.applyPermitDefaultValues()

    config.allowCredentials = true
    config.addAllowedOrigin("https://domain1.com")
    config.addAllowedHeader("*")
    config.addAllowedMethod("*")

    val source = UrlBasedCorsConfigurationSource().apply {
        registerCorsConfiguration("/**", config)
    }
    return CorsWebFilter(source)
}
```

# Web Security

[Web MVC](web.xml#mvc-web-security)

The [Spring Security](https://projects.spring.io/spring-security/) project provides support for protecting web applications from malicious exploits. See the Spring Security reference documentation, including:

[Spring Security](https://projects.spring.io/spring-security/) 项目为保护 Web 应用程序免受恶意攻击提供支持。 请参阅 Spring Security 参考文档，包括：

  - [WebFlux Security](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#jc-webflux)

  - [WebFlux Testing Support](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#test-webflux)

  - [CSRF Protection](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#csrf)

  - [Security Response Headers]https://docs.spring.io/spring-security/site/docs/current/reference/html5/#headers)

# View Technologies

# 视图技术

[Web MVC](web.xml#mvc-view)

The use of view technologies in Spring WebFlux is pluggable. Whether you decide to use Thymeleaf, FreeMarker, or some other view technology is primarily a matter of a configuration change. This chapter covers the view technologies integrated with Spring WebFlux. We assume you are already familiar with [View Resolution](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-viewresolution).

Spring WebFlux 中视图技术的使用是可插拔的。 无论你决定使用 Thymeleaf、FreeMarker 还是其他一些视图技术，主要是配置更改的问题。 本章涵盖了与 Spring WebFlux 集成的视图技术。 我们假设你已经熟悉 [View Resolution](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-viewresolution)。

## Thymeleaf

[Web MVC](web.xml#mvc-view-thymeleaf)

Thymeleaf is a modern server-side Java template engine that emphasizes natural HTML templates that can be previewed in a browser by double-clicking, which is very helpful for independent work on UI templates (for example, by a designer) without the need for a running server. Thymeleaf offers an extensive set of features, and it is actively developed and maintained. For a more complete introduction, see the [Thymeleaf](https://www.thymeleaf.org/) project home page.

The Thymeleaf integration with Spring WebFlux is managed by the Thymeleaf project. The configuration involves a few bean declarations, such as `SpringResourceTemplateResolver`, `SpringWebFluxTemplateEngine`, and `ThymeleafReactiveViewResolver`. For more details, see [Thymeleaf+Spring](https://www.thymeleaf.org/documentation.html) and the WebFlux integration [announcement](http://forum.thymeleaf.org/Thymeleaf-3-0-8-JUST-PUBLISHED-td4030687.html).

Thymeleaf 是一个现代的服务器端 Java 模板引擎，它的 HTML 模板可以直接在浏览器中进行预览，非常适合 UI 模板设计工作（例如，设计师），而无需运行服务器。 Thymeleaf 提供了广泛的功能集，并且正在积极开发和维护。 更完整的介绍见[Thymeleaf](https://www.thymeleaf.org/)项目主页。

Thymeleaf 与 Spring WebFlux 的集成由 Thymeleaf 项目管理。 配置涉及一些 bean 声明，例如 `SpringResourceTemplateResolver`、`SpringWebFluxTemplateEngine` 和 `ThymeleafReactiveViewResolver`。 有关更多详细信息，请参阅 [Thymeleaf+Spring](https://www.thymeleaf.org/documentation.html) 和 WebFlux 集成 [公告](http://forum.thymeleaf.org/Thymeleaf-3-0-8-JUST-PUBLISHED-td4030687.html)。

## FreeMarker

[Web MVC](web.xml#mvc-view-freemarker)

[Apache FreeMarker](https://freemarker.apache.org/) is a template engine for generating any kind of text output from HTML to email and others. The Spring Framework has built-in integration for using Spring WebFlux with FreeMarker templates.

[Apache FreeMarker](https://freemarker.apache.org/) 是一个模板引擎，用于生成从 HTML 到电子邮件等的任何类型的文本输出。 Spring 框架具有内置支持将 Spring WebFlux 与 FreeMarker 模板集成。

### View Configuration

[Web MVC](web.xml#mvc-view-freemarker-contextconfig)

The following example shows how to configure FreeMarker as a view technology:

以下示例显示了如何将 FreeMarker 配置为视图展示：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.freeMarker();
    }

    // Configure FreeMarker...

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath("classpath:/templates/freemarker");
        return configurer;
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun configureViewResolvers(registry: ViewResolverRegistry) {
        registry.freeMarker()
    }

    // Configure FreeMarker...

    @Bean
    fun freeMarkerConfigurer() = FreeMarkerConfigurer().apply {
        setTemplateLoaderPath("classpath:/templates/freemarker")
    }
}
```

Your templates need to be stored in the directory specified by the `FreeMarkerConfigurer`, shown in the preceding example. Given the preceding configuration, if your controller returns the view name, `welcome`, the resolver looks for the `classpath:/templates/freemarker/welcome.ftl` template.

模板需要存储在“FreeMarkerConfigurer”指定的目录中，如前面的示例所示。 基于前面的配置，如果控制器返回视图名称“welcome”，解析器将按照这个路径“classpath:/templates/freemarker/welcome.ftl”来查找模板。

### FreeMarker Configuration

### FreeMarker 配置

[Web MVC](web.xml#mvc-views-freemarker)

You can pass FreeMarker 'Settings' and 'SharedVariables' directly to the FreeMarker `Configuration` object (which is managed by Spring) by setting the appropriate bean properties on the `FreeMarkerConfigurer` bean. The `freemarkerSettings` property requires a `java.util.Properties` object, and the `freemarkerVariables` property requires a `java.util.Map`. The following example shows how to use a `FreeMarkerConfigurer`:

你可以通过在 `FreeMarkerConfigurer` bean 上设置适当的 bean 属性，将 FreeMarker 的“Settings”和“SharedVariables”直接传递给 FreeMarker `Configuration` 对象（由 Spring 管理）。 `freemarkerSettings` 属性需要一个 `java.util.Properties` 对象， `freemarkerVariables` 属性需要一个 `java.util.Map`对象。 以下示例显示了如何使用 `FreeMarkerConfigurer`：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    // ...

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        Map<String, Object> variables = new HashMap<>();
        variables.put("xml_escape", new XmlEscape());

        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath("classpath:/templates");
        configurer.setFreemarkerVariables(variables);
        return configurer;
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    // ...

    @Bean
    fun freeMarkerConfigurer() = FreeMarkerConfigurer().apply {
        setTemplateLoaderPath("classpath:/templates")
        setFreemarkerVariables(mapOf("xml_escape" to XmlEscape()))
    }
}
```

See the FreeMarker documentation for details of settings and variables as they apply to the `Configuration` object.

有关适用于 `Configuration` 对象的设置和变量的详细信息，请参阅 FreeMarker 文档。

### Form Handling

### 处理表单

[Web MVC](web.xml#mvc-view-freemarker-forms)

Spring provides a tag library for use in JSPs that contains, among others, a `<spring:bind/>` element. This element primarily lets forms display values from form-backing objects and show the results of failed validations from a `Validator` in the web or business tier. Spring also has support for the same functionality in FreeMarker, with additional convenience macros for generating form input elements themselves.

Spring 提供了一个用于 JSP 的标记库，其中包含一个 `<spring:bind/>` 元素。 此元素主要让表单显示来自表单支持对象的值，并显示来自 Web 或业务层中的“验证器”的验证失败的结果。 Spring 还支持 FreeMarker 中的相同功能，以及用于生成表单输入元素本身的宏。

#### The Bind Macros

[Web MVC](web.xml#mvc-view-bind-macros)

A standard set of macros are maintained within the `spring-webflux.jar` file for FreeMarker, so they are always available to a suitably configured application.

Some of the macros defined in the Spring templating libraries are considered internal (private), but no such scoping exists in the macro definitions, making all macros visible to calling code and user templates. The following sections concentrate only on the macros you need to directly call from within your templates. If you wish to view the macro code directly, the file is called `spring.ftl` and is in the `org.springframework.web.reactive.result.view.freemarker` package.

在 FreeMarker 的 `spring-webflux.jar` 文件中维护了一组标准的宏，因此它们始终可供适当配置的应用程序使用。

Spring 模板库中定义的一些宏被认为是内部的（私有的），但宏定义中不是这样的，所有宏都对调用代码和用户模板可见。 以下部分仅关注你需要从模板中直接调用的宏。 如果你想直接查看宏代码，这个文件叫做 `spring.ftl` ，在 `org.springframework.web.reactive.result.view.freemarker` 这个包中。

For additional details on binding support, see [Simple Binding](web.xml#mvc-view-simple-binding) for Spring MVC.

关于绑定支持的更多信息，请参考[Simple Binding](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-view-simple-binding)

#### Form Macros

#### 表单宏

For details on Spring’s form macro support for FreeMarker templates, consult the following sections of the Spring MVC documentation.

有关 Spring 对 FreeMarker 模板的表单宏支持的详细信息，请参阅 Spring MVC 文档的以下部分。

  - [Input Macros](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-views-form-macros)

  - [Input Fields](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-views-form-macros-input)

  - [Selection Fields](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-views-form-macros-select)

  - [HTML Escaping](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-views-form-macros-html-escaping)

## Script Views

## 脚本视图

[Web MVC](web.xml#mvc-view-script)

The Spring Framework has a built-in integration for using Spring WebFlux with any templating library that can run on top of the [JSR-223](https://www.jcp.org/en/jsr/detail?id=223) Java scripting engine. The following table shows the templating libraries that we have tested on different script engines:

Spring 框架具有内置集成，用于将 Spring WebFlux 与任何可以在 [JSR-223](https://www.jcp.org/en/jsr/detail?id=223) Java 脚本引擎之上运行的模板库一起使用。 下表显示了我们在不同脚本引擎上测试过的模板库：

| Scripting Library                                                                  | Scripting Engine                                      |
| ---------------------------------------------------------------------------------- | ----------------------------------------------------- |
| [Handlebars](https://handlebarsjs.com/)                                            | [Nashorn](https://openjdk.java.net/projects/nashorn/) |
| [Mustache](https://mustache.github.io/)                                            | [Nashorn](https://openjdk.java.net/projects/nashorn/) |
| [React](https://facebook.github.io/react/)                                         | [Nashorn](https://openjdk.java.net/projects/nashorn/) |
| [EJS](https://www.embeddedjs.com/)                                                 | [Nashorn](https://openjdk.java.net/projects/nashorn/) |
| [ERB](https://www.stuartellis.name/articles/erb/)                                  | [JRuby](https://www.jruby.org)                        |
| [String templates](https://docs.python.org/2/library/string.html#template-strings) | [Jython](https://www.jython.org/)                     |
| [Kotlin Script templating](https://github.com/sdeleuze/kotlin-script-templating)   | [Kotlin](https://kotlinlang.org/)                     |

> **Tip**
> 
> The basic rule for integrating any other script engine is that it must implement the `ScriptEngine` and `Invocable` interfaces.

> **提示**
>
> 集成任何其他脚本引擎的基本规则是它必须实现`ScriptEngine` 和`Invocable` 接口。

### Requirements

### 需求

[Web MVC](web.xml#mvc-view-script-dependencies)

You need to have the script engine on your classpath, the details of which vary by script engine:

  - The [Nashorn](https://openjdk.java.net/projects/nashorn/) JavaScript engine is provided with Java 8+. Using the latest update release available is highly recommended.

  - [JRuby](https://www.jruby.org) should be added as a dependency for Ruby support.

  - [Jython](https://www.jython.org) should be added as a dependency for Python support.

  - `org.jetbrains.kotlin:kotlin-script-util` dependency and a `META-INF/services/javax.script.ScriptEngineFactory` file containing a `org.jetbrains.kotlin.script.jsr223.KotlinJsr223JvmLocalScriptEngineFactory` line should be added for Kotlin script support. See [this example](https://github.com/sdeleuze/kotlin-script-templating) for more detail.

You need to have the script templating library. One way to do that for Javascript is through [WebJars](https://www.webjars.org/).

在类路径上需要有脚本引擎相关依赖，其详细信息因脚本引擎而异：

   - [Nashorn](https://openjdk.java.net/projects/nashorn/) JavaScript 引擎随 Java 8+ 提供。 强烈建议使用可用的最新更新版本。

   - [JRuby](https://www.jruby.org) 应添加为 Ruby 支持的依赖项。

   - [Jython](https://www.jython.org) 应添加为 Python 支持的依赖项。

   - 应添加`org.jetbrains.kotlin:kotlin-script-util` 依赖项和包含`org.jetbrains.kotlin.script.jsr223.KotlinJsr223JvmLocalScriptEngineFactory` 行的`META-INF/services/javax.script.ScriptEngineFactory` 文件 用于 Kotlin 脚本支持。 有关更多详细信息，请参阅 [此示例](https://github.com/sdeleuze/kotlin-script-templating)。

你需要有脚本模板库。 对 Javascript 执行此操作的一种方法是通过 [WebJars](https://www.webjars.org/)。

### Script Templates

### 脚本模板

[Web MVC](web.xml#mvc-view-script-integrate)

You can declare a `ScriptTemplateConfigurer` bean to specify the script engine to use, the script files to load, what function to call to render templates, and so on. The following example uses Mustache templates and the Nashorn JavaScript engine:

你可以声明一个 `ScriptTemplateConfigurer` bean 来指定要使用的脚本引擎、要加载的脚本文件、要调用哪些函数来渲染模板等。 以下示例使用 Mustache 模板和 Nashorn JavaScript 引擎：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.scriptTemplate();
    }

    @Bean
    public ScriptTemplateConfigurer configurer() {
        ScriptTemplateConfigurer configurer = new ScriptTemplateConfigurer();
        configurer.setEngineName("nashorn");
        configurer.setScripts("mustache.js");
        configurer.setRenderObject("Mustache");
        configurer.setRenderFunction("render");
        return configurer;
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun configureViewResolvers(registry: ViewResolverRegistry) {
        registry.scriptTemplate()
    }

    @Bean
    fun configurer() = ScriptTemplateConfigurer().apply {
        engineName = "nashorn"
        setScripts("mustache.js")
        renderObject = "Mustache"
        renderFunction = "render"
    }
}
```

The `render` function is called with the following parameters:

  - `String template`: The template content

  - `Map model`: The view model

  - `RenderingContext renderingContext`: The [`RenderingContext`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/web/servlet/view/script/RenderingContext.html) that gives access to the application context, the locale, the template loader, and the URL (since 5.0)

`Mustache.render()` is natively compatible with this signature, so you can call it directly.

If your templating technology requires some customization, you can provide a script that implements a custom render function. For example, [Handlerbars](https://handlebarsjs.com) needs to compile templates before using them and requires a [polyfill](https://en.wikipedia.org/wiki/Polyfill) in order to emulate some browser facilities not available in the server-side script engine. The following example shows how to set a custom render function:

`render` 函数使用以下参数调用：

   - `String template`：模板内容

   - `地图模型`：视图模型

   - `RenderingContext renderingContext`：[`RenderingContext`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/web/servlet/view/script/RenderingContext.html) 提供对应用程序上下文、语言环境、模板加载器和 URL（从5.0版本开始）

`Mustache.render()` 与此签名原生兼容，因此你可以直接调用它。

如果您的模板技术需要一些自定义，你可以提供一个实现自定义渲染功能的脚本。 例如，[Handlerbars](https://handlebarsjs.com) 需要在使用模板之前编译它们并且需要一个 [polyfill](https://en.wikipedia.org/wiki/Polyfill) 以模拟一些在服务器端脚本引擎中不可用的浏览器设施。 以下示例显示了如何设置自定义渲染功能：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.scriptTemplate();
    }

    @Bean
    public ScriptTemplateConfigurer configurer() {
        ScriptTemplateConfigurer configurer = new ScriptTemplateConfigurer();
        configurer.setEngineName("nashorn");
        configurer.setScripts("polyfill.js", "handlebars.js", "render.js");
        configurer.setRenderFunction("render");
        configurer.setSharedEngine(false);
        return configurer;
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun configureViewResolvers(registry: ViewResolverRegistry) {
        registry.scriptTemplate()
    }

    @Bean
    fun configurer() = ScriptTemplateConfigurer().apply {
        engineName = "nashorn"
        setScripts("polyfill.js", "handlebars.js", "render.js")
        renderFunction = "render"
        isSharedEngine = false
    }
}
```

> **Note**
> 
> Setting the `sharedEngine` property to `false` is required when using non-thread-safe script engines with templating libraries not designed for concurrency, such as Handlebars or React running on Nashorn. In that case, Java SE 8 update 60 is required, due to [this bug](https://bugs.openjdk.java.net/browse/JDK-8076099), but it is generally recommended to use a recent Java SE patch release in any case.

`polyfill.js` defines only the `window` object needed by Handlebars to run properly, as the following snippet shows:

> **注意**
>
> 当使用非线程安全脚本引擎和非并发设计的模板库时，需要将 `sharedEngine` 属性设置为 `false`，例如在 Nashorn 上运行的 Handlebars 或 React。 在这种情况下，由于 [此错误](https://bugs.openjdk.java.net/browse/JDK-8076099)，需要 Java SE 8 更新 60，但通常建议使用最新release版本的Java SE补丁。

`polyfill.js` 仅定义了 Handlebars 正常运行所需的 `window` 对象，如下面的代码片段所示：

``` javascript
var window = {};
```

This basic `render.js` implementation compiles the template before using it. A production ready implementation should also store and reused cached templates or pre-compiled templates. This can be done on the script side, as well as any customization you need (managing template engine configuration for example). The following example shows how compile a template:

这个基本的 `render.js` 实现会在使用模板之前对其进行编译。 可用于生产环境的实现还应存储和重用缓存模板或预编译模板。 这可以在脚本端完成，也可以在你需要的任何自定义（例如管理模板引擎配置）上完成。 以下示例显示了如何编译模板：

``` javascript
function render(template, model) {
    var compiledTemplate = Handlebars.compile(template);
    return compiledTemplate(model);
}
```

Check out the Spring Framework unit tests, {spring-framework-main-code}/spring-webflux/src/test/java/org/springframework/web/reactive/result/view/script\[Java\], and {spring-framework-main-code}/spring-webflux/src/test/resources/org/springframework/web/reactive/result/view/script\[resources\], for more configuration examples.

查看 Spring Framework 单元测试，{spring-framework-main-code}/spring-webflux/src/test/java/org/springframework/web/reactive/result/view/script\[Java\] 和 {spring -framework-main-code}/spring-webflux/src/test/resources/org/springframework/web/reactive/result/view/script\[resources\]，更多配置示例。

## JSON and XML

[Web MVC](web.xml#mvc-view-jackson)

For [Content Negotiation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-multiple-representations) purposes, it is useful to be able to alternate between rendering a model with an HTML template or as other formats (such as JSON or XML), depending on the content type requested by the client. To support doing so, Spring WebFlux provides the `HttpMessageWriterView`, which you can use to plug in any of the available [section\_title](#webflux-codecs) from `spring-web`, such as `Jackson2JsonEncoder`, `Jackson2SmileEncoder`, or `Jaxb2XmlEncoder`.

Unlike other view technologies, `HttpMessageWriterView` does not require a `ViewResolver` but is instead [configured](#webflux-config-view-resolvers) as a default view. You can configure one or more such default views, wrapping different `HttpMessageWriter` instances or `Encoder` instances. The one that matches the requested content type is used at runtime.

In most cases, a model contains multiple attributes. To determine which one to serialize, you can configure `HttpMessageWriterView` with the name of the model attribute to use for rendering. If the model contains only one attribute, that one is used.

为了 [内容协商](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-multiple-representations) 的目的，如果能够根据客户端请求的内容类型在渲染HTML模板或其他格式（例如 JSON 或 XML）之间自动切换是很有用的。 为了支持这样做，Spring WebFlux 提供了 `HttpMessageWriterView`，你可以使用它来插入来自 `spring-web` 的任何可用的[编解码器](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-codecs)，例如 `Jackson2JsonEncoder`、`Jackson2SmileEncoder `，或`Jaxb2XmlEncoder`。

与其他视图技术不同，`HttpMessageWriterView` 不需要 `ViewResolver` 而是 [配置](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-config-view-resolvers) 作为默认视图。 你可以配置一个或多个这样的默认视图，包装不同的 `HttpMessageWriter` 实例或 `Encoder` 实例。 在运行时使用与请求的内容类型匹配的内容类型。

在大多数情况下，一个模型包含多个属性。要确定要序列化哪一个，你可以使用用于渲染的模型属性的名称配置 `HttpMessageWriterView`。如果模型仅包含一个属性，则使用该属性。

# HTTP Caching

# HTTP缓存

[Web MVC](web.xml#mvc-caching)

HTTP caching can significantly improve the performance of a web application. HTTP caching revolves around the `Cache-Control` response header and subsequent conditional request headers, such as `Last-Modified` and `ETag`. `Cache-Control` advises private (for example, browser) and public (for example, proxy) caches how to cache and re-use responses. An `ETag` header is used to make a conditional request that may result in a 304 (NOT\_MODIFIED) without a body, if the content has not changed. `ETag` can be seen as a more sophisticated successor to the `Last-Modified` header.

This section describes the HTTP caching related options available in Spring WebFlux.

HTTP 缓存可以显着提高 Web 应用程序的性能。 HTTP 缓存围绕“Cache-Control”响应头和后续条件请求头，例如“Last-Modified”和“ETag”。 `Cache-Control` 建议私有（例如，浏览器）和公共（例如，代理）缓存如何缓存和重用响应。 `ETag` 标头用于发出条件请求，如果内容没有改变，可能会导致 304 (NOT\_MODIFIED) 没有正文。 `ETag` 可以看作是 `Last-Modified` 首部的更复杂的继承者。

本节介绍 Spring WebFlux 中可用的 HTTP 缓存相关选项。

## `CacheControl`

[Web MVC](web.xml#mvc-caching-cachecontrol)

[`CacheControl`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/http/CacheControl.html) provides support for configuring settings related to the `Cache-Control` header and is accepted as an argument in a number of places:

[`CacheControl`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/http/CacheControl.html) 提供支持配置与`Cache-Control` 首部相关的内容并在许多地方被接受为参数：

  - [Controllers](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-caching-etag-lastmodified)

  - [Static Resources](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-caching-static-resources)

While [RFC 7234](https://tools.ietf.org/html/rfc7234#section-5.2.2) describes all possible directives for the `Cache-Control` response header, the `CacheControl` type takes a use case-oriented approach that focuses on the common scenarios, as the following example shows:

虽然 [RFC 7234](https://tools.ietf.org/html/rfc7234#section-5.2.2) 描述了 `Cache-Control` 响应首部的所有可能指令，`CacheControl` 类型采用一个面向用例的方法，聚焦于通用场景，如以下示例所示：

**Java.**

``` java
// Cache for an hour - "Cache-Control: max-age=3600"
CacheControl ccCacheOneHour = CacheControl.maxAge(1, TimeUnit.HOURS);

// Prevent caching - "Cache-Control: no-store"
CacheControl ccNoStore = CacheControl.noStore();

// Cache for ten days in public and private caches,
// public caches should not transform the response
// "Cache-Control: max-age=864000, public, no-transform"
CacheControl ccCustom = CacheControl.maxAge(10, TimeUnit.DAYS).noTransform().cachePublic();
```

**Kotlin.**

``` kotlin
// Cache for an hour - "Cache-Control: max-age=3600"
val ccCacheOneHour = CacheControl.maxAge(1, TimeUnit.HOURS)

// Prevent caching - "Cache-Control: no-store"
val ccNoStore = CacheControl.noStore()

// Cache for ten days in public and private caches,
// public caches should not transform the response
// "Cache-Control: max-age=864000, public, no-transform"
val ccCustom = CacheControl.maxAge(10, TimeUnit.DAYS).noTransform().cachePublic()
```

## Controllers

[Web MVC](web.xml#mvc-caching-etag-lastmodified)

Controllers can add explicit support for HTTP caching. We recommend doing so, since the `lastModified` or `ETag` value for a resource needs to be calculated before it can be compared against conditional request headers. A controller can add an `ETag` and `Cache-Control` settings to a `ResponseEntity`, as the following example shows:

控制器可以添加对 HTTP 缓存的显式支持。 我们建议这样做，因为需要先计算资源的 `lastModified` 或 `ETag` 值，然后才能将其与条件请求首部进行比较。 控制器可以将“ETag”和“Cache-Control”设置添加到“ResponseEntity”，如下例所示：

**Java.**

``` java
@GetMapping("/book/{id}")
public ResponseEntity<Book> showBook(@PathVariable Long id) {

    Book book = findBook(id);
    String version = book.getVersion();

    return ResponseEntity
            .ok()
            .cacheControl(CacheControl.maxAge(30, TimeUnit.DAYS))
            .eTag(version) // lastModified is also available
            .body(book);
}
```

**Kotlin.**

``` kotlin
@GetMapping("/book/{id}")
fun showBook(@PathVariable id: Long): ResponseEntity<Book> {

    val book = findBook(id)
    val version = book.getVersion()

    return ResponseEntity
            .ok()
            .cacheControl(CacheControl.maxAge(30, TimeUnit.DAYS))
            .eTag(version) // lastModified is also available
            .body(book)
}
```

The preceding example sends a 304 (NOT\_MODIFIED) response with an empty body if the comparison to the conditional request headers indicates the content has not changed. Otherwise, the `ETag` and `Cache-Control` headers are added to the response.

You can also make the check against conditional request headers in the controller, as the following example shows:

如果与条件请求首部的比较表明内容未更改，则前面的示例发送带有空正文的 304 (NOT\_MODIFIED) 响应。 否则，`ETag` 和 `Cache-Control` 首部被添加到响应中。

你还可以对控制器中的条件请求标头进行检查，如以下示例所示：

**Java.**

``` java
@RequestMapping
public String myHandleMethod(ServerWebExchange exchange, Model model) {

    long eTag = ... 

    if (exchange.checkNotModified(eTag)) {
        return null; 
    }

    model.addAttribute(...); 
    return "myViewName";
}
```

  - Application-specific calculation.

  - Response has been set to 304 (NOT\_MODIFIED). No further processing.

  - Continue with request processing.

  - 特定于应用程序的计算。

  - 响应已设置为 304 (NOT\_MODIFIED)。 没有进一步处理。

  - 继续请求处理。

**Kotlin.**

``` kotlin
@RequestMapping
fun myHandleMethod(exchange: ServerWebExchange, model: Model): String? {

    val eTag: Long = ... 

    if (exchange.checkNotModified(eTag)) {
        return null
    }

    model.addAttribute(...) 
    return "myViewName"
}
```

  - Application-specific calculation.

  - Response has been set to 304 (NOT\_MODIFIED). No further processing.

  - Continue with request processing.

  - 特定于应用程序的计算。

  - 响应已设置为 304 (NOT\_MODIFIED)。 没有进一步处理。

  - 继续请求处理。

There are three variants for checking conditional requests against `eTag` values, `lastModified` values, or both. For conditional `GET` and `HEAD` requests, you can set the response to 304 (NOT\_MODIFIED). For conditional `POST`, `PUT`, and `DELETE`, you can instead set the response to 412 (PRECONDITION\_FAILED) to prevent concurrent modification.


有三种变体用于根据“eTag”值、“lastModified”值或两者来检查条件请求。 对于条件“GET”和“HEAD”请求，你可以将响应设置为 304（NOT\_MODIFIED）。 对于条件`POST`、`PUT`和`DELETE`，你可以将响应设置为412（PRECONDITION\_FAILED）以防止并发修改。

## Static Resources

## 静态资源

[Web MVC](web.xml#mvc-caching-static-resources)

You should serve static resources with a `Cache-Control` and conditional response headers for optimal performance. See the section on configuring [Static Resources](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-config-static-resources).

你应该使用 `Cache-Control` 和条件响应首部为静态资源提供最佳性能。 参见配置[静态资源](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-config-static-resources)部分。

# WebFlux Config

[Web MVC](web.xml#mvc-config)

The WebFlux Java configuration declares the components that are required to process requests with annotated controllers or functional endpoints, and it offers an API to customize the configuration. That means you do not need to understand the underlying beans created by the Java configuration. However, if you want to understand them, you can see them in `WebFluxConfigurationSupport` or read more about what they are in [Special Bean Types](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-special-bean-types).

WebFlux Java 配置声明了处理带有带注解的控制器或功能端点的请求所需的组件，并提供了一个 API 来自定义配置。 这意味着你不需要了解由 Java 配置创建的底层 bean。 然而，如果你想了解它们，你可以在`WebFluxConfigurationSupport`中找到它们或者在[Special Bean Types](https://docs.spring.io/spring-framework/docs/current/reference)中阅读更多关于它们的内容。

For more advanced customizations, not available in the configuration API, you can gain full control over the configuration through the [Advanced Configuration Mode](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-config-advanced-java).

对于配置 API 中没有的更高级的自定义，您可以通过[高级配置模式](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-config-advanced-java)

## Enabling WebFlux Config

[Web MVC](web.xml#mvc-config-enable)

You can use the `@EnableWebFlux` annotation in your Java config, as the following example shows:

你可以在Java配置中使用 `@EnableWebFlux` 注解，如下例所示：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig {
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig
```

The preceding example registers a number of Spring WebFlux [infrastructure beans](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-special-bean-types) and adapts to dependencies available on the classpath — for JSON, XML, and others.

前面的示例根据类路径上可用的依赖项（如JSON、XML 等）注册了多个 Spring WebFlux [基础架构 bean](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-special-bean-types) 。

## WebFlux config API

[Web MVC](web.xml#mvc-config-customize)

In your Java configuration, you can implement the `WebFluxConfigurer` interface, as the following example shows:

在 Java 配置中，你可以实现 `WebFluxConfigurer` 接口，如以下示例所示：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    // Implement configuration methods...
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    // Implement configuration methods...
}
```

## Conversion, formatting

[Web MVC](web.xml#mvc-config-conversion)

By default, formatters for various number and date types are installed, along with support for customization via `@NumberFormat` and `@DateTimeFormat` on fields.

To register custom formatters and converters in Java config, use the following:

默认情况下，安装了各种数字和日期类型的格式化器，并支持通过字段上的“@NumberFormat”和“@DateTimeFormat”注解进行自定义。

要在 Java 配置中注册自定义格式化器和转换器，请使用以下命令：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        // ...
    }

}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun addFormatters(registry: FormatterRegistry) {
        // ...
    }
}
```

By default Spring WebFlux considers the request Locale when parsing and formatting date values. This works for forms where dates are represented as Strings with "input" form fields. For "date" and "time" form fields, however, browsers use a fixed format defined in the HTML spec. For such cases date and time formatting can be customized as follows:

默认情况下，Spring WebFlux 在解析和格式化日期值时会考虑请求区域设置。 这适用于将日期表示为带有“输入”表单字段的字符串的表单。 但是，对于“日期”和“时间”表单字段，浏览器使用 HTML 规范中定义的固定格式。 对于这种情况，可以按如下方式自定义日期和时间格式：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        DateTimeFormatterRegistrar registrar = new DateTimeFormatterRegistrar();
        registrar.setUseIsoFormat(true);
        registrar.registerFormatters(registry);
        }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun addFormatters(registry: FormatterRegistry) {
        val registrar = DateTimeFormatterRegistrar()
        registrar.setUseIsoFormat(true)
        registrar.registerFormatters(registry)
    }
}
```

> **Note**
> 
> See [`FormatterRegistrar` SPI](core.xml#format-FormatterRegistrar-SPI) and the `FormattingConversionServiceFactoryBean` for more information on when to use `FormatterRegistrar` implementations.

> **注意**
>
> 请参阅 [`FormatterRegistrar` SPI](core.xml#format-FormatterRegistrar-SPI) 和 `FormattingConversionServiceFactoryBean` 以获取有关何时使用 `FormatterRegistrar` 实现的更多信息。

## Validation

[Web MVC](web.xml#mvc-config-validation)

By default, if [Bean Validation](core.xml#validation-beanvalidation-overview) is present on the classpath (for example, the Hibernate Validator), the `LocalValidatorFactoryBean` is registered as a global [validator](core.xml#validator) for use with `@Valid` and `@Validated` on `@Controller` method arguments.

In your Java configuration, you can customize the global `Validator` instance, as the following example shows:

默认情况下，如果 [Bean Validation](core.xml#validation-beanvalidation-overview) 存在于类路径上（例如，Hibernate Validator），则`LocalValidatorFactoryBean` 被注册为全局 [validator](core.xml#validator) ，与 `@Valid` 和 `@Validated` 在 `@Controller` 方法参数上配合使用。

在您的 Java 配置中，您可以自定义全局 `Validator` 实例，如以下示例所示：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public Validator getValidator(); {
        // ...
    }

}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun getValidator(): Validator {
        // ...
    }

}
```

Note that you can also register `Validator` implementations locally, as the following example shows:

请注意，你还可以在局部注册 `Validator` 实现，如以下示例所示：

**Java.**

``` java
@Controller
public class MyController {

    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.addValidators(new FooValidator());
    }

}
```

**Kotlin.**

``` kotlin
@Controller
class MyController {

    @InitBinder
    protected fun initBinder(binder: WebDataBinder) {
        binder.addValidators(FooValidator())
    }
}
```

> **Tip**
> 
> If you need to have a `LocalValidatorFactoryBean` injected somewhere, create a bean and mark it with `@Primary` in order to avoid conflict with the one declared in the MVC config.

> **提示**
>
> 如果您需要在某处注入 `LocalValidatorFactoryBean`，请创建一个 bean 并用 `@Primary` 标记它以避免与 MVC 配置中声明的那个冲突。

## Content Type Resolvers

[Web MVC](web.xml#mvc-config-content-negotiation)

You can configure how Spring WebFlux determines the requested media types for `@Controller` instances from the request. By default, only the `Accept` header is checked, but you can also enable a query parameter-based strategy.

The following example shows how to customize the requested content type resolution:

你可以配置 Spring WebFlux 如何从请求中为 `@Controller` 实例确定请求的媒体类型。 默认情况下，仅检查 `Accept` 首部，但你也可以启用基于查询参数的策略。

以下示例显示了如何自定义请求的内容类型解析：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void configureContentTypeResolver(RequestedContentTypeResolverBuilder builder) {
        // ...
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun configureContentTypeResolver(builder: RequestedContentTypeResolverBuilder) {
        // ...
    }
}
```

## HTTP message codecs

[Web MVC](web.xml#mvc-config-message-converters)

The following example shows how to customize how the request and response body are read and written:

以下示例显示如何自定义读取和写入请求和响应正文的方式：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        configurer.defaultCodecs().maxInMemorySize(512 * 1024);
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun configureHttpMessageCodecs(configurer: ServerCodecConfigurer) {
        // ...
    }
}
```

`ServerCodecConfigurer` provides a set of default readers and writers. You can use it to add more readers and writers, customize the default ones, or replace the default ones completely.

For Jackson JSON and XML, consider using [`Jackson2ObjectMapperBuilder`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/http/converter/json/Jackson2ObjectMapperBuilder.html), which customizes Jackson’s default properties with the following ones:

  - [`DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES`](https://fasterxml.github.io/jackson-databind/javadoc/2.6/com/fasterxml/jackson/databind/DeserializationFeature.html#FAIL_ON_UNKNOWN_PROPERTIES) is disabled.

  - [`MapperFeature.DEFAULT_VIEW_INCLUSION`](https://fasterxml.github.io/jackson-databind/javadoc/2.6/com/fasterxml/jackson/databind/MapperFeature.html#DEFAULT_VIEW_INCLUSION) is disabled.

It also automatically registers the following well-known modules if they are detected on the classpath:

  - [`jackson-datatype-joda`](https://github.com/FasterXML/jackson-datatype-joda): Support for Joda-Time types.

  - [`jackson-datatype-jsr310`](https://github.com/FasterXML/jackson-datatype-jsr310): Support for Java 8 Date and Time API types.

  - [`jackson-datatype-jdk8`](https://github.com/FasterXML/jackson-datatype-jdk8): Support for other Java 8 types, such as `Optional`.

  - [`jackson-module-kotlin`](https://github.com/FasterXML/jackson-module-kotlin): Support for Kotlin classes and data classes.

`ServerCodecConfigurer` 提供了一组默认的读取器和写入器。你可以使用它来添加更多的读取器和写入器，自定义默认的，或完全替换默认的。

对于 Jackson JSON 和 XML，请考虑使用 [`Jackson2ObjectMapperBuilder`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/http/converter/json/Jackson2ObjectMapperBuilder.html)，它使用以下属性自定义 Jackson 的默认属性：

  - [`DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES`](https://fasterxml.github.io/jackson-databind/javadoc/2.6/com/fasterxml/jackson/databind/DeserializationFeature.html#FAIL_ON_UNKNOWN_PROPERTIES)被禁用。

  - [`MapperFeature.DEFAULT_VIEW_INCLUSION`](https://fasterxml.github.io/jackson-databind/javadoc/2.6/com/fasterxml/jackson/databind/MapperFeature.html#DEFAULT_VIEW_INCLUSION) 被禁用。

如果在类路径中检测到以下模块，它还会自动注册：

  - [`jackson-datatype-joda`](https://github.com/FasterXML/jackson-datatype-joda)：支持 Joda-Time 类型。

  - [`jackson-datatype-jsr310`](https://github.com/FasterXML/jackson-datatype-jsr310)：支持 Java 8 日期和时间 API 类型。

  - [`jackson-datatype-jdk8`](https://github.com/FasterXML/jackson-datatype-jdk8)：支持其他Java 8类型，例如`Optional`。

  - [`jackson-module-kotlin`](https://github.com/FasterXML/jackson-module-kotlin)：支持 Kotlin 类和数据类。

## View Resolvers

[Web MVC](web.xml#mvc-config-view-resolvers)

The following example shows how to configure view resolution:

以下示例展示如何配置视图解析：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        // ...
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun configureViewResolvers(registry: ViewResolverRegistry) {
        // ...
    }
}
```

The `ViewResolverRegistry` has shortcuts for view technologies with which the Spring Framework integrates. The following example uses FreeMarker (which also requires configuring the underlying FreeMarker view technology):

`ViewResolverRegistry` 为 Spring 框架集成的视图技术提供了快捷方式。 下面的例子使用了 FreeMarker（也需要配置底层的 FreeMarker 视图技术）：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {


    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.freeMarker();
    }

    // Configure Freemarker...

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath("classpath:/templates");
        return configurer;
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun configureViewResolvers(registry: ViewResolverRegistry) {
        registry.freeMarker()
    }

    // Configure Freemarker...

    @Bean
    fun freeMarkerConfigurer() = FreeMarkerConfigurer().apply {
        setTemplateLoaderPath("classpath:/templates")
    }
}
```

You can also plug in any `ViewResolver` implementation, as the following example shows:

你也可以自定义 `ViewResolver`的实现，如下例所示：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {


    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        ViewResolver resolver = ... ;
        registry.viewResolver(resolver);
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun configureViewResolvers(registry: ViewResolverRegistry) {
        val resolver: ViewResolver = ...
        registry.viewResolver(resolver
    }
}
```

To support [Content Negotiation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-multiple-representations) and rendering other formats through view resolution (besides HTML), you can configure one or more default views based on the `HttpMessageWriterView` implementation, which accepts any of the available [Codecs](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-codecs) from `spring-web`. The following example shows how to do so:

为了支持 [Content Negotiation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-multiple-representations) 和通过视图分辨率渲染其他格式（除了 HTML)，你可以基于 `HttpMessageWriterView` 实现配置一个或多个默认视图，它接受任何可用的来自`spring-web` [Codecs](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-codecs)。 以下示例显示了如何执行此操作：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {


    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.freeMarker();

        Jackson2JsonEncoder encoder = new Jackson2JsonEncoder();
        registry.defaultViews(new HttpMessageWriterView(encoder));
    }

    // ...
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {


    override fun configureViewResolvers(registry: ViewResolverRegistry) {
        registry.freeMarker()

        val encoder = Jackson2JsonEncoder()
        registry.defaultViews(HttpMessageWriterView(encoder))
    }

    // ...
}
```

See [View Technologies](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-view) for more on the view technologies that are integrated with Spring WebFlux.

有关与 Spring WebFlux 集成的视图技术的更多信息，请参阅 [视图技术](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-view) .

## Static Resources

[Web MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-static-resources)

This option provides a convenient way to serve static resources from a list of [`Resource`](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/core/io/Resource.html)-based locations.

In the next example, given a request that starts with `/resources`, the relative path is used to find and serve static resources relative to `/static` on the classpath. Resources are served with a one-year future expiration to ensure maximum use of the browser cache and a reduction in HTTP requests made by the browser. The `Last-Modified` header is also evaluated and, if present, a `304` status code is returned. The following list shows the example:

此选项提供了一种基于一系列[`Resource`](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/core/io/Resource.html) 的位置提供静态资源的方法。

在下一个示例中，给定以 `/resources` 开头的请求，相对路径用于在类路径上查找和提供相对于 `/static` 的静态资源。 资源有效期为1年，以确保最大限度地使用浏览器缓存并减少浏览器发出的 HTTP 请求。 `Last-Modified` 首部也会被评估，如果存在，则返回 `304` 状态代码。 以下列表显示了示例：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
            .addResourceLocations("/public", "classpath:/static/")
            .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS));
    }

}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun addResourceHandlers(registry: ResourceHandlerRegistry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("/public", "classpath:/static/")
                .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS))
    }
}
```

The resource handler also supports a chain of [ResourceResolver](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/web/reactive/resource/ResourceResolver.html) implementations and [ResourceTransformer](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/web/reactive/resource/ResourceTransformer.html) implementations, which can be used to create a toolchain for working with optimized resources.

You can use the `VersionResourceResolver` for versioned resource URLs based on an MD5 hash computed from the content, a fixed application version, or other information. A `ContentVersionStrategy` (MD5 hash) is a good choice with some notable exceptions (such as JavaScript resources used with a module loader).

The following example shows how to use `VersionResourceResolver` in your Java configuration:

资源处理器还支持配置 [ResourceResolver](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/web/reactive/resource/ResourceResolver.html) 实现和 [ResourceTransformer](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/web/reactive/resource/ResourceTransformer.html) 实现， 用来创建处理可优化资源的工具链。

你可以根据从内容、固定应用程序版本或其他信息计算出的 MD5 哈希，将 `VersionResourceResolver` 用于版本化资源 URL。 `ContentVersionStrategy`（MD5 哈希）是一个不错的选择，但有一些值得注意的例外（例如与模块加载器一起使用的 JavaScript 资源）。

以下示例显示了如何在 Java 配置中使用 `VersionResourceResolver`：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("/public/")
                .resourceChain(true)
                .addResolver(new VersionResourceResolver().addContentVersionStrategy("/**"));
    }

}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    override fun addResourceHandlers(registry: ResourceHandlerRegistry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("/public/")
                .resourceChain(true)
                .addResolver(VersionResourceResolver().addContentVersionStrategy("/**"))
    }

}
```

You can use `ResourceUrlProvider` to rewrite URLs and apply the full chain of resolvers and transformers (for example, to insert versions). The WebFlux configuration provides a `ResourceUrlProvider` so that it can be injected into others.

Unlike Spring MVC, at present, in WebFlux, there is no way to transparently rewrite static resource URLs, since there are no view technologies that can make use of a non-blocking chain of resolvers and transformers. When serving only local resources, the workaround is to use `ResourceUrlProvider` directly (for example, through a custom element) and block.

Note that, when using both `EncodedResourceResolver` (for example, Gzip, Brotli encoded) and `VersionedResourceResolver`, they must be registered in that order, to ensure content-based versions are always computed reliably based on the unencoded file.

[WebJars](https://www.webjars.org/documentation) are also supported through the `WebJarsResourceResolver` which is automatically registered when the `org.webjars:webjars-locator-core` library is present on the classpath. The resolver can re-write URLs to include the version of the jar and can also match against incoming URLs without versions—for example, from `/jquery/jquery.min.js` to `/jquery/1.2.0/jquery.min.js`.

你可以使用 `ResourceUrlProvider` 来重写 URL 并应用完整的解析器和转换器链（例如，插入版本）。 WebFlux 配置提供了一个 `ResourceUrlProvider`，以便它可以注入到其他地方。

与 Spring MVC 不同，目前在 WebFlux 中，没有办法透明地重写静态资源 URL，因为没有视图技术可以利用解析器和转换器的非阻塞链。当仅提供本地资源时，解决方法是直接使用 `ResourceUrlProvider`（例如，通过自定义元素）并阻塞。

请注意，当同时使用 `EncodedResourceResolver` （例如，Gzip、Brotli 编码）和 `VersionedResourceResolver` 时，它们必须按该顺序注册，以确保始终基于未编码的文件可靠地计算基于内容的版本。

[WebJars](https://www.webjars.org/documentation) 也通过 `WebJarsResourceResolver` 提供支持，当`org.webjars:webjars-locator-core` 库出现在类路径中时，它会自动注册。解析器可以重写 URL 以包含 jar 的版本，也可以匹配没有版本的传入 URL——例如，从 `/jquery/jquery.min.js` 到 `/jquery/1.2.0/jquery.min.js`。

## Path Matching

[Web MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-path-matching)

You can customize options related to path matching. For details on the individual options, see the [PathMatchConfigurer](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/web/reactive/config/PathMatchConfigurer.html) javadoc. The following example shows how to use `PathMatchConfigurer`:

你可以自定义路径匹配相关的配置。有关各个选项的详细信息，请参阅 [PathMatchConfigurer](https://docs.spring.io/spring-framework/docs/5.3.8/javadoc-api/org/springframework/web/reactive/config/PathMatchConfigurer.html ) 文档。 以下示例演示了如何使用 `PathMatchConfigurer`：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer
            .setUseCaseSensitiveMatch(true)
            .setUseTrailingSlashMatch(false)
            .addPathPrefix("/api",
                    HandlerTypePredicate.forAnnotation(RestController.class));
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    @Override
    fun configurePathMatch(configurer: PathMatchConfigurer) {
        configurer
            .setUseCaseSensitiveMatch(true)
            .setUseTrailingSlashMatch(false)
            .addPathPrefix("/api",
                    HandlerTypePredicate.forAnnotation(RestController::class.java))
    }
}
```

> **Tip**
> 
> Spring WebFlux relies on a parsed representation of the request path called `RequestPath` for access to decoded path segment values, with semicolon content removed (that is, path or matrix variables). That means, unlike in Spring MVC, you need not indicate whether to decode the request path nor whether to remove semicolon content for path matching purposes.
> 
> Spring WebFlux also does not support suffix pattern matching, unlike in Spring MVC, where we are also [recommend](web.xml#mvc-ann-requestmapping-suffix-pattern-match) moving away from reliance on it.

> **提示**
>
> Spring WebFlux 依赖于称为 `RequestPath` 的请求路径的解析表示来访问解码的路径段值，其中删除了分号内容（即路径或矩阵变量）。 这意味着，与 Spring MVC 不同，你不需要指示是否对请求路径进行解码，也不需要出于路径匹配的目的而删除分号内容。
>
> Spring WebFlux 也不支持后缀模式匹配，不像在 Spring MVC 中，我们也在 [推荐](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-requestmapping-suffix-pattern-match) 摆脱对它的依赖。

## WebSocketService

The WebFlux Java config declares of a `WebSocketHandlerAdapter` bean which provides support for the invocation of WebSocket handlers. That means all that remains to do in order to handle a WebSocket handshake request is to map a `WebSocketHandler` to a URL via `SimpleUrlHandlerMapping`.

In some cases it may be necessary to create the `WebSocketHandlerAdapter` bean with a provided `WebSocketService` service which allows configuring WebSocket server properties. For example:

WebFlux Java 配置声明了一个`WebSocketHandlerAdapter` bean，它为调用 WebSocket 处理程序提供支持。 这意味着为了处理 WebSocket 握手请求，剩下要做的就是通过 `SimpleUrlHandlerMapping` 将 `WebSocketHandler` 映射到 URL。

在某些情况下，可能需要使用提供的 WebSocketService 服务创建 WebSocketHandlerAdapter bean，该服务允许配置 WebSocket 服务器属性。 例如：

**Java.**

``` java
@Configuration
@EnableWebFlux
public class WebConfig implements WebFluxConfigurer {

    @Override
    public WebSocketService getWebSocketService() {
        TomcatRequestUpgradeStrategy strategy = new TomcatRequestUpgradeStrategy();
        strategy.setMaxSessionIdleTimeout(0L);
        return new HandshakeWebSocketService(strategy);
    }
}
```

**Kotlin.**

``` kotlin
@Configuration
@EnableWebFlux
class WebConfig : WebFluxConfigurer {

    @Override
    fun webSocketService(): WebSocketService {
        val strategy = TomcatRequestUpgradeStrategy().apply {
            setMaxSessionIdleTimeout(0L)
        }
        return HandshakeWebSocketService(strategy)
    }
}
```

## Advanced Configuration Mode

[Web MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-advanced-java)

`@EnableWebFlux` imports `DelegatingWebFluxConfiguration` that:

  - Provides default Spring configuration for WebFlux applications

  - detects and delegates to `WebFluxConfigurer` implementations to customize that configuration.

For advanced mode, you can remove `@EnableWebFlux` and extend directly from `DelegatingWebFluxConfiguration` instead of implementing `WebFluxConfigurer`, as the following example shows:

`@EnableWebFlux` 导入了 `DelegatingWebFluxConfiguration`：

   - 为 WebFlux 应用程序提供默认的 Spring 配置

   - 检测并委托给`WebFluxConfigurer` 实现来自定义该配置。

对于高级模式，你可以删除 `@EnableWebFlux` 注解并直接从 `DelegatingWebFluxConfiguration` 扩展，而不是实现 `WebFluxConfigurer`，如下例所示：

**Java.**

``` java
@Configuration
public class WebConfig extends DelegatingWebFluxConfiguration {

    // ...
}
```

**Kotlin.**

``` kotlin
@Configuration
class WebConfig : DelegatingWebFluxConfiguration {

    // ...
}
```

You can keep existing methods in `WebConfig`, but you can now also override bean declarations from the base class and still have any number of other `WebMvcConfigurer` implementations on the classpath.

你可以在 `WebConfig` 中保留现有方法，也可以覆盖基类中的 bean 声明，并且在类路径上仍然有任意数量的其他 `WebMvcConfigurer` 实现。

# HTTP/2

[Web MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-http2)

HTTP/2 is supported with Reactor Netty, Tomcat, Jetty, and Undertow. However, there are considerations related to server configuration. For more details, see the [HTTP/2 wiki page](https://github.com/spring-projects/spring-framework/wiki/HTTP-2-support).

Reactor Netty、Tomcat、Jetty 和 Undertow 支持 HTTP/2。 但是，有一些与服务器配置相关的注意事项。 更多详情请查看[HTTP/2 wiki页面](https://github.com/spring-projects/spring-framework/wiki/HTTP-2-support)。
