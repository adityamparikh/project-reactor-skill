# Bridging Blocking and Reactive Worlds in Project Reactor

Reactor pipelines are non-blocking by design. Introducing blocking calls without care
causes thread starvation and, in the worst case, deadlocks. This document covers every
common bridging pattern and when to use each one.

---

## 1. Wrapping Blocking Calls → Reactive

### Mono.fromCallable + subscribeOn(boundedElastic)

`fromCallable` defers execution until subscription; `subscribeOn` moves it off the
reactive thread. `boundedElastic` grows on demand (default cap: 10 × CPU cores),
parks idle threads after 60 s, and queues tasks beyond the cap — never use
`Schedulers.parallel()` for blocking work.

```java
// ✅ Correct: blocking call on boundedElastic thread pool
Mono<User> findUser(String id) {
    return Mono.fromCallable(() -> jdbcUserRepo.findById(id))
               .subscribeOn(Schedulers.boundedElastic());
}
```

```kotlin
fun findUser(id: String): Mono<User> =
    Mono.fromCallable { jdbcUserRepo.findById(id) }
        .subscribeOn(Schedulers.boundedElastic())
```

**JDBC / JPA note:** the entire unit-of-work, including lazy loading, must complete
on the same thread. Wrap the full `TransactionTemplate` block inside `fromCallable`.

```java
Mono<User> findWithOrders(String id) {
    return Mono.fromCallable(() ->
            txTemplate.execute(status -> {
                User u = repo.findById(id).orElseThrow();
                Hibernate.initialize(u.getOrders()); // lazy load — same thread
                return u;
            }))
           .subscribeOn(Schedulers.boundedElastic());
}
```

### Mono.fromFuture / fromCompletionStage

```java
// Eager — future already running before subscription (avoid for retryable code)
Mono<Result> eager(String id) { return Mono.fromFuture(asyncService.fetch(id)); }

// Lazy (preferred) — future starts only on subscribe; retries re-execute correctly
Mono<Result> lazy(String id)  { return Mono.fromCompletionStage(() -> asyncService.fetch(id)); }
// Reactor 3.5+: Mono.fromFuture(() -> asyncService.fetch(id)) also works
```

```kotlin
fun lazy(id: String): Mono<Result> = Mono.fromCompletionStage { asyncService.fetch(id) }
```

---

## 2. Reactive → Blocking (at Composition Roots)

### When it's OK to block

CLI tools, batch jobs, `main()`, `CommandLineRunner`, JUnit tests without
`StepVerifier`, and adapters that must return a plain value to a synchronous caller.

```java
User user    = userMono.block();                         // null if empty
User user    = userMono.blockOptional().orElseThrow();   // safer — throws on empty
User user    = userMono.block(Duration.ofSeconds(5));    // throws on timeout

List<Order> orders = orderFlux.collectList().block();
Order first  = orderFlux.blockFirst();
Order last   = orderFlux.blockLast();
Order first  = orderFlux.blockFirst(Duration.ofSeconds(5));
```

### toFuture().get() — bridge back in mixed contexts

```java
CompletableFuture<User> future = userMono.toFuture();
User user = future.get(5, TimeUnit.SECONDS); // blocks the calling thread
```

### Never block inside a reactive chain

Blocking on a scheduler thread deadlocks because the blocked thread holds a pool slot
while waiting for a result that needs another pool slot on the same scheduler.

```java
// ❌ Deadlock — blocks the parallel scheduler thread
Flux<User> bad = idFlux.flatMap(id -> Mono.just(userMono.block()));

// ✅ Correct — delegate to boundedElastic
Flux<User> ok = idFlux.flatMap(id ->
    Mono.fromCallable(() -> jdbcRepo.findById(id))
        .subscribeOn(Schedulers.boundedElastic()));
```

---

## 3. Virtual Thread Considerations (JDK 21+)

With virtual threads, blocking a thread no longer pins an OS carrier thread, so the
no-blocking rule relaxes **only on virtual-thread schedulers**. However:

- `Schedulers.parallel()` / `Schedulers.single()` still use platform threads — blocking
  is still forbidden there.
- `Schedulers.boundedElastic()` + `fromCallable` remains the correct Reactor pattern
  for offloading blocking calls.
- Spring Boot 3.2+ with `spring.threads.virtual.enabled=true` runs MVC on virtual
  threads — keep that code synchronous and let the framework handle concurrency.
- Use Reactor for **streaming / backpressure** needs, not merely for concurrency,
  because virtual threads already cover that.
- Wrap `Executors.newVirtualThreadPerTaskExecutor()` with `Schedulers.fromExecutorService`
  if you must use it inside a Reactor chain, and configure `ContextPropagation` explicitly.

See `virtual-threads-decision.md` for a full decision matrix.

---

## 4. Mixed Codebases

**Reactive service calling blocking code** — wrap with `fromCallable` + `boundedElastic` (§1).

**Blocking service calling reactive** — call `.block()` at the boundary. If that
blocking service is itself invoked from inside a reactive chain, it is a bug: use
`fromCallable` + `subscribeOn` instead.

**Gateway patterns — where to draw the reactive boundary:**

| Strategy | Description | Trade-off |
|---|---|---|
| Reactive shell, sync core | WebFlux at HTTP layer, `.block()` into sync service | Simple migration; no async benefit inside |
| Reactive all the way down | Full pipeline from HTTP to DB | Max throughput; highest complexity |
| Reactive islands | Reactive only for external I/O; sync internally | Most pragmatic for incremental migrations |

---

## 5. Spring-Specific Patterns

### @Async + Reactive

`@Async` returns `CompletableFuture` — bridge with the lazy `fromFuture` supplier.

```java
Mono<Result> callAsyncMethod() {
    return Mono.fromFuture(asyncBean::doWork); // lazy supplier (Reactor 3.5+)
}
```

### Calling WebFlux from MVC

`.block()` at the controller boundary works but defeats the purpose. Prefer migrating
the controller to WebFlux (`Mono`/`Flux` return types) rather than blocking every request.

### BlockingExecutionConfigurer (Spring 6.1+ / Boot 3.2+)

Register an executor for WebFlux controllers that must block; Spring dispatches to it
automatically for non-reactive return types.

```java
@Override
public void configureBlockingExecution(BlockingExecutionConfigurer configurer) {
    configurer.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}
```

### Mixing MVC and WebFlux

Spring Boot supports **one** dispatcher type per application. Use reactive `WebClient`
in MVC by blocking at the boundary, or migrate the application to WebFlux fully.

---

## 6. BlockHound Integration

BlockHound is a Java agent that throws `BlockingOperationError` when a blocking call
is detected on a thread that must be non-blocking (Reactor schedulers, Netty event loops).

```java
// JUnit 5 — install once per test run
@BeforeAll
static void setup() { BlockHound.install(); }

// Spring Boot test
@SpringBootTest
@ExtendWith(BlockHoundSpringExtension.class)
class MyTest { }
```

```kotlin
companion object {
    @BeforeAll @JvmStatic fun setup() = BlockHound.install()
}
```

### Allowlisting false positives

```java
BlockHound.install(builder ->
    builder.allowBlockingCallsInside(
        "com.zaxxer.hikari.pool.HikariPool", "getConnection")
);
```

Common false-positive categories: Logback/SLF4J first-use initialization, class
loading and static initializers, Hibernate schema validation, HikariCP internal pool
management, and certain JDK internals (`InetAddress.getLocalHost()`, zip operations).

When BlockHound fires, inspect the stack trace: if it is a one-time init path,
allowlist the fully-qualified class and method; if it is a hot-path blocking call,
wrap it with `fromCallable` + `subscribeOn(Schedulers.boundedElastic())` (§1).
