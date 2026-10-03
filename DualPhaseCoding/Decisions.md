# Decisions — Dual-Phase Multi-Agent Coding System

*Notable forks, dropped ideas, and why. Future-you will thank present-you.*

## Decisions
- **Folder naming:** `DualPhaseCoding` — PascalCase, no timestamp prefix (date kept in `ReadMe.md` header per repo rules).
- **Handoff format:** lean toward **JSONL handoff ledger** as the universal bridge (rule #1, #2, #4). Framework-agnostic and crash-safe.
- **Slicing:** require **dependency graph + topological sort**; don't just list slices.
- **Phase 2:** **per-slice Docker isolation + rollback-on-regression** is the core robustness story.
- **Backend-first build strategy (chosen / heuristic).** The backend owns the data model / persistence / API contract and is *usually* the critical path, so the default build order is backend-first. This is a **priority signal, not a hard "must"**: dev order should never block a better design choice, so if another order makes more sense, it's fine.
- **Deterministic night loop (chosen).** A **closed per-slice loop** — write code → Docker-run tests → feed failure → retry with a hard cap → else mark `blocked` and refuse to advance — over an open, chatty multi-agent loop. This bounds LLM calls and guarantees a run never advances past a failed slice.
- **Framework moved away from chatty AG2 `GroupChat` (chosen).** Ollama is cheap-but-slow, so model calls must be minimized. We now target a **deterministic executor + an LLM at decision points** (e.g. via LiteLLM): deterministic steps (Docker, test runner) run without the model; the model is called only on genuine failure, capped by `retry_budget`. LangGraph remains a viable enterprise option (state-machine checkpointing).
- **Dependency DAG in YAML (chosen).** Nodes = slices, edges = data/dependency edges. YAML is preferred over JSON for human-editability during day-phase review, and over graph DSLs (Dot) because the DAG needs rich per-node metadata (weights, risk, phase) that a Dot node can't carry cleanly. A topological sort applies a **backend-first priority signal** as the default order (heuristic, not a design-lock — order can be overridden when a better choice exists).
- **Handoff schema as the day→night contract (chosen).** A per-slice record carrying a `test_contract` (the failing tests = the spec), `depends_on` (DAG input), `deterministic_execution_procedure`, `retry_budget`, `priority` (backend-first priority signal — a heuristic ordering hint, not a hard must), and a `blocked`-slice contract so morning review can unambiguously decide genuine-failure vs. test-gaming.

## Confidence labels
- 🟢 solid: Docker sandbox for headless runs; `max_round` token budgets; external audit logging.
- 🟡 untested: Per-slice rollback; embedding-based slice retrieval (RAG); JSONL ledger (proposed, not implemented).
- 🔴 wild: CRIU container checkpoint/restore; event-sourced plan store; parallel developers.

## Dropped / deferred
- **CRIU container snapshots** — deferred (host-dependent, heavy).
- **Event-sourced plan store** — deferred (over-engineered for MVP).
- **Message-bus decoupling** — deferred (infra overhead for a solo tool).

## Repo housekeeping
- **Removed the top-level `docs/` folder.** It was a generic docs folder, which **violates this repo's explicit rule** (`.copilot-instructions.md`) that each Thinktank session must live in its own PascalCase folder — NOT in a generic `docs/` folder. The session-specific `docs/` folder was the violation; removing it (rather than leaving an empty/generic folder) restores compliance.
- **The two design artifacts were relocated** into the existing session folder as `DualPhaseCoding\design/`:
  - `docs\dependency-dag.md` → `DualPhaseCoding\design\dependency-dag.md`
  - `docs\day-phase-handoff-schema.md` → `DualPhaseCoding\design\day-phase-handoff-schema.md`
  Nothing was discarded — the artifacts are preserved as real files, linked from `Thoughts.md` and distilled into `Summary.md` / `Decisions.md`.
- **Reused the existing `DualPhaseCoding/` folder rather than creating a new one** (e.g. `DualPhaseLocalLLMCoding`). The design artifacts are the *convergence* of the same 2026-10-03 session whose divergent brainstorm lives in `Thoughts.md` — same topic, same date. Creating a second folder for the same topic would violate the repo's isolation/no-duplication rule.
- **Residual question (flagged, not resolved):** was the top-level `docs/` folder intended for anything else in the future (e.g. repo-wide API/user docs)? It was empty at the time of removal, so nothing was lost — but if you want repo-level docs later, re-create a proper `docs/` folder then (kept separate from per-session PascalCase folders).

## What's next
1. Lock the handoff format: **YAML** per-slice schema is the chosen day→night contract (see `design/day-phase-handoff-schema.md`).
2. Draft a minimal Phase 2 loop (the closed per-slice loop is sketched; the deterministic procedure + `retry_budget` + `blocked` contract are the scaffold).
3. Implement the YAML dependency DAG + backend-first topological sort (backend-first as a default priority signal; see `design/dependency-dag.md`).
