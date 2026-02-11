---
description: Fix Rust build errors, borrow checker issues, and clippy warnings incrementally. Invokes the rust-build-resolver agent for minimal, surgical fixes.
---

# Rust Build and Fix

This command invokes the **rust-build-resolver** agent to incrementally fix Rust build errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `cargo check`, `cargo clippy`
2. **Parse Errors**: Group by file and sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run `cargo check` after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/rust-build` when:
- `cargo build` fails with errors
- `cargo clippy` reports warnings
- Borrow checker rejects your code
- After pulling changes that break the build
- Dependency or feature flag issues

## Diagnostic Commands Run

```bash
# Primary build check
cargo check 2>&1

# Full build
cargo build 2>&1

# Lint checks
cargo clippy -- -D warnings 2>&1

# Format check
cargo fmt -- --check

# Dependency issues
cargo update --dry-run 2>&1
```

## Example Session

```text
User: /rust-build

Agent:
# Rust Build Resolution

## Initial Diagnostics

```bash
$ cargo check 2>&1
error[E0382]: borrow of moved value: `data`
  --> src/service/processor.rs:25:20
error[E0277]: the trait bound `MyStruct: Debug` is not satisfied
  --> src/handler/api.rs:42:15
error[E0308]: mismatched types
  --> src/handler/api.rs:58:12
```

Errors found: 3

## Fix 1: Borrow of Moved Value

File: src/service/processor.rs:25
Error: borrow of moved value: `data`
Cause: Value used after being moved into a function call

```rust
// Changed
process(data);
println!("{data}");
// To
process(&data);
println!("{data}");
```

```bash
$ cargo check 2>&1
# 2 errors remaining
```

## Fix 2: Missing Trait Bound

File: src/handler/api.rs:42
Error: the trait bound `MyStruct: Debug` is not satisfied

```rust
// Added derive
#[derive(Debug)]
struct MyStruct {
    name: String,
    count: usize,
}
```

```bash
$ cargo check 2>&1
# 1 error remaining
```

## Fix 3: Type Mismatch

File: src/handler/api.rs:58
Error: expected `String`, found `&str`

```rust
// Changed
let name: String = config.get("name");
// To
let name: String = config.get("name").to_string();
```

```bash
$ cargo check 2>&1
# Build successful!
```

## Final Verification

```bash
$ cargo clippy -- -D warnings 2>&1
# No warnings

$ cargo test 2>&1
test result: ok. 15 passed; 0 failed
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Clippy warnings fixed | 0 |
| Files modified | 2 |
| Remaining issues | 0 |

Build Status: ✅ SUCCESS
```

## Common Errors Fixed

| Error | Typical Fix |
|-------|-------------|
| `E0382: borrow of moved value` | Use reference or clone |
| `E0277: trait bound not satisfied` | Add derive or impl |
| `E0308: mismatched types` | Type conversion (.into(), as, to_string()) |
| `E0106: missing lifetime specifier` | Add lifetime annotations |
| `E0412: cannot find type` | Add use statement |
| `E0433: cannot find crate` | Add to Cargo.toml |
| `E0599: no method named X` | Import trait or fix method name |
| `E0515: cannot return reference to local` | Return owned data |

## Fix Strategy

1. **Compilation errors first** - Code must compile
2. **Borrow checker errors second** - Fix ownership issues
3. **Clippy warnings third** - Style and best practices
4. **One fix at a time** - Verify each change
5. **Minimal changes** - Don't refactor, just fix

## Stop Conditions

The agent will stop and report if:
- Same error persists after 3 attempts
- Fix introduces more errors
- Requires architectural changes
- Missing external crate dependencies

## Related Commands

- `/rust-test` - Run tests after build succeeds
- `/rust-review` - Review code quality
- `/verify` - Full verification loop

## Related

- Agent: `agents/rust-build-resolver.md`
- Skill: `skills/rust-patterns/`
