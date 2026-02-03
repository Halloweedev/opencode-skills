# Complete Error Patterns and Solutions

This document contains detailed solutions for every compilation error in your Readr backend.

## Scope and Assumptions

**This guide fixes COMPILER errors only:**
- ✅ `cargo check` failures
- ✅ Type mismatches, missing modules, import errors
- ❌ NOT runtime bugs, logic errors, or performance issues

**Framework Versions:**
- **Axum 0.7+** - Uses `State` extractor (not `Extension`)
- **Diesel Async** - Uses `diesel-async` with async connections
- **Tokio** - Async runtime

If using older versions, patterns may differ.

## Your Specific Errors (11 Total)

### Error 1-3: Module Not Found (E0432, E0433)

```
error[E0432]: unresolved import `readr_backend::database`
error[E0433]: could not find `middleware` in `readr_backend`
error[E0432]: unresolved import `readr_backend::services::jwt_middleware`
```

**Root Cause**: Files exist but not declared in `lib.rs`

**Your File Structure**:
```
src/
├── lib.rs          ← Missing declarations
├── database.rs     ← Exists but not exported
├── middleware/     ← Directory exists but not exported
│   ├── auth.rs
│   ├── cors.rs
│   └── rate_limit.rs
└── services/
    └── middleware/ ← jwt_middleware here, not in services/
```

**Complete Fix for lib.rs**:
```rust
// src/lib.rs
pub mod config;
pub mod error;
pub mod schema;
pub mod database;     // ← ADD: Exports src/database.rs
pub mod middleware;   // ← ADD: Exports src/middleware/ directory
pub mod models;
pub mod routes;
pub mod services;
pub mod repositories;
pub mod utils;

// Re-exports for convenience
pub use config::Config;
pub use error::AppError;
pub use database::Database;  // ← ADD: Makes Database easily accessible
```

**Why This Fixes It**:
- `pub mod database` makes `readr_backend::database::Database` available
- `pub mod middleware` makes `readr_backend::middleware::cors::cors_layer` available
- Re-exports let you use `readr_backend::Database` instead of `readr_backend::database::Database`

### Error 4: jwt_middleware Location Wrong

```
error[E0432]: unresolved import `readr_backend::services::jwt_middleware`
  --> src/main.rs:23:30
23 | use readr_backend::services::jwt_middleware::jwt_middleware;
   |                              ^^^^^^^^^^^^^^ could not find `jwt_middleware` in `services`
```

**Problem**: `jwt_middleware` is in `src/services/middleware/` not `src/services/`

**Fix Option 1 - Update Import**:
```rust
// src/main.rs - CHANGE THIS LINE
use readr_backend::services::middleware::jwt_middleware::jwt_middleware;
//                            ^^^^^^^^^^^ ADD THIS
```

**Fix Option 2 - Move File** (Better):
```bash
# Move jwt_middleware to correct location
mv src/services/middleware/jwt_middleware.rs src/middleware/jwt_middleware.rs
```

Then update `src/middleware/mod.rs`:
```rust
// src/middleware/mod.rs (create if doesn't exist)
pub mod auth;
pub mod cors;
pub mod rate_limit;
pub mod jwt_middleware;  // ← ADD
```

Then import becomes:
```rust
// src/main.rs
use readr_backend::middleware::jwt_middleware::jwt_middleware;
```

### Error 5: Variable Not in Scope

```
error[E0425]: cannot find value `database` in this scope
  --> src/main.rs:96:53
96 |         .route("/health", move |_: ()| health_check(database.clone()))
   |                                                     ^^^^^^^^ not found in this scope
```

**Problem**: `database` is moved into `app_state`, then trying to use it again

**Your Code**:
```rust
let app_state = AppState { 
    config: config.clone(),
    database,  // ← database moved here
};

// Later:
.route("/health", move |_: ()| health_check(database.clone()))
                                            ^^^^^^^^ ← Can't use, already moved!
```

**Fix**: Use State extractor pattern

```rust
// ❌ DELETE THIS WRONG PATTERN
.route("/health", move |_: ()| health_check(database.clone()))

// ✅ ADD THIS CORRECT PATTERN
.route("/health", get(health_check))

// Update health_check signature:
async fn health_check(
    State(database): State<Database>  // ← Extract from router state
) -> Result<Json<serde_json::Value>, (StatusCode, String)> {
    match database.health_check().await {
        Ok(_) => Ok(Json(json!({ "status": "healthy", "database": "connected" }))),
        Err(e) => Err((
            StatusCode::SERVICE_UNAVAILABLE,
            format!("Database unhealthy: {}", e)
        ))
    }
}
```

### Error 6: Diesel Table Import

```
error[E0425]: cannot find value `users` in module `diesel::dsl`
   --> src/main.rs:130:58
130 |     match diesel::select(diesel::dsl::count(diesel::dsl::users))
    |                                                          ^^^^^ not found in `diesel::dsl`
```

**Problem**: Diesel tables are in your schema, not in `diesel::dsl`

**Fix**:
```rust
// ❌ WRONG - users is not in diesel::dsl
use diesel::prelude::*;
diesel::select(diesel::dsl::count(diesel::dsl::users))

// ✅ CORRECT - Import from your schema
use diesel::prelude::*;
use crate::schema::users::dsl::*;  // ← Imports 'users' table

diesel::select(count(users))  // ← Now users is available
```

**Better: Move to Database Method**:
```rust
// src/database.rs
impl Database {
    pub async fn health_check(&self) -> Result<()> {
        use diesel::prelude::*;
        use diesel_async::RunQueryDsl;
        use crate::schema::users::dsl::*;  // ← Import here
        
        let mut conn = self.pool.get().await?;
        
        let _count: i64 = users
            .count()
            .get_result(&mut conn)
            .await?;
        
        Ok(())
    }
}

// src/main.rs - Now simple
async fn health_check(
    State(database): State<Database>
) -> Result<Json<Value>, (StatusCode, String)> {
    database.health_check().await
        .map(|_| Json(json!({ "status": "healthy" })))
        .map_err(|e| (StatusCode::SERVICE_UNAVAILABLE, e.to_string()))
}
```

### Error 7-8: Type Mismatches in Routing

```
error[E0308]: mismatched types
96 |         .route("/health", move |_: ()| health_check(database.clone()))
   |          -----            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 
   |                           expected `MethodRouter<_>`, found closure

error[E0308]: mismatched types
99 |         .with_state(state)
   |          ---------- ^^^^^ expected `()`, found `AppState`
```

**Problem 1**: Axum routing expects `MethodRouter`, not raw closures
**Problem 2**: Router has no state generic but `.with_state()` is called

**Complete Fix**:
```rust
// ❌ WRONG - Your current code
fn create_app(state: AppState) -> Router {
    Router::new()
        .route("/health", move |_: ()| health_check(database.clone()))
        .with_state(state)  // ← Router<()> doesn't accept state
}

// ✅ CORRECT - Fixed version
fn create_app(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health_check))  // ← Use get() wrapper
        .nest("/api/v1", api_routes())        // ← Sub-routes
        .with_state(state)                    // ← Now Router<AppState>
        .layer(TraceLayer::new_for_http())
}

// Handler with State extraction
async fn health_check(
    State(AppState { database, .. }): State<AppState>
) -> Result<Json<Value>, (StatusCode, String)> {
    database.health_check().await
        .map(|_| Json(json!({ "status": "healthy" })))
        .map_err(|e| (StatusCode::SERVICE_UNAVAILABLE, e.to_string()))
}
```

**Key Changes**:
1. Use `get(health_check)` not closures
2. Extract state with `State(app_state): State<AppState>`
3. Router becomes `Router<AppState>` when you call `.with_state(state)`

### Error 9: Wrong Async Return Type

```
error[E0277]: the `?` operator can only be used in an async function 
              that returns `Result` or `Option`
131 |  .first::<i64>(&mut database.get_pool().get().await?)
    |                                                     ^ 
```

**Problem**: Function returns `impl IntoResponse` but tries to use `?`

**Your Code**:
```rust
async fn health_check(database: Database) -> impl IntoResponse {
    let result = query.first(&mut pool.get().await?)?;  // ← Can't use ?
    Response::new(...)
}
```

**Fix - Return Result**:
```rust
// ✅ CORRECT - Return Result type
async fn health_check(
    State(database): State<Database>
) -> Result<Json<Value>, (StatusCode, String)> {
    // Now ? operator works
    let mut conn = database.get_pool().get().await
        .map_err(|e| (StatusCode::SERVICE_UNAVAILABLE, e.to_string()))?;
    
    let count: i64 = users
        .count()
        .get_result(&mut conn)
        .await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
    
    Ok(Json(json!({
        "status": "healthy",
        "user_count": count
    })))
}

// Note: Error type is illustrative - use your project's error type.
// Common patterns:
// - Result<Json<T>, AppError> (custom error with IntoResponse)
// - Result<Json<T>, (StatusCode, String)> (quick and simple)
// - Result<Response, AppError> (for full response control)
```

**Or with AppError**:
```rust
async fn health_check(
    State(database): State<Database>
) -> Result<Json<Value>, AppError> {
    database.health_check().await?;  // ← ? works with AppError
    Ok(Json(json!({ "status": "healthy" })))
}

// Make sure AppError implements IntoResponse
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::DatabaseError(e) => 
                (StatusCode::SERVICE_UNAVAILABLE, e.to_string()),
            AppError::NotFound => 
                (StatusCode::NOT_FOUND, "Not found".to_string()),
            // ... other variants
        };
        
        (status, Json(json!({ "error": message }))).into_response()
    }
}
```

### Error 10-11: Type Annotations Needed

```
error[E0282]: type annotations needed
166 |                 "error": e.to_string()
    |                          ^ cannot infer type

error[E0282]: type annotations needed
 45 |     let database = Database::new(&config).await
    |                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot infer type
```

**Problem**: Rust can't infer the error type in generic context

**Fix for Error 166**:
```rust
// ❌ WRONG - Type inference fails in json! macro
Err(e) => Json(json!({
    "error": e.to_string()  // ← What's the type of e?
}))

// ✅ CORRECT - Explicitly handle error
Err(e) => {
    let error_msg: String = e.to_string();  // ← Explicit type
    Json(json!({
        "error": error_msg
    }))
}

// OR - Use a match with concrete types
match result {
    Ok(data) => Ok(Json(json!({ "data": data }))),
    Err(AppError::DatabaseError(e)) => {
        Err((StatusCode::SERVICE_UNAVAILABLE, e.to_string()))
    }
}
```

**Fix for Error 45**:
```rust
// ❌ WRONG - Can't infer Database type
let database = Database::new(&config).await
    .map_err(|e| anyhow::anyhow!("Failed: {}", e))?;

// ✅ CORRECT - Add explicit type
let database: Database = Database::new(&config).await
    .map_err(|e| anyhow::anyhow!("Failed to initialize database: {}", e))?;

// OR - Annotate the variable
let database = Database::new(&config).await
    .map_err(|e| anyhow::anyhow!("Failed: {}", e))?;
```

## Complete Working main.rs Structure

Here's how your `main.rs` should look after all fixes:

```rust
use axum::{
    extract::State,
    http::StatusCode,
    response::IntoResponse,
    routing::get,
    Json, Router,
};
use serde_json::{json, Value};
use std::net::SocketAddr;
use tower_http::trace::TraceLayer;
use tracing::info;
use tracing_subscriber::EnvFilter;

use readr_backend::{Config, Database, AppError};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Initialize tracing
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("info")))
        .init();

    let config = Config::from_env()?;
    info!("Starting server on port {}", config.server.port);

    let database: Database = Database::new(&config).await?;
    
    let app_state = AppState { 
        config: config.clone(),
        database,
    };
    
    let app = create_app(app_state);
    let addr = SocketAddr::from(([0, 0, 0, 0], config.server.port));
    let listener = tokio::net::TcpListener::bind(addr).await?;

    info!("Listening on {}", addr);
    axum::serve(listener, app).await?;
    Ok(())
}

#[derive(Clone)]
struct AppState {
    config: Config,
    database: Database,
}

fn create_app(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health_check))
        // Add more routes here
        .with_state(state)  // ← Mandatory when handlers use State<AppState>
        .layer(TraceLayer::new_for_http())
}

// Handler with State extraction (Axum 0.7+ pattern)
async fn health_check(
    State(AppState { database, .. }): State<AppState>
) -> Result<Json<Value>, (StatusCode, String)> {
    database.health_check().await
        .map(|_| Json(json!({ "status": "healthy", "database": "connected" })))
        .map_err(|e| (StatusCode::SERVICE_UNAVAILABLE, format!("Database error: {}", e)))
}

// ⚠️ IMPORTANT: If health_check didn't use State<AppState>, then:
// 1. Don't call .with_state(state) on the router
// 2. OR handler will fail to compile
//
// The handler signature determines if .with_state() is required.
```

## Summary of All Fixes

| Error | Root Cause | Fix |
|-------|-----------|-----|
| E0432 database | Missing `pub mod database` | Add to lib.rs |
| E0433 middleware | Missing `pub mod middleware` | Add to lib.rs |
| E0432 jwt_middleware | Wrong path | Move file or fix import |
| E0425 database | Variable moved | Use State extractor |
| E0425 users | Wrong Diesel import | Import from schema |
| E0308 routing | Closure instead of handler | Use get(fn_name) |
| E0308 with_state | Type mismatch | Match state type |
| E0277 async ? | Wrong return type | Return Result |
| E0282 type inference | Generic ambiguity | Add explicit types |

All 11 errors stem from 4 root causes:
1. Module visibility (`lib.rs`)
2. Axum patterns (routing, state)
3. Async return types
4. Diesel imports

Fix in this order for fastest resolution.
