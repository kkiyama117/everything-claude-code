# Rust Testing

> This file extends [common/testing.md](../common/testing.md) with Rust specific content.

## Framework

Use the built-in `#[test]` framework with `cargo test`.

## Coverage

```bash
cargo install cargo-tarpaulin
cargo tarpaulin --out Html
```

## Test Organization

- Unit tests: `#[cfg(test)] mod tests` in the same file
- Integration tests: `tests/` directory at crate root
- Doc tests: code examples in `///` doc comments

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    #[should_panic(expected = "division by zero")]
    fn test_divide_by_zero() {
        divide(1, 0);
    }
}
```

## Reference

See skill: `rust-testing` for detailed Rust testing patterns including property-based testing and async tests.
