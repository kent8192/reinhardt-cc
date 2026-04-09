---
name: dependency-injection
description: Use when configuring dependency injection in reinhardt-web applications - covers injectable services, scoping, and integration with database and auth
---

# Reinhardt Dependency Injection

Guide developers through DI configuration using reinhardt-di, including service registration, scoping, and integration with database and authentication.

## When to Use

- User configures or creates injectable services
- User asks about DI patterns or scoping
- User mentions: "DI", "dependency injection", "inject", "Provider", "scope", "singleton", "request-scoped", "Injectable"

## Workflow

### Adding a New Injectable Service

1. Read `references/di-patterns.md` for injection patterns
2. Determine scope (request-scoped vs singleton)
3. Implement `Injectable` trait or use auto-implementation
4. Use `#[inject]` in handlers to receive the dependency

### Integrating with Database/Auth

1. Read `references/di-with-db.md` for database pool and auth injection
2. Use built-in types: `DatabaseConnection`, `AuthUser<T>`, `SessionData`
3. These are already injectable — just use `#[inject]` in handlers

## Important Rules

- Types implementing `Default + Clone + Send + Sync + 'static` get auto-injection
- Custom injection logic requires `#[async_trait] impl Injectable`
- Reinhardt DI checks request scope first, then singleton scope
- Circular dependencies are detected at runtime — avoid them by design

## Dynamic References

For the latest DI API:
1. Read `reinhardt/crates/reinhardt-di/src/lib.rs` for types and traits
2. Read `reinhardt/crates/reinhardt-di/macros/src/lib.rs` for macro documentation
3. Grep for `#[inject]` in `reinhardt/tests/` for real usage examples
