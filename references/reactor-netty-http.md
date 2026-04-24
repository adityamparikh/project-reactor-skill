# WebClient and Reactor Netty HTTP Patterns

## WebClient Setup

```java
@Bean
WebClient webClient() {
    return WebClient.builder()
        .baseUrl("https://api.example.com")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .codecs(codecs -> codecs.defaultCodecs().maxInMemorySize(10 * 1024 * 1024)) // 10MB
        .build();
}
```

## Request Patterns

```java
// GET with path variable and error handling
Mono<User> getUser(String id) {
    return webClient.get()
        .uri("/users/{id}", id)
        .retrieve()
        .onStatus(status -> status.value() == 404,
                  resp -> Mono.error(new UserNotFoundException(id)))
        .onStatus(HttpStatusCode::is5xxServerError,
                  resp -> resp.bodyToMono(ErrorBody.class)
                              .flatMap(err -> Mono.error(new ServerException(err))))
        .bodyToMono(User.class);
}

// POST with body
Mono<Order> createOrder(OrderRequest req) {
    return webClient.post()
        .uri("/orders")
        .bodyValue(req)
        .retrieve()
        .bodyToMono(Order.class);
}

// Streaming response
Flux<Event> streamEvents(String topic) {
    return webClient.get()
        .uri("/events/stream/{topic}", topic)
        .accept(MediaType.TEXT_EVENT_STREAM)
        .retrieve()
        .bodyToFlux(Event.class);
}
```

## Timeout + Retry with Backoff

```java
Mono<User> getWithRetry(String id) {
    return webClient.get().uri("/users/{id}", id)
        .retrieve()
        .bodyToMono(User.class)
        .timeout(Duration.ofSeconds(5))
        .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
            .maxBackoff(Duration.ofSeconds(2))
            .jitter(0.5)
            .filter(ex -> ex instanceof WebClientRequestException  // network errors
                       || ex instanceof TimeoutException)
            .onRetryExhaustedThrow((spec, signal) ->
                new ServiceUnavailableException("Exhausted retries")));
}
```

## Connection Pool Tuning

```java
@Bean
WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
        .responseTimeout(Duration.ofSeconds(30))
        .doOnConnected(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(30, TimeUnit.SECONDS))
            .addHandlerLast(new WriteTimeoutHandler(30, TimeUnit.SECONDS)))
        .connectionProvider(ConnectionProvider.builder("custom-pool")
            .maxConnections(200)
            .maxIdleTime(Duration.ofSeconds(60))
            .maxLifeTime(Duration.ofMinutes(5))
            .pendingAcquireTimeout(Duration.ofSeconds(10))
            .evictInBackground(Duration.ofSeconds(30))
            .build());

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}
```

## Metrics

```java
// Reactor Netty metrics (Micrometer)
HttpClient httpClient = HttpClient.create()
    .metrics(true, Function.identity()); // enable default metrics

// Metrics include:
// reactor.netty.http.client.connect.time
// reactor.netty.http.client.data.sent
// reactor.netty.http.client.data.received
// reactor.netty.http.client.response.time
```

## ExchangeStrategies for Large Payloads

```java
WebClient webClient = WebClient.builder()
    .exchangeStrategies(ExchangeStrategies.builder()
        .codecs(c -> c.defaultCodecs().maxInMemorySize(50 * 1024 * 1024)) // 50MB
        .build())
    .build();

// For streaming large files without buffering:
webClient.get().uri("/large-file")
    .retrieve()
    .bodyToFlux(DataBuffer.class)    // stream bytes
    .map(DataBufferUtils::retain)
    .doOnNext(buf -> writeToFile(buf))
    .doOnNext(DataBufferUtils::release);
```

## Key Rules

- Always set `responseTimeout` on `HttpClient` in addition to `.timeout()` on the `Mono`; the former cuts the TCP-level read, the latter cuts the reactive chain.
- Prefer `onStatus()` over `retrieve().toEntity()` for error mapping — it keeps response body decoding lazy and avoids leaking connections.
- Release `DataBuffer` instances manually (via `DataBufferUtils::release`) when streaming raw bytes; failure to do so causes memory leaks under Netty's pooled allocator.
- The default `maxConnections` is `max(500, 2 * Runtime.getRuntime().availableProcessors())`; tune it explicitly for services with many downstream targets.
- `jitter(0.5)` on backoff spreads retry thundering herds; omitting it can cause synchronized retry storms under high concurrency.
