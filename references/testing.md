# Testing Reactive Code with Project Reactor

## StepVerifier — Core Tool

```java
// Basic: verify sequence
StepVerifier.create(userService.findById("123"))
    .expectNextMatches(user -> "123".equals(user.getId()))
    .verifyComplete();

// Multiple items
StepVerifier.create(userService.findAll())
    .expectNext(user1)
    .expectNextMatches(u -> u.isActive())
    .expectNextCount(48)  // 48 more items
    .verifyComplete();

// Error verification
StepVerifier.create(userService.findById("missing"))
    .expectError(NotFoundException.class)
    .verify();

StepVerifier.create(userService.findById("missing"))
    .expectErrorMessage("User not found: missing")
    .verify();

StepVerifier.create(userService.findById("missing"))
    .expectErrorMatches(e -> e instanceof NotFoundException
                          && e.getMessage().contains("missing"))
    .verify();

// Empty
StepVerifier.create(userService.findById("notexist"))
    .verifyComplete();  // expects nothing then complete

// Timeout (fail test if verification takes > 5s)
StepVerifier.create(service.slowOperation())
    .expectNext(result)
    .verifyComplete(Duration.ofSeconds(5));
```

## Virtual Time — Testing Delays Without Waiting

```java
// Use StepVerifier.withVirtualTime() for interval(), delay(), timeout()
StepVerifier.withVirtualTime(() ->
    Flux.interval(Duration.ofSeconds(1)).take(3)
)
.expectSubscription()
.expectNoEvent(Duration.ofSeconds(1))    // advance virtual clock 1s
.expectNext(0L)
.thenAwait(Duration.ofSeconds(1))        // advance 1s
.expectNext(1L)
.thenAwait(Duration.ofSeconds(1))
.expectNext(2L)
.verifyComplete();

// Testing retry with backoff
StepVerifier.withVirtualTime(() ->
    service.withRetry()
        .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)))
)
.expectSubscription()
.thenAwait(Duration.ofSeconds(10))  // skip all backoff waits
.expectNextCount(1)
.verifyComplete();
```

## TestPublisher — Control a Publisher in Tests

```java
// Create a controllable publisher
TestPublisher<String> publisher = TestPublisher.create();

StepVerifier.create(publisher.flux()
    .map(String::toUpperCase))
    .then(() -> publisher.next("hello"))   // push "hello"
    .expectNext("HELLO")
    .then(() -> publisher.next("world"))
    .expectNext("WORLD")
    .then(publisher::complete)
    .verifyComplete();

// Simulate errors
TestPublisher<String> publisher = TestPublisher.create();
publisher.error(new RuntimeException("oops"));

// Non-compliant publisher (for testing error handling)
TestPublisher<String> violating = TestPublisher.createNoncompliant(
    TestPublisher.Violation.REQUEST_OVERFLOW
);
```

## PublisherProbe — Verify Publisher Was Subscribed

```java
// Test that a fallback Mono was actually subscribed
PublisherProbe<User> fallback = PublisherProbe.of(Mono.just(defaultUser));

Mono<User> result = userService.findById("missing")
    .switchIfEmpty(fallback.mono());

StepVerifier.create(result)
    .expectNext(defaultUser)
    .verifyComplete();

fallback.assertWasSubscribed();
fallback.assertWasRequested();
fallback.assertWasNotCancelled();
```

## Testing Context Propagation

```java
StepVerifier.create(
    Mono.deferContextual(ctx -> Mono.just(ctx.get("userId")))
        .contextWrite(Context.of("userId", "user-123"))
)
.expectNext("user-123")
.verifyComplete();

// Service that reads context
StepVerifier.create(
    auditService.logAction("DELETE")
        .contextWrite(ReactiveSecurityContextHolder
            .withAuthentication(mockAuth("admin")))
)
.verifyComplete();
```

## WebTestClient — Integration Testing WebFlux

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerTest {
    @Autowired WebTestClient client;

    @Test
    void getUser() {
        client.get().uri("/users/123")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .value(user -> assertThat(user.getId()).isEqualTo("123"));
    }

    @Test
    void notFound() {
        client.get().uri("/users/missing")
            .exchange()
            .expectStatus().isNotFound();
    }

    @Test
    void streamEvents() {
        client.get().uri("/events/stream")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .returnResult(Event.class)
            .getResponseBody()
            .take(3)
            .as(StepVerifier::create)
            .expectNextCount(3)
            .thenCancel()
            .verify();
    }
}
```

## Testing Error Scenarios

```java
// Verify doOnError side effects
AtomicBoolean errorLogged = new AtomicBoolean(false);
Mono<User> mono = userService.findById("bad")
    .doOnError(e -> errorLogged.set(true));

StepVerifier.create(mono)
    .expectError(ServiceException.class)
    .verify();

assertThat(errorLogged.get()).isTrue();
```
