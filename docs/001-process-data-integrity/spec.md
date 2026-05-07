# Specification: Process Data Integrity During Termination

**Feature**: Handle incomplete process data when process terminates during refresh
**Status**: In Progress
**Date**: May 7, 2026
**Issue**: [GitHub #6](https://github.com/ssiddarthapexon/sysinfo/issues/6)

## Executive Summary

Currently, when a process terminates during the middle of a system refresh operation, the process data becomes incomplete but callers are not notified. This causes consumers of the sysinfo API to operate on process objects with incorrect or default values without awareness. This specification addresses how to detect and preserve data integrity when processes terminate unexpectedly during data collection.

## Problem Statement

### Current Behavior

1. **Data Loss on Re-read**: If a process was previously successfully read and has cmdline/environ data, those values are overwritten with empty vectors when the process terminates during a subsequent refresh (because the `/proc` files no longer exist).

2. **Silent Incompleteness**: When a process has never been successfully read before and then terminates during the first refresh attempt, the fields are populated with default values (empty vectors). The caller has no way to know if this represents "no data" or "data collection failed."

3. **Uncertain State**: Consumers cannot reliably distinguish between:
   - A process that legitimately has no command-line arguments or environment variables
   - A process whose data is incomplete due to termination during collection
   - A process whose data is stale because it disappeared mid-refresh

### Impact

- **Data Corruption**: Applications relying on process cmdline/environ data make incorrect decisions based on incomplete information
- **Debugging Difficulty**: Intermittent bugs caused by race conditions during process termination are hard to diagnose
- **API Uncertainty**: Without explicit status signals, callers must implement defensive coding patterns or accept data uncertainty

## Objectives

1. **Preserve Previous Data**: When a process terminates during refresh, retain previously collected valid data instead of overwriting with defaults
2. **Signal Data Completeness**: Provide APIs for callers to determine whether process data is complete and reliable
3. **Track Liveliness**: Allow callers to identify whether a process was alive during the last refresh attempt
4. **Maintain Backward Compatibility**: Preserve existing public API contracts where possible

## User Scenarios & Testing

### Scenario 1: Long-lived Process Refresh Succeeds
**Actor**: System monitoring application  
**Context**: Monitoring a stable process that persists across refresh cycles  
**Actions**:
1. Call `system.refresh_processes()` while process is running
2. Read process.cmdline() and process.environ()
3. Call system.refresh_processes() again
4. Read the same process data

**Expected Outcome**: Data is available and consistent across refreshes

**Test**: `test_process_data_preserved_across_refreshes()`

---

### Scenario 2: Process Terminates During Refresh (Previously Read)
**Actor**: Monitoring tool tracking short-lived processes  
**Context**: Process that was successfully read now terminates mid-refresh  
**Actions**:
1. Successfully refresh process data (cmdline="sleep 1")
2. Process terminates
3. Call refresh_processes() while process is terminating
4. Read process.cmdline() and process.environ()
5. Check is_data_complete()

**Expected Outcome**: 
- cmdline() still returns "sleep 1" (old data preserved)
- is_data_complete() indicates data may be stale
- Process remains queryable with last-known state

**Test**: `test_process_data_preserved_on_termination()`

---

### Scenario 3: Process Terminates Before First Successful Read
**Actor**: Process inventory collector  
**Context**: Short-lived process that starts and stops before first refresh completes  
**Actions**:
1. Create short-lived process (sleep 0.1)
2. Call system.refresh_processes() while process is exiting
3. Locate process by PID
4. Read process data
5. Check is_data_complete() and is_alive()

**Expected Outcome**:
- is_alive() returns false
- is_data_complete() returns false
- Fields contain default values but caller is aware of status
- last_refreshed() timestamp is available

**Test**: `test_process_incomplete_data_signaled()`

---

### Scenario 4: Cross-platform Consistency
**Actor**: Desktop monitoring application  
**Context**: Same monitoring code runs on Linux, macOS, Windows, BSD  
**Actions**:
1. Create monitoring code using is_data_complete() and is_alive()
2. Run same test on each supported platform
3. Verify data integrity APIs work consistently

**Expected Outcome**: Data integrity status APIs work on all platforms

**Test**: `test_data_integrity_apis_cross_platform()`

---

## Functional Requirements

### FR1: Preserve Previously Collected Data
**Requirement**: Process fields must retain their previously collected values when process termination occurs during a refresh operation. Only update fields that are successfully read; leave unchanged fields with old values.

**Acceptance Criteria**:
- When a process termination occurs mid-refresh, previously populated cmdline and environ fields are not overwritten
- New process attempts to read data, failure to read does not clear old cached values
- On platforms with proc filesystem (Linux), ENOENT errors during read trigger preservation mode
- Data preservation works for all platform implementations (Linux, macOS, Windows, BSD)

---

### FR2: Data Completeness Status API
**Requirement**: Provide public API method `is_data_complete()` that indicates whether a process's data was fully collected in the last refresh.

**Acceptance Criteria**:
- `Process::is_data_complete() -> bool` returns true only if all fields were successfully populated in last refresh
- Returns false if process terminated/disappeared mid-refresh
- Returns false if process has never been successfully refreshed
- Method is available on all platform implementations

---

### FR3: Liveliness Status API
**Requirement**: Provide public API method `is_alive()` that indicates whether the process was detected as running during the last refresh.

**Acceptance Criteria**:
- `Process::is_alive() -> bool` returns true if process was found and readable during last refresh
- Returns false if process no longer exists or is unreadable
- Method is available on all platform implementations
- Status is updated with each refresh operation

---

### FR4: Last Refresh Timestamp API
**Requirement**: Provide public API method `last_refreshed()` that returns when the process data was last refreshed.

**Acceptance Criteria**:
- `Process::last_refreshed() -> Option<SystemTime>` returns timestamp of last refresh attempt
- Timestamp is updated on every refresh call regardless of success
- Returns None for processes never refreshed
- Enables callers to detect stale data (e.g., data older than 5 seconds)

---

### FR5: Cross-platform Implementation
**Requirement**: All data integrity features must be implemented consistently across all supported platforms.

**Acceptance Criteria**:
- Linux implementation preserves data and tracks status
- macOS implementation preserves data and tracks status  
- Windows implementation preserves data and tracks status
- BSD implementations (FreeBSD, NetBSD, OpenBSD) preserve data and track status
- Unknown/unsupported platforms gracefully handle status APIs
- All implementations pass the same test suite

---

### FR6: Backward Compatibility
**Requirement**: Changes must not break existing public API contracts.

**Acceptance Criteria**:
- All existing public methods remain unchanged in signature
- Existing process field types remain unchanged
- New fields are added to Process struct without removing existing fields
- New methods are additions, not replacements of existing methods
- Existing code continues to compile without modification

---

## Success Criteria

1. **Data Preservation Rate**: 95%+ of previously collected process data is retained when process terminates during refresh (measurable via test suite)
2. **API Coverage**: All three status APIs (is_data_complete, is_alive, last_refreshed) are implemented and tested on all 5+ supported platforms
3. **Test Success Rate**: 100% of new integration tests pass across all platforms (Linux, macOS, Windows, BSD variants)
4. **Backward Compatibility**: 0 breaking changes to public API; all existing code compiles without modification
5. **Documentation Quality**: All new public APIs have doc comments with examples; usage patterns documented in spec
6. **Performance**: No measurable performance regression in refresh operations (< 2% variance in timing)

---

## Key Entities

### Process Structure Enhancement

#### Current
```
Process {
    pid: Pid,
    name: String,
    cmdline: Vec<String>,
    environ: Vec<String>,
    ... other fields ...
}
```

#### Enhanced
```
Process {
    // Existing fields unchanged
    pid: Pid,
    name: String,
    cmdline: Vec<String>,
    environ: Vec<String>,
    ... other fields ...
    
    // New private fields for tracking
    #[doc(hidden)]
    is_data_complete: bool,  // Whether all fields were successfully populated
    
    #[doc(hidden)]
    is_alive: bool,  // Whether process was alive during last refresh
    
    #[doc(hidden)]
    last_refreshed: Option<SystemTime>,  // When data was last refreshed
}
```

### New Public Methods

```rust
impl Process {
    /// Returns whether all process data was successfully collected in the last refresh
    pub fn is_data_complete(&self) -> bool;
    
    /// Returns whether the process was alive and accessible during the last refresh
    pub fn is_alive(&self) -> bool;
    
    /// Returns the timestamp of the last refresh attempt
    pub fn last_refreshed(&self) -> Option<SystemTime>;
}
```

---

## Assumptions

1. **Proc filesystem availability**: Linux implementation assumes `/proc` filesystem; behavior on unusual Linux configurations documented
2. **Platform-specific accuracy**: Each platform implementation may have slightly different success conditions due to OS differences; specification documents minimum expected behavior
3. **Time availability**: All platforms can provide SystemTime for last_refreshed tracking
4. **No breaking changes acceptable**: Project maintains public API compatibility as highest priority
5. **Test infrastructure**: Test framework supports process lifecycle control (spawning, terminating, timing)
6. **Memory/Performance acceptable**: Tracking additional booleans and timestamps per Process is acceptable storage cost

---

## Dependencies & Constraints

### External Dependencies
- None (no new external crates required; uses std::time::SystemTime)

### Internal Dependencies  
- Must integrate with existing Process implementations across all platforms
- Must work with existing refresh_processes() infrastructure
- Changes to src/common/system.rs Process trait
- Changes to all platform-specific implementations

### Technical Constraints
- Must not introduce undefined behavior or race conditions
- Must work in multi-threaded contexts where applicable
- Must handle symlink and mount point variations across platforms
- Cannot assume process PIDs remain stable across refresh operations

### Scope Constraints
- Focuses only on Process data, not Disk/Network/CPU entities
- Does not add new field types to Process, only tracking booleans/timestamps
- Does not modify process collection frequency or strategy, only failure handling
- Does not add configuration options for preservation behavior (always preserve)

---

## Related Issues & References

- **GitHub Issue #6**: Hidden incomplete process data when process terminates while refreshing
- **GitHub Issue #5**: (Related example fix - Product printing bug)
- **Upstream Issue**: https://github.com/GuillaumeGomez/sysinfo (original crate)

---

## Next Steps

1. Review and clarify specification with stakeholders
2. Create detailed design artifacts and data model
3. Generate implementation tasks based on this specification
4. Begin platform-specific implementations (Linux first, then macOS/Windows/BSD)
5. Add comprehensive test coverage for all scenarios
