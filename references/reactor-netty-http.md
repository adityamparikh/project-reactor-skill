# WebClient and Reactor Netty HTTP

## Setup

```java
@Bean WebClient webClient() {
    return WebClient.builder()
        .baseUrl("https://api.example.com")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .codecs(c -> c.defaultCodecs().maxInMemorySize(10 * 1024 * 1024))   // 10MB
        .build();
}
```

## Requests

```java
// GET + error mapping
Mono<User> getUser(String id) {
    return webClient.get().uri("/users/{id}", id)
        .retrieve()
        .onStatus(s -> s.value() == 404,
                  r -> Mono.error(new UserNotFoundException(id)))
        .onStatus(HttpStatusCode::is5xxServerError,
                  r -> r.bodyToMono(ErrorBody.class)
                        .flatMap(err -> Mono.error(new ServerException(err))))
        .bodyToMono(User.class);
}

// POST
Mono<Order> createOrder(OrderRequest req) {
    return webClient.post().uri("/orders").bodyValue(req)
        .retrieve().bodyToMono(Order.class);
}

// Streaming SSE
Flux<Event> streamEvents(String topic) {
    return webClient.get().uri("/events/stream/{topic}", topic)
        .accept(MediaType.TEXT_EVENT_STREAM)
        .retrieve().bodyToFlux(Event.class);
}
```

## Timeout + Retry with Backoff

```java
Mono<User> getWithRetry(String id) {
    return webClient.get().uri("/users/{id}", id)
        .retrieve().bodyToMono(User.class)
        .timeout(Duration.ofSeconds(5))
        .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
            .maxBackoff(Duration.ofSeconds(2))
            .jitter(0.5)
            .filter(ex -> ex instanceof WebClientRequestException
                       || ex instanceof TimeoutException)
            .onRetryExhaustedThrow((spec, signal) ->
                new ServiceUnavailableException("Exhausted retries")));
}
```

## Connection Pool Tuning

```java
@Bean WebClient webClient() {
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
HttpClient httpClient = HttpClient.create().metrics(true, Function.identity());

// Includes:
// reactor.netty.http.client.connect.time
// reactor.netty.http.client.data.sent / received
// reactor.netty.http.client.response.time
```

## Large Payloads / Streaming Bytes

```java
WebClient webClient = WebClient.builder()
    .exchangeStrategies(ExchangeStrategies.builder()
        .codecs(c -> c.defaultCodecs().maxInMemorySize(50 * 1024 * 1024))
        .build())
    .build();

// Stream raw bytes without buffering
webClient.get().uri("/large-file")
    .retrieve().bodyToFlux(DataBuffer.class)
    .map(DataBufferUtils::retain)
    .doOnNext(buf -> writeToFile(buf))
    .doOnNext(DataBufferUtils::release);
```

## Key Rules

- Always set `responseTimeout` on `HttpClient` **and** `.timeout()` on the `Mono` — former cuts TCP read, latter cuts reactive chain.
- Prefer `onStatus()` over `retrieve().toEntity()` for error mapping — keeps body decoding lazy, avoids connection leaks.
- Release `DataBuffer` (`DataBufferUtils::release`) when streaming raw bytes; otherwise Netty's pooled allocator leaks.
- Default `maxConnections` is `max(500, 2 * availableProcessors())` — tune explicitly for services with many downstream targets.
- `jitter(0.5)` spreads retry thundering herds; omitting it causes synchronized retry storms.
