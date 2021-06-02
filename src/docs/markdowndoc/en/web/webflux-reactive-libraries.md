`spring-webflux` depends on `reactor-core` and uses it internally to compose asynchronous logic and to provide Reactive Streams support. Generally, WebFlux APIs return `Flux` or `Mono` (since those are used internally) and leniently accept any Reactive Streams `Publisher` implementation as input. The use of `Flux` versus `Mono` is important, because it helps to express cardinality — for example, whether a single or multiple asynchronous values are expected, and that can be essential for making decisions (for example, when encoding or decoding HTTP messages).

For annotated controllers, WebFlux transparently adapts to the reactive library chosen by the application. This is done with the help of the [`ReactiveAdapterRegistry`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/core/ReactiveAdapterRegistry.html), which provides pluggable support for reactive library and other asynchronous types. The registry has built-in support for RxJava 2/3, RxJava 1 (via RxJava Reactive Streams bridge), and `CompletableFuture`, but you can register others, too.

As of Spring Framework 5.3, support for RxJava 1 is deprecated.

For functional APIs (such as [Functional Endpoints](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-fn), the `WebClient`, and others), the general rules for WebFlux APIs apply — `Flux` and `Mono` as return values and a Reactive Streams `Publisher` as input. When a `Publisher`, whether custom or from another reactive library, is provided, it can be treated only as a stream with unknown semantics (0..N). If, however, the semantics are known, you can wrap it with Flux or Mono.from(`Publisher`) instead of passing the raw `Publisher`.

For example, given a `Publisher` that is not a `Mono`, the Jackson JSON message writer expects multiple values. If the media type implies an infinite stream (for example, `application/json+stream`), values are written and flushed individually. Otherwise, values are buffered into a list and rendered as a JSON array.