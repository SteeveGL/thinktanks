# 2026-10-03 — Dual-Phase Multi-Agent Coding System

## Session overview

A brainstorm on a **dual-phase multi-agent coding system**:

- **Phase 1 (daytime, interactive):** a planning team — *Product Owner (human gate)* → *Product Manager* → *Lead Architect* — slices an open-ended requirement into atomic, logical "slices" and blocks execution until the user signs off on a markdown plan.
- **Phase 2 (overnight, headless):** a *Senior Developer* and *QA Tester* (inside a Docker sandbox) implement each slice, run tests, fix errors, and persist the workspace.
- **Framework selection:** AG2 (AutoGen fork) chosen for conversational planning + sandbox execution; LangGraph as the enterprise alternative (state-machine checkpointing).
- **Guardrails:** `max_round` token budgets, context compaction, strict Docker sandboxing, external audit logging.

This session is the **raw divergent brainstorm**. See [`Summary.md`](./Summary.md) for the distilled, always-up-to-date conclusion.

## Files

| File | Purpose |
| --- | --- |
| `ReadMe.md` | This file — session overview + links |
| `Summary.md` | Distilled summary (single source of truth) |
| `Thoughts.md` | Raw, organized brainstorm (this session's output) |
| `Decisions.md` | Notable forks, confidence labels, next steps |

## Related sessions

None yet. Past sessions are treated as inspiration, not ground truth.
