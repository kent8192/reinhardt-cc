# Reinhardt View Patterns Reference

## Function-Based Views

Use decorator attributes to define HTTP method handlers. Each function receives a `Request` and returns a `Response`.

```rust
use reinhardt::views::prelude::*;
use reinhardt::rest::prelude::*;

#[get("/users/{id}")]
pub async fn get_user(request: Request, id: Path<i64>) -> Response {
    let user = User::objects()
        .get(*id)
        .await
        .map_err(|_| HttpError::not_found("User not found"))?;

    let serializer = UserSerializer::build(&user);
    Response::json(serializer.serialize())
}

#[post("/users")]
pub async fn create_user(request: Request) -> Response {
    let data = request.json().await?;
    let serializer = UserCreateSerializer::deserialize(&data)?;
    serializer.validate()?;

    let user = serializer.save().await?;
    let output = UserSerializer::build(&user);
    Response::json(output.serialize()).status(StatusCode::CREATED)
}

#[put("/users/{id}")]
pub async fn update_user(request: Request, id: Path<i64>) -> Response {
    let user = User::objects().get(*id).await?;
    let data = request.json().await?;
    let serializer = UserSerializer::deserialize_with(&data, &user)?;
    serializer.validate()?;

    let updated = serializer.save().await?;
    let output = UserSerializer::build(&updated);
    Response::json(output.serialize())
}

#[delete("/users/{id}")]
pub async fn delete_user(request: Request, id: Path<i64>) -> Response {
    let user = User::objects().get(*id).await?;
    user.delete().await?;
    Response::no_content()
}
```

### Available HTTP Method Decorators

| Decorator | HTTP Method | Typical Use |
|-----------|-------------|-------------|
| `#[get("...")]` | GET | Retrieve resource(s) |
| `#[post("...")]` | POST | Create a resource |
| `#[put("...")]` | PUT | Full update of a resource |
| `#[patch("...")]` | PATCH | Partial update of a resource |
| `#[delete("...")]` | DELETE | Delete a resource |

## Views with Dependency Injection

Use `#[inject]` to receive services from the DI container:

```rust
use reinhardt::di::prelude::*;
use reinhardt::views::prelude::*;

#[get("/users")]
#[inject]
pub async fn list_users(
    request: Request,
    user_service: Inject<Arc<UserService>>,
    auth: AuthUser,
) -> Response {
    let users = user_service.list_active().await?;
    let serializer = UserSerializer::build_many(&users);
    Response::json(serializer.serialize())
}
```

### Common Injectable Types

| Type | Description |
|------|-------------|
| `Inject<Arc<T>>` | Shared service from the DI container |
| `AuthUser` | Authenticated user (extracted from request) |
| `DatabaseConnection` | Database connection from the pool |
| `Path<T>` | URL path parameters |
| `QueryParams<T>` | URL query string parameters |

## Generic Views

Generic views provide pre-built CRUD behavior. Override methods for customization.

### ListView

```rust
use reinhardt::views::generic::*;

pub struct UserListView;

impl ListView for UserListView {
    type Model = User;
    type Serializer = UserSerializer;

    fn get_queryset(&self) -> QuerySet<User> {
        User::objects().filter(User::is_active.eq(true))
    }

    fn get_pagination(&self) -> Option<Box<dyn Pagination>> {
        Some(Box::new(PageNumberPagination::new(20)))
    }
}
```

### DetailView

```rust
pub struct UserDetailView;

impl DetailView for UserDetailView {
    type Model = User;
    type Serializer = UserSerializer;
    type LookupField = i64;

    fn get_object(&self, id: Self::LookupField) -> QuerySet<User> {
        User::objects().filter(User::id.eq(id))
    }
}
```

### CreateView

```rust
pub struct UserCreateView;

impl CreateView for UserCreateView {
    type Model = User;
    type Serializer = UserCreateSerializer;

    fn perform_create(&self, serializer: &Self::Serializer) -> Result<User, ApiError> {
        serializer.save()
    }
}
```

### UpdateView and DestroyView

```rust
pub struct UserUpdateView;

impl UpdateView for UserUpdateView {
    type Model = User;
    type Serializer = UserSerializer;
    type LookupField = i64;
}

pub struct UserDestroyView;

impl DestroyView for UserDestroyView {
    type Model = User;
    type LookupField = i64;
}
```

### ViewSet (Combines All CRUD)

```rust
pub struct UserViewSet;

impl ViewSet for UserViewSet {
    type Model = User;
    type Serializer = UserSerializer;
    type CreateSerializer = UserCreateSerializer;
    type LookupField = i64;

    fn get_queryset(&self) -> QuerySet<User> {
        User::objects().all()
    }

    fn get_permissions(&self) -> Vec<Box<dyn Permission>> {
        vec![Box::new(IsAuthenticated)]
    }
}
```

## Server Functions (Pages/WASM)

For full-stack applications using `--with-pages`, server functions allow RPC-style calls from client-side WASM:

```rust
use reinhardt::pages::prelude::*;

#[server_fn]
pub async fn get_user_profile(user_id: i64) -> Result<UserProfile, ServerFnError> {
    let db = use_context::<DatabaseConnection>()?;
    let user = User::objects().get(user_id).await?;
    Ok(UserProfile::from(user))
}

#[server_fn]
pub async fn update_profile(
    user_id: i64,
    name: String,
    bio: String,
) -> Result<(), ServerFnError> {
    let db = use_context::<DatabaseConnection>()?;
    User::objects()
        .filter(User::id.eq(user_id))
        .update(User::name.set(name), User::bio.set(bio))
        .await?;
    Ok(())
}
```

## Response Helpers

| Helper | Description | Status Code |
|--------|-------------|-------------|
| `Response::json(value)` | JSON response body | 200 OK |
| `Response::json(value).status(code)` | JSON with custom status | Custom |
| `Response::no_content()` | Empty 204 response | 204 No Content |
| `Response::created(value)` | JSON with 201 status | 201 Created |
| `HttpError::bad_request(msg)` | 400 error response | 400 Bad Request |
| `HttpError::unauthorized(msg)` | 401 error response | 401 Unauthorized |
| `HttpError::forbidden(msg)` | 403 error response | 403 Forbidden |
| `HttpError::not_found(msg)` | 404 error response | 404 Not Found |
| `HttpError::conflict(msg)` | 409 error response | 409 Conflict |
| `HttpError::internal(msg)` | 500 error response | 500 Internal Server Error |
