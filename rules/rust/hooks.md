# Rust Hooks

> This file extends [common/hooks.md](../common/hooks.md) with Rust specific content.

## PostToolUse Hooks

Configure in `~/.claude/settings.json`:

- **rustfmt**: Auto-format `.rs` files after edit
- **cargo check**: Run type/borrow checker after editing `.rs` files
- **clippy**: Run extended lint checks on modified crates
