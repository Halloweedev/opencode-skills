# Critical Anti-Patterns

10 concrete anti-patterns that will destroy your architecture. If you see these, fix immediately.

## Anti-Pattern 1: Direct Database Access in Handlers

### ❌ The Problem

```rust
async fn get_user(
    State(pool): State<DbPool>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, AppError> {
    use crate::schema::users::dsl::*;
    let mut conn = pool.get().await?;
    
    // ❌ Handler doing Diesel query
    let user = users
        .find(id)
        .first(&mut conn)
        .await?;
    
    Ok(Json(user))
}
```

**Why it's wrong:**
- Handler coupled to Diesel (can't swap DB)
- Can't reuse from CLI/workers
- Hard to test (needs real database)
- Business logic will leak in here next

### ✅ The Fix

```rust
// Handler
async fn get_user(
    State(user_service): State<Arc<UserService>>,
    Path(id): Path<Uuid>,
) -> Result<Json<UserResponse>, AppError> {
    let user = user_service.get_by_id(id).await?;
    Ok(Json(user.into()))
}

// Service
impl UserService {
    pub async fn get_by_id(&self, id: Uuid) -> Result<User, UserError> {
        self.repo.find_by_id(id).await
    }
}

// Repository
impl UserRepository {
    pub async fn find_by_id(&self, id: Uuid) -> Result<User, InfraError> {
        use crate::schema::users::dsl::*;
        let mut conn = self.pool.get().await?;
        users.find(id).first(&mut conn).await.map_err(Into::into)
    }
}
```

## Anti-Pattern 2: Business Logic in Handlers

### ❌ The Problem

```rust
async fn create_subscription(
    State(pool): State<DbPool>,
    Json(payload): Json<CreateSubscriptionRequest>,
) -> Result<Json<Subscription>, AppError> {
    // ❌ Validation in handler
    if payload.plan_id.is_empty() {
        return Err(AppError::Validation("Plan ID required".into()));
    }
    
    // ❌ Business rules in handler
    let price = match payload.plan_id.as_str() {
        "basic" => 9.99,
        "pro" => 19.99,
        "enterprise" => 49.99,
        _ => return Err(AppError::Validation("Invalid plan".into())),
    };
    
    // ❌ Trial period logic in handler
    let trial_end = if payload.use_trial {
        Some(Utc::now() + Duration::days(14))
    } else {
        None
    };
    
    // ❌ Database access in handler
    use crate::schema::subscriptions::dsl::*;
    let mut conn = pool.get().await?;
    let subscription = diesel::insert_into(subscriptions)
        .values(&NewSubscription {
            user_id: payload.user_id,
            plan_id: payload.plan_id,
            price,
            trial_end,
            status: "active",
        })
        .get_result(&mut conn)
        .await?;
    
    // ❌ Side effects in handler
    send_welcome_email(&subscription).await?;
    
    Ok(Json(subscription))
}
```

**Why it's wrong:**
- Handler is 50+ lines
- Can't test business logic without HTTP
- Can't reuse logic from CLI
- Will grow to 200+ lines as features added

### ✅ The Fix

```rust
// Handler - 3 lines
async fn create_subscription(
    State(service): State<Arc<SubscriptionService>>,
    Json(payload): Json<CreateSubscriptionRequest>,
) -> Result<Json<SubscriptionResponse>, AppError> {
    let subscription = service.create(payload.into()).await?;
    Ok(Json(subscription.into()))
}

// Service - business logic
impl SubscriptionService {
    pub async fn create(&self, data: CreateSubscriptionData) -> Result<Subscription, SubscriptionError> {
        // Validation
        self.validate_plan(&data.plan_id)?;
        
        // Business rules
        let price = self.calculate_price(&data.plan_id)?;
        let trial_end = if data.use_trial {
            Some(self.calculate_trial_end())
        } else {
            None
        };
        
        // Create subscription
        let subscription = self.repo.create(NewSubscription {
            user_id: data.user_id,
            plan_id: data.plan_id,
            price,
            trial_end,
            status: SubscriptionStatus::Active,
        }).await?;
        
        // Side effects
        self.email_service.send_welcome(&subscription).await?;
        
        Ok(subscription)
    }
}
```

## Anti-Pattern 3: Leaking Diesel Types to HTTP Layer

### ❌ The Problem

```rust
// Using Diesel's Queryable struct directly in API
#[derive(Queryable, Serialize)]  // ❌ Both DB and API concerns
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub password_hash: String,  // ❌ Will be exposed in JSON!
    pub internal_role: String,
    pub created_at: DateTime<Utc>,
}

async fn list_users() -> Result<Json<Vec<User>>, AppError> {
    // ❌ Returning Diesel model directly
    let users = repo.list().await?;
    Ok(Json(users))
}
```

**Why it's wrong:**
- Password hash exposed in API
- Internal fields exposed
- API shape tied to database schema
- Can't version API independently

### ✅ The Fix

```rust
// Domain model (internal)
#[derive(Queryable)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub password_hash: String,  // ✅ Internal only
    pub internal_role: String,
    pub created_at: DateTime<Utc>,
}

// API response (external)
#[derive(Serialize)]
pub struct UserResponse {
    pub id: Uuid,
    pub email: String,
    pub created_at: DateTime<Utc>,
    // ✅ No password_hash or internal_role
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

async fn list_users() -> Result<Json<Vec<UserResponse>>, AppError> {
    let users = service.list().await?;
    Ok(Json(users.into_iter().map(Into::into).collect()))
}
```

## Anti-Pattern 4: God AppState

### ❌ The Problem

```rust
#[derive(Clone)]
struct AppState {
    db_pool: DbPool,
    redis: RedisPool,
    s3: S3Client,
    stripe: StripeClient,
    sendgrid: SendgridClient,
    jwt_secret: String,
    api_keys: HashMap<String, String>,
    config: Config,
    logger: Logger,
    metrics: MetricsClient,
    // ... 20 more fields
}

// Every handler gets EVERYTHING
async fn get_user(State(state): State<AppState>) {
    // Can access ANY infrastructure
    state.db_pool.get().await;  // ❌ Handler doing queries
    state.redis.get().await;     // ❌ Handler using cache
    state.s3.put_object().await; // ❌ Handler using S3
}
```

**Why it's wrong:**
- Everything coupled to everything
- Handlers have too much access
- Can't enforce boundaries
- Testing requires all infrastructure
- Hard to understand what depends on what

### ✅ The Fix

```rust
// Slim AppState - services only
#[derive(Clone)]
struct AppState {
    user_service: Arc<UserService>,
    auth_service: Arc<AuthService>,
    subscription_service: Arc<SubscriptionService>,
}

// Services own dependencies
pub struct UserService {
    repo: Arc<UserRepository>,
    cache: Arc<CacheService>,
    email: Arc<EmailService>,
}

pub struct UserRepository {
    pool: DbPool,  // ✅ Hidden from handlers
}

// Handlers see services only
async fn get_user(State(user_service): State<Arc<UserService>>) {
    user_service.get_by_id(id).await  // ✅ Clean interface
}
```

## Anti-Pattern 5: Error Matching in Handlers

### ❌ The Problem

```rust
async fn get_user(/*...*/) -> Result<Json<User>, (StatusCode, String)> {
    match service.get_user(id).await {
        Ok(user) => Ok(Json(user)),
        Err(UserError::NotFound) => 
            Err((StatusCode::NOT_FOUND, "User not found".into())),
        Err(UserError::Deleted) => 
            Err((StatusCode::GONE, "User deleted".into())),
        Err(UserError::DatabaseError(e)) => 
            Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
        Err(UserError::InvalidId) => 
            Err((StatusCode::BAD_REQUEST, "Invalid ID".into())),
        // ❌ Repeated in EVERY handler
    }
}
```

**Why it's wrong:**
- Error mapping duplicated everywhere
- Easy to forget cases
- Inconsistent error responses
- Can't change mapping globally

### ✅ The Fix

```rust
// Implement IntoResponse once
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::User(UserError::NotFound(_)) => 
                (StatusCode::NOT_FOUND, self.to_string()),
            AppError::User(UserError::Deleted) => 
                (StatusCode::GONE, self.to_string()),
            AppError::User(UserError::InvalidId) => 
                (StatusCode::BAD_REQUEST, self.to_string()),
            AppError::Infra(_) => 
                (StatusCode::INTERNAL_SERVER_ERROR, 
                 "Internal server error".to_string()),
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}

// Handlers become simple
async fn get_user(/*...*/) -> Result<Json<UserResponse>, AppError> {
    let user = service.get_user(id).await?;  // ✅ Just use ?
    Ok(Json(user.into()))
}
```

## Anti-Pattern 6: Services Returning HTTP Types

### ❌ The Problem

```rust
// Service coupled to HTTP
pub async fn get_user(&self, id: Uuid) -> Result<Json<UserResponse>, StatusCode> {
    let user = self.repo.find_by_id(id).await
        .map_err(|_| StatusCode::NOT_FOUND)?;  // ❌ Service knows about HTTP
    
    Ok(Json(UserResponse::from(user)))  // ❌ Service creating JSON
}
```

**Why it's wrong:**
- Service can't be used from CLI
- Service can't be used from workers
- Can't change HTTP library
- Service doing serialization

### ✅ The Fix

```rust
// Service returns domain types
pub async fn get_user(&self, id: Uuid) -> Result<User, UserError> {
    self.repo.find_by_id(id).await
        .map_err(|_| UserError::NotFound(id))
}

// Handler does HTTP mapping
async fn get_user(/*...*/) -> Result<Json<UserResponse>, AppError> {
    let user = service.get_user(id).await?;
    Ok(Json(UserResponse::from(user)))  // ✅ Handler's job
}
```

## Anti-Pattern 7: Config Cloning in AppState

### ❌ The Problem

```rust
#[derive(Clone)]
struct AppState {
    config: Config,  // ❌ Cloned on every request!
}

async fn handler(State(state): State<AppState>) {
    let timeout = state.config.timeout;  // ❌ Config cloned
}
```

**Why it's wrong:**
- Unnecessary allocation
- Config doesn't change
- Memory waste

### ✅ The Fix

```rust
// Option 1: Static reference
lazy_static! {
    static ref CONFIG: Config = Config::from_env().unwrap();
}

pub struct UserService {
    repo: Arc<UserRepository>,
    config: &'static Config,  // ✅ No cloning
}

// Option 2: Arc once (if needed)
#[derive(Clone)]
struct AppState {
    services: Arc<Services>,  // ✅ One Arc
}

struct Services {
    user_service: UserService,
    config: Config,  // ✅ Not cloned per-request
}
```

## Anti-Pattern 8: No Separation Between Domain and Transport

### ❌ The Problem

```rust
// One type for everything
#[derive(Serialize, Deserialize, Queryable, Insertable)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub password: String,  // ❌ Plaintext in requests!
    pub password_hash: String,  // ❌ Exposed in responses!
}
```

**Why it's wrong:**
- Can't have different shapes for input/output
- Forced to expose all fields
- Can't version API
- Validation tied to database

### ✅ The Fix

```rust
// Domain model
#[derive(Queryable)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub password_hash: String,
}

// Request DTO
#[derive(Deserialize)]
pub struct CreateUserRequest {
    pub email: String,
    pub password: String,  // ✅ Plaintext allowed in input
}

// Response DTO
#[derive(Serialize)]
pub struct UserResponse {
    pub id: Uuid,
    pub email: String,
    // ✅ No password or password_hash
}
```

## Anti-Pattern 9: Middleware Doing Business Logic

### ❌ The Problem

```rust
pub async fn subscription_check<B>(
    request: Request<B>,
    next: Next<B>,
) -> Result<Response, AppError> {
    // ❌ Middleware doing database queries
    let user = fetch_user_from_db(&request).await?;
    let subscription = fetch_subscription(&user).await?;
    
    // ❌ Business logic in middleware
    if !subscription.is_active() {
        send_reminder_email(&user).await?;
        check_payment_method(&subscription).await?;
        attempt_charge(&subscription).await?;
        // 100 more lines...
    }
    
    Ok(next.run(request).await)
}
```

**Why it's wrong:**
- Middleware should be fast
- This blocks every request
- Business logic should be in service
- Hard to test

### ✅ The Fix

```rust
// Middleware - fast check only
pub async fn subscription_check<B>(
    State(auth_service): State<Arc<AuthService>>,
    mut request: Request<B>,
    next: Next<B>,
) -> Result<Response, AppError> {
    let user_id = auth_service.verify_token(&request).await?;
    
    // ✅ Just verify subscription exists
    let has_subscription = auth_service.has_active_subscription(user_id).await?;
    if !has_subscription {
        return Err(AppError::SubscriptionRequired);
    }
    
    request.extensions_mut().insert(user_id);
    Ok(next.run(request).await)
}

// Business logic in service
impl SubscriptionService {
    pub async fn handle_inactive(&self, subscription: &Subscription) {
        // ✅ Business logic here
        self.send_reminder().await;
        self.check_payment().await;
        self.attempt_charge().await;
    }
}
```

## Anti-Pattern 10: Testing Against Real Database

### ❌ The Problem

```rust
#[tokio::test]
async fn test_user_service() {
    // ❌ Setup real database
    let pool = setup_test_postgres().await;
    run_migrations(&pool).await;
    
    // ❌ Test requires real DB
    let service = UserService::new(pool);
    let user = service.create_user(data).await.unwrap();
    
    assert_eq!(user.email, "test@example.com");
    
    // ❌ Cleanup
    cleanup_database(&pool).await;
}
```

**Why it's wrong:**
- Tests are slow (100ms+ per test)
- Flaky (database connection issues)
- Requires Docker/Postgres
- Can't run in CI easily
- Tests not isolated

### ✅ The Fix

```rust
// Repository trait
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<User, InfraError>;
    async fn create(&self, user: NewUser) -> Result<User, InfraError>;
}

// Mock for testing
pub struct MockUserRepository {
    users: Arc<RwLock<HashMap<Uuid, User>>>,
}

#[async_trait]
impl UserRepository for MockUserRepository {
    async fn find_by_id(&self, id: Uuid) -> Result<User, InfraError> {
        self.users.read().await
            .get(&id)
            .cloned()
            .ok_or(InfraError::NotFound)
    }
    
    async fn create(&self, new_user: NewUser) -> Result<User, InfraError> {
        let user = User {
            id: Uuid::new_v4(),
            email: new_user.email,
            // ...
        };
        self.users.write().await.insert(user.id, user.clone());
        Ok(user)
    }
}

// Fast unit tests
#[tokio::test]
async fn test_user_service() {
    // ✅ No database needed
    let mock_repo = Arc::new(MockUserRepository::new());
    let service = UserService::new(mock_repo);
    
    let user = service.create_user(data).await.unwrap();
    assert_eq!(user.email, "test@example.com");
    
    // ✅ Fast (<1ms), no cleanup needed
}
```

## Summary

If you see these patterns, fix them immediately:

1. ❌ Database in handlers → ✅ Use repositories
2. ❌ Logic in handlers → ✅ Move to services
3. ❌ Diesel types in API → ✅ Separate DTOs
4. ❌ God AppState → ✅ Compose services
5. ❌ Error matching → ✅ IntoResponse
6. ❌ Services return HTTP → ✅ Return domain types
7. ❌ Clone Config → ✅ Use &'static
8. ❌ One type for all → ✅ Separate concerns
9. ❌ Middleware logic → ✅ Fast checks only
10. ❌ Real DB tests → ✅ Mock repositories

These anti-patterns compound - fix early before they become architectural debt.
