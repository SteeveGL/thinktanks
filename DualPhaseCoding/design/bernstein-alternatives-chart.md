---
title: "Bernstein Alternatives — Multi-Agent Orchestration Landscape"
status: "draft"
date: "2026-10-03"
decision: "Comparative landscape of multi-agent orchestration tools, mapped against the thinktank's resolved decisions. No tool fully replicates Bernstein's combination."
---

# Bernstein Alternatives — Multi-Agent Orchestration Landscape

> Comparative landscape for the dual-phase multi-agent coding system. Design-level only:
> a research artifact, not an implementation. Findings verified via live web search against
> primary sources (GitHub repos, official docs). Star counts fluctuate — treat as approximate.

## Why this doc exists

The thinktank resolved two decisions that constrain the tool choice:
- **D1 — deterministic executor + LLM only on failure** (resolved), local Ollama models.
- **D2 — per-slice DAG topological order** (resolved).

This doc maps the multi-agent orchestration landscape against those decisions and against the
five dimensions that define Bernstein's profile. It answers: *which tools are real alternatives
to Bernstein, and where does each one fall on the dimensions that matter?*

## Column definitions

| Column | Definition |
|---|---|
| **Type** | Category of product: full orchestrator, framework (compose-able building blocks), single-agent assistant, worktree utility, or enterprise platform. |
| **Deterministic scheduler** | Whether the coordination layer (who runs, when, in what order, retry logic) is plain code with **no LLM in the loop** — or driven by an LLM reasoning about state. This is resolved D1. ✅ fully deterministic / ❌ LLM-driven / ~ partial. |
| **Git worktree isolation** | Whether each agent runs in its **own git worktree** (parallel-edit isolation, no file conflicts). Key differentiator vs. Docker. ✅ yes / ❌ no / ~ nascent. |
| **Docker / sandbox** | Whether the tool provides a **security sandbox** (network egress, destructive-command detection, filesystem allowlist) around the agent. This is the Docker-sandbox requirement. |
| **Local / Ollama** | Whether the tool supports **local models** (Ollama or OpenAI-compatible backends), not cloud-only. This is resolved D1. |
| **Audit lineage** | Whether the tool produces a **tamper-evident, verifiable record** of what happened (signed receipts, HMAC chains, replay journal). This is external audit logging (🟢 solid). |
| **VS Code** | Whether the tool ships as a **VS Code extension** vs. a CLI / TUI / browser dashboard. |
| **Best fit** | A one-line summary of what the tool is genuinely good at. |

The first five columns (**Deterministic scheduler**, **Git worktree isolation**, **Docker/sandbox**,
**Local/Ollama**, **Audit lineage**) are the five dimensions that define Bernstein's unique profile —
and also the five dimensions the thinktank has resolved decisions on. The last two (**VS Code**,
**Best fit**) are practical selection criteria.

## The chart

| Tool | Type | Deterministic scheduler | Git worktree isolation | Docker/sandbox | Local/Ollama | Audit lineage | VS Code | Best fit |
|---|---|---|---|---|---|---|---|---|
| **Bernstein** | Governance orchestrator | ✅ Yes (zero LLM) | ✅ Yes | ✅ Docker backend | ✅ Native | ✅ Signed lineage + HMAC | ❌ CLI/TUI | The full combo |
| **fractal** (plasma-ai) | Coding orchestrator | ~ Partial | ✅ Yes | ❌ | ✅ Via harness | ~ | ❌ | Closest git-worktree peer |
| **Microsoft Foundry / Agent Framework** | Enterprise platform | ✅ Yes | ❌ | ✅ Azure | ~ Via Azure | ✅ Governance | ✅ Ext | Enterprise governance |
| **Microsoft Agent Governance Toolkit** | Governance layer | ✅ Yes | ❌ | ✅ Sandbox | ~ | ✅ Policy/zero-trust | ~ | Closest governance-layer peer |
| **LangGraph** | Framework | ✅ Yes | ❌ | ❌ | ✅ Ollama | ~ LangSmith | ❌ | Stateful control flow |
| **Temporal** | Workflow engine | ✅ Yes | ❌ | ❌ | N/A | ✅ Web | ❌ | Durable long-running |
| **Prefect** | Data orchestration | ✅ Yes | ❌ | ❌ | N/A | ✅ | ❌ | Data/agent pipelines |
| **Dagster** | Data orchestration | ✅ Yes | ❌ | ❌ | N/A | ✅ Cloud | ❌ | Data-centric DAGs |
| **CrewAI** | Framework | ❌ LLM-driven | ❌ | ❌ | ✅ Ollama | ~ LangSmith | ❌ | Role-based crews |
| **AG2 / AutoGen** | Framework | ~ Partial | ❌ | ❌ | ✅ | ~ | ❌ | Conversation-based |
| **OpenAI Agents SDK** | Framework | ~ Partial | ❌ | ❌ | Via proxy | ~ | ❌ | Handoffs |
| **Google ADK + A2A** | Framework | ~ Partial | ❌ | ❌ | Via providers | ~ Vertex AI | ❌ | Google ecosystem |
| **Anthropic Agent SDK** | Framework | ❌ | ❌ | ❌ | Claude-only | ~ MCP | ❌ | Claude tool-use loops |
| **NVIDIA NeMo Agent Toolkit** | Framework | ~ Partial | ❌ | ❌ | ✅ Nemotron | ~ | ❌ | Production multi-agent |
| **OpenHands** | SWE platform | ~ Partial | ❌ | ✅ Docker | ✅ Ollama | ~ | ~ | Open-source Devin |
| **LlamaIndex** | Framework | ~ Partial | ❌ | ❌ | ✅ | ~ | ❌ | RAG + local agents |
| **LangChain** | Framework | ~ Partial | ❌ | ❌ | ✅ Ollama | ~ | ❌ | General LLM chaining |
| **MetaGPT** | Framework | ❌ LLM-driven | ❌ | ❌ | ✅ | ~ | ❌ | Software-company roleplay |
| **git-parsec** | Worktree utility | ❌ | ✅ Yes | ❌ | ~ | ❌ | ~ | Worktree lifecycle |
| **worktrunk** (max-sixty) | Worktree utility | ❌ | ✅ Yes | ❌ | ~ | ❌ | ~ | Parallel worktrees |
| **Orca** | Desktop control plane | ~ | ✅ Yes | ~ | ✅ | ~ | ~ | Parallel agents, worktrees |
| **Branchlet** | Worktree utility | ❌ | ✅ Yes | ❌ | ~ | ❌ | ~ | Simple worktree manager |
| **Amazon Q Developer / Bedrock AgentCore** | Enterprise platform | ❌ | ❌ | ✅ AWS | No (any model) | ~ CloudWatch | ✅ Ext | Managed AWS orchestration |
| **Cognition Devin** | Proprietary SaaS | ❌ | ❌ | ✅ | No | ~ | ❌ | The proprietary benchmark |
| **Continue.dev** | Single-agent IDE ext | ❌ | ❌ | ❌ | ✅ Ollama | ~ | ✅ | Local IDE assistant (acquired by Cursor) |
| **Aider** | Single-agent CLI | ❌ | ❌ | ❌ | ✅ | ~ | ❌ | Pair programmer, any LLM |
| **Codex CLI** | Single-agent CLI | ❌ | ❌ | ❌ | Via proxy | ~ | ❌ | OpenAI single-agent |
| **Gemini CLI** | Single-agent CLI | ❌ | ❌ | ❌ | Via proxy | ~ | ❌ | Google single-agent |
| **OpenCode** | Single-agent CLI | ❌ | ❌ | ❌ | ✅ 75+ providers | ~ | ❌ | Terminal agent, no code storage |

## Where Bernstein sits

Bernstein's **unique combination** — `git-worktree isolation` + `signed audit lineage` +
`per-node model routing` + `deterministic coordinator` + `Docker sandboxing` — is not fully
replicated by any single tool:

- **Closest governance-layer peers:** Microsoft Agent Governance Toolkit, Microsoft Foundry
  (deterministic + governance + sandbox).
- **Closest git-worktree peers:** `fractal` (Apache-2.0) — the one open tool doing per-node
  git-worktree orchestration.
- **Closest local-LLM + sandbox peer:** OpenHands (BSD, Ollama-capable, Docker sandbox).
- **Closest deterministic-scheduler peers:** LangGraph, Temporal, Prefect, Dagster.
- **Closest git-worktree isolation utilities:** git-parsec, worktrunk, Orca (all small / nascent).

**Bottom line:** Bernstein occupies a narrow, underserved slot at the intersection of
*governance + git-worktree isolation + local models*. The governance space is dominated by
Microsoft Foundry (enterprise / closed), the orchestration space by LangGraph / CrewAI / AG2
(frameworks, no isolation), and the local-coding space by OpenHands / Continue (no governance /
audit lineage).

## Notes on the thinktank's five dimensions

- **Deterministic scheduler** — Bernstein, LangGraph, Temporal, Prefect, Dagster, Microsoft Foundry
  all score ✅. CrewAI and AG2 are LLM-driven. This is resolved D1.
- **Git worktree isolation** — only Bernstein, fractal, git-parsec, worktrunk, Orca, Branchlet offer
  it. Most tools use Docker container isolation instead (OpenHands, Devin). This is a thin, nascent
  niche.
- **Docker / sandbox** — Bernstein (opt-in docker backend), OpenHands, Microsoft Foundry, AWS, Devin.
  Most frameworks have no sandbox. This is the thinktank's Docker-sandbox requirement.
- **Local / Ollama** — Bernstein, CrewAI, AG2, OpenHands, LlamaIndex, LangChain, Aider, OpenCode,
  Continue, fractal all support local models. Microsoft Foundry / AWS / Anthropic are cloud-first.
  This is resolved D1.
- **Audit lineage** — Bernstein is the only tool with signed lineage + HMAC audit chain + replay
  journal out of the box. Microsoft Foundry has governance/observability; the rest are weaker.

## Open questions / what's next

- **Fork-vs-adopt:** Bernstein already has most of what the thinktank needs (deterministic scheduler,
  Ollama adapter, cascade routing, tournament runs, docker sandbox, audit chain). The gaps are
  **(1)** the day→night handoff schema (your JSONL contract) and **(2)** a VS Code UI (Bernstein is a
  CLI/TUI). Both are smaller surfaces than forking the orchestrator.
- **Day→night handoff:** Bernstein does one run (decompose → execute → verify → merge), not the
  day (interactive) / night (headless) split specifically. Its durable work ledger (crash-safe,
  machine-portable resume) and detached run service (submit → disconnect → reattach) are the closest
  analog — your JSONL handoff would layer on top, or the durable ledger becomes the handoff bridge.
- **Dynamic re-planning:** Bernstein cannot adapt the plan mid-run (a known limitation, shared with
  the thinktank's resolved D2). Tournament runs (N parallel attempts) partially compensate.
- **Verify deeper:** `fractal`'s git-worktree isolation, OpenHands' Docker sandbox, and Microsoft
  Foundry's governance are the three closest peers — worth a deeper architecture look before deciding.
