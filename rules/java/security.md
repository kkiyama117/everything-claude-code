---
paths: "**/*.java"
---

# Java Security

> This file extends [common/security.md](../common/security.md) with Java specific content.

## Secret Management

```java
String apiKey = System.getenv("OPENAI_API_KEY");
if (apiKey == null || apiKey.isBlank()) {
    throw new IllegalStateException("OPENAI_API_KEY must be set");
}
```

## Security Scanning

- Use **SpotBugs** with **Find Security Bugs** plugin:
  ```bash
  mvn spotbugs:check
  ```

- Use **OWASP Dependency-Check** for vulnerability scanning:
  ```bash
  mvn org.owasp:dependency-check-maven:check
  ```

## SQL Injection Prevention

Always use parameterized queries:

```java
// Bad
String query = "SELECT * FROM users WHERE id = '" + userId + "'";

// Good: PreparedStatement
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
stmt.setString(1, userId);

// Good: JPA/Hibernate named parameter
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);
```

## Deserialization

Never deserialize untrusted data without validation:

```java
// Bad: Arbitrary deserialization
ObjectInputStream ois = new ObjectInputStream(untrustedInput);
Object obj = ois.readObject(); // Remote code execution risk

// Good: Use JSON with explicit types
ObjectMapper mapper = new ObjectMapper();
mapper.activateDefaultTyping(/* ... */); // Only if needed, with allow-list
User user = mapper.readValue(json, User.class);
```
