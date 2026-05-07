# Quickstart: Product API Usage

**Feature**: Fix Product Example in simple.rs  
**Date**: 2026-05-07

---

## Overview

This guide shows how to retrieve and display hardware product information using the sysinfo `Product` API, as demonstrated in the fixed `examples/simple.rs`.

---

## Basic Usage

### Get Product Name

```rust
use sysinfo::Product;

if let Some(name) = Product::name() {
    println!("Product name: {}", name);
}
```

**Returns**: `Option<String>` - The hardware product name or None if unavailable

---

### Get Product Family

```rust
if let Some(family) = Product::family() {
    println!("Product family: {}", family);
}
```

**Returns**: `Option<String>` - Product family identifier or None

---

### Get Product Version

```rust
if let Some(version) = Product::version() {
    println!("Product version: {}", version);
}
```

**Returns**: `Option<String>` - Human-readable product version string or None

---

### Get Serial Number

```rust
if let Some(serial) = Product::serial_number() {
    println!("Serial number: {}", serial);
}
```

**Returns**: `Option<String>` - Hardware serial number or None

---

### Get UUID

```rust
if let Some(uuid) = Product::uuid() {
    println!("System UUID: {}", uuid);
}
```

**Returns**: `Option<String>` - Unique system identifier or None

---

### Get Vendor Name

```rust
if let Some(vendor) = Product::vendor_name() {
    println!("Vendor: {}", vendor);
}
```

**Returns**: `Option<String>` - Hardware manufacturer name or None

---

### Get SKU (Stock Keeping Unit)

```rust
#[cfg(not(target_os = "macos"))]
if let Some(sku) = Product::stock_keeping_unit() {
    println!("SKU: {}", sku);
}
```

**Returns**: `Option<String>` - Machine-readable product identifier or None  
**Platform Note**: ⚠️ Not available on macOS/iOS; use `#[cfg]` guard

---

## Complete Example

```rust
use sysinfo::Product;

fn display_product_info() {
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

fn main() {
    display_product_info();
}
```

**Output** (example on Linux):
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

---

## Common Patterns

### Pattern 1: Collect All Available Fields

```rust
use sysinfo::Product;

let mut fields = Vec::new();

if let Some(name) = Product::name() {
    fields.push(("Name", name));
}
if let Some(family) = Product::family() {
    fields.push(("Family", family));
}
// ... etc ...

for (label, value) in fields {
    println!("{}: {}", label, value);
}
```

### Pattern 2: Get Field with Default

```rust
let name = Product::name().unwrap_or_else(|| "Unknown".to_string());
println!("Product: {}", name);
```

⚠️ **Note**: Using `.unwrap_or_else()` in production code; be aware that "Unknown" may not accurately represent missing data. Consider using Option directly.

### Pattern 3: Check if Data Available

```rust
fn has_product_info() -> bool {
    Product::name().is_some()
        || Product::family().is_some()
        || Product::version().is_some()
        || Product::uuid().is_some()
}

if has_product_info() {
    println!("Product information is available");
} else {
    println!("No product information available on this system");
}
```

---

## Platform Differences

### Linux
- ✅ All fields typically available
- **Source**: /sys/firmware/dmi/id/ or /sys/class/dmi/id/
- **Availability**: Depends on system DMI data

### macOS/iOS
- ✅ All fields except SKU
- ❌ SKU not available (cfg guard prevents compilation)
- **Source**: System Profiler or IOKit
- **Availability**: Varies by system

### Windows
- ✅ All fields available
- **Source**: WMI (Win32_ComputerSystemProduct)
- **Availability**: Depends on WMI availability

### BSD
- ✅ All fields potentially available
- **Source**: dmidecode or system-specific method
- **Availability**: Depends on configuration

---

## Error Handling Best Practices

### ✅ Correct: Handle None Gracefully

```rust
match Product::name() {
    Some(name) => println!("Name: {}", name),
    None => println!("Product name not available"),
}
```

### ✅ Correct: Use if-let for Single Case

```rust
if let Some(serial) = Product::serial_number() {
    println!("Serial: {}", serial);
}
```

### ❌ Avoid: Unwrapping Without Reason

```rust
// Don't do this in production:
let name = Product::name().unwrap(); // May panic!
```

### ❌ Avoid: Silent Defaults

```rust
// Be explicit about what "missing" means:
let uuid = Product::uuid().unwrap_or_default(); // Is "" the same as "missing"?
```

---

## Troubleshooting

### No Product Information Displayed

**Possible Causes**:
- System firmware/BIOS not reporting data (uncommon)
- BIOS data not accessible due to permissions (on some systems)
- Virtualized environment without DMI data

**Solution**: Check that `has_product_info()` returns true; if not, system simply doesn't report this data.

### SKU Missing on macOS

**Expected Behavior**: SKU field is not available on macOS

**Solution**: Use `#[cfg(not(target_os = "macos"))]` guard:
```rust
#[cfg(not(target_os = "macos"))]
if let Some(sku) = Product::stock_keeping_unit() {
    // ...
}
```

### Fields Return None Sometimes

**Normal Behavior**: Product fields may return None even on supported systems if:
- Hardware doesn't report specific fields
- Virtualized environment
- System doesn't have DMI data configured

**Solution**: Always treat as Optional; never assume data is present.

---

## See Also

- [Product API Documentation](../../src/common/system.rs)
- [Constitution Principle 1: Cross-Platform Compatibility](../../.state/memory/constitution.md)
- [Data Model](../data-model.md)
- [API Contract](../contracts/api.md)
