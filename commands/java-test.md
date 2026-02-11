---
description: Enforce TDD workflow for Java. Write JUnit 5 tests first, then implement. Verify 80%+ coverage with JaCoCo.
---

# Java TDD Command

This command enforces test-driven development methodology for Java code using JUnit 5 and modern testing patterns.

## What This Command Does

1. **Define Types/Interfaces**: Scaffold class signatures and interfaces first
2. **Write JUnit 5 Tests**: Create comprehensive test cases (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Code**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ coverage

## When to Use

Use `/java-test` when:
- Implementing new Java classes or methods
- Adding test coverage to existing code
- Fixing bugs (write failing test first)
- Building critical business logic
- Learning TDD workflow in Java

## TDD Cycle

```
RED     → Write failing JUnit 5 test
GREEN   → Implement minimal code to pass
REFACTOR → Improve code, tests stay green
REPEAT  → Next test case
```

## Example Session

```text
User: /java-test I need a service to validate email addresses

Agent:
# TDD Session: Email Validator

## Step 1: Define Interface

```java
// src/main/java/com/example/validator/EmailValidator.java
package com.example.validator;

public class EmailValidator {

    public void validate(String email) {
        throw new UnsupportedOperationException("not implemented");
    }
}
```

## Step 2: Write Tests (RED)

```java
// src/test/java/com/example/validator/EmailValidatorTest.java
package com.example.validator;

import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import static org.assertj.core.api.Assertions.*;

class EmailValidatorTest {

    private final EmailValidator validator = new EmailValidator();

    @Nested
    class ValidEmails {
        @ParameterizedTest
        @ValueSource(strings = {
            "user@example.com",
            "user@mail.example.com",
            "user+tag@example.com",
            "first.last@example.com"
        })
        void shouldAcceptValidEmail(String email) {
            assertThatNoException().isThrownBy(() -> validator.validate(email));
        }
    }

    @Nested
    class InvalidEmails {
        @Test
        void shouldRejectEmptyString() {
            assertThatThrownBy(() -> validator.validate(""))
                .isInstanceOf(ValidationException.class)
                .hasMessageContaining("empty");
        }

        @Test
        void shouldRejectNull() {
            assertThatThrownBy(() -> validator.validate(null))
                .isInstanceOf(ValidationException.class);
        }

        @ParameterizedTest
        @ValueSource(strings = {"userexample.com", "user@", "@example.com", "user@@example.com"})
        void shouldRejectInvalidFormat(String email) {
            assertThatThrownBy(() -> validator.validate(email))
                .isInstanceOf(ValidationException.class);
        }
    }
}
```

## Step 3: Run Tests - Verify FAIL

```bash
$ mvn test -pl validator
Tests run: 8, Failures: 0, Errors: 8
BUILD FAILURE
```

✓ Tests fail as expected (UnsupportedOperationException).

## Step 4: Implement Minimal Code (GREEN)

```java
package com.example.validator;

import java.util.regex.Pattern;

public class EmailValidator {

    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");

    public void validate(String email) {
        if (email == null || email.isBlank()) {
            throw new ValidationException("email cannot be empty");
        }
        if (!EMAIL_PATTERN.matcher(email).matches()) {
            throw new ValidationException("email format is invalid: " + email);
        }
    }
}
```

## Step 5: Run Tests - Verify PASS

```bash
$ mvn test -pl validator
Tests run: 8, Failures: 0, Errors: 0
BUILD SUCCESS
```

✓ All tests passing!

## Step 6: Check Coverage

```bash
$ mvn test jacoco:report -pl validator
Line Coverage: 100%
Branch Coverage: 100%
```

✓ Coverage: 100%

## TDD Complete!
```

## Test Patterns

### Parameterized Tests
```java
@ParameterizedTest
@CsvSource({
    "2, 3, 5",
    "-1, 1, 0",
    "0, 0, 0"
})
void testAdd(int a, int b, int expected) {
    assertThat(calculator.add(a, b)).isEqualTo(expected);
}
```

### Mockito
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repository;
    @InjectMocks UserService service;

    @Test
    void shouldReturnUser() {
        when(repository.findById("1")).thenReturn(Optional.of(new User("Alice")));

        var user = service.getUser("1");

        assertThat(user).isPresent();
        assertThat(user.get().getName()).isEqualTo("Alice");
        verify(repository).findById("1");
    }
}
```

### Testcontainers
```java
@Testcontainers
class DatabaseIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Test
    void shouldConnectToDatabase() {
        try (var conn = DriverManager.getConnection(
                postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword())) {
            assertThat(conn.isValid(1)).isTrue();
        }
    }
}
```

## Coverage Commands

```bash
# Maven with JaCoCo
mvn test jacoco:report
# Report at: target/site/jacoco/index.html

# Gradle with JaCoCo
./gradlew test jacocoTestReport
# Report at: build/reports/jacoco/test/html/index.html

# Coverage check (fail build if below threshold)
mvn jacoco:check
```

## Coverage Targets

| Code Type | Target |
|-----------|--------|
| Critical business logic | 100% |
| Public APIs | 90%+ |
| General code | 80%+ |
| Generated code | Exclude |

## TDD Best Practices

**DO:**
- Write test FIRST, before any implementation
- Run tests after each change
- Use AssertJ for fluent, readable assertions
- Use `@Nested` for logical test grouping
- Use `@ParameterizedTest` for data-driven tests
- Include edge cases (null, empty, boundary values)

**DON'T:**
- Write implementation before tests
- Skip the RED phase
- Test private methods directly (test via public API)
- Use `Thread.sleep` in tests
- Ignore flaky tests

## Related Commands

- `/java-build` - Fix build errors
- `/java-review` - Review code after implementation
- `/verify` - Run full verification loop

## Related

- Skill: `skills/java-testing/`
- Skill: `skills/tdd-workflow/`
