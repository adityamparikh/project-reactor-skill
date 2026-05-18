# Debugging Project Reactor

## The Problem: Lost Stack Traces

Operators are assembled in one place and subscribed in another. By default, stack traces show Reactor internals, not your code — the assembly trace is lost by the time the error bubbles up.

```
java.lang.NullPointerException: null
    at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(...)
    at reactor.core.publisher.FluxFilterFuseable$FilterFuseableSubscriber.onNext(...)
    // ...no trace to your code
```

## checkpoint() — Lightweight Targeted Debugging

```java
userService.findById(id)
    .checkpoint("finding user by id")          // lightweight: description only
    .map(this::transform)
    .checkpoint("after transform", true)        // true = force full stack trace
    .flatMap(this::saveToDb)
    .checkpoint("after db save");

// Output:
// Assembly trace from checkpoint [finding user by id]:
//     at com.example.UserService.getUser(UserService.java:42)
```

Use at service boundaries; low overhead, OK in production.

## log() — Signal Tracing

```java
flux.log("com.example.myFlux");         // SLF4J at DEBUG
flux.log("myFlux", Level.INFO,
         SignalType.ON_NEXT, SignalType.ON_ERROR);   // filter signals
```

Logs: `onSubscribe`, `request(n)`, `onNext(value)`, `onComplete`, `onError`, `cancel`.

**Warning:** `log()` is very verbose in production. Filter signals or guard behind `if (log.isDebugEnabled())`.

## Hooks.onOperatorDebug() — Full Assembly Capture

```java
Hooks.onOperatorDebug();        // captures full assembly trace for ALL operators
Hooks.resetOnOperatorDebug();   // disable
```

**Cost:** 3–10× overhead due to stack capture at every operator. Dev/test only.

## ReactorDebugAgent — Production-Safe Full Tracing

Bytecode instrumentation — no per-operator overhead:

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-tools</artifactId>
</dependency>
```
```java
ReactorDebugAgent.init();                     // at app start
ReactorDebugAgent.processExistingClasses();   // for already-loaded classes
```

With Spring Boot, `reactor-tools` auto-initializes.

## Reading Assembly Stack Traces

```
Error has been observed at the following site(s):
    *__checkpoint ⇢ HTTP GET "/api/users/{id}" [ExceptionHandlingWebHandler]
    *__checkpoint ⇢ com.example.UserController#getUser(String) [DispatcherHandler]

Original Stack Trace:
    at com.example.UserService.findById(UserService.java:35)
```

Read **bottom-up**: "Original Stack Trace" is the origin. `__checkpoint` lines above show contexts the error bubbled through.

## BlockHound — Detect Blocking Calls

```java
// Dep: io.projectreactor.tools:blockhound

BlockHound.install();   // globally, in @BeforeAll or main

// Spring Boot test
@ExtendWith(BlockHoundJUnit5Extension.class)
class MyReactiveTest { }
```

When blocking happens on a reactive thread:
```
BlockingOperationError: Blocking call! java.io.FileInputStream#readBytes
    at com.example.BadService.doWork(BadService.java:25)
```

**Allowlist false positives:**
```java
BlockHound.install(builder -> builder
    .allowBlockingCallsInside("sun.security.provider.SHA", "implCompress")
    .allowBlockingCallsInside("ch.qos.logback.core.AsyncAppenderBase", "put"));
```

Common false positives:
- JDK crypto/TLS init (first call only)
- Logback/Log4j async appenders
- Class loading / static initializers
- Hibernate validator (first use)
- Some Reactor internals (pre-allowlisted in `BlockHound.install()`)

## Quick Debug Checklist

1. Add `log("tag")` to see where the stream terminates unexpectedly
2. Add `checkpoint("where")` at boundaries for location info
3. Empty subscription? Check for `switchIfEmpty` swallowing or wrong operator order
4. Deadlock? Run BlockHound — likely blocking call on wrong scheduler
5. Error disappearing? Look for `onErrorResume → Mono.empty()`
6. Still lost? Enable `Hooks.onOperatorDebug()` temporarily
7. Or use ReactorDebugAgent with IDE debugger
