# Project Reactor Operator Reference

> Target: Reactor 3.6+ / BOM 2024.0+
> Common operators across all six categories with marble descriptions, use-case guidance, and concise Java/Kotlin examples.

---

## Table of Contents

1. [Creating / Source](#1-creating--source-operators) — just, empty, error, never, defer, fromCallable, fromIterable, fromStream, fromFuture, interval, range, create, generate, using
2. [Transforming](#2-transforming-operators) — map, flatMap, concatMap, switchMap, flatMapMany, flatMapSequential, transform, as, handle, cast, expand/expandDeep, collectList/Map/Multimap
3. [Filtering](#3-filtering-operators) — filter, take/takeLast, skip/skipLast, distinct, distinctUntilChanged, elementAt, single, next, ignoreElements, takeWhile/skipWhile, takeUntilOther
4. [Combining](#4-combining-operators) — zip/zipWith, merge, mergeSequential, concat, combineLatest, withLatestFrom, startWith, switchOnFirst
5. [Error Handling](#5-error-handling-operators) — onErrorReturn, onErrorResume, onErrorMap, onErrorComplete, retry, retryWhen, doOnError, timeout
6. [Utility / Side-Effect](#6-utility--side-effect-operators) — doOnNext, doOnSubscribe, doOnComplete/doOnTerminate/doFinally, doOnDiscard, log, checkpoint, timestamp/elapsed/timed, cache, share, replay, publish(Function), hide

---

## 1. Creating / Source Operators

### `just`
Emits pre-defined items, then completes. Use when values are already in hand at assembly time. Don't use if value is expensive or may throw — use `fromCallable`.
```
just(1, 2, 3):  ──[1]──[2]──[3]──|
```
```java
Mono<String>  greeting = Mono.just("Hello, World!");
Flux<Integer> numbers  = Flux.just(1, 2, 3, 4, 5);
```
```kotlin
val greeting: Mono<String> = "Hello, World!".toMono()
```

---

### `empty`
Completes immediately, no items. Use as a "no data, no error" signal — e.g., default in `onErrorResume`.
```
empty():  ──|
```
```java
Flux<String> nothing  = Flux.empty();
Mono<String> noValue  = Mono.empty();
```

---

### `error`
Terminates immediately with the error. Use to represent a known error eagerly, or to short-circuit inside `flatMap`/`switchMap`.
```
error(ex):  ──X(ex)
```
```java
Mono<String>  failed = Mono.error(new IllegalArgumentException("invalid input"));
Flux<Integer> boom   = Flux.error(new RuntimeException("boom"));
```

---

### `never`
Never emits, never terminates. Test utility (holds subscription open, timeout testing); rarely used in production.
```
never():  ────────────────── (no end)
```
```java
Flux<String> infinite = Flux.never();
```

---

### `defer`
On each subscription, invokes the supplier to create a fresh publisher. Each subscriber gets an independent sequence. Use for **lazy evaluation** when the publisher depends on runtime state (timestamp, request-scoped value, DB connection).
```
defer(() -> p):
  sub A: ──[p-A-items]──|
  sub B: ──[p-B-items]──|
```
```java
Mono<Instant> now = Mono.defer(() -> Mono.just(Instant.now()));
now.subscribe(System.out::println); // T1
now.subscribe(System.out::println); // T2 (later)
```
> **Alternative:** `fromCallable` is simpler for a single lazy value that may throw. `defer` is more powerful (can return any publisher, including `Flux`).

---

### `fromCallable`
Invokes the `Callable` on subscription; emits single result or error.
```
fromCallable(...): on subscribe → ──[result]──|  or → ──X(exception)
```
```java
Mono<String> result = Mono.fromCallable(() -> Files.readString(Path.of("/etc/hosts")))
    .subscribeOn(Schedulers.boundedElastic());   // pair with boundedElastic for blocking
```
> **Always** pair with `.subscribeOn(Schedulers.boundedElastic())` when the callable does I/O or blocks.

---

### `fromIterable`
Iterates collection on demand, emitting one element per request. Respects backpressure natively.
```
fromIterable([A, B, C]):  ──[A]──[B]──[C]──|
```
```java
Flux<String> flux = Flux.fromIterable(List.of("Alice", "Bob", "Carol"));
```
```kotlin
val flux: Flux<String> = listOf("Alice", "Bob", "Carol").toFlux()
```

---

### `fromStream`
Iterates a Java `Stream`. **Streams are single-use** — second subscription throws `IllegalStateException`.
```java
Flux<String> flux = Flux.fromStream(() -> Stream.of("a", "b", "c")); // supplier form is safer
```
> **Warning:** Prefer `fromIterable` for replay safety. If using `Stream`, always pass a **supplier** (`() -> myStream()`) — creates a fresh stream per subscription.

---

### `fromFuture`
Bridges `CompletableFuture` to `Mono`. Use the **supplier overload** to defer future creation to subscription time:
```java
CompletableFuture<String> cf = someAsyncApi.fetchData();
Mono<String> mono = Mono.fromFuture(cf);                              // eager
Mono<String> lazy = Mono.fromFuture(() -> someAsyncApi.fetchData());  // lazy (preferred for retries)
```
```kotlin
val mono: Mono<String> = someAsyncApi.fetchData().toMono()
```

---

### `interval`
Emits 0L, 1L, 2L... at a fixed period on `Schedulers.parallel()` (or specified). Never completes on its own.
```
interval(1s):  ──[0]──1s──[1]──1s──[2]── (continues indefinitely)
```
```java
Flux<Long> ticker = Flux.interval(Duration.ofSeconds(1))
    .take(10)
    .doOnNext(i -> System.out.println("tick " + i));
```
> Emits on `parallel` by default — do not block downstream without `publishOn(Schedulers.boundedElastic())`.

---

### `range`
Emits a contiguous integer sequence.
```
range(1, 5):  ──[1]──[2]──[3]──[4]──[5]──|
```
```java
Flux<Integer> pages = Flux.range(1, 10).flatMap(httpClient::getPage);
```

---

### `create` (FluxSink)
Provides a `FluxSink` to an imperative callback. The callback calls `sink.next()`, `complete()`, or `error()` from any thread. Supports `OverflowStrategy` (BUFFER, DROP, LATEST, ERROR).

Use for bridging event-driven or callback-based APIs (listeners, WebSocket frames, message queues).
```java
Flux<Event> events = Flux.create(sink -> {
    EventListener listener = sink::next;
    eventBus.register(listener);
    sink.onDispose(() -> eventBus.unregister(listener));
}, FluxSink.OverflowStrategy.BUFFER);
```
> **Alternative:** `generate` for pull-model synchronous production. `push` for single-threaded variant.

---

### `generate` (SynchronousSink)
Calls the generator once per subscriber demand. Generator must call `sink.next()` exactly once (or `complete`/`error`). More efficient than `create` for stateful, pull-based sequences (cursors, linked lists, Fibonacci).
```
generate(...):  demand(1) → ──[A], demand(1) → ──[B], demand(1) → ──[C]──|
```
```java
Flux<Long> fibonacci = Flux.generate(
    () -> new long[]{0, 1},
    (state, sink) -> {
        sink.next(state[0]);
        return new long[]{state[1], state[0] + state[1]};
    }).take(10);
```

---

### `using`
Acquires a resource, creates a publisher, guarantees cleanup on terminate/error/cancel. Use for any closeable: DB connections, files, HTTP handles.
```java
Flux<String> lines = Flux.using(
    () -> new BufferedReader(new FileReader("/data/input.txt")),
    reader -> Flux.fromStream(reader.lines()),
    reader -> { try { reader.close(); } catch (IOException ignored) {} }
);
```
> **Alternative:** `Flux.usingWhen` for reactive cleanup (e.g., `Mono<Connection>` → async `connection.close()`).

---

## 2. Transforming Operators

### `map`
Synchronous 1:1 transformation. If the function throws, it's forwarded as `onError`.
```
──[1]──[2]──[3]──|  →  map(x -> x*2)  →  ──[2]──[4]──[6]──|
```
```java
Flux<String> upper = Flux.just("hello", "world").map(String::toUpperCase);
```
```kotlin
val upper: Flux<String> = Flux.just("hello", "world").map { it.uppercase() }
```
> Use `flatMap` when the transformation returns a `Publisher`. `map` with a function returning `Mono` gives you `Flux<Mono<T>>`.

---

### `flatMap`
Maps each item to an inner publisher, **concurrently** subscribes. Results interleave — order not guaranteed. Limit via `concurrency` param.
```
──[A]──[B]──[C]──|  →  flatMap(fetchAsync)  →  ──[b1]──[a1]──[c1]──[a2]──[b2]──| (interleaved)
```
```java
Flux<Response> responses = Flux.fromIterable(urls)
    .flatMap(url -> httpClient.get(url), 8);   // concurrency = 8
```

| Operator | Order | Concurrency | Use when |
|---|---|---|---|
| `flatMap` | Interleaved | Concurrent | Parallel I/O, order unimportant |
| `concatMap` | Preserved | Sequential | Order matters |
| `flatMapSequential` | Preserved | Concurrent | Both |

---

### `concatMap`
Maps to inner publisher, subscribes **one at a time** in order. Next inner only starts after previous completes. Lower throughput than `flatMap`.

Use when order must be preserved and inner ops must not overlap (DB writes in order, chained dependent calls).
```
──[A]──[B]──[C]──|  →  concatMap(fetchAsync)  →  ──[a1]──[a2]──[b1]──[b2]──[c1]──|
```
```java
Flux<Result> ordered = Flux.range(1, 5).concatMap(id -> repository.findById(id));
```
> If you need concurrency **and** ordering, use `flatMapSequential`.

---

### `switchMap`
On each new upstream signal, **cancels** the current inner and subscribes to the new one. Only the latest inner is active.

Use for "latest wins": search-as-you-type, live config reloads.
```
──[A]──[B]──[C]──|  →  switchMap(slowFetch)  →  ──[a1]──X [b1]──X [c1]──[c2]──|
```
```java
Flux<SearchResult> results = userInput.switchMap(searchService::search);
```

---

### `flatMapMany`
On a `Mono<T>`, maps the single item to a `Publisher<R>` returning `Flux<R>`. Use to expand a single value into a stream.
```
Mono[A]──|  →  flatMapMany(a -> Flux.just(a1, a2, a3))  →  ──[a1]──[a2]──[a3]──|
```
```java
Flux<Order> orders = Mono.just(userId).flatMapMany(orderRepository::findByUserId);
```

---

### `flatMapSequential`
Concurrent subscription (like `flatMap`) but emits in **subscription order** (like `concatMap`). Use for parallel lookups when result order must match input order.
```java
Flux<Data> ordered = Flux.range(1, 100).flatMapSequential(service::fetch, 16);
```

---

### `transform`
Applies a reusable `Function<Flux<T>, Publisher<R>>` at assembly time. Use for reusable pipeline policies.
```java
Function<Flux<String>, Flux<String>> retryPolicy = flux -> flux
    .retry(3).timeout(Duration.ofSeconds(5));

Flux<String> result = Flux.just("a", "b", "c").transform(retryPolicy);
```
> **vs `as`:** `transform` returns `Publisher<R>`; `as` returns any type.

---

### `as`
Adapts a `Flux` to any type via `Function<Flux<T>, P>`. Evaluated eagerly.
```java
StepVerifier verifier   = Flux.just(1, 2, 3).as(StepVerifier::create);
MyStream<Integer> stream = Flux.just(1, 2, 3).as(MyStream::new);
```

---

### `handle`
`(item, SynchronousSink)`: `sink.next(v)` emits, `complete()` ends, neither skips silently. Combines map+filter in one pass. Useful for `Optional<R>` flattening.
```
──[1]──[2]──[3]──[4]──|  →  handle(even → next(x*10))  →  ──[20]──[40]──|
```
```java
Flux<String> result = Flux.just("a", "bb", "ccc")
    .handle((s, sink) -> { if (s.length() > 1) sink.next(s.toUpperCase()); });
```

---

### `cast`
Cast each item via `Class.cast()`. Emits `ClassCastException` on mismatch.
```java
Flux<Dog> dogs = animals.cast(Dog.class);
```
> Use `ofType(Class)` to **filter+cast** — silently skips mismatches.

---

### `expand` / `expandDeep`
- `expand`: BFS — emits root, then children at depth 1, depth 2, etc.
- `expandDeep`: DFS — follows first child down before backtracking.

Use for hierarchies, recursive structures (file trees, org charts, dependency graphs).
```java
Flux<Path> allFiles = Flux.just(rootPath)
    .expand(p -> Flux.fromStream(Files.isDirectory(p) ? Files.list(p) : Stream.empty()));
```

---

### `collectList` / `collectMap` / `collectMultimap`
Gathers items into a `Mono` of a collection. **Buffers entire sequence** — not for unbounded streams.
```
──[A]──[B]──[C]──|  →  collectList()  →  ──[List(A,B,C)]──|
```
```java
Mono<List<String>>                list   = Flux.just("a","b","c").collectList();
Mono<Map<String, User>>           byId   = Flux.fromIterable(users).collectMap(User::getId);
Mono<Map<String, List<User>>>     byDept = Flux.fromIterable(users).collectMultimap(User::getDepartment);
```

---

## 3. Filtering Operators

### `filter`
Passes items satisfying the predicate. Sync only — use `filterWhen` for async.
```
──[1]──[2]──[3]──[4]──[5]──|  →  filter(even)  →  ──[2]──[4]──|
```
```java
Flux<Integer> evens = Flux.range(1, 10).filter(n -> n % 2 == 0);
```

---

### `take` / `takeLast`
- `take(n)`: emit ≤ N then complete (cancels upstream)
- `take(Duration)`: emit for the duration, then complete
- `takeLast(n)`: buffer last N, emit after upstream completes
```
take(3):     ──[A]──[B]──[C]──[D]──[E]──|  →  ──[A]──[B]──[C]──|
takeLast(2): ──[A]──[B]──[C]──|           →  ──[B]──[C]──|
```
```java
Flux<Long>   fiveSeconds = Flux.interval(Duration.ofMillis(100)).take(Duration.ofSeconds(5));
Flux<String> lastThree   = eventStream.takeLast(3);
```

---

### `skip` / `skipLast`
- `skip(n)` / `skip(Duration)` — drop initial items
- `skipLast(n)` — drop final N (buffers N ahead)
```
skip(2):  ──[A]──[B]──[C]──[D]──|  →  ──[C]──[D]──|
```
```java
Flux<Item> page3 = Flux.fromIterable(allItems).skip(20).take(10);
```

---

### `distinct`
Global dedup using `equals/hashCode` in a `HashSet`. **Keeps all seen keys in memory** — avoid on unbounded streams.
```
──[A]──[B]──[A]──[C]──[B]──|  →  distinct()  →  ──[A]──[B]──[C]──|
```
```java
Flux<String> unique       = Flux.just("a","b","a","c").distinct();
Flux<User>   uniqueByEmail = users.distinct(User::getEmail);
```

---

### `distinctUntilChanged`
Drops item only if equal to the **immediately preceding** one. Stateless over time — safe for unbounded streams.
```
──[A]──[A]──[B]──[B]──[A]──|  →  ──[A]──[B]──[A]──|
```
```java
Flux<Status> changes = statusStream.distinctUntilChanged();
Flux<Event>  byType  = events.distinctUntilChanged(Event::getType);
```

---

### `elementAt`
Zero-based positional item. Errors if sequence ends short, unless default provided.
```java
Mono<String> third       = Flux.just("a","b","c","d").elementAt(2);
Mono<String> withDefault = flux.elementAt(99, "default");
```

---

### `single`
Emits the one and only item. Errors if 0 (`NoSuchElementException`) or 2+ (`IndexOutOfBoundsException`). Use when business logic requires exactly one result.
```java
Mono<User> user = userRepository.findByEmail("alice@example.com").single();
```
> `next()` takes the first without erroring on multiples.

---

### `next`
First item from `Flux` as `Mono`; cancels upstream after.
```
──[A]──[B]──[C]──|  →  next()  →  Mono[A]──|
```

---

### `ignoreElements`
Discards all `onNext`, passes only `onComplete`/`onError`. Use when you only care about completion (e.g., bulk writes).
```java
Mono<Void> done = writeOperations.ignoreElements().then();
```

---

### `takeWhile` / `skipWhile`
- `takeWhile`: emit while predicate true; complete when false
- `skipWhile`: drop while true; pass-through once false
```
takeWhile(x<4):  ──[1]──[2]──[3]──[4]──[5]──|  →  ──[1]──[2]──[3]──|
skipWhile(x<4):  ──[1]──[2]──[3]──[4]──[5]──|  →  ──[4]──[5]──|
```
```java
Flux<Integer> ascending   = Flux.just(1,2,3,5,2,8).takeWhile(n -> n < 5);   // stops at 5
Flux<String>  afterHeader = lines.skipWhile(line -> line.startsWith("#"));
```

---

### `takeUntilOther`
Emits from source until `other` emits/completes. Use for external cancellation (shutdown signals).
```java
Flux<Data> stream = dataStream.takeUntilOther(shutdownSignal);
```

---

## 4. Combining Operators

### `zip` / `zipWith`
Waits for one item from **each** source, combines, emits. Stops when **shortest** completes.
```
A: ──[1]──────[2]──────[3]──|
B: ──[a]──[b]──────[c]──────|
zip(A, B):  ──[1a]──[2b]──[3c]──|
```
```java
Mono<UserProfile> profile = Mono.zip(
    userService.findUser(id),
    settingsService.findSettings(id),
    UserProfile::new);

Flux<Pair<Integer,String>> zipped = Flux.range(1,3).zipWith(Flux.just("a","b","c"), Pair::of);
```

---

### `merge`
Subscribes to **all** sources simultaneously; interleaves arrivals. Completes when all sources complete.
```
A: ──[A1]──────[A2]──|
B: ──────[B1]──────[B2]──|
merge(A, B):  ──[A1]──[B1]──[A2]──[B2]──|
```
```java
Flux<Event> combined = Flux.merge(
    kafkaPartition0.consume(), kafkaPartition1.consume(), kafkaPartition2.consume());
```
> **vs `concat`:** `merge` is parallel; `concat` is sequential.

---

### `mergeSequential`
Concurrent subscription, emits in subscription order.
```java
Flux<Result> results = Flux.mergeSequential(service.fetchA(), service.fetchB(), service.fetchC());
```

---

### `concat`
Subscribes one source at a time, in order. Next only starts after previous completes.
```
concat(A, B):  ──[A1]──[A2]──[B1]──[B2]──|
```
```java
Flux<Item> all = Flux.concat(recentItemsRepository.findAll(), archivedItemsRepository.findAll());
```

---

### `combineLatest`
Emits combined value of **latest from each** source whenever any source emits. Needs at least one item from each before any output.
```
A: ──[1]──────────[3]──|
B: ──────[x]──[y]──────|
combineLatest:  ──────[1x]──[1y]──[3y]──|
```
```java
Flux<FilteredResults> results = Flux.combineLatest(
    searchTerms, categoryFilter, sortOrder,
    (term, cat, sort) -> searchService.search(term, cat, sort))
    .flatMap(Function.identity());
```

---

### `withLatestFrom`
Emits only when **main** source emits, combined with latest from `other`. Pre-other items from main are dropped.
```
main:  ──[A]──[B]──────[C]──|
other: ──────────[x]──[y]───|
withLatestFrom(other):  ──────────[B/x]──[C/y]──|
```
```java
Flux<Trade> enriched = tradeEvents
    .withLatestFrom(priceStream, (trade, price) -> trade.withPrice(price));
```

---

### `startWith`
Prepends items before the sequence. Use for initial/default value (loading state, header).
```
──[B]──[C]──|  →  startWith(A)  →  ──[A]──[B]──[C]──|
```
```java
Flux<State> ui = dataStream.map(State::loaded).startWith(State.loading());
```

---

### `switchOnFirst`
Receives first element + remaining `Flux`; lets you decide downstream processing based on it. Protocol negotiation pattern; more efficient than `take(1) + concat`.
```java
Flux<Data> result = dataStream.switchOnFirst((firstSignal, flux) -> {
    if (firstSignal.hasValue() && firstSignal.get().isCompressed()) {
        return flux.map(Data::decompress);
    }
    return flux;
});
```

---

## 5. Error Handling Operators

### `onErrorReturn`
On error, emits a fallback value then completes. Simple, but can't retry.
```
──[A]──[B]──X(err)  →  onErrorReturn("fallback")  →  ──[A]──[B]──[fallback]──|
```
```java
Mono<List<Product>> products = productService.findAll().onErrorReturn(List.of());

// With predicate — only specific errors
Mono<String> cached = remoteService.fetchData()
    .onErrorReturn(TimeoutException.class, "cached-default");
```

---

### `onErrorResume`
On error, switches to a fallback publisher. More powerful than `onErrorReturn` (fallback is itself a publisher).
```
──[A]──[B]──X(err)  →  onErrorResume(e -> fb)  →  ──[A]──[B]──[F1]──[F2]──|
```
```java
Mono<User> user = primaryService.findUser(id)
    .onErrorResume(ServiceUnavailableException.class, e -> cacheService.findUser(id));
```
> **vs `retry`:** `onErrorResume` switches to a *different* source; `retry` resubscribes to the *same* source.

---

### `onErrorMap`
Translates the error type without recovering. Use at service boundaries to convert low-level → domain exceptions.
```java
Mono<Account> account = jdbcRepository.findAccount(id)
    .onErrorMap(SQLException.class,
        e -> new AccountNotFoundException("Account " + id + " not found", e));
```

---

### `onErrorComplete`
Swallows error, completes normally. **Very sparing use** — only when error genuinely means "end of stream" (e.g., `ClosedChannelException` on a WebSocket).
```java
Flux<Frame> frames = wsConnection.frames().onErrorComplete(ClosedChannelException.class);
```

---

### `retry`
On error, resubscribes up to N times. **Cold publishers only** (HTTP, DB queries); hot publishers lose events on resubscribe.
```
source: ──[A]──X → retry → ──[A]──X → retry → ──[A]──[B]──|
retry(2): ──[A]──[A]──[A]──[B]──|
```
```java
Mono<Response> response = httpClient.get("/api/data").retry(3);
```
> **Warning:** unlimited `retry()` retries forever — never use in production.

---

### `retryWhen`
Advanced retry with `Retry` spec: backoff, jitter, max attempts, error filtering.
```java
Retry retrySpec = Retry.backoff(3, Duration.ofMillis(100))
    .maxBackoff(Duration.ofSeconds(5))
    .jitter(0.5)
    .filter(e -> e instanceof ServerErrorException)
    .onRetryExhaustedThrow((spec, signal) ->
        new ServiceException("Exhausted after " + signal.totalRetries() + " retries", signal.failure()));

Mono<Response> robust = httpClient.get("/api/data").retryWhen(retrySpec);
```

| Parameter | Purpose |
|---|---|
| `backoff(n, minBackoff)` | Exponential backoff from `minBackoff` |
| `maxBackoff(Duration)` | Cap on backoff delay |
| `jitter(factor)` | Randomize delay ±factor (0.0–1.0) |
| `filter(Predicate)` | Only retry specific errors |
| `onRetryExhaustedThrow` | Custom exception after exhaustion |

---

### `doOnError`
Side-effect on error; **doesn't catch** — error continues. Pair with a real error handler downstream.
```java
Mono<User> user = userService.findUser(id)
    .doOnError(e -> log.error("Failed to find user {}: {}", id, e.getMessage()));
```

---

### `timeout`
If no item within `Duration`, emits `TimeoutException`. Cancels upstream. Use for HTTP, DB, any I/O that could hang.
```
──[A]─── (no items for 5s) ──  →  timeout(5s)  →  ──[A]──X(TimeoutException)
```
```java
Mono<Response> response = httpClient.get("/api/slow")
    .timeout(Duration.ofSeconds(3))
    .onErrorResume(TimeoutException.class, e -> cacheClient.get("/api/slow"));

// Flux: per-inter-item timeout
Flux<Data> stream = dataStream.timeout(Duration.ofSeconds(1));
```

---

## 6. Utility / Side-Effect Operators

### `doOnNext`
Side-effect per item; item unchanged. Use for logging, metrics, audit — never blocking.
```java
Flux<Order> orders = orderStream
    .doOnNext(order -> log.debug("Processing: {}", order.getId()))
    .doOnNext(order -> metrics.increment("orders.processed"));
```
> Don't block in `doOnNext`. For blocking side-effects, use `flatMap` + `fromCallable` + `subscribeOn`.

---

### `doOnSubscribe`
Fires when a subscriber subscribes. Use for subscription logging, timestamping.
```java
Flux<Data> traced = dataStream
    .doOnSubscribe(sub -> log.info("Subscription started at {}", Instant.now()));
```

---

### `doOnComplete` / `doOnTerminate` / `doFinally`

| Operator | Fires on | Receives |
|---|---|---|
| `doOnComplete()` | `onComplete` only | nothing |
| `doOnTerminate()` | `onComplete` or `onError` | nothing |
| `doFinally(SignalType)` | complete, error, **or cancel** | `SignalType` |

Use `doFinally` for cleanup that must run regardless of termination cause.
```java
Flux<Event> monitored = eventStream
    .doOnComplete(() -> log.info("Stream completed normally"))
    .doOnTerminate(() -> metrics.record("stream.ended"))
    .doFinally(signal -> {
        log.info("Stream ended with signal: {}", signal);
        connectionPool.release();
    });
```

---

### `doOnDiscard`
Cleanup callback for items dropped by operators (bounded buffers, `filter`, `take`, overflow strategies). Critical for pooled resources (`ByteBuf`, file handles).
```java
Flux<ByteBuf> buffers = source
    .doOnDiscard(ByteBuf.class, ByteBuf::release)
    .filter(buf -> buf.readableBytes() > 0);
```
> Place early in chain — hooks register globally for the segment.

---

### `log`
Logs all reactive signals (subscribe, request, onNext, onComplete, onError, cancel) at DEBUG.
```java
Flux<Integer> debugged = Flux.range(1, 5)
    .log("com.example.myFlux")
    .map(n -> n * 2)
    .log("com.example.afterMap");
```
Sample output:
```
[com.example.myFlux]    | onSubscribe([SynchronousSubscription])
[com.example.myFlux]    | request(unbounded)
[com.example.myFlux]    | onNext(1)
[com.example.afterMap]  | onNext(2)
```
Verbose in production — filter via `log("tag", Level.INFO, SignalType.ON_NEXT, SignalType.ON_ERROR)`.

---

### `checkpoint`
Adds assembly-time stack-trace hint into error traces. Doesn't modify data. Critical for readable traces (fusion otherwise obscures them).
```java
Flux<Data> pipeline = source
    .map(this::parse)
    .checkpoint("after parse")
    .flatMap(this::enrich)
    .checkpoint("after enrich", true);   // forceStackTrace
```
> `forceStackTrace=true` has measurable overhead. Use description-only in production.

---

### `timestamp` / `elapsed` / `timed`

| Operator | Emits | Use for |
|---|---|---|
| `timestamp()` | `Tuple2<Long, T>` (epoch ms, item) | Absolute time |
| `elapsed()` | `Tuple2<Long, T>` (ms since last, item) | Inter-item latency |
| `timed()` | `Timed<T>` (richer: elapsed, timestamp, total elapsed) | Comprehensive timing |

```java
Flux<Tuple2<Long, Event>> latency = eventStream
    .elapsed()
    .doOnNext(t -> metrics.record("event.latency.ms", t.getT1()));

// Reactor 3.5+: Timed
dataStream.timed().doOnNext(t -> log.debug(
    "Item: {}, elapsed: {}ms, total: {}ms",
    t.get(), t.elapsed().toMillis(), t.elapsedSinceSubscription().toMillis()));
```

---

### `cache` / `cache(Duration)`
Subscribes upstream once, replays all items to every subscriber. Cold → hot. TTL via `cache(Duration)`.
```
source (cold): ─[1]─[2]─[3]─|
cache(): sub A ─[1]─[2]─[3]─| (triggers upstream); sub B ─[1]─[2]─[3]─| (replayed)
```
```java
Mono<Config> config = configService.load().cache(Duration.ofMinutes(5));
config.subscribe(this::applyConfig);
config.subscribe(this::validateConfig);
```
> **vs `share`:** `cache` replays to late subscribers; `share` does not.

---

### `share`
Multicasts: subscribes upstream once on first subscriber, stops when last cancels. Late subscribers see only future items.

Use for live broadcasts (WebSocket, sensor data) to multiple consumers without re-subscribing.
```java
Flux<Event> shared = eventBusStream.share();
shared.subscribe(consumerA);
shared.subscribe(consumerB);
```

---

### `replay`
Like `cache`, configurable buffer of last N items.
```
source: ─[1]─[2]─[3]─[4]─[5]─...
replay(3): late subscriber after [5] → receives ─[3]─[4]─[5]─ then live
```
```java
ConnectableFlux<Data> replayable = dataStream.replay(10);
replayable.connect();           // start upstream
replayable.subscribe(lateConsumer);
```
> Returns `ConnectableFlux` — `.connect()` or `.autoConnect(n)` to start.

---

### `publish(Function)`
Multicasts source to a function that subscribes multiple times to the same shared `Flux` without duplicating upstream. Use for deriving multiple ops from one source.
```java
Mono<Stats> stats = metricStream.publish(shared -> Mono.zip(
    shared.reduce(0L, Long::sum),
    shared.count(),
    Stats::new));
```

---

### `hide`
Wraps publisher to **disable operator fusion**. Doesn't change data flow. Debugging/testing aid; never use in production for performance.
```java
Flux<Integer> debuggable = Flux.range(1, 100)
    .hide()
    .map(n -> n * 2)
    .hide()
    .filter(n -> n > 50);
```

---

## Quick Reference

### Creating
| Need | Operator |
|---|---|
| Fixed known values | `just` |
| Lazy single value that may throw | `fromCallable` |
| Collection → stream | `fromIterable` |
| CompletableFuture | `fromFuture` |
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
