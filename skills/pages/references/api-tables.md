# API Client & Table Component Reference

## API Client (Django QuerySet-like)

### ApiModel

Derive `ApiModel` to connect a struct to a REST endpoint:

```rust
use reinhardt::pages::prelude::*;

#[derive(ApiModel)]
#[api(endpoint = "/api/users/")]
struct User {
    id: i64,
    username: String,
    email: String,
}
```

### ApiQuerySet

Familiar Django-style query interface for WASM:

```rust
// Get all
let users = User::objects()
    .all()
    .await?;

// Get by ID
let user = User::objects()
    .get(1)
    .await?;

// Filter
let active_users = User::objects()
    .filter("is_active", true)
    .order_by(&["-created_at"])
    .limit(10)
    .all()
    .await?;
```

### Filter and FilterOp

| Method | Description |
|--------|-------------|
| `.filter(field, value)` | Equality filter |
| `.exclude(field, value)` | Exclusion filter |
| `.order_by(&[fields])` | Sort (prefix `-` for descending) |
| `.limit(n)` | Limit results |
| `.all()` | Execute and return all results |
| `.get(id)` | Get single item by ID |

```rust
use reinhardt::pages::prelude::{Filter, FilterOp};

let filter = Filter::new("age", FilterOp::Gte, 18);
let users = User::objects()
    .filter_with(filter)
    .all()
    .await?;
```

### CSRF Integration

API calls automatically include CSRF tokens:

```rust
use reinhardt::pages::prelude::get_csrf_token;

// Manual CSRF token access (rarely needed)
let token = get_csrf_token();
```

## Table Component

Django-tables2 equivalent for rendering data tables.

### Column Types

| Column | Description |
|--------|-------------|
| `Column<T>` | Basic typed column |
| `LinkColumn` | Column with link |
| `BooleanColumn` | True/false display |
| `CheckBoxColumn` | Checkbox column |
| `DateTimeColumn` | Date/time formatting |
| `EmailColumn` | Email link |
| `ChoiceColumn` | Display mapped choices |
| `TemplateColumn` | Custom template rendering |
| `JSONColumn` | JSON data display |
| `URLColumn` | URL link |

### Basic Table Setup

```rust
use reinhardt::pages::tables::*;
use reinhardt::pages::tables::columns::*;

let name_col = Column::<String>::new("name", "Name");
let email_col = EmailColumn::new("email", "Email");
let active_col = BooleanColumn::new("is_active", "Active");
let profile_col = LinkColumn::new("id", "Profile", "/users/{id}");

let table = Table::new(vec![
    Box::new(name_col),
    Box::new(email_col),
    Box::new(active_col),
    Box::new(profile_col),
]);
```

### Sorting

```rust
use reinhardt::pages::tables::{Sortable, SortDirection};

// Tables implement Sortable trait
table.sort_by("name", SortDirection::Ascending);
```

### Pagination

```rust
use reinhardt::pages::tables::Pagination;

let pagination = Pagination::new(total_items, per_page, current_page);
table.paginate(pagination);
```

### Filtering

```rust
use reinhardt::pages::tables::Filterable;

// Tables implement Filterable trait
table.filter("is_active", "true");
```

### Export

```rust
use reinhardt::pages::tables::{ExportFormat, Exportable};

// CSV export
let csv = table.export(ExportFormat::Csv);

// JSON export
let json = table.export(ExportFormat::Json);
```
