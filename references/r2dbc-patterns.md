# R2DBC Reactive Database Patterns

## DatabaseClient — Direct SQL

```java
@Autowired DatabaseClient databaseClient;

Mono<User> findById(String id) {
    return databaseClient.sql("SELECT id, name, email FROM users WHERE id = :id")
        .bind("id", id)
        .map((row, meta) -> new User(
            row.get("id", String.class),
            row.get("name", String.class),
            row.get("email", String.class)))
        .one();   // Mono — errors if multiple rows
}

Flux<User> findByStatus(String status) {
    return databaseClient.sql("SELECT * FROM users WHERE status = :status")
        .bind("status", status)
        .mapProperties(User.class)   // Spring 6.1+: convention-based mapping
        .all();
}

// Insert + generated key
Mono<Long> insert(User user) {
    return databaseClient.sql("INSERT INTO users (name, email) VALUES (:name, :email)")
        .bind("name", user.getName()).bind("email", user.getEmail())
        .filter(stmt -> stmt.returnGeneratedValues("id"))
        .fetch().one()
        .map(row -> (Long) row.get("id"));
}
```

## R2dbcEntityTemplate — Higher-Level

```java
@Autowired R2dbcEntityTemplate template;

Mono<User> find(String id) {
    return template.selectOne(query(where("id").is(id)), User.class);
}

Flux<User> findActive() {
    return template.select(
        query(where("status").is("ACTIVE").and("deleted").isFalse())
            .sort(Sort.by("name")).limit(100),
        User.class);
}

Mono<User> save(User user)         { return template.insert(user); }
Mono<Integer> delete(String id)    { return template.delete(query(where("id").is(id)), User.class); }
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

## Transactions

```java
// Declarative — works with WebFlux via Reactor Context
@Transactional
public Mono<Order> createOrder(OrderRequest req) {
    return userRepository.findById(req.getUserId())
        .switchIfEmpty(Mono.error(new UserNotFoundException()))
        .flatMap(user -> orderRepository.save(new Order(user, req.getItems())))
        .flatMap(order -> inventoryService.reserve(order));
}

// Programmatic
@Autowired TransactionalOperator tx;

Mono<Order> createOrder(OrderRequest req) {
    return doCreate(req).as(tx::transactional);
}

// Inline with rollback signal
Mono<Order> createOrder(OrderRequest req) {
    return tx.execute(status ->
        doCreate(req).doOnError(e -> status.setRollbackOnly())
    ).next();
}
```

## Connection Pool (io.r2dbc.pool)

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost/mydb
    pool:
      initial-size: 5
      max-size: 20
      max-idle-time: 30m
      max-acquire-time: 5s
      validation-query: SELECT 1
```

## Avoiding N+1

```java
// BAD — N+1
Flux<UserWithOrders> bad() {
    return userRepo.findAll()
        .flatMap(user -> orderRepo.findByUserId(user.getId())
            .collectList()
            .map(orders -> new UserWithOrders(user, orders)));
}

// BETTER — batch with IN
Flux<UserWithOrders> batched() {
    return userRepo.findAll().collectList()
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

- `DatabaseClient` for complex SQL; `R2dbcEntityTemplate` for straightforward CRUD.
- `@Transactional` on `Mono`/`Flux` methods works in WebFlux via Reactor Context — never block inside the transaction.
- Avoid `collectList()` on unbounded streams; prefer streaming `flatMap` or windowed batching.
- `mapProperties()` (Spring 6.1+) needs target class fields matching column names (camelCase → snake_case auto).
