# Reinhardt QuerySet API Reference

All database operations use `reinhardt-query` for type-safe query construction. NEVER use raw SQL strings.

## Schema Operations (DDL)

### CREATE TABLE

```rust
use reinhardt_query::{Query, ColumnDef, Table, ColumnType, ForeignKey, Index};

let create_stmt = Query::create_table()
    .table(Users::Table)
    .if_not_exists()
    .col(
        ColumnDef::new(Users::Id)
            .big_integer()
            .not_null()
            .auto_increment()
            .primary_key(),
    )
    .col(
        ColumnDef::new(Users::Username)
            .string_len(150)
            .not_null()
            .unique_key(),
    )
    .col(
        ColumnDef::new(Users::Email)
            .string_len(254)
            .not_null(),
    )
    .col(
        ColumnDef::new(Users::IsActive)
            .boolean()
            .not_null()
            .default(true),
    )
    .col(
        ColumnDef::new(Users::CreatedAt)
            .timestamp_with_time_zone()
            .not_null()
            .default(Expr::current_timestamp()),
    )
    .col(
        ColumnDef::new(Users::LastLogin)
            .timestamp_with_time_zone()
            .null(),
    )
    .to_owned();

db.execute(create_stmt).await?;
```

### ALTER TABLE

```rust
// Add a column
let alter_stmt = Query::alter_table()
    .table(Users::Table)
    .add_column(
        ColumnDef::new(Users::Bio)
            .text()
            .null(),
    )
    .to_owned();

db.execute(alter_stmt).await?;

// Modify a column
let alter_stmt = Query::alter_table()
    .table(Users::Table)
    .modify_column(
        ColumnDef::new(Users::Username)
            .string_len(255)
            .not_null(),
    )
    .to_owned();

db.execute(alter_stmt).await?;

// Drop a column
let alter_stmt = Query::alter_table()
    .table(Users::Table)
    .drop_column(Users::Bio)
    .to_owned();

db.execute(alter_stmt).await?;
```

### DROP TABLE

```rust
let drop_stmt = Query::drop_table()
    .table(Users::Table)
    .if_exists()
    .to_owned();

db.execute(drop_stmt).await?;
```

### CREATE INDEX

```rust
let index_stmt = Query::index_create()
    .name("idx_users_email")
    .table(Users::Table)
    .col(Users::Email)
    .unique()
    .to_owned();

db.execute(index_stmt).await?;
```

### FOREIGN KEY

```rust
let create_stmt = Query::create_table()
    .table(Posts::Table)
    .col(
        ColumnDef::new(Posts::Id)
            .big_integer()
            .not_null()
            .auto_increment()
            .primary_key(),
    )
    .col(
        ColumnDef::new(Posts::AuthorId)
            .big_integer()
            .not_null(),
    )
    .foreign_key(
        ForeignKey::create()
            .name("fk_posts_author")
            .from(Posts::Table, Posts::AuthorId)
            .to(Users::Table, Users::Id)
            .on_delete(ForeignKeyAction::Cascade)
            .on_update(ForeignKeyAction::Cascade),
    )
    .to_owned();
```

## Data Operations (DML)

### INSERT

```rust
use reinhardt_query::{Query, Expr};

// Single row
let insert_stmt = Query::insert()
    .into_table(Users::Table)
    .columns([Users::Username, Users::Email, Users::IsActive])
    .values_panic(["alice".into(), "alice@example.com".into(), true.into()])
    .to_owned();

db.execute(insert_stmt).await?;

// Multiple rows
let insert_stmt = Query::insert()
    .into_table(Users::Table)
    .columns([Users::Username, Users::Email])
    .values_panic(["alice".into(), "alice@example.com".into()])
    .values_panic(["bob".into(), "bob@example.com".into()])
    .to_owned();

db.execute(insert_stmt).await?;

// Insert with RETURNING (PostgreSQL)
let insert_stmt = Query::insert()
    .into_table(Users::Table)
    .columns([Users::Username, Users::Email])
    .values_panic(["alice".into(), "alice@example.com".into()])
    .returning_col(Users::Id)
    .to_owned();

let id: i64 = db.query_one(insert_stmt).await?;
```

### SELECT with Filter, Order, Limit

```rust
// Basic select
let select_stmt = Query::select()
    .columns([Users::Id, Users::Username, Users::Email])
    .from(Users::Table)
    .to_owned();

let rows = db.query_all(select_stmt).await?;

// Filtered select
let select_stmt = Query::select()
    .columns([Users::Id, Users::Username])
    .from(Users::Table)
    .and_where(Expr::col(Users::IsActive).eq(true))
    .and_where(Expr::col(Users::Username).like("a%"))
    .to_owned();

// Ordering
let select_stmt = Query::select()
    .columns([Users::Id, Users::Username])
    .from(Users::Table)
    .order_by(Users::CreatedAt, Order::Desc)
    .to_owned();

// Limit and offset (pagination)
let select_stmt = Query::select()
    .columns([Users::Id, Users::Username])
    .from(Users::Table)
    .order_by(Users::Id, Order::Asc)
    .limit(10)
    .offset(20)
    .to_owned();

// Join
let select_stmt = Query::select()
    .columns([Posts::Id, Posts::Title])
    .column((Users::Table, Users::Username))
    .from(Posts::Table)
    .inner_join(
        Users::Table,
        Expr::col((Posts::Table, Posts::AuthorId))
            .equals((Users::Table, Users::Id)),
    )
    .to_owned();

// Aggregation
let select_stmt = Query::select()
    .expr_as(Expr::col(Users::Id).count(), Alias::new("total"))
    .from(Users::Table)
    .and_where(Expr::col(Users::IsActive).eq(true))
    .to_owned();
```

### UPDATE

```rust
let update_stmt = Query::update()
    .table(Users::Table)
    .value(Users::IsActive, false.into())
    .value(Users::UpdatedAt, Expr::current_timestamp().into())
    .and_where(Expr::col(Users::Id).eq(42))
    .to_owned();

db.execute(update_stmt).await?;
```

### DELETE

```rust
let delete_stmt = Query::delete()
    .from_table(Users::Table)
    .and_where(Expr::col(Users::IsActive).eq(false))
    .to_owned();

db.execute(delete_stmt).await?;
```

## Column Definition Methods

Quick reference for `ColumnDef` builder methods:

| Method | SQL Effect | Notes |
|--------|------------|-------|
| `.big_integer()` | `BIGINT` | 64-bit integer |
| `.integer()` | `INTEGER` | 32-bit integer |
| `.small_integer()` | `SMALLINT` | 16-bit integer |
| `.float()` | `REAL` | 32-bit float |
| `.double()` | `DOUBLE PRECISION` | 64-bit float |
| `.decimal_len(p, s)` | `DECIMAL(p, s)` | Fixed-point number |
| `.string_len(n)` | `VARCHAR(n)` | Variable-length string |
| `.text()` | `TEXT` | Unbounded text |
| `.boolean()` | `BOOLEAN` | True/false |
| `.timestamp_with_time_zone()` | `TIMESTAMPTZ` | Timestamp with timezone |
| `.timestamp()` | `TIMESTAMP` | Timestamp without timezone |
| `.date()` | `DATE` | Date only |
| `.time()` | `TIME` | Time only |
| `.binary()` | `BYTEA` / `BLOB` | Binary data |
| `.json()` | `JSON` | JSON data |
| `.json_binary()` | `JSONB` | Binary JSON (PostgreSQL) |
| `.uuid()` | `UUID` | UUID type |
| `.not_null()` | `NOT NULL` | Disallow NULL |
| `.null()` | (nullable) | Allow NULL |
| `.default(val)` | `DEFAULT val` | Set default value |
| `.auto_increment()` | `SERIAL` / `AUTO_INCREMENT` | Auto-incrementing |
| `.primary_key()` | `PRIMARY KEY` | Primary key constraint |
| `.unique_key()` | `UNIQUE` | Unique constraint |
