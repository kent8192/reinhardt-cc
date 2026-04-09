# Reinhardt Function-Like Macros Reference

Function-like procedural macros invoked with `macro_name!(...)` syntax.

---

## `guard!`

**Crate:** `reinhardt-auth/macros`

Generate type-safe permission guard expressions with boolean operators.

### Syntax

```rust
guard!(Permission)              // Single permission
guard!(A & B)                   // AND: All<(A, B)>
guard!(A | B)                   // OR: Any<(A, B)>
guard!(!A)                      // NOT: Not<A>
guard!((A | B) & C)             // Grouping
guard!(mod::path::Permission)   // Qualified paths
```

### Operator Precedence

| Operator | Precedence | Expands To |
|----------|-----------|------------|
| `!` | Highest | `Not<P>` |
| `&` | Middle | `All<(P1, P2)>` |
| `\|` | Lowest | `Any<(P1, P2)>` |

### Expansion Examples

| Input | Expanded Type |
|-------|---------------|
| `guard!(IsAdminUser)` | `Guard<IsAdminUser>` |
| `guard!(IsAuthenticated & IsActiveUser)` | `Guard<All<(IsAuthenticated, IsActiveUser)>>` |
| `guard!(IsAdminUser \| IsStaffUser)` | `Guard<Any<(IsAdminUser, IsStaffUser)>>` |
| `guard!(!IsAnonymous)` | `Guard<Not<IsAnonymous>>` |
| `guard!((A \| B) & C)` | `Guard<All<(Any<(A, B)>, C)>>` |

### Usage

```rust
use reinhardt::auth::prelude::*;

#[get("/admin/", name = "admin_panel")]
pub async fn admin_panel(
    _guard: guard!(IsAdminUser & IsActiveUser),
    #[inject] AuthUser(user): AuthUser<User>,
) -> ViewResult<Response> {
    // Only active admins reach here
}
```

---

## `installed_apps!`

**Crate:** `reinhardt-core/macros`

Compile-time validated application registry.

### Syntax

```rust
installed_apps! {
    users: "users",
    posts: "posts",
    comments: "comments",
}
```

Each entry maps a Rust identifier to an app label string. Duplicate labels are detected at compile time.

### Usage

```rust
// src/config/apps.rs
use reinhardt::installed_apps;

installed_apps! {
    auth: "auth",
    blog: "blog",
    api: "api",
}
```

---

## `path!`

**Crate:** `reinhardt-urls/routers-macros`

Validate URL path syntax at compile time.

### Syntax

```rust
path!("/users/")
path!("/users/{user_id}/")
path!("/users/{user_id}/posts/{post_id}/")
```

### Validation Rules

| Rule | Description |
|------|-------------|
| Must start with `/` | `path!("users/")` is invalid |
| Parameters must be `{snake_case}` | `path!("/users/{userId}/")` is invalid |
| No double slashes | `path!("/users//posts/")` is invalid |
| No `..` path traversal | `path!("/users/../admin/")` is invalid |
| No nested parameters | `path!("/users/{{id}}/")` is invalid |
| Unique parameter names | `path!("/users/{id}/posts/{id}/")` is invalid |

### Usage in Routing

```rust
use reinhardt::urls::path;

let router = UnifiedRouter::new()
    .route(path!("/users/"), user_list)
    .route(path!("/users/{user_id}/"), user_detail);
```

---

## `page!`

**Crate:** `reinhardt-pages/macros`

**Feature:** `pages`

Anonymous component DSL for WASM frontend views.

```rust
page!(|name: String| {
    div {
        h1 { "Hello, {name}!" }
        p { "Welcome to Reinhardt Pages." }
    }
})
```

---

## `head!`

**Crate:** `reinhardt-pages/macros`

**Feature:** `pages`

HTML `<head>` section DSL.

```rust
head!({
    title { "My Page Title" }
    meta { charset: "utf-8" }
    link { rel: "stylesheet", href: "/static/styles.css" }
})
```

---

## `form!`

**Crate:** `reinhardt-pages/macros`

**Feature:** `pages`

Type-safe form component with reactive bindings.

```rust
form! {
    name: String,
    email: String,
    age: i32,
}
```

Generates a form component with input fields bound to the specified types, including validation.

## Dynamic References

For the latest function-like macro definitions:
1. Read `reinhardt/crates/reinhardt-auth/macros/src/lib.rs` for guard!
2. Read `reinhardt/crates/reinhardt-core/macros/src/lib.rs` for installed_apps!
3. Read `reinhardt/crates/reinhardt-urls/routers-macros/src/lib.rs` for path!
4. Read `reinhardt/crates/reinhardt-pages/macros/src/lib.rs` for page!, head!, form!
