# Specification: Fix Product Example in simple.rs

**Feature**: Correct Product information display in the simple.rs example  
**Status**: Proposed  
**Date**: May 7, 2026  
**Issue**: [GitHub #5](https://github.com/ssiddarthapexon/sysinfo/issues/5)

## Executive Summary

The `examples/simple.rs` file contains a bug where the `product` command prints the type name `Product` instead of actual product information from the system. This specification addresses the fix to display correct hardware product details (name, family, version, UUID, etc.) to help users understand how to retrieve and display product data.

## Problem Statement

### Current Behavior

When a user runs the example and enters the `product` command, line 340 executes:
```rust
println!("{:#?}", Product);
```

This prints the type name rather than useful product information, producing output like:
```
Product
```

instead of actual system product details.

### Impact

- **Incomplete Example**: Users learning how to use the sysinfo library cannot understand how to properly retrieve and display product information
- **Confusion**: New developers trying to learn from the example will be confused about why `Product` prints the type instead of data
- **Documentation Gap**: The example fails to demonstrate the correct API usage pattern for accessing product data

## Objectives

1. **Display Actual Product Data**: Show real hardware product information (name, family, version, UUID, vendor, serial number, SKU)
2. **Demonstrate Correct API Usage**: Show users how to call Product static methods correctly
3. **Maintain Example Clarity**: Keep the example easy to read and understand
4. **Follow Existing Patterns**: Use the same style as other commands in the example (like `motherboard`)

## User Scenarios & Testing

### Scenario 1: User Learns Product API from Example
**Actor**: Developer new to sysinfo  
**Context**: Reading simple.rs to learn how to retrieve product information  
**Actions**:
1. Open examples/simple.rs
2. Look for how to use Product API
3. Run example and try `product` command
4. Observe the output

**Expected Outcome**: 
- Example shows calling Product methods like `Product::name()`, `Product::family()`, etc.
- Output displays actual product information
- User understands the correct pattern for accessing product data

**Test**: `test_product_command_displays_real_data()`

---

### Scenario 2: Cross-platform Consistency
**Actor**: Example maintainer  
**Context**: Ensuring example works across all supported platforms  
**Actions**:
1. Run example on Linux, macOS, Windows, BSD
2. Execute `product` command on each platform
3. Verify output is consistent and meaningful

**Expected Outcome**:
- All platforms display product information (or gracefully indicate unavailability)
- No platform-specific crashes or errors

**Test**: `test_product_command_cross_platform()`

---

## Functional Requirements

### FR1: Display Product Information
**Requirement**: The `product` command must retrieve and display actual hardware product information from the system.

**Acceptance Criteria**:
- Command calls Product static methods to retrieve data (name, family, version, UUID, vendor, serial number, SKU)
- Output includes all available product information
- Missing information is handled gracefully (e.g., "N/A" or None values displayed appropriately)
- Output is formatted for readability (similar to motherboard output)

---

### FR2: Handle Platform Differences
**Requirement**: The command must work correctly across all supported platforms, gracefully handling platform-specific limitations.

**Acceptance Criteria**:
- Linux displays all available product fields
- macOS displays available fields (SKU not supported on macOS per Product docs)
- Windows displays available product information
- BSD variants display available information
- Platforms with no product data available display a helpful message

---

### FR3: Follow Code Style
**Requirement**: The fix must match the existing code style and patterns in the example.

**Acceptance Criteria**:
- Follows the same formatting pattern as `motherboard` command (line 336)
- Uses consistent error handling (Some/None pattern)
- Comments are clear and helpful
- Variable naming is consistent with rest of file

---

## Success Criteria

1. **Correctness**: The product command displays actual system product information, not the type name
2. **Test Coverage**: All new code changes are tested (both at unit and manual level)
3. **Cross-Platform**: Works on all 4+ supported platforms without errors
4. **Documentation**: Code includes helpful comments explaining the API usage
5. **User Understanding**: Any user reading the example can understand how to use Product API

---

## Key Entities

### Changes to examples/simple.rs

**Current Code** (line 340):
```rust
"product" => {
    println!("{:#?}", Product);
}
```

**Corrected Code** (proposed):
```rust
"product" => {
    // Display all available product information using static methods
    println!("Product Information:");
    if let Some(name) = Product::name() {
        println!("  Name: {}", name);
    }
    if let Some(family) = Product::family() {
        println!("  Family: {}", family);
    }
    if let Some(version) = Product::version() {
        println!("  Version: {}", version);
    }
    if let Some(serial) = Product::serial_number() {
        println!("  Serial Number: {}", serial);
    }
    if let Some(uuid) = Product::uuid() {
        println!("  UUID: {}", uuid);
    }
    if let Some(vendor) = Product::vendor_name() {
        println!("  Vendor: {}", vendor);
    }
    #[cfg(not(target_os = "macos"))]
    if let Some(sku) = Product::stock_keeping_unit() {
        println!("  SKU: {}", sku);
    }
}
```

---

## Assumptions

1. **Static Methods Pattern**: Product uses static methods rather than instances (confirmed by API review)
2. **Option Types**: All Product methods return `Option<String>` and can be None on some platforms
3. **Platform Support**: Not all Product methods are available on all platforms (e.g., SKU not on macOS)
4. **Simple Approach Acceptable**: A straightforward implementation that calls each method separately is preferred over complex formatting macros

---

## Dependencies & Constraints

### External Dependencies
- None (uses only existing Product API)

### Internal Dependencies
- Changes only to examples/simple.rs
- No changes to src code required
- Leverages existing Product implementation in src/common/system.rs

### Technical Constraints
- Must use only public APIs from Product struct
- Cannot import additional dependencies
- Must follow existing code style conventions

### Scope Constraints
- Fixes only the simple.rs example
- Does not modify the Product API itself
- Does not add new methods to Product
- Does not affect any tests or library code

---

## Related Issues & References

- **GitHub Issue #5**: Bug report with example code and expected behavior
- **Documentation**: [Product struct in src/common/system.rs](../../src/common/system.rs)
- **Similar Example**: Motherboard command shows the correct pattern to follow

---

## Next Steps

1. Implement the fix in examples/simple.rs
2. Test the fix manually on available platforms
3. Run the example to verify product command output
4. Submit as pull request referencing issue #5
