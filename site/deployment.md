# Deployment and Inference Evaluation

This section documents the practical deployment of the fine-tuned Qwen-2.5-1.5B LoRA-merged model as a LiteRT-LM artifact, including infrastructure used, inference setup on consumer hardware, measured performance, and key constraints observed in real-world edge execution.

## Deployment Infrastructure

### Conversion Environment (PyTorch → .tflite)

Conversion was executed using Google AI Edge Torch on a high-memory CPU-only AWS EC2 instance:

- **Instance type**: r6i.4xlarge
- **Compute**: 16 vCPUs
- **Memory**: 128 GB RAM
- **GPU**: None required
- **OS**: Ubuntu 24.04 LTS
- **Static KV-cache length**: 4096 tokens

Conversion completed in ~30–35 minutes with peak memory usage of ~30–35 GB, producing a ~1.6 GB `.tflite` artifact. No GPU resources were needed; the process is memory-bound due to graph lowering and KV-cache materialization. The final cost of conversion, including ~10 minutes of environment setup, was ~0.60 USD, as per the instance's ~1.00 USD per hour pricing.

### LiteRT-LM Packaging

Packaging of the `.tflite` model into `.litertlm` format (for LiteRT-LM inference) was performed on a lighter instance:

- **Instance type**: r6i.2xlarge
- **Compute**: 8 vCPUs
- **Memory**: 64 GB RAM
- **OS**: Ubuntu 24.04 LTS

This step was lightweight, completing in minutes with minimal resource demands. The LiteRT-LM builder handled metadata alignment for Qwen tokenizer conventions.
Including environment setup (~10 minutes), the final cost of LiteRT-LM packaging was ~0.075 USD, as per the instance's ~0.50 USD per hour pricing.

## Inference Platform and Performance

Inference evaluation was conducted locally on consumer-grade hardware to reflect realistic offline/on-device use (e.g., farmer-facing mobile tools or extension devices in low-connectivity areas).

- **Device**: Mac mini (Apple M4)
- **CPU**: 10-core
- **GPU**: 10-core (utilized via LiteRT-LM GPU backend)
- **Neural Engine**: 16-core (not leveraged by current LiteRT-LM as of Dec 2025)
- **Unified memory**: 16 GB
- **OS**: macOS 26.1 "Tahoe"

**Runtime configuration**:
- Backend: LiteRT-LM (with GPU acceleration on Apple Silicon)
- Context length: 4096 tokens (static KV cache)
- Precision: Quantized graph as produced by AI Edge Torch
- Decoding: Conservative parameters aligned with Qwen conventions

**Measured performance** (typical farmer advisory queries, 50–150 token responses):
- Time-to-first-token (TTFT): < 1 second
- Token generation: Incremental decoding, stable throughput
- End-to-end response time: ~2.5–4 seconds

These latencies support interactive, offline advisory scenarios without requiring constant connectivity.

## Summary of Deployment Findings

- The fine-tuned Qwen-2.5-1.5B model is deployable as a LiteRT-LM artifact using public AI Edge and LiteRT-LM tooling.
- Conversion requires only high-RAM CPU instances (no GPUs needed), enabling cost-effective cloud workflows.
- On modern consumer hardware (e.g., Apple M4), interactive latencies (~2.5–4 s end-to-end) are achieved, suitable for offline agricultural advisory use.
- Generation characteristics remain runtime-dependent, with LiteRT-LM outputs showing reduced coherence relative to PyTorch — highlighting the importance of runtime-specific validation.

All deployment scripts, configurations, and artifacts are included in the repository for full reproducibility.