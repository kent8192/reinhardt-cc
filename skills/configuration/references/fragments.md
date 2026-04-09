# Settings Fragments

Fragments are independent configuration sections that compose into `ProjectSettings`. Each fragment owns a TOML section and can be validated independently.

## Built-in Fragments

| Fragment | TOML Section | Description |
|----------|-------------|-------------|
| `CoreSettings` | `[core]` | Base dir, secret key, debug, hosts, databases, security |
| `I18nSettings` | `[i18n]` | Language, timezone, locale |
| `StaticSettings` | `[static_files]` | Static file serving |
| `MediaSettings` | `[media]` | User-uploaded file storage |
| `CacheSettings` | `[cache]` | Caching configuration |
| `EmailSettings` | `[email]` | Email backend configuration |
| `LoggingSettings` | `[logging]` | Logging configuration |
| `CorsSettings` | `[cors]` | CORS configuration |
| `SecuritySettings` | `[core.security]` | Security (nested under core) |

## Creating Custom Fragments

Use the `#[settings]` macro on a struct to implement the `SettingsFragment` trait:

```rust
use reinhardt::conf::settings::fragment::SettingsFragment;
use reinhardt::settings;
use serde::{Deserialize, Serialize};

#[settings]
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct MyAppSettings {
    pub api_key: String,
    pub max_retries: u32,
    #[serde(default = "default_timeout")]
    pub timeout_secs: u64,
}

fn default_timeout() -> u64 { 30 }
```

### Registering in ProjectSettings

```rust
// In your ProjectSettings:
#[settings(core: CoreSettings | myapp: MyAppSettings)]
pub struct ProjectSettings;
```

### Corresponding TOML

```toml
[myapp]
api_key = "sk-..."
max_retries = 3
timeout_secs = 60
```

### Accessing the Fragment

```rust
#[get("/status/", name = "status")]
pub async fn status(
    #[inject] settings: Inject<Arc<ProjectSettings>>,
) -> ViewResult<Response> {
    let api_key = &settings.myapp.api_key;
    let timeout = settings.myapp.timeout_secs;
    // ...
}
```

## Validation

Fragments can implement `SettingsValidation` for custom validation logic. Validation runs automatically during `build_composed()`.

```rust
impl SettingsValidation for MyAppSettings {
    fn validate(&self) -> ValidationResult {
        if self.max_retries == 0 {
            return Err(ValidationError::new("max_retries must be > 0"));
        }
        Ok(())
    }
}
```

### Common Validation Patterns

```rust
impl SettingsValidation for MyAppSettings {
    fn validate(&self) -> ValidationResult {
        // Required field check
        if self.api_key.is_empty() {
            return Err(ValidationError::new("api_key must not be empty"));
        }

        // Range check
        if self.timeout_secs > 300 {
            return Err(ValidationError::new("timeout_secs must be <= 300"));
        }

        Ok(())
    }
}
```

## Profile-Specific Validation

Fragments can validate differently based on the environment profile. For example, requiring a real secret key in production but allowing a placeholder in development:

```rust
impl SettingsValidation for CoreSettings {
    fn validate_with_profile(&self, profile: &Profile) -> ValidationResult {
        match profile {
            Profile::Production => {
                if self.secret_key.starts_with("dev-") {
                    return Err(ValidationError::new(
                        "production secret_key must not use dev- prefix"
                    ));
                }
                if self.debug {
                    return Err(ValidationError::new(
                        "debug must be false in production"
                    ));
                }
            }
            _ => {}
        }
        Ok(())
    }
}
```

## Fragment Composition Patterns

### Minimal (core only)

```rust
#[settings(core: CoreSettings)]
pub struct ProjectSettings;
```

### Standard web app

```rust
#[settings(
    core: CoreSettings
    | I18nSettings
    | static_files: StaticSettings
    | MediaSettings
    | CorsSettings
)]
pub struct ProjectSettings;
```

### Full-featured with custom fragments

```rust
#[settings(
    core: CoreSettings
    | I18nSettings
    | static_files: StaticSettings
    | MediaSettings
    | CacheSettings
    | EmailSettings
    | LoggingSettings
    | CorsSettings
    | myapp: MyAppSettings
    | billing: BillingSettings
)]
pub struct ProjectSettings;
```

## Fragment Design Guidelines

- Keep fragments focused on a single concern (e.g., email, caching, billing)
- Use `#[serde(default)]` for optional fields with sensible defaults
- Implement `SettingsValidation` for any fragment with constraints
- Use nested structs for sub-sections (e.g., `SecuritySettings` under `CoreSettings`)
- Prefer typed enums over string fields for fixed choices (e.g., database engine)
