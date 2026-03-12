---
applyTo: "docs/**/*.md"
---

Documentation in this repository should be concise, technical, and directly tied to the implemented RTL and tests.

## Module docs (`docs/<module>.md`)

When writing or updating module documentation:

- Start with a short overview: what the module does and where it fits in `docs/ARCHITECTURE.md`.
- Document the interface:
  - Port name, direction, width, and meaning (tables are preferred).
- Describe control flow:
  - FSM states and transitions (ASCII or Mermaid diagrams are fine).
- Note any key constraints:
  - Timing assumptions, backpressure/handshake semantics, reset behavior.
- Cross-link the code:
  - `rtl/<module>.v` and `tb/<module>_tb.v`.

## Plan/architecture edits

- `docs/ARCHITECTURE.md` is the system-level source of truth (modules + interfaces).
- `docs/PLAN.md` is the incremental implementation checklist (phases + dependency notes).
- Keep these consistent with any new modules, renamed modules, or interface changes.

