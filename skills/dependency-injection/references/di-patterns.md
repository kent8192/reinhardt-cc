# DI Patterns Reference

## Auto-Injectable Types

Any type that implements `Default + Clone + Send + Sync + 'static` is automatically injectable without any additional configuration. Reinhardt DI creates instances using `Default::default()`.

```rust
#[derive(Debug, Clone, Default)]
pub struct AppConfig {
    pub debug: bool,
    pub max_retries: u32,
}

// AppConfig is auto-injectable because it implements Default + Clone + Send + Sync + 'static.
// No registration needed.

#[get("/config")]
#[inject]
pub async fn show_config(
    request: Request,
    config: Inject<AppConfig>,
) -> Response {
    Response::json(json!({ "debug": config.debug }))
}
```

## Custom Injectable with `#[async_trait] impl Injectable`

For types that need custom construction logic (database lookups, external service clients, etc.), implement the `Injectable` trait manually.

```rust
use reinhardt::di::prelude::*;
use async_trait::async_trait;

pub struct EmailService {
    api_key: String,
    sender: String,
}

#[async_trait]
impl Injectable for EmailService {
    async fn resolve(ctx: &InjectionContext) -> Result<Self, InjectionError> {
        let config = ctx.resolve::<AppConfig>().await?;
        Ok(Self {
            api_key: std::env::var("EMAIL_API_KEY")
                .map_err(|_| InjectionError::missing("EMAIL_API_KEY env var"))?,
            sender: config.default_sender.clone(),
        })
    }
}
```

### InjectionContext Methods

| Method | Description |
|--------|-------------|
| `ctx.resolve::<T>().await` | Resolve another injectable type from the context |
| `ctx.resolve_optional::<T>().await` | Resolve a type, returning `None` if not registered |
| `ctx.get_request()` | Access the current HTTP request (request-scoped only) |

## Using `#[inject]` in Handlers

Apply `#[inject]` to view functions to receive dependencies as parameters.

```rust
use reinhardt::di::prelude::*;
use reinhardt::views::prelude::*;

#[get("/users")]
#[inject]
pub async fn list_users(
    request: Request,
    user_service: Inject<Arc<UserService>>,
) -> Response {
    let users = user_service.list_active().await?;
    Response::json(users)
}

#[post("/users")]
#[inject]
pub async fn create_user(
    request: Request,
    user_service: Inject<Arc<UserService>>,
    email_service: Inject<Arc<EmailService>>,
) -> Response {
    let data = request.json().await?;
    let user = user_service.create(data).await?;
    email_service.send_welcome(&user).await?;
    Response::created(user)
}
```

### Multiple Injections

Handlers can receive multiple injected dependencies. Each is resolved independently.

```rust
#[get("/dashboard")]
#[inject]
pub async fn dashboard(
    request: Request,
    auth: AuthUser,
    user_service: Inject<Arc<UserService>>,
    analytics: Inject<Arc<AnalyticsService>>,
    config: Inject<AppConfig>,
) -> Response {
    let stats = analytics.get_user_stats(auth.id()).await?;
    Response::json(stats)
}
```

## Using `#[inject]` in Server Functions

Server functions (`#[server_fn]`) also support dependency injection via `use_context`.

```rust
use reinhardt::pages::prelude::*;

#[server_fn]
pub async fn get_dashboard_data(user_id: i64) -> Result<DashboardData, ServerFnError> {
    let user_service = use_context::<Arc<UserService>>()?;
    let analytics = use_context::<Arc<AnalyticsService>>()?;

    let user = user_service.get(user_id).await?;
    let stats = analytics.get_stats(user_id).await?;

    Ok(DashboardData { user, stats })
}
```

## Scoping

Reinhardt DI supports two scopes. The resolver checks request scope first, then falls back to singleton scope.

| Scope | Lifetime | Registration | Use Case |
|-------|----------|--------------|----------|
| Request-scoped | One per HTTP request | `app.inject_request::<T>()` | Per-request state, database transactions, auth context |
| Singleton | One for app lifetime | `app.inject_singleton::<T>(instance)` | Shared services, connection pools, configuration |

### Request-Scoped Registration

```rust
use reinhardt::prelude::*;

let app = Reinhardt::new()
    .inject_request::<RequestLogger>()
    .inject_request::<TransactionContext>();
```

A new instance is created for each request and dropped when the request completes.

### Singleton Registration

```rust
let email_service = Arc::new(EmailService::new("api-key"));
let cache = Arc::new(CacheService::connect("redis://localhost").await);

let app = Reinhardt::new()
    .inject_singleton::<Arc<EmailService>>(email_service)
    .inject_singleton::<Arc<CacheService>>(cache);
```

Singleton instances are shared across all requests. Wrap in `Arc<T>` for thread-safe sharing.

## `Arc<T>` Wrapping

Services registered as singletons should always be wrapped in `Arc<T>`:

```rust
// Registration
let service = Arc::new(UserService::new(pool.clone()));
app.inject_singleton::<Arc<UserService>>(service);

// Injection in handler
#[get("/users")]
#[inject]
pub async fn list_users(
    request: Request,
    user_service: Inject<Arc<UserService>>,
) -> Response {
    // user_service is Arc<UserService> — clone is cheap
    let users = user_service.list_all().await?;
    Response::json(users)
}
```

**Why `Arc<T>`:**
- Singleton services are shared across threads
- `Arc` provides thread-safe reference counting
- Cloning `Arc` is cheap (atomic increment)

## Avoiding Circular Dependencies

Circular dependencies are detected at runtime and will panic. Prevent them by design.

### BAD — Circular

```rust
// UserService depends on OrderService
impl Injectable for UserService {
    async fn resolve(ctx: &InjectionContext) -> Result<Self, InjectionError> {
        let orders = ctx.resolve::<Arc<OrderService>>().await?;  // depends on OrderService
        Ok(Self { orders })
    }
}

// OrderService depends on UserService — CIRCULAR!
impl Injectable for OrderService {
    async fn resolve(ctx: &InjectionContext) -> Result<Self, InjectionError> {
        let users = ctx.resolve::<Arc<UserService>>().await?;  // depends on UserService
        Ok(Self { users })
    }
}
```

### GOOD — Break the Cycle

Extract shared logic into a third service, or use events/traits to decouple:

```rust
// Both depend on UserRepository (no cycle)
impl Injectable for UserService {
    async fn resolve(ctx: &InjectionContext) -> Result<Self, InjectionError> {
        let repo = ctx.resolve::<Arc<UserRepository>>().await?;
        Ok(Self { repo })
    }
}

impl Injectable for OrderService {
    async fn resolve(ctx: &InjectionContext) -> Result<Self, InjectionError> {
        let repo = ctx.resolve::<Arc<UserRepository>>().await?;
        Ok(Self { user_repo: repo })
    }
}
```

## Resolution Order

When resolving a type `T`:

1. Check request-scoped registrations
2. Check singleton registrations
3. Check auto-injectable (if `T: Default + Clone + Send + Sync + 'static`)
4. Return `InjectionError::not_found::<T>()` if none matched
