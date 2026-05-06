# Skill: Code Explorer

## Purpose
Navigate sysinfo's multi-platform codebase to understand existing implementations and patterns.

## When to Use
When analyzing an issue, locating affected code, or finding similar implementations to follow.

## Directory Mapping

### Common Interfaces (all platforms use these)
- `src/common/mod.rs` — Defines traits: `System`, `Process`, `Component`, etc.
- `src/common/system.rs` — System trait & implementations
- `src/common/disk.rs` — Disk trait
- `src/common/network.rs` — Network trait
- `src/common/component.rs` — Component (CPU, temperature) trait
- `src/common/user.rs` — User trait

### Platform-Specific Implementations

**Linux** (`src/unix/linux/`)
- `system.rs` — Parse /proc, /sys
- `process.rs` — Read process info from /proc
- `disk.rs` — Disk info from mount tables
- `cpu.rs` — CPU info from /proc/cpuinfo
- `network.rs` — Network stats from /proc/net

**macOS/iOS** (`src/unix/apple/`)
- `system.rs` — Use `sysctl`, Foundation framework
- `process.rs` — Use `libproc`, `ps_for_pid`
- `component.rs` — Temperature via IOKit
- `network.rs` — Use `getifaddrs`, kernel stats

**Windows** (`src/windows/`)
- `system.rs` — WMI queries, Windows API
- `process.rs` — PSAPI, CreateToolhelp32Snapshot
- `disk.rs` — GetDiskFreeSpaceEx
- `cpu.rs` — WMI CPU queries
- `network.rs` — GetIfTable, GetIpStatisticsEx

**FreeBSD** (`src/unix/bsd/freebsd/`)
- Uses `sysctl`, `libdevstat`, `libprocstat`
- Similar patterns to other BSD variants

### Key Files to Know
- `src/lib.rs` — Public API, feature gates, imports
- `Cargo.toml` — Feature definitions, dependencies per platform
- `src/sysinfo.h` — C interface for FFI
- `src/c_interface.rs` — C bindings

## Navigation Patterns

### How to Find Implementation for X
1. Check `src/common/[x].rs` for trait definition
2. Find `impl [Trait] for [PlatformStruct]` in platform dirs
3. Look at similar functionality for code patterns

### How to Check Feature Gates
1. Open `Cargo.toml` and find `[features]` section
2. See which modules are conditionally compiled
3. Check `#[cfg(feature = "...")]` in source

### How to Understand FFI Boundaries
1. Check `src/[platform]/ffi.rs` for C declarations
2. Look for `unsafe { ... }` blocks using FFI
3. Find corresponding safety comments

## Query Examples

**Q: How does sysinfo read CPU count on Linux?**
- File: `src/unix/linux/cpu.rs`
- Pattern: Read `/proc/cpuinfo`, parse CPU entries
- Key function: Parse number of entries per line starting with "processor"

**Q: How does macOS get process memory?**
- File: `src/unix/apple/process.rs`
- Pattern: Use `libproc.h`, call `proc_pidinfo`
- Key struct: `proc_regionwithinfo` from libproc

**Q: What features affect disk reading?**
- Search `Cargo.toml` for feature "disk"
- Check `src/lib.rs` for conditional imports
- Find implementations: `src/common/disk.rs`, platform-specific versions

## Common Patterns to Follow

### Adding a New Getter
1. Add method to trait in `src/common/[module].rs`
2. Implement for each platform under `src/[platform]/`
3. Handle "unknown" platform in `src/unknown/`
4. Add feature gate if applicable
5. Add test in `tests/`

### Handling Missing Data
- Return 0 for "unsupported" scenarios
- Return empty collections for optional data
- Document behavior in comments
- Consider feature gates for expensive operations

### Platform Detection
- Use `#[cfg(target_os = "...")]` at module level
- Use `#[cfg(any(target_os = "macos", target_os = "ios"))]` for groups
- Check `src/lib.rs` for how modules are conditionally included
