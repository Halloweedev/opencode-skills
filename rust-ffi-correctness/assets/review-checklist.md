# FFI Function Review Checklist

Use this checklist to review every FFI function before committing.

## ⚠️ CRITICAL FFI SAFETY (Fix These First)

These items cause **undefined behavior** or **compilation failures**. Check these before anything else:

### Must-Fix Items
- [ ] **Error enum has `#[repr(C)]`** - Without this = undefined behavior across FFI
- [ ] **`catch_unwind` uses `AssertUnwindSafe`** - Required for compilation with captured state
- [ ] **Output pointers zeroed after validation** - Contract: null means error
- [ ] **Thread-safety explicitly stated** - "Thread-safe" or "NOT thread-safe" in doc comment
- [ ] **No panics across FFI boundary** - All functions wrapped in catch_unwind
- [ ] **All pointers null-checked** - Before any dereference

❌ **If any of these are missing, stop and fix immediately.**

## Safety & Correctness

### Panic Safety
- [ ] Function wrapped in `catch_unwind` with `AssertUnwindSafe`
- [ ] `catch_unwind(AssertUnwindSafe(|| ...))` pattern used correctly
- [ ] No `.unwrap()`, `.expect()`, or panic-inducing operations
- [ ] All `Result` types properly handled
- [ ] Panic result returns `InternalError` code
- [ ] Imports include `std::panic::{catch_unwind, AssertUnwindSafe}`
- [ ] Captured mutable state is safe if panic occurs (or none exists)

### Pointer Validation
- [ ] ALL pointer parameters checked for null at function start
- [ ] Null checks before ANY dereference
- [ ] Use `check_null!` macro or explicit checks
- [ ] Invalid pointers return `InvalidInput` error
- [ ] Output pointers zeroed immediately after validation
- [ ] Output pointers set to null on all error paths (or stay null)
- [ ] Failure contract documented: "null means error"

### SAFETY Comments
- [ ] Every `unsafe` block has a SAFETY comment
- [ ] SAFETY comment addresses: non-null, aligned, lifetime, aliasing
- [ ] SAFETY comment explains why invariants hold
- [ ] Use checklist format: `✓ non-null ✓ aligned ✓ lifetime ✓ aliasing`

### Memory Management
- [ ] Ownership clearly documented in doc comment
- [ ] Matching free function exists if Rust allocates
- [ ] Free function properly reconstructs allocations
- [ ] No double-free possible
- [ ] No memory leaks on error paths
- [ ] Cleanup code for partial allocations on error

## Types & Signatures

### Type Safety
- [ ] All types are C-compatible (`#[repr(C)]` or primitives)
- [ ] No `String`, `Vec`, `Option` without `#[repr]`
- [ ] Integers are fixed-size (`u32`, not `usize`)
- [ ] Pointers are raw (`*const`, `*mut`)
- [ ] Enums use `#[repr(i32)]` or `#[repr(C)]` - **MANDATORY FOR FFI**
- [ ] Error enums have `#[repr(C)]` - **WITHOUT THIS = UNDEFINED BEHAVIOR**
- [ ] Success error code = 0 (C convention)
- [ ] Error codes are explicitly assigned (not relying on defaults)

### String Handling
- [ ] C strings assumed null-terminated
- [ ] UTF-8 validation performed
- [ ] Invalid UTF-8 returns error, not panic
- [ ] Null terminator added when creating C strings
- [ ] Interior nulls handled (CString::new can fail)

### Array Handling
- [ ] Array pointer passed correctly (ptr, not *ptr)
- [ ] Both array AND elements freed if pointer array
- [ ] Length parameter validated (not zero)
- [ ] Use `into_boxed_slice()` for exact allocation
- [ ] Capacity tracked if using Vec + mem::forget

## Documentation

### Doc Comments
- [ ] Function has doc comment with description
- [ ] Safety requirements section present
- [ ] Parameters documented
- [ ] Return values documented
- [ ] Memory contract documented (who owns what)
- [ ] Example usage provided
- [ ] Thread-safety **EXPLICITLY STATED** ("Thread-safe" or "NOT thread-safe")
- [ ] If thread-safe: conditions documented (e.g., "if internal_fn is thread-safe")
- [ ] Output parameter behavior on failure documented (e.g., "set to null")

### Inline Comments
- [ ] Complex logic explained
- [ ] All SAFETY comments present
- [ ] Error handling paths documented
- [ ] Why validation checks exist

## Error Handling

### Error Codes
- [ ] Error enum uses `#[repr(i32)]` or `#[repr(C)]` - **MANDATORY**
- [ ] WITHOUT `#[repr(C)]` = undefined behavior across FFI
- [ ] Success = 0 by convention
- [ ] All error cases have unique codes
- [ ] Error names prefixed (e.g., `RidrErrorCode`, not `ErrorCode`)
- [ ] Conversion from internal errors implemented
- [ ] Error values explicitly assigned (0, 1, 2, not auto)
- [ ] High error codes reserved for internal errors (e.g., 255)

### Error Paths
- [ ] All error paths return error codes
- [ ] No panics on error
- [ ] Resources cleaned up on error
- [ ] Partial allocations freed on error
- [ ] Output parameters set to null on error (if pointer type)
- [ ] Output parameters unchanged on error (if value type)
- [ ] Early returns maintain invariants (e.g., null output)

## Performance

### Allocations
- [ ] Minimize allocations in hot paths
- [ ] Use caller-provided buffers when possible
- [ ] Cache validated strings if reused
- [ ] Batch operations for large datasets
- [ ] Consider zero-copy approaches

### UTF-8 Validation
- [ ] Use `to_str()` in hot paths (no alloc)
- [ ] Only use `to_string_lossy()` in cold paths
- [ ] Cache validation results if possible
- [ ] Document performance characteristics

## Testing

### Unit Tests
- [ ] Test with valid inputs
- [ ] Test with null pointers (verify null in output on error)
- [ ] Test with invalid UTF-8
- [ ] Test with empty inputs
- [ ] Test with boundary conditions
- [ ] Test full lifecycle (create + free)
- [ ] Test output parameter is null after error
- [ ] Test UnwindSafe behavior (no compilation errors)
- [ ] Test with captured state if any exists

### Memory Tests
- [ ] Run with valgrind (no leaks)
- [ ] Run with miri (no undefined behavior)
- [ ] Test cleanup on error paths
- [ ] Verify no double-frees possible
- [ ] Check with AddressSanitizer

### Integration Tests
- [ ] Test from C/Swift caller
- [ ] Verify correct data transfer
- [ ] Test error handling from caller side
- [ ] Verify thread-safety if claimed

## Platform Considerations

### Cross-Platform
- [ ] Fixed-size integers used
- [ ] Platform assumptions documented
- [ ] Test on 32-bit and 64-bit
- [ ] Test on Windows and Unix if using `long`
- [ ] No endianness assumptions

### Build System
- [ ] Correct `crate-type` in Cargo.toml
- [ ] cbindgen configuration if generating headers
- [ ] Library builds for all target platforms
- [ ] No warnings in release build

## Code Quality

### Clarity
- [ ] Function does one thing
- [ ] Logic is easy to follow
- [ ] Unsafe blocks are minimal
- [ ] Complex logic extracted to safe functions
- [ ] Names are descriptive

### Maintainability
- [ ] No code duplication
- [ ] Use helper macros for repetitive checks
- [ ] Consistent error handling pattern
- [ ] Follow project conventions
- [ ] Add TODO comments for future improvements

## Pre-Commit Checklist

Quick verification before committing:

1. [ ] All unsafe blocks have SAFETY comments
2. [ ] All pointers checked for null
3. [ ] Function wrapped in catch_unwind with AssertUnwindSafe
4. [ ] Output pointers zeroed after validation
5. [ ] Error enum has `#[repr(C)]` - **MANDATORY**
6. [ ] Thread-safety explicitly documented
7. [ ] Doc comment complete
8. [ ] Tests pass
9. [ ] No compiler warnings
10. [ ] cargo clippy clean
11. [ ] Memory tests pass (valgrind/miri)

## Review Questions

Ask yourself:

1. Can this panic? → If yes, add catch_unwind with AssertUnwindSafe
2. Is this pointer null? → If maybe, check it
3. Is output pointer zeroed on entry? → If no, add zeroing after validation
4. Why is this unsafe? → Add SAFETY comment
5. Who owns this memory? → Document it
6. How is this freed? → Provide free function
7. What if this fails? → Handle the error and zero output
8. Is this thread-safe? → **Explicitly document "Thread-safe" or "NOT thread-safe"**
9. Can I test this? → Write the test
10. Does error enum have `#[repr(C)]`? → **MANDATORY - check now**
11. Is output null on error? → Verify caller contract
12. Are error values explicit? → No auto-assigned discriminants

## Common Issues Quick Check

🚨 Look for these patterns:

```rust
// ❌ CRITICAL ERRORS

// 1. Missing AssertUnwindSafe
catch_unwind(|| { ... })     // ❌ Won't compile with mutable captures
catch_unwind(AssertUnwindSafe(|| { ... }))  // ✅ Correct

// 2. Output not zeroed
#[no_mangle]
pub extern "C" fn f(out: *mut *mut T) -> Code {
    if out.is_null() { return InvalidInput; }
    // ❌ Missing: unsafe { *out = null_mut(); }
    
// 3. Missing #[repr(C)] on error enum
pub enum ErrorCode { ... }   // ❌ UNDEFINED BEHAVIOR
#[repr(C)]
pub enum ErrorCode { ... }   // ✅ Correct

// 4. Thread-safety not documented
/// Does stuff
pub extern "C" fn f(...) { } // ❌ Thread-safety unknown
/// # Thread Safety: NOT thread-safe
pub extern "C" fn f(...) { } // ✅ Explicit

// 5. Other critical issues
*out_array = *ptr;           // ❌ Dereferencing array pointer
Vec::from_raw_parts(p,l,l);  // ❌ Wrong capacity
.unwrap()                    // ❌ Can panic
unsafe { (*p).f }            // ❌ No null check
pub extern "C" fn f()        // ❌ No catch_unwind
-> usize                     // ❌ Platform-dependent
```

**Priority Order (Fix These First):**
1. ⚠️ Missing `#[repr(C)]` on error enum → UNDEFINED BEHAVIOR
2. ⚠️ Missing `AssertUnwindSafe` → Won't compile
3. ⚠️ Output not zeroed → Caller bugs
4. ⚠️ Thread-safety undocumented → Misuse
5. ⚠️ Missing catch_unwind → Panic escapes FFI

If you see any of these, fix immediately!

## Sign-Off

After completing this checklist:

- [ ] I have reviewed all items
- [ ] All critical items are addressed
- [ ] Tests pass and cover error cases
- [ ] Documentation is complete
- [ ] Code is ready for review

## Notes

Additional considerations or issues found:

_________________________________
_________________________________
_________________________________
