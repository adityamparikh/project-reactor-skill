# Context Propagation

## Reactor Context Basics

Reactor `Context` is an **immutable key-value store** that travels with the reactive pipeline, flowing from subscriber **upstream**. It solves MDC and tracing in async code where `ThreadLocal` doesn't work — threads change between operators, but Context follows the logical execution path.

```java
// Writing to context (subscriber side, bottom of chain)
Mono.just("hello")
    .contextWrite(ctx -> ctx.put("userId", "user-123")
                            .put("traceId", "abc-456"))

// Reading from context (upstream, in operators)
Mono.deferContextual(ctx -> {
    String userId = ctx.get("userId");
    return Mono.just("Processing for: " + userId);
})

// Or inside a flatMap
.flatMap(item -> Mono.deferContextual(ctx ->
    Mono.just(process(item, ctx.getOrDefault("userId", "anonymous")))
))
```

Key rules:
- `contextWrite` is placed at the **bottom** of the chain; it affects operators **above** it (context flows upstream)
- Context is immutable — each `put` creates a new `Context` instance
- Read with `deferContextual`, `transformDeferredContextual`, or via `ContextView` in a `Sink` callback
- Keys are typically typed `Class<T>` or `String`; prefer typed keys to avoid collisions

## MDC Bridging

Because MDC uses `ThreadLocal`, values written in one operator may be lost by the next. Bridge via `doOnEach`, which provides access to the `ContextView` for every signal.

```java
// Bridge Reactor Context -> MDC for each onNext
Flux<String> traced = source
    .doOnEach(signal -> {
        if (signal.hasContext()) {
            ContextView ctx = signal.getContextView();
            MDC.put("traceId", ctx.getOrDefault("traceId", ""));
            MDC.put("userId", ctx.getOrDefault("userId", ""));
        }
    })
    .doFinally(signalType -> MDC.clear());
```

`doOnEach` fires for `onNext`, `onError`, and `onComplete` signals — all carry the `ContextView`.

For Spring applications, prefer the `reactor-context-propagation` library with Micrometer for automatic MDC/tracing propagation without manual bridging.

## Micrometer Tracing + Context Propagation

Reactor 3.5.3+ supports automatic context propagation via `io.micrometer:context-propagation`. Enable once at startup:

```java
// Enable automatic context propagation (call once at application startup)
Hooks.enableAutomaticContextPropagation();

// Creates a trace-aware reactive chain
Mono.just(request)
    .flatMap(req -> userService.findById(req.getUserId()))
    // Micrometer tracing context flows automatically via Reactor Context
    .contextWrite(Context.of(ObservationRegistry.NOOP)) // inject real registry in production
```

With **Spring Boot 3.x and WebFlux**, auto-configuration handles `Hooks.enableAutomaticContextPropagation()` and wires the `ObservationRegistry`. Observations created in WebFlux filters and controllers propagate automatically through the chain.

## Spring Security ReactiveSecurityContextHolder

Spring Security stores authentication in Reactor Context under its own key. Use `ReactiveSecurityContextHolder` to read or write it anywhere in the chain.

```java
// Read security context anywhere in the chain
ReactiveSecurityContextHolder.getContext()
    .map(SecurityContext::getAuthentication)
    .map(auth -> (User) auth.getPrincipal())
    .flatMap(user -> userService.findById(user.getUsername()))

// Set security context (e.g., in tests or gateway filters)
return chain.filter(exchange)
    .contextWrite(ReactiveSecurityContextHolder.withAuthentication(auth));
```

The authentication context is written by Spring Security's `ReactiveAuthenticationManager` and is available to all downstream operators without any additional wiring.

## Context vs ContextView

- `Context` — mutable builder interface (used in `contextWrite` lambdas); `put`, `delete`, etc.
- `ContextView` — read-only projection; used in `deferContextual` and `doOnEach` signals
- `Context.of(key, value)` — creates a new immutable Context from scratch

```java
// Merging contexts
Context merged = baseContext.putAll(additionalContext.readOnly());
```

## Testing Context Propagation

```java
StepVerifier.create(
    Mono.deferContextual(ctx -> Mono.just(ctx.get("key")))
        .contextWrite(Context.of("key", "value"))
)
.expectNext("value")
.verifyComplete();

// Testing with security context
StepVerifier.create(
    ReactiveSecurityContextHolder.getContext()
        .map(ctx -> ctx.getAuthentication().getName())
        .contextWrite(ReactiveSecurityContextHolder.withAuthentication(mockAuth))
)
.expectNext("testuser")
.verifyComplete();
```

Note: `contextWrite` in `StepVerifier.create(...)` applies to the assembled publisher — place it at the end of the inner chain, not on the `StepVerifier` itself.
