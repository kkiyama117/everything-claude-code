---
name: java-testing
description: Java testing patterns using JUnit 5, Mockito, AssertJ, Testcontainers, and TDD methodology. Follows modern Java testing best practices.
---

# Java Testing Patterns

Comprehensive Java testing patterns for writing reliable, maintainable tests following TDD methodology.

## When to Activate

- Writing new Java classes or methods
- Adding test coverage to existing code
- Creating integration tests with databases or external services
- Following TDD workflow in Java projects
- Setting up testing infrastructure

## TDD Workflow for Java

### The RED-GREEN-REFACTOR Cycle

```
RED     → Write a failing test first
GREEN   → Write minimal code to pass the test
REFACTOR → Improve code while keeping tests green
REPEAT  → Continue with next requirement
```

### Step-by-Step TDD in Java

```java
// Step 1: Define the interface/signature
public class Calculator {
    public int add(int a, int b) {
        throw new UnsupportedOperationException("not implemented");
    }
}

// Step 2: Write failing test (RED)
@Test
void shouldAddTwoNumbers() {
    var calc = new Calculator();
    assertThat(calc.add(2, 3)).isEqualTo(5);
}

// Step 3: Run test - verify FAIL
// UnsupportedOperationException: not implemented

// Step 4: Implement minimal code (GREEN)
public int add(int a, int b) {
    return a + b;
}

// Step 5: Run test - verify PASS
// Step 6: Refactor if needed, verify tests still pass
```

## JUnit 5 Fundamentals

### Basic Test Structure

```java
import org.junit.jupiter.api.*;
import static org.assertj.core.api.Assertions.*;

class UserServiceTest {

    private UserService service;
    private UserRepository repository;

    @BeforeEach
    void setUp() {
        repository = new InMemoryUserRepository();
        service = new UserService(repository);
    }

    @Test
    @DisplayName("should create user with valid input")
    void createUser_validInput_returnsUser() {
        var user = service.create("Alice", "alice@example.com");

        assertThat(user.getName()).isEqualTo("Alice");
        assertThat(user.getEmail()).isEqualTo("alice@example.com");
        assertThat(user.getId()).isNotNull();
    }

    @Test
    void createUser_emptyName_throwsValidationException() {
        assertThatThrownBy(() -> service.create("", "alice@example.com"))
            .isInstanceOf(ValidationException.class)
            .hasMessageContaining("name");
    }
}
```

### AssertJ Assertions

```java
// Basic
assertThat(actual).isEqualTo(expected);
assertThat(actual).isNotNull();
assertThat(actual).isInstanceOf(User.class);

// Strings
assertThat(name).startsWith("Al").endsWith("ce").hasSize(5);

// Collections
assertThat(users).hasSize(3)
    .extracting(User::name)
    .containsExactly("Alice", "Bob", "Charlie");

assertThat(items).filteredOn(Item::isActive).hasSize(2);

// Exceptions
assertThatThrownBy(() -> service.process(null))
    .isInstanceOf(NullPointerException.class)
    .hasMessageContaining("must not be null");

assertThatCode(() -> service.validate(input)).doesNotThrowAnyException();

// Optional
assertThat(result).isPresent().hasValueSatisfying(user ->
    assertThat(user.name()).isEqualTo("Alice")
);
```

### Nested Tests

```java
class OrderServiceTest {

    @Nested
    @DisplayName("when creating an order")
    class WhenCreating {

        @Test
        void shouldCalculateTotal() { /* ... */ }

        @Test
        void shouldApplyDiscount() { /* ... */ }

        @Nested
        @DisplayName("with empty cart")
        class WithEmptyCart {
            @Test
            void shouldThrowException() { /* ... */ }
        }
    }

    @Nested
    @DisplayName("when cancelling an order")
    class WhenCancelling {

        @Test
        void shouldRefundPayment() { /* ... */ }

        @Test
        void shouldUpdateStatus() { /* ... */ }
    }
}
```

### Parameterized Tests

```java
@ParameterizedTest
@ValueSource(strings = {"user@example.com", "admin@test.org", "a+b@c.com"})
void shouldAcceptValidEmails(String email) {
    assertThat(validator.isValid(email)).isTrue();
}

@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {"invalid", "@example.com", "user@"})
void shouldRejectInvalidEmails(String email) {
    assertThat(validator.isValid(email)).isFalse();
}

@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "0, 0, 0",
    "-1, 1, 0",
    "100, 200, 300"
})
void shouldAddNumbers(int a, int b, int expected) {
    assertThat(calculator.add(a, b)).isEqualTo(expected);
}

@ParameterizedTest
@MethodSource("provideOrders")
void shouldCalculateDiscount(Order order, BigDecimal expectedDiscount) {
    assertThat(service.calculateDiscount(order)).isEqualByComparingTo(expectedDiscount);
}

static Stream<Arguments> provideOrders() {
    return Stream.of(
        Arguments.of(orderWithTotal(100), new BigDecimal("0")),
        Arguments.of(orderWithTotal(500), new BigDecimal("25")),
        Arguments.of(orderWithTotal(1000), new BigDecimal("100"))
    );
}
```

### Lifecycle and Ordering

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class IntegrationTest {

    @BeforeAll
    static void setUpOnce() {
        // Expensive setup, run once
    }

    @BeforeEach
    void setUp() {
        // Run before each test
    }

    @AfterEach
    void tearDown() {
        // Cleanup after each test
    }

    @Test
    @Order(1)
    void firstStep() { /* ... */ }

    @Test
    @Order(2)
    void secondStep() { /* ... */ }
}
```

## Mocking with Mockito

### Basic Mocking

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository repository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService service;

    @Test
    void shouldReturnUserWhenFound() {
        var user = new User("1", "Alice");
        when(repository.findById("1")).thenReturn(Optional.of(user));

        var result = service.getUser("1");

        assertThat(result).isPresent();
        assertThat(result.get().getName()).isEqualTo("Alice");
        verify(repository).findById("1");
        verifyNoMoreInteractions(repository);
    }

    @Test
    void shouldSendWelcomeEmailOnCreate() {
        when(repository.save(any(User.class)))
            .thenAnswer(inv -> inv.getArgument(0));

        service.createUser("Alice", "alice@example.com");

        verify(emailService).sendWelcome(argThat(user ->
            user.getEmail().equals("alice@example.com")
        ));
    }
}
```

### Argument Captors

```java
@Captor
private ArgumentCaptor<User> userCaptor;

@Test
void shouldSaveUserWithDefaults() {
    service.createUser("Alice", "alice@example.com");

    verify(repository).save(userCaptor.capture());

    var savedUser = userCaptor.getValue();
    assertThat(savedUser.getName()).isEqualTo("Alice");
    assertThat(savedUser.getRole()).isEqualTo(Role.USER); // Default role
    assertThat(savedUser.getCreatedAt()).isNotNull();
}
```

### Spying

```java
@Test
void shouldCallRealMethodExceptOverridden() {
    var realService = new UserService(repository);
    var spy = Mockito.spy(realService);

    // Override specific method
    doReturn(true).when(spy).isFeatureEnabled("new-flow");

    spy.processUser(userId);

    // Real methods are called except the overridden one
    verify(spy).processUser(userId);
}
```

## Integration Testing

### Testcontainers

```java
@Testcontainers
class PostgresIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    private JdbcUserRepository repository;

    @BeforeEach
    void setUp() {
        var dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(postgres.getJdbcUrl());
        dataSource.setUsername(postgres.getUsername());
        dataSource.setPassword(postgres.getPassword());

        repository = new JdbcUserRepository(dataSource);
    }

    @Test
    void shouldPersistAndRetrieveUser() {
        var user = new User("Alice", "alice@example.com");
        repository.save(user);

        var found = repository.findByEmail("alice@example.com");

        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Alice");
    }
}
```

### Spring Boot Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserApiIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateUser() {
        var request = new CreateUserRequest("Alice", "alice@example.com");

        var response = restTemplate.postForEntity("/api/users", request, UserResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().name()).isEqualTo("Alice");
    }
}
```

### MockMvc for Controller Tests

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.findById("1")).thenReturn(Optional.of(new User("1", "Alice")));

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }

    @Test
    void shouldReturn404WhenNotFound() throws Exception {
        when(userService.findById("999")).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/users/999"))
            .andExpect(status().isNotFound());
    }

    @Test
    void shouldValidateInput() throws Exception {
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"name": "", "email": "invalid"}
                    """))
            .andExpect(status().isBadRequest());
    }
}
```

## Test Fixtures and Factories

### Test Data Builder

```java
public class TestUserBuilder {
    private String id = UUID.randomUUID().toString();
    private String name = "Test User";
    private String email = "test@example.com";
    private Role role = Role.USER;
    private boolean active = true;

    public static TestUserBuilder aUser() { return new TestUserBuilder(); }

    public TestUserBuilder withName(String name) { this.name = name; return this; }
    public TestUserBuilder withEmail(String email) { this.email = email; return this; }
    public TestUserBuilder withRole(Role role) { this.role = role; return this; }
    public TestUserBuilder inactive() { this.active = false; return this; }

    public User build() {
        return new User(id, name, email, role, active);
    }
}

// Usage
var admin = TestUserBuilder.aUser().withName("Admin").withRole(Role.ADMIN).build();
var inactiveUser = TestUserBuilder.aUser().inactive().build();
```

## Coverage Commands

```bash
# Maven with JaCoCo
mvn test jacoco:report
# Report: target/site/jacoco/index.html

# Gradle with JaCoCo
./gradlew test jacocoTestReport
# Report: build/reports/jacoco/test/html/index.html

# Coverage check threshold
mvn jacoco:check -Djacoco.minimum=0.80

# Run specific test class
mvn test -Dtest=UserServiceTest

# Run specific test method
mvn test -Dtest="UserServiceTest#shouldCreateUser"

# Run tagged tests
mvn test -Dgroups="integration"
```

## Coverage Targets

| Code Type | Target |
|-----------|--------|
| Critical business logic | 100% |
| Public APIs | 90%+ |
| General code | 80%+ |
| Generated code | Exclude |
| DTOs/Records | Exclude |

## Test Organization Best Practices

### Naming Convention

```java
// Pattern: methodName_scenario_expectedBehavior
@Test void createUser_validInput_returnsCreatedUser() { }
@Test void createUser_emptyName_throwsValidationException() { }
@Test void createUser_duplicateEmail_throwsConflictException() { }

// Or use @DisplayName for readability
@Test
@DisplayName("should create user when input is valid")
void createUser() { }
```

### Test Categories with Tags

```java
@Tag("unit")
class UnitTest { }

@Tag("integration")
class IntegrationTest { }

@Tag("slow")
class PerformanceTest { }
```

## TDD Best Practices

**DO:**
- Write test FIRST, before any implementation
- Run tests after each change
- Use AssertJ for fluent, readable assertions
- Use `@Nested` for logical test grouping
- Use `@ParameterizedTest` for data-driven tests
- Include edge cases (null, empty, boundary values)
- Test behavior, not implementation

**DON'T:**
- Write implementation before tests
- Skip the RED phase
- Test private methods directly
- Use `Thread.sleep` in tests (use Awaitility instead)
- Ignore flaky tests
- Mock everything (prefer real objects when simple)

## Quick Reference

| Action | Command |
|--------|---------|
| Run all tests | `mvn test` / `./gradlew test` |
| Run specific class | `mvn test -Dtest=FooTest` |
| Run specific method | `mvn test -Dtest="FooTest#bar"` |
| Run by tag | `mvn test -Dgroups=unit` |
| Coverage report | `mvn test jacoco:report` |
| Skip tests | `mvn install -DskipTests` |
