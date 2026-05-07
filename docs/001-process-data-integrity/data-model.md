# Data Model: Process Data Integrity During Termination

**Date**: May 7, 2026  
**Feature**: [Process Data Integrity During Termination](plan.md)

---

## Overview

This document defines the data model for tracking process data integrity across refresh cycles. It describes:

1. The Process entity with new tracking fields
2. Field semantics and invariants
3. State transition diagrams
4. Validation rules
5. Platform-specific considerations

---

## Process Entity Enhanced Model

### Current Process Structure

```rust
pub struct Process {
    // Platform-specific internal state
    // (not shown here - varies by platform)
    
    pub pid: Pid,
    pub name: String,
    pub memory: u64,
    pub virtual_memory: u64,
    pub cpu_usage: f32,
    pub cmdline: Vec<String>,
    pub environ: Vec<String>,
    pub status: ProcessStatus,
    pub run_time: u64,
    pub start_time: u64,
    // ... additional fields ...
}
```

### Enhanced Process Structure

```rust
pub struct Process {
    // Existing fields (unchanged - shown for context)
    pub pid: Pid,
    pub name: String,
    pub memory: u64,
    pub virtual_memory: u64,
    pub cpu_usage: f32,
    pub cmdline: Vec<String>,
    pub environ: Vec<String>,
    pub status: ProcessStatus,
    pub run_time: u64,
    pub start_time: u64,
    
    // NEW PRIVATE TRACKING FIELDS
    // These are implementation details not exposed in public API
    // Accessed only via public methods
    
    /// Whether all process data was successfully collected in the last refresh.
    /// 
    /// - `true`: All readable fields were populated
    /// - `false`: Process terminated, became inaccessible, or never refreshed
    #[doc(hidden)]
    pub(crate) data_complete: bool,
    
    /// Whether the process was alive and accessible in the last refresh.
    ///
    /// - `true`: Process was found and readable
    /// - `false`: Process was not found or completely unreadable
    #[doc(hidden)]
    pub(crate) alive: bool,
    
    /// Timestamp of the last refresh attempt (success or failure).
    ///
    /// - `Some(time)`: Process was refreshed at this time
    /// - `None`: Process has never been refreshed
    #[doc(hidden)]
    pub(crate) last_refreshed_at: Option<SystemTime>,
}
```

### Public Accessor Methods

```rust
impl Process {
    pub fn is_data_complete(&self) -> bool {
        self.data_complete
    }

    pub fn is_alive(&self) -> bool {
        self.alive
    }

    pub fn last_refreshed(&self) -> Option<SystemTime> {
        self.last_refreshed_at
    }
}
```

---

## Field Semantics & Invariants

### Field: `data_complete: bool`

**Semantic**:  
Indicates whether all process fields were successfully populated in the last refresh operation. This is used to signal data integrity to consumers.

**Semantics**:

| Value | Meaning |
|-------|---------|
| `true` | All process fields were successfully read and are reliable |
| `false` | At least one field failed to read, or process never refreshed |

**Invariants**:
1. `data_complete == true` implies `alive == true` (if data is complete, process must have been alive)
2. `data_complete == false` does NOT imply `alive == false` (process could be inaccessible due to permissions)

**Transitions**:
- Created: `false` (initial state, never refreshed)
- After refresh success: `true`
- After refresh failure: `false`
- After process termination during refresh: `false`

**Never Reset to Default**: When refresh fails, `data_complete` remains `false`; it's never reset to a default value. Once set `true` by a successful refresh, it only becomes `false` when a later refresh fails.

---

### Field: `alive: bool`

**Semantic**:  
Indicates whether the process was detected as running and accessible during the last refresh attempt. This helps callers understand if a process exit occurred.

**Semantics**:

| Value | Meaning |
|-------|---------|
| `true` | Process was found and accessible in the last refresh |
| `false` | Process was not found (terminated), or became completely inaccessible |

**Invariants**:
1. `alive == true` and `last_refreshed_at.is_some()` → must have just been refreshed
2. `alive == false` → process is either terminated or severely restricted (no readable data)
3. `alive == false` and `data_complete == true` → IMPOSSIBLE (contradiction)

**Transitions**:
- Created: `false` (not yet verified to be alive)
- After refresh success: `true` (process was found and readable)
- After process not found (ENOENT/ESRCH): `false`
- After permission denied: `false` (conservative - we can't read it, so we can't confirm it's alive)
- After any other read failure: `false` (conservative approach)

**Note**: `alive` is more about "process was accessible to us" rather than "process was running". If permission denied, we set `alive = false` conservatively.

---

### Field: `last_refreshed_at: Option<SystemTime>`

**Semantic**:  
Tracks when the process was last refreshed (attempted), whether the refresh succeeded or failed. Allows callers to detect stale data.

**Semantics**:

| Value | Meaning |
|-------|---------|
| `None` | Process has never been refreshed |
| `Some(time)` | Process was refreshed at this time (success or failure) |

**Invariants**:
1. `last_refreshed_at.is_some()` implies the refresh operation was attempted at least once
2. `last_refreshed_at.is_some()` but `data_complete == false` → refresh was attempted but failed
3. Time values always increase monotonically (refresh times never go backwards)

**Transitions**:
- Created: `None` (no refresh yet)
- After any refresh attempt: `Some(now)` (even if it failed)
- Never decreases; once set to a time, only increases on next refresh

**Use Case**: Allows callers to calculate data age:
```rust
let age = SystemTime::now()
    .duration_since(process.last_refreshed()?)
    .unwrap_or_default();
```

---

## State Transition Diagram

```
[Initial State]
  data_complete: false
  alive: false
  last_refreshed_at: None
       ↓
       
[Refresh Attempted]
  last_refreshed_at: Some(now)
       ↓
       ├─ [All Fields Read Successfully]
       │  data_complete: true
       │  alive: true
       │  last_refreshed_at: Some(now)
       │
       └─ [Some/All Fields Failed to Read]
          ├─ Error: NotFound (ENOENT/ESRCH)
          │  data_complete: false
          │  alive: false
          │  last_refreshed_at: Some(now)
          │  Fields: OLD VALUES PRESERVED ← KEY POINT
          │
          └─ Error: PermissionDenied/Other
             data_complete: false
             alive: false  ← conservative
             last_refreshed_at: Some(now)
             Fields: OLD VALUES PRESERVED ← KEY POINT
```

---

## Scenarios & State Examples

### Scenario 1: Normal Process Lifecycle

```
Time T0: Process Created
  Process instance created in HashMap
  State: data_complete=false, alive=false, last_refreshed_at=None

Time T1: First Refresh (Successful)
  System discovers process and reads all fields
  State: data_complete=true, alive=true, last_refreshed_at=Some(T1)
  Fields: name="sleep", cmd=["sleep", "10"], environ=[...], etc.

Time T2: Second Refresh (Successful)
  Process still running, all fields updated
  State: data_complete=true, alive=true, last_refreshed_at=Some(T2)
  Fields: name="sleep", cmd=["sleep", "10"], environ=[...], etc.

Time T3: Third Refresh (Process Terminated Mid-Refresh)
  Process terminates after name/memory are read, before cmd/environ
  OS removes /proc/<pid> directory
  Attempt to read cmd: ENOENT ← process gone!
  State: data_complete=false, alive=false, last_refreshed_at=Some(T3)
  Fields: name="sleep" (kept), cmd=["sleep", "10"] (old value kept!), environ=[...]
  
Result: Caller can detect that data is stale AND preserve what was readable
```

---

### Scenario 2: Process With Permissions Issue

```
Time T0: Process Created
  System discovers process (owned by different user)
  State: data_complete=false, alive=false, last_refreshed_at=None

Time T1: First Refresh (Permission Denied)
  Attempt to read /proc/<pid>/cmdline: Permission denied (EACCES)
  State: data_complete=false, alive=false, last_refreshed_at=Some(T1)
  Fields: name="<unknown>", cmd=[], environ=[], etc.
  
Note: Process likely IS running, but we can't access it
      is_alive=false reflects "we can't confirm it's alive"
```

---

### Scenario 3: Very Short-Lived Process

```
Time T0: Process Created
  System discovers process that will exit in 10ms
  State: data_complete=false, alive=false, last_refreshed_at=None

Time T1: First Refresh Attempted (But Process Already Exited)
  Attempt to read /proc/<pid>: No such file/directory (ENOENT)
  State: data_complete=false, alive=false, last_refreshed_at=Some(T1)
  Fields: cmd=[], environ=[], name="<unknown>"
  
Result: Caller knows refresh was attempted but data is incomplete
        Can see is_alive()=false and is_data_complete()=false
```

---

## Validation Rules

### Rule V1: Consistency Invariants

When `is_data_complete() == true`, these MUST hold:
- ✅ `is_alive() == true` (data can only be complete if process was alive)
- ✅ `last_refreshed().is_some() == true` (must have been refreshed)

**Enforcement**: In refresh logic, atomically set all three fields together

---

### Rule V2: Monotonic Timestamps

`last_refreshed()` timestamps MUST be monotonically increasing:
- ✅ Each refresh sets timestamp to current time
- ✅ Never decreases
- ✅ Multiple refreshes in same millisecond OK (timestamps may be equal)

**Enforcement**: Always use current `SystemTime::now()` during refresh

---

### Rule V3: No Forced Data Loss

When a field read fails, the old value MUST be preserved:
- ✅ Previous value in `process.cmdline` is kept when read fails
- ✅ Previous value in `process.environ` is kept when read fails
- ✅ Only update if read succeeds

**Enforcement**: Use match/if let to check read success before assignment

---

### Rule V4: No Undefined `data_complete` Transitions

The `data_complete` flag can only transition:
- `false → true` (all fields read successfully)
- `true → false` (any field fails to read)
- Never: `true → true` (stays true)
- Never: `false → false` unchanged (should stay false unless refresh attempts again)

**Enforcement**: Set `data_complete` only after checking all field reads

---

## Platform-Specific Data Model Notes

### Linux Data Model

```rust
struct ProcessLinux {
    // Existing fields
    pid: Pid,
    // ...
    
    // Status tracking
    is_root: bool,
    disabled: bool,  // Avoid re-reading permission-denied processes
    
    // New tracking fields (inherited from Process)
    data_complete: bool,
    alive: bool,
    last_refreshed_at: Option<SystemTime>,
}
```

**Platform-Specific Considerations**:
- Linux provides ENOENT when process is truly gone
- Can detect termination with high confidence
- `/proc/<pid>` removal is atomic - no race conditions

---

### macOS Data Model

```rust
struct ProcessMacOS {
    // Existing fields
    pid: Pid,
    // ...
    
    // New tracking fields (inherited from Process)
    data_complete: bool,
    alive: bool,
    last_refreshed_at: Option<SystemTime>,
}
```

**Platform-Specific Considerations**:
- macOS libproc doesn't provide error codes
- Must treat failed libproc call as "unknown" (could be termination or permission)
- Be conservative: mark data incomplete on any libproc failure
- Cannot always distinguish termination from permission denied

---

### Windows Data Model

```rust
struct ProcessWindows {
    // Existing fields
    pid: Pid,
    handle: Option<Handle>,
    // ...
    
    // New tracking fields (inherited from Process)
    data_complete: bool,
    alive: bool,
    last_refreshed_at: Option<SystemTime>,
}
```

**Platform-Specific Considerations**:
- Windows provides explicit error codes (ERROR_PROCESS_NOT_FOUND, etc.)
- Can detect termination with moderate confidence
- Handle validity changes when process exits
- Use GetLastError() to distinguish failure types

---

### BSD Data Model

```rust
struct ProcessBSD {
    // Existing fields
    pid: Pid,
    // ...
    
    // New tracking fields (inherited from Process)
    data_complete: bool,
    alive: bool,
    last_refreshed_at: Option<SystemTime>,
}
```

**Platform-Specific Considerations**:
- BSD provides ESRCH error code for "no such process"
- Similar to Linux in error reporting
- sysctl() is reliable for getting process info
- ENOENT if using /proc filesystem (variant-dependent)

---

## Data Model Validation Checklist

- ✅ Three new private fields defined
- ✅ Public accessor methods specified
- ✅ Field semantics documented clearly
- ✅ State transitions diagrammed
- ✅ Invariants listed and enforceable
- ✅ Validation rules specified
- ✅ Example scenarios walkthrough successful
- ✅ Platform-specific considerations noted
- ✅ Backward compatibility verified (no breaking changes)

---

## Summary

The enhanced data model for Process adds minimal storage overhead (2 bools + 1 Option<SystemTime> ≈ 16 bytes) while providing critical signals for data integrity. The model maintains backward compatibility, enforces consistency invariants, and enables callers to make informed decisions about data reliability.
