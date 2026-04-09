---
name: scaffolding
description: Use when creating a new reinhardt project or adding an app - guides feature flag selection, template type, database backend, and authentication setup
---

# Reinhardt Project Scaffolding

Guide developers through creating new reinhardt-web projects and adding apps with correct configuration.

## When to Use

- User wants to create a new reinhardt project
- User wants to add a new app to an existing reinhardt project
- User mentions: "new project", "start project", "add app", "reinhardt-admin startproject", "scaffold", "initialize"

## Prerequisites

- Rust toolchain installed (edition 2024, >= 1.94.0)
- `reinhardt-admin` CLI available (installed via `cargo install reinhardt-admin-cli`)
- For database features: Docker Desktop running (needed for TestContainers)

## Workflow

### New Project

1. **Ask project name** — must be a valid Rust crate name (lowercase, underscores)
2. **Ask template type** — read `references/project-templates.md` for options
3. **Guide feature selection** — read `references/feature-flags.md` for presets and individual features
4. **Ask DB backend** — postgres (recommended), mysql, sqlite, cockroachdb, or none
5. **Ask auth method** — jwt, session, oauth, token, or none
6. **Execute scaffolding**:
   ```bash
   reinhardt-admin startproject <name> [--restful|--with-pages]
   ```
7. **Adjust Cargo.toml** — set feature flags based on selections
8. **Verify** — run `cargo check` to confirm configuration compiles

### Adding an App

1. **Ask app name** — lowercase, singular (e.g., "user", "post", "order")
2. **Execute**:
   ```bash
   reinhardt-admin startapp <name>
   ```
3. **Verify structure** — read `references/app-structure.md` for expected layout
4. **Register app** — ensure app module is added to `src/apps/` and registered

## Important Rules

- ALWAYS use Rust 2024 Edition module system: `module.rs` + `module/` directory, NEVER `mod.rs`
- If generated templates contain `mod.rs` files, convert them to the new module system
- ALL code comments must be in English
- Use `pub use` for explicit re-exports, NEVER `pub use module::*`

## Cross-Domain References

If the user wants to immediately set up models after scaffolding, read
`${CLAUDE_PLUGIN_ROOT}/skills/modeling/references/model-patterns.md`.

## Dynamic References

When you need the latest CLI options or template details:
1. Run `reinhardt-admin startproject --help` and `reinhardt-admin startapp --help`
2. Read `reinhardt/crates/reinhardt-admin-cli/src/` for template implementation
3. Read `reinhardt/crates/reinhardt-commands/src/start_commands.rs` for command definitions
