---
paths: "**/*.java"
---

# Java Patterns

> This file extends [common/patterns.md](../common/patterns.md) with Java specific content.

## Builder Pattern

```java
public class ServerConfig {
    private final int port;
    private final String host;
    private final int maxConnections;

    private ServerConfig(Builder builder) {
        this.port = builder.port;
        this.host = builder.host;
        this.maxConnections = builder.maxConnections;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private int port = 8080;
        private String host = "localhost";
        private int maxConnections = 100;

        public Builder port(int port) { this.port = port; return this; }
        public Builder host(String host) { this.host = host; return this; }
        public Builder maxConnections(int max) { this.maxConnections = max; return this; }

        public ServerConfig build() {
            return new ServerConfig(this);
        }
    }
}
```

## Interface-Based Dependency Injection

Define interfaces where they are consumed:

```java
public interface UserRepository {
    Optional<User> findById(String id);
    User save(User user);
}

public class UserService {
    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

## Sealed Classes (Java 17+)

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}

public record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
}

public record Rectangle(double width, double height) implements Shape {
    public double area() { return width * height; }
}
```

## Reference

See skill: `java-patterns` for comprehensive Java patterns including concurrency, streams, and module organization.
