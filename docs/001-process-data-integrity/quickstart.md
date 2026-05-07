# Quick Start Guide: Process Data Integrity APIs

**Date**: May 7, 2026  
**Feature**: [Process Data Integrity During Termination](../plan.md)

---

## Overview

This guide provides practical examples of using the three new Process data integrity APIs:
- `is_data_complete()` - Check if all process fields were successfully collected
- `is_alive()` - Check if process was running during last refresh
- `last_refreshed()` - Get timestamp of last refresh attempt

---

## Scenario 1: Detect Incomplete Process Data

**Goal**: Find processes where data collection was incomplete and warn the user.

```rust
use sysinfo::System;

fn check_data_integrity(system: &System) {
    println!("=== Data Integrity Check ===\n");
    
    for (_pid, process) in system.processes() {
        if !process.is_data_complete() {
            eprintln!("⚠️  Incomplete data for process: {}", process.name());
            
            if process.is_alive() {
                eprintln!("   → Some fields failed to read (possible permission issue)");
            } else {
                eprintln!("   → Process may have terminated during refresh");
            }
            
            eprintln!("   → Last refresh: {:?}", process.last_refreshed());
        }
    }
}

fn main() {
    let mut system = System::new_all();
    system.refresh_processes();
    check_data_integrity(&system);
}
```

**Output**:
```
=== Data Integrity Check ===

⚠️  Incomplete data for process: sleep
   → Process may have terminated during refresh
   → Last refresh: Some(SystemTime { tv_sec: 1715132804, tv_nsec: 123456789 })
```

---

## Scenario 2: Filter for Complete Data Only

**Goal**: Work only with processes that have complete, reliable data.

```rust
use sysinfo::System;

fn get_complete_processes(system: &System) -> Vec<(&u32, &sysinfo::Process)> {
    system
        .processes()
        .iter()
        .filter(|(_, process)| process.is_data_complete())
        .collect()
}

fn main() {
    let mut system = System::new_all();
    system.refresh_processes();
    
    let complete = get_complete_processes(&system);
    println!("Processes with complete data: {}", complete.len());
    
    for (_pid, process) in complete {
        println!(
            "✓ {} - Memory: {} MB",
            process.name(),
            process.memory() / (1024 * 1024)
        );
    }
}
```

---

## Scenario 3: Detect Process Termination

**Goal**: Monitor a specific process and detect when it terminates.

```rust
use sysinfo::{System, Pid};
use std::time::Duration;
use std::thread;

fn monitor_process(target_pid: Pid) {
    let mut system = System::new_all();
    let mut process_was_alive = false;
    
    for refresh_count in 0..10 {
        system.refresh_processes();
        
        if let Some(process) = system.processes().get(&target_pid) {
            if process.is_alive() {
                if !process_was_alive {
                    println!("✓ Process found: {}", process.name());
                }
                process_was_alive = true;
            } else {
                if process_was_alive {
                    println!("✗ Process terminated: {}", process.name());
                    return;
                }
            }
        } else {
            println!("Process not in map at refresh #{}", refresh_count);
        }
        
        thread::sleep(Duration::from_millis(500));
    }
    
    if process_was_alive {
        println!("Process still alive after monitoring period");
    }
}

fn main() {
    // Get current process PID as example
    let my_pid = std::process::id() as Pid;
    monitor_process(my_pid);
}
```

---

## Scenario 4: Detect Stale Data

**Goal**: Alert user when process data hasn't been refreshed recently.

```rust
use sysinfo::System;
use std::time::{Duration, SystemTime};

fn check_data_freshness(system: &System, max_age_secs: u64) {
    println!("=== Checking Data Freshness (max age: {}s) ===\n", max_age_secs);
    
    let now = SystemTime::now();
    let max_age = Duration::from_secs(max_age_secs);
    let mut stale_count = 0;
    
    for (_pid, process) in system.processes() {
        if let Some(last_refresh) = process.last_refreshed() {
            let age = now.duration_since(last_refresh).unwrap_or_default();
            
            if age > max_age {
                println!(
                    "⏱️  STALE: {} - Last refresh: {:.2}s ago",
                    process.name(),
                    age.as_secs_f64()
                );
                stale_count += 1;
            }
        } else {
            println!("❓ NEVER: {} - Never been refreshed", process.name());
        }
    }
    
    println!("\nTotal stale processes: {}", stale_count);
}

fn main() {
    let mut system = System::new_all();
    system.refresh_processes();
    
    check_data_freshness(&system, 5);  // Alert if data older than 5 seconds
}
```

---

## Scenario 5: Comprehensive Process Report

**Goal**: Generate detailed report on process data integrity status.

```rust
use sysinfo::System;
use std::time::{Duration, SystemTime};

#[derive(Debug)]
struct ProcessStatus {
    name: String,
    pid: u32,
    data_complete: bool,
    is_alive: bool,
    last_refreshed_ago_ms: Option<u128>,
}

fn generate_process_report(system: &System) -> Vec<ProcessStatus> {
    let now = SystemTime::now();
    
    system
        .processes()
        .iter()
        .map(|(pid, process)| {
            let last_refreshed_ago_ms = process
                .last_refreshed()
                .and_then(|t| {
                    now.duration_since(t)
                        .ok()
                        .map(|d| d.as_millis())
                })
                .or_else(|| Some(0));  // 0 if just refreshed
            
            ProcessStatus {
                name: process.name().to_string(),
                pid: pid.as_u32(),
                data_complete: process.is_data_complete(),
                is_alive: process.is_alive(),
                last_refreshed_ago_ms,
            }
        })
        .collect()
}

fn print_report(statuses: &[ProcessStatus]) {
    println!("╔═══════════════════════════════════════════════════════════════╗");
    println!("║            Process Data Integrity Report                       ║");
    println!("╠═══════════════════════════════════════════════════════════════╣");
    
    for status in statuses {
        let complete_str = if status.data_complete { "✓" } else { "✗" };
        let alive_str = if status.is_alive { "🔄" } else { "💤" };
        let age_str = status
            .last_refreshed_ago_ms
            .map(|ms| format!("{:3}ms ago", ms))
            .unwrap_or_else(|| "N/A".to_string());
        
        println!(
            "│ {} {:3} {} {:30} {:20} │",
            complete_str, status.pid, alive_str, status.name, age_str
        );
    }
    
    println!("╚═══════════════════════════════════════════════════════════════╝");
}

fn main() {
    let mut system = System::new_all();
    system.refresh_processes();
    
    let report = generate_process_report(&system);
    print_report(&report);
}
```

**Output**:
```
╔═══════════════════════════════════════════════════════════════╗
║            Process Data Integrity Report                       ║
╠═══════════════════════════════════════════════════════════════╣
│ ✓   1 🔄 systemd                          0ms ago              │
│ ✓ 100 🔄 bash                             1ms ago              │
│ ✗ 200 💤 sh                             500ms ago              │
│ ✓ 300 🔄 firefox                          2ms ago              │
│ ✗ 400 💤 sleep                          999ms ago              │
╚═══════════════════════════════════════════════════════════════╝
```

---

## Scenario 6: Safe Process Data Access

**Goal**: Implement safe, defensive access to process data with automatic fallback for incomplete data.

```rust
use sysinfo::{System, Pid, ProcessStatus};

fn get_process_info_safely(system: &System, pid: Pid) -> String {
    match system.processes().get(&pid) {
        Some(process) => {
            if process.is_data_complete() {
                // Data is reliable - use it directly
                format!(
                    "Process: {} (PID: {})\n  Status: {:?}\n  Memory: {} MB\n  CPU: {:.2}%",
                    process.name(),
                    process.pid(),
                    process.status(),
                    process.memory() / (1024 * 1024),
                    process.cpu_usage()
                )
            } else if process.is_alive() {
                // Process exists but some data is incomplete
                format!(
                    "Process: {} (PID: {})\n  ⚠️  Data incomplete but process still running\n  Status: {:?}",
                    process.name(),
                    process.pid(),
                    process.status()
                )
            } else {
                // Process appears to be terminated
                format!(
                    "Process: {} (PID: {})\n  ⚠️  Process may have terminated\n  Last seen: {:?}",
                    process.name(),
                    process.pid(),
                    process.last_refreshed()
                )
            }
        }
        None => {
            format!("Process {} not found", pid)
        }
    }
}

fn main() {
    let mut system = System::new_all();
    system.refresh_processes();
    
    // Example: Get info about first process
    if let Some((pid, _)) = system.processes().iter().next() {
        println!("{}", get_process_info_safely(&system, *pid));
    }
}
```

---

## Scenario 7: Cross-Refresh Comparison

**Goal**: Track how process data changes across multiple refresh cycles.

```rust
use sysinfo::{System, Pid, ProcessStatus};
use std::collections::HashMap;
use std::time::Duration;
use std::thread;

#[derive(Clone, Debug)]
struct ProcessSnapshot {
    name: String,
    status: ProcessStatus,
    memory: u64,
    data_complete: bool,
    is_alive: bool,
}

fn take_snapshot(system: &System) -> HashMap<Pid, ProcessSnapshot> {
    system
        .processes()
        .iter()
        .map(|(pid, process)| {
            (
                *pid,
                ProcessSnapshot {
                    name: process.name().to_string(),
                    status: process.status(),
                    memory: process.memory(),
                    data_complete: process.is_data_complete(),
                    is_alive: process.is_alive(),
                },
            )
        })
        .collect()
}

fn compare_snapshots(
    before: &HashMap<Pid, ProcessSnapshot>,
    after: &HashMap<Pid, ProcessSnapshot>,
) {
    println!("=== Process Changes ===\n");
    
    // Find terminated processes
    for (pid, process) in before {
        if !after.contains_key(pid) {
            println!("❌ TERMINATED: {} (PID: {})", process.name, pid);
        }
    }
    
    // Find new processes
    for (pid, process) in after {
        if !before.contains_key(pid) {
            println!("✨ NEW: {} (PID: {})", process.name, pid);
        }
    }
    
    // Find data integrity changes
    for (pid, after_proc) in after {
        if let Some(before_proc) = before.get(pid) {
            if before_proc.data_complete && !after_proc.data_complete {
                println!(
                    "⚠️  DATA INCOMPLETE: {} (was complete, now incomplete)",
                    after_proc.name
                );
            }
        }
    }
}

fn main() {
    let mut system = System::new_all();
    
    system.refresh_processes();
    let snapshot1 = take_snapshot(&system);
    println!("First snapshot: {} processes", snapshot1.len());
    
    thread::sleep(Duration::from_secs(2));
    
    system.refresh_processes();
    let snapshot2 = take_snapshot(&system);
    println!("Second snapshot: {} processes\n", snapshot2.len());
    
    compare_snapshots(&snapshot1, &snapshot2);
}
```

---

## Common Patterns

### Pattern 1: Conditional Data Usage

```rust
for (_pid, process) in system.processes() {
    match (process.is_data_complete(), process.is_alive()) {
        (true, _) => {
            // Safe to use all fields
            println!("Memory: {}", process.memory());
        }
        (false, true) => {
            // Process exists but data is incomplete - use with caution
            if !process.cmd().is_empty() {
                println!("Command: {:?}", process.cmd());
            }
        }
        (false, false) => {
            // Process is gone - only use cached data
            println!("Process terminated: {}", process.name());
        }
    }
}
```

### Pattern 2: Data Freshness Check

```rust
use std::time::{Duration, SystemTime};

fn is_data_fresh(process: &sysinfo::Process, max_age: Duration) -> bool {
    if let Some(last_refresh) = process.last_refreshed() {
        let now = SystemTime::now();
        let age = now.duration_since(last_refresh).unwrap_or_default();
        age <= max_age
    } else {
        false  // Never refreshed = not fresh
    }
}
```

### Pattern 3: Collect Statistics

```rust
let stats = system.processes()
    .values()
    .fold(
        (0, 0, 0),
        |(complete, incomplete, terminated), process| {
            match (process.is_data_complete(), process.is_alive()) {
                (true, _) => (complete + 1, incomplete, terminated),
                (false, true) => (complete, incomplete + 1, terminated),
                (false, false) => (complete, incomplete, terminated + 1),
            }
        },
    );

println!("Complete: {}, Incomplete: {}, Terminated: {}", stats.0, stats.1, stats.2);
```

---

## Migration Guide

### Before (Old Code)

```rust
// Old code assumed silent data loss was acceptable
for (_pid, process) in system.processes() {
    if !process.cmd().is_empty() {
        println!("Command: {:?}", process.cmd());
    } else {
        // Can't tell if empty because: no args, permission denied, or process terminated
        println!("No command line available");
    }
}
```

### After (New Code with Data Integrity)

```rust
// New code can reliably detect why data is unavailable
for (_pid, process) in system.processes() {
    if !process.cmd().is_empty() {
        println!("Command: {:?}", process.cmd());
    } else if process.is_data_complete() {
        println!("Process has no command line arguments");
    } else if process.is_alive() {
        println!("Could not access command line (permission denied)");
    } else {
        println!("Process terminated before we could read data");
    }
}
```

---

## Summary

These scenarios demonstrate how the new data integrity APIs enable:
- **Robust error handling**: Distinguish between "no data" and "data unavailable"
- **Defensive programming**: Safely handle processes that disappear mid-refresh
- **Monitoring reliability**: Detect stale data and incomplete reads
- **Cross-platform consistency**: Same APIs work identically on all platforms

For more details, see:
- [API Contract](../contracts/api.md) - Detailed method specifications
- [Data Model](../data-model.md) - Field semantics and state transitions
- [Research](../research.md) - Platform-specific implementation details
