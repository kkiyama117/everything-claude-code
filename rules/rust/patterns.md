---
paths: "**/*.rs"
---

# Rust Patterns

> This file extends [common/patterns.md](../common/patterns.md) with Rust specific content.

## Builder Pattern

```rust
pub struct ServerBuilder {
    port: u16,
    host: String,
    max_connections: usize,
}

impl ServerBuilder {
    pub fn new() -> Self {
        Self {
            port: 8080,
            host: "127.0.0.1".to_string(),
            max_connections: 100,
        }
    }

    pub fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.host = host.into();
        self
    }

    pub fn build(self) -> Server {
        Server {
            port: self.port,
            host: self.host,
            max_connections: self.max_connections,
        }
    }
}
```

## NewType Pattern

Wrap primitive types for type safety:

```rust
pub struct UserId(String);
pub struct Email(String);

fn send_email(to: &Email, subject: &str) { /* ... */ }
// Cannot accidentally pass UserId where Email is expected
```

## Trait-Based Dependency Injection

Define traits where they are consumed, not where they are implemented:

```rust
pub trait UserRepository: Send + Sync {
    fn find_by_id(&self, id: &str) -> Result<Option<User>>;
    fn save(&self, user: &User) -> Result<()>;
}

pub struct UserService<R: UserRepository> {
    repo: R,
}

impl<R: UserRepository> UserService<R> {
    pub fn new(repo: R) -> Self {
        Self { repo }
    }
}
```

## Error Enum Pattern

Define domain-specific error types:

```rust
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("not found: {0}")]
    NotFound(String),
    #[error("validation failed: {0}")]
    Validation(String),
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}
```

## Reference

See skill: `rust-patterns` for comprehensive Rust patterns including concurrency, async, and module organization.
