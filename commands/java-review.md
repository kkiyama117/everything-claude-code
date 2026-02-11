---
description: Comprehensive Java code review for modern idioms, concurrency safety, error handling, and security. Invokes the java-reviewer agent.
---

# Java Code Review

This command invokes the **java-reviewer** agent for comprehensive Java-specific code review.

## What This Command Does

1. **Identify Java Changes**: Find modified `.java` files via `git diff`
2. **Run Static Analysis**: Execute `mvn compile`, SpotBugs, Checkstyle
3. **Security Scan**: Check for SQL injection, deserialization, XXE, command injection
4. **Concurrency Review**: Analyze thread safety, lock ordering, virtual thread compatibility
5. **Idiomatic Java Check**: Verify code follows modern Java conventions
6. **Generate Report**: Categorize issues by severity

## When to Use

Use `/java-review` when:
- After writing or modifying Java code
- Before committing Java changes
- Reviewing pull requests with Java code
- Onboarding to a new Java codebase
- Learning modern Java patterns

## Review Categories

### CRITICAL (Must Fix)
- SQL/Command injection vulnerabilities
- Unsafe deserialization
- XXE in XML parsing
- Hardcoded credentials
- Empty catch blocks hiding errors
- Resource leaks (missing try-with-resources)

### HIGH (Should Fix)
- Catching generic `Exception` instead of specific types
- Missing `Optional` for nullable returns on public APIs
- Race conditions without synchronization
- Raw types (missing generics)
- Blocking in virtual threads with `synchronized`
- Mutable collections exposed in APIs

### MEDIUM (Consider)
- Non-idiomatic patterns (C-style loops, string concatenation)
- Missing Javadoc on public methods
- Not using records for DTOs (Java 16+)
- Inefficient stream operations
- Magic numbers without named constants
- Not using `var` for obvious local types

## Automated Checks Run

```bash
# Maven
mvn compile -q 2>&1
mvn spotbugs:check 2>&1
mvn checkstyle:check 2>&1
mvn test 2>&1

# Gradle
./gradlew compileJava 2>&1
./gradlew spotbugsMain 2>&1
./gradlew checkstyleMain 2>&1
./gradlew test 2>&1

# Security
mvn org.owasp:dependency-check-maven:check 2>&1
```

## Example Usage

```text
User: /java-review

Agent:
# Java Code Review Report

## Files Reviewed
- src/main/java/com/example/handler/UserHandler.java (modified)
- src/main/java/com/example/service/AuthService.java (modified)

## Static Analysis Results
✓ mvn compile: No errors
⚠️ SpotBugs: 1 issue found
✓ Checkstyle: No issues

## Issues Found

[CRITICAL] SQL Injection vulnerability
File: src/main/java/com/example/handler/UserHandler.java:42
Issue: User input directly concatenated into SQL query
```java
String query = "SELECT * FROM users WHERE id = '" + userId + "'";
```
Fix: Use PreparedStatement
```java
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
stmt.setString(1, userId);
```

[HIGH] Catching generic Exception
File: src/main/java/com/example/service/AuthService.java:28
Issue: Generic exception catch hides specific failure modes
```java
catch (Exception e) { log.error("failed", e); }
```
Fix: Catch specific exceptions
```java
catch (AuthenticationException e) { log.error("auth failed", e); throw e; }
catch (IOException e) { log.error("network error", e); return fallback(); }
```

## Summary
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

Recommendation: ❌ Block merge until CRITICAL issue is fixed
```

## Approval Criteria

| Status | Condition |
|--------|-----------|
| ✅ Approve | No CRITICAL or HIGH issues |
| ⚠️ Warning | Only MEDIUM issues (merge with caution) |
| ❌ Block | CRITICAL or HIGH issues found |

## Integration with Other Commands

- Use `/java-test` first to ensure tests pass
- Use `/java-build` if build errors occur
- Use `/java-review` before committing
- Use `/code-review` for non-Java specific concerns

## Framework-Specific Reviews

### Spring Boot Projects
- N+1 query issues (use `@EntityGraph` or JOIN FETCH)
- Missing `@Transactional` for multi-step operations
- Improper bean scoping (prototype vs singleton)
- Missing validation annotations (`@Valid`, `@NotNull`)

### Jakarta EE Projects
- CDI scope misuse
- Missing `@Inject` qualifiers
- EJB transaction boundaries
- JAX-RS resource lifecycle

## Related

- Agent: `agents/java-reviewer.md`
- Skills: `skills/java-patterns/`, `skills/java-testing/`
