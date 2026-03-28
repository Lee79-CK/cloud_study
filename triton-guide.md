# NVIDIA Triton Inference Server 완벽 가이드

## 목차
1. 기본 개념
2. 아키텍처 심층 분석
3. 고급 응용 개념
4. Step-by-Step 실습: 환경 구축부터 프로덕션 배포까지

---

## 1. 기본 개념

### 1.1 Triton이란?

NVIDIA Triton Inference Server는 다양한 딥러닝/머신러닝 프레임워크의 모델을 **단일 서버에서 통합 서빙**할 수 있는 오픈소스 추론 서버다. GPU/CPU 리소스를 최적화하면서 높은 처리량(throughput)과 낮은 지연(latency)을 동시에 달성하는 것이 핵심 목표다.

### 1.2 핵심 특징

**멀티 프레임워크 지원**: TensorRT, TensorFlow, PyTorch, ONNX Runtime, OpenVINO, Python Backend 등을 동시에 서빙할 수 있다. 하나의 Triton 인스턴스에서 TensorRT 모델과 PyTorch 모델을 동시에 로드하고, 각각 최적의 백엔드로 추론을 실행한다.

**동적 배칭(Dynamic Batching)**: 개별 요청을 자동으로 묶어 하나의 배치로 처리한다. GPU utilization을 극대화하면서도 개별 요청의 latency SLA를 준수할 수 있다.

**동시 모델 실행(Concurrent Model Execution)**: 하나의 GPU에서 여러 모델을 동시에 실행하거나, 하나의 모델을 여러 GPU에 분산 실행할 수 있다.

**모델 앙상블(Model Ensemble)**: 여러 모델을 DAG(Directed Acyclic Graph) 형태로 연결하여 파이프라인을 구성한다. 전처리 → 추론 → 후처리를 서버 내부에서 처리하므로 네트워크 오버헤드가 제거된다.

**모델 관리**: 모델의 로드/언로드를 런타임에 수행할 수 있으며, A/B 테스트를 위한 모델 버전 관리를 지원한다.

### 1.3 지원 백엔드

| 백엔드 | 용도 | 최적 시나리오 |
|--------|------|--------------|
| TensorRT | NVIDIA 최적화 추론 엔진 | 최대 성능이 필요한 프로덕션 |
| ONNX Runtime | 크로스 플랫폼 추론 | 멀티 프레임워크 모델 통합 |
| PyTorch (LibTorch) | PyTorch 네이티브 | R&D에서 빠른 서빙 전환 |
| TensorFlow | TF SavedModel | TF 에코시스템 모델 |
| Python Backend | 커스텀 로직 | 전처리/후처리, LLM 서빙 |
| vLLM Backend | LLM 최적화 | 대규모 언어 모델 서빙 |
| FIL (Forest Inference) | 트리 기반 모델 | XGBoost, LightGBM, Random Forest |

### 1.4 Model Repository 구조

Triton의 모든 것은 Model Repository에서 시작된다. 파일 시스템 기반의 디렉토리 구조로, 로컬 경로, S3, GCS, Azure Blob Storage를 지원한다.

```
model_repository/
├── text_classifier/           # 모델 이름
│   ├── config.pbtxt           # 모델 설정 (필수)
│   ├── 1/                     # 버전 1
│   │   └── model.onnx         # 모델 파일
│   ├── 2/                     # 버전 2
│   │   └── model.onnx
│   └── labels.txt             # (선택) 레이블 파일
│
├── image_detector/
│   ├── config.pbtxt
│   └── 1/
│       └── model.plan         # TensorRT 엔진
│
└── ensemble_pipeline/
    ├── config.pbtxt           # 앙상블 설정
    └── 1/                     # 빈 디렉토리 (앙상블은 모델 파일 불필요)
```

### 1.5 config.pbtxt 기본 구조

```protobuf
name: "text_classifier"
platform: "onnxruntime_onnx"       # 백엔드 지정
max_batch_size: 64                 # 최대 배치 크기

input [
  {
    name: "input_ids"
    data_type: TYPE_INT64
    dims: [ 512 ]                  # 배치 차원 제외
  },
  {
    name: "attention_mask"
    data_type: TYPE_INT64
    dims: [ 512 ]
  }
]

output [
  {
    name: "logits"
    data_type: TYPE_FP32
    dims: [ 3 ]                    # 3-class classification
  }
]

# 버전 정책: 최신 2개 버전만 로드
version_policy: { latest { num_versions: 2 } }

# 인스턴스 그룹: GPU 0에 2개 인스턴스
instance_group [
  {
    count: 2
    kind: KIND_GPU
    gpus: [ 0 ]
  }
]

# 동적 배칭 설정
dynamic_batching {
  preferred_batch_size: [ 8, 16, 32 ]
  max_queue_delay_microseconds: 100
}
```

### 1.6 통신 프로토콜

Triton은 세 가지 통신 방식을 지원한다.

**HTTP/REST** (포트 8000): 범용 클라이언트 호환, 디버깅 용이. JSON 기반으로 직관적이지만 직렬화 오버헤드가 존재한다.

**gRPC** (포트 8001): 프로덕션 권장. Protocol Buffers 기반의 바이너리 직렬화로 HTTP 대비 2~5배 빠른 통신 성능을 제공한다. 스트리밍, 양방향 통신을 지원한다.

**C API**: Triton을 라이브러리로 임베드할 때 사용한다. 네트워크 오버헤드 없이 직접 호출이 가능하다.

---

## 2. 아키텍처 심층 분석

### 2.1 요청 처리 파이프라인

```
Client Request
     │
     ▼
┌─────────────┐
│ HTTP/gRPC   │  ← 프로토콜 디코딩
│ Frontend    │
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  Scheduler  │  ← Dynamic Batching / Sequence Batching
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  Backend    │  ← TensorRT / ONNX / PyTorch / Python
│  Framework  │
└─────┬───────┘
      │
      ▼
┌─────────────┐
│ Model       │  ← GPU/CPU 추론 실행
│ Instance    │
└─────┬───────┘
      │
      ▼
   Response
```

### 2.2 스케줄러 유형

**Default Scheduler**: 배칭 없이 요청을 즉시 처리한다. latency가 가장 중요한 단일 요청 시나리오에 적합하다.

**Dynamic Batcher**: 시간 윈도우 내에 도착한 요청들을 하나의 배치로 묶는다. throughput 최적화의 핵심이다. `preferred_batch_size`와 `max_queue_delay_microseconds`로 배치 크기와 최대 대기 시간을 제어한다.

**Sequence Batcher**: 상태를 유지해야 하는 순차 요청(예: 챗봇, 스트리밍 ASR)을 처리한다. 동일 시퀀스의 요청이 동일 모델 인스턴스로 라우팅되도록 보장한다. `DIRECT`(명시적 슬롯 할당)과 `OLDEST`(자동 슬롯 할당) 두 가지 전략을 지원한다.

### 2.3 메모리 관리

Triton은 GPU 메모리를 효율적으로 관리하기 위해 여러 전략을 사용한다. `instance_group`의 `count`를 늘리면 모델 복제본이 GPU 메모리에 추가 로드된다. 멀티 GPU 환경에서는 `gpus` 필드로 특정 GPU에 모델을 배치할 수 있다. `rate_limiter`를 통해 리소스 경합을 제어하고, `model_warmup`으로 첫 요청의 cold-start latency를 제거한다.

---

## 3. 고급 응용 개념

### 3.1 Model Ensemble (파이프라인)

여러 모델을 서버 내부에서 체인으로 연결한다. 클라이언트는 단일 요청만 보내면 내부적으로 전처리 → 추론 → 후처리가 순차 실행된다. 모든 데이터 전달이 GPU 메모리 내에서 이루어지므로 CPU-GPU 간 복사 오버헤드를 최소화한다.

```protobuf
# ensemble_pipeline/config.pbtxt
name: "ensemble_pipeline"
platform: "ensemble"
max_batch_size: 64

input [
  { name: "RAW_TEXT", data_type: TYPE_STRING, dims: [ 1 ] }
]
output [
  { name: "CLASSIFICATION", data_type: TYPE_FP32, dims: [ 3 ] }
]

ensemble_scheduling {
  step [
    {
      model_name: "text_preprocessor"
      model_version: -1    # 최신 버전
      input_map {
        key: "RAW_INPUT"
        value: "RAW_TEXT"
      }
      output_map {
        key: "PROCESSED_IDS"
        value: "preprocessed_ids"
      }
    },
    {
      model_name: "text_classifier"
      model_version: -1
      input_map {
        key: "input_ids"
        value: "preprocessed_ids"
      }
      output_map {
        key: "logits"
        value: "CLASSIFICATION"
      }
    }
  ]
}
```

### 3.2 BLS (Business Logic Scripting) — Python Backend

Ensemble보다 유연한 파이프라인 구성이 필요할 때 Python Backend를 사용한다. 조건 분기, 반복, 외부 API 호출 등 임의의 비즈니스 로직을 추론 파이프라인에 삽입할 수 있다.

```python
# bls_pipeline/1/model.py
import triton_python_backend_utils as pb_utils
import numpy as np
import json

class TritonPythonModel:
    def initialize(self, args):
        self.model_config = json.loads(args['model_config'])
        self.logger = pb_utils.Logger

    def execute(self, requests):
        responses = []
        for request in requests:
            # 1) 입력 추출
            raw_text = pb_utils.get_input_tensor_by_name(
                request, "RAW_TEXT"
            ).as_numpy().astype(str)[0][0]

            # 2) 언어 감지 모델 호출 (Triton 내부 호출)
            lang_input = pb_utils.Tensor(
                "TEXT", np.array([[raw_text]], dtype=object)
            )
            lang_request = pb_utils.InferenceRequest(
                model_name="language_detector",
                requested_output_names=["LANGUAGE"],
                inputs=[lang_input]
            )
            lang_response = lang_request.exec()
            language = pb_utils.get_output_tensor_by_name(
                lang_response, "LANGUAGE"
            ).as_numpy().astype(str)[0][0]

            # 3) 조건 분기: 언어에 따라 다른 모델 호출
            if language == "ko":
                model_name = "korean_classifier"
            elif language == "en":
                model_name = "english_classifier"
            else:
                model_name = "multilingual_classifier"

            self.logger.log(
                f"Routing to {model_name} for language: {language}",
                self.logger.INFO
            )

            # 4) 선택된 분류 모델 호출
            cls_input = pb_utils.Tensor(
                "TEXT_INPUT", np.array([[raw_text]], dtype=object)
            )
            cls_request = pb_utils.InferenceRequest(
                model_name=model_name,
                requested_output_names=["LOGITS"],
                inputs=[cls_input]
            )
            cls_response = cls_request.exec()
            logits = pb_utils.get_output_tensor_by_name(
                cls_response, "LOGITS"
            ).as_numpy()

            # 5) 응답 구성
            output_tensor = pb_utils.Tensor("RESULT", logits)
            responses.append(
                pb_utils.InferenceResponse(output_tensors=[output_tensor])
            )

        return responses

    def finalize(self):
        self.logger.log("BLS pipeline finalized", self.logger.INFO)
```

### 3.3 Rate Limiter

멀티모델 서빙 시 GPU 리소스 경합을 방지한다. 각 모델 인스턴스가 소비하는 리소스를 선언하고, Triton이 전역적으로 스케줄링한다.

```protobuf
# config.pbtxt
rate_limiter {
  resources [
    {
      name: "GPU_MEMORY"
      global: true       # 전역 리소스
      count: 4           # 이 모델이 4 단위 소비
    }
  ]
  priority: 2            # 우선순위 (낮을수록 높음)
}
```

### 3.4 모델 Warmup

첫 추론 요청의 latency spike를 방지하기 위해 서버 시작 시 더미 추론을 실행한다.

```protobuf
model_warmup [
  {
    name: "warmup_sample"
    batch_size: 1
    inputs {
      key: "input_ids"
      value: {
        data_type: TYPE_INT64
        dims: [ 512 ]
        zero_data: true       # 0으로 채운 더미 데이터
      }
    }
  }
]
```

### 3.5 Response Cache

동일한 입력에 대해 캐시된 결과를 반환하여 중복 연산을 제거한다. 결정론적 모델(동일 입력 → 동일 출력)에서만 사용해야 한다.

```protobuf
# config.pbtxt
response_cache { enable: true }
```

서버 시작 시 캐시 크기를 지정한다:
```bash
tritonserver --model-repository=/models \
  --response-cache-byte-size=1073741824   # 1GB 캐시
```

### 3.6 Custom Backend 개발

C++ 기반 고성능 커스텀 백엔드를 개발할 수 있다. Python Backend보다 낮은 latency가 필요하거나, 특수한 하드웨어 가속기를 활용해야 할 때 사용한다.

### 3.7 Metrics & Monitoring

Triton은 Prometheus 호환 메트릭을 포트 8002로 노출한다. GPU utilization, 요청 수, latency 분포, 배칭 통계 등을 실시간으로 모니터링할 수 있다.

주요 메트릭:
- `nv_inference_request_success`: 성공한 추론 요청 수
- `nv_inference_request_failure`: 실패한 추론 요청 수
- `nv_inference_count`: 총 추론 횟수
- `nv_inference_exec_count`: 총 배치 실행 횟수
- `nv_inference_request_duration_us`: 요청 처리 시간 (마이크로초)
- `nv_inference_queue_duration_us`: 큐 대기 시간
- `nv_inference_compute_infer_duration_us`: 순수 추론 시간
- `nv_gpu_utilization`: GPU 사용률
- `nv_gpu_memory_used_bytes`: GPU 메모리 사용량

### 3.8 LLM 서빙 (vLLM Backend + TensorRT-LLM)

대규모 언어 모델 서빙을 위해 Triton은 vLLM Backend과 TensorRT-LLM Backend을 제공한다. PagedAttention 기반의 효율적 KV-Cache 관리, Continuous Batching, Tensor Parallelism 등을 지원한다.

```protobuf
# llm_model/config.pbtxt (TensorRT-LLM backend 예시)
name: "llm_model"
backend: "tensorrtllm"
max_batch_size: 128

model_transaction_policy { decoupled: true }   # 스트리밍 응답

input [
  { name: "text_input",     data_type: TYPE_STRING, dims: [ -1 ] },
  { name: "max_tokens",     data_type: TYPE_INT32,  dims: [ 1 ] },
  { name: "temperature",    data_type: TYPE_FP32,   dims: [ 1 ] },
  { name: "stream",         data_type: TYPE_BOOL,   dims: [ 1 ] }
]

output [
  { name: "text_output",    data_type: TYPE_STRING, dims: [ -1 ] }
]

instance_group [
  { count: 1, kind: KIND_GPU, gpus: [ 0, 1 ] }   # 2-GPU Tensor Parallel
]

parameters {
  key: "max_beam_width"
  value: { string_value: "1" }
}
parameters {
  key: "batching_type"
  value: { string_value: "inflight" }   # Continuous batching
}
parameters {
  key: "kv_cache_free_gpu_mem_fraction"
  value: { string_value: "0.85" }
}
```

### 3.9 Decoupled Models (스트리밍)

`model_transaction_policy: { decoupled: true }`를 설정하면 하나의 요청에 대해 여러 응답을 스트리밍으로 반환할 수 있다. LLM 토큰 스트리밍, 실시간 비디오 프로세싱 등에 사용한다.

---

## 4. Step-by-Step 실습

### Step 1: 환경 구축

#### 1-1. Docker 기반 설치 (권장)

```bash
# NVIDIA Container Toolkit 확인
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# Triton 서버 이미지 풀
docker pull nvcr.io/nvidia/tritonserver:24.08-py3

# Triton 클라이언트 SDK 이미지
docker pull nvcr.io/nvidia/tritonserver:24.08-py3-sdk
```

#### 1-2. 프로젝트 디렉토리 구성

```bash
mkdir -p triton-lab/{model_repository,client,scripts}
cd triton-lab
```

#### 1-3. Python 클라이언트 설치

```bash
pip install tritonclient[all] transformers torch onnx onnxruntime numpy
```

### Step 2: ONNX 모델 준비 (텍스트 분류)

#### 2-1. HuggingFace 모델을 ONNX로 변환

```python
# scripts/export_onnx.py
import torch
from transformers import AutoModelForSequenceClassification, AutoTokenizer

MODEL_NAME = "distilbert-base-uncased-finetuned-sst-2-english"
EXPORT_PATH = "../model_repository/sentiment_model/1/model.onnx"

# 모델 & 토크나이저 로드
tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
model = AutoModelForSequenceClassification.from_pretrained(MODEL_NAME)
model.eval()

# 더미 입력 생성
dummy = tokenizer(
    "This is a sample sentence",
    return_tensors="pt",
    max_length=128,
    padding="max_length",
    truncation=True,
)

# ONNX 익스포트
torch.onnx.export(
    model,
    (dummy["input_ids"], dummy["attention_mask"]),
    EXPORT_PATH,
    input_names=["input_ids", "attention_mask"],
    output_names=["logits"],
    dynamic_axes={
        "input_ids": {0: "batch_size"},
        "attention_mask": {0: "batch_size"},
        "logits": {0: "batch_size"},
    },
    opset_version=17,
)

print(f"Model exported to {EXPORT_PATH}")
print(f"Input shape: input_ids={list(dummy['input_ids'].shape)}, "
      f"attention_mask={list(dummy['attention_mask'].shape)}")
```

#### 2-2. Model Repository 구성

```bash
mkdir -p model_repository/sentiment_model/1
python scripts/export_onnx.py
```

#### 2-3. config.pbtxt 작성

```protobuf
# model_repository/sentiment_model/config.pbtxt
name: "sentiment_model"
platform: "onnxruntime_onnx"
max_batch_size: 64

input [
  {
    name: "input_ids"
    data_type: TYPE_INT64
    dims: [ 128 ]
  },
  {
    name: "attention_mask"
    data_type: TYPE_INT64
    dims: [ 128 ]
  }
]

output [
  {
    name: "logits"
    data_type: TYPE_FP32
    dims: [ 2 ]
  }
]

instance_group [
  {
    count: 2
    kind: KIND_GPU
    gpus: [ 0 ]
  }
]

dynamic_batching {
  preferred_batch_size: [ 4, 8, 16, 32 ]
  max_queue_delay_microseconds: 200
}

# ONNX Runtime 최적화 설정
optimization {
  execution_accelerators {
    gpu_execution_accelerator: [
      {
        name: "tensorrt"
        parameters {
          key: "precision_mode"
          value: "FP16"
        }
        parameters {
          key: "max_workspace_size_bytes"
          value: "1073741824"
        }
      }
    ]
  }
}

model_warmup [
  {
    name: "warmup_batch"
    batch_size: 1
    inputs {
      key: "input_ids"
      value: {
        data_type: TYPE_INT64
        dims: [ 128 ]
        zero_data: true
      }
    }
    inputs {
      key: "attention_mask"
      value: {
        data_type: TYPE_INT64
        dims: [ 128 ]
        zero_data: true
      }
    }
  }
]
```

### Step 3: Triton 서버 실행

```bash
# 기본 실행
docker run --rm --gpus all \
  -p 8000:8000 \
  -p 8001:8001 \
  -p 8002:8002 \
  -v $(pwd)/model_repository:/models \
  nvcr.io/nvidia/tritonserver:24.08-py3 \
  tritonserver \
    --model-repository=/models \
    --log-verbose=1 \
    --strict-model-config=false \
    --response-cache-byte-size=536870912

# 정상 확인
curl -s http://localhost:8000/v2/health/ready
# 응답: {"ready":true}

# 모델 상태 확인
curl -s http://localhost:8000/v2/models/sentiment_model | python -m json.tool
```

### Step 4: 클라이언트 구현

#### 4-1. HTTP 클라이언트

```python
# client/http_client.py
import numpy as np
import tritonclient.http as httpclient
from transformers import AutoTokenizer

TRITON_URL = "localhost:8000"
MODEL_NAME = "sentiment_model"
TOKENIZER_NAME = "distilbert-base-uncased-finetuned-sst-2-english"

def create_client():
    return httpclient.InferenceServerClient(
        url=TRITON_URL,
        verbose=False,
        concurrency=16,
    )

def preprocess(texts: list[str], tokenizer, max_length=128):
    encoded = tokenizer(
        texts,
        max_length=max_length,
        padding="max_length",
        truncation=True,
        return_tensors="np",
    )
    return encoded["input_ids"].astype(np.int64), \
           encoded["attention_mask"].astype(np.int64)

def infer(client, input_ids, attention_mask):
    inputs = [
        httpclient.InferInput("input_ids", input_ids.shape, "INT64"),
        httpclient.InferInput("attention_mask", attention_mask.shape, "INT64"),
    ]
    inputs[0].set_data_from_numpy(input_ids)
    inputs[1].set_data_from_numpy(attention_mask)

    outputs = [httpclient.InferRequestedOutput("logits")]

    response = client.infer(
        model_name=MODEL_NAME,
        inputs=inputs,
        outputs=outputs,
        request_id="req-001",
        headers={"X-Request-ID": "req-001"},
    )

    logits = response.as_numpy("logits")
    return logits

def postprocess(logits):
    # Softmax
    exp_logits = np.exp(logits - np.max(logits, axis=1, keepdims=True))
    probs = exp_logits / np.sum(exp_logits, axis=1, keepdims=True)

    labels = ["NEGATIVE", "POSITIVE"]
    results = []
    for prob in probs:
        idx = np.argmax(prob)
        results.append({
            "label": labels[idx],
            "confidence": float(prob[idx]),
            "scores": {l: float(p) for l, p in zip(labels, prob)},
        })
    return results

if __name__ == "__main__":
    tokenizer = AutoTokenizer.from_pretrained(TOKENIZER_NAME)
    client = create_client()

    # 서버 상태 확인
    assert client.is_server_ready(), "Triton server is not ready"
    assert client.is_model_ready(MODEL_NAME), f"{MODEL_NAME} is not ready"

    # 추론
    texts = [
        "This movie was absolutely fantastic!",
        "Terrible experience, would not recommend.",
        "It was okay, nothing special.",
        "I loved every minute of this film!",
    ]

    input_ids, attention_mask = preprocess(texts, tokenizer)
    logits = infer(client, input_ids, attention_mask)
    results = postprocess(logits)

    for text, result in zip(texts, results):
        print(f"Text: {text}")
        print(f"  → {result['label']} (confidence: {result['confidence']:.4f})")
        print()

    # 모델 메타데이터 조회
    metadata = client.get_model_metadata(MODEL_NAME)
    print(f"Model: {metadata['name']}, Versions: {metadata['versions']}")

    # 추론 통계 조회
    stats = client.get_inference_statistics(MODEL_NAME)
    print(f"Inference count: {stats['model_stats'][0]['inference_count']}")
```

#### 4-2. gRPC 클라이언트 (프로덕션 권장)

```python
# client/grpc_client.py
import numpy as np
import tritonclient.grpc as grpcclient
from tritonclient.utils import InferenceServerException
from transformers import AutoTokenizer
import time

TRITON_URL = "localhost:8001"
MODEL_NAME = "sentiment_model"

def create_grpc_client():
    return grpcclient.InferenceServerClient(
        url=TRITON_URL,
        verbose=False,
        ssl=False,       # 프로덕션에서는 True + 인증서
    )

def infer_grpc(client, input_ids, attention_mask):
    inputs = [
        grpcclient.InferInput("input_ids", input_ids.shape, "INT64"),
        grpcclient.InferInput("attention_mask", attention_mask.shape, "INT64"),
    ]
    inputs[0].set_data_from_numpy(input_ids)
    inputs[1].set_data_from_numpy(attention_mask)

    outputs = [
        grpcclient.InferRequestedOutput("logits"),
    ]

    response = client.infer(
        model_name=MODEL_NAME,
        inputs=inputs,
        outputs=outputs,
        client_timeout=5.0,      # 5초 타임아웃
        compression_algorithm="gzip",  # 네트워크 압축
    )

    return response.as_numpy("logits")

def infer_grpc_async(client, input_ids, attention_mask, callback):
    """비동기 추론 — 고처리량 시나리오"""
    inputs = [
        grpcclient.InferInput("input_ids", input_ids.shape, "INT64"),
        grpcclient.InferInput("attention_mask", attention_mask.shape, "INT64"),
    ]
    inputs[0].set_data_from_numpy(input_ids)
    inputs[1].set_data_from_numpy(attention_mask)

    outputs = [grpcclient.InferRequestedOutput("logits")]

    client.async_infer(
        model_name=MODEL_NAME,
        inputs=inputs,
        outputs=outputs,
        callback=callback,
    )

if __name__ == "__main__":
    tokenizer = AutoTokenizer.from_pretrained(
        "distilbert-base-uncased-finetuned-sst-2-english"
    )
    client = create_grpc_client()

    texts = ["Great product!", "Waste of money."]
    encoded = tokenizer(
        texts, max_length=128, padding="max_length",
        truncation=True, return_tensors="np"
    )
    input_ids = encoded["input_ids"].astype(np.int64)
    attn_mask = encoded["attention_mask"].astype(np.int64)

    # --- 동기 추론 ---
    start = time.perf_counter()
    logits = infer_grpc(client, input_ids, attn_mask)
    elapsed = (time.perf_counter() - start) * 1000
    print(f"gRPC sync inference: {elapsed:.2f}ms")

    # --- 비동기 추론 ---
    results_async = []
    def _callback(result, error):
        if error:
            print(f"Async error: {error}")
        else:
            results_async.append(result.as_numpy("logits"))

    for _ in range(10):
        infer_grpc_async(client, input_ids, attn_mask, _callback)

    # 모든 비동기 요청 완료 대기
    time.sleep(2)
    print(f"Async results received: {len(results_async)}")
```

### Step 5: Python Backend 전처리 모델

토크나이저를 서버 사이드에 두어 클라이언트 의존성을 제거한다.

#### 5-1. 전처리 모델 코드

```python
# model_repository/tokenizer_model/1/model.py
import triton_python_backend_utils as pb_utils
import numpy as np
from transformers import AutoTokenizer
import json

class TritonPythonModel:
    def initialize(self, args):
        self.model_config = json.loads(args["model_config"])
        self.tokenizer = AutoTokenizer.from_pretrained(
            "distilbert-base-uncased-finetuned-sst-2-english"
        )
        self.max_length = 128

    def execute(self, requests):
        responses = []
        for request in requests:
            raw_texts = pb_utils.get_input_tensor_by_name(
                request, "RAW_TEXT"
            ).as_numpy()

            # bytes → str 디코딩
            texts = [t[0].decode("utf-8") if isinstance(t[0], bytes) else str(t[0])
                     for t in raw_texts]

            encoded = self.tokenizer(
                texts,
                max_length=self.max_length,
                padding="max_length",
                truncation=True,
                return_tensors="np",
            )

            input_ids = pb_utils.Tensor(
                "INPUT_IDS", encoded["input_ids"].astype(np.int64)
            )
            attention_mask = pb_utils.Tensor(
                "ATTENTION_MASK", encoded["attention_mask"].astype(np.int64)
            )

            response = pb_utils.InferenceResponse(
                output_tensors=[input_ids, attention_mask]
            )
            responses.append(response)

        return responses

    def finalize(self):
        pass
```

#### 5-2. 전처리 모델 config.pbtxt

```protobuf
# model_repository/tokenizer_model/config.pbtxt
name: "tokenizer_model"
backend: "python"
max_batch_size: 64

input [
  {
    name: "RAW_TEXT"
    data_type: TYPE_STRING
    dims: [ 1 ]
  }
]

output [
  {
    name: "INPUT_IDS"
    data_type: TYPE_INT64
    dims: [ 128 ]
  },
  {
    name: "ATTENTION_MASK"
    data_type: TYPE_INT64
    dims: [ 128 ]
  }
]

instance_group [
  { count: 2, kind: KIND_CPU }
]

dynamic_batching {
  preferred_batch_size: [ 4, 8, 16 ]
  max_queue_delay_microseconds: 100
}
```

### Step 6: Ensemble 파이프라인 구성

#### 6-1. Ensemble config.pbtxt

```protobuf
# model_repository/sentiment_pipeline/config.pbtxt
name: "sentiment_pipeline"
platform: "ensemble"
max_batch_size: 64

input [
  {
    name: "RAW_TEXT"
    data_type: TYPE_STRING
    dims: [ 1 ]
  }
]

output [
  {
    name: "LOGITS"
    data_type: TYPE_FP32
    dims: [ 2 ]
  }
]

ensemble_scheduling {
  step [
    {
      model_name: "tokenizer_model"
      model_version: -1
      input_map {
        key: "RAW_TEXT"
        value: "RAW_TEXT"
      }
      output_map {
        key: "INPUT_IDS"
        value: "tokenized_ids"
      }
      output_map {
        key: "ATTENTION_MASK"
        value: "tokenized_mask"
      }
    },
    {
      model_name: "sentiment_model"
      model_version: -1
      input_map {
        key: "input_ids"
        value: "tokenized_ids"
      }
      input_map {
        key: "attention_mask"
        value: "tokenized_mask"
      }
      output_map {
        key: "logits"
        value: "LOGITS"
      }
    }
  ]
}
```

#### 6-2. Ensemble 클라이언트

```python
# client/ensemble_client.py
import numpy as np
import tritonclient.http as httpclient

def infer_pipeline(texts: list[str]):
    client = httpclient.InferenceServerClient(url="localhost:8000")

    # 문자열 입력 준비
    text_array = np.array(
        [[t.encode("utf-8")] for t in texts], dtype=object
    )

    inputs = [
        httpclient.InferInput("RAW_TEXT", text_array.shape, "BYTES"),
    ]
    inputs[0].set_data_from_numpy(text_array)

    outputs = [httpclient.InferRequestedOutput("LOGITS")]

    response = client.infer(
        model_name="sentiment_pipeline",
        inputs=inputs,
        outputs=outputs,
    )

    logits = response.as_numpy("LOGITS")
    exp_l = np.exp(logits - np.max(logits, axis=1, keepdims=True))
    probs = exp_l / exp_l.sum(axis=1, keepdims=True)

    labels = ["NEGATIVE", "POSITIVE"]
    for text, prob in zip(texts, probs):
        idx = np.argmax(prob)
        print(f"{text}")
        print(f"  → {labels[idx]}: {prob[idx]:.4f}")

if __name__ == "__main__":
    infer_pipeline([
        "Absolutely wonderful experience!",
        "Never buying this again.",
        "The quality is decent for the price.",
    ])
```

### Step 7: 성능 분석 (Perf Analyzer)

Triton에 내장된 `perf_analyzer`로 모델 서빙 성능을 측정한다.

```bash
# SDK 컨테이너에서 실행
docker run --rm --net=host \
  nvcr.io/nvidia/tritonserver:24.08-py3-sdk \
  perf_analyzer \
    -m sentiment_model \
    -u localhost:8001 \
    -i grpc \
    --input-data zero \
    --shape input_ids:1,128 \
    --shape attention_mask:1,128 \
    -b 1 \
    --concurrency-range 1:32:4 \
    --measurement-interval 10000 \
    --percentile-metrics p50,p90,p99 \
    -f perf_results.csv

# 결과 분석: Concurrency별 Throughput / Latency
# concurrency=1  → baseline latency 확인
# concurrency=8  → dynamic batching 효과 확인
# concurrency=32 → 최대 throughput 확인
```

#### 커스텀 입력 데이터로 측정

```bash
# 실제 데이터 기반 벤치마크
# 1) 입력 데이터 JSON 생성
cat > real_data.json << 'EOF'
{
  "data": [
    {
      "input_ids": { "content": [101, 2023, 2003, ...], "shape": [128] },
      "attention_mask": { "content": [1, 1, 1, ...], "shape": [128] }
    }
  ]
}
EOF

# 2) 실제 데이터로 벤치마크
perf_analyzer \
  -m sentiment_model \
  -u localhost:8001 \
  -i grpc \
  --input-data real_data.json \
  --concurrency-range 1:64:8 \
  -f real_perf.csv
```

### Step 8: Model Analyzer (최적 설정 자동 탐색)

```bash
# Model Analyzer 설치
pip install triton-model-analyzer

# 분석 설정 파일
cat > analyzer_config.yaml << 'EOF'
model_repository: /path/to/model_repository

profile_models:
  sentiment_model:
    parameters:
      batch_sizes: [1, 4, 8, 16, 32, 64]
      concurrency: [1, 2, 4, 8, 16, 32]

    model_config_parameters:
      instance_group:
        - kind: KIND_GPU
          count: [1, 2, 4]

      dynamic_batching:
        preferred_batch_size: [[4,8], [8,16], [16,32]]
        max_queue_delay_microseconds: [100, 500, 1000]

    objectives:
      perf_throughput: 10    # throughput 가중치
      perf_latency_p99: 5    # p99 latency 가중치
      gpu_used_memory: 1     # GPU 메모리 가중치

    constraints:
      perf_latency_p99:
        max: 50              # p99 latency ≤ 50ms
      gpu_used_memory:
        max: 8589934592      # GPU 메모리 ≤ 8GB
EOF

# 실행
model-analyzer profile \
  --model-repository /path/to/model_repository \
  --config-file analyzer_config.yaml \
  --output-model-repository-path optimized_models \
  --export-path results

# 결과 리포트
model-analyzer report \
  --report-model-configs sentiment_model \
  --export-path results
```

### Step 9: Kubernetes 배포 (Helm Chart)

#### 9-1. Helm 차트 설치

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# values.yaml 작성
cat > triton-values.yaml << 'EOF'
image:
  repository: nvcr.io/nvidia/tritonserver
  tag: "24.08-py3"
  pullPolicy: IfNotPresent

replicaCount: 2

resources:
  limits:
    nvidia.com/gpu: 1
    memory: "16Gi"
    cpu: "8"
  requests:
    nvidia.com/gpu: 1
    memory: "8Gi"
    cpu: "4"

modelRepository:
  # S3 모델 저장소
  storageType: "s3"
  s3:
    bucketName: "my-triton-models"
    region: "ap-northeast-2"
    # IAM Role 사용 시 credentials 생략 가능

service:
  type: ClusterIP
  httpPort: 8000
  grpcPort: 8001
  metricsPort: 8002

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: nv_inference_queue_duration_us
        target:
          type: AverageValue
          averageValue: "1000"    # 큐 대기 1ms 초과 시 스케일 아웃

ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/grpc-backend: "true"
  hosts:
    - host: triton.internal.example.com
      paths:
        - path: /
          pathType: Prefix

startupProbe:
  httpGet:
    path: /v2/health/ready
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 30

livenessProbe:
  httpGet:
    path: /v2/health/live
    port: 8000
  periodSeconds: 15
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /v2/health/ready
    port: 8000
  periodSeconds: 10
  timeoutSeconds: 5

extraArgs:
  - "--model-control-mode=poll"
  - "--repository-poll-secs=30"
  - "--response-cache-byte-size=1073741824"
  - "--log-verbose=0"

# Prometheus ServiceMonitor
metrics:
  serviceMonitor:
    enabled: true
    interval: 15s
    labels:
      release: prometheus
EOF

# 배포
helm install triton nvidia/tritonserver \
  -f triton-values.yaml \
  -n ml-serving \
  --create-namespace
```

#### 9-2. Prometheus + Grafana 모니터링

```yaml
# grafana-dashboard-configmap.yaml (핵심 패널 예시)
apiVersion: v1
kind: ConfigMap
metadata:
  name: triton-grafana-dashboard
  labels:
    grafana_dashboard: "1"
data:
  triton.json: |
    {
      "panels": [
        {
          "title": "Inference Throughput (req/s)",
          "targets": [{
            "expr": "sum(rate(nv_inference_request_success{model=~\"$model\"}[1m]))"
          }]
        },
        {
          "title": "P99 Latency (ms)",
          "targets": [{
            "expr": "histogram_quantile(0.99, rate(nv_inference_request_duration_us_bucket{model=~\"$model\"}[5m])) / 1000"
          }]
        },
        {
          "title": "GPU Utilization (%)",
          "targets": [{
            "expr": "nv_gpu_utilization * 100"
          }]
        },
        {
          "title": "Batch Size Distribution",
          "targets": [{
            "expr": "rate(nv_inference_count{model=~\"$model\"}[1m]) / rate(nv_inference_exec_count{model=~\"$model\"}[1m])"
          }]
        },
        {
          "title": "Queue Wait Time (ms)",
          "targets": [{
            "expr": "rate(nv_inference_queue_duration_us{model=~\"$model\"}[1m]) / rate(nv_inference_request_success{model=~\"$model\"}[1m]) / 1000"
          }]
        }
      ]
    }
```

### Step 10: CI/CD 파이프라인 — 모델 배포 자동화

```yaml
# .github/workflows/model-deploy.yaml
name: Model Deploy to Triton

on:
  push:
    paths:
      - 'models/**'
    branches: [main]

env:
  S3_BUCKET: my-triton-models
  AWS_REGION: ap-northeast-2

jobs:
  validate-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install tritonclient[all] onnx onnxruntime numpy

      - name: Validate model configs
        run: |
          python scripts/validate_configs.py \
            --model-repo models/

      - name: Validate ONNX models
        run: |
          python scripts/validate_onnx.py \
            --model-repo models/

      - name: Run integration tests (local Triton)
        run: |
          docker run -d --name triton-test \
            -v $(pwd)/models:/models \
            nvcr.io/nvidia/tritonserver:24.08-py3 \
            tritonserver --model-repository=/models

          # 헬스 체크 대기
          for i in $(seq 1 30); do
            curl -sf http://localhost:8000/v2/health/ready && break
            sleep 2
          done

          python scripts/integration_test.py
          docker stop triton-test

      - name: Deploy to S3 model repository
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - run: |
          aws s3 sync models/ s3://${{ env.S3_BUCKET }}/ \
            --delete --exclude "*.pyc"

      - name: Trigger Triton model reload
        run: |
          # poll 모드일 경우 자동 감지됨
          # explicit 모드일 경우 API 호출
          curl -X POST \
            "http://triton.internal.example.com/v2/repository/index" \
            -H "Content-Type: application/json"
```

```python
# scripts/validate_configs.py
import sys, os, re
from pathlib import Path

def validate_model_repo(repo_path: str):
    errors = []
    repo = Path(repo_path)

    for model_dir in repo.iterdir():
        if not model_dir.is_dir():
            continue

        config_path = model_dir / "config.pbtxt"
        if not config_path.exists():
            errors.append(f"{model_dir.name}: config.pbtxt missing")
            continue

        config_text = config_path.read_text()

        # 이름 일치 검증
        match = re.search(r'name:\s*"([^"]+)"', config_text)
        if match and match.group(1) != model_dir.name:
            errors.append(
                f"{model_dir.name}: name mismatch "
                f"(config={match.group(1)}, dir={model_dir.name})"
            )

        # 버전 디렉토리 존재 확인
        versions = [
            d for d in model_dir.iterdir()
            if d.is_dir() and d.name.isdigit()
        ]
        if not versions and "ensemble" not in config_text:
            errors.append(f"{model_dir.name}: no version directories found")

        # 필수 필드 확인
        for field in ["input", "output"]:
            if field not in config_text:
                errors.append(f"{model_dir.name}: '{field}' section missing")

    if errors:
        print("Validation FAILED:")
        for e in errors:
            print(f"  ✗ {e}")
        sys.exit(1)
    else:
        print("All model configs validated successfully")

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--model-repo", required=True)
    args = parser.parse_args()
    validate_model_repo(args.model_repo)
```

### Step 11: A/B 테스트 & Canary 배포

```protobuf
# 버전 정책으로 A/B 테스트
# sentiment_model/config.pbtxt

# 버전 1(기존)과 버전 2(신규)를 동시에 로드
version_policy: {
  specific { versions: [ 1, 2 ] }
}
```

```python
# client/ab_test_client.py
import random
import tritonclient.http as httpclient

def infer_with_ab_test(client, inputs, outputs, traffic_split=0.1):
    """
    traffic_split=0.1 → 10%가 v2(canary), 90%가 v1(stable)
    """
    version = "2" if random.random() < traffic_split else "1"

    response = client.infer(
        model_name="sentiment_model",
        model_version=version,
        inputs=inputs,
        outputs=outputs,
    )

    return response, version
```

### 최종 디렉토리 구조

```
triton-lab/
├── model_repository/
│   ├── sentiment_model/
│   │   ├── config.pbtxt
│   │   ├── 1/
│   │   │   └── model.onnx
│   │   └── 2/
│   │       └── model.onnx
│   ├── tokenizer_model/
│   │   ├── config.pbtxt
│   │   └── 1/
│   │       └── model.py
│   └── sentiment_pipeline/
│       ├── config.pbtxt
│       └── 1/
│           (empty — ensemble)
├── client/
│   ├── http_client.py
│   ├── grpc_client.py
│   ├── ensemble_client.py
│   └── ab_test_client.py
├── scripts/
│   ├── export_onnx.py
│   ├── validate_configs.py
│   ├── validate_onnx.py
│   └── integration_test.py
├── triton-values.yaml
├── analyzer_config.yaml
└── .github/
    └── workflows/
        └── model-deploy.yaml
```

---

## 빠른 참조: 자주 사용하는 서버 옵션

```bash
tritonserver \
  --model-repository=/models \
  # 모델 로드 제어
  --model-control-mode=poll \          # none | poll | explicit
  --repository-poll-secs=30 \          # poll 모드 간격
  --load-model=sentiment_model \       # explicit 모드: 특정 모델만 로드
  --strict-model-config=false \        # config 자동 생성 허용
  # 성능 관련
  --response-cache-byte-size=1G \      # 응답 캐시 크기
  --pinned-memory-pool-byte-size=256M \# CPU 고정 메모리 풀
  --cuda-memory-pool-byte-size=0:1G \  # GPU 0의 CUDA 메모리 풀
  --min-supported-compute-capability=7.5 \ # Turing 이상
  --backend-config=onnxruntime,default-max-batch-size=64 \
  # 로깅 & 모니터링
  --log-verbose=1 \                    # 0=off, 1~3=verbose
  --metrics-port=8002 \
  --allow-metrics=true \
  --allow-gpu-metrics=true \
  # 보안
  --grpc-use-ssl=true \
  --grpc-server-cert=/certs/server.crt \
  --grpc-server-key=/certs/server.key \
  --grpc-root-cert=/certs/ca.crt
```
