# NVIDIA TensorRT 완벽 가이드: 기본 개념부터 DGX Spark 실습까지

---

## 1. TensorRT란 무엇인가

NVIDIA TensorRT는 학습이 완료된 딥러닝 모델을 NVIDIA GPU 위에서 **최고 성능으로 추론(Inference)** 할 수 있도록 최적화하는 SDK이다. PyTorch, TensorFlow, ONNX 등 주요 프레임워크에서 학습된 모델을 입력으로 받아, 해당 모델의 연산 그래프를 분석하고 GPU 하드웨어에 맞춰 재구성함으로써 추론 지연시간(Latency)을 낮추고 처리량(Throughput)을 극대화한다.

핵심을 한 문장으로 요약하면 다음과 같다.

> **TensorRT = 학습된 모델 → GPU 최적화 추론 엔진 변환기**

TensorRT 에코시스템은 단일 라이브러리가 아니라, 여러 구성 요소가 협력하는 생태계로 확장되었다.

| 구성 요소 | 역할 |
|---|---|
| **TensorRT (Core)** | ONNX 모델을 받아 최적화된 추론 엔진을 빌드·실행 |
| **TensorRT-LLM** | LLM 특화 추론 라이브러리 (In-Flight Batching, KV Cache, 텐서 병렬 등) |
| **TensorRT Model Optimizer** | 양자화(Quantization), 프루닝(Pruning), 증류(Distillation) 등 모델 압축 |
| **Torch-TensorRT** | PyTorch 모듈을 TensorRT 엔진으로 직접 컴파일 |
| **TensorRT Cloud** | CLI 기반으로 원격에서 초최적화 엔진을 생성하는 서비스 |
| **Triton Inference Server** | 멀티 모델·멀티 GPU 서빙을 위한 상위 레벨 추론 서버 |

---

## 2. TensorRT의 핵심 최적화 기법

TensorRT가 모델을 최적화할 때 내부적으로 적용하는 주요 기법들을 이해하면, 왜 같은 모델이라도 TensorRT 엔진으로 변환하면 수배 빨라지는지 알 수 있다.

### 2.1 레이어 및 텐서 퓨전 (Layer & Tensor Fusion)

딥러닝 모델은 수백~수천 개의 개별 레이어로 구성된다. GPU에서 각 레이어를 개별 커널로 실행하면, 커널 런치 오버헤드와 메모리 읽기/쓰기가 반복되어 비효율이 발생한다. TensorRT는 Conv + BatchNorm + ReLU처럼 연속된 레이어들을 하나의 CUDA 커널로 합쳐(fuse) 실행한다. 이로써 중간 텐서를 글로벌 메모리에 저장했다가 다시 읽는 과정이 사라지고, 커널 런치 횟수가 대폭 감소한다.

### 2.2 혼합 정밀도 (Mixed Precision)

TensorRT는 FP32, FP16, BF16, FP8, INT8, 그리고 최신 NVFP4까지 다양한 수치 정밀도를 지원한다. 모델의 각 레이어별로 정밀도를 달리 설정할 수 있으며, 정확도 손실이 허용 범위 내에 있는 레이어는 더 낮은 정밀도로 연산하여 텐서 코어 활용률을 극대화한다. 예를 들어 Blackwell 아키텍처의 5세대 텐서 코어는 FP4 연산을 네이티브로 지원하므로, NVFP4 양자화 시 FP16 대비 메모리를 1/4로 줄이면서 처리량은 크게 향상된다.

### 2.3 커널 자동 튜닝 (Kernel Auto-Tuning)

TensorRT는 빌드 시점에 타겟 GPU에서 다양한 CUDA 커널 구현체를 실제로 실행해 보고(Profiling), 각 레이어에 가장 빠른 커널을 선택한다. 이 과정을 통해 동일 모델이라도 A100, H100, Blackwell 등 GPU 세대별로 최적의 커널 조합이 자동으로 결정된다.

### 2.4 동적 텐서 메모리 관리

TensorRT는 추론 과정에서 필요한 중간 텐서의 생명 주기를 분석하여, 동시에 사용되지 않는 텐서끼리 GPU 메모리를 공유하도록 할당한다. 이 최적화 덕분에 전체 메모리 사용량이 크게 줄어들어, 더 큰 배치 사이즈나 더 큰 모델을 같은 GPU에서 실행할 수 있다.

### 2.5 동적 셰이프 (Dynamic Shapes)

실제 서비스 환경에서는 입력 크기가 요청마다 달라진다. TensorRT는 빌드 시 최소·최적·최대 입력 크기를 지정하면, 런타임에 해당 범위 내의 어떤 입력 크기에도 대응하는 단일 엔진을 생성한다. 이를 통해 NLP 모델의 가변 시퀀스 길이나 CV 모델의 가변 이미지 해상도를 하나의 엔진으로 처리할 수 있다.

---

## 3. TensorRT 워크플로우: 모델에서 엔진까지

TensorRT를 활용한 추론 파이프라인의 전체 흐름은 다음과 같다.

```
[학습된 모델]
    │
    ▼
[ONNX 변환] ─── PyTorch: torch.onnx.export()
    │              TensorFlow: tf2onnx
    ▼
[Polygraphy로 상수 폴딩] ─── 선택적이지만 권장
    │
    ▼
[TensorRT 엔진 빌드]
    │  ├─ 정밀도 설정 (FP16, INT8, FP8, NVFP4 등)
    │  ├─ 동적 셰이프 프로파일 설정
    │  ├─ 커널 자동 튜닝 실행
    │  └─ 레이어 퓨전 최적화
    │
    ▼
[직렬화된 엔진 파일 (.engine / .plan)]
    │
    ▼
[런타임 추론 실행]
    ├─ TensorRT Runtime (C++ / Python)
    ├─ Triton Inference Server
    └─ Torch-TensorRT (PyTorch 통합)
```

### 3.1 ONNX 변환

TensorRT의 주요 모델 임포트 경로는 ONNX(Open Neural Network Exchange)이다. PyTorch에서는 `torch.onnx.export()`를 통해, TensorFlow에서는 `tf2onnx` 도구를 통해 ONNX 포맷으로 변환한다.

```python
# PyTorch → ONNX 변환 예시
import torch

model = MyModel().eval().cuda()
dummy_input = torch.randn(1, 3, 224, 224).cuda()

torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    opset_version=17,
    input_names=["input"],
    output_names=["output"],
    dynamic_axes={"input": {0: "batch_size"}, "output": {0: "batch_size"}}
)
```

### 3.2 엔진 빌드 (Python API)

```python
import tensorrt as trt

logger = trt.Logger(trt.Logger.WARNING)
builder = trt.Builder(logger)
network = builder.create_network(
    1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH)
)
parser = trt.OnnxParser(network, logger)

# ONNX 모델 파싱
with open("model.onnx", "rb") as f:
    parser.parse(f.read())

# 빌드 설정
config = builder.create_builder_config()
config.set_memory_pool_limit(trt.MemoryPoolType.WORKSPACE, 1 << 30)  # 1GB

# FP16 혼합 정밀도 활성화
config.set_flag(trt.BuilderFlag.FP16)

# 동적 셰이프 프로파일
profile = builder.create_optimization_profile()
profile.set_shape("input", (1, 3, 224, 224), (8, 3, 224, 224), (32, 3, 224, 224))
config.add_optimization_profile(profile)

# 엔진 빌드 및 직렬화
serialized_engine = builder.build_serialized_network(network, config)
with open("model.engine", "wb") as f:
    f.write(serialized_engine)
```

### 3.3 엔진 로드 및 추론

```python
import tensorrt as trt
import numpy as np
import pycuda.driver as cuda
import pycuda.autoinit

# 엔진 로드
runtime = trt.Runtime(trt.Logger(trt.Logger.WARNING))
with open("model.engine", "rb") as f:
    engine = runtime.deserialize_cuda_engine(f.read())

context = engine.create_execution_context()
context.set_input_shape("input", (1, 3, 224, 224))

# 입출력 버퍼 할당
input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)
output_data = np.empty((1, 1000), dtype=np.float32)

d_input = cuda.mem_alloc(input_data.nbytes)
d_output = cuda.mem_alloc(output_data.nbytes)

cuda.memcpy_htod(d_input, input_data)
context.execute_v2(bindings=[int(d_input), int(d_output)])
cuda.memcpy_dtoh(output_data, d_output)

print("추론 결과:", output_data.argmax())
```

---

## 4. Torch-TensorRT: PyTorch 네이티브 통합

매번 ONNX로 변환하는 과정이 번거롭다면, Torch-TensorRT를 사용하여 PyTorch 코드 내에서 직접 TensorRT 최적화를 적용할 수 있다.

```python
import torch
import torch_tensorrt

model = MyModel().eval().cuda()

# Torch-TensorRT 컴파일
optimized_model = torch_tensorrt.compile(
    model,
    inputs=[
        torch_tensorrt.Input(
            min_shape=(1, 3, 224, 224),
            opt_shape=(8, 3, 224, 224),
            max_shape=(32, 3, 224, 224),
            dtype=torch.float16
        )
    ],
    enabled_precisions={torch.float16},
    workspace_size=1 << 30
)

# 일반 PyTorch 모델처럼 사용
input_tensor = torch.randn(8, 3, 224, 224).cuda().half()
output = optimized_model(input_tensor)
```

Torch-TensorRT는 내부적으로 모델 그래프를 분석하여 TensorRT로 가속 가능한 서브그래프와 PyTorch 네이티브로 실행할 서브그래프를 자동 분리한다. 따라서 TensorRT가 지원하지 않는 커스텀 연산이 포함된 모델도 문제없이 최적화할 수 있다.

---

## 5. TensorRT-LLM: 대규모 언어 모델 추론 최적화

TensorRT-LLM은 LLM 추론에 특화된 오픈소스 라이브러리로, 일반 TensorRT 위에 LLM 서빙에 필수적인 다음 기능들을 추가로 제공한다.

### 5.1 핵심 기능

**In-Flight Batching**: 전통적인 배칭은 한 배치 내 모든 요청의 생성이 끝날 때까지 기다려야 한다. In-Flight Batching은 생성이 완료된 요청 슬롯에 새 요청을 즉시 삽입하여 GPU 유휴 시간을 최소화한다.

**Paged Attention & KV Cache 관리**: LLM의 어텐션 레이어에서 생성되는 Key-Value 캐시를 페이지 단위로 관리하여 메모리 단편화를 방지하고, 더 많은 동시 요청을 처리할 수 있게 한다.

**텐서/파이프라인/전문가 병렬화**: 모델이 단일 GPU 메모리에 들어가지 않을 때, 여러 GPU 또는 노드에 걸쳐 모델을 분할하여 분산 추론을 수행한다.

**투기적 디코딩(Speculative Decoding)**: 작은 드래프트 모델이 여러 토큰을 미리 생성하고, 큰 타겟 모델이 한 번에 검증하는 방식으로 토큰 생성 속도를 높인다.

### 5.2 지원 모델 예시

TensorRT-LLM은 Llama, Mistral, Mixtral, DeepSeek, Falcon, GPT, Phi 등 주요 LLM 아키텍처를 폭넓게 지원하며, 새 모델이 출시되면 빠르게 지원을 추가하는 것을 목표로 한다. Blackwell GPU에서는 NVFP4와 FP8 양자화를 통해 70B 파라미터 모델도 단일 노드에서 효율적으로 서빙할 수 있다.

---

## 6. 실제 활용 사례

### 6.1 실시간 컴퓨터 비전 서비스

자율주행, 제조 라인 검사, 스마트 시티 등에서 카메라 영상을 실시간으로 분석해야 하는 경우, 객체 탐지 모델(YOLO, EfficientDet 등)을 TensorRT로 변환하면 프레임당 처리 시간을 수 밀리초 수준으로 줄일 수 있다. NVIDIA의 Metropolis 프레임워크는 TensorRT 엔진을 핵심 추론 백엔드로 사용한다.

### 6.2 LLM 서빙 인프라

GPT 계열, Llama, DeepSeek 등 대규모 언어 모델을 프로덕션 환경에서 서빙할 때, TensorRT-LLM + Triton Inference Server 조합은 사실상의 표준이 되고 있다. NVFP4/FP8 양자화로 메모리 사용량을 줄이고, In-Flight Batching으로 처리량을 극대화하며, 텐서 병렬화로 초대형 모델도 분산 서빙할 수 있다.

### 6.3 이미지 생성 (Diffusion 모델)

Stable Diffusion, FLUX 등 디퓨전 모델의 UNet/Transformer 블록을 TensorRT로 변환하면 이미지 생성 시간이 크게 단축된다. NVIDIA의 TensorRT 데모에는 Stable Diffusion XL 파이프라인이 포함되어 있으며, Model Optimizer를 함께 사용해 FP8 양자화까지 적용하면 추가적인 속도 향상이 가능하다.

### 6.4 음성 인식 및 TTS

Whisper, Riva 등 음성 인식 모델의 인코더/디코더를 TensorRT로 최적화하여 실시간 스트리밍 음성 인식 파이프라인을 구축하는 사례도 활발하다.

### 6.5 추천 시스템

대규모 추천 모델(DLRM 등)은 높은 처리량이 필요하다. TensorRT로 임베딩 테이블 참조를 제외한 DNN 부분을 최적화하고, Triton Inference Server에서 앙상블 파이프라인으로 구성하여 초당 수백만 건의 추천 요청을 처리한다.

---

## 7. NVIDIA DGX Spark 소개

### 7.1 하드웨어 개요

DGX Spark는 NVIDIA GB10 Grace Blackwell Superchip을 탑재한 데스크톱 크기의 AI 슈퍼컴퓨터이다. 2025년 3월 GTC에서 발표되어 10월에 출시되었으며, 가격은 약 $3,999(Founder's Edition)부터 시작한다.

| 항목 | 사양 |
|---|---|
| **SoC** | NVIDIA GB10 Grace Blackwell Superchip (MediaTek 공동 설계) |
| **CPU** | 20코어 ARM (Cortex-X925 ×10 + Cortex-A725 ×10) |
| **GPU** | Blackwell 아키텍처, 5세대 텐서 코어 |
| **AI 성능** | 최대 1 PFLOP (FP4 스파시티 기준) / 1,000 TOPS |
| **메모리** | 128GB LPDDR5X 통합 메모리 (~273 GB/s) |
| **스토리지** | 4TB NVMe SSD |
| **네트워크** | ConnectX-7 200Gb/s (QSFP ×2), 2대 연결 시 405B 모델까지 |
| **크기/무게** | 150 × 150 × 50.5mm / 약 1.2kg |
| **전력** | USB-C 전원, 최대 240W (외장 PSU) |
| **OS** | DGX OS (Ubuntu 24.04 기반 커스텀 Linux) |

### 7.2 왜 DGX Spark에서 TensorRT인가

DGX Spark의 가장 큰 장점은 **128GB 통합 메모리(Unified Memory)**이다. CPU와 GPU가 하나의 메모리 풀을 공유하므로, 기존 디스크리트 GPU에서 필수적이었던 시스템 RAM ↔ VRAM 간 PCIe 데이터 전송이 사라진다. 70B 파라미터 FP16 모델은 약 140GB의 메모리를 필요로 하는데, 일반 GPU로는 2대의 H100 80GB가 필요하지만 DGX Spark에서는 NVFP4 양자화를 적용하면 단일 유닛에서 로드할 수 있다.

TensorRT는 이 통합 메모리 위에서 커널 퓨전, 양자화, 자동 튜닝을 수행하여, DGX Spark의 제한된 메모리 대역폭(273 GB/s)을 최대한 효율적으로 활용한다. 연산 밀도가 높아질수록 대역폭 병목이 줄어들기 때문에, TensorRT의 최적화는 DGX Spark 같은 대역폭 제약 환경에서 더욱 빛을 발한다.

### 7.3 DGX Spark 소프트웨어 스택

DGX OS에는 다음이 사전 설치되어 있다.

- CUDA Toolkit
- cuDNN
- **TensorRT**
- NVIDIA Container Runtime (Docker GPU 지원)
- NGC CLI (NVIDIA GPU Cloud 컨테이너 레지스트리 접근)
- DGX Dashboard (웹 기반 시스템 모니터링 + JupyterLab)

ARM64(aarch64) 아키텍처이므로, x86 전용으로 빌드된 일부 라이브러리는 ARM64 빌드를 사용하거나 소스에서 컴파일해야 할 수 있다는 점을 유의해야 한다.

---

## 8. DGX Spark에서 TensorRT 실습

### 8.1 실습 1: TensorRT-LLM으로 LLM 추론 서빙

DGX Spark에서 가장 임팩트 있는 TensorRT 활용은 LLM 추론이다. NVIDIA가 제공하는 공식 DGX Spark Playbook을 기반으로 실습한다.

**Step 1 — 환경 준비**

```bash
# HuggingFace 토큰 설정 (비공개 모델 접근 시)
export HF_TOKEN="<your_huggingface_token>"

# DGX Spark 전용 TensorRT-LLM 컨테이너 풀
docker pull nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev
```

**Step 2 — 사전 양자화된 모델로 서빙 (예: Llama 3.1 8B NVFP4)**

```bash
# 모델 핸들 설정
export MODEL_HANDLE="nvidia/Llama-3.1-8B-Instruct-NVFP4"

# TensorRT-LLM 서빙 시작
docker run --rm -it --gpus all \
  -e HF_TOKEN="$HF_TOKEN" \
  -p 8000:8000 \
  nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev \
  trtllm-serve \
    --model "$MODEL_HANDLE" \
    --backend tensorrt \
    --max_batch_size 4 \
    --max_input_len 2048 \
    --max_output_len 512
```

**Step 3 — API 호출 테스트**

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Llama-3.1-8B-Instruct-NVFP4",
    "messages": [
      {"role": "user", "content": "TensorRT의 핵심 최적화 기법 3가지를 설명해줘."}
    ],
    "max_tokens": 512
  }'
```

서빙 서버는 OpenAI 호환 API를 제공하므로, 기존 OpenAI SDK 기반 애플리케이션의 엔드포인트만 변경하면 로컬 추론으로 전환할 수 있다.

### 8.2 실습 2: 컴퓨터 비전 모델 TensorRT 최적화 (YOLO)

```bash
# Docker 컨테이너 실행
t=ultralytics/ultralytics:latest-nvidia-arm64
sudo docker pull $t
sudo docker run -it --ipc=host --runtime=nvidia --gpus all $t
```

컨테이너 내부에서 다음을 실행한다.

```python
from ultralytics import YOLO

# 모델 로드
model = YOLO("yolo11n.pt")

# TensorRT 엔진으로 변환 (FP16)
model.export(format="engine", half=True)

# TensorRT 엔진으로 추론
trt_model = YOLO("yolo11n.engine")
results = trt_model.predict(source="test_image.jpg", show=False)
print(f"탐지된 객체 수: {len(results[0].boxes)}")
```

DGX Spark의 Blackwell 텐서 코어와 TensorRT 엔진이 결합되어, 소형 YOLO 모델 기준으로 수백 FPS의 추론 성능을 달성할 수 있다.

### 8.3 실습 3: Diffusion 모델을 TensorRT로 가속

```bash
# PyTorch NGC 컨테이너 실행
docker run --gpus all --ipc=host --ulimit memlock=-1 \
  --ulimit stack=67108864 -it --rm \
  -v $HOME/.cache/huggingface:/root/.cache/huggingface \
  nvcr.io/nvidia/pytorch:25.11-py3
```

컨테이너 내부에서 TensorRT Diffusion 데모를 설정한다.

```bash
# TensorRT 리포지토리 클론
git clone https://github.com/NVIDIA/TensorRT.git -b main --single-branch
cd TensorRT

export TRT_OSSPATH=/workspace/TensorRT/
cd $TRT_OSSPATH/demo/Diffusion

# 의존성 설치
pip install nvidia-modelopt[onnx,hf]
sed -i '/^nvidia-modelopt\[.*\]=.*/d' requirements.txt
pip install -r requirements.txt

apt-get update && \
apt-get install -y libgl1 libglib2.0-0 libsm6 libxrender1 libxext6
```

```bash
# Stable Diffusion XL TensorRT 파이프라인 실행
python demo_txt2img_xl.py \
  "A futuristic cityscape at sunset, cyberpunk style" \
  --version xl-1.0 \
  --num-warmup-runs 1 \
  --use-cuda-graph
```

TensorRT는 UNet의 복잡한 어텐션 블록과 컨볼루션 레이어를 퓨전하고, FP16/FP8으로 양자화하여 이미지 생성 시간을 PyTorch 대비 크게 단축시킨다.

### 8.4 실습 4: NVFP4 커스텀 양자화

사전 양자화 체크포인트가 없는 모델을 직접 NVFP4로 양자화하여 DGX Spark에 배포할 수 있다.

```python
import modelopt.torch.quantization as mtq
from modelopt.torch.export import export_tensorrt_llm_checkpoint

# 모델 로드
model = load_my_model()  # HuggingFace 등에서 로드

# NVFP4 양자화 캘리브레이션
mtq.quantize(
    model,
    config=mtq.NVFP4_DEFAULT_CFG,
    forward_loop=calibration_loop  # 대표 데이터로 캘리브레이션
)

# TensorRT-LLM 체크포인트로 내보내기
export_tensorrt_llm_checkpoint(
    model,
    decoder_type="llama",
    dtype="bfloat16",
    export_dir="./nvfp4_checkpoint",
    inference_tensor_parallel=1
)
```

NVFP4 양자화는 70B 모델을 약 35GB로 압축하여 DGX Spark의 128GB 메모리 내에서 여유 있게 서빙할 수 있게 하며, KV Cache를 위한 메모리까지 확보할 수 있다.

### 8.5 실습 5: 2대의 DGX Spark 클러스터링

DGX Spark는 ConnectX-7 200Gb/s 네트워크를 통해 2대를 연결하여 하나의 추론 클러스터로 사용할 수 있다. 이를 통해 256GB의 통합 메모리 풀을 확보하고, 텐서 병렬화를 사용하여 Llama 3.1 405B 같은 초대형 모델도 로컬에서 추론할 수 있다.

```bash
# 두 번째 Spark와 NCCL 기반 텐서 병렬 추론
# Spark 1 (리더)
docker run --rm -it --gpus all \
  --network host \
  -e NCCL_SOCKET_IFNAME=enp1s0 \
  nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev \
  trtllm-serve \
    --model "nvidia/Llama-3.1-70B-Instruct-NVFP4" \
    --tp_size 2 \
    --rank 0
```

---

## 9. MLOps/LLMOps 관점에서의 TensorRT 통합

프로덕션 ML 파이프라인에서 TensorRT를 통합할 때 고려해야 할 운영 측면을 정리한다.

### 9.1 CI/CD 파이프라인 통합

TensorRT 엔진은 타겟 GPU에서 빌드해야 하므로, CI/CD 파이프라인에 GPU 러너를 포함시켜야 한다. 모델이 업데이트될 때마다 ONNX 변환 → TensorRT 빌드 → 정확도 검증 → 성능 벤치마크 → 배포를 자동화하는 것이 이상적이다. TensorRT Cloud를 활용하면 원격에서 빌드를 수행할 수 있어 로컬 GPU 의존성을 줄일 수도 있다.

### 9.2 모델 버저닝과 레지스트리

TensorRT 엔진은 GPU 아키텍처와 TensorRT 버전에 종속적이다. 따라서 모델 레지스트리에 엔진을 저장할 때는 반드시 빌드 환경 메타데이터(GPU 타입, TensorRT 버전, CUDA 버전, 양자화 설정 등)를 함께 기록해야 한다.

### 9.3 모니터링 및 프로파일링

NVIDIA Nsight Systems를 사용하면 TensorRT 엔진의 런타임 성능을 상세히 프로파일링할 수 있다. Nsight Deep Learning Designer는 ONNX 모델 편집, 성능 프로파일링, TensorRT 엔진 빌드를 통합한 IDE 환경을 제공한다. 서빙 시에는 Triton Inference Server의 메트릭을 Prometheus + Grafana로 수집하여 추론 지연시간, 처리량, GPU 활용률을 실시간 모니터링한다.

### 9.4 DGX Spark의 MLOps 포지셔닝

DGX Spark는 프로덕션 서빙 머신이라기보다는 **개발·실험·프로토타이핑 스테이지**에 위치한다. DGX OS가 데이터센터의 DGX 시스템과 동일한 소프트웨어 스택을 사용하므로, Spark에서 검증된 TensorRT 엔진과 서빙 설정을 클라우드나 데이터센터의 H100/B200 클러스터로 그대로 이관할 수 있다. 이 "dev → staging → prod" 흐름의 일관성이 DGX Spark의 핵심 MLOps 가치이다.

---

## 10. 성능 최적화 팁

### 10.1 빌드 시간 최적화

TensorRT 엔진 빌드는 커널 자동 튜닝 때문에 수 분에서 수십 분이 걸릴 수 있다. `timing_cache`를 활성화하면 한 번 측정된 커널 타이밍 정보를 캐싱하여 재빌드 시간을 크게 줄일 수 있다.

```python
# 타이밍 캐시 활성화
config = builder.create_builder_config()
cache = config.create_timing_cache(b"")
config.set_timing_cache(cache, ignore_mismatch=False)
```

### 10.2 DGX Spark에서의 메모리 관리

128GB 통합 메모리는 모델 가중치, KV Cache, 활성화 텐서가 모두 공유한다. 대형 모델 서빙 시 KV Cache 크기를 적절히 제한하여 OOM(Out of Memory)을 방지해야 한다.

```bash
# KV Cache 메모리 비율 조정
trtllm-serve \
  --model "..." \
  --kv_cache_free_gpu_memory_fraction 0.4
```

### 10.3 ARM64 호환성 체크리스트

DGX Spark는 ARM64 아키텍처이므로 다음을 확인해야 한다.

- NGC 컨테이너는 ARM64 태그(`-arm64` 또는 DGX Spark 전용 태그) 사용
- 커스텀 TensorRT 플러그인은 ARM64 대상으로 크로스 컴파일 필요
- x86 전용 `.so` 파일은 동작하지 않음 (반드시 aarch64용으로 재빌드)
- CUDA compute capability 확인: GB10은 12.1

---

## 11. 참고 자료

| 리소스 | URL |
|---|---|
| TensorRT 공식 문서 | https://docs.nvidia.com/deeplearning/tensorrt/latest/ |
| TensorRT-LLM 문서 | https://nvidia.github.io/TensorRT-LLM/ |
| TensorRT GitHub | https://github.com/NVIDIA/TensorRT |
| DGX Spark Playbooks | https://github.com/NVIDIA/dgx-spark-playbooks |
| DGX Spark 공식 빌드 가이드 | https://build.nvidia.com/spark |
| Torch-TensorRT | https://github.com/pytorch/TensorRT |
| TensorRT Model Optimizer | https://github.com/NVIDIA/TensorRT-Model-Optimizer |
| NVIDIA Triton Inference Server | https://github.com/triton-inference-server/server |
| DGX Spark 사용자 가이드 | https://docs.nvidia.com/dgx/dgx-spark/ |

---

> **문서 작성일**: 2026년 4월  
> **대상 독자**: LLMOps / MLOps / Cloud 엔지니어  
> **참고**: 본 문서의 코드 예시는 TensorRT 10.x 및 DGX OS 7.x 환경을 기준으로 작성되었습니다. 버전에 따라 API가 다를 수 있으므로 공식 문서를 병행 참조하시기 바랍니다.
