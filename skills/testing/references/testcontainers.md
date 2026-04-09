# TestContainers Reference

## Prerequisites

- **Docker Desktop** must be installed and running (NOT Podman)
- `DOCKER_HOST` must point to the Docker socket (not Podman socket)
- `.testcontainers.properties` in the project root forces Docker usage (already configured in reinhardt projects)

Verify Docker is available:

```bash
docker info | head -5
```

## PostgreSQL Container Fixture

Use `testcontainers::GenericImage` to spin up a PostgreSQL container for integration tests.

```rust
use rstest::*;
use std::sync::Arc;
use testcontainers::{runners::AsyncRunner, GenericImage, ImageExt};
use sqlx::postgres::PgPoolOptions;
use sqlx::PgPool;

pub struct TestPool {
    pub pool: Arc<PgPool>,
    // Hold the container handle to keep it alive for the test duration
    _container: testcontainers::ContainerAsync<GenericImage>,
}

impl TestPool {
    pub fn pool(&self) -> &PgPool {
        &self.pool
    }
}

#[fixture]
async fn test_pool() -> TestPool {
    let container = GenericImage::new("postgres", "16-alpine")
        .with_exposed_port(5432.into())
        .with_wait_for(
            testcontainers::core::WaitFor::message_on_stderr(
                "database system is ready to accept connections"
            )
        )
        .with_startup_timeout(std::time::Duration::from_secs(30))
        .with_env_var("POSTGRES_HOST_AUTH_METHOD", "trust")
        .with_env_var("POSTGRES_DB", "test_db")
        .with_env_var("POSTGRES_USER", "test_user")
        .start()
        .await
        .expect("Failed to start PostgreSQL container");

    let host_port = container
        .get_host_port_ipv4(5432)
        .await
        .expect("Failed to get mapped port");

    let database_url = format!(
        "postgres://test_user@127.0.0.1:{}/test_db",
        host_port
    );

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to test database");

    TestPool {
        pool: Arc::new(pool),
        _container: container,
    }
}
```

## Table Setup Fixture

Use `reinhardt_query::Query::create_table` with `ColumnDef` to set up test tables.

```rust
use reinhardt_query::{Query, ColumnDef, Table, Iden};
use sqlx::Executor;

#[derive(Iden)]
enum Users {
    Table,
    Id,
    Username,
    Email,
    IsActive,
    CreatedAt,
}

#[fixture]
async fn test_pool_with_users(#[future] test_pool: TestPool) -> TestPool {
    let pool = test_pool.await;

    let create_stmt = Query::create_table()
        .table(Users::Table)
        .if_not_exists()
        .col(
            ColumnDef::new(Users::Id)
                .big_integer()
                .auto_increment()
                .primary_key(),
        )
        .col(
            ColumnDef::new(Users::Username)
                .string_len(150)
                .not_null()
                .unique_key(),
        )
        .col(
            ColumnDef::new(Users::Email)
                .string_len(254)
                .not_null(),
        )
        .col(
            ColumnDef::new(Users::IsActive)
                .boolean()
                .not_null()
                .default(true),
        )
        .col(
            ColumnDef::new(Users::CreatedAt)
                .timestamp_with_time_zone()
                .not_null()
                .default(reinhardt_query::Expr::current_timestamp()),
        )
        .build_any(&reinhardt_query::PostgresQueryBuilder);

    pool.pool()
        .execute(sqlx::query(&create_stmt))
        .await
        .expect("Failed to create users table");

    pool
}
```

## Usage in Tests

```rust
use rstest::*;
use serial_test::serial;

#[rstest]
#[serial(db)]
#[tokio::test]
async fn test_insert_and_query_user(#[future] test_pool_with_users: TestPool) {
    // Arrange
    let pool = test_pool_with_users.await;
    let insert_stmt = Query::insert()
        .into_table(Users::Table)
        .columns([Users::Username, Users::Email])
        .values_panic(["testuser".into(), "test@example.com".into()])
        .build_any(&reinhardt_query::PostgresQueryBuilder);
    pool.pool().execute(sqlx::query(&insert_stmt)).await.unwrap();

    // Act
    let row: (String, String) = sqlx::query_as(
        "SELECT username, email FROM users WHERE username = $1"
    )
        .bind("testuser")
        .fetch_one(pool.pool())
        .await
        .unwrap();

    // Assert
    assert_eq!(row.0, "testuser");
    assert_eq!(row.1, "test@example.com");
}
```

## Redis Container Fixture

```rust
use testcontainers::{runners::AsyncRunner, GenericImage, ImageExt};

pub struct TestRedis {
    pub url: String,
    _container: testcontainers::ContainerAsync<GenericImage>,
}

#[fixture]
async fn test_redis() -> TestRedis {
    let container = GenericImage::new("redis", "7-alpine")
        .with_exposed_port(6379.into())
        .with_wait_for(
            testcontainers::core::WaitFor::message_on_stdout(
                "Ready to accept connections"
            )
        )
        .with_startup_timeout(std::time::Duration::from_secs(15))
        .start()
        .await
        .expect("Failed to start Redis container");

    let host_port = container
        .get_host_port_ipv4(6379)
        .await
        .expect("Failed to get mapped port");

    TestRedis {
        url: format!("redis://127.0.0.1:{}", host_port),
        _container: container,
    }
}

#[rstest]
#[tokio::test]
async fn test_redis_set_get(#[future] test_redis: TestRedis) {
    // Arrange
    let redis = test_redis.await;
    let client = redis::Client::open(redis.url.as_str()).unwrap();
    let mut conn = client.get_multiplexed_async_connection().await.unwrap();

    // Act
    redis::cmd("SET")
        .arg("key")
        .arg("value")
        .exec_async(&mut conn)
        .await
        .unwrap();
    let result: String = redis::cmd("GET")
        .arg("key")
        .query_async(&mut conn)
        .await
        .unwrap();

    // Assert
    assert_eq!(result, "value");
}
```

## MySQL Container Fixture

```rust
#[fixture]
async fn test_mysql() -> TestMySql {
    let container = GenericImage::new("mysql", "8.0")
        .with_exposed_port(3306.into())
        .with_wait_for(
            testcontainers::core::WaitFor::message_on_stderr(
                "ready for connections"
            )
        )
        .with_startup_timeout(std::time::Duration::from_secs(60))
        .with_env_var("MYSQL_ROOT_PASSWORD", "test")
        .with_env_var("MYSQL_DATABASE", "test_db")
        .start()
        .await
        .expect("Failed to start MySQL container");

    let host_port = container
        .get_host_port_ipv4(3306)
        .await
        .expect("Failed to get mapped port");

    let database_url = format!(
        "mysql://root:test@127.0.0.1:{}/test_db",
        host_port
    );

    let pool = sqlx::mysql::MySqlPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to test MySQL");

    TestMySql {
        pool: Arc::new(pool),
        _container: container,
    }
}
```

## Cargo.toml Dev-Dependencies

```toml
[dev-dependencies]
rstest = "0.23"
serial_test = "3"
testcontainers = "0.23"
testcontainers-modules = { version = "0.11", features = ["postgres", "redis"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "mysql"] }
tokio = { version = "1", features = ["full"] }
```

## Best Practices

- **Always hold the container handle** (`_container` field) to keep the container alive for the test duration. Dropping the handle stops the container.
- **Use `#[serial(db)]`** when tests share a database or modify global state.
- **Set startup timeouts** to avoid flaky tests from slow container pulls.
- **Use alpine images** (e.g., `postgres:16-alpine`) for faster startup.
- **Clean up test data** between tests or use separate containers per test for isolation.
- **Never hardcode ports** — always use `get_host_port_ipv4()` for the dynamically assigned port.
