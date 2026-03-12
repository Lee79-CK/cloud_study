# 🤖 LLM 최신 라이브러리 완전 가이드 (2025~2026)

---

## 📌 1. 라이브러리 전체 비교 Overview

| 라이브러리 | 주요 목적 | 난이도 | GitHub Stars | 특징 |
|---|---|---|---|---|
| **OpenAI SDK** | OpenAI API 직접 호출 | ⭐ 쉬움 | 25k+ | 공식 SDK, Async 지원 |
| **Anthropic SDK** | Claude API 호출 | ⭐ 쉬움 | 3k+ | Tool use, Streaming 강점 |
| **LangChain** | LLM 파이프라인 구성 | ⭐⭐⭐ 어려움 | 92k+ | 가장 넓은 생태계 |
| **LlamaIndex** | RAG / 데이터 인덱싱 | ⭐⭐ 중간 | 37k+ | 문서 기반 QA 최강 |
| **LiteLLM** | 멀티 LLM 프록시/라우팅 | ⭐ 쉬움 | 16k+ | 100+ 모델 통합 인터페이스 |
| **Instructor** | 구조화된 출력(Pydantic) | ⭐ 쉬움 | 9k+ | JSON/Structured Output 특화 |
| **DSPy** | LLM 파이프라인 최적화 | ⭐⭐⭐ 어려움 | 20k+ | 프롬프트 자동 최적화 |
| **Outlines** | Structured Generation | ⭐⭐ 중간 | 10k+ | 로컬 모델 출력 제어 |
| **Haystack** | 엔터프라이즈 RAG 파이프라인 | ⭐⭐ 중간 | 17k+ | 프로덕션 RAG 특화 |
| **Transformers (HF)** | 모델 로딩/Fine-tuning | ⭐⭐⭐ 어려움 | 135k+ | 로컬 모델의 표준 |

---

## 🔵 2. OpenAI SDK

### 기본 개념
OpenAI의 공식 Python SDK. GPT-4o, o1, o3 등의 모델을 호출하는 가장 기본적인 방법. `v1.0+` 이후 완전히 리팩터링되어 동기/비동기 클라이언트 분리.

### 주요 사용 사례

| 사용 사례 | 설명 |
|---|---|
| Chat Completion | 대화형 응답 생성 |
| Streaming | 실시간 토큰 스트리밍 |
| Function Calling / Tool Use | 외부 함수 호출 |
| Structured Outputs | JSON Schema 기반 출력 강제 |
| Embeddings | 텍스트 벡터화 |
| Vision | 이미지 + 텍스트 멀티모달 |

### 샘플 소스

```python
# pip install openai
from openai import OpenAI, AsyncOpenAI
import asyncio

client = OpenAI(api_key="YOUR_API_KEY")  # 환경변수 OPENAI_API_KEY 권장

# ── 1. 기본 Chat Completion ──────────────────────────────
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "LangChain이 뭔가요?"}
    ],
    temperature=0.7,
    max_tokens=500
)
print(response.choices[0].message.content)

# ── 2. Streaming ─────────────────────────────────────────
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Python의 장점을 설명해줘"}],
    stream=True
)
for chunk in stream:
    delta = chunk.choices[0].delta.content or ""
    print(delta, end="", flush=True)

# ── 3. Structured Output (JSON Schema) ───────────────────
from pydantic import BaseModel

class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

completion = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[{"role": "user", "content": "내일 오후 2시에 팀 미팅이 있어. 참석자는 김철수, 이영희야"}],
    response_format=CalendarEvent,
)
event = completion.choices[0].message.parsed
print(f"이벤트: {event.name}, 날짜: {event.date}")

# ── 4. Tool Calling ───────────────────────────────────────
import json

tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "특정 도시의 날씨를 가져옵니다",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "도시명"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["city"]
        }
    }
}]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "서울 날씨 어때?"}],
    tools=tools,
    tool_choice="auto"
)

tool_call = response.choices[0].message.tool_calls[0]
args = json.loads(tool_call.function.arguments)
print(f"호출된 함수: {tool_call.function.name}, 인자: {args}")

# ── 5. Async 비동기 호출 ──────────────────────────────────
async def async_call():
    aclient = AsyncOpenAI()
    tasks = [
        aclient.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": f"숫자 {i}에 대해 설명해줘"}]
        )
        for i in range(3)
    ]
    results = await asyncio.gather(*tasks)
    for r in results:
        print(r.choices[0].message.content[:50])

asyncio.run(async_call())

# ── 6. Embeddings ─────────────────────────────────────────
embedding = client.embeddings.create(
    model="text-embedding-3-small",
    input="LLMOps는 머신러닝 운영의 핵심입니다"
)
vector = embedding.data[0].embedding
print(f"벡터 차원: {len(vector)}")  # 1536
```

---

## 🟤 3. Anthropic SDK

### 기본 개념
Anthropic의 Claude 모델(Claude 3.5 Sonnet, Claude 3 Opus 등)을 호출하는 공식 SDK. Tool use, Vision, Streaming이 강력하며 `claude-sonnet-4-6`, `claude-opus-4-6` 등 최신 모델 지원.

### 주요 사용 사례

| 사용 사례 | 설명 |
|---|---|
| Messages API | 기본 대화 생성 |
| Tool Use | 함수 호출 (OpenAI의 Function Calling 대응) |
| Computer Use | 컴퓨터 자동화 (Beta) |
| Document / Vision | PDF, 이미지 분석 |
| Streaming | 실시간 응답 스트리밍 |

### 샘플 소스

```python
# pip install anthropic
import anthropic

client = anthropic.Anthropic(api_key="YOUR_ANTHROPIC_KEY")

# ── 1. 기본 메시지 ─────────────────────────────────────────
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="당신은 LLMOps 전문가입니다.",
    messages=[
        {"role": "user", "content": "RAG 파이프라인의 핵심 구성요소를 설명해줘"}
    ]
)
print(message.content[0].text)

# ── 2. Streaming ─────────────────────────────────────────
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=500,
    messages=[{"role": "user", "content": "Python async의 원리를 설명해줘"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

# ── 3. Tool Use ───────────────────────────────────────────
tools = [
    {
        "name": "search_docs",
        "description": "기술 문서를 검색합니다",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "검색 쿼리"},
                "top_k": {"type": "integer", "description": "반환할 결과 수", "default": 5}
            },
            "required": ["query"]
        }
    }
]

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "벡터 데이터베이스 사용법을 검색해줘"}]
)

# Tool use 블록 파싱
for block in response.content:
    if block.type == "tool_use":
        print(f"도구: {block.name}, 입력: {block.input}")

# ── 4. 멀티턴 대화 ────────────────────────────────────────
conversation_history = []

def chat(user_message: str) -> str:
    conversation_history.append({"role": "user", "content": user_message})
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=conversation_history
    )
    assistant_msg = response.content[0].text
    conversation_history.append({"role": "assistant", "content": assistant_msg})
    return assistant_msg

print(chat("안녕! 나는 MLOps 엔지니어야"))
print(chat("내가 방금 뭐라고 했지?"))  # 컨텍스트 유지 확인
```

---

## 🟢 4. LiteLLM

### 기본 개념
100개 이상의 LLM을 **하나의 통합 인터페이스**로 호출할 수 있는 라이브러리. OpenAI 호환 포맷으로 Anthropic, Gemini, Bedrock, Azure, Ollama 등을 동일한 코드로 호출. **LLMOps에서 필수** 도구.

### 지원 Provider 주요 목록

| Provider | 모델 예시 | prefix |
|---|---|---|
| OpenAI | gpt-4o, o3 | `gpt-4o` |
| Anthropic | claude-sonnet-4-6 | `anthropic/claude-sonnet-4-6` |
| Google Gemini | gemini-2.0-flash | `gemini/gemini-2.0-flash` |
| AWS Bedrock | llama3, claude | `bedrock/...` |
| Azure OpenAI | gpt-4o | `azure/deployment-name` |
| Ollama (로컬) | llama3, mistral | `ollama/llama3` |
| Groq | llama3-70b | `groq/llama3-70b-8192` |

### 샘플 소스

```python
# pip install litellm
import litellm
from litellm import completion, acompletion
import asyncio

# ── 1. 멀티 프로바이더 통합 호출 ──────────────────────────
providers = [
    "gpt-4o",
    "anthropic/claude-sonnet-4-6",
    "gemini/gemini-2.0-flash",
    "ollama/llama3"  # 로컬 Ollama 실행 필요
]

for model in providers[:2]:  # 처음 2개만 실행
    try:
        resp = completion(
            model=model,
            messages=[{"role": "user", "content": "Hello! Who are you?"}]
        )
        print(f"[{model}] {resp.choices[0].message.content[:80]}")
    except Exception as e:
        print(f"[{model}] 오류: {e}")

# ── 2. Fallback 설정 (자동 장애 전환) ────────────────────
litellm.set_verbose = False

def call_with_fallback(prompt: str):
    response = completion(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        fallbacks=["anthropic/claude-sonnet-4-6", "gemini/gemini-2.0-flash"],
        num_retries=2,
        timeout=30
    )
    return response.choices[0].message.content

# ── 3. Load Balancing (여러 API 키 로드밸런싱) ──────────
from litellm import Router

router = Router(
    model_list=[
        {
            "model_name": "gpt-4o",
            "litellm_params": {
                "model": "gpt-4o",
                "api_key": "KEY_1",
                "rpm": 60  # Requests per minute
            }
        },
        {
            "model_name": "gpt-4o",  # 동일 이름, 다른 키
            "litellm_params": {
                "model": "gpt-4o",
                "api_key": "KEY_2",
                "rpm": 60
            }
        }
    ],
    routing_strategy="least-busy"
)

# ── 4. 비용 추적 ──────────────────────────────────────────
resp = completion(
    model="gpt-4o",
    messages=[{"role": "user", "content": "비용 계산 테스트"}]
)
cost = litellm.completion_cost(completion_response=resp)
print(f"요청 비용: ${cost:.6f}")
print(f"입력 토큰: {resp.usage.prompt_tokens}")
print(f"출력 토큰: {resp.usage.completion_tokens}")

# ── 5. Async 병렬 호출 ───────────────────────────────────
async def parallel_completions():
    prompts = ["Python이란?", "Docker란?", "Kubernetes란?"]
    tasks = [
        acompletion(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": p}]
        )
        for p in prompts
    ]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    for p, r in zip(prompts, results):
        if not isinstance(r, Exception):
            print(f"Q: {p}\nA: {r.choices[0].message.content[:100]}\n")

asyncio.run(parallel_completions())
```

---

## 🟡 5. Instructor

### 기본 개념
LLM의 출력을 **Pydantic 모델로 자동 파싱**해주는 라이브러리. OpenAI, Anthropic, Gemini, Ollama 등 다양한 백엔드 지원. Validation, retry 자동 처리. **LLM 출력 구조화의 표준**.

### 샘플 소스

```python
# pip install instructor pydantic openai
import instructor
from openai import OpenAI
from pydantic import BaseModel, Field, field_validator
from typing import Optional
from enum import Enum

# OpenAI 클라이언트에 instructor 패치
client = instructor.from_openai(OpenAI())

# ── 1. 기본 구조화 출력 ───────────────────────────────────
class UserInfo(BaseModel):
    name: str
    age: int
    email: Optional[str] = None
    skills: list[str] = Field(description="보유 기술 목록")

user = client.chat.completions.create(
    model="gpt-4o",
    response_model=UserInfo,
    messages=[{"role": "user", "content": "김철수, 30세, DevOps 엔지니어. Python, Kubernetes, Terraform 잘함"}]
)
print(f"이름: {user.name}, 나이: {user.age}, 스킬: {user.skills}")

# ── 2. Validation이 있는 복잡한 모델 ──────────────────────
class SeverityLevel(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class Incident(BaseModel):
    title: str
    severity: SeverityLevel
    affected_services: list[str]
    estimated_impact_users: int = Field(ge=0)
    recommended_actions: list[str]
    
    @field_validator("title")
    @classmethod
    def title_not_empty(cls, v):
        if not v.strip():
            raise ValueError("제목은 비어있을 수 없습니다")
        return v

incident = client.chat.completions.create(
    model="gpt-4o",
    response_model=Incident,
    messages=[{
        "role": "user",
        "content": "프로덕션 DB 연결 오류 발생. API 서버, 인증 서비스 다운. 약 5만명 영향"
    }]
)
print(f"심각도: {incident.severity.value}")
print(f"영향 서비스: {incident.affected_services}")
print(f"권장 조치: {incident.recommended_actions}")

# ── 3. Anthropic Claude + Instructor ────────────────────
import anthropic
claude_client = instructor.from_anthropic(anthropic.Anthropic())

class CodeReview(BaseModel):
    overall_score: int = Field(ge=1, le=10)
    issues: list[str]
    suggestions: list[str]
    is_production_ready: bool

code = """
def process_data(data):
    result = []
    for i in data:
        result.append(i * 2)
    return result
"""

review = claude_client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1000,
    response_model=CodeReview,
    messages=[{"role": "user", "content": f"이 코드를 리뷰해줘:\n{code}"}]
)
print(f"점수: {review.overall_score}/10")
print(f"이슈: {review.issues}")
```

---

## 🔴 6. LangChain

### 기본 개념
LLM 파이프라인을 구성하는 가장 큰 생태계. **LCEL(LangChain Expression Language)** 기반으로 체인을 `|` 연산자로 연결. LangSmith를 통한 트레이싱/모니터링 통합.

### 핵심 컴포넌트

| 컴포넌트 | 설명 | 예시 |
|---|---|---|
| ChatModels | LLM 래퍼 | `ChatOpenAI`, `ChatAnthropic` |
| PromptTemplates | 프롬프트 템플릿 | `ChatPromptTemplate` |
| OutputParsers | 출력 파싱 | `StrOutputParser`, `JsonOutputParser` |
| Chains (LCEL) | 파이프라인 구성 | `prompt \| llm \| parser` |
| Retrievers | 문서 검색 | `VectorStoreRetriever` |
| Agents | 자율 실행 에이전트 | `create_react_agent` |
| Memory | 대화 기록 관리 | `ConversationBufferMemory` |

### 샘플 소스

```python
# pip install langchain langchain-openai langchain-community
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# ── 1. LCEL 기본 체인 ─────────────────────────────────────
prompt = ChatPromptTemplate.from_messages([
    ("system", "당신은 {role} 전문가입니다."),
    ("human", "{question}")
])
parser = StrOutputParser()

chain = prompt | llm | parser  # LCEL 파이프라인

result = chain.invoke({
    "role": "클라우드 아키텍처",
    "question": "Kubernetes와 ECS의 차이점은?"
})
print(result)

# ── 2. 병렬 체인 ─────────────────────────────────────────
pros_prompt = ChatPromptTemplate.from_template("{topic}의 장점 3가지를 알려줘")
cons_prompt = ChatPromptTemplate.from_template("{topic}의 단점 3가지를 알려줘")

parallel_chain = RunnableParallel({
    "pros": pros_prompt | llm | parser,
    "cons": cons_prompt | llm | parser
})

result = parallel_chain.invoke({"topic": "마이크로서비스 아키텍처"})
print("장점:", result["pros"])
print("단점:", result["cons"])

# ── 3. RAG 파이프라인 ─────────────────────────────────────
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings
from langchain_core.documents import Document

# 문서 준비
docs = [
    Document(page_content="LangChain은 LLM 파이프라인 구성 라이브러리입니다", metadata={"source": "docs"}),
    Document(page_content="RAG는 검색 증강 생성으로 외부 지식을 LLM에 주입합니다", metadata={"source": "docs"}),
    Document(page_content="벡터 데이터베이스는 임베딩을 저장하고 유사도 검색을 지원합니다", metadata={"source": "docs"}),
]

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(docs, embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 2})

rag_prompt = ChatPromptTemplate.from_template("""
다음 컨텍스트를 바탕으로 질문에 답해줘:

컨텍스트:
{context}

질문: {question}
""")

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | rag_prompt
    | llm
    | parser
)

print(rag_chain.invoke("RAG란 무엇인가요?"))

# ── 4. LangGraph Agent (최신 방식) ──────────────────────
# pip install langgraph
from langgraph.prebuilt import create_react_agent
from langchain_core.tools import tool

@tool
def get_system_status(service: str) -> str:
    """서비스 상태를 확인합니다"""
    status_map = {"api": "정상", "db": "경고: 응답시간 증가", "cache": "정상"}
    return status_map.get(service, "알 수 없는 서비스")

@tool
def restart_service(service: str) -> str:
    """서비스를 재시작합니다"""
    return f"{service} 서비스 재시작 완료"

agent = create_react_agent(llm, tools=[get_system_status, restart_service])

result = agent.invoke({
    "messages": [("user", "DB 서비스 상태 확인하고 이상있으면 재시작해줘")]
})
print(result["messages"][-1].content)
```

---

## 🟣 7. LlamaIndex

### 기본 개념
**데이터 연결과 RAG에 특화**된 라이브러리. 다양한 데이터 소스(PDF, CSV, DB, 웹 등)를 인덱싱하고 LLM과 연결. `QueryEngine`, `ChatEngine`, `SubQuestionQueryEngine` 등 고급 검색 기능 제공.

### 샘플 소스

```python
# pip install llama-index llama-index-llms-openai llama-index-embeddings-openai
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.core import Document
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.core.node_parser import SentenceSplitter

# 전역 설정
Settings.llm = OpenAI(model="gpt-4o", temperature=0)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
Settings.chunk_size = 512
Settings.chunk_overlap = 50

# ── 1. 기본 인덱싱 및 쿼리 ───────────────────────────────
documents = [
    Document(text="Kubernetes는 컨테이너 오케스트레이션 플랫폼입니다. Pod, Service, Deployment 등의 리소스로 구성됩니다."),
    Document(text="Helm은 Kubernetes 패키지 매니저입니다. Chart를 통해 복잡한 애플리케이션을 쉽게 배포할 수 있습니다."),
    Document(text="ArgoCD는 GitOps 기반의 Kubernetes 배포 도구입니다. Git 저장소를 단일 진실 소스로 사용합니다."),
]

index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine()

response = query_engine.query("ArgoCD가 무엇인지 설명해줘")
print(response)
print("\n소스 노드:", response.source_nodes[0].text[:100])

# ── 2. 디렉터리 기반 인덱싱 ──────────────────────────────
# reader = SimpleDirectoryReader("./docs/", recursive=True)
# documents = reader.load_data()

# ── 3. 고급 RAG: Sub-Question Query Engine ───────────────
from llama_index.core.tools import QueryEngineTool
from llama_index.core.query_engine import SubQuestionQueryEngine

k8s_docs = [Document(text="Kubernetes는 오픈소스 컨테이너 오케스트레이션 플랫폼입니다")]
docker_docs = [Document(text="Docker는 컨테이너 런타임 플랫폼입니다")]

k8s_index = VectorStoreIndex.from_documents(k8s_docs)
docker_index = VectorStoreIndex.from_documents(docker_docs)

tools = [
    QueryEngineTool.from_defaults(
        query_engine=k8s_index.as_query_engine(),
        name="kubernetes",
        description="Kubernetes 관련 정보"
    ),
    QueryEngineTool.from_defaults(
        query_engine=docker_index.as_query_engine(),
        name="docker",
        description="Docker 관련 정보"
    )
]

sub_query_engine = SubQuestionQueryEngine.from_defaults(query_engine_tools=tools)
response = sub_query_engine.query("Kubernetes와 Docker의 차이점은?")
print(response)

# ── 4. Chat Engine (대화형 RAG) ───────────────────────────
index = VectorStoreIndex.from_documents(documents)
chat_engine = index.as_chat_engine(
    chat_mode="condense_plus_context",
    verbose=True
)

r1 = chat_engine.chat("ArgoCD를 설명해줘")
r2 = chat_engine.chat("그것과 Helm의 차이는?")  # 이전 대화 기억
print(r2)
```

---

## ⚫ 8. DSPy

### 기본 개념
프롬프트를 **자동으로 최적화**하는 혁신적인 프레임워크. 직접 프롬프트를 작성하는 대신 **Signature(입출력 명세)**를 정의하면 DSPy가 자동으로 최적 프롬프트를 학습. **"프롬프트 엔지니어링의 자동화"**.

### 샘플 소스

```python
# pip install dspy
import dspy

# LLM 설정
lm = dspy.LM("openai/gpt-4o")
dspy.configure(lm=lm)

# ── 1. Signature 정의 ────────────────────────────────────
class SentimentAnalysis(dspy.Signature):
    """텍스트의 감성을 분석합니다"""
    text: str = dspy.InputField(desc="분석할 텍스트")
    sentiment: str = dspy.OutputField(desc="positive/negative/neutral 중 하나")
    confidence: float = dspy.OutputField(desc="0.0~1.0 사이의 확신도")
    reasoning: str = dspy.OutputField(desc="분석 근거")

analyzer = dspy.Predict(SentimentAnalysis)
result = analyzer(text="이 서비스는 정말 빠르고 안정적이에요!")
print(f"감성: {result.sentiment}, 확신도: {result.confidence}")
print(f"근거: {result.reasoning}")

# ── 2. Chain-of-Thought ───────────────────────────────────
class TechQA(dspy.Signature):
    """기술 질문에 단계적으로 답변합니다"""
    question: str = dspy.InputField()
    answer: str = dspy.OutputField(desc="상세한 기술적 답변")

cot_module = dspy.ChainOfThought(TechQA)
result = cot_module(question="Kubernetes에서 Pod와 Container의 차이는?")
print(result.answer)

# ── 3. RAG 모듈 구성 ─────────────────────────────────────
class RAGSignature(dspy.Signature):
    """검색된 문서를 바탕으로 질문에 답변합니다"""
    context: list[str] = dspy.InputField(desc="검색된 관련 문서들")
    question: str = dspy.InputField()
    answer: str = dspy.OutputField(desc="근거 기반의 정확한 답변")

class SimpleRAG(dspy.Module):
    def __init__(self):
        self.generate = dspy.ChainOfThought(RAGSignature)
    
    def forward(self, question: str, docs: list[str]):
        return self.generate(context=docs, question=question)

rag = SimpleRAG()
docs = [
    "MLflow는 ML 실험 추적 도구입니다",
    "MLflow는 모델 레지스트리와 배포 기능을 제공합니다"
]
result = rag(question="MLflow의 주요 기능은?", docs=docs)
print(result.answer)
```

---

## ⚙️ 9. LLMOps 실전 조합 패턴

### 권장 스택 조합

| 목적 | 권장 조합 |
|---|---|
| **프로덕션 RAG** | LlamaIndex + LiteLLM + Qdrant/Pinecone |
| **멀티 에이전트** | LangGraph + LiteLLM + LangSmith |
| **구조화 출력** | Instructor + LiteLLM |
| **실험/평가** | DSPy + LangSmith + MLflow |
| **로컬 모델** | Ollama + LiteLLM + Outlines |
| **엔터프라이즈** | Haystack + Azure OpenAI + LiteLLM |

### 실전 LLMOps 파이프라인 샘플

```python
# 실전 패턴: LiteLLM + Instructor + 비용 추적
import litellm
import instructor
from openai import OpenAI
from pydantic import BaseModel
from datetime import datetime
import json

# 비용 추적 콜백
cost_log = []

def track_cost(kwargs, completion_response, start_time, end_time):
    cost = litellm.completion_cost(completion_response=completion_response)
    cost_log.append({
        "model": kwargs.get("model"),
        "cost": cost,
        "tokens": completion_response.usage.total_tokens,
        "latency_ms": (end_time - start_time).total_seconds() * 1000,
        "timestamp": datetime.now().isoformat()
    })

litellm.success_callback = [track_cost]

# 구조화 출력 모델 정의
class IncidentReport(BaseModel):
    summary: str
    root_cause: str
    severity: str
    resolution_steps: list[str]
    estimated_downtime_minutes: int

# Instructor + LiteLLM 조합
client = instructor.from_litellm(litellm.completion)

def analyze_incident(log_text: str, model: str = "gpt-4o") -> IncidentReport:
    return client.chat.completions.create(
        model=model,
        response_model=IncidentReport,
        messages=[{
            "role": "user",
            "content": f"다음 인시던트 로그를 분석해줘:\n{log_text}"
        }],
        max_retries=3  # Instructor 자동 재시도
    )

# 사용 예시
log = """
2026-03-12 14:23:01 ERROR: Database connection timeout after 30s
2026-03-12 14:23:05 ERROR: API server health check failed (5xx rate: 85%)
2026-03-12 14:25:10 WARN: Auto-scaling triggered, adding 3 instances
2026-03-12 14:28:00 INFO: DB connection pool exhausted (max: 100)
"""

report = analyze_incident(log)
print(f"요약: {report.summary}")
print(f"원인: {report.root_cause}")
print(f"해결 단계: {report.resolution_steps}")
print(f"\n총 비용 로그: {json.dumps(cost_log, indent=2)}")
```

---

## 📊 10. 라이브러리 선택 가이드

| 상황 | 추천 라이브러리 | 이유 |
|---|---|---|
| LLM API 처음 시작 | OpenAI SDK / Anthropic SDK | 공식, 안정적, 문서 풍부 |
| 여러 LLM 동시 사용 | **LiteLLM** | 통합 인터페이스, 비용 추적 |
| JSON/구조화 출력 필요 | **Instructor** | 가장 간단하고 안정적 |
| RAG 시스템 구축 | **LlamaIndex** | 데이터 연결 최강 |
| 복잡한 에이전트 워크플로우 | **LangGraph** | 상태머신 기반 에이전트 |
| 프롬프트 자동 최적화 | **DSPy** | ML 방식의 프롬프트 학습 |
| 로컬 모델 + 출력 제어 | Outlines + Ollama | 오프라인, 비용 0 |
| 엔터프라이즈 RAG | **Haystack** | 프로덕션 성숙도 높음 |
