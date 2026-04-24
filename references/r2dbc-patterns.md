# R2DBC Reactive Database Patterns

## DatabaseClient — Direct SQL

```java
@Autowired DatabaseClient databaseClient;

// Simple query
Mono<User> findById(String id) {
    return databaseClient.sql("SELECT id, name, email FROM users WHERE id = :id")
        .bind("id", id)
        .map((row, meta) -> new User(
            row.get("id", String.class),
            row.get("name", String.class),
            row.get("email", String.class)
        ))
        .one();  // Mono — error if multiple rows
}

// Multiple results
Flux<User> findByStatus(String status) {
    return databaseClient.sql("SELECT * FROM users WHERE status = :status")
        .bind("status", status)
        .mapProperties(User.class)  // Spring 6.1+: map to record/class by convention
        .all();
}

// Insert and get generated key
Mono<Long> insert(User user) {
    return databaseClient.sql("INSERT INTO users (name, email) VALUES (:name, :email)")
        .bind("name", user.getName())
        .bind("email", user.getEmail())
        .filter(stmt -> stmt.returnGeneratedValues("id"))
        .fetch().one()
        .map(row -> (Long) row.get("id"));
}
```

## R2dbcEntityTemplate — Higher-Level API

```java
@Autowired R2dbcEntityTemplate template;

// CRUD
Mono<User> find(String id) {
    return template.selectOne(
        query(where("id").is(id)), User.class
    );
}

Flux<User> findActive() {
    return template.select(
        query(where("status").is("ACTIVE").and("deleted").isFalse())
            .sort(Sort.by("name"))
            .limit(100),
        User.class
    );
}

Mono<User> save(User user) {
    return template.insert(user);  // or update()
}

Mono<Integer> delete(String id) {
    return template.delete(
        query(where("id").is(id)), User.class
    );
}
```

## Reactive Repositories (Spring Data R2DBC)

```java
public interface UserRepository extends ReactiveCrudRepository<User, String> {
    Flux<User> findByStatus(String status);
    Mono<User> findByEmail(String email);

    @Query("SELECT * FROM users WHERE name LIKE :pattern ORDER BY name LIMIT :limit")
    Flux<User> searchByName(String pattern, int limit);
}
```

## Transaction Handling

```java
// Declarative (Spring @Transactional — works with WebFlux)
@Transactional
public Mono<Order> createOrder(OrderRequest req) {
    return userRepository.findById(req.getUserId())
        .switchIfEmpty(Mono.error(new UserNotFoundException()))
        .flatMap(user -> orderRepository.save(new Order(user, req.getItems())))
        .flatMap(order -> inventoryService.reserve(order)); // still in transaction
}

// Programmatic with TransactionalOperator
@Autowired TransactionalOperator tx;

Mono<Order> createOrder(OrderRequest req) {
    return doCreate(req).as(tx::transactional);  // wrap whole mono in transaction
}

// Or inline
Mono<Order> createOrder(OrderRequest req) {
    return tx.execute(status ->
        doCreate(req).doOnError(e -> status.setRollbackOnly())
    ).next();
}
```

## Connection Pool Configuration (io.r2dbc.pool)

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost/mydb
    pool:
      initial-size: 5
      max-size: 20
      max-idle-time: 30m
      max-acquire-time: 5s     # timeout waiting for connection
      validation-query: SELECT 1
```

## Avoiding N+1 in Reactive Repos

```java
// BAD: N+1 — fetches users one-by-one
Flux<UserWithOrders> badNPlusOne() {
    return userRepo.findAll()
        .flatMap(user -> orderRepo.findByUserId(user.getId())
            .collectList()
            .map(orders -> new UserWithOrders(user, orders)));
}

// BETTER: batch fetch with IN query
Flux<UserWithOrders> batched() {
    return userRepo.findAll()
        .collectList()
        .flatMapMany(users -> {
            Set<String> ids = users.stream().map(User::getId).collect(toSet());
            return orderRepo.findByUserIdIn(ids)
                .collectMultimap(Order::getUserId)
                .map(orderMap -> users.stream()
                    .map(u -> new UserWithOrders(u, orderMap.getOrDefault(u.getId(), List.of())))
                    .collect(toList()))
                .flatMapMany(Flux::fromIterable);
        });
}
```

## Key Rules

- Use `DatabaseClient` for complex SQL or when you need fine-grained control; use `R2dbcEntityTemplate` for straightforward CRUD.
- `@Transactional` on a method returning `Mono`/`Flux` works in WebFlux because Spring propagates the transaction via Reactor Context — do not block inside the transaction.
- Avoid `collectList()` on unbounded streams; prefer streaming with `flatMap` or windowed batching.
- `mapProperties()` (Spring 6.1+) requires the target class to have fields that match column names by convention (camelCase to snake_case mapping is automatic).
