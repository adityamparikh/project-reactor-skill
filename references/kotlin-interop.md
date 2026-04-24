# Kotlin Interop with Project Reactor

## Reactor <-> Kotlin Flow / Coroutines

```kotlin
// Flux → Flow (collect in coroutine scope)
val flow: Flow<User> = userFlux.asFlow()

// Flow → Flux
val flux: Flux<User> = userFlow.asFlux()

// Mono → suspend (await single item)
suspend fun getUser(id: String): User =
    userMono.awaitSingle()           // error if empty

suspend fun getUserOrNull(id: String): User? =
    userMono.awaitSingleOrNull()     // null if empty

suspend fun getFirst(id: String): User =
    userFlux.awaitFirst()            // error if empty

suspend fun getFirstOrNull(): User? =
    userFlux.awaitFirstOrNull()

suspend fun getLast(): User =
    userFlux.awaitLast()

// Collect all to list
suspend fun getAll(): List<User> =
    userFlux.asFlow().toList()
// or:
    userFlux.collectList().awaitSingle()
```

## Coroutine Builders for Reactor

```kotlin
// mono{} — suspend function → Mono
fun findUser(id: String): Mono<User> = mono {
    val raw = httpClient.get(id).awaitBody<RawUser>()  // suspend call inside
    User(raw.id, raw.name)
}

// flux{} — channel-based → Flux
fun streamUsers(): Flux<User> = flux {
    users.forEach { send(it) }  // send items to flux
}

// Or use channelFlow for complex patterns
fun streamWithBackpressure(): Flux<User> = flux {
    for (page in pages) {
        val users = fetchPage(page).awaitSingle()
        users.forEach { send(it) }
    }
}
```

## Structured Concurrency in Reactor Context

```kotlin
// Parallel fetches with coroutine scope
fun fetchBoth(id: String): Mono<Pair<User, Profile>> = mono {
    val user = async { userService.findById(id).awaitSingle() }
    val profile = async { profileService.findById(id).awaitSingle() }
    Pair(user.await(), profile.await())  // structured concurrency
}
// Note: Reactor's zip() is an alternative without coroutines
```

## Context Propagation in Kotlin

```kotlin
// Reactor Context in Kotlin suspend context
suspend fun getUserWithContext(): User {
    val userId = currentCoroutineContext()[ReactorContext.Key]
        ?.context?.get<String>("userId") ?: "anonymous"
    return userService.findById(userId).awaitSingle()
}
```

## When to Choose Flow vs Flux

| Situation | Prefer |
|-----------|--------|
| New Kotlin-only service, no Java interop | Flow + coroutines |
| Integrating with Java reactive libs | Flux (or bridge) |
| Advanced backpressure control | Flux (onBackpressureBuffer etc.) |
| Simple sequential async | suspend functions |
| Spring WebFlux handler | Both work; Mono/Flux natural for WebFlux |
| Complex operator composition | Flux (richer operator set) |
| Streaming with structured cancellation | Flow |

## Extension Function Patterns

```kotlin
// Kotlin extension for common patterns
fun <T> Mono<T>.toResult(): Mono<Result<T>> =
    map { Result.success(it) }
    .onErrorResume { Mono.just(Result.failure(it)) }

// Reified generic for cleaner code
inline fun <reified T> WebClient.ResponseSpec.bodyToMono(): Mono<T> =
    bodyToMono(T::class.java)
```

## Dependency Setup

Add the Kotlin coroutines reactor bridge to your build:

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.7.x")
    // Provides: mono{}, flux{}, asFlow(), asFlux(), awaitSingle(), etc.
}
```

## Key Rules

- `awaitSingle()` throws `NoSuchElementException` on empty — use `awaitSingleOrNull()` when absence is normal.
- `mono {}` and `flux {}` inherit the coroutine dispatcher; pair with `Schedulers.boundedElastic()` for blocking calls inside.
- Cancellation propagates both ways: cancelling the coroutine cancels the upstream Flux/Mono, and disposing the Flux cancels the coroutine job.
- Prefer `zip()` over `async/await` pairs when you do not need structured cancellation or conditional branching.
