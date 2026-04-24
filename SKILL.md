---
name: project-reactor
description: |
  Expert guide for Project Reactor (reactor-core 3.6+, BOM 2024.0+). Use this skill
  whenever the user is writing, reviewing, debugging, or reasoning about reactive code
  involving Mono, Flux, Publisher, Subscriber, or reactive pipelines. Triggers on:
  operator selection (flatMap, switchIfEmpty, concatMap, switchMap, zipWith, mergeWith,
  combineLatest, handle, expand, cache, share, replay); scheduler and threading questions
  (publishOn, subscribeOn, Schedulers.boundedElastic, Schedulers.parallel, parallel Flux,
  thread-safety); backpressure (onBackpressureBuffer, limitRate, prefetch); context
  propagation (Reactor Context, MDC bridging, Micrometer tracing, Spring Security
  ReactiveSecurityContextHolder); blocking-to-reactive and reactive-to-blocking bridging
  (Mono.fromCallable, Mono.fromFuture, block(), BlockHound, @Async interop, MVC+WebFlux);
  Kotlin coroutine/Flow interop (awaitSingle, mono{}, flux{}, asFlow, asFlux); R2DBC
  reactive database patterns (DatabaseClient, R2dbcEntityTemplate, reactive repositories,
  R2DBC transactions); reactor-netty HTTP client (WebClient, connection pooling, retry
  with backoff, timeouts); testing reactive code (StepVerifier, TestPublisher,
  PublisherProbe, virtual time, WebTestClient); debugging (checkpoint, log,
  Hooks.onOperatorDebug, ReactorDebugAgent, BlockHound); and deciding when to prefer
  virtual threads over Reactor. Also triggers on: reactive streams, hot vs cold publisher,
  marble diagrams, operator fusion, onNext/onError/onComplete signals, reactive security,
  WebFlux, R2DBC, reactor-netty, BlockHound, StepVerifier.
---

# Project Reactor Skill

## Section A: Assess the Situation

Before diving in, identify which mode applies:

- **Writing new reactive code?** Start with Section B to confirm Reactor is the right choice, then Section D for operator guidance.
- **Reviewing existing code?** Go directly to Section C — work through the full checklist, not just the reported concern.
- **Debugging a problem?** Go to Section J first. Debugging reactive code requires specific tooling before you can reason about what went wrong.
- **Deciding whether to adopt Reactor?** Section B is your decision tree.
- **Translating imperative code to reactive (or back)?** Section E has side-by-side examples.
- **Kotlin project?** Check Section F before writing any interop code.

## Section B: Should I Use Reactor Here?

Reactor is a tool, not an ideology. Make the choice deliberately.

**Use Reactor when:**
- I/O-bound fan-out with backpressure control is needed (e.g., calling 10 downstream services per request)
- Streaming data over SSE, WebSocket, or chunked HTTP responses
- Many concurrent async operations where thread-per-request doesn't scale
- Back-pressure from producer to consumer must be enforced
- Building a fully async pipeline where no step may block

**Consider virtual threads (JDK 21+) instead when:**
- Simple CRUD with blocking JDBC/I/O and no streaming
- Team is unfamiliar with reactive; learning curve isn't justified
- Existing codebase is imperative and the migration cost is high
- No streaming or backpressure requirements exist
- You're on Spring Boot 3.2+ — virtual threads are one config line

**Mixed case:** Use reactive at the transport layer (WebFlux + WebClient), but let individual service methods be synchronous or run on virtual threads internally. The boundary is clean.

> "Reactor is a tool. If virtual threads solve the problem cleanly, use them."

See `references/virtual-threads-decision.md` for the full decision tree.

## Section C: Code Review Checklist

When reviewing reactive code, check **all** of these — not just the specific concern raised. Issues compound across these dimensions.

### 1. Anti-patterns
- Nested `subscribe()` inside another `subscribe()` or inside an operator callback — breaks error propagation and backpressure
- Blocking calls (`Thread.sleep`, JDBC, blocking HTTP) inside a reactive chain on a non-`boundedElastic` thread
- Ignoring the returned `Mono`/`Flux` (calling a method and discarding the publisher without subscribing)
- Shared mutable state mutated in `doOnNext`/`map` without synchronization

See `references/anti-patterns.md` for the full catalog.

### 2. Scheduler usage
- Is `publishOn` used to shift execution to the correct pool after the chain is assembled?
- Is `subscribeOn` used to shift where subscription starts (I/O sources)?
- Is all blocking I/O wrapped in `Mono.fromCallable(...)` and executed on `Schedulers.boundedElastic()`?

See `references/schedulers-and-threading.md`.

### 3. Backpressure
- Is overflow handled with `onBackpressureBuffer`, `onBackpressureDrop`, or `onBackpressureLatest`?
- Are `prefetch` values tuned for the workload (`flatMap(fn, concurrency, prefetch)`)?

See `references/backpressure.md`.

### 4. Context propagation
- Is Reactor `Context` used (not `ThreadLocal`) for MDC/tracing/security values across operators?
- Is `ReactorContextAccessor` registered for Micrometer tracing?
- Is `ReactiveSecurityContextHolder` used (not `SecurityContextHolder`) in WebFlux?

See `references/context-propagation.md`.

### 5. Error handling
- Are errors surfaced, not silently swallowed?
- `doOnError` is for side effects (logging) only — it does not recover. Use `onErrorResume` or `onErrorReturn` to handle errors.
- `onErrorResume` that re-emits the same error is a no-op and hides intent.

### 6. Resource cleanup
- Files, connections, and locks opened inside a reactive chain must be closed with `Flux.using()` or `doFinally()`.
- Never rely on GC for resource cleanup in a reactive pipeline.

### 7. Test coverage
- Is `StepVerifier` used? (Never `block()` in tests.)
- Are error paths tested, not just happy paths?
- Are time-based operators (`delay`, `interval`) tested with virtual time?

See `references/testing.md`.

## Section D: Operator Guidance

When selecting or explaining an operator, always cover: what it does to the sequence (marble behavior), why this operator vs alternatives, and any threading or backpressure implications.

### Core selection guide

| Goal | Operator | Why not the others |
|---|---|---|
| Parallel I/O, results may interleave | `flatMap` | `concatMap` is sequential; `switchMap` cancels previous |
| Sequential, order preserved | `concatMap` | `flatMap` doesn't guarantee order |
| Latest-wins (search-as-you-type) | `switchMap` | `flatMap` accumulates all; `concatMap` queues all |
| Mono → Flux (one item produces a stream) | `flatMapMany` | `flatMap` returns `Flux<Flux<T>>` without this |
| Default value / 404 handling | `switchIfEmpty` | `defaultIfEmpty` only works for scalar values |
| Map + filter in one pass | `handle` | Cleaner than chaining `map` + `filter` |
| Recursive / tree traversal | `expand` | No clean alternative in the operator set |

### Key operator behaviors

**`flatMap`** — subscribes to all inner publishers concurrently up to `concurrency` limit. Results arrive as inner publishers complete, so order is not preserved. Default concurrency is `Queues.SMALL_BUFFER_SIZE` (256). Use `flatMap(fn, maxConcurrency)` to limit.

**`concatMap`** — subscribes to inner publishers one at a time. Inner must complete before the next starts. Order is preserved. Use when operations must not overlap (database writes, rate-limited APIs with serial requirement).

**`switchMap`** — on each upstream signal, cancels the current inner publisher and subscribes to a new one. Use for user-driven searches where only the latest request matters.

**`switchIfEmpty`** — the fallback publisher is subscribed to only if the upstream completes without emitting any element. The fallback is cold — it is not evaluated eagerly.

**`cache` / `share` / `replay`** — convert a cold publisher to hot. `cache()` replays all elements to new subscribers. `share()` multicasts to current subscribers only. `replay(n)` replays the last `n` elements. Use when the upstream is expensive and multiple subscribers should share one execution.

**`handle`** — receives each element and a `SynchronousSink`. Call `sink.next(value)` to emit, `sink.error(e)` to fail, or neither to skip the element. More expressive than `map` + `filter`.

**`expand`** — performs breadth-first traversal. Each element produces more elements via the provided function; the stream ends when all produced publishers complete empty.

See `references/operators.md` for the full catalog with marble diagram descriptions.

## Section E: Paradigm Translation (Imperative to Reactive)

Always show both versions side by side so the reader can verify equivalence.

### Imperative to reactive

```java
// Imperative
User user = userRepo.findById(id);
if (user == null) throw new NotFoundException();
List<Order> orders = orderRepo.findByUserId(user.getId());
return orders;

// Reactive equivalent
return userRepo.findById(id)                                          // Mono<User>
    .switchIfEmpty(Mono.error(new NotFoundException()))
    .flatMapMany(user -> orderRepo.findByUserId(user.getId()));        // Flux<Order>
```

### Blocking I/O wrapped reactively

```java
// Blocking call wrapped safely on boundedElastic
Mono<User> userMono = Mono.fromCallable(() -> jdbcUserRepo.findById(id))
    .subscribeOn(Schedulers.boundedElastic());
```

### CompletableFuture bridging

```java
// Non-lazy: future starts immediately
Mono<User> fromFuture = Mono.fromFuture(asyncClient.getUser(id));

// Lazy: future starts on subscription
Mono<User> lazy = Mono.fromFuture(() -> asyncClient.getUser(id));
```

### Reactive to blocking (at composition roots only)

```java
// Only acceptable at the edge of the application: main(), CLI tools, or tests
User user = userMono.block();
List<Order> orders = orderFlux.collectList().block();
```

Never call `.block()` inside a reactive chain or inside a controller method that returns `Mono`/`Flux`. Use BlockHound (Section J) to catch accidental blocking in CI.

See `references/bridging.md` for full patterns including Spring `@Async` interop and MVC+WebFlux coexistence.

## Section F: Kotlin Guidance

When code is Kotlin, evaluate coroutines/Flow before reaching for Reactor directly.

**Prefer coroutines/Flow when:**
- New Kotlin code with no existing Reactor dependency
- The team knows coroutines better than Reactor operators
- Suspend functions and structured concurrency fit the design

**Use Reactor in Kotlin when:**
- Integrating with Java reactive libraries that expose `Mono`/`Flux`
- Advanced backpressure or operator composition is needed
- Operators like `expand`, `handle`, or `groupBy` are required

### Key interop patterns

```kotlin
// suspend ↔ Mono
suspend fun getUser(id: String): User =
    userMono.awaitSingle()             // throws NoSuchElementException if empty

suspend fun getUserOrNull(id: String): User? =
    userMono.awaitSingleOrNull()       // returns null if empty

// Flux ↔ Flow
fun getOrders(): Flow<Order> =
    orderFlux.asFlow()

fun fromFlow(flow: Flow<Order>): Flux<Order> =
    flow.asFlux()

// Coroutine builder → Mono/Flux
val result: Mono<User> = mono {
    userService.findUser(id)           // suspend call inside
}

val stream: Flux<Event> = flux {
    for (event in eventChannel) {
        send(event)                    // emits into Flux
    }
}
```

See `references/kotlin-interop.md` for `awaitFirst`, `awaitFirstOrNull`, `awaitLast`, structured concurrency, and Flow backpressure.

## Section G: Data Access (R2DBC)

### Basic patterns

```java
// DatabaseClient — SQL-level control
Mono<User> user = databaseClient
    .sql("SELECT * FROM users WHERE id = :id")
    .bind("id", id)
    .map(row -> new User(
        row.get("id", Long.class),
        row.get("name", String.class)
    ))
    .one();

// R2dbcEntityTemplate — higher-level, type-safe
Mono<User> user = template.selectOne(
    query(where("id").is(id)),
    User.class
);

// Reactive repository
Flux<Order> orders = orderRepository.findByUserId(userId);
```

### Key considerations

- **Connection pool sizing**: R2DBC pool defaults are conservative. Tune `initialSize`, `maxSize`, and `maxIdleTime` in `ConnectionPoolConfiguration`.
- **Transaction boundaries**: Use `TransactionalOperator` to wrap a reactive chain in a transaction. Do not use `@Transactional` with reactive return types unless Spring Data R2DBC's support is confirmed for your version.
- **N+1 queries**: Reactive repositories do not join automatically. Fetch related data with explicit `flatMap` + a second query, or write a join in `DatabaseClient`.

```java
// TransactionalOperator usage
transactionalOperator.transactional(
    userRepository.save(user)
        .flatMap(saved -> auditRepository.save(new AuditEntry(saved.getId())))
).subscribe();
```

See `references/r2dbc-patterns.md` for connection pooling, reactive transactions, and repository composition patterns.

## Section H: HTTP Client (WebClient / Reactor Netty)

### Production-ready WebClient pattern

```java
webClient.get()
    .uri("/api/resource/{id}", id)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError,
        response -> response.bodyToMono(ErrorBody.class)
                            .map(body -> new ClientException(body.getMessage())))
    .onStatus(HttpStatusCode::is5xxServerError,
        response -> Mono.error(new ServerException()))
    .bodyToMono(Resource.class)
    .timeout(Duration.ofSeconds(5))
    .retryWhen(
        Retry.backoff(3, Duration.ofMillis(100))
             .maxBackoff(Duration.ofSeconds(2))
             .filter(ex -> ex instanceof IOException || ex instanceof TimeoutException)
    );
```

### Configuration notes

- **Connection pool**: `ReactorClientHttpConnector` with a custom `ConnectionProvider` — set `maxConnections`, `pendingAcquireMaxCount`, and `pendingAcquireTimeout`.
- **Codecs**: Configure `maxInMemorySize` on `WebClientCodecCustomizer` to avoid `DataBufferLimitException` on large response bodies.
- **Metrics**: Register `reactor.netty.http.client` meters via Micrometer for connection pool and request latency visibility.

See `references/reactor-netty-http.md` for netty tuning, codec configuration, and connection pool metrics.

## Section I: Testing

Never use `.block()` in tests. Always use `StepVerifier`.

### Happy path

```java
StepVerifier.create(userService.findById("123"))
    .expectNextMatches(user -> user.getId().equals("123"))
    .verifyComplete();
```

### Error path

```java
StepVerifier.create(userService.findById("missing"))
    .expectError(NotFoundException.class)
    .verify();
```

### Virtual time (for delay/interval)

```java
StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofSeconds(1)).take(3))
    .expectSubscription()
    .thenAwait(Duration.ofSeconds(3))
    .expectNextCount(3)
    .verifyComplete();
```

### TestPublisher (controlling upstream)

```java
TestPublisher<String> source = TestPublisher.create();
StepVerifier.create(myService.process(source.flux()))
    .then(() -> source.emit("a", "b"))
    .expectNext("A", "B")
    .then(source::complete)
    .verifyComplete();
```

### PublisherProbe (verifying branch execution)

```java
PublisherProbe<Void> probe = PublisherProbe.empty();
// Replace a branch of your pipeline with the probe, then assert:
probe.assertWasSubscribed();
probe.assertWasRequested();
```

See `references/testing.md` for WebTestClient, testing Reactor Context, and testing with security.

## Section J: Debugging

When something is broken, start here before reading code.

### Step 1: Add `log()` to see signal flow

```java
flux.log("myFlux")   // logs onSubscribe, request, onNext, onError, onComplete to SLF4J
```

### Step 2: Add `checkpoint()` for meaningful stack traces

```java
userRepository.findById(id)
    .checkpoint("finding user by id")
    .flatMap(user -> orderRepository.findByUserId(user.getId()))
    .checkpoint("fetching orders for user");
```

### Step 3: Enable assembly tracing

```java
// Development only — significant overhead
Hooks.onOperatorDebug();

// Production-safe alternative (attach at JVM startup with -javaagent)
ReactorDebugAgent.init();
```

### Reading assembly traces

```
Error has been observed at the following site(s):
    *__checkpoint ⇢ HTTP GET "/api/users" [ExceptionHandlingWebHandler]
    *__checkpoint ⇢ Handler com.example.UserController#getUser(String) [DispatcherHandler]
```

Read bottom-up. Each `checkpoint` is a boundary in your code — the error propagated through each one on its way out.

### BlockHound (detecting accidental blocking)

Add `BlockHound.install()` to your test setup or application startup in development. It throws `BlockingOperationError` when a blocking call is detected on a non-blocking thread. Common false positives: `ClassLoader`, JVM internals — allowlist them via `BlockHound.builder().allowBlockingCallsInside(...)`.

See `references/debugging.md` for BlockHound setup, common false positives, and production debugging strategies.

## Always Do / Never Do

### Always

- When reviewing reactive code, check **all** anti-patterns (Section C), not just the one asked about — issues compound
- When suggesting an operator, briefly describe its marble diagram behavior before explaining usage
- When the use case has no backpressure, streaming, or complex async need, proactively suggest virtual threads
- When working in Kotlin, evaluate whether coroutines/Flow would be more natural before reaching for Reactor operators
- Show both paradigm versions side by side when translating imperative to reactive (or back)
- Explain the "why" behind operator choices — why `concatMap` instead of `flatMap`, what the ordering guarantee buys

### Never

- Do not recommend Reactor for simple CRUD without justifying the tradeoff over virtual threads
- Do not suggest `.block()` inside a reactive chain — only at composition roots (main, CLI, test)
- Do not ignore the threading model when writing or reviewing reactive code
- Do not write reactive code that silently swallows errors — `doOnError` logging is a side effect, not a recovery; surface errors with `onErrorResume` or `onErrorReturn`
- Do not use `subscribeOn` to "fix" blocking I/O inside a chain without explaining that `boundedElastic` is correct specifically because it is sized for blocking work, not because it's a magic escape hatch

**Tone**: Reactor is a powerful tool, not an ideology. Be pragmatic. Explain the "why" behind choices. If something simpler works, say so.
