# Debugging Project Reactor

## The Problem: Lost Stack Traces

In Reactor, operators are assembled in one place and subscribed in another. By default, stack traces show Reactor internals, not your code. The assembly trace is lost by the time the error bubbles up.

```
// Without debug: unhelpful
java.lang.NullPointerException: null
    at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(...)
    at reactor.core.publisher.FluxFilterFuseable$FilterFuseableSubscriber.onNext(...)
    // ...no trace to your code
```

## checkpoint() — Lightweight Targeted Debugging

Add `checkpoint()` to the chain at boundaries you care about:

```java
userService.findById(id)
    .checkpoint("finding user by id")           // lightweight description
    .map(this::transform)
    .checkpoint("after transform", true)         // true = force full stack trace
    .flatMap(this::saveToDb)
    .checkpoint("after db save");

// Output:
// Assembly trace from checkpoint [finding user by id]:
//     at com.example.UserService.getUser(UserService.java:42)
```

Use `checkpoint()` at service boundaries and in production — it's low overhead.

## log() — Signal Tracing

```java
// Log all signals with a category tag
flux.log("com.example.myFlux")         // SLF4J at DEBUG
    .log("myFlux", Level.INFO,          // specific level
         SignalType.ON_NEXT,            // only specific signals
         SignalType.ON_ERROR)
```

Logs: `onSubscribe`, `request(n)`, `onNext(value)`, `onComplete`, `onError`, `cancel`.

**Warning**: `log()` in production can be very verbose. Use `log("tag", Level.INFO, SignalType.ON_NEXT, SignalType.ON_ERROR)` to filter.

## Hooks.onOperatorDebug() — Full Assembly Capture

```java
// Enable BEFORE building your pipelines (at app startup or test setup)
Hooks.onOperatorDebug();  // captures full assembly trace for ALL operators
```

**Cost**: 3-10x overhead due to stack capture at every operator. Use in development or test environments only, not production.

```java
// Disable when done
Hooks.resetOnOperatorDebug();
```

## ReactorDebugAgent — Production-Safe Full Tracing

Uses bytecode instrumentation (like Java agents) — no runtime overhead per operator:

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-tools</artifactId>
</dependency>
```

```java
// Initialize at app start (before any reactor pipelines are built)
ReactorDebugAgent.init();
// Or:
ReactorDebugAgent.processExistingClasses(); // for already-loaded classes
```

With Spring Boot, add `reactor-tools` and it auto-initializes.

## Reading Assembly Stack Traces

```
Error has been observed at the following site(s):
    *__checkpoint ⇢ HTTP GET "/api/users/{id}" [ExceptionHandlingWebHandler]
    *__checkpoint ⇢ com.example.UserController#getUser(String) [DispatcherHandler]

Original Stack Trace:
    at com.example.UserService.findById(UserService.java:35)
```

**Read bottom-up**: the "Original Stack Trace" is where the error originated. The `__checkpoint` lines (above it) show the chain of reactive contexts the error bubbled through.

## BlockHound — Detect Blocking Calls

```java
// Add dependency
// io.projectreactor.tools:blockhound:1.0.9.RELEASE

// Install globally (in @BeforeAll or main)
BlockHound.install();

// With Spring Boot test
@ExtendWith(BlockHoundJUnit5Extension.class)
class MyReactiveTest { }
```

When a blocking call is made on a reactive thread, BlockHound throws:
```
BlockingOperationError: Blocking call! java.io.FileInputStream#readBytes
    at com.example.BadService.doWork(BadService.java:25)
```

**Allowlist false positives**:
```java
BlockHound.install(builder -> builder
    .allowBlockingCallsInside("sun.security.provider.SHA", "implCompress")  // JDK crypto
    .allowBlockingCallsInside("ch.qos.logback.core.AsyncAppenderBase", "put") // logging
);
```

Common false positives:
- JDK crypto/TLS initialization (first call only)
- Logback/Log4j async appenders
- Class loading / static initializers
- Hibernate validator (first use)
- Some Reactor internals (already allowlisted in `BlockHound.install()`)

## Quick Debug Checklist

1. Add `log("tag")` to see where the stream terminates unexpectedly
2. Add `checkpoint("where")` at boundaries to get location info
3. Empty subscription? Check for `switchIfEmpty` swallowing items, or wrong operator order
4. Deadlock? Run BlockHound — likely blocking call on wrong scheduler
5. Error disappearing? Check for `onErrorResume` returning `Mono.empty()` silently
6. Enable `Hooks.onOperatorDebug()` temporarily for full trace
7. Still lost? Use ReactorDebugAgent with IDE debugger
