# Project Constitution: sysinfo

**Constitution Version**: 1.0.0  
**Ratification Date**: 2026-05-07  
**Last Amended**: 2026-05-07

---

## Preamble

This constitution establishes the foundational principles and governance framework for the **sysinfo** project—a Rust crate providing reliable, cross-platform system information retrieval. All contributors, maintainers, and stakeholders are bound to uphold these principles in their work.

The sysinfo library serves as a critical dependency for system monitoring, diagnostics, and resource management across Linux, macOS, Windows, and BSD platforms. This constitution ensures that every change—from feature additions to bug fixes—maintains the integrity, stability, and trustworthiness that users depend upon.

---

## Core Principles

### Principle 1: Cross-Platform Compatibility

**Commitment**: The library MUST function correctly and consistently across all supported platforms (Linux, macOS, Windows, and BSD variants).

**Rules**:
- No platform-specific optimizations shall break behavior on other platforms.
- Platform-specific implementations MUST be accompanied by platform-neutral tests or equivalents.
- New public APIs MUST consider how they translate across all supported platforms before acceptance.
- Documentation MUST explicitly note any platform-specific limitations or behaviors.

**Rationale**: System information is inherently platform-dependent, yet users expect a unified, predictable interface. Maintaining compatibility ensures the library remains useful across the entire ecosystem.

---

### Principle 2: API Stability & Backwards Compatibility

**Commitment**: The public API MUST remain stable and backwards-compatible within major version releases (semver MAJOR.MINOR.PATCH).

**Rules**:
- Removing or changing public function signatures requires a MAJOR version bump.
- Adding new public APIs requires a MINOR version bump.
- Bug fixes and internal refactors use PATCH version bumps.
- Deprecation warnings MUST precede removals by at least one MAJOR release.
- Breaking changes MUST be documented in CHANGELOG.md with migration guidance.

**Rationale**: Users embed sysinfo into production systems. API stability allows them to upgrade with confidence, reducing friction and maintenance burden.

---

### Principle 3: Security & Safety

**Commitment**: The library MUST maintain memory safety and prevent unauthorized or unsafe access to system data.

**Rules**:
- All code MUST compile without `unsafe` blocks unless unavoidable (platform FFI calls are acceptable with rigorous review).
- Unsafe FFI bindings MUST be isolated, documented, and validated for correctness.
- Process data MUST NOT be accessed without appropriate permissions; errors MUST be gracefully handled.
- No privilege escalation exploits or unvalidated system calls are acceptable.
- Security issues MUST be reported and patched promptly.

**Rationale**: System information can expose sensitive details. The library must protect user privacy and system integrity at all times.

---

### Principle 4: Data Integrity & Reliability

**Commitment**: Process and system data MUST be accurate, consistent, and preserved even during edge cases (e.g., process termination, system state changes).

**Rules**:
- Data structures MUST NOT lose information when processes terminate or system state shifts.
- Refreshes MUST explicitly fail or degrade gracefully if data becomes unavailable; defaults MUST NOT be substituted without user awareness.
- APIs MUST expose data completeness (e.g., `is_data_complete()`, `last_refreshed()`) so callers can trust the data they receive.
- All data reads MUST be tested under stress (rapid updates, concurrent access, resource exhaustion).
- Breaking or incomplete data retrievals MUST be reported in debug logs for diagnostic purposes.

**Rationale**: Users rely on sysinfo for critical decisions (monitoring, alerting, scaling). Corrupted or silent data loss undermines trust and can lead to system failures.

---

## Governance

### Amendment Procedure

1. **Proposal**: Any principle amendment MUST be proposed and discussed in the issue tracker or discussions forum with clear rationale.
2. **Review Period**: A minimum 7-day review period MUST pass before any amendment takes effect.
3. **Consensus**: Amendments MUST receive approval from at least two project maintainers.
4. **Documentation**: All amendments MUST be recorded with a rationale update and version bump.

### Versioning

Constitution versions follow semantic versioning:
- **MAJOR**: Fundamental principle removal or redefinition (backwards incompatible governance).
- **MINOR**: New principle added or materially expanded guidance.
- **PATCH**: Clarifications, wording refinements, or non-semantic updates.

All amendments update `Last Amended` date to the approval date.

### Compliance Review

- **Quarterly**: Maintainers review whether ongoing work aligns with established principles.
- **Per Release**: Before each release, the constitution MUST be checked for conflicts with new features or changes.
- **Ad Hoc**: Any contributor MAY raise a compliance concern; disputes are resolved by maintainer consensus.

---

## Revision History

| Version | Date       | Change Summary                                                   |
|---------|------------|------------------------------------------------------------------|
| 1.0.0   | 2026-05-07 | Initial constitution: 4 core principles + governance framework   |
