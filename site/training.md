# Training Process, Challenges, and Limitations

## Setup

The model was fine-tuned using unsloth/Qwen2.5-1.5B-Instruct with parameter-efficient adaptation via LoRA, leveraging the Unsloth framework and TRL’s `SFTTrainer`. The training corpus consisted of cleaned Kisan Call Centre (KCC) question–answer pairs from the public dataset, with only null and empty entries removed to preserve original fidelity. No synthetic data, paraphrasing, or external supervision was added.

Given Colab resource constraints (Tesla T4 GPUs with limited session duration and VRAM headroom), training relied on short, step-limited exploratory runs rather than full multi-epoch passes. This enabled quick validation of prompt formatting, loss behavior, qualitative inference style, and downstream conversion/deployability before investing in extended training.

## Engineering Challenges Encountered

Several practical engineering hurdles arose during setup and training:

1. **Trainer instability from environment-specific issues**  
   Early runs failed due to a runtime error tied to an unresolved `psutil` dependency in Unsloth’s compiled trainer cache, triggered when dataset parallelism was active. The issue was resolved by adopting TRL’s updated `SFTConfig` and clearing stale compilation artifacts, which restored stable, reproducible training.

2. **Hardware and session constraints**  
   Colab’s free-tier Tesla T4 GPUs impose tight limits on continuous runtime (~12 hours max per session) and available VRAM. While Unsloth’s optimizations (4-bit quantization support, gradient checkpointing, and efficient kernels) make LoRA fine-tuning of a 1.5B model feasible on T4 hardware, full multi-epoch training risked session termination and required frequent checkpointing. To prioritize rapid iteration and pipeline validation, we adopted short step-limited runs with the following key settings:

   - `warmup_steps = 5`
   - `max_steps = 60` (pilot sanity check)

   This approach confirmed training stability and basic domain adaptation while staying within practical hardware/session bounds.

## Observed Model Behavior and Interim Limitations

The fine-tuned adapter showed clear domain alignment with KCC-style advisory content, yet exhibited limitations stemming directly from the constrained training regime:

- **Response style mirroring / overfitting**  
  Outputs frequently adopted the terse, bullet-list-heavy structure typical of the original KCC expert replies instead of generating more natural, flowing conversational responses. This reflects the limited step count and lack of style-diversifying data.

- **Occasional lack of contextual flexibility**  
  Some responses reused near-verbatim canonical phrases from the dataset with minimal adaptation to nuanced query variations — a predictable outcome of under-exposure during short training.

- **Incomplete convergence**  
  The step-limited schedule validated pipeline correctness, inference coherence, and conversion compatibility but did not allow full loss stabilization or deeper generalization. Extended multi-epoch training on more capable hardware is expected to yield smoother fluency, reduced repetition, and stronger reasoning.

These characteristics were observed before LiteRT conversion and packaging, and are attributable to training duration/hardware limits rather than artifacts from the deployment pipeline.

## Limitations and Future Work

The dominant limitation remains significant undertraining due to restricted GPU access and Colab session volatility. While the current step-based runs provide a verified, reproducible baseline for the full pipeline (data → LoRA → merge → LiteRT-LM), they fall short of producing a production-grade advisory model.

Future iterations will address this through:

- Multi-epoch fine-tuning on dedicated GPU instances (e.g., AWS A10G/A100)
- Controlled experiments with targeted data augmentation (if needed)
- Broader multilingual and region-specific evaluation
- Quantitative latency/quality benchmarks post-LiteRT-LM conversion

Even with these constraints, the work establishes an early-validated deployment path — a deliberate choice to surface feasibility and failure modes before over-investing in training.

## Key Takeaway

This effort prioritizes engineering pragmatism over peak benchmark scores: catching tooling instabilities early, confirming conversion/deployability during short runs, and transparently documenting hardware-imposed trade-offs. These practices are essential when bridging research prototypes to resource-constrained, on-device real-world systems such as farmer advisory tools.