# Decisions — Docker Multi-Model Agent Network on a Single VM

Notable forks, dropped ideas, and confidence labels. Past sessions are inspiration, not ground truth.

## Kept the architecture (🟢 solid)
Single VM + Docker Compose + internal bridge + per-GPU pinning. Right topology for isolation, zero external lag, and avoiding PCIe tensor-parallelism lag.

## VRAM mitigation is partial, not complete (🟡 untested)
`depends_on` sequential startup reduces but does not eliminate the shared-GPU VRAM race.
- `depends_on` guarantees *container start order*, not *model-load readiness*.
- Concurrent compute on one P40 competes for real VRAM bandwidth/memory → correctness/perf issues, not just OOM.
- The small 3B router mostly leaves GPU 0 to the 8B worker; the mitigation is a real reduction.
- **Proposed:** add a `healthcheck` + `depends_on: condition: service_healthy` so the worker waits for the router to actually finish loading, and treat the worker budget as a hard ceiling validated with `nvidia-smi` under load.

## Memory math needs empirical validation (🔴 wild)
The stated ~3 GB (router) / ~6 GB (worker) splits ignore KV-cache growth. The worker's `--max-model-len 32768` in fp16 is the weak link.
- **Proposed:** lower the worker budget or trim its context length; measure real footprint before declaring production.

## `ipc: host` is a liability (🟡 untested)
Sharing the host `/dev/shm` breaks isolation and is unreliable across hosts.
- **Proposed:** replace with an anonymous volume mounted at `/dev/shm` per container (or `--vllm-mounts`), giving PyTorch a dedicated shm volume without exposing host memory.

## Orchestrator: LiteLLM + LangGraph (🟡 untested)
- LiteLLM: framework-agnostic gateway; CrewAI/AutoGen/LangGraph all call `:4000/v1`.
- LangGraph: state-machine, matches the "closed per-slice loop" decision; calls the gateway only on genuine failure (DualPhaseCoding efficiency constraint).
- CrewAI/AutoGen: chatty group-chat loops — a poorer fit for a deterministic executor.

## What's next
- Draft a corrected `docker-compose.yaml` (shm volume, `service_healthy` waits, tighter worker budget).
- Validate P40 VRAM math empirically.
- Decide: build the orchestrator against LiteLLM, or keep the gateway idea and pick the framework later.
