# 4. Testing
[Same in Spring MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#testing)

spring-test 模块提供了 `ServerHttpRequest`、`ServerHttpResponse` 和 `ServerWebExchange` 的模拟实现。 有关模拟对象的讨论，请参阅 [Spring Web Reactive](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#mock-objects-web-reactive)。

[`WebTestClient`](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#webtestclient) 建立在这些模拟请求和响应对象之上，为测试 WebFlux 应用程序提供支持，而无需 一个 HTTP 服务器。 你也可以使用“WebTestClient”进行端到端集成测试。
