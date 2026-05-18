# Bridging Blocking and Reactive in Project Reactor

Reactor pipelines are non-blocking by design. Introducing blocking calls carelessly causes thread starvation or deadlocks. This covers the common bridging patterns.

---

## 1. Blocking → Reactive

### Mono.fromCallable + subscribeOn(boundedElastic)

`fromCallable` defers execution until subscription; `subscribeOn` moves it off the reactive thread. `boundedElastic` grows on demand (default cap: 10×CPU cores), parks idle threads after 60s, queues tasks beyond the cap — never use `Schedulers.parallel()` for blocking work.

```java
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

**JDBC/JPA note:** the entire unit-of-work (including lazy loading) must complete on the same thread. Wrap the full `TransactionTemplate` block inside `fromCallable`:
```java
Mono.fromCallable(() ->
    txTemplate.execute(status -> {
        User u = repo.findById(id).orElseThrow();
        Hibernate.initialize(u.getOrders()); // lazy load — same thread
        return u;
    }))
.subscribeOn(Schedulers.boundedElastic());
```

### Mono.fromFuture / fromCompletionStage

```java
// Eager — future already running before subscription (avoid for retryable code)
Mono.fromFuture(asyncService.fetch(id));

// Lazy (preferred) — future starts only on subscribe
Mono.fromCompletionStage(() -> asyncService.fetch(id));
// Reactor 3.5+: Mono.fromFuture(() -> asyncService.fetch(id)) also works
```

---

## 2. Reactive → Blocking (Composition Roots Only)

OK at: CLI tools, batch jobs, `main()`, `CommandLineRunner`, tests without `StepVerifier`, adapters returning a plain value to sync callers.

```java
User user = userMono.block();                          // null if empty
User user = userMono.blockOptional().orElseThrow();    // throws on empty
User user = userMono.block(Duration.ofSeconds(5));     // throws on timeout

List<Order> orders = orderFlux.collectList().block();
Order first = orderFlux.blockFirst();
Order last  = orderFlux.blockLast();

CompletableFuture<User> future = userMono.toFuture();  // mixed-context bridge
```

### Never block inside a reactive chain

Blocking on a scheduler thread deadlocks because the blocked thread holds a pool slot while waiting for a result that needs another slot on the same scheduler.

```java
// ❌ Deadlock
Flux<User> bad = idFlux.flatMap(id -> Mono.just(userMono.block()));

// ✅ Correct — delegate to boundedElastic
Flux<User> ok = idFlux.flatMap(id ->
    Mono.fromCallable(() -> jdbcRepo.findById(id))
        .subscribeOn(Schedulers.boundedElastic()));
```

---

## 3. Virtual Threads (JDK 21+)

With virtual threads, blocking no longer pins an OS carrier thread, so the no-blocking rule relaxes **only on virtual-thread schedulers**. Still:

- `Schedulers.parallel()` / `Schedulers.single()` use platform threads — blocking still forbidden.
- `Schedulers.boundedElastic()` + `fromCallable` remains the correct Reactor pattern.
- Spring Boot 3.2+ with `spring.threads.virtual.enabled=true` runs MVC on virtual threads — keep that code sync.
- Use Reactor for **streaming/backpressure**, not just concurrency — virtual threads already cover that.
- Wrap `Executors.newVirtualThreadPerTaskExecutor()` via `Schedulers.fromExecutorService` if using inside a Reactor chain; configure `ContextPropagation` explicitly.

See `virtual-threads-decision.md`.

---

## 4. Mixed Codebases

- **Reactive calling blocking** → wrap with `fromCallable` + `boundedElastic` (§1).
- **Blocking calling reactive** → `.block()` at the boundary. If the blocking caller is itself inside a reactive chain, that's a bug — use `fromCallable` + `subscribeOn` instead.

**Gateway patterns:**

| Strategy | Description | Trade-off |
|---|---|---|
| Reactive shell, sync core | WebFlux at HTTP, `.block()` into sync service | Simple migration; no async benefit inside |
| Reactive all the way down | Full pipeline HTTP→DB | Max throughput; highest complexity |
| Reactive islands | Reactive only for external I/O, sync internally | Most pragmatic for incremental migration |

---

## 5. Spring-Specific

### @Async + Reactive

`@Async` returns `CompletableFuture` — bridge via lazy `fromFuture` supplier:
```java
Mono.fromFuture(asyncBean::doWork); // lazy supplier (Reactor 3.5+)
```

### Calling WebFlux from MVC

`.block()` at the controller boundary works but defeats the purpose. Prefer migrating the controller to WebFlux.

### BlockingExecutionConfigurer (Spring 6.1+ / Boot 3.2+)

Register an executor for WebFlux controllers that must block; Spring dispatches automatically for non-reactive return types.
```java
@Override
public void configureBlockingExecution(BlockingExecutionConfigurer configurer) {
    configurer.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}
```

### MVC + WebFlux mix

Spring Boot supports **one** dispatcher type per app. Use reactive `WebClient` in MVC by blocking at the boundary, or migrate fully to WebFlux.

---

## 6. BlockHound

Java agent that throws `BlockingOperationError` when a blocking call hits a thread that must be non-blocking (Reactor schedulers, Netty event loops).

```java
// JUnit 5
@BeforeAll static void setup() { BlockHound.install(); }

// Spring Boot
@SpringBootTest
@ExtendWith(BlockHoundSpringExtension.class)
class MyTest { }
```

### Allowlisting false positives
```java
BlockHound.install(builder ->
    builder.allowBlockingCallsInside(
        "com.zaxxer.hikari.pool.HikariPool", "getConnection"));
```

Common false-positive categories: Logback/SLF4J first-use init, class loading and static initializers, Hibernate schema validation, HikariCP internal pool management, certain JDK internals (`InetAddress.getLocalHost()`, zip operations).

When BlockHound fires: if it's a one-time init path, allowlist class+method; if hot-path, wrap with `fromCallable` + `subscribeOn(Schedulers.boundedElastic())`.
