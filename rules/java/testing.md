---
paths: "**/*.java"
---

# Java Testing

> This file extends [common/testing.md](../common/testing.md) with Java specific content.

## Framework

Use **JUnit 5** as the testing framework with **Mockito** for mocking.

## Coverage

```bash
# Maven with JaCoCo
mvn test jacoco:report

# Gradle with JaCoCo
./gradlew test jacocoTestReport
```

## Test Organization

- Unit tests: `src/test/java/` mirroring source structure
- Integration tests: separate source set or `@Tag("integration")`
- Use `@Nested` for grouping related tests

```java
import org.junit.jupiter.api.*;
import static org.assertj.core.api.Assertions.*;

class UserServiceTest {

    @Nested
    class WhenCreatingUser {
        @Test
        void shouldCreateWithValidInput() {
            var user = service.create("Alice", "alice@example.com");
            assertThat(user.getName()).isEqualTo("Alice");
        }

        @Test
        void shouldRejectEmptyName() {
            assertThatThrownBy(() -> service.create("", "alice@example.com"))
                .isInstanceOf(ValidationException.class)
                .hasMessageContaining("name");
        }
    }
}
```

## Reference

See skill: `java-testing` for detailed Java testing patterns including parameterized tests and Testcontainers.
