# Rust Security

> This file extends [common/security.md](../common/security.md) with Rust specific content.

## Secret Management

```rust
let api_key = std::env::var("OPENAI_API_KEY")
    .expect("OPENAI_API_KEY must be set");
```

## Security Scanning

- Use **cargo-audit** for dependency vulnerability scanning:
  ```bash
  cargo install cargo-audit
  cargo audit
  ```

- Use **cargo-deny** for license and advisory checks:
  ```bash
  cargo install cargo-deny
  cargo deny check
  ```

## Unsafe Code

Minimize `unsafe` usage — every `unsafe` block must have a `// SAFETY:` comment:

```rust
// SAFETY: We verified that the pointer is non-null and properly aligned
// in the check above, and the data it points to is valid for 'a.
unsafe {
    &*ptr
}
```

## Input Validation

Validate all external input at boundaries:

```rust
pub fn parse_port(input: &str) -> Result<u16, AppError> {
    input.parse::<u16>()
        .map_err(|_| AppError::InvalidInput(format!("invalid port: {input}")))
}
```
