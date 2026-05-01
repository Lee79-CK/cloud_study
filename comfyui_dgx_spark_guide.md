# ComfyUI 완벽 가이드: 기본 개념부터 NVIDIA DGX Spark 실습까지

> **대상 독자**: LLMOps / MLOps / Cloud Engineer 백그라운드를 가진 엔지니어
> **소요 시간**: 이론 30분 + 실습 45분 ~ 1시간
> **선수 지식**: 리눅스 CLI, Docker 또는 Python venv 기본 사용 경험

---

## 강의 개요

이 강의는 세 부분으로 구성됩니다.

1. **1부 — 이론**: ComfyUI가 무엇이고 왜 쓰는지, 그리고 핵심 개념(노드, 워크플로우, 체크포인트, VAE, CLIP 등)을 이해합니다.
2. **2부 — 워크플로우 해부**: 가장 기본적인 Text-to-Image 워크플로우를 노드 단위로 뜯어보면서 "이미지 1장이 만들어지는 파이프라인"을 머릿속에 그립니다.
3. **3부 — DGX Spark 실습**: NVIDIA DGX Spark의 ARM64 + Blackwell(GB10) 환경에서 ComfyUI를 설치하고 실제로 이미지를 생성해봅니다. 마지막으로 운영(MLOps) 관점에서 신경 써야 할 포인트를 정리합니다.

---

# 1부. ComfyUI란 무엇인가?

## 1.1 한 문장 정의

**ComfyUI는 Stable Diffusion 같은 디퓨전(diffusion) 모델을 "노드 기반 그래프"로 실행하는 오픈소스 GUI 툴**입니다.

비유를 하나 들어보겠습니다.

- **AUTOMATIC1111 (A1111)** 같은 기존 WebUI = "버튼 누르면 이미지가 나오는 자판기"
- **ComfyUI** = "데이터 파이프라인을 직접 그리는 Apache Airflow / n8n 같은 도구"

LLMOps/MLOps 관점에서 익숙한 표현으로 바꾸면, ComfyUI는 **이미지 생성 파이프라인을 DAG(Directed Acyclic Graph)로 명시적으로 정의하고 실행하는 툴**입니다. 각 노드는 하나의 연산(모델 로드, 텍스트 인코딩, 샘플링, 디코딩, 저장 등)을 담당하고, 노드 간 엣지가 데이터 흐름을 나타냅니다.

## 1.2 왜 ComfyUI를 쓰는가?

엔지니어 입장에서 ComfyUI가 매력적인 이유는 다음과 같습니다.

**(1) 재현성(Reproducibility)이 매우 좋다.**
워크플로우 자체가 JSON으로 직렬화됩니다. 생성된 PNG 이미지의 메타데이터에 워크플로우 전체가 박혀 있어서, 누가 만든 이미지를 받아 ComfyUI에 드래그하면 **동일한 파이프라인이 그대로 재현**됩니다. MLOps에서 늘 강조하는 "실험 재현성"이 GUI 레벨에서 자연스럽게 보장됩니다.

**(2) VRAM 효율이 좋다.**
ComfyUI는 노드 그래프를 분석해서 필요한 모듈만 GPU에 올리고, 안 쓰는 건 내립니다. 같은 모델로 A1111이 메모리 부족(OOM)이 나는 상황에서도 ComfyUI는 돌아가는 경우가 흔합니다. DGX Spark처럼 **128GB 통합 메모리(unified memory)** 환경에서는 이 특성이 특히 빛납니다.

**(3) 커스터마이즈가 자유롭다.**
새로운 모델 아키텍처(예: SDXL → Flux → SD3 → Wan 2.x 등)가 나올 때마다 커스텀 노드(Custom Node)를 통해 빠르게 지원됩니다. 직접 Python으로 노드를 작성해서 자체 파이프라인에 끼워 넣을 수도 있습니다.

**(4) API 서버로 쓰기 쉽다.**
ComfyUI는 그 자체로 HTTP/WebSocket 서버입니다. 워크플로우 JSON을 POST하면 큐에 들어가고 결과를 받을 수 있어, **이미지 생성 백엔드 마이크로서비스**로 곧바로 활용 가능합니다. 사내 서비스의 이미지 생성 모듈로 박아 넣을 때 가장 흔히 쓰는 방식입니다.

## 1.3 ComfyUI가 잘 안 맞는 경우

물론 만능은 아닙니다.

- "한 줄 프롬프트 쳐서 이미지 한 장만 빠르게" 뽑고 싶은 비개발자에게는 진입장벽이 높습니다.
- 노드를 처음 보면 "이게 다 뭐지?" 싶어서 압도당할 수 있습니다.

이 강의는 그 진입장벽을 정확하게 무너뜨리는 게 목표입니다.

---

# 2부. 핵심 개념 정리

ComfyUI를 쓰려면 먼저 디퓨전 모델의 구성요소를 이해해야 합니다. 어렵지 않습니다. 부품 5개짜리 기계라고 생각하면 됩니다.

## 2.1 디퓨전 모델의 5대 구성요소

| 구성요소 | 역할 | 일상 비유 |
|---|---|---|
| **Checkpoint (체크포인트)** | 학습된 모델 가중치 전체. 보통 UNet + VAE + CLIP이 묶여 있음 | "공장 전체" |
| **UNet** | 노이즈를 점점 제거하면서 이미지를 그리는 핵심 신경망 | "그림 그리는 화가" |
| **VAE (Variational Auto-Encoder)** | 잠재공간(latent)과 픽셀공간(image)을 변환 | "스케치를 컬러 사진으로 현상해주는 인화기" |
| **CLIP (Text Encoder)** | 텍스트 프롬프트를 모델이 이해하는 벡터로 변환 | "한국어 → 화가의 언어 번역기" |
| **Sampler / Scheduler** | 노이즈를 몇 단계에 걸쳐 어떻게 제거할지 결정하는 알고리즘 | "화가의 작업 순서와 붓질 리듬" |

이 5개가 **순서대로 연결**되면 이미지 한 장이 나옵니다. 이 순서가 바로 ComfyUI의 노드 그래프입니다.

## 2.2 노드(Node)와 워크플로우(Workflow)

- **노드(Node)**: 하나의 연산 단위. 입력 소켓과 출력 소켓이 있고, 클릭해서 파라미터를 조정할 수 있는 박스
- **링크(Link)**: 노드의 출력 → 다른 노드의 입력으로 이어지는 연결선
- **워크플로우(Workflow)**: 노드 + 링크의 전체 그래프

워크플로우는 JSON으로 저장됩니다. Git에 커밋해서 버전 관리할 수 있고, 팀원과 공유할 수 있고, CI에서 자동으로 실행할 수도 있습니다. **여기가 MLOps 관점에서 가장 중요한 포인트**입니다.

## 2.3 잠재 공간(Latent Space)이라는 개념

ComfyUI 노드를 보면 `LATENT`라는 데이터 타입이 자주 등장합니다. 디퓨전 모델은 보통 픽셀 공간(예: 1024×1024×3)에서 직접 그림을 그리지 않고, **압축된 잠재 공간(예: 128×128×4)** 에서 작업한 뒤 마지막에 VAE로 픽셀 이미지로 풀어냅니다.

왜 이렇게 할까요? 메모리와 연산량이 수십 배 줄어들기 때문입니다. 그래서 워크플로우의 흐름은 이렇습니다.

```
Empty Latent (빈 도화지) → KSampler (노이즈 제거하며 그리기) → VAE Decode (픽셀로 변환) → Save Image
```

이 흐름을 머릿속에 박아두세요. 모든 ComfyUI 워크플로우의 뼈대가 이것입니다.

---

# 3부. 기본 워크플로우 해부 (Text-to-Image)

ComfyUI를 처음 켜면 기본 워크플로우(default workflow)가 자동으로 로드됩니다. 노드는 7개입니다. 하나씩 따라가 봅시다.

## 3.1 7개 노드의 역할

```
┌─────────────────────┐
│ Load Checkpoint     │  ① 체크포인트 파일에서 MODEL, CLIP, VAE 3개를 꺼냄
└──┬───────┬───────┬──┘
   │MODEL  │CLIP   │VAE
   │       │       │
   │       ├──────────┐
   │       ▼          ▼
   │  ┌──────────┐ ┌──────────┐
   │  │CLIP Text │ │CLIP Text │  ②③ Positive / Negative 프롬프트 인코딩
   │  │ (Pos)    │ │ (Neg)    │
   │  └────┬─────┘ └────┬─────┘
   │       │CONDITIONING│
   │       │            │
   ▼       ▼            ▼
┌─────────────────────────────┐    ┌──────────────────┐
│       KSampler              │ ◄──│ Empty Latent     │ ④ 빈 잠재 도화지
│ (디노이징 루프)              │    │  Image           │
└──────────────┬──────────────┘    └──────────────────┘
               │ LATENT                ⑤ 메인 샘플러
               ▼
        ┌──────────────┐
        │ VAE Decode   │  ⑥ 잠재 → 픽셀 변환
        └──────┬───────┘
               │ IMAGE
               ▼
        ┌──────────────┐
        │ Save Image   │  ⑦ 디스크에 PNG 저장
        └──────────────┘
```

## 3.2 노드별 상세 설명

**① Load Checkpoint**
체크포인트 파일(예: `v1-5-pruned-emaonly-fp16.safetensors`)을 불러와서 MODEL(UNet), CLIP, VAE 세 개의 출력 소켓을 제공합니다. 가장 먼저 실행되고, GPU에 모델 가중치를 로드하는 단계라 시간이 가장 오래 걸립니다.

**② CLIP Text Encode (Positive Prompt)**
"a cat sitting on a sofa, photorealistic, 4k" 같은 **원하는 것**을 텍스트로 입력합니다. CLIP이 이 문장을 768차원 또는 1024차원 벡터로 변환합니다.

**③ CLIP Text Encode (Negative Prompt)**
"blurry, low quality, deformed" 같은 **피하고 싶은 것**을 입력합니다. 모델이 이 방향으로는 가지 않도록 가이드합니다.

**④ Empty Latent Image**
"가로 × 세로 × 배치 크기"로 빈 잠재 도화지를 만듭니다. 예: 1024×1024 이미지를 만들고 싶으면 잠재 공간에서는 128×128로 시작합니다(VAE 압축률 8배 기준).

**⑤ KSampler — 가장 중요한 노드**
실제로 이미지를 "그리는" 노드입니다. 다음 파라미터들이 결과를 좌우합니다.

- `seed`: 난수 시드. 같은 시드 + 같은 워크플로우 = 같은 이미지(재현성!)
- `steps`: 디노이징 단계 수. 보통 SD1.5는 20~30, SDXL은 25~40
- `cfg`: Classifier Free Guidance scale. 높을수록 프롬프트에 충실하지만 너무 높으면 이미지가 망가짐. 보통 6~8
- `sampler_name`: 샘플링 알고리즘. `euler`, `dpmpp_2m`, `dpmpp_2m_sde` 등
- `scheduler`: 노이즈 스케줄. `normal`, `karras`, `exponential` 등
- `denoise`: 1.0이면 처음부터 그림, 0.5면 절반만 변형 (img2img에서 사용)

**⑥ VAE Decode**
디노이징된 잠재 텐서를 받아서 RGB 픽셀 이미지로 변환합니다.

**⑦ Save Image**
PNG로 저장합니다. 이때 PNG 메타데이터에 **워크플로우 JSON 전체가 자동으로 박힙니다**. 이게 ComfyUI의 시그니처 기능입니다.

## 3.3 한 번만 머릿속에 그려두면 끝

다른 어떤 복잡한 워크플로우(LoRA, ControlNet, IPAdapter, img2img, inpainting, video generation 등)도 모두 **이 7-노드 뼈대의 변형**입니다. 노드 몇 개를 끼워 넣거나 갈아 끼우는 거지, 구조 자체는 똑같습니다.

---

# 4부. NVIDIA DGX Spark 실습

이제 본격적으로 DGX Spark에서 ComfyUI를 돌려봅시다.

## 4.1 DGX Spark 빠른 소개

NVIDIA DGX Spark는 데스크탑 폼팩터의 "개인용 AI 슈퍼컴퓨터"입니다. 핵심 스펙을 정리하면:

- **SoC**: NVIDIA GB10 Grace Blackwell Superchip (TSMC 3nm)
- **CPU**: 20코어 ARM (Cortex-X925 10개 + Cortex-A725 10개) — MediaTek 협업
- **GPU**: Blackwell 아키텍처, 6144 CUDA 코어, **컴퓨팅 캐퍼빌리티 sm_121**
- **메모리**: 128GB LPDDR5X **통합 메모리** (CPU/GPU coherent), 약 273 GB/s 대역폭
- **스토리지**: 4TB NVMe SSD
- **네트워크**: ConnectX-7 SmartNIC (200GbE × 2)
- **AI 성능**: FP4 sparsity 기준 최대 1 PFLOP
- **OS**: DGX OS (Ubuntu 24.04 LTS 기반)
- **최대 모델 규모**: 단일 머신 200B 파라미터, 2대 클러스터링 시 405B 파라미터

엔지니어가 알아야 할 두 가지 중요한 특성:

**(1) ARM64 아키텍처입니다.**
일반적인 x86_64용 Docker 이미지나 PyTorch 휠이 그대로 안 돌아가는 경우가 있습니다. DGX Spark용으로 빌드된 이미지/패키지를 써야 합니다.

**(2) GPU의 컴퓨팅 캐퍼빌리티가 sm_121입니다.**
이건 매우 새로운 값입니다. 따라서 **CUDA 13.0 이상 + 최신 PyTorch**가 필요하고, 일부 가속 라이브러리(Triton, SageAttention 등)는 sm_121을 지원하도록 별도로 빌드된 버전을 써야 합니다.

## 4.2 사전 준비

### 4.2.1 DGX Spark 접속

DGX Spark는 두 가지 방식으로 접근 가능합니다.

**방식 A — 직접 연결**: 키보드/마우스/모니터를 꽂고 일반 Ubuntu PC처럼 사용

**방식 B — 헤드리스(추천)**: NVIDIA Sync 앱이나 SSH로 원격 접속

엔지니어 워크플로우에서는 보통 B를 씁니다. SSH 접속:

```bash
ssh <username>@<spark-ip>
```

### 4.2.2 시스템 점검

먼저 환경이 제대로 갖춰졌는지 확인합니다.

```bash
# Python 버전 확인 (3.10 이상 권장)
python3 --version

# pip 확인
pip3 --version

# CUDA 툴킷 확인 (13.0 이상이 보여야 함)
nvcc --version

# GPU 인식 확인
nvidia-smi
```

`nvidia-smi`에서 "GB10" 또는 Blackwell GPU가 잡히면 OK입니다. CUDA 버전이 13.0 미만이면 NVIDIA 공식 가이드대로 드라이버/툴킷을 업데이트하세요.

## 4.3 설치 방법: 두 가지 길

DGX Spark에서 ComfyUI를 돌리는 방법은 크게 두 가지입니다.

| 방법 | 장점 | 단점 | 추천 대상 |
|---|---|---|---|
| **A. 네이티브 설치 (venv)** | 단순, 디버깅 쉬움, NVIDIA 공식 가이드 존재 | 의존성 충돌 가능, 환경 격리가 약함 | 처음 써보는 경우, 빠른 PoC |
| **B. Docker 컨테이너** | 환경 격리, 재배포 용이, MLOps 친화적 | 초기 빌드 시간, ARM64 + sm_121용 이미지 필요 | 운영 환경, 팀 공유, 재현성 중시 |

이 강의에서는 두 가지를 모두 안내하되, **방법 A를 메인 실습**으로 진행합니다.

---

## 4.4 실습 A — 네이티브 설치 (NVIDIA 공식 절차)

NVIDIA의 build.nvidia.com 공식 플레이북을 따라가는 방식입니다. 약 45분 소요됩니다.

### Step 1. 가상 환경 생성

호스트 시스템 패키지와 충돌을 피하기 위해 venv를 만듭니다.

```bash
cd ~
python3 -m venv comfyui-env
source comfyui-env/bin/activate
```

프롬프트 앞에 `(comfyui-env)`가 붙으면 활성화 성공.

### Step 2. PyTorch 설치 (Blackwell 호환)

DGX Spark의 GPU(sm_121)는 **CUDA 13.0 호환** PyTorch가 필요합니다. NVIDIA 공식 가이드는 다음 명령을 안내합니다.

```bash
pip3 install torch torchvision --index-url https://download.pytorch.org/whl/cu130
```

> **주의**: 일반 PyPI(`pip install torch`)로 설치하면 sm_121 미지원 버전이 깔려서 GPU를 못 씁니다. 반드시 위 인덱스 URL을 쓰세요.

설치 검증:

```bash
python3 -c "import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

`True`와 함께 GPU 이름이 출력되면 OK.

### Step 3. ComfyUI 클론

```bash
cd ~
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI/
```

### Step 4. 의존성 설치

```bash
pip install -r requirements.txt
```

웹 인터페이스 컴포넌트, 모델 핸들러 등 ComfyUI 동작에 필요한 모든 패키지가 설치됩니다.

### Step 5. 체크포인트 모델 다운로드

가장 가벼운 SD 1.5 모델로 시작합니다. 약 2GB.

```bash
cd models/checkpoints/
wget https://huggingface.co/Comfy-Org/stable-diffusion-v1-5-archive/resolve/main/v1-5-pruned-emaonly-fp16.safetensors
cd ../../
```

> ComfyUI의 모델 디렉토리 구조는 다음과 같습니다. 외워두면 편합니다.
> ```
> ComfyUI/models/
> ├── checkpoints/   ← 메인 모델 (.safetensors, .ckpt)
> ├── vae/           ← 분리된 VAE
> ├── loras/         ← LoRA 파인튜닝 가중치
> ├── controlnet/    ← ControlNet 모델
> ├── clip/          ← 분리된 CLIP
> └── upscale_models/← 업스케일러
> ```

### Step 6. ComfyUI 서버 실행

네트워크 전체 인터페이스에 바인드해서, 다른 머신에서도 브라우저로 접속할 수 있게 합니다.

```bash
python main.py --listen 0.0.0.0
```

기본 포트는 **8188**입니다.

### Step 7. 동작 확인

다른 터미널에서 헬스체크:

```bash
curl -I http://localhost:8188
```

`HTTP/1.1 200 OK`가 나오면 성공.

이제 같은 네트워크의 PC/맥/태블릿 브라우저에서 다음 주소로 접속:

```
http://<SPARK_IP>:8188
```

ComfyUI 웹 UI가 뜨면 기본 워크플로우(2부에서 해부한 7-노드 그래프)가 자동으로 로드됩니다.

### Step 8. 첫 이미지 생성

1. **Load Checkpoint** 노드에서 `v1-5-pruned-emaonly-fp16.safetensors` 선택
2. Positive 프롬프트에 원하는 문장 입력 (예: `a futuristic city at sunset, cinematic, ultra detailed`)
3. 우측 상단 **Queue Prompt** 또는 **Run** 버튼 클릭
4. 보통 30~60초 안에 이미지 생성 완료

별도 터미널에서 `nvidia-smi`를 1초 간격으로 모니터링해보세요.

```bash
watch -n 1 nvidia-smi
```

KSampler가 도는 동안 GPU 사용률이 치솟는 게 보일 겁니다.

### Step 9. (선택) 정리/롤백

설치를 완전히 제거하려면:

```bash
deactivate
rm -rf ~/comfyui-env/
rm -rf ~/ComfyUI/
```

---

## 4.5 실습 B — Docker 기반 설치 (운영 환경 추천)

운영 환경에 배포하거나 팀에 동일한 환경을 공유하려면 Docker가 답입니다. DGX Spark는 ARM64 + sm_121이라는 특수 조합이라 일반 ComfyUI Docker 이미지가 안 돌아갑니다. 다행히 커뮤니티에서 DGX Spark 전용 이미지를 만들어두었습니다.

대표적인 옵션 두 가지:

**(1) `mmartial/comfyui-nvidia-docker`** (DGX Spark 빌드 포함)
GitHub: `mmartial/ComfyUI-Nvidia-Docker`
ARM64 + CUDA 13.1 베이스, SageAttention 빌드 스크립트 포함, 비루트 실행 지원.

```bash
# 이미지 빌드 (또는 DockerHub에서 pull)
git clone https://github.com/mmartial/ComfyUI-Nvidia-Docker.git
cd ComfyUI-Nvidia-Docker
make build-dgx

# compose 파일로 기동
docker compose -f compose-dgx_spark.yml up -d
```

**(2) `ecarmen16/SparkyUI`**
GitHub: `ecarmen16/SparkyUI`
DGX Spark 전용으로 처음부터 설계된 컨테이너. 통합 메모리 패브릭에 맞춘 ComfyUI 실행 옵션이 사전 적용되어 있습니다. 핵심 옵션:

- `--disable-pinned-memory` — 통합 메모리 패브릭에서 오버헤드 감소
- `--force-fp16`, `--fp16-unet`, `--fp16-vae`, `--fp16-text-enc` — 전 구간 FP16
- `--dont-upcast-attention` — 어텐션을 FP16으로 유지

**핵심 통찰**: SparkyUI 문서가 강조하는 한 마디 — *"Don't fight the fabric."* DGX Spark의 통합 메모리 위에서는 `--gpu-only`나 `--cache-none` 같은 강제 GPU 옵션이 오히려 성능을 떨어뜨립니다. 일반 GPU 머신과 다른 멘탈 모델이 필요합니다.

## 4.6 DGX Spark 특화 성능 팁

### 4.6.1 통합 메모리를 활용하라

일반적인 RTX 5090 같은 머신은 24GB VRAM이 한계입니다. ComfyUI에서 큰 모델(SDXL, Flux, Wan 비디오 모델 등)을 돌리려면 모델 오프로딩, 시퀀셜 로딩 등 온갖 트릭이 필요합니다.

DGX Spark는 **128GB가 통째로 GPU 메모리처럼 동작**합니다. 그래서:

- Flux.1 (12B 파라미터) 같은 대형 모델도 양자화 없이 로드 가능
- Multiple LoRA 동시 로드, 대형 ControlNet 체이닝, 긴 시퀀스 비디오 생성에 유리
- 다만 메모리 **대역폭**이 273 GB/s로 RTX 5090(GDDR7 1.7 TB/s)보다 훨씬 낮으므로, **순수 처리 속도(throughput)는 5090보다 느립니다**

→ **DGX Spark의 가치 = 절대 속도가 아니라 "큰 모델을 돌릴 수 있다"와 "조용한 데스크탑에서 1 PFLOP 클래스 워크로드를 돌린다"** 입니다. 벤치마크 비교할 때 이 점을 잊지 마세요.

### 4.6.2 SageAttention 빌드 (선택, 고급)

기본 어텐션 구현보다 빠른 SageAttention을 sm_121용으로 빌드하면 SDXL 기준 약 1.3~1.7배 가속을 볼 수 있습니다. 위에서 소개한 Docker 이미지들이 빌드 스크립트를 포함하고 있으니 그걸 쓰는 게 가장 빠릅니다.

### 4.6.3 NVFP4 양자화

Blackwell 5세대 텐서 코어는 NVFP4(4-bit floating point) 연산을 네이티브 지원합니다. ComfyUI 자체는 아직 NVFP4 가중치 포맷을 광범위하게 지원하지 않지만, NVIDIA가 모델 변환 가이드를 공개하고 있어 향후 더 많은 모델이 4-bit로 돌아갈 예정입니다. (build.nvidia.com의 "NVFP4 Quantization" 플레이북 참고)

---

# 5부. MLOps / 운영 관점 체크리스트

DGX Spark + ComfyUI를 단순 실험용이 아니라 **사내 서비스 백엔드**나 **팀 공유 인프라**로 운영할 때 신경 써야 할 포인트입니다.

## 5.1 워크플로우 버전 관리

워크플로우 JSON을 Git 리포지토리에 두세요. 디렉토리 구조 예시:

```
my-comfy-workflows/
├── workflows/
│   ├── product-photo-v1.json
│   ├── product-photo-v2.json
│   └── thumbnail-generator.json
├── models-manifest.yaml      # 어떤 체크포인트/LoRA를 쓰는지 기록
└── README.md
```

PR 리뷰에서 노드 추가/파라미터 변경을 추적할 수 있고, 실패한 워크플로우의 원인을 git blame으로 추적 가능합니다.

## 5.2 모델 자산 관리 (Model Registry)

체크포인트, LoRA 등 모델 파일은 보통 GB 단위라 Git에 올리면 안 됩니다. 옵션:

- **Hugging Face Hub** (private repo + git-lfs)
- **MinIO / S3** + 사내 다운로드 스크립트
- **NVIDIA NGC** 모델 카탈로그
- **DVC** (Data Version Control)

모델 매니페스트(`models-manifest.yaml`)에 SHA256 해시까지 적어두면 변조 감지에도 좋습니다.

## 5.3 API 모드로 띄우기

ComfyUI는 GUI를 안 띄우고도 REST/WebSocket API로 동작합니다.

```bash
python main.py --listen 0.0.0.0 --port 8188 --disable-auto-launch
```

워크플로우 JSON을 POST하는 엔드포인트:

```
POST http://<spark-ip>:8188/prompt
GET  http://<spark-ip>:8188/history/<prompt_id>
```

프론트엔드 서비스 → 큐(Redis/RabbitMQ) → ComfyUI worker(여러 대의 Spark) 같은 구조로 확장하면 자연스러운 마이크로서비스가 됩니다.

## 5.4 모니터링

LLMOps에서 익숙한 패턴 그대로 적용 가능합니다.

- **GPU 메트릭**: `dcgm-exporter` → Prometheus → Grafana
- **ComfyUI 큐 상태**: `/queue` 엔드포인트를 폴링해서 메트릭 노출
- **이미지 생성 시간 / 성공률 / VRAM 피크**: 각 워크플로우 실행에 trace ID를 붙여서 로깅

## 5.5 보안 주의사항

기본 ComfyUI 서버는 **인증이 없습니다**. `--listen 0.0.0.0`으로 외부에 열면 누구나 GPU를 쓸 수 있게 됩니다. 운영 시:

- 사내망/VPN 안에서만 접근 가능하게 방화벽 설정
- 또는 nginx + Basic Auth / OAuth 프록시로 감싸기
- Tailscale로 zero-trust 네트워킹 구축 (NVIDIA가 DGX Spark용 Tailscale 가이드도 제공)
- 커스텀 노드는 임의 코드 실행이 가능하니, 신뢰 가능한 출처에서만 설치

---

# 6부. 다음 단계 학습 로드맵

이 강의를 마쳤다면 다음 순서로 깊이를 더해가세요.

1. **LoRA 적용 워크플로우**: 캐릭터 일관성, 화풍 고정에 필수
2. **ControlNet**: 포즈/엣지/뎁스 등으로 구도를 제어
3. **IPAdapter / Reference**: 레퍼런스 이미지 기반 생성
4. **img2img / Inpainting**: 부분 수정 워크플로우
5. **SDXL → Flux.1 → Wan 2.x** 등 최신 모델로 이주
6. **API 모드**: 워크플로우 JSON을 코드에서 직접 호출하는 패턴
7. **DGX Spark 클러스터링**: ConnectX-7으로 Spark 2대 묶어서 405B 파라미터급 모델 다루기

---

## 정리: 한 페이지 요약

- **ComfyUI**는 디퓨전 모델을 노드 그래프로 명시적으로 정의/실행하는 툴이다.
- 핵심 7-노드 뼈대(`Checkpoint → CLIP × 2 → EmptyLatent → KSampler → VAE Decode → Save`)만 이해하면 모든 워크플로우가 이 변형으로 보인다.
- 워크플로우는 JSON으로 직렬화되고 PNG 메타데이터에도 박힌다 → **재현성이 디폴트**다.
- **DGX Spark**는 ARM64 + Blackwell sm_121 + 128GB 통합 메모리 + Ubuntu 24.04(DGX OS) 환경이다.
- ComfyUI 설치는 NVIDIA 공식 venv 방식(`pip install torch --index-url .../cu130` → 클론 → 모델 다운 → `python main.py --listen 0.0.0.0`) 또는 ARM64/sm_121 전용 Docker 이미지로 한다.
- DGX Spark의 강점은 **절대 속도가 아니라 큰 모델을 돌릴 수 있다는 것**. 통합 메모리에 맞는 옵션을 쓰고, 일반 GPU 패턴(`--gpu-only` 등)을 강제하지 마라.
- 운영 시에는 워크플로우 Git 관리 + 모델 매니페스트 + API 모드 + 인증 프록시 + GPU 모니터링을 갖춰야 한다.

수고하셨습니다. 이제 DGX Spark에 SSH로 접속해서 직접 한 번 돌려보시면 됩니다. 🚀
