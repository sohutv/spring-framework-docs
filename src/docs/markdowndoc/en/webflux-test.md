# 4. Testing
[Same in Spring MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#testing)

The spring-test module provides mock implementations of `ServerHttpRequest`, `ServerHttpResponse`, and `ServerWebExchange`. See [Spring Web Reactive](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#mock-objects-web-reactive) for a discussion of mock objects.

[`WebTestClient`](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#webtestclient) builds on these mock request and response objects to provide support for testing WebFlux applications without an HTTP server. You can use the `WebTestClient` for end-to-end integration tests, too.