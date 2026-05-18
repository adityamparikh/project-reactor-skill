# Backpressure

## What is Backpressure?

Backpressure is the subscriber→publisher signal: "I can handle N more items." Subscribers request demand via `Subscription.request(n)`; operators propagate demand upstream automatically.

When upstream produces faster than downstream consumes (e.g., Kafka feeding a slow DB writer), overflow must be handled. Without a strategy: unbounded buffering or `MissingBackpressureException`.

## Overflow Strategies

```java
// Buffer up to N, then error
source.onBackpressureBuffer(1000);
source.onBackpressureBuffer(1000, dropped -> log.warn("Dropped: {}", dropped));
source.onBackpressureBuffer(1000, dropped -> {}, BufferOverflowStrategy.DROP_OLDEST);

// Drop when downstream can't keep up
source.onBackpressureDrop();
source.onBackpressureDrop(dropped -> metrics.increment("dropped"));

// Keep only the latest (UI updates, sensors)
source.onBackpressureLatest();

// Error immediately on overflow (default if none specified)
source.onBackpressureError();
```

`BufferOverflowStrategy` options: `ERROR` (default), `DROP_OLDEST`, `DROP_LATEST`.

## limitRate — Request Tuning

`limitRate(n)` requests up to n items at a time and refills when consumption reaches 75% (the "replenish ratio"). Prevents thundering-herd demand.

```java
databaseClient.sql("SELECT * FROM events").fetch().all()
    .limitRate(100)                      // requests 100, refills at 75
    .flatMap(this::processEvent, 10);    // 10 concurrent processors

source.limitRate(100, 50);              // custom low watermark (50%)
```

## prefetch Tuning

Many operators accept `prefetch` controlling how many items they request from upstream. Lower = less memory pressure; higher = better throughput.

```java
source.flatMap(this::fetch, /*maxConcurrency*/ 16, /*prefetch*/ 8);
source.concatMap(this::fetch, /*prefetch*/ 1);  // 1 = no lookahead (safest)
```

Default prefetch is typically `Queues.SMALL_BUFFER_SIZE` (256 in 3.6). Reduce for memory-sensitive pipelines or large payloads.

## limitRequest

```java
source.limitRequest(1000);  // hard cap; cancels upstream after N items
// equivalent: source.take(1000);
```

## Strategy Guide

| Scenario | Strategy |
|---|---|
| Cannot afford to lose data (payments, events) | `onBackpressureBuffer` with large buffer |
| Latest value all that matters (sensor, UI) | `onBackpressureLatest` |
| Old data worthless once new arrives | `onBackpressureDrop` + metrics |
| Detect unexpected overflow as a bug | `onBackpressureError` (or none) |
| Batch DB reads | `limitRate(n)` |
| Control fan-out concurrency | `flatMap(fn, maxConcurrency, prefetch)` |

## Hot vs Cold and Backpressure

Cold sources (`Flux.fromIterable`, DB queries) respect backpressure natively — produce only what is requested. Hot sources (`Sinks`, event buses, Kafka) produce independently of demand — an explicit overflow strategy is mandatory.

```java
hotSource
    .onBackpressureBuffer(500, dropped -> log.warn("Overflow: {}", dropped))
    .flatMap(this::processEvent, 8);
```
