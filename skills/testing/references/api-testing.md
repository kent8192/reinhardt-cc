# API Testing Reference

## APIClient Usage

The `APIClient` from `reinhardt_test` provides a test HTTP client for making requests against reinhardt views.

```rust
use reinhardt_test::prelude::*;
use rstest::*;

#[rstest]
#[tokio::test]
async fn test_list_users() {
    // Arrange
    let client = APIClient::new();

    // Act
    let response = client.get("/api/users").await.unwrap();

    // Assert
    assert_eq!(response.status(), StatusCode::OK);
    let body: Vec<UserResponse> = response.json().await.unwrap();
    assert!(!body.is_empty());
}
```

### APIClient Methods

| Method | Description |
|--------|-------------|
| `APIClient::new()` | Create a new test client with default settings |
| `client.get(path)` | Send a GET request |
| `client.post(path)` | Send a POST request (use `.json(&body)` to set body) |
| `client.put(path)` | Send a PUT request |
| `client.patch(path)` | Send a PATCH request |
| `client.delete(path)` | Send a DELETE request |
| `client.set_header(name, value)` | Set a request header for subsequent requests |

### Setting Headers

```rust
let mut client = APIClient::new();
client.set_header("Accept-Language", "en");
client.set_header("X-Request-ID", "test-123");

let response = client.get("/api/users").await.unwrap();
```

### POST with JSON Body

```rust
use serde_json::json;

#[rstest]
#[tokio::test]
async fn test_create_user() {
    // Arrange
    let client = APIClient::new();
    let body = json!({
        "username": "newuser",
        "email": "new@example.com"
    });

    // Act
    let response = client.post("/api/users").json(&body).await.unwrap();

    // Assert
    assert_eq!(response.status(), StatusCode::CREATED);
    let user: UserResponse = response.json().await.unwrap();
    assert_eq!(user.username, "newuser");
}
```

## Authenticated Requests

Use Bearer tokens for authenticated endpoints:

```rust
#[rstest]
#[tokio::test]
async fn test_authenticated_endpoint() {
    // Arrange
    let mut client = APIClient::new();
    let token = create_test_token("testuser").await;
    client.set_header("Authorization", &format!("Bearer {}", token));

    // Act
    let response = client.get("/api/profile").await.unwrap();

    // Assert
    assert_eq!(response.status(), StatusCode::OK);
    let profile: ProfileResponse = response.json().await.unwrap();
    assert_eq!(profile.username, "testuser");
}

#[rstest]
#[tokio::test]
async fn test_unauthorized_without_token() {
    // Arrange
    let client = APIClient::new();

    // Act
    let response = client.get("/api/profile").await.unwrap();

    // Assert
    assert_eq!(response.status(), StatusCode::UNAUTHORIZED);
}
```

## APIRequestFactory

`APIRequestFactory` creates raw `Request` objects for testing views directly without HTTP transport.

```rust
use reinhardt_test::prelude::*;
use rstest::*;

#[rstest]
#[tokio::test]
async fn test_view_directly() {
    // Arrange
    let factory = APIRequestFactory::new();
    let request = factory
        .get("/api/users/1")
        .with_user(test_user())
        .build();

    // Act
    let response = get_user(request, Path(1)).await;

    // Assert
    assert_eq!(response.status(), StatusCode::OK);
}
```

### Factory Methods

| Method | Description |
|--------|-------------|
| `APIRequestFactory::new()` | Create a new request factory |
| `factory.get(path)` | Build a GET request |
| `factory.post(path)` | Build a POST request |
| `factory.put(path)` | Build a PUT request |
| `factory.delete(path)` | Build a DELETE request |
| `.with_user(user)` | Attach an authenticated user to the request |
| `.with_json(&body)` | Set JSON request body |
| `.with_header(name, value)` | Add a request header |
| `.build()` | Finalize and return the `Request` |

## Testing Response Body

```rust
#[rstest]
#[tokio::test]
async fn test_response_parsing() {
    // Arrange
    let client = APIClient::new();

    // Act
    let response = client.get("/api/users/1").await.unwrap();

    // Assert — status
    assert_eq!(response.status(), StatusCode::OK);

    // Assert — JSON body
    let user: UserResponse = response.json().await.unwrap();
    assert_eq!(user.id, 1);
    assert_eq!(user.username, "testuser");
    assert!(user.is_active);

    // Assert — headers
    assert_eq!(
        response.headers().get("content-type").unwrap(),
        "application/json"
    );
}
```

## `impl_test_model!` Macro

The `impl_test_model!` macro generates test helper methods for model types, providing convenient factory methods for creating test instances.

```rust
use reinhardt_test::impl_test_model;

impl_test_model!(User, {
    id: i64 = 1,
    username: String = "testuser".to_string(),
    email: String = "test@example.com".to_string(),
    is_active: bool = true,
});
```

This generates:
- `User::test_default()` — create an instance with all default values
- `User::test_with(overrides)` — create an instance with specific field overrides

Usage:

```rust
#[rstest]
fn test_with_default_model() {
    let user = User::test_default();
    assert_eq!(user.username, "testuser");
}

#[rstest]
fn test_with_override() {
    let user = User::test_with(|u| {
        u.username = "custom".to_string();
        u.is_active = false;
    });
    assert_eq!(user.username, "custom");
    assert!(!user.is_active);
}
```

## Factory Pattern for Test Data

For complex test data setup, use dedicated factory functions or fixtures:

```rust
use rstest::*;

#[fixture]
fn user_factory() -> UserFactory {
    UserFactory::new()
}

struct UserFactory {
    counter: std::cell::Cell<i64>,
}

impl UserFactory {
    fn new() -> Self {
        Self { counter: std::cell::Cell::new(0) }
    }

    fn create(&self) -> User {
        let n = self.counter.get();
        self.counter.set(n + 1);
        User {
            id: n + 1,
            username: format!("user_{}", n),
            email: format!("user_{}@example.com", n),
            is_active: true,
        }
    }

    fn create_inactive(&self) -> User {
        let mut user = self.create();
        user.is_active = false;
        user
    }
}

#[rstest]
fn test_with_factory(user_factory: UserFactory) {
    // Arrange
    let active_user = user_factory.create();
    let inactive_user = user_factory.create_inactive();

    // Assert
    assert!(active_user.is_active);
    assert!(!inactive_user.is_active);
    assert_ne!(active_user.id, inactive_user.id);
}
```

## Test Placement Rules

| Test Type | Location | When to Use |
|-----------|----------|-------------|
| Unit tests | `#[cfg(test)] mod tests` in the functional crate | Testing individual functions, methods, or types in isolation |
| Integration tests (within-crate) | `#[cfg(test)] mod tests` in the functional crate | Testing interactions between modules within the same crate |
| Integration tests (cross-crate) | `tests/` crate at workspace root | Testing interactions between multiple reinhardt crates |
| E2E / API tests | `tests/` crate at workspace root | Testing full request-response cycles through the HTTP layer |

**Key rules:**
- Unit tests go in the same file as the code they test
- Cross-crate integration tests go in the workspace `tests/` crate
- Every test MUST use at least one reinhardt component
- Functional crates MUST NOT use `{ workspace = true }` for `reinhardt-test` in `[dev-dependencies]`
