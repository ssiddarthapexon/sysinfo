# Research: Process Data Integrity During Termination

**Date**: May 7, 2026  
**Feature**: [Process Data Integrity During Termination](plan.md)

---

## Overview

This research documents the technical findings necessary to implement process data integrity tracking across all sysinfo platforms. The investigation focuses on:

1. How each platform currently handles process termination during refresh
2. Platform-specific error codes and conditions indicating termination
3. Existing patterns for field-level refresh tracking
4. System state management capabilities for Process instances

---

## Research Findings

### R1: Current Process Termination Handling by Platform

#### Linux Implementation (`src/unix/linux/process.rs`)

**Current Behavior**:
- Reads process data from `/proc/<pid>/` filesystem
- Key files: `/proc/<pid>/cmdline`, `/proc/<pid>/environ`, `/proc/<pid>/stat`, etc.
- When process terminates, these files are deleted by kernel

**Error Handling**:
- File read operations return `std::io::Error` with kind `ErrorKind::NotFound`
- Current code responds by setting fields to empty collections
- **Problem**: Lines 937-939, 944-947 - No distinction between "empty data" and "file missing"

**Example - Problematic Code**:
```rust
// From linux/process.rs (simplified)
match read_to_string(&format!("/proc/{}/cmdline", pid)) {
    Ok(content) => {
        self.cmd = content.split('\0').map(|s| s.to_string()).collect();
    }
    Err(e) => {
        // Currently: silently overwrite with empty
        self.cmd = Vec::new();  // ← DATA LOSS HERE
    }
}
```

**Error Codes Observable**:
- `ENOENT` (2): File/directory doesn't exist - **indicates process terminated**
- `EACCES` (13): Permission denied - process exists but unreadable
- `EIO` (5): I/O error - filesystem issue
- `ESRCH` (3): No such process - used in some contexts

**Distinguishing Termination**:
```rust
// ENOENT + valid PID format = process terminated
// EACCES + valid PID = process exists but permission denied
// ESRCH = no process with that PID exists (kernel confirmation)
```

**Cross-Platform Consideration**: Linux is most reliable because `/proc` is atomic - if directory vanishes, process is definitely gone.

---

#### macOS Implementation (`src/unix/apple/macos/process.rs`)

**Current Behavior**:
- Uses `libproc` APIs (Darwin-specific system library)
- Key functions: `libproc::proc_pidinfo()`, `libproc::proc_listpids()`, etc.
- No `/proc` filesystem; data is kernel-derived

**Error Handling**:
- libproc functions return `0` or `NULL` pointer on failure
- No errno-style error codes directly; success/failure implicit in return value
- Current code checks return value and uses default values on failure

**Error Conditions for Termination**:
- `libproc::proc_pidinfo()` returns 0 → process not found
- `mach_port_t` invalid → process no longer valid
- **No explicit ENOENT equivalent** - termination is implicit

**Distinguishing Termination**:
```rust
// Check if process lookup succeeds
let info = libproc::proc_pidinfo(pid, PROC_PIDVNODEPATHINFO, ...);
if info.is_null() || result == 0 {
    // Could be:
    // 1. Process terminated
    // 2. Permission denied (common for other user processes)
    // Cannot always distinguish at API level
}
```

**Challenge**: macOS provides limited insight into *why* a lookup failed. Success = process definitely alive; failure could be termination OR permission.

---

#### Windows Implementation (`src/windows/process.rs`)

**Current Behavior**:
- Uses WMI (Windows Management Instrumentation) API via `windows` crate
- Alternative: Direct Win32 APIs (`OpenProcess`, `GetProcessImageFileName`, etc.)
- Process handle validity determines access

**Error Handling**:
- `OpenProcess()` returns `NULL` handle on failure with `GetLastError()`
- WMI queries return `S_FALSE` or error codes for missing processes
- Current code checks return values and defaults to empty strings

**Error Codes for Termination**:
- `ERROR_PROCESS_NOT_FOUND` (ERROR_NOT_FOUND): Process no longer exists
- `ERROR_ACCESS_DENIED` (5): Process exists but user lacks permissions
- `ERROR_INVALID_HANDLE` (6): Handle is no longer valid (process terminated)
- `HRESULT` codes from WMI (e.g., `0x80041010` for "object not found")

**Distinguishing Termination**:
```rust
// Direct Win32 approach
let handle = unsafe { OpenProcess(PROCESS_QUERY_INFORMATION, false, pid) };
if handle.is_null() {
    let error = GetLastError();
    match error {
        ERROR_PROCESS_NOT_FOUND => { /* Process terminated */ }
        ERROR_ACCESS_DENIED => { /* Permission denied */ }
        _ => { /* Other error */ }
    }
}
```

**Cross-Platform Note**: Windows provides most explicit error codes for distinguishing termination from permission issues.

---

#### BSD Implementations (`src/unix/bsd/*/process.rs`)

**Variants**: FreeBSD, NetBSD, OpenBSD (and potentially others)

**Current Behavior** (varies by BSD variant):
- FreeBSD/NetBSD: Use `/proc` filesystem similar to Linux (when available)
- All use `sysctl` APIs as fallback/alternative (e.g., `sysctl(CTL_KERN, KERN_PROC, ...)`)
- Alternative: `procfs` kernel module

**Error Handling**:
- `sysctl()` returns `-1` on error with errno set
- Error codes similar to POSIX: `ESRCH` (No such process), `EACCES` (Permission denied)
- If `/proc` is used, similar behavior to Linux

**Error Codes for Termination**:
- `ESRCH` (3): No such process (Darwin/FreeBSD/NetBSD convention)
- `ENOENT` (2): No such file (if using `/proc` filesystem)
- `EACCES` (13): Permission denied (process exists, not accessible)

**Distinguishing Termination**:
```rust
// sysctl approach (BSD)
let mut info = /* process info structure */;
let result = unsafe { 
    sysctl(
        &mut mib,
        mib_len as u32,
        &mut info as *mut _,
        &mut info_size,
        ptr::null_mut(),
        0,
    ) 
};

if result == -1 {
    match errno() {
        ESRCH => { /* Process terminated */ }
        EACCES => { /* Permission denied */ }
        _ => { /* Other error */ }
    }
}
```

**Consistency**: BSD implementations are more consistent with Linux than macOS in error reporting.

---

### R2: Existing Field-Level Refresh Patterns in Codebase

#### Pattern Analysis

**Current Refresh Strategy**:
The codebase follows a field-by-field collection approach where each field is independently fetched and updated. Example from `linux/process.rs`:

```rust
pub fn refresh(&mut self) {
    // Read CPU information
    if let Ok(stat_data) = read_stat_file(&self.pid) {
        self.cpu_usage = extract_cpu(stat_data);
    } else {
        self.cpu_usage = 0.0;  // Default on error
    }

    // Read command line
    match read_file(&format!("/proc/{}/cmdline", self.pid)) {
        Ok(content) => self.cmd = parse_cmdline(content),
        Err(_) => self.cmd = Vec::new(),  // Overwrites old data!
    }

    // Read environment
    match read_file(&format!("/proc/{}/environ", self.pid)) {
        Ok(content) => self.environ = parse_environ(content),
        Err(_) => self.environ = Vec::new(),  // ← BUG: Loses previous data
    }
}
```

**Problem Identified**: 
- Each field is independently reset to default on read failure
- No mechanism to distinguish "this field has no data" from "read operation failed"
- No tracking of which fields succeeded vs. failed

#### Recommended Pattern

To implement data preservation, modify refresh to:

```rust
pub fn refresh(&mut self) {
    let now = SystemTime::now();
    let mut all_fields_read = true;

    // Attempt to read each field, preserving old values on error
    match read_file(&format!("/proc/{}/cmdline", self.pid)) {
        Ok(content) => self.cmd = parse_cmdline(content),
        Err(e) => {
            // On error, KEEP old value instead of overwriting
            if is_process_not_found(e) {
                self.is_alive = false;
            }
            all_fields_read = false;
        }
    }

    match read_file(&format!("/proc/{}/environ", self.pid)) {
        Ok(content) => self.environ = parse_environ(content),
        Err(e) => {
            // Keep old value
            if is_process_not_found(e) {
                self.is_alive = false;
            }
            all_fields_read = false;
        }
    }

    // Update tracking fields
    self.last_refreshed = Some(now);
    self.is_data_complete = all_fields_read && self.is_alive;
}
```

---

### R3: System State Management for Processes

#### Current Architecture

**Process Storage**:
```rust
// In System struct
pub processes: HashMap<Pid, Process>
```

**Lifecycle**:
1. **Creation**: New Process instance created when system first discovers a running PID
2. **Updates**: `system.refresh_processes()` updates existing Process instances in-place
3. **Removal**: Processes are NOT explicitly removed from HashMap (persists even after process exits)
4. **Persistence**: Old Process instances remain queryable even after process terminates

**Implication for Data Preservation**:
- ✅ We CAN preserve old field values because Process instances persist in HashMap
- ✅ Process lifespan in HashMap > process lifespan in OS
- ✅ Can track multiple refresh attempts on same Process instance

#### State Mutation Architecture

**Where Process state changes**:

1. **common/system.rs** - `System::refresh_processes()`:
   - Iterates current OS process list
   - Calls platform-specific `update_process_status()` for each PID
   - Passes mutable reference to Process instance

2. **Platform-specific refresh** (e.g., `linux/process.rs`):
   - Implements the actual field collection
   - Reads platform-specific data sources
   - Calls methods like `set_cmdline()`, `set_environ()`, etc.

3. **Process struct** - stores collected data:
   - Fields are private (accessed via getters)
   - Mutable access limited to refresh operations
   - Persistence across refresh cycles

#### How to Integrate Tracking Fields

**Option A: Centralized Tracking** (Recommended)
- Track `is_data_complete`, `is_alive`, `last_refreshed` in common `Process` struct
- All platforms update these fields consistently
- Benefits: Single source of truth, easier testing

**Option B: Platform-Specific Tracking**
- Each platform manages tracking internally
- Expose via trait methods
- Downside: More complex, more duplication

**Recommendation**: Use Option A - centralized tracking in Process struct with platform implementations reporting success/failure status.

---

### R4: Error Code Mapping by Platform

This matrix helps implementations distinguish termination vs. other errors:

| Platform | File Not Found | Process Not Found | Permission Denied | Other |
|----------|---|---|---|---|
| **Linux** | ENOENT (2) | ESRCH (3) | EACCES (13) | EIO (5), etc. |
| **macOS** | N/A (libproc) | libproc returns 0 | libproc returns 0 | libproc returns -1 |
| **Windows** | ERROR_FILE_NOT_FOUND (2) | ERROR_PROCESS_NOT_FOUND (3) | ERROR_ACCESS_DENIED (5) | Various HRESULT |
| **BSD** | ENOENT (2) | ESRCH (3) | EACCES (13) | EIO (5), etc. |

**Key Insight**: 
- Linux/BSD are consistent (ENOENT/ESRCH pattern)
- macOS requires checking libproc return values
- Windows has explicit error codes

---

## Platform-Specific Implementation Guidance

### Linux Strategy

**Error Code Handling**:
```rust
match read_operation() {
    Ok(data) => {
        self.field = parse_data(data);
    }
    Err(e) => {
        match e.kind() {
            std::io::ErrorKind::NotFound => {
                // ENOENT: Process terminated
                self.is_alive = false;
                // KEEP old field value
            }
            std::io::ErrorKind::PermissionDenied => {
                // EACCES: Permission denied
                // KEEP old field value
                // Don't mark is_alive = false (process may still exist)
            }
            _ => {
                // Other errors
                // KEEP old field value
            }
        }
        all_read_successfully = false;
    }
}
```

**Advantages**: Atomic `/proc` filesystem provides definitive "process no longer exists" signal

---

### macOS Strategy

**Challenge**: libproc doesn't provide errno-style codes. Strategy: assume successful lookup = alive, failed lookup = unknown (could be terminated or permission denied).

```rust
let result = unsafe {
    libproc::proc_pidinfo(
        self.pid,
        PROC_PIDVNODEPATHINFO,
        &mut info,
    )
};

if result > 0 {
    // Successfully read
    self.field = extract_field(&info);
} else {
    // Read failed
    // Could be: (1) Process terminated, (2) Permission denied
    // macOS doesn't distinguish at API level
    // Conservative approach: treat as "not complete"
    all_read_successfully = false;
    // KEEP old field value
}
```

**Advantages**: libproc is kernel-backed, reliable for running processes  
**Disadvantages**: Limited error diagnostics, can't always confirm termination vs. permission

---

### Windows Strategy

**Leverage explicit error codes**:
```rust
unsafe {
    let handle = OpenProcess(
        PROCESS_QUERY_INFORMATION,
        false,
        self.pid,
    );

    if handle.is_null() {
        let error = GetLastError();
        match error {
            ERROR_PROCESS_NOT_FOUND => {
                // Process definitely terminated
                self.is_alive = false;
            }
            ERROR_ACCESS_DENIED => {
                // Permission denied, process may still exist
                // Conservative: don't mark as dead
            }
            _ => {}
        }
        // KEEP old field values
    } else {
        // Successfully opened process
        // Read field data here
        // ...
        CloseHandle(handle);
    }
}
```

**Advantages**: Most explicit error reporting  
**Disadvantages**: Requires Win32 API understanding across different Windows versions

---

### BSD Strategy

**Similar to Linux** but use sysctl as alternative:

```rust
match sysctl_read_process_info(self.pid) {
    Ok(info) => {
        self.field = extract_field(&info);
    }
    Err(e) => {
        let errno_val = errno();
        match errno_val {
            ESRCH => {
                // No such process: definitely terminated
                self.is_alive = false;
            }
            EACCES => {
                // Permission denied
                all_read_successfully = false;
            }
            _ => {
                all_read_successfully = false;
            }
        }
        // KEEP old field values
    }
}
```

---

## Conclusions & Recommendations

### R-1: Data Preservation is Feasible
✅ All platforms allow preservation of old field values during read failures  
✅ Process instances persist in System's HashMap across refresh cycles  
✅ No breaking changes required to achieve preservation

### R-2: Termination Detection is Reliable (Mostly)
✅ Linux/BSD: ENOENT/ESRCH provide definitive signals  
✅ Windows: Explicit error codes available  
⚠️ macOS: Limited error diagnostics, requires conservative assumptions

### R-3: Cross-Platform Consistency is Achievable
✅ All platforms can implement `is_data_complete()`, `is_alive()`, `last_refreshed()`  
✅ Behavior will be consistent across platforms (same method semantics)  
⚠️ macOS may be more conservative (mark data incomplete more often due to permission issues)

### R-4: Backward Compatibility is Preserved
✅ New tracking fields can be private (crate-visible)  
✅ New methods are additions only  
✅ Existing behavior remains unchanged for existing API users

---

## Next Steps

1. ✅ **Research Complete**: All R1-R4 findings documented
2. ⏳ **Design Phase**: Create data-model.md and contracts
3. ⏳ **Implementation**: Platform-specific implementations based on this research
4. ⏳ **Testing**: Comprehensive tests covering all termination scenarios

---

## References

- Linux `/proc` filesystem documentation
- macOS libproc documentation
- Windows Win32 API documentation  
- BSD sysctl documentation
- POSIX error codes (errno.h)
- Original sysinfo crate: https://github.com/GuillaumeGomez/sysinfo
