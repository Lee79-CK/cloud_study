# Agentic Design Patterns: 개념, 활용, 실습 완벽 가이드

> **대상**: LLMOps / MLOps / Cloud Engineer  
> **최종 업데이트**: 2026-04-06  
> **난이도**: Intermediate → Advanced

---

## 1. 서론 — 왜 Agentic AI인가?

### 1.1 패러다임의 전환

기존 LLM 활용 방식은 단일 프롬프트에 단일 응답을 반환하는 **비에이전틱(non-agentic) 워크플로**였습니다. 사용자가 "이 코드를 리팩터링해줘"라고 요청하면, 모델은 한 번에 결과를 생성하고 그것으로 상호작용이 끝났습니다.

Agentic AI는 이 구조를 근본적으로 바꿉니다. LLM이 **자율적으로 판단하고, 도구를 호출하며, 결과를 스스로 평가하고, 필요하면 재시도**하는 루프를 형성합니다. 즉, LLM이 단순한 "텍스트 생성기"에서 **"목표 지향적 워커(goal-oriented worker)"**로 진화하는 것입니다.

### 1.2 Agenticity 스펙트럼

에이전트의 자율성 수준은 이진법이 아니라 **스펙트럼** 위에 존재합니다.

| 수준 | 설명 | 예시 |
|------|------|------|
| **Level 0** — 정적 자동화 | 규칙 기반, AI 없음 | RPA, cron job |
| **Level 1** — 비에이전틱 AI | LLM이 고정된 파이프라인 안에서 동작 | 단순 RAG, 분류기 |
| **Level 2** — 제한적 에이전트 | LLM이 제한된 범위 내 도구를 선택 | Tool-use 챗봇 |
| **Level 3** — 자율 에이전트 | LLM이 계획, 실행, 평가를 모두 수행 | 코딩 에이전트, 리서치 에이전트 |
| **Level 4** — 멀티 에이전트 | 복수의 에이전트가 협업/경쟁하며 문제 해결 | 멀티 에이전트 디베이트, 조직 시뮬레이션 |

핵심 원칙은 **"가장 단순한 아키텍처에서 시작하고, 측정 가능한 이점이 있을 때만 복잡도를 올리라"**는 것입니다. 자율성이 높아질수록 비용, 레이턴시, 디버깅 난이도가 기하급수적으로 증가합니다.

---

## 2. 핵심 Agentic Design Patterns

### 2.1 인지 제어 루프 (Cognitive Control Loop)

모든 에이전틱 시스템의 기반이 되는 아키텍처입니다. 단순 RAG 파이프라인이 방향 비순환 그래프(DAG)인 것과 달리, 에이전트는 **순환(cycle)** 구조를 가집니다.

```
┌─────────────┐
│  Perception  │ ← 사용자 입력 + 환경 상태 수집
└──────┬──────┘
       ▼
┌─────────────┐
│  Reasoning   │ ← LLM이 현재 상태를 분석하고 다음 행동 결정 (Thought)
└──────┬──────┘
       ▼
┌─────────────┐
│   Action     │ ← 도구 호출, API 요청, 코드 실행 등
└──────┬──────┘
       ▼
┌─────────────┐
│ Observation  │ ← 행동 결과를 수집하고 루프 재진입 여부 판단
└──────┬──────┘
       │
       └──────→ (다시 Perception으로, 또는 종료)
```

이 Perception → Reasoning → Action → Observation 사이클이 **목표 달성 조건(termination condition)**을 만족할 때까지 반복됩니다.

---

### 2.2 Pattern 1: Prompt Chaining (프롬프트 체이닝)

**개념**: 하나의 복잡한 태스크를 여러 개의 순차적 LLM 호출로 분해합니다. 각 단계의 출력이 다음 단계의 입력이 됩니다.

**작동 방식**: 태스크 분해 → 단계 1 실행 → 검증 게이트(gate) → 단계 2 실행 → … → 최종 출력

**언제 사용하는가**:
- 태스크가 자연스럽게 순차적 하위 태스크로 분해 가능할 때
- 각 단계에서 품질 검증이 필요할 때
- 전체 워크플로의 제어권을 유지하고 싶을 때

**실제 사례**: 문서 번역 파이프라인에서 (1) 원문 분석 → (2) 초벌 번역 → (3) 용어 일관성 검증 → (4) 최종 교정의 4단계를 각각 별도의 LLM 호출로 처리합니다.

```python
# 개념적 구현
def translation_pipeline(source_text: str) -> str:
    # Step 1: 원문 분석
    analysis = llm.call("원문의 문체, 전문 용어, 맥락을 분석하라", source_text)
    
    # Step 2: 초벌 번역
    draft = llm.call(f"분석 결과를 참고하여 번역하라: {analysis}", source_text)
    
    # Gate: 품질 체크
    quality = llm.call("번역 품질을 1-10으로 평가하라", draft)
    if int(quality) < 7:
        draft = llm.call("다음 피드백을 반영하여 재번역하라", draft)
    
    # Step 3: 최종 교정
    final = llm.call("용어 일관성과 자연스러움을 검수하라", draft)
    return final
```

---

### 2.3 Pattern 2: Routing (라우팅)

**개념**: 입력을 분류한 뒤, 분류 결과에 따라 **특화된 다운스트림 프로세스**로 분기합니다. LLM이 라우터 역할을 수행합니다.

**작동 방식**: 입력 → LLM 분류기 → 분기 (경로 A / 경로 B / 경로 C) → 각 경로별 전문 처리

**언제 사용하는가**:
- 입력의 종류에 따라 완전히 다른 처리 로직이 필요할 때
- 각 카테고리에 최적화된 프롬프트/모델을 사용하고 싶을 때

**실제 사례**: 고객 지원 시스템에서 티켓을 "기술 문제", "결제 문의", "일반 질문"으로 분류하고, 각각 다른 전문 에이전트에게 라우팅합니다.

---

### 2.4 Pattern 3: Reflection (리플렉션)

**개념**: 에이전트가 자신의 출력을 **스스로 평가하고 비판**한 뒤, 피드백을 반영하여 결과를 개선하는 자기 참조적 패턴입니다.

**작동 방식**: 초기 생성 → 자체 평가 (Critic) → 피드백 생성 → 피드백 반영 재생성 → 반복

**핵심 구현 전략**:
- **Self-Reflection**: 동일 모델이 생성자와 비평가 역할을 번갈아 수행
- **Cross-Reflection**: 생성 모델과 평가 모델을 분리하여 더 객관적인 피드백 확보

**언제 사용하는가**:
- 초안 작성 후 품질을 높여야 할 때 (코드 리뷰, 글쓰기 교정 등)
- 단일 호출로는 충분한 품질이 나오지 않을 때

**실제 사례**: 코드 생성 에이전트가 코드를 작성한 뒤, 별도의 LLM 호출로 "이 코드에 버그, 보안 취약점, 성능 이슈가 있는지 검토하라"는 리뷰를 수행하고, 발견된 문제를 수정한 코드를 재생성합니다.

```python
def reflect_and_improve(task: str, max_iterations: int = 3) -> str:
    draft = llm.generate(task)
    
    for i in range(max_iterations):
        critique = llm.evaluate(
            f"다음 결과의 문제점과 개선 방향을 구체적으로 제시하라:\n{draft}"
        )
        
        if "문제 없음" in critique:
            break
        
        draft = llm.generate(
            f"원래 작업: {task}\n이전 결과: {draft}\n피드백: {critique}\n"
            f"피드백을 반영하여 개선된 결과를 생성하라."
        )
    
    return draft
```

---

### 2.5 Pattern 4: Tool Use (도구 사용)

**개념**: LLM이 외부 도구(API, 데이터베이스, 코드 실행 환경 등)를 호출하여 자신의 한계를 보완합니다. 학습 데이터에 없는 실시간 정보를 가져오거나, 정확한 계산을 수행하거나, 실제 시스템에 작업을 지시할 수 있습니다.

**도구 사용의 표준화 — MCP (Model Context Protocol)**: Anthropic이 제안한 MCP는 도구 사용 프로세스를 표준화하여 모델과 도구 간의 연결을 단순화합니다. 도구의 스키마를 명시적으로 정의하고, LLM이 정해진 형식으로 도구를 호출하도록 강제합니다.

**핵심 도구 유형**:
- **정보 검색**: 웹 검색, DB 쿼리, 문서 검색
- **실행/조작**: 코드 실행, API 호출, 파일 시스템 조작
- **분석/계산**: 수학 연산, 데이터 분석, 시각화

**운영 고려사항 (MLOps 관점)**:
- Schema Enforcement: Pydantic 등으로 도구 호출 파라미터를 강제 검증
- 타임아웃 및 재시도 정책 설정
- 도구 호출 비용 모니터링 (API 비용, 레이턴시)

---

### 2.6 Pattern 5: ReAct (Reasoning + Acting)

**개념**: LLM이 **사고(Thought) → 행동(Action) → 관찰(Observation)**을 명시적으로 분리하여 수행하는 패턴입니다. 추론 토큰과 도구 호출을 분리함으로써 도구 사용의 신뢰성과 계획 수립 능력을 높입니다.

**ReAct 루프 예시**:

```
Thought: 사용자가 서울의 오늘 날씨를 물어봤다. 날씨 API를 호출해야 한다.
Action: weather_api.get(location="Seoul", date="today")
Observation: 맑음, 기온 18°C, 습도 45%
Thought: 정보를 얻었으니 사용자에게 자연스럽게 전달하자.
Answer: 오늘 서울 날씨는 맑으며, 기온은 18°C, 습도는 45%입니다.
```

**ReAct vs. 순수 Chain-of-Thought**: 순수 CoT는 추론만 하고 행동하지 못하며, 순수 Action-only 접근은 왜 그 행동을 하는지 설명하지 못합니다. ReAct는 양쪽의 장점을 결합합니다.

**프로덕션에서의 구현**: 실제 프로덕션 환경에서는 사고(Thought) 과정이 내부적으로 처리되거나 구조화되어 기록되는 경우가 많습니다. 스크래치패드(scratchpad) 메커니즘으로 중간 추론 과정을 저장하고, 디버깅과 관찰성(observability)에 활용합니다.

---

### 2.7 Pattern 6: Planning (계획 수립)

**개념**: 복잡한 태스크를 하위 목표(sub-goals)로 분해하고, 논리적 순서로 배열한 뒤 순차 또는 병렬로 실행합니다.

**주요 계획 전략**:
- **Task Decomposition**: "이 프로젝트를 완수하기 위한 단계를 나열하라"
- **Dynamic Re-planning**: 중간 결과에 따라 계획을 수정
- **MCTS 기반 계획**: Monte Carlo Tree Search를 LLM에 적용하여 시뮬레이션 대신 자체 평가로 최적 경로를 탐색

**언제 사용하는가**:
- 태스크가 복수의 하위 단계를 포함하고, 단계 간 의존성이 있을 때
- 중간에 조건 분기가 필요할 때

---

### 2.8 Pattern 7: Multi-Agent Collaboration (멀티 에이전트 협업)

**개념**: 각각 **전문화된 역할**을 가진 복수의 에이전트가 하나의 태스크를 협업하여 해결합니다.

**오케스트레이션 패턴**:

| 패턴 | 설명 | 적합한 상황 |
|------|------|-------------|
| **Supervisor** | 중앙 조정자가 하위 에이전트에게 태스크를 분배 | 명확한 역할 분리, 중앙 통제 필요 시 |
| **Debate / Critique** | 에이전트들이 서로의 출력을 비판하며 합의에 도달 | 창의적 작업, 의사결정 품질 향상 |
| **Pipeline** | 에이전트가 순차적으로 태스크를 이어받아 처리 | 순차 의존성이 강한 워크플로 |
| **Swarm** | 자율적 에이전트 집단이 분산 협업 | 탐색적 작업, 비정형 문제 |

**주의사항**: 멀티 에이전트 시스템은 비용과 복잡도가 가장 높습니다. 예측 불가능한 결과를 어느 정도 수용할 수 있는 실험적 태스크에 한정하여 사용하는 것이 권장됩니다.

**실제 사례**: 소프트웨어 개발 에이전트 팀 — PM 에이전트(요구사항 분석), 아키텍트 에이전트(설계), 개발 에이전트(구현), QA 에이전트(테스트)가 Supervisor 패턴으로 협업합니다.

---

### 2.9 Pattern 8: Human-in-the-Loop (사람 참여)

**개념**: 에이전트 루프의 특정 지점에 **사람의 승인이나 피드백**을 삽입합니다. 고위험 의사결정에서 안전장치 역할을 합니다.

**설계 포인트**:
- 승인이 필요한 행동의 범위를 명확히 정의 (예: 결제 실행, 데이터 삭제)
- 사람의 응답 지연(human bottleneck)을 고려한 타임아웃 설계
- 에스컬레이션 경로 정의

---

## 3. 패턴 선택 가이드 — 의사결정 프레임워크

패턴을 선택할 때 고려해야 할 핵심 파라미터:

| 파라미터 | 낮음 → 높음 방향 |
|----------|------------------|
| **응답 신뢰성 요구** | Prompt Chaining → ReAct → Multi-Agent |
| **워크플로 구조화 정도** | 자유로운 Agent → Routing → Prompt Chaining |
| **태스크 복잡도** | 단일 Tool Use → Planning → Multi-Agent |
| **실패 허용도** | Human-in-the-Loop → Reflection → Autonomous Agent |

**실전 가이드라인**:
1. 작동 가능한 가장 단순한 아키텍처로 시작하라
2. 성능을 정량적으로 측정하라 (성공률, 레이턴시, 비용)
3. 복잡도를 올릴 명확한 근거가 있을 때만 패턴을 추가하라
4. 반복 횟수 제한, 성공 메트릭, 가드레일을 사전에 정의하라

---

## 4. 프로덕션 운영 — LLMOps 관점

### 4.1 실패 관리 전략

에이전틱 시스템에서 실패는 "고치는" 것이 아니라 **"관리하는"** 것입니다. 결정론적 소프트웨어의 버그와 달리, 확률적 AI의 분산(variance)을 다뤄야 합니다.

| 실패 유형 | 디자인 패턴 |
|-----------|------------|
| 도구 환각 (존재하지 않는 도구 호출) | Schema Enforcement (Pydantic 등으로 검증) |
| 컨텍스트 드리프트 (원래 목적을 잊음) | 주기적 목표 재주입, 메모리 관리 |
| 무한 루프 | 최대 반복 횟수 설정, 비용 상한 |
| 보안 위협 (Prompt Injection) | 입력 샌드박싱, 권한 분리, 네트워크 정책 |

### 4.2 관찰성(Observability) 설계

에이전트 시스템에서는 단일 요청이 수십 번의 LLM 호출과 도구 호출로 분기될 수 있으므로, 분산 트레이싱이 필수적입니다.

**필수 메트릭**:
- 루프 반복 횟수 (iteration count per request)
- 도구 호출 성공률 및 레이턴시
- 토큰 소비량 (입력/출력 분리)
- 종료 조건 달성률 (정상 종료 vs 최대 반복 초과 종료)

### 4.3 메모리 아키텍처

| 계층 | 역할 | 구현 |
|------|------|------|
| **Working Memory** | 현재 활성 컨텍스트 | 프롬프트 내 컨텍스트 윈도우 |
| **Short-term Memory** | 최근 대화 히스토리 | Redis, 세션 스토어 |
| **Long-term Memory** | 영구 지식, 학습 결과 | Vector DB (Milvus, Chroma 등) |

---

## 5. NVIDIA DGX Spark에서의 실습

### 5.1 DGX Spark 하드웨어 개요

NVIDIA DGX Spark은 Grace Blackwell GB10 Superchip을 탑재한 데스크톱형 AI 슈퍼컴퓨터로, 에이전틱 AI 개발에 최적화된 로컬 환경을 제공합니다.

**핵심 사양**:
- **프로세서**: GB10 Grace Blackwell Superchip (20-core ARM CPU + Blackwell GPU)
- **메모리**: 128GB 통합 LPDDR5x (CPU/GPU 공유)
- **AI 성능**: 최대 1 PFLOP (FP4 Sparse), 1,000 TOPS
- **네트워크**: ConnectX-7 200Gbps (최대 4대 클러스터링)
- **OS**: DGX OS (Ubuntu 기반)
- **가격**: $4,699 (2026년 2월 기준)

**에이전트 개발에서의 이점**:
- 128GB 통합 메모리로 70B 파라미터 모델을 양자화 없이 로컬 실행 가능
- 200B 파라미터 모델까지 추론 가능
- 대형 컨텍스트 윈도우 처리에 필요한 prefill 처리량 확보
- 클라우드 의존 없이 프라이버시가 보장된 에이전트 개발 환경

### 5.2 소프트웨어 스택 — NemoClaw & OpenShell

GTC 2026에서 NVIDIA는 에이전트 개발을 위한 소프트웨어 스택을 공개했습니다.

**NVIDIA Agent Toolkit 구성**:

```
NVIDIA Agent Toolkit
├── NemoClaw          ← OpenClaw + 보안/프라이버시 계층
│   ├── Nemotron 모델  ← 로컬 추론 모델 (120B MoE 등)
│   └── OpenShell      ← 커널 수준 샌드박싱 런타임
├── OpenClaw           ← 에이전트 런타임 프레임워크
└── NVIDIA NIM         ← 모델 서빙 최적화 (TensorRT-LLM)
```

**NemoClaw의 역할**: OpenClaw 에이전트를 보안성 있게 실행하기 위한 오픈소스 스택입니다. 로컬 모델 추론(토큰 비용 제로), 샌드박싱(호스트 파일시스템/네트워크 격리), 프라이버시 라우팅 기능을 제공합니다.

**OpenShell의 역할**: 자율 AI 에이전트를 커널 수준에서 격리하여 실행하는 런타임입니다. 에이전트가 호스트 시스템에 접근하지 못하도록 차단하면서도, 승인된 도구와 API에만 접근을 허용합니다.

### 5.3 실습 1: DGX Spark에 NemoClaw 환경 구축

**Step 1 — Docker 및 NVIDIA 런타임 구성**

```bash
# Docker 그룹 추가 후 재로그인
sudo usermod -aG docker $USER

# NVIDIA 컨테이너 런타임 설정
sudo nvidia-ctk runtime configure --runtime=docker

# DGX Spark cgroup v2 호환 설정 (필수)
sudo python3 -c "
import json, os
path = '/etc/docker/daemon.json'
d = json.load(open(path)) if os.path.exists(path) else {}
d['default-cgroupns-mode'] = 'host'
json.dump(d, open(path, 'w'), indent=2)
"

sudo systemctl restart docker
```

**Step 2 — Ollama 및 모델 설정**

```bash
# Ollama가 실행 중인지 확인
ollama list

# Nemotron 3 Super 120B 모델 다운로드
ollama pull nemotron-3-super:120b

# 모든 인터페이스에서 수신하도록 설정 (컨테이너 접근 허용)
sudo systemctl edit ollama
# Environment="OLLAMA_HOST=0.0.0.0:11434" 추가
sudo systemctl restart ollama
```

**Step 3 — NemoClaw 설치 및 초기화**

```bash
# NemoClaw 클론 및 설치
git clone https://github.com/NVIDIA/NemoClaw.git
cd NemoClaw
./install.sh

# 또는 원격 스크립트 설치
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash

# 온보딩 마법사 실행
nemoclaw onboard
```

온보딩 마법사가 Docker, OpenShell 게이트웨이, 샌드박스를 자동으로 구성합니다.

**Step 4 — 에이전트 샌드박스 접속 및 테스트**

```bash
# 샌드박스 연결
nemoclaw my-assistant connect

# 샌드박스 내부에서 에이전트 테스트
openclaw agent --agent main --local -m "hello" --session-id test

# TUI(터미널 UI) 채팅 시작
openclaw chat
# Ctrl+C로 종료
```

**Step 5 — 웹 UI 대시보드 접근**

```bash
# 로컬 접속 (DGX Spark에 직접 모니터 연결 시)
# 브라우저에서: http://127.0.0.1:18789/#token=<your-gateway-token>

# 원격 접속 (SSH 포트 포워딩)
ssh -L 18789:127.0.0.1:18789 user@dgx-spark-ip
# 이후 로컬 브라우저에서: http://127.0.0.1:18789/#token=<token>
```

> **주의**: `localhost` 대신 반드시 `127.0.0.1`을 사용하세요. 게이트웨이의 origin 체크가 정확한 주소 매칭을 요구합니다.

### 5.4 실습 2: Agentic Design Pattern 구현 — ReAct 에이전트

DGX Spark에서 Nemotron 모델을 사용하여 ReAct 패턴 에이전트를 직접 구현해봅니다.

```python
# react_agent.py — DGX Spark 로컬 추론 ReAct 에이전트
import requests, json

OLLAMA_URL = "http://localhost:11434/api/generate"
MODEL = "nemotron-3-super:120b"

TOOLS = {
    "search_docs": lambda q: f"[검색 결과: '{q}'에 대한 내부 문서 3건 발견]",
    "run_sql": lambda q: f"[SQL 결과: 매출 총합 $1,234,567]",
    "send_alert": lambda msg: f"[알림 전송 완료: {msg}]",
}

SYSTEM_PROMPT = """당신은 ReAct 패턴으로 동작하는 AI 에이전트입니다.
반드시 다음 형식을 따르세요:

Thought: (현재 상황 분석과 다음 행동 계획)
Action: tool_name(argument)
Observation: (도구 실행 결과 — 시스템이 자동 입력)

최종 답변이 준비되면:
Thought: 충분한 정보를 수집했다.
Answer: (최종 응답)

사용 가능한 도구: search_docs, run_sql, send_alert
"""

def run_react_loop(user_query: str, max_steps: int = 5):
    context = f"{SYSTEM_PROMPT}\n\nUser: {user_query}\n"
    
    for step in range(max_steps):
        resp = requests.post(OLLAMA_URL, json={
            "model": MODEL,
            "prompt": context,
            "stream": False,
        })
        output = resp.json()["response"]
        context += output
        
        # 최종 답변 감지
        if "Answer:" in output:
            answer = output.split("Answer:")[-1].strip()
            print(f"[최종 답변] {answer}")
            return answer
        
        # 도구 호출 감지 및 실행
        if "Action:" in output:
            action_line = [l for l in output.split("\n") if "Action:" in l][0]
            tool_call = action_line.split("Action:")[-1].strip()
            tool_name = tool_call.split("(")[0]
            tool_arg = tool_call.split("(")[1].rstrip(")")
            
            if tool_name in TOOLS:
                observation = TOOLS[tool_name](tool_arg)
            else:
                observation = f"[오류: 알 수 없는 도구 '{tool_name}']"
            
            context += f"\nObservation: {observation}\n"
            print(f"  Step {step+1}: {tool_name}({tool_arg}) → {observation}")
    
    return "[최대 반복 횟수 초과]"

# 실행
result = run_react_loop("지난 분기 매출 데이터를 조회하고, 목표 미달이면 팀에 알림을 보내줘")
```

### 5.5 실습 3: 멀티 에이전트 Supervisor 패턴

```python
# multi_agent_supervisor.py
import requests, json

OLLAMA_URL = "http://localhost:11434/api/chat"
MODEL = "nemotron-3-super:120b"

def call_agent(role: str, system_prompt: str, task: str) -> str:
    """특정 역할의 에이전트를 호출합니다."""
    resp = requests.post(OLLAMA_URL, json={
        "model": MODEL,
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": task}
        ],
        "stream": False,
    })
    return resp.json()["message"]["content"]

# 에이전트 정의
AGENTS = {
    "researcher": "당신은 리서치 전문가입니다. 주어진 주제를 깊이 분석하세요.",
    "writer": "당신은 기술 문서 작성 전문가입니다. 명확하고 구조화된 문서를 작성하세요.",
    "reviewer": "당신은 QA 리뷰어입니다. 문서의 정확성과 완성도를 평가하세요.",
}

def supervisor_workflow(task: str) -> str:
    """Supervisor 패턴으로 멀티 에이전트 워크플로를 실행합니다."""
    
    # Supervisor가 태스크를 분해
    plan = call_agent("supervisor",
        "당신은 프로젝트 매니저입니다. 태스크를 하위 작업으로 분해하세요.",
        f"다음 태스크를 researcher, writer, reviewer에게 배분하라: {task}"
    )
    print(f"[Supervisor] 계획 수립 완료\n{plan}\n")
    
    # Researcher 실행
    research = call_agent("researcher", AGENTS["researcher"], task)
    print(f"[Researcher] 리서치 완료 ({len(research)} chars)\n")
    
    # Writer 실행
    draft = call_agent("writer", AGENTS["writer"],
        f"다음 리서치 결과를 바탕으로 문서를 작성하라:\n{research}")
    print(f"[Writer] 초안 작성 완료 ({len(draft)} chars)\n")
    
    # Reviewer 실행 (Reflection 패턴 결합)
    review = call_agent("reviewer", AGENTS["reviewer"],
        f"다음 문서를 검토하고 구체적인 개선점을 제시하라:\n{draft}")
    print(f"[Reviewer] 리뷰 완료\n{review}\n")
    
    # Writer가 피드백 반영 (Reflection)
    final = call_agent("writer", AGENTS["writer"],
        f"원래 초안:\n{draft}\n\n리뷰 피드백:\n{review}\n\n"
        f"피드백을 반영하여 최종 문서를 완성하라.")
    print(f"[Writer] 최종본 완성 ({len(final)} chars)")
    
    return final

# 실행
result = supervisor_workflow("Kubernetes에서의 GPU 스케줄링 최적화 가이드 작성")
```

### 5.6 실습 4: 4-Node DGX Spark 클러스터 구성

DGX Spark은 최대 4대까지 ConnectX-7 NIC를 통해 클러스터링이 가능합니다. 이를 통해 512GB 통합 메모리 풀을 구성하고, 최대 700B 파라미터 모델의 추론 및 분산 파인튜닝을 수행할 수 있습니다.

```bash
# 2대 직접 연결 (스위치 불필요, 200Gbps 케이블만 필요)
# Spark A <──200Gbps──> Spark B
# 결과: 256GB 통합 메모리, 405B 파라미터 모델 추론 가능

# 4대 클러스터 (ConnectX-7 호환 스위치 필요)
# Spark A ──┐
# Spark B ──┤── 200Gbps 스위치
# Spark C ──┤
# Spark D ──┘
# 결과: 512GB 통합 메모리, 700B 파라미터 모델까지 지원

# 클러스터 상태 확인
nvidia-smi  # 각 노드에서 실행
ibstat      # InfiniBand/RoCE 연결 상태 확인
```

---

## 6. 프레임워크 비교 및 기술 스택 선택

### 6.1 주요 에이전트 프레임워크

| 프레임워크 | 특징 | DGX Spark 호환 | 적합한 상황 |
|-----------|------|---------------|------------|
| **LangChain / LangGraph** | 가장 넓은 생태계, 그래프 기반 워크플로 | O | 프로토타이핑, 다양한 패턴 실험 |
| **LlamaIndex** | 데이터 인덱싱/검색 특화, 에이전트 기능 확장 중 | O | RAG 중심 에이전트 |
| **AutoGen (Microsoft)** | 멀티 에이전트 대화 프레임워크 | O | 멀티 에이전트 연구/프로토타이핑 |
| **CrewAI** | 역할 기반 멀티 에이전트 오케스트레이션 | O | 역할이 명확한 팀 워크플로 |
| **OpenClaw + NemoClaw** | NVIDIA 네이티브, 보안 샌드박싱 내장 | ◎ (최적) | 보안이 중요한 프로덕션 에이전트 |

### 6.2 추천 기술 스택 (DGX Spark 기반)

```
┌─────────────────────────────────────────────────┐
│                  Application Layer                │
│  LangGraph / CrewAI / Custom Orchestrator        │
├─────────────────────────────────────────────────┤
│              Agent Runtime Layer                  │
│  OpenClaw + NemoClaw (샌드박싱, 보안)             │
├─────────────────────────────────────────────────┤
│             Model Serving Layer                   │
│  Ollama / vLLM / TensorRT-LLM / SGLang          │
├─────────────────────────────────────────────────┤
│            Inference Engine Layer                  │
│  Nemotron 3 Super 120B / Llama 3.3 70B / Qwen   │
├─────────────────────────────────────────────────┤
│              Hardware Layer                        │
│  DGX Spark (GB10, 128GB, 1 PFLOP)               │
└─────────────────────────────────────────────────┘
```

---

## 7. 실전 사례 및 응용 시나리오

### 7.1 사례 1: DevOps 자동화 에이전트

**패턴 조합**: Routing + Tool Use + Human-in-the-Loop

Slack에서 인시던트 알림을 수신하면, 에이전트가 로그를 분석하고 근본 원인을 추론합니다. 자동 복구가 가능한 경우 Runbook을 실행하고, 고위험 조치(스케일 아웃, 롤백)는 사람의 승인을 받은 후 실행합니다.

### 7.2 사례 2: 데이터 파이프라인 모니터링 에이전트

**패턴 조합**: ReAct + Reflection

Airflow DAG의 실패를 감지하면, 에이전트가 로그를 조회하고(Tool Use), 원인을 추론하며(ReAct), 수정 코드를 생성한 뒤 자체 검증합니다(Reflection). 검증 통과 시 PR을 자동 생성합니다.

### 7.3 사례 3: 보안 이벤트 분류 및 대응 에이전트

**패턴 조합**: Routing + Multi-Agent + Planning

SIEM 알림을 심각도와 유형으로 분류(Routing)한 뒤, 각 유형별 전문 에이전트(네트워크, 엔드포인트, IAM)가 분석을 수행합니다. Supervisor 에이전트가 결과를 종합하고 대응 계획을 수립합니다(Planning).

---

## 8. 보안 고려사항

에이전틱 시스템의 보안은 기존 LLM 애플리케이션보다 공격 표면이 훨씬 넓습니다.

### 8.1 주요 위협

- **Prompt Injection**: 외부 입력(이메일, 문서, 웹페이지)에 삽입된 악성 지시문이 에이전트의 행동을 탈취
- **Tool Abuse**: 에이전트가 의도치 않게 위험한 도구를 호출하거나 잘못된 파라미터를 전달
- **Data Exfiltration**: 에이전트가 수집한 민감 정보가 외부 API로 유출

### 8.2 방어 패턴

- **Deny-by-Default 네트워크 정책**: NemoClaw의 기본 설정처럼, 명시적으로 허용된 엔드포인트만 접근 가능
- **입력/출력 필터링**: Nemotron 기반 PII 탐지, 의도 분류
- **샌드박스 격리**: OpenShell의 커널 수준 격리로 호스트 시스템 보호
- **Operator Approval Workflow**: 고위험 행동에 대한 사람 승인 필수화

---

## 9. 마무리 — 핵심 요약

1. **Agentic Design Pattern은 LLM의 자율성을 구조화하는 설계 청사진**입니다. Prompt Chaining, Routing, Reflection, Tool Use, ReAct, Planning, Multi-Agent, Human-in-the-Loop의 8가지 핵심 패턴을 이해하고 조합하면 대부분의 에이전틱 시스템을 설계할 수 있습니다.

2. **가장 단순한 아키텍처에서 시작하라.** 복잡도를 올릴수록 비용, 레이턴시, 디버깅 난이도가 기하급수적으로 증가합니다. 측정 가능한 이점이 있을 때만 패턴을 추가하세요.

3. **NVIDIA DGX Spark은 로컬 에이전트 개발의 이상적인 플랫폼**입니다. 128GB 통합 메모리로 대형 모델을 로컬에서 실행하고, NemoClaw/OpenShell로 보안이 강화된 에이전트를 데스크톱에서 프로토타이핑할 수 있습니다.

4. **LLMOps 관점에서 에이전트 시스템은 관찰성, 실패 관리, 보안이 핵심**입니다. 에이전트의 자율성이 높아질수록 이 세 가지의 중요성도 비례하여 커집니다.

---

## 부록: 참고 자료

- NVIDIA DGX Spark 공식 페이지: https://www.nvidia.com/en-us/products/workstations/dgx-spark/
- NVIDIA DGX Spark 플레이북: https://github.com/NVIDIA/dgx-spark-playbooks
- NemoClaw GitHub: https://github.com/NVIDIA/NemoClaw
- NVIDIA build.nvidia.com/spark: https://build.nvidia.com/spark
- AWS Agentic AI Patterns Guide: https://docs.aws.amazon.com/prescriptive-guidance/latest/agentic-ai-patterns/introduction.html
- Andrew Ng의 Agentic Design Patterns 강연 자료
- LangGraph 공식 문서: https://langchain-ai.github.io/langgraph/
