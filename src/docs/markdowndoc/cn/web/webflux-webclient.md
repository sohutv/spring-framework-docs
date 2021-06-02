Spring WebFlux includes a client to perform HTTP requests with. `WebClient` has a functional, fluent API based on Reactor, see [web-reactive.xml](web-reactive.xml#webflux-reactive-libraries), which enables declarative composition of asynchronous logic without the need to deal with threads or concurrency. It is fully non-blocking, it supports streaming, and relies on the same [codecs](web-reactive.xml#webflux-codecs) that are also used to encode and decode request and response content on the server side.

Spring WebFlux包括一个用于执行HTTP请求的客户端。 `WebClient`使用函数风格的流式API，基于Reactor实现，详细内容请参见[web-reactive.xml](web-reactive.xml＃webflux-reactive-libraries)，它实现了声明式的异步逻辑组合，而无需处理线程或者并发相关内容。 它是完全非阻塞的，支持流，并且依赖于与服务端相同的[编解码器](web-reactive.xml＃webflux-codecs)，该[编解码器]也用于在服务器端对请求和响应内容进行编码和解码。

`WebClient` needs an HTTP client library to perform requests with. There is built-in support for the following:

`WebClient` 需要一个HTTP客户端库来发出请求。内建支持以下几种：

  - [Reactor Netty](https://github.com/reactor/reactor-netty)

  - [Jetty Reactive HttpClient](https://github.com/jetty-project/jetty-reactive-httpclient)

  - [Apache HttpComponents](https://hc.apache.org/index.html)

  - Others can be plugged via `ClientHttpConnector`.

# Configuration

# 配置

The simplest way to create a `WebClient` is through one of the static factory methods:

创建`WebClient`最简单的方式是通过下面的静态工厂方法：

  - `WebClient.create()`

  - `WebClient.create(String baseUrl)`

You can also use `WebClient.builder()` with further options:

也可以使用`WebClient.builder()`进行相关配置的自定义：

  - `uriBuilderFactory`: Customized `UriBuilderFactory` to use as a base URL.

  - `uriBuilderFactory`: 自定义 `UriBuilderFactory` 使用一个基本的URL。

  - `defaultUriVariables`: default values to use when expanding URI templates.

  - `defaultUriVariables`: 填充URI模板时使用的默认值。

  - `defaultHeader`: Headers for every request.

  - `defaultHeader`: 请求默认Header。

  - `defaultCookie`: Cookies for every request.

  - `defaultCookie`: 请求默认Cookies。

  - `defaultRequest`: `Consumer` to customize every request.

  - `defaultRequest`: 用来定制请求的`Consumer`。

  - `filter`: Client filter for every request.

  - `filter`: 处理请求的客户端过滤器。

  - `exchangeStrategies`: HTTP message reader/writer customizations.

  - `exchangeStrategies`: 定制HTTP message reader/writer。

  - `clientConnector`: HTTP client library settings.

  - `clientConnector`: HTTP客户端库相关配置。

For example:

例如：

**Java.**

``` java
WebClient client = WebClient.builder()
        .codecs(configurer -> ... )
        .build();
```

**Kotlin.**

``` kotlin
val webClient = WebClient.builder()
        .codecs { configurer -> ... }
        .build()
```

Once built, a `WebClient` is immutable. However, you can clone it and build a modified copy as follows:

一旦构建完成，`WebClient`就是不可变。但是，你可以创建一个克隆实例来进行配置修改。

**Java.**

``` java
WebClient client1 = WebClient.builder()
        .filter(filterA).filter(filterB).build();

WebClient client2 = client1.mutate()
        .filter(filterC).filter(filterD).build();

// client1 has filterA, filterB

// client2 has filterA, filterB, filterC, filterD
```

**Kotlin.**

``` kotlin
val client1 = WebClient.builder()
        .filter(filterA).filter(filterB).build()

val client2 = client1.mutate()
        .filter(filterC).filter(filterD).build()

// client1 has filterA, filterB

// client2 has filterA, filterB, filterC, filterD
```

## MaxInMemorySize

Codecs have [limits](web-reactive.xml#webflux-codecs-limits) for buffering data in memory to avoid application memory issues. By the default those are set to 256KB. If that’s not enough you’ll get the following error:

编解码器[限制](web-reactive.xml#webflux-codecs-limits)了内存缓冲区的大小来避免一些内存问题。

    org.springframework.core.io.buffer.DataBufferLimitException: Exceeded limit on max bytes to buffer

To change the limit for default codecs, use the following:

可以通过一下方式来修改限制：

**Java.**

``` java
WebClient webClient = WebClient.builder()
        .codecs(configurer -> configurer.defaultCodecs().maxInMemorySize(2 * 1024 * 1024))
        .build();
```

**Kotlin.**

``` kotlin
val webClient = WebClient.builder()
        .codecs { configurer -> configurer.defaultCodecs().maxInMemorySize(2 * 1024 * 1024) }
        .build()
```

## Reactor Netty

To customize Reactor Netty settings, provide a pre-configured `HttpClient`:

如果想定制Reactor Netty的配置，提供一个预定义的`HttpClient`：

**Java.**

``` java
HttpClient httpClient = HttpClient.create().secure(sslSpec -> ...);

WebClient webClient = WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
```

**Kotlin.**

``` kotlin
val httpClient = HttpClient.create().secure { ... }

val webClient = WebClient.builder()
    .clientConnector(ReactorClientHttpConnector(httpClient))
    .build()
```

### Resources

By default, `HttpClient` participates in the global Reactor Netty resources held in `reactor.netty.http.HttpResources`, including event loop threads and a connection pool. This is the recommended mode, since fixed, shared resources are preferred for event loop concurrency. In this mode global resources remain active until the process exits.

默认情况下，`HttpClient`使用存储在`reactor.netty.http.HttpResources`中的全局Reactor Netty资源，包括事件循环和连接池。对于事件循环并发模式来说这是推荐的，优化的使用方法。在这种使用方法下，全局资源在进程结束前一直保持活跃状态。

If the server is timed with the process, there is typically no need for an explicit shutdown. However, if the server can start or stop in-process (for example, a Spring MVC application deployed as a WAR), you can declare a Spring-managed bean of type `ReactorResourceFactory` with `globalResources=true` (the default) to ensure that the Reactor Netty global resources are shut down when the Spring `ApplicationContext` is closed, as the following example shows:

如果服务器与进程的生命周期相同，通常没有必要显示的去关闭。但是，如果服务器可以在进程内启动或者停止（例如，以WAR方式部署的Spring MVC应用），可以定义一个Spring管理的`ReactorResourceFactory`类型（默认`globalResources=true`）的bean来保证Reactor Netty全局资源在Spring `ApplicationContext`关闭的时候也随之关闭，如下所示：

**Java.**

``` java
@Bean
public ReactorResourceFactory reactorResourceFactory() {
    return new ReactorResourceFactory();
}
```

**Kotlin.**

``` kotlin
@Bean
fun reactorResourceFactory() = ReactorResourceFactory()
```

You can also choose not to participate in the global Reactor Netty resources. However, in this mode, the burden is on you to ensure that all Reactor Netty client and server instances use shared resources, as the following example shows:

你也可以选择不使用Reactor Netty全局资源。但是在这种情况下，需要你自行确保所有的Reactor Netty客户端和服务端都是用同样的共享资源，如下：

**Java.**

``` java
@Bean
public ReactorResourceFactory resourceFactory() {
    ReactorResourceFactory factory = new ReactorResourceFactory();
    factory.setUseGlobalResources(false); 
    return factory;
}

@Bean
public WebClient webClient() {

    Function<HttpClient, HttpClient> mapper = client -> {
        // Further customizations...
    };

    ClientHttpConnector connector =
            new ReactorClientHttpConnector(resourceFactory(), mapper); 

    return WebClient.builder().clientConnector(connector).build(); 
}
```

  - Create resources independent of global ones.

  - 创建独立于全局的资源。

  - Use the `ReactorClientHttpConnector` constructor with resource factory.

  - 使用`ReactorClientHttpConnector` 带有资源工厂参数的构造函数。

  - Plug the connector into the `WebClient.Builder`.

  - 把connector注入到`WebClient.Builder`。

**Kotlin.**

``` kotlin
@Bean
fun resourceFactory() = ReactorResourceFactory().apply {
    isUseGlobalResources = false 
}

@Bean
fun webClient(): WebClient {

    val mapper: (HttpClient) -> HttpClient = {
        // Further customizations...
    }

    val connector = ReactorClientHttpConnector(resourceFactory(), mapper) 

    return WebClient.builder().clientConnector(connector).build() 
}
```

  - Create resources independent of global ones.

  - 创建独立于全局的资源。

  - Use the `ReactorClientHttpConnector` constructor with resource factory.

  - 使用`ReactorClientHttpConnector` 带有资源工厂参数的构造函数。

  - Plug the connector into the `WebClient.Builder`.

  - 把connector注入到`WebClient.Builder`。

### Timeouts

To configure a connection timeout:

配置连接超时时间：

**Java.**

``` java
import io.netty.channel.ChannelOption;

HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000);

WebClient webClient = WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
```

**Kotlin.**

``` kotlin
import io.netty.channel.ChannelOption

val httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000);

val webClient = WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
```

To configure a read or write timeout:

配置读写超时时间：

**Java.**

``` java
import io.netty.handler.timeout.ReadTimeoutHandler;
import io.netty.handler.timeout.WriteTimeoutHandler;

HttpClient httpClient = HttpClient.create()
        .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(10))
                .addHandlerLast(new WriteTimeoutHandler(10)));

// Create WebClient...
```

**Kotlin.**

``` kotlin
import io.netty.handler.timeout.ReadTimeoutHandler
import io.netty.handler.timeout.WriteTimeoutHandler

val httpClient = HttpClient.create()
        .doOnConnected { conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(10))
                .addHandlerLast(new WriteTimeoutHandler(10))
        }

// Create WebClient...
```

To configure a response timeout for all requests:

为所有请求配置响应超时时间：

**Java.**

``` java
HttpClient httpClient = HttpClient.create()
        .responseTimeout(Duration.ofSeconds(2));

// Create WebClient...
```

**Kotlin.**

``` kotlin
val httpClient = HttpClient.create()
        .responseTimeout(Duration.ofSeconds(2));

// Create WebClient...
```

To configure a response timeout for a specific request:

为单个请求配置响应超时时间：

**Java.**

``` java
WebClient.create().get()
        .uri("https://example.org/path")
        .httpRequest(httpRequest -> {
            HttpClientRequest reactorRequest = httpRequest.getNativeRequest();
            reactorRequest.responseTimeout(Duration.ofSeconds(2));
        })
        .retrieve()
        .bodyToMono(String.class);
```

**Kotlin.**

``` kotlin
WebClient.create().get()
        .uri("https://example.org/path")
        .httpRequest { httpRequest: ClientHttpRequest ->
            val reactorRequest = httpRequest.getNativeRequest<HttpClientRequest>()
            reactorRequest.responseTimeout(Duration.ofSeconds(2))
        }
        .retrieve()
        .bodyToMono(String::class.java)
```

## Jetty

The following example shows how to customize Jetty `HttpClient` settings:

以下示例展示了如何定制Jetty `HttpClient` 的配置：

**Java.**

``` java
HttpClient httpClient = new HttpClient();
httpClient.setCookieStore(...);

WebClient webClient = WebClient.builder()
        .clientConnector(new JettyClientHttpConnector(httpClient))
        .build();
```

**Kotlin.**

``` kotlin
val httpClient = HttpClient()
httpClient.cookieStore = ...

val webClient = WebClient.builder()
        .clientConnector(new JettyClientHttpConnector(httpClient))
        .build();
```

By default, `HttpClient` creates its own resources (`Executor`, `ByteBufferPool`, `Scheduler`), which remain active until the process exits or `stop()` is called.

默认情况下， `HttpClient` 创建它自己的资源 (`Executor`, `ByteBufferPool`, `Scheduler`)，直到进程推出或者 `stop()` 方法被调用才会销毁。

You can share resources between multiple instances of the Jetty client (and server) and ensure that the resources are shut down when the Spring `ApplicationContext` is closed by declaring a Spring-managed bean of type `JettyResourceFactory`, as the following example shows:

在多个 Jetty client (and server) 之间可以共享资源，并且确保在Spring `ApplicationContext`关闭的时候被关闭，可以通过定义一个受Spring管理的类型 `JettyResourceFactory`的Bean来实现，例如：

**Java.**

``` java
@Bean
public JettyResourceFactory resourceFactory() {
    return new JettyResourceFactory();
}

@Bean
public WebClient webClient() {

    HttpClient httpClient = new HttpClient();
    // Further customizations...

    ClientHttpConnector connector =
            new JettyClientHttpConnector(httpClient, resourceFactory()); 

    return WebClient.builder().clientConnector(connector).build(); 
}
```

  - Use the `JettyClientHttpConnector` constructor with resource factory.

  - Plug the connector into the `WebClient.Builder`.

**Kotlin.**

``` kotlin
@Bean
fun resourceFactory() = JettyResourceFactory()

@Bean
fun webClient(): WebClient {

    val httpClient = HttpClient()
    // Further customizations...

    val connector = JettyClientHttpConnector(httpClient, resourceFactory()) 

    return WebClient.builder().clientConnector(connector).build() 
}
```

  - Use the `JettyClientHttpConnector` constructor with resource factory.

  - Plug the connector into the `WebClient.Builder`.

## HttpComponents

The following example shows how to customize Apache HttpComponents `HttpClient` settings:

下面的例子展示了如何定制化Apache HttpComponents `HttpClient` 的配置：

**Java.**

``` java
HttpAsyncClientBuilder clientBuilder = HttpAsyncClients.custom();
clientBuilder.setDefaultRequestConfig(...);
CloseableHttpAsyncClient client = clientBuilder.build();
ClientHttpConnector connector = new HttpComponentsClientHttpConnector(client);

WebClient webClient = WebClient.builder().clientConnector(connector).build();
```

**Kotlin.**

``` kotlin
val client = HttpAsyncClients.custom().apply {
    setDefaultRequestConfig(...)
}.build()
val connector = HttpComponentsClientHttpConnector(client)
val webClient = WebClient.builder().clientConnector(connector).build()
```

# `retrieve()`

The `retrieve()` method can be used to declare how to extract the response. For example:

 `retrieve()` 方法可以用来声明如何获取响应。例如：

**Java.**

``` java
WebClient client = WebClient.create("https://example.org");

Mono<ResponseEntity<Person>> result = client.get()
        .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
        .retrieve()
        .toEntity(Person.class);
```

**Kotlin.**

``` kotlin
val client = WebClient.create("https://example.org")

val result = client.get()
        .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
        .retrieve()
        .toEntity<Person>().awaitSingle()
```

Or to get only the body:

或者仅仅获取响应体：

**Java.**

``` java
WebClient client = WebClient.create("https://example.org");

Mono<Person> result = client.get()
        .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
        .retrieve()
        .bodyToMono(Person.class);
```

**Kotlin.**

``` kotlin
val client = WebClient.create("https://example.org")

val result = client.get()
        .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
        .retrieve()
        .awaitBody<Person>()
```

To get a stream of decoded objects:

获取一个已经解码的对象流：

**Java.**

``` java
Flux<Quote> result = client.get()
        .uri("/quotes").accept(MediaType.TEXT_EVENT_STREAM)
        .retrieve()
        .bodyToFlux(Quote.class);
```

**Kotlin.**

``` kotlin
val result = client.get()
        .uri("/quotes").accept(MediaType.TEXT_EVENT_STREAM)
        .retrieve()
        .bodyToFlow<Quote>()
```

By default, 4xx or 5xx responses result in an `WebClientResponseException`, including sub-classes for specific HTTP status codes. To customize the handling of error responses, use `onStatus` handlers as follows:

默认情况下，4xx和5xx响应导致 `WebClientResponseException`及其子类。可以使用 `onStatus` 对异常响应的处理逻辑进行自定义，例如：


**Java.**

``` java
Mono<Person> result = client.get()
        .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
        .retrieve()
        .onStatus(HttpStatus::is4xxClientError, response -> ...)
        .onStatus(HttpStatus::is5xxServerError, response -> ...)
        .bodyToMono(Person.class);
```

**Kotlin.**

``` kotlin
val result = client.get()
        .uri("/persons/{id}", id).accept(MediaType.APPLICATION_JSON)
        .retrieve()
        .onStatus(HttpStatus::is4xxClientError) { ... }
        .onStatus(HttpStatus::is5xxServerError) { ... }
        .awaitBody<Person>()
```

# Exchange

The `exchangeToMono()` and `exchangeToFlux()` methods (or `awaitExchange { }` and `exchangeToFlow { }` in Kotlin) are useful for more advanced cases that require more control, such as to decode the response differently depending on the response status:

方法 `exchangeToMono()` 和 `exchangeToFlux()` 用来处理需要更多控制的用例，例如根据不同的响应状态来进行解码响应体：

**Java.**

``` java
Mono<Object> entityMono = client.get()
        .uri("/persons/1")
        .accept(MediaType.APPLICATION_JSON)
        .exchangeToMono(response -> {
            if (response.statusCode().equals(HttpStatus.OK)) {
                return response.bodyToMono(Person.class);
            }
            else if (response.statusCode().is4xxClientError()) {
                // Suppress error status code
                return response.bodyToMono(ErrorContainer.class);
            }
            else {
                // Turn to error
                return response.createException().flatMap(Mono::error);
            }
        });
```

**Kotlin.**

``` kotlin
val entity = client.get()
  .uri("/persons/1")
  .accept(MediaType.APPLICATION_JSON)
  .awaitExchange {
        if (response.statusCode() == HttpStatus.OK) {
             return response.awaitBody<Person>()
        }
        else if (response.statusCode().is4xxClientError) {
             return response.awaitBody<ErrorContainer>()
        }
        else {
             throw response.createExceptionAndAwait()
        }
  }
```

When using the above, after the returned `Mono` or `Flux` completes, the response body is checked and if not consumed it is released to prevent memory and connection leaks. Therefore the response cannot be decoded further downstream. It is up to the provided function to declare how to decode the response if needed.

当按照上述方式进行响应处理时，在返回`Mono` 或者 `Flux` 完成后，响应体会被检查，如果没有被消耗掉会自动进行释放以防止内存和连接泄露。因此响应不能再被下游进行解码处理。如果需要的话，必须由提供的函数进行响应体的解码操作。

# Request Body

# 请求体

The request body can be encoded from any asynchronous type handled by `ReactiveAdapterRegistry`, like `Mono` or Kotlin Coroutines `Deferred` as the following example shows:

请求体可以来自任何可以被 `ReactiveAdapterRegistry`处理的异步类型，例如 `Mono` 或者Kotlin协程 `Deferred` 类型，如下示例：

**Java.**

``` java
Mono<Person> personMono = ... ;

Mono<Void> result = client.post()
        .uri("/persons/{id}", id)
        .contentType(MediaType.APPLICATION_JSON)
        .body(personMono, Person.class)
        .retrieve()
        .bodyToMono(Void.class);
```

**Kotlin.**

``` kotlin
val personDeferred: Deferred<Person> = ...

client.post()
        .uri("/persons/{id}", id)
        .contentType(MediaType.APPLICATION_JSON)
        .body<Person>(personDeferred)
        .retrieve()
        .awaitBody<Unit>()
```

You can also have a stream of objects be encoded, as the following example shows:

你也可以使用一个对象流，如下所示：

**Java.**

``` java
Flux<Person> personFlux = ... ;

Mono<Void> result = client.post()
        .uri("/persons/{id}", id)
        .contentType(MediaType.APPLICATION_STREAM_JSON)
        .body(personFlux, Person.class)
        .retrieve()
        .bodyToMono(Void.class);
```

**Kotlin.**

``` kotlin
val people: Flow<Person> = ...

client.post()
        .uri("/persons/{id}", id)
        .contentType(MediaType.APPLICATION_JSON)
        .body(people)
        .retrieve()
        .awaitBody<Unit>()
```

Alternatively, if you have the actual value, you can use the `bodyValue` shortcut method, as the following example shows:

如果你也对象实际的值，也可以直接使用 `bodyValue` 方法，如下所示：

**Java.**

``` java
Person person = ... ;

Mono<Void> result = client.post()
        .uri("/persons/{id}", id)
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(person)
        .retrieve()
        .bodyToMono(Void.class);
```

**Kotlin.**

``` kotlin
val person: Person = ...

client.post()
        .uri("/persons/{id}", id)
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(person)
        .retrieve()
        .awaitBody<Unit>()
```

## Form Data

## 表单数据

To send form data, you can provide a `MultiValueMap<String, String>` as the body. Note that the content is automatically set to `application/x-www-form-urlencoded` by the `FormHttpMessageWriter`. The following example shows how to use `MultiValueMap<String, String>`:

为了发送表单数据，你可以提供一个 `MultiValueMap<String, String>` 对象作为请求体。注意内容类型会被 `FormHttpMessageWriter`自动设置为 `application/x-www-form-urlencoded` 。以下示例展示了如何使用 `MultiValueMap<String, String>`：

**Java.**

``` java
MultiValueMap<String, String> formData = ... ;

Mono<Void> result = client.post()
        .uri("/path", id)
        .bodyValue(formData)
        .retrieve()
        .bodyToMono(Void.class);
```

**Kotlin.**

``` kotlin
val formData: MultiValueMap<String, String> = ...

client.post()
        .uri("/path", id)
        .bodyValue(formData)
        .retrieve()
        .awaitBody<Unit>()
```

You can also supply form data in-line by using `BodyInserters`, as the following example shows:

你也可以使用 `BodyInserters`直接提供键值对，如下示例：

**Java.**

``` java
import static org.springframework.web.reactive.function.BodyInserters.*;

Mono<Void> result = client.post()
        .uri("/path", id)
        .body(fromFormData("k1", "v1").with("k2", "v2"))
        .retrieve()
        .bodyToMono(Void.class);
```

**Kotlin.**

``` kotlin
import org.springframework.web.reactive.function.BodyInserters.*

client.post()
        .uri("/path", id)
        .body(fromFormData("k1", "v1").with("k2", "v2"))
        .retrieve()
        .awaitBody<Unit>()
```

## Multipart Data

To send multipart data, you need to provide a `MultiValueMap<String, ?>` whose values are either `Object` instances that represent part content or `HttpEntity` instances that represent the content and headers for a part. `MultipartBodyBuilder` provides a convenient API to prepare a multipart request. The following example shows how to create a `MultiValueMap<String, ?>`:

要发送multipart数据，你需要提供一个 `MultiValueMap<String, ?>`，其值是表示part内容的 `Object` 实例或表示part内容和首部的 `HttpEntity` 实例。 `MultipartBodyBuilder` 提供了一个方便的 API 来准备multipart请求。 下面的例子展示了如何创建一个 `MultiValueMap<String, ?>`：


**Java.**

``` java
MultipartBodyBuilder builder = new MultipartBodyBuilder();
builder.part("fieldPart", "fieldValue");
builder.part("filePart1", new FileSystemResource("...logo.png"));
builder.part("jsonPart", new Person("Jason"));
builder.part("myPart", part); // Part from a server request

MultiValueMap<String, HttpEntity<?>> parts = builder.build();
```

**Kotlin.**

``` kotlin
val builder = MultipartBodyBuilder().apply {
    part("fieldPart", "fieldValue")
    part("filePart1", new FileSystemResource("...logo.png"))
    part("jsonPart", new Person("Jason"))
    part("myPart", part) // Part from a server request
}

val parts = builder.build()
```

In most cases, you do not have to specify the `Content-Type` for each part. The content type is determined automatically based on the `HttpMessageWriter` chosen to serialize it or, in the case of a `Resource`, based on the file extension. If necessary, you can explicitly provide the `MediaType` to use for each part through one of the overloaded builder `part` methods.

在大多数情况下，您不必为每个part指定 `Content-Type`。 内容类型是根据为序列化选择的 `HttpMessageWriter` 自动确定的，或者在 `Resource` 的情况下，根据文件扩展名自动确定。 如有必要，可以显示的通过重载的构建器 `part` 方法提供用于每个part的 `MediaType` 。

Once a `MultiValueMap` is prepared, the easiest way to pass it to the `WebClient` is through the `body` method, as the following example shows:

准备好“MultiValueMap”后，将其传递给 `WebClient` 的最简单方法是通过 `body` 方法，如下例所示：

**Java.**

``` java
MultipartBodyBuilder builder = ...;

Mono<Void> result = client.post()
        .uri("/path", id)
        .body(builder.build())
        .retrieve()
        .bodyToMono(Void.class);
```

**Kotlin.**

``` kotlin
val builder: MultipartBodyBuilder = ...

client.post()
        .uri("/path", id)
        .body(builder.build())
        .retrieve()
        .awaitBody<Unit>()
```

If the `MultiValueMap` contains at least one non-`String` value, which could also represent regular form data (that is, `application/x-www-form-urlencoded`), you need not set the `Content-Type` to `multipart/form-data`. This is always the case when using `MultipartBodyBuilder`, which ensures an `HttpEntity` wrapper.

As an alternative to `MultipartBodyBuilder`, you can also provide multipart content, inline-style, through the built-in `BodyInserters`, as the following example shows:

如果 `MultiValueMap` 包含至少一个非 `String` 值，该值也可以表示常规表单数据（即 `application/x-www-form-urlencoded`），则不需要设置 `Content-Type` 为 `multipart/form-data`。 使用 `MultipartBodyBuilder` 时总是如此，它确保使用了一个 `HttpEntity` 包装器。

作为 `MultipartBodyBuilder` 的替代方案，您还可以通过内置的 `BodyInserters` 提供内联样式的multipart内容，如下例所示：

**Java.**

``` java
import static org.springframework.web.reactive.function.BodyInserters.*;

Mono<Void> result = client.post()
        .uri("/path", id)
        .body(fromMultipartData("fieldPart", "value").with("filePart", resource))
        .retrieve()
        .bodyToMono(Void.class);
```

**Kotlin.**

``` kotlin
import org.springframework.web.reactive.function.BodyInserters.*

client.post()
        .uri("/path", id)
        .body(fromMultipartData("fieldPart", "value").with("filePart", resource))
        .retrieve()
        .awaitBody<Unit>()
```

# Filters

# 过滤器

You can register a client filter (`ExchangeFilterFunction`) through the `WebClient.Builder` in order to intercept and modify requests, as the following example shows:

可以通过 `WebClient.Builder` 注册一个客户端过滤器 (`ExchangeFilterFunction`) 来拦截和修改请求，如下例所示：

**Java.**

``` java
WebClient client = WebClient.builder()
        .filter((request, next) -> {

            ClientRequest filtered = ClientRequest.from(request)
                    .header("foo", "bar")
                    .build();

            return next.exchange(filtered);
        })
        .build();
```

**Kotlin.**

``` kotlin
val client = WebClient.builder()
        .filter { request, next ->

            val filtered = ClientRequest.from(request)
                    .header("foo", "bar")
                    .build()

            next.exchange(filtered)
        }
        .build()
```

This can be used for cross-cutting concerns, such as authentication. The following example uses a filter for basic authentication through a static factory method:

这可用于横切关注点，例如鉴权。 以下示例通过静态工厂方法使用过滤器进行基本身份验证：


**Java.**

``` java
import static org.springframework.web.reactive.function.client.ExchangeFilterFunctions.basicAuthentication;

WebClient client = WebClient.builder()
        .filter(basicAuthentication("user", "password"))
        .build();
```

**Kotlin.**

``` kotlin
import org.springframework.web.reactive.function.client.ExchangeFilterFunctions.basicAuthentication

val client = WebClient.builder()
        .filter(basicAuthentication("user", "password"))
        .build()
```

Filters can be added or removed by mutating an existing `WebClient` instance, resulting in a new `WebClient` instance that does not affect the original one. For example:

可以通过改变现有的 `WebClient` 实例来添加或删除过滤器，从而产生一个不影响原始实例的新 `WebClient` 实例。 例如：

**Java.**

``` java
import static org.springframework.web.reactive.function.client.ExchangeFilterFunctions.basicAuthentication;

WebClient client = webClient.mutate()
        .filters(filterList -> {
            filterList.add(0, basicAuthentication("user", "password"));
        })
        .build();
```

**Kotlin.**

``` kotlin
val client = webClient.mutate()
        .filters { it.add(0, basicAuthentication("user", "password")) }
        .build()
```

`WebClient` is a thin facade around the chain of filters followed by an `ExchangeFunction`. It provides a workflow to make requests, to encode to and from higher level objects, and it helps to ensure that response content is always consumed. When filters handle the response in some way, extra care must be taken to always consume its content or to otherwise propagate it downstream to the `WebClient` which will ensure the same. Below is a filter that handles the `UNAUTHORIZED` status code but ensures that any response content, whether expected or not, is released:

`WebClient` 是一个围绕过滤器链的薄门面，后跟一个 `ExchangeFunction`。 它提供了一个工作流来发出请求、编解码更高抽象级别的对象，并确保响应内容被消费掉。 当过滤器以某种方式处理响应时，必须格外小心始终确保其内容被消费掉或者将其向下游传递到 `WebClient` 。 下面是一个处理 `UNAUTHORIZED` 状态代码的过滤器，但确保任何响应内容，无论是否是预期的，都被释放：

**Java.**

``` java
public ExchangeFilterFunction renewTokenFilter() {
    return (request, next) -> next.exchange(request).flatMap(response -> {
        if (response.statusCode().value() == HttpStatus.UNAUTHORIZED.value()) {
            return response.releaseBody()
                    .then(renewToken())
                    .flatMap(token -> {
                        ClientRequest newRequest = ClientRequest.from(request).build();
                        return next.exchange(newRequest);
                    });
        } else {
            return Mono.just(response);
        }
    });
}
```

**Kotlin.**

``` kotlin
fun renewTokenFilter(): ExchangeFilterFunction? {
    return ExchangeFilterFunction { request: ClientRequest?, next: ExchangeFunction ->
        next.exchange(request!!).flatMap { response: ClientResponse ->
            if (response.statusCode().value() == HttpStatus.UNAUTHORIZED.value()) {
                return@flatMap response.releaseBody()
                        .then(renewToken())
                        .flatMap { token: String? ->
                            val newRequest = ClientRequest.from(request).build()
                            next.exchange(newRequest)
                        }
            } else {
                return@flatMap Mono.just(response)
            }
        }
    }
}
```

# Attributes

# 属性

You can add attributes to a request. This is convenient if you want to pass information through the filter chain and influence the behavior of filters for a given request. For example:

可以向请求添加属性。 如果你想通过过滤器链传递信息并影响给定请求的过滤器行为，这很方便。 例如：

**Java.**

``` java
WebClient client = WebClient.builder()
        .filter((request, next) -> {
            Optional<Object> usr = request.attribute("myAttribute");
            // ...
        })
        .build();

client.get().uri("https://example.org/")
        .attribute("myAttribute", "...")
        .retrieve()
        .bodyToMono(Void.class);

    }
```

**Kotlin.**

``` kotlin
val client = WebClient.builder()
        .filter { request, _ ->
            val usr = request.attributes()["myAttribute"];
            // ...
        }
        .build()

    client.get().uri("https://example.org/")
            .attribute("myAttribute", "...")
            .retrieve()
            .awaitBody<Unit>()
```

Note that you can configure a `defaultRequest` callback globally at the `WebClient.Builder` level which lets you insert attributes into all requests, which could be used for example in a Spring MVC application to populate request attributes based on `ThreadLocal` data.

请注意，你可以在 `WebClient.Builder` 级别全局配置 `defaultRequest` 回调，它允许您将属性插入到所有请求中，例如可以在 Spring MVC 应用程序中使用以根据 `ThreadLocal` 数据填充请求属性。

# Context

[Attributes](#webflux-client-attributes) provide a convenient way to pass information to the filter chain but they only influence the current request. If you want to pass information that propagates to additional requests that are nested, e.g. via `flatMap`, or executed after, e.g. via `concatMap`, then you’ll need to use the Reactor `Context`.

The Reactor `Context` needs to be populated at the end of a reactive chain in order to apply to all operations. For example:

[Attributes](#webflux-client-attributes) 提供了一种向过滤器链传递信息的便捷方式，但它们只影响当前请求。 如果您想传递传播到其他嵌套请求的信息，例如 通过`flatMap`，或之后执行，例如 通过`concatMap`，那么你需要使用Reactor`Context`。

Reactor `Context` 需要填充在反应链的末尾，以便应用于所有操作。 例如：

**Java.**

``` java
WebClient client = WebClient.builder()
        .filter((request, next) ->
                Mono.deferContextual(contextView -> {
                    String value = contextView.get("foo");
                    // ...
                }))
        .build();

client.get().uri("https://example.org/")
        .retrieve()
        .bodyToMono(String.class)
        .flatMap(body -> {
                // perform nested request (context propagates automatically)...
        })
        .contextWrite(context -> context.put("foo", ...));
```

# Synchronous Use

# 同步使用

`WebClient` can be used in synchronous style by blocking at the end for the result:

`WebClient` 可以通过在结尾处阻塞在同步风格的代码中使用：

**Java.**

``` java
Person person = client.get().uri("/person/{id}", i).retrieve()
    .bodyToMono(Person.class)
    .block();

List<Person> persons = client.get().uri("/persons").retrieve()
    .bodyToFlux(Person.class)
    .collectList()
    .block();
```

**Kotlin.**

``` kotlin
val person = runBlocking {
    client.get().uri("/person/{id}", i).retrieve()
            .awaitBody<Person>()
}

val persons = runBlocking {
    client.get().uri("/persons").retrieve()
            .bodyToFlow<Person>()
            .toList()
}
```

However if multiple calls need to be made, it’s more efficient to avoid blocking on each response individually, and instead wait for the combined result:

但是，如果需要进行多次调用，避免单独阻塞每个响应，而是等待组合结果更有效：

**Java.**

``` java
Mono<Person> personMono = client.get().uri("/person/{id}", personId)
        .retrieve().bodyToMono(Person.class);

Mono<List<Hobby>> hobbiesMono = client.get().uri("/person/{id}/hobbies", personId)
        .retrieve().bodyToFlux(Hobby.class).collectList();

Map<String, Object> data = Mono.zip(personMono, hobbiesMono, (person, hobbies) -> {
            Map<String, String> map = new LinkedHashMap<>();
            map.put("person", person);
            map.put("hobbies", hobbies);
            return map;
        })
        .block();
```

**Kotlin.**

``` kotlin
val data = runBlocking {
        val personDeferred = async {
            client.get().uri("/person/{id}", personId)
                    .retrieve().awaitBody<Person>()
        }

        val hobbiesDeferred = async {
            client.get().uri("/person/{id}/hobbies", personId)
                    .retrieve().bodyToFlow<Hobby>().toList()
        }

        mapOf("person" to personDeferred.await(), "hobbies" to hobbiesDeferred.await())
    }
```

The above is merely one example. There are lots of other patterns and operators for putting together a reactive pipeline that makes many remote calls, potentially some nested, inter-dependent, without ever blocking until the end.

以上只是一个例子。 有许多其他模式和操作符可以组合一个反应式管道，这些管道可以进行许多远程调用，可能是一些嵌套的、相互依赖的，直到最后都不会阻塞。

> **Note**
> 
> With `Flux` or `Mono`, you should never have to block in a Spring MVC or Spring WebFlux controller. Simply return the resulting reactive type from the controller method. The same principle apply to Kotlin Coroutines and Spring WebFlux, just use suspending function or return `Flow` in your controller method .

> **注意**
>
> 使用 `Flux` 或 `Mono`，你永远不必在 Spring MVC 或 Spring WebFlux 控制器中阻塞。 简单地从控制器方法返回结果反应类型。 相同的原则适用于 Kotlin Coroutines 和 Spring WebFlux，只需在控制器方法中使用挂起函数或返回 `Flow`。 

# Testing

To test code that uses the `WebClient`, you can use a mock web server, such as the [OkHttp MockWebServer](https://github.com/square/okhttp#mockwebserver). To see an example of its use, check out {spring-framework-main-code}/spring-webflux/src/test/java/org/springframework/web/reactive/function/client/WebClientIntegrationTests.java\[`WebClientIntegrationTests`\] in the Spring Framework test suite or the [`static-server`](https://github.com/square/okhttp/tree/master/samples/static-server) sample in the OkHttp repository.

# 测试

要测试使用 `WebClient` 的代码，您可以使用模拟 Web 服务器，例如 [OkHttp MockWebServer](https://github.com/square/okhttp#mockwebserver)。 要查看其使用示例，请查看 {spring-framework-main-code}/spring-webflux/src/test/java/org/springframework/web/reactive/function/client/WebClientIntegrationTests.java\[`WebClientIntegrationTests` \] 在 Spring Framework 测试套件或 OkHttp 存储库中的 [`static-server`](https://github.com/square/okhttp/tree/master/samples/static-server) 示例中。 