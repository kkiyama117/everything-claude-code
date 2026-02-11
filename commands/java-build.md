---
description: Fix Java build errors, type mismatches, and linter warnings incrementally. Invokes the java-build-resolver agent for minimal, surgical fixes.
---

# Java Build and Fix

This command invokes the **java-build-resolver** agent to incrementally fix Java build errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `mvn compile` or `./gradlew compileJava`
2. **Parse Errors**: Group by file and sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run compilation after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/java-build` when:
- `mvn compile` or `./gradlew compileJava` fails
- SpotBugs or Checkstyle reports issues
- Annotation processing fails
- Dependency resolution is broken
- After pulling changes that break the build

## Diagnostic Commands Run

```bash
# Maven
mvn compile -q 2>&1
mvn spotbugs:check 2>&1
mvn checkstyle:check 2>&1

# Gradle
./gradlew compileJava 2>&1
./gradlew spotbugsMain 2>&1
./gradlew checkstyleMain 2>&1

# Dependency issues
mvn dependency:tree -Dverbose 2>&1
```

## Example Session

```text
User: /java-build

Agent:
# Java Build Resolution

## Initial Diagnostics

```bash
$ mvn compile -q 2>&1
[ERROR] src/main/java/com/example/service/UserService.java:[25,15] cannot find symbol
  symbol: class UserRepository
[ERROR] src/main/java/com/example/handler/ApiHandler.java:[42,9] incompatible types
[ERROR] src/main/java/com/example/handler/ApiHandler.java:[58,2] missing return statement
```

Errors found: 3

## Fix 1: Cannot Find Symbol

File: src/main/java/com/example/service/UserService.java:25
Error: cannot find symbol: class UserRepository
Cause: Missing import

```java
// Added import
import com.example.repository.UserRepository;
```

```bash
$ mvn compile -q 2>&1
# 2 errors remaining
```

## Fix 2: Incompatible Types

File: src/main/java/com/example/handler/ApiHandler.java:42
Error: incompatible types: String cannot be converted to int

```java
// Changed
int count = request.getParameter("count");
// To
int count = Integer.parseInt(request.getParameter("count"));
```

```bash
$ mvn compile -q 2>&1
# 1 error remaining
```

## Fix 3: Missing Return

File: src/main/java/com/example/handler/ApiHandler.java:58
Error: missing return statement

```java
// Added missing return
public Optional<User> getUser(String id) {
    if (id == null) {
        return Optional.empty();
    }
    var user = repository.findById(id);
    return Optional.ofNullable(user);
}
```

```bash
$ mvn compile -q 2>&1
# Build successful!
```

## Final Verification

```bash
$ mvn test -q 2>&1
Tests run: 24, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Lint warnings fixed | 0 |
| Files modified | 2 |
| Remaining issues | 0 |

Build Status: ✅ SUCCESS
```

## Common Errors Fixed

| Error | Typical Fix |
|-------|-------------|
| `cannot find symbol` | Add import or fix typo |
| `incompatible types` | Type conversion or fix assignment |
| `missing return statement` | Add return statement |
| `does not override abstract method` | Add missing method implementation |
| `package does not exist` | Add dependency to pom.xml/build.gradle |
| `unreported exception` | Add throws clause or try-catch |
| `variable might not have been initialized` | Initialize variable |

## Fix Strategy

1. **Compilation errors first** - Code must compile
2. **Type errors second** - Fix generics and conversions
3. **Lint warnings third** - Style and best practices
4. **One fix at a time** - Verify each change
5. **Minimal changes** - Don't refactor, just fix

## Stop Conditions

The agent will stop and report if:
- Same error persists after 3 attempts
- Fix introduces more errors
- Requires architectural changes
- Missing external library dependencies

## Related Commands

- `/java-test` - Run tests after build succeeds
- `/java-review` - Review code quality
- `/verify` - Full verification loop

## Related

- Agent: `agents/java-build-resolver.md`
- Skill: `skills/java-patterns/`
