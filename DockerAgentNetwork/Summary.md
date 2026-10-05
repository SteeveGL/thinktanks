# Summary — Docker Multi-Model Agent Network on a Single VM

*Single source of truth. Keep this up to date as ideas converge.*

## What's being designed
A single Linux VM with two Tesla P40 GPUs (24 GB each) running a three-role agent network:
- **Router** — Qwen 2.5 3B on `vllm-router`, pinned to GPU 0.
- **Worker** — Llama 3.1 8B GPTQ on `vllm-worker`, pinned to GPU 0 (shares the card with the router).
- **Manager/Critic** — Qwen 2.5 14B GPTQ on `vllm-manager`, pinned to GPU 1.
- **Gateway** — LiteLLM OpenAI-compatible proxy on `:4000` routes to the three internal vLLM ports.
- **Orchestrator** — a Python agent framework (CrewAI / AutoGen / LangGraph) calls the gateway.

The stated problem: two vLLM containers on the *same physical GPU* both assume 100% of the P40 is free at startup, so the second crashes with an OOM error. The proposed mitigation is explicit `--gpu-memory-utilization` budgets + `depends_on` sequential startup.

## What we concluded (this session)
The overall architecture is sound; the VRAM-mitigation is only *partial*; `ipc: host` is a liability; and the LiteLLM + LangGraph stack is the coherent choice for the orchestrator. Key points:

- **Whole idea:** single VM + Docker Compose + internal bridge is the right call for isolation and zero external-network latency. Per-GPU pinning (`CUDA_VISIBLE_DEVICES` / `devices`) sidesteps the PCIe tensor-parallelism bottleneck.
- **The VRAM quirk is real and the mitigation is real too, but partial.** `depends_on` guarantees *container start order*, not *model-load readiness* — vLLM takes seconds to a minute to load weights, so the worker can begin loading while the router is still initializing, and the startup race you're avoiding still exists, just delayed. Two vLLM processes doing *concurrent compute* on one P40 also compete for real VRAM bandwidth and memory, which surfaces as correctness/perf issues, not just OOM. The router being small (3B, 20% budget ≈ 4.8 GB) mostly leaves GPU 0 to the worker.
- **The memory math is loose.** The worker's `--max-model-len 32768` with `fp16` KV cache is the weak link: KV-cache VRAM is *additive* on top of the model weight and scales with context length. Either the budget must be lower or the worker's context length trimmed. The stated "~6 GB model space" ignores KV-cache growth.
- **`ipc: host` is a blunt instrument.** Sharing the host `/dev/shm` breaks container isolation and is a reliability/security liability. The proper fix is an **anonymous volume mounted at `/dev/shm`** inside the container (or `--vllm-mounts`), which gives PyTorch a dedicated shm volume without exposing host memory.
- **LiteLLM + LangGraph** is the coherent orchestrator choice: LiteLLM is framework-agnostic (CrewAI/AutoGen/LangGraph all just call `:4000/v1`), and LangGraph's state-machine nature matches the DualPhaseCoding "closed per-slice loop" decision better than CrewAI/AutoGen's chatty group-chat loops.

## Converged recommendations (top 6)
1. **Keep the architecture.** Single VM + Docker Compose + internal bridge + per-GPU pinning. This is the right topology.
2. **Fix the shm mount.** Replace `ipc: host` with an anonymous volume at `/dev/shm` per inference container — proper isolation, reproducible across hosts.
3. **Tighten GPU 0 budgets.** The router (3B, 20%) must finish loading before the worker (8B) meaningfully uses the card; use readiness-based waits (`healthcheck` + `depends_on: condition: service_healthy`) instead of plain `depends_on`, and cap the worker's `--max-model-len` so its fp16 KV cache doesn't blow past the 65% budget.
4. **Budgets are per-container, not a shared guarantee.** Two vLLM processes on GPU 0 still compete for real VRAM during concurrent compute; treat the 65% worker budget as a hard ceiling and validate actual usage with `nvidia-smi` under load.
5. **Orchestrate LiteLLM + LangGraph.** LiteLLM gateway routes model→port; LangGraph executes the deterministic per-slice loop and calls the gateway only on genuine failure (matching the DualPhaseCoding efficiency constraint).
6. **Validate memory empirically before declaring production.** Measure each service's real VRAM footprint on the P40s; the stated ~3/6 GB splits are rough estimates that must be confirmed before trusting the budgets.

## Related tools / prior art
- **vLLM VRAM reservation quirk** — the source claims (discuss.vllm.ai threads [#1620](https://discuss.vllm.ai/t/2-vllm-docker-on-same-host/1620) and [#2405](https://discuss.vllm.ai/t/how-to-serve-two-vllm-instance-using-docker/2405)) describe the startup-assumes-empty-card problem and the `--gpu-memory-utilization` / `--kv-cache-dtype fp16` mitigations. Confirmed plausible; the mitigation is partial, not a full fix.
- **LiteLLM** — OpenAI-compatible proxy; config-driven `model_list` maps model names to internal `api_base` endpoints. Framework-agnostic inference gateway.
- **LangGraph** — state-machine orchestration; best fit for a deterministic executor over a DAG, vs. CrewAI/AutoGen's conversational loops.
- **DualPhaseCoding** — the parent thinktank; this session resolves its "orchestration library" open question (proposes LangGraph via LiteLLM).
