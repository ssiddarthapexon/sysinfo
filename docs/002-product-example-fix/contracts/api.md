# API Contract: Product Example

**Feature**: Fix Product Example in simple.rs  
**Type**: Example Code Interface  
**Date**: 2026-05-07

---

## Overview

This document specifies the contract for how the `product` command in `examples/simple.rs` will interact with the Product API and display results to users.

---

## Input Specification

### Command: `product`

**Trigger**: User enters "product" at the interactive prompt in simple.rs

**Parameters**: None

**Preconditions**:
- Example is running
- User has access to system firmware/BIOS information (varies by platform and permissions)

---

## Output Specification

### Success Case: Product Data Available

When the system reports product information, the output displays all available fields:

```
Product Information:
  Name: <value>
  Family: <value>
  Version: <value>
  Serial Number: <value>
  UUID: <value>
  Vendor: <value>
  SKU: <value>
```

**Rules**:
- Each field appears only if Product::method() returns Some(value)
- Fields return None in any order (processed independently)
- Label is followed by colon and space: `<Label>: <value>`
- Each field on separate line, indented with 2 spaces
- No quotes around values (raw string values)

**Example Output**:
```
Product Information:
  Name: 20AN
  Family: T440p
  Version: Lenovo ThinkPad T440p
  Serial Number: W1KS427111E
  UUID: 407488fe-960a-43b5-a265-8fd0e9200b8f
  Vendor: LENOVO
  SKU: LENOVO_MT_20AN
```

### Partial Data Case: Some Fields Available

When some fields return None:

```
Product Information:
  Name: 20AN
  Version: Lenovo ThinkPad T440p
  Vendor: LENOVO
```

**Rule**: Fields that return None are omitted from output (no placeholders)

### No Data Case: No Product Information

When all Product methods return None:

```
No product information available
```

**Trigger**: All seven product methods return None  
**Message**: Single line, informative, no special formatting

---

## Platform-Specific Contracts

### Linux
- **Expected Fields**: All seven (name, family, version, serial, UUID, vendor, SKU)
- **Typical Availability**: High
- **Command Output**: Full product information with all available fields

### macOS/iOS
- **Expected Fields**: Six (all except SKU)
- **SKU Handling**: Excluded via `#[cfg(not(target_os = "macos"))]`
- **Typical Availability**: Variable (some systems provide less data)
- **Command Output**: Product information minus SKU field

### Windows
- **Expected Fields**: All seven
- **Typical Availability**: High (WMI usually available)
- **Command Output**: Full product information with all available fields

### BSD Variants (FreeBSD, NetBSD, OpenBSD)
- **Expected Fields**: All seven
- **Typical Availability**: Depends on dmidecode and firmware
- **Command Output**: Full product information with available fields

### Unknown/Unsupported Platforms
- **Expected Behavior**: Graceful degradation (display available fields or "No product information available")
- **No Errors**: Example does not panic or error on unknown platforms

---

## Error Handling Contract

### API Level Guarantees
- **No panics**: All Product methods return Option; never panic
- **No blocking**: Methods execute quickly (no I/O waits in user-facing code)

### Example Level Guarantees
- **No panics**: Example handles all None cases gracefully
- **No unwraps**: No `.unwrap()` or `.expect()` calls in product command code
- **User-friendly**: All error conditions display meaningful messages

---

## Behavior Specification

### When Executed
1. Example prints "Product Information:" header
2. For each Product method:
   - Call the method
   - If Some(value): print "  Field: value"
   - If None: skip field (no output)
3. If all methods return None: print "No product information available"

### Performance Guarantees
- **Responsiveness**: Output appears immediately (< 100ms typical)
- **No blocking**: No user-facing I/O waits
- **No spins**: No retry loops or polling

### Consistency Guarantees
- **Idempotent**: Running command multiple times produces identical output
- **No side effects**: Command does not modify system state
- **Thread-safe**: Safe to call in multi-threaded contexts (if example is multi-threaded)

---

## Backward Compatibility

**Current Behavior**:
```rust
println!("{:#?}", Product);  // Prints: Product
```

**New Behavior**:
```rust
println!("Product Information:");
if let Some(name) = Product::name() {
    println!("  Name: {}", name);
}
// ... etc for other fields
```

**Impact**: 
- Output changes completely (this is a bug fix)
- Existing scripts parsing output will break (output is now useful)
- No API changes; no library code affected
- Example only; documented as a fix

**Migration**: 
- Users need to update parsing logic if they relied on the broken output
- Breakage is acceptable as prior output was incorrect

---

## Testing Contract

### Test Scenarios

**Scenario 1: Product Information Available**
- **Input**: `product` command with system reporting data
- **Expected Output**: Structured field display
- **Validation**: All available fields appear correctly formatted
- **Platforms**: Linux (primary), macOS (secondary), Windows (secondary)

**Scenario 2: Partial Product Information**
- **Input**: `product` command with system reporting some data
- **Expected Output**: Only non-None fields displayed
- **Validation**: No empty fields, no placeholders
- **Platforms**: All (varies by system)

**Scenario 3: No Product Information**
- **Input**: `product` command with system reporting no data
- **Expected Output**: "No product information available"
- **Validation**: Message appears, no errors, no panics
- **Platforms**: All (if needed)

**Scenario 4: macOS Specific**
- **Input**: `product` command on macOS
- **Expected Output**: All fields except SKU
- **Validation**: SKU never appears (cfg guard works)
- **Platforms**: macOS/iOS only

---

## Code Contract

### Method Calls
```rust
Product::name() -> Option<String>
Product::family() -> Option<String>
Product::version() -> Option<String>
Product::serial_number() -> Option<String>
Product::uuid() -> Option<String>
Product::vendor_name() -> Option<String>
Product::stock_keeping_unit() -> Option<String>  // macOS: cfg-gated
```

### No New Dependencies
- Example does not add external crates
- Uses only std library and sysinfo imports

### Code Location
- **File**: examples/simple.rs
- **Lines**: ~340 (replacing current buggy code)
- **Scope**: Single match arm for "product" command

---

## Success Criteria

1. ✅ Command displays real product information (not type name)
2. ✅ All Product methods called and results displayed
3. ✅ Optional fields handled gracefully (no errors on None)
4. ✅ SKU excluded on macOS via cfg gate
5. ✅ Output is readable and formatted consistently
6. ✅ No panics or unwraps in code
7. ✅ Works on all supported platforms
8. ✅ No new dependencies added
