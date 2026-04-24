# Backpressure

## What is Backpressure?

Backpressure is the signal from subscriber to publisher: "I can handle N more items." In Reactor, subscribers request demand via `Subscription.request(n)`. Operators manage this automatically in most cases, propagating demand upstream through the chain.

When upstream produces faster than downstream can consume (e.g., a Kafka consumer feeding a slow database writer), overflow must be handled explicitly. Without an overflow strategy, unbounded buffering or `MissingBackpressureException` results.

## Overflow Strategies

```java
// Buffer overflowing items (up to maxSize, then error)
source.onBackpressureBuffer(1000)

// With overflow callback
source.onBackpressureBuffer(1000, dropped -> log.warn("Dropped: {}", dropped))

// With explicit full strategy
source.onBackpressureBuffer(1000, dropped -> {}, BufferOverflowStrategy.DROP_OLDEST)

// Drop items when downstream can't keep up
source.onBackpressureDrop()
source.onBackpressureDrop(dropped -> metrics.increment("dropped"))

// Keep only the latest item (good for UI updates, sensor readings)
source.onBackpressureLatest()

// Error immediately on overflow (good for detecting unexpected overflow)
source.onBackpressureError() // default behavior if none specified
```

`BufferOverflowStrategy` options: `ERROR` (default), `DROP_OLDEST`, `DROP_LATEST`.

## limitRate — Request Tuning

`limitRate(n)` tells upstream "request up to n items at a time" and automatically refills when consumption reaches 75% of the limit (the "replenish ratio"). This prevents thundering-herd demand spikes.

```java
// Process database results in batches of 100
databaseClient.sql("SELECT * FROM events")
    .fetch().all()
    .limitRate(100)                          // requests 100, refills at 75
    .flatMap(this::processEvent, 10);        // 10 concurrent processors

// Custom replenish ratio (second arg: low watermark)
source.limitRate(100, 50)                   // refill when 50 remain (50% threshold)
```

## prefetch Tuning

Many operators accept a `prefetch` parameter controlling how many items they request from upstream. Lower values reduce memory pressure; higher values improve throughput.

```java
// flatMap prefetch (inner publisher concurrency)
source.flatMap(
    this::fetch,
    /* maxConcurrency */ 16,
    /* prefetch */ 8           // request 8 from each inner publisher
)

// concatMap prefetch
source.concatMap(
    this::fetch,
    /* prefetch */ 1           // 1 = no lookahead (safest for ordering)
)

// subscribeOn with limited rate
source.limitRate(50)
      .subscribeOn(Schedulers.boundedElastic())
```

Default `prefetch` is typically `Queues.SMALL_BUFFER_SIZE` (256 in Reactor 3.6). Reduce for memory-sensitive pipelines or large payloads.

## LimitRequest

```java
// Hard cap on total items consumed; completes after N items
source.limitRequest(1000)

// Equivalent to:
source.take(1000)
```

`limitRequest` cancels the upstream subscription after the limit is reached, preventing unnecessary work.

## When to Apply Each Strategy

| Scenario | Strategy |
|----------|----------|
| Cannot afford to lose data (payments, events) | `onBackpressureBuffer` with large buffer |
| Latest value is all that matters (sensor, UI) | `onBackpressureLatest` |
| Old data is worthless when new arrives | `onBackpressureDrop` + metrics |
| Detect unexpected overflow as a bug | `onBackpressureError` (or none) |
| Batch DB reads | `limitRate(n)` |
| Control fan-out concurrency | `flatMap(fn, maxConcurrency, prefetch)` |

## Hot vs Cold Sources and Backpressure

Cold sources (e.g., `Flux.fromIterable`, database queries) respect backpressure natively — they produce only what is requested. Hot sources (e.g., `Sinks`, event buses, Kafka) produce independently of demand, making an explicit overflow strategy mandatory.

```java
// Hot source: must attach overflow strategy
hotSource
    .onBackpressureBuffer(500, dropped -> log.warn("Overflow: {}", dropped))
    .flatMap(this::processEvent, 8);
```
