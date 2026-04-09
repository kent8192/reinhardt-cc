# rstest Patterns Reference

## Basic Test Structure

Every test uses `#[rstest]` instead of `#[test]`, combined with the Arrange-Act-Assert (AAA) pattern.

```rust
use rstest::*;
use reinhardt::db::prelude::*;

#[rstest]
fn test_user_creation() {
    // Arrange
    let username = "testuser";
    let email = "test@example.com";

    // Act
    let user = User::new(username, email);

    // Assert
    assert_eq!(user.username(), username);
    assert_eq!(user.email(), email);
    assert!(user.is_active());
}
```

## AAA Label Rules

| Label | Status | Usage |
|-------|--------|-------|
| `// Arrange` | STANDARD | Set up test preconditions and inputs |
| `// Act` | STANDARD | Execute the behavior under test |
| `// Assert` | STANDARD | Verify the expected outcomes |
| `// Setup` | FORBIDDEN | Never use — use `// Arrange` |
| `// Execute` | FORBIDDEN | Never use — use `// Act` |
| `// Verify` | FORBIDDEN | Never use — use `// Assert` |
| `// Given` / `// When` / `// Then` | FORBIDDEN | BDD-style labels are not allowed |

**Omission rule:** AAA comments MAY be omitted when the test body is 5 lines or fewer.

```rust
// Short test — AAA comments omitted
#[rstest]
fn test_default_is_active() {
    let user = User::default();
    assert!(user.is_active());
}
```

## Async Tests

Combine `#[rstest]` with `#[tokio::test]` for async tests:

```rust
use rstest::*;
use reinhardt::db::prelude::*;

#[rstest]
#[tokio::test]
async fn test_user_query() {
    // Arrange
    let pool = test_pool().await;
    let user = create_test_user(&pool).await;

    // Act
    let found = User::objects()
        .filter(User::id.eq(user.id))
        .get(&pool)
        .await
        .unwrap();

    // Assert
    assert_eq!(found.username(), user.username());
}
```

## Fixtures with `#[fixture]`

### Sync Fixtures

```rust
use rstest::*;

#[fixture]
fn sample_config() -> AppConfig {
    AppConfig {
        debug: true,
        database_url: "postgres://localhost/test".to_string(),
        max_connections: 5,
    }
}

#[rstest]
fn test_config_debug_mode(sample_config: AppConfig) {
    assert!(sample_config.debug);
}
```

### Async Fixtures

```rust
use rstest::*;

#[fixture]
async fn test_pool() -> TestPool {
    TestPool::setup().await
}

#[fixture]
async fn seeded_pool(#[future] test_pool: TestPool) -> TestPool {
    let pool = test_pool.await;
    seed_test_data(&pool).await;
    pool
}

#[rstest]
#[tokio::test]
async fn test_with_seeded_data(#[future] seeded_pool: TestPool) {
    // Arrange
    let pool = seeded_pool.await;

    // Act
    let count = User::objects().count(&pool).await.unwrap();

    // Assert
    assert!(count > 0);
}
```

### Fixture with Parameters

```rust
#[fixture]
fn user_with_role(#[default("viewer")] role: &str) -> User {
    User::new_with_role("testuser", role)
}

#[rstest]
fn test_default_role(user_with_role: User) {
    assert_eq!(user_with_role.role(), "viewer");
}

#[rstest]
fn test_admin_role(#[with("admin")] user_with_role: User) {
    assert_eq!(user_with_role.role(), "admin");
}
```

## Parameterized Tests with `#[case]`

```rust
use rstest::*;

#[rstest]
#[case("admin", true)]
#[case("editor", true)]
#[case("viewer", false)]
#[case("guest", false)]
fn test_can_edit_permission(#[case] role: &str, #[case] expected: bool) {
    // Arrange
    let user = User::new_with_role("testuser", role);

    // Act
    let can_edit = user.can_edit();

    // Assert
    assert_eq!(can_edit, expected);
}
```

### Parameterized Async Tests

```rust
#[rstest]
#[case(StatusCode::OK, "/api/users")]
#[case(StatusCode::NOT_FOUND, "/api/nonexistent")]
#[case(StatusCode::METHOD_NOT_ALLOWED, "/api/readonly")]
#[tokio::test]
async fn test_endpoint_status(#[case] expected: StatusCode, #[case] path: &str) {
    // Arrange
    let client = APIClient::new();

    // Act
    let response = client.get(path).await.unwrap();

    // Assert
    assert_eq!(response.status(), expected);
}
```

## Serial Tests with `#[serial]`

Use `#[serial(group_name)]` from the `serial_test` crate when tests share global state.

```rust
use rstest::*;
use serial_test::serial;

#[rstest]
#[serial(i18n)]
fn test_locale_en() {
    // Arrange
    set_global_locale("en");

    // Act
    let greeting = translate("hello");

    // Assert
    assert_eq!(greeting, "Hello");
}

#[rstest]
#[serial(i18n)]
fn test_locale_ja() {
    // Arrange
    set_global_locale("ja");

    // Act
    let greeting = translate("hello");

    // Assert
    assert_eq!(greeting, "こんにちは");
}

#[rstest]
#[serial(registry)]
#[tokio::test]
async fn test_plugin_registration() {
    // Arrange
    clear_global_registry();

    // Act
    register_plugin("auth").await;

    // Assert
    assert!(is_registered("auth"));
}
```

**Group naming:** Use descriptive names matching the shared resource: `#[serial(db)]`, `#[serial(i18n)]`, `#[serial(registry)]`, `#[serial(config)]`.

## Assertion Best Practices

### GOOD — Exact Verification

```rust
// Exact value comparison
assert_eq!(user.username(), "testuser");
assert_ne!(user.id(), 0);

// Pattern matching for enums
assert!(matches!(result, Ok(User { .. })));
assert!(matches!(err, Err(QueryError::NotFound { .. })));

// Unwrap and check the value
let user = result.unwrap();
assert_eq!(user.email(), "test@example.com");
```

### BAD — Loose Verification

```rust
// BAD: Does not check the actual value
assert!(result.is_ok());

// BAD: Fragile substring check
assert!(error_msg.contains("not found"));

// BAD: Only checks existence, not correctness
assert!(users.len() > 0);
```

### Exception — Loose Assertions with Justification

```rust
// NOTE: Exact ID comparison not possible because IDs are auto-generated
assert!(user.id() > 0);

// NOTE: Timestamp depends on system clock; verify within 1-second window
let now = Utc::now();
assert!(user.created_at() <= now);
assert!(user.created_at() >= now - chrono::Duration::seconds(1));
```

## Test Module Organization

```rust
// src/apps/polls/models.rs

#[cfg(test)]
mod tests {
    use super::*;
    use rstest::*;
    use reinhardt_test::prelude::*;

    #[fixture]
    fn sample_question() -> Question {
        Question {
            id: 1,
            question_text: "What is Rust?".to_string(),
            pub_date: Utc::now(),
        }
    }

    #[rstest]
    fn test_question_text(sample_question: Question) {
        assert_eq!(sample_question.question_text, "What is Rust?");
    }
}
```

## Running Tests

```bash
# Run all tests with cargo-nextest (preferred)
cargo nextest run --workspace --all-features

# Run tests in a specific crate
cargo nextest run -p reinhardt-core --all-features

# Run a specific test by name
cargo nextest run --workspace --all-features test_user_creation

# Run doc tests (not supported by nextest)
cargo test --doc

# Run tests with output displayed
cargo nextest run --workspace --all-features --nocapture
```
