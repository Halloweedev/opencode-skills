# Helper Macros for FFI

FFI code is repetitive. These macros reduce boilerplate while maintaining safety.

## Null Pointer Check Macro

```rust
/// Returns early with InvalidInput if pointer is null
macro_rules! check_null {
    ($ptr:expr) => {
        if $ptr.is_null() {
            return RidrErrorCode::InvalidInput;
        }
    };
    ($($ptr:expr),+ $(,)?) => {
        $(
            if $ptr.is_null() {
                return RidrErrorCode::InvalidInput;
            }
        )+
    };
}
```

**Usage:**
```rust
// Before: Verbose
if input.is_null() {
    return RidrErrorCode::InvalidInput;
}
if out1.is_null() {
    return RidrErrorCode::InvalidInput;
}
if out2.is_null() {
    return RidrErrorCode::InvalidInput;
}

// After: Clean
check_null!(input, out1, out2);
```

## C String Conversion Macro

```rust
/// Convert C string to Rust &str with error handling
macro_rules! c_str_to_rust {
    ($ptr:expr) => {
        unsafe {
            match std::ffi::CStr::from_ptr($ptr).to_str() {
                Ok(s) => s,
                Err(_) => return RidrErrorCode::InvalidUtf8,
            }
        }
    };
}
```

**Usage:**
```rust
// Before: Repetitive
let text = unsafe {
    match CStr::from_ptr(input).to_str() {
        Ok(s) => s,
        Err(_) => return RidrErrorCode::InvalidUtf8,
    }
};

// After: Clean
let text = c_str_to_rust!(input);
```

## Panic Catching Macro

```rust
/// Wrap function body in catch_unwind with UnwindSafe
/// 
/// CRITICAL: Must use AssertUnwindSafe for Rust to accept the closure.
/// Without it, any captured mutable state causes compilation errors.
macro_rules! ffi_catch_unwind {
    ($body:expr) => {
        match std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| $body)) {
            Ok(result) => result,
            Err(_) => RidrErrorCode::InternalError,
        }
    };
}
```

**Why AssertUnwindSafe is necessary:**

```rust
// ❌ WRONG - Won't compile with captured mutable refs
let mut data = vec![1, 2, 3];
let result = catch_unwind(|| {
    data.push(4);  // Capture mutable ref - breaks UnwindSafe!
});

// ✅ CORRECT - AssertUnwindSafe allows this
let mut data = vec![1, 2, 3];
let result = catch_unwind(AssertUnwindSafe(|| {
    data.push(4);  // Now compiles - developer asserts this is safe
}));
```

**When is AssertUnwindSafe safe?**
- No invariants are violated if panic occurs
- Shared state uses proper synchronization (Mutex, RwLock)
- Or: function has no shared mutable state

**Usage:**
```rust
// Before: Manual
#[no_mangle]
pub extern "C" fn tokenize(...) -> RidrErrorCode {
    match std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| {
        // ... lots of code ...
    })) {
        Ok(code) => code,
        Err(_) => RidrErrorCode::InternalError,
    }
}

// After: Clean
#[no_mangle]
pub extern "C" fn tokenize(...) -> RidrErrorCode {
    ffi_catch_unwind!({
        // ... lots of code ...
    })
}
```

## Box Slice Helper Macro

```rust
/// Box slice helper for exact-size allocation
macro_rules! box_slice {
    ($vec:expr) => {{
        let boxed = $vec.into_boxed_slice();
        Box::into_raw(boxed) as *mut _
    }};
}
```

**Usage:**
```rust
// Before: Boilerplate
let boxed = items.into_boxed_slice();
let ptr = Box::into_raw(boxed) as *mut *mut c_char;

// After: Clean
let ptr = box_slice!(items);
```

## Complete Example with All Macros

```rust
#[no_mangle]
pub extern "C" fn process_text_clean(
    text: *const c_char,
    wpm: u32,
    out_array: *mut *mut c_char,
    out_count: *mut u32,
) -> RidrErrorCode {
    ffi_catch_unwind!({
        // Check all pointers in one line
        check_null!(text, out_array, out_count);
        
        // Convert C string in one line
        let text_str = c_str_to_rust!(text);
        
        if wpm == 0 || wpm > 2000 {
            return RidrErrorCode::InvalidInput;
        }
        
        let words = tokenize_internal(text_str);
        
        // Convert to C strings
        let mut c_strings = Vec::with_capacity(words.len());
        for word in words {
            match CString::new(word) {
                Ok(s) => c_strings.push(s.into_raw()),
                Err(_) => {
                    for ptr in c_strings {
                        unsafe { drop(CString::from_raw(ptr)); }
                    }
                    return RidrErrorCode::ProcessingError;
                }
            }
        }
        
        let count = c_strings.len();
        let ptr = box_slice!(c_strings);  // One line
        
        unsafe {
            *out_array = ptr;
            *out_count = count as u32;
        }
        
        RidrErrorCode::Success
    })
}
```

## Before and After Comparison

### Example 1: Null Checking

```rust
// BEFORE: 12 lines
#[no_mangle]
pub extern "C" fn process(
    input: *const c_char,
    out1: *mut Data,
    out2: *mut Stats,
) -> RidrErrorCode {
    if input.is_null() {
        return RidrErrorCode::InvalidInput;
    }
    if out1.is_null() {
        return RidrErrorCode::InvalidInput;
    }
    if out2.is_null() {
        return RidrErrorCode::InvalidInput;
    }
    // ...
}

// AFTER: 6 lines
#[no_mangle]
pub extern "C" fn process(
    input: *const c_char,
    out1: *mut Data,
    out2: *mut Stats,
) -> RidrErrorCode {
    check_null!(input, out1, out2);  // ✅ One line
    // ...
}
```

### Example 2: String Conversion

```rust
// BEFORE: 7 lines
let text = unsafe {
    match CStr::from_ptr(input).to_str() {
        Ok(s) => s,
        Err(_) => return RidrErrorCode::InvalidUtf8,
    }
};

// AFTER: 1 line
let text = c_str_to_rust!(input);  // ✅ One line
```

### Example 3: Panic Protection

```rust
// BEFORE: 9 lines
#[no_mangle]
pub extern "C" fn tokenize(...) -> RidrErrorCode {
    match std::panic::catch_unwind(|| {
        // ... lots of code ...
    }) {
        Ok(code) => code,
        Err(_) => RidrErrorCode::InternalError,
    }
}

// AFTER: 5 lines
#[no_mangle]
pub extern "C" fn tokenize(...) -> RidrErrorCode {
    ffi_catch_unwind!({
        // ... lots of code ...
    })
}
```

## Macro Guidelines

**DO:**
- Use macros for repetitive safety checks
- Keep macros simple and obvious
- Document what each macro does
- Test macros thoroughly

**DON'T:**
- Hide complex logic in macros
- Overuse - balance readability vs brevity
- Use macros for one-off operations
- Create macros that are harder to understand than the original code

## Advanced Macros

### SAFETY Comment Helper

```rust
/// Generate standard SAFETY comment
macro_rules! safety_comment {
    (non_null $ptr:expr) => {
        concat!(
            "// SAFETY: ", stringify!($ptr), " validated as non-null above\n"
        )
    };
    (aligned $ty:ty) => {
        concat!(
            "// SAFETY: Properly aligned for ", stringify!($ty), "\n"
        )
    };
}
```

### Error Conversion Helper

```rust
/// Convert Result to FFI error code
macro_rules! ffi_try {
    ($expr:expr) => {
        match $expr {
            Ok(val) => val,
            Err(e) => return RidrErrorCode::from(e),
        }
    };
}
```

**Usage:**
```rust
// Before
let result = match internal_function() {
    Ok(val) => val,
    Err(e) => return RidrErrorCode::from(e),
};

// After
let result = ffi_try!(internal_function());
```

## Macro Module Organization

```rust
// ffi_macros.rs
#[macro_export]
macro_rules! check_null { /* ... */ }

#[macro_export]
macro_rules! c_str_to_rust { /* ... */ }

#[macro_export]
macro_rules! ffi_catch_unwind { /* ... */ }

#[macro_export]
macro_rules! box_slice { /* ... */ }
```

Import in your FFI modules:
```rust
use crate::ffi_macros::*;
```

## Testing Macros

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_check_null_macro() {
        fn test_fn(ptr: *const i32) -> RidrErrorCode {
            check_null!(ptr);
            RidrErrorCode::Success
        }
        
        assert_eq!(test_fn(std::ptr::null()), RidrErrorCode::InvalidInput);
        assert_eq!(test_fn(&42 as *const i32), RidrErrorCode::Success);
    }
    
    #[test]
    fn test_c_str_conversion() {
        let s = CString::new("hello").unwrap();
        
        fn convert(ptr: *const c_char) -> &'static str {
            c_str_to_rust!(ptr)
        }
        
        let result = convert(s.as_ptr());
        assert_eq!(result, "hello");
    }
}
```

## Summary

Helper macros reduce FFI boilerplate by:
- Eliminating repetitive null checks
- Simplifying string conversions
- Standardizing panic protection
- Making box slice creation cleaner

Use them judiciously to improve code clarity without hiding important safety logic.
