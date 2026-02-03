# Critical FFI Safety Rules

This document contains the 8 fundamental rules that prevent common FFI bugs and undefined behavior.

## Rule 1: NEVER PANIC ACROSS FFI BOUNDARIES

**Panicking in FFI = Undefined Behavior**

```rust
// ❌ WRONG - Can panic
#[no_mangle]
pub extern "C" fn process_text(text: *const c_char) -> *mut Result {
    let text = unsafe { CStr::from_ptr(text).to_str().unwrap() }; // Panics!
    // ...
}

// ✅ CORRECT - Panic-safe
#[no_mangle]
pub extern "C" fn process_text(text: *const c_char) -> *mut Result {
    let result = std::panic::catch_unwind(|| {
        if text.is_null() {
            return create_error("Null pointer");
        }
        
        let text = unsafe {
            match CStr::from_ptr(text).to_str() {
                Ok(s) => s,
                Err(_) => return create_error("Invalid UTF-8"),
            }
        };
        
        // Process text...
        create_success(data)
    });
    
    match result {
        Ok(r) => r,
        Err(_) => create_error("Internal panic occurred"),
    }
}
```

**ALWAYS wrap FFI function bodies in `catch_unwind`.**

## Rule 2: POINTER PASSING vs POINTER DEREFERENCING

**The Most Common FFI Bug**

```rust
// When returning an array of pointers:

// ❌ ABSOLUTELY WRONG - Dereferences the array pointer!
let ptr = word_ptrs.as_mut_ptr();
mem::forget(word_ptrs);
unsafe {
    *out_array = *ptr;  // ❌ This gives ONE element, not the array!
}

// ✅ CORRECT - Passes the array pointer itself
let ptr = word_ptrs.as_mut_ptr();
mem::forget(word_ptrs);
unsafe {
    *out_array = ptr;  // ✅ This gives the whole array
}
```

**Rule**: When the output parameter is `*mut *mut T` (pointer to pointer), assign the pointer itself, not the dereferenced value.

## Rule 3: MEMORY OWNERSHIP MUST BE EXPLICIT

**Every allocation must have a clear owner and cleanup path.**

```rust
// Pattern 1: Rust allocates, caller frees
#[no_mangle]
pub extern "C" fn create_string() -> *mut c_char {
    let s = CString::new("Hello").unwrap();
    s.into_raw()  // Caller must call free_string
}

#[no_mangle]
pub extern "C" fn free_string(ptr: *mut c_char) {
    if !ptr.is_null() {
        unsafe { drop(CString::from_raw(ptr)); }
    }
}

// Pattern 2: Caller allocates, Rust fills
#[no_mangle]
pub extern "C" fn fill_data(out: *mut Data) -> RidrErrorCode {
    if out.is_null() {
        return RidrErrorCode::InvalidInput;
    }
    
    unsafe {
        // SAFETY: Caller guarantees valid, aligned, initialized memory
        *out = compute_data();
    }
    
    RidrErrorCode::Success
}
// No free function needed - caller manages memory
```

**Document ownership in comments:**

```rust
/// Returns allocated string - caller MUST call free_string when done
#[no_mangle]
pub extern "C" fn get_version() -> *mut c_char { ... }

/// Fills caller-provided buffer - caller manages memory
#[no_mangle]
pub extern "C" fn get_stats(out: *mut Stats) -> RidrErrorCode { ... }
```

## Rule 4: Vec MEMORY MANAGEMENT

**Using `mem::forget` requires matching reconstruction.**

```rust
// ❌ WRONG - Reconstruction doesn't match allocation
let mut vec = vec![1, 2, 3];
let ptr = vec.as_mut_ptr();
mem::forget(vec);

// Later, trying to free:
unsafe {
    drop(Vec::from_raw_parts(ptr, 3, 3));  // ❌ CRITICAL BUG!
    // Vec capacity is NOT guaranteed to equal length!
    // Vec may have allocated 4, 8, or more elements for growth
    // Using wrong capacity = memory corruption or leak
}

// ✅ CORRECT - Use Box<[T]> pattern instead
let vec = vec![1, 2, 3];
let boxed = vec.into_boxed_slice();  // Shrinks to exact-fit allocation
let ptr = Box::into_raw(boxed) as *mut i32;

// Later, to free:
unsafe {
    // Box<[T]> always has exact size, no capacity tracking needed
    let slice = std::slice::from_raw_parts_mut(ptr, 3);
    let boxed = Box::from_raw(slice);
    // Drops automatically
}

// ✅ ALTERNATIVE - Store capacity explicitly
let mut vec = vec![1, 2, 3];
let ptr = vec.as_mut_ptr();
let len = vec.len();
let cap = vec.capacity();  // ✅ MUST save this! Often > len!
mem::forget(vec);

// You MUST pass capacity to the caller somehow, e.g.:
// struct ArrayInfo { ptr: *mut T, len: usize, cap: usize }

// Later, with saved capacity:
unsafe {
    drop(Vec::from_raw_parts(ptr, len, cap));  // ✅ Use actual capacity
}
```

**Why `into_boxed_slice()` is better:**
- Shrinks allocation to exact size (len == capacity)
- No capacity tracking needed
- Simpler to reason about
- Cleaner FFI contract

**Box Slice Memory Flow:**
```
Vec [1,2,3,_,_,_] cap=6 len=3
    ↓ into_boxed_slice()
Box<[T]> [1,2,3] cap=len=3
    ↓ Box::into_raw()
*mut T → passed to caller
    ↓ ... time passes ...
from_raw_parts_mut(ptr, len)
    ↓ Box::from_raw(slice)
drop(boxed) ← freed
```

**Prefer `into_boxed_slice()` over `mem::forget(Vec)` for array passing.**

## Rule 5: NULL POINTER CHECKS ARE MANDATORY

```rust
// ❌ WRONG - No null check
#[no_mangle]
pub extern "C" fn process(data: *const Data, out: *mut Result) -> RidrErrorCode {
    unsafe {
        let value = (*data).field;  // Crashes if null!
        *out = process_value(value);
    }
    RidrErrorCode::Success
}

// ✅ CORRECT - Always validate
#[no_mangle]
pub extern "C" fn process(data: *const Data, out: *mut Result) -> RidrErrorCode {
    if data.is_null() || out.is_null() {
        return RidrErrorCode::InvalidInput;
    }
    
    unsafe {
        // SAFETY: Validated non-null above
        let value = (*data).field;
        *out = process_value(value);
    }
    
    RidrErrorCode::Success
}
```

**Check ALL pointer parameters at function start.**

## Rule 6: SAFETY COMMENTS ARE REQUIRED

**Every `unsafe` block MUST explain why it's safe.**

```rust
// ❌ WRONG - No explanation
unsafe {
    *out = value;
}

// ✅ CORRECT - Clear safety contract
unsafe {
    // SAFETY: We validated out is non-null above, and Value is
    // a Copy type that can be safely written to aligned memory.
    *out = value;
}
```

**SAFETY Comment Checklist:**
Every unsafe block should address these 4 points:
```rust
unsafe {
    // SAFETY:
    // - Pointer: non-null (checked above) and properly aligned
    // - Lifetime: valid for entire operation
    // - Aliasing: no other references exist
    // - Initialization: memory is properly initialized
    *ptr = value;
}
```

**Quick checklist format:**
```rust
// SAFETY: ✓ non-null ✓ aligned ✓ valid lifetime ✓ no aliasing
```

## Rule 7: C-COMPATIBLE TYPES ONLY

```rust
// ❌ WRONG - Not FFI-safe types
#[repr(C)]
pub struct Data {
    text: String,        // ❌ Not C-compatible
    items: Vec<i32>,     // ❌ Not C-compatible
    opt: Option<i32>,    // ❌ Not C-compatible (without repr)
}

// ✅ CORRECT - C-compatible
#[repr(C)]
pub struct Data {
    text: *mut c_char,   // ✅ Raw pointer
    items: *mut i32,     // ✅ Raw pointer
    item_count: usize,   // ✅ Array length
    value: i32,          // ✅ Primitive
    has_value: bool,     // ✅ bool is FFI-safe
}

// ✅ CORRECT - Option with repr
#[repr(i32)]
pub enum RidrErrorCode {
    Success = 0,
    InvalidInput = 1,
    ProcessingError = 2,
}
```

**FFI-safe types**: primitives, raw pointers, `#[repr(C)]` structs/enums, `bool`.

## Rule 8: TYPE SIZE MATTERS FOR INTEGERS

```rust
// ❌ WRONG - Platform-dependent size
#[no_mangle]
pub extern "C" fn get_count() -> usize {  // 32-bit vs 64-bit!
    42
}

// ✅ CORRECT - Fixed size
#[no_mangle]
pub extern "C" fn get_count() -> u32 {  // Always 32-bit
    42
}
```

**Integer Type Matching Table:**

| C Type | Rust Type | Size | Safety |
|--------|-----------|------|--------|
| `int32_t` | `i32` | 32-bit | ✅ Always safe |
| `uint32_t` | `u32` | 32-bit | ✅ Always safe |
| `int64_t` | `i64` | 64-bit | ✅ Always safe |
| `size_t` | `usize` | Platform | ⚠️ Document assumptions |
| `long` | Platform | Varies | ⚠️ 32-bit Windows, 64-bit Unix |

**Use fixed-size types** (`u32`, `i64`) instead of platform-dependent (`usize`, `isize`) in FFI signatures unless matching a specific C API.

## Summary

Follow these 8 rules to write correct, safe FFI code:

1. ✅ Wrap all FFI functions in `catch_unwind`
2. ✅ Pass array pointers, don't dereference them
3. ✅ Document ownership explicitly
4. ✅ Use `into_boxed_slice()` for arrays
5. ✅ Check ALL pointers for null
6. ✅ Comment EVERY `unsafe` block
7. ✅ Use only C-compatible types
8. ✅ Use fixed-size integer types

These rules prevent the most common sources of undefined behavior and memory corruption in FFI code.
