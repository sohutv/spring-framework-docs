`spring-webflux` 依赖于 `reactor-core` 并在内部使用它来组合异步逻辑并提供响应式流支持。通常，WebFlux API 返回 `Flux` 或 `Mono`（因为它们在内部使用）并可以接受任何 Reactive Streams `Publisher` 实现作为输入。 `Flux` 和 `Mono` 的使用很重要，因为它有助于表达基数——例如，是否需要单个或多个异步值，这对于做出决策至关重要（例如，在编码或解码 HTTP 时）。

对于带注解的控制器，WebFlux 透明地适应应用程序选择的反应库。这是在 [`ReactiveAdapterRegistry`](https://docs.spring.io/spring-framework/docs/5.3.7/javadoc-api/org/springframework/core/ReactiveAdapterRegistry.html) 的帮助下完成的，它为反应式库和其他异步类型提供可插拔支持。注册中心内置了对 RxJava 2/3、RxJava 1（通过 RxJava Reactive Streams 桥接器）和“CompletableFuture”的支持，但你也可以注册其他的。

从 Spring Framework 5.3 开始，不再对 RxJava 1 提供支持。

对于函数式 API（例如 [Functional Endpoints](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-fn)，`WebClient`和其他），WebFlux API 的一般规则适用 — `Flux` 和 `Mono` 作为返回值，Reactive Streams `Publisher` 作为输入。当提供了一个 `Publisher`，无论是自定义的还是来自另一个反应式库的，它只能被视为具有未知语义 (0..N) 的流。但是，如果语义是已知的，您可以使用 Flux 或 Mono.from(`Publisher`) 包装它，而不是传递原始的 `Publisher`。

例如，给定一个不是 `Mono` 的 `Publisher`，Jackson JSON 消息编写器需要多个值。如果媒体类型意味着无限流（例如，`application/json+stream`），则单独写入和刷新值。否则，值将缓冲到列表中并呈现为 JSON 数组。
