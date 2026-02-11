---
name: rust-testing
description: Rust testing patterns including unit tests, integration tests, property-based testing, async tests, benchmarks, and coverage. Follows TDD methodology with idiomatic Rust practices.
---

# Rust Testing Patterns

Comprehensive Rust testing patterns for writing reliable, maintainable tests following TDD methodology.

## When to Activate

- Writing new Rust functions or types
- Adding test coverage to existing code
- Creating benchmarks for performance-critical code
- Implementing property-based tests for input validation
- Following TDD workflow in Rust projects

## TDD Workflow for Rust

### The RED-GREEN-REFACTOR Cycle

```
RED     → Write a failing test first
GREEN   → Write minimal code to pass the test
REFACTOR → Improve code while keeping tests green
REPEAT  → Continue with next requirement
```

### Step-by-Step TDD in Rust

```rust
// Step 1: Define the interface/signature
// src/calculator.rs
pub fn divide(a: f64, b: f64) -> Result<f64, MathError> {
    todo!()
}

#[derive(Debug, thiserror::Error, PartialEq)]
pub enum MathError {
    #[error("division by zero")]
    DivisionByZero,
}

// Step 2: Write failing test (RED)
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn divide_valid() {
        assert_eq!(divide(10.0, 2.0).unwrap(), 5.0);
    }

    #[test]
    fn divide_by_zero() {
        assert_eq!(divide(10.0, 0.0), Err(MathError::DivisionByZero));
    }
}

// Step 3: Run test - verify FAIL
// $ cargo test
// thread panicked at 'not yet implemented'

// Step 4: Implement minimal code (GREEN)
pub fn divide(a: f64, b: f64) -> Result<f64, MathError> {
    if b == 0.0 {
        return Err(MathError::DivisionByZero);
    }
    Ok(a / b)
}

// Step 5: Run test - verify PASS
// $ cargo test
// test tests::divide_valid ... ok
// test tests::divide_by_zero ... ok

// Step 6: Refactor if needed, verify tests still pass
```

## Unit Tests

### Basic Test Structure

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_basic_addition() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn test_with_message() {
        let result = process("hello");
        assert_eq!(result, "HELLO", "expected uppercase but got: {result}");
    }

    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn test_panic() {
        let v: Vec<i32> = vec![];
        let _ = v[0];
    }

    #[test]
    fn test_result() -> Result<(), Box<dyn std::error::Error>> {
        let config = parse_config("key=value")?;
        assert_eq!(config.get("key"), Some(&"value".to_string()));
        Ok(())
    }
}
```

### Assertion Macros

```rust
// Equality
assert_eq!(actual, expected);
assert_ne!(actual, unexpected);

// Boolean
assert!(condition);
assert!(!condition);

// Pattern matching
assert!(matches!(result, Ok(42)));
assert!(matches!(result, Err(Error::NotFound { .. })));

// Float comparison (approximate)
assert!((actual - expected).abs() < f64::EPSILON);

// Custom messages
assert_eq!(actual, expected, "Failed for input: {input}");
```

### Parameterized Tests via Macro

```rust
macro_rules! test_cases {
    ($($name:ident: $input:expr => $expected:expr),* $(,)?) => {
        $(
            #[test]
            fn $name() {
                let result = process($input);
                assert_eq!(result, $expected, "Failed for input: {:?}", $input);
            }
        )*
    };
}

test_cases! {
    empty_string: "" => 0,
    single_char: "a" => 1,
    multiple_chars: "hello" => 5,
    unicode: "こんにちは" => 5,
}
```

### Test Helpers

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// Create a test user with sensible defaults.
    fn test_user(name: &str) -> User {
        User {
            id: UserId::new(format!("test-{name}")).unwrap(),
            name: name.to_string(),
            email: Email::parse(&format!("{name}@test.com")).unwrap(),
            active: true,
        }
    }

    #[test]
    fn test_deactivate_user() {
        let mut user = test_user("alice");
        user.deactivate();
        assert!(!user.active);
    }
}
```

### Test Fixtures with Setup/Teardown

```rust
struct TestDb {
    pool: PgPool,
    schema: String,
}

impl TestDb {
    async fn new() -> Self {
        let schema = format!("test_{}", uuid::Uuid::new_v4().simple());
        let pool = PgPool::connect(&database_url()).await.unwrap();
        sqlx::query(&format!("CREATE SCHEMA {schema}"))
            .execute(&pool)
            .await
            .unwrap();
        Self { pool, schema }
    }

    fn pool(&self) -> &PgPool {
        &self.pool
    }
}

impl Drop for TestDb {
    fn drop(&mut self) {
        // Cleanup runs automatically
        let pool = self.pool.clone();
        let schema = self.schema.clone();
        tokio::spawn(async move {
            sqlx::query(&format!("DROP SCHEMA {schema} CASCADE"))
                .execute(&pool)
                .await
                .ok();
        });
    }
}

#[tokio::test]
async fn test_create_user() {
    let db = TestDb::new().await;
    let repo = PostgresUserRepo::new(db.pool().clone());

    let user = repo.create("Alice", "alice@example.com").await.unwrap();
    assert_eq!(user.name, "Alice");
}
```

## Integration Tests

### File Structure

```
my_crate/
├── src/
│   └── lib.rs
├── tests/
│   ├── common/
│   │   └── mod.rs      # Shared test helpers
│   ├── api_tests.rs     # API integration tests
│   └── db_tests.rs      # Database integration tests
```

### Shared Test Helpers

```rust
// tests/common/mod.rs
use my_crate::Config;

pub fn test_config() -> Config {
    Config {
        database_url: "postgres://localhost/test".to_string(),
        port: 0, // Random available port
        log_level: "debug".to_string(),
    }
}

pub async fn spawn_app() -> TestApp {
    let config = test_config();
    let app = my_crate::Application::build(config).await.unwrap();
    let port = app.port();
    tokio::spawn(app.run());

    TestApp {
        address: format!("http://127.0.0.1:{port}"),
        client: reqwest::Client::new(),
    }
}

pub struct TestApp {
    pub address: String,
    pub client: reqwest::Client,
}
```

### API Integration Test

```rust
// tests/api_tests.rs
mod common;

#[tokio::test]
async fn health_check_works() {
    let app = common::spawn_app().await;

    let response = app.client
        .get(format!("{}/health", app.address))
        .send()
        .await
        .expect("failed to send request");

    assert!(response.status().is_success());
}

#[tokio::test]
async fn create_user_returns_201() {
    let app = common::spawn_app().await;

    let response = app.client
        .post(format!("{}/users", app.address))
        .json(&serde_json::json!({
            "name": "Alice",
            "email": "alice@example.com"
        }))
        .send()
        .await
        .expect("failed to send request");

    assert_eq!(response.status(), 201);

    let body: serde_json::Value = response.json().await.unwrap();
    assert_eq!(body["name"], "Alice");
}
```

## Async Tests

### With Tokio

```rust
#[tokio::test]
async fn test_async_operation() {
    let result = fetch_data("https://example.com").await;
    assert!(result.is_ok());
}

// Multi-threaded runtime
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_concurrent_operations() {
    let (tx, mut rx) = tokio::sync::mpsc::channel(10);

    tokio::spawn(async move {
        tx.send(42).await.unwrap();
    });

    let value = rx.recv().await.unwrap();
    assert_eq!(value, 42);
}
```

### Timeout Tests

```rust
#[tokio::test]
async fn test_with_timeout() {
    let result = tokio::time::timeout(
        Duration::from_secs(5),
        long_running_operation(),
    )
    .await;

    assert!(result.is_ok(), "operation timed out");
}
```

## Mocking

### Manual Mock via Trait

```rust
pub trait EmailSender: Send + Sync {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), SendError>;
}

#[cfg(test)]
struct MockEmailSender {
    sent: std::sync::Mutex<Vec<(String, String, String)>>,
}

#[cfg(test)]
impl MockEmailSender {
    fn new() -> Self {
        Self {
            sent: std::sync::Mutex::new(Vec::new()),
        }
    }

    fn sent_count(&self) -> usize {
        self.sent.lock().unwrap().len()
    }
}

#[cfg(test)]
impl EmailSender for MockEmailSender {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), SendError> {
        self.sent.lock().unwrap().push((
            to.to_string(),
            subject.to_string(),
            body.to_string(),
        ));
        Ok(())
    }
}

#[test]
fn test_notification_sends_email() {
    let sender = Arc::new(MockEmailSender::new());
    let service = NotificationService::new(sender.clone());

    service.notify_user("alice@example.com", "Welcome!").unwrap();

    assert_eq!(sender.sent_count(), 1);
}
```

### With mockall Crate

```rust
use mockall::automock;

#[automock]
pub trait UserRepository {
    fn find_by_id(&self, id: &str) -> Result<Option<User>, DbError>;
    fn save(&self, user: &User) -> Result<(), DbError>;
}

#[test]
fn test_get_user() {
    let mut mock = MockUserRepository::new();
    mock.expect_find_by_id()
        .with(eq("user-1"))
        .times(1)
        .returning(|_| Ok(Some(User {
            id: "user-1".to_string(),
            name: "Alice".to_string(),
        })));

    let service = UserService::new(mock);
    let user = service.get_user("user-1").unwrap();
    assert_eq!(user.unwrap().name, "Alice");
}
```

## Property-Based Testing

### With proptest

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_parse_roundtrip(s in "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}") {
        let email = Email::parse(&s).unwrap();
        assert_eq!(email.as_str(), s.to_lowercase());
    }

    #[test]
    fn test_sort_is_idempotent(mut v in prop::collection::vec(any::<i32>(), 0..100)) {
        v.sort();
        let sorted = v.clone();
        v.sort();
        assert_eq!(v, sorted);
    }

    #[test]
    fn test_encode_decode_roundtrip(data in prop::collection::vec(any::<u8>(), 0..1000)) {
        let encoded = encode(&data);
        let decoded = decode(&encoded).unwrap();
        assert_eq!(data, decoded);
    }
}
```

## Benchmarks

### With criterion

```rust
// benches/throughput.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_process(c: &mut Criterion) {
    let data = generate_test_data(1000);

    c.bench_function("process 1000 items", |b| {
        b.iter(|| process(black_box(&data)))
    });
}

fn bench_comparison(c: &mut Criterion) {
    let data = generate_test_data(1000);
    let mut group = c.benchmark_group("processing");

    group.bench_function("v1", |b| {
        b.iter(|| process_v1(black_box(&data)))
    });

    group.bench_function("v2", |b| {
        b.iter(|| process_v2(black_box(&data)))
    });

    group.finish();
}

criterion_group!(benches, bench_process, bench_comparison);
criterion_main!(benches);
```

```toml
# Cargo.toml
[[bench]]
name = "throughput"
harness = false

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
```

## Doc Tests

```rust
/// Adds two numbers together.
///
/// # Examples
///
/// ```
/// use my_crate::add;
///
/// assert_eq!(add(2, 3), 5);
/// assert_eq!(add(-1, 1), 0);
/// ```
///
/// # Panics
///
/// Panics on integer overflow in debug mode.
///
/// ```should_panic
/// use my_crate::add;
///
/// add(i32::MAX, 1); // Overflow!
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

## Coverage Commands

```bash
# Install tarpaulin
cargo install cargo-tarpaulin

# Basic coverage report
cargo tarpaulin --out Stdout

# HTML report
cargo tarpaulin --out Html

# With specific packages
cargo tarpaulin -p my_crate --out Stdout

# Exclude test code from coverage
cargo tarpaulin --ignore-tests --out Stdout

# With all features
cargo tarpaulin --all-features --out Stdout

# XML report for CI
cargo tarpaulin --out Xml
```

## Coverage Targets

| Code Type | Target |
|-----------|--------|
| Critical business logic | 100% |
| Public APIs | 90%+ |
| General code | 80%+ |
| Generated code | Exclude |
| Unsafe blocks | 100% |

## Test Organization Best Practices

### Naming Conventions

```rust
#[cfg(test)]
mod tests {
    // Pattern: test_<function>_<scenario>_<expected>
    #[test]
    fn parse_email_valid_input_returns_ok() { /* ... */ }

    #[test]
    fn parse_email_empty_string_returns_error() { /* ... */ }

    #[test]
    fn parse_email_no_at_sign_returns_invalid_format() { /* ... */ }
}
```

### Test Module Organization

```rust
#[cfg(test)]
mod tests {
    use super::*;

    mod creation {
        use super::*;

        #[test]
        fn new_user_has_default_role() { /* ... */ }

        #[test]
        fn new_user_is_active() { /* ... */ }
    }

    mod validation {
        use super::*;

        #[test]
        fn rejects_empty_name() { /* ... */ }

        #[test]
        fn rejects_invalid_email() { /* ... */ }
    }
}
```

## TDD Best Practices

**DO:**
- Write test FIRST, before any implementation
- Run tests after each change (`cargo test`)
- Use `#[should_panic]` for expected panics
- Test behavior, not implementation details
- Include edge cases (empty, None, MAX, boundary)
- Use `assert_eq!` with descriptive messages
- Keep tests fast (< 1 second each)

**DON'T:**
- Write implementation before tests
- Skip the RED phase
- Test private implementation details
- Use `thread::sleep` in tests (use `tokio::time::pause`)
- Ignore flaky tests
- Over-mock (prefer real implementations when cheap)

## Quick Reference

| Command | Description |
|---------|-------------|
| `cargo test` | Run all tests |
| `cargo test test_name` | Run specific test |
| `cargo test -- --nocapture` | Show println output |
| `cargo test -- --test-threads=1` | Run sequentially |
| `cargo test --doc` | Run doc tests only |
| `cargo test --lib` | Run unit tests only |
| `cargo test --test integration` | Run specific integration test file |
| `cargo bench` | Run benchmarks |
| `cargo tarpaulin` | Run with coverage |
