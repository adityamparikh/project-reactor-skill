# Schedulers and Threading

## publishOn vs subscribeOn

Most important threading concept in Reactor.

- `subscribeOn(scheduler)` — affects where the **source subscription** happens; propagates upstream; **first** `subscribeOn` (closest to source) wins.
- `publishOn(scheduler)` — switches execution thread for everything **downstream** of where it appears; multiple `publishOn` can hop threads mid-chain.

```java
// subscribeOn: moves source subscription (good for cold blocking sources)
Mono.fromCallable(() -> blockingDbCall())
    .subscribeOn(Schedulers.boundedElastic())   // subscription on boundedElastic
    .map(this::process);                        // also runs there (no publishOn)

// publishOn: switches thread mid-chain
Flux.range(1, 100)
    .publishOn(Schedulers.boundedElastic())     // I/O below
    .flatMap(this::fetchFromDb)
    .publishOn(Schedulers.parallel())           // CPU below
    .map(this::transformData);
```

`subscribeOn` changes where the *assembly* subscription starts; `publishOn` changes where *signals* flow from that point on.

## Schedulers Reference

| Scheduler | Threads | Use For |
|---|---|---|
| `Schedulers.immediate()` | caller thread | no-op; testing |
| `Schedulers.single()` | 1 shared thread | serialized work; never block |
| `Schedulers.parallel()` | N = CPU cores | CPU-bound work; `Flux.interval` |
| `Schedulers.boundedElastic()` | ≤ 10×CPU + max 100K queue | blocking I/O; JDBC; legacy APIs |
| `Schedulers.fromExecutor(exec)` | your executor | custom thread pools |
| `Schedulers.newSingle(name)` | 1 new thread | per-use single thread |
| `Schedulers.newParallel(name, n)` | n new threads | custom parallel pool |
| `Schedulers.newBoundedElastic(...)` | custom bounded elastic | tuned I/O pools |

`boundedElastic` is the recommended scheduler for blocking work. Caps thread creation at 10×CPU and queues the rest.

## Parallel Flux

`ParallelFlux` splits into "rails" for concurrent processing then merges back.

```java
Flux.range(1, 1000)
    .parallel()                          // split (default: CPU cores)
    .runOn(Schedulers.parallel())        // dispatch rails
    .map(this::cpuIntensiveWork)
    .sequential();                       // merge back

Flux.fromIterable(items)
    .parallel(4)
    .runOn(Schedulers.boundedElastic())  // 4 rails on I/O scheduler
    .flatMap(this::fetchData)
    .sequential();
```

`parallel()` alone does not change threads — only partitions. `runOn()` does the dispatch.

## Thread Safety Rules

- Never share mutable state between `onNext` calls without synchronization
- `parallel()` rail dispatch is non-deterministic — no ordering across rails
- `publishOn` creates an internal queue between upstream and downstream — inherently thread-safe handoff
- Reactor `Context` is thread-safe (immutable + copy-on-write)
- Don't call blocking APIs on `parallel()` or `single()`; reserve for `boundedElastic()`

## Common Pitfall: Multiple subscribeOn

Only the **first** `subscribeOn` (closest to the source) applies:

```java
Mono.fromCallable(() -> blockingCall())
    .subscribeOn(Schedulers.boundedElastic())   // this one wins
    .map(this::step1)
    .subscribeOn(Schedulers.parallel())          // ignored
    .map(this::step2);
```

To switch threads mid-chain, use `publishOn`, not a second `subscribeOn`.
