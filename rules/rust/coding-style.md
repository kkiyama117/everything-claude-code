---
paths: "**/*.rs"
---

# Rust Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with Rust specific content.

## Formatting

- **rustfmt** is mandatory — no style debates
- Use `cargo fmt` before every commit

## Naming Conventions

- Types and traits: `PascalCase`
- Functions, methods, variables: `snake_case`
- Constants and statics: `SCREAMING_SNAKE_CASE`
- Lifetimes: short lowercase (`'a`, `'b`)
- Modules and crates: `snake_case`

## Ownership & Borrowing

Prefer borrowing over cloning:

```rust
// Good: Borrow when you don't need ownership
fn process(data: &str) -> usize {
    data.len()
}

// Bad: Unnecessary clone
fn process(data: String) -> usize {
    data.len()
}
```

## Error Handling

Use the `?` operator and provide context:

```rust
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {path}"))?;
    let config: Config = toml::from_str(&content)
        .context("failed to parse config")?;
    Ok(config)
}
```

For libraries, define typed errors with `thiserror`:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("user {id} not found")]
    UserNotFound { id: String },
    #[error("invalid input: {0}")]
    InvalidInput(String),
    #[error(transparent)]
    Io(#[from] std::io::Error),
}
```

## Immutability

Variables are immutable by default — use `mut` only when necessary:

```rust
// Good: Immutable by default
let config = load_config()?;

// Only use mut when mutation is required
let mut buffer = Vec::new();
buffer.push(42);
```

## Reference

See skill: `rust-patterns` for comprehensive Rust idioms and patterns.
