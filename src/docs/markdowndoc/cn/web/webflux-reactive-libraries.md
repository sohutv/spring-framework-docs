`spring-webflux` depends on `reactor-core` and uses it internally to compose asynchronous logic and to provide Reactive Streams support. Generally, WebFlux APIs return `Flux` or `Mono` (since those are used internally) and leniently accept any Reactive Streams `Publisher` implementation as input. The use of `Flux` versus `Mono` is important, because it helps to express cardinality — for example, whether a single or multiple asynchronous values are expected, and that can be essential for making decisions (for example, when encoding or decoding HTTP messages).

For annotated controllers, WebFlux transparently adapts to the reactive library chosen by the application. This is done with the help of the [`ReactiveAdapterRegistry`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/core/ReactiveAdapterRegistry.html), which provides pluggable support for reactive library and other asynchronous types. The registry has built-in support for RxJava 2/3, RxJava 1 (via RxJava Reactive Streams bridge), and `CompletableFuture`, but you can register others, too.

As of Spring Framework 5.3, support for RxJava 1 is deprecated.

For functional APIs (such as [Functional Endpoints](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-fn), the `WebClient`, and others), the general rules for WebFlux APIs apply — `Flux` and `Mono` as return values and a Reactive Streams `Publisher` as input. When a `Publisher`, whether custom or from another reactive library, is provided, it can be treated only as a stream with unknown semantics (0..N). If, however, the semantics are known, you can wrap it with Flux or Mono.from(`Publisher`) instead of passing the raw `Publisher`.

For example, given a `Publisher` that is not a `Mono`, the Jackson JSON message writer expects multiple values. If the media type implies an infinite stream (for example, `application/json+stream`), values are written and flushed individually. Otherwise, values are buffered into a list and rendered as a JSON array.

`spring-webflux` 依赖于 `reactor-core` 并在内部使用它来组合异步逻辑并提供响应式流支持。通常，WebFlux API 返回 `Flux` 或 `Mono`（因为它们在内部使用）并可以接受任何 Reactive Streams `Publisher` 实现作为输入。 `Flux` 和 `Mono` 的使用很重要，因为它有助于表达基数——例如，是否需要单个或多个异步值，这对于做出决策至关重要（例如，在编码或解码 HTTP 时）。

对于带注解的控制器，WebFlux 透明地适应应用程序选择的反应库。这是在 [`ReactiveAdapterRegistry`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/core/ReactiveAdapterRegistry.html) 的帮助下完成的，它为反应式库和其他异步类型提供可插拔支持。注册中心内置了对 RxJava 2/3、RxJava 1（通过 RxJava Reactive Streams 桥接器）和“CompletableFuture”的支持，但你也可以注册其他的。

从 Spring Framework 5.3 开始，不再对 RxJava 1 提供支持。

对于函数式 API（例如 [Functional Endpoints](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-fn)，`WebClient`和其他），WebFlux API 的一般规则适用 — `Flux` 和 `Mono` 作为返回值，Reactive Streams `Publisher` 作为输入。当提供了一个 `Publisher`，无论是自定义的还是来自另一个反应式库的，它只能被视为具有未知语义 (0..N) 的流。但是，如果语义是已知的，您可以使用 Flux 或 Mono.from(`Publisher`) 包装它，而不是传递原始的 `Publisher`。

例如，给定一个不是 `Mono` 的 `Publisher`，Jackson JSON 消息编写器需要多个值。如果媒体类型意味着无限流（例如，`application/json+stream`），则单独写入和刷新值。否则，值将缓冲到列表中并呈现为 JSON 数组。