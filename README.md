# Hugging Face Docker Workspace

This repository provides a Docker Compose environment for running Hugging Face
Transformers with PyTorch GPU support.

## Requirements

- Docker
- Docker Compose
- NVIDIA GPU driver
- NVIDIA Container Toolkit
- A host dataset directory at `/mnt/datasets` if you want to use the mounted
  datasets path

## Services

### `pytorch`

The Compose file starts one long-running container:

- Image: locally built `huggingface-workspace:latest`
- Base image: `pytorch/pytorch:2.11.0-cuda12.8-cudnn9-devel`
- Container name: `pytorch`
- Working directory: `/workspace`
- GPU access: `gpus: all`
- Shared memory: `128gb`
- IPC mode: `host`
- Command: `sleep infinity`

The container is intended to stay running so you can enter it interactively and
run notebooks, scripts, training jobs, or inference commands.

### `triton`

The Triton container uses NVIDIA's TensorRT-LLM-enabled Triton image and serves
the shared model repository over host ports HTTP (`18000`), gRPC (`18001`), and
metrics (`18002`). These map to container ports `8000`, `8001`, and `8002`.

## Directory Layout

```text
.
├── huggingface/
│   └── Dockerfile
├── triton/
│   └── Dockerfile
├── docker-compose.yaml
├── requirements.txt
├── README.md
```

The `huggingface` directory is mounted into the container at `/workspace`. Put
your training scripts, notebooks, model code, and experiment files there.

## Cache and Data Mounts

The PyTorch container uses these Hugging Face cache paths, all visible under
the host's `huggingface/.cache/huggingface` directory:

```text
HF_HOME=/workspace/.cache/huggingface
HF_HUB_CACHE=/workspace/.cache/huggingface/hub
HF_DATASETS_CACHE=/workspace/.cache/huggingface/datasets
```

The same host cache is mounted in the Triton container at:

```text
/root/.cache/huggingface
```

Create or export Triton models from the PyTorch container under:

```text
/workspace/model_repository
```

This is the host's `huggingface/model_repository` directory, mounted into the
Triton container at `/models`. For example:

```text
huggingface/model_repository/<model-name>/config.pbtxt
huggingface/model_repository/<model-name>/1/<model-file>
```

The host path `/mnt/datasets` is mounted into the container as:

```text
/datasets
```

Create `/mnt/datasets` on the host or edit `docker-compose.yaml` if your datasets
are stored somewhere else.

## Hugging Face Token

The Compose file reads `HF_TOKEN` from your shell environment:

```bash
export HF_TOKEN=your_huggingface_token
```

This is optional, but required for private models, gated models, or authenticated
Hub access.

## Usage

Build and start both containers:

```bash
docker compose up -d
```

Start only the development container or only the inference server:

```bash
docker compose up -d pytorch
docker compose up -d triton
```

On the first run, Compose builds the local image and installs the packages from
`requirements.txt`. Rebuild the image after changing dependencies:

```bash
docker compose build
docker compose up -d
```

The dependency set includes JupyterLab, IPykernel, common Hugging Face
libraries, and standard data science tools. PyTorch is provided by the base
image.

Open an interactive shell:

```bash
docker exec -it pytorch bash
```

Run a quick GPU check inside the container:

```bash
python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

Stop the container:

```bash
docker compose down
```

Stop the container and remove the named cache volume:

```bash
docker compose down -v
```

## Notes

- Local build outputs use `latest`; their base images are pinned in each
  Dockerfile.
- `shm_size: "128gb"` is useful for large dataloaders, but it should not exceed
  what your host can reasonably provide.
- `ipc: host` is commonly used for PyTorch multiprocessing workloads, but it
  reduces container isolation.
