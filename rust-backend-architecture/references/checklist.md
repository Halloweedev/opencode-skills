# Architecture Review Checklist

Use this checklist before committing code or during code review.

## Quick Pre-Commit Check (2 minutes)

- [ ] main.rs < 100 lines
- [ ] No handler > 50 lines
- [ ] No database queries in handlers
- [ ] Services return domain types, not HTTP types
- [ ] AppState contains services only
- [ ] No circular imports

**If any fail, fix before committing.**

## Complete Architecture Review

### Module Structure

#### main.rs
- [ ] < 100 lines total
- [ ] Only contains: config loading, DI, server startup
- [ ] No route definitions inline
- [ ] No business logic
- [ ] No database queries

#### lib.rs
- [ ] Clearly defines public API
- [ ] Implementation modules private
- [ ] Re-exports commonly used types
- [ ] Repositories not exposed
- [ ] Infrastructure not exposed

#### Handlers
- [ ] Each handler < 50 lines
- [ ] No business logic
- [ ] No database access
- [ ] No validation logic
- [ ] Just: extract → call service → map response
- [ ] Located in `handlers/` directory
- [ ] DTOs in `handlers/dto/`

#### Services
- [ ] Each service < 500 lines
- [ ] Contains business logic
- [ ] No HTTP types in signatures
- [ ] No Diesel queries
- [ ] Testable without HTTP
- [ ] Located in `services/` directory

#### Repositories
- [ ] Each repository < 200 lines
- [ ] Only place with Diesel queries
- [ ] Implements trait interface
- [ ] No business logic
- [ ] Returns domain types
- [ ] Located in `repositories/` directory

### Dependencies & Imports

#### Dependency Flow
- [ ] Handlers import services (not repositories)
- [ ] Services import repositories (not handlers)
- [ ] Repositories import schema (not services)
- [ ] No circular dependencies
- [ ] Dependencies flow inward only

#### Import Rules
- [ ] `use crate::services::X` in handlers ✅
- [ ] `use crate::repositories::X` in services ✅
- [ ] `use crate::handlers::X` anywhere ❌
- [ ] `use crate::services::X` in repositories ❌

### State Management

#### AppState Structure
- [ ] Contains `Arc<Service>` only
- [ ] No raw `DbPool`
- [ ] No raw infrastructure clients
- [ ] No `Config` (use `&'static` instead)
- [ ] One `Arc` per service (no double-wrapping)

#### Arc Usage
- [ ] Services wrapped in single `Arc`
- [ ] No `Arc<Arc<T>>`
- [ ] Config uses `&'static` not `Arc`
- [ ] Clone only when spawning tasks or storing

#### Service Composition
- [ ] Services own repositories
- [ ] Repositories own pools
- [ ] Handlers don't see pools
- [ ] Clear ownership hierarchy

### Error Handling

#### Error Layering
- [ ] Domain errors separate from infrastructure errors
- [ ] AppError wraps both layers
- [ ] `IntoResponse` implemented on AppError
- [ ] Handlers use `?` operator, never match

#### Error to HTTP Mapping
- [ ] Domain errors → 4xx status codes
- [ ] Infrastructure errors → 5xx status codes
- [ ] Mapping done once in `IntoResponse`
- [ ] Consistent error response format

#### Error Location
- [ ] Domain errors in `domain/errors.rs`
- [ ] Infrastructure errors in `infrastructure/errors.rs`
- [ ] AppError in `error.rs`
- [ ] Handlers never match on errors

### Database & Persistence

#### Diesel Usage
- [ ] Queries only in repositories
- [ ] Schema imports only in repositories
- [ ] Handlers never see `DbPool`
- [ ] Services never see `DbPool`

#### Repository Pattern
- [ ] Repositories implement traits
- [ ] Traits enable mocking
- [ ] One repository per table/aggregate
- [ ] CRUD operations clearly named

#### Connection Management
- [ ] Pool hidden in repository
- [ ] Connections obtained per-request
- [ ] No connection leaks
- [ ] Proper error handling on connection

 failure

### API Design

#### DTOs vs Domain
- [ ] Request/Response types in `handlers/dto/`
- [ ] Domain types in `domain/`
- [ ] Never serialize domain types directly
- [ ] Conversion at handler boundary only

#### Type Safety
- [ ] No Diesel types in API responses
- [ ] No `Queryable` on API types
- [ ] Sensitive fields not exposed
- [ ] API shape independent of DB schema

#### Versioning
- [ ] Routes prefixed with `/api/v1/`
- [ ] Can add v2 without breaking v1
- [ ] Domain types separate from DTOs

### Testing

#### Unit Tests
- [ ] Services testable without database
- [ ] Repository traits enable mocking
- [ ] No real database in service tests
- [ ] Fast tests (< 100ms per test)

#### Integration Tests
- [ ] Separate from unit tests
- [ ] Test full request/response cycle
- [ ] Can use real database here
- [ ] Test error paths

#### Test Coverage
- [ ] Business logic tested
- [ ] Error cases tested
- [ ] Happy paths tested
- [ ] Edge cases tested

### Cross-Cutting Concerns

#### Middleware
- [ ] Auth in middleware, not handlers
- [ ] Middleware is fast (no heavy logic)
- [ ] Request context via Extensions
- [ ] Tracing at boundaries

#### Logging & Tracing
- [ ] Structured logging used
- [ ] Correlation IDs on requests
- [ ] Error logging consistent
- [ ] No secrets in logs

#### Configuration
- [ ] Loaded once at startup
- [ ] Validated before use
- [ ] Not cloned per-request
- [ ] Secrets not in code

### Code Quality

#### Readability
- [ ] Clear variable names
- [ ] Functions have single responsibility
- [ ] No "god" functions
- [ ] Comments explain "why", not "what"

#### Maintainability
- [ ] Clear where new features go
- [ ] Easy to find things
- [ ] Consistent patterns used
- [ ] No duplication

#### Performance
- [ ] No unnecessary allocations
- [ ] Arc cloned only when needed
- [ ] Database queries optimized
- [ ] N+1 queries avoided

## Red Flags (Fix Immediately)

### 🚨 Critical Issues

- Handler > 50 lines
- Database query in handler
- Business logic in handler
- Service returns HTTP type
- `DbPool` in AppState
- Everything in `main.rs`
- No separation of domain/DTO
- God service (does everything)
- Circular dependencies
- Real database in unit tests

### ⚠️ Warning Signs

- Handler 30-50 lines (getting big)
- Service > 300 lines (consider splitting)
- Multiple `Arc<Arc<T>>` (wrong pattern)
- Config in AppState (should be static)
- Middleware doing business logic
- Error matching in multiple handlers
- DTOs passed between layers

### 📊 Health Metrics

**Healthy Project:**
- main.rs: 50-100 lines
- Handler: 10-30 lines
- Service: 100-300 lines
- Repository: 50-150 lines
- Adding feature touches 3-4 files

**Unhealthy Project:**
- main.rs: 500+ lines
- Handler: 100+ lines
- Service: 1000+ lines
- Repository: 500+ lines
- Adding feature touches 10+ files

## Review Questions

Ask yourself:

1. **"Where does X go?"**
   - If unclear → architecture problem

2. **"Can I test this without HTTP?"**
   - If no → too coupled to transport

3. **"Can I test this without database?"**
   - If no → repository should be trait

4. **"What happens if I swap the database?"**
   - If hard → Diesel leaked out

5. **"What happens if I add this feature?"**
   - If touches > 5 files → boundaries wrong

6. **"Can someone new understand this?"**
   - If no → refactor for clarity

7. **"Is this SOLID?"**
   - Single responsibility?
   - Open/closed?
   - Dependency inversion?

## Before Merging PR

Final checks:

- [ ] All checklist items pass
- [ ] No red flags present
- [ ] Tests added for new functionality
- [ ] Documentation updated
- [ ] No compiler warnings
- [ ] `cargo clippy` clean
- [ ] Code reviewed by peer

## Monthly Architecture Review

Do this monthly:

1. **Check file sizes**
   ```bash
   find src -name "*.rs" -exec wc -l {} + | sort -rn | head -20
   ```

2. **Check dependencies**
   ```bash
   cargo modules generate graph | grep "circular"
   ```

3. **Check test coverage**
   ```bash
   cargo tarpaulin --out Html
   ```

4. **Review scaling signals**
   - Any service > 500 lines?
   - Any handler > 50 lines?
   - State growing?
   - Tests slowing down?

5. **Team feedback**
   - Where do people get stuck?
   - What's confusing?
   - What takes too long?

## Summary

**Good architecture:**
- Clear boundaries
- Easy to test
- Easy to understand
- Easy to change

**Bad architecture:**
- Everything coupled
- Can't test
- Confusing
- Change ripples everywhere

**Use this checklist every commit to maintain quality over time.**
