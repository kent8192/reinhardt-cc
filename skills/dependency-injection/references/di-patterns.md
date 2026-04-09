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

#[get("/config/", name = "show_config")]
pub async fn show_config(
    #[inject] config: Inject<AppConfig>,
) -> ViewResult<Response> {
    Ok(Response::new(StatusCode::OK)
        .with_header("Content-Type", "application/json")
        .with_body(json::to_vec(&json!({ "debug": config.debug }))?))
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

Apply `#[inject]` to handler parameters to receive dependencies. Handlers use HTTP method decorators (`#[get]`, `#[post]`, etc.) for routing.

```rust
use reinhardt::di::prelude::*;
use reinhardt::views::prelude::*;

#[get("/users/", name = "user_list")]
pub async fn list_users(
    #[inject] user_service: Inject<Arc<UserService>>,
) -> ViewResult<Response> {
    let users = user_service.list_active().await?;
    Ok(Response::new(StatusCode::OK)
        .with_header("Content-Type", "application/json")
        .with_body(json::to_vec(&users)?))
}

#[post("/users/", name = "user_create")]
pub async fn create_user(
    Json(body): Json<CreateUserRequest>,
    #[inject] user_service: Inject<Arc<UserService>>,
    #[inject] email_service: Inject<Arc<EmailService>>,
) -> ViewResult<Response> {
    let user = user_service.create(&body).await?;
    email_service.send_welcome(&user).await?;
    Ok(Response::new(StatusCode::CREATED)
        .with_header("Content-Type", "application/json")
        .with_body(json::to_vec(&UserResponse::from(user))?))
}
```

### Multiple Injections

Handlers can receive multiple injected dependencies. Each is resolved independently.

```rust
#[get("/dashboard/", name = "dashboard")]
pub async fn dashboard(
    #[inject] AuthInfo(state): AuthInfo,
    #[inject] user_service: Inject<Arc<UserService>>,
    #[inject] analytics: Inject<Arc<AnalyticsService>>,
    #[inject] config: Inject<AppConfig>,
) -> ViewResult<Response> {
    let stats = analytics.get_user_stats(state.user_id()).await?;
    Ok(Response::new(StatusCode::OK)
        .with_header("Content-Type", "application/json")
        .with_body(json::to_vec(&stats)?))
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

Reinhardt DI supports three scopes, declared via `#[injectable]` or `#[injectable_factory]` macros:

| Scope | Lifetime | Declaration | Use Case |
|-------|----------|-------------|----------|
| Singleton | One for app lifetime | `scope = Singleton` / `scope = "singleton"` | Shared services, connection pools, configuration |
| Request | One per HTTP request | `scope = Request` / `scope = "request"` | Per-request state, auth context |
| Transient | New instance each time | `scope = Transient` / `scope = "transient"` | Stateless helpers, short-lived objects |

### `#[injectable]` for Structs

Declare scope directly on the struct:

```rust
use reinhardt::di::prelude::*;

#[injectable(scope = Singleton)]
pub struct AppConfig {
    #[no_inject]
    pub database_url: String,
    #[no_inject]
    pub debug: bool,
}

#[injectable(scope = Request)]
pub struct RequestLogger {
    #[inject]
    config: AppConfig,
}
```

### `#[injectable_factory]` for Functions

Use when construction logic is complex or requires async initialization:

```rust
use reinhardt::di::prelude::*;

#[injectable_factory(scope = "singleton")]
async fn create_database(#[inject] config: Arc<AppConfig>) -> DatabaseConnection {
    DatabaseConnection::connect(&config.database_url).await.unwrap()
}

#[injectable_factory(scope = "singleton")]
async fn create_email_service(#[inject] config: Arc<AppConfig>) -> EmailService {
    EmailService::new(&config.email_api_key)
}
```

**Rules for `#[injectable_factory]`:**
- Function MUST be `async`
- Function MUST have an explicit return type
- ALL parameters MUST be marked with `#[inject]`
- Scope is specified as a string: `"singleton"`, `"request"`, `"transient"`

### Using Injected Services in Handlers

```rust
#[get("/users/", name = "user_list")]
pub async fn list_users(
    #[inject] user_service: Inject<Arc<UserService>>,
) -> ViewResult<Response> {
    let users = user_service.list_all().await?;
    Ok(Response::new(StatusCode::OK)
        .with_header("Content-Type", "application/json")
        .with_body(json::to_vec(&users)?))
}
```

## `Arc<T>` Wrapping

Singleton services are resolved as `Arc<T>`. Factory parameters and handler injection can receive either `Arc<T>` or `T` (cloned from Arc):

```rust
// Factory receives Arc<T> — no clone, just reference counting
#[injectable_factory(scope = "singleton")]
async fn create_user_service(#[inject] config: Arc<AppConfig>) -> UserService {
    UserService::new(config)
}

// Factory receives T (non-Arc) — cloned out of Arc automatically
#[injectable_factory(scope = "transient")]
async fn make_handler(#[inject] service: MyService) -> String {
    service.value
}
```

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

## Accessing DI Context: `get_di_context`

Inside `#[injectable]` or `#[injectable_factory]` execution, use the global `get_di_context` function to access the DI context:

```rust
use reinhardt::di::{get_di_context, ContextLevel};

// ContextLevel::Root — resolves from the singleton scope
let root_ctx = get_di_context(ContextLevel::Root);

// ContextLevel::Current — resolves from the request/transient scope
let current_ctx = get_di_context(ContextLevel::Current);
```

| ContextLevel | Maps to scope | Use case |
|-------------|---------------|----------|
| `Root` | `scope = "singleton"` | Access app-wide singletons |
| `Current` | `scope = "request"` / `scope = "transient"` | Access per-request or transient dependencies |

## Resolution Order

When resolving a type `T`:

1. Check request-scoped registrations
2. Check singleton registrations
3. Check auto-injectable (if `T: Default + Clone + Send + Sync + 'static`)
4. Return `InjectionError::not_found::<T>()` if none matched
