# Summary — Dual-Phase Multi-Agent Coding System

*Single source of truth. Keep this up to date as ideas converge.*

## What we're designing
A dual-phase multi-agent coding system:
- **Phase 1 (daytime, interactive):** Product Owner (human gate) → Product Manager → Lead Architect slice a requirement into atomic, testable slices and require human sign-off before execution.
- **Phase 2 (overnight, headless):** Senior Developer + QA Tester (Docker sandbox) implement slices, run tests, fix errors, persist workspace.
- **Framework:** **moved away** from a chatty AG2 `GroupChat` toward a **deterministic executor + an LLM at decision points** (e.g. via LiteLLM). Rationale: Ollama is cheap-but-slow, so model calls must be minimized — prefer a small, deterministic, low-call-count pipeline over chatty multi-agent loops. LangGraph remains an enterprise option for state-machine checkpointing.

## Converged recommendations (top 6)
1. **Handoff:** Use a **JSONL handoff ledger** (append-only, crash-safe, framework-agnostic) as the day→night bridge, rather than in-memory state or heavy container snapshots.
2. **Backend-first critical path (default heuristic):** the backend owns the data model / persistence / API contract, so the default build order is backend-first. This is a priority signal, not a hard must — dev order should never block a better design choice.
3. **Deterministic night loop:** A **closed per-slice loop** — write code → Docker-run tests → feed failure → retry with a hard cap → else mark `blocked` and refuse to advance. No open-ended conversational loops.
4. **Dependency DAG (YAML):** Nodes = slices, edges = data/dependency edges. YAML chosen for human-editability + rich per-node metadata (weights, risk, phase). A topological sort **enforces** backend-first ordering.
5. **Day-phase handoff schema:** The day→night contract — a per-slice record carrying a `test_contract` (the failing tests = the spec), `depends_on` (DAG input), `deterministic_execution_procedure`, `retry_budget`, `priority` (backend-first), and a `blocked`-slice contract for morning review.
6. **Local-LLM constraints:** Ollama is cheap-but-slow → minimize model calls; deterministic steps (Docker, test runner) run without the model; the model is called only on genuine failure, capped by `retry_budget`.

## Related tools / prior art
- ⚠️ **Evaluated — vscode-copilot-orchestrator** (https://github.com/JeromySt/vscode-copilot-orchestrator): a VS Code extension that runs **multiple Copilot agents in parallel**, each in its own git worktree, over a DAG with an 8-phase pipeline (merge → prechecks → AI work → commit → postchecks → merge → cleanup), auto-heal (4 retries with fresh agents + failure context), snapshot validation, and pause/resume.
  > *Resume:* **Not adopted — evaluated as prior art.** It proves the parallel-DAG + auto-heal + snapshot-validation + pause/resume patterns are real and battle-tested, and several map onto resolved decisions: Kahn-level parallelism (Phase-2), `retry_budget`/rollback-on-regression (auto-heal), and day→night handoff (pause/resume). It **IS compatible with D1** — Ollama drives Copilot CLI with headless mode (`ollama launch copilot --model <x> --yes -- -p "..."`), so the extension runs on local models, not cloud-only. Caveats: worktree isolation gives parallel-*edit* isolation but **none** of the Docker sandbox properties (network egress, destructive-command detection, capability separation); and it runs *many* agents in parallel, which sits at the opposite pole from the deterministic-executor core. Worth **adapting** (auto-heal, snapshot validation, pause/resume), not adopting wholesale. Final call: user's to decide.
- ⚠️ **Evaluated — swarm-style multi-agent tools** (`zippytechnologiesllc.autoclaw`, `lukapetrovic12.swarm`, `simoncoombes.swarm-engine`, `alex-chernysh.bernstein`, `expxagents.expxagents`): parallel/fan-out orchestration, multi-agent squads. **Disqualified** as alternatives for this thinktank: none support **local Ollama** (Copilot Orchestrator is the only one confirmed Ollama-compatible, via Copilot CLI). AutoClaw's GitHub report returns 404; `swarm-engine` and `bernstein` appear unmaintained; `expxagents` has no Copilot Chat / no local LLM. The upstream `sipyourdrink-ltd/bernstein` repo is flagged as a lead to explore next.

## Open questions / what's next
- 🟢 solid: Docker sandbox for headless runs; `max_round` token budgets; external audit logging.
- 🟡 untested: Per-slice rollback; embedding-based slice retrieval (RAG); JSONL ledger (proposed, not implemented).
- 🔴 wild: CRIU container checkpoint/restore; event-sourced plan store; parallel developers.

## Open questions / what's next
- ✅ **RESOLVED — framework:** the chatty AG2 `GroupChat` → deterministic-executor + LLM-at-decision-points (e.g. LiteLLM) pattern is now chosen; LangGraph remains the enterprise checkpointing option.
  > *Resume:* deterministic executor for cheap-but-slow Ollama; model calls only on genuine failure, capped by `retry_budget`. Decision stands — the red-team pass reframed the cost question (latency/quality per call vs. call count) but did not overturn it.
- ✅ **RESOLVED — Phase 2 execution:** slices run **per-slice in DAG topological order**, with Kahn levels indicating which slices are parallelizable — not a single open loop. Backend-first is a **priority signal, not a hard "must"**: dev order should never block a better design choice, so if another order makes more sense, it's fine.
  > *Resume:* per-slice closed loop in topological order; ordering is heuristic, not a design-lock.
- Lock the handoff format: **YAML** per-slice schema is the chosen day→night contract (see `design/day-phase-handoff-schema.md`).
- Draft a minimal Phase 2 loop (the closed per-slice loop is sketched; the deterministic procedure + `retry_budget` + `blocked` contract are the scaffold).
