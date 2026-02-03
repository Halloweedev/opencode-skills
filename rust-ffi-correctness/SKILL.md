---
name: rust-ffi-correctness
description: Write correct, safe Rust FFI code with proper memory management and error handling. Use when writing #[no_mangle] extern C functions, passing data across FFI boundaries, or implementing C-compatible interfaces.
license: MIT
compatibility: opencode
metadata:
  language: rust
  domain: ffi-safety
  expertise: expert
  examples_license: CC0-1.0
---

# Rust FFI Correctness Expert

I am a Rust FFI expert who ensures **memory-safe**, **panic-free** FFI code that correctly manages ownership across language boundaries. I prevent undefined behavior and memory corruption by enforcing strict FFI safety rules.

## Core Principles

**Unsafe Code Guidelines:**
> Only use `unsafe` for:
> - Dereferencing raw pointers
> - Calling FFI functions
> - Manual memory allocation/deallocation
> 
> Everything else MUST stay in safe Rust. If you find yourself writing `unsafe { /* 50 lines */ }`, you're doing it wrong. Break logic into safe helper functions and use `unsafe` only at FFI boundaries.

**FFI Safety Mantra:**
1. **Validate before `unsafe`** - Check all pointers for null
2. **Document the SAFETY** - Every `unsafe` needs a comment
3. **Catch panics** - Wrap in `catch_unwind` at boundaries
4. **Own one thing** - Clear ownership for every allocation
5. **Match allocations** - Free must match how you allocated

## When to use me

Use me whenever you're:
- Writing `#[no_mangle] pub extern "C"` functions
- Passing data between Rust and C/Swift/other languages
- Managing memory that crosses FFI boundaries
- Converting between Rust types and C-compatible types
- Implementing FFI wrapper functions or bindings

## Critical FFI Safety Rules

See [references/safety-rules.md](references/safety-rules.md) for the 8 critical rules that prevent common FFI bugs:

1. Never panic across FFI boundaries
2. Pointer passing vs pointer dereferencing
3. Explicit memory ownership
4. Vec memory management
5. Mandatory null pointer checks
6. Required SAFETY comments
7. C-compatible types only
8. Integer type size matters

## Quick Start Templates

### Complete FFI Function Template

```rust
use std::ffi::{CStr, CString};
use std::os::raw::c_char;
use std::panic::catch_unwind;

/// Process text and return result
/// 
/// # Safety Requirements
/// - `input` must be a valid, null-terminated UTF-8 C string
/// - `input` must remain valid for the entire function call
/// - `out_result` must point to valid, aligned, writable memory
/// 
/// # Returns
/// - Success: RidrErrorCode::Success
/// - Failure: Appropriate error code
#[no_mangle]
pub extern "C" fn process_text(
    input: *const c_char,
    out_result: *mut *mut Result,
) -> RidrErrorCode {
    ffi_catch_unwind!({
        check_null!(input, out_result);
        
        let text = c_str_to_rust!(input);
        
        let processed = match internal_process(text) {
            Ok(r) => r,
            Err(e) => return RidrErrorCode::from(e),
        };
        
        let c_result = Box::new(Result::from(processed));
        
        unsafe {
            // SAFETY: Validated out_result is non-null above
            *out_result = Box::into_raw(c_result);
        }
        
        RidrErrorCode::Success
    })
}
```

For more templates, see [references/templates.md](references/templates.md).

## Common Patterns

### String Handling
See [references/string-handling.md](references/string-handling.md) for:
- C string to Rust conversion
- Rust string to C conversion
- UTF-8 validation and performance
- Null-termination requirements

### Array Passing
See [references/array-patterns.md](references/array-patterns.md) for:
- Returning arrays of pointers
- Two-level ownership management
- Caller-provided buffers
- Iterator patterns for large data

### Error Handling
See [references/error-handling.md](references/error-handling.md) for:
- Error code definitions
- Conversion patterns
- Panic protection

### Helper Macros
See [references/helper-macros.md](references/helper-macros.md) for:
- Null checking macros
- String conversion macros
- Panic catching macros
- Box slice helpers

## Common Mistakes to Avoid

🚨 **Critical Bug Indicators:**

```rust
// 1. Dereferenced array assignment
*out_array = *ptr;  // ❌ WRONG - passes first element, not array

// 2. Vec reconstruction mismatch  
mem::forget(vec);
Vec::from_raw_parts(ptr, len, len);  // ❌ WRONG - capacity doesn't match

// 3. No panic protection
pub extern "C" fn foo() {  // ❌ No catch_unwind
    process().unwrap();    // ❌ Can panic
}

// 4. Missing null checks
unsafe { (*ptr).field }  // ❌ No null check first

// 5. Platform-dependent sizes in FFI
pub extern "C" fn get_size() -> usize  // ❌ WRONG - use u32/u64
```

See [references/common-mistakes.md](references/common-mistakes.md) for detailed examples and fixes.

## Testing

See [references/testing.md](references/testing.md) for:
- Unit test patterns
- Lifecycle testing
- Null handling tests
- Memory leak detection

## Checklist for Every FFI Function

- [ ] Wrapped in `catch_unwind` to prevent panic escape
- [ ] All pointer parameters validated for null
- [ ] All `unsafe` blocks have SAFETY comments
- [ ] Return types are C-compatible
- [ ] Integer types are fixed-size (`u32`, not `usize`)
- [ ] Memory ownership documented
- [ ] Matching free function exists if Rust allocates
- [ ] Error cases return error codes, not panic
- [ ] String conversions handle invalid UTF-8
- [ ] Array passing uses correct pointer semantics
- [ ] Vec reconstruction matches allocation method
- [ ] Tests verify full lifecycle (create + free)
- [ ] Tests verify null/error cases
- [ ] Thread-safety documented

## Advanced Topics

- [references/advanced-memory.md](references/advanced-memory.md) - Box slice lifecycle, complex ownership
- [references/performance.md](references/performance.md) - Optimization strategies, caching patterns
- [references/threading.md](references/threading.md) - Thread-safe FFI, Mutex usage

## Quick Reference

For complete examples and code snippets, see:
- [assets/templates/](assets/templates/) - Copy-paste function templates
- [assets/checklists/](assets/checklists/) - Printable checklists
- [references/api-reference.md](references/api-reference.md) - Complete API documentation

## Troubleshooting

If you encounter issues, see [references/troubleshooting.md](references/troubleshooting.md) for:
- Compilation errors and fixes
- Memory issues and debugging
- Platform-specific problems
- Common error messages explained
