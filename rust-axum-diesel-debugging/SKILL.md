---
name: rust-axum-diesel-debugging
description: Diagnose and fix Rust backend compiler errors in Axum + Diesel projects by mapping error codes (E0432, E0308, E0277) to canonical fixes. Handles module visibility, routing patterns, and async types. Not for runtime bugs or logic errors.
license: MIT
compatibility: opencode
metadata:
  language: rust
  frameworks: axum-0.7, diesel-async
  domain: backend-web
  expertise: intermediate-advanced
---

# Rust Backend Debugging Expert

I fix common Rust backend compilation errors in Axum, Diesel, and async Rust projects. I identify root causes and provide complete, working solutions.

## Assumptions

**Framework Versions:**
- **Axum 0.7+** - Uses `State` extractor, not `Extension` (deprecated)
- **Diesel Async** - Uses `diesel-async` with `AsyncPgConnection` / `bb8` / `deadpool`
- **Tokio** - Async runtime for all async operations

**Important:** If you're using Axum 0.6 or sync Diesel, patterns will differ.

## What I Do

- ✅ Fix **compiler errors** (E0432, E0308, E0277, E0433, E0425, E0282)
- ✅ Map error codes to canonical fixes
- ✅ Diagnose module visibility issues
- ✅ Correct Axum routing patterns
- ✅ Fix async return type mismatches
- ✅ Resolve Diesel import problems

## What I Do NOT Handle

- ❌ Runtime panics or crashes
- ❌ Business logic bugs
- ❌ Performance optimization
- ❌ SQL query correctness (beyond imports/types)
- ❌ Authentication/authorization logic
- ❌ Database schema design
- ❌ Production deployment issues

**This skill focuses on making `cargo check` pass, not making your app correct.**

## Critical Anti-Patterns

**Don't do these - they hide real problems:**

### ❌ Anti-Pattern 1: Clone Everything to Fix Borrow Checker
```rust
// ❌ WRONG - Masking ownership issues
let db = database.clone();
let db2 = database.clone();
let db3 = database.clone();
// If you need this many clones, your architecture is wrong
```

**Fix**: Use `State` extractor or Arc once at the boundary.

### ❌ Anti-Pattern 2: Closures to Bypass State Typing
```rust
// ❌ WRONG - Avoiding proper State extraction
.route("/user", move |_: ()| {
    get_user(db.clone())  // Capture in closure to avoid State
})
```

**Fix**: Use proper handler with `State<T>` extraction.

### ❌ Anti-Pattern 3: `impl IntoResponse` Everywhere
```rust
// ❌ WRONG - Avoiding Result just to skip error handling
async fn handler() -> impl IntoResponse {
    match do_thing() {
        Ok(x) => Json(x).into_response(),
        Err(e) => (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()).into_response()
    }
}
```

**Fix**: Return `Result<Json<T>, AppError>` and use `?` operator.

### ❌ Anti-Pattern 4: Wildcard Diesel Imports
```rust
// ❌ WRONG - Importing everything from DSL
use diesel::dsl::*;  // Conflicts with your schema
```

**Fix**: Import specific items or use schema modules:
```rust
use crate::schema::users::dsl::*;  // ✅ Table-specific
```

### ❌ Anti-Pattern 5: Empty State to Silence Errors
```rust
// ❌ WRONG - Adding with_state(()) to compile
.with_state(())  // "It compiles now!"
```

**Fix**: Either don't use `with_state` at all, or pass real state that handlers need.

**Remember**: If a fix feels hacky, it probably is. Compiler errors point to architectural issues, not nuisances to bypass.

## When to use me

Use this skill when:
- `cargo check` fails with module not found errors (E0432, E0433)
- Axum routing doesn't compile (E0308 type mismatches)
- Async function return types are wrong (E0277)
- Diesel queries fail to import correctly
- State management in Axum handlers breaks
- You see "could not find X in Y" errors

## Critical Error Patterns

See [references/error-patterns.md](references/error-patterns.md) for complete solutions to:

1. **E0432: Module not found** - `lib.rs` missing `pub mod`
2. **E0308: Type mismatch in routing** - Wrong Axum routing pattern
3. **E0277: Wrong async return type** - Missing `Result` in signature
4. **E0425: Value not in scope** - Variable shadowing or missing import
5. **Diesel import errors** - Schema imports not correct

## Quick Diagnostic Guide

### Error Type 1: "could not find `database` in `readr_backend`"

**Problem**: Module exists as file but not exported in `lib.rs`

```rust
// ❌ WRONG - lib.rs
pub mod config;
pub mod error;
// database.rs exists but not declared!

// ✅ CORRECT - lib.rs
pub mod config;
pub mod error;
pub mod database;  // ← ADD THIS

pub use database::Database;  // Optional: re-export
```

**Rule**: Every module file/directory needs `pub mod` in `lib.rs`

### Error Type 2: "expected `MethodRouter`, found closure"

**Problem**: Axum `.route()` needs handlers wrapped in `get()`, `post()`, etc.

```rust
// ❌ WRONG
.route("/health", move |_: ()| health_check(database.clone()))

// ✅ CORRECT - Use get() wrapper
.route("/health", get(health_check))
```

**Rule**: Handlers must be wrapped in routing methods

### Error Type 3: "the `?` operator can only be used in... `Result`"

**Problem**: Function returns `impl IntoResponse` but uses `?` operator

```rust
// ❌ WRONG
async fn health_check(database: Database) -> impl IntoResponse {
    let result = query.first(&mut pool.get().await?)?;  // ❌ Can't use ?
    // ...
}

// ✅ CORRECT - Return Result
async fn health_check(
    State(database): State<Database>
) -> Result<impl IntoResponse, AppError> {
    let result = query.first(&mut pool.get().await?)?;  // ✅ Now works
    Ok(Json(json!({ "status": "healthy" })))
}
```

**Rule**: Use `?` only in functions returning `Result` or `Option`

### Error Type 4: "expected `()`, found `AppState`"

**Problem**: Router state type doesn't match

```rust
// ❌ WRONG - Router has no generic, but with_state called
let app = Router::new()
    .route("/health", get(health_check))
    .with_state(state);  // ❌ Router<()> doesn't accept state

// ✅ CORRECT - Don't call with_state if no handlers use State
let app = Router::new()
    .route("/health", get(health_check));  // ✅ No state needed

// OR if handlers DO use State:
async fn handler(State(db): State<Database>) -> String {
    "ok".to_string()
}

let app = Router::new()
    .route("/api", get(handler))
    .with_state(database);  // ✅ Now Router<Database>
```

**Rule**: Only use `.with_state()` if handlers extract `State<T>`

### Error Type 5: "could not find `users` in `diesel::dsl`"

**Problem**: Diesel schema not imported correctly

```rust
// ❌ WRONG
use diesel::prelude::*;
diesel::select(diesel::dsl::count(diesel::dsl::users))  // ❌ users is not in dsl

// ✅ CORRECT - Import from schema
use diesel::prelude::*;
use crate::schema::users::dsl::*;  // Imports users table

diesel::select(count(users))  // ✅ Works
```

**Rule**: Diesel tables come from `schema::table_name::dsl::*`

## Complete Fix Examples

### Example 1: Module Not Found

**Error**:
```
error[E0432]: unresolved import `readr_backend::database`
  --> src/main.rs:15:20
15 | use readr_backend::database::Database;
   |                    ^^^^^^^^ could not find `database` in `readr_backend`
```

**Root Cause**: `src/database.rs` exists but `src/lib.rs` doesn't declare it

**Fix**:
```rust
// src/lib.rs - ADD THESE LINES
pub mod database;  // ← Declare the module
pub mod middleware;  // ← If you have src/middleware/ directory

pub use database::Database;  // ← Optional re-export for convenience
```

### Example 2: Axum Routing Type Mismatch

**Error**:
```
error[E0308]: mismatched types
96 |         .route("/health", move |_: ()| health_check(database.clone()))
   |          -----            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 
   |          |                expected `MethodRouter<_>`, found closure
```

**Root Cause**: Axum routing expects wrapped handler functions, not closures

**Fix**:
```rust
// ❌ WRONG
.route("/health", move |_: ()| health_check(database.clone()))

// ✅ CORRECT - Pass handler directly with State extraction
async fn health_check(
    State(database): State<Database>
) -> Result<Json<serde_json::Value>, AppError> {
    database.health_check().await?;
    Ok(Json(json!({ "status": "healthy" })))
}

// In router setup:
let app = Router::new()
    .route("/health", get(health_check))
    .with_state(database);  // Provides State<Database> to handlers
```

### Example 3: Async Return Type with `?`

**Error**:
```
error[E0277]: the `?` operator can only be used in an async function 
              that returns `Result` or `Option`
131 |  .first::<i64>(&mut database.get_pool().get().await?)
    |                                                     ^ 
    |  this function should return `Result` or `Option` to accept `?`
```

**Fix**:
```rust
// ❌ WRONG
async fn health_check(database: Database) -> impl IntoResponse {
    let result = query.first(&mut pool.get().await?)?;  // Can't use ?
    Response::new(...)
}

// ✅ CORRECT - Return Result
async fn health_check(
    State(database): State<Database>
) -> Result<Json<Value>, (StatusCode, String)> {
    let result = query.first(&mut pool.get().await?)?;  // ✅ Works
    Ok(Json(json!({ "count": result })))
}

// Or with custom error type:
async fn health_check(
    State(database): State<Database>
) -> Result<Json<Value>, AppError> {
    database.health_check().await?;  // ✅ Works with ?
    Ok(Json(json!({ "status": "healthy" })))
}
```

## Systematic Debugging Process

When `cargo check` fails:

### Step 1: Identify Error Category

```bash
cargo check 2>&1 | grep "error\[E"
```

- **E0432/E0433**: Module visibility issues → Check `lib.rs`
- **E0308**: Type mismatch → Check Axum routing patterns
- **E0277**: Trait not satisfied → Check async return types
- **E0425**: Value not found → Check variable scope/imports
- **E0282**: Type inference → Add explicit types

### Step 2: Fix in Order

1. **Module errors first** - Fix `lib.rs` exports
2. **Import errors** - Add missing `use` statements
3. **Type errors** - Fix function signatures
4. **Logic errors** - Fix implementation

### Step 3: Verify Fix

```bash
cargo check  # Should show fewer errors
cargo clippy  # Check for warnings
cargo build  # Full compilation
```

## Common Patterns Reference

See [references/common-patterns.md](references/common-patterns.md) for:
- Axum handler patterns
- State management
- Error handling with AppError
- Diesel query patterns
- Middleware setup

## Module Organization Checklist

When adding new modules:

- [ ] Create the file/directory (`src/database.rs` or `src/database/mod.rs`)
- [ ] Add `pub mod module_name;` to `src/lib.rs`
- [ ] Add `pub use module_name::Type;` if re-exporting
- [ ] Import in `main.rs` with `use crate_name::module_name::Type;`
- [ ] Run `cargo check` to verify

## State Management in Axum

**Axum 0.7+ State Pattern** (Current Standard):

Axum 0.7 deprecated `Extension` in favor of the `State` extractor for better type safety and ergonomics.

**Three ways to pass state:**

### 1. Extension (Deprecated in Axum 0.7+)
```rust
// ❌ OLD WAY - Don't use in Axum 0.7+
.layer(Extension(database))

// This still works but is deprecated
// Prefer State extractor below
```

### 2. State Extractor (Recommended for Axum 0.7+)
```rust
// ✅ CORRECT - Axum 0.7+ preferred pattern
async fn handler(State(db): State<Database>) -> String {
    "ok".to_string()
}

let app = Router::new()
    .route("/", get(handler))
    .with_state(database);  // ← Provides State<Database> to handlers

// Key points:
// - Handler signature determines state type
// - .with_state(T) makes Router<T>
// - All handlers on this router can extract State<T>
```

### 3. Closure Capture (For Simple Cases)
```rust
// ✅ CORRECT - For routes without extractors
let db = database.clone();
let app = Router::new()
    .route("/health", get(|| async move {
        db.health_check().await.is_ok().to_string()
    }));
// ✅ No .with_state() needed - handler doesn't use State<T>

// ⚠️ WARNING: If you later change the handler to use State<T>:
async fn health_check(State(db): State<Database>) -> String {
    db.health_check().await.is_ok().to_string()
}
// Then you MUST add .with_state(database) or compilation breaks:
let app = Router::new()
    .route("/health", get(health_check))
    .with_state(database);  // ← Now mandatory
```

**Rule**: Handler signature determines if `.with_state()` is needed. If handler adds `State<T>` later, `.with_state(T)` becomes mandatory.

## Diesel Query Patterns

**Diesel Async Configuration:**
- Uses `diesel-async` crate (not sync `diesel`)
- Connection types: `AsyncPgConnection` (PostgreSQL) or `AsyncMysqlConnection` (MySQL)
- Pool types: `bb8::Pool` or `deadpool::Pool`
- Query trait: `diesel_async::RunQueryDsl` (not sync `diesel::RunQueryDsl`)

**If you're using sync Diesel**, these patterns won't work - you need blocking connections.

```rust
// ✅ CORRECT - Full async pattern with diesel-async
use diesel::prelude::*;
use diesel_async::RunQueryDsl;  // ← Async version
use crate::schema::users::dsl::*;  // ← Import table

async fn get_user_count(pool: &DbPool) -> Result<i64, AppError> {
    let mut conn = pool.get().await?;  // ← Async get connection
    
    let count = users
        .count()
        .get_result::<i64>(&mut conn)
        .await?;  // ← Async query execution
    
    Ok(count)
}

// Connection pool type
type DbPool = bb8::Pool<diesel_async::AsyncPgConnection>;
// OR
type DbPool = deadpool::Pool<diesel_async::AsyncPgConnection>;
```

**Common Diesel Async Patterns:**

```rust
// SELECT with filter
let user = users
    .filter(email.eq(user_email))
    .first::<User>(&mut conn)
    .await?;

// INSERT
diesel::insert_into(users)
    .values(&new_user)
    .execute(&mut conn)
    .await?;

// UPDATE
diesel::update(users.find(user_id))
    .set(name.eq(new_name))
    .execute(&mut conn)
    .await?;

// DELETE
diesel::delete(users.find(user_id))
    .execute(&mut conn)
    .await?;
```

## Testing Your Fixes

```bash
# 1. Check compilation
cargo check

# 2. Run clippy
cargo clippy

# 3. Try building
cargo build

# 4. Run tests
cargo test

# 5. Check specific file
cargo check --bin readr-backend
```

## Summary

**Root causes of your 11 errors:**

1. ❌ `lib.rs` missing `pub mod database` and `pub mod middleware`
2. ❌ Axum routing using closures instead of method wrappers
3. ❌ `health_check` returns `impl IntoResponse` but uses `?`
4. ❌ Router state type mismatch (no generic vs with_state)
5. ❌ Diesel `users` table not imported from schema
6. ❌ Variable `database` shadowed/out of scope in closure

**Fix order:**
1. Add missing modules to `lib.rs`
2. Fix Axum routing patterns
3. Fix async return types
4. Fix Diesel imports
5. Clean up unused imports (warnings)

All fixes preserve your architecture and just correct the Rust patterns.
