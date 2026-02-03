# Array Passing Patterns

Complete guide to passing arrays across FFI boundaries with proper ownership management.

## Ownership Rules for Array of Pointers

**Critical Ownership Distinction:**

When passing `*mut *mut T` (array of pointers), there are TWO levels of ownership:

```
*mut *mut c_char
 │     │
 │     └─> Each element (individual c_char strings)
 └───────> The array container itself (Box<[*mut c_char]>)
```

**Ownership Contract:**
1. **Array Container**: Created by `Box::into_raw(boxed_slice)` → MUST be freed by reconstructing Box
2. **Each Element**: Created by `CString::into_raw()` → MUST be freed individually with `CString::from_raw()`

**Double-Free Prevention:**
```rust
// ❌ WRONG - This only frees the array, not the strings!
unsafe {
    let slice = std::slice::from_raw_parts_mut(array, count);
    let boxed = Box::from_raw(slice);
    drop(boxed);  // ❌ Strings are leaked!
}

// ✅ CORRECT - Free both levels
unsafe {
    let slice = std::slice::from_raw_parts_mut(array, count);
    
    // Level 1: Free each string
    for &ptr in slice.iter() {
        if !ptr.is_null() {
            drop(CString::from_raw(ptr));  // ✅ Free element
        }
    }
    
    // Level 2: Free the array container
    let boxed = Box::from_raw(slice);  // ✅ Free array
    drop(boxed);
}
```

**Ownership Summary Table:**

| Item | Allocated By | Freed By | Method |
|------|-------------|----------|--------|
| Array container | `Box::into_raw(boxed_slice)` | Caller must call free function | `Box::from_raw(slice)` |
| Each string element | `CString::into_raw()` | Caller must call free function | `CString::from_raw(ptr)` |
| Both together | Rust FFI function | Rust free function | Free elements first, then array |

## Pattern 1: Return Array of Pointers

```rust
/// Returns array of C strings (null-terminated)
/// Caller must call free_string_array when done
#[no_mangle]
pub extern "C" fn tokenize(
    text: *const c_char,
    out_array: *mut *mut c_char,
    out_count: *mut u32,
) -> RidrErrorCode {
    catch_unwind(|| {
        if text.is_null() || out_array.is_null() || out_count.is_null() {
            return RidrErrorCode::InvalidInput;
        }
        
        let text = unsafe {
            // SAFETY: text must be null-terminated C string
            match CStr::from_ptr(text).to_str() {
                Ok(s) => s,
                Err(_) => return RidrErrorCode::InvalidUtf8,
            }
        };
        
        let words = internal_tokenize(text);
        
        // Convert to C strings (automatically null-terminated)
        let mut c_strings = Vec::with_capacity(words.len());
        for word in words {
            match CString::new(word) {
                Ok(s) => c_strings.push(s.into_raw()),
                Err(_) => {
                    // Cleanup on error
                    for ptr in c_strings {
                        unsafe { drop(CString::from_raw(ptr)); }
                    }
                    return RidrErrorCode::ProcessingError;
                }
            }
        }
        
        let count = c_strings.len();
        
        // Use boxed slice for exact allocation
        let boxed = c_strings.into_boxed_slice();
        let ptr = Box::into_raw(boxed) as *mut *mut c_char;
        
        unsafe {
            // SAFETY: Validated out_array and out_count non-null above
            *out_array = ptr;  // ✅ Assign pointer, not *ptr!
            *out_count = count as u32;
        }
        
        RidrErrorCode::Success
    }).unwrap_or(RidrErrorCode::InternalError)
}

#[no_mangle]
pub extern "C" fn free_string_array(array: *mut *mut c_char, count: u32) {
    if array.is_null() || count == 0 {
        return;
    }
    
    unsafe {
        // SAFETY: array came from Box::into_raw, count matches allocation
        let slice = std::slice::from_raw_parts_mut(array, count as usize);
        
        // Free each null-terminated string
        for &ptr in slice.iter() {
            if !ptr.is_null() {
                drop(CString::from_raw(ptr));
            }
        }
        
        // Free the array
        let boxed = Box::from_raw(slice);
        drop(boxed);
    }
}
```

## Pattern 2: Fill Caller-Provided Buffer

```rust
/// Fills caller-allocated array
/// Buffer must have space for at least max_count items
#[no_mangle]
pub extern "C" fn get_items(
    buffer: *mut i32,
    max_count: u32,
    out_actual: *mut u32,
) -> RidrErrorCode {
    if buffer.is_null() || out_actual.is_null() {
        return RidrErrorCode::InvalidInput;
    }
    
    if max_count == 0 {
        return RidrErrorCode::InvalidInput;
    }
    
    let items = internal_get_items();
    let count = items.len().min(max_count as usize);
    
    unsafe {
        // SAFETY: Caller guarantees buffer has max_count capacity
        let slice = std::slice::from_raw_parts_mut(buffer, count);
        slice.copy_from_slice(&items[..count]);
        
        // SAFETY: Validated out_actual non-null above
        *out_actual = count as u32;
    }
    
    RidrErrorCode::Success
}
```

## Pattern 3: Iterator for Large Data

**Thread Safety Warning:**
> FFI iterators are typically **NOT thread-safe** unless explicitly documented.
> - Multiple threads calling `iterator_next()` = undefined behavior
> - Caller must ensure exclusive access
> - For thread-safety, wrap in `Mutex<T>`

```rust
/// Opaque iterator handle
/// 
/// # Thread Safety
/// NOT thread-safe. Caller must ensure only one thread accesses at a time.
pub struct TextIterator {
    words: Vec<String>,
    position: usize,
}

#[no_mangle]
pub extern "C" fn create_iterator(text: *const c_char) -> *mut TextIterator {
    if text.is_null() {
        return std::ptr::null_mut();
    }
    
    let text = unsafe {
        match CStr::from_ptr(text).to_str() {
            Ok(s) => s,
            Err(_) => return std::ptr::null_mut(),
        }
    };
    
    let words = internal_tokenize(text);
    
    Box::into_raw(Box::new(TextIterator {
        words,
        position: 0,
    }))
}

/// Returns next word or null if done
/// Caller must free returned string with free_string
/// 
/// Performance: Allocates new CString each call.
/// For >1000 items, use batch retrieval instead.
#[no_mangle]
pub extern "C" fn iterator_next(iter: *mut TextIterator) -> *mut c_char {
    if iter.is_null() {
        return std::ptr::null_mut();
    }
    
    unsafe {
        let iter = &mut *iter;
        
        if iter.position >= iter.words.len() {
            return std::ptr::null_mut();
        }
        
        let word = &iter.words[iter.position];
        iter.position += 1;
        
        match CString::new(word.as_str()) {
            Ok(s) => s.into_raw(),
            Err(_) => std::ptr::null_mut(),
        }
    }
}

#[no_mangle]
pub extern "C" fn free_iterator(iter: *mut TextIterator) {
    if !iter.is_null() {
        unsafe { drop(Box::from_raw(iter)); }
    }
}
```

## Pattern 4: Batch Retrieval (High Performance)

```rust
/// Returns batch of words for high-frequency iteration
/// Much faster than iterator_next() for large datasets
#[no_mangle]
pub extern "C" fn iterator_next_batch(
    iter: *mut TextIterator,
    out_buffer: *mut *mut c_char,
    max_count: u32,
    out_actual: *mut u32,
) -> RidrErrorCode {
    if iter.is_null() || out_buffer.is_null() || out_actual.is_null() {
        return RidrErrorCode::InvalidInput;
    }
    
    if max_count == 0 {
        return RidrErrorCode::InvalidInput;
    }
    
    unsafe {
        let iter_ref = &mut *iter;
        
        let remaining = iter_ref.words.len() - iter_ref.position;
        let batch_size = remaining.min(max_count as usize);
        
        if batch_size == 0 {
            *out_actual = 0;
            return RidrErrorCode::Success;
        }
        
        let mut batch = Vec::with_capacity(batch_size);
        
        for _ in 0..batch_size {
            let word = &iter_ref.words[iter_ref.position];
            iter_ref.position += 1;
            
            match CString::new(word.as_str()) {
                Ok(s) => batch.push(s.into_raw()),
                Err(_) => {
                    // Cleanup on error
                    for ptr in batch {
                        drop(CString::from_raw(ptr));
                    }
                    return RidrErrorCode::ProcessingError;
                }
            }
        }
        
        let count = batch.len();
        let boxed = batch.into_boxed_slice();
        let ptr = Box::into_raw(boxed) as *mut *mut c_char;
        
        *out_buffer = ptr;
        *out_actual = count as u32;
        
        RidrErrorCode::Success
    }
}
```

## Pattern 5: Zero-Allocation (Caller Buffer)

```rust
/// Write directly to caller's buffer - zero allocations
#[no_mangle]
pub extern "C" fn iterator_next_into(
    iter: *mut TextIterator,
    buffer: *mut c_char,
    buffer_size: u32,
    out_written: *mut u32,
) -> RidrErrorCode {
    if iter.is_null() || buffer.is_null() || out_written.is_null() {
        return RidrErrorCode::InvalidInput;
    }
    
    if buffer_size == 0 {
        return RidrErrorCode::InvalidInput;
    }
    
    unsafe {
        let iter_ref = &mut *iter;
        
        if iter_ref.position >= iter_ref.words.len() {
            *out_written = 0;
            return RidrErrorCode::Success;
        }
        
        let word = &iter_ref.words[iter_ref.position];
        
        // Check if word fits (including null terminator)
        if word.len() + 1 > buffer_size as usize {
            return RidrErrorCode::InvalidInput;
        }
        
        // Copy with null terminator
        let word_bytes = word.as_bytes();
        std::ptr::copy_nonoverlapping(
            word_bytes.as_ptr(),
            buffer as *mut u8,
            word_bytes.len()
        );
        
        *buffer.add(word_bytes.len()) = 0;  // Null terminator
        
        *out_written = (word_bytes.len() + 1) as u32;
        iter_ref.position += 1;
        
        RidrErrorCode::Success
    }
}
```

## Performance Comparison

| Pattern | Allocations | Speed | Use Case |
|---------|-------------|-------|----------|
| `iterator_next()` | 1 per item | Slow | < 1000 items |
| `iterator_next_batch(100)` | 1 per 100 items | 100x faster | Large datasets |
| `iterator_next_into()` | 0 | Fastest | Critical performance |

## Common Array Mistakes

### Mistake 1: Dereferencing Array Pointer
```rust
// ❌ WRONG
*out_array = *ptr;  // Passes first element, not array

// ✅ RIGHT
*out_array = ptr;   // Passes the array pointer
```

### Mistake 2: Wrong Vec Reconstruction
```rust
// ❌ WRONG - capacity doesn't match
let ptr = vec.as_mut_ptr();
mem::forget(vec);
drop(Vec::from_raw_parts(ptr, len, len));  // ❌

// ✅ RIGHT - use into_boxed_slice
let boxed = vec.into_boxed_slice();
let ptr = Box::into_raw(boxed);
```

### Mistake 3: Freeing Only Array, Not Elements
```rust
// ❌ WRONG - elements leaked!
unsafe {
    let boxed = Box::from_raw(slice);
    drop(boxed);
}

// ✅ RIGHT - free both levels
unsafe {
    // Free elements first
    for &ptr in slice.iter() {
        drop(CString::from_raw(ptr));
    }
    // Then free array
    let boxed = Box::from_raw(slice);
    drop(boxed);
}
```

## Testing Array Patterns

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_array_lifecycle() {
        let text = CString::new("hello world").unwrap();
        let mut array = std::ptr::null_mut();
        let mut count = 0;
        
        let result = tokenize(
            text.as_ptr(),
            &mut array,
            &mut count,
        );
        
        assert_eq!(result, RidrErrorCode::Success);
        assert_eq!(count, 2);
        assert!(!array.is_null());
        
        // Verify contents
        unsafe {
            let slice = std::slice::from_raw_parts(array, count as usize);
            let word1 = CStr::from_ptr(slice[0]).to_str().unwrap();
            let word2 = CStr::from_ptr(slice[1]).to_str().unwrap();
            assert_eq!(word1, "hello");
            assert_eq!(word2, "world");
        }
        
        // Clean up
        free_string_array(array, count);
    }
}
```

## Summary

Array passing requires:
1. Understanding two-level ownership for pointer arrays
2. Freeing both elements AND container
3. Using correct pointer semantics (ptr, not *ptr)
4. Choosing appropriate pattern for performance needs
5. Testing full lifecycle with proper cleanup
