# Implementation Plan: Fix Product Example in simple.rs

**Feature**: Correct Product information display in the simple.rs example  
**Status**: Planning  
**Date**: 2026-05-07  
**Issue**: [GitHub #5](https://github.com/ssiddarthapexon/sysinfo/issues/5)

---

## Phase 0: Setup & Technical Context

### Technical Context

**Language & Framework**:
- Rust programming language
- No external dependencies for this fix

**Current Implementation**:
- File: `examples/simple.rs` line 340
- Current code: `println!("{:#?}", Product);`
- Bug: Prints the type name instead of instance data

**API to Use**:
- `Product::name()` → `Option<String>`
- `Product::family()` → `Option<String>`
- `Product::version()` → `Option<String>`
- `Product::uuid()` → `Option<String>`
- `Product::vendor_name()` → `Option<String>`
- `Product::serial_number()` → `Option<String>`
- `Product::stock_keeping_unit()` → `Option<String>` (not available on macOS)

**Related Code Patterns**:
- Motherboard command (line 336-338) shows the correct pattern: `Some(m) => println!("{m:#?}"), None => println!("No motherboard information available")`
- Or individual method calls with if-let pattern

**Platform Considerations**:
- SKU not supported on macOS/iOS
- Some product information may be unavailable on certain platforms
- All methods return Option types to handle missing data

---

## Phase 1: Design & Contracts

### 1.1 Specification Review Against Constitution

**Principle 1 (Cross-Platform Compatibility)** ✅
- Fix includes platform-specific handling (SKU cfg gate)
- All platforms will display available product information
- No platform-specific optimizations break other platforms

**Principle 2 (API Stability & Backwards Compatibility)** ✅
- This is an example fix, not an API change
- No public API modifications
- Backward compatible (example code only)

**Principle 3 (Security & Safety)** ✅
- No unsafe code additions
- Only uses public Product API methods
- No privilege escalation or unvalidated access

**Principle 4 (Data Integrity & Reliability)** ✅
- Example demonstrates correct API usage
- Gracefully handles missing data (None values)
- No silent data loss

**Constitution Status**: All principles satisfied ✅

---

### 1.2 Data Model

#### Product Information Display

**Entity**: Product Information Output

**Fields to Display**:
| Field | Source | Optional | Platform Notes |
|-------|--------|----------|-----------------|
| Name | `Product::name()` | Yes | All platforms |
| Family | `Product::family()` | Yes | All platforms |
| Version | `Product::version()` | Yes | All platforms |
| Serial Number | `Product::serial_number()` | Yes | All platforms |
| UUID | `Product::uuid()` | Yes | All platforms |
| Vendor Name | `Product::vendor_name()` | Yes | All platforms |
| SKU | `Product::stock_keeping_unit()` | Yes | Not on macOS/iOS |

**Validation**:
- All values are Option<String>
- None values display gracefully (omitted or marked as N/A)
- No validation needed (API methods return validated data)

---

### 1.3 API Contracts

#### Public Example Interface (examples/simple.rs)

**Command**: `product`

**Usage**:
```
> product
Product Information:
  Name: <string or N/A>
  Family: <string or N/A>
  Version: <string or N/A>
  Serial Number: <string or N/A>
  UUID: <string or N/A>
  Vendor: <string or N/A>
  SKU: <string or N/A>
```

**Error Handling**:
- If all product information unavailable: display "No product information available"
- If partial information: display only available fields
- No errors/panics from API calls

**Platform Compliance**:
- Linux: All fields may be available
- macOS: SKU excluded (cfg gate)
- Windows: All fields may be available
- BSD: All fields may be available

---

### 1.4 Implementation Approach

**Strategy**: Use individual method calls with if-let pattern for clarity

**Rationale**:
- Demonstrates how to use each Product method
- Shows proper error handling (Some/None pattern)
- More educational for users learning from example
- Each field is independent and optional

**Code Location**: `examples/simple.rs` line 340

**Implementation Steps**:
1. Replace the single println statement
2. Add header: `println!("Product Information:");`
3. For each product method: `if let Some(value) = Product::method() { println!("  Field: {}", value); }`
4. Add cfg gate for SKU: `#[cfg(not(target_os = "macos"))]`
5. Add fallback if no data: check if any product data exists

**Testing Approach**:
- Manual testing on available platforms
- Verify all fields display correctly
- Verify platform-specific exclusions work
- Verify graceful handling when data unavailable

---

## Phase 2: Review & Gate Evaluation

### Constitution Gate Check

**Result**: ✅ ALL GATES PASSED

**Review**:
- No API changes → No stability violations
- No unsafe code → Security principle satisfied
- Platform handling included → Compatibility principle satisfied
- No data integrity issues → Data integrity principle satisfied

**Justification**: This is a documentation/example fix, not a feature that would violate any constitutional principles. It improves code clarity without changing behavior.

---

## Summary of Design Artifacts

**Generated**:
- ✅ data-model.md - Product information fields and display contract
- ✅ contracts/api.md - Example interface specification
- ✅ research.md - Not needed (no technical unknowns)
- ✅ quickstart.md - Usage guide for Product API

**Status**: Ready for implementation phase

---

## Next Steps

1. Generate implementation tasks in tasks.md
2. Execute implementation tasks
3. Test on available platforms
4. Submit pull request

---

## Appendix: Template Context

**FEATURE_SPEC**: `docs/002-product-example-fix/spec.md`  
**IMPL_PLAN**: This file (`docs/002-product-example-fix/plan.md`)  
**DOCS_DIR**: `docs/002-product-example-fix`  
**CONSTITUTION**: `.state/memory/constitution.md`
