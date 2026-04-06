# Claude를 활용한 Agentic AI 구현 및 활용 방안

## 실무 강의자료 | 2026년 4월 기준

---

## 1강. Agentic AI란 무엇인가?

### 1.1 정의와 핵심 개념

Agentic AI는 단순히 프롬프트에 응답하는 수준을 넘어, **LLM이 스스로 도구(Tool)를 선택하고, 환경으로부터 피드백을 받으며, 루프 안에서 자율적으로 작업을 수행하는 시스템**을 의미합니다.

Anthropic은 이를 다음과 같이 구분합니다:

- **Workflow**: LLM과 도구가 **사전에 정의된 코드 경로**를 통해 오케스트레이션되는 시스템. 예측 가능하고 안정적입니다.
- **Agent**: LLM이 **자신의 프로세스와 도구 사용을 동적으로 결정**하며, 작업 수행 방법에 대한 자율적 제어권을 갖는 시스템.

핵심 공식은 놀랍도록 단순합니다:

```
Agent = LLM + Tools + Loop
```

Anthropic의 연구에 따르면, 가장 성공적인 에이전트 구현은 복잡한 프레임워크가 아니라 **단순하고 조합 가능한(composable) 패턴**을 활용합니다. 복잡성은 에이전트 아키텍처가 아닌 **도구 설계**에 집중해야 합니다.

### 1.2 2025-2026 Agentic AI 생태계 현황

2026년 현재 Agentic AI는 개념 증명 단계를 넘어 프로덕션으로 진입했습니다:

- **Claude Code**가 6개월 만에 $1B ARR을 달성하고, 2026년 2월 기준 $2.5B ARR을 기록
- 기업의 **57%가 멀티스텝 에이전트 워크플로우**를 배포 중
- **47%의 조직**이 범용 AI 코딩 도구와 자체 구축 에이전트를 결합하는 **하이브리드 아키텍처** 채택
- METR 연구에 따르면 AI 작업 지속 시간이 **7개월마다 2배씩 증가** — 2025년 초 1시간 → 2026년 말 8시간 자율 작업 예상

---

## 2강. Anthropic의 Agentic 아키텍처 패턴

Anthropic이 제시하는 5가지 핵심 워크플로우 패턴과 자율 에이전트 패턴을 살펴봅니다. 핵심 원칙은 **"단순한 것부터 시작하고, 명확한 성능 향상이 있을 때만 복잡성을 추가하라"**는 것입니다.

### 2.1 Prompt Chaining (프롬프트 체이닝)

순차적 LLM 호출로, 각 단계의 출력이 다음 단계의 입력이 됩니다. 단계 사이에 프로그래밍적 품질 게이트를 삽입합니다.

```
[LLM 호출 1] → 품질 게이트 → [LLM 호출 2] → 품질 게이트 → [LLM 호출 3]
```

**적합한 경우**: 작업을 고정된 하위 단계로 분해할 수 있을 때. 각 단계의 정확도를 위해 약간의 지연을 감수할 수 있을 때.

**예시**: 문서 생성 → 문서 검토 → 최종 편집의 3단계 파이프라인

### 2.2 Routing (라우팅)

입력을 분류하고 전문화된 핸들러로 라우팅합니다.

```
[입력] → [분류 LLM] → 카테고리 A → [전문 핸들러 A]
                     → 카테고리 B → [전문 핸들러 B]
                     → 카테고리 C → [전문 핸들러 C]
```

**적합한 경우**: 입력 유형에 따라 다른 전문 처리가 필요할 때. 고객 지원 시스템에서 billing/technical/general을 구분하는 것이 대표적입니다.

### 2.3 Parallelization (병렬화)

동일한 작업을 병렬로 실행하여 결과를 집계하거나, 서로 다른 하위 작업을 동시에 처리합니다.

- **Sectioning**: 독립적인 하위 작업을 병렬 처리
- **Voting**: 동일 작업을 여러 번 실행하여 다수결 또는 합의로 결과 선택

### 2.4 Orchestrator-Workers (오케스트레이터-워커)

중앙 오케스트레이터 LLM이 작업을 분석하고, 동적으로 하위 작업을 생성하여 워커 LLM에 위임합니다.

```
[사용자 요청] → [오케스트레이터 LLM]
                    ├── 하위 작업 1 → [워커 A]
                    ├── 하위 작업 2 → [워커 B]
                    └── 하위 작업 N → [워커 N]
                         ↓
               [오케스트레이터: 결과 합성]
```

**적합한 경우**: 하위 작업을 사전에 예측할 수 없는 복잡한 작업. 코딩 에이전트에서 Opus를 오케스트레이터로, Sonnet을 워커로 사용하는 패턴이 대표적입니다.

### 2.5 Evaluator-Optimizer (평가자-최적화자)

생성 LLM과 평가 LLM이 반복적으로 피드백 루프를 형성합니다.

```
[생성 LLM] → 응답 → [평가 LLM] → 피드백 → [생성 LLM] → ... → 최종 결과
```

**적합한 경우**: 명확한 평가 기준이 있고, 반복적 개선이 측정 가능한 가치를 제공할 때.

### 2.6 Autonomous Agent (자율 에이전트)

가장 높은 수준의 자율성입니다. 에이전트가 환경 피드백(도구 실행 결과, 코드 실행 결과 등)을 기반으로 루프 안에서 스스로 다음 단계를 결정합니다.

```python
while not task_complete:
    action = llm.decide_next_action(context, tools)
    result = execute(action)
    context.update(result)
    if needs_human_input(result):
        pause_for_feedback()
```

**핵심 원칙**: 각 단계에서 도구 실행 결과 같은 **"ground truth"**를 확인하여 진행 상황을 평가합니다. LLM 자체 평가보다 테스트 결과, 컴파일러 출력, 린터 등 객관적 결과를 우선합니다.

### 패턴 선택 의사결정 트리

```
새로운 작업 도착
├── 단일 도메인의 명확한 작업인가? → Pattern 1: 단일 에이전트 (70% 케이스)
├── 명확한 순차적 단계가 있는가? → Pattern 2: Prompt Chaining
├── 하위 작업이 독립적이고 병렬 처리 가능한가? → Pattern 3: Parallelization
├── 하위 작업이 동적/예측 불가능한가? → Pattern 4: Orchestrator-Workers
├── 명확한 기준으로 반복 개선이 필요한가? → Pattern 5: Evaluator-Optimizer
├── 개방형 탐색/디버깅인가? → Pattern 6: Autonomous Agent
└── 그 외 → 단일 에이전트로 시작, 필요시 확대
```

---

## 3강. Model Context Protocol (MCP) — AI 에이전트의 통합 표준

### 3.1 MCP 개요

MCP는 Anthropic이 2024년 11월 오픈소스로 공개한 AI 에이전트와 외부 시스템을 연결하는 **개방형 표준 프로토콜**입니다. 2025년 12월 Linux Foundation 산하 Agentic AI Foundation(AAIF)에 기증되었으며, Anthropic, Block, OpenAI가 공동 설립하고 Google, Microsoft, AWS, Cloudflare, Bloomberg이 지원합니다.

2026년 현재 MCP의 규모:

- **10,000개 이상**의 활성 공개 MCP 서버
- 월 **9,700만 이상**의 SDK 다운로드
- ChatGPT, Cursor, Gemini, Microsoft Copilot, VS Code 등 주요 AI 제품이 채택

### 3.2 MCP 아키텍처

MCP는 클라이언트-서버 아키텍처로 동작합니다:

```
[AI 애플리케이션 (MCP Client)]
        ↕ MCP Protocol (Streamable HTTP)
[MCP Server A: GitHub]  [MCP Server B: PostgreSQL]  [MCP Server C: Slack]
```

MCP 서버는 세 가지 핵심 요소를 제공합니다:

- **Tools**: AI 에이전트가 호출하여 특정 작업을 수행하는 함수 (예: API 호출, 명령 실행)
- **Resources**: AI 에이전트가 접근하는 데이터 소스 (REST API 엔드포인트와 유사)
- **Prompts**: 도구와 리소스를 최적으로 사용하도록 안내하는 사전 정의 템플릿

### 3.3 Anthropic API의 MCP Connector

2025년 10월 발표된 MCP Connector를 통해, 별도의 클라이언트 코드 없이 API 요청에 원격 MCP 서버 URL을 추가하는 것만으로 연결이 가능합니다:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Asana에서 내 작업 목록을 가져와줘"}],
    mcp_servers=[
        {
            "type": "url",
            "url": "https://mcp.asana.com/sse",
            "authorization_token": "Bearer <token>"
        }
    ]
)
```

Claude가 MCP 서버가 설정된 요청을 받으면 자동으로 사용 가능한 도구를 검색하고, 적절한 도구를 선택하여 호출합니다.

### 3.4 대규모 도구 환경에서의 효율화: 코드 실행 패턴

수백 개의 MCP 도구를 사용할 때 직접 호출 방식은 컨텍스트 윈도우를 과도하게 소비합니다. Anthropic 엔지니어링 팀은 **코드 실행 패턴**을 권장합니다:

```
기존 방식 (직접 호출):
  모든 도구 정의 → 컨텍스트에 로드 → 도구 호출 → 결과를 컨텍스트로 → 다음 호출
  → 토큰 비용 높음, 지연 시간 증가

코드 실행 방식:
  필요한 도구만 코드로 호출 → 결과를 메모리에서 처리 → 요약만 컨텍스트로
  → 토큰 비용 절감, 복잡한 로직 가능
```

에이전트가 MCP 도구를 직접 호출하는 대신, 코드를 생성하여 프로그래밍적으로 도구와 상호작용하면 토큰 사용량과 지연 시간을 크게 줄일 수 있습니다.

---

## 4강. Claude API를 활용한 Agentic 시스템 구축

### 4.1 핵심 API 기능 (2025-2026)

Anthropic API는 에이전트 구축을 위한 4가지 핵심 기능을 제공합니다:

| 기능 | 설명 | 에이전트 활용 |
|------|------|-------------|
| **Code Execution Tool** | 서버리스 코드 실행 환경 | 데이터 분석, 계산, 시각화 |
| **MCP Connector** | 원격 MCP 서버 직접 연결 | 외부 시스템 통합 |
| **Files API** | 세션 간 파일 저장/접근 | 문서 처리, 보고서 생성 |
| **Extended Prompt Caching** | 최대 60분 캐싱 | 반복 쿼리 비용 90% 절감 |

### 4.2 Tool Use 기본 패턴

Claude의 Tool Use는 에이전트의 핵심 빌딩 블록입니다:

```python
import anthropic

client = anthropic.Anthropic()

# 도구 정의
tools = [
    {
        "name": "get_weather",
        "description": "지정된 위치의 현재 날씨를 가져옵니다",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "도시명 (예: Seoul, Tokyo)"
                }
            },
            "required": ["location"]
        }
    }
]

# 에이전트 루프
messages = [{"role": "user", "content": "서울과 도쿄의 날씨를 비교해줘"}]

while True:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        tools=tools,
        messages=messages
    )

    # 도구 호출이 없으면 루프 종료
    if response.stop_reason == "end_turn":
        break

    # 도구 호출 처리
    for block in response.content:
        if block.type == "tool_use":
            # 실제 도구 실행
            result = execute_tool(block.name, block.input)
            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result
                }]
            })
```

### 4.3 멀티 에이전트 시스템 구성

Opus를 플래너/오케스트레이터로, Sonnet을 실행자로 사용하는 패턴:

```python
# 오케스트레이터 (Opus)
plan = client.messages.create(
    model="claude-opus-4-6",
    system="당신은 작업을 분석하고 하위 작업으로 분해하는 오케스트레이터입니다. "
           "JSON 형식으로 실행 계획을 반환하세요.",
    messages=[{"role": "user", "content": complex_task}]
)

# 워커 (Sonnet) — 각 하위 작업 병렬 실행
import asyncio

async def execute_subtask(subtask):
    return client.messages.create(
        model="claude-sonnet-4-20250514",
        system=f"당신은 {subtask['domain']} 전문가입니다.",
        tools=subtask['tools'],
        messages=[{"role": "user", "content": subtask['instruction']}]
    )

results = await asyncio.gather(*[
    execute_subtask(task) for task in plan.subtasks
])

# 오케스트레이터가 결과를 합성
final = client.messages.create(
    model="claude-opus-4-6",
    messages=[{
        "role": "user",
        "content": f"다음 하위 작업 결과를 합성하세요: {results}"
    }]
)
```

---

## 5강. Context Engineering — 에이전트 성능의 핵심

### 5.1 프롬프트 엔지니어링에서 컨텍스트 엔지니어링으로

에이전트가 루프 안에서 여러 턴의 추론을 수행할 때, 단일 프롬프트 최적화만으로는 부족합니다. **컨텍스트 엔지니어링**은 시스템 지시사항, 도구, MCP, 외부 데이터, 메시지 히스토리 등 끊임없이 변화하는 정보에서 **제한된 컨텍스트 윈도우에 무엇을 넣을지 큐레이션하는 기술**입니다.

### 5.2 핵심 전략

**전략 1: 컨텍스트 압축 (Compaction)**
대화가 길어질 때 이전 메시지를 요약하여 핵심 정보만 유지합니다.

```
[초기 지시사항] + [최근 N턴 전체] + [이전 대화 요약] + [현재 도구 결과]
```

**전략 2: 노트 테이킹 (Scratchpad)**
에이전트가 마일스톤마다 진행 노트를 작성하여 별도 저장소에 보관하고, 필요 시 참조합니다.

**전략 3: 멀티 에이전트 분리**
검색/분석 등 무거운 작업을 서브 에이전트에 위임하고, 리드 에이전트는 합성과 의사결정에 집중합니다. 세부 컨텍스트가 서브 에이전트 내에 격리되어 리드 에이전트의 컨텍스트를 오염시키지 않습니다.

**전략 4: CLAUDE.md 파일 활용**
프로젝트 루트에 CLAUDE.md를 배치하여 에이전트에게 프로젝트 컨텍스트, 코딩 규칙, 아키텍처 패턴 등을 지속적으로 주입합니다. Claude Code 실행 시 자동으로 로드됩니다.

### 5.3 Just-in-Time Context 패턴

사전에 모든 컨텍스트를 로드하지 않고, 필요한 시점에 동적으로 로드합니다:

```
Turn 1: 사용자 요청 분석 → 필요한 도구/데이터 식별
Turn 2: 식별된 도구만 로드 → 실행
Turn 3: 결과 기반으로 추가 필요 도구 식별 → 로드 → 실행
...
```

---

## 6강. Agent Skills — 에이전트의 전문 역량 정의

### 6.1 Skills 개요

2025년 말 Anthropic이 공개한 Agent Skills는 에이전트에게 **내재화된 전문 역량과 지식**을 부여하는 시스템입니다. MCP가 "외부 도구 연결"이라면, Skills는 "에이전트의 내부 전문성 정의"에 해당합니다.

- 공식 사양: agentskills.io
- Claude.ai, Claude Code, API 모두에서 동일하게 작동
- 크로스 플랫폼 호환 — 다른 AI 프레임워크에서도 채택 가능

### 6.2 MCP vs Skills

| 비교 항목 | MCP | Agent Skills |
|----------|-----|-------------|
| 역할 | 외부 시스템 연결 | 내부 전문성 정의 |
| 리소스 소비 | 도구 정의 로드 시 수만 토큰 가능 | 경량, 온디맨드 로드 |
| 상태 관리 | 프로토콜 기반 영속적 컨텍스트 | 조건 매칭 시 자동 로드 |
| 주요 사용처 | API 호출, DB 쿼리, SaaS 통합 | 문서 형식, 분석 패턴, 코딩 규칙 |

**실무에서의 최적 전략**: 두 가지를 결합합니다. Skills로 에이전트의 행동 패턴과 전문성을 정의하고, MCP로 실제 외부 시스템과의 상호작용을 처리합니다.

---

## 7강. 프로덕션 배포와 운영 (LLMOps/MLOps 관점)

### 7.1 보안 및 거버넌스

에이전트가 자율적으로 도구를 사용하므로 보안이 최우선 과제입니다:

- **최소 권한 원칙**: 에이전트에게 작업 수행에 필요한 최소한의 권한만 부여
- **작업 감사 추적**: 에이전트의 모든 단계를 로깅하고 추적
- **MCP Gateway 패턴**: 직접 서버 연결 대신 중앙화된 게이트웨이를 통해 인증, 모니터링, 정책 집행을 일원화
- **OAuth/OIDC 통합**: 기업 ID 프로바이더와의 연동

### 7.2 비용 최적화

```
비용 최적화 전략:
├── Prompt Caching: 반복 쿼리 비용 최대 90% 절감
├── 모델 티어링: Opus(플래닝) + Sonnet(실행) + Haiku(분류/필터)
├── 코드 실행 패턴: MCP 도구 호출 시 토큰 절감
├── 컨텍스트 압축: 불필요한 히스토리 제거
└── 배치 처리: Batch API로 비대화형 작업 50% 할인
```

### 7.3 신뢰성과 장애 복구

8시간 자율 작업 중 7시간 째에 실패하면 치명적입니다. 핵심 전략:

- **체크포인트 저장**: 주기적으로 상태를 저장하여 실패 시 마지막 체크포인트부터 재개
- **Graceful Degradation**: 전체 실패 대신 부분 결과 반환
- **Human-in-the-Loop**: 고위험 작업이나 불확실성 높은 시점에서 인간 확인 요청
- **자동 재시도 + 지수 백오프**: 일시적 도구 실행 실패에 대한 복원력

### 7.4 관찰 가능성 (Observability)

```
에이전트 모니터링 스택:
├── 트레이싱: 각 에이전트 턴의 도구 호출 체인 추적
├── 메트릭스: 턴 수, 토큰 사용량, 지연 시간, 성공률
├── 로깅: 의사결정 과정, 도구 입출력, 오류 상세
└── 평가: 작업 완료율, 정확도, 사용자 만족도
```

---

## 8강. 실전 활용 사례

### 8.1 코딩 에이전트 (Claude Code)

가장 성숙한 Agentic AI 활용 영역입니다:

- Rakuten: 1,250만 라인 코드베이스(vLLM)에서 7시간 자율 작업으로 99.9% 수치 정확도 달성
- TELUS: 13,000개 이상의 맞춤형 AI 솔루션 구축, 엔지니어링 코드 배포 30% 가속
- Zapier: 전 조직 89% AI 도입, 800개 이상의 내부 에이전트 배포

경험이 쌓일수록 자율성이 증가합니다. Anthropic 연구에 따르면, 50회 미만 세션의 신규 사용자는 약 20%만 완전 자동 승인을 사용하지만, 750회 이상 사용자는 40% 이상이 자동 승인을 활용합니다.

### 8.2 고객 지원 에이전트

대화 흐름과 도구 통합이 자연스럽게 결합되는 영역입니다:

```
[고객 문의] → [의도 분류 (Routing)]
  ├── 결제 문제 → CRM 조회 → 환불 처리 도구 → 응답
  ├── 기술 지원 → KB 검색 → 진단 도구 → 에스컬레이션 판단
  └── 일반 문의 → FAQ 검색 → 응답
```

### 8.3 엔터프라이즈 워크플로우 자동화

Salesforce + Claude 통합 (Agentforce)으로 CRM 레코드 업데이트, 고객 아웃리치 캠페인 실행, Slack 채널 요약 등을 에이전트가 자율적으로 수행합니다.

---

## 9강. 실무 체크리스트 및 권장 사항

### 시작할 때

1. **단순하게 시작하세요**: 프레임워크 없이 API 직접 호출부터. 대부분의 패턴은 수십 줄의 코드로 구현 가능합니다.
2. **도구 설계에 집중하세요**: Anthropic은 SWE-bench 작업에서 프롬프트보다 도구 인터페이스에 더 많은 시간을 투자했습니다.
3. **평가 체계를 먼저 구축하세요**: 에이전트 개선은 측정 가능해야 합니다.

### 확장할 때

4. **MCP로 통합을 표준화하세요**: 커스텀 통합 대신 MCP 서버를 구축하면 재사용성이 극대화됩니다.
5. **모델 티어링을 적용하세요**: 모든 작업에 Opus를 쓸 필요 없습니다.
6. **컨텍스트 엔지니어링에 투자하세요**: 프롬프트 엔지니어링을 넘어선 다음 단계입니다.

### 프로덕션에서

7. **보안을 아키텍처에 내장하세요**: 사후 추가가 아닌 설계 단계부터.
8. **Human-in-the-Loop을 유지하세요**: 완전 자율보다는 인간 감독 하의 자율이 현실적입니다.
9. **비용을 지속적으로 모니터링하세요**: 에이전트 시스템은 비용이 예측하기 어렵습니다.
10. **투명성을 확보하세요**: 사용자에게 에이전트가 어떤 패턴을 사용하고 왜 그런 결정을 내렸는지 보여주세요.

---

## 참고 자료

- Anthropic, "Building Effective Agents" — anthropic.com/research/building-effective-agents
- Anthropic, "Effective Context Engineering for AI Agents" — anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic, "Code Execution with MCP" — anthropic.com/engineering/code-execution-with-mcp
- Anthropic, "Measuring AI Agent Autonomy in Practice" — anthropic.com/research/measuring-agent-autonomy
- Anthropic, "2026 Agentic Coding Trends Report" — resources.anthropic.com
- Model Context Protocol 공식 사이트 — modelcontextprotocol.io
- Agentic AI Foundation (AAIF) — Linux Foundation
