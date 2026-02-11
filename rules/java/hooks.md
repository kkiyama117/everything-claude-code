---
paths: "**/*.java"
---

# Java Hooks

> This file extends [common/hooks.md](../common/hooks.md) with Java specific content.

## PostToolUse Hooks

Configure in `~/.claude/settings.json`:

- **google-java-format / Spotless**: Auto-format `.java` files after edit
- **javac / Maven compile**: Run compilation check after editing `.java` files
- **SpotBugs / Error Prone**: Run static analysis on modified classes
