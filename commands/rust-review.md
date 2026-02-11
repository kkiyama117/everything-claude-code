---
description: Comprehensive Rust code review for ownership safety, idiomatic patterns, error handling, concurrency, and security. Invokes the rust-reviewer agent.
---

# Rust Code Review

This command invokes the **rust-reviewer** agent for comprehensive Rust-specific code review.

## What This Command Does

1. **Identify Rust Changes**: Find modified `.rs` files via `git diff`
2. **Run Static Analysis**: Execute `cargo check`, `cargo clippy`
3. **Security Scan**: Check for unsafe misuse, SQL injection, command injection
4. **Ownership Review**: Analyze borrowing, lifetimes, and ownership correctness
5. **Concurrency Review**: Analyze Send/Sync bounds, async safety, lock ordering
6. **Idiomatic Rust Check**: Verify code follows Rust conventions and best practices
7. **Generate Report**: Categorize issues by severity

## When to Use

Use `/rust-review` when:
- After writing or modifying Rust code
- Before committing Rust changes
- Reviewing pull requests with Rust code
- Onboarding to a new Rust codebase
- Learning idiomatic Rust patterns

## Review Categories

### CRITICAL (Must Fix)
- Unsafe blocks without SAFETY comments
- SQL/Command injection vulnerabilities
- Hardcoded credentials
- `.unwrap()` in production paths
- Data races via incorrect Send/Sync usage
- Memory safety violations

### HIGH (Should Fix)
- Missing error context (bare `?` without `.context()`)
- Unnecessary cloning in hot paths
- Blocking calls inside async functions
- Missing trait bounds
- Panic instead of Result returns
- Deadlock-prone lock ordering

### MEDIUM (Consider)
- Non-idiomatic patterns (manual loops vs iterators)
- Missing derive macros (Debug, Clone)
- Clippy pedantic suggestions
- Missing doc comments on public items
- String where &str suffices
- Missing `Vec::with_capacity` when size is known

## Automated Checks Run

```bash
# Type and borrow checking
cargo check 2>&1

# Lint checks
cargo clippy -- -D warnings 2>&1

# Format check
cargo fmt -- --check

# Security audit (if installed)
cargo audit 2>/dev/null || echo "cargo-audit not installed"

# Tests
cargo test --all-features 2>&1
```

## Example Usage

```text
User: /rust-review

Agent:
# Rust Code Review Report

## Files Reviewed
- src/handler/user.rs (modified)
- src/service/auth.rs (modified)

## Static Analysis Results
✓ cargo check: No errors
⚠️ cargo clippy: 2 warnings

## Issues Found

[CRITICAL] Unsafe block without SAFETY comment
File: src/service/auth.rs:67
Issue: unsafe block has no documentation of invariants
```rust
unsafe { std::str::from_utf8_unchecked(bytes) }
```
Fix: Add SAFETY comment or use safe alternative
```rust
// Prefer safe version
std::str::from_utf8(bytes)?

// Or if unsafe is truly needed:
// SAFETY: bytes are guaranteed valid UTF-8 because they come
// from a String that was split on ASCII boundaries only.
unsafe { std::str::from_utf8_unchecked(bytes) }
```

[HIGH] Missing error context
File: src/handler/user.rs:32
Issue: Error propagated without context
```rust
let user = repo.find_by_id(id)?;  // No context
```
Fix: Add context
```rust
let user = repo.find_by_id(id)
    .with_context(|| format!("failed to find user {id}"))?;
```

## Summary
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

Recommendation: ❌ Block merge until CRITICAL issue is fixed
```

## Approval Criteria

| Status | Condition |
|--------|-----------|
| ✅ Approve | No CRITICAL or HIGH issues |
| ⚠️ Warning | Only MEDIUM issues (merge with caution) |
| ❌ Block | CRITICAL or HIGH issues found |

## Integration with Other Commands

- Use `/rust-test` first to ensure tests pass
- Use `/rust-build` if build errors occur
- Use `/rust-review` before committing
- Use `/code-review` for non-Rust specific concerns

## Framework-Specific Reviews

### Actix-web Projects
- Correct use of extractors and guards
- Proper error response types
- Middleware configuration
- State management with `web::Data`

### Axum Projects
- Router composition patterns
- Extractor ordering
- Tower middleware integration
- State sharing with `Extension` or `State`

### Tokio Projects
- Runtime configuration (multi-thread vs current-thread)
- Spawn vs spawn_blocking usage
- Cancellation safety
- Graceful shutdown patterns

## Related

- Agent: `agents/rust-reviewer.md`
- Skills: `skills/rust-patterns/`, `skills/rust-testing/`
