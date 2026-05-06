# Skill: Test Validator

## Purpose
Validate that generated fixes pass all CI checks and testing requirements.

## When to Use
After generating a fix, before creating the PR.

## CI Pipeline Understanding

The sysinfo CI (`.github/workflows/CI.yml`) runs:

### Job: `rustfmt`
**Requirement**: All Rust files must be formatted consistently
```bash
cargo fmt -- --check
```
**If fails**: Run `cargo fmt` to auto-fix, then verify

### Job: `clippy`
**Requirement**: No clippy warnings on multiple platforms
```bash
cargo clippy --all-targets --features serde -- -D warnings
cargo clippy --all-targets --features multithread -- -D warnings
```
**Platforms**: ubuntu-latest, macos-latest, windows-latest

### Job: `check`
**Requirement**: Code must compile on all supported targets and toolchains
**Toolchains**: 1.95.0 (MSRV), stable, nightly
**Targets**: x86_64-linux-gnu, i686-linux-gnu, x86_64-darwin, x86_64-windows-msvc, iOS, Android, ARM variants, etc.

**Feature combinations tested**:
- Default features
- `--features=debug`
- `--features=serde`
- `--features=debug,serde,system`
- `--features=apple-sandbox` (macOS only)
- No default features

### Job: `tests`
**Requirement**: Unit tests pass on all major platforms
**Special Rules**:
- **macOS**: Uses `--test-threads 1` (resource contention issue)
- **Linux/Windows**: Parallel testing allowed
- **Environment**: `APPLE_CI=1` set for macOS
- **Rust versions**: 1.95.0, stable, nightly

**Test commands**:
```bash
# With default features
cargo test

# No default features
cargo test --no-default-features

# With serde
cargo test --features serde --doc

# CPU-intensive tests (single threaded)
cargo test --lib -- --ignored --test-threads 1
```

### Job: `c_interface`
**Requirement**: C interface compiles and works
```bash
make
```

### Job: `unknown-targets`
**Requirement**: Unknown platform handling works
```bash
cargo clippy --features unknown-ci -- -D warnings
cargo check --features unknown-ci
cargo test --features unknown-ci
cargo install wasm-pack && cd test-unknown && wasm-pack build --target web
```

## Local Validation Commands

Before submitting PR, run these to match CI:

```bash
# Format check
cargo fmt -- --check

# Clippy on multiple feature sets
cargo clippy --all-targets --features serde -- -D warnings
cargo clippy --all-targets --features multithread -- -D warnings

# Compile check (at least MSRV and stable)
rustup install 1.95.0
cargo +1.95.0 check
cargo +stable check

# Full test suite
cargo test --all
cargo test --no-default-features
cargo test --features serde --doc

# Unknown platform (if available)
cargo clippy --features unknown-ci -- -D warnings
cargo test --features unknown-ci

# macOS-specific (if on macOS)
cargo test -- --test-threads 1
```

## Test Coverage Requirements

### Existing Tests
Located in `tests/`:
- `tests/system.rs` — System info
- `tests/process.rs` — Process info
- `tests/disk.rs` — Disk info
- `tests/cpu.rs` — CPU info
- `tests/network.rs` — Network info
- `tests/users.rs` — User info
- `tests/components.rs` — Component/temperature info

### When to Add Tests
- ✅ Adding new public API method
- ✅ Fixing a bug (add regression test)
- ✅ Improving platform coverage
- ✅ Adding feature-specific functionality

### Test Structure Pattern
```rust
#[test]
fn test_my_feature() {
    let mut sys = System::new_all();
    
    // For time-dependent data, refresh first
    sys.refresh_processes(ProcessRefreshKind::everything());
    
    // Assertions
    assert!(!sys.processes().is_empty());
}

// Platform-specific tests
#[test]
#[cfg(target_os = "linux")]
fn test_linux_specific_feature() {
    // Linux-only test
}
```

## Validation Checklist

- [ ] `cargo fmt -- --check` passes
- [ ] `cargo clippy --all-targets --features serde -- -D warnings` passes
- [ ] `cargo clippy --all-targets --features multithread -- -D warnings` passes
- [ ] `cargo +1.95.0 check` passes (MSRV)
- [ ] `cargo check` passes (latest)
- [ ] `cargo test` passes (all tests)
- [ ] `cargo test --no-default-features` passes
- [ ] `cargo test --features serde --doc` passes
- [ ] Tests added if behavior changed
- [ ] No new compiler warnings
- [ ] Documentation examples still accurate

## Common CI Failures & Fixes

| Failure | Cause | Fix |
|---------|-------|-----|
| rustfmt check fails | Code not formatted | `cargo fmt` |
| clippy warnings | Code style issues | `cargo clippy --fix` or manual fix |
| MSRV check fails | Using features > 1.95 | Use stable alternatives |
| macOS test timeout | Parallel tests on limited resources | Ensure tests respect resource limits |
| Feature combination fails | Feature gate issue | Check `#[cfg(feature = "...")]` |
| Unsafe unsoundness | Invalid FFI usage | Add `// SAFETY:` comment, review logic |

## Reporting Validation Results

```markdown
## Validation Report
- [x] Code formatting: PASS
- [x] Clippy checks: PASS
- [x] MSRV (1.95): PASS
- [x] Feature combinations: PASS
- [x] Tests: PASS (all platforms)
- [x] Documentation: PASS

**Result**: ✅ Ready for PR
```
