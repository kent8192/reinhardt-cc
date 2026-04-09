---
name: modeling
description: Use when defining database models, working with QuerySets, or managing migrations in reinhardt-web applications
---

# Reinhardt Data Modeling

Guide developers through model definition, database operations, and migration management using reinhardt-db and reinhardt-query.

## When to Use

- User defines or modifies database models
- User works with QuerySet operations
- User generates or applies migrations
- User mentions: "model", "table", "migration", "QuerySet", "field", "relation", "ForeignKey", "ManyToMany", "database", "schema"

## Workflow

### Defining a Model

1. Read `references/model-patterns.md` for field types and relation patterns
2. Guide model struct definition with `#[model]` attribute
3. Choose appropriate field types and constraints
4. Define relations (ForeignKey, ManyToMany, OneToOne) if needed
5. Implement `pub use` re-exports in the module entry file

### QuerySet Operations

1. Read `references/queryset-api.md` for available operations
2. Use reinhardt-query for type-safe query construction
3. NEVER use raw SQL — always use reinhardt-query

### Migrations

1. Read `references/migration-guide.md` for the full workflow
2. Generate migration after model changes
3. Review generated migration before applying
4. Apply migration and verify

## Important Rules

- ALWAYS use `reinhardt-query` for query construction, NEVER raw SQL
- Use `reinhardt_query::Query` for schema DDL operations
- Field types map to Rust types (String, i32, i64, bool, Option<T>, DateTime<Utc>)
- ALL model struct fields that can be NULL must use `Option<T>`

## Cross-Domain References

For testing models, read `${CLAUDE_PLUGIN_ROOT}/skills/testing/references/testcontainers.md`.

## Dynamic References

For the latest model API and field types:
1. Read `reinhardt/crates/reinhardt-db/src/lib.rs` for module docs
2. Read `reinhardt/crates/reinhardt-query/src/lib.rs` for query builder API
3. Grep for `#[model]` usage in `reinhardt/tests/` for real examples
