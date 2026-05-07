# Research: Product Example Fix

**Feature**: Fix Product Example in simple.rs  
**Date**: 2026-05-07  
**Status**: Completed

## Summary

This feature is a straightforward example code fix with no technical unknowns. All design decisions are based on inspection of existing code patterns and the Product API documentation in the sysinfo crate.

---

## Findings

### 1. Product API Design

**Decision**: Use static methods, not instances

**Source**: Review of src/common/system.rs lines 1045-1170

**Details**:
- Product is a unit struct: `pub struct Product;`
- All product information is accessed via static methods: `Product::name()`, `Product::family()`, etc.
- No constructor exists; Product cannot be instantiated
- All methods return `Option<String>` to handle missing data

**Rationale**: This is the only way to access product information in the sysinfo API

---

### 2. Platform-Specific API Support

**Decision**: Exclude SKU field on macOS

**Source**: Product documentation comments in src/common/system.rs lines 1094-1100

**Exact Quote**:
```rust
/// Returns the product Stock Keeping Unit (SKU).
/// ...
/// ⚠️ Not supported on macOS/iOS.
```

**Handling**: Use `#[cfg(not(target_os = "macos"))]` guard on SKU method call

**Rationale**: API explicitly documents macOS/iOS limitation

---

### 3. Code Pattern for Examples

**Decision**: Follow the motherboard pattern

**Source**: Examples/simple.rs lines 336-338

**Current Pattern**:
```rust
"motherboard" => match Motherboard::new() {
    Some(m) => println!("{m:#?}"),
    None => println!("No motherboard information available"),
},
```

**Why Not Use This Pattern**: Motherboard returns an Option from constructor. Product doesn't have a constructor. Instead, individual Product methods return Options.

**Actual Pattern to Use**: Individual if-let statements for each method, like documentation examples in src/common/system.rs

---

### 4. Option Handling in Examples

**Decision**: Use if-let pattern for each optional field

**Source**: Product method documentation examples in src/common/system.rs

**Examples from Docs**:
```rust
if let Some(name) = Product::name() {
    println!("Product name: {:?}", name);
}
```

**Rationale**: 
- Matches the patterns shown in Product documentation
- Makes example educational (shows how to handle Options)
- Each field is independent and optional
- Clear and explicit handling

---

### 5. Output Formatting

**Decision**: Use structured format with field labels

**Reasoning**:
- Motherboard output is formatted with `{m:#?}` (pretty debug)
- Individual fields are smaller and benefit from labeled format
- More readable and educational than debug format
- Matches CLI tool conventions

---

## Alternatives Considered

### Alternative 1: Use Motherboard Pattern Exactly
```rust
let product = Product::... // No, Product can't be constructed
```
**Rejected**: Product doesn't have a constructor; only static methods exist

### Alternative 2: Single printf with all fields
```rust
println!("Product: name={}, family={}, ...", 
    Product::name().unwrap_or_default(), 
    Product::family().unwrap_or_default(), 
    ...);
```
**Rejected**: 
- Less educational
- Single line too long
- Less readable
- Doesn't match existing pattern

### Alternative 3: Use Product::name() directly without if-let
```rust
println!("Product name: {:?}", Product::name());
```
**Rejected**: Would print "Some(value)" or "None", not just the value

---

## Dependencies & Compatibility

**External Dependencies**: None  
**Breaking Changes**: None  
**Rust Edition**: 2021 (current)  
**MSRV Impact**: None (example, not library code)

---

## Conclusion

All design decisions are straightforward and based on:
1. Inspection of existing Product API
2. Following established code patterns in the crate
3. Documentation guidance from Product struct

**No technical blockers or unknowns identified.**
