# Rust FFI Correctness Skill

A comprehensive OpenCode skill for writing memory-safe, panic-free Rust FFI code with proper ownership management.

## Installation

### For Project (Recommended for Readr)

```bash
# Copy to your project
cp -r rust-ffi-correctness .opencode/skills/

# Enable in project opencode.json
{
  "permission": {
    "skill": {
      "rust-ffi-correctness": "allow"
    }
  }
}
```

### For Global Use

```bash
# Copy to global config
cp -r rust-ffi-correctness ~/.config/opencode/skills/

# Enable in global opencode.json
{
  "permission": {
    "skill": {
      "rust-ffi-correctness": "allow"
    }
  }
}
```

## Structure

This skill uses OpenCode's **Progressive Loading Architecture** for optimal performance:

```
rust-ffi-correctness/
├── SKILL.md                      # Core skill (loads immediately)
│                                 # < 500 lines, fast loading
│
├── references/                   # Detailed docs (load on-demand)
│   ├── safety-rules.md          # 8 critical FFI rules
│   ├── array-patterns.md        # Complete array passing guide
│   └── helper-macros.md         # Boilerplate reduction macros
│
├── assets/                       # Templates (load when needed)
│   ├── function-template.md     # Copy-paste FFI function
│   └── review-checklist.md      # Pre-commit verification
│
└── scripts/                      # (Future) Automation tools
```

## How It Works

### Level 1: Skill Discovery
OpenCode sees:
- **Name**: `rust-ffi-correctness`
- **Description**: Triggers when you write FFI code
- **Metadata**: Language=rust, domain=ffi-safety

### Level 2: Core Instructions (SKILL.md)
When skill activates, loads:
- Critical safety rules summary
- Quick-start templates
- Links to detailed documentation
- Common mistakes checklist

### Level 3: On-Demand Details
Claude loads specific files only when needed:
- Writing array code → loads `references/array-patterns.md`
- Need macros → loads `references/helper-macros.md`
- Copy template → loads `assets/function-template.md`

## Usage Examples

### Example 1: Writing New FFI Function

**You**: "I need to write an FFI function that takes a C string and returns an array of tokens"

**OpenCode**:
1. Loads `SKILL.md` (core patterns)
2. Sees array requirement → loads `references/array-patterns.md`
3. Applies Pattern 1: Return Array of Pointers
4. Uses safety rules from `references/safety-rules.md`
5. Generates complete, safe code

### Example 2: Reviewing Existing Code

**You**: "Review this FFI code for bugs: [paste code]"

**OpenCode**:
1. Loads `SKILL.md` (quick bug indicators)
2. Loads `assets/review-checklist.md`
3. Checks against 8 critical rules
4. Identifies specific issues (e.g., "missing catch_unwind")
5. Provides fixes with references

### Example 3: Reducing Boilerplate

**You**: "This FFI code has too many null checks, can we simplify?"

**OpenCode**:
1. Loads `references/helper-macros.md`
2. Shows `check_null!` macro
3. Refactors code using macros
4. Maintains safety while improving clarity

## Key Features

### ✅ Prevents Common FFI Bugs
- **Pointer dereferencing** (`*out_array = *ptr` bug)
- **Vec capacity mismatch** (use `into_boxed_slice()`)
- **Panic across boundaries** (enforces `catch_unwind`)
- **Null pointer dereferences** (mandatory checks)
- **Memory leaks** (ownership documentation)

### ✅ Performance Optimized
- Main SKILL.md is < 500 lines (fast loading)
- Detailed examples separated into references
- On-demand loading of specific patterns
- No unnecessary context consumption

### ✅ Comprehensive Coverage
- **8 Critical Safety Rules** with examples
- **5 Array Patterns** including iterators
- **Helper Macros** to reduce boilerplate
- **Complete Templates** ready to copy
- **Review Checklist** for quality assurance

### ✅ Real-World Validated
All patterns tested against actual FFI bugs found in production code.

## What Makes This Different

### Traditional Approach
```
Single 1500-line SKILL.md
├── Everything loaded at once
├── Slow context window usage
└── Hard to maintain
```

### This Modular Approach
```
Lean 300-line SKILL.md
├── Fast initial load
├── Progressive detail loading
├── Easy to update specific sections
└── Optimal context usage
```

## File Details

### SKILL.md (Core - Always Loaded)
- Core principles and safety mantra
- Quick-start template
- Common mistakes summary
- Links to detailed docs
- Function review checklist

### references/safety-rules.md (On-Demand)
The 8 rules that prevent 95% of FFI bugs:
1. Never panic across FFI
2. Pointer passing vs dereferencing
3. Explicit memory ownership
4. Vec memory management
5. Mandatory null checks
6. Required SAFETY comments
7. C-compatible types only
8. Integer type size matters

### references/array-patterns.md (On-Demand)
Complete array handling guide:
- Two-level ownership explanation
- Return array of pointers
- Fill caller-provided buffer
- Iterator patterns (3 variants)
- Performance comparison
- Testing examples

### references/helper-macros.md (On-Demand)
Boilerplate reduction without losing safety:
- `check_null!` - Multiple pointer validation
- `c_str_to_rust!` - String conversion
- `ffi_catch_unwind!` - Panic protection
- `box_slice!` - Clean array passing
- Before/after examples

### assets/function-template.md (Copy-Paste)
Complete FFI function template with:
- Full safety documentation
- Step-by-step implementation
- Customization checklist
- Common variations
- Testing template

### assets/review-checklist.md (Quality Assurance)
Pre-commit verification covering:
- Safety & correctness (20+ items)
- Types & signatures (10+ items)
- Documentation requirements
- Error handling patterns
- Testing coverage
- Platform considerations

## Integration with Readr Project

This skill is specifically designed to work with the Readr backend FFI layer:

```rust
// Your code in readr-core/src/ffi.rs
#[no_mangle]
pub extern "C" fn ridr_start_reading(...) -> RidrErrorCode {
    // OpenCode will apply patterns from this skill
}
```

OpenCode will:
1. Recognize FFI function signatures
2. Apply safety rules automatically
3. Suggest helper macros
4. Validate against checklist
5. Generate proper documentation

## Maintenance

### Updating the Skill

To add new patterns or fix issues:

```bash
# Edit specific file
vim rust-ffi-correctness/references/array-patterns.md

# OpenCode automatically picks up changes
# No need to restart or reload
```

### Version History

Track changes in skill:
```bash
# Add changelog section to SKILL.md
## Changelog
- v1.0.0 (2026-02-03): Initial modular release
```

## Troubleshooting

### Skill Not Loading
1. Check file path: `.opencode/skills/rust-ffi-correctness/SKILL.md`
2. Verify permissions in `opencode.json`
3. Ensure SKILL.md has valid YAML frontmatter

### Too Much/Too Little Detail
Adjust which files load by modifying SKILL.md links:
- More concise → link to references earlier
- More detail → inline more examples

### Performance Issues
If skill loads slowly:
1. Check SKILL.md is < 500 lines
2. Move large examples to references/
3. Use progressive disclosure pattern

## Contributing

To add new patterns:

1. Keep SKILL.md under 500 lines
2. Add detailed content to `references/`
3. Link from SKILL.md to new reference
4. Update this README
5. Test with actual FFI code

## License

- Skill framework: MIT
- Code examples: CC0-1.0 (public domain)
- All examples free to copy without attribution

## Support

For issues or questions:
1. Check the review checklist
2. Review safety rules
3. Examine similar pattern in references
4. File issue with code example

## Credits

Created for the Readr project FFI layer.
Based on real bugs found and fixed in production Rust FFI code.
