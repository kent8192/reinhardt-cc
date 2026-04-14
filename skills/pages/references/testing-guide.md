# Testing Guide for Reinhardt Pages

## 3-Layer Test Architecture

### Layer 1: Server Function Unit Tests

Direct testing of server functions without HTTP overhead.

```rust
use reinhardt_pages::testing::ServerFnTestContext;

#[tokio::test]
async fn test_get_user() {
    // Arrange
    let ctx = ServerFnTestContext::new(singleton)
        .with_database(pool)
        .build();

    // Act
    let result = get_user(1).await;

    // Assert
    assert!(result.is_ok());
    assert_eq!(result.unwrap().username, "alice");
}
```

### Layer 2: WASM Component Tests with Mocked HTTP

Test WASM components with mocked server function responses.

```rust
use reinhardt_pages::testing::{mock_server_fn, clear_mocks, assert_server_fn_called};
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_user_list_component() {
    // Arrange
    let users = vec![User { id: 1, username: "alice".into() }];
    mock_server_fn("/api/server_fn/get_users", &users);

    // Act: render component
    // ...

    // Assert
    assert_server_fn_called("/api/server_fn/get_users");
    clear_mocks();
}
```

### Layer 3: End-to-End Tests

Full integration tests with real server and WASM frontend.

```rust
use reinhardt_pages::testing::e2e;
```

## Mock Utilities

> **Deprecation Notice**: `mock_server_fn`, `mock_server_fn_error`, and `mock_server_fn_custom` are deprecated since v0.1.0-rc.16. Use `MockServiceWorker` from `reinhardt_test::msw` instead. The examples below still work but will emit deprecation warnings.

### mock_server_fn (deprecated)

```rust
use reinhardt_pages::testing::*;

// Mock a successful response
mock_server_fn("/api/server_fn/get_user", &user_data);

// Mock an error response (requires status code)
mock_server_fn_error("/api/server_fn/get_user", 404, "Not found");

// Mock with custom response
mock_server_fn_custom("/api/server_fn/get_user", MockResponse {
    status: 200,
    body: serde_json::to_string(&data).unwrap(),
    headers: std::collections::HashMap::new(),
});
```

### Assertions

| Function | Description |
|----------|-------------|
| `assert_server_fn_called(path)` | Verify endpoint was called |
| `assert_server_fn_not_called(path)` | Verify endpoint was NOT called |
| `assert_server_fn_called_with(path, expected)` | Verify call with specific body |
| `assert_server_fn_call_count(path, n)` | Verify exact call count |

### Call History

```rust
// Get all mock calls
let history = get_call_history();

// Get calls for specific endpoint
let calls = get_call_history_for("/api/server_fn/login");
for call in calls {
    println!("Called with: {:?}", call);
}
```

### Cleanup

Always clear mocks after tests:

```rust
#[wasm_bindgen_test]
async fn test_something() {
    mock_server_fn("/api/endpoint", &data);
    // ... test ...
    clear_mocks(); // Required cleanup
}
```

## MockableServerFn (MSW-style)

Requires `msw` feature flag in `Cargo.toml`:

```toml
[dev-dependencies]
reinhardt-pages = { workspace = true, features = ["testing", "msw"] }
```

Enables the `MockableServerFn` trait for type-safe mocking of individual server functions.

## Feature Flags for Testing

| Feature | Description |
|---------|-------------|
| `testing` | Core testing utilities (mock_fetch, mock_http) |
| `msw` | MockableServerFn trait for MSW-style mocking |
| `debug-hooks` | Debug hooks (use_debug_value) |

```toml
# Cargo.toml
[dev-dependencies]
reinhardt-pages = { workspace = true, features = ["testing"] }
wasm-bindgen-test = "0.3"
```

## cfg_aliases in Tests

Ensure `build.rs` is set up for `wasm`/`native` aliases (see routing-ssr.md). Both test targets use the same aliases.

## Testing Standards

- ALL tests MUST use `rstest` (per project standards)
- Follow AAA pattern: `// Arrange`, `// Act`, `// Assert`
- Server function unit tests: `#[tokio::test]` with rstest
- WASM component tests: `#[wasm_bindgen_test]` with rstest
- Always call `clear_mocks()` in test teardown
- Use `reinhardt-query` for SQL in tests, NEVER raw SQL
