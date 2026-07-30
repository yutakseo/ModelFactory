# ModelFactory Docker Workspace

![ModelFactory AI model workshop](assets/modelfactory-hero_title.png)

PyTorch와 Hugging Face로 모델을 개발하고, NVIDIA Triton Inference Server와
TensorRT-LLM으로 모델을 서빙하기 위한 Docker Compose 환경입니다.

## 사전 요구사항

- Docker 및 Docker Compose
- NVIDIA GPU 드라이버
- NVIDIA Container Toolkit
- 호스트 데이터셋 경로 `/mnt/datasets` (사용하지 않으면 Compose에서 제거 가능)

## 서비스

### `huggingface`

모델 개발, 학습 및 변환 작업을 위한 컨테이너입니다.

- 로컬 이미지: `mf-huggingface:1.0`
- 베이스 이미지: `pytorch/pytorch:2.11.0-cuda12.8-cudnn9-devel`
- 컨테이너 이름: `huggingface`
- 작업 경로: `/workspace`
- Codex 영속 경로: `/root/.codex`
- 공유 메모리: `128gb`
- GPU: 전체 GPU 사용
- 실행 명령: `sleep infinity`

`requirements.txt`에 정의된 JupyterLab, IPykernel, Transformers, Datasets,
Ultralytics 및 데이터 과학 패키지가 이미지 빌드 중 설치됩니다. PyTorch는
베이스 이미지에 포함되어 있습니다. Node.js와 npm은 베이스 이미지의
Ubuntu/Debian 패키지 저장소를 통해 설치됩니다.

### `triton`

TensorRT-LLM backend가 포함된 Triton 추론 서버입니다.

- 로컬 이미지: `mf-triton-server:1.0`
- 베이스 이미지: `nvcr.io/nvidia/tritonserver:25.12-trtllm-python-py3`
- 컨테이너 이름: `triton`
- 모델 저장소: `/models`
- 공유 메모리: `64gb`
- GPU: 전체 GPU 사용

| 용도 | 호스트 포트 | 컨테이너 포트 |
|---|---:|---:|
| HTTP API | `18000` | `8000` |
| gRPC API | `18001` | `8001` |
| Prometheus Metrics | `18002` | `8002` |

## 디렉터리 구조

```text
.
├── .dockerignore
├── docker-compose.yaml
├── requirements.txt
├── huggingface/
│   ├── Dockerfile
│   ├── workspace/
│   │   └── .cache/huggingface/   # 실행 중 생성되는 Hugging Face 캐시
│   ├── cache/                     # 컨테이너 생성 시 만들어지는 Codex 데이터
│   └── model_repository/         # Hugging Face와 Triton이 공유하는 모델 저장소
└── triton/
    └── Dockerfile
```

## 볼륨과 경로 연결

모든 작업 파일, 모델 저장소 및 캐시는 Docker named volume이 아닌 호스트의
`huggingface/` 폴더에 저장됩니다.

| 호스트 경로 | `huggingface` 컨테이너 | `triton` 컨테이너 |
|---|---|---|
| `./huggingface/workspace` | `/workspace` | - |
| `./huggingface/cache` | `/root/.codex` | - |
| `./huggingface/workspace/.cache/huggingface` | `/workspace/.cache/huggingface` | `/root/.cache/huggingface` |
| `./huggingface/model_repository` | `/workspace/model_repository` | `/models` |
| `/mnt/datasets` | `/workspace/datasets` | - |

Codex 실행 파일은 이미지의 `/opt/codex` 아래에 설치되고
`/usr/local/bin/codex`로 실행됩니다. 설정, 인증 및 세션 데이터가 기록되는
원래 사용자 경로 `/root/.codex`만 호스트의 `huggingface/cache` 폴더와
연결됩니다. 이 호스트 폴더는 저장소에 미리 만들지 않으며, 컨테이너를
처음 생성할 때 Docker가 자동으로 만듭니다. 따라서 Codex 데이터는
컨테이너의 `/workspace`와 호스트의 `huggingface/workspace` 안에 나타나지
않습니다.

PyTorch 컨테이너에서 `/workspace/model_repository`에 모델을 내보내면 호스트의
`huggingface/model_repository`에 저장되고, Triton은 같은 파일을 `/models`에서
읽습니다.

## 모델 개발에서 Triton 서빙까지

Hugging Face 컨테이너에서 모델을 학습·변환·양자화한 뒤 Triton 모델 저장소
형식으로 `/workspace/model_repository`에 내보냅니다.

```text
/workspace/model_repository/
└── <model-name>/
    ├── config.pbtxt
    └── 1/
        └── <quantized-model-file>
```

이 경로는 두 컨테이너에 동시에 마운트되어 별도 복사 없이 Triton에서
`/models/<model-name>`으로 보입니다. Triton은 기본 설정에서 시작 시 모델
저장소를 읽으므로, 실행 중에 새 모델을 추가했다면 Triton을 재시작합니다.

```bash
docker compose restart triton
curl http://localhost:18000/v2/health/ready
```

Hugging Face 캐시는 다음 경로를 사용합니다.

```text
HF_HOME=/workspace/.cache/huggingface
HF_HUB_CACHE=/workspace/.cache/huggingface/hub
HF_DATASETS_CACHE=/workspace/.cache/huggingface/datasets
```

## Hugging Face 토큰

비공개 모델이나 gated 모델을 사용하려면 실행 전에 토큰을 설정합니다.

```bash
export HF_TOKEN=your_huggingface_token
```

토큰을 설정하지 않아도 공개 모델은 사용할 수 있습니다.

## 실행 방법

두 이미지를 빌드하고 모든 서비스를 시작합니다.

```bash
docker compose up -d --build
```

서비스를 개별적으로 시작할 수도 있습니다.

```bash
docker compose up -d huggingface
docker compose up -d triton
```

PyTorch 컨테이너에 접속합니다.

```bash
docker exec -it huggingface bash
```

GPU 인식 여부를 확인합니다.

```bash
docker exec huggingface python -c \
  "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

Triton 로그와 준비 상태를 확인합니다.

```bash
docker compose logs -f triton
curl http://localhost:18000/v2/health/ready
```

Prometheus metrics는 다음 주소에서 확인할 수 있습니다.

```text
http://localhost:18002/metrics
```

서비스를 종료합니다. 호스트의 모델과 캐시는 그대로 유지됩니다.

```bash
docker compose down
```

## 의존성 변경

`requirements.txt`를 변경한 뒤 PyTorch 이미지를 다시 빌드합니다.

```bash
docker compose build huggingface
docker compose up -d huggingface
```

## 참고사항

- 두 서비스가 동시에 같은 GPU를 사용하면 GPU 메모리가 부족할 수 있습니다.
- `shm_size`는 호스트의 실제 메모리 용량에 맞게 조절해야 합니다.
- `ipc: host`는 PyTorch 멀티프로세싱에 유용하지만 컨테이너 격리 수준을
  낮춥니다.
- Hugging Face 캐시, 모델 저장소 및 Codex 영속 데이터의 내용은
  `.gitignore`에서 제외되지만 호스트에는 계속 보존됩니다.
- Codex 인증 정보를 포함할 수 있는 `huggingface/cache`는
  `.dockerignore`에서도 제외되어 이미지 빌드 컨텍스트로 전송되지 않습니다.
