# Decisions — Dual-Phase Multi-Agent Coding System

*Notable forks, dropped ideas, and why. Future-you will thank present-you.*

## Decisions
- **Folder naming:** `DualPhaseCoding` — PascalCase, no timestamp prefix (date kept in `ReadMe.md` header per repo rules).
- **Handoff format:** lean toward **JSONL handoff ledger** as the universal bridge (rule #1, #2, #4). Framework-agnostic and crash-safe.
- **Slicing:** require **dependency graph + topological sort**; don't just list slices.
- **Phase 2:** **per-slice Docker isolation + rollback-on-regression** is the core robustness story.

## Confidence labels
- 🟢 solid: Docker sandbox for headless runs; `max_round` token budgets; external audit logging.
- 🟡 untested: Per-slice rollback; embedding-based slice retrieval (RAG); JSONL ledger (proposed, not implemented).
- 🔴 wild: CRIU container checkpoint/restore; event-sourced plan store; parallel developers.

## Dropped / deferred
- **CRIU container snapshots** — deferred (host-dependent, heavy).
- **Event-sourced plan store** — deferred (over-engineered for MVP).
- **Message-bus decoupling** — deferred (infra overhead for a solo tool).

## What's next
1. Pick ONE framework concretely (AG2 vs LangGraph) and lock the handoff format.
2. Decide sequential vs parallel Phase 2 execution.
3. Draft a minimal Phase 2 loop: QA writes failing test → Dev implements → QA verifies → atomic commit.
