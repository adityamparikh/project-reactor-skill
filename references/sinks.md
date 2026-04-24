# Sinks — Programmatic Signal Emission in Reactor 3.4+

Sinks are the modern API for pushing signals into a reactive pipeline from outside it. They replace
the deprecated `FluxProcessor`, `MonoProcessor`, `EmitterProcessor`, `ReplayProcessor`, and
`DirectProcessor`. Use them when emitting from a callback, event listener, or imperative code.

## 1. Sinks.One — Single Value

```java
Sinks.One<User> sink = Sinks.one();
sink.tryEmitValue(user);   // emit a value
sink.tryEmitEmpty();       // complete with no value
sink.tryEmitError(new RuntimeException("failed"));
Mono<User> mono = sink.asMono();
```

Use cases: deferred callback bridge, request-reply, `CompletableFuture` replacement.
```kotlin
val sink = Sinks.one<User>()
val mono: Mono<User> = sink.asMono()
userService.fetchAsync(id) { user -> sink.tryEmitValue(user) }
```

## 2. Sinks.Many — Multicast, Unicast, and Replay

**Multicast** (multiple subscribers):
```java
Sinks.Many<Event> buffered    = Sinks.many().multicast().onBackpressureBuffer();
Sinks.many().multicast().onBackpressureBuffer(256, false); // no history for late subs
Sinks.Many<Event> allOrNothing = Sinks.many().multicast().directAllOrNothing();
Sinks.Many<Event> bestEffort   = Sinks.many().multicast().directBestEffort();
```

**Unicast** (single subscriber, buffers until one connects):
```java
Sinks.Many<Event> unicast = Sinks.many().unicast().onBackpressureBuffer();
Sinks.many().unicast().onBackpressureBuffer(Queues.<Event>unboundedMultiproducer().get());
```

**Replay** (late subscribers receive history):
```java
Sinks.Many<Event> replayLast = Sinks.many().replay().limit(10);
Sinks.Many<Event> latest     = Sinks.many().replay().latest();     // BehaviorSubject equiv
Sinks.Many<Event> all        = Sinks.many().replay().all();        // unbounded — use carefully
Sinks.many().replay().limit(10, Duration.ofMinutes(5));            // bounded by count + time
Flux<Event> flux = replayLast.asFlux();
```

## 3. Emitting — tryEmitNext vs emitNext

```java
// tryEmitNext: caller handles the result
EmitResult result = sink.tryEmitNext(event);
if (result.isFailure()) {
    if (result == EmitResult.FAIL_NON_SERIALIZED) { /* concurrent emission — see §4 */ }
    log.warn("Emission failed: {}", result);
}

// emitNext: supply a failure handler
sink.emitNext(event, EmitFailureHandler.FAIL_FAST); // throws on any failure
```

Custom handler (retry only when safe — see §4):
```java
sink.emitNext(event, (signalType, emitResult) ->
    emitResult == EmitResult.FAIL_NON_SERIALIZED); // true = retry
```

## 4. Thread Safety

Safe sinks (the default) return `FAIL_NON_SERIALIZED` on concurrent `tryEmitNext` instead of
corrupting state. This is not automatic retry — the caller must act on it.

```java
// Option 1: serialize externally
synchronized (sink) { sink.tryEmitNext(event); }

// Option 2: Sinks.unsafe() — no serialization check, single-thread emission only
Sinks.Many<Event> unsafe = Sinks.unsafe().many().multicast().directBestEffort();
```

Use `Sinks.safe()` (default) for multi-threaded producers. Use `Sinks.unsafe()` only when
emission is guaranteed single-threaded (e.g., one event-loop thread). Retrying
`FAIL_NON_SERIALIZED` in a loop from multiple threads causes a busy spin.

## 5. Common Use Cases

**Callback / listener bridge:**
```java
Sinks.Many<Message> sink = Sinks.many().multicast().onBackpressureBuffer(1024);
jmsListener.setMessageHandler(msg -> sink.tryEmitNext(msg).orThrow());
Flux<Message> messages = sink.asFlux();
```

**Hot SSE stream:**
```java
private final Sinks.Many<ServerEvent> eventSink =
    Sinks.many().multicast().onBackpressureBuffer(256);
public Flux<ServerEvent> subscribe() { return eventSink.asFlux(); }
public void publish(ServerEvent e)   { eventSink.tryEmitNext(e); }
```

**Request-reply:**
```java
Map<String, Sinks.One<Response>> pending = new ConcurrentHashMap<>();

Mono<Response> sendRequest(String id, Request req) {
    Sinks.One<Response> reply = Sinks.one();
    pending.put(id, reply);
    transport.send(req);
    return reply.asMono().timeout(Duration.ofSeconds(10))
                         .doFinally(s -> pending.remove(id));
}
void onResponse(String id, Response resp) {
    Sinks.One<Response> r = pending.get(id);
    if (r != null) r.tryEmitValue(resp);
}
```

**Kotlin coroutine bridge:**
```kotlin
val sink = Sinks.many().multicast().onBackpressureBuffer<Event>(256)
scope.launch {
    events.collect { event -> sink.emitNext(event, EmitFailureHandler.FAIL_FAST) }
}
val flux: Flux<Event> = sink.asFlux()
```

## 6. Anti-Patterns

**Prefer `Flux.create()` for simple in-process bridging** — manages lifecycle, no result checking:
```java
Flux<Event> flux = Flux.create(sink -> {
    listener.onEvent(sink::next);
    listener.onDone(sink::complete);
    listener.onError(sink::error);
});
```

**Always check `EmitResult`** — `tryEmitNext` silently drops items if the result is ignored:
```java
// BAD
sink.tryEmitNext(event);
// GOOD
EmitResult r = sink.tryEmitNext(event);
if (r.isFailure()) handleFailure(r);
```

**Do not retry `FAIL_NON_SERIALIZED` on a safe sink from multiple threads** — it busy-spins.
Use external synchronization with the safe sink, or `Sinks.unsafe()` with single-thread emission.
