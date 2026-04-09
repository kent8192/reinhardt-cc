# Reinhardt Migration Guide

## Migration Workflow

The standard workflow for applying model changes to the database:

```
Define/Modify Models → Generate Migration → Review → Apply → Verify
```

### Step 1: Define or Modify Models

Edit your model structs in `src/apps/<name>/models.rs`. See `model-patterns.md` for field types, relations, and attribute options.

### Step 2: Generate Migration

After modifying models, generate a migration file:

```bash
reinhardt-admin makemigrations <app_name>
```

This compares the current model definitions against the last migration state and produces a new migration file in `migrations/<app_name>/`.

Migration files are named with a sequential number and description:
```
migrations/
└── polls/
    ├── 0001_initial.rs
    ├── 0002_add_choice_votes_default.rs
    └── 0003_add_question_is_published.rs
```

### Step 3: Review the Generated Migration

**Always review migrations before applying.** Check that:

- The SQL operations match your intended changes
- No unintended columns are added, dropped, or modified
- Foreign key constraints and indexes are correct
- Default values are appropriate
- The migration is reversible (has both `up` and `down` operations)

```bash
# Preview the SQL that will be executed
reinhardt-admin sqlmigrate <app_name> <migration_name>
```

### Step 4: Apply the Migration

```bash
# Apply all pending migrations
reinhardt-admin migrate

# Apply migrations for a specific app
reinhardt-admin migrate <app_name>

# Apply up to a specific migration
reinhardt-admin migrate <app_name> <migration_name>
```

### Step 5: Verify

```bash
# Check migration status
reinhardt-admin showmigrations

# Run tests to verify models work correctly
cargo nextest run --workspace --all-features
```

## Rollback Procedure

To revert a migration, migrate to the previous migration number:

```bash
# Roll back the last migration for an app
reinhardt-admin migrate <app_name> <previous_migration_name>

# Roll back all migrations for an app (use with caution)
reinhardt-admin migrate <app_name> zero
```

**Rules for rollback:**
- Only roll back migrations that have not been deployed to production
- Ensure the `down` method is implemented in the migration
- After rollback, update or remove the corresponding model changes
- Run tests after rollback to confirm consistency

## Manual Schema Changes with reinhardt-query

For schema changes that cannot be expressed through model definitions (custom indexes, partial indexes, database-specific features), write manual migrations using `reinhardt-query`:

```rust
use reinhardt::db::migration::{Migration, MigrationContext};
use reinhardt_query::{Query, ColumnDef, Expr, Index};

pub struct Migration0004;

impl Migration for Migration0004 {
    fn name(&self) -> &str {
        "0004_add_partial_index_active_users"
    }

    async fn up(&self, ctx: &MigrationContext) -> Result<(), MigrationError> {
        // Create a partial index (PostgreSQL-specific)
        let stmt = Query::index_create()
            .name("idx_users_active_email")
            .table(Users::Table)
            .col(Users::Email)
            .and_where(Expr::col(Users::IsActive).eq(true))
            .to_owned();

        ctx.execute(stmt).await?;
        Ok(())
    }

    async fn down(&self, ctx: &MigrationContext) -> Result<(), MigrationError> {
        let stmt = Query::index_drop()
            .name("idx_users_active_email")
            .table(Users::Table)
            .to_owned();

        ctx.execute(stmt).await?;
        Ok(())
    }
}
```

## Best Practices

1. **One logical change per migration** — Do not combine unrelated schema changes in a single migration. This makes rollback safer and history clearer.

2. **Always review before applying** — Auto-generated migrations may include unintended changes, especially after renaming fields or restructuring models.

3. **Test with TestContainers** — Run migration tests against a real database using TestContainers to catch driver-specific issues:
   ```rust
   #[rstest]
   async fn test_migration_applies_cleanly(db: DatabaseConnection) {
       // Arrange: provided by TestContainers fixture

       // Act
       let result = run_migrations(&db).await;

       // Assert
       assert!(result.is_ok());
   }
   ```

4. **Never edit applied migrations** — Once a migration has been applied to any environment (including other developers' machines), create a new migration instead of modifying the existing one.

5. **Always use reinhardt-query** — Never write raw SQL strings in migrations. Use `reinhardt_query::Query` builders for all DDL and DML operations. This ensures database portability and type safety.

6. **Include both `up` and `down`** — Every migration should be reversible. If a `down` operation is truly impossible (e.g., dropping a column with data), document this clearly in the migration and return an error from `down`.

7. **Coordinate migrations in teams** — When multiple developers create migrations for the same app simultaneously, merge conflicts in migration numbering should be resolved by renumbering and re-testing.
