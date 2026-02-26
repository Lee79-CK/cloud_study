# MLOps & LLMOps: 박사급 심화 가이드 + 빅테크 면접 완전 정복

> 대상: Senior Cloud / MLOps / LLMOps Engineer
> 언어: 한국어

---

# PART 1. MLOps 고급 개념 및 아키텍처

## 1.1 MLOps의 이론적 기반: 소프트웨어 공학과의 교차점

MLOps는 단순히 "ML + DevOps"의 합성어가 아니라, **재현 가능성(Reproducibility)**, **관측 가능성(Observability)**, **지속적 학습(Continual Learning)**이라는 세 가지 핵심 공리 위에 구축된 공학 분야다. 전통적인 소프트웨어 시스템은 코드가 동일하면 출력이 동일하지만, ML 시스템은 코드, 데이터, 하이퍼파라미터, 환경, 하드웨어 시드(seed)까지 모두 동일해야 비로소 재현 가능하다. 이 다차원적 의존성이 MLOps를 DevOps보다 근본적으로 복잡하게 만드는 이유다.

구글 리서치의 "Hidden Technical Debt in Machine Learning Systems (NIPS 2015)" 논문은 ML 시스템에서 실제 모델 코드가 전체 코드베이스에서 차지하는 비중이 5% 미만임을 밝혔다. 나머지 95%는 데이터 수집, 특성 추출, 프로세스 관리, 서빙 인프라 등 주변 코드(Glue Code)로 구성된다. MLOps는 이 95%를 체계화하는 학문이다.

MLOps의 성숙도는 구글이 제안한 Level 0 ~ Level 2의 스펙트럼으로 분류된다. Level 0은 수동 프로세스로 데이터 사이언티스트가 직접 스크립트를 실행하는 단계이고, Level 1은 ML 파이프라인 자동화를 통해 모델을 자동으로 재학습하는 단계이며, Level 2는 CI/CD 파이프라인 자동화까지 포함해 새로운 아이디어를 빠르게 프로덕션에 배포하는 단계다. 대부분의 기업이 현실적으로 Level 1과 Level 2 사이에 위치한다.

## 1.2 Feature Store: 특성의 일관성과 재사용성

Feature Store는 MLOps 아키텍처의 핵심 구성 요소로, **학습 시점과 서빙 시점 간의 특성 불일치(Training-Serving Skew)** 문제를 해결하기 위해 탄생했다. 이 문제는 모델을 학습할 때 사용된 특성 계산 로직과 실시간 추론 시점에 사용되는 로직이 달라져서 발생하는 미묘하지만 치명적인 버그다.

Feature Store의 이중 저장소 아키텍처를 이해하는 것이 중요하다. **Online Store**(Redis, DynamoDB, Bigtable 등)는 저지연 실시간 조회를 위해 최근 특성값을 저장하고, **Offline Store**(S3, GCS, HDFS 등의 Parquet/Delta Lake)는 학습용 히스토리 데이터를 시계열로 저장한다. 핵심은 두 저장소가 **동일한 특성 정의(Feature Definition)**를 공유하며, Point-in-time Correctness를 보장한다는 점이다. Point-in-time Correctness란 학습 데이터를 생성할 때 미래 데이터가 과거 레이블에 누출(Leakage)되지 않도록 타임스탬프 기반으로 정확한 시점의 특성값만 가져오는 개념이다.

주요 Feature Store 구현체로는 오픈소스인 Feast, 클라우드 네이티브인 AWS SageMaker Feature Store, Vertex AI Feature Store, 그리고 Databricks Feature Store가 있다. Uber의 Michelangelo, LinkedIn의 Feathr, Airbnb의 Zipline은 대규모 내부 Feature Store의 선구자적 구현 사례다.

## 1.3 데이터 버전 관리와 데이터 중심 AI

전통적인 버전 관리가 코드에 초점을 맞춘다면, **데이터 중심 AI(Data-Centric AI)** 패러다임은 모델 아키텍처를 고정하고 데이터 품질과 일관성을 개선하는 것이 더 높은 성능 향상을 가져온다는 관점이다. Andrew Ng이 주창한 이 접근법은 특히 소규모 데이터셋이나 도메인 특화 문제에서 강력한 효과를 보인다.

DVC(Data Version Control)는 Git과 연동하여 데이터셋과 모델 파일을 추적하는 오픈소스 도구다. DVC는 실제 대용량 파일을 Git에 저장하는 대신, 포인터 파일(.dvc 파일)만 Git에 커밋하고 실제 데이터는 S3, GCS, Azure Blob 등의 원격 저장소에 보관한다. Delta Lake와 Apache Iceberg는 ACID 트랜잭션, 타임 트래블(Time Travel), 스키마 진화(Schema Evolution)를 지원하는 오픈 테이블 포맷으로, 대규모 데이터 파이프라인에서 데이터 버전 관리의 사실상 표준이 되고 있다.

## 1.4 모델 모니터링: 드리프트의 분류학

모델 성능 저하의 근본 원인을 정확히 진단하기 위해서는 드리프트(Drift)의 세 가지 유형을 구분해야 한다.

**Data Drift(Covariate Shift)**는 입력 특성의 분포 P(X)가 변하지만 조건부 분포 P(Y|X)는 유지되는 경우다. 예를 들어 금융 사기 탐지 모델에서 사용자 연령 분포가 젊어지는 것이 이에 해당한다. 이를 감지하기 위해 Population Stability Index(PSI), Kolmogorov-Smirnov 테스트, Jensen-Shannon Divergence 등의 통계 검정을 사용한다.

**Concept Drift**는 P(Y|X)가 변하는 경우로, 세상의 패턴 자체가 변한 것이다. COVID-19 팬데믹 이후 소비자 구매 패턴이 급변한 것이 대표적 예시다. Concept Drift는 Data Drift보다 탐지하기 어렵고, 모델 예측값 분포의 변화나 A/B 테스트 결과 악화를 통해 간접적으로 감지하는 경우가 많다.

**Label Drift**는 레이블 분포 P(Y)가 변하는 것으로, 클래스 불균형이 시간에 따라 변화하는 상황이다.

Evidently AI, WhyLabs, Arize AI 등이 이러한 드리프트 모니터링을 전문으로 하는 플랫폼이다. 대규모 시스템에서는 이러한 모니터링 신호를 트리거로 삼아 자동 재학습(Auto Retraining) 파이프라인을 구동하는 것이 Level 2 MLOps의 핵심 패턴이다.

## 1.5 분산 학습 심화: 병렬화 전략의 이론

대규모 모델 학습에서 병렬화 전략은 단순히 GPU를 늘리는 것 이상의 정교한 수학적 설계를 요구한다.

**데이터 병렬화(Data Parallelism)**는 가장 단순한 형태로, 각 GPU가 동일한 모델의 복사본을 가지고 미니배치의 서로 다른 부분을 처리한다. AllReduce 알고리즘(Ring-AllReduce, Tree-AllReduce 등)을 통해 그라디언트를 동기화한다. PyTorch의 DDP(DistributedDataParallel)가 이 방식의 표준 구현이며, 통신 오버헤드를 최소화하기 위해 그라디언트 압축(Gradient Compression), Gradient Checkpointing, Mixed Precision Training(FP16/BF16)을 함께 사용한다.

**모델 병렬화(Model Parallelism)**는 모델 자체가 단일 GPU 메모리에 들어가지 않을 때 사용한다. 레이어를 여러 GPU에 분산시키는 **Tensor Parallelism**(Megatron-LM 방식)과 서로 다른 레이어 그룹을 서로 다른 GPU에 배치하는 **Pipeline Parallelism**으로 나뉜다. Pipeline Parallelism은 파이프라인 버블(Bubble)이라는 비효율을 발생시키는데, GPipe와 PipeDream 알고리즘은 이를 Micro-batching과 비동기적 그라디언트 업데이트로 완화한다.

**ZeRO(Zero Redundancy Optimizer)**는 Microsoft DeepSpeed가 제안한 혁신적 접근법으로, Optimizer State(ZeRO-1), Gradient(ZeRO-2), Parameter(ZeRO-3)를 데이터 병렬화 워커들에게 분산 저장하여 각 GPU의 메모리 사용량을 획기적으로 줄인다. ZeRO-3는 이론상 무한한 모델 크기를 단일 GPU 메모리 제약 없이 학습 가능하게 한다.

**3D Parallelism**은 Data Parallelism, Tensor Parallelism, Pipeline Parallelism을 동시에 적용하는 방식으로, Megatron-Turing NLG, GPT-4 등 초대형 모델 학습에 사용된 것으로 알려진 기법이다.

---

# PART 2. LLMOps 고급 개념 및 아키텍처

## 2.1 LLMOps의 등장 배경과 MLOps와의 차별성

LLMOps는 MLOps의 연장선이지만, LLM 특유의 **규모(Scale)**, **프롬프트 의존성(Prompt Dependency)**, **생성적 불확실성(Generative Uncertainty)**, **지식 한계(Knowledge Cutoff)** 라는 네 가지 특성으로 인해 별도의 공학 체계가 필요하다.

전통적인 MLOps에서 모델 업데이트는 재학습(Retraining)을 의미했지만, LLMOps에서는 프롬프트 엔지니어링, Few-shot Learning, RAG 업데이트, Fine-tuning이라는 다양한 계층의 업데이트 전략이 존재한다. 이러한 업데이트 전략들은 비용, 속도, 효과성 측면에서 서로 다른 트레이드오프를 가지며, LLMOps 엔지니어는 이 트레이드오프를 이해하고 상황에 맞는 전략을 선택해야 한다.

또한 LLM의 출력은 확률적이고 개방형(Open-ended)이기 때문에 전통적인 정확도(Accuracy) 기반 평가 메트릭이 불충분하다. LLMOps는 BLEU, ROUGE와 같은 n-gram 유사도부터 BERTScore, G-Eval과 같은 임베딩 기반 평가, 그리고 LLM-as-Judge 패턴에 이르기까지 다층적 평가 체계를 요구한다.

## 2.2 RAG(Retrieval-Augmented Generation) 아키텍처 심화

RAG는 LLM의 지식 한계와 환각(Hallucination) 문제를 해결하는 가장 실용적인 접근법으로, 단순한 검색 + 생성의 조합을 넘어 다양한 고급 변형이 존재한다.

**Naive RAG**는 쿼리를 임베딩하고, 벡터 DB에서 Top-K 청크를 검색하여 컨텍스트로 주입하는 기본 형태다. 이 방식의 한계는 청크 크기 선택의 민감성, 의미적 유사성이 높지 않은 청크 검색 실패, 긴 문서에서의 "Lost in the Middle" 현상(컨텍스트 중간 정보를 LLM이 무시하는 경향)이다.

**Advanced RAG**는 이러한 한계를 극복하기 위한 여러 기법을 포함한다. **Hybrid Search**는 벡터 검색(Semantic Search)과 키워드 검색(BM25 등 Sparse Retrieval)을 RRF(Reciprocal Rank Fusion) 알고리즘으로 결합한다. **Query Rewriting**은 HyDE(Hypothetical Document Embeddings) 등을 활용해 원본 쿼리를 변환하거나 확장하여 검색 품질을 높인다. **Re-ranking**은 초기 검색된 청크를 Cross-encoder 모델(Cohere Rerank, BGE-Reranker 등)로 재순위화하여 관련성이 높은 청크를 상위로 올린다. **Contextual Compression**은 검색된 청크에서 쿼리와 관련된 부분만 추출하여 컨텍스트 윈도우를 효율적으로 사용한다.

**Modular RAG**는 검색, 생성, 평가 단계를 독립적인 모듈로 구성하고 라우팅(Routing), 반복(Iterative) 검색, 자가 반성(Self-reflection) 등을 유연하게 조합하는 아키텍처다. Self-RAG는 LLM이 검색 필요 여부를 스스로 판단하고, 검색 결과의 관련성과 생성 출력의 사실성을 자가 비판하는 특수 토큰(특수 Reflection Token)을 활용한다.

**GraphRAG(Microsoft)**는 문서를 단순 청크로 분리하는 대신, 개체(Entity)와 관계(Relation)를 추출하여 지식 그래프(Knowledge Graph)를 구축한다. 이 그래프 위에서 커뮤니티 탐지 알고리즘을 실행하여 계층적 요약을 생성하고, 글로벌 쿼리(전체 문서에 걸친 추상적 질문)에 강한 답변을 생성한다. 반면 기존 RAG는 로컬 쿼리(특정 사실 검색)에 더 적합하다.

## 2.3 벡터 데이터베이스 심화: ANN 알고리즘과 선택 기준

벡터 DB의 핵심 성능은 ANN(Approximate Nearest Neighbor) 알고리즘의 선택에 달려 있다.

**HNSW(Hierarchical Navigable Small World)**는 계층적 그래프 구조를 구축하여 상위 레이어에서 빠르게 후보를 좁히고 하위 레이어에서 정밀 탐색을 수행한다. O(log N)의 쿼리 복잡도를 가지며, 메모리 사용량이 높은 대신 매우 빠른 검색 속도를 제공한다. Qdrant, Weaviate, pgvector가 주로 사용한다.

**IVF(Inverted File Index)**는 벡터 공간을 Voronoi 셀로 분할하고, 쿼리 시 가장 가까운 K개의 셀만 탐색한다. 메모리 효율이 좋고 대규모 데이터셋에 적합하지만, 클러스터 경계 근처의 벡터를 놓칠 수 있는 리콜 손실이 있다. Faiss의 핵심 인덱스 유형이다.

**ScaNN(Google)**은 Anisotropic Vector Quantization을 사용하여 내적 연산에 최적화된 방식으로 벡터를 압축한다. Google의 내부 검색 시스템에 사용되며, 최신 벤치마크에서 최고 수준의 QPS/Recall 트레이드오프를 보인다.

벡터 DB 선택 시 고려해야 할 차원은 크게 다섯 가지다. 첫째, 데이터 규모(수백만 vs 수십억 벡터), 둘째, 일관성 모델(강한 일관성 vs 최종 일관성), 셋째, 필터링 지원(메타데이터 기반 프리-필터 vs 포스트-필터의 성능 차이), 넷째, 멀티 테넌시 지원(네임스페이스/컬렉션 격리), 다섯째, 클라우드 네이티브 통합(관리형 서비스 제공 여부)이다.

## 2.4 LLM Fine-tuning 전략: PEFT의 수학적 이해

전체 Fine-tuning은 수십억 개의 파라미터를 업데이트해야 하므로 대부분의 조직에서 비실용적이다. **PEFT(Parameter-Efficient Fine-Tuning)**는 전체 파라미터의 극히 일부만 업데이트하면서도 Full Fine-tuning에 근접한 성능을 달성하는 방법론의 총칭이다.

**LoRA(Low-Rank Adaptation)**의 핵심 아이디어는 LLM의 가중치 행렬 W ∈ ℝ^(d×k)에 대한 업데이트 ΔW를 저순위(Low-Rank) 분해 ΔW = BA로 근사하는 것이다(B ∈ ℝ^(d×r), A ∈ ℝ^(r×k), r << min(d,k)). 전향 연산은 h = Wx + BAx/α로 수행되며, 추론 시에는 W' = W + BA를 병합하여 지연 시간 오버헤드가 없다. 업데이트 파라미터 수는 r(d+k)로, r=8이면 전체 파라미터의 0.1% 수준이다.

**QLoRA**는 4bit NormalFloat(NF4) 양자화로 기본 모델을 압축하고, LoRA 어댑터는 BF16으로 학습한다. Double Quantization(양자화 상수 자체를 양자화)과 Paged Optimizers(GPU OOM 방지를 위한 CPU 오프로딩)를 결합하여 단일 48GB GPU로 65B 모델을 Fine-tuning 가능하게 한 혁신적 기법이다.

**DoRA(Weight-Decomposition Low-Rank Adaptation)**는 가중치를 크기(Magnitude)와 방향(Direction) 성분으로 분해하여, LoRA가 방향 성분만 업데이트하는 반면 DoRA는 두 성분을 독립적으로 업데이트함으로써 LoRA의 표현력 한계를 극복한다.

Fine-tuning 데이터 준비에서 가장 중요한 원칙은 **품질 > 양(Quality over Quantity)**이다. Alpaca, LIMA 등의 연구는 수천 개의 고품질 예시가 수십만 개의 저품질 데이터보다 뛰어난 결과를 가져올 수 있음을 보였다.

## 2.5 LLM 서빙 최적화: 추론 효율화 기법

LLM 추론은 메모리 대역폭에 병목이 걸리는(Memory-Bandwidth Bound) 작업이다. 이를 최적화하는 기법들은 크게 세 범주로 나뉜다.

**KV Cache 관리**에서 PagedAttention(vLLM)은 OS의 가상 메모리 페이징 개념을 KV Cache에 적용하여, 서로 다른 요청들이 KV Cache 메모리를 유연하게 공유(Prefix Sharing)할 수 있게 한다. 이를 통해 GPU 메모리 단편화를 제거하고 처리량을 최대 24배 향상시킨다.

**Speculative Decoding**은 소형 Draft 모델(예: GPT-2)이 여러 토큰을 빠르게 생성하고, 대형 Target 모델이 이를 병렬로 검증하는 방식이다. Target 모델의 자기회귀적 순차 생성을 부분적으로 병렬화하여 지연 시간을 줄인다. 수락 확률을 높이기 위한 Tree-based Speculative Decoding, Medusa(다중 헤드 병렬 드래프팅)등의 변형이 있다.

**Continuous Batching**은 전통적인 Static Batching이 가장 긴 시퀀스가 완료될 때까지 배치 전체를 대기시키는 비효율을 해결한다. 각 요청이 생성을 완료하는 즉시 새 요청을 배치에 삽입하여 GPU 활용률을 극대화한다. TGI(Text Generation Inference), vLLM 모두 지원한다.

**양자화(Quantization)**는 가중치와 활성화값을 낮은 비트로 표현하는 기법이다. GPTQ(Post-Training Quantization, 역헤시안 기반 가중치 근사), AWQ(Activation-aware Weight Quantization, 채널별 현저성 가중치 스케일링), GGUF(llama.cpp 생태계)가 대표적이다. FP8 학습 및 추론은 NVIDIA H100 이상에서 하드웨어 지원이 제공되어 BF16 대비 2배의 처리량 향상이 가능하다.

## 2.6 LLM 평가 체계: 다차원 품질 측정

LLM 평가는 단일 메트릭으로 완성되지 않으며, 목적에 따라 다층적으로 설계해야 한다.

**기능적 정확성** 평가는 코딩 태스크의 경우 HumanEval, MBPP 등의 unit test 통과율, 수학 추론의 경우 GSM8K, MATH 벤치마크, 일반 지식의 경우 MMLU, HellaSwag, ARC를 활용한다.

**RAG 파이프라인 평가** 프레임워크인 RAGAS는 네 가지 차원을 측정한다. Faithfulness(생성된 답변이 검색된 컨텍스트에 근거하는 정도), Answer Relevancy(답변이 질문과 관련된 정도), Context Recall(관련 정보가 검색된 정도), Context Precision(검색된 청크 중 실제 유용한 것의 비율)이다.

**LLM-as-Judge** 패턴은 GPT-4나 Claude와 같은 강력한 LLM을 평가자로 활용한다. G-Eval은 체인 오브 쏘트(Chain-of-Thought)를 활용한 평가 프레임워크이며, MT-Bench는 멀티턴 대화 능력을 측정한다. 이 방식의 한계는 평가 LLM 자체의 편향(자기 출력 선호, 포지션 편향, 장문 선호 등)이 결과에 영향을 줄 수 있다는 점이다. 이를 완화하기 위해 평가 순서를 무작위화하고, 여러 독립적 평가자를 앙상블하는 것이 권장된다.

## 2.7 AI Agent 및 Multi-Agent 오케스트레이션

AI Agent는 LLM을 "뇌"로 사용하여 도구(Tool) 호출, 메모리 관리, 행동 계획을 반복적으로 수행하는 시스템이다. ReAct(Reasoning + Acting) 프레임워크는 추론(Thought) → 행동(Action) → 관찰(Observation)의 사이클을 반복하며, 이것이 현대 Agent 시스템의 기반이다.

**Multi-Agent 시스템**에서 주목할 아키텍처 패턴은 다음과 같다. **Supervisor-Worker** 패턴은 오케스트레이터 에이전트가 태스크를 전문화된 서브에이전트들에게 위임하는 계층 구조다. **Reflection** 패턴은 생성자(Generator) 에이전트와 비평자(Critic) 에이전트가 협력하여 출력 품질을 반복적으로 개선한다. **Debate** 패턴은 여러 에이전트가 서로 다른 관점을 주장하게 하여 더 강인한 추론을 유도한다.

LangGraph와 AutoGen은 이러한 Multi-Agent 워크플로우를 그래프 형태로 모델링하는 프레임워크다. LangGraph는 사이클(Cycle)을 허용하는 방향성 그래프(DAG 아님)로 복잡한 에이전트 루프를 표현하며, 상태 지속성(State Persistence), 스트리밍, Human-in-the-Loop 인터럽트를 기본 지원한다.

**MCP(Model Context Protocol)**는 Anthropic이 제안한 표준 프로토콜로, LLM 애플리케이션이 외부 도구 및 데이터 소스와 상호작용하는 방식을 표준화한다. 이를 통해 다양한 LLM과 도구 생태계 간의 상호운용성(Interoperability)을 높인다.

## 2.8 Prompt Management와 LLMOps 플랫폼

프롬프트는 LLM 시스템에서 코드만큼 중요한 아티팩트로 관리되어야 한다. **Prompt Versioning**은 Git처럼 프롬프트 변경 이력을 추적하고, A/B 테스트를 통해 프롬프트 변경의 효과를 측정하는 것을 의미한다. LangSmith, Langfuse, Helicone, Arize Phoenix 등의 플랫폼이 이러한 기능을 제공한다.

**비용 모니터링**은 LLMOps에서 전통 MLOps보다 훨씬 중요한 위치를 차지한다. 토큰 사용량, 모델별 비용, 사용자별 비용, 캐시 히트율 등을 추적해야 한다. **시맨틱 캐싱(Semantic Caching)**은 의미적으로 유사한 쿼리에 대해 LLM 호출 없이 캐시된 응답을 반환하여 비용을 절감한다. GPTCache, Redis + 벡터 유사도 검색이 이에 활용된다.

## 2.9 LLM 보안: Red Teaming과 가드레일

**Prompt Injection**은 악의적 사용자가 시스템 프롬프트를 무력화하거나 LLM에게 의도치 않은 행동을 유발하는 공격이다. Direct Injection은 사용자가 직접 시스템 프롬프트를 덮어쓰려는 시도이고, Indirect Injection은 LLM이 처리하는 외부 데이터(웹 페이지, 문서 등)에 악성 지시를 숨기는 공격이다.

방어 전략으로는 입력 샌드박싱, 컨텍스트 구분자(명확한 시스템/사용자 프롬프트 경계), 출력 검증(NeMo Guardrails, Llama Guard), 그리고 최소 권한 원칙(LLM 에이전트에 불필요한 도구 권한 부여 금지)이 있다.

OWASP Top 10 for LLM Applications는 LLM 보안의 표준 참고 문서로, Prompt Injection, Insecure Output Handling, Training Data Poisoning, Model Denial of Service 등 10가지 위협을 정의한다.

---

# PART 3. 최신 표준 아키텍처 패턴

## 3.1 LLM Gateway 패턴

프로덕션 LLM 시스템에서 LLM Gateway는 모든 LLM API 호출의 단일 진입점(Single Entry Point) 역할을 하며, 다음 기능들을 중앙화한다.

**모델 라우팅**: 요청의 복잡도, 비용, 지연 시간 요구사항에 따라 적절한 모델(GPT-4, Claude, Gemini, 사내 모델 등)로 요청을 라우팅한다. Fallback 로직을 통해 모델 장애 시 대체 모델로 자동 전환한다. **Rate Limiting & Cost Control**: 사용자/팀별 토큰 사용량 제한 및 비용 할당량을 관리한다. **시맨틱 캐싱**: 유사 쿼리를 캐싱하여 API 비용을 절감한다. **감사 로깅**: 모든 요청/응답을 로깅하여 규정 준수와 디버깅에 활용한다. **PII 감지 및 마스킹**: 민감 정보가 LLM API로 전송되기 전에 필터링한다.

## 3.2 온라인 학습(Continual Learning) 아키텍처

스트리밍 데이터 환경에서 모델을 지속적으로 업데이트하는 Continual Learning 시스템은 **Catastrophic Forgetting**이라는 근본적 도전에 직면한다. 새로운 데이터로 학습하면 이전에 학습한 지식을 잃는 현상이다.

이를 해결하는 아키텍처 패턴으로는 **Elastic Weight Consolidation(EWC)**이 이전 태스크에 중요한 파라미터의 변화를 제약하는 정규화 항을 추가한다. **Replay Buffer**는 과거 데이터의 대표 샘플을 보관하고 새 데이터와 함께 혼합 학습한다. **Progressive Neural Networks**는 새 태스크마다 새로운 모듈을 추가하고 기존 모듈을 동결한다.

## 3.3 Compound AI Systems와 DSPy

단일 LLM 호출로는 해결하기 어려운 복잡한 태스크를 여러 LLM 호출과 외부 도구를 조합한 복합 시스템(Compound AI System)으로 해결하는 패러다임이 대두되고 있다. LMSYS의 STORM, 알파코드 2, 딥마인드의 알파지오메트리 등이 이 접근법의 대표 사례다.

**DSPy**는 Stanford가 개발한 프레임워크로, LLM 파이프라인을 선언적(Declarative)으로 정의하고, 프롬프트와 Few-shot 예시를 자동으로 최적화(Compile)한다. 기존의 수동 프롬프트 엔지니어링을 자동화된 프로그래밍 패러다임으로 전환한다는 점에서 혁신적이다.

---

# PART 4. 빅테크 면접 Q&A 30+ 문항

---

## [Google / DeepMind 면접]

### Q1. Google의 대규모 ML 시스템에서 Training-Serving Skew를 어떻게 탐지하고 완화하는가?

**A:** Training-Serving Skew는 학습 시 특성 계산 로직과 서빙 시 특성 계산 로직이 달라 발생하는 조용한 버그다. 탐지 방법은 크게 두 레이어로 나뉜다.

통계적 탐지 레이어에서는 학습 데이터셋의 각 특성에 대한 통계 요약(분포, 분위수, 결측률)을 TFX의 TFDV(TensorFlow Data Validation)로 사전에 계산하고 저장한다. 서빙 시점에 동일한 통계를 샘플링하여 기준선과 비교하며, PSI 또는 KS 테스트로 분포 이탈을 감지한다.

아키텍처적 완화 전략은 더 근본적이다. 동일한 특성 변환 코드를 학습과 서빙에서 공유하도록 강제하는 것이 핵심인데, TFX의 Transform 컴포넌트는 이를 TensorFlow 그래프로 저장하여 학습과 서빙 모두에 동일한 SavedModel로 배포한다. Feature Store(Vertex AI Feature Store)를 도입하면 오프라인/온라인 스토어가 동일한 특성 정의를 공유하므로 스큐가 구조적으로 불가능해진다. Shadow Mode 배포를 통해 새 서빙 로직의 출력을 기존 학습 파이프라인 출력과 비교하여 출시 전에 스큐를 검증한다.

### Q2. Google의 Borg/Kubernetes 환경에서 ML 워크로드의 리소스 할당 전략과 선점(Preemption) 처리는 어떻게 하는가?

**A:** ML 학습 워크로드는 장시간 실행되는 특성상 Kubernetes 환경에서 특별한 리소스 관리 전략이 필요하다.

리소스 티어링 전략으로는 세 가지 QoS 클래스를 활용한다. 중요 학습 실험은 Guaranteed QoS(request = limit)로 설정하여 선점을 방지하고, 배치 학습 작업은 Burstable QoS로 설정하여 여유 자원을 활용하되 선점 가능하게 한다. 비교적 짧은 하이퍼파라미터 탐색 작업은 BestEffort로 설정하여 저렴한 스팟/프리엠프티블 VM을 최대한 활용한다.

선점에 대한 복원력은 체크포인팅으로 확보한다. Google의 Pathways와 같은 시스템은 체크포인트를 GCS에 주기적으로 저장하고, 선점 발생 시 최신 체크포인트에서 재시작한다. PodDisruptionBudget과 graceful termination period를 적절히 설정하여 체크포인트 저장 완료 후 노드가 종료되도록 보장한다. Volcano와 같은 Gang Scheduling 플러그인을 사용하면 분산 학습에 필요한 모든 워커가 동시에 스케줄링되는 것을 보장하여 부분 할당으로 인한 교착상태(Deadlock)를 방지한다.

### Q3. Transformer 모델의 Attention 메커니즘에서 FlashAttention이 기존 대비 어떤 개선을 가져오는가?

**A:** 표준 Attention 계산 `softmax(QK^T/√d)V`의 병목은 메모리 I/O에 있다. Q, K, V 행렬과 중간 결과인 N×N 어텐션 행렬(시퀀스 길이 N에 대해 O(N²) 메모리)을 HBM(High Bandwidth Memory)과 SRAM 사이에서 반복적으로 읽고 쓰는 I/O 비용이 계산 비용보다 지배적이다.

FlashAttention은 **타일링(Tiling)**과 **재계산(Recomputation)** 두 가지 핵심 아이디어로 이를 해결한다. Q, K, V 행렬을 SRAM에 들어가는 작은 블록으로 분할하고, 블록 단위로 Attention을 계산하여 HBM 접근을 최소화한다. Backward Pass에서는 중간 어텐션 행렬을 저장하지 않고 Forward Pass를 재계산하여 메모리 사용량을 O(N²)에서 O(N)으로 줄인다.

결과적으로 FlashAttention은 시퀀스 길이에 따른 메모리를 선형으로 줄이고, 실제 벽시계 시간(Wall-clock Time)을 2~4배 단축한다. FlashAttention-2는 워크 파티셔닝을 개선하여 GPU 활용률을 더욱 높였고, FlashAttention-3는 H100의 특수 하드웨어 기능(Tensor Core 파이프라이닝)을 활용한다.

---

## [Amazon / AWS 면접]

### Q4. SageMaker 아키텍처에서 모델 학습과 추론 비용 최적화를 어떻게 접근하는가?

**A:** AWS에서 ML 비용 최적화는 학습과 추론 두 단계로 나누어 접근한다.

학습 비용 최적화에서는 스팟 인스턴스(EC2 Spot Instances)를 SageMaker Managed Spot Training과 함께 활용하면 온디맨드 대비 최대 90%의 비용을 절감할 수 있다. 이를 위해 학습 코드는 체크포인팅을 S3에 주기적으로 수행하고, SageMaker가 제공하는 `checkpoint_s3_uri`와 `use_spot_instances=True` 설정을 활용한다. 또한 SageMaker Experiments를 통해 각 실험의 compute-hour 대비 메트릭 향상을 추적하여 ROI를 관리한다.

추론 비용 최적화는 트래픽 패턴에 따라 전략이 다르다. 예측 가능한 고정 트래픽에는 Provisioned Concurrency를 가진 SageMaker Real-time Endpoint가 적합하다. 간헐적 트래픽에는 SageMaker Serverless Inference가 콜드 스타트를 허용하면서 유휴 비용을 제거한다. 배치 예측에는 SageMaker Batch Transform이 인스턴스를 실행 기간에만 과금하여 가장 경제적이다. Multi-Model Endpoint는 여러 소형 모델을 단일 엔드포인트에 호스팅하여 인스턴스 비용을 분산한다. AWS Inferentia2와 Graviton3 기반 inf2/trn1 인스턴스는 GPU 대비 가격성능비를 크게 향상시킨다.

### Q5. Amazon이 실시간 추천 시스템에서 Feature Store와 온라인/오프라인 일관성을 어떻게 유지하는가?

**A:** Amazon의 추천 시스템 아키텍처는 이벤트 기반 아키텍처(Event-Driven Architecture) 위에 Feature Store를 구축한다.

사용자 행동 이벤트(클릭, 구매, 검색)는 Kinesis Data Streams로 실시간 스트리밍되며, Lambda 또는 Flink 애플리케이션이 스트림을 처리하여 실시간 특성(최근 30분간 클릭한 카테고리 등)을 계산한다. 이 실시간 특성은 SageMaker Feature Store의 온라인 스토어(내부적으로 DynamoDB 기반)에 낮은 레이턴시로 저장된다.

동시에 동일한 이벤트 스트림은 S3에 Parquet 형태로 저장되어 오프라인 특성 계산(과거 30일 구매 패턴 등)에 활용된다. 핵심 일관성 메커니즘은 Feature Group을 정의할 때 온라인/오프라인 스토어 모두 동일한 특성 변환 코드를 사용하도록 강제하는 것이다. SageMaker Feature Store는 `ingest_as_of_time`을 기록하여 학습 데이터셋 생성 시 Point-in-time Correctness를 보장한다.

### Q6. AWS에서 수천 개의 ML 모델을 운영하는 Multi-tenant MLOps 플랫폼을 설계한다면?

**A:** 수천 개 모델의 멀티 테넌트 운영은 격리, 효율성, 거버넌스 세 가지 축에서 설계해야 한다.

**격리 전략**으로 Hard Isolation이 필요한 테넌트(민감 데이터, 규정 준수 요구사항)에는 전용 AWS 계정과 VPC를 할당하고 AWS Organizations와 SCP로 거버넌스를 적용한다. 일반 테넌트는 동일 계정에서 IAM Role-based Access Control로 리소스 접근을 제한하고, SageMaker Studio Domain의 사용자별 EFS 격리를 활용한다.

**효율성 전략**으로 SageMaker Inference Recommender를 활용해 각 모델 유형에 최적의 인스턴스를 자동으로 선택한다. 유사한 모델들(동일 프레임워크, 유사 크기)은 Multi-Model Endpoint에 통합 호스팅한다. SageMaker Savings Plans를 통해 예측 가능한 워크로드에 1-3년 약정으로 최대 64% 절감한다.

**거버넌스**는 SageMaker Model Registry를 중앙 모델 카탈로그로 활용하고, 모든 모델에 메타데이터(학습 데이터 버전, 학습 메트릭, 승인 상태)를 태그로 부착한다. AWS Service Catalog를 통해 사전 승인된 ML 템플릿만 배포 가능하게 제한하여 조직 표준 준수를 강제한다.

---

## [Microsoft / Azure 면접]

### Q7. Azure ML에서 LLM Fine-tuning 파이프라인을 프로덕션 수준으로 설계하는 방법은?

**A:** Azure ML에서 LLM Fine-tuning 프로덕션 파이프라인은 다섯 단계로 구성된다.

**데이터 준비 단계**에서는 Azure Data Factory로 원시 데이터를 수집하고 Azure Blob Storage에 저장한다. Azure ML Dataset을 통해 데이터를 버전 관리하며, 데이터 품질 검증 컴포넌트(스키마 검사, 클래스 분포 검증, PII 스캔)를 파이프라인 초반에 배치한다.

**학습 단계**에서는 Azure ML Compute Cluster의 ND A100 v4 시리즈 노드에서 DeepSpeed + ZeRO-3를 활용한 분산 학습을 수행한다. Azure ML의 Environment 기능으로 Dockerfile을 버전 관리하고 ACR에 등록하여 재현 가능한 학습 환경을 확보한다. 학습 메트릭은 MLflow Autolog를 통해 Azure ML Workspace에 실시간 기록한다.

**평가 단계**에서는 도메인 특화 벤치마크(사내 평가 데이터셋), 안전성 평가(Azure AI Content Safety), 성능 회귀 테스트를 자동화한다. Azure ML Pipeline의 조건부 분기를 활용해 평가 메트릭이 임계값을 통과한 경우에만 Model Registry 등록을 진행한다.

**배포 단계**에서는 Azure ML Managed Online Endpoint의 Blue/Green 배포로 무중단 전환하고, Azure Monitor와 Application Insights를 통해 레이턴시, 처리량, 에러율을 모니터링한다. ONNX Runtime 또는 TensorRT를 활용한 모델 최적화로 Azure Kubernetes Service(AKS) 기반 추론 비용을 절감한다.

### Q8. Microsoft의 Copilot 스타일 LLM 애플리케이션에서 Prompt Injection 공격을 방어하는 아키텍처를 설명하라.

**A:** Prompt Injection 방어는 심층 방어(Defense in Depth) 전략으로 여러 레이어를 구성해야 한다.

**입력 레이어** 방어로는 Azure AI Content Safety의 PromptShield API를 활용하여 입력을 사전 스크리닝하고, 시스템 프롬프트와 사용자 입력 사이에 명확한 구분자와 컨텍스트 전환 방지 지시를 추가한다. 사용자 입력을 XML 태그나 코드 블록으로 래핑하여 LLM이 이를 데이터로 인식하도록 유도한다.

**처리 레이어**에서는 LLM에게 최소 권한만 부여하는 원칙이 핵심이다. 에이전트가 접근할 수 있는 도구와 API를 태스크에 필요한 최소로 제한하고, 도구 호출 전에 허용 목록(Allowlist) 기반으로 파라미터를 검증한다. NeMo Guardrails와 같은 프레임워크를 사용하여 LLM이 특정 주제 영역을 벗어나지 못하도록 레일을 설정한다.

**출력 레이어**에서는 모든 LLM 출력을 신뢰하지 않는 원칙으로 HTML 이스케이핑, SQL 파라미터화, 명령어 인젝션 방지 처리를 적용한다. Indirect Injection 방어를 위해 LLM이 처리하는 외부 콘텐츠(웹 페이지, 문서)에서 메타 지시(Meta-instruction)를 포함할 수 있는 특정 패턴을 정규식으로 필터링한다.

### Q9. Azure에서 Retrieval-Augmented Generation 시스템의 인덱싱 파이프라인을 설계할 때 청크 전략과 임베딩 선택 기준은?

**A:** 청크 전략은 문서 유형과 쿼리 패턴에 따라 맞춤화되어야 하며, 단일 전략이 모든 경우에 최적이지 않다.

**청크 전략**으로 고정 크기(Fixed-size) 청킹(512 ~ 1024 토큰)은 구현이 단순하지만 의미 단위를 파괴할 수 있다. 의미 기반(Semantic) 청킹은 문장 임베딩 유사도를 기반으로 의미 경계를 찾아 청크를 분할하므로 품질이 높지만 처리 비용이 크다. 계층적(Hierarchical) 청킹은 문서를 섹션 → 단락 → 문장의 계층으로 표현하여, 검색 시 적절한 세분화 수준을 선택한다. Parent Document Retriever 패턴은 소형 청크로 검색하되 부모 청크를 컨텍스트로 반환하여 정밀 검색과 충분한 컨텍스트를 동시에 달성한다.

**임베딩 모델 선택 기준**은 MTEB(Massive Text Embedding Benchmark) 리더보드를 출발점으로 삼되, 반드시 도메인 특화 평가를 수행해야 한다. 영어 범용에는 `text-embedding-3-large`(OpenAI)나 `e5-mistral-7b-instruct`가 강력하다. 한국어 포함 다국어에는 `multilingual-e5-large` 또는 `bge-m3`가 높은 성능을 보인다. 코드 검색에는 `voyage-code-2`가 특화되어 있다. 임베딩 차원은 높을수록 표현력이 좋지만 벡터 DB 저장 비용과 검색 속도에 영향을 주므로, Matryoshka Representation Learning(MRL)을 지원하는 모델을 사용하면 차원을 가변적으로 줄일 수 있다.

---

## [NVIDIA 면접]

### Q10. NVIDIA의 Triton Inference Server를 프로덕션에 배포할 때 성능 최적화 전략을 서술하라.

**A:** Triton Inference Server 최적화는 배칭, 모델 최적화, 리소스 관리의 세 차원에서 접근한다.

**배칭 최적화**에서 Dynamic Batching은 Triton의 핵심 기능으로, 설정된 max_batch_size와 max_queue_delay_microseconds 파라미터로 수집 지연 시간을 조절한다. 일반적으로 지연 시간보다 처리량을 우선시하는 오프라인 추론에서는 배치 크기를 크게, 레이턴시 SLA가 엄격한 온라인 추론에서는 배치 크기를 작게 설정한다. Sequence Batching은 RNN 또는 자기회귀 모델에서 여러 요청의 시퀀스를 상태를 유지하며 배치 처리할 수 있게 한다.

**모델 최적화**로는 TensorRT 변환이 가장 효과적이다. ONNX로 내보낸 모델을 TensorRT로 컴파일하면 FP16/INT8 정밀도 최적화, 레이어 융합(Layer Fusion), 커널 자동 선택(Kernel Autotuning)을 통해 2~4배의 처리량 향상을 달성한다. TensorRT-LLM은 LLM에 특화된 최적화(In-flight Batching, Paged KV Cache, Speculative Decoding)를 Triton과 함께 제공한다.

**리소스 관리**에서 Model Instances(config.pbtxt의 `count`) 설정은 GPU 활용률을 높이는 핵심이다. 단일 GPU에 여러 모델 인스턴스를 배치하거나(작은 모델), 단일 모델에 여러 GPU를 할당하는(큰 모델) 전략을 GPU 메모리와 처리량 목표에 맞게 선택한다. Triton의 Rate Limiter와 Priority Scheduler를 활용하면 여러 모델과 요청 클래스 간의 리소스 공유를 정밀하게 제어할 수 있다.

### Q11. GPU 클러스터에서 LLM 분산 학습 시 통신 병목(Communication Bottleneck)을 어떻게 프로파일링하고 해결하는가?

**A:** 분산 학습에서 통신 병목은 GPU 간, 노드 간 두 레이어로 분리하여 분석해야 한다.

**프로파일링 도구**로는 NVIDIA Nsight Systems와 PyTorch Profiler를 활용한다. PyTorch Profiler의 trace를 TensorBoard에서 분석하면 `ncclAllReduce`, `ncclBroadcast` 등의 통신 커널이 전체 스텝 시간에서 차지하는 비율을 확인할 수 있다. NCCL_DEBUG=INFO 환경변수로 NCCL의 링 토폴로지와 알고리즘 선택을 로깅하여 비최적 구성을 확인한다. Nsight Systems의 NVLink/PCIe 대역폭 타임라인에서 실제 통신 대역폭이 이론 최대치 대비 얼마나 달성되는지 분석한다.

**해결 전략**은 병목 위치에 따라 다르다. 노드 내 GPU 간 통신은 NVLink/NVSwitch를 활용하며, `NCCL_P2P_LEVEL=NVL` 설정으로 NVLink를 강제 사용한다. 노드 간 통신은 InfiniBand HDR/NDR 네트워크와 RDMA를 활용하고, NCCL의 NET/IB 플러그인이 올바르게 구성되었는지 확인한다. 그라디언트 압축(PowerSGD, 1-bit Adam) 또는 Gradient Accumulation으로 통신 빈도와 크기를 줄인다. ZeRO-1/2에서 ZeRO-3로 전환하면 파라미터 통신이 증가하지만 활성화 메모리가 감소하여 더 큰 배치 처리가 가능하므로, 통신-배치 크기 트레이드오프를 실험적으로 측정해야 한다.

### Q12. Transformer 모델에서 Key-Value Cache의 메모리 복잡도와 이를 줄이기 위한 Multi-Query Attention(MQA) 및 Grouped Query Attention(GQA)의 차이를 설명하라.

**A:** 표준 Multi-Head Attention(MHA)에서 KV Cache의 메모리는 레이어 수 × 배치 크기 × 헤드 수 × 시퀀스 길이 × 헤드 차원 × 2(K와 V) × 데이터 타입 크기로 계산된다. 70B LLaMA 모델에서 BF16으로 배치 크기 1, 시퀀스 길이 4096일 때 약 35GB의 KV Cache가 필요하여 모델 가중치(140GB)의 1/4에 달한다. 배치 크기와 시퀀스 길이에 선형적으로 증가하므로 실제 서빙에서 KV Cache가 주요 메모리 병목이 된다.

**MQA(Multi-Query Attention)**는 모든 쿼리 헤드가 단일 K, V 헤드를 공유하는 방식으로, KV Cache 크기를 헤드 수(H)배 줄인다. PaLM, Falcon 등이 채택했으나 모델 품질의 저하가 발생할 수 있다.

**GQA(Grouped Query Attention)**는 MHA와 MQA의 절충점으로, 쿼리 헤드를 G개의 그룹으로 나누고 각 그룹이 하나의 K, V 헤드를 공유한다. KV Cache를 H/G배 줄이면서 MQA보다 높은 모델 품질을 유지한다. LLaMA 2/3, Mistral, Gemma가 GQA를 채택하였다. Llama 3 70B는 8개의 KV 헤드를 64개의 쿼리 헤드가 공유(그룹 크기 8)하여 KV Cache를 1/8로 절감한다.

---

## [Apple 면접]

### Q13. Apple의 온디바이스(On-device) ML 모델 배포에서 Core ML 최적화와 프라이버시 보존 ML의 기술적 접근은?

**A:** Apple의 온디바이스 ML은 **프라이버시 바이 디자인(Privacy by Design)** 원칙 위에 구축된다.

**Core ML 최적화**는 모델 압축의 여러 기법을 계층적으로 적용한다. 가중치 양자화(Weight Quantization)는 Core ML Tools의 `coremltools.optimize.coreml`을 통해 FP32 → INT4/INT8 변환을 수행하고, 정밀도 손실을 최소화하기 위해 K-Means 기반 팔레타이제이션(Palettization)을 활용한다. 구조적 가지치기(Structured Pruning)는 전체 필터나 레이어를 제거하여 모델 크기와 연산량을 줄이며, Unstructured Pruning + 희소 행렬 연산을 지원하는 ANE(Apple Neural Engine)의 특성을 활용한다. Core ML의 stateful prediction API는 어텐션 KV Cache를 디바이스 측에 유지하여 LLM 추론의 반복 계산을 줄인다.

**프라이버시 보존 기법**으로 Differential Privacy(DP)는 On-device 학습(Federated Learning) 시 그라디언트에 보정된 가우시안 노이즈를 추가하여 개인 데이터 추론을 방지한다. DP-SGD는 각 샘플의 그라디언트를 L2 노름으로 클리핑하고 가우시안 노이즈를 추가하며, 프라이버시 예산(ε, δ)으로 보호 수준을 정량화한다. Apple의 Private Cloud Compute는 서버 측 처리가 필요한 경우 하드웨어 기반 TEE(Trusted Execution Environment)를 활용하여 서버 운영자도 사용자 데이터에 접근하지 못하도록 보장한다.

### Q14. 모바일 환경에서 Neural Architecture Search(NAS)를 활용한 효율적인 모델 아키텍처 설계 접근법은?

**A:** NAS는 탐색 공간(Search Space), 탐색 전략(Search Strategy), 성능 추정(Performance Estimation)의 세 요소로 구성된다.

탐색 공간 설계가 NAS의 품질을 결정한다. 모바일 특화 탐색 공간은 MobileNet 계열의 Depthwise Separable Convolution, MBConv(Mobile Inverted Bottleneck Convolution), Fused-MBConv 연산자를 포함하며, 레이어별 커널 크기, 확장 비율(Expansion Ratio), 채널 수를 탐색 변수로 설정한다.

탐색 전략에서 DARTS(Differentiable Architecture Search)는 탐색 공간을 연속적으로 완화하여 그라디언트 하강으로 아키텍처를 최적화한다. 그러나 메모리 비용이 크고, 아키텍처 파라미터와 가중치의 동시 학습으로 인한 불안정성 문제가 있다. Evolution 기반 탐색(NASNet, EfficientNet)은 신뢰성이 높지만 탐색 비용이 크다. 

Apple의 접근법과 유사한 Hardware-aware NAS는 목적 함수에 실제 디바이스 지연 시간(Latency)을 포함하여 정확도와 속도의 Pareto Front를 탐색한다. Once-for-All(OFA) 네트워크는 다양한 리소스 제약에 맞게 서브넷을 즉시 추출할 수 있는 수퍼넷을 한 번만 학습하여 NAS 비용을 크게 줄인다.

---

## [Tesla 면접]

### Q15. Tesla의 자율주행 시스템에서 데이터 플라이휠(Data Flywheel) 아키텍처와 Shadow Mode 배포 전략을 설명하라.

**A:** Tesla의 데이터 플라이휠은 플릿(Fleet) 데이터 수집 → 자동 레이블링 → 모델 학습 → 배포 → 더 많은 데이터 수집의 선순환 구조다.

**Shadow Mode**는 Tesla의 핵심 안전 검증 메커니즘이다. 새 모델 버전은 차량에 배포되지만 실제 제어권은 없고, 기존 모델과 동시에 실행되어 의사결정의 차이(divergence)를 기록한다. 기존 모델이 브레이크를 밟았지만 새 모델은 브레이크를 밟지 않은 상황이 로그에 기록되고, 이 차이 사례들이 중요 엣지 케이스로 레이블링 큐에 들어간다. 이를 통해 수억 마일의 실제 주행 데이터에서 희귀하지만 중요한 시나리오를 자동으로 채굴(Mining)한다.

**자동 레이블링 파이프라인**은 여러 카메라와 레이더 센서의 퓨전, 고화질 지도 정보, 다중 프레임에 걸친 3D 객체 추적을 통해 사람의 개입 없이 레이블을 생성한다. Dojo 슈퍼컴퓨터는 이 방대한 데이터셋으로의 빠른 학습을 위한 전용 ML 슈퍼컴퓨터로, NVLink 수준의 통신 대역폭을 달성하는 커스텀 인터커넥트 아키텍처를 특징으로 한다.

**Active Learning** 전략으로 모델이 낮은 신뢰도를 보이는 샘플, 앙상블 불일치가 높은 샘플, 최근 사고와 관련된 시나리오를 우선적으로 레이블링 큐에 넣어 레이블링 비용 대비 모델 성능 향상을 극대화한다.

### Q16. 자율주행 인식 모델에서 Distribution Shift를 처리하는 Domain Adaptation 전략은?

**A:** 자율주행에서 Distribution Shift는 날씨(비, 눈, 안개), 시간대(밤, 황혼), 지역(북미 vs 유럽 vs 아시아 도로 패턴) 등 수많은 요인에 의해 발생한다.

**Domain Randomization**은 학습 시 렌더링된 합성 데이터의 조명, 질감, 날씨, 카메라 노이즈 등을 무작위로 변화시켜 모델이 특정 도메인의 시각적 특성에 과적합하지 않도록 한다. CARLA, Unreal Engine 기반 시뮬레이터가 이에 활용된다.

**Adversarial Domain Adaptation**은 Domain Adversarial Training(DANN 등)으로 특성 추출기가 소스 도메인과 타깃 도메인을 구분하지 못하는 도메인 불변 특성(Domain-invariant Feature)을 학습하도록 한다. 그라디언트 반전 레이어(Gradient Reversal Layer)를 통해 도메인 분류기 손실을 최소화하면서 태스크 손실을 동시에 최소화한다.

**Test-Time Adaptation(TTA)**은 배포 후 추론 시 입력 데이터의 분포를 실시간으로 감지하고 배치 정규화 통계를 업데이트하는 기법이다. Tent(Test Entropy Minimization)는 추론 시 예측 불확실성을 최소화하는 방향으로 배치 정규화 파라미터를 온라인으로 업데이트한다. Tesla 수준의 플릿에서는 국가/기후 클러스터별로 별도 배치 정규화 통계를 유지하는 방식도 활용된다.

---

## [공통 심화 면접 문항]

### Q17. MLflow와 Kubeflow의 역할 분담과 통합 아키텍처를 설명하라.

**A:** MLflow와 Kubeflow는 ML 라이프사이클의 서로 다른 레이어를 담당하며 상호 보완적이다.

**MLflow**는 실험 추적(Experiment Tracking), 모델 패키징(MLflow Models), 모델 레지스트리(Model Registry), 파라미터 추적에 특화된 경량 추적 레이어다. 데이터 사이언티스트가 로컬 개발 환경에서부터 사용하며, Python SDK만으로 모든 실험 메타데이터를 중앙 추적 서버에 기록한다. MLflow의 강점은 프레임워크 독립성(sklearn, PyTorch, TensorFlow, XGBoost 모두 지원)과 낮은 도입 장벽이다.

**Kubeflow**는 Kubernetes 위에서 ML 워크플로우의 오케스트레이션과 인프라 관리에 집중한다. Kubeflow Pipelines는 ML 워크플로우를 DAG로 정의하고 재현 가능한 실행을 보장하며, Katib는 Kubernetes 네이티브 하이퍼파라미터 최적화를, Training Operator는 PyTorchJob/TFJob으로 분산 학습을 관리한다.

통합 아키텍처에서는 Kubeflow Pipeline 컴포넌트 내에서 MLflow를 임베드하는 패턴이 일반적이다. 각 파이프라인 단계가 실행될 때 MLflow를 통해 파라미터와 메트릭을 기록하고, 파이프라인 런 ID를 MLflow 실험에 태그로 연결하여 인프라 오케스트레이션(Kubeflow)과 실험 추적(MLflow) 간의 양방향 추적성을 확보한다.

### Q18. 모델 서빙에서 Canary 배포와 Shadow 배포의 차이점 및 각각의 적합한 사용 사례는?

**A:** 두 전략은 리스크 관리 철학이 근본적으로 다르다.

**Canary 배포**는 실제 프로덕션 트래픽의 일부(예: 5%)를 새 모델 버전으로 점진적으로 라우팅한다. 실제 사용자가 새 모델의 응답을 받으므로, 실제 사용 조건에서의 성능(CTR 변화, 전환율)을 측정할 수 있다. 단, 새 모델에 문제가 있으면 해당 비율의 사용자가 영향을 받는다. 롤아웃 결정 기준(예: 레이턴시 P99 < 200ms, 에러율 < 0.1%)을 사전에 정의하고, 기준을 충족하면 자동으로 트래픽 비율을 높이는 Progressive Delivery 패턴으로 발전시킬 수 있다.

**Shadow 배포**는 모든 프로덕션 트래픽을 기존 모델과 새 모델 양쪽에 복제(Mirror)하여 새 모델의 응답을 실제 서빙에 사용하지 않고 로깅만 한다. 사용자에게 전혀 노출되지 않으므로 실험의 리스크가 없다. 단, 실제 사용자 반응(Feedback Signal)을 얻을 수 없으며, 트래픽 복제로 인한 인프라 비용 증가가 발생한다.

적합한 사용 사례로, 금융 사기 탐지처럼 오류의 비용이 매우 높은 경우에는 Shadow 배포를 통해 충분히 검증 후 Canary로 전환한다. 추천 시스템처럼 A/B 테스트가 핵심인 경우에는 Canary가 유일한 방법이다. Tesla의 예시처럼 안전이 최우선인 자율주행에서는 Shadow Mode가 기본 검증 단계다.

### Q19. MLOps에서 데이터 드리프트 감지를 위한 통계적 검정 방법을 수학적으로 비교하라.

**A:** 데이터 드리프트 감지는 참조 분포(Reference Distribution)와 현재 분포(Current Distribution) 간의 차이를 정량화하는 문제다.

**PSI(Population Stability Index)**는 금융 업계에서 유래한 방법으로, 두 분포를 버킷으로 이산화하여 PSI = Σ(P_i - Q_i) × ln(P_i/Q_i)로 계산한다. 0.1 이하는 변화 없음, 0.1~0.25는 경미한 변화, 0.25 이상은 중대한 변화로 해석한다. 직관적이고 단방향성(현재 ← 참조)이라는 장점이 있으나, 버킷 수와 경계에 민감하고 연속형 분포에서 정보 손실이 크다.

**KS(Kolmogorov-Smirnov) 검정**은 두 연속 분포의 최대 누적 분포 함수 차이 D = max|F_1(x) - F_2(x)|를 통계량으로 사용한다. 분포의 형태 전반에 걸친 차이를 탐지하지만 분포의 꼬리(Tail) 부분의 차이를 놓칠 수 있고, 다변량 확장이 어렵다.

**MMD(Maximum Mean Discrepancy)**는 재현 커널 힐베르트 공간(RKHS)에서 두 분포의 평균 임베딩 거리를 측정한다. 커널 함수를 통해 암묵적으로 무한 차원으로 매핑하므로, 다변량 분포와 복잡한 분포 간의 차이를 포착하는 데 특히 강력하다. 딥러닝 특성 공간의 드리프트 감지에 활용된다.

**CUSUM(Cumulative Sum)** 등의 시퀀셜 검정은 단일 시점 비교가 아닌 스트리밍 데이터에서 변화점(Change Point)을 온라인으로 탐지하는 데 적합하다. 생산 환경에서는 PSI와 KS를 일상 모니터링에, MMD를 딥러닝 모델의 고차원 드리프트 감지에 조합하는 다층 전략이 권장된다.

### Q20. LLM 시스템에서 Hallucination을 평가하고 완화하는 아키텍처 수준의 전략은?

**A:** 환각은 LLM이 사실에 근거하지 않은 내용을 사실인 것처럼 생성하는 현상으로, 완전 제거보다 체계적 완화가 현실적 목표다.

**평가 방법**으로 FactScore는 생성된 텍스트를 원자적 사실 단위로 분해하고 각 사실을 위키피디아 등 신뢰 소스에 대해 검증한다. SelfCheckGPT는 동일한 프롬프트로 여러 번 샘플링하여 일관성이 낮은 내용을 환각으로 감지한다. RAGAS의 Faithfulness 메트릭은 RAG 컨텍스트에서 생성된 답변의 각 문장이 검색된 청크에 귀속 가능한지 검증한다.

**완화 아키텍처**는 다단계로 구성된다. 소스 귀속(Source Attribution) 강제를 위해 모델이 각 주장에 대한 출처 인용을 생성하도록 프롬프트를 설계하고, 인용이 없는 주장에 대해 경고를 표시한다. Chain-of-Verification(CoVe)은 LLM이 자신의 초기 응답을 검증 질문으로 분해하고, 각 질문에 독립적으로 답변한 후 초기 응답을 수정하는 자가 검증 프로세스다. RAG 기반 시스템에서는 검색된 컨텍스트에서 직접 답변을 추출하도록 유도하고(Extractive Answer), 컨텍스트 밖의 정보를 생성하지 못하도록 지시한다. 외부 팩트체킹 API(예: Google Fact Check API) 또는 전용 검증 모델을 출력 후처리 단계에 추가하는 것도 효과적이다.

### Q21. Kubernetes에서 ML 워크로드를 위한 GPU 리소스 스케줄링 고급 기법을 설명하라.

**A:** Kubernetes의 기본 GPU 스케줄러는 GPU를 정수 단위로만 할당하며, ML 워크로드의 다양한 리소스 요구에 최적이 아닐 수 있다.

**GPU Sharing** 기법으로 NVIDIA의 MIG(Multi-Instance GPU)는 A100/H100 GPU를 최대 7개의 독립적인 GPU 인스턴스로 하드웨어 수준에서 파티셔닝한다. 각 인스턴스는 전용 메모리, SM, NVLink 포트를 가지므로 완벽한 격리가 보장되어 멀티 테넌트 환경에 적합하다. 반면 Time-Slicing은 여러 프로세스가 GPU 시간을 순차적으로 공유하므로 격리가 없지만 구성이 단순하다.

**NVIDIA의 MPS(Multi-Process Service)**는 여러 CUDA 컨텍스트를 단일 CUDA 컨텍스트로 통합하여 컨텍스트 전환 오버헤드를 줄이고, 작은 배치 크기의 추론 워크로드에서 GPU SM 활용률을 높인다.

**Gang Scheduling**은 분산 학습에서 모든 워커 포드가 동시에 스케줄링되지 않으면 교착 상태가 발생하는 문제를 해결한다. Volcano와 Yunikorn 스케줄러는 PodGroup 개념을 통해 동일 그룹의 포드를 All-or-Nothing으로 스케줄링한다. Kubernetes의 Topology Aware Scheduling을 활용하면 GPU 토폴로지(같은 NVSwitch 도메인, 같은 노드, 같은 랙)를 고려하여 통신 대역폭을 최대화하는 배치를 결정한다.

### Q22. MLOps에서 CI/CD 파이프라인을 ML 시스템에 적용할 때 전통 소프트웨어와 다른 특수 고려사항은?

**A:** ML을 위한 CI/CD는 코드 변경뿐만 아니라 데이터, 모델, 환경의 변경 모두에 대해 통합 테스트와 배포를 자동화해야 한다.

**테스트 피라미드의 확장**으로, 전통 소프트웨어의 단위/통합/E2E 테스트 외에 ML 특화 테스트가 필요하다. 학습 루프 테스트(손실이 감소하는지 단일 배치로 확인), 모델 성능 회귀 테스트(기준 메트릭 대비 새 버전 비교), 불변성 테스트(데이터 변환의 특성 값 범위 확인), 데이터 유효성 검사(Great Expectations, TFX/TFDV)가 CI 파이프라인에 포함되어야 한다.

**모델 배포 게이트(Deployment Gate)**는 ML CI/CD의 핵심 차별점이다. 코드 테스트 통과만으로는 불충분하며, 오프라인 평가 메트릭(F1, AUC, BLEU 등)이 임계값을 통과해야 배포가 진행된다. 모델 카드(Model Card)와 편향성 분석 리포트가 자동 생성되어 검토된다.

**데이터와 모델의 버전 연동**이 중요하다. 각 모델 버전은 학습에 사용된 데이터셋 버전(DVC 해시), 코드 버전(Git SHA), 환경 버전(Docker 이미지 다이제스트)과 함께 Model Registry에 등록된다. 특정 모델 버전을 완전히 재현하려면 이 세 가지 버전 정보가 모두 필요하다. GitHub Actions + Weights & Biases + DVC + MLflow를 조합하거나, ZenML / Metaflow를 사용하면 이 전체 파이프라인을 코드로 정의(Pipeline as Code)할 수 있다.

### Q23. LLM 기반 멀티 에이전트 시스템에서 신뢰성(Reliability)과 일관성(Consistency)을 보장하는 방법은?

**A:** 멀티 에이전트 시스템의 신뢰성 문제는 LLM 호출의 비결정성(Non-determinism), 에이전트 루프의 무한 실행 가능성, 도구 호출 실패의 전파 등에서 비롯된다.

**비결정성 관리**를 위해 중요한 에이전트 결정(단계 완료 확인, 데이터베이스 업데이트 등)에는 temperature=0을 사용하고 JSON Schema 기반의 구조화된 출력(Function Calling)을 강제하여 파싱 오류를 방지한다. 동일 입력에 대해 여러 번 샘플링하고 다수결(Majority Voting) 또는 가장 일관된 출력을 선택하는 Self-Consistency 기법을 고위험 결정에 적용한다.

**멱등성(Idempotency)과 재시도** 전략으로, 모든 도구 호출과 외부 API 호출은 멱등하게 설계하고 지수 백오프(Exponential Backoff)와 Jitter를 포함한 재시도 로직을 구현한다. 에이전트 실행 그래프를 LangGraph와 같은 프레임워크로 관리하면 중간 상태를 영속화(Persistence)할 수 있어 실패 지점부터 재실행이 가능하다.

**서킷 브레이커(Circuit Breaker)** 패턴을 에이전트 루프에 적용하여 최대 반복 횟수, 최대 실행 시간, 비용 상한을 설정한다. 이를 초과하면 현재 최선의 답변을 반환하거나 사람의 개입을 요청하는 Human-in-the-Loop 인터럽트를 발동한다.

### Q24. 대규모 LLM 추론 시스템에서 비용 최적화를 위한 토큰 예산 관리 전략은?

**A:** LLM 추론 비용은 입력 토큰 + 출력 토큰의 합산이므로, 두 축 모두에서 최적화가 필요하다.

**입력 토큰 최적화**에서 프롬프트 압축 기법인 LLMLingua는 LLM의 중요하지 않은 토큰을 제거하여 프롬프트를 최대 20배 압축하면서 성능 저하를 최소화한다. RAG 파이프라인에서 Contextual Compression은 검색된 청크에서 쿼리와 관련된 부분만 추출하여 컨텍스트 크기를 줄인다. 프롬프트 캐싱(Anthropic, OpenAI 지원)은 반복되는 시스템 프롬프트를 서버 측에 캐시하여 입력 비용을 크게 절감한다.

**출력 토큰 최적화**에서 Speculative Decoding은 출력 토큰 수를 줄이는 것은 아니지만 단위 시간당 처리 토큰을 늘려 처리량을 높인다. max_tokens 파라미터를 태스크 유형에 맞게 동적으로 설정하여 불필요하게 긴 응답을 방지한다. 모델 라우팅 전략으로, 단순한 쿼리는 저렴한 소형 모델(GPT-4o-mini, Claude Haiku)로 라우팅하고 복잡한 쿼리만 고성능 모델을 사용한다. 쿼리 복잡도 분류기를 LLM Gateway에 내장하거나, 소형 모델이 답변 불가 시 대형 모델로 에스컬레이션하는 계단식(Cascading) 전략을 적용한다.

### Q25. Transformer 기반 모델의 Long Context 처리를 위한 최신 아키텍처 기법을 설명하라.

**A:** Long Context 처리는 어텐션의 O(N²) 복잡도와 KV Cache의 메모리 병목이라는 두 가지 근본 문제를 다룬다.

**효율적 어텐션 메커니즘**으로 Sliding Window Attention(Mistral)은 각 토큰이 전체 시퀀스가 아닌 고정 크기 로컬 윈도우 내 토큰만 참조한다. Rolling Buffer Cache로 오래된 KV를 순환 삭제하여 상수 크기의 KV Cache를 유지한다. Longformer와 BigBird는 로컬 어텐션 + 전역 어텐션(CLS 토큰 또는 미리 지정된 전역 토큰)을 결합하여 O(N)의 복잡도를 달성한다.

**Position Encoding의 외삽(Extrapolation)**은 학습 컨텍스트 길이를 넘어 더 긴 시퀀스를 처리하는 핵심 기법이다. RoPE(Rotary Position Embedding)는 ALiBi와 함께 Long Context 외삽에 강한 것으로 알려져 있으며, YaRN(Yet another RoPE extensioN) 기법은 특정 주파수 성분의 스케일을 조정하여 학습 길이의 수배에 달하는 시퀀스를 처리한다.

**Chunked/Streaming Attention**은 입력을 청크로 나누어 어텐션을 계산하고 상태를 누적하는 방식으로, Mamba와 같은 SSM(State Space Model) 아키텍처는 이 아이디어를 극단적으로 발전시켜 O(N)의 복잡도로 매우 긴 시퀀스를 처리한다.

### Q26. LLM 서빙 인프라에서 Multi-LoRA 서빙의 아키텍처와 구현 도전을 설명하라.

**A:** 하나의 기반 모델(Base Model)에 수십~수천 개의 LoRA 어댑터를 고객별로 서빙하는 Multi-LoRA 시나리오는 LLM SaaS 플랫폼의 핵심 요구사항이다.

**아키텍처 도전**은 기반 모델(수십 GB)을 중복 로드하지 않고 여러 LoRA 어댑터를 동적으로 교체/공유하는 것이다. 기반 모델을 GPU에 고정하고, 각 요청이 지정한 LoRA 어댑터를 CPU 메모리나 스토리지에서 온디맨드로 로드하는 **LoRA Hot-Swapping** 방식이 기본이다. 그러나 어댑터 로드 레이턴시가 문제가 된다.

**punica(OSDI 2024)**와 **S-LoRA**는 이 문제를 근본적으로 해결한다. 동일 기반 모델의 여러 LoRA 어댑터를 가진 요청들을 하나의 배치로 처리하되, 각 요청에 대해 서로 다른 LoRA 가중치를 적용하는 배치 GEMM(Segmented GEMM) 커널을 구현한다. 이를 통해 기반 모델 가중치는 공유하면서 LoRA 어댑터만 요청별로 다르게 적용하는 진정한 Multi-LoRA 배칭이 가능해진다. 어댑터 캐싱 전략(LRU, Access Pattern 기반)으로 자주 사용되는 어댑터는 GPU VRAM에 상주시키고, 나머지는 CPU 메모리나 NVMe에 계층적으로 관리한다.

### Q27. Federated Learning을 프로덕션 MLOps 파이프라인에 통합할 때의 아키텍처 과제는?

**A:** 연합 학습(Federated Learning)은 데이터를 중앙 서버로 전송하지 않고 분산된 디바이스나 사일로(Silo)에서 로컬 모델을 학습하고 그라디언트/모델 업데이트만 집계하는 프라이버시 보존 학습 방식이다.

**Federated Averaging(FedAvg)**은 가장 기본적인 집계 알고리즘으로, 각 클라이언트가 로컬 데이터로 수 에포크 학습한 후 가중치 평균을 계산한다. Non-IID(독립 동일 분포가 아닌) 데이터 분포에서는 클라이언트 드리프트(Client Drift)가 발생하여 수렴이 불안정하며, FedProx는 로컬 학습에 근접항(Proximal Term)을 추가하여 이를 완화한다.

**프로덕션 통합 도전**으로 클라이언트 가용성 관리가 핵심이다. 모바일 디바이스는 학습 중 충전 중이고 Wi-Fi에 연결된 경우에만 참여하므로, Cross-device FL에서는 클라이언트 선택 전략(무작위 샘플링, 중요도 기반 선택)이 중요하다. 모델 이질성 문제(클라이언트마다 모델 아키텍처가 다를 수 있음)는 SplitFed나 헤드-테일 분리 아키텍처로 접근한다. PySyft, TFF(TensorFlow Federated), Flower(flwr)가 프로덕션 FL 파이프라인의 주요 프레임워크다. 보안 집계(Secure Aggregation)는 서버도 개별 클라이언트의 그라디언트를 볼 수 없도록 암호화 프로토콜(비밀 공유, 동형 암호화)을 적용한다.

### Q28. MLOps 파이프라인에서 대규모 하이퍼파라미터 최적화(HPO)의 알고리즘 비교와 분산 실행 전략은?

**A:** HPO는 탐색 공간, 평가 비용, 병렬화 가능성의 세 가지 차원에서 알고리즘을 선택해야 한다.

**알고리즘 비교**에서 Grid Search와 Random Search는 간단하지만 차원의 저주에 취약하다. Random Search는 Grid Search 대비 고차원 공간에서 동일한 계산 예산으로 더 넓은 공간을 탐색한다. **Bayesian Optimization(BO)**은 가우시안 프로세스(GP) 또는 TPE(Tree-structured Parzen Estimator)를 대리 모델(Surrogate Model)로 구축하고 획득 함수(Acquisition Function: EI, UCB, PI)로 다음 탐색 지점을 결정한다. BO는 저차원 공간에서 효율적이지만, 고차원 + 병렬 평가 환경에서 GP의 계산 비용이 크다. **HyperBand와 ASHA(Asynchronous Successive Halving)**는 일찍 유망하지 않은 트라이얼을 중단하는 조기 중단(Early Stopping) 전략으로, 동일 예산에서 더 많은 구성을 탐색한다. **BOHB(Bayesian Optimization + HyperBand)**는 BO의 샘플 효율성과 HyperBand의 조기 중단을 결합하여 현재 HPO 분야의 강력한 기준선이다.

**분산 실행 전략**으로 Kubernetes + Ray Tune은 동적 리소스 할당으로 GPU/CPU 혼합 클러스터에서 병렬 트라이얼을 실행한다. Katib(Kubeflow)는 Kubernetes CRD로 HPO 실험을 선언적으로 정의하고, Suggestion 서비스 플러그인으로 다양한 알고리즘을 교체 가능하게 지원한다. 비동기적 병렬 평가(Asynchronous Parallel Evaluation)는 워커가 트라이얼을 완료하는 즉시 새 구성을 수신하여 느린 트라이얼로 인한 대기를 방지한다.

### Q29. LLM 애플리케이션에서 RAG vs Fine-tuning vs Prompt Engineering의 의사결정 프레임워크를 설명하라.

**A:** 세 전략은 업데이트 비용, 지식의 성격, 추론 레이턴시 측면에서 근본적으로 다른 트레이드오프를 가진다.

**Prompt Engineering만으로 충분한 경우**는 모델이 이미 해당 지식을 가지고 있고, 특정 출력 형식이나 행동 패턴만 조정하면 될 때다. 비용이 가장 낮고 변경이 빠르지만, 모델의 기본 지식과 능력의 범위 내에서만 효과가 있다. 특히 Few-shot 예시를 잘 설계하면 Fine-tuning 없이도 상당한 성능을 달성할 수 있다.

**RAG가 적합한 경우**는 지식이 자주 변경되거나(뉴스, 제품 카탈로그, 내부 문서), 출처 귀속이 중요하거나, 지식 범위가 명확히 구분되는 도메인일 때다. RAG는 새 문서 추가만으로 즉시 지식을 업데이트할 수 있고, 검색된 컨텍스트로 환각을 감소시킨다는 장점이 있다. 그러나 복잡한 암묵적 지식(Procedural Knowledge), 특정 스타일이나 톤, 다단계 추론 능력은 RAG로 주입하기 어렵다.

**Fine-tuning이 필요한 경우**는 특정 도메인 어휘와 표현 방식이 중요하거나(의학, 법률, 코드), 모델 행동(응답 형식, 거절 패턴, 페르소나)을 일관되게 변경하거나, 응답 속도가 중요하여 긴 컨텍스트 주입이 불가한 경우다. Fine-tuning은 학습/평가/배포 비용이 높지만, 암묵적 지식을 파라미터에 내재화하여 추론 시 컨텍스트 없이도 작동한다.

최근의 Best Practice는 **RAG + Light Fine-tuning 결합**이다. Fine-tuning으로 모델이 RAG 결과를 더 잘 활용하고 도메인 특화 형식으로 응답하도록 조정하고, RAG로 최신 동적 지식을 주입하는 전략이다.

### Q30. MLOps 관점에서 대규모 모델 학습 시 재현성(Reproducibility) 완전 보장을 위한 체크리스트와 기술적 과제는?

**A:** 완전한 재현성은 코드, 데이터, 환경, 무작위성이라는 네 축의 결정론을 보장해야 한다.

**코드 재현성**은 Git SHA와 함께 모든 의존성의 정확한 버전을 pip freeze 또는 conda lock 파일로 고정하고, Docker 이미지를 다이제스트(SHA256)로 참조하여 베이스 이미지 업데이트의 영향을 받지 않게 한다.

**데이터 재현성**은 DVC 또는 Delta Lake Time Travel로 학습에 사용된 데이터셋의 정확한 버전을 기록하고, 데이터 전처리 파이프라인도 코드와 함께 버전 관리한다. 데이터 셔플링에 고정 시드(Fixed Seed)를 사용하되, 분산 학습 환경에서 각 워커의 셔플 시드 설정이 올바르게 복원되는지 확인해야 한다.

**환경 재현성**은 CUDA, cuDNN, NCCL 버전이 모두 동일해야 하며, 이는 컨테이너 이미지로 고정하는 것이 유일한 실용적 방법이다.

**무작위성 재현성**은 가장 까다롭다. Python random, NumPy, PyTorch의 시드를 모두 고정해야 하며, 특히 분산 학습에서 각 프로세스의 시드를 다르게 설정(rank * seed)해야 한다. CUDA 연산의 비결정성(cuDNN 알고리즘 선택의 무작위성)은 `torch.use_deterministic_algorithms(True)`로 강제하지만, 이는 일부 연산의 성능 저하를 유발한다. DataLoader의 `num_workers > 0`일 때 워커 초기화 함수에 시드 설정이 필요하다.

완전한 재현성은 실제로 성능 비용을 수반하므로, 프로덕션 환경에서는 코드+데이터 재현성을 우선시하고 연산 레벨의 완전한 결정론은 디버깅이 필요한 경우에만 활성화하는 실용적 접근이 일반적이다.

### Q31. 스트리밍 데이터 환경에서 Online Learning 시스템과 배치 학습 시스템을 혼합한 Lambda Architecture의 현대적 변형을 설명하라.

**A:** Lambda Architecture는 배치 레이어(정확하지만 느림)와 스피드 레이어(빠르지만 근사)를 병렬 운영하여 장단점을 보완하는 아키텍처다. ML에서의 현대적 변형은 다음과 같다.

**배치 레이어**에서는 전체 히스토리 데이터로 주기적으로(일별, 주별) 완전 재학습한다. 이 모델은 전체 데이터 분포를 학습했으므로 가장 안정적이고 강인하다. Spark + Delta Lake 파이프라인이 배치 학습 데이터를 준비하고, MLflow + Kubeflow가 학습과 배포를 오케스트레이션한다.

**스피드 레이어**에서는 Kafka/Kinesis 스트림에서 새 이벤트를 실시간으로 처리하여 모델을 부분 업데이트한다. River(Python), Vowpal Wabbit 등의 Online Learning 라이브러리가 Mini-batch SGD 또는 완전 온라인 방식으로 모델을 점진적으로 업데이트한다. 이 레이어는 최신 트렌드를 빠르게 반영하지만 노이즈에 민감하다.

**서빙 레이어**는 배치 모델과 온라인 모델의 예측을 앙상블하여 최종 출력을 생성한다. 앙상블 가중치를 최근 데이터 품질과 각 모델의 성능에 따라 동적으로 조정한다.

**Kappa Architecture**는 Lambda의 복잡성을 줄이기 위해 스트리밍 레이어만으로 배치와 실시간을 모두 처리한다. Apache Flink의 상태 관리 능력과 Exactly-once 처리 보장이 이를 실용적으로 만들었다. ML에서는 Flink + Online Learning 모델의 조합으로 Lambda의 이중 코드베이스 문제를 해결한다.

### Q32. LLMOps에서 모델 선택 및 A/B 테스트의 통계적 엄밀성 확보 방법은?

**A:** LLM A/B 테스트는 전통적인 소프트웨어 A/B 테스트보다 복잡하다. LLM 출력이 연속형이거나 개방형이기 때문에 이진 클릭/전환율보다 측정이 어렵다.

**메트릭 설계**에서 프록시 메트릭과 최종 비즈니스 메트릭의 연결이 중요하다. 즉각적인 사용자 만족도(Thumbs Up/Down 피드백), 대화 완료율(Conversation Completion Rate), 후속 질문 비율(재질의는 불만족 신호), 업무 완료율(Task Completion Rate) 등의 메트릭을 파이프라인에 통합한다.

**샘플 사이즈 계산**은 실험 설계의 기반이다. 최소 탐지 효과 크기(MDE), 유의 수준(α, 보통 0.05), 검정력(1-β, 보통 0.8)을 설정하고 필요한 샘플 수를 사전에 계산해야 한다. LLM 출력의 높은 분산(Variance)은 대규모 샘플을 요구하므로, Variance Reduction 기법(CUPED: Controlled-experiment Using Pre-Experiment Data)으로 공변량을 통제하여 검정력을 높인다.

**다중 검정 문제(Multiple Testing Problem)**는 여러 메트릭을 동시에 측정할 때 발생한다. Bonferroni 수정이나 FDR(False Discovery Rate) 제어(Benjamini-Hochberg)를 적용하여 1종 오류 확률을 제어한다. Sequential Testing(SPRT: Sequential Probability Ratio Test, Always-valid p-values)은 고정된 샘플 수 없이 실험을 계속 모니터링하면서 조기 종료나 연장 결정을 p-value 인플레이션 없이 할 수 있게 한다.

---

# PART 5. 최신 트렌드 및 미래 방향

## 5.1 Test-Time Compute와 추론 시점 스케일링

OpenAI o1, DeepSeek R1, Gemini 2.0 Flash Thinking으로 대표되는 **추론 모델(Reasoning Model)**은 학습 시점이 아닌 추론 시점에 더 많은 컴퓨팅을 사용하는 패러다임이다. Chain-of-Thought를 내재화하여 복잡한 수학, 코딩, 과학 추론에서 기존 모델 대비 획기적인 성능 향상을 보인다.

MLOps 관점에서 이는 토큰 예산 관리와 비용 모니터링의 새로운 도전을 만든다. 동일한 질문이라도 추론 난이도에 따라 수십 배의 토큰 사용량 차이가 발생하므로, 비용 예측과 SLA 관리가 더욱 복잡해진다.

## 5.2 멀티모달 모델의 MLOps

GPT-4o, Gemini, Claude 3.5 Sonnet 등 텍스트-이미지-오디오-비디오를 통합 처리하는 멀티모달 모델의 등장은 MLOps 파이프라인의 데이터 관리를 복잡하게 만든다. 이미지/오디오 데이터 버전 관리, 멀티모달 평가 벤치마크 구축, 모달리티별 편향 감지 등 새로운 과제들이 등장했다.

## 5.3 AI 규제와 MLOps 거버넌스

EU AI Act와 NIST AI RMF를 비롯한 글로벌 AI 규제는 MLOps에 새로운 컴플라이언스 레이어를 추가하고 있다. 고위험 AI 시스템에 대한 투명성, 설명 가능성(Explainability), 편향 감사(Bias Audit), 인간 감독(Human Oversight)의 기술적 구현이 MLOps 파이프라인의 필수 요소가 되고 있다. Model Card, Datasheets for Datasets, Responsible AI 체크리스트가 ML 거버넌스 도구로 표준화되는 추세다.

---

*이 가이드는 2025년 기준 최신 아키텍처와 기법을 반영하였으며, MLOps/LLMOps 분야의 빠른 발전에 따라 지속적인 업데이트가 권장됩니다.*
