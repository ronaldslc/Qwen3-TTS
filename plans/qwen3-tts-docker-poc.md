# Qwen3 TTS Docker POC - Blackwell GPU Testing

## Objective

Get the Qwen3 TTS FastAPI server running on an RTX 5090 (Blackwell) GPU with CUDA 13.1.

## Source Repository

**Fork:** https://github.com/ronaldslc/Qwen3-TTS/tree/fastapi

## Status: ✅ COMPLETED

The Docker POC has been successfully built and tested on the RTX 5090 (Blackwell) GPU.

## Changes Made

### 1. Updated Dockerfile Base Images

The Dockerfile was updated to use NVIDIA NGC CUDA 13.1.1 images:

```dockerfile
# Base stage (line 7)
ARG BASE_IMAGE=nvcr.io/nvidia/cuda:13.1.1-runtime-ubuntu24.04

# Builder stage (line 53)
FROM nvcr.io/nvidia/cuda:13.1.1-devel-ubuntu24.04 AS builder
```

### 2. Updated Python Version

Changed from Python 3.11 to Python 3.12 (required for Ubuntu 24.04):

```dockerfile
# Base stage (lines 27-41)
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3.12 \
    python3.12-venv \
    python3.12-dev \
    ...

# Builder stage (lines 56-66)
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3.12 \
    python3.12-venv \
    python3.12-dev \
    ...
```

### 3. Added Build Parallelism Limits

To prevent OOM during flash-attn compilation, added environment variables to limit build parallelism:

```dockerfile
# Base stage (lines 16-19)
ENV MAX_JOBS=2
ENV MAKEFLAGS=-j2
ENV CMAKE_BUILD_PARALLEL_LEVEL=2

# Builder stage (lines 55-58)
ENV MAX_JOBS=2
ENV MAKEFLAGS=-j2
ENV CMAKE_BUILD_PARALLEL_LEVEL=2
```

### 4. Flash-Attention Installation

Flash-attn is installed with limited parallelism to prevent OOM:

```dockerfile
# Builder stage (lines 109-114)
RUN export MAKEFLAGS="-j3" && \
    export NINJAJOBS=3 && \
    export MAX_JOBS=3 && \
    pip install --no-cache-dir flash-attn --no-build-isolation || true
```

### 5. PyTorch CUDA Version

PyTorch is installed with CUDA 13.0 wheels (`cu130`):

```dockerfile
# Builder stage (lines 80-83)
RUN pip install --no-cache-dir \
    torch>=2.0.0 \
    torchaudio>=2.0.0 \
    --index-url https://download.pytorch.org/whl/cu130
```

## Execution Steps

### Step 1: Build the Docker Image

```bash
docker build -t qwen3-tts-blackwell --target production .
```

### Step 2: Run the Container

```bash
docker run -d --gpus all -p 8880:8880 --name qwen3-tts-blackwell qwen3-tts-blackwell
```

### Step 3: Verify It's Working

```bash
# Check if server is running
curl http://localhost:8880/v1/models

# List available voices
curl http://localhost:8880/v1/audio/voices
```

### Step 4: Test TTS Generation

```bash
# Test TTS endpoint
curl -X POST http://localhost:8880/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-tts",
    "input": "Hello, this is a test of Qwen3 TTS on Blackwell GPU.",
    "voice": "vivian",
    "language": "English"
  }' \
  -o test_output.wav
```

## Environment Details

- **GPU:** NVIDIA RTX 5090 (Blackwell architecture)
- **CUDA Version:** 13.1.1
- **Base Image:** `nvcr.io/nvidia/cuda:13.1.1-runtime-ubuntu24.04`
- **Python:** 3.12
- **PyTorch:** 2.10.0+cu130
- **Flash-Attention:** 2.8.3
- **Port:** 8880

## Verification Results

The container was successfully started and verified:

```
==========
== CUDA ==
==========
CUDA Version 13.1.1
...
2026-02-22 10:30:38,230 - api.backends.official_qwen3_tts - INFO - Successfully loaded model with flash_attention_2 attention
2026-02-22 10:30:38,230 - api.main - INFO - GPU: NVIDIA GeForce RTX 5090
2026-02-22 10:30:38,230 - api.main - INFO - VRAM: 31.84 GB
```

All endpoints tested successfully:
- ✅ `GET /v1/models` - 200 OK
- ✅ `GET /v1/audio/voices` - 200 OK  
- ✅ `POST /v1/audio/speech` - 200 OK (TTS generation works!)

## Troubleshooting

### Issue: OOM during build

**Solution:** Use limited parallelism for flash-attn:
```dockerfile
RUN export MAKEFLAGS="-j3" && \
    export NINJAJOBS=3 && \
    export MAX_JOBS=3 && \
    pip install --no-cache-dir flash-attn --no-build-isolation
```

### Issue: Python 3.11 not found

**Solution:** Use Python 3.12 for Ubuntu 24.04 base images.

## Success Criteria

1. ✅ Docker image builds successfully
2. ✅ Container starts without CUDA errors
3. ✅ `/v1/models` endpoint returns model info
4. ✅ `/v1/audio/voices` returns available voices
5. ✅ TTS generation produces valid audio file
6. ✅ Flash-attention is installed and working
7. ✅ GPU detected: NVIDIA GeForce RTX 5090

## Next Steps (After POC)

1. Document the exact Dockerfile changes needed
2. Update the main integration plan
3. Proceed to OpenClaw backend integration
