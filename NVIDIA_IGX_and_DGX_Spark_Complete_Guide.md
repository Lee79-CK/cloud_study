# NVIDIA IGX & DGX Spark 완전 가이드

> **작성일**: 2026년 4월 14일  
> **대상 독자**: MLOps/LLMOps 엔지니어, Cloud/Edge AI 인프라 엔지니어, Physical AI 개발자  
> **키워드**: Edge AI, Physical AI, Functional Safety, GB10 Superchip, Blackwell Architecture

---

## 1장. NVIDIA IGX란 무엇인가

### 1.1 IGX의 정의와 포지셔닝

NVIDIA IGX(Industrial-Grade eXtreme)는 산업 현장과 의료 환경에 특화된 엔터프라이즈급 엣지 AI 플랫폼이다. 클라우드 데이터센터의 GPU 연산 능력을 공장, 수술실, 물류 창고 등 "현장(Edge)"으로 가져오되, 단순한 성능 확보를 넘어 **기능 안전(Functional Safety)**, **보안(Security)**, **10년 장기 지원(Long-Term Support)**이라는 산업 필수 요구사항을 동시에 충족하도록 설계되었다.

기존 산업용 AI 솔루션은 특정 용도에 맞게 커스텀 제작해야 했고, 비용과 시간이 막대했다. IGX는 이를 하나의 프로그래밍 가능한 플랫폼으로 통합함으로써, 다양한 산업 분야에서 공통된 하드웨어·소프트웨어 스택 위에 각기 다른 AI 애플리케이션을 배포할 수 있게 한다.

### 1.2 IGX 세대별 진화

| 세대 | 코드명 | GPU 아키텍처 | iGPU AI 연산 | 출시 시점 |
|---|---|---|---|---|
| 1세대 | IGX Orin | Ampere | 기준치 | 2023년 |
| 2세대 | **IGX Thor** | **Blackwell** | **8x 향상** | 2025년 12월 GA |

IGX Thor는 Orin 대비 iGPU에서 최대 8배, dGPU에서 최대 2.5배의 AI 연산 성능 향상을 달성했으며, 네트워크 대역폭도 2배로 증가하여 듀얼 200GbE 연결을 지원한다. FP4 정밀도 기준 최대 5,581 TFLOPS의 AI 연산을 제공한다.

### 1.3 핵심 아키텍처 구성 요소

**하드웨어 패밀리:**

IGX Thor 제품군은 두 가지 프로덕션 SKU로 구성된다.

**IGX T5000 SoM (System-on-Module)** — 컴팩트 임베디드 폼팩터의 모듈이다. Blackwell 아키텍처 iGPU, 14코어 Arm Neoverse V3AE CPU, 전용 가속기, 유연한 I/O를 통합하고 있다. 커스텀 캐리어 보드 설계에 적합하며, NVIDIA Jetson T5000과 폼팩터 및 핀 호환이 가능하다. 모바일 로봇, 자율 머신 등 공간·전력이 제한된 환경에 최적화되어 있다.

**IGX T7000 Board Kit** — MicroATX 폼팩터 기반의 고성능 보드 킷이다. NVIDIA RTX PRO 6000(Blackwell 기반) dGPU를 추가로 장착할 수 있어, 멀티 센서·멀티 모델 워크로드가 요구되는 환경에서 강력한 AI 컴퓨팅을 제공한다. ConnectX-7 네트워킹, BMC(Baseboard Management Controller), 기능 안전 모듈을 포함한다.

**개발자 킷:**

IGX Thor Developer Kit은 T7000 기반으로 RTX PRO 6000 dGPU가 포함된 풀 스케일 개발 플랫폼이며, IGX Thor Developer Kit Mini는 T5000 기반의 소형 개발 킷으로 안전 MCU가 포함되어 있다.

### 1.4 기능 안전(Functional Safety) 아키텍처

IGX Thor의 가장 차별화된 특징은 실리콘 레벨부터 설계된 기능 안전이다.

**Functional Safety Island** — 독립적인 안전 프로세서로, 안전 크리티컬 워크로드를 메인 AI 워크로드로부터 격리한다. ISO 26262(자동차), IEC 61508(산업 자동화) 표준을 ASIL D/SC3 및 ASIL/SIL 2 수준으로 충족하도록 설계되었다.

IGX의 안전 체계는 세 가지 레이어로 구분된다.

첫째, **반응적 안전(Reactive Safety)**은 위협이 발생한 후 이를 완화하는 것이다. 예를 들어, 로봇 경로에 사람이 들어오면 로봇이 정지하는 방식이다. 기존 자율 머신에 이미 구현되어 있는 메커니즘이다.

둘째, **능동적 안전(Proactive Safety)**은 사고가 발생하기 전에 잠재적 위험을 식별하는 것이다. IGX는 환경 내 AI 센서를 통해 사람이 로봇 경로로 향하는 것을 미리 감지하고, 로봇에게 경로 변경을 지시하여 충돌 자체를 방지한다. 이것이 NVIDIA Halos 안전 시스템의 핵심이다.

셋째, **예측적 안전(Predictive Safety)**은 과거 데이터를 기반으로 미래의 안전 위협을 예측하는 것이다. 시뮬레이션과 디지털 트윈을 활용하여 안전 사고 패턴을 식별하고 공장 레이아웃을 개선한다.

### 1.5 소프트웨어 스택

IGX는 NVIDIA AI Enterprise 소프트웨어 스택 위에서 동작하며, 다음과 같은 프레임워크를 네이티브로 지원한다.

**NVIDIA Holoscan** — 실시간 센서 데이터 처리 플랫폼으로, 의료 기기의 고대역폭 센서 데이터를 실시간으로 처리한다. 내시경, 초음파, 수술 로봇 등의 영상 데이터 파이프라인에 활용된다.

**NVIDIA Isaac** — 로보틱스 개발 플랫폼으로, 로봇의 인지(perception), 내비게이션, 조작(manipulation) 기능을 구현한다. Isaac Sim을 통한 시뮬레이션 기반 훈련과 Isaac ROS를 통한 실시간 추론을 포함한다.

**NVIDIA Metropolis** — 비전 AI 플랫폼으로, 공장·창고·도시 인프라에서 수백 대의 카메라와 IoT 센서를 연결하여 지능형 영상 분석을 수행한다.

**NVIDIA Halos** — 자율 차량과 로봇을 위한 풀스택 AI 안전 시스템으로, 온보드 센서를 활용한 "inside-out" 안전과 인프라 센서를 활용한 "outside-in" 안전을 모두 지원한다. KION Group은 Halos Outside-In Safety 워크플로우를 활용하여 인프라 장착 카메라와 동적 가상 안전 펜스로 자율 로봇의 기능 안전을 강화하고 있다.

**NVIDIA NIM** — 최적화된 추론 마이크로서비스로, VLA(Vision-Language-Action) 모델인 Isaac GR00T N이나 LLM/VLM 모델인 Cosmos Reason 등의 생성형 AI 모델을 엣지에서 실행한다.

### 1.6 IGX vs Jetson: 어떤 차이가 있는가

IGX와 Jetson은 모두 엣지 AI 플랫폼이지만 대상 시장이 다르다. Jetson은 범용 임베디드 AI와 로보틱스에 초점을 맞추며 가격 대비 성능과 전력 효율을 중시한다. IGX는 산업·의료 등 규제 환경에서의 엔터프라이즈 배포에 초점을 맞추며, 기능 안전 인증, 10년 장기 지원, 엔터프라이즈 보안, NVIDIA 인증 시스템 요구사항 등이 추가된다. IGX T5000은 Jetson T5000과 핀 호환이 되므로, Jetson으로 프로토타이핑한 후 IGX로 프로덕션 전환이 가능하다.

---

## 2장. NVIDIA IGX 실제 활용 사례

### 2.1 의료 및 헬스케어

의료 분야는 IGX의 가장 활발한 활용 영역 중 하나다.

**수술 로봇** — Johnson & Johnson은 Polyphonic 디지털 수술 플랫폼에 IGX Thor를 채택하여 수술실에서 실시간 AI 추론을 수행한다. CMR Surgical은 자사 수술 로봇 시스템에 IGX Thor를 평가하여 실시간 분석과 적응적 의사결정을 구현하고 있으며, LEM Surgical과 Horizon Surgical Systems는 수술 로봇의 정밀도와 기능 안전을 위해 IGX Thor를 채택했다.

**내시경 AI** — Medtronic은 IGX와 Holoscan을 활용하여 GI Genius 지능형 내시경 모듈을 구동한다. 이는 FDA 인증을 받은 최초의 AI 지원 대장내시경 도구로, 대장암으로 이어질 수 있는 용종을 탐지하는 데 활용된다.

**의료 영상** — KARL STORZ는 IGX Thor로 차세대 내시경 및 영상 도구를 개발하여 더 정확한 진단을 가능하게 하고 있다. Barco, Cosmo, XRlabs는 IGX Thor와 Holoscan을 채택하여 의료 등급의 엣지 AI 플랫폼을 구축하고 있다.

### 2.2 제조업 및 산업 자동화

**자율 공장** — Siemens는 NVIDIA와 협력하여 자율 공장 비전을 구현하고 있다. IGX 기술을 산업 컴퓨팅 포트폴리오에 통합하여 공장 내 반복 작업을 줄이고 작업자를 지원한다. 디지털 트윈과 산업 메타버스를 위한 Omniverse 플랫폼과의 연계 작업도 진행 중이다.

**산업용 로봇** — Maven Robotics는 IGX Thor를 차세대 범용 산업 로봇의 핵심 컴퓨팅 플랫폼으로 활용하고 있다. ADLINK은 IGX와 Holoscan을 사용하여 머신 이동 경로, 로봇 팔 조작, 충전소 모니터링을 동시에 최적화하는 산업 등급 엣지 AI 솔루션을 구축하고 있다.

**건설 장비** — Caterpillar는 IGX Thor 기반의 인캐빈 대화형 AI 어시스턴트를 개발하여 건설 장비 운전자의 생산성과 안전을 향상시키고 있다.

**물류** — KION Group은 IGX Thor와 NVIDIA Halos Outside-In Safety 워크플로우를 통해 인프라 장착 카메라와 동적 가상 안전 펜스를 활용한 "outside-in" 인지 기능을 구현하고 있다.

### 2.3 교통 및 항공

**철도** — Hitachi Rail은 IGX Thor를 사용하여 철도 네트워크에 고급 예측 유지보수 및 자율 점검 시스템을 배포하고 있다.

**항공** — Joby Aviation과 Archer는 IGX Thor를 채택하여 항공기 안전, 공역 통합, 자율 비행 준비 시스템에서 AI를 활용하고 있다.

### 2.4 과학 연구

**우주 탐사** — SETI Institute는 IGX를 활용하여 레이더 처리와 전파 천문학에서 실시간 엣지 AI를 구현하고 있다. Planet Labs는 IGX Thor를 채택하여 테라바이트 규모의 다차원 위성 데이터를 궤도상에서 실행 가능한 인텔리전스로 변환하고 있다.

### 2.5 파트너 생태계

IGX Thor는 광범위한 파트너 생태계를 보유하고 있다. Advantech, ADLINK, ASRock Rack, Barco, Connect Tech, Curtiss-Wright, Dedicated Computing, EIZO Rugged Solutions, Inventec, NexCOBOT, Onyx, WOLF Advanced Technology, YUAN 등의 장비 제조업체가 IGX Thor 기반 엣지 서버, 커스텀 캐리어 보드, 카메라·센서, AI·시스템 소프트웨어를 제공한다.

---

## 3장. NVIDIA DGX Spark란 무엇인가

### 3.1 DGX Spark의 정의와 위치

NVIDIA DGX Spark는 NVIDIA GB10 Grace Blackwell Superchip을 탑재한 개인용 AI 슈퍼컴퓨터다. Mac Mini와 유사한 크기(약 1.2kg)의 데스크톱 폼팩터에 최대 1 PetaFLOP(FP4 스파시티 기준)의 AI 연산 성능과 128GB 통합 메모리를 담았다.

원래 CES 2025에서 "Project DIGITS"라는 이름으로 공개되었으며, GTC 2025에서 DGX Spark로 정식 명명된 후 2025년 말에 출하가 시작되었다. 2026년 2월 기준 가격은 $4,699이다.

핵심 가치는 명확하다 — 데이터센터 수준의 AI 연산 능력을 개인 개발자의 책상 위로 가져온 것이다. 클라우드 GPU 인스턴스 비용, SSH 접속 대기, 데이터 주권 문제 없이 로컬에서 대규모 AI 모델을 다룰 수 있다.

### 3.2 하드웨어 사양

| 항목 | 사양 |
|---|---|
| SoC | NVIDIA GB10 Grace Blackwell Superchip |
| CPU | ARM 기반 Grace CPU |
| GPU | Blackwell 아키텍처, 5세대 Tensor Core |
| 메모리 | 128GB 통합(Unified) LPDDR5x |
| AI 연산 | 최대 1,000 TOPS (1 PFLOP FP4 스파시티) |
| 정밀도 지원 | FP4, FP8, BF16, FP16, FP32 |
| 칩 간 연결 | NVLink-C2C |
| 네트워킹 | ConnectX-7 (200GbE QSFP) |
| OS | DGX OS (Ubuntu 24.04 기반) |
| 무게 | 약 1.2kg |
| 전력 | 최대 300W |

핵심적인 아키텍처 특징은 **통합 메모리 아키텍처(UMA)**다. CPU와 GPU가 128GB LPDDR5x 메모리 풀을 공유하므로, GPU 메모리 부족으로 모델을 로드하지 못하는 문제가 크게 줄어든다. 이를 통해 Llama 3.3 70B를 양자화 없이 BF16으로 실행하거나, 405B 규모 모델을 Q2-Q3 양자화로 구동할 수 있다.

ConnectX-7 네트워킹을 통해 두 대의 DGX Spark를 QSFP 케이블로 직접 연결하면, 총 256GB 메모리 풀에서 최대 405B 파라미터 모델의 추론이 가능하다.

### 3.3 소프트웨어 스택

DGX Spark는 DGX OS를 기본 탑재하고 있다. 이는 Ubuntu 24.04 기반의 커스텀 Linux 배포판으로, CUDA 툴킷, cuDNN, TensorRT, NVIDIA Container Runtime이 사전 구성되어 있다. 대규모 DGX 시스템과 동일한 DGX OS 기반이므로, Spark에서 개발한 코드가 프로덕션 인프라로 호환성 문제 없이 마이그레이션된다.

사전 설치된 주요 구성 요소는 다음과 같다: CUDA 툴킷 및 드라이버, cuDNN과 TensorRT(최적화 추론), NVIDIA Container Runtime(Docker GPU 패스스루), PyTorch, vLLM, SGLang, llama.cpp 등 추론 엔진, NVIDIA RAPIDS(cuDF, cuML)를 통한 데이터 사이언스 가속, JupyterLab 개발 환경 등이다.

### 3.4 CES 2026 소프트웨어 업데이트

CES 2026에서 발표된 소프트웨어 업데이트는 DGX Spark의 역할을 크게 확장했다.

**성능 향상** — TensorRT-LLM 최적화, NVFP4 양자화, 스펙큘러티브 디코딩 등을 통해 주요 워크로드에서 출시 대비 최대 2.5배 성능 향상을 달성했다. Qwen-235B 모델의 경우 FP8에서 NVFP4 양자화와 Eagle3 스펙큘러티브 디코딩을 적용하면 처리량이 2배 이상 증가한다.

**DGX Spark Playbooks** — 40개 이상의 단계별 배포 가이드가 공개되었다. 추론 엔진 설정, 파인튜닝 프레임워크, 개발 환경 구성, 네트워킹 인프라, 엔드투엔드 AI 애플리케이션 등을 포괄한다.

**NVIDIA NemoClaw** — 안전하고 프라이빗한 로컬 AI 에이전트를 구축·배포하기 위한 오픈소스 에이전트 개발 플랫폼이다. DGX Spark에서 항시 구동되는 자율 에이전트를 보안과 프라이버시를 유지하면서 운영할 수 있게 한다.

### 3.5 DGX Spark의 주요 사용 시나리오

**로컬 LLM 추론 및 테스트** — 최대 200B 파라미터 모델을 로컬에서 추론하고, 다양한 양자화 옵션(FP4, FP8, BF16)을 테스트할 수 있다. 클라우드 API 비용 없이 프롬프트 엔지니어링과 모델 평가를 반복 수행할 수 있다.

**모델 파인튜닝** — 128GB 통합 메모리를 활용하여 최대 70B 파라미터 모델의 파인튜닝이 가능하다. LoRA, QLoRA 등의 파라미터 효율적 기법과 전체 파인튜닝 모두 지원한다.

**데이터 사이언스** — NVIDIA RAPIDS를 통해 pandas, scikit-learn 워크플로우를 코드 변경 없이 GPU 가속한다. cuDF는 pandas 워크플로우를, cuML은 scikit-learn 알고리즘(UMAP, HDBSCAN, SVC 등)을 가속한다.

**엣지 애플리케이션 개발** — NVIDIA Isaac(로보틱스), Metropolis(비전 AI), Holoscan(센서 처리) 등의 프레임워크를 활용하여 엣지 애플리케이션을 프로토타이핑할 수 있다.

**로컬 AI 에이전트** — NemoClaw/OpenClaw를 통해 프라이빗하게 동작하는 항시 가동 AI 에이전트를 구축한다. 파일, 앱, 통신 채널과 통합하면서도 모든 데이터가 로컬에 유지된다.

---

## 4장. DGX Spark 실습 가이드

### 4.1 초기 설정 및 접속

DGX Spark를 처음 시작할 때 세 가지 접속 방법을 선택할 수 있다.

**방법 1: NVIDIA Sync (권장)** — macOS, Windows, Ubuntu용 데스크톱 앱으로, SSH 키 생성과 배포, 포트 포워딩을 자동화한다. 설치 후 원클릭으로 Spark에 연결할 수 있다.

**방법 2: 수동 SSH** — mDNS 호스트네임(예: `spark-abcd.local`) 또는 직접 IP를 통해 터미널에서 SSH 접속한다.

```bash
# mDNS를 통한 SSH 접속
ssh username@spark-abcd.local

# 또는 IP로 접속
ssh username@192.168.1.xxx
```

**방법 3: Tailscale (원격 접속)** — 같은 로컬 네트워크에 있지 않을 때 Tailscale VPN을 통해 안전하게 원격 접속한다.

```bash
# DGX Spark에서 Tailscale 설치 및 인증
sudo tailscale up
# 브라우저에서 인증 완료 후
# 클라이언트에서 접속
ssh user@spark-hostname
```

### 4.2 시스템 확인

접속 후 기본적인 시스템 상태를 확인한다.

```bash
# GPU 드라이버 확인
nvidia-smi

# 드라이버 버전 확인 (580.126.09 이상 필요)
nvidia-smi | grep "Driver Version"

# CUDA 버전 확인
nvcc --version

# Docker NVIDIA 런타임 확인
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu22.04 nvidia-smi

# 디스크 공간 확인
df -h

# 통합 메모리 상태 확인
free -h
```

### 4.3 실습 1: LLM 로컬 추론 (Open WebUI + Ollama)

가장 빠르게 로컬 LLM을 체험하는 방법은 Open WebUI를 배포하는 것이다.

```bash
# Ollama 설치 및 모델 다운로드
ollama pull llama3.3:70b

# Open WebUI Docker 컨테이너 실행
docker run -d \
  --gpus=all \
  -p 3000:8080 \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

브라우저에서 `http://localhost:3000` 또는 `http://<SPARK_IP>:3000`으로 접속하면 ChatGPT와 유사한 인터페이스에서 로컬 LLM과 대화할 수 있다. 128GB 통합 메모리 덕분에 70B 모델을 양자화 없이 로드할 수 있다는 것이 핵심 장점이다.

### 4.4 실습 2: vLLM을 활용한 고성능 추론 서버

프로덕션급 추론 서버를 구축하려면 vLLM을 사용한다.

```bash
# vLLM 서버 실행 (Qwen3-30B 예시)
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen3-30B \
  --quantization nvfp4 \
  --tensor-parallel-size 1 \
  --max-model-len 8192 \
  --port 8000
```

OpenAI 호환 API 엔드포인트가 생성되므로, 기존 OpenAI 클라이언트 코드를 그대로 사용하되 base_url만 변경하면 된다.

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"
)

response = client.chat.completions.create(
    model="Qwen/Qwen3-30B",
    messages=[{"role": "user", "content": "Edge AI란 무엇인가요?"}]
)
print(response.choices[0].message.content)
```

### 4.5 실습 3: NVFP4 양자화로 성능 최적화

DGX Spark의 5세대 Tensor Core는 FP4를 네이티브 지원한다. NVFP4 양자화를 적용하면 메모리 사용량을 크게 줄이면서도 품질 저하를 최소화할 수 있다.

```bash
# TensorRT-LLM을 활용한 NVFP4 양자화 추론
# (DGX Spark Playbooks의 nvfp4-quantization 참조)
# 대형 모델도 128GB 메모리 안에서 효율적으로 구동 가능
```

Qwen-235B 같은 초대형 모델도 NVFP4 양자화와 스펙큘러티브 디코딩(Eagle3)을 결합하면 DGX Spark 단일 노드에서 실행이 가능해진다.

### 4.6 실습 4: 모델 파인튜닝

```python
# LoRA 파인튜닝 예시 (Hugging Face + PEFT)
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import LoraConfig, get_peft_model

model_name = "meta-llama/Llama-3.1-8B"
model = AutoModelForCausalLM.from_pretrained(model_name)
tokenizer = AutoTokenizer.from_pretrained(model_name)

# LoRA 구성
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05,
    task_type="CAUSAL_LM"
)
model = get_peft_model(model, lora_config)

# 128GB UMA 덕분에 대형 모델도 메모리 부족 없이 파인튜닝 가능
# 학습 루프 구현...
```

128GB 통합 메모리를 활용하면 최대 70B 파라미터 모델까지 파인튜닝할 수 있으며, 8B~13B 모델의 경우 전체(full) 파인튜닝도 가능하다.

### 4.7 실습 5: GPU 가속 데이터 사이언스 (RAPIDS)

DGX Spark에서 RAPIDS를 사용하면 기존 pandas/scikit-learn 코드를 변경하지 않고 GPU 가속을 적용할 수 있다.

```python
# JupyterLab에서 cuDF 활성화 (pandas 가속)
%load_ext cudf.pandas
import pandas as pd  # 내부적으로 cuDF가 GPU 가속 처리

df = pd.read_csv("large_dataset.csv")
result = df.groupby("category").agg({"value": "mean"})
# GPU에서 자동 가속됨!
```

```python
# cuML 활성화 (scikit-learn 가속)
%load_ext cuml.accel
from sklearn.cluster import HDBSCAN
from sklearn.decomposition import PCA

# GPU에서 자동 가속됨
clustering = HDBSCAN(min_cluster_size=50)
labels = clustering.fit_predict(features)
```

### 4.8 실습 6: 두 대의 DGX Spark 클러스터링

405B 파라미터 모델을 실행하려면 두 대의 DGX Spark를 연결한다.

```bash
# 1. QSFP 케이블로 물리적 연결
# 2. 네트워크 인터페이스 확인
ibdev2netdev

# 3. 각 노드에 정적 IP 할당
# Node 1:
sudo ip addr add 192.168.100.10/24 dev enp1s0f1np1
sudo ip link set enp1s0f1np1 up

# Node 2:
sudo ip addr add 192.168.100.11/24 dev enp1s0f1np1
sudo ip link set enp1s0f1np1 up

# 4. SSH 키 교환으로 패스워드리스 접속 구성
# 5. 분산 추론 실행
```

이렇게 구성하면 200GbE 직접 연결을 통해 256GB 메모리 풀에서 Llama 3.1 405B 같은 초대형 모델의 추론이 가능해진다.

### 4.9 실습 7: VSS(Video Search & Summarization) AI Blueprint

영상 분석 파이프라인을 DGX Spark에 배포하는 실습이다.

```bash
# NGC API 키 설정
export NGC_CLI_API_KEY='your_ngc_api_key'

# VSS Event Reviewer 배포 (완전 로컬, VLM 파이프라인)
scripts/dev-profile.sh up -p base -H DGX-SPARK

# 또는 Standard VSS (하이브리드, 원격 LLM 연동)
export LLM_ENDPOINT_URL=https://your-llm-endpoint.com
scripts/dev-profile.sh up -p base -H DGX-SPARK \
  --use-remote-llm --llm <REMOTE_LLM_MODEL_NAME>
```

브라우저에서 `http://localhost:3000`으로 접속하여 영상을 업로드하고, AI가 영상 내용을 자동으로 분석·요약하는 것을 확인할 수 있다.

### 4.10 메모리 관리 팁

DGX Spark의 UMA 환경에서는 Linux 버퍼 캐시 때문에 실제 메모리 용량이 충분함에도 "Out of Memory" 에러가 발생할 수 있다. 이 경우 다음 명령으로 캐시를 해제한다.

```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

이는 UMA에 아직 최적화되지 않은 애플리케이션에서 흔히 발생하는 문제이며, DGX Dashboard에서 통합 메모리 사용량을 실시간으로 모니터링할 수 있다.

---

## 5장. IGX와 DGX Spark의 연계 — 개발에서 배포까지

### 5.1 워크플로우 아키텍처

MLOps/LLMOps 관점에서 IGX와 DGX Spark는 AI 모델 수명주기의 서로 다른 단계를 담당한다.

**개발 단계 (DGX Spark)** → 로컬에서 모델 프로토타이핑, 파인튜닝, 테스트를 수행한다. 128GB 통합 메모리를 활용하여 다양한 모델과 양자화 옵션을 빠르게 실험한다.

**데이터센터 학습 (DGX SuperPOD/Cloud)** → 대규모 학습이 필요한 경우 데이터센터 또는 클라우드 GPU 클러스터에서 분산 학습을 수행한다. DGX Spark에서 개발한 코드가 동일한 DGX OS/CUDA 스택에서 호환성 문제 없이 동작한다.

**엣지 배포 (IGX Thor)** → 학습·최적화된 모델을 산업 현장에 배포한다. IGX Thor의 기능 안전, 엔터프라이즈 보안, 10년 장기 지원 하에서 프로덕션 워크로드를 운영한다. NVIDIA NIM을 통해 클라우드에서 엣지까지 동일한 추론 마이크로서비스를 사용한다.

### 5.2 실무 시나리오: 수술 로봇 AI 파이프라인

구체적인 예시를 들어보자. 수술 로봇용 AI 비전 모델을 개발하는 시나리오다.

1단계 — DGX Spark에서 NVIDIA Holoscan SDK를 사용하여 내시경 영상 처리 파이프라인을 프로토타이핑한다. 다양한 VLM(Vision Language Model)을 로컬에서 테스트하고, 수술 영상 데이터로 파인튜닝한다.

2단계 — 검증된 모델을 DGX Cloud 또는 온프레미스 DGX 클러스터로 이전하여 대규모 학습 데이터로 본격 학습한다.

3단계 — 학습된 모델을 TensorRT로 최적화하고, NVIDIA NIM으로 패키징한다.

4단계 — IGX Thor Developer Kit에서 실제 의료 기기 환경 시뮬레이션을 수행한다. 기능 안전 요구사항(IEC 61508)을 검증하고, 실시간 센서 데이터 처리 성능을 테스트한다.

5단계 — IGX Thor T7000 보드 킷 기반의 인증된 시스템에 배포하여 실제 수술실에서 운영한다.

### 5.3 Cloud Engineer를 위한 인프라 고려사항

**Fleet Management** — NVIDIA Fleet Command를 통해 원격에서 IGX 디바이스 플릿에 보안 OTA(Over-the-Air) 소프트웨어 업데이트를 배포할 수 있다.

**컨테이너 기반 배포** — DGX Spark와 IGX Thor 모두 Docker 컨테이너 기반으로 AI 애플리케이션을 배포한다. NGC(NVIDIA GPU Cloud) 컨테이너 레지스트리에서 최적화된 컨테이너 이미지를 가져와 사용한다.

**모니터링** — DGX Dashboard를 통해 GPU 사용률, 메모리 상태, 온도, 전력 소비를 실시간으로 모니터링한다.

**보안** — NVIDIA AI Enterprise는 엔터프라이즈급 보안을 제공하며, IGX의 BMC(Baseboard Management Controller)를 통해 시스템 관리와 복구를 수행한다.

---

## 6장. 참고 리소스

### 공식 문서 및 개발자 포털

- NVIDIA IGX 공식 페이지: https://www.nvidia.com/en-us/edge-computing/products/igx/
- NVIDIA IGX 개발자 가이드: https://docs.nvidia.com/igx/user-guide/latest/introduction.html
- NVIDIA IGX 개발자 포털: https://developer.nvidia.com/igx
- DGX Spark 공식 페이지: https://www.nvidia.com/en-us/products/workstations/dgx-spark/
- DGX Spark 사용자 가이드: https://docs.nvidia.com/dgx/dgx-spark/
- DGX Spark 개발자 포털: https://developer.nvidia.com/topics/ai/dgx-spark

### 실습 자료

- DGX Spark Playbooks (GitHub): https://github.com/NVIDIA/dgx-spark-playbooks — 40개 이상의 단계별 배포 가이드로, 추론, 파인튜닝, 데이터 사이언스, 네트워킹, AI 에이전트 등을 포괄한다.
- NVIDIA DLI 핸즈온 AI 코스: DGX Spark 구매 시 무료 제공($90 상당)

### 소프트웨어 프레임워크

- NVIDIA Holoscan: 실시간 센서 처리 플랫폼
- NVIDIA Isaac: 로보틱스 개발 플랫폼
- NVIDIA Metropolis: 비전 AI 플랫폼
- NVIDIA Halos: 풀스택 AI 안전 시스템
- NVIDIA NIM: 최적화 추론 마이크로서비스
- NVIDIA RAPIDS: GPU 가속 데이터 사이언스
- NVIDIA NemoClaw: AI 에이전트 개발 플랫폼

---

> **요약**: NVIDIA IGX는 산업·의료 현장에서 안전하고 안정적인 AI 배포를 위한 엔터프라이즈 엣지 플랫폼이며, DGX Spark는 개발자 책상 위에서 데이터센터급 AI 연산을 가능하게 하는 개인용 AI 슈퍼컴퓨터다. 두 플랫폼은 동일한 NVIDIA 소프트웨어 스택을 공유하므로, DGX Spark에서 프로토타이핑하고 IGX Thor에서 프로덕션 배포하는 매끄러운 워크플로우를 구성할 수 있다.
