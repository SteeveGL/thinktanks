# Summary — Dual-Phase Multi-Agent Coding System

*Single source of truth. Keep this up to date as ideas converge.*

## What we're designing
A dual-phase multi-agent coding system:
- **Phase 1 (daytime, interactive):** Product Owner (human gate) → Product Manager → Lead Architect slice a requirement into atomic, testable slices and require human sign-off before execution.
- **Phase 2 (overnight, headless):** Senior Developer + QA Tester (Docker sandbox) implement slices, run tests, fix errors, persist workspace.
- **Framework:** AG2 for planning + sandbox execution; LangGraph as the enterprise checkpointing alternative.

## Converged recommendations (top 3)
1. **Handoff:** Use a **JSONL handoff ledger** (append-only, crash-safe, framework-agnostic) as the day→night bridge, rather than in-memory state or heavy container snapshots.
2. **Slicing:** Build a **dependency graph + topological sort + parallelizable-slice detection**; require each slice to declare inputs, outputs, and a testable contract.
3. **Phase 2 robustness:** **Per-slice Docker isolation + rollback-on-regression + external structured audit logging**.

## Confidence labels
- 🟢 solid: Docker sandbox for headless runs; `max_round` token budgets; external audit logging.
- 🟡 untested: Per-slice rollback; embedding-based slice retrieval (RAG); JSONL ledger (proposed, not implemented).
- 🔴 wild: CRIU container checkpoint/restore; event-sourced plan store; parallel developers.

## Open questions / what's next
- Pick ONE framework concretely (AG2 vs LangGraph) and lock the handoff format.
- Decide whether Phase 2 runs slices sequentially or in parallel.
- Draft a minimal Phase 2 loop (TDD per slice: QA writes failing test → Dev implements → QA verifies → commit).
