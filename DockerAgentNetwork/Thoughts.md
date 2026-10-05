# Raw thinking — Docker Multi-Model Agent Network on a Single VM

Chronological trail of the raw exploration. The distilled conclusion lives in `Summary.md`.

## The proposed architecture (as given)
A single Linux VM, two Tesla P40 GPUs (24 GB each), three vLLM containers:
- GPU 0: `vllm-router` (Qwen 2.5 3B, `--gpu-memory-utilization 0.20`, port 8000).
- GPU 0: `vllm-worker` (Llama 3.1 8B GPTQ, `--gpu-memory-utilization 0.65`, port 8001, `depends_on: vllm-router`).
- GPU 1: `vllm-manager` (Qwen 2.5 14B GPTQ, `--gpu-memory-utilization 0.90`, port 8002).
- LiteLLM gateway on `:4000` reads `litellm-config.yaml` and routes model names → internal ports.
- A Python orchestrator (CrewAI / AutoGen / LangGraph) calls the gateway.

Motivation: isolated Docker environments, zero external-network lag over an internal bridge, and per-GPU pinning to dodge the PCIe tensor-parallelism bottleneck.

## What held up
- Single VM + Docker Compose + internal bridge = right call for isolation + latency.
- Per-GPU pinning (`CUDA_VISIBLE_DEVICES` / `devices`) genuinely sidesteps the PCIe bottleneck when models aren't sharded across cards.
- The VRAM reservation quirk is real: vLLM reserves VRAM at process startup, doesn't reclaim it, so two processes on one physical GPU do collide.

## Where I pushed back
- **`depends_on` is only start-order, not load-readiness.** vLLM takes seconds–minutes to load weights. So the worker can begin loading while the router is still initializing — the memory race still exists, just delayed. Plus, two vLLM processes doing *concurrent compute* on one P40 compete for real VRAM bandwidth/memory → correctness/perf issues, not just OOM. The MDEV (emulated memory-device) layer mostly protects the *reservation*. The small 3B router mostly leaves GPU 0 to the worker, so this is a real reduction, not a full solution.
- **The memory math is loose.** The worker's `--max-model-len 32768` with `fp16` KV cache is the weak link. KV-cache VRAM is *additive* on top of the model weight and scales with context length. The "~6 GB model space" claim ignores KV-cache growth. Either lower the budget or trim the worker's context length.
- **`ipc: host` is a blunt instrument.** Sharing the host `/dev/shm` breaks container isolation and is a reliability/security liability. The proper fix is an anonymous volume at `/dev/shm` inside the container (or `--vllm-mounts`), giving PyTorch a dedicated shm volume without exposing host memory. Also more reproducible across hosts.

## The orchestration question
The proposed stack answers the DualPhaseCoding open question ("which orchestration library?") partially: **LiteLLM + LangGraph**. LiteLLM is framework-agnostic (CrewAI/AutoGen/LangGraph all just call `:4000/v1`), and LangGraph's state-machine nature matches the "closed per-slice loop" decision better than CrewAI/AutoGen's chatty loops. LiteLLM also serves the DualPhaseCoding efficiency constraint — the deterministic executor calls it *only on genuine failure*.

## What's next
- Draft a corrected `docker-compose.yaml` with the proper shm mount + tighter worker budgets + `service_healthy` waits.
- Validate the P40 VRAM math empirically (router 3B ≈ 3–4.8 GB, worker 8B GPTQ + fp16 KV cache at trimmed context).
- Consider whether readiness-based waits + a lower worker budget are worth the added compose complexity.
