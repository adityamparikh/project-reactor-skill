# Virtual Threads vs. Project Reactor: Decision Guide

Both Project Reactor and virtual threads (introduced in Java 21 LTS, refined
through Java 25 LTS) solve concurrent I/O. The right choice depends on your
context.

> Java baseline for this skill: minimum **Java 17**, recommended **Java 25 LTS**.
> Supported versions: 17, 21, 25. Virtual threads require Java 21+; new code
> should target Java 25 LTS where possible.

## Decision Tree

```
Is this new greenfield code or migration?
├── Migration from existing blocking code?
│   ├── Team knows reactive? → Migrate to Reactor (for full benefits)
│   └── Team is imperative-first? → Virtual threads (lower risk; Java 21+, prefer Java 25 LTS)
│
├── New greenfield code (target Java 25 LTS where possible)
│   ├── Needs streaming / SSE / WebSocket?         → Reactor (Flux)
│   ├── Needs fine-grained backpressure control?   → Reactor
│   ├── Complex async fan-out with composition?    → Reactor
│   ├── Simple CRUD with blocking DB / HTTP?       → Virtual threads
│   ├── Batch processing, file I/O?                → Virtual threads
│   └── Event-driven with Kafka/messaging?         → Reactor (or both)
│
└── Stuck on Java 17 (oldest supported) or older?
    └── Must use Reactor (or other reactive lib) for async I/O — virtual threads
        are unavailable until Java 21
```

## When Reactor Is the Right Tool

- **Streaming data** to clients (SSE, WebSocket, server-sent events, chunked responses)
- **Fan-out with backpressure**: calling N downstream services with rate limiting
- **Hot publishers**: shared event streams, reactive Kafka, reactive messaging
- **Complex operator composition**: multiple sources combined, filtered, transformed with rich operator set
- **Existing reactive stack**: Spring WebFlux already in use, R2DBC for DB, reactive security
- **Fine-grained control over concurrency**: `flatMap(fn, maxConcurrency)` for exact throttling

## When Virtual Threads Are Simpler

- **Simple request-response CRUD**: read from DB, transform, return — no streaming needed
- **Java 21+ available** (Java 25 LTS preferred) and team finds reactive model hard to reason about
- **Existing blocking code** that works correctly — virtual threads make it scale without a rewrite
- **Testing**: blocking/imperative tests are simpler than `StepVerifier`
- **Debugging**: stack traces make sense; no assembly trace complexity
- **Library compatibility**: some libraries don't play well with reactive; all work with virtual threads

## Hybrid Approaches

**"Reactive shell, sync core"** — WebFlux at HTTP layer, virtual threads for business logic:
```java
// WebFlux controller
@GetMapping("/users/{id}")
Mono<User> getUser(@PathVariable String id) {
    return Mono.fromCallable(() -> userService.findById(id))  // sync service
               .subscribeOn(Schedulers.boundedElastic());     // or virtual thread executor (Java 21+)
}
```

**"Virtual threads everywhere, Reactor for streaming"** (Java 21+; Java 25 LTS recommended):
```yaml
# Spring Boot 3.2+
spring.threads.virtual.enabled: true  # enables virtual threads for Tomcat/Jetty
```
```java
// Use WebFlux only for streaming endpoints; MVC + virtual threads for the rest
@GetMapping(value = "/events/stream", produces = TEXT_EVENT_STREAM_VALUE)
Flux<Event> stream() {
    return eventService.stream();  // only this endpoint uses Flux
}
```

## Migration Considerations

**From blocking to Reactor:**
1. Start at the infrastructure layer (HTTP client → WebClient, JDBC → R2DBC)
2. Wrap remaining blocking calls with `Mono.fromCallable + subscribeOn(boundedElastic)`
3. Migrate inward (services, repositories) gradually
4. Use BlockHound to detect remaining blocking calls in reactive chains

**From Reactor to virtual threads (requires Java 21+; Java 25 LTS recommended):**
1. Replace `WebClient` with `RestClient` or `HttpClient` (`java.net.http`, available since Java 11)
2. Replace R2DBC with JDBC/JPA — simpler, more tooling
3. Replace `StepVerifier` tests with standard JUnit assertions
4. Keep WebFlux for SSE/streaming if needed

## Performance Reality Check

- Virtual threads scale well for request-response (thousands of concurrent blocking calls)
- Reactor has lower overhead per-operation (no context switching, no thread stack)
- For pure throughput with I/O-bound work: both are competitive at scale
- Reactor wins when: streaming, backpressure, complex async composition
- Virtual threads win when: developer productivity, simpler debugging, library compatibility

## Spring Boot Context

| Spring Boot 3.x Feature | Reactor             | Virtual Threads (Java 21+, prefer Java 25 LTS) |
|--------------------------|---------------------|-------------------------|
| Web layer                | Spring WebFlux      | Spring MVC + virtual threads |
| DB access                | R2DBC               | JDBC/JPA                |
| HTTP client              | WebClient           | RestClient              |
| Security                 | Spring Security Reactive | Spring Security (standard) |
| Testing                  | StepVerifier, WebTestClient | MockMvc, standard JUnit |
| Streaming/SSE            | Native Flux         | Possible but awkward    |
