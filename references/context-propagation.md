# Context Propagation

## Reactor Context Basics

Reactor `Context` is an **immutable key-value store** that travels with the pipeline, flowing from subscriber **upstream**. Solves MDC/tracing in async code where `ThreadLocal` doesn't work — threads change between operators but Context follows the logical execution path.

```java
// Write (subscriber side, bottom of chain)
Mono.just("hello")
    .contextWrite(ctx -> ctx.put("userId", "user-123").put("traceId", "abc-456"));

// Read (upstream, in operators)
Mono.deferContextual(ctx -> {
    String userId = ctx.get("userId");
    return Mono.just("Processing for: " + userId);
});

// Inside flatMap
.flatMap(item -> Mono.deferContextual(ctx ->
    Mono.just(process(item, ctx.getOrDefault("userId", "anonymous")))));
```

Rules:
- `contextWrite` goes at the **bottom** of the chain; affects operators **above** it (context flows upstream)
- Immutable — each `put` creates a new Context
- Read with `deferContextual`, `transformDeferredContextual`, or `ContextView` in `Sink` callbacks
- Keys are typically typed `Class<T>` or `String`; prefer typed keys to avoid collisions

## MDC Bridging

MDC uses `ThreadLocal` — values may be lost across operators. Bridge via `doOnEach`, which exposes `ContextView` per signal.

```java
Flux<String> traced = source
    .doOnEach(signal -> {
        if (signal.hasContext()) {
            ContextView ctx = signal.getContextView();
            MDC.put("traceId", ctx.getOrDefault("traceId", ""));
            MDC.put("userId", ctx.getOrDefault("userId", ""));
        }
    })
    .doFinally(s -> MDC.clear());
```

`doOnEach` fires for onNext, onError, onComplete — all carry `ContextView`.

For Spring apps, prefer `reactor-context-propagation` + Micrometer for automatic MDC/tracing without manual bridging.

## Micrometer Tracing + Context Propagation

Reactor 3.5.3+ supports automatic propagation via `io.micrometer:context-propagation`:

```java
Hooks.enableAutomaticContextPropagation();   // once at startup

Mono.just(request)
    .flatMap(req -> userService.findById(req.getUserId()))
    .contextWrite(Context.of(ObservationRegistry.NOOP));
```

With **Spring Boot 3.x + WebFlux**, auto-config calls `Hooks.enableAutomaticContextPropagation()` and wires `ObservationRegistry`. Observations in filters and controllers propagate automatically.

## Spring Security ReactiveSecurityContextHolder

Spring Security stores auth in Reactor Context under its own key.

```java
// Read anywhere in the chain
ReactiveSecurityContextHolder.getContext()
    .map(SecurityContext::getAuthentication)
    .map(auth -> (User) auth.getPrincipal())
    .flatMap(user -> userService.findById(user.getUsername()));

// Write (tests, gateway filters)
chain.filter(exchange)
    .contextWrite(ReactiveSecurityContextHolder.withAuthentication(auth));
```

`ReactiveAuthenticationManager` writes the context; downstream operators read it without extra wiring.

## Context vs ContextView

- `Context` — mutable builder (used in `contextWrite` lambdas); `put`, `delete`
- `ContextView` — read-only projection; in `deferContextual` and signals
- `Context.of(key, value)` — new immutable Context from scratch

```java
Context merged = baseContext.putAll(additionalContext.readOnly());
```

## Testing

```java
StepVerifier.create(
    Mono.deferContextual(ctx -> Mono.just(ctx.get("key")))
        .contextWrite(Context.of("key", "value")))
    .expectNext("value").verifyComplete();
```

Note: `contextWrite` in `StepVerifier.create(...)` applies to the assembled publisher — place it inside the chain, not on the StepVerifier itself.
