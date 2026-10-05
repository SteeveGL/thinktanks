# 2026-10-04 — Docker Multi-Model Agent Network on a Single VM

## Session overview

A review of a proposed **single-VM, Docker Compose agent network** for a local LLM stack:

- **Hardware:** a single Linux VM with **two Tesla P40 GPUs** (24 GB each).
- **Topology:** three vLLM inference containers pinned to GPUs — `vllm-router` (Qwen 2.5 3B) + `vllm-worker` (Llama 3.1 8B GPTQ) share **GPU 0**, `vllm-manager` (Qwen 2.5 14B GPTQ) is pinned to **GPU 1**.
- **Gateway:** a **LiteLLM** OpenAI-compatible proxy (`:4000`) routes traffic to the three internal vLLM ports; a Python orchestrator (CrewAI / AutoGen / LangGraph) sits on top.
- **Motivation:** isolate environments, zero external-network latency over an internal bridge, and pin models to physical GPUs to avoid PCIe tensor-parallelism lag.
- **Key technical claim:** `--gpu-memory-utilization` budgets + `depends_on` sequential startup mitigate a vLLM VRAM reservation quirk when two containers share one physical GPU.

This session is the **inference-infrastructure layer** for the DualPhaseCoding system — it answers the open question *"which agent orchestration library are you writing your application code in?"* by proposing **LiteLLM + LangGraph**. It also stress-tests the VRAM-mitigation and isolation claims.

See [`Summary.md`](./Summary.md) for the distilled, always-up-to-date conclusion.

## Files

| File | Purpose |
| --- | --- |
| `ReadMe.md` | This file — session overview + links |
| `Summary.md` | Distilled summary (single source of truth) |
| `Thoughts.md` | Raw, chronological notes (this session's output) |
| `Decisions.md` | Notable forks, confidence labels, next steps |

## Related sessions

- [`DualPhaseCoding`](../DualPhaseCoding/) — the parent design this infrastructure supports; this session resolves its "orchestration library" open question.
