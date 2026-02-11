---
name: java-patterns
description: Modern Java patterns, best practices, and conventions for building robust, efficient, and maintainable Java applications (Java 17+).
---

# Java Development Patterns

Modern, idiomatic Java patterns and best practices for building robust, efficient, and maintainable applications.

## When to Activate

- Writing new Java code
- Reviewing Java code
- Refactoring existing Java code
- Designing Java packages/modules

## Core Principles

### 1. Favor Immutability

Design immutable objects by default. Mutability should be the exception, not the rule.

```java
// Good: Immutable record (Java 16+)
public record User(String name, String email, List<String> roles) {
    public User {
        // Compact constructor for validation + defensive copy
        Objects.requireNonNull(name, "name must not be null");
        Objects.requireNonNull(email, "email must not be null");
        roles = List.copyOf(roles); // Immutable copy
    }
}

// Good: Immutable class (pre-Java 16)
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = Objects.requireNonNull(amount);
        this.currency = Objects.requireNonNull(currency);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    // getters only, no setters
    public BigDecimal amount() { return amount; }
    public Currency currency() { return currency; }
}

// Good: Unmodifiable collections
var items = List.of("a", "b", "c");
var config = Map.of("key1", "val1", "key2", "val2");
var uniqueNames = Set.of("Alice", "Bob");
```

### 2. Use the Type System to Prevent Bugs

Leverage sealed types, enums, and Optional to make illegal states unrepresentable.

```java
// Good: Sealed interface (Java 17+)
public sealed interface PaymentResult permits PaymentSuccess, PaymentFailure, PaymentPending {
}

public record PaymentSuccess(String transactionId, Instant timestamp) implements PaymentResult {}
public record PaymentFailure(String reason, ErrorCode code) implements PaymentResult {}
public record PaymentPending(String referenceId) implements PaymentResult {}

// Good: Pattern matching with sealed types (Java 21+)
String describe(PaymentResult result) {
    return switch (result) {
        case PaymentSuccess s -> "Paid: " + s.transactionId();
        case PaymentFailure f -> "Failed: " + f.reason();
        case PaymentPending p -> "Pending: " + p.referenceId();
    };
}

// Good: Optional for nullable returns
public Optional<User> findById(String id) {
    return Optional.ofNullable(userMap.get(id));
}
```

### 3. Program to Interfaces

Depend on abstractions, not implementations.

```java
// Good: Depend on interface
public class OrderService {
    private final PaymentGateway gateway;     // Interface
    private final OrderRepository repository; // Interface

    public OrderService(PaymentGateway gateway, OrderRepository repository) {
        this.gateway = Objects.requireNonNull(gateway);
        this.repository = Objects.requireNonNull(repository);
    }
}

// Bad: Depend on implementation
public class OrderService {
    private final StripePaymentGateway gateway;  // Concrete class
    private final JdbcOrderRepository repository; // Concrete class
}
```

### 4. Prefer Composition Over Inheritance

```java
// Bad: Deep inheritance
class Animal { }
class Mammal extends Animal { }
class Dog extends Mammal { }
class GuideDog extends Dog { }

// Good: Composition with interfaces
interface Walkable { void walk(); }
interface Trainable { void train(String command); }

record Dog(String name, WalkBehavior walkBehavior) implements Walkable, Trainable {
    public void walk() { walkBehavior.walk(); }
    public void train(String command) { /* ... */ }
}
```

## Error Handling

### Custom Exception Hierarchy

```java
// Base application exception
public abstract sealed class AppException extends RuntimeException
        permits NotFoundException, ValidationException, ConflictException {

    private final ErrorCode code;

    protected AppException(String message, ErrorCode code) {
        super(message);
        this.code = code;
    }

    protected AppException(String message, ErrorCode code, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public ErrorCode getCode() { return code; }
}

public final class NotFoundException extends AppException {
    public NotFoundException(String entity, String id) {
        super("%s not found: %s".formatted(entity, id), ErrorCode.NOT_FOUND);
    }
}

public final class ValidationException extends AppException {
    private final List<FieldError> errors;

    public ValidationException(List<FieldError> errors) {
        super("Validation failed", ErrorCode.VALIDATION_ERROR);
        this.errors = List.copyOf(errors);
    }

    public List<FieldError> getErrors() { return errors; }
}
```

### Try-With-Resources

```java
// Always use try-with-resources for AutoCloseable
public List<User> queryUsers(String sql) throws SQLException {
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement(sql);
         var rs = stmt.executeQuery()) {

        var users = new ArrayList<User>();
        while (rs.next()) {
            users.add(mapRow(rs));
        }
        return List.copyOf(users);
    }
}
```

### Result Pattern (Alternative to Exceptions)

```java
public sealed interface Result<T> permits Result.Success, Result.Failure {
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String error) implements Result<T> {}

    static <T> Result<T> success(T value) { return new Success<>(value); }
    static <T> Result<T> failure(String error) { return new Failure<>(error); }

    default <U> Result<U> map(Function<T, U> fn) {
        return switch (this) {
            case Success<T> s -> success(fn.apply(s.value()));
            case Failure<T> f -> failure(f.error());
        };
    }
}
```

## Stream and Collection Patterns

### Stream Operations

```java
// Filter, transform, and collect
List<String> activeNames = users.stream()
    .filter(User::isActive)
    .map(User::name)
    .sorted()
    .toList(); // Java 16+

// Grouping
Map<Department, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department));

// Partitioning
Map<Boolean, List<User>> partitioned = users.stream()
    .collect(Collectors.partitioningBy(u -> u.age() >= 18));

// Reducing
BigDecimal total = orders.stream()
    .map(Order::amount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// Flat mapping nested collections
List<Order> allOrders = customers.stream()
    .flatMap(c -> c.orders().stream())
    .toList();

// Collecting to map
Map<String, User> userById = users.stream()
    .collect(Collectors.toMap(User::id, Function.identity()));
```

### Collectors

```java
// Join strings
String csv = names.stream().collect(Collectors.joining(", "));

// Statistics
IntSummaryStatistics stats = orders.stream()
    .collect(Collectors.summarizingInt(Order::quantity));

// Custom collector: toUnmodifiableMap
Map<String, Integer> scores = entries.stream()
    .collect(Collectors.toUnmodifiableMap(Entry::key, Entry::value));
```

## Concurrency Patterns

### Virtual Threads (Java 21+)

```java
// Simple virtual thread
Thread.startVirtualThread(() -> processRequest(request));

// Virtual thread executor
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var futures = urls.stream()
        .map(url -> executor.submit(() -> fetch(url)))
        .toList();

    var results = futures.stream()
        .map(f -> {
            try { return f.get(); }
            catch (Exception e) { throw new RuntimeException(e); }
        })
        .toList();
}

// Structured Concurrency (Java 21+ preview)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var userTask = scope.fork(() -> fetchUser(userId));
    var ordersTask = scope.fork(() -> fetchOrders(userId));

    scope.join().throwIfFailed();

    return new UserProfile(userTask.get(), ordersTask.get());
}
```

### Thread-Safe Collections

```java
// Concurrent map
private final ConcurrentHashMap<String, Session> sessions = new ConcurrentHashMap<>();

// Atomic operations
sessions.computeIfAbsent(userId, id -> createSession(id));
sessions.merge(key, newValue, (old, new_) -> old.merge(new_));

// Copy-on-write for read-heavy, write-rare
private final CopyOnWriteArrayList<EventListener> listeners = new CopyOnWriteArrayList<>();
```

### CompletableFuture

```java
CompletableFuture<UserProfile> profile = CompletableFuture
    .supplyAsync(() -> fetchUser(userId))
    .thenCombine(
        CompletableFuture.supplyAsync(() -> fetchOrders(userId)),
        (user, orders) -> new UserProfile(user, orders)
    )
    .exceptionally(ex -> {
        log.error("Failed to build profile", ex);
        return UserProfile.empty();
    });
```

## Design Patterns

### Builder Pattern

```java
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final Duration timeout;

    private HttpRequest(Builder builder) {
        this.url = Objects.requireNonNull(builder.url, "url is required");
        this.method = builder.method;
        this.headers = Map.copyOf(builder.headers);
        this.timeout = builder.timeout;
    }

    public static Builder builder(String url) {
        return new Builder(url);
    }

    public static class Builder {
        private final String url;
        private String method = "GET";
        private final Map<String, String> headers = new LinkedHashMap<>();
        private Duration timeout = Duration.ofSeconds(30);

        private Builder(String url) { this.url = url; }

        public Builder method(String method) { this.method = method; return this; }
        public Builder header(String key, String value) { headers.put(key, value); return this; }
        public Builder timeout(Duration timeout) { this.timeout = timeout; return this; }

        public HttpRequest build() { return new HttpRequest(this); }
    }
}

// Usage
var request = HttpRequest.builder("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .timeout(Duration.ofSeconds(10))
    .build();
```

### Strategy Pattern with Functional Interfaces

```java
@FunctionalInterface
public interface PricingStrategy {
    BigDecimal calculate(Order order);
}

public class PricingService {
    private final Map<CustomerTier, PricingStrategy> strategies = Map.of(
        CustomerTier.STANDARD, order -> order.subtotal(),
        CustomerTier.PREMIUM, order -> order.subtotal().multiply(new BigDecimal("0.9")),
        CustomerTier.VIP, order -> order.subtotal().multiply(new BigDecimal("0.8"))
    );

    public BigDecimal calculatePrice(Order order, CustomerTier tier) {
        return strategies.getOrDefault(tier, o -> o.subtotal()).calculate(order);
    }
}
```

### Repository Pattern

```java
public interface Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    T save(T entity);
    void deleteById(ID id);
    boolean existsById(ID id);
}

public interface UserRepository extends Repository<User, String> {
    Optional<User> findByEmail(String email);
    List<User> findByRole(Role role);
}
```

## Module Organization

### Recommended Package Structure

```
com.example.myapp/
├── Application.java           # Main entry point
├── config/                    # Configuration classes
│   ├── AppConfig.java
│   └── SecurityConfig.java
├── domain/                    # Domain models
│   ├── User.java
│   ├── Order.java
│   └── event/                 # Domain events
├── repository/                # Data access
│   ├── UserRepository.java    # Interface
│   └── JdbcUserRepository.java
├── service/                   # Business logic
│   ├── UserService.java
│   └── OrderService.java
├── handler/                   # HTTP/API layer
│   ├── UserHandler.java
│   └── dto/                   # Request/Response DTOs
│       ├── CreateUserRequest.java
│       └── UserResponse.java
└── exception/                 # Custom exceptions
    ├── AppException.java
    └── NotFoundException.java
```

## Quick Reference: Modern Java Idioms

| Idiom | Version | Description |
|-------|---------|-------------|
| `var x = ...` | 10+ | Local variable type inference |
| `List.of(...)` | 9+ | Immutable collection factory |
| `"text".formatted(args)` | 15+ | String formatting |
| `"""text block"""` | 15+ | Multi-line strings |
| `record Foo(...)` | 16+ | Immutable data carriers |
| `instanceof Pattern p` | 16+ | Pattern matching for instanceof |
| `sealed interface` | 17+ | Restricted type hierarchies |
| `switch expression` | 14+ | Switch as expression with `->` |
| Virtual Threads | 21+ | Lightweight threads |
| `SequencedCollection` | 21+ | Ordered collection interface |

## Anti-Patterns to Avoid

```java
// Bad: Returning null
public User findById(String id) { return null; }
// Good: Return Optional
public Optional<User> findById(String id) { return Optional.empty(); }

// Bad: Raw types
List items = new ArrayList();
// Good: Parameterized types
List<String> items = new ArrayList<>();

// Bad: Checked exception abuse
public void process() throws Exception { }
// Good: Specific exceptions
public void process() throws ProcessingException { }

// Bad: Mutable singleton state
public class AppState {
    public static Map<String, Object> data = new HashMap<>(); // Mutable global state
}
// Good: Immutable configuration
public record AppConfig(String dbUrl, int port) {}

// Bad: String concatenation in loops
String result = "";
for (var s : parts) { result += s; }
// Good: StringBuilder or String.join
String result = String.join("", parts);

// Bad: C-style loop when enhanced for-loop works
for (int i = 0; i < list.size(); i++) { process(list.get(i)); }
// Good: Enhanced for-loop
for (var item : list) { process(item); }
```
