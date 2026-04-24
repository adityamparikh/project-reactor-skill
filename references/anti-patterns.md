# Project Reactor Anti-Patterns

Common mistakes and how to avoid them. Each section shows a **BAD** example (what not to do) and a **GOOD** replacement with an explanation of the consequences.

---

## 1. Nested `subscribe()` (Fire-and-Forget Inside a Chain)

Calling `.subscribe()` inside an operator like `flatMap` creates an inner subscription whose lifecycle is completely detached from the outer chain.

**BAD — inner subscription is unmanaged:**
```java
// Java
flux.flatMap(item -> {
    innerMono(item).subscribe(); // fire-and-forget: errors silently swallowed
    return Mono.just(item);
});
```
```kotlin
// Kotlin
flux.flatMap { item ->
    innerMono(item).subscribe() // lifecycle detached; backpressure lost
    item.toMono()
}
```

**GOOD — return the inner publisher so the chain manages it:**
```java
// Java
flux.flatMap(item -> innerMono(item).thenReturn(item));
```
```kotlin
// Kotlin
flux.flatMap { item -> innerMono(item).thenReturn(item) }
```

**Why it's bad:** The inner subscription has no parent; errors are silently lost, cancellation signals do not propagate, backpressure is broken, and resources (connections, threads) may never be released.

---

## 2. Blocking Inside a Reactive Chain

Calling any blocking operation — `.block()`, `Thread.sleep()`, synchronous JDBC, or any blocking I/O — inside `map` or `flatMap` can starve or deadlock the scheduler thread pool.

**BAD — blocks a scheduler thread:**
```java
// Deadlock scenario: parallel() has N threads; if all N block, the scheduler starves
Flux.range(1, 100)
    .parallel()
    .runOn(Schedulers.parallel())
    .map(i -> blockingDbCall(i)) // blocks a parallel thread → pool exhaustion → deadlock
    .sequential()
    .subscribe();
```

**GOOD — offload to boundedElastic, or wrap in `Mono.fromCallable`:**
```java
Flux.range(1, 100)
    .flatMap(i ->
        Mono.fromCallable(() -> blockingDbCall(i))
            .subscribeOn(Schedulers.boundedElastic())
    )
    .subscribe();
```

**Virtual threads alternative (Java 21+):**
```java
Flux.range(1, 100)
    .flatMap(i ->
        Mono.fromCallable(() -> blockingDbCall(i))
            .subscribeOn(Schedulers.fromExecutor(
                Executors.newVirtualThreadPerTaskExecutor()))
    )
    .subscribe();
```

**Why it's bad:** `Schedulers.parallel()` has a fixed thread count equal to CPU cores. One blocking call per thread exhausts the pool; subsequent tasks queue forever — a classic reactive deadlock.

---

## 3. Ignoring Return Values (Cold Publishers)

Mono and Flux are **lazy** — nothing executes until someone subscribes. Calling a method that returns a publisher and discarding the result means the operation never runs.

**BAD — the save never executes:**
```java
void saveUser(User user) {
    userRepository.save(user); // returns Mono<User> — silently ignored!
}
```

**GOOD — return the publisher so callers can subscribe:**
```java
Mono<User> saveUser(User user) {
    return userRepository.save(user);
}
```

**GOOD — or chain it into an existing pipeline:**
```java
Mono<Void> handleRequest(User user) {
    return validate(user)
        .flatMap(userRepository::save)
        .then();
}
```

**Why it's bad:** This is the single most common mistake for developers new to reactive programming. No exception is thrown; the code compiles and runs silently — the operation just never happens.

---

## 4. Shared Mutable State Without Synchronization

After `publishOn` or `parallel()`, `onNext` callbacks may execute on different threads concurrently. Mutating shared collections or counters without synchronization causes race conditions.

**BAD — unsynchronized external list:**
```java
List<String> results = new ArrayList<>();
flux.parallel()
    .runOn(Schedulers.parallel())
    .doOnNext(results::add) // race condition: ArrayList is not thread-safe
    .sequential()
    .blockLast();
```

**BAD — non-atomic counter:**
```java
int[] count = {0};
flux.doOnNext(item -> count[0]++).blockLast(); // lost updates under concurrency
```

**GOOD — use `collectList()` to aggregate safely:**
```java
Mono<List<String>> results = flux.collectList(); // Reactor handles thread safety
```

**GOOD — use `AtomicInteger` for counting:**
```java
AtomicInteger count = new AtomicInteger();
flux.doOnNext(item -> count.incrementAndGet()).blockLast();
```

**GOOD — use `scan()` for running aggregations:**
```java
flux.scan(0, (acc, item) -> acc + item.value())
    .last()
    .subscribe(total -> log.info("Total: {}", total));
```

**Why it's bad:** Reactor does not guarantee single-threaded delivery after `publishOn` or `parallel()`. Data races are non-deterministic and hard to reproduce under test.

---

## 5. Swallowing Errors Silently

Returning `Mono.empty()` from `onErrorResume` without logging discards the exception entirely. Callers receive an empty stream with no indication that anything went wrong.

**BAD — error disappears without a trace:**
```java
userService.findById(id)
    .onErrorResume(e -> Mono.empty()); // exception silently swallowed
```

**BAD — `doOnError` only logs; it does not handle the error:**
```java
userService.findById(id)
    .doOnError(e -> log.error("Failed", e)); // error still propagates un-handled
```

**GOOD — log and either propagate a typed error or return a meaningful fallback:**
```java
userService.findById(id)
    .onErrorResume(e -> {
        log.error("Failed to load user id={}", id, e);
        return Mono.error(new ServiceException("User unavailable", e));
    });
```

**GOOD — return a safe fallback value with context:**
```java
userService.findById(id)
    .onErrorResume(NotFoundException.class, e -> Mono.just(User.anonymous()))
    .onErrorResume(e -> {
        log.error("Unexpected error loading user", e);
        return Mono.error(new ServiceException("User service failure", e));
    });
```

**Why it's bad:** Silent empty results are indistinguishable from "not found." Debugging becomes nearly impossible when errors vanish without logs or typed signals.

---

## 6. Using `subscribeOn` Incorrectly

`subscribeOn` affects where the **subscription** happens and shifts the entire upstream if no `publishOn` precedes it. It does **not** move a downstream operator to a different thread. Stacking multiple `subscribeOn` calls has no additional effect — only the first one wins.

**BAD — trying to fix a blocking `map` with `subscribeOn` placed after it:**
```java
flux.map(this::blockingOp)      // still runs on the thread that subscribed
    .subscribeOn(Schedulers.boundedElastic()); // too late; blockingOp already ran
```

**BAD — multiple `subscribeOn` calls (only first takes effect):**
```java
flux.subscribeOn(Schedulers.boundedElastic())
    .map(this::op1)
    .subscribeOn(Schedulers.parallel()); // ignored
```

**GOOD — use `publishOn` to switch the thread for downstream operators:**
```java
flux.publishOn(Schedulers.boundedElastic()) // everything below runs on boundedElastic
    .map(this::blockingOp)
    .publishOn(Schedulers.parallel())       // switch back for fast CPU work
    .map(this::fastOp)
    .subscribe();
```

**Rule of thumb:** use `subscribeOn` once to schedule the cold source subscription; use `publishOn` to switch threads mid-chain.

---

## 7. Creating Hot Publishers by Mistake

`Flux.create()`, `Flux.interval()`, and similar sources are cold by default — each subscriber gets its own independent stream. Sharing a reference without multicasting gives every subscriber its own timer, connection, or event loop.

**BAD — three subscribers, three independent timers:**
```java
Flux<Long> ticks = Flux.interval(Duration.ofSeconds(1));
ticks.subscribe(t -> serviceA.handle(t));
ticks.subscribe(t -> serviceB.handle(t));
ticks.subscribe(t -> serviceC.handle(t)); // 3 separate intervals running
```

**GOOD — `share()` for a multicast that auto-connects on first subscriber:**
```java
Flux<Long> ticks = Flux.interval(Duration.ofSeconds(1)).share();
ticks.subscribe(t -> serviceA.handle(t));
ticks.subscribe(t -> serviceB.handle(t)); // same interval, two consumers
```

**GOOD — `publish().autoConnect(n)` to start after n subscribers:**
```java
Flux<Long> ticks = Flux.interval(Duration.ofSeconds(1))
    .publish()
    .autoConnect(2); // starts emitting only after 2 subscribers connect
```

**GOOD — `replay(n)` to cache the last n items for late subscribers:**
```java
Flux<Event> events = source.replay(10).autoConnect(1);
```

**Why it's bad:** Each cold subscription to `Flux.interval()` spawns an independent scheduler task. Three subscribers = three timers, three sets of events, and potentially three connections to the upstream source.

---

## 8. Overcomplicating Simple Operations

Wrapping a pure synchronous transformation inside a `Mono.just().map()` chain adds overhead and misleads readers into thinking there is real async work. If the result is never composed into a reactive pipeline, the reactor machinery is pure noise.

**BAD — unnecessary reactive wrapper around a sync call:**
```java
Mono<String> format(String s) {
    return Mono.just(s).map(String::toUpperCase); // no async work; just call it directly
}
```

**GOOD — call the function directly; wrap only at the reactive boundary:**
```java
String format(String s) {
    return s.toUpperCase();
}

// At the boundary where reactive composition is needed:
Mono<String> result = Mono.just(input).map(this::format);
```

**GOOD — if the whole flow can be sync, consider virtual threads instead:**
```java
// Java 21+: structured concurrency may be cleaner than reactive for fully-sync services
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var future = scope.fork(() -> blockingService.call());
    scope.join().throwIfFailed();
    return future.get();
}
```

**Why it's bad:** Unnecessary reactive layers make stack traces harder to read, add scheduling overhead, and signal to maintainers that async is happening when it is not.

---

## 9. Not Handling Empty Upstream

A reactive repository or upstream service may return `Mono.empty()` when a record is not found. Downstream `flatMap`, `map`, and other operators silently skip — no item emitted, no error thrown, no log message.

**BAD — downstream flatMap never runs; caller gets empty with no explanation:**
```java
userRepository.findById(id) // returns Mono.empty() when not found
    .flatMap(user -> enrichWithProfile(user)) // skipped silently
    .subscribe(result -> log.info("Got: {}", result)); // never logged
```

**GOOD — handle empty explicitly with `switchIfEmpty`:**
```java
userRepository.findById(id)
    .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
    .flatMap(user -> enrichWithProfile(user))
    .subscribe();
```

**GOOD — provide a default value with `defaultIfEmpty`:**
```java
userRepository.findById(id)
    .defaultIfEmpty(User.anonymous())
    .flatMap(user -> enrichWithProfile(user))
    .subscribe();
```

**GOOD — always verify the empty case in tests:**
```java
StepVerifier.create(userRepository.findById(-1L))
    .expectNextCount(0)
    .verifyComplete(); // confirm empty is intentional, not a bug
```

**Why it's bad:** Silent empty completions are the reactive equivalent of returning `null` without documentation. Callers cannot distinguish "not found" from "service failed silently."

---

## 10. Missing Backpressure on Unbounded Sources

A hot `Flux` that emits faster than the downstream can consume will overflow. Without an explicit overflow strategy, Reactor will throw `reactor.core.Exceptions$OverflowException` or silently drop items depending on the operator.

**BAD — no overflow strategy on a fast source:**
```java
Flux<Event> events = hotEventBus.asFlux(); // may emit thousands/sec
events.flatMap(e -> slowProcessing(e))     // processing takes 100ms each
      .subscribe();                         // OverflowException or item loss
```

**GOOD — buffer with a bounded queue and overflow callback:**
```java
hotEventBus.asFlux()
    .onBackpressureBuffer(
        1000,                                   // max buffered items
        dropped -> log.warn("Dropped: {}", dropped), // overflow callback
        BufferOverflowStrategy.DROP_OLDEST      // eviction strategy
    )
    .flatMap(e -> slowProcessing(e))
    .subscribe();
```

**GOOD — drop with logging when losing some events is acceptable:**
```java
hotEventBus.asFlux()
    .onBackpressureDrop(dropped -> metrics.increment("events.dropped"))
    .flatMap(e -> slowProcessing(e))
    .subscribe();
```

**GOOD — use `limitRate` to signal demand to a cooperative source:**
```java
cooperativeSource.asFlux()
    .limitRate(64)          // request 64 items at a time from upstream
    .flatMap(e -> process(e), 16) // max 16 concurrent inner subscriptions
    .subscribe();
```

**Strategy guide:**
- `onBackpressureBuffer(n)` — keep recent items; fail fast when buffer fills. Use when dropping is unacceptable and you need to detect overload.
- `onBackpressureDrop` — discard newest items silently or with a callback. Use for metrics, telemetry, or UI refresh where stale data is fine.
- `onBackpressureLatest` — keep only the most recent item. Use for live sensor readings or UI state updates.
- `limitRate` — cooperative flow control when the source supports backpressure signals.

**Why it's bad:** An unbounded buffer grows until the JVM runs out of heap. An unchecked drop discards data without any record. Neither is acceptable in production without an explicit, logged strategy.
