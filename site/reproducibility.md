# Reproducibility

This repository aims to be **fully reproducible** from the author's environment.

## Model Fine-tuning

Please see the [Jupyter notebook](https://github.com/Uralstech/Qwen-KCC-On-Device-Pipeline/blob/main/notebooks/Finetuning.ipynb) included in the repository for the fine tuning code.

## LiteRT Conversion

### Environment Setup

```bash
# Ensure Python 3.11+

git clone https://github.com/google-ai-edge/ai-edge-torch.git
cd ai-edge-torch

# Checkout to latest stable release
git checkout -b build "v0.7.1"

# Below is an example, install the version of ensurepip
# which corresponds to the installed Python 3 version
sudo apt update & sudo apt install python3-ensurepip

python -m venv --prompt ai-edge-torch venv
source venv/bin/activate

pip install ai-edge-torch
```

### Transfer artifacts to EC2

```bash
scp -i access_key.pem -r Qwen-2.5-1.5B-KCC ubuntu@<EC2_IP>:/home/ubuntu/
```

### Conversion

```bash
python3 -m ai_edge_torch.generative.examples.qwen.convert_to_tflite --model_size=1.5b --checkpoint_path="/home/ubuntu/Qwen-2.5-1.5B-KCC" --output_path="/home/ubuntu/" --output_name_prefix="tflite-" --kv_cache_max_len=4096
```

### Download Converted Model

```bash
scp -i access_key.pem ubuntu@<EC2_IP>:/home/ubuntu/tflite-_q8_ekv4096.tflite .
```

## LiteRT-LM Packaging

### Environment Setup

```bash
sudo apt update & sudo apt install build-essential clang

git clone https://github.com/google-ai-edge/LiteRT-LM.git
cd LiteRT-LM

# Checkout to latest stable release
git checkout -b build "v0.8.1"

wget --output-document "../bazel" https://github.com/bazelbuild/bazelisk/releases/download/v1.27.0/bazelisk-linux-amd64
chmod +x ../bazel

../bazel build //schema/cc:litertlm_export_main
../bazel build //schema/cc:litertlm_peek
```

### Transfer artifacts to EC2

See [llm_metadata.textproto](https://github.com/Uralstech/Qwen-KCC-On-Device-Pipeline/blob/main/proto/llm_metadata.textproto) in the repository for the LlmMetadata object used.

```bash
# Here, Qwen-2.5-1.5B-KCC_tflite is assumed to contain the HuggingFace
# tokenizer.json file, LlmMetadata .textproto file, and the tflite model.
scp -i access_key.pem -r Qwen-2.5-1.5B-KCC_tflite ubuntu@<EC2_IP>:/home/ubuntu/
```

### Packaging

```bash
./bazel-bin/schema/cc/litertlm_export_main \
  --hf_tokenizer_json_file="/home/ubuntu/Qwen-2.5-1.5B-KCC_tflite/tokenizer.json" \
  --tflite_file="/home/ubuntu/Qwen-2.5-1.5B-KCC_tflite/tflite-_q8_ekv4096.tflite" \
  --llm_metadata_text="/home/ubuntu/Qwen-2.5-1.5B-KCC_tflite/llm_metadata.textproto" \
  --output_path="/home/ubuntu/model.litertlm"

# Confirm all components have been successfully packaged into the .litertlm file
./bazel-bin/schema/cc/litertlm_peek --litertlm_file="/home/ubuntu/model.litertlm"
```

### Download Packaged Model

```bash
scp -i access_key.pem ubuntu@<EC2_IP>:/home/ubuntu/model.litertlm .
```