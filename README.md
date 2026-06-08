# Qwen hybrid local server

Docker Compose setup for a local Ollama server with an OpenAI-compatible API.

This stack is GPU-required hybrid by design:

- GPU must be visible to Docker, otherwise the stack fails closed.
- CPU/RAM offload is allowed for models that do not fit fully in VRAM.
- CPU-only inference is a failed deployment.

The default model is `qwen3:30b` with `OLLAMA_CONTEXT_LENGTH=4096`. On the checked `192.168.0.111` host, this is the practical maximum Qwen setup for mixed GPU+CPU execution: RTX 3070-class GPU plus 31 GiB RAM.

`qwen3:235b` is not viable here. Ollama lists it as a 142 GB model, while this host does not have 150+ GB usable RAM/VRAM.

## Required GPU control path

Do not start the stack until these commands pass on `192.168.0.111`:

```bash
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.9.0-base-ubuntu22.04 nvidia-smi
```

The last checked state of `192.168.0.111` did not pass this: `nvidia-smi`, `/dev/nvidia*`, and the NVIDIA container runtime were missing. The compose file fails closed instead of starting Ollama on CPU.

## Start

```bash
cp .env.example .env
docker compose up -d
docker compose logs --tail=100 ollama
docker compose logs --tail=100 qwen-pull
docker compose logs --tail=100 qwen-hybrid-smoke
```

The API is bound to localhost only:

```text
http://127.0.0.1:11434
```

## Verify

The `qwen-hybrid-smoke` service loads the configured model and fails unless `ollama ps` reports GPU participation.

Manual checks:

```bash
docker exec qwen-ollama ollama ps
curl http://localhost:11434/api/tags

curl http://localhost:11434/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3:30b",
    "messages": [
      { "role": "user", "content": "Reply with one short sentence." }
    ],
    "stream": false,
    "max_tokens": 64
  }'
```

For `qwen3:30b`, `ollama ps` must show `GPU` in `PROCESSOR`. A mixed CPU/GPU split is expected. `100% CPU` is a failed deployment.

## Model and context tuning

Default:

```dotenv
QWEN_MODEL=qwen3:30b
OLLAMA_CONTEXT_LENGTH=4096
OLLAMA_NUM_PARALLEL=1
OLLAMA_KEEP_ALIVE=10m
```

Keep `4096` for `qwen3:30b` on this host. Increasing context consumes more KV-cache memory and can reduce GPU offload or push the host into swap.

If you need larger context with stronger GPU usage, switch down:

```dotenv
QWEN_MODEL=qwen3:8b
OLLAMA_CONTEXT_LENGTH=8192
```

Then rerun:

```bash
docker compose up -d --force-recreate qwen-pull qwen-hybrid-smoke
docker compose logs --tail=100 qwen-hybrid-smoke
docker exec qwen-ollama ollama ps
```

## Stop

```bash
docker compose down
```

Model files remain in the `ollama-models` Docker volume.
