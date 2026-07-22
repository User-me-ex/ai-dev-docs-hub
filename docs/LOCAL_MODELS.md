# Local Models

> Guide to setting up, configuring, and managing local model inference for AI Dev OS. Covers supported engines, model downloading, performance tuning, and troubleshooting.

## Overview

Local models provide zero-cost inference with no external API dependencies, full data privacy, and offline capability. AI Dev OS supports three local engines:

| Engine | Platform | Format | GPU Support |
|--------|----------|--------|-------------|
| **Ollama** | Linux, macOS, Windows | GGUF | CUDA, Vulkan, Metal |
| **llama.cpp** | Linux, macOS, Windows | GGUF | CUDA, Vulkan, Metal, SYCL |
| **MLX** | macOS (Apple Silicon) | safetensors | Metal (M1–M4) |

Ollama is the recommended default. llama.cpp is for advanced GPU tuning. MLX is for Apple Silicon only.

## Setup

### Ollama
```
curl -fsSL https://ollama.com/install.sh | sh    # Linux/macOS
# Windows: download from https://ollama.com/download/windows
ollama --version
```
```yaml
providers: { ollama: { base_url: http://127.0.0.1:11434, default_keep_alive: 5m, auto_pull: false } }
```

### llama.cpp
```
git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
cmake -B build -DGGML_CUDA=ON && cmake --build build --config Release -j
./build/bin/llama-server -m model.gguf --host 127.0.0.1 --port 8080
```
```yaml
providers: { llamacpp: { base_url: http://127.0.0.1:8080 } }
```

### MLX
```
pip install mlx-lm && mlx_lm.server --model mlx-community/Qwen2.5-7B-4bit --port 8080
```
```yaml
providers: { mlx: { base_url: http://127.0.0.1:8080 } }
```

## Model Downloading

| Engine | Command |
|--------|---------|
| Ollama | `ollama pull <name>` (e.g., `ollama pull llama3.2:3b`) |
| llama.cpp | Download .gguf from Hugging Face |
| MLX | Pre-converted models on Hugging Face (`mlx-community`) |

Set `auto_pull: true` to auto-fetch missing models.

## Configuration

| Engine | Context param | GPU param | Thread param |
|--------|--------------|-----------|-------------|
| Ollama | `num_ctx` (default 2048) | `OLLAMA_NUM_PARALLEL` | `num_thread` (auto) |
| llama.cpp | `ctx_size` (default 512) | `-ngl N` | `-t N` |
| MLX | `max_tokens` (default 2048) | Automatic (always Metal) | N/A |

Recommended `num_ctx`: 4096–8192. For llama.cpp, `-ngl 99` offloads all layers to GPU.

## Performance Tuning

| Quant | Size vs FP16 | Quality | When to use |
|-------|-------------|---------|-------------|
| FP16 | 1× | Baseline | 24 GB+ VRAM |
| Q8_0 | 0.5× | Negligible | Best for 7B on 12 GB |
| Q4_K_M | 0.27× | Low loss | Default |
| Q3_K_M | 0.20× | Moderate | Large models on limited VRAM |
| Q2_K | 0.15× | Significant | Last resort |

Choose the highest quantization fitting entirely in VRAM. 8 GB → Q4_K_M for 7B. 24 GB → Q8_0 for 7B.

## Health Checks

| Engine | Endpoint | Success |
|--------|----------|---------|
| Ollama | `GET /api/version` | `{ "version": "0.5.x" }` |
| llama.cpp | `GET /v1/models` | `{ "object": "list" }` |
| MLX | `GET /v1/models` | `{ "object": "list" }` |

Adapters poll every 30 s and publish state transitions to the SCE.

## Troubleshooting

| Symptom | Cause | Solution |
|---------|-------|----------|
| OOM | Model too large for VRAM | Lower quantization or smaller model |
| < 5 tok/s | Too few GPU layers | Increase `-ngl` |
| Repetitive output | Temperature too low | Increase to 0.7–0.9 |
| Incoherent output | Excessive quantization | Use Q4_K_M or higher |
| Model reloading per request | No keep-alive | Set `keep_alive: 5m` (Ollama) |

## Related Documents
- [Model Providers](./MODEL_PROVIDERS.md)
- [Ollama Integration](./OLLAMA_INTEGRATION.md)
- [Performance](./PERFORMANCE.md)
- [Installation](./INSTALLATION.md)
- [Model Discovery](./MODEL_DISCOVERY.md)
- [Cost Management](./COST_MANAGEMENT.md)

