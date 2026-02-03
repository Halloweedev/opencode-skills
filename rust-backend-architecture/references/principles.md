# Architectural Principles

The 4 fundamental principles that guide every architectural decision in production Axum backends.

## Principle 1: Explicit Boundaries Over Convenience

**Rule**: Handlers orchestrate, they don't implement business logic or touch databases.

### ❌ Bad - Handler Knows Too Much

```rust
async fn get_user(
    State(pool): State<DbPool>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, AppError> {
    use crate::schema::users::dsl::*;
    let mut conn = pool.get().await?;
    
    // Handler doing database query
    let user = users.find(id).first(&mut conn).await?;
    
    // Handler doing business logic
    if user.deleted_at.is_some() {
        return Err(AppError::NotFound);
    }
    
    // Handler doing authorization
    if !user.email_verified {
        return Err(AppError::Unauthorized);
    }
    
    Ok(Json(user))
}
```

**Problems:**
- Handler coupled to Diesel
- Business logic scattered
- Can't reuse logic from CLI/workers
- Hard to test (needs real database)
- Can't swap database implementation

### ✅ Good - Clean Boundaries

```rust
// Handler - thin orchestration only
async fn get_user(
    State(user_service): State<Arc<UserService>>,
    Path(id): Path<Uuid>,
) -> Result<Json<UserResponse>, AppError> {
    let user = user_service.get_by_id(id).await?;
    Ok(Json(UserResponse::from(user)))
}

// Service - business logic
impl UserService {
    pub async fn get_by_id(&self, id: Uuid) -> Result<User, UserError> {
        let user = self.repo.find_by_id(id).await?;
        
        if user.deleted_at.is_some() {
            return Err(UserError::NotFound(id));
        }
        
        if !user.email_verified {
            return Err(UserError::NotVerified);
        }
        
        Ok(user)
    }
}

// Repository - data access only
impl UserRepository {
    pub async fn find_by_id(&self, id: Uuid) -> Result<User, InfraError> {
        use crate::schema::users::dsl::*;
        let mut conn = self.pool.get().await?;
        
        users.find(id)
            .first(&mut conn)
            .await
            .map_err(InfraError::from)
    }
}
```

**Benefits:**
- Handler = 3 lines
- Service testable without database
- Business logic reusable
- Can swap repository implementation
- Clear ownership

## Principle 2: Dependencies Flow Inward

**Rule**: Higher layers depend on lower layers, never reversed.

### Dependency Direction

```
HTTP Layer (handlers)
    ↓ depends on
Service Layer (business logic)
    ↓ depends on
Repository Layer (data access)
    ↓ depends on
Database (infrastructure)
```

### ❌ Bad - Reversed Dependencies

```rust
// repositories/user_repository.rs
use crate::services::notification_service::NotificationService;  // ❌ WRONG

impl UserRepository {
    async fn create_user(&self, new_user: NewUser) -> Result<User, InfraError> {
        let user = self.insert(new_user).await?;
        
        // ❌ Repository calling service - WRONG DIRECTION
        self.notification_service.send_welcome_email(&user).await?;
        
        Ok(user)
    }
}
```

**Problems:**
- Circular dependency risk
- Repository doing too much
- Can't test repository without notification service
- Violates single responsibility

### ✅ Good - Correct Flow

```rust
// services/user_service.rs
impl UserService {
    pub async fn create_user(&self, data: CreateUserData) -> Result<User, UserError> {
        // Service orchestrates multiple repositories/services
        let user = self.user_repo.create(data).await?;
        
        // ✅ Service decides when to notify
        self.notification_service.send_welcome_email(&user).await?;
        
        Ok(user)
    }
}

// repositories/user_repository.rs
impl UserRepository {
    pub async fn create(&self, new_user: NewUser) -> Result<User, InfraError> {
        // ✅ Repository just does data access
        self.insert(new_user).await
    }
}
```

**Benefits:**
- Clear separation of concerns
- Repository testable in isolation
- Service orchestrates cross-cutting concerns
- No circular dependencies

### Dependency Rules

1. **Handlers** can import services, NOT repositories or database
2. **Services** can import repositories, NOT handlers
3. **Repositories** can import database, NOT services
4. **Domain** imports nothing (pure logic)

## Principle 3: Domain Types vs Transport Types

**Rule**: API types live in `handlers/dto/`, domain types in `domain/`. Never expose internal models directly.

### ❌ Bad - Exposing Internal Model

```rust
// domain/user.rs
#[derive(Serialize, Queryable)]  // ❌ Diesel + API in same type
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub password_hash: String,  // ❌ LEAKED TO API!
    pub internal_notes: Option<String>,  // ❌ Internal data exposed
    pub deleted_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

// handlers/users.rs
async fn get_user(/* ... */) -> Result<Json<User>, AppError> {
    let user = service.get_user(id).await?;
    Ok(Json(user))  // ❌ Exposes password_hash, internal_notes, deleted_at!
}
```

**Problems:**
- Sensitive data leaks (password_hash)
- Internal structure exposed
- Can't change DB schema without breaking API
- No version control over API shape

### ✅ Good - Separate Domain and API

```rust
// domain/user.rs - Internal model
#[derive(Queryable)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub password_hash: String,  // ✅ Internal only
    pub internal_notes: Option<String>,
    pub deleted_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

// handlers/dto/user_dto.rs - API model
#[derive(Serialize)]
pub struct UserResponse {
    pub id: Uuid,
    pub email: String,
    pub created_at: DateTime<Utc>,
    // ✅ No password_hash, internal_notes, or deleted_at
}

impl From<User> for UserResponse {
    fn from(user: User) -> Self {
        UserResponse {
            id: user.id,
            email: user.email,
            created_at: user.created_at,
        }
    }
}

// handlers/users.rs
async fn get_user(/* ... */) -> Result<Json<UserResponse>, AppError> {
    let user = service.get_user(id).await?;
    Ok(Json(UserResponse::from(user)))  // ✅ Safe transformation
}
```

**Benefits:**
- API contract independent of database
- Can add/remove DB fields without breaking API
- Sensitive data never exposed
- API versioning possible

### Type Separation Rules

1. **Domain types** (`domain/`) - Internal truth, Diesel models
2. **DTOs** (`handlers/dto/`) - External API contracts
3. **Never** serialize domain types directly
4. **Always** convert at handler boundary

## Principle 4: Ownership Rules for State

**Rule**: AppState contains services (Arc), services contain repositories (Arc), repositories contain pools.

### ❌ Bad - Everything in AppState

```rust
#[derive(Clone)]
struct AppState {
    db_pool: DbPool,              // ❌ Too low-level
    redis: RedisPool,              // ❌ Too low-level
    s3_client: S3Client,          // ❌ Infrastructure detail
    jwt_secret: String,            // ❌ Config shouldn't be cloned
    sendgrid_key: String,          // ❌ Config shouldn't be cloned
    stripe_client: StripeClient,  // ❌ Payment detail
    config: Config,                // ❌ Cloned on every request
}

async fn handler(State(state): State<AppState>) {
    // Handler now has access to EVERYTHING
    state.db_pool.get().await;  // ❌ Handler doing queries
    state.redis.get().await;     // ❌ Handler using cache directly
}
```

**Problems:**
- Everything coupled to everything
- Handlers have too much power
- Can't enforce boundaries
- Testing requires all infrastructure
- Config cloned unnecessarily

### ✅ Good - Layered Ownership

```rust
// AppState - services only
#[derive(Clone)]
struct AppState {
    user_service: Arc<UserService>,
    auth_service: Arc<AuthService>,
    subscription_service: Arc<SubscriptionService>,
}

// Services own their dependencies
pub struct UserService {
    repo: Arc<UserRepository>,      // ✅ Arc because shared
    email: Arc<EmailService>,        // ✅ Arc because shared
    config: &'static Config,         // ✅ Static ref, not cloned
}

pub struct UserRepository {
    pool: DbPool,  // ✅ Hidden from handlers and services
}

// Handlers see services only
async fn handler(State(user_service): State<Arc<UserService>>) {
    user_service.get_user(id).await;  // ✅ Clean interface
    // ✅ No access to db_pool, config, etc.
}
```

**Benefits:**
- Handlers can't access database
- Services encapsulate infrastructure
- Easy to test (mock services)
- Clear boundaries
- No unnecessary cloning

### Ownership Rules

1. **AppState**: Contains `Arc<Service>` only
2. **Services**: Contain `Arc<Repository>` and clients
3. **Repositories**: Contain pools (hidden)
4. **Config**: Use `&'static` or load once, don't clone

### Arc Usage Rules

```rust
// ❌ WRONG
#[derive(Clone)]
struct AppState {
    user_service: Arc<Arc<UserService>>,  // ❌ Double Arc
    config: Arc<Config>,                   // ❌ Config doesn't need Arc
}

// ✅ CORRECT
#[derive(Clone)]
struct AppState {
    user_service: Arc<UserService>,  // ✅ One Arc
}

// Config handling
lazy_static! {
    static ref CONFIG: Config = Config::from_env().unwrap();
}

impl UserService {
    fn new(repo: Arc<UserRepository>) -> Self {
        UserService {
            repo,
            config: &CONFIG,  // ✅ Static reference
        }
    }
}
```

**Arc rules:**
1. One Arc per shared service
2. Config: use `&'static` or `once_cell`, NOT Arc
3. Read-only data: consider `&'static`
4. Mutable state: `Arc<RwLock<T>>` or `Arc<Mutex<T>>`

## Summary

These 4 principles prevent 90% of architectural problems:

1. **Explicit Boundaries** → Handlers delegate
2. **Dependencies Flow Inward** → No reversed imports
3. **Domain vs Transport** → Separate API from internal
4. **Ownership Rules** → Services in state, not pools

Follow these, and your codebase stays clean at 50k LOC.
