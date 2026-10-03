# Thoughts — Raw brainstorm

*Raw, organized divergent thinking. This is the trail, not the destination.*

## 1. Day/Night Handoff Architecture
- **LangGraph persistent checkpointing** — serialize whole graph state (history, tool outputs, slice cursor) to Postgres/SQLite with `checkpointer`. Night resumes by loading checkpoint. True resumability; larger serialization surface.
- **JSONL "handoff ledger" + append-only log** — day writes `handoff.jsonl` with signed-off plan, per-slice status, and cursor (next slice index + partial-work marker). Crash-safe, framework-agnostic, human-readable.
- **Docker checkpoint/restore (CRIU)** — freeze sandbox between phases. Preserves exact in-memory state; host-dependent and heavy.
- **Message-bus / task-queue decoupling** — day enforces slices to RabbitMQ/NATS/Redis; night is a long-running consumer. Infinite resilience; infra overhead for a solo tool.
- **Event-sourced plan store** — markdown plan as derived output from an event log. Full audit + re-derivation; over-engineered for MVP.

## 2. Smarter Phase 1 Slicing
- **Dependency graph + topological sort** for a valid execution order.
- **Parallelizable-slice detection** via connected components / critical-path analysis.
- **Risk scoring** (size, unknowns, blast-radius) to queue risky slices for human review first.
- **Slice validation gates** — each slice must declare inputs, outputs, and a testable contract.
- **Incremental-buildability invariant** — every prefix of the slice order must be compilable.
- **Requirement ambiguity flagging** before slicing; **slice size bounds** so no slice exceeds one overnight run.

## 3. Phase 2 Robustness
- **Per-slice Docker isolation** — each slice in its own clean container at its parent commit.
- **TDD ordering** — QA writes the failing test before implementation.
- **Rollback on regression** — run full prior suite after each slice; auto-rollback to last green commit.
- **Atomic slice commits** (one git commit per slice, bisectable).
- **Slice-level wall-clock timeouts**; overflow spawns a "decompose" slice back to Phase 1.
- **Deterministic seeding** (pin RNG + tool versions) for reproducible resumes.

## 4. Token-Cost & Context Management
- **Context compaction** into a compressed "state digest."
- **Per-slice context window** — night loads only the approved plan + current slice.
- **Embedding-based slice retrieval (RAG)** instead of dumping all context.
- **Plan summarization** as the night phase's system-prompt seed.
- **Tiered memory** — short-term (current slice) vs long-term (vector store); only short-term loaded at runtime.
- **Deduplication** via file hashes so agents skip stale files.

## 5. Security & Safety Beyond Docker
- **Filesystem allowlist** (project root + temp only; block `../`, `/etc`).
- **Network egress control** — Docker network none or PyPI-mirror-only.
- **Destructive-command detection** (`rm -rf`, `mkfs`, `DROP TABLE`) abort + alert before execution.
- **Resource limits** — cgroup CPU/mem/pids, drop all Linux caps.
- **Secret scanning** pre-commit + executor hook.
- **Capability separation** — Dev gets write; QA/Reviewer get read-only exec.

## 6. Failure Modes & Infinite Loops
- **Detection signals:** repeated identical tool outputs, no net progress over N turns, budget exhaustion, state oscillation.
- **Loop breaker:** hash recent state; force a divergent action or abort on repeat.
- **Hallucinated file/exec guard:** validate referenced files exist before running.
- **Atomic writes** (temp + rename) to avoid mid-write corruption.
- **Model API failure:** retry with backoff + fallback model.

## 7. Observability & Audit Trails
- **Structured JSONL logs** shipped to an external sink (every LLM call, tool call, token count, cost).
- **Per-call cost metering** and a run-cost summary.
- **Diff-based audit:** consolidated git diff of the whole night.
- **Post-run review UX:** per-slice status, tests, diff, cost, escalations with one-click approve/rollback.
- **Reproducibility manifest:** pin model, versions, seeds, slice order.

## 8. Creative "What-If" Extensions
- **Parallel developers** owning disjoint slice sets (enabled by parallelizable detection).
- **Specialist agents:** DB engineer, security reviewer (SAST), docs writer, refactoring/linter, profiler.
- **Automated PR generation** — one aggregated PR at dawn with a changelog.
- **Self-evaluation loop** — a "Critic" scores outputs against the contract.
- **Cross-project memory** via embeddings from past projects.
- **Multi-model routing** — cheap model for slicing/tests, expensive for hard implementation.
- **Feature-flagged incremental delivery** so partial functionality is usable immediately.

## 9. Design Artifacts

The divergent brainstorm above has converged. Two **design-level** artifacts
(schema + illustrative snippets, no full implementation) capture the converged
decisions. They live in [`design/`](./design) and are linked from here — their
content is NOT duplicated inline.

- [`design/dependency-dag.md`](./design/dependency-dag.md) — **Dependency DAG.**
  Nodes = slices, edges = data/dependency edges, YAML-based, with a topological
  sort that *enforces* backend-first. Includes Kahn-levels (parallelism), critical
  path, risk/size scoring, and integration with the closed per-slice night loop.
- [`design/day-phase-handoff-schema.md`](./design/day-phase-handoff-schema.md) —
  **Day-Phase Handoff Schema.** The day→night contract: a per-slice record with a
  `test_contract` (the spec), `depends_on` (DAG input), `deterministic_execution_procedure`,
  `retry_budget`, and a `blocked`-slice contract for morning review.
