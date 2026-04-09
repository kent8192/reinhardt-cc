# DI with Database and Auth Reference

## DatabaseConnection Injection

`DatabaseConnection` is automatically available as an injectable type when the database feature is enabled. It provides a connection from the pool for the current request.

```rust
use reinhardt::db::prelude::*;
use reinhardt::di::prelude::*;
use reinhardt::views::prelude::*;

#[inject]
pub async fn get_user(
    request: Request,
    db: Inject<DatabaseConnection>,
) -> Response {
    let id: i64 = request.path_param("id")?;
    let user = User::objects()
        .filter(User::id.eq(id))
        .get(&*db)
        .await
        .map_err(|_| HttpError::not_found("User not found"))?;

    Response::json(UserSerializer::build(&user).serialize())
}
```

### Transaction Support

```rust
#[inject]
pub async fn transfer_funds(
    request: Request,
    db: Inject<DatabaseConnection>,
) -> Response {
    let data: TransferRequest = request.json().await?;

    // Begin a transaction
    let tx = db.begin().await?;

    Account::objects()
        .filter(Account::id.eq(data.from_id))
        .update(Account::balance.sub(data.amount))
        .execute(&tx)
        .await?;

    Account::objects()
        .filter(Account::id.eq(data.to_id))
        .update(Account::balance.add(data.amount))
        .execute(&tx)
        .await?;

    tx.commit().await?;

    Response::json(json!({ "status": "completed" }))
}
```

## AuthUser Injection

`AuthUser<T>` extracts the authenticated user from the request. It reads the authentication token (JWT, session, etc.) and resolves the user model.

```rust
use reinhardt::auth::prelude::*;
use reinhardt::di::prelude::*;
use reinhardt::views::prelude::*;

#[inject]
pub async fn get_profile(
    request: Request,
    auth: AuthUser<User>,
) -> Response {
    // auth.user() returns the authenticated User instance
    let user = auth.user();
    Response::json(ProfileSerializer::build(user).serialize())
}
```

### Checking Permissions

```rust
#[inject]
pub async fn delete_user(
    request: Request,
    auth: AuthUser<User>,
) -> Response {
    // Check if the authenticated user has admin permissions
    if !auth.user().is_staff {
        return Err(HttpError::forbidden("Admin access required"));
    }

    let id: i64 = request.path_param("id")?;
    let user = User::objects().get(id).await?;
    user.delete().await?;
    Response::no_content()
}
```

### Optional Authentication

Use `Option<AuthUser<T>>` for endpoints that work for both authenticated and anonymous users:

```rust
#[inject]
pub async fn list_posts(
    request: Request,
    auth: Option<AuthUser<User>>,
) -> Response {
    let mut query = Post::objects().filter(Post::is_published.eq(true));

    // Authenticated users see their own drafts too
    if let Some(auth) = &auth {
        query = query.or_filter(Post::author_id.eq(auth.user().id));
    }

    let posts = query.all().await?;
    Response::json(PostSerializer::build_many(&posts).serialize())
}
```

## Session Injection

`Session` provides access to the current request's session key-value store.

```rust
use reinhardt::auth::prelude::*;
use reinhardt::di::prelude::*;
use reinhardt::views::prelude::*;

#[inject]
pub async fn get_cart(
    request: Request,
    session: Inject<Session>,
) -> Response {
    let cart: Option<Cart> = session.get("cart").await?;
    match cart {
        Some(cart) => Response::json(cart),
        None => Response::json(json!({ "items": [] })),
    }
}

#[inject]
pub async fn add_to_cart(
    request: Request,
    session: Inject<Session>,
) -> Response {
    let item: CartItem = request.json().await?;

    let mut cart: Cart = session
        .get("cart")
        .await?
        .unwrap_or_default();

    cart.add(item);
    session.set("cart", &cart).await?;

    Response::json(cart)
}
```

## Combining Multiple Injections

Handlers can receive any combination of injectable types:

```rust
#[inject]
pub async fn create_order(
    request: Request,
    auth: AuthUser<User>,
    db: Inject<DatabaseConnection>,
    email_service: Inject<Arc<EmailService>>,
    session: Inject<Session>,
) -> Response {
    let cart: Cart = session
        .get("cart")
        .await?
        .ok_or_else(|| HttpError::bad_request("Cart is empty"))?;

    // Create order in a transaction
    let tx = db.begin().await?;
    let order = Order::create_from_cart(&cart, auth.user(), &tx).await?;
    tx.commit().await?;

    // Clear cart from session
    session.remove("cart").await?;

    // Send confirmation email (non-blocking)
    email_service.send_order_confirmation(&order, auth.user()).await?;

    Response::created(OrderSerializer::build(&order).serialize())
}
```

## Custom Repository Pattern with Injectable

Define repository types that encapsulate database access and inject them into handlers.

```rust
use reinhardt::db::prelude::*;
use reinhardt::di::prelude::*;
use async_trait::async_trait;
use std::sync::Arc;

pub struct UserRepository {
    pool: Arc<PgPool>,
}

#[async_trait]
impl Injectable for UserRepository {
    async fn resolve(ctx: &InjectionContext) -> Result<Self, InjectionError> {
        let db = ctx.resolve::<DatabaseConnection>().await?;
        Ok(Self {
            pool: db.pool().clone(),
        })
    }
}

impl UserRepository {
    pub async fn find_by_email(&self, email: &str) -> Result<Option<User>, QueryError> {
        User::objects()
            .filter(User::email.eq(email))
            .first(&*self.pool)
            .await
    }

    pub async fn find_active(&self) -> Result<Vec<User>, QueryError> {
        User::objects()
            .filter(User::is_active.eq(true))
            .order_by(User::username.asc())
            .all(&*self.pool)
            .await
    }

    pub async fn create(&self, username: &str, email: &str) -> Result<User, QueryError> {
        User::objects()
            .create(|u| {
                u.username = username.to_string();
                u.email = email.to_string();
            })
            .execute(&*self.pool)
            .await
    }

    pub async fn deactivate(&self, user_id: i64) -> Result<(), QueryError> {
        User::objects()
            .filter(User::id.eq(user_id))
            .update(User::is_active.set(false))
            .execute(&*self.pool)
            .await?;
        Ok(())
    }
}
```

### Using the Repository in Handlers

```rust
#[inject]
pub async fn list_users(
    request: Request,
    user_repo: Inject<UserRepository>,
) -> Response {
    let users = user_repo.find_active().await?;
    Response::json(UserSerializer::build_many(&users).serialize())
}

#[inject]
pub async fn find_by_email(
    request: Request,
    user_repo: Inject<UserRepository>,
) -> Response {
    let email: String = request.path_param("email")?;
    let user = user_repo
        .find_by_email(&email)
        .await?
        .ok_or_else(|| HttpError::not_found("User not found"))?;

    Response::json(UserSerializer::build(&user).serialize())
}
```

### Service Layer on Top of Repository

```rust
pub struct UserService {
    repo: Arc<UserRepository>,
    email: Arc<EmailService>,
}

#[async_trait]
impl Injectable for UserService {
    async fn resolve(ctx: &InjectionContext) -> Result<Self, InjectionError> {
        let repo = Arc::new(ctx.resolve::<UserRepository>().await?);
        let email = ctx.resolve::<Arc<EmailService>>().await?;
        Ok(Self { repo, email })
    }
}

impl UserService {
    pub async fn register(&self, username: &str, email: &str) -> Result<User, ApiError> {
        // Check for duplicates
        if self.repo.find_by_email(email).await?.is_some() {
            return Err(ApiError::conflict("Email already registered"));
        }

        // Create user
        let user = self.repo.create(username, email).await?;

        // Send welcome email
        self.email.send_welcome(&user).await?;

        Ok(user)
    }
}
```
