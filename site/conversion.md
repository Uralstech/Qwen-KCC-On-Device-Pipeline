# LiteRT and LiteRT-LM Conversion: Observations, Constraints, and Open Questions

## Overview

To assess on-device deployment feasibility, the fine-tuned Qwen2.5-1.5B farmer advisory model (with LoRA adapter merged) was converted from PyTorch to LiteRT format (via `.tflite` flatbuffer) using the AI Edge Torch toolchain, then packaged into LiteRT-LM for efficient text generation on edge runtimes. The conversion and packaging completed successfully, producing executable artifacts and confirming that a ~1.5B instruction-tuned model can be structurally lowered and run in an on-device-oriented runtime.

However, text generation from the LiteRT-LM artifact showed noticeably reduced coherence compared to PyTorch inference. The root cause remains unidentified, and no dedicated debugging or ablation study has been performed. The observations below are purely descriptive; any discussion of potential causes is explicitly speculative.

## PyTorch → LiteRT Conversion (.tflite)

Conversion from PyTorch to LiteRT-compatible `.tflite` format was executed using the AI Edge Torch converter and succeeded once system-level resource bottlenecks were resolved.

### Key Observations During Conversion

1. **High peak memory usage in the conversion pipeline**  
   Lowering attention operations and materializing the KV cache during MLIR graph construction resulted in substantial memory pressure.  
   - Conversions routinely failed on systems with 12–16 GB RAM.  
   - Availability of GPUs or high vCPU counts provided no meaningful relief once peak memory thresholds were hit.

2. **Strong sensitivity to maximum context length**  
   Configuring larger context windows (e.g., 32k tokens) caused exponential growth in intermediate graph sizes, triggering out-of-memory errors during graph construction.

3. **Limited impact from increased parallelism**  
   Adding more CPU cores did not appreciably reduce memory footprint or prevent allocation failures once the peak demand was exceeded.

### Resolution Path

No modifications were made to the model architecture or weights. Instead, the environment was scaled up:  
- Conversion was run on a high-memory CPU-only AWS instance (~128 GB RAM).  
- Maximum KV cache length was restricted to 4096 tokens, aligning with realistic on-device constraints.

Under these conditions, conversion completed in ~30–35 minutes, yielding a ~1.6 GB `.tflite` artifact. This confirms that PyTorch → LiteRT conversion remains viable for models of this scale when sufficient host memory is available during the lowering step.

## LiteRT (.tflite) → LiteRT-LM Packaging

The resulting `.tflite` model was then wrapped into LiteRT-LM format (`.litertlm`) to enable streamlined text generation using the LiteRT-LM pipeline library. Artifact creation and basic execution succeeded without errors.

### Inference Observations in LiteRT-LM

Qualitative differences emerged during text generation with LiteRT-LM compared to the same merged model in PyTorch:  
- Outputs occasionally showed repetition or short looping patterns.  
- Generated text sometimes appeared fragmented or less naturally conversational.  
- Certain responses resembled weakly conditioned, generic continuations rather than tightly grounded advisory replies.

These behaviors were specific to LiteRT-LM inference runs and were not investigated further through debugging, parameter tuning, or comparative ablation. No firm conclusions can be drawn about their origin at this stage.

## Possible Explanations (Speculative Only)

The following are plausible — but entirely unverified — hypotheses for the observed divergence:  
- Variations in decoding logic, default sampling parameters, or temperature/top-k handling between PyTorch and LiteRT-LM runtimes.  
- Differences in end-of-sequence / stop-token interpretation or prompt boundary processing.  
- Subtle impacts of the fixed (capped) KV-cache length on extended generation dynamics.  
- Mismatches in expected prompt formatting, metadata, or tokenization assumptions between training-time and LiteRT-LM runtime.  
- General maturity constraints in current on-device LLM pipeline runtimes when processing instruction-tuned models with domain-specific fine-tuning.

These are listed for transparency and completeness; none have been confirmed or ruled out.

## Implications

Successful structural conversion and execution do not guarantee behavioral equivalence across runtimes. While the AI Edge Torch → LiteRT → LiteRT-LM toolchain enables deployment of large models on edge hardware, qualitative generation characteristics can diverge in ways that are not yet fully understood or documented.

Importantly, these results do **not** point to a confirmed issue in:  
- the underlying model or fine-tuning,  
- the LoRA merging step, or  
- the conversion process itself.  

They instead surface open questions about inference fidelity in emerging on-device generative AI runtimes — questions that warrant further characterization as the tooling matures.

## Summary

- PyTorch → LiteRT (.tflite) conversion of a ~1.5B instruction-tuned model is feasible on high-memory CPU hosts.  
- Memory availability during graph lowering is the primary bottleneck; context length must be conservatively capped.  
- LiteRT-LM packaging yields runnable artifacts but reveals reduced coherence and increased repetition/fragmentation relative to PyTorch inference.  
- The cause of this behavioral difference is currently unknown; no targeted investigation has been conducted.  
- Several speculative explanations exist, but none are validated.

This account deliberately separates observed facts from unconfirmed hypotheses, providing a clear, reproducible record of current capabilities and limitations in deploying fine-tuned LLMs via LiteRT and LiteRT-LM.