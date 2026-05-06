# Skill: Fix Generator

## Purpose
Generate safe, idiomatic fixes following sysinfo patterns and constraints.

## When to Use
After analyzing an issue and identifying what needs to change.

## Core Principles

### 1. Respect MSRV (1.95)
- ❌ Avoid: `let x = vec![];` (use in stable Rust)
- ✅ Use: Stable, proven patterns
- ✅ Reference: Existing code in the repository
- Check [Rust 1.95 features](https://doc.rust-lang.org/1.95.0/) before using new syntax

### 2. Follow Existing Patterns
Look for similar code and replicate structure:
```rust
// Pattern: Platform-specific implementation
#[cfg(target_os = "linux")]
fn get_cpu_count() -> usize {
    // implementation
}

#[cfg(target_os = "macos")]
fn get_cpu_count() -> usize {
    // implementation
}
```

### 3. Unsafe Code Guidelines
- ❌ Avoid unless FFI or critical performance needed
- ✅ Document why with `// SAFETY: ...` comments
- ✅ Keep scope minimal
- ✅ Example from codebase:
```rust
unsafe {
    // SAFETY: ptr is valid from sysctl call above
    let value = *ptr as u64;
}
```

### 4. Feature Gate Awareness
- Check `Cargo.toml` for feature flags
- Use `#[cfg(feature = "...")]` for conditional compilation
- Ensure code compiles with/without features
- Example:
```rust
#[cfg(feature = "serde")]
use serde::{Deserialize, Serialize};

#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
pub struct MyStruct {
    // ...
}
```

### 5. Platform-Specific Code Structure
```rust
// In src/common/my_feature.rs (trait definition)
pub trait MyTrait {
    fn my_method(&self) -> Result<Data>;
}

// In src/unix/linux/my_feature.rs
#[cfg(target_os = "linux")]
impl MyTrait for System {
    fn my_method(&self) -> Result<Data> {
        // Linux-specific implementation
    }
}

// In src/unix/apple/my_feature.rs
#[cfg(any(target_os = "macos", target_os = "ios"))]
impl MyTrait for System {
    fn my_method(&self) -> Result<Data> {
        // macOS/iOS implementation
    }
}
```

## Fix Templates

### Template: Documentation Fix
```rust
// Change existing comment/doc comment

/// Gets the total CPU count.
///
/// This returns the number of logical CPUs available.
/// On unsupported platforms, returns 0.
///
/// # Example
/// ```
/// let sys = System::new_all();
/// println!("CPU count: {}", sys.cpus().len());
/// ```
pub fn cpus(&self) -> &[Cpu] {
    // ...
}
```

### Template: Bug Fix (Platform-Specific)
```rust
// File: src/unix/apple/process.rs

fn get_memory(&self) -> u64 {
    // OLD: incorrect calculation
    // NEW: correct calculation using libproc
    
    unsafe {
        // SAFETY: Called with valid process info from earlier call
        let info = proc_pidinfo(...);
        info.resident_size
    }
}
```

### Template: Test Addition
```rust
// File: tests/process.rs

#[test]
fn test_process_memory_not_zero() {
    let mut sys = System::new_all();
    sys.refresh_processes(ProcessRefreshKind::everything());
    
    let current_process = sys
        .process(Pid::from(std::process::id() as usize))
        .expect("should find current process");
    
    // Memory should be > 0 for running process
    assert!(current_process.memory() > 0);
}
```

### Template: Feature-Gated Code
```rust
#[cfg(feature = "my_feature")]
pub fn new_method(&self) -> MyType {
    // implementation
}

#[cfg(not(feature = "my_feature"))]
pub fn new_method(&self) -> MyType {
    // fallback implementation or panic
    panic!("my_feature not enabled")
}
```

## Validation Checklist Before Generating PR

- [ ] Code compiles on MSRV (1.95)
- [ ] No clippy warnings (`cargo clippy -- -D warnings`)
- [ ] rustfmt compliant (`cargo fmt -- --check`)
- [ ] All feature combinations build:
  - [ ] `cargo build`
  - [ ] `cargo build --no-default-features`
  - [ ] `cargo build --features serde`
  - [ ] `cargo build --features debug`
- [ ] Platform-specific code properly gated with `#[cfg(...)]`
- [ ] Tests added/updated
- [ ] No breaking changes to public API
- [ ] Unsafe code justified with `// SAFETY:` comments
- [ ] Documentation examples compile (if added)
