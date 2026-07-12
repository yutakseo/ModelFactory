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

- 로컬 이미지: `huggingface:latest`
- 베이스 이미지: `pytorch/pytorch:2.11.0-cuda12.8-cudnn9-devel`
- 컨테이너 이름: `huggingface`
- 작업 경로: `/workspace`
- 공유 메모리: `128gb`
- GPU: 전체 GPU 사용
- 실행 명령: `sleep infinity`

`requirements.txt`에 정의된 JupyterLab, IPykernel, Transformers, Datasets,
Ultralytics 및 데이터 과학 패키지가 이미지 빌드 중 설치됩니다. PyTorch는
베이스 이미지에 포함되어 있습니다.

### `triton`

TensorRT-LLM backend가 포함된 Triton 추론 서버입니다.

- 로컬 이미지: `triton-inference-server:latest`
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
├── docker-compose.yaml
├── requirements.txt
├── huggingface/
│   ├── Dockerfile
│   ├── .cache/huggingface/       # 실행 중 생성되는 Hugging Face 캐시
│   └── model_repository/         # Triton 모델 저장소
└── triton/
    └── Dockerfile
```

## 볼륨과 경로 연결

모든 작업 파일과 캐시는 Docker named volume이 아닌 호스트의 `huggingface/`
폴더에 저장됩니다.

| 호스트 경로 | `huggingface` 컨테이너 | `triton` 컨테이너 |
|---|---|---|
| `./huggingface` | `/workspace` | - |
| `./huggingface/.cache/huggingface` | `/workspace/.cache/huggingface` | `/root/.cache/huggingface` |
| `./huggingface/model_repository` | `/workspace/model_repository` | `/models` |
| `/mnt/datasets` | `/datasets` | - |

PyTorch 컨테이너에서 `/workspace/model_repository`에 모델을 내보내면 호스트의
`huggingface/model_repository`에 저장되고, Triton은 같은 파일을 `/models`에서
읽습니다.

일반적인 모델 저장소 구조는 다음과 같습니다.

```text
huggingface/model_repository/
└── <model-name>/
    ├── config.pbtxt
    └── 1/
        └── <model-file>
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
- `.cache`와 모델 저장소 내용은 `.gitignore`에서 제외되지만 호스트에는
  계속 보존됩니다.
