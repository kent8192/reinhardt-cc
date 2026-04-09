# Reinhardt Social Authentication (OAuth2/OIDC) Reference

**Feature:** `social`

**Module:** `reinhardt_auth::social` (re-exported via `reinhardt::auth::social`)

---

## Supported Providers

| Provider | Type | Protocol | Feature |
|----------|------|----------|---------|
| `GoogleProvider` | OIDC | OpenID Connect | `social` |
| `GitHubProvider` | OAuth2 | OAuth 2.0 | `social` |
| `AppleProvider` | OIDC | OpenID Connect (JWT client_secret) | `social` |
| `MicrosoftProvider` | OIDC | OpenID Connect / Azure AD | `social` |

---

## Architecture

```
User → Authorization URL → Provider → Callback URL → Token Exchange → User Mapping
         (with PKCE)                     (state validation)   (ID token verification)
```

### Module Structure

| Module | Purpose |
|--------|---------|
| `backend` | `SocialAuthBackend` — main entry point |
| `core` | `OAuth2Config`, `OIDCConfig`, `ProviderConfig`, `OAuthToken`, `IdToken`, `StandardClaims` |
| `flow` | `AuthorizationFlow`, `PkceFlow`, `RefreshFlow`, `TokenExchangeFlow`, `StateStore` |
| `oidc` | `DiscoveryClient`, `IdTokenValidator`, `JwkSet`, `JwksCache`, `OIDCDiscovery`, `UserInfoClient` |
| `providers` | Provider implementations |
| `storage` | `SocialAccountStorage`, `InMemorySocialAccountStorage` |
| `user_mapping` | `UserMapper`, `DefaultUserMapper`, `MappedUser` |

---

## Setup

### Feature Flag

```toml
[dependencies]
reinhardt = { version = "...", features = ["social", "argon2-hasher"] }
```

### Provider Configuration

```rust
use reinhardt::auth::social::*;

// Google OIDC
let google = ProviderConfig::google(
    "your-client-id.apps.googleusercontent.com",
    "your-client-secret",
    "https://yourapp.com/auth/google/callback",
);

// GitHub OAuth2
let github = ProviderConfig::github(
    "your-github-client-id",
    "your-github-client-secret",
    "https://yourapp.com/auth/github/callback",
);

// Apple OIDC (requires team_id, key_id, private_key for JWT client_secret)
let apple = ProviderConfig::apple(
    "your-apple-client-id",
    "your-team-id",
    "your-key-id",
    include_str!("../keys/AuthKey.p8"),
    "https://yourapp.com/auth/apple/callback",
);

// Microsoft OIDC / Azure AD
let microsoft = ProviderConfig::microsoft(
    "your-azure-client-id",
    "your-azure-client-secret",
    "https://yourapp.com/auth/microsoft/callback",
);
```

### Backend Registration

```rust
use reinhardt::auth::social::SocialAuthBackend;
use reinhardt::di::prelude::*;

#[injectable_factory(scope = "singleton")]
async fn social_auth(#[inject] settings: Depends<ProjectSettings>) -> SocialAuthBackend {
    SocialAuthBackend::new()
        .with_provider(ProviderConfig::google(
            &settings.google_client_id,
            &settings.google_client_secret,
            &settings.google_callback_url,
        ))
        .with_provider(ProviderConfig::github(
            &settings.github_client_id,
            &settings.github_client_secret,
            &settings.github_callback_url,
        ))
}
```

---

## OAuth2 Flow

### Step 1: Authorization URL

```rust
use reinhardt::auth::social::flow::{AuthorizationFlow, PkceFlow};

// Generate authorization URL with PKCE
let flow = AuthorizationFlow::new(&provider_config);
let (auth_url, state, pkce_verifier) = flow.authorization_url_with_pkce(
    &["openid", "email", "profile"], // scopes
)?;

// Store state and PKCE verifier in session/cache for callback validation
state_store.store(state.clone(), StateData {
    pkce_verifier,
    provider: "google".to_string(),
}).await?;

// Redirect user to auth_url
```

### Step 2: Callback Handling

```rust
// In callback handler
let callback = CallbackResult {
    code: query.code,
    state: query.state,
};

// Validate state parameter (CSRF protection)
let state_data = state_store.get(&callback.state).await?
    .ok_or(SocialAuthError::InvalidState)?;

// Exchange code for tokens
let token_exchange = TokenExchangeFlow::new(&provider_config);
let token_response = token_exchange.exchange_code(
    &callback.code,
    Some(&state_data.pkce_verifier),
).await?;
```

### Step 3: User Mapping

```rust
// Verify ID token (for OIDC providers)
let validator = IdTokenValidator::new(&provider_config);
let claims = validator.validate(&token_response.id_token).await?;

// Map claims to user
let mapper = DefaultUserMapper::new();
let mapped_user = mapper.map_user(&claims).await?;

// Find or create user in database
let user = social_account_storage
    .find_or_create_user(&mapped_user, "google")
    .await?;
```

---

## Core Types

### StandardClaims

OpenID Connect standard claims extracted from ID tokens:

```rust
pub struct StandardClaims {
    pub sub: String,           // Subject (unique provider ID)
    pub email: Option<String>,
    pub email_verified: Option<bool>,
    pub name: Option<String>,
    pub given_name: Option<String>,
    pub family_name: Option<String>,
    pub picture: Option<String>,
    pub locale: Option<String>,
}
```

### OAuthToken / TokenResponse

```rust
pub struct TokenResponse {
    pub access_token: String,
    pub token_type: String,
    pub expires_in: Option<u64>,
    pub refresh_token: Option<String>,
    pub scope: Option<String>,
    pub id_token: Option<String>,  // OIDC only
}
```

### SocialAccount

Maps OAuth identity to local user:

```rust
pub struct SocialAccount {
    pub provider: String,       // "google", "github", etc.
    pub provider_id: String,    // Provider-specific user ID
    pub user_id: String,        // Local user ID
    pub extra_data: serde_json::Value,
}
```

### SocialAccountStorage Trait

```rust
#[async_trait]
pub trait SocialAccountStorage: Send + Sync {
    async fn find_by_provider(&self, provider: &str, provider_id: &str)
        -> Result<Option<SocialAccount>, Error>;
    async fn create(&mut self, account: SocialAccount) -> Result<(), Error>;
    async fn delete(&mut self, provider: &str, provider_id: &str) -> Result<(), Error>;
}
```

### UserMapper Trait

```rust
#[async_trait]
pub trait UserMapper: Send + Sync {
    async fn map_user(&self, claims: &StandardClaims) -> Result<MappedUser, Error>;
}
```

---

## Security Features

| Feature | Description |
|---------|-------------|
| **PKCE** (RFC 7636) | Proof Key for Code Exchange — prevents authorization code interception |
| **State Parameter** | CSRF protection — validates callback matches initiated flow |
| **ID Token Verification** | JWKS signature verification for OIDC providers |
| **Nonce Validation** | Prevents replay attacks in OIDC flows |
| **JWKS Caching** | Caches provider public keys to avoid repeated fetches |

---

## Error Types

```rust
pub enum SocialAuthError {
    InvalidState,
    TokenExchangeFailed(String),
    IdTokenValidationFailed(String),
    ProviderNotConfigured(String),
    UserMappingFailed(String),
    StorageError(String),
    NetworkError(String),
}
```

## Dynamic References

For the latest social auth API:
1. Read `reinhardt/crates/reinhardt-auth/src/social.rs` for module structure
2. Read `reinhardt/crates/reinhardt-auth/src/social/backend.rs` for SocialAuthBackend
3. Read `reinhardt/crates/reinhardt-auth/src/social/providers/` for provider implementations
4. Read `reinhardt/crates/reinhardt-auth/src/social/flow.rs` for OAuth2 flows
5. Read `reinhardt/crates/reinhardt-auth/src/social/oidc.rs` for OIDC validation
