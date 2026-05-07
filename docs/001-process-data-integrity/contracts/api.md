# API Contract: Process Data Integrity APIs

**Date**: May 7, 2026  
**Feature**: [Process Data Integrity During Termination](../plan.md)

---

## Overview

This document specifies the public API contract for the three new Process data integrity methods. It defines method signatures, behavior guarantees, error handling, and platform-specific notes.

---

## Method 1: `is_data_complete()`

### Signature

```rust
impl Process {
    /// Returns whether all process data was successfully collected in the last refresh.
    pub fn is_data_complete(&self) -> bool
}
```

### Contract

**Preconditions**:
- None (method is always callable on valid Process reference)

**Return Value**:
- `true`: All process fields were successfully populated during the most recent refresh
- `false`: Process has never been refreshed, OR at least one field failed to populate in the last refresh

**Postconditions**:
- If returns `true`, then `is_alive()` also returns `true`
- If returns `false`, at least one of these is true:
  - Process has never been refreshed (`last_refreshed().is_none()`)
  - Process terminated during refresh (`is_alive() == false`)
  - Process became inaccessible during refresh (`is_alive() == false`)
  - Permission denied accessing some fields

**Side Effects**: None (pure query method)

**Thread Safety**: Safe to call concurrently from multiple threads. Return value reflects state at method call time.

---

### Semantics

| Value | Meaning | Recommended Action |
|-------|---------|-------------------|
| `true` | Process data is complete and reliable | Use data with confidence |
| `false` | Process data may be incomplete or stale | Check `is_alive()` to determine why |

---

### Examples

#### Example 1: Detecting Incomplete Data

```rust
let mut system = System::new_all();
system.refresh_processes();

for (_pid, process) in system.processes() {
    if !process.is_data_complete() {
        eprintln!(
            "Warning: Incomplete data for process {}",
            process.name()
        );
        
        if process.is_alive() {
            eprintln!("  Process still exists but some fields couldn't be read");
        } else {
            eprintln!("  Process may have terminated during refresh");
        }
    }
}
```

#### Example 2: Filtering for Complete Data Only

```rust
let mut system = System::new_all();
system.refresh_processes();

let complete_processes: Vec<_> = system
    .processes()
    .iter()
    .filter(|(_, proc)| proc.is_data_complete())
    .collect();

println!("Processes with complete data: {}", complete_processes.len());
```

---

### Platform-Specific Behavior

**Linux**:
- Returns `true` only if all `/proc/<pid>` files were successfully read
- Returns `false` if any read fails (ENOENT = process terminated, EACCES = permission denied)
- Most reliable platform for detecting completion

**macOS**:
- Returns `true` only if all libproc APIs succeeded
- Returns `false` if any libproc call returned 0 or error
- Cannot distinguish termination from permission denied at API level
- May return `false` more conservatively than other platforms

**Windows**:
- Returns `true` only if process handle was valid and all WMI queries succeeded
- Returns `false` if handle was invalid or any WMI query failed
- Explicit error codes available via GetLastError()

**BSD**:
- Returns `true` only if all sysctl calls succeeded
- Returns `false` if any sysctl call returned ESRCH or other error
- Similar to Linux in behavior and reliability

---

### Contract Guarantees

✅ **Always returns valid boolean**: Never panics, never returns undefined state  
✅ **Consistent with other APIs**: If `true`, then `is_alive() == true`  
✅ **Monotonic within refresh**: Between refreshes, value is stable  
✅ **Changes on each refresh**: Value may change based on latest refresh outcome  

---

### Anti-Contract (What This Does NOT Guarantee)

❌ **Does not indicate process is currently running**: Between refresh cycles, process could terminate  
❌ **Does not guarantee all fields have values**: Might indicate "read succeeded with empty data" (e.g., no environment variables)  
❌ **Is not persistent across refreshes**: May change from `true` to `false` if process terminates mid-refresh  

---

## Method 2: `is_alive()`

### Signature

```rust
impl Process {
    /// Returns whether the process was alive and accessible during the last refresh.
    pub fn is_alive(&self) -> bool
}
```

### Contract

**Preconditions**:
- None (method is always callable on valid Process reference)

**Return Value**:
- `true`: Process was found and readable during the most recent refresh
- `false`: Process has never been refreshed, OR was not found/not accessible during last refresh

**Postconditions**:
- If returns `true`, then `last_refreshed()` also returns `Some(...)` (was actually refreshed)
- If returns `false`:
  - Either process has never been refreshed, OR
  - Process not found (terminated), OR
  - Process completely unreadable (permission denied)

**Side Effects**: None (pure query method)

**Thread Safety**: Safe to call concurrently from multiple threads

---

### Semantics

| Value | Meaning | Implied State |
|-------|---------|---------------|
| `true` | Process was found and accessible in last refresh | `data_complete` could be `true` or `false` (partial read OK) |
| `false` | Process not found or inaccessible | Could be terminated, permission denied, or never refreshed |

---

### Examples

#### Example 1: Detect Process Termination

```rust
let mut system = System::new_all();
system.refresh_processes();

// First refresh
if let Some(process) = system.processes().get(&some_pid) {
    println!("Process running: {}", process.name());
    let was_alive = process.is_alive();
}

// Later...
system.refresh_processes();

if let Some(process) = system.processes().get(&some_pid) {
    if !process.is_alive() {
        println!("Process has terminated!");
    } else {
        println!("Process still running");
    }
}
```

#### Example 2: Handle Process Lifecycle

```rust
let mut system = System::new_all();

for i in 0..10 {
    system.refresh_processes();
    
    for (_pid, process) in system.processes() {
        if i == 0 {
            if process.is_alive() {
                println!("Found: {}", process.name());
            }
        } else {
            if !process.is_alive() {
                println!("Lost: {} - process terminated", process.name());
            }
        }
    }
    
    std::thread::sleep(Duration::from_secs(1));
}
```

---

### Platform-Specific Behavior

**Linux**:
- Returns `true` if `/proc/<pid>` directory exists and is readable
- Returns `false` if ENOENT (process gone) or EACCES (permission denied)
- Highly reliable for detecting termination

**macOS**:
- Returns `true` if libproc lookup succeeded
- Returns `false` if libproc lookup failed (could be terminated or permission denied)
- Cannot reliably distinguish termination from permission denied

**Windows**:
- Returns `true` if process handle could be opened
- Returns `false` if OpenProcess failed with ERROR_PROCESS_NOT_FOUND or ERROR_ACCESS_DENIED
- Fairly reliable based on handle validity

**BSD**:
- Returns `true` if sysctl lookup succeeded
- Returns `false` if ESRCH error (no such process)
- Similar to Linux in reliability

---

### Contract Guarantees

✅ **Always returns valid boolean**: Never panics  
✅ **Consistent with data_complete**: If `data_complete() == true`, then `is_alive() == true`  
✅ **Cleared on termination**: Will flip from `true` to `false` when process exits  
✅ **Observable**: Can be polled repeatedly in loop  

---

### Anti-Contract (What This Does NOT Guarantee)

❌ **Real-time accuracy**: May lag actual process state between refreshes  
❌ **Permission level consistency**: Permission denied on Linux marked as "not alive" (conservative)  
❌ **Definitive termination**: False doesn't guarantee process is dead, could be permission issue  

---

## Method 3: `last_refreshed()`

### Signature

```rust
impl Process {
    /// Returns the timestamp of the last refresh attempt for this process.
    pub fn last_refreshed(&self) -> Option<SystemTime>
}
```

### Contract

**Preconditions**:
- None (method is always callable on valid Process reference)

**Return Value**:
- `Some(time)`: Process was refreshed at this timestamp (success or failure)
- `None`: Process has never been refreshed

**Postconditions**:
- If returns `Some(t1)` on first call and `Some(t2)` on second call, then `t1 <= t2` (monotonic)
- Between refreshes, return value does not change
- After calling `system.refresh_processes()`, return value should be updated

**Side Effects**: None (pure query method)

**Thread Safety**: Safe to call concurrently. Return value is stable within a refresh cycle.

---

### Semantics

| Value | Meaning | Use Case |
|-------|---------|----------|
| `None` | Never been refreshed | New process just discovered |
| `Some(time)` | Was refreshed at this time (success or failure) | Detect stale data via duration since time |

---

### Examples

#### Example 1: Detect Stale Data

```rust
use std::time::{Duration, SystemTime};

let mut system = System::new_all();
system.refresh_processes();

let now = SystemTime::now();
let max_age = Duration::from_secs(5);

for (_pid, process) in system.processes() {
    if let Some(last_refresh) = process.last_refreshed() {
        let age = now.duration_since(last_refresh).unwrap_or_default();
        if age > max_age {
            println!(
                "Stale data: {} - refreshed {} ago",
                process.name(),
                age.as_secs()
            );
        }
    } else {
        // Never refreshed
        println!("No refresh data for {}", process.name());
    }
}
```

#### Example 2: Calculate Data Freshness

```rust
fn data_freshness_percentage(process: &Process) -> f64 {
    if let Some(last_refresh) = process.last_refreshed() {
        let age = SystemTime::now()
            .duration_since(last_refresh)
            .unwrap_or_default()
            .as_secs();
        
        let target_age = 5; // seconds
        let freshness = (1.0 - (age as f64 / target_age as f64)).max(0.0);
        (freshness * 100.0).min(100.0)
    } else {
        0.0 // Never refreshed
    }
}
```

#### Example 3: Track Refresh History

```rust
let mut system = System::new_all();
let mut refresh_times = HashMap::new();

for refresh_count in 0..10 {
    system.refresh_processes();
    
    for (_pid, process) in system.processes() {
        if let Some(time) = process.last_refreshed() {
            refresh_times
                .entry(process.pid())
                .or_insert_with(Vec::new)
                .push(time);
        }
    }
    
    std::thread::sleep(Duration::from_millis(100));
}
```

---

### Platform-Specific Behavior

**All Platforms**:
- Behavior is consistent across all platforms
- Timestamp is set to current time during refresh attempt (success or failure)
- Timestamp never decreases (monotonic increasing)

**Precision**:
- Linux: Millisecond precision (System::now() resolution)
- macOS: Millisecond precision
- Windows: Millisecond precision
- BSD: Millisecond precision

---

### Contract Guarantees

✅ **Always returns Option**: Never panics, always well-defined  
✅ **Monotonic**: Timestamps never decrease  
✅ **Updated on every refresh**: Changes on each system.refresh_processes() call  
✅ **Includes failed refreshes**: Timestamp set even if refresh data collection failed  
✅ **Cross-platform consistent**: Behavior identical across all platforms  

---

### Anti-Contract (What This Does NOT Guarantee)

❌ **Nanosecond precision**: Resolution is OS-dependent (typically milliseconds)  
❌ **Real-time clock**: Based on SystemTime, subject to system clock adjustments  
❌ **Frequency control**: Doesn't force refresh interval, just reports when it happened  

---

## API Consistency Rules

### Rule A1: Logical Consistency

```rust
// If data is complete, process MUST have been alive
assert!(
    !(process.is_data_complete() && !process.is_alive()),
    "Contradiction: data complete but process not alive"
);
```

**Enforcement**: Atomically update all three fields during refresh

---

### Rule A2: Timestamp Invariant

```rust
// If ever refreshed, timestamp must exist
assert!(
    !process.is_alive() || process.last_refreshed().is_some(),
    "If process was ever alive, must have refresh timestamp"
);
```

**Enforcement**: Set timestamp before checking process aliveness

---

### Rule A3: Monotonic Timestamps

```rust
// Timestamps never decrease across refresh cycles
let t1 = process.last_refreshed();
system.refresh_processes();
let t2 = process.last_refreshed();

if let (Some(time1), Some(time2)) = (t1, t2) {
    assert!(
        time1 <= time2,
        "Timestamps must be monotonically increasing"
    );
}
```

**Enforcement**: Always use SystemTime::now() during refresh

---

## Backward Compatibility

### Guarantee

✅ **No breaking changes**:
- Existing public methods unchanged
- Existing field types unchanged
- New methods are additions only
- New fields are private (crate-internal)

### Verification

All existing code like this continues to work without modification:

```rust
// Old code - should continue working
let mut system = System::new_all();
system.refresh_processes();

for (_pid, process) in system.processes() {
    println!("Process: {}", process.name());
    println!("CMD: {:?}", process.cmd());
}
// ← No changes needed to existing code
```

---

## Summary

The three new APIs (`is_data_complete()`, `is_alive()`, `last_refreshed()`) provide clear, well-defined contracts for determining process data integrity. They are:

- **Consistent**: All three follow logical invariants
- **Cross-platform**: Behavior is identical across all supported platforms
- **Thread-safe**: Safe for concurrent access
- **Non-breaking**: Fully backward compatible
- **Useful**: Enable robust application-level handling of process lifecycle
