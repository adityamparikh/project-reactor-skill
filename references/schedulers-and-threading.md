# Schedulers and Threading

## publishOn vs subscribeOn

The most important threading concept in Reactor.

- `subscribeOn(scheduler)` — affects where the **source subscription** happens; propagates upstream through the chain; only the **first** `subscribeOn` encountered (closest to the source) matters
- `publishOn(scheduler)` — switches the execution thread for everything **downstream** of where it appears; multiple `publishOn` calls can hop threads mid-chain

```java
// subscribeOn: moves source subscription (good for cold blocking sources)
Mono.fromCallable(() -> blockingDbCall())     // source
    .subscribeOn(Schedulers.boundedElastic())  // subscription happens on boundedElastic
    .map(this::process);                       // also runs on boundedElastic (no publishOn)

// publishOn: switches thread mid-chain
Flux.range(1, 100)
    .publishOn(Schedulers.boundedElastic())    // I/O work below
    .flatMap(this::fetchFromDb)
    .publishOn(Schedulers.parallel())          // CPU work below
    .map(this::transformData);
```

Key distinction: `subscribeOn` changes where the *assembly* subscription starts; `publishOn` changes where *signals* flow downstream from that point forward.

## Schedulers Reference

| Scheduler | Threads | Use For |
|-----------|---------|---------|
| `Schedulers.immediate()` | caller thread | no-op; testing |
| `Schedulers.single()` | 1 shared thread | serialized work; avoid blocking |
| `Schedulers.parallel()` | N threads (= CPU cores) | CPU-bound work; `Flux.interval` |
| `Schedulers.boundedElastic()` | up to 10×CPU + max 100K queue | blocking I/O; JDBC; legacy APIs |
| `Schedulers.fromExecutor(exec)` | your executor | custom thread pools |
| `Schedulers.newSingle(name)` | 1 new dedicated thread | per-use single thread |
| `Schedulers.newParallel(name, n)` | n new threads | custom parallel pool |
| `Schedulers.newBoundedElastic(n, q, name)` | custom bounded elastic | tuned I/O pools |

`boundedElastic` is the recommended scheduler for any blocking work. It prevents unbounded thread creation by capping at 10×CPU cores and queuing the rest.

## Parallel Flux

`ParallelFlux` splits a sequence into multiple "rails" for concurrent processing, then merges back.

```java
// Distribute work across CPU cores
Flux.range(1, 1000)
    .parallel()                         // split into rails (default: CPU cores)
    .runOn(Schedulers.parallel())       // run each rail on parallel scheduler
    .map(this::cpuIntensiveWork)
    .sequential();                      // merge back to single Flux

// With custom parallelism
Flux.fromIterable(items)
    .parallel(4)
    .runOn(Schedulers.boundedElastic()) // 4 rails, each on boundedElastic (I/O)
    .flatMap(this::fetchData)
    .sequential();
```

`parallel()` without `runOn()` does not change threads — it only partitions the sequence. `runOn()` is what actually dispatches rails to the scheduler.

## Thread Safety Rules

- Never share mutable state between `onNext` calls without synchronization
- `parallel()` rail dispatch is non-deterministic — do not assume ordering across rails
- `publishOn` creates an internal queue between upstream and downstream — inherently thread-safe handoff
- Reactor `Context` is thread-safe by design (immutable + copy-on-write semantics)
- Avoid calling blocking APIs on `parallel()` or `single()` schedulers; reserve blocking work for `boundedElastic()`

## Common Pitfall: Multiple subscribeOn

Only the **first** `subscribeOn` (nearest the source) takes effect for source subscription:

```java
Mono.fromCallable(() -> blockingCall())
    .subscribeOn(Schedulers.boundedElastic()) // this one wins
    .map(this::step1)
    .subscribeOn(Schedulers.parallel())        // ignored for source; no effect here
    .map(this::step2);
```

To switch threads mid-chain, use `publishOn`, not a second `subscribeOn`.
