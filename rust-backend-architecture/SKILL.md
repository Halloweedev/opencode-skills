---
name: rust-backend-architecture
description: Architectural guidance for Axum-based Rust backends. Focuses on module boundaries, state management, error layering, domain separation, and scalability. Not for compilation fixes - assumes code compiles.
license: MIT
compatibility: opencode
metadata:
  language: rust
  frameworks: axum, tokio, diesel
  domain: backend-architecture
  expertise: advanced
---

# Rust Backend Architecture Expert

I design and review architectures for production Axum backends. I focus on boundaries, ownership, layering, and long-term maintainability. I assume your code compiles - I'm here to make it not turn into a mess at 50k LOC.

## When to use me

Use this skill when:
- Code compiles but feels wrong
- Adding features is painful (touch 10 files for 1 feature)
- Unclear where new code should go
- Handlers are doing database queries directly
- Everything is in `main.rs`
- State is growing unbounded with `Arc<Arc<Arc<...>>>`
- Errors are a mess (custom error per module)
- Testing is hard (everything coupled)
- Considering refactoring but don't know how

**Not for:**
- Compilation errors (use `rust-axum-diesel-debugging` skill)
- Performance optimization (different skill)
- Choosing between frameworks (you're already on Axum)

## Core Architectural Principles

See [references/principles.md](references/principles.md) for the 4 fundamental principles:

1. **Explicit Boundaries Over Convenience** - Handlers delegate, don't implement
2. **Dependencies Flow Inward** - HTTP → Service → Repository → Database
3. **Domain Types vs Transport Types** - API types separate from internal models
4. **Ownership Rules for State** - Services in AppState, not raw infrastructure

## Module Structure

See [references/module-boundaries.md](references/module-boundaries.md) for complete guide on:
- Directory structure (handlers, services, repositories, domain)
- What goes in `main.rs` (< 100 lines bootstrap only)
- What goes in `lib.rs` (public API surface)
- File size guidelines

## State Management

See [references/state-management.md](references/state-management.md) for:
- What goes in AppState (services, not pools)
- Arc usage rules (one Arc per service)
- Clone vs borrow patterns
- Common mistakes

## Error Architecture

See [references/error-architecture.md](references/error-architecture.md) for:
- Layered errors (domain vs infrastructure)
- Mapping to HTTP responses
- Where `IntoResponse` lives
- Anti-patterns to avoid

## Database Layer

See [references/database-architecture.md](references/database-architecture.md) for:
- Repository pattern with Diesel
- Where queries should live
- Preventing Diesel from leaking into handlers
- Testing strategies

## Anti-Patterns (Critical)

See [references/anti-patterns.md](references/anti-patterns.md) for 10 concrete anti-patterns with examples:

1. Direct database access in handlers
2. Business logic in handlers
3. Leaking Diesel types to HTTP layer
4. God AppState
5. Error matching in handlers
6. Services returning HTTP types
7. Config cloning in AppState
8. No separation between domain and transport
9. Middleware doing business logic
10. Testing against real database

## Quick Decision Trees

**Where does X go?**
- Database query? → `repositories/`
- Business logic? → `services/`
- HTTP mapping? → `handlers/`
- Domain entity? → `domain/`
- Request/Response type? → `handlers/dto/`

**How do I pass Y?**
- Config? → `&'static` or load once
- Database pool? → Inside repository, not AppState
- Service? → `Arc<Service>` in AppState
- Request context? → `Extension<RequestContext>`

## Project Structure Template

See [references/project-structure.md](references/project-structure.md) for:
- Complete directory tree
- Explanation of each folder
- File size guidelines
- Example implementations

## Scaling Signals

See [references/scaling-signals.md](references/scaling-signals.md) for red flags:
- Handler bloat (> 50 lines)
- God service (one service does everything)
- Circular dependencies
- State explosion
- DTOs passing between layers
- Testing requires database

## Architecture Review Checklist

Use this before committing:

### Module Structure
- [ ] `main.rs` < 100 lines
- [ ] Handlers < 50 lines (thin orchestration)
- [ ] Services contain business logic
- [ ] Repositories are only place touching database
- [ ] Domain types separate from DTOs

### Dependencies
- [ ] Dependencies flow inward (handlers → services → repos)
- [ ] No circular dependencies
- [ ] Services don't import handlers

### State
- [ ] AppState contains services, not infrastructure
- [ ] One `Arc<Service>` per service
- [ ] Config is `&'static` or loaded once

### Errors
- [ ] Errors layered (domain vs infrastructure)
- [ ] `IntoResponse` on AppError
- [ ] Handlers use `?`, never match

### Database
- [ ] Diesel queries only in repositories
- [ ] Handlers never see `DbPool`

See [references/checklist.md](references/checklist.md) for complete review checklist.

## Templates

Pre-built templates in [assets/templates/](assets/templates/):
- Handler template
- Service template
- Repository template
- Error type template
- AppState setup template

## Summary

**Architecture is about boundaries and ownership:**
1. Handlers orchestrate
2. Services implement business logic
3. Repositories access data
4. Domain types separate from DTOs
5. Errors flow upward, dependencies flow inward

**If you can't answer "where does X go?", your architecture needs work.**

**If adding a feature touches > 5 files, your boundaries are wrong.**

**Good architecture makes the next feature easier, not harder.**
