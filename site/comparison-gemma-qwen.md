# Comparison of Gemma-3n E2B and Qwen-2.5-1.5B for On-Device Farmer Advisory

Building on the prior Gemma-3n E2B fine-tuning effort for on-device farmer advisory, we evaluated Qwen-2.5-1.5B-Instruct as an alternative under real-world edge deployment constraints. Both models adapted successfully to the Kisan Call Centre (KCC) dataset via supervised fine-tuning with LoRA, exhibiting comparable domain alignment in short pilot runs. However, differences in long-context stability, response style, and — most critically — toolchain compatibility for LiteRT / LiteRT-LM conversion drove the shift to Qwen.

## Model Size and Architectural Notes

Qwen-2.5-1.5B has a lower nominal parameter count than Gemma-3n E2B. While Gemma-3n E2B leverages a MatFormer architecture with ~6B raw parameters but an effective footprint comparable to traditional 2B models on edge hardware (via selective activation and offloading), Qwen remains a standard dense 1.5B design. Gemma-3n offers stronger native multilingual coverage out-of-the-box. In practice, both models converged similarly on the KCC task during limited-step fine-tuning, indicating that raw capacity was not the primary differentiator here.

## Training Behavior and Response Style

Both models internalized KCC-style advisory content effectively under the constrained training regime (short step-limited runs). Qualitative differences emerged in output style:

- Gemma-3n E2B generated more naturalistic, conversational responses, often paraphrasing advice and structuring replies in a dialog-friendly way.
- Qwen-2.5-1.5B tended to mirror the dataset's terse, list-oriented structure more literally, producing responses with closer one-to-one fidelity to training examples.

This stylistic divergence reflects mild over-alignment to the KCC expert tone rather than a deep semantic gap. Importantly, both models produced occasional incorrect or incomplete agronomic recommendations — a direct consequence of the minimal training steps and lack of full convergence rather than an inherent quality difference between architectures.

## Long-Context Stability

A clear distinction appeared in extended-context behavior, which is critical for maintaining multi-turn advisory conversations on-device without server fallback:

- Gemma-3n E2B exhibited higher rates of repetition, degeneration, and instability as prompt lengths increased beyond typical single-query advisory interactions.
- Qwen-2.5-1.5B maintained more consistent coherence under longer contexts, even while retaining its characteristically concise style.

This robustness makes Qwen more suitable for edge scenarios where conversation state is preserved locally.

## Conversion and Deployment Feasibility

Toolchain maturity proved the decisive factor — not raw model quality or fine-tuning outcomes.

- **Gemma-3n E2B** faced insurmountable barriers:
  - Public AI Edge Torch and LiteRT-LM tooling could not reliably convert the fine-tuned (LoRA-merged) checkpoint.
  - Attempts via ONNX / Optimum paths failed to load and export the adapted model for LiteRT-compatible inference.
  - This created a hard deployment dead-end beyond PyTorch/Colab environments.

- **Qwen-2.5-1.5B** succeeded end-to-end:
  - Fully supported by Google AI Edge Torch for PyTorch → `.tflite` conversion.
  - Packaged cleanly into LiteRT-LM format for on-device text generation.
  - Enabled practical experimentation with bounded KV-cache lengths and edge runtime constraints.

These differences were external to the models themselves but determined real deployability on target hardware.

## Summary of Trade-offs

Gemma-3n E2B offers advantages in multilingual breadth and more natural conversational flow post-fine-tuning. However, its long-context instability and — crucially — blocked conversion path to LiteRT-LM limited its viability for fully on-device systems. Qwen-2.5-1.5B, while stylistically more rigid and marginally smaller, delivered superior deployment reliability, conversion success, and long-context stability — making it the pragmatic choice for edge-first farmer advisory at the current stage of tooling maturity.

This shift is systems-driven: prioritizing verifiable deployability and runtime robustness over marginal gains in dialogue naturalness. Future work with improved Gemma-3n conversion support or larger training budgets could revisit this trade-off.

*(Note: Screenshots and artifacts from the original Gemma-3n E2B pilot — e.g., inference examples [Fig. 5], loss curves [Fig. 6], and LoRA adapters — are retained in the prior repository for reference and reproducibility: <https://doi.org/10.22541/au.176659877.79810429/v1>)*