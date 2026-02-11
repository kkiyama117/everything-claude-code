---
description: Enforce TDD workflow for Rust. Write tests first using #[test] and assert macros, then implement. Verify 80%+ coverage with cargo-tarpaulin.
---

# Rust TDD Command

This command enforces test-driven development methodology for Rust code using idiomatic Rust testing patterns.

## What This Command Does

1. **Define Types/Traits**: Scaffold function signatures and trait definitions first
2. **Write Tests**: Create comprehensive test cases (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Code**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ coverage

## When to Use

Use `/rust-test` when:
- Implementing new Rust functions or types
- Adding test coverage to existing code
- Fixing bugs (write failing test first)
- Building critical business logic
- Learning TDD workflow in Rust

## TDD Cycle

```
RED     → Write failing test with #[test]
GREEN   → Implement minimal code to pass
REFACTOR → Improve code, tests stay green
REPEAT  → Next test case
```

## Example Session

```text
User: /rust-test I need a function to validate email addresses

Agent:
# TDD Session: Email Validator

## Step 1: Define Interface

```rust
// src/validator.rs

/// Validates that the given string is a valid email address.
/// Returns Ok(()) if valid, Err describing the issue otherwise.
pub fn validate_email(email: &str) -> Result<(), ValidationError> {
    todo!()
}

#[derive(Debug, thiserror::Error)]
pub enum ValidationError {
    #[error("email cannot be empty")]
    Empty,
    #[error("email format is invalid: {0}")]
    InvalidFormat(String),
}
```

## Step 2: Write Tests (RED)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_simple_email() {
        assert!(validate_email("user@example.com").is_ok());
    }

    #[test]
    fn valid_with_subdomain() {
        assert!(validate_email("user@mail.example.com").is_ok());
    }

    #[test]
    fn valid_with_plus() {
        assert!(validate_email("user+tag@example.com").is_ok());
    }

    #[test]
    fn invalid_empty() {
        let err = validate_email("").unwrap_err();
        assert!(matches!(err, ValidationError::Empty));
    }

    #[test]
    fn invalid_no_at_sign() {
        assert!(validate_email("userexample.com").is_err());
    }

    #[test]
    fn invalid_no_domain() {
        assert!(validate_email("user@").is_err());
    }

    #[test]
    fn invalid_no_local_part() {
        assert!(validate_email("@example.com").is_err());
    }

    #[test]
    fn invalid_spaces() {
        assert!(validate_email("user @example.com").is_err());
    }
}
```

## Step 3: Run Tests - Verify FAIL

```bash
$ cargo test

running 8 tests
test tests::valid_simple_email ... FAILED (not yet implemented)

FAILED. 8 tests failed.
```

✓ Tests fail as expected (todo!() panic).

## Step 4: Implement Minimal Code (GREEN)

```rust
use once_cell::sync::Lazy;
use regex::Regex;

static EMAIL_REGEX: Lazy<Regex> = Lazy::new(|| {
    Regex::new(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$").unwrap()
});

pub fn validate_email(email: &str) -> Result<(), ValidationError> {
    if email.is_empty() {
        return Err(ValidationError::Empty);
    }
    if !EMAIL_REGEX.is_match(email) {
        return Err(ValidationError::InvalidFormat(email.to_string()));
    }
    Ok(())
}
```

## Step 5: Run Tests - Verify PASS

```bash
$ cargo test

running 8 tests
test tests::valid_simple_email ... ok
test tests::valid_with_subdomain ... ok
test tests::valid_with_plus ... ok
test tests::invalid_empty ... ok
test tests::invalid_no_at_sign ... ok
test tests::invalid_no_domain ... ok
test tests::invalid_no_local_part ... ok
test tests::invalid_spaces ... ok

test result: ok. 8 passed; 0 failed
```

✓ All tests passing!

## Step 6: Check Coverage

```bash
$ cargo tarpaulin --out Stdout

|| Tested/Total Lines:
|| src/validator.rs: 8/8 100.00%
||
|| 100.00% coverage, 8/8 lines covered
```

✓ Coverage: 100%

## TDD Complete!
```

## Test Patterns

### Unit Tests (Same File)
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_basic_case() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn test_panic_case() {
        add(u32::MAX, 1);
    }
}
```

### Parameterized Tests (via macro)
```rust
macro_rules! test_cases {
    ($($name:ident: $input:expr => $expected:expr),* $(,)?) => {
        $(
            #[test]
            fn $name() {
                assert_eq!(process($input), $expected);
            }
        )*
    };
}

test_cases! {
    empty_input: "" => 0,
    single_word: "hello" => 5,
    multiple_words: "hello world" => 11,
}
```

### Async Tests (with tokio)
```rust
#[tokio::test]
async fn test_async_fetch() {
    let result = fetch_data("https://example.com").await;
    assert!(result.is_ok());
}
```

### Test Fixtures
```rust
struct TestContext {
    db: TestDatabase,
    service: UserService,
}

impl TestContext {
    fn new() -> Self {
        let db = TestDatabase::new();
        let service = UserService::new(db.pool());
        Self { db, service }
    }
}

impl Drop for TestContext {
    fn drop(&mut self) {
        self.db.cleanup();
    }
}

#[test]
fn test_with_fixture() {
    let ctx = TestContext::new();
    let user = ctx.service.create_user("Alice").unwrap();
    assert_eq!(user.name, "Alice");
}
```

## Coverage Commands

```bash
# Basic coverage
cargo tarpaulin --out Stdout

# HTML report
cargo tarpaulin --out Html

# With specific packages
cargo tarpaulin -p my_crate --out Stdout

# Exclude test code from coverage
cargo tarpaulin --ignore-tests --out Stdout

# With all features
cargo tarpaulin --all-features --out Stdout
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
- Use `#[should_panic]` for expected panics
- Test behavior, not implementation details
- Include edge cases (empty, None, max values, boundary)
- Use `assert_eq!` for better error messages

**DON'T:**
- Write implementation before tests
- Skip the RED phase
- Test private functions directly (test via public API)
- Use `thread::sleep` in tests
- Ignore flaky tests

## Related Commands

- `/rust-build` - Fix build errors
- `/rust-review` - Review code after implementation
- `/verify` - Run full verification loop

## Related

- Skill: `skills/rust-testing/`
- Skill: `skills/tdd-workflow/`
