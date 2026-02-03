# Module & Layer Boundaries

Complete guide on organizing code in layers with clear boundaries.

## Directory Structure

```
src/
├── main.rs              # Bootstrap only: config, DI, server startup
├── lib.rs               # Public API surface
│
├── handlers/            # HTTP layer - thin orchestration
│   ├── mod.rs
│   ├── auth.rs          # Auth endpoints
│   ├── users.rs         # User endpoints
│   ├── subscriptions.rs # Subscription endpoints
│   └── dto/             # Request/Response types
│       ├── mod.rs
│       ├── auth_dto.rs
│       ├── user_dto.rs
│       └── subscription_dto.rs
│
├── services/            # Business logic layer
│   ├── mod.rs
│   ├── user_service.rs
│   ├── auth_service.rs
│   └── subscription_service.rs
│
├── repositories/        # Data access layer
│   ├── mod.rs
│   ├── user_repository.rs
│   └── subscription_repository.rs
│
├── domain/              # Core types and logic
│   ├── mod.rs
│   ├── user.rs          # User entity
│   ├── subscription.rs  # Subscription entity
│   └── errors.rs        # Domain errors
│
├── infrastructure/      # External concerns
│   ├── mod.rs
│   ├── database.rs      # DB setup
│   ├── email.rs         # Email client
│   └── storage.rs       # S3/file storage
│
├── middleware/          # Cross-cutting concerns
│   ├── mod.rs
│   ├── auth.rs
│   ├── tracing.rs
│   └── rate_limit.rs
│
├── config.rs            # Configuration
├── error.rs             # Application-wide error type
└── schema.rs            # Diesel schema
```

## main.rs - Bootstrap Only

**Rule**: `main.rs` should be < 100 lines. If longer, you're doing too much.

### ✅ Good main.rs

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 1. Setup tracing
    setup_tracing();
    
    // 2. Load config
    let config = Config::from_env()?;
    
    // 3. Setup infrastructure
    let db_pool = setup_database(&config).await?;
    let email_client = setup_email(&config)?;
    let storage_client = setup_storage(&config)?;
    
    // 4. Build dependency graph
    let user_repo = Arc::new(UserRepository::new(db_pool.clone()));
    let subscription_repo = Arc::new(SubscriptionRepository::new(db_pool.clone()));
    
    let user_service = Arc::new(UserService::new(
        user_repo.clone(),
        email_client.clone(),
    ));
    
    let subscription_service = Arc::new(SubscriptionService::new(
        subscription_repo,
        user_repo,
        storage_client,
    ));
    
    let auth_service = Arc::new(AuthService::new(
        user_service.clone(),
        &config.jwt_secret,
    ));
    
    // 5. Create app state
    let state = AppState {
        user_service,
        auth_service,
        subscription_service,
    };
    
    // 6. Build and run server
    let app = create_router(state);
    let addr = SocketAddr::from(([0, 0, 0, 0], config.server.port));
    let listener = tokio::net::TcpListener::bind(addr).await?;
    
    info!("Server listening on {}", addr);
    axum::serve(listener, app).await?;
    
    Ok(())
}

fn setup_tracing() {
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .init();
}

async fn setup_database(config: &Config) -> anyhow::Result<DbPool> {
    Database::connect(&config.database_url).await
}

fn setup_email(config: &Config) -> anyhow::Result<Arc<EmailClient>> {
    Ok(Arc::new(EmailClient::new(&config.email)))
}

fn setup_storage(config: &Config) -> anyhow::Result<Arc<S3Client>> {
    Ok(Arc::new(S3Client::new(&config.s3)))
}

fn create_router(state: AppState) -> Router {
    Router::new()
        .nest("/api/v1", api_routes())
        .with_state(state)
        .layer(TraceLayer::new_for_http())
}

fn api_routes() -> Router<AppState> {
    Router::new()
        .merge(auth_routes())
        .merge(user_routes())
        .merge(subscription_routes())
}
```

### ❌ Bad main.rs

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 500 lines of:
    // - Route definitions inline
    // - Business logic
    // - Error handling
    // - Middleware setup
    // - Database queries
    // - Everything
}
```

## lib.rs - Public Surface

**Rule**: Only export what external crates need. Implementation stays private.

### ✅ Good lib.rs

```rust
// Public modules
pub mod config;
pub mod error;
pub mod handlers;
pub mod services;
pub mod domain;

// Re-export commonly used items
pub use config::Config;
pub use error::AppError;
pub use domain::{User, Subscription};

// Private modules (internal implementation)
mod repositories;
mod infrastructure;
mod middleware;
pub(crate) mod schema;  // Visible to crate, not external
```

### ❌ Bad lib.rs

```rust
// Everything public
pub mod config;
pub mod error;
pub mod handlers;
pub mod services;
pub mod repositories;  // ❌ Should be private
pub mod infrastructure;  // ❌ Should be private
pub mod middleware;  // ❌ Should be private
pub mod schema;  // ❌ Diesel internals exposed
```

## Handler Layer

**Rules:**
- < 50 lines per handler
- No business logic
- No database access
- Extract State, parse input, call service, map response

### ✅ Good Handler

```rust
// handlers/users.rs
use axum::{extract::{Path, State}, Json};
use uuid::Uuid;
use crate::{AppError, services::UserService, handlers::dto::UserResponse};

pub async fn get_user(
    State(user_service): State<Arc<UserService>>,
    Path(id): Path<Uuid>,
) -> Result<Json<UserResponse>, AppError> {
    let user = user_service.get_by_id(id).await?;
    Ok(Json(UserResponse::from(user)))
}

pub async fn create_user(
    State(user_service): State<Arc<UserService>>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<Json<UserResponse>, AppError> {
    let user = user_service.create(payload.into()).await?;
    Ok(Json(UserResponse::from(user)))
}

pub async fn update_user(
    State(user_service): State<Arc<UserService>>,
    Path(id): Path<Uuid>,
    Json(payload): Json<UpdateUserRequest>,
) -> Result<Json<UserResponse>, AppError> {
    let user = user_service.update(id, payload.into()).await?;
    Ok(Json(UserResponse::from(user)))
}
```

**Characteristics:**
- Each handler: 3-5 lines
- No `if` statements
- No business logic
- Just extract → call → map

### ❌ Bad Handler

```rust
pub async fn create_user(
    State(pool): State<DbPool>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<Json<User>, AppError> {
    // ❌ Validation in handler
    if payload.email.is_empty() {
        return Err(AppError::Validation("Email required".into()));
    }
    
    // ❌ Business logic in handler
    let password_hash = hash_password(&payload.password)?;
    
    // ❌ Database query in handler
    use crate::schema::users::dsl::*;
    let mut conn = pool.get().await?;
    
    let user = diesel::insert_into(users)
        .values(&NewUser {
            email: payload.email,
            password_hash,
        })
        .get_result(&mut conn)
        .await?;
    
    // ❌ Side effects in handler
    send_welcome_email(&user).await?;
    
    Ok(Json(user))
}
```

## Service Layer

**Rules:**
- Contains business logic
- Orchestrates repositories
- Returns domain types (not HTTP types)
- Testable without HTTP

### ✅ Good Service

```rust
// services/user_service.rs
pub struct UserService {
    repo: Arc<UserRepository>,
    email: Arc<EmailService>,
    config: &'static Config,
}

impl UserService {
    pub fn new(
        repo: Arc<UserRepository>,
        email: Arc<EmailService>,
    ) -> Self {
        UserService {
            repo,
            email,
            config: &CONFIG,
        }
    }
    
    pub async fn create(&self, data: CreateUserData) -> Result<User, UserError> {
        // Business logic: validation
        self.validate_email(&data.email)?;
        
        // Business logic: password hashing
        let password_hash = self.hash_password(&data.password)?;
        
        // Data access: delegate to repository
        let user = self.repo.create(NewUser {
            email: data.email,
            password_hash,
        }).await?;
        
        // Business logic: side effects
        self.email.send_welcome(&user).await?;
        
        Ok(user)
    }
    
    pub async fn get_by_id(&self, id: Uuid) -> Result<User, UserError> {
        self.repo.find_by_id(id).await
            .map_err(|_| UserError::NotFound(id))
    }
    
    fn validate_email(&self, email: &str) -> Result<(), UserError> {
        if !email.contains('@') {
            return Err(UserError::InvalidEmail);
        }
        Ok(())
    }
    
    fn hash_password(&self, password: &str) -> Result<String, UserError> {
        argon2::hash_encoded(
            password.as_bytes(),
            self.config.password_salt.as_bytes(),
            &argon2::Config::default(),
        ).map_err(|_| UserError::HashingFailed)
    }
}
```

**Characteristics:**
- < 300 lines per service
- No HTTP types
- No Diesel queries
- Calls repositories
- Contains business rules

## Repository Layer

**Rules:**
- Only place with Diesel queries
- Implements trait for testing
- Returns domain types
- No business logic

### ✅ Good Repository

```rust
// repositories/user_repository.rs
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<User, InfraError>;
    async fn find_by_email(&self, email: &str) -> Result<User, InfraError>;
    async fn create(&self, user: NewUser) -> Result<User, InfraError>;
    async fn update(&self, id: Uuid, updates: UserUpdate) -> Result<User, InfraError>;
}

pub struct PgUserRepository {
    pool: DbPool,
}

impl PgUserRepository {
    pub fn new(pool: DbPool) -> Self {
        PgUserRepository { pool }
    }
}

#[async_trait]
impl UserRepository for PgUserRepository {
    async fn find_by_id(&self, user_id: Uuid) -> Result<User, InfraError> {
        use crate::schema::users::dsl::*;
        let mut conn = self.pool.get().await?;
        
        users.find(user_id)
            .first(&mut conn)
            .await
            .map_err(InfraError::from)
    }
    
    async fn find_by_email(&self, user_email: &str) -> Result<User, InfraError> {
        use crate::schema::users::dsl::*;
        let mut conn = self.pool.get().await?;
        
        users.filter(email.eq(user_email))
            .first(&mut conn)
            .await
            .map_err(InfraError::from)
    }
    
    async fn create(&self, new_user: NewUser) -> Result<User, InfraError> {
        use crate::schema::users::dsl::*;
        let mut conn = self.pool.get().await?;
        
        diesel::insert_into(users)
            .values(&new_user)
            .get_result(&mut conn)
            .await
            .map_err(InfraError::from)
    }
}
```

**Characteristics:**
- Trait-based (testable)
- Only Diesel queries
- No business logic
- < 200 lines

## File Size Guidelines

| File Type | Max Lines | Ideal |
|-----------|-----------|-------|
| main.rs | 100 | 50-75 |
| Handler | 50 | 20-30 |
| Service | 500 | 200-300 |
| Repository | 200 | 100-150 |
| Domain type | 200 | 50-100 |

**If exceeding limits:**
- Handler too big → Extract to service
- Service too big → Split into multiple services
- Repository too big → Split by table

## Import Rules

```rust
// ✅ CORRECT
// handlers/users.rs
use crate::services::UserService;  // ✅ Handler imports service
// NOT: use crate::repositories::UserRepository;  // ❌ Handler skips service

// services/user_service.rs
use crate::repositories::UserRepository;  // ✅ Service imports repository
// NOT: use crate::handlers::user_handler;  // ❌ Service doesn't import handler

// repositories/user_repository.rs
use crate::schema::users;  // ✅ Repository imports schema
// NOT: use crate::services::UserService;  // ❌ Repository doesn't import service
```

## Summary

**Layers:**
1. **Handlers** (handlers/) - HTTP boundary, < 50 lines
2. **Services** (services/) - Business logic, < 300 lines
3. **Repositories** (repositories/) - Data access, < 200 lines
4. **Domain** (domain/) - Core types, < 200 lines

**main.rs:** < 100 lines, just wiring
**lib.rs:** Public API, hide implementation

**If confused where code goes, ask: "Is this HTTP, business logic, or data access?"**
