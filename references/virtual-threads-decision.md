# Virtual Threads vs. Project Reactor: Decision Guide

Both solve concurrent I/O. Right choice depends on context.

## Decision Tree

```
Greenfield or migration?
├── Migration from blocking?
│   ├── Team knows reactive? → Migrate to Reactor (full benefits)
│   └── Imperative-first?    → Virtual threads (lower risk)
│
├── Greenfield
│   ├── Streaming / SSE / WebSocket?              → Reactor (Flux)
│   ├── Fine-grained backpressure?                → Reactor
│   ├── Complex async fan-out with composition?   → Reactor
│   ├── Simple CRUD with blocking DB/HTTP?        → Virtual threads
│   ├── Batch / file I/O?                         → Virtual threads
│   └── Event-driven Kafka/messaging?             → Reactor (or both)
│
└── JDK < 21? → Must use Reactor (or other reactive lib)
```

## When Reactor Is Right

- **Streaming** (SSE, WebSocket, chunked responses)
- **Fan-out with backpressure**: N downstream services with rate limiting
- **Hot publishers**: shared event streams, reactive Kafka, reactive messaging
- **Complex operator composition**: multiple sources combined, filtered, transformed
- **Existing reactive stack**: WebFlux, R2DBC, reactive security in use
- **Fine-grained concurrency control**: `flatMap(fn, maxConcurrency)` for exact throttling

## When Virtual Threads Are Simpler

- **Simple request-response CRUD** — no streaming
- **JDK 21+** and team finds reactive hard to reason about
- **Existing blocking code that works** — virtual threads scale it without rewrite
- **Testing**: blocking tests simpler than `StepVerifier`
- **Debugging**: stack traces make sense; no assembly trace complexity
- **Library compatibility**: all libraries work with virtual threads

## Hybrid Approaches

**Reactive shell, sync core** — WebFlux at HTTP, virtual threads for business logic:
```java
@GetMapping("/users/{id}")
Mono<User> getUser(@PathVariable String id) {
    return Mono.fromCallable(() -> userService.findById(id))
               .subscribeOn(Schedulers.boundedElastic());   // or virtual thread executor
}
```

**Virtual threads everywhere, Reactor for streaming:**
```yaml
spring.threads.virtual.enabled: true  # MVC on virtual threads (Boot 3.2+)
```
```java
// Use WebFlux only for streaming endpoints; MVC + virtual threads for the rest
@GetMapping(value = "/events/stream", produces = TEXT_EVENT_STREAM_VALUE)
Flux<Event> stream() { return eventService.stream(); }
```

## Migration Notes

**Blocking → Reactor:**
1. Start at infrastructure (HTTP client → WebClient, JDBC → R2DBC)
2. Wrap remaining blocking with `Mono.fromCallable + subscribeOn(boundedElastic)`
3. Migrate inward (services, repos) gradually
4. Use BlockHound to detect remaining blocking calls

**Reactor → Virtual threads:**
1. `WebClient` → `RestClient` or JDK `HttpClient`
2. R2DBC → JDBC/JPA
3. `StepVerifier` → standard JUnit
4. Keep WebFlux for SSE/streaming if needed

## Performance Reality Check

- Virtual threads scale well for request-response (thousands of concurrent blocking calls)
- Reactor has lower per-operation overhead (no context switching, no thread stack)
- For pure throughput on I/O-bound work: both competitive at scale
- Reactor wins: streaming, backpressure, complex async composition
- Virtual threads win: developer productivity, simpler debugging, library compatibility

## Spring Boot 3.x Matrix

| Layer | Reactor | Virtual Threads |
|---|---|---|
| Web | Spring WebFlux | Spring MVC + virtual threads |
| DB | R2DBC | JDBC/JPA |
| HTTP client | WebClient | RestClient |
| Security | Spring Security Reactive | Spring Security (standard) |
| Testing | StepVerifier, WebTestClient | MockMvc, JUnit |
| Streaming/SSE | Native Flux | Possible but awkward |
