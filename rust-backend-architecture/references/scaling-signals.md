# Scaling & Maintainability Signals

Red flags that your architecture is breaking down as the codebase grows.

## Red Flag 1: Handler Bloat

### Symptoms

```rust
// 🚨 Handler is 200 lines
async fn create_subscription(/* ... */) -> Result</*...*/, AppError> {
    // 50 lines of validation
    // 50 lines of business logic
    // 50 lines of database queries
    // 50 lines of side effects
}
```

**When it appears**: Around 5,000-10,000 LOC

**Why it's bad:**
- Can't reuse logic
- Can't test without HTTP
- Changes ripple everywhere
- Becomes afraid to touch

### Fix

Extract to service layer:

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
        // All logic here
    }
}
```

## Red Flag 2: God Service

### Symptoms

```rust
// 🚨 One service does everything
pub struct AppService {
    // Handles users, posts, comments, subscriptions, 
    // emails, notifications, analytics, billing...
}

impl AppService {
    // 2000 lines of methods
}
```

**When it appears**: Around 10,000-20,000 LOC

**Why it's bad:**
- Impossible to understand
- Changes conflict (merge hell)
- Can't test parts in isolation
- Loading time increases

### Fix

Split by domain:

```rust
// Separate services per domain
pub struct UserService { /* ... */ }
pub struct SubscriptionService { /* ... */ }
pub struct BillingService { /* ... */ }
pub struct NotificationService { /* ... */ }
```

**Rule**: One service per aggregate root or bounded context.

## Red Flag 3: Circular Dependencies

### Symptoms

```rust
// 🚨 services/user_service.rs
use crate::services::post_service::PostService;

impl UserService {
    async fn delete_user(&self, id: Uuid) {
        self.post_service.delete_user_posts(id).await;  // Calls post service
    }
}

// 🚨 services/post_service.rs
use crate::services::user_service::UserService;

impl PostService {
    async fn create_post(&self, data: CreatePost) {
        let user = self.user_service.get_user(data.user_id).await;  // Calls user service
    }
}
```

**When it appears**: Around 15,000-30,000 LOC

**Why it's bad:**
- Compilation takes forever
- Refactoring impossible
- Tests setup nightmare
- Actual circular dependency in graph

### Fix

Extract shared logic:

```rust
// NEW: services/user_post_service.rs
pub struct UserPostService {
    user_repo: Arc<UserRepository>,
    post_repo: Arc<PostRepository>,
}

impl UserPostService {
    pub async fn delete_user_with_posts(&self, user_id: Uuid) {
        self.post_repo.delete_by_user(user_id).await;
        self.user_repo.delete(user_id).await;
    }
}
```

## Red Flag 4: State Explosion

### Symptoms

```rust
// 🚨 AppState has 30+ fields
#[derive(Clone)]
struct AppState {
    user_service: Arc<UserService>,
    post_service: Arc<PostService>,
    comment_service: Arc<CommentService>,
    subscription_service: Arc<SubscriptionService>,
    billing_service: Arc<BillingService>,
    notification_service: Arc<NotificationService>,
    analytics_service: Arc<AnalyticsService>,
    search_service: Arc<SearchService>,
    // ... 20 more services
    // Plus config, pools, clients...
}
```

**When it appears**: Around 20,000-40,000 LOC

**Why it's bad:**
- Every handler gets everything
- Can't enforce access control
- Testing requires all services
- Deployment coupling

### Fix

Group into domains:

```rust
#[derive(Clone)]
struct AppState {
    user_domain: Arc<UserDomain>,
    content_domain: Arc<ContentDomain>,
    billing_domain: Arc<BillingDomain>,
}

pub struct UserDomain {
    user_service: UserService,
    auth_service: AuthService,
    profile_service: ProfileService,
}
```

## Red Flag 5: DTOs Passing Between Layers

### Symptoms

```rust
// 🚨 DTO passing through layers
// handlers/users.rs
let dto = CreateUserDto::from(request);
let result = user_service.create(dto).await;  // ❌ DTO to service

// services/user_service.rs
pub async fn create(&self, dto: CreateUserDto) -> Result<UserDto, AppError> {
    let result = self.repo.create(dto).await;  // ❌ DTO to repository
}

// repositories/user_repository.rs
pub async fn create(&self, dto: CreateUserDto) -> Result<UserDto, AppError> {
    // ❌ Repository working with DTOs
}
```

**When it appears**: From day 1 if not careful

**Why it's bad:**
- API shape coupled to database
- Can't evolve API independently
- Business logic sees HTTP concerns
- Testing awkward

### Fix

DTOs at boundary only:

```rust
// handlers/users.rs
let data: CreateUserData = request.into();  // ✅ Convert to domain
let user = service.create(data).await?;     // ✅ Domain through layers
Ok(Json(UserResponse::from(user)))          // ✅ Convert back to DTO

// services/user_service.rs
pub async fn create(&self, data: CreateUserData) -> Result<User, UserError> {
    // ✅ Works with domain types
}
```

## Red Flag 6: Testing Requires Database

### Symptoms

```rust
// 🚨 Every test needs real database
#[tokio::test]
async fn test_user_service() {
    let pool = setup_test_database().await;  // ❌ Slow
    run_migrations(&pool).await;              // ❌ Flaky
    
    let service = UserService::new(pool);
    // Test takes 500ms
}
```

**When it appears**: Early if not using repository traits

**Why it's bad:**
- Tests are slow (minutes for full suite)
- Tests are flaky
- CI is slow
- Developers run tests less often

### Fix

Repository traits + mocks:

```rust
#[tokio::test]
async fn test_user_service() {
    let mock_repo = Arc::new(MockUserRepository::new());  // ✅ Fast
    let service = UserService::new(mock_repo);
    // Test takes <1ms
}
```

## Red Flag 7: Adding Feature Touches 10+ Files

### Symptoms

To add "user can favorite posts":

```diff
+ handlers/favorites.rs      # New handler
+ handlers/dto/favorite.rs   # New DTO
+ services/favorite_service.rs  # New service
+ repositories/favorite_repository.rs  # New repository
+ domain/favorite.rs         # New domain type
+ migrations/add_favorites.sql  # New migration
+ models/favorite.rs         # New Diesel model
+ schema.rs                  # Update schema
+ error.rs                   # Add errors
+ main.rs                    # Wire up service
# Plus updating 5 other files...
```

**When it appears**: Around 30,000+ LOC with poor boundaries

**Why it's bad:**
- High friction for new features
- Merge conflicts
- Easy to miss a step
- Fear of change

### Fix

Better boundaries and organization. Should touch 3-5 files max:

```diff
+ repositories/favorite_repository.rs  # Data access
+ services/user_service.rs            # Add favorite method (existing service)
+ handlers/users.rs                    # Add endpoint (existing handler)
+ migrations/add_favorites.sql         # Database
```

## Red Flag 8: Can't Find Anything

### Symptoms

- "Where does X live?"
- "Which service handles Y?"
- "Who calls this function?"
- Grep required for everything

**When it appears**: Around 40,000+ LOC

**Why it's bad:**
- Onboarding takes weeks
- Features take longer
- Bugs hard to find
- Technical debt accumulates

### Fix

Clear naming and organization:

```
services/
├── user_service.rs          # Everything user-related
├── subscription_service.rs  # Everything subscription-related
├── billing_service.rs       # Everything billing-related
```

**Rule**: If you can't guess where code lives, organization is wrong.

## Red Flag 9: Multiple People Can't Work Simultaneously

### Symptoms

- Constant merge conflicts
- Everyone editing same files
- Can't parallelize work
- Deployments break

**When it appears**: Around 50,000+ LOC with poor modularity

**Why it's bad:**
- Team velocity drops
- Frustration increases
- Quality decreases
- Coordination overhead

### Fix

Better domain boundaries:

```
Team A: User domain (users/, auth/)
Team B: Content domain (posts/, comments/)
Team C: Billing domain (subscriptions/, payments/)
```

Each team works in different directories.

## Red Flag 10: Fear of Changing Code

### Symptoms

- "This works, don't touch it"
- "I don't know what this does"
- "Tests will break"
- "Too risky to refactor"

**When it appears**: Any size if architecture is bad

**Why it's bad:**
- Technical debt compounds
- Velocity decreases over time
- Quality decreases
- Eventually need rewrite

### Fix

Good architecture prevents this:

- Clear boundaries
- Comprehensive tests
- Small, focused functions
- No hidden dependencies

## When to Refactor

### Immediate (Red Alert)

- Handler > 100 lines
- Circular dependencies
- No tests
- Everything in main.rs

### Soon (Yellow Alert)

- Handler 50-100 lines
- Service > 500 lines
- Testing requires database
- State > 10 fields

### Eventually (Monitor)

- Handler 30-50 lines
- Service 300-500 lines
- Adding feature touches 6-8 files

## Health Metrics

### Green (Healthy)

- main.rs: < 100 lines
- Handlers: 10-30 lines
- Services: 100-300 lines
- Tests: < 100ms each
- New feature: 3-5 files

### Yellow (Warning)

- main.rs: 100-200 lines
- Handlers: 30-50 lines
- Services: 300-500 lines
- Tests: 100-500ms each
- New feature: 5-8 files

### Red (Critical)

- main.rs: > 200 lines
- Handlers: > 50 lines
- Services: > 500 lines
- Tests: > 500ms each
- New feature: > 8 files

## Prevention

**Catch problems early:**

1. Code review checklist
2. Automated checks (file size, cyclic deps)
3. Regular architecture reviews
4. Refactor continuously
5. Boy scout rule (leave better than found)

## Summary

**If you see these signals, refactor immediately:**

1. 🚨 Handler bloat → Extract to service
2. 🚨 God service → Split by domain
3. 🚨 Circular deps → Extract shared logic
4. 🚨 State explosion → Group by domain
5. 🚨 DTOs everywhere → Use at boundary only
6. 🚨 Slow tests → Mock repositories
7. 🚨 Touch 10+ files → Better boundaries
8. 🚨 Can't find code → Better organization
9. 🚨 Merge conflicts → Better modularity
10. 🚨 Fear of change → Improve architecture

**Good architecture scales from 1,000 to 100,000 LOC without major rewrites.**
