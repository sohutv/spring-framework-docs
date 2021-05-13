Spring WebFlux includes a client to perform HTTP requests with. `WebClient` has a functional, fluent API based on Reactor, see [web-reactive.xml](web-reactive.xml#webflux-reactive-libraries), which enables declarative composition of asynchronous logic without the need to deal with threads or concurrency. It is fully non-blocking, it supports streaming, and relies on the same [codecs](web-reactive.xml#webflux-codecs) that are also used to encode and decode request and response content on the server side.

`WebClient` needs an HTTP client library to perform requests with. There is built-in support for the following:

  - [Reactor Netty](https://github.com/reactor/reactor-netty)

  - [Jetty Reactive HttpClient](https://github.com/jetty-project/jetty-reactive-httpclient)

  - [Apache HttpComponents](https://hc.apache.org/index.html)

  - Others can be plugged via `ClientHttpConnector`.

# Configuration

The simplest way to create a `WebClient` is through one of the static factory methods:

  - `WebClient.create()`

  - `WebClient.create(String baseUrl)`

You can also use `WebClient.builder()` with further options:

  - `uriBuilderFactory`: Customized `UriBuilderFactory` to use as a base URL.

  - `defaultUriVariables`: default values to use when expanding URI templates.

  - `defaultHeader`: Headers for every request.

  - `defaultCookie`: Cookies for every request.

  - `defaultRequest`: `Consumer` to customize every request.

  - `filter`: Client filter for every request.

  - `exchangeStrategies`: HTTP message reader/writer customizations.

  - `clientConnector`: HTTP client library settings.

For example:

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

    org.springframework.core.io.buffer.DataBufferLimitException: Exceeded limit on max bytes to buffer

To change the limit for default codecs, use the following:

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

If the server is timed with the process, there is typically no need for an explicit shutdown. However, if the server can start or stop in-process (for example, a Spring MVC application deployed as a WAR), you can declare a Spring-managed bean of type `ReactorResourceFactory` with `globalResources=true` (the default) to ensure that the Reactor Netty global resources are shut down when the Spring `ApplicationContext` is closed, as the following example shows:

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

  - Use the `ReactorClientHttpConnector` constructor with resource factory.

  - Plug the connector into the `WebClient.Builder`.

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

  - Use the `ReactorClientHttpConnector` constructor with resource factory.

  - Plug the connector into the `WebClient.Builder`.

### Timeouts

To configure a connection timeout:

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

You can share resources between multiple instances of the Jetty client (and server) and ensure that the resources are shut down when the Spring `ApplicationContext` is closed by declaring a Spring-managed bean of type `JettyResourceFactory`, as the following example shows:

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

# Request Body

The request body can be encoded from any asynchronous type handled by `ReactiveAdapterRegistry`, like `Mono` or Kotlin Coroutines `Deferred` as the following example shows:

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

To send form data, you can provide a `MultiValueMap<String, String>` as the body. Note that the content is automatically set to `application/x-www-form-urlencoded` by the `FormHttpMessageWriter`. The following example shows how to use `MultiValueMap<String, String>`:

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

Once a `MultiValueMap` is prepared, the easiest way to pass it to the `WebClient` is through the `body` method, as the following example shows:

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

You can register a client filter (`ExchangeFilterFunction`) through the `WebClient.Builder` in order to intercept and modify requests, as the following example shows:

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

You can add attributes to a request. This is convenient if you want to pass information through the filter chain and influence the behavior of filters for a given request. For example:

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

# Context

[section\_title](#webflux-client-attributes) provide a convenient way to pass information to the filter chain but they only influence the current request. If you want to pass information that propagates to additional requests that are nested, e.g. via `flatMap`, or executed after, e.g. via `concatMap`, then you’ll need to use the Reactor `Context`.

The Reactor `Context` needs to be populated at the end of a reactive chain in order to apply to all operations. For example:

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

`WebClient` can be used in synchronous style by blocking at the end for the result:

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

> **Note**
> 
> With `Flux` or `Mono`, you should never have to block in a Spring MVC or Spring WebFlux controller. Simply return the resulting reactive type from the controller method. The same principle apply to Kotlin Coroutines and Spring WebFlux, just use suspending function or return `Flow` in your controller method .

# Testing

To test code that uses the `WebClient`, you can use a mock web server, such as the [OkHttp MockWebServer](https://github.com/square/okhttp#mockwebserver). To see an example of its use, check out {spring-framework-main-code}/spring-webflux/src/test/java/org/springframework/web/reactive/function/client/WebClientIntegrationTests.java\[`WebClientIntegrationTests`\] in the Spring Framework test suite or the [`static-server`](https://github.com/square/okhttp/tree/master/samples/static-server) sample in the OkHttp repository.
