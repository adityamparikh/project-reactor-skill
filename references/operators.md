# Project Reactor Operator Reference

> Target: Reactor 3.6+ / BOM 2024.0+  
> This reference covers the most commonly used operators across all six categories, with marble diagram descriptions, use-case guidance, and concise Java and Kotlin examples.

---

## Table of Contents

1. [Creating / Source Operators](#1-creating--source-operators)
   - [just](#just)
   - [empty](#empty)
   - [error](#error)
   - [never](#never)
   - [defer](#defer)
   - [fromCallable](#fromcallable)
   - [fromIterable](#fromiterable)
   - [fromStream](#fromstream)
   - [fromFuture](#fromfuture)
   - [interval](#interval)
   - [range](#range)
   - [create (FluxSink)](#create-fluxsink)
   - [generate (SynchronousSink)](#generate-synchronoussink)
   - [using](#using)
2. [Transforming Operators](#2-transforming-operators)
   - [map](#map)
   - [flatMap](#flatmap)
   - [concatMap](#concatmap)
   - [switchMap](#switchmap)
   - [flatMapMany](#flatmapmany)
   - [flatMapSequential](#flatmapsequential)
   - [transform](#transform)
   - [as](#as)
   - [handle](#handle)
   - [cast](#cast)
   - [expand / expandDeep](#expand--expanddeep)
   - [collectList / collectMap / collectMultimap](#collectlist--collectmap--collectmultimap)
3. [Filtering Operators](#3-filtering-operators)
   - [filter](#filter)
   - [take / takeLast](#take--takelast)
   - [skip / skipLast](#skip--skiplast)
   - [distinct](#distinct)
   - [distinctUntilChanged](#distinctuntilchanged)
   - [elementAt](#elementat)
   - [single](#single)
   - [next](#next)
   - [ignoreElements](#ignoreelements)
   - [takeWhile / skipWhile](#takewhile--skipwhile)
   - [takeUntilOther](#takeuntilother)
4. [Combining Operators](#4-combining-operators)
   - [zip / zipWith](#zip--zipwith)
   - [merge](#merge)
   - [mergeSequential](#mergesequential)
   - [concat](#concat)
   - [combineLatest](#combinelatest)
   - [withLatestFrom](#withlatestfrom)
   - [startWith](#startwith)
   - [switchOnFirst](#switchonfirst)
5. [Error Handling Operators](#5-error-handling-operators)
   - [onErrorReturn](#onerrorreturn)
   - [onErrorResume](#onerrorresume)
   - [onErrorMap](#onerrormap)
   - [onErrorComplete](#onerrorcomplete)
   - [retry](#retry)
   - [retryWhen](#retrywhen)
   - [doOnError](#doonerror)
   - [timeout](#timeout)
6. [Utility / Side-Effect Operators](#6-utility--side-effect-operators)
   - [doOnNext](#doonnext)
   - [doOnSubscribe](#doonsubscribe)
   - [doOnComplete / doOnTerminate / doFinally](#dooncomplete--doonterminate--dofinally)
   - [doOnDiscard](#doondiscard)
   - [log](#log)
   - [checkpoint](#checkpoint)
   - [timestamp / elapsed / timed](#timestamp--elapsed--timed)
   - [cache / cache(Duration)](#cache--cacheduration)
   - [share](#share)
   - [replay](#replay)
   - [publish(Function)](#publishfunction)
   - [hide](#hide)

---

## 1. Creating / Source Operators

### `just`

**Marble diagram:** Emits one or more pre-defined items and then immediately completes.

```
just(1, 2, 3):
──[1]──[2]──[3]──|
```

**When to use:** You already have the values in hand at assembly time. Use `Mono.just(value)` for a single known value, `Flux.just(a, b, c)` for a small fixed set. Do not use when the value is expensive to compute or may throw—use `fromCallable` instead.

```java
// Java
Mono<String> greeting = Mono.just("Hello, World!");
Flux<Integer> numbers = Flux.just(1, 2, 3, 4, 5);
```

```kotlin
// Kotlin
val greeting: Mono<String> = "Hello, World!".toMono()
val numbers: Flux<Int> = Flux.just(1, 2, 3, 4, 5)
```

> **Alternative:** `fromCallable` if the value might throw or is computed lazily.

---

### `empty`

**Marble diagram:** Completes immediately without emitting any items.

```
empty():
──|
```

**When to use:** Signal "no data, but no error" — useful as a default return value in `onErrorResume`, or to represent an intentionally empty result set.

```java
// Java
Flux<String> nothing = Flux.empty();
Mono<String> noValue = Mono.empty();
```

```kotlin
// Kotlin
val nothing: Flux<String> = Flux.empty()
val noValue: Mono<String> = Mono.empty()
```

---

### `error`

**Marble diagram:** Immediately terminates with the given error, emitting no items.

```
error(ex):
──X(ex)
```

**When to use:** Represent a known error condition eagerly, or in operator chains where you need to short-circuit with an error (e.g., inside `flatMap` or `switchMap` after a validation check).

```java
// Java
Mono<String> failed = Mono.error(new IllegalArgumentException("invalid input"));
Flux<Integer> alwaysFails = Flux.error(new RuntimeException("boom"));
```

```kotlin
// Kotlin
val failed: Mono<String> = Mono.error(IllegalArgumentException("invalid input"))
```

---

### `never`

**Marble diagram:** Never emits any item and never terminates (no complete, no error signal).

```
never():
────────────────── (no end)
```

**When to use:** Testing (to hold a subscription open), as a placeholder publisher, or in `timeout` testing scenarios. Rarely used in production outside test utilities.

```java
// Java
Flux<String> infinite = Flux.never();
```

---

### `defer`

**Marble diagram:** On each subscription, invokes the supplier to create a fresh publisher. Each subscriber sees its own independent sequence.

```
defer(() -> somePublisher):
  subscriber A: ──[publisher-A-items]──|
  subscriber B: ──[publisher-B-items]──|
```

**When to use:** Lazy evaluation—when the publisher itself depends on runtime state (e.g., a timestamp, a request-scoped value, or a database connection). Without `defer`, `just` or `fromCallable` captures the value at assembly time.

```java
// Java – captures time at subscription, not assembly
Mono<Instant> now = Mono.defer(() -> Mono.just(Instant.now()));

// Each subscriber sees the current time
now.subscribe(System.out::println); // T1
now.subscribe(System.out::println); // T2 (different)
```

```kotlin
// Kotlin
val now: Mono<Instant> = Mono.defer { Mono.just(Instant.now()) }
```

> **Alternative:** `fromCallable` is simpler when the lazy value is a single result from a blocking/throwing call. `defer` is more powerful because it can return any publisher, including a `Flux`.

---

### `fromCallable`

**Marble diagram:** Invokes the `Callable` on subscription, emits the single result, then completes. If the callable throws, emits the error.

```
fromCallable(() -> blockingCall()):
  on subscribe → invoke callable → ──[result]──|
                               or → ──X(exception)
```

**When to use:** Wrapping blocking library calls, I/O, or any method that may throw a checked exception. Safer than `just` because evaluation is deferred to subscription time.

```java
// Java
Mono<String> result = Mono.fromCallable(() -> Files.readString(Path.of("/etc/hosts")));

// Pair with subscribeOn to move blocking call off the event-loop thread
result.subscribeOn(Schedulers.boundedElastic())
      .subscribe(System.out::println);
```

```kotlin
// Kotlin
val result: Mono<String> = Mono.fromCallable { Files.readString(Path.of("/etc/hosts")) }
    .subscribeOn(Schedulers.boundedElastic())
```

> **Note:** Always pair `fromCallable` with `.subscribeOn(Schedulers.boundedElastic())` when the callable does I/O or blocks.

---

### `fromIterable`

**Marble diagram:** Iterates through the collection on demand, emitting one element per request until exhausted.

```
fromIterable([A, B, C]):
──[A]──[B]──[C]──|
```

**When to use:** You have an in-memory `List`, `Set`, or any `Iterable` that you want to stream. Respects backpressure naturally.

```java
// Java
List<String> names = List.of("Alice", "Bob", "Carol");
Flux<String> flux = Flux.fromIterable(names);
```

```kotlin
// Kotlin
val flux: Flux<String> = listOf("Alice", "Bob", "Carol").toFlux()
```

---

### `fromStream`

**Marble diagram:** Iterates through the `Stream`, emitting each element. The stream is consumed on the first subscription.

```
fromStream(stream):
──[A]──[B]──[C]──|
```

**When to use with caution:** Java `Stream`s are single-use; once consumed they cannot be replayed. A second subscription will throw `IllegalStateException`.

```java
// Java – CAREFUL: stream is consumed on first subscribe
Flux<String> flux = Flux.fromStream(() -> Stream.of("a", "b", "c")); // supplier form is safer
```

> **Warning:** Prefer `fromIterable` over `fromStream` for replay safety. If you must use a `Stream`, always pass a **supplier** (`Flux.fromStream(() -> myStream())`), not the stream directly—the supplier form creates a fresh stream per subscription.

---

### `fromFuture`

**Marble diagram:** Subscribes to the `CompletableFuture`, emits the result when it resolves, then completes. If the future completes exceptionally, emits the error.

```
fromFuture(cf):
  on subscribe → await future → ──[result]──|
                             or → ──X(exception)
```

**When to use:** Bridging legacy async code or third-party libraries that return `CompletableFuture`. For new code, prefer native Reactor APIs.

```java
// Java
CompletableFuture<String> cf = someAsyncApi.fetchData();
Mono<String> mono = Mono.fromFuture(cf);

// Lazy variant (creates future on subscription)
Mono<String> lazy = Mono.fromFuture(() -> someAsyncApi.fetchData());
```

```kotlin
// Kotlin
val mono: Mono<String> = someAsyncApi.fetchData().toMono()
```

> **Note:** Use the supplier overload `Mono.fromFuture(() -> cf)` to defer future creation to subscription time. Without it, the future starts executing at assembly time.

---

### `interval`

**Marble diagram:** Emits `0L`, `1L`, `2L`... at a fixed period on the `Schedulers.parallel()` scheduler (or a specified one). Never completes on its own.

```
interval(Duration.ofSeconds(1)):
──[0]──────[1]──────[2]──────[3]──── (continues indefinitely)
      1s         1s        1s
```

**When to use:** Polling, heartbeats, rate-limiting, or any periodic task. Always combine with `take`, `takeUntilOther`, or cancellation to bound the sequence.

```java
// Java – poll every second, take 10 readings
Flux<Long> ticker = Flux.interval(Duration.ofSeconds(1))
    .take(10)
    .doOnNext(i -> System.out.println("tick " + i));
```

```kotlin
// Kotlin
val ticker: Flux<Long> = Flux.interval(Duration.ofSeconds(1))
    .take(10)
```

> **Note:** `interval` emits on the `parallel` scheduler by default—do not block in downstream operators without `publishOn(Schedulers.boundedElastic())`.

---

### `range`

**Marble diagram:** Emits a contiguous sequence of integers from `start` to `start + count - 1`, then completes.

```
range(1, 5):
──[1]──[2]──[3]──[4]──[5]──|
```

**When to use:** Generating test data, paginating through a fixed number of pages, or producing index sequences.

```java
// Java
Flux<Integer> pages = Flux.range(1, 10)
    .flatMap(page -> httpClient.getPage(page));
```

```kotlin
// Kotlin
val pages: Flux<Response> = Flux.range(1, 10)
    .flatMap { page -> httpClient.getPage(page) }
```

---

### `create` (FluxSink)

**Marble diagram:** Provides a `FluxSink` to an imperative callback. The callback calls `sink.next()`, `sink.complete()`, or `sink.error()` at any time, from any thread.

```
create(sink -> { push items imperatively }):
──[A]──[B]──[C]──| (driven externally)
```

**When to use:** Bridging event-driven or callback-based APIs (listeners, WebSocket frames, message queues) to Reactor. Supports multiple emission modes via `OverflowStrategy` (BUFFER, DROP, LATEST, ERROR).

```java
// Java – bridge a listener-based API
Flux<Event> events = Flux.create(sink -> {
    EventListener listener = event -> sink.next(event);
    eventBus.register(listener);
    sink.onDispose(() -> eventBus.unregister(listener));
}, FluxSink.OverflowStrategy.BUFFER);
```

```kotlin
// Kotlin
val events: Flux<Event> = Flux.create({ sink ->
    val listener = EventListener { event -> sink.next(event) }
    eventBus.register(listener)
    sink.onDispose { eventBus.unregister(listener) }
}, FluxSink.OverflowStrategy.BUFFER)
```

> **Alternative:** Use `generate` when items are produced synchronously on demand (pull model). Use `push` (single-threaded variant of `create`) when only one thread calls `sink.next()`.

---

### `generate` (SynchronousSink)

**Marble diagram:** Calls the generator function once per subscriber demand. The generator must call `sink.next()` exactly once per invocation (or `sink.complete()`/`sink.error()`).

```
generate(...):
  demand(1) → call generator → ──[A]
  demand(1) → call generator → ──[B]
  demand(1) → call generator → ──[C]──|
```

**When to use:** Stateful, pull-based sequences where each element depends on the previous state—e.g., reading from a cursor, walking a linked list, generating Fibonacci numbers. More efficient than `create` for pull scenarios because it avoids buffering.

```java
// Java – Fibonacci sequence
Flux<Long> fibonacci = Flux.generate(
    () -> new long[]{0, 1},
    (state, sink) -> {
        sink.next(state[0]);
        return new long[]{state[1], state[0] + state[1]};
    }
).take(10);
```

```kotlin
// Kotlin
val fibonacci: Flux<Long> = Flux.generate(
    { longArrayOf(0L, 1L) },
    { state, sink ->
        sink.next(state[0])
        longArrayOf(state[1], state[0] + state[1])
    }
).take(10)
```

---

### `using`

**Marble diagram:** Acquires a resource, uses it to create a publisher, subscribes, and then guarantees the resource is released when the sequence terminates (complete, error, or cancel).

```
using(acquire, factory, cleanup):
  acquire resource → ──[items from factory]──| → cleanup
                  or → ──X(error)             → cleanup
                  or → cancel                 → cleanup
```

**When to use:** Any resource that must be closed after use: database connections, file handles, HTTP connections. The cleanup is invoked in all termination cases, including cancellation.

```java
// Java
Flux<String> lines = Flux.using(
    () -> new BufferedReader(new FileReader("/data/input.txt")),
    reader -> Flux.fromStream(reader.lines()),
    reader -> {
        try { reader.close(); } catch (IOException ignored) {}
    }
);
```

```kotlin
// Kotlin
val lines: Flux<String> = Flux.using(
    { BufferedReader(FileReader("/data/input.txt")) },
    { reader -> Flux.fromStream(reader.lines()) },
    { reader -> reader.close() }
)
```

> **Alternative:** `Flux.usingWhen` for reactive resource cleanup (e.g., `Mono<Connection>` → async `connection.close()`).

---

## 2. Transforming Operators

### `map`

**Marble diagram:** Applies a synchronous function to each item, emitting the transformed result. One item in, one item out.

```
──[1]──[2]──[3]──|
    map(x -> x*2)
──[2]──[4]──[6]──|
```

**When to use:** Any synchronous, non-blocking 1:1 transformation. If the mapping function throws, the exception is caught and forwarded as an error signal.

```java
// Java
Flux<String> upper = Flux.just("hello", "world")
    .map(String::toUpperCase);
```

```kotlin
// Kotlin
val upper: Flux<String> = Flux.just("hello", "world")
    .map { it.uppercase() }
```

> **Alternative:** Use `flatMap` when the transformation returns a `Publisher` (e.g., an async call). Using `map` with a function that returns a `Mono` gives you a `Flux<Mono<T>>`, not `Flux<T>`.

---

### `flatMap`

**Marble diagram:** Maps each item to an inner publisher and **concurrently** subscribes to all inner publishers. Results interleave as they arrive—order is not guaranteed.

```
──[A]──[B]──[C]──|
    flatMap(x -> fetchAsync(x))
──[b1]──[a1]──[c1]──[a2]──[b2]──|  (interleaved)
```

**When to use:** Parallel async I/O—fetching multiple resources simultaneously where order does not matter. Use the `concurrency` parameter to limit the number of simultaneous inner subscriptions.

```java
// Java – fetch up to 8 URLs concurrently
Flux<Response> responses = Flux.fromIterable(urls)
    .flatMap(url -> httpClient.get(url), 8); // concurrency = 8
```

```kotlin
// Kotlin
val responses: Flux<Response> = Flux.fromIterable(urls)
    .flatMap({ url -> httpClient.get(url) }, 8)
```

| Operator | Order | Concurrency | Use when |
|---|---|---|---|
| `flatMap` | Interleaved | Concurrent | Parallel I/O, order unimportant |
| `concatMap` | Preserved | Sequential | Order matters |
| `flatMapSequential` | Preserved | Concurrent | Both |

---

### `concatMap`

**Marble diagram:** Maps each item to an inner publisher and subscribes to them **one at a time**, in order. The next inner publisher is only subscribed after the previous one completes.

```
──[A]──[B]──[C]──|
    concatMap(x -> fetchAsync(x))
──[a1]──[a2]──[b1]──[b2]──[c1]──|  (sequential, ordered)
```

**When to use:** When order must be preserved and inner operations must not overlap—e.g., processing database records in order, chained dependent API calls.

```java
// Java
Flux<Result> ordered = Flux.range(1, 5)
    .concatMap(id -> repository.findById(id));
```

```kotlin
// Kotlin
val ordered: Flux<Result> = Flux.range(1, 5)
    .concatMap { id -> repository.findById(id) }
```

> **Note:** `concatMap` has lower throughput than `flatMap` due to sequential inner subscriptions. Use `flatMapSequential` when you need both concurrency and ordering.

---

### `switchMap`

**Marble diagram:** Maps each upstream item to an inner publisher. When a new upstream item arrives, the **previous inner publisher is cancelled** and the new one is subscribed. Only the latest inner publisher is active at any time.

```
──[A]─────[B]──────[C]──|
    switchMap(x -> slowFetch(x))
──[a1]──X  [b1]──X  [c1]──[c2]──|
```

**When to use:** "Latest wins" semantics—user search-as-you-type (cancel the previous search when a new character is typed), live configuration reloads, or any scenario where older requests are stale.

```java
// Java – search-as-you-type
Flux<SearchResult> results = userInput
    .switchMap(query -> searchService.search(query));
```

```kotlin
// Kotlin
val results: Flux<SearchResult> = userInput
    .switchMap { query -> searchService.search(query) }
```

---

### `flatMapMany`

**Marble diagram:** On a `Mono<T>`, maps the single item to a `Publisher<R>`, returning a `Flux<R>`.

```
Mono[A]──|
    flatMapMany(a -> Flux.just(a1, a2, a3))
──[a1]──[a2]──[a3]──|
```

**When to use:** Expanding a single value from a `Mono` into a stream—e.g., a single user ID from an auth token expanded into a stream of their orders.

```java
// Java
Flux<Order> orders = Mono.just(userId)
    .flatMapMany(id -> orderRepository.findByUserId(id));
```

```kotlin
// Kotlin
val orders: Flux<Order> = Mono.just(userId)
    .flatMapMany { id -> orderRepository.findByUserId(id) }
```

---

### `flatMapSequential`

**Marble diagram:** Concurrently subscribes to all inner publishers (like `flatMap`), but buffers results and emits them in the **order of subscription** (like `concatMap`).

```
──[A]──[B]──[C]──|
    flatMapSequential(x -> fetchAsync(x))
──[a1]──[a2]──[b1]──[b2]──[c1]──|  (concurrent fetch, ordered emit)
```

**When to use:** When you want the throughput of concurrent execution but need results in original order—e.g., parallel database lookups where result order must match input order.

```java
// Java
Flux<Data> ordered = Flux.range(1, 100)
    .flatMapSequential(id -> service.fetch(id), 16);
```

---

### `transform`

**Marble diagram:** Applies a reusable `Function<Flux<T>, Publisher<R>>` at assembly time. Used to compose operator pipelines into reusable functions.

**When to use:** Extracting common operator chains into reusable pipeline components (e.g., a standard retry+timeout policy).

```java
// Java – define a reusable policy
Function<Flux<String>, Flux<String>> retryPolicy = flux -> flux
    .retry(3)
    .timeout(Duration.ofSeconds(5));

Flux<String> result = Flux.just("a", "b", "c")
    .transform(retryPolicy);
```

```kotlin
// Kotlin
val retryPolicy: (Flux<String>) -> Flux<String> = { flux ->
    flux.retry(3).timeout(Duration.ofSeconds(5))
}
val result: Flux<String> = Flux.just("a", "b", "c")
    .transform(retryPolicy)
```

> **vs `as`:** `transform` is evaluated at assembly time like `as`, but its return type must be `Publisher<R>`, while `as` can return any type (e.g., `StepVerifier`, a wrapper object).

---

### `as`

**Marble diagram:** Converts the `Flux` to any arbitrary type via a `Function<Flux<T>, P>`. Evaluated eagerly at assembly time.

**When to use:** Adapting a `Flux` to a non-Reactor type—e.g., converting to a test verifier, wrapping in a custom class, or integrating with a library that accepts a different type.

```java
// Java
StepVerifier verifier = Flux.just(1, 2, 3)
    .as(StepVerifier::create);

// Or wrapping in a domain object
MyStream<Integer> stream = Flux.just(1, 2, 3)
    .as(MyStream::new);
```

---

### `handle`

**Marble diagram:** Combines `map` and `filter` in a single pass. For each item, calls a `BiConsumer<T, SynchronousSink<R>>`. The sink can call `next()` (emit), `complete()` (end the sequence), or neither (skip the item silently).

```
──[1]──[2]──[3]──[4]──|
    handle((item, sink) -> { if (item % 2 == 0) sink.next(item * 10); })
──[20]──[40]──|
```

**When to use:** When you need to both transform and conditionally filter in one step—avoids chaining `map` then `filter`. Also useful for extracting `Optional<R>` values without `filter(Optional::isPresent).map(Optional::get)`.

```java
// Java – flatten Optional results
Flux<String> result = Flux.just("a", "bb", "ccc")
    .handle((s, sink) -> {
        if (s.length() > 1) sink.next(s.toUpperCase());
        // else: silently skip
    });
```

```kotlin
// Kotlin
val result: Flux<String> = Flux.just("a", "bb", "ccc")
    .handle { s, sink ->
        if (s.length > 1) sink.next(s.uppercase())
    }
```

---

### `cast`

**Marble diagram:** Casts each item to the target type using `Class.cast()`. If an item is not assignable, emits a `ClassCastException` error.

**When to use:** When you have a `Flux<Object>` or `Flux<Base>` and know at runtime that all items are of type `Derived`. Prefer this over unchecked casts; the error surfaces clearly.

```java
// Java
Flux<Animal> animals = Flux.just(new Dog(), new Dog());
Flux<Dog> dogs = animals.cast(Dog.class);
```

> **Alternative:** Use `ofType(Class)` to **filter** items by type and cast—it silently skips items that don't match, instead of erroring.

---

### `expand` / `expandDeep`

**Marble diagram:**
- `expand`: Breadth-first traversal. Emits the root, then all children at depth 1, then depth 2, etc.
- `expandDeep`: Depth-first traversal. Emits the root, then follows the first child all the way down before backtracking.

**When to use:** Traversing hierarchical or recursive data structures: file system trees, organizational charts, dependency graphs, comment threads.

```java
// Java – BFS directory traversal
Flux<Path> allFiles = Flux.just(rootPath)
    .expand(path -> Flux.fromStream(
        Files.isDirectory(path) ? Files.list(path) : Stream.empty()
    ));

// DFS
Flux<Path> dfsFiles = Flux.just(rootPath)
    .expandDeep(path -> Flux.fromStream(
        Files.isDirectory(path) ? Files.list(path) : Stream.empty()
    ));
```

---

### `collectList` / `collectMap` / `collectMultimap`

**Marble diagram:** Consumes all items from the `Flux` and gathers them into a single collection returned as a `Mono`.

```
──[A]──[B]──[C]──|
    collectList()
──[List(A,B,C)]──|
```

**When to use:** When you need all items before proceeding—e.g., batching for a bulk API call, grouping results. **Caution:** buffers the entire sequence in memory; not suitable for unbounded streams.

```java
// Java
Mono<List<String>> list = Flux.just("a", "b", "c").collectList();

Mono<Map<String, User>> byId = Flux.fromIterable(users)
    .collectMap(User::getId);

Mono<Map<String, List<User>>> byDept = Flux.fromIterable(users)
    .collectMultimap(User::getDepartment);
```

```kotlin
// Kotlin
val list: Mono<List<String>> = Flux.just("a", "b", "c").collectList()
val byId: Mono<Map<String, User>> = Flux.fromIterable(users).collectMap(User::id)
```

---

## 3. Filtering Operators

### `filter`

**Marble diagram:** Passes only items that satisfy the predicate; drops the rest silently.

```
──[1]──[2]──[3]──[4]──[5]──|
    filter(x -> x % 2 == 0)
──[2]──[4]──|
```

**When to use:** The most common filtering operator. The predicate must be synchronous and non-blocking. For async filtering, use `filterWhen`.

```java
// Java
Flux<Integer> evens = Flux.range(1, 10).filter(n -> n % 2 == 0);
```

```kotlin
// Kotlin
val evens: Flux<Int> = Flux.range(1, 10).filter { it % 2 == 0 }
```

---

### `take` / `takeLast`

**Marble diagram:**
- `take(n)`: Emits at most N items then completes (cancels upstream).
- `take(Duration)`: Emits items for the given duration, then completes.
- `takeLast(n)`: Buffers the last N items, emits them after upstream completes.

```
take(3):  ──[A]──[B]──[C]──[D]──[E]──|  →  ──[A]──[B]──[C]──|
takeLast(2):  ──[A]──[B]──[C]──|  →  ──[B]──[C]──|
```

**When to use:** `take(n)` for pagination or bounding infinite sequences. `take(Duration)` with `interval` for time-windowed polling. `takeLast(n)` for recent-item summaries (buffers all items until completion—use with caution on large sequences).

```java
// Java
Flux<Long> fiveSeconds = Flux.interval(Duration.ofMillis(100))
    .take(Duration.ofSeconds(5));

Flux<String> lastThree = eventStream.takeLast(3);
```

---

### `skip` / `skipLast`

**Marble diagram:**
- `skip(n)`: Drops the first N items, passes the rest.
- `skip(Duration)`: Drops items emitted within the first duration.
- `skipLast(n)`: Drops the last N items (buffers N items ahead).

```
skip(2):  ──[A]──[B]──[C]──[D]──|  →  ──[C]──[D]──|
```

**When to use:** `skip(n)` for offset-based pagination. `skip(Duration)` for warm-up periods (e.g., ignore first second of metrics). `skipLast(n)` when the last N items are trailing noise.

```java
// Java – page 3 of 10-item pages
Flux<Item> page3 = Flux.fromIterable(allItems)
    .skip(20)
    .take(10);
```

---

### `distinct`

**Marble diagram:** Tracks all emitted keys (using `equals`/`hashCode`) in a `HashSet`. Drops any item whose key has been seen before.

```
──[A]──[B]──[A]──[C]──[B]──|
    distinct()
──[A]──[B]──[C]──|
```

**When to use:** Global deduplication across the entire sequence. **Warning:** keeps all seen keys in memory. Avoid on unbounded streams unless you supply a custom key extractor with bounded cardinality.

```java
// Java
Flux<String> unique = Flux.just("a", "b", "a", "c").distinct();

// With key extractor
Flux<User> uniqueByEmail = users.distinct(User::getEmail);
```

---

### `distinctUntilChanged`

**Marble diagram:** Drops an item only if it equals the **immediately preceding** item. No memory grows over time—purely stateless.

```
──[A]──[A]──[B]──[B]──[A]──|
    distinctUntilChanged()
──[A]──[B]──[A]──|
```

**When to use:** Suppressing repeated identical values in a stream of state changes or sensor readings. Prefer this over `distinct` for unbounded streams.

```java
// Java
Flux<Status> changes = statusStream.distinctUntilChanged();

// With key extractor
Flux<Event> keyChanges = events.distinctUntilChanged(Event::getType);
```

---

### `elementAt`

**Marble diagram:** Emits the item at the given zero-based index, then completes. Errors if the sequence ends before reaching that index (or returns a default value if provided).

```
──[A]──[B]──[C]──[D]──|
    elementAt(2)
──[C]──|
```

**When to use:** When you need a specific positional item. Prefer `next()` for the first item.

```java
// Java
Mono<String> third = Flux.just("a", "b", "c", "d").elementAt(2);

// With fallback
Mono<String> withDefault = flux.elementAt(99, "default");
```

---

### `single`

**Marble diagram:** Emits the one and only item. If the sequence has zero items, emits `NoSuchElementException`. If it has two or more, emits `IndexOutOfBoundsException`.

**When to use:** When business logic requires exactly one result—e.g., querying by a unique key. Forces an error contract that helps detect data integrity issues early.

```java
// Java
Mono<User> user = userRepository.findByEmail("alice@example.com")
    .single(); // errors if 0 or 2+ results
```

> **Alternative:** `next()` takes the first item without erroring on multiples.

---

### `next`

**Marble diagram:** Takes the first item from the `Flux` as a `Mono`, cancels the upstream immediately after.

```
──[A]──[B]──[C]──|
    next()
Mono[A]──|  (upstream cancelled)
```

**When to use:** When you only need the first result and want to cancel the rest of the stream immediately. Equivalent to `take(1).singleOrEmpty()`.

```java
// Java
Mono<String> first = Flux.just("a", "b", "c").next();
```

---

### `ignoreElements`

**Marble diagram:** Discards all `onNext` signals, passes through only `onComplete` or `onError`.

```
──[A]──[B]──[C]──|
    ignoreElements()
──|
```

**When to use:** When you care only about completion or error—e.g., waiting for a `Flux` of write operations to finish without processing their acknowledgments.

```java
// Java
Mono<Void> done = writeOperations.ignoreElements().then();
```

---

### `takeWhile` / `skipWhile`

**Marble diagram:**
- `takeWhile`: Emits items while the predicate is true; stops (completes) as soon as it is false.
- `skipWhile`: Drops items while the predicate is true; passes all items once it turns false.

```
takeWhile(x -> x < 4):  ──[1]──[2]──[3]──[4]──[5]──|  →  ──[1]──[2]──[3]──|
skipWhile(x -> x < 4):  ──[1]──[2]──[3]──[4]──[5]──|  →  ──[4]──[5]──|
```

**When to use:** `takeWhile` for processing until a sentinel condition (e.g., stop when price exceeds threshold). `skipWhile` for skipping a header section in a data stream.

```java
// Java
Flux<Integer> ascending = Flux.just(1, 2, 3, 5, 2, 8)
    .takeWhile(n -> n < 5); // stops at 5, does not resume

Flux<String> afterHeader = lines
    .skipWhile(line -> line.startsWith("#")); // skip comment lines
```

---

### `takeUntilOther`

**Marble diagram:** Emits items from the source until the `other` publisher emits or completes, then completes.

```
source: ──[A]──[B]──[C]──[D]──|
other:  ──────────────[X]──|
    takeUntilOther(other)
──[A]──[B]──[C]──|
```

**When to use:** Externally cancelling a stream on a signal—e.g., stop a polling loop when a shutdown event arrives, or stop streaming data when the user navigates away.

```java
// Java
Flux<Data> stream = dataStream
    .takeUntilOther(shutdownSignal);
```

---

## 4. Combining Operators

### `zip` / `zipWith`

**Marble diagram:** Waits for one item from **each** source, combines them with the combinator, emits the combined result. Stops when the **shortest** source completes.

```
A: ──[1]──────[2]──────[3]──|
B: ──[a]──[b]──────[c]──────|
    zip(A, B, (n, s) -> n + s)
──[1a]──[2b]──[3c]──|
```

**When to use:** Combining results from multiple parallel async operations that must be paired by position—e.g., a user record and their settings fetched simultaneously.

```java
// Java
Mono<UserProfile> profile = Mono.zip(
    userService.findUser(id),
    settingsService.findSettings(id),
    (user, settings) -> new UserProfile(user, settings)
);

// Instance method
Flux<Pair<Integer, String>> zipped = Flux.range(1, 3)
    .zipWith(Flux.just("a", "b", "c"), Pair::of);
```

```kotlin
// Kotlin
val profile: Mono<UserProfile> = Mono.zip(
    userService.findUser(id),
    settingsService.findSettings(id)
).map { (user, settings) -> UserProfile(user, settings) }
```

---

### `merge`

**Marble diagram:** Subscribes to **all** publishers simultaneously and interleaves their items as they arrive. Completes when **all** sources complete.

```
A: ──[A1]──────[A2]──|
B: ──────[B1]──────[B2]──|
    merge(A, B)
──[A1]──[B1]──[A2]──[B2]──|
```

**When to use:** Combining multiple independent event streams into one—e.g., merging events from multiple message queue partitions, or aggregating WebSocket messages from multiple connections.

```java
// Java
Flux<Event> combined = Flux.merge(
    kafkaPartition0.consume(),
    kafkaPartition1.consume(),
    kafkaPartition2.consume()
);
```

> **vs `concat`:** `merge` subscribes to all sources immediately (parallel); `concat` subscribes one at a time (sequential).

---

### `mergeSequential`

**Marble diagram:** Subscribes to all publishers concurrently (like `merge`) but buffers inner results to emit them in the **order of subscription** (like `concatMap`).

**When to use:** When you need the throughput of concurrent subscriptions but require results in a deterministic order.

```java
// Java
Flux<Result> results = Flux.mergeSequential(
    service.fetchA(),
    service.fetchB(),
    service.fetchC()
);
```

---

### `concat`

**Marble diagram:** Subscribes to publishers **one at a time**, in order. The next publisher is only subscribed after the previous one completes.

```
A: ──[A1]──[A2]──|
B: ──[B1]──[B2]──|
    concat(A, B)
──[A1]──[A2]──[B1]──[B2]──|
```

**When to use:** Sequential processing of multiple sources where order is important—e.g., process current items, then archived items.

```java
// Java
Flux<Item> all = Flux.concat(
    recentItemsRepository.findAll(),
    archivedItemsRepository.findAll()
);
```

---

### `combineLatest`

**Marble diagram:** When **any** source emits, combines the latest value from **all** sources and emits the result. Requires at least one item from each source before any output is produced.

```
A: ──[1]──────────[3]──|
B: ──────[x]──[y]──────|
    combineLatest(A, B, combine)
──────[1x]──[1y]──[3y]──|
```

**When to use:** Reactive UI state—combine multiple form fields or filters where any change should trigger a new result. Also useful for continuously-updating metric combinations.

```java
// Java
Flux<FilteredResults> results = Flux.combineLatest(
    searchTerms,
    categoryFilter,
    sortOrder,
    (term, cat, sort) -> searchService.search(term, cat, sort)
).flatMap(Function.identity()); // flatten the inner Mono
```

---

### `withLatestFrom`

**Marble diagram:** Emits a combined value **only when the main (left) source emits**, using the latest value from the `other` (right) source. If `other` has not emitted yet, items from the main source are dropped.

```
main:  ──[A]──[B]──────[C]──|
other: ──────────[x]──[y]───|
    withLatestFrom(other, combine)
──────────────[B/x]──[C/y]──|
```

**When to use:** Combining a trigger stream with a continuously-updated reference value—e.g., sampling the latest price whenever a trade event fires.

```java
// Java
Flux<Trade> enriched = tradeEvents
    .withLatestFrom(priceStream, (trade, price) -> trade.withPrice(price));
```

---

### `startWith`

**Marble diagram:** Prepends one or more items before the existing `Flux` sequence.

```
──[B]──[C]──|
    startWith(A)
──[A]──[B]──[C]──|
```

**When to use:** Providing an initial/default value before actual data arrives—e.g., emitting a `Loading` state before async data, or prepending a header record.

```java
// Java
Flux<State> ui = dataStream
    .map(State::loaded)
    .startWith(State.loading());
```

```kotlin
// Kotlin
val ui: Flux<State> = dataStream
    .map { State.loaded(it) }
    .startWith(State.loading())
```

---

### `switchOnFirst`

**Marble diagram:** Receives the first element (and the remaining `Flux`) and lets you decide how to process the rest based on that first element.

**When to use:** Protocol negotiation patterns—inspect the first message to determine routing or transformation for the rest of the stream. More efficient than `take(1)` + `concat` because it does not resubscribe.

```java
// Java
Flux<Data> result = dataStream.switchOnFirst((firstSignal, flux) -> {
    if (firstSignal.hasValue() && firstSignal.get().isCompressed()) {
        return flux.map(Data::decompress);
    }
    return flux; // pass through unchanged
});
```

---

## 5. Error Handling Operators

### `onErrorReturn`

**Marble diagram:** If an error occurs, emits a fallback value and then completes normally.

```
──[A]──[B]──X(err)
    onErrorReturn("fallback")
──[A]──[B]──[fallback]──|
```

**When to use:** Providing a default value on failure—e.g., returning an empty list or a sentinel value when a service is unavailable. Simple, but does not allow retrying the original source.

```java
// Java
Mono<List<Product>> products = productService.findAll()
    .onErrorReturn(List.of());

// With predicate – only catch specific errors
Mono<String> cached = remoteService.fetchData()
    .onErrorReturn(TimeoutException.class, "cached-default");
```

```kotlin
// Kotlin
val products: Mono<List<Product>> = productService.findAll()
    .onErrorReturn(emptyList())
```

---

### `onErrorResume`

**Marble diagram:** If an error occurs, switches to a fallback publisher instead of propagating the error.

```
──[A]──[B]──X(err)
    onErrorResume(e -> fallbackFlux)
──[A]──[B]──[F1]──[F2]──|
```

**When to use:** Fallback to a cache, secondary service, or default publisher on failure. More powerful than `onErrorReturn` because the fallback is itself a publisher (can do async work, retry with a different source, etc.).

```java
// Java
Mono<User> user = primaryService.findUser(id)
    .onErrorResume(ServiceUnavailableException.class,
        e -> cacheService.findUser(id));
```

```kotlin
// Kotlin
val user: Mono<User> = primaryService.findUser(id)
    .onErrorResume(ServiceUnavailableException::class.java) { _ ->
        cacheService.findUser(id)
    }
```

> **vs `retry`:** `onErrorResume` switches to a *different* source; `retry` resubscribes to the *same* source.

---

### `onErrorMap`

**Marble diagram:** Transforms the error type without changing the terminal behavior—the sequence still ends with an error, just a different one.

```
──[A]──X(SQLException)
    onErrorMap(e -> new DataAccessException(e))
──[A]──X(DataAccessException)
```

**When to use:** Translating low-level infrastructure errors to domain exceptions at service boundaries. Keeps implementation details from leaking to callers.

```java
// Java
Mono<Account> account = jdbcRepository.findAccount(id)
    .onErrorMap(SQLException.class,
        e -> new AccountNotFoundException("Account " + id + " not found", e));
```

---

### `onErrorComplete`

**Marble diagram:** Swallows the error and completes the sequence normally (as if no error occurred).

```
──[A]──[B]──X(err)
    onErrorComplete()
──[A]──[B]──|
```

**When to use:** Very sparingly—only when an error genuinely means "end of stream" and callers do not need to know about it (e.g., a connection closed signal in a protocol). Silently hiding errors is almost always wrong in business logic.

```java
// Java
Flux<Frame> frames = wsConnection.frames()
    .onErrorComplete(ClosedChannelException.class);
```

---

### `retry`

**Marble diagram:** On error, resubscribes to the upstream publisher up to N times. Each retry restarts from the beginning.

```
source: ──[A]──X → retry → ──[A]──X → retry → ──[A]──[B]──|
    retry(2)
──[A]──[A]──[A]──[B]──|
```

**When to use:** Simple retry for transient errors on cold publishers (publishers that restart on resubscription—HTTP calls, DB queries). **Do not use on hot publishers** (you'll miss events). Always pair with error-type filtering via `retryWhen`.

```java
// Java
Mono<Response> response = httpClient.get("/api/data")
    .retry(3); // retry up to 3 times on any error
```

> **Warning:** Unlimited `retry()` (no argument) will retry forever and should never be used in production.

---

### `retryWhen`

**Marble diagram:** Advanced retry control using a `Retry` spec. The spec receives the error signal stream and returns a companion publisher that controls when/whether to retry.

**When to use:** Production retry logic with exponential backoff, jitter, maximum attempts, and error-type filtering. Reactor's `Retry` builder (from `reactor-extra` or built-in since 3.3.4) provides all common patterns.

```java
// Java – exponential backoff with jitter, 3 attempts, only for 5xx errors
Retry retrySpec = Retry.backoff(3, Duration.ofMillis(100))
    .maxBackoff(Duration.ofSeconds(5))
    .jitter(0.5)
    .filter(e -> e instanceof ServerErrorException)
    .onRetryExhaustedThrow((spec, signal) ->
        new ServiceException("Exhausted after " + signal.totalRetries() + " retries", signal.failure()));

Mono<Response> robust = httpClient.get("/api/data")
    .retryWhen(retrySpec);
```

```kotlin
// Kotlin
val retrySpec: Retry = Retry.backoff(3, Duration.ofMillis(100))
    .maxBackoff(Duration.ofSeconds(5))
    .jitter(0.5)
    .filter { it is ServerErrorException }

val robust: Mono<Response> = httpClient.get("/api/data")
    .retryWhen(retrySpec)
```

| Parameter | Purpose |
|---|---|
| `backoff(n, minBackoff)` | Exponential backoff starting at `minBackoff` |
| `maxBackoff(Duration)` | Cap the maximum backoff delay |
| `jitter(factor)` | Randomize delay by ±factor (0.0–1.0) |
| `filter(Predicate<Throwable>)` | Only retry for specific error types |
| `onRetryExhaustedThrow` | Custom exception after all retries fail |

---

### `doOnError`

**Marble diagram:** Executes a side-effect consumer when an error passes through. The error is **not caught**—it continues to propagate downstream.

**When to use:** Logging errors without altering the error flow. Always pair with a proper error handler downstream.

```java
// Java
Mono<User> user = userService.findUser(id)
    .doOnError(e -> log.error("Failed to find user {}: {}", id, e.getMessage()));
```

```kotlin
// Kotlin
val user: Mono<User> = userService.findUser(id)
    .doOnError { e -> log.error("Failed to find user {}: {}", id, e.message) }
```

---

### `timeout`

**Marble diagram:** If no item is emitted within the specified `Duration`, emits a `TimeoutException` error. Cancels the upstream.

```
──[A]─────────────────── (no items for 5s)
    timeout(Duration.ofSeconds(5))
──[A]──X(TimeoutException)
```

**When to use:** Bounding slow upstream operations. Always use in production HTTP clients, database queries, and any I/O that could hang indefinitely.

```java
// Java – timeout with fallback
Mono<Response> response = httpClient.get("/api/slow")
    .timeout(Duration.ofSeconds(3))
    .onErrorResume(TimeoutException.class, e -> cacheClient.get("/api/slow"));

// Per-item timeout (Flux variant)
Flux<Data> stream = dataStream
    .timeout(Duration.ofSeconds(1)); // error if no item for 1s between items
```

```kotlin
// Kotlin
val response: Mono<Response> = httpClient.get("/api/slow")
    .timeout(Duration.ofSeconds(3))
    .onErrorResume(TimeoutException::class.java) { cacheClient.get("/api/slow") }
```

---

## 6. Utility / Side-Effect Operators

### `doOnNext`

**Marble diagram:** Executes a `Consumer<T>` for each item passing through. The item is unchanged and continues downstream.

**When to use:** Logging, metrics counters, debug printing, or auditing without modifying the stream. All `doOn*` operators are "peek" operators—they do not affect data flow.

```java
// Java
Flux<Order> orders = orderStream
    .doOnNext(order -> log.debug("Processing order: {}", order.getId()))
    .doOnNext(order -> metrics.increment("orders.processed"));
```

```kotlin
// Kotlin
val orders: Flux<Order> = orderStream
    .doOnNext { order -> log.debug("Processing order: {}", order.id) }
```

> **Warning:** Do not perform blocking operations in `doOnNext`. Use `flatMap` with `fromCallable` + `subscribeOn` for blocking side effects.

---

### `doOnSubscribe`

**Marble diagram:** Executes a `Consumer<Subscription>` when a subscriber subscribes. Fires before any items are requested.

**When to use:** Logging subscription start, initializing resources, or recording subscription timestamps for latency tracking.

```java
// Java
Flux<Data> traced = dataStream
    .doOnSubscribe(sub -> log.info("Subscription started at {}", Instant.now()));
```

---

### `doOnComplete` / `doOnTerminate` / `doFinally`

| Operator | Fires on | Receives |
|---|---|---|
| `doOnComplete()` | `onComplete` only | Nothing |
| `doOnTerminate()` | `onComplete` or `onError` | Nothing |
| `doFinally(SignalType)` | Complete, error, **or cancel** | `SignalType` |

**When to use:**
- `doOnComplete`: Log successful completion.
- `doOnTerminate`: Run cleanup on any terminal signal (but not cancel).
- `doFinally`: The most complete hook—fires on complete, error, **and cancel**. Use this for resource cleanup or metrics that must run no matter how the stream ends.

```java
// Java
Flux<Event> monitored = eventStream
    .doOnComplete(() -> log.info("Stream completed normally"))
    .doOnTerminate(() -> metrics.record("stream.ended"))
    .doFinally(signal -> {
        log.info("Stream ended with signal: {}", signal);
        connectionPool.release();
    });
```

```kotlin
// Kotlin
val monitored: Flux<Event> = eventStream
    .doFinally { signal -> connectionPool.release() }
```

---

### `doOnDiscard`

**Marble diagram:** Registers a cleanup callback for items that are discarded by operators (not emitted to subscribers)—typically items dropped by bounded buffers, `filter`, or `take` operators.

**When to use:** Resource management for items that hold resources (e.g., pooled byte buffers, file handles). Prevents resource leaks in backpressure scenarios. Especially important with `FluxSink.OverflowStrategy.DROP/LATEST`.

```java
// Java – release pooled ByteBuf on discard
Flux<ByteBuf> buffers = source
    .doOnDiscard(ByteBuf.class, ByteBuf::release)
    .filter(buf -> buf.readableBytes() > 0);
```

> **Note:** `doOnDiscard` hooks are registered globally for the pipeline segment. Place it early in the chain to cover all downstream operators.

---

### `log`

**Marble diagram:** Logs all reactive signals—`subscribe`, `request(n)`, `onNext`, `onComplete`, `onError`, `cancel`—using a `Logger` at `DEBUG` level by default.

**When to use:** Debugging reactive pipelines in development. Remove or guard behind `if (log.isDebugEnabled())` in production due to high verbosity.

```java
// Java
Flux<Integer> debugged = Flux.range(1, 5)
    .log("com.example.myFlux")        // category becomes the logger name
    .map(n -> n * 2)
    .log("com.example.afterMap");     // log at multiple points
```

Sample output:
```
[com.example.myFlux] | onSubscribe([SynchronousSubscription])
[com.example.myFlux] | request(unbounded)
[com.example.myFlux] | onNext(1)
[com.example.afterMap] | onNext(2)
...
```

---

### `checkpoint`

**Marble diagram:** Does not modify the data stream. Adds an assembly-time stack trace hint that appears in error stack traces, making it easier to locate where in the pipeline an error originated.

**When to use:** Debugging in development and staging environments. Without `checkpoint`, Reactor operator fusion can make stack traces nearly unreadable. Place at key pipeline boundaries.

```java
// Java
Flux<Data> pipeline = source
    .map(this::parse)
    .checkpoint("after parse")      // lightweight: description only
    .flatMap(this::enrich)
    .checkpoint("after enrich", true); // forceStackTrace: includes assembly stack
```

> **Note:** `checkpoint` with `forceStackTrace=true` has a measurable overhead. Use the description-only form (default) in production if needed; reserve force-stack for development.

---

### `timestamp` / `elapsed` / `timed`

| Operator | Emits | Use for |
|---|---|---|
| `timestamp()` | `Tuple2<Long, T>` (epoch ms + item) | Absolute emission time |
| `elapsed()` | `Tuple2<Long, T>` (ms since last item + item) | Inter-item latency |
| `timed()` | `Timed<T>` (richer: elapsed, timestamp, total elapsed) | Comprehensive timing |

```java
// Java – measure inter-item latency
Flux<Tuple2<Long, Event>> latency = eventStream
    .elapsed()
    .doOnNext(t -> metrics.record("event.latency.ms", t.getT1()));

// Absolute timestamp
Flux<Tuple2<Long, Data>> timestamped = dataStream.timestamp();
long epochMs = timestamped.blockFirst().getT1();

// Timed – richer API (Reactor 3.5+)
Flux<Timed<Data>> timed = dataStream.timed();
timed.doOnNext(t -> {
    log.debug("Item: {}, elapsed: {}ms, total: {}ms",
        t.get(), t.elapsed().toMillis(), t.elapsedSinceSubscription().toMillis());
});
```

---

### `cache` / `cache(Duration)`

**Marble diagram:** Subscribes to the upstream once, replays all emitted items to every subscriber. Converts a cold publisher into a hot one.

```
source (cold): ─[1]─[2]─[3]─|
    cache()
subscriber A: ─[1]─[2]─[3]─|  (triggers upstream subscription)
subscriber B: ─[1]─[2]─[3]─|  (replayed from cache, no new upstream subscription)
```

**When to use:** Sharing expensive computations (DB queries, HTTP calls) among multiple subscribers. `cache(Duration)` adds TTL-based cache expiry.

```java
// Java
Mono<Config> config = configService.load()
    .cache(Duration.ofMinutes(5));

// All callers within 5 minutes share the same result
config.subscribe(this::applyConfig);
config.subscribe(this::validateConfig);
```

> **vs `share`:** `cache` replays to late subscribers; `share` does not (late subscribers miss past items).

---

### `share`

**Marble diagram:** Creates a multicast `Flux` that subscribes to the upstream once when the first subscriber arrives and stops when the last subscriber cancels. Late subscribers see only items emitted **after** they subscribe.

**When to use:** Broadcasting real-time events (WebSocket frames, sensor data) to multiple consumers without re-subscribing to the source per consumer. The source is live only while there are active subscribers.

```java
// Java
Flux<Event> shared = eventBusStream.share();

// Subscriber A and B both receive live events from the same upstream subscription
shared.subscribe(consumerA);
shared.subscribe(consumerB);
```

> **vs `cache`:** `share` is for live/hot streams; `cache` is for replay of completed or long-lived streams.

---

### `replay`

**Marble diagram:** Like `cache`, but with a configurable buffer. Replays the last N items to any new subscriber.

```
source: ─[1]─[2]─[3]─[4]─[5]─...
    replay(3)
late subscriber (after [5]): receives ─[3]─[4]─[5]─ then live items
```

**When to use:** Late-joining subscribers that need recent context—e.g., a new UI component that needs the last few state updates, or a secondary consumer reconnecting after a network blip.

```java
// Java
ConnectableFlux<Data> replayable = dataStream.replay(10);
replayable.connect(); // start upstream

// Late subscriber gets last 10 items immediately
replayable.subscribe(lateConsumer);
```

> **Note:** `replay()` returns a `ConnectableFlux`. Call `.connect()` to start the upstream, or use `.autoConnect(n)` to connect automatically when `n` subscribers are ready.

---

### `publish(Function)`

**Marble diagram:** Multicasts the source to a function that receives a shared `Flux`. The function can subscribe multiple times to the same source (e.g., for different operator chains) without triggering multiple upstream subscriptions.

**When to use:** Applying multiple derived operations to a single stream without duplicating the upstream—e.g., computing both an average and a max from the same source in one subscription.

```java
// Java – compute sum and count from one upstream subscription
Mono<Stats> stats = metricStream
    .publish(shared -> Mono.zip(
        shared.reduce(0L, Long::sum),
        shared.count(),
        Stats::new
    ));
```

```kotlin
// Kotlin
val stats: Mono<Stats> = metricStream.publish { shared ->
    Mono.zip(
        shared.reduce(0L, Long::sum),
        shared.count()
    ).map { (sum, count) -> Stats(sum, count) }
}
```

---

### `hide`

**Marble diagram:** Wraps the publisher in an opaque layer that prevents the Reactor operator fusion mechanism from optimizing across the boundary. Does not change the data flow.

**When to use:** Debugging—when you suspect operator fusion is hiding a bug, `hide()` disables it at that point so each operator executes independently. Also useful in benchmarks to measure individual operator cost, and to intentionally break fusion in tests.

```java
// Java – disable fusion to debug behavior
Flux<Integer> debuggable = Flux.range(1, 100)
    .hide()         // break fusion here
    .map(n -> n * 2)
    .hide()         // break fusion after map too
    .filter(n -> n > 50);
```

> **Note:** Never use `hide()` in production code for performance reasons—fusion is a significant throughput optimization. It is exclusively a debugging and testing aid.

---

## Quick Reference: Choosing the Right Operator

### Creating

| Need | Operator |
|---|---|
| Fixed known values | `just` |
| Lazy single value that may throw | `fromCallable` |
| Collection → stream | `fromIterable` |
| Existing CompletableFuture | `fromFuture` |
| Periodic ticks | `interval` |
| Integer range | `range` |
| Push-based external source | `create(FluxSink)` |
| Pull-based stateful generator | `generate(SynchronousSink)` |
| Resource lifecycle | `using` |
| Different publisher per subscriber | `defer` |

### Transforming

| Need | Operator |
|---|---|
| Sync 1:1 transform | `map` |
| Async parallel (order unimportant) | `flatMap` |
| Async sequential (order preserved) | `concatMap` |
| Async concurrent (order preserved) | `flatMapSequential` |
| Cancel previous on new item | `switchMap` |
| Mono → Flux | `flatMapMany` |
| Map + filter in one pass | `handle` |
| Reusable operator chain | `transform` |

### Error Handling

| Need | Operator |
|---|---|
| Default value on error | `onErrorReturn` |
| Fallback publisher on error | `onErrorResume` |
| Translate error type | `onErrorMap` |
| Simple retry | `retry(n)` |
| Retry with backoff/jitter | `retryWhen(Retry.backoff(...))` |
| Bound slow operations | `timeout(Duration)` |
| Log error without catching | `doOnError` |
