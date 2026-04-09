# Reinhardt Authentication Configuration Reference

## Auth Backends

Reinhardt supports multiple authentication backends, each enabled via feature flags.

| Backend | Feature Flag | Transport | Stateful | Use Case |
|---------|-------------|-----------|----------|----------|
| JWT | `auth-jwt` | `Authorization: Bearer <token>` | No | APIs, mobile clients, SPAs |
| Session | `auth-session` | Cookie (`sessionid`) | Yes | Traditional web apps, admin panel |
| OAuth 2.0 | `auth-oauth` | `Authorization: Bearer <token>` | Depends | Third-party login (Google, GitHub, etc.) |
| Token | `auth-token` | `Authorization: Token <key>` | Yes (DB) | Persistent API keys, service accounts |
| Basic | (always available) | `Authorization: Basic <b64>` | No | Development, simple integrations |

## JWT Authentication Setup

### Feature Flag

```toml
[dependencies]
reinhardt = { version = "0.1.0-alpha", features = ["auth-jwt", "argon2-hasher"] }
```

### Configuration

```rust
// src/settings.rs
use reinhardt::auth::jwt::{JwtConfig, Algorithm};

pub fn jwt_config() -> JwtConfig {
    JwtConfig {
        secret_key: std::env::var("JWT_SECRET_KEY")
            .expect("JWT_SECRET_KEY must be set"),
        algorithm: Algorithm::HS256,
        access_token_lifetime: Duration::from_secs(60 * 15),   // 15 minutes
        refresh_token_lifetime: Duration::from_secs(60 * 60 * 24 * 7), // 7 days
        issuer: Some("my-app".to_string()),
        audience: None,
    }
}
```

### Usage in Views

```rust
use reinhardt::auth::prelude::*;
use reinhardt::views::prelude::*;

/// Login endpoint: validates credentials and returns JWT tokens
pub async fn login(request: Request) -> Response {
    let data = request.json().await?;
    let username = data["username"].as_str().unwrap_or_default();
    let password = data["password"].as_str().unwrap_or_default();

    let user = authenticate(username, password).await
        .map_err(|_| HttpError::unauthorized("Invalid credentials"))?;

    let tokens = JwtToken::create_pair(&user)?;
    Response::json(serde_json::json!({
        "access": tokens.access,
        "refresh": tokens.refresh,
    }))
}

/// Token refresh endpoint
pub async fn refresh_token(request: Request) -> Response {
    let data = request.json().await?;
    let refresh = data["refresh"].as_str().unwrap_or_default();

    let tokens = JwtToken::refresh(refresh)?;
    Response::json(serde_json::json!({
        "access": tokens.access,
    }))
}

/// Protected endpoint: requires valid JWT
#[inject]
pub async fn get_profile(request: Request, auth: AuthUser) -> Response {
    let user = auth.user();
    let serializer = UserSerializer::build(user);
    Response::json(serializer.serialize())
}
```

## Session Authentication Setup

### Feature Flag

```toml
[dependencies]
reinhardt = { version = "0.1.0-alpha", features = ["auth-session", "sessions", "argon2-hasher"] }
```

### Configuration

```rust
// src/settings.rs
use reinhardt::sessions::{SessionConfig, SessionEngine};

pub fn session_config() -> SessionConfig {
    SessionConfig {
        engine: SessionEngine::Database, // or Redis, Cookie
        cookie_name: "sessionid".to_string(),
        cookie_age: Duration::from_secs(60 * 60 * 24 * 14), // 2 weeks
        cookie_secure: true,    // HTTPS only in production
        cookie_httponly: true,   // Not accessible via JavaScript
        cookie_samesite: SameSite::Lax,
    }
}
```

### Session Backends

| Backend | Feature Flag | Storage | Performance | Use Case |
|---------|-------------|---------|-------------|----------|
| Database | `sessions` + `db-*` | Database table | Moderate | Default, no extra infrastructure |
| Redis | `redis-backend` | Redis server | Fast | High-traffic applications |
| Cookie | `sessions` | Signed cookie | Fastest | Small session data, no server state |

### Usage in Views

```rust
use reinhardt::auth::prelude::*;
use reinhardt::views::prelude::*;

pub async fn session_login(request: Request) -> Response {
    let data = request.json().await?;
    let username = data["username"].as_str().unwrap_or_default();
    let password = data["password"].as_str().unwrap_or_default();

    let user = authenticate(username, password).await
        .map_err(|_| HttpError::unauthorized("Invalid credentials"))?;

    // Creates a session and sets the session cookie
    login(&request, &user).await?;

    Response::json(serde_json::json!({ "status": "ok" }))
}

pub async fn session_logout(request: Request) -> Response {
    logout(&request).await?;
    Response::json(serde_json::json!({ "status": "ok" }))
}
```

## Password Hashing

Reinhardt uses pluggable password hashers. The `argon2-hasher` feature is recommended for production.

| Hasher | Feature Flag | Security Level | Speed |
|--------|-------------|----------------|-------|
| Argon2id | `argon2-hasher` | Highest (recommended) | Slow (by design) |
| PBKDF2-SHA256 | (default) | High | Moderate |
| Bcrypt | (built-in) | High | Moderate |

```rust
use reinhardt::auth::hashers::make_password;

// Hash a password
let hashed = make_password("user_password")?;

// Verify a password
let is_valid = check_password("user_password", &hashed)?;
```

## Permission Classes

Permission classes control access to views. Apply them via `get_permissions()` on ViewSets or as middleware.

| Permission | Description |
|-----------|-------------|
| `AllowAny` | No authentication required |
| `IsAuthenticated` | User must be authenticated |
| `IsAdminUser` | User must have `is_staff = true` |
| `IsAuthenticatedOrReadOnly` | Authenticated for write, anyone for read |
| `HasPermission(perm)` | User must have the named permission |
| `HasGroupPermission(group)` | User must belong to the named group |

### Applying Permissions

```rust
// On a ViewSet
impl ViewSet for UserViewSet {
    fn get_permissions(&self) -> Vec<Box<dyn Permission>> {
        vec![Box::new(IsAuthenticated)]
    }
}

// On a function-based view (via middleware)
#[permission(IsAdminUser)]
pub async fn admin_dashboard(request: Request, auth: AuthUser) -> Response {
    // Only staff users reach here
    Response::json(serde_json::json!({ "status": "ok" }))
}

// Per-action permissions on a ViewSet
impl ViewSet for ArticleViewSet {
    fn get_permissions_for_action(&self, action: &Action) -> Vec<Box<dyn Permission>> {
        match action {
            Action::List | Action::Retrieve => vec![Box::new(AllowAny)],
            _ => vec![Box::new(IsAuthenticated)],
        }
    }
}
```

## Security Best Practices

1. **Always use HTTPS in production** — Set `cookie_secure: true` for session cookies
2. **Use `argon2-hasher`** — It is the most resistant to brute-force attacks
3. **Set short JWT access token lifetimes** — 15 minutes is recommended; use refresh tokens for longer sessions
4. **Never store secrets in code** — Use environment variables for `JWT_SECRET_KEY`, database credentials, and OAuth client secrets
5. **Enable CORS carefully** — Only whitelist known origins, never use `*` in production
6. **Use `HttpOnly` and `SameSite` cookies** — Prevent XSS and CSRF attacks on session cookies
7. **Rate-limit auth endpoints** — Apply `RateLimitMiddleware` to login, registration, and token endpoints
8. **Rotate secrets periodically** — Plan for JWT key rotation and session secret rotation
