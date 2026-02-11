---
paths: "**/*.java"
---

# Java Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with Java specific content.

## Formatting

- Use a consistent formatter (google-java-format, Spotless, or IDE built-in)
- Follow [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html) or team convention

## Naming Conventions

- Classes and interfaces: `PascalCase`
- Methods and variables: `camelCase`
- Constants: `SCREAMING_SNAKE_CASE`
- Packages: `all.lowercase.dotted`
- Type parameters: single uppercase letter (`T`, `E`, `K`, `V`)

## Immutability

Prefer immutable objects:

```java
// Good: Immutable record (Java 16+)
public record User(String name, String email) {}

// Good: Unmodifiable collections
var users = List.of("Alice", "Bob");
var config = Map.of("port", "8080", "host", "localhost");

// Bad: Mutable with exposed internals
public class User {
    public List<String> roles; // Mutable and exposed
}

// Good: Defensive copy
public class User {
    private final List<String> roles;

    public User(List<String> roles) {
        this.roles = List.copyOf(roles);
    }

    public List<String> getRoles() {
        return roles; // Already unmodifiable
    }
}
```

## Error Handling

Use specific exceptions with context:

```java
// Good: Specific exception with context
throw new UserNotFoundException("User not found: " + userId);

// Bad: Generic exception
throw new Exception("error");

// Good: Try-with-resources for AutoCloseable
try (var conn = dataSource.getConnection();
     var stmt = conn.prepareStatement(sql)) {
    return stmt.executeQuery();
}
```

## Reference

See skill: `java-patterns` for comprehensive Java idioms and patterns.
