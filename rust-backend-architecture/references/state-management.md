# State Management in Axum

Complete guide on managing state in Axum 0.7+ applications.

## What Goes in AppState

**Rule**: AppState contains services only, not raw infrastructure.

### ❌ Bad - Raw Infrastructure

```rust
#[derive(Clone)]
struct AppState {
    db_pool: DbPool,              // ❌ Too low-level
    redis: RedisPool,              // ❌ Too low-level
    s3_client: S3Client,          // ❌ Infrastructure detail
    jwt_secret: String,            // ❌ Config cloned
    config: Config,                // ❌ Cloned on every request
}
```

**Problems:**
- Handlers can access database directly
- Everything coupled
- Testing requires all infrastructure
- Unnecessary cloning

### ✅ Good - Services Only

```rust
#[derive(Clone)]
struct AppState {
    user_service: Arc<UserService>,
    auth_service: Arc<AuthService>,
    subscription_service: Arc<SubscriptionService>,
}
```

**Benefits:**
- Handlers can't bypass service layer
- Clear boundaries
- Easy to test
- Minimal cloning

## Arc Usage Rules

### Rule 1: One Arc per Service

```rust
// ❌ WRONG - Double Arc
#[derive(Clone)]
struct AppState {
    user_service: Arc<Arc<UserService>>,  // ❌ Unnecessary nesting
}

// ✅ CORRECT - One Arc
#[derive(Clone)]
struct AppState {
    user_service: Arc<UserService>,  // ✅ One layer
}
```

### Rule 2: Config Doesn't Need Arc

```rust
// ❌ WRONG - Config in Arc
#[derive(Clone)]
struct AppState {
    config: Arc<Config>,  // ❌ Config doesn't change
}

// ✅ CORRECT - Static config
lazy_static! {
    static ref CONFIG: Config = Config::from_env().unwrap();
}

pub struct UserService {
    repo: Arc<UserRepository>,
    config: &'static Config,  // ✅ Static reference
}
```

### Rule 3: Read-Only Data Can Be Static

```rust
// ❌ WRONG - Cloning constants
#[derive(Clone)]
struct AppState {
    api_keys: Arc<HashMap<String, String>>,  // ❌ Doesn't change
}

// ✅ CORRECT - Static data
lazy_static! {
    static ref API_KEYS: HashMap<String, String> = load_api_keys();
}

pub struct AuthService {
    api_keys: &'static HashMap<String, String>,  // ✅ No cloning
}
```

### Rule 4: Mutable State Needs RwLock/Mutex

```rust
// ✅ CORRECT - Shared mutable state
#[derive(Clone)]
struct AppState {
    cache: Arc<RwLock<Cache>>,  // ✅ Read-heavy → RwLock
    counter: Arc<Mutex<Counter>>,  // ✅ Write-heavy → Mutex
}
```

## Service Composition

### Services Own Dependencies

```rust
pub struct UserService {
    repo: Arc<UserRepository>,      // ✅ Repository
    email: Arc<EmailService>,        // ✅ External service
    cache: Arc<RwLock<UserCache>>,  // ✅ Shared cache
    config: &'static Config,         // ✅ Static config
}

impl UserService {
    pub fn new(
        repo: Arc<UserRepository>,
        email: Arc<EmailService>,
    ) -> Self {
        UserService {
            repo,
            email,
            cache: Arc::new(RwLock::new(UserCache::new())),
            config: &CONFIG,
        }
    }
}
```

**Rule**: Services encapsulate their dependencies. AppState doesn't know about repositories or caches.

## Clone vs Borrow

### When to Clone Arc

```rust
// ✅ GOOD - Clone when spawning tasks
async fn handler(State(service): State<Arc<UserService>>) {
    let service = service.clone();  // ✅ Moving into task
    tokio::spawn(async move {
        service.background_task().await;
    });
}

// ✅ GOOD - Clone when storing for later
async fn handler(State(service): State<Arc<UserService>>) {
    let service = service.clone();  // ✅ Storing in struct
    MyStruct { service }
}
```

### When NOT to Clone Arc

```rust
// ❌ BAD - Unnecessary clone
async fn handler(State(service): State<Arc<UserService>>) {
    let service_clone = service.clone();  // ❌ Why?
    service_clone.get_user(id).await  // Could just use service
}

// ✅ GOOD - Use directly
async fn handler(State(service): State<Arc<UserService>>) {
    service.get_user(id).await  // ✅ Arc auto-derefs
}
```

**Rule**: Arc clones are cheap (just incrementing refcount), but don't clone unless you need to move it.

## Dependency Injection Patterns

### Constructor Injection

```rust
// ✅ CORRECT - Dependencies passed in constructor
impl UserService {
    pub fn new(
        repo: Arc<UserRepository>,
        email: Arc<EmailService>,
    ) -> Self {
        UserService { repo, email, config: &CONFIG }
    }
}

// In main.rs
let user_repo = Arc::new(UserRepository::new(pool));
let email = Arc::new(EmailService::new(&config));
let user_service = Arc::new(UserService::new(user_repo, email));
```

### ❌ Service Locator Anti-Pattern

```rust
// ❌ WRONG - Global service registry
lazy_static! {
    static ref SERVICES: ServiceRegistry = ServiceRegistry::new();
}

impl UserService {
    pub async fn create_user(&self, data: CreateUserData) -> Result<User, UserError> {
        // ❌ Getting dependencies from global
        let email = SERVICES.get::<EmailService>();  // ❌ Hidden dependency
        email.send_welcome().await?;
    }
}
```

**Problem**: Hidden dependencies, hard to test, global state.

## State Extraction Patterns

### Basic Extraction

```rust
async fn handler(
    State(user_service): State<Arc<UserService>>,
) -> Result<Json<UserResponse>, AppError> {
    user_service.get_user(id).await
}
```

### Multiple State Items

```rust
async fn handler(
    State(AppState { user_service, auth_service, .. }): State<AppState>,
) -> Result<Json<UserResponse>, AppError> {
    let user_id = auth_service.verify_token(&token).await?;
    user_service.get_user(user_id).await
}
```

### State + Other Extractors

```rust
async fn handler(
    State(user_service): State<Arc<UserService>>,
    Path(id): Path<Uuid>,
    Json(payload): Json<UpdateUserRequest>,
) -> Result<Json<UserResponse>, AppError> {
    user_service.update(id, payload.into()).await
}
```

## Request Context Pattern

### Using Extensions for Request-Scoped Data

```rust
// Middleware sets context
pub async fn auth_middleware<B>(
    State(auth): State<Arc<AuthService>>,
    mut request: Request<B>,
    next: Next<B>,
) -> Result<Response, AppError> {
    let token = extract_token(&request)?;
    let user_id = auth.verify_token(token).await?;
    
    // ✅ Store in extensions
    request.extensions_mut().insert(RequestContext {
        user_id,
        correlation_id: Uuid::new_v4(),
    });
    
    Ok(next.run(request).await)
}

// Handler extracts context
async fn handler(
    Extension(ctx): Extension<RequestContext>,
    State(service): State<Arc<UserService>>,
) -> Result<Json<UserResponse>, AppError> {
    service.get_user(ctx.user_id).await
}
```

### What Goes in Extensions

**✅ Request-scoped data:**
- User ID from authentication
- Correlation ID for tracing
- IP address
- Request start time

**❌ NOT for:**
- Application config
- Database connections
- Services (use State for these)

## Common Mistakes

### Mistake 1: Everything in AppState

```rust
// ❌ BAD
#[derive(Clone)]
struct AppState {
    db_pool: DbPool,
    redis: RedisPool,
    s3: S3Client,
    config: Config,
    // ... 20 more things
}
```

**Fix**: Only services in AppState.

### Mistake 2: Cloning Everywhere

```rust
// ❌ BAD
async fn handler(State(service): State<Arc<UserService>>) {
    let s1 = service.clone();
    let s2 = service.clone();
    let s3 = service.clone();
    s1.do_thing().await;
}
```

**Fix**: Arc auto-derefs, just use `service` directly.

### Mistake 3: Mutable State Without Lock

```rust
// ❌ BAD - Data race!
#[derive(Clone)]
struct AppState {
    counter: Arc<u64>,  // ❌ No lock = undefined behavior
}
```

**Fix**: Use `Arc<Mutex<u64>>` or `Arc<RwLock<u64>>`.

### Mistake 4: Config in AppState

```rust
// ❌ BAD
#[derive(Clone)]
struct AppState {
    config: Config,  // ❌ Cloned per request
}
```

**Fix**: Use `&'static Config` or `Arc<Config>` once.

## Testing with State

### Mocking Services

```rust
// Create mock state for testing
#[cfg(test)]
mod tests {
    use super::*;
    
    #[tokio::test]
    async fn test_handler() {
        let mock_service = Arc::new(MockUserService::new());
        let state = AppState {
            user_service: mock_service.clone(),
        };
        
        let response = handler(State(mock_service), Path(user_id))
            .await
            .unwrap();
        
        // Assert response
    }
}
```

## Summary

**State Rules:**
1. AppState contains `Arc<Service>` only
2. Services contain `Arc<Repository>` and clients
3. One Arc per service, not double-wrapped
4. Config is `&'static` or `Arc` once
5. Mutable state needs `RwLock`/`Mutex`
6. Clone Arc only when moving into tasks
7. Use Extensions for request-scoped data

**If state is growing unbounded, you're doing it wrong.**
