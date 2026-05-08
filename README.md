# project-reactor-skill

A Claude Code skill that provides expert guidance for [Project Reactor](https://projectreactor.io/) — the reactive programming library for the JVM.

## What This Skill Does

This skill turns Claude into a knowledgeable Project Reactor assistant. When working in a codebase that uses Reactor, Claude will draw on curated reference material to help with operator selection, reactive pipeline design, debugging strategies, testing with StepVerifier, threading and scheduler decisions, backpressure configuration, context propagation, and integration patterns with Spring WebFlux, R2DBC, and WebClient. It also helps you decide when Reactor is the right tool versus simpler alternatives like virtual threads.

## When It Triggers

The skill activates in the following scenarios:

- Writing or reviewing reactive pipelines (`Mono`, `Flux`, operator chains)
- Debugging reactive code using `checkpoint()`, `log()`, or ReactorDebugAgent
- Choosing between operators: `flatMap` vs `concatMap`, `merge` vs `concat`, `zip` vs `combineLatest`
- Threading and scheduler decisions: `publishOn` vs `subscribeOn`, `Schedulers.parallel()` vs `boundedElastic()`
- Backpressure strategy selection and `limitRate`/`prefetch` tuning
- Propagating context through reactive chains (MDC bridging, Micrometer, Spring Security)
- Bridging blocking and reactive code safely (BlockHound, `fromCallable`, `defer`)
- Testing reactive code with `StepVerifier`, `TestPublisher`, and virtual time
- Kotlin interop: `awaitSingle`, coroutine builders, `Flow` ↔ `Flux` conversion
- R2DBC patterns: `DatabaseClient`, `R2dbcEntityTemplate`, transactions, avoiding N+1
- WebClient configuration: connection pooling, retry with exponential backoff, metrics
- Deciding between Project Reactor and virtual threads (JDK 21+)

## Coverage

| File | Description |
|------|-------------|
| `SKILL.md` | Main skill: workflow, decision trees, and quick examples |
| `references/operators.md` | Operator catalog by category with marble diagram descriptions |
| `references/schedulers-and-threading.md` | `publishOn` vs `subscribeOn`, `Schedulers.*`, parallel Flux |
| `references/backpressure.md` | Overflow strategies, `limitRate`, prefetch tuning |
| `references/context-propagation.md` | Reactor Context, MDC bridging, Micrometer, Spring Security |
| `references/bridging.md` | Blocking↔Reactive patterns, BlockHound, Spring interop |
| `references/kotlin-interop.md` | Reactor ↔ Flow/coroutines, `awaitSingle`, `mono{}`/`flux{}` builders |
| `references/r2dbc-patterns.md` | `DatabaseClient`, `R2dbcEntityTemplate`, transactions, N+1 avoidance |
| `references/reactor-netty-http.md` | WebClient, connection pooling, retry with backoff, metrics |
| `references/testing.md` | `StepVerifier`, `TestPublisher`, virtual time, `WebTestClient` |
| `references/debugging.md` | `checkpoint()`, `log()`, Hooks, ReactorDebugAgent, BlockHound |
| `references/anti-patterns.md` | Common mistakes with bad/good code examples |
| `references/virtual-threads-decision.md` | When to prefer virtual threads over Reactor |

## Installation

Clone the skill into your Claude Code skills directory:

```bash
git clone https://github.com/adityamparikh/project-reactor-skill.git ~/.claude/skills/project-reactor
```

Claude Code will pick up the skill on the next session.

## Target Library Versions

- Reactor Core: 3.6.x (Reactor 2024.0 release train)
- Spring Boot: 3.4.x or 4.0.x
- R2DBC: 1.0.x
- BlockHound: 1.0.9.x
- Kotlin: 1.9+

> Versions above are illustrative — verify against current upstream releases when applying examples.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
