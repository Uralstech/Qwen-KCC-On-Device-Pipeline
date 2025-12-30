# Project Overview

This repository documents a practical, artifact-driven pipeline to build and deploy an on-device farmer advisory model using the KCC dataset. The pipeline comprises two main stages:

- **Fine-tuning:**
  - Dataset: ["Farmers Call Query Data"](https://www.kaggle.com/datasets/daskoushik/farmers-call-query-data-qa). A cleaning step was applied to remove all null and empty rows; the corresponding code is included in this repository.

    > This was generated using data from data.gov.in, an open data platform by Govt. of India.

  - Base model: ["unsloth/Qwen2.5-1.5B-Instruct"](https://huggingface.co/unsloth/Qwen2.5-1.5B-Instruct) — chosen for its strong instruction-following performance and suitability for edge deployment.

- **Conversion & Packaging:**
  - The fine-tuned model was converted to LiteRT format (formerly TFLite) using the ["AI Edge Torch"](https://github.com/google-ai-edge/ai-edge-torch) toolkit.
  - The resulting LiteRT model was packaged into LiteRT-LM format using utilities from the ["LiteRT-LM"](https://github.com/google-ai-edge/LiteRT-LM) project. The final packaged model artifact is available on Hugging Face at <https://doi.org/10.57967/hf/7392>.

This pipeline closely mirrors the engineering principles and reproducibility focus of the prior Gemma-3n work, now adapted to the Qwen2.5 family to enable efficient on-device inference while preserving domain-specific relevance for advisory tasks among Indian smallholder farmers.