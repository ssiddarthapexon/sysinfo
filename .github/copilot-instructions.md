<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan:

**Current Feature**: Fix Product Example in simple.rs
**Plan**: docs/002-product-example-fix/plan.md
**Specification**: docs/002-product-example-fix/spec.md

**Key Context**:
- Rust crate for cross-platform system information retrieval
- Fixing example bug where Product type prints instead of product data
- Example demonstrates correct Product API usage (static methods)
- Platform-specific handling: SKU not available on macOS
- Output displays hardware product name, family, version, serial, UUID, vendor, SKU

**Important Files**:
- examples/simple.rs - Example file with the bug (line 340)
- src/common/system.rs - Product API definition (lines 1045-1170)
- docs/002-product-example-fix/ - Feature documentation

**Design Artifacts**:
- Research: docs/002-product-example-fix/research.md
- Data Model: docs/002-product-example-fix/data-model.md
- API Contracts: docs/002-product-example-fix/contracts/api.md
- Quickstart: docs/002-product-example-fix/quickstart.md
<!-- SPECKIT END -->
