---
name: project-reactor
description: |
  Expert guide for Project Reactor (reactor-core 3.6+, BOM 2024.0+). Use whenever
  the user is writing, reviewing, debugging, or reasoning about reactive code
  involving Mono, Flux, Publisher, Subscriber, or reactive pipelines. Triggers on:
  operator selection (flatMap, switchIfEmpty, concatMap, switchMap, zipWith, mergeWith,
  combineLatest, handle, expand, cache, share, replay); schedulers and threading
  (publishOn, subscribeOn, Schedulers.boundedElastic, Schedulers.parallel, parallel
  Flux, thread-safety); backpressure (onBackpressureBuffer, limitRate, prefetch);
  context propagation (Reactor Context, MDC bridging, Micrometer tracing, Spring
  Security ReactiveSecurityContextHolder); blocking/reactive bridging (Mono.fromCallable,
  Mono.fromFuture, block(), BlockHound, @Async, MVC+WebFlux); Kotlin coroutine/Flow
  interop (awaitSingle, mono{}, flux{}, asFlow, asFlux); R2DBC patterns (DatabaseClient,
  R2dbcEntityTemplate, reactive repositories, R2DBC transactions); reactor-netty HTTP
  client (WebClient, connection pooling, retry with backoff, timeouts); testing
  (StepVerifier, TestPublisher, PublisherProbe, virtual time, WebTestClient);
  debugging (checkpoint, log, Hooks.onOperatorDebug, ReactorDebugAgent, BlockHound);
  and deciding when to prefer virtual threads. Also triggers on: reactive streams,
  hot vs cold publisher, marble diagrams, operator fusion, onNext/onError/onComplete,
  reactive security, WebFlux, R2DBC, reactor-netty, BlockHound, StepVerifier, Sinks,
  Sinks.One, Sinks.Many, EmitterProcessor, FluxProcessor, MonoProcessor, programmatic
  emission, EmitResult, EmitFailureHandler, tryEmitNext, sink.asFlux.
---

# Project Reactor Skill

## A: Assess the Situation

- **Writing new reactive code?** → Section B (is Reactor the right choice?), then D.
- **Reviewing existing code?** → Section C — work through the *full* checklist.
- **Debugging?** → Section J first; debugging reactive code needs specific tooling.
- **Adopting Reactor?** → Section B decision tree.
- **Translating imperative ↔ reactive?** → Section E.
- **Kotlin project?** → Section F before any interop code.

## B: Should I Use Reactor Here?

Reactor is a tool, not an ideology. Choose deliberately.

**Use Reactor when:**
- I/O-bound fan-out with backpressure (calling N downstream services per request)
- Streaming over SSE, WebSocket, or chunked HTTP
- Many concurrent async ops where thread-per-request doesn't scale
- Producer→consumer backpressure must be enforced
- Fully async pipeline where no step may block

**Consider virtual threads (JDK 21+) instead when:**
- Simple CRUD with blocking JDBC/I/O, no streaming
- Team unfamiliar with reactive; learning curve isn't justified
- Existing imperative codebase; migration cost is high
- No streaming or backpressure requirements
- On Spring Boot 3.2+ — virtual threads is one config line

**Mixed:** Use reactive at the transport layer (WebFlux + WebClient) but let service methods be sync or run on virtual threads internally. Clean boundary.

> "Reactor is a tool. If virtual threads solve the problem cleanly, use them."

See `references/virtual-threads-decision.md` for the full decision tree.

## C: Code Review Checklist

Check **all** of these — issues compound.

### 1. Anti-patterns
- Nested `subscribe()` inside another `subscribe()` or inside an operator callback — breaks error propagation and backpressure
- Blocking calls (`Thread.sleep`, JDBC, blocking HTTP) inside a reactive chain on a non-`boundedElastic` thread
- Ignoring the returned `Mono`/`Flux` (discarding the publisher without subscribing)
- Shared mutable state mutated in `doOnNext`/`map` without synchronization

See `references/anti-patterns.md`.

### 2. Scheduler usage
- `publishOn` shifts execution downstream after assembly
- `subscribeOn` shifts where subscription starts (cold I/O sources)
- All blocking I/O wrapped in `Mono.fromCallable(...)` on `Schedulers.boundedElastic()`

See `references/schedulers-and-threading.md`.

### 3. Backpressure
- Overflow handled with `onBackpressureBuffer/Drop/Latest`
- `prefetch` tuned for the workload (`flatMap(fn, concurrency, prefetch)`)

See `references/backpressure.md`.

### 4. Context propagation
- Reactor `Context` (not `ThreadLocal`) for MDC/tracing/security across operators
- `ReactorContextAccessor` registered for Micrometer tracing
- `ReactiveSecurityContextHolder` (not `SecurityContextHolder`) in WebFlux

See `references/context-propagation.md`.

### 5. Error handling
- Errors surfaced, not swallowed
- `doOnError` is for side effects (logging) only — does not recover. Use `onErrorResume`/`onErrorReturn`.
- `onErrorResume` that re-emits the same error is a no-op and hides intent

### 6. Resource cleanup
- Files, connections, locks opened inside the chain must close via `Flux.using()` or `doFinally()`
- Never rely on GC for resource cleanup

### 7. Test coverage
- Use `StepVerifier` — never `block()` in tests
- Test error paths, not just happy paths
- Time-based operators (`delay`, `interval`) tested with virtual time

See `references/testing.md`.

## D: Operator Guidance

When suggesting an operator, cover: marble behavior, why this over alternatives, threading/backpressure implications.

### Core selection guide

| Goal | Operator | Why not the others |
|---|---|---|
| Parallel I/O, order may interleave | `flatMap` | `concatMap` sequential; `switchMap` cancels previous |
| Sequential, order preserved | `concatMap` | `flatMap` doesn't guarantee order |
| Latest-wins (search-as-you-type) | `switchMap` | `flatMap` accumulates; `concatMap` queues |
| Mono → Flux | `flatMapMany` | `flatMap` returns `Flux<Flux<T>>` |
| Default / 404 handling | `switchIfEmpty` | `defaultIfEmpty` only handles scalars |
| Map + filter in one pass | `handle` | Cleaner than chained `map`+`filter` |
| Recursive / tree traversal | `expand` | No clean alternative |

### Key operator behaviors

- **`flatMap`** — subscribes to all inners concurrently up to `concurrency` (default `Queues.SMALL_BUFFER_SIZE` = 256). Order *not* preserved. Limit via `flatMap(fn, maxConcurrency)`.
- **`concatMap`** — subscribes one inner at a time; previous must complete first. Order preserved. Use for serial DB writes, rate-limited APIs.
- **`switchMap`** — cancels current inner on each new upstream signal. Use for user-driven searches.
- **`switchIfEmpty`** — fallback subscribed only if upstream completes empty. Cold/lazy.
- **`cache` / `share` / `replay`** — cold→hot. `cache()` replays all; `share()` multicasts to current subs only; `replay(n)` replays last n. Use when upstream is expensive and subscribers should share execution.
- **`handle`** — `(item, SynchronousSink)`: `sink.next` emits, `sink.error` fails, neither skips. More expressive than `map`+`filter`.
- **`expand`** — BFS traversal; each element produces more via the supplied function; ends when all produced publishers complete empty.

See `references/operators.md` for the full catalog.

For **programmatic emission** (callbacks, hot event streams, request-reply) use `Sinks` — the modern replacement for deprecated `EmitterProcessor`/`FluxProcessor`/`MonoProcessor`. See `references/sinks.md`.

## E: Paradigm Translation

Always show both versions side by side.

### Imperative → reactive
```java
// Imperative
User user = userRepo.findById(id);
if (user == null) throw new NotFoundException();
List<Order> orders = orderRepo.findByUserId(user.getId());
return orders;

// Reactive
return userRepo.findById(id)
    .switchIfEmpty(Mono.error(new NotFoundException()))
    .flatMapMany(user -> orderRepo.findByUserId(user.getId()));
```

### Blocking I/O wrapped reactively
```java
Mono<User> userMono = Mono.fromCallable(() -> jdbcUserRepo.findById(id))
    .subscribeOn(Schedulers.boundedElastic());
```

### CompletableFuture bridging
```java
Mono<User> eager = Mono.fromFuture(asyncClient.getUser(id));        // starts now
Mono<User> lazy  = Mono.fromFuture(() -> asyncClient.getUser(id));  // starts on subscribe
```

### Reactive → blocking (composition roots only)
```java
// Only at the edge: CLI main(), or integration-test fixture setup.
User user = userMono.block();
List<Order> orders = orderFlux.collectList().block();
```

Never call `.block()` inside a reactive chain or in a controller that returns `Mono`/`Flux`. In tests, drive assertions with `StepVerifier`. Use BlockHound (Section J) in CI.

See `references/bridging.md` for `@Async` interop and MVC+WebFlux coexistence.

## F: Kotlin Guidance

**Prefer coroutines/Flow when:**
- New Kotlin code with no existing Reactor dependency
- Team knows coroutines better
- Suspend functions + structured concurrency fit the design

**Use Reactor in Kotlin when:**
- Integrating with Java reactive libs that expose `Mono`/`Flux`
- Advanced backpressure or operator composition needed
- Operators like `expand`, `handle`, `groupBy` required

### Key interop patterns
```kotlin
// suspend ↔ Mono
suspend fun getUser(id: String): User = userMono.awaitSingle()        // throws if empty
suspend fun getUserOrNull(id: String): User? = userMono.awaitSingleOrNull()

// Flux ↔ Flow
fun getOrders(): Flow<Order> = orderFlux.asFlow()
fun fromFlow(flow: Flow<Order>): Flux<Order> = flow.asFlux()

// Coroutine builder → Mono/Flux
val result: Mono<User> = mono { userService.findUser(id) }            // suspend inside
val stream: Flux<Event> = flux { for (e in eventChannel) send(e) }
```

See `references/kotlin-interop.md` for `awaitFirst`, structured concurrency, Flow backpressure.

## G: Data Access (R2DBC)

```java
// DatabaseClient — SQL-level control
Mono<User> user = databaseClient
    .sql("SELECT * FROM users WHERE id = :id")
    .bind("id", id)
    .map(row -> new User(row.get("id", Long.class), row.get("name", String.class)))
    .one();

// R2dbcEntityTemplate — higher-level, type-safe
Mono<User> user = template.selectOne(query(where("id").is(id)), User.class);

// Reactive repository
Flux<Order> orders = orderRepository.findByUserId(userId);
```

- **Connection pool**: defaults are conservative. Tune `initialSize`, `maxSize`, `maxIdleTime` in `ConnectionPoolConfiguration`.
- **Transactions**: With `R2dbcTransactionManager` (default when Spring Data R2DBC is on classpath), `@Transactional` on a `Mono`/`Flux` method works — Spring propagates via Reactor Context. For programmatic control use `TransactionalOperator`. Never block inside a reactive transaction.
- **N+1**: Reactive repos do *not* join automatically. Use explicit `flatMap`+second query, or write a join in `DatabaseClient`.

```java
transactionalOperator.transactional(
    userRepository.save(user)
        .flatMap(saved -> auditRepository.save(new AuditEntry(saved.getId())))
).subscribe();
```

See `references/r2dbc-patterns.md`.

## H: HTTP Client (WebClient / Reactor Netty)

```java
webClient.get()
    .uri("/api/resource/{id}", id)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError,
        r -> r.bodyToMono(ErrorBody.class).map(b -> new ClientException(b.getMessage())))
    .onStatus(HttpStatusCode::is5xxServerError, r -> Mono.error(new ServerException()))
    .bodyToMono(Resource.class)
    .timeout(Duration.ofSeconds(5))
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
        .maxBackoff(Duration.ofSeconds(2))
        .filter(ex -> ex instanceof IOException || ex instanceof TimeoutException));
```

- **Pool**: `ReactorClientHttpConnector` with custom `ConnectionProvider` — `maxConnections`, `pendingAcquireMaxCount`, `pendingAcquireTimeout`.
- **Codecs**: Configure `maxInMemorySize` to avoid `DataBufferLimitException` on large bodies.
- **Metrics**: Register `reactor.netty.http.client` meters via Micrometer for pool + latency visibility.

See `references/reactor-netty-http.md`.

## I: Testing

Never `.block()` in tests; always use `StepVerifier`.

```java
// Happy path
StepVerifier.create(userService.findById("123"))
    .expectNextMatches(u -> u.getId().equals("123"))
    .verifyComplete();

// Error
StepVerifier.create(userService.findById("missing"))
    .expectError(NotFoundException.class)
    .verify();

// Virtual time (delay/interval)
StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofSeconds(1)).take(3))
    .expectSubscription()
    .thenAwait(Duration.ofSeconds(3))
    .expectNextCount(3)
    .verifyComplete();

// TestPublisher — control upstream
TestPublisher<String> source = TestPublisher.create();
StepVerifier.create(myService.process(source.flux()))
    .then(() -> source.emit("a", "b"))
    .expectNext("A", "B")
    .then(source::complete)
    .verifyComplete();

// PublisherProbe — verify branch executed
PublisherProbe<Void> probe = PublisherProbe.empty();
// ...replace a branch with probe.mono(), then:
probe.assertWasSubscribed();
probe.assertWasRequested();
```

See `references/testing.md` for WebTestClient, Reactor Context testing, security testing.

## J: Debugging

Start here before reading code.

### 1. `log()` to see signal flow
```java
flux.log("myFlux")   // onSubscribe, request, onNext, onError, onComplete via SLF4J
```

### 2. `checkpoint()` for meaningful stack traces
```java
userRepository.findById(id)
    .checkpoint("finding user by id")
    .flatMap(user -> orderRepository.findByUserId(user.getId()))
    .checkpoint("fetching orders for user");
```

### 3. Assembly tracing
```java
Hooks.onOperatorDebug();    // development only — significant overhead
ReactorDebugAgent.init();   // production-safe (attach as -javaagent)
```

### Reading assembly traces
```
Error has been observed at the following site(s):
    *__checkpoint ⇢ HTTP GET "/api/users" [ExceptionHandlingWebHandler]
    *__checkpoint ⇢ Handler com.example.UserController#getUser(String) [DispatcherHandler]
```
Read bottom-up. Each `checkpoint` is a boundary the error propagated through.

### BlockHound
`BlockHound.install()` in test setup or dev startup throws `BlockingOperationError` when a blocking call hits a non-blocking thread. Allowlist false positives (`ClassLoader`, JVM internals) via `BlockHound.builder().allowBlockingCallsInside(...)`.

See `references/debugging.md`.

## Always Do / Never Do

### Always
- When reviewing reactive code, check **all** anti-patterns (Section C), not just the one asked about — issues compound
- When suggesting an operator, briefly describe its marble behavior before usage
- When the use case has no backpressure/streaming/complex async need, proactively suggest virtual threads
- In Kotlin, evaluate whether coroutines/Flow are more natural before reaching for Reactor operators
- Show both paradigm versions side by side when translating
- Explain the "why" behind operator choices

### Never
- Recommend Reactor for simple CRUD without justifying the tradeoff vs virtual threads
- Suggest `.block()` inside a reactive chain — only at composition roots (main, CLI, test)
- Ignore the threading model when writing or reviewing reactive code
- Write code that silently swallows errors — `doOnError` is a side effect, not recovery; surface errors via `onErrorResume`/`onErrorReturn`
- Use `subscribeOn` to "fix" blocking I/O without explaining that `boundedElastic` is correct *specifically because* it's sized for blocking work, not a magic escape hatch

**Tone**: Reactor is a powerful tool, not an ideology. Be pragmatic. Explain the "why". If something simpler works, say so.
