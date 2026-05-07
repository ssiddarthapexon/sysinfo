<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan:

**Current Feature**: Process Data Integrity During Termination
**Plan**: docs/001-process-data-integrity/plan.md
**Specification**: docs/001-process-data-integrity/spec.md

**Key Context**:
- Rust crate for cross-platform system information retrieval
- Fixing bug where process termination causes data loss
- Adding status tracking APIs (is_data_complete, is_alive, last_refreshed)
- Platform-specific implementations: Linux, macOS, Windows, BSD
- Must preserve existing data when process terminates (don't overwrite with defaults)

**Important Files**:
- src/common/system.rs - Process public API
- src/unix/linux/process.rs - Linux implementation
- src/unix/apple/macos/process.rs - macOS implementation  
- src/windows/process.rs - Windows implementation
- src/unix/bsd/*/process.rs - BSD implementations

**Design Artifacts**:
- Research: docs/001-process-data-integrity/research.md
- Data Model: docs/001-process-data-integrity/data-model.md
- API Contracts: docs/001-process-data-integrity/contracts/api.md
- Quickstart: docs/001-process-data-integrity/quickstart.md
<!-- SPECKIT END -->
