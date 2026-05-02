# Kubeflow 완벽 강의 — 기본 개념부터 NVIDIA DGX Spark 실습까지

> **대상 독자**: MLOps / LLMOps / Cloud Engineer 직군에서 Kubeflow를 처음 도입하거나, DGX Spark 같은 ARM64 기반 AI 워크스테이션 위에 ML 플랫폼을 구축하려는 엔지니어
> **선수 지식**: Linux 기본 명령어, Docker 컨테이너 개념, Kubernetes 기본 용어(Pod, Deployment, Service) 정도면 충분합니다.
> **최종 업데이트 기준**: 2026년 5월 시점의 Kubeflow 1.10/1.11, DGX OS 7.x, Kubernetes 1.31~1.33

---

## 목차

1. [Kubeflow가 등장한 배경 — "왜 또 새로운 도구가 필요한가?"](#1부-kubeflow가-등장한-배경)
2. [Kubeflow의 핵심 개념과 아키텍처](#2부-kubeflow의-핵심-개념과-아키텍처)
3. [Kubeflow 핵심 컴포넌트 자세히 보기](#3부-kubeflow-핵심-컴포넌트-자세히-보기)
4. [실전 워크플로 — 코드와 함께 보는 사용법](#4부-실전-워크플로--코드와-함께-보는-사용법)
5. [NVIDIA DGX Spark 이해하기](#5부-nvidia-dgx-spark-이해하기)
6. [DGX Spark에서 Kubeflow 설치 실습](#6부-dgx-spark에서-kubeflow-설치-실습)
7. [LLM Fine-tuning 파이프라인 만들기](#7부-llm-fine-tuning-파이프라인-만들기)
8. [트러블슈팅과 운영 팁](#8부-트러블슈팅과-운영-팁)
9. [다음 단계와 학습 리소스](#9부-다음-단계와-학습-리소스)

---

## 1부. Kubeflow가 등장한 배경

### 1.1 머신러닝 엔지니어링의 고질적 문제

ML 모델을 처음 만들 때는 보통 노트북 한 대로 시작합니다. 데이터 불러오고, 전처리하고, `model.fit()` 한 줄이면 끝나죠. 그런데 이 모델을 실제 서비스에 올리려고 하면 갑자기 다음과 같은 일들이 생깁니다.

- 데이터셋이 너무 커서 노트북 메모리에 안 올라간다
- 학습 시간이 너무 길어서 GPU 8장으로 분산 학습해야 한다
- 어떤 하이퍼파라미터 조합이 가장 좋은지 100번씩 돌려봐야 한다
- 학습한 모델을 REST API로 띄우고, 트래픽에 따라 오토스케일을 해야 한다
- 매주 새 데이터로 자동 재학습을 돌려야 한다
- 누가 어떤 데이터로 어떤 코드로 학습한 모델인지 추적해야 한다

이걸 다 따로따로 도구를 골라서 엮으면, 곧 *"우리 회사만의 누더기 ML 플랫폼"* 이 탄생합니다. 그리고 그 누더기를 유지하는 사람이 퇴사하면 모두가 곤란해지죠.

### 1.2 그래서 Kubeflow

Kubeflow는 이 문제를 **"Kubernetes 위에서 ML 라이프사이클 전체를 표준화된 컴포넌트로 묶어 제공하자"** 라는 발상으로 해결합니다. 2018년 KubeCon에서 0.1 버전이 처음 공개되었고, 2020년 1.0이 나오면서 프로덕션급으로 자리잡았으며, 2023년에는 CNCF의 인큐베이팅 프로젝트로 받아들여졌습니다.

Kubeflow의 한 줄 정의는 이렇습니다:

> **"Kubernetes 위에서 동작하는, ML 워크플로 전 단계를 위한 오픈소스 플랫폼"**

핵심 가치는 세 가지입니다.

| 가치 | 의미 |
|---|---|
| **Composability (조립 가능성)** | 노트북, 파이프라인, 서빙 등을 필요한 것만 골라 쓸 수 있다 |
| **Portability (이식성)** | 노트북에서 짠 코드 그대로 GKE, EKS, 온프레, DGX 어디든 돌아간다 |
| **Scalability (확장성)** | Kubernetes의 스케일링 능력을 그대로 활용한다 |

### 1.3 Kubeflow vs MLflow — 자주 받는 질문

> "MLflow도 비슷하다는데 뭐가 다른가요?"

가장 짧은 답:

- **MLflow** — 가벼운 실험 추적/모델 레지스트리. 단일 머신에서 시작하기 쉬움. *"파이썬 스크립트의 친구"*
- **Kubeflow** — 풀스택 ML 플랫폼. 분산 학습/서빙/오케스트레이션을 Kubernetes 네이티브로 처리. *"Kubernetes의 자식"*

작은 팀이면 MLflow로 충분하지만, 멀티테넌시·분산 학습·자동 오토스케일링이 필요한 조직 규모에서는 Kubeflow의 장점이 두드러집니다.

---

## 2부. Kubeflow의 핵심 개념과 아키텍처

### 2.1 큰 그림 한 장으로

```
┌─────────────────────────────────────────────────────────────┐
│                    Central Dashboard (UI)                   │
│   하나의 웹 UI에서 모든 컴포넌트 진입 — Profile/Namespace 단위 │
└─────────────────────────────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
   ┌─────────┐         ┌──────────┐         ┌──────────┐
   │Notebooks│         │ Pipelines│         │  Katib   │
   │(Jupyter,│         │  (KFP)   │         │  (HPO)   │
   │ VSCode, │         │          │         │          │
   │ RStudio)│         │          │         │          │
   └─────────┘         └──────────┘         └──────────┘
        │                    │                    │
        ▼                    ▼                    ▼
   ┌──────────────────────────────────────────────────┐
   │              Trainer / Training Operator          │
   │   (PyTorchJob, TFJob, MPIJob ... 분산 학습 CRD)   │
   └──────────────────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
   ┌──────────┐        ┌──────────┐         ┌──────────┐
   │ KServe   │        │  Model   │         │ Spark    │
   │(Serving) │        │ Registry │         │ Operator │
   └──────────┘        └──────────┘         └──────────┘
                             │
   ═══════════════════ Kubernetes ═══════════════════
                  (GPU Operator, Istio, Knative)
```

### 2.2 Kubeflow에서의 "Profile"이라는 개념

Kubeflow는 사용자별 작업공간을 **Profile** 이라는 CR(Custom Resource)로 관리합니다. 사용자가 로그인하면 자기 이름의 Kubernetes 네임스페이스가 자동 생성되고, 그 안에서 노트북·파이프라인·모델이 격리되어 동작합니다. 이 덕분에 한 클러스터에서 여러 팀이 안전하게 같이 쓸 수 있습니다(멀티테넌시).

### 2.3 의존하는 인프라 컴포넌트

Kubeflow는 단독으로 동작하지 않고 다음 도구들 위에 올라갑니다.

| 컴포넌트 | 역할 |
|---|---|
| **Istio** | 서비스 메시 — 인증/라우팅/mTLS |
| **Dex + OAuth2-Proxy** | 사용자 인증 (Identity Provider) |
| **Knative Serving** | KServe의 기반 — 0-스케일링과 트래픽 분할 |
| **cert-manager** | 인증서 자동 발급/갱신 |
| **Argo Workflows** | KFP의 기반 워크플로 엔진 |
| **MinIO / SeaweedFS** | 파이프라인 아티팩트 저장소 (S3 호환) |

Kubeflow `manifests` 레포에서 한번에 같이 설치되니, 처음에는 "이런 게 같이 깔린다" 정도만 알아두시면 됩니다.

### 2.4 현재 안정 버전 (2026년 5월 기준)

Kubeflow Platform 최신 매니페스트는 KFP 2.15.0, Trainer 2.1.0, Training Operator 1.9.2, Katib 0.19.0, KServe 0.15.2, Spark Operator 2.4.0, Model Registry 0.3.4, Istio 1.28.0 등을 포함하며, Kubernetes 1.31~1.33+를 지원 목표로 합니다. Kubeflow 커뮤니티는 KubeCon EU/NA 시기에 맞춰 연 2회 정기 릴리스를 진행합니다.

---

## 3부. Kubeflow 핵심 컴포넌트 자세히 보기

### 3.1 Notebooks — "주피터를 클러스터에서"

가장 친숙한 입구입니다. 웹 UI에서 "Create Notebook" 버튼을 누르면 Kubernetes 위에 Jupyter/VSCode/RStudio Pod이 뜨고, 거기에 GPU·CPU·메모리·PVC를 붙일 수 있습니다.

**핵심 가치**: 노트북 사용자가 GPU 노드에 SSH 안 해도 되고, 노트북이 죽어도 PVC에 코드는 남아있습니다.

```yaml
# Notebook CR 예시 (Kubeflow가 자동 생성)
apiVersion: kubeflow.org/v1
kind: Notebook
metadata:
  name: my-notebook
  namespace: kubeflow-user-example-com
spec:
  template:
    spec:
      containers:
      - name: my-notebook
        image: kubeflownotebookswg/jupyter-pytorch-cuda:latest
        resources:
          limits:
            nvidia.com/gpu: 1
            memory: 16Gi
            cpu: "4"
```

### 3.2 Kubeflow Pipelines (KFP) — "ML 워크플로의 심장"

Kubeflow에서 가장 많이 쓰이는 기능입니다. 데이터 준비 → 학습 → 평가 → 배포로 이어지는 단계별 작업을 **DAG(방향성 비순환 그래프)**로 정의하고, 각 단계를 컨테이너로 실행합니다.

핵심 개념:

- **Component**: 컨테이너 한 개로 실행되는 작업 단위 (함수처럼 입출력 있음)
- **Pipeline**: Component들을 엮은 DAG
- **Run**: Pipeline의 실제 실행 인스턴스
- **Experiment**: Run들의 그룹 (예: "v1-튜닝 실험들")
- **Artifact**: Component가 만든 파일/모델/메트릭 등의 산출물

KFP SDK v2를 쓰면 데코레이터 한두 개로 Python 함수를 컴포넌트화할 수 있습니다(자세한 코드는 4부에서).

### 3.3 Katib — "하이퍼파라미터의 자동 최적화"

`learning_rate`, `batch_size`, `dropout` 같은 하이퍼파라미터의 최적 조합을 자동으로 찾아주는 컴포넌트입니다. 지원 알고리즘:

- Random Search
- Grid Search
- Bayesian Optimization
- Hyperband, ASHA
- Population Based Training (PBT)
- TPE, CMA-ES, Sobol

**실험(Experiment)**을 정의하면 Katib가 알아서 여러 개의 **Trial(=학습 잡)**을 띄워가며 목표 메트릭(예: validation accuracy)을 최대화/최소화합니다.

### 3.4 Trainer / Training Operator — "분산 학습 추상화"

PyTorch DDP, TensorFlow MultiWorker, MPI 등 분산 학습 프레임워크를 Kubernetes CRD로 추상화합니다. 가장 자주 쓰이는 것은 `PyTorchJob`입니다.

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pytorch-distributed-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          containers:
          - name: pytorch
            image: my-training-image:latest
            resources:
              limits:
                nvidia.com/gpu: 1
    Worker:
      replicas: 3
      template:
        spec:
          containers:
          - name: pytorch
            image: my-training-image:latest
            resources:
              limits:
                nvidia.com/gpu: 1
```

위 매니페스트만 적용하면 마스터 1대 + 워커 3대의 분산 PyTorch 학습이 자동으로 셋업됩니다. `RANK`, `WORLD_SIZE`, `MASTER_ADDR` 같은 환경변수를 컨트롤러가 알아서 주입합니다.

> 2025년 이후 Kubeflow는 차세대 **Trainer V2** API로 전환 중입니다. 기존 Training Operator는 호환을 위해 함께 제공됩니다.

### 3.5 KServe — "모델 서빙의 표준"

학습 끝난 모델을 REST/gRPC로 서비스하는 컴포넌트. 과거 KFServing이 분리되어 KServe가 되었고, TensorFlow, XGBoost, scikit-learn, PyTorch, ONNX 등 여러 프레임워크를 위한 Kubernetes 커스텀 리소스를 제공하며 Google, IBM, Bloomberg, NVIDIA, Seldon이 함께 개발에 참여했습니다.

핵심 기능:

- **Serverless 추론**: Knative 기반으로 트래픽이 없으면 0대까지 줄임
- **Canary 배포**: v1 → v2 전환 시 트래픽 비율 조절
- **Inference Graph**: 여러 모델을 파이프라인으로 묶기
- **LLMInferenceService**: 최신 버전(v0.16+)에서 LLM 전용 CRD 추가
- **Transformer**: 전/후처리를 별도 컨테이너로 분리

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: my-llm
spec:
  predictor:
    model:
      modelFormat:
        name: huggingface
      args: ["--model_name=Qwen/Qwen3-4B"]
      resources:
        limits:
          nvidia.com/gpu: "1"
```

### 3.6 Model Registry — "모델 버전과 메타데이터의 단일 진실원"

상대적으로 새로운 컴포넌트로, 모델의 버전·계보·태그·메타데이터를 한 곳에서 관리합니다. 학습 → 등록 → 스테이징 → 프로덕션의 라이프사이클 흐름을 표준화합니다. ML Metadata(MLMD) 기반.

### 3.7 Spark Operator — "데이터 전처리는 Spark로"

Apache Spark 잡을 Kubernetes 네이티브 CRD(`SparkApplication`)로 실행할 수 있게 해줍니다. 대용량 데이터 전처리/피처 엔지니어링이 필요한 ML 파이프라인에서 자주 함께 사용됩니다.

---

## 4부. 실전 워크플로 — 코드와 함께 보는 사용법

이론은 충분히 봤으니 실제 코드를 보겠습니다. 가장 자주 쓰는 KFP 파이프라인 작성 패턴입니다.

### 4.1 KFP SDK 설치

```bash
pip install "kfp>=2.7.0"
```

### 4.2 가장 간단한 파이프라인

```python
# pipeline.py
from kfp import dsl, compiler

@dsl.component(base_image="python:3.11")
def add(a: int, b: int) -> int:
    return a + b

@dsl.component(base_image="python:3.11")
def multiply(a: int, b: int) -> int:
    return a * b

@dsl.pipeline(name="hello-pipeline")
def my_pipeline(x: int = 3, y: int = 5):
    sum_task = add(a=x, b=y)
    mul_task = multiply(a=sum_task.output, b=2)

if __name__ == "__main__":
    compiler.Compiler().compile(my_pipeline, "pipeline.yaml")
```

`python pipeline.py`를 돌리면 `pipeline.yaml`이 생성되고, 이걸 Kubeflow UI에 업로드하거나 SDK로 직접 제출할 수 있습니다.

```python
from kfp.client import Client

client = Client(host="http://<kubeflow-host>")
client.create_run_from_pipeline_func(
    my_pipeline,
    arguments={"x": 10, "y": 20},
    experiment_name="my-experiment",
)
```

### 4.3 ML 다운스트림 패턴 — 학습 컴포넌트

```python
from kfp import dsl
from kfp.dsl import Input, Output, Dataset, Model

@dsl.component(
    base_image="pytorch/pytorch:2.4.0-cuda12.1-cudnn9-runtime",
    packages_to_install=["scikit-learn", "pandas"],
)
def train_model(
    train_data: Input[Dataset],
    model_output: Output[Model],
    learning_rate: float = 0.001,
    epochs: int = 10,
):
    import torch
    import pandas as pd
    # ... 학습 로직 ...
    df = pd.read_csv(train_data.path)
    # 학습 후
    torch.save(state_dict, model_output.path)
    model_output.metadata["accuracy"] = float(accuracy)
    model_output.metadata["framework"] = "pytorch"
```

`Input[Dataset]`, `Output[Model]` 타입을 쓰면 KFP가 알아서 아티팩트 저장소(MinIO/S3)에 파일을 올려주고, 다음 컴포넌트가 받을 때도 알아서 다운로드합니다. 사용자는 그냥 *로컬 파일*인 것처럼 다루면 됩니다.

### 4.4 GPU 리소스 요청

```python
from kfp.kubernetes import use_secret_as_env

@dsl.pipeline
def gpu_pipeline():
    train_task = train_model(...)
    train_task.set_gpu_limit("1")            # GPU 1개
    train_task.set_memory_limit("32Gi")
    train_task.set_cpu_limit("8")
    train_task.set_accelerator_type("nvidia.com/gpu")
```

### 4.5 Katib 실험 정의 (선언적 YAML)

```yaml
apiVersion: kubeflow.org/v1beta1
kind: Experiment
metadata:
  name: bayesian-tuning
  namespace: kubeflow-user-example-com
spec:
  objective:
    type: maximize
    goal: 0.95
    objectiveMetricName: validation-accuracy
  algorithm:
    algorithmName: bayesianoptimization
  parallelTrialCount: 3
  maxTrialCount: 24
  parameters:
  - name: lr
    parameterType: double
    feasibleSpace: {min: "0.0001", max: "0.1"}
  - name: batch-size
    parameterType: int
    feasibleSpace: {min: "16", max: "128"}
  trialTemplate:
    primaryContainerName: training-container
    trialParameters:
    - name: learningRate
      reference: lr
    - name: batchSize
      reference: batch-size
    trialSpec:
      apiVersion: batch/v1
      kind: Job
      spec:
        template:
          spec:
            containers:
            - name: training-container
              image: my-trainer:latest
              command: ["python", "train.py", "--lr=${trialParameters.learningRate}", "--bs=${trialParameters.batchSize}"]
            restartPolicy: Never
```

이 YAML 한 장을 적용하면 베이지안 최적화로 24번의 학습을 (동시에 3개씩) 돌리며 최적 조합을 찾아냅니다.

### 4.6 KServe로 모델 배포

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: pytorch-classifier
spec:
  predictor:
    pytorch:
      storageUri: "s3://my-bucket/models/classifier-v1"
      resources:
        limits:
          nvidia.com/gpu: "1"
          memory: 4Gi
    minReplicas: 0   # 트래픽 없으면 0개로 스케일다운
    maxReplicas: 5
```

이 한 장이면 끝입니다. URL을 받아 `curl`로 추론 요청을 날릴 수 있습니다.

---

## 5부. NVIDIA DGX Spark 이해하기

### 5.1 한 줄 정의

> **DGX Spark = "내 책상 위의 AI 슈퍼컴퓨터"**

2025년 10월 15일 일반 출시된 GB10 Grace Blackwell Superchip 기반 개인용 AI 슈퍼컴퓨터로, 데스크톱에서 대규모 AI 모델을 프로토타이핑·파인튜닝·추론할 수 있도록 설계되었습니다.

### 5.2 하드웨어 사양 핵심 정리

| 항목 | 사양 |
|---|---|
| SoC | NVIDIA GB10 Grace Blackwell Superchip |
| CPU | 20-core Arm (10× Cortex-X925 + 10× Cortex-A725) |
| GPU | NVIDIA Blackwell (5세대 Tensor Core, FP4 지원) |
| 메모리 | 128GB LPDDR5X **통합 메모리(Unified)** |
| 메모리 대역폭 | ~273 GB/s |
| AI 성능 | 최대 1 PFLOP (FP4, sparsity 적용) / 1,000 TOPS |
| 스토리지 | 4TB NVMe SSD |
| 네트워크 | 듀얼 ConnectX-7 200GbE (RDMA 지원) |
| 폼팩터 | 150 × 150 × 50.5 mm, 약 1.1L |
| TDP | 140W |
| OS | DGX OS (Ubuntu 24.04 LTS 기반) |
| 아키텍처 | **ARM64 / aarch64** |

CUDA 생태계와 주요 프레임워크는 ARM64를 지원하지만, x86을 가정한 일부 특수 도구나 라이브러리는 추가 설정이나 대체가 필요할 수 있습니다. 이 점이 Kubeflow 설치 시 가장 큰 변수입니다.

### 5.3 통합 메모리(Unified Memory)가 왜 중요한가

전통적인 GPU 서버에서 70B 파라미터 모델을 학습하려면 모델 가중치를 **시스템 RAM → VRAM** 으로 PCIe(보통 64GB/s) 통해 옮겨야 합니다. DGX Spark의 통합 메모리는 CPU와 GPU가 동일한 128GB 풀에 직접 접근하기 때문에 PCIe 병목이 사라지며, GPU가 텐서 연산을 시작하는 동안 CPU는 같은 주소공간에서 전처리를 처리할 수 있습니다.

대신 **메모리 대역폭은 273 GB/s**로, 1700+ GB/s를 자랑하는 RTX 5090 같은 디스크리트 GPU보다는 낮습니다. **"개발/프로토타이핑·파인튜닝용 워크스테이션"**이라는 포지셔닝이 정확합니다. 대규모 학습이나 프로덕션 추론은 여전히 데이터센터 DGX나 클라우드 GPU 클러스터의 영역입니다.

### 5.4 두 대 묶기 — ConnectX-7 RDMA

내장 ConnectX-7 200GbE 네트워킹으로 두 대의 DGX Spark를 묶으면 256GB 메모리로 더 큰 모델까지 다룰 수 있습니다. 실제로 두 노드에 Kubernetes를 깔고 RDMA를 활성화한 사례들이 보고되고 있습니다.

### 5.5 사전 설치된 소프트웨어 스택

DGX OS는 Ubuntu 24.04 기반에 다음이 미리 깔려 있습니다:

- NVIDIA Driver, CUDA Toolkit
- NVIDIA Container Toolkit (Docker에서 GPU 사용)
- Docker Engine
- JupyterLab
- DGX Dashboard (웹 UI 모니터링)
- NVIDIA AI 소프트웨어 스택 (PyTorch, NeMo 등)

### 5.6 LLMOps 관점에서의 활용 시나리오

| 시나리오 | 적합도 | 비고 |
|---|---|---|
| 7B~13B 모델 LoRA 파인튜닝 | ★★★★★ | 통합 메모리로 여유롭게 실험 |
| 70B 모델 4비트 양자화 추론 | ★★★★☆ | 가능하지만 토큰 속도는 데이터센터 GPU보다 느림 |
| 200B 모델 추론 (FP4) | ★★★☆☆ | 통합 메모리 덕에 로딩은 가능 |
| 405B 모델 추론 | ★★★☆☆ | 두 대 묶어야 가능 |
| 대규모 분산 사전학습 | ☆☆☆☆☆ | 이 용도가 아님 |
| Kubeflow 학습 클러스터 노드 | ★★★☆☆ | ARM64 호환성 변수 있음 |

---

## 6부. DGX Spark에서 Kubeflow 설치 실습

> **솔직한 안내**: Kubeflow의 manifests 레포는 "ARM64/aarch64에서는 일부 OCI 이미지가 제공되지 않을 수 있어 완전 지원되지 않을 수 있다"고 명시하고 있습니다. 즉 DGX Spark에서 Kubeflow를 돌리는 것은 **"가능하지만 일부 컴포넌트는 손이 갈 수 있다"** 는 점을 미리 알아두세요. 이 가이드는 K3s + 핵심 컴포넌트만 골라 설치하는 실용적 경로를 따라갑니다.

### 6.1 사전 점검 — 환경 확인

먼저 DGX Spark에 SSH로 접속하거나 모니터를 연결한 뒤 다음을 확인합니다.

```bash
# 1) OS 정보
cat /etc/os-release        # Ubuntu 24.04 (DGX OS)
uname -m                    # aarch64

# 2) NVIDIA 드라이버와 GPU
nvidia-smi

# 3) Docker 및 NVIDIA Container Runtime
docker --version
docker info | grep -i runtime
# nvidia 런타임이 보여야 함

# 4) GPU 컨테이너 동작 확인
docker run --rm --gpus all \
  nvcr.io/nvidia/cuda:13.0.1-devel-ubuntu24.04 nvidia-smi
```

마지막 명령에서 호스트와 동일한 `nvidia-smi` 결과가 나오면 컨테이너에서 GPU를 잘 보고 있는 겁니다.

### 6.2 Kubernetes 클러스터 구성 — K3s 권장

DGX Spark처럼 **단일 노드** 환경에서는 Full Kubernetes 대신 **K3s**가 가볍고 편합니다. Rancher Labs가 만든 경량 K8s 디스트리뷰션으로, 단일 바이너리 설치, 적은 메모리 사용, ARM64 정식 지원이 장점입니다.

#### 6.2.1 NVIDIA Container Runtime을 K3s 기본 런타임으로 등록

```bash
# /etc/docker/daemon.json — Docker 사용 시
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}
EOF
sudo systemctl restart docker
```

#### 6.2.2 K3s 설치 (containerd 기반, 권장)

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --disable traefik

# kubectl 사용 환경 설정
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config

# 확인
kubectl get nodes
kubectl get pods -A
```

K3s는 기본으로 containerd를 쓰므로, NVIDIA Container Toolkit이 컨테이너 런타임에 잘 등록되어 있는지 확인합니다.

```bash
sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart k3s
```

> 참고: 두 대 이상의 DGX Spark로 멀티노드 Kubernetes 1.31 클러스터를 kubeadm으로 구성하고, Flannel CNI와 NVIDIA Network Operator로 RDMA를 활성화한 실제 사례도 보고되었습니다.

#### 6.2.3 Helm 설치

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | \
  sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update && sudo apt-get install -y helm
helm version
```

### 6.3 NVIDIA GPU Operator 설치

GPU Operator는 GPU 드라이버, 디바이스 플러그인, DCGM 모니터링을 한꺼번에 관리해주는 컴포넌트입니다. 다만 DGX Spark처럼 **드라이버가 이미 호스트에 깔려 있는** 환경에서는 옵션을 잘 줘야 합니다.

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install gpu-operator -n gpu-operator --create-namespace \
  nvidia/gpu-operator \
  --set driver.enabled=false \
  --set toolkit.enabled=false \
  --set devicePlugin.enabled=true \
  --set dcgmExporter.enabled=true \
  --set migManager.enabled=false
```

> ⚠️ **알려진 이슈**: DGX Spark의 통합 메모리(Unified RAM) 때문에 GPU Operator의 디바이스 플러그인이 "error getting device memory: Not Supported" 에러를 낼 수 있다는 보고가 있습니다. 이 경우 NVIDIA의 `gpu-operator` 최신 패치 버전(또는 GitHub Issue #1794의 워크어라운드)을 사용하거나, Time Slicing 모드로 우회해야 합니다.

설치 후 확인:

```bash
kubectl -n gpu-operator get pods
kubectl describe node | grep -A5 "Allocatable:"
# nvidia.com/gpu: 1 이 보여야 함
```

간단 GPU 테스트 Pod:

```bash
kubectl run gpu-test --rm -it --restart=Never \
  --image=nvcr.io/nvidia/cuda:13.0.1-devel-ubuntu24.04 \
  --overrides='{"spec":{"containers":[{"name":"gpu-test","image":"nvcr.io/nvidia/cuda:13.0.1-devel-ubuntu24.04","command":["nvidia-smi"],"resources":{"limits":{"nvidia.com/gpu":"1"}}}]}}'
```

### 6.4 Kubeflow 설치 — 두 가지 경로

DGX Spark에서는 풀 Kubeflow 매니페스트 전체보다 **필요한 것만 골라 깔기**를 권장합니다. ARM64 이미지 가용성 때문입니다.

#### 경로 A: 미니멀 — KServe만

가장 빠른 시작. LLM 서빙만 필요하다면 이걸로 충분합니다.

```bash
# Knative Serving (KServe 의존성)
KNATIVE_VERSION=v1.15.0
kubectl apply -f https://github.com/knative/operator/releases/download/knative-${KNATIVE_VERSION}/operator.yaml

cat <<EOF | kubectl apply -f -
apiVersion: operator.knative.dev/v1beta1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
EOF

# cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.yaml

# KServe
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.15.2/kserve.yaml
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.15.2/kserve-cluster-resources.yaml
```

#### 경로 B: 풀 매니페스트 (Kustomize 단일 명령)

```bash
git clone -b v1.10-branch https://github.com/kubeflow/manifests.git
cd manifests

# 한 방 설치 (성공할 때까지 재시도)
while ! kustomize build example | kubectl apply --server-side -f -; do
  echo "Retry in 20s..."
  sleep 20
done
```

> ⚠️ ARM64 이미지가 없는 컴포넌트는 `ImagePullBackOff`로 실패할 수 있습니다. 이때는 해당 Deployment/Pod 로그를 보고 (1) 같은 기능의 ARM64 호환 이미지로 교체하거나 (2) 필요 없으면 해당 컴포넌트를 빼고 설치합니다.

설치 후 Central Dashboard 접근:

```bash
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
# 브라우저에서 http://localhost:8080
# 기본 계정: user@example.com / 12341234
```

> 🔒 **운영 환경에서는 반드시 기본 비밀번호를 변경하세요.** Kubeflow 매니페스트는 데모용 기본값이라 그대로 두면 큰 보안 위험입니다.

### 6.5 첫 번째 노트북 띄우기

1. Central Dashboard에서 좌측 메뉴 → **Notebooks**
2. **+ New Notebook** 클릭
3. 폼 입력:
   - Name: `my-first-notebook`
   - Image: PyTorch CUDA 이미지 선택 (ARM64 호환 확인 필수)
   - CPU: 4, Memory: 16Gi
   - GPU: 1 (`nvidia.com/gpu`)
   - Workspace Volume: 10Gi
4. **Launch**
5. 대시보드에 노트북이 `Running` 상태가 되면 **Connect** 클릭 → JupyterLab 진입

JupyterLab에서 GPU 확인:

```python
import torch
print(torch.cuda.is_available())   # True
print(torch.cuda.get_device_name(0))
```

---

## 7부. LLM Fine-tuning 파이프라인 만들기

DGX Spark의 통합 메모리 장점을 살린 **7B 모델 LoRA 파인튜닝** 파이프라인 예시를 보겠습니다.

### 7.1 파이프라인 구조

```
[데이터 다운로드] → [전처리/토큰화] → [LoRA 학습] → [평가] → [모델 등록] → [KServe 배포]
```

### 7.2 KFP 코드

```python
# llm_finetune_pipeline.py
from kfp import dsl, compiler
from kfp.dsl import Input, Output, Dataset, Model

BASE_IMAGE = "nvcr.io/nvidia/pytorch:25.01-py3"  # ARM64 지원 NGC 이미지

@dsl.component(base_image=BASE_IMAGE,
               packages_to_install=["datasets==2.21.0", "huggingface_hub"])
def download_dataset(
    dataset_name: str,
    out_data: Output[Dataset],
):
    from datasets import load_dataset
    ds = load_dataset(dataset_name, split="train[:1000]")
    ds.save_to_disk(out_data.path)

@dsl.component(base_image=BASE_IMAGE,
               packages_to_install=["transformers==4.45.0", "datasets"])
def tokenize(
    raw_data: Input[Dataset],
    tokenized: Output[Dataset],
    model_name: str,
    max_length: int = 512,
):
    from datasets import load_from_disk
    from transformers import AutoTokenizer
    tok = AutoTokenizer.from_pretrained(model_name)
    ds = load_from_disk(raw_data.path)
    ds = ds.map(lambda x: tok(x["text"], truncation=True,
                              max_length=max_length, padding="max_length"),
                batched=True)
    ds.save_to_disk(tokenized.path)

@dsl.component(base_image=BASE_IMAGE,
               packages_to_install=["transformers", "peft", "datasets",
                                    "accelerate", "bitsandbytes"])
def lora_finetune(
    tokenized: Input[Dataset],
    model_out: Output[Model],
    base_model: str,
    lr: float = 2e-4,
    epochs: int = 3,
    lora_r: int = 16,
):
    import torch
    from datasets import load_from_disk
    from transformers import (AutoModelForCausalLM, AutoTokenizer,
                              TrainingArguments, Trainer)
    from peft import LoraConfig, get_peft_model, TaskType

    ds = load_from_disk(tokenized.path)
    model = AutoModelForCausalLM.from_pretrained(
        base_model, torch_dtype=torch.bfloat16, device_map="auto")
    config = LoraConfig(
        task_type=TaskType.CAUSAL_LM,
        r=lora_r, lora_alpha=32, lora_dropout=0.05,
        target_modules=["q_proj", "v_proj"],
    )
    model = get_peft_model(model, config)

    args = TrainingArguments(
        output_dir=model_out.path,
        per_device_train_batch_size=2,
        gradient_accumulation_steps=4,
        learning_rate=lr,
        num_train_epochs=epochs,
        bf16=True,
        save_strategy="epoch",
        logging_steps=10,
    )
    trainer = Trainer(model=model, args=args, train_dataset=ds)
    trainer.train()
    model.save_pretrained(model_out.path)
    model_out.metadata["base_model"] = base_model
    model_out.metadata["framework"] = "peft-lora"

@dsl.component(base_image=BASE_IMAGE,
               packages_to_install=["transformers", "peft"])
def evaluate(
    model_in: Input[Model],
    base_model: str,
    test_prompt: str = "한국의 수도는",
) -> str:
    from transformers import AutoModelForCausalLM, AutoTokenizer
    from peft import PeftModel
    tok = AutoTokenizer.from_pretrained(base_model)
    base = AutoModelForCausalLM.from_pretrained(base_model, device_map="auto")
    model = PeftModel.from_pretrained(base, model_in.path)
    inputs = tok(test_prompt, return_tensors="pt").to(model.device)
    output = model.generate(**inputs, max_new_tokens=50)
    return tok.decode(output[0], skip_special_tokens=True)


@dsl.pipeline(name="llm-lora-finetune-dgx-spark")
def llm_pipeline(
    base_model: str = "Qwen/Qwen3-4B",
    dataset_name: str = "tatsu-lab/alpaca",
    learning_rate: float = 2e-4,
    epochs: int = 3,
):
    raw = download_dataset(dataset_name=dataset_name)
    tok = tokenize(raw_data=raw.outputs["out_data"],
                   model_name=base_model)
    train = lora_finetune(
        tokenized=tok.outputs["tokenized"],
        base_model=base_model,
        lr=learning_rate,
        epochs=epochs,
    )
    train.set_gpu_limit("1")
    train.set_memory_limit("64Gi")  # 통합 메모리 활용

    eval_task = evaluate(model_in=train.outputs["model_out"],
                         base_model=base_model)


if __name__ == "__main__":
    compiler.Compiler().compile(llm_pipeline, "llm_pipeline.yaml")
```

### 7.3 파이프라인 제출

```python
from kfp.client import Client

client = Client(host="http://<your-kubeflow-host>:8080")
run = client.create_run_from_pipeline_func(
    llm_pipeline,
    arguments={
        "base_model": "Qwen/Qwen3-4B",
        "epochs": 3,
        "learning_rate": 2e-4,
    },
    experiment_name="dgx-spark-lora",
)
print(f"Run URL: {run.run_url}")
```

### 7.4 학습 결과를 KServe로 배포

학습이 끝나면 `model_out` 아티팩트가 MinIO에 저장됩니다. 이를 InferenceService로 띄웁니다.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: qwen-lora
  namespace: kubeflow-user-example-com
spec:
  predictor:
    model:
      modelFormat:
        name: huggingface
      storageUri: "s3://mlpipeline/v2/artifacts/llm-lora-finetune-dgx-spark/.../model_out"
      args:
      - --model_name=Qwen/Qwen3-4B
      - --enable-lora
      resources:
        limits:
          nvidia.com/gpu: "1"
          memory: 32Gi
    minReplicas: 0
    maxReplicas: 1
```

```bash
kubectl apply -f inference-service.yaml
kubectl get isvc -w
```

---

## 8부. 트러블슈팅과 운영 팁

### 8.1 자주 만나는 ARM64 이미지 문제

**증상**: Pod이 `ImagePullBackOff` 또는 `exec format error`

**원인**: 이미지가 amd64만 제공됨

**해결**:
```bash
# 이미지 multi-arch 지원 확인
docker buildx imagetools inspect <image-name>
# linux/arm64 가 manifest에 있는지 확인
```
없으면 (1) 같은 기능의 ARM64 이미지로 교체, (2) 직접 멀티아키 빌드, (3) 해당 컴포넌트 비활성화 중 선택.

### 8.2 통합 메모리 + GPU Operator

DGX Spark의 GB10 통합 메모리 환경에서 GPU 디바이스 플러그인이 "error getting device memory: Not Supported" 오류를 보고하는 케이스가 있습니다.

**대처**:
- GPU Operator 최신 패치 적용
- Time Slicing 설정으로 단일 GPU를 여러 Pod에 공유
- 임시로 호스트 Docker로 직접 추론 컨테이너를 띄우는 우회

### 8.3 단일 노드 K3s에서의 PVC

기본 K3s는 `local-path` provisioner를 씁니다. 단일 노드라 충분하지만, 노트북·파이프라인이 동시에 같은 PVC를 ReadWriteMany로 쓰려면 NFS나 Longhorn 같은 추가 설정이 필요합니다.

### 8.4 Knative와 KServe Cold Start

`minReplicas: 0`은 비용을 줄이지만, 첫 요청에서 모델 로딩 시간(LLM은 수십 초)이 그대로 노출됩니다. 운영에서는 `minReplicas: 1`을 권장합니다.

### 8.5 모니터링 — DCGM Exporter + Prometheus + Grafana

GPU Operator에 포함된 `dcgm-exporter`를 Prometheus가 스크랩하면 GPU utilization, memory, temperature를 모두 볼 수 있습니다. NVIDIA 공식 Grafana 대시보드 ID `12239`를 임포트하면 바로 시각화됩니다.

```bash
# Prometheus + Grafana (kube-prometheus-stack)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

### 8.6 리소스 점검 명령 모음

```bash
# 노드의 GPU 가용량
kubectl describe node | grep -E "(Allocatable|nvidia.com/gpu)"

# 네임스페이스별 사용량
kubectl top pod -A --sort-by=memory

# 파이프라인 잡의 로그
kubectl logs -f -n kubeflow-user-example-com <pod-name>

# 호스트 GPU 모니터
watch -n 1 nvidia-smi

# 컨테이너에서 통합 메모리 사용량
nvidia-smi --query-gpu=memory.used,memory.total --format=csv
```

---

## 9부. 다음 단계와 학습 리소스

### 9.1 추천 학습 순서

1. **이 문서의 4부 KFP 예제를 직접 한 번 돌려보기** — `add`/`multiply` 같은 장난감 파이프라인부터
2. **자기 데이터로 작은 분류 모델 파이프라인 만들기**
3. **Katib로 하이퍼파라미터 5개 정도 튜닝해 보기**
4. **KServe로 모델 1개 배포 + 메트릭 확인**
5. **DGX Spark 두 대 묶어 분산 학습** (가능한 경우)

### 9.2 공식 문서 & 커뮤니티

- Kubeflow 공식 사이트: `https://www.kubeflow.org`
- KFP SDK 문서: `https://kubeflow-pipelines.readthedocs.io`
- KServe 문서: `https://kserve.github.io/website`
- Katib 예제: `github.com/kubeflow/katib/tree/master/examples`
- DGX Spark 사용자 가이드: `https://docs.nvidia.com/dgx/dgx-spark`
- DGX Spark 개발자 포럼: `forums.developer.nvidia.com` (DGX Spark / GB10 카테고리)

### 9.3 인접 도구들 — 함께 보면 좋은 것

| 도구 | Kubeflow와의 관계 |
|---|---|
| **Argo Workflows** | KFP의 백엔드. 직접 쓸 수도 있음 |
| **MLflow** | 실험 추적 — Kubeflow Notebook 안에서 쓸 수 있음 |
| **Feast** | 피처 스토어 — KFP 컴포넌트로 통합 가능 |
| **Ray** | 분산 학습/튜닝 — KubeRay와 KFP 연동 |
| **vLLM** | LLM 추론 엔진 — KServe의 백엔드로 사용 |
| **NVIDIA NIM** | 추론 마이크로서비스 — DGX Spark에 사전 통합 |

### 9.4 마무리 한마디

Kubeflow는 처음 보면 컴포넌트도 많고 Kubernetes 지식도 요구해서 진입장벽이 있습니다. 하지만 일단 한 사이클(노트북 → 파이프라인 → 서빙)을 직접 굴려보고 나면, 이후의 모든 ML 작업이 *"코드 짠다 → 컨테이너로 만든다 → CRD로 선언한다"* 라는 동일한 패턴으로 단순해진다는 것을 체감할 수 있습니다.

DGX Spark는 그 학습을 책상 위에서 시작하기에 좋은 플랫폼입니다. 다만 *"ARM64라서 어떤 이미지는 직접 빌드해야 할 수 있다"* 는 점, *"통합 메모리는 매력적이지만 일부 K8s 컴포넌트가 그것을 가정하지 않고 만들어졌다"* 는 점을 염두에 두세요. 이 두 가지만 안고 가면, 개인 AI 슈퍼컴퓨터 위의 풀 MLOps 환경이라는 제법 멋진 결과를 얻을 수 있습니다.

---

> **부록 A. 자주 쓰는 명령어 한 장 요약**
>
> ```bash
> # 클러스터 상태
> kubectl get nodes -o wide
> kubectl get pods -A
> kubectl top nodes
>
> # Kubeflow 접속
> kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
>
> # KFP CLI
> kfp pipeline list
> kfp run list --experiment-id=<id>
>
> # KServe
> kubectl get inferenceservice -A
> kubectl logs -n <ns> <isvc-pod> -c kserve-container
>
> # GPU
> nvidia-smi
> nvidia-smi dmon  # 실시간
> kubectl describe node | grep nvidia.com/gpu
> ```

> **부록 B. 이 문서의 주요 버전 가정**
>
> - Kubeflow Platform 1.10 / 1.11
> - KFP 2.15+, KServe 0.15+, Katib 0.19+
> - Kubernetes 1.31~1.33
> - DGX OS 7.x (Ubuntu 24.04 기반)
> - K3s v1.31+
> - CUDA 13, NVIDIA Container Toolkit 최신
