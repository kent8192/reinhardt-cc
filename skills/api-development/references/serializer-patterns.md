# Reinhardt Serializer Patterns Reference

## ModelSerializer

`ModelSerializer` automatically generates serialization/deserialization logic from a model definition. It is the recommended approach for standard CRUD endpoints.

```rust
use reinhardt::rest::prelude::*;
use crate::apps::user::models::User;

#[derive(ModelSerializer)]
#[serializer(model = "User")]
pub struct UserSerializer {
    #[serializer(read_only)]
    pub id: i64,

    #[serializer(max_length = 150)]
    pub username: String,

    #[serializer(max_length = 254)]
    pub email: String,

    #[serializer(read_only)]
    pub created_at: String,

    #[serializer(read_only)]
    pub is_active: bool,
}
```

### ModelSerializer Options

| Option | Description |
|--------|-------------|
| `model = "ModelName"` | The model this serializer maps to |
| `fields = [...]` | Explicit list of fields to include |
| `exclude = [...]` | Fields to exclude from the model |
| `read_only` | Field is included in output but ignored on input |
| `write_only` | Field is accepted on input but excluded from output |
| `required` | Field must be present in input (default for non-Option fields) |
| `max_length = N` | Maximum string length validation |
| `min_length = N` | Minimum string length validation |
| `min_value = N` | Minimum numeric value validation |
| `max_value = N` | Maximum numeric value validation |

### Build, Serialize, and Deserialize

```rust
// Build a serializer from a model instance
let user = User::objects().get(id).await?;
let serializer = UserSerializer::build(&user);

// Serialize to JSON-compatible value
let json_value = serializer.serialize();

// Deserialize from request data
let data: serde_json::Value = request.json().await?;
let serializer = UserSerializer::deserialize(&data)?;

// Validate and save
let user = serializer.save().await?;
```

## Serializer Field Types

| Field Type | Rust Type | Validation |
|------------|-----------|------------|
| `CharField` | `String` | `max_length`, `min_length`, `blank` |
| `IntegerField` | `i32` / `i64` | `min_value`, `max_value` |
| `FloatField` | `f32` / `f64` | `min_value`, `max_value` |
| `BooleanField` | `bool` | None |
| `EmailField` | `String` | Email format validation |
| `URLField` | `String` | URL format validation |
| `DateField` | `NaiveDate` | ISO 8601 date format |
| `DateTimeField` | `DateTime<Utc>` | ISO 8601 datetime format |
| `ChoiceField` | `String` / enum | Validates against allowed choices |
| `UUIDField` | `Uuid` | UUID format validation |
| `DecimalField` | `Decimal` | `max_digits`, `decimal_places` |
| `JSONField` | `serde_json::Value` | Valid JSON |

## Relation Fields

| Field Type | Description | Use Case |
|------------|-------------|----------|
| `PrimaryKeyRelatedField` | Represents the relation by its primary key | Default for ForeignKey fields |
| `SlugRelatedField` | Represents the relation by a slug/unique field | Lookup by username, slug, etc. |
| `HyperlinkedRelatedField` | Represents the relation as a URL | HATEOAS-style APIs |
| `StringRelatedField` | Represents the relation by its `Display` implementation | Read-only display |
| `NestedSerializer` | Embeds the full related object (read-only) | Detail views with nested data |
| `WritableNestedSerializer` | Embeds the full related object (read-write) | Creating/updating nested objects |

### Relation Field Examples

```rust
#[derive(ModelSerializer)]
#[serializer(model = "Post")]
pub struct PostSerializer {
    #[serializer(read_only)]
    pub id: i64,

    pub title: String,

    // ForeignKey as primary key (default)
    pub author: PrimaryKeyRelatedField<User>,

    // ForeignKey as nested object
    #[serializer(read_only)]
    pub author_detail: NestedSerializer<UserSerializer>,

    // ManyToMany as list of primary keys
    pub tags: Vec<PrimaryKeyRelatedField<Tag>>,
}
```

## Custom Serializer

For non-model serialization or complex validation logic, implement the `Serializer` trait directly:

```rust
use reinhardt::rest::prelude::*;

pub struct LoginSerializer {
    pub username: String,
    pub password: String,
}

impl Serializer for LoginSerializer {
    type Output = LoginData;

    fn deserialize(data: &serde_json::Value) -> Result<Self, ValidationError> {
        let username = data
            .get("username")
            .and_then(|v| v.as_str())
            .ok_or_else(|| ValidationError::field("username", "This field is required."))?
            .to_string();

        let password = data
            .get("password")
            .and_then(|v| v.as_str())
            .ok_or_else(|| ValidationError::field("password", "This field is required."))?
            .to_string();

        Ok(Self { username, password })
    }

    fn validate(&self) -> Result<(), ValidationError> {
        if self.username.is_empty() {
            return Err(ValidationError::field("username", "Username cannot be empty."));
        }
        if self.password.len() < 8 {
            return Err(ValidationError::field(
                "password",
                "Password must be at least 8 characters.",
            ));
        }
        Ok(())
    }

    fn serialize(&self) -> serde_json::Value {
        serde_json::json!({
            "username": self.username,
        })
    }
}
```

## Validation Example

Custom field-level and object-level validation:

```rust
#[derive(ModelSerializer)]
#[serializer(model = "User")]
pub struct UserCreateSerializer {
    pub username: String,
    pub email: String,

    #[serializer(write_only, min_length = 8)]
    pub password: String,

    #[serializer(write_only)]
    pub password_confirm: String,
}

impl UserCreateSerializer {
    /// Object-level validation: ensure passwords match
    pub fn validate(&self) -> Result<(), ValidationError> {
        if self.password != self.password_confirm {
            return Err(ValidationError::non_field(
                "Passwords do not match.",
            ));
        }
        Ok(())
    }

    /// Field-level validation: ensure username is not reserved
    pub fn validate_username(value: &str) -> Result<(), ValidationError> {
        let reserved = ["admin", "root", "system"];
        if reserved.contains(&value.to_lowercase().as_str()) {
            return Err(ValidationError::field(
                "username",
                "This username is reserved.",
            ));
        }
        Ok(())
    }
}
```
