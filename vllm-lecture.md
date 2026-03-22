# vLLM 종합 강의 자료 — 기본 개념부터 프로덕션 배포까지

---

## Part 1. vLLM 개요

### 1.1 vLLM이란?

vLLM(Virtual Large Language Model)은 UC Berkeley의 Sky Computing Lab에서 개발한 **고성능 LLM 추론(Inference) 및 서빙(Serving) 엔진**이다. 2023년 SOSP 논문 *"Efficient Memory Management for Large Language Model Serving with PagedAttention"*에서 처음 공개되었으며, LLM 서빙 시 가장 큰 병목인 **GPU 메모리의 비효율적 사용 문제**를 OS의 가상 메모리(Virtual Memory) 개념을 차용하여 해결한 것이 핵심이다.

기존 LLM 서빙 프레임워크(Hugging Face `transformers`, FasterTransformer 등)에서는 각 요청마다 KV Cache를 고정 크기로 사전 할당하는 방식을 사용했다. 이 방식은 시퀀스 길이가 가변적인 실제 워크로드에서 심각한 메모리 단편화(fragmentation)와 낭비를 초래한다. vLLM은 이 문제를 PagedAttention이라는 혁신적인 메커니즘으로 해결하여, 동일 GPU에서 **2~4배 더 높은 처리량(throughput)**을 달성한다.

### 1.2 왜 vLLM인가? — 기존 서빙의 한계

LLM 추론의 핵심 비용은 **KV Cache 메모리 관리**에서 발생한다. Transformer 기반 LLM은 Auto-Regressive 방식으로 토큰을 하나씩 생성하는데, 이전에 계산된 Key-Value 텐서를 캐싱(KV Cache)해야 중복 계산을 피할 수 있다. 문제는 이 KV Cache의 크기가 시퀀스 길이에 비례하여 선형적으로 증가한다는 점이다.

**기존 방식의 문제점 세 가지:**

첫째, **내부 단편화(Internal Fragmentation)**. 요청이 들어오면 최대 시퀀스 길이만큼의 메모리를 미리 예약한다. 실제 생성이 최대 길이보다 훨씬 짧게 끝나면, 예약된 메모리의 상당 부분이 낭비된다. 평균적으로 GPU 메모리의 20~40%가 이런 식으로 사라진다.

둘째, **외부 단편화(External Fragmentation)**. 다양한 길이의 요청이 도착하고 완료되면서, 메모리에 사용 불가능한 빈 공간(hole)이 생긴다. 총 여유 메모리는 충분하지만 연속된 공간이 없어 새 요청을 수용하지 못하는 상황이 발생한다.

셋째, **메모리 공유 불가**. Beam Search나 Parallel Sampling처럼 여러 시퀀스가 동일한 프롬프트를 공유하는 경우에도, 기존 방식에서는 KV Cache를 각각 별도로 복사해야 했다.

### 1.3 vLLM의 핵심 가치 정리

vLLM이 제공하는 핵심 가치는 다음과 같다. 높은 처리량(Throughput)으로 동일 GPU 대비 2~4배 더 많은 요청을 처리할 수 있다. 낮은 지연시간(Latency)으로 Continuous Batching과 효율적 스케줄링을 통해 응답 시간을 최소화한다. 메모리 효율성 측면에서 PagedAttention으로 GPU 메모리 활용률을 극대화한다. OpenAI 호환 API를 제공하여 기존 시스템과의 통합이 용이하다. 다양한 모델을 지원하여 LLaMA, Mistral, Qwen, GPT-NeoX, Falcon 등 주요 LLM 아키텍처를 모두 커버한다.

---

## Part 2. 핵심 기술 — PagedAttention 심층 분석

### 2.1 OS 가상 메모리에서 영감을 얻다

운영체제는 물리 메모리를 고정 크기의 **페이지(Page)**로 나누고, 가상 주소와 물리 주소를 **페이지 테이블(Page Table)**로 매핑한다. 프로세스는 연속된 가상 메모리를 보지만, 실제 물리 메모리는 불연속적으로 흩어져 있을 수 있다. 이 추상화 덕분에 외부 단편화가 사라지고, 메모리를 유연하게 관리할 수 있다.

PagedAttention은 이 아이디어를 그대로 KV Cache 관리에 적용한다.

### 2.2 PagedAttention 동작 원리

**블록(Block) 단위 관리:** KV Cache를 고정 크기의 블록으로 나눈다. 하나의 블록은 고정된 수의 토큰(예: 16개)에 대한 Key-Value 텐서를 저장한다. 이 블록은 GPU 메모리상에서 연속적일 필요가 없다.

**블록 테이블(Block Table):** 각 시퀀스는 자신만의 블록 테이블을 가진다. 이 테이블은 논리적 블록 번호(0, 1, 2, ...)를 실제 물리적 GPU 메모리 블록으로 매핑한다. Attention 연산 시, 커스텀 CUDA 커널이 이 블록 테이블을 참조하여 불연속적으로 흩어진 KV Cache 블록들을 올바르게 접근한다.

**동적 할당:** 시퀀스 생성이 진행되면서 새 토큰이 만들어질 때마다, 현재 블록에 여유가 있으면 그 블록에 추가하고, 블록이 가득 차면 새 블록을 할당한다. 생성이 끝나면 해당 시퀀스의 블록들을 즉시 해제한다. 이로써 내부 단편화는 마지막 블록 하나에서만 발생하며, 외부 단편화는 원천적으로 제거된다.

**메모리 낭비율 비교:** 기존 시스템에서 평균 60~80%였던 메모리 활용률이, vLLM에서는 **96% 이상**으로 향상된다. 낭비되는 메모리는 이론적으로 시퀀스당 마지막 블록의 빈 슬롯뿐이며, 이는 전체 메모리의 4% 미만이다.

### 2.3 Copy-on-Write (CoW) 메커니즘

Beam Search나 Parallel Sampling에서는 하나의 프롬프트에서 여러 출력 시퀀스가 분기된다. 이 시퀀스들은 프롬프트 부분의 KV Cache를 공유할 수 있다.

PagedAttention은 OS의 Copy-on-Write 기법을 적용한다. 분기 시점에서 새 시퀀스는 기존 시퀀스의 블록 테이블을 복사하되, 실제 물리 블록은 공유한다. 각 물리 블록은 참조 카운트(reference count)를 유지하며, 어느 한 시퀀스가 공유 블록을 수정하려 할 때에만 해당 블록을 복사한다. 이를 통해 Beam Search에서 메모리 사용량이 최대 55%까지 절감된다.

### 2.4 블록 크기(Block Size) 튜닝

블록 크기는 성능에 직접적인 영향을 미치는 하이퍼파라미터다. 블록 크기가 작으면(예: 1) 내부 단편화가 최소화되지만 블록 테이블이 커지고 커널 오버헤드가 증가한다. 블록 크기가 크면(예: 64) 커널 효율은 좋아지지만 내부 단편화가 커진다. 일반적으로 16이 좋은 기본값이며, 워크로드 특성에 따라 8~32 사이에서 벤치마크를 통해 최적값을 찾는 것이 권장된다.

---

## Part 3. Continuous Batching과 스케줄링

### 3.1 Static Batching vs Continuous Batching

**Static Batching (기존 방식):** 배치 내 모든 요청이 완료될 때까지 새 요청을 추가하지 못한다. 빠르게 끝난 요청의 GPU 자원이 다른 요청이 끝날 때까지 유휴 상태로 놀게 된다. 배치 내 가장 긴 시퀀스에 의해 전체 지연시간이 결정되는 구조적 비효율이 존재한다.

**Continuous Batching (vLLM 방식):** 매 iteration(한 토큰 생성 단위)마다 스케줄링을 수행한다. 생성이 완료된 요청은 즉시 배치에서 제거되고, 대기 중인 새 요청이 빈자리를 채운다. GPU가 항상 최대한 바쁘게 유지되므로 처리량이 극대화된다.

### 3.2 vLLM 스케줄러 상세

vLLM의 스케줄러는 세 개의 큐를 관리한다.

**Waiting Queue:** 아직 Prefill(프롬프트 처리)이 시작되지 않은 요청들이 대기하는 곳이다. FCFS(First Come First Served) 순서로 처리된다.

**Running Queue:** 현재 Decode(토큰 생성) 단계에 있는 요청들이다. 매 iteration마다 한 토큰씩 생성한다.

**Swapped Queue:** GPU 메모리 부족 시 KV Cache가 CPU 메모리로 스왑 아웃된 요청들이다. GPU 메모리에 여유가 생기면 다시 스왑 인된다.

**스케줄링 흐름:** 매 iteration마다 스케줄러는 먼저 Running Queue의 요청들이 계속 실행 가능한지 확인한다. GPU 메모리가 부족하면 우선순위가 낮은 요청을 Swapped Queue로 보낸다. 그 다음, Waiting Queue에서 새 요청을 가져와 Prefill을 수행할 수 있는지 판단한다. 마지막으로, GPU 메모리에 여유가 있으면 Swapped Queue에서 요청을 복귀시킨다.

### 3.3 Prefill과 Decode의 분리 이해

LLM 추론은 두 단계로 나뉜다.

**Prefill 단계:** 전체 프롬프트 토큰을 한 번에 처리하여 KV Cache를 구성한다. 이 단계는 compute-bound로 GPU 연산 능력이 병목이다. 병렬로 처리되므로 프롬프트 길이에 비해 상대적으로 빠르다.

**Decode 단계:** 한 번에 한 토큰씩 auto-regressive하게 생성한다. 이 단계는 memory-bandwidth-bound로 KV Cache 읽기가 병목이다. 토큰 수만큼 반복되므로 전체 추론 시간의 대부분을 차지한다.

vLLM은 이 두 단계를 인식하고, Prefill 요청과 Decode 요청을 같은 배치에 포함시킬 수 있다(Chunked Prefill 지원). 이를 통해 긴 프롬프트의 Prefill이 짧은 요청의 Decode를 블로킹하지 않도록 한다.

### 3.4 Chunked Prefill

긴 프롬프트의 Prefill을 한 번에 처리하면, 그 동안 Decode 중인 다른 요청들이 대기해야 한다. Chunked Prefill은 긴 프롬프트를 여러 청크로 나눠서 여러 iteration에 걸쳐 Prefill을 수행한다. 각 iteration에서 Prefill 청크와 Decode 토큰을 함께 처리하므로, 기존 Decode 요청의 지연시간 증가를 방지한다. `--enable-chunked-prefill` 옵션으로 활성화하며, `--max-num-batched-tokens`으로 청크 크기를 조절한다.

---

## Part 4. 고급 최적화 기법

### 4.1 Tensor Parallelism (TP)

하나의 모델이 단일 GPU 메모리에 들어가지 않을 때, 모델의 각 레이어를 여러 GPU에 걸쳐 수평 분할하는 기법이다. 예를 들어 70B 모델을 4개 GPU에 분산하면, 각 Attention Head와 FFN 레이어가 4등분되어 각 GPU에 배치된다. 각 레이어 연산 후 All-Reduce 통신으로 결과를 동기화한다.

```bash
# 4-GPU Tensor Parallelism
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-70B-Instruct \
    --tensor-parallel-size 4 \
    --gpu-memory-utilization 0.90
```

TP는 NVLink 같은 고대역폭 GPU 간 연결이 있을 때 효과적이다. PCIe 연결에서는 통신 오버헤드가 커져 성능이 저하될 수 있다.

### 4.2 Pipeline Parallelism (PP)

모델의 레이어를 여러 GPU에 순차적으로 배치하는 기법이다. GPU 0이 레이어 0~19를, GPU 1이 레이어 20~39를 담당하는 식이다. TP와 달리 All-Reduce가 필요 없어 통신 오버헤드가 작지만, 파이프라인 버블(일부 GPU가 유휴 상태)이 발생할 수 있다. PCIe 연결 환경에서는 TP보다 PP가 더 효율적인 경우가 많다.

```bash
# TP=2, PP=2로 4-GPU 활용 (총 4 GPU)
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-70B-Instruct \
    --tensor-parallel-size 2 \
    --pipeline-parallel-size 2
```

### 4.3 Quantization (양자화)

모델 가중치를 낮은 정밀도로 변환하여 메모리 사용량과 연산량을 줄이는 기법이다.

**AWQ (Activation-aware Weight Quantization):** 활성화 분포를 기반으로 중요한 가중치 채널을 식별하고, 이를 보호하면서 4bit 양자화를 수행한다. 품질 저하가 적고 속도 향상이 크다.

**GPTQ:** Post-training 양자화 기법으로, 레이어별 최적화를 통해 양자화 오차를 최소화한다. AWQ와 유사한 성능을 보인다.

**FP8:** NVIDIA H100/H200에서 지원하는 8bit 부동소수점 형식. 양자화 오차가 매우 작으면서 약 2배의 메모리 절감과 성능 향상을 제공한다.

**INT4/INT8 W8A8:** 가중치(W)와 활성화(A)를 각각 양자화하는 방식. `--quantization` 플래그로 지정한다.

```bash
# AWQ 양자화 모델 서빙
python -m vllm.entrypoints.openai.api_server \
    --model TheBloke/Llama-2-70B-Chat-AWQ \
    --quantization awq \
    --dtype half

# FP8 양자화 (H100 이상)
python -m vllm.entrypoints.openai.api_server \
    --model neuralmagic/Meta-Llama-3.1-70B-Instruct-FP8 \
    --quantization fp8
```

### 4.4 Speculative Decoding (투기적 디코딩)

작은 Draft 모델이 여러 토큰을 빠르게 생성한 뒤, 큰 Target 모델이 한 번의 Forward Pass로 이를 검증하는 기법이다. Draft 모델의 예측이 맞으면 여러 토큰을 한 번에 확정하고, 틀리면 해당 위치에서 Target 모델의 결과를 사용한다. 출력 분포가 Target 모델과 수학적으로 동일하게 유지되므로 품질 저하가 없다.

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-70B-Instruct \
    --speculative-model meta-llama/Llama-3.2-1B-Instruct \
    --num-speculative-tokens 5 \
    --speculative-draft-tensor-parallel-size 1
```

투기적 디코딩은 acceptance rate(수락률)에 따라 효과가 크게 달라진다. 코드 생성이나 번역처럼 예측 가능한 패턴이 많은 태스크에서 효과적이고, 창의적 글쓰기처럼 분포가 넓은 태스크에서는 효과가 제한적이다.

### 4.5 Prefix Caching (Automatic Prefix Caching)

동일한 시스템 프롬프트나 Few-shot 예시를 여러 요청이 공유하는 경우, 해당 Prefix의 KV Cache를 캐싱하여 재사용하는 기법이다. 첫 번째 요청에서 계산된 Prefix KV Cache를 해시 기반으로 저장하고, 동일한 Prefix를 가진 후속 요청은 캐시를 재사용하여 Prefill을 건너뛴다.

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --enable-prefix-caching
```

RAG 파이프라인이나 Chatbot처럼 긴 시스템 프롬프트가 반복되는 워크로드에서 TTFT(Time To First Token)를 크게 단축한다.

### 4.6 Multi-LoRA 서빙

하나의 Base 모델 위에 여러 LoRA 어댑터를 동시에 서빙하는 기능이다. 요청마다 다른 LoRA를 지정할 수 있어, 멀티테넌트 환경에서 모델을 여러 개 띄우지 않고도 고객별 커스텀 모델을 서빙할 수 있다.

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --enable-lora \
    --lora-modules customer-a=/path/to/lora-a customer-b=/path/to/lora-b \
    --max-loras 4 \
    --max-lora-rank 64
```

API 호출 시 `model` 필드에 LoRA 이름을 지정한다:

```python
import openai
client = openai.OpenAI(base_url="http://localhost:8000/v1", api_key="dummy")

response = client.chat.completions.create(
    model="customer-a",  # LoRA 어댑터 이름
    messages=[{"role": "user", "content": "Hello!"}]
)
```

---

## Part 5. 실전 활용 가이드

### 5.1 설치

```bash
# pip 설치 (CUDA 12.1 기준)
pip install vllm

# 특정 CUDA 버전
pip install vllm --extra-index-url https://download.pytorch.org/whl/cu121

# 소스 빌드 (커스텀 CUDA 커널 수정 시)
git clone https://github.com/vllm-project/vllm.git
cd vllm
pip install -e .
```

### 5.2 기본 오프라인 추론 (Python API)

서버 없이 Python 스크립트에서 직접 추론을 수행하는 방법이다.

```python
from vllm import LLM, SamplingParams

# 모델 로드
llm = LLM(
    model="meta-llama/Llama-3.1-8B-Instruct",
    dtype="auto",                    # 자동 dtype 선택
    gpu_memory_utilization=0.90,     # GPU 메모리 90% 사용
    max_model_len=8192,              # 최대 컨텍스트 길이
    trust_remote_code=True,          # 커스텀 모델 코드 허용
)

# 샘플링 파라미터 설정
sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.9,
    top_k=50,
    max_tokens=1024,
    repetition_penalty=1.1,
    stop=["<|eot_id|>"],             # 종료 토큰
)

# 배치 추론
prompts = [
    "Explain the concept of PagedAttention in simple terms.",
    "Write a Python function that implements binary search.",
    "What are the benefits of using vLLM for LLM serving?",
]

outputs = llm.generate(prompts, sampling_params)

for output in outputs:
    prompt = output.prompt
    generated = output.outputs[0].text
    print(f"Prompt: {prompt[:50]}...")
    print(f"Output: {generated[:200]}...")
    print("---")
```

### 5.3 OpenAI 호환 API 서버

vLLM의 가장 일반적인 사용 방식이다.

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --host 0.0.0.0 \
    --port 8000 \
    --api-key "your-secret-key" \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.90 \
    --enable-prefix-caching \
    --disable-log-requests         # 프로덕션에서 로깅 비활성화
```

**지원하는 OpenAI 호환 엔드포인트:**
- `POST /v1/chat/completions` — 채팅 완성
- `POST /v1/completions` — 텍스트 완성
- `POST /v1/embeddings` — 임베딩 생성
- `GET /v1/models` — 사용 가능한 모델 목록
- `GET /health` — 서버 상태 확인

**클라이언트 호출 예시:**

```python
import openai

client = openai.OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="your-secret-key"
)

# 스트리밍 응답
stream = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain Kubernetes in 3 sentences."}
    ],
    stream=True,
    temperature=0.7,
    max_tokens=512,
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### 5.4 Structured Output (Guided Decoding)

JSON Schema나 정규식에 맞는 출력을 강제하는 기능이다. 프로덕션에서 LLM 출력을 안정적으로 파싱해야 할 때 필수적이다.

```python
from openai import OpenAI
import json

client = OpenAI(base_url="http://localhost:8000/v1", api_key="dummy")

# JSON Schema 기반 출력 강제
response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[
        {"role": "user", "content": "Extract info: John is 30 years old and lives in Seoul."}
    ],
    extra_body={
        "guided_json": json.dumps({
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "age": {"type": "integer"},
                "city": {"type": "string"}
            },
            "required": ["name", "age", "city"]
        })
    }
)

result = json.loads(response.choices[0].message.content)
# {"name": "John", "age": 30, "city": "Seoul"}
```

### 5.5 Docker 배포

```dockerfile
# Dockerfile
FROM vllm/vllm-openai:latest

# 모델을 이미지에 포함시키려면 (선택사항)
# RUN python -c "from huggingface_hub import snapshot_download; \
#     snapshot_download('meta-llama/Llama-3.1-8B-Instruct', local_dir='/models/llama')"

ENV VLLM_API_KEY=your-secret-key

ENTRYPOINT ["python", "-m", "vllm.entrypoints.openai.api_server"]
CMD ["--model", "meta-llama/Llama-3.1-8B-Instruct", \
     "--host", "0.0.0.0", \
     "--port", "8000", \
     "--gpu-memory-utilization", "0.90"]
```

```bash
docker run --gpus all -p 8000:8000 \
    -v /path/to/models:/models \
    -e HF_TOKEN=your-hf-token \
    vllm/vllm-openai:latest \
    --model /models/Llama-3.1-8B-Instruct \
    --host 0.0.0.0 --port 8000
```

---

## Part 6. 프로덕션 배포 아키텍처

### 6.1 Kubernetes 배포

```yaml
# vllm-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-server
  labels:
    app: vllm
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vllm
  template:
    metadata:
      labels:
        app: vllm
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        args:
        - "--model"
        - "meta-llama/Llama-3.1-8B-Instruct"
        - "--host"
        - "0.0.0.0"
        - "--port"
        - "8000"
        - "--gpu-memory-utilization"
        - "0.90"
        - "--enable-prefix-caching"
        - "--max-model-len"
        - "8192"
        ports:
        - containerPort: 8000
        resources:
          limits:
            nvidia.com/gpu: 1
          requests:
            nvidia.com/gpu: 1
            memory: "32Gi"
            cpu: "8"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 120
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 180
          periodSeconds: 30
        env:
        - name: HF_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-secret
              key: token
        - name: VLLM_API_KEY
          valueFrom:
            secretKeyRef:
              name: vllm-secret
              key: api-key
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      nodeSelector:
        gpu-type: a100
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-service
spec:
  selector:
    app: vllm
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vllm-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
spec:
  rules:
  - host: llm.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vllm-service
            port:
              number: 80
```

### 6.2 로드밸런싱 전략

LLM 서빙에서 로드밸런싱은 일반 웹 서비스와 다른 고려사항이 있다. 요청마다 처리 시간이 크게 다르고(짧은 질문 vs 긴 생성), 스트리밍 연결이 오래 유지되기 때문이다.

**권장 전략:**
- Round-Robin 대신 **Least Connections** 방식을 사용한다. 현재 처리 중인 요청이 가장 적은 인스턴스에 새 요청을 라우팅한다.
- Prefix Caching 효과를 극대화하려면 **Session Affinity (Sticky Session)**를 적용하여 동일 사용자의 요청이 같은 인스턴스로 가도록 한다.
- 각 인스턴스의 `/health` 엔드포인트와 GPU 메모리 사용량을 모니터링하여 오버로드를 방지한다.

### 6.3 오토스케일링

```yaml
# vllm-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm-server
  minReplicas: 1
  maxReplicas: 8
  metrics:
  - type: Pods
    pods:
      metric:
        name: vllm_num_requests_running
      target:
        type: AverageValue
        averageValue: "20"    # 인스턴스당 평균 동시 요청 20개 초과 시 스케일아웃
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 120
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 300
```

GPU 인스턴스는 프로비저닝 시간이 길므로, scaleDown의 stabilizationWindow를 넉넉하게(5분 이상) 설정하는 것이 중요하다.

### 6.4 모니터링 — Prometheus 메트릭

vLLM은 Prometheus 호환 메트릭을 기본 제공한다.

```bash
# 메트릭 엔드포인트 활성화
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --disable-log-requests \
    --served-model-name llama-3.1-8b
```

**핵심 메트릭들:**

| 메트릭 | 설명 | 알림 기준 (예시) |
|---|---|---|
| `vllm:num_requests_running` | 현재 실행 중인 요청 수 | > 배치 임계값의 80% |
| `vllm:num_requests_waiting` | 대기 중인 요청 수 | > 50 (장기 지속 시) |
| `vllm:gpu_cache_usage_perc` | GPU KV Cache 사용률 | > 95% |
| `vllm:cpu_cache_usage_perc` | CPU 스왑 캐시 사용률 | > 80% |
| `vllm:time_to_first_token_seconds` | TTFT (P50/P95/P99) | P99 > 2초 |
| `vllm:time_per_output_token_seconds` | TPOT (P50/P95/P99) | P95 > 100ms |
| `vllm:e2e_request_latency_seconds` | 전체 요청 지연시간 | P99 > 30초 |
| `vllm:num_preemptions_total` | Preemption(선점) 횟수 | 급격한 증가 시 |

**Grafana 대시보드 구성 권장 패널:**
- 처리량 (requests/sec) 시계열
- TTFT / TPOT 히스토그램 (P50, P95, P99)
- GPU KV Cache 사용률 시계열
- Running / Waiting / Swapped 큐 크기
- 토큰 생성 속도 (tokens/sec)

---

## Part 7. 성능 튜닝 가이드

### 7.1 핵심 파라미터 튜닝 체크리스트

**`--gpu-memory-utilization`** (기본값: 0.90): KV Cache에 할당할 GPU 메모리 비율이다. 높을수록 더 많은 동시 요청을 처리할 수 있지만, OOM 위험이 증가한다. 0.85~0.95 사이에서 조정하되, OOM이 발생하면 낮춘다.

**`--max-model-len`**: 지원할 최대 시퀀스 길이(프롬프트 + 생성)이다. 모델의 최대 컨텍스트보다 작게 설정하면 메모리를 절약하여 동시 처리량이 늘어난다. 실제 워크로드의 최대 길이에 맞게 설정한다.

**`--max-num-seqs`** (기본값: 256): 동시에 처리할 수 있는 최대 시퀀스 수이다. 너무 높으면 메모리 부족, 너무 낮으면 GPU 활용률 저하로 이어진다.

**`--max-num-batched-tokens`**: 한 iteration에서 처리할 최대 토큰 수이다. Chunked Prefill과 함께 사용하여 Prefill과 Decode의 균형을 조절한다.

**`--dtype`**: 모델 가중치의 데이터 타입이다. `auto`가 기본값이며, BF16 지원 GPU(A100, H100)에서는 `bfloat16`이 선택된다. `float16`은 V100 등 구세대 GPU에서 사용한다.

**`--enforce-eager`**: PyTorch의 CUDA Graph 캡처를 비활성화한다. 디버깅 시 유용하지만, 프로덕션에서는 CUDA Graph를 활성화한 상태가 성능이 더 좋다. 기본적으로 vLLM은 CUDA Graph를 사용한다.

### 7.2 워크로드별 최적화 가이드

**짧은 입력, 짧은 출력 (Chatbot Q&A 스타일):**
- `--enable-prefix-caching`을 활성화하여 시스템 프롬프트를 캐싱한다.
- `--max-model-len`을 실제 필요한 최소한으로 줄인다(예: 4096).
- `--max-num-seqs`를 높여 동시 처리량을 극대화한다.

**긴 입력, 짧은 출력 (RAG, 문서 요약):**
- `--enable-chunked-prefill`로 긴 프롬프트의 Prefill이 다른 요청을 블로킹하지 않도록 한다.
- `--enable-prefix-caching`으로 공통 문서 청크를 캐싱한다.
- `--max-num-batched-tokens`을 적절히 설정한다.

**긴 입력, 긴 출력 (코드 생성, 장문 작성):**
- `--gpu-memory-utilization`을 높인다(0.92~0.95).
- 동시 요청 수(`--max-num-seqs`)를 적당히 낮춘다.
- Speculative Decoding을 고려한다(코드 생성에서 특히 효과적).

**배치 추론 (오프라인):**
- Python API(`LLM` 클래스)를 직접 사용한다.
- 입력 길이별로 정렬하여 배치 효율을 높인다.
- `--swap-space`를 넉넉히 설정하여 CPU 스왑을 활용한다.

### 7.3 벤치마킹

vLLM은 자체 벤치마크 스크립트를 제공한다.

```bash
# 처리량 벤치마크
python benchmarks/benchmark_throughput.py \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --input-len 512 \
    --output-len 128 \
    --num-prompts 1000 \
    --backend vllm

# 서빙 벤치마크 (지연시간 + 처리량)
python benchmarks/benchmark_serving.py \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --dataset-name sharegpt \
    --request-rate 10 \
    --num-prompts 500 \
    --backend openai-chat \
    --base-url http://localhost:8000
```

핵심 측정 지표는 다음과 같다. Throughput은 초당 처리하는 총 토큰 수(tokens/sec)이다. TTFT(Time to First Token)는 요청 후 첫 토큰이 생성되기까지의 시간이다. TPOT(Time per Output Token)는 출력 토큰 하나를 생성하는 데 걸리는 평균 시간이다. ITL(Inter-Token Latency)은 연속된 두 출력 토큰 사이의 시간 간격이다.

---

## Part 8. vLLM vs 경쟁 프레임워크 비교

### 8.1 주요 프레임워크 비교

**vLLM vs TGI (Text Generation Inference, Hugging Face):** TGI도 Continuous Batching과 FlashAttention을 지원하지만, PagedAttention 수준의 메모리 최적화는 부재하다. vLLM이 일반적으로 처리량에서 우위를 보인다. TGI는 Hugging Face 생태계와의 통합이 강점이다.

**vLLM vs TensorRT-LLM (NVIDIA):** TensorRT-LLM은 NVIDIA GPU에 최적화된 컴파일러 기반 접근으로, 단일 요청 지연시간에서 가장 빠를 수 있다. 그러나 설정 복잡도가 높고, 모델 변환 과정이 필요하며, 새 모델 지원이 느리다. vLLM은 범용성과 사용 편의성에서 우위에 있다.

**vLLM vs SGLang:** SGLang은 복잡한 LLM 프로그램(멀티턴, 분기, 함수 호출 등)에 최적화된 프레임워크로, RadixAttention을 통해 트리 구조 Prefix 공유를 지원한다. 단순 서빙에서는 vLLM과 유사한 성능을 보이며, 복잡한 LLM 파이프라인에서는 SGLang이 더 효율적일 수 있다.

**vLLM vs Ollama:** Ollama는 로컬 LLM 실행에 특화된 도구로, 설치와 사용이 매우 간편하다. 그러나 프로덕션 서빙, 멀티 GPU, 고급 최적화 기능은 부재하다. 개발/테스트용으로 Ollama, 프로덕션 서빙용으로 vLLM이 적합하다.

### 8.2 선택 기준 요약

프로덕션에서 높은 처리량과 OpenAI 호환 API가 필요하다면 vLLM을 선택한다. 최극한의 단일 요청 지연시간이 필요하고 NVIDIA GPU 전용이라면 TensorRT-LLM을 고려한다. 복잡한 LLM 체인/에이전트를 서빙해야 한다면 SGLang을 검토한다. Hugging Face 모델 허브와의 긴밀한 통합이 필요하다면 TGI도 좋은 선택이다.

---

## Part 9. 최신 동향과 로드맵 (2024~2025)

### 9.1 주요 최신 기능

**Disaggregated Prefill/Decode:** Prefill 전용 인스턴스와 Decode 전용 인스턴스를 분리하여, 각각의 특성에 최적화된 하드웨어를 할당하는 아키텍처이다. Prefill은 compute-bound이므로 GPU 연산 능력이 중요하고, Decode는 memory-bandwidth-bound이므로 HBM 대역폭이 중요하다.

**FlashAttention 통합:** vLLM은 PagedAttention의 커스텀 커널 외에도, FlashAttention-2/3와 FlashInfer 백엔드를 지원한다. `--attention-backend` 플래그로 선택할 수 있으며, 모델과 하드웨어에 따라 최적의 백엔드가 다르다.

**Multimodal 지원:** LLaVA, Qwen-VL, InternVL 등 비전-언어 모델의 서빙을 지원한다. 이미지와 텍스트를 함께 입력으로 받아 처리할 수 있다.

**V1 아키텍처 리팩토링:** vLLM 프로젝트는 내부 아키텍처를 대폭 리팩토링하여 확장성과 유지보수성을 개선하고 있다. 새로운 스케줄러, 실행 엔진, 그리고 더 모듈화된 구조를 도입하고 있다.

### 9.2 생태계와 통합

vLLM은 LangChain, LlamaIndex, Haystack, Ray Serve 등 주요 LLM 프레임워크 및 오케스트레이션 도구와 통합된다. OpenAI 호환 API 덕분에 기존 OpenAI SDK 기반 코드를 최소한의 수정으로 마이그레이션할 수 있다. KServe, Triton Inference Server, SkyPilot 등과도 연동되어 다양한 프로덕션 환경에 배포할 수 있다.

---

## Part 10. 트러블슈팅 가이드

### 10.1 자주 발생하는 문제와 해결

**OOM (Out of Memory) 에러:**
`--gpu-memory-utilization`을 낮춘다(0.85 → 0.80). `--max-model-len`을 줄인다. `--max-num-seqs`를 줄인다. 양자화(AWQ, GPTQ, FP8)를 적용한다. Tensor Parallelism으로 GPU를 추가한다.

**CUDA Graph 관련 에러:**
`--enforce-eager` 플래그로 CUDA Graph를 비활성화하여 디버깅한다. 이 플래그는 성능이 떨어지므로 문제 진단 후 제거한다.

**모델 로딩 실패:**
`--trust-remote-code` 플래그가 필요한지 확인한다. HF_TOKEN 환경변수가 설정되어 있는지 확인한다(게이티드 모델의 경우). 디스크 공간과 메모리가 충분한지 확인한다.

**Throughput이 예상보다 낮을 때:**
NVIDIA SMI로 GPU 활용률을 확인한다. `vllm:gpu_cache_usage_perc` 메트릭을 확인한다(KV Cache가 가득 차 있다면 메모리 부족). 요청 패턴을 분석하여 Prefill이 Decode를 과도하게 블로킹하지 않는지 확인한다. Prefix Caching 활성화 여부를 확인한다.

**높은 TTFT:**
Prefill 병목을 확인한다. Chunked Prefill을 활성화한다. 프롬프트 길이를 줄이거나, 시스템 프롬프트가 너무 긴 경우 Prefix Caching을 활성화한다.

### 10.2 로깅과 디버깅

```bash
# 상세 로깅 활성화
VLLM_LOGGING_LEVEL=DEBUG python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct

# 요청별 로깅 (디버깅 시만 사용)
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --log-requests

# 프로파일링
VLLM_TORCH_PROFILER_DIR=./profiles python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct
```

---

## 부록: 빠른 참조 — 주요 CLI 옵션

```text
모델 설정
  --model                       모델 이름 또는 경로
  --tokenizer                   별도 토크나이저 지정
  --dtype                       데이터 타입 (auto, float16, bfloat16, float32)
  --quantization                양자화 방식 (awq, gptq, fp8, squeezellm 등)
  --max-model-len               최대 시퀀스 길이
  --trust-remote-code           커스텀 모델 코드 허용

병렬화
  --tensor-parallel-size        Tensor Parallelism GPU 수
  --pipeline-parallel-size      Pipeline Parallelism 스테이지 수
  --distributed-executor-backend  분산 실행 백엔드 (ray, mp)

메모리 / 스케줄링
  --gpu-memory-utilization      GPU 메모리 사용 비율 (0.0~1.0)
  --max-num-seqs                최대 동시 시퀀스 수
  --max-num-batched-tokens      iteration당 최대 토큰 수
  --swap-space                  CPU 스왑 공간 (GB)
  --enable-prefix-caching       Prefix 캐싱 활성화
  --enable-chunked-prefill      Chunked Prefill 활성화

최적화
  --speculative-model           투기적 디코딩 Draft 모델
  --num-speculative-tokens      Draft 모델 생성 토큰 수
  --enable-lora                 LoRA 서빙 활성화
  --enforce-eager               CUDA Graph 비활성화 (디버깅)

서버
  --host                        바인드 호스트 (기본: localhost)
  --port                        바인드 포트 (기본: 8000)
  --api-key                     API 인증 키
  --served-model-name           API에 노출할 모델 이름
  --disable-log-requests        요청 로깅 비활성화
```
