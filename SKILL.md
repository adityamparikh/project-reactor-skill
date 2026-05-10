---
name: project-reactor
description: Expert guidance for Project Reactor (reactor-core 3.6+, BOM 2024.0+) on the JVM (Java 17, 21, or 25; Java 25 LTS recommended). Triggers when the user writes, reviews, or debugs reactive code involving Mono, Flux, Publisher, Subscriber, Sinks, or reactive pipelines: operator selection (flatMap, concatMap, switchMap, switchIfEmpty, handle, expand, zip, merge, cache, share, replay), Schedulers and threading (publishOn, subscribeOn, boundedElastic, parallel), backpressure (onBackpressureBuffer/Drop/Latest, limitRate, prefetch), Reactor Context and tracing/MDC/security propagation, blocking-to-reactive bridging (fromCallable, fromFuture, BlockHound), Kotlin coroutine/Flow interop, Spring WebFlux, R2DBC, reactor-netty WebClient, StepVerifier/TestPublisher/PublisherProbe/virtual time, debugging (checkpoint, log, Hooks.onOperatorDebug, ReactorDebugAgent), and choosing between Reactor and JDK virtual threads (introduced in Java 21, refined through 25). Does not apply to non-reactive Spring MVC or plain JDBC/imperative code unless they bridge into a reactive pipeline.
---

# Project Reactor Skill

> Java baseline: **Java 17 minimum** (oldest supported), **Java 25 LTS recommended** for new code. Supported versions: 17, 21, 25. Virtual-thread guidance assumes Java 21+ (where `Thread.ofVirtual` first shipped) and is most ergonomic on Java 25.

## Section A: Assess the Situation

Identify the mode, then jump to the relevant section:

- **Writing new reactive code?** Section B (is Reactor right?), then Section D (operators).
- **Reviewing existing code?** Section C — work the full checklist, not just the reported concern.
- **Debugging?** Section J first. Reactive debugging needs specific tooling before reasoning about behavior.
- **Adoption decision?** Section B and `references/virtual-threads-decision.md`.
- **Imperative ↔ reactive translation?** Section E and `references/bridging.md`.
- **Kotlin?** Section F before any interop code.

## Section B: Should Reactor Be Used Here?

**Use Reactor when:**
- I/O-bound fan-out with backpressure control (e.g., calling 10 downstream services per request)
- Streaming over SSE, WebSocket, or chunked HTTP
- Many concurrent async operations where thread-per-request doesn't scale
- Producer-to-consumer backpressure must be enforced
- Building a fully async pipeline where no step may block

**Prefer virtual threads (Java 21+, ideally Java 25 LTS) when:**
- Simple CRUD with blocking JDBC/I/O and no streaming
- Team is unfamiliar with reactive; learning curve isn't justified
- Existing imperative codebase with high migration cost
- No streaming or backpressure requirements
- On Spring Boot 3.2+ — virtual threads are one config line

**Mixed case:** reactive at the transport layer (WebFlux + WebClient), synchronous or virtual-thread internals at the service boundary.

Full decision tree: `references/virtual-threads-decision.md`.

## Section C: Code Review Checklist

Check **all** of these — issues compound across dimensions.

### 1. Anti-patterns
- Nested `subscribe()` inside another `subscribe()` or inside an operator callback — breaks error propagation and backpressure
- Blocking calls (`Thread.sleep`, JDBC, blocking HTTP) inside a reactive chain on a non-`boundedElastic` thread
- Discarding the returned `Mono`/`Flux` (no subscriber)
- Shared mutable state mutated in `doOnNext`/`map` without synchronization

Catalog: `references/anti-patterns.md`.

### 2. Scheduler usage
- `publishOn` to shift execution after assembly; `subscribeOn` to shift where subscription starts (I/O sources)
- All blocking I/O wrapped in `Mono.fromCallable(...)` on `Schedulers.boundedElastic()`

Details: `references/schedulers-and-threading.md`.

### 3. Backpressure
- Overflow handled with `onBackpressureBuffer`, `onBackpressureDrop`, or `onBackpressureLatest`
- `prefetch` tuned for the workload (`flatMap(fn, concurrency, prefetch)`)

Details: `references/backpressure.md`.

### 4. Context propagation
- Reactor `Context` (not `ThreadLocal`) for MDC/tracing/security across operators
- `ReactorContextAccessor` registered for Micrometer tracing
- `ReactiveSecurityContextHolder` (not `SecurityContextHolder`) in WebFlux

Details: `references/context-propagation.md`.

### 5. Error handling
- Errors surfaced, not silently swallowed
- `doOnError` is for side effects only; recover with `onErrorResume` / `onErrorReturn`
- `onErrorResume` that re-emits the same error is a no-op and hides intent

### 6. Resource cleanup
- Files, connections, locks closed via `Flux.using()` or `doFinally()`
- Never rely on GC for resource cleanup in a reactive pipeline

### 7. Test coverage
- `StepVerifier` everywhere; never `block()` in tests
- Error paths tested, not just happy paths
- Time-based operators (`delay`, `interval`) tested with virtual time

Details: `references/testing.md`.

## Section D: Operator Guidance

When recommending an operator, cover: marble behavior, why it beats alternatives, threading and backpressure implications.

### Selection guide

| Goal | Operator | Why not the others |
|---|---|---|
| Parallel I/O, results may interleave | `flatMap` | `concatMap` is sequential; `switchMap` cancels previous |
| Sequential, order preserved | `concatMap` | `flatMap` doesn't guarantee order |
| Latest-wins (search-as-you-type) | `switchMap` | `flatMap` accumulates all; `concatMap` queues all |
| Mono → Flux (one item produces a stream) | `flatMapMany` | `flatMap` returns `Flux<Flux<T>>` without this |
| Default value / 404 handling | `switchIfEmpty` | `defaultIfEmpty` only works for scalar values |
| Map + filter in one pass | `handle` | Cleaner than chaining `map` + `filter` |
| Recursive / tree traversal | `expand` | No clean alternative |

Key behaviors to remember:

- **`flatMap`** — concurrent, unordered; default concurrency 256 (`Queues.SMALL_BUFFER_SIZE`); use `flatMap(fn, maxConcurrency)` to cap.
- **`concatMap`** — sequential, ordered; next inner waits for previous to complete.
- **`switchMap`** — cancels current inner on each new upstream signal.
- **`switchIfEmpty`** — fallback subscribed only on empty completion; cold (not eager).
- **`cache` / `share` / `replay`** — cold→hot conversion: `cache()` replays all, `share()` multicasts to current subscribers, `replay(n)` replays last `n`.
- **`handle`** — `(value, sink)` callback; emit, error, or skip — more expressive than `map` + `filter`.
- **`expand`** — breadth-first traversal; ends when produced publishers complete empty.

Full catalog with marble descriptions: `references/operators.md`.

For **programmatic emission** (callback bridges, hot streams, request-reply) use `Sinks` — the modern replacement for the deprecated `EmitterProcessor` / `FluxProcessor` / `MonoProcessor`. See `references/sinks.md`.

## Section E: Paradigm Translation

When translating, always show both versions side by side so equivalence is verifiable. Patterns covered in `references/bridging.md`:

- Imperative null-check + lookup → `switchIfEmpty` + `flatMapMany`
- Blocking JDBC/HTTP → `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`
- `CompletableFuture` → `Mono.fromFuture(supplier)` (lazy) vs `Mono.fromFuture(future)` (eager)
- Reactive → blocking with `.block()` — **only** at composition roots (CLI `main`, integration test fixtures); never in a controller or inside a chain
- Spring `@Async` interop and MVC + WebFlux coexistence

Drive test assertions with `StepVerifier` (Section I), not `.block()`. Use BlockHound (Section J) to catch accidental blocking in CI.

## Section F: Kotlin Guidance

When code is Kotlin, evaluate coroutines/Flow before reaching for Reactor directly.

**Prefer coroutines/Flow when:**
- New Kotlin code with no existing Reactor dependency
- Team knows coroutines better than Reactor operators
- Suspend functions and structured concurrency fit the design

**Use Reactor in Kotlin when:**
- Integrating Java reactive libraries that expose `Mono`/`Flux`
- Advanced backpressure or operator composition is needed
- Operators like `expand`, `handle`, `groupBy` are required

Interop touchpoints: `awaitSingle` / `awaitSingleOrNull`, `Flux.asFlow()` / `Flow.asFlux()`, `mono { ... }` / `flux { ... }` builders. Full patterns including `awaitFirst`, structured concurrency, and Flow backpressure: `references/kotlin-interop.md`.

## Section G: Data Access (R2DBC)

API choices:
- **`DatabaseClient`** — SQL-level control with manual row mapping
- **`R2dbcEntityTemplate`** — higher-level, type-safe queries
- **Reactive repositories** — Spring Data interfaces returning `Mono`/`Flux`

Things that bite:
- **Connection pool sizing** — defaults are conservative; tune `initialSize`, `maxSize`, `maxIdleTime` in `ConnectionPoolConfiguration`.
- **Transactions** — with reactive `R2dbcTransactionManager` (default when Spring Data R2DBC is on the classpath), `@Transactional` on a `Mono`/`Flux` method propagates through Reactor Context. Use `TransactionalOperator` for programmatic control. Never block inside a reactive transaction.
- **N+1 queries** — reactive repositories don't auto-join; either `flatMap` a second query or write a join in `DatabaseClient`.

Code examples (declarative vs programmatic transactions, pool tuning, repository composition): `references/r2dbc-patterns.md`.

## Section H: HTTP Client (WebClient / Reactor Netty)

Production WebClient call must include: per-status `onStatus` handlers, `bodyToMono`, `timeout`, and a filtered `retryWhen` with backoff (e.g., `Retry.backoff(3, Duration.ofMillis(100)).filter(ex -> ex instanceof IOException || ex instanceof TimeoutException)`).

Configuration to tune:
- **Connection pool** — `ReactorClientHttpConnector` with custom `ConnectionProvider` (`maxConnections`, `pendingAcquireMaxCount`, `pendingAcquireTimeout`)
- **Codecs** — `maxInMemorySize` on `WebClientCodecCustomizer` to avoid `DataBufferLimitException` on large bodies
- **Metrics** — register `reactor.netty.http.client` meters via Micrometer

Full patterns: `references/reactor-netty-http.md`.

## Section I: Testing

Never `.block()` in tests. Always `StepVerifier`. Patterns covered in `references/testing.md`:

- Happy path: `expectNext` / `expectNextMatches` + `verifyComplete`
- Error path: `expectError(...).verify()`
- Time-based operators: `StepVerifier.withVirtualTime(...)` with `thenAwait`
- Controlling upstream: `TestPublisher` (`emit`, `complete`, `error`)
- Verifying branch execution: `PublisherProbe` (`assertWasSubscribed`, `assertWasRequested`)
- HTTP layer: `WebTestClient`
- Context and security in tests

## Section J: Debugging

When something breaks, reach for tooling before reading code:

1. **`log()`** — logs `onSubscribe`, `request`, `onNext`, `onError`, `onComplete` to SLF4J on the named publisher.
2. **`checkpoint("description")`** — adds a labeled boundary; appears in assembly traces.
3. **Assembly tracing** — `Hooks.onOperatorDebug()` in development (significant overhead) or `ReactorDebugAgent.init()` (production-safe via `-javaagent`).
4. **Read traces bottom-up** — each `*__checkpoint` is a boundary in your code; the error propagated through each on its way out.
5. **BlockHound** — `BlockHound.install()` in test/dev startup throws `BlockingOperationError` on a blocking call from a non-blocking thread; allowlist JVM/`ClassLoader` false positives via `BlockHound.builder().allowBlockingCallsInside(...)`.

Setup, false positives, and production strategies: `references/debugging.md`.

## Always Do / Never Do

### Always
- Run the full Section C checklist on a review, not just the reported concern
- Describe an operator's marble behavior before its usage
- Suggest virtual threads when the case has no backpressure, streaming, or complex async need
- In Kotlin, evaluate coroutines/Flow before reaching for Reactor operators
- Show both paradigm versions side by side when translating
- Explain the "why" — why `concatMap` over `flatMap`, what ordering buys

### Never
- Recommend Reactor for simple CRUD without comparing to virtual threads
- Suggest `.block()` inside a reactive chain — only at composition roots
- Ignore the threading model
- Allow silent error swallowing — `doOnError` is logging, not recovery
- Use `subscribeOn` as a generic "fix" for blocking I/O without explaining that `boundedElastic` is sized for blocking work, not a magic escape hatch
