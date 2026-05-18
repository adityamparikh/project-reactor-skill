# Project Reactor Anti-Patterns

Common mistakes and how to avoid them. Each section shows a **BAD** example and a **GOOD** replacement with consequences.

---

## 1. Nested `subscribe()` (Fire-and-Forget Inside a Chain)

Calling `.subscribe()` inside an operator like `flatMap` creates an inner subscription whose lifecycle is detached from the outer chain.

**BAD — inner subscription is unmanaged:**
```java
flux.flatMap(item -> {
    innerMono(item).subscribe(); // fire-and-forget: errors silently swallowed
    return Mono.just(item);
});
```
```kotlin
flux.flatMap { item ->
    innerMono(item).subscribe() // lifecycle detached; backpressure lost
    item.toMono()
}
```

**GOOD — return the inner publisher so the chain manages it:**
```java
flux.flatMap(item -> innerMono(item).thenReturn(item));
```

**Why:** Inner subscription has no parent. Errors lost, cancellation doesn't propagate, backpressure broken, resources (connections, threads) may never be released.

---

## 2. Blocking Inside a Reactive Chain

Any blocking op — `.block()`, `Thread.sleep()`, JDBC, blocking HTTP — inside `map`/`flatMap` can starve or deadlock the scheduler.

**BAD — blocks a scheduler thread:**
```java
// parallel() has N=CPU threads; if all N block, scheduler starves → deadlock
Flux.range(1, 100)
    .parallel()
    .runOn(Schedulers.parallel())
    .map(i -> blockingDbCall(i)) // pool exhaustion
    .sequential()
    .subscribe();
```

**GOOD — offload to boundedElastic via `fromCallable`:**
```java
Flux.range(1, 100)
    .flatMap(i ->
        Mono.fromCallable(() -> blockingDbCall(i))
            .subscribeOn(Schedulers.boundedElastic()))
    .subscribe();
```

**Virtual threads alternative (Java 21+):**
```java
.subscribeOn(Schedulers.fromExecutor(Executors.newVirtualThreadPerTaskExecutor()))
```

**Why:** `Schedulers.parallel()` has fixed thread count = CPU cores. One blocking call per thread exhausts the pool; subsequent tasks queue forever.

---

## 3. Ignoring Return Values (Cold Publishers)

Mono/Flux are **lazy** — nothing executes until subscribed. Discarding a returned publisher means the op never runs.

**BAD — the save never executes:**
```java
void saveUser(User user) {
    userRepository.save(user); // returns Mono<User> — silently ignored!
}
```

**GOOD — return the publisher, or chain it in:**
```java
Mono<User> saveUser(User user) { return userRepository.save(user); }

Mono<Void> handleRequest(User user) {
    return validate(user).flatMap(userRepository::save).then();
}
```

**Why:** Single most common mistake for reactive newcomers. No exception thrown; the code compiles and runs — the operation just never happens.

---

## 4. Shared Mutable State Without Synchronization

After `publishOn` or `parallel()`, `onNext` callbacks may run on different threads concurrently. Mutating shared collections/counters unsafely causes races.

**BAD — unsynchronized list / non-atomic counter:**
```java
List<String> results = new ArrayList<>();
flux.parallel().runOn(Schedulers.parallel())
    .doOnNext(results::add)            // race: ArrayList not thread-safe
    .sequential().blockLast();

int[] count = {0};
flux.doOnNext(item -> count[0]++).blockLast(); // lost updates
```

**GOOD — `collectList`, `AtomicInteger`, or `scan`:**
```java
Mono<List<String>> results = flux.collectList();              // Reactor handles safety

AtomicInteger count = new AtomicInteger();
flux.doOnNext(item -> count.incrementAndGet()).blockLast();

flux.scan(0, (acc, item) -> acc + item.value()).last()
    .subscribe(total -> log.info("Total: {}", total));
```

**Why:** Reactor does *not* guarantee single-threaded delivery after `publishOn` or `parallel()`. Data races are non-deterministic and hard to reproduce.

---

## 5. Swallowing Errors Silently

Returning `Mono.empty()` from `onErrorResume` discards the exception. Callers see an empty stream with no signal that anything went wrong.

**BAD:**
```java
userService.findById(id).onErrorResume(e -> Mono.empty());      // disappears
userService.findById(id).doOnError(e -> log.error("Failed", e)); // doOnError only logs; error still propagates
```

**GOOD — log and propagate typed, or return meaningful fallback:**
```java
userService.findById(id)
    .onErrorResume(NotFoundException.class, e -> Mono.just(User.anonymous()))
    .onErrorResume(e -> {
        log.error("Unexpected error loading user", e);
        return Mono.error(new ServiceException("User service failure", e));
    });
```

**Why:** Silent empty results are indistinguishable from "not found". Debugging becomes nearly impossible when errors vanish without logs or typed signals.

---

## 6. Using `subscribeOn` Incorrectly

`subscribeOn` affects where **subscription** happens and shifts the upstream if no `publishOn` precedes it. It does **not** move a downstream operator. Stacking multiple `subscribeOn` calls has no effect — only the first wins.

**BAD:**
```java
// Too late: blockingOp ran before the scheduler switch took effect
flux.map(this::blockingOp).subscribeOn(Schedulers.boundedElastic());

// Multiple subscribeOn: only first applies
flux.subscribeOn(Schedulers.boundedElastic())
    .map(this::op1)
    .subscribeOn(Schedulers.parallel()); // ignored
```

**GOOD — `publishOn` to switch threads downstream:**
```java
flux.publishOn(Schedulers.boundedElastic())   // everything below on boundedElastic
    .map(this::blockingOp)
    .publishOn(Schedulers.parallel())          // switch back for CPU work
    .map(this::fastOp)
    .subscribe();
```

**Rule of thumb:** use `subscribeOn` once to schedule the cold source subscription; use `publishOn` to switch threads mid-chain.

---

## 7. Creating Hot Publishers by Mistake

`Flux.create()`, `Flux.interval()`, similar sources are **cold** by default — each subscriber gets its own independent stream.

**BAD — three subscribers, three independent timers:**
```java
Flux<Long> ticks = Flux.interval(Duration.ofSeconds(1));
ticks.subscribe(serviceA::handle);
ticks.subscribe(serviceB::handle);
ticks.subscribe(serviceC::handle); // 3 separate intervals running
```

**GOOD — multicast options:**
```java
Flux<Long> ticks = Flux.interval(Duration.ofSeconds(1)).share();    // multicast, auto-connect
// or .publish().autoConnect(n) — start after n subscribers
// or .replay(10).autoConnect(1) — cache last 10 for late subscribers
```

**Why:** Each cold subscription to `Flux.interval()` spawns an independent scheduler task. Three subscribers = three timers, three sets of events, potentially three upstream connections.

---

## 8. Overcomplicating Simple Operations

Wrapping a pure sync transform inside `Mono.just().map()` adds overhead and misleads readers.

**BAD:**
```java
Mono<String> format(String s) {
    return Mono.just(s).map(String::toUpperCase); // no async work
}
```

**GOOD — call directly; wrap only at the reactive boundary:**
```java
String format(String s) { return s.toUpperCase(); }
Mono<String> result = Mono.just(input).map(this::format);
```

If the whole flow can be sync, consider virtual threads + structured concurrency instead.

**Why:** Unnecessary reactive layers make stack traces harder to read, add scheduling overhead, and mislead maintainers about what is async.

---

## 9. Not Handling Empty Upstream

A reactive repo may return `Mono.empty()` when not found. Downstream `flatMap`/`map` silently skip — no item, no error, no log.

**BAD — caller gets empty with no explanation:**
```java
userRepository.findById(id)       // empty when not found
    .flatMap(this::enrichWithProfile) // skipped silently
    .subscribe(r -> log.info("Got: {}", r)); // never logs
```

**GOOD — handle empty explicitly:**
```java
userRepository.findById(id)
    .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
    .flatMap(this::enrichWithProfile)
    .subscribe();

// or defaultIfEmpty(User.anonymous())
```

Always verify the empty case in tests:
```java
StepVerifier.create(userRepository.findById(-1L)).expectNextCount(0).verifyComplete();
```

**Why:** Silent empty completions are the reactive equivalent of returning `null` undocumented. Callers can't distinguish "not found" from "service failed silently."

---

## 10. Missing Backpressure on Unbounded Sources

A hot `Flux` emitting faster than the consumer can take will overflow — either `OverflowException` or silent drops depending on the operator.

**BAD:**
```java
Flux<Event> events = hotEventBus.asFlux();   // thousands/sec
events.flatMap(e -> slowProcessing(e))        // 100ms each
      .subscribe();                            // OverflowException or item loss
```

**GOOD — pick an explicit strategy:**
```java
hotEventBus.asFlux()
    .onBackpressureBuffer(1000,
        dropped -> log.warn("Dropped: {}", dropped),
        BufferOverflowStrategy.DROP_OLDEST)
    .flatMap(this::slowProcessing)
    .subscribe();

// or .onBackpressureDrop(dropped -> metrics.increment("events.dropped"))
// or .limitRate(64) for a cooperative source
```

**Strategy guide:**
- `onBackpressureBuffer(n)` — keep recent; fail fast when full. Dropping unacceptable, detect overload.
- `onBackpressureDrop` — discard newest, optionally with callback. Telemetry/UI where stale data is fine.
- `onBackpressureLatest` — keep only the most recent. Live sensors, UI state.
- `limitRate` — cooperative flow control when source supports backpressure signals.

**Why:** Unbounded buffer → OOM. Unchecked drop → silent data loss. Neither acceptable in production without an explicit, logged strategy.
