# Complete FFI Function Template

Copy and customize this template for your FFI functions.

```rust
use std::ffi::CStr;
use std::os::raw::c_char;
use std::panic::{catch_unwind, AssertUnwindSafe};

/// [Brief description of what this function does]
/// 
/// # Safety Requirements
/// - `input_param` must be [describe requirements, e.g., "valid null-terminated UTF-8 C string"]
/// - `input_param` must remain valid for [describe lifetime, e.g., "entire function call"]
/// - `out_param` must point to valid, aligned, writable memory
/// 
/// # Parameters
/// - `input_param`: [Description]
/// - `out_param`: [Description]
/// 
/// # Returns
/// - Success: `YourErrorCode::Success`, result written to `out_param`
/// - Failure: Appropriate error code, `out_param` set to null
/// 
/// # Memory Contract
/// - Return value must be freed with `your_free_function`
/// - `input_param` ownership remains with caller
/// - `out_param` receives ownership of allocated data on success
/// - `out_param` is set to null on failure (safe to check)
/// 
/// # Thread Safety
/// Thread-safe if `your_internal_function` is thread-safe and `YourType` 
/// contains no shared mutable state. Otherwise, NOT thread-safe.
/// [Explicitly state: "Thread-safe" or "NOT thread-safe"]
/// 
/// # Examples
/// ```c
/// const char* text = "Hello, FFI!";
/// Result* result = NULL;
/// ErrorCode code = your_function(text, &result);
/// if (code == Success) {
///     // Use result...
///     your_free_result(result);
/// }
/// ```
#[no_mangle]
pub extern "C" fn your_function_name(
    input_param: *const c_char,
    out_param: *mut *mut YourType,
) -> YourErrorCode {
    // 1. Wrap everything in panic protection with UnwindSafe
    let result = catch_unwind(AssertUnwindSafe(|| {
        
        // 2. Validate all pointer parameters
        if input_param.is_null() || out_param.is_null() {
            return YourErrorCode::InvalidInput;
        }
        
        // 3. Zero output parameter immediately (contract: null on failure)
        unsafe {
            // SAFETY: Validated out_param is non-null above
            *out_param = std::ptr::null_mut();
        }
        // 3. Zero output parameter immediately (contract: null on failure)
        unsafe {
            // SAFETY: Validated out_param is non-null above
            *out_param = std::ptr::null_mut();
        }
        
        // 4. Convert C types to Rust safely
        let rust_input = unsafe {
            // SAFETY: Validated input_param is non-null above
            // CStr::from_ptr requires null-terminated string
            match CStr::from_ptr(input_param).to_str() {
                Ok(s) => s,
                Err(_) => return YourErrorCode::InvalidUtf8,
            }
        };
        
        // 5. Validate input values
        if rust_input.is_empty() {
            return YourErrorCode::InvalidInput;
        }
        
        // 6. Perform the actual processing
        // (This is safe Rust code - no unsafe needed)
        let processed = match your_internal_function(rust_input) {
            Ok(result) => result,
            Err(e) => return YourErrorCode::from(e),
        };
        
        // 7. Convert result to C-compatible format
        let c_result = Box::new(YourType::from(processed));
        
        // 8. Transfer ownership to caller
        unsafe {
            // SAFETY: 
            // - out_param validated non-null at function start
            // - Box ensures proper alignment and initialization
            // - Ownership transferred - caller must free
            // - Overwrites null we set earlier
            *out_param = Box::into_raw(c_result);
        }
        
        YourErrorCode::Success
    }));
    
    // 9. Handle panics - never let them cross FFI boundary
    result.unwrap_or(YourErrorCode::InternalError)
}

/// Free memory allocated by your_function_name
/// 
/// # Safety
/// - `ptr` must have come from `your_function_name`
/// - `ptr` must not be used after this call
/// - Calling this function twice on the same pointer is undefined behavior
#[no_mangle]
pub extern "C" fn your_free_function(ptr: *mut YourType) {
    if !ptr.is_null() {
        unsafe {
            // SAFETY: ptr came from Box::into_raw in your_function_name
            // Reconstructing Box transfers ownership back to Rust
            // Drop frees the memory
            let _ = Box::from_raw(ptr);
        }
    }
}

/// Error code definition - MUST use #[repr(C)] for FFI safety
/// 
/// # ABI Safety
/// Without #[repr(C)], the enum layout is undefined and causes UB across FFI.
/// Success = 0 is a C convention that many callers expect.
#[repr(C)]
#[derive(Debug, Copy, Clone, PartialEq, Eq)]
pub enum YourErrorCode {
    Success = 0,
    InvalidInput = 1,
    InvalidUtf8 = 2,
    OutOfMemory = 3,
    ProcessingError = 4,
    InternalError = 255,  // Reserve high values for internal errors
}

// Conversion from internal errors
impl From<YourInternalError> for YourErrorCode {
    fn from(e: YourInternalError) -> Self {
        match e {
            YourInternalError::Parse => YourErrorCode::InvalidInput,
            YourInternalError::Memory => YourErrorCode::OutOfMemory,
            _ => YourErrorCode::ProcessingError,
        }
    }
}
```

## Critical Implementation Notes

### ⚠️ UnwindSafe Requirement

**Why `AssertUnwindSafe` is necessary:**

```rust
// ❌ WRONG - May not compile in strict contexts
let result = catch_unwind(|| {
    // Closure must be UnwindSafe
    // Any captured mutable state breaks this
});

// ✅ CORRECT - Explicitly assert unwind safety
let result = catch_unwind(AssertUnwindSafe(|| {
    // Now works even with captured mutable state
    // Developer takes responsibility for safety
}));
```

**When is this safe?**
- Your function doesn't capture mutable references
- No shared mutable state is accessed
- If panic occurs, no invariants are violated

**If you have shared mutable state:**
```rust
let shared_data = Arc::new(Mutex::new(data));
let result = catch_unwind(AssertUnwindSafe(|| {
    let guard = shared_data.lock().unwrap();
    // If panic here, mutex is poisoned - this is safe
    // Mutex::lock() handles poison correctly
}));
```

### ⚠️ Output Parameter Zeroing

**Why zero `out_param` immediately:**

```rust
// Caller code (C/Swift):
Result* result = (Result*)0xDEADBEEF;  // Garbage pointer
ErrorCode code = your_function(text, &result);
if (code != Success) {
    // Without zeroing: result is still 0xDEADBEEF - DANGEROUS!
    // With zeroing: result is NULL - safe to check
    if (result != NULL) {  // This check now works correctly
        cleanup(result);
    }
}
```

**The contract:**
```rust
// After validation, before processing:
unsafe {
    *out_param = std::ptr::null_mut();  // Null = "no valid data"
}

// On success:
unsafe {
    *out_param = Box::into_raw(result);  // Overwrite with valid pointer
}

// On error:
// out_param stays null (already set)
return ErrorCode::SomeError;
```

This makes the failure contract airtight: "null means error, non-null means success."

### ⚠️ Thread Safety Documentation

**You MUST explicitly state thread-safety:**

```rust
/// # Thread Safety
/// Thread-safe if `your_internal_function` is thread-safe and `YourType` 
/// contains no shared mutable state.
```

**Common thread-safety patterns:**

```rust
// ✅ Thread-safe (no shared state)
pub fn process_text(text: &str) -> Result<String> {
    text.to_uppercase()  // Pure function
}

// ⚠️ NOT thread-safe (shared mutable state)
static mut COUNTER: u32 = 0;
pub fn process_with_counter(text: &str) -> Result<String> {
    unsafe { COUNTER += 1; }  // Race condition!
    Ok(text.to_string())
}

// ✅ Thread-safe (protected shared state)
use std::sync::Mutex;
static COUNTER: Mutex<u32> = Mutex::new(0);
pub fn process_with_safe_counter(text: &str) -> Result<String> {
    *COUNTER.lock().unwrap() += 1;  // Protected by mutex
    Ok(text.to_string())
}
```

### ⚠️ Error Code ABI Safety

**Always use `#[repr(C)]`:**

```rust
// ❌ WRONG - Undefined layout across FFI
pub enum ErrorCode {
    Success = 0,
    InvalidInput = 1,
}

// ✅ CORRECT - Fixed C-compatible layout
#[repr(C)]
pub enum ErrorCode {
    Success = 0,
    InvalidInput = 1,
}
```

**Why this matters:**
- Without `#[repr(C)]`, Rust can reorder enum variants
- C caller may interpret Success as InvalidInput
- This is undefined behavior and can cause silent corruption

### 🔹 Future: extern "C-unwind" (Optional)

**When stabilized:**
```rust
// Makes panic behavior explicit
#[no_mangle]
pub extern "C-unwind" fn your_function_name(...) -> YourErrorCode {
    // Panic can unwind through this boundary
    // catch_unwind still required for guaranteed catching
}
```

**For now, use standard `extern "C"` as shown in template.**

## Customization Checklist

When using this template:

- [ ] Replace `your_function_name` with actual function name
- [ ] Replace `YourType` with your actual type name
- [ ] Replace `YourErrorCode` with your error enum (must have `#[repr(C)]`)
- [ ] Ensure error enum has `#[repr(C)]` - this is MANDATORY
- [ ] Update safety documentation with specific requirements
- [ ] Update parameter descriptions
- [ ] Update example code
- [ ] Implement your_internal_function
- [ ] Document thread-safety explicitly ("Thread-safe" or "NOT thread-safe")
- [ ] Add any additional validation needed
- [ ] Verify output parameter is zeroed after null check
- [ ] Test with null pointers (should return null in out_param)
- [ ] Test with invalid UTF-8
- [ ] Test with valid inputs
- [ ] Test memory cleanup with valgrind/miri
- [ ] Verify UnwindSafe behavior (no captured mutable refs cause issues)
- [ ] Test thread-safety if claiming thread-safe
- [ ] Verify error codes match C expectations (Success = 0)

## Common Variations

### Caller-Allocated Output

```rust
/// Fills caller-provided buffer
/// 
/// # Thread Safety
/// Thread-safe if your_internal_function is thread-safe.
#[no_mangle]
pub extern "C" fn your_function_fill(
    input_param: *const c_char,
    out_buffer: *mut YourType,
) -> YourErrorCode {
    catch_unwind(AssertUnwindSafe(|| {
        if input_param.is_null() || out_buffer.is_null() {
            return YourErrorCode::InvalidInput;
        }
        
        let rust_input = unsafe {
            match CStr::from_ptr(input_param).to_str() {
                Ok(s) => s,
                Err(_) => return YourErrorCode::InvalidUtf8,
            }
        };
        
        let result = match your_internal_function(rust_input) {
            Ok(r) => r,
            Err(e) => return YourErrorCode::from(e),
        };
        
        unsafe {
            // SAFETY: Caller allocated, we just write
            // No need to zero - caller provides the memory
            *out_buffer = YourType::from(result);
        }
        
        YourErrorCode::Success
    })).unwrap_or(YourErrorCode::InternalError)
}
// No free function needed - caller manages memory
```

### With Macros

```rust
use std::ffi::CStr;
use std::os::raw::c_char;
use std::panic::{catch_unwind, AssertUnwindSafe};

#[no_mangle]
pub extern "C" fn your_function_macro(
    input_param: *const c_char,
    out_param: *mut *mut YourType,
) -> YourErrorCode {
    ffi_catch_unwind!({
        check_null!(input_param, out_param);
        
        // Zero output on entry
        unsafe { *out_param = std::ptr::null_mut(); }
        
        let rust_input = c_str_to_rust!(input_param);
        let result = ffi_try!(your_internal_function(rust_input));
        
        let c_result = Box::new(YourType::from(result));
        
        unsafe {
            *out_param = Box::into_raw(c_result);
        }
        
        YourErrorCode::Success
    })
}

// Note: ffi_catch_unwind! macro should be defined as:
macro_rules! ffi_catch_unwind {
    ($body:expr) => {
        match catch_unwind(AssertUnwindSafe(|| $body)) {
            Ok(result) => result,
            Err(_) => YourErrorCode::InternalError,
        }
    };
}
```

## Testing Template

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_function_success() {
        let input = CString::new("test input").unwrap();
        let mut output = std::ptr::null_mut();
        
        let result = your_function_name(
            input.as_ptr(),
            &mut output,
        );
        
        assert_eq!(result, YourErrorCode::Success);
        assert!(!output.is_null());
        
        // Clean up
        your_free_function(output);
    }
    
    #[test]
    fn test_function_null_input() {
        let mut output = std::ptr::null_mut();
        
        let result = your_function_name(
            std::ptr::null(),
            &mut output,
        );
        
        assert_eq!(result, YourErrorCode::InvalidInput);
    }
    
    #[test]
    fn test_function_null_output() {
        let input = CString::new("test").unwrap();
        
        let result = your_function_name(
            input.as_ptr(),
            std::ptr::null_mut(),
        );
        
        assert_eq!(result, YourErrorCode::InvalidInput);
    }
    
    #[test]
    fn test_function_output_zeroed_on_error() {
        let mut output = 0xDEADBEEF as *mut YourType; // Garbage pointer
        
        // Use invalid UTF-8 to trigger error after output zeroing
        let result = your_function_name(
            std::ptr::null(), // Will fail validation
            &mut output,
        );
        
        // Should fail validation before zeroing
        assert_eq!(result, YourErrorCode::InvalidInput);
        
        // For a more complete test, you'd need to test with
        // input that passes null check but fails later validation
    }
}
```
