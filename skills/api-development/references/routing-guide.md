# Reinhardt Routing Guide Reference

## App-Level URL Configuration

Each app defines its routes using a `ServerRouter`. Routes are defined in the app's `urls.rs` file.

```rust
// src/apps/user/urls.rs
use reinhardt::urls::prelude::*;
use super::views;

pub fn router() -> ServerRouter {
    let mut router = ServerRouter::new();

    router.route("/", views::list_users);
    router.route("/{id}", views::get_user);
    router.route("/", views::create_user);
    router.route("/{id}", views::update_user);
    router.route("/{id}", views::delete_user);

    router
}
```

### ViewSet Routing

For ViewSets, the router auto-generates standard CRUD routes:

```rust
// src/apps/user/urls.rs
use reinhardt::urls::prelude::*;
use super::views::UserViewSet;

pub fn router() -> ServerRouter {
    let mut router = ServerRouter::new();

    // Registers: GET /, POST /, GET /{id}, PUT /{id}, PATCH /{id}, DELETE /{id}
    router.register_viewset::<UserViewSet>("/");

    router
}
```

## Root-Level URL Configuration

The project's root `urls.rs` uses `UnifiedRouter` to combine all app routers:

```rust
// src/urls.rs
use reinhardt::urls::prelude::*;
use crate::apps;

pub fn root_router() -> UnifiedRouter {
    let mut router = UnifiedRouter::new();

    // Mount app routers under path prefixes
    router.include("/api/users", apps::user::urls::router());
    router.include("/api/posts", apps::post::urls::router());
    router.include("/api/auth", apps::auth::urls::router());

    // Static/admin routes
    router.include("/admin", reinhardt::admin::urls::router());

    router
}
```

## URL Patterns in View Decorators

URL patterns support path parameters with type annotations:

```rust
// Simple path parameter
#[get("/users/{id}")]
pub async fn get_user(request: Request, id: Path<i64>) -> Response { ... }

// Multiple path parameters
#[get("/users/{user_id}/posts/{post_id}")]
pub async fn get_user_post(
    request: Request,
    user_id: Path<i64>,
    post_id: Path<i64>,
) -> Response { ... }

// String path parameter
#[get("/users/{username}")]
pub async fn get_by_username(request: Request, username: Path<String>) -> Response { ... }

// UUID path parameter
#[get("/resources/{uuid}")]
pub async fn get_resource(request: Request, uuid: Path<Uuid>) -> Response { ... }
```

### Path Parameter Types

| Type | Pattern Match | Example |
|------|---------------|---------|
| `Path<i64>` | Integer | `/users/42` |
| `Path<i32>` | Integer | `/items/7` |
| `Path<String>` | Any string segment | `/users/alice` |
| `Path<Uuid>` | UUID format | `/resources/550e8400-e29b-41d4-a716-446655440000` |

## Mounting and Nesting

Routers can be nested to any depth:

```rust
pub fn api_v1_router() -> ServerRouter {
    let mut router = ServerRouter::new();
    router.include("/users", apps::user::urls::router());
    router.include("/posts", apps::post::urls::router());
    router
}

pub fn api_v2_router() -> ServerRouter {
    let mut router = ServerRouter::new();
    router.include("/users", apps::user::urls::v2_router());
    router.include("/posts", apps::post::urls::v2_router());
    router
}

pub fn root_router() -> UnifiedRouter {
    let mut router = UnifiedRouter::new();
    router.include("/api/v1", api_v1_router());
    router.include("/api/v2", api_v2_router());
    router
}
```

This produces routes like:
- `GET /api/v1/users/` -> `apps::user::views::list_users`
- `GET /api/v2/users/` -> `apps::user::views::v2::list_users`

## Per-Route Middleware

Apply middleware to specific routes or groups of routes:

```rust
use reinhardt::middleware::prelude::*;

pub fn router() -> ServerRouter {
    let mut router = ServerRouter::new();

    // Public routes (no auth required)
    router.route("/health", views::health_check);
    router.route("/login", views::login);

    // Protected group with authentication middleware
    let mut protected = ServerRouter::new();
    protected.middleware(AuthenticationMiddleware::new());
    protected.route("/profile", views::get_profile);
    protected.route("/settings", views::update_settings);
    router.include("/", protected);

    // Admin group with additional authorization
    let mut admin = ServerRouter::new();
    admin.middleware(AuthenticationMiddleware::new());
    admin.middleware(RequirePermission::new("is_staff"));
    admin.route("/dashboard", views::admin_dashboard);
    admin.route("/users", views::admin_list_users);
    router.include("/admin", admin);

    router
}
```

### Common Middleware

| Middleware | Description |
|-----------|-------------|
| `AuthenticationMiddleware` | Validates auth credentials and populates `AuthUser` |
| `RequirePermission::new(perm)` | Checks user has the specified permission |
| `CorsMiddleware` | Cross-Origin Resource Sharing headers |
| `RateLimitMiddleware` | Request rate limiting |
| `CompressionMiddleware` | Response compression (gzip, brotli) |
| `LoggingMiddleware` | Request/response logging |
| `SecurityHeadersMiddleware` | Adds security headers (CSP, HSTS, X-Frame-Options) |
