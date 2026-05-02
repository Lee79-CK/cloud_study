# 관찰가능성(Observability) 스택 완전 정복 강의
## Prometheus · Grafana · Loki · Thanos with NVIDIA DGX Spark

> **강의 목표**
> 이 강의를 끝마치면 여러분은 (1) 관찰가능성의 핵심 개념을 이해하고, (2) Prometheus·Grafana·Loki·Thanos 각각이 왜 필요한지 설명할 수 있으며, (3) NVIDIA DGX Spark 위에서 LLM/AI 워크로드를 모니터링하는 실제 환경을 직접 구축할 수 있게 됩니다.

> **대상 청중**
> LLMOps / MLOps / 클라우드 엔지니어, 그리고 AI 인프라를 운영하는 모든 분.

---

## 📚 목차

1. [강의 개요와 큰 그림](#1-강의-개요와-큰-그림)
2. [Part 0: 관찰가능성(Observability)이란?](#part-0-관찰가능성observability이란)
3. [Part 1: Prometheus — 메트릭 수집의 표준](#part-1-prometheus--메트릭-수집의-표준)
4. [Part 2: Grafana — 모든 데이터를 한눈에](#part-2-grafana--모든-데이터를-한눈에)
5. [Part 3: Loki — 로그를 위한 Prometheus](#part-3-loki--로그를-위한-prometheus)
6. [Part 4: Thanos — Prometheus의 한계를 넘어서](#part-4-thanos--prometheus의-한계를-넘어서)
7. [Part 5: 네 가지 도구의 통합 아키텍처](#part-5-네-가지-도구의-통합-아키텍처)
8. [Part 6: NVIDIA DGX Spark 실습](#part-6-nvidia-dgx-spark-실습)
9. [부록 A: 트러블슈팅](#부록-a-트러블슈팅)
10. [부록 B: 다음 학습 단계](#부록-b-다음-학습-단계)

---

## 1. 강의 개요와 큰 그림

### 왜 이 도구들을 함께 배워야 하나요?

여러분이 LLM 추론 서비스를 DGX Spark 위에서 운영한다고 상상해 봅시다. 어느 날 사용자가 "응답이 느려졌다"라고 말합니다. 여러분은 다음을 알아야 합니다.

- **GPU가 얼마나 일하고 있나?** → 메트릭(metric)이 필요
- **어떤 요청에서 에러가 났나?** → 로그(log)가 필요
- **장기 추세는 어떤가?** → 시계열을 오랫동안 보관해야 함
- **이 모든 걸 한 화면에 보여달라** → 시각화 도구가 필요

이 네 가지 요구사항이 정확히 우리가 다룰 네 가지 도구와 매핑됩니다.

| 요구사항 | 도구 | 한 줄 요약 |
|---|---|---|
| 메트릭 수집·저장 | **Prometheus** | 시계열 데이터베이스 + 수집 엔진 |
| 시각화 | **Grafana** | 대시보드의 사실상 표준 |
| 로그 수집·검색 | **Loki** | 비싸지 않은 로그 시스템 |
| 장기 보관·고가용성 | **Thanos** | Prometheus를 무한히 확장 |

이 네 가지는 **CNCF(Cloud Native Computing Foundation) 생태계**의 핵심 요소이고, 클라우드 네이티브 모니터링의 사실상 표준입니다.

### 강의 진행 방식

각 도구를 다음 순서로 다룹니다.

1. **개념(Concept)** — 이 도구가 풀려는 문제는 무엇인가
2. **아키텍처(Architecture)** — 내부적으로 어떻게 동작하는가
3. **사용법(Usage)** — 실제로 어떻게 쓰는가
4. **모범 사례(Best Practice)** — 무엇을 조심해야 하는가

마지막에 DGX Spark에서 직접 손으로 만져보는 실습이 이어집니다.

---

## Part 0: 관찰가능성(Observability)이란?

### 모니터링과 관찰가능성의 차이

많은 분이 두 용어를 혼용하지만, 미묘하지만 중요한 차이가 있습니다.

- **모니터링(Monitoring)** — 우리가 **이미 알고 있는** 문제를 감시. 예: "CPU가 90%를 넘으면 알려줘."
- **관찰가능성(Observability)** — 우리가 **아직 모르는** 문제까지 진단할 수 있는 능력. 예: "어제 새벽 3시에 왜 갑자기 latency가 튀었는지 그 원인을 추적할 수 있어야 한다."

LLM/AI 시스템은 매우 복잡합니다. GPU·CPU·메모리·네트워크·모델·배치 크기·KV 캐시 등 수많은 요인이 얽혀 있어서, 단순한 모니터링으로는 부족합니다. **관찰가능성**이 필요한 이유입니다.

### 세 기둥 (Three Pillars of Observability)

관찰가능성은 전통적으로 세 가지 데이터 종류로 구성됩니다.

```
            ┌─────────────────────────────────────┐
            │       Observability                 │
            ├──────────┬───────────┬──────────────┤
            │ Metrics  │   Logs    │   Traces     │
            │ (숫자)   │  (텍스트) │  (요청 흐름) │
            ├──────────┼───────────┼──────────────┤
            │Prometheus│   Loki    │ Tempo/Jaeger │
            │ + Thanos │           │              │
            └──────────┴───────────┴──────────────┘
                       │
                  ┌────┴─────┐
                  │ Grafana  │  ← 모두 시각화
                  └──────────┘
```

이번 강의에서는 **Metrics**와 **Logs**에 집중합니다. (Traces는 다른 강의 주제로 미루어 둡니다.)

### 데이터의 종류, 좀 더 자세히

| 종류 | 형태 | 예시 | 용도 |
|---|---|---|---|
| **메트릭** | (시간, 숫자) 쌍 | `gpu_utilization{gpu="0"} = 85` | 추세, 알림 |
| **로그** | 타임스탬프 + 텍스트 | `2026-05-02 12:34:56 ERROR CUDA OOM` | 디버깅, 사후 분석 |
| **트레이스** | 분산 요청 경로 | `request_id=abc → service-A → service-B` | 분산 시스템 디버깅 |

**핵심 통찰**: 메트릭은 "무슨 일이 일어났는지(what)"를 빠르게, 로그는 "왜 일어났는지(why)"를 자세히 알려줍니다. 둘은 서로 보완 관계입니다.

---

## Part 1: Prometheus — 메트릭 수집의 표준

### 1.1 Prometheus가 풀려는 문제

기존의 모니터링 시스템(Nagios, Zabbix 등)은 두 가지 한계가 있었습니다.

1. **정적 구성** — 서버 한 대 추가하려면 설정 파일을 일일이 고쳐야 함
2. **단순한 데이터 모델** — 메트릭에 임의의 라벨을 붙일 수 없음

Prometheus는 SoundCloud에서 2012년 개발되었고, 다음 두 가지를 핵심 가치로 삼았습니다.

- **다차원 데이터 모델** — `gpu_temperature{gpu="0", host="dgx-01"}`처럼 라벨로 차원을 추가
- **풍부한 쿼리 언어 PromQL** — SQL과 비슷하지만 시계열에 특화

오늘날 Prometheus는 **CNCF Graduated 프로젝트**(가장 성숙한 단계)이며, Kubernetes 모니터링의 사실상 표준입니다.

### 1.2 데이터 모델 — Prometheus의 심장

Prometheus의 모든 데이터는 다음 형태로 표현됩니다.

```
<메트릭 이름>{<라벨>=<값>, ...} <숫자 값> <타임스탬프>
```

실제 예시:

```
http_requests_total{method="POST", endpoint="/api/v1/chat", status="200"} 1027 @1714600000
```

이걸 풀어 쓰면 "method가 POST이고, endpoint가 /api/v1/chat이며, 응답 코드가 200인 HTTP 요청이, 2024-05-02 시점에 누적 1027번 일어났다"는 뜻입니다.

#### 메트릭의 4가지 타입

| 타입 | 설명 | 예시 |
|---|---|---|
| **Counter** | 증가만 함(서버 재시작 시 0으로 리셋) | 누적 요청 수, 에러 수 |
| **Gauge** | 오르내림 가능 | 현재 메모리 사용량, GPU 온도 |
| **Histogram** | 구간별 분포 + 합계 + 개수 | 응답 시간 분포 |
| **Summary** | 분위수(quantile)를 클라이언트에서 계산 | p99 latency |

> 💡 **꿀팁**: 분포 측정에는 거의 항상 **Histogram**을 쓰세요. Histogram은 여러 인스턴스에서 집계가 가능하지만 Summary는 그렇지 않습니다.

### 1.3 아키텍처 — Pull 모델

Prometheus의 가장 큰 특징은 **Pull 기반**이라는 점입니다. 즉, 서버들이 데이터를 보내는 게 아니라, Prometheus가 주기적으로 가서 가져옵니다.

```
                        ┌──────────────┐
                        │  Prometheus  │
                        │   (서버)     │
                        └──────┬───────┘
                               │ HTTP GET /metrics
                               │ (15초마다)
                ┌──────────────┼──────────────┐
                ▼              ▼              ▼
         ┌───────────┐  ┌──────────────┐  ┌──────────────┐
         │  앱 1     │  │ Node Exporter│  │ DCGM Exporter│
         │ /metrics  │  │  (CPU/메모리)│  │  (GPU 메트릭)│
         └───────────┘  └──────────────┘  └──────────────┘
```

#### 왜 Pull인가?

- **모니터링 대상이 살아있는지 자동으로 확인**됩니다(scrape 실패 = 서비스 다운)
- 모든 대상이 동일한 형식으로 `/metrics` 엔드포인트를 노출하면 됩니다
- 디버깅이 쉽습니다(브라우저로 `/metrics`를 열어 보면 됨)

#### 단, 예외 — Pushgateway

배치 잡(batch job)처럼 짧게 실행되고 사라지는 작업은 Pull로 잡기 어렵습니다. 이때는 **Pushgateway**라는 중계 서버에 잠깐 푸시해 두면, Prometheus가 거기서 가져갑니다.

### 1.4 핵심 컴포넌트

```
                  ┌────────────────────────┐
                  │   Prometheus Server    │
                  ├────────────────────────┤
                  │ • Retrieval (스크레이프)│
                  │ • TSDB (저장)          │
                  │ • PromQL Engine        │
                  │ • HTTP API             │
                  └────────────────────────┘
                          ▲     │
                Scrape    │     │ Query / Alert
                          │     ▼
                ┌─────────┴───┐ ┌──────────────┐
                │  Exporters  │ │ Alertmanager │
                │  / Apps     │ │              │
                └─────────────┘ └──────────────┘
```

- **Retrieval** — 스크레이프 작업을 수행
- **TSDB** — 자체 시계열 데이터베이스 (로컬 디스크)
- **PromQL Engine** — 쿼리 엔진
- **Alertmanager** — 알림 라우팅·억제·그룹화 (별도 컴포넌트)
- **Exporters** — 외부 시스템의 메트릭을 Prometheus 형식으로 변환 (예: node_exporter, dcgm-exporter)

### 1.5 PromQL 입문

PromQL은 처음엔 낯설지만, 패턴을 익히면 매우 강력합니다.

#### 기본 셀렉터

```promql
# 모든 GPU의 utilization
DCGM_FI_DEV_GPU_UTIL

# 특정 GPU만
DCGM_FI_DEV_GPU_UTIL{gpu="0"}

# 정규식
DCGM_FI_DEV_GPU_UTIL{gpu=~"0|1"}
```

#### 시간 범위와 집계

```promql
# 지난 5분간 HTTP 요청률 (초당 요청 수)
rate(http_requests_total[5m])

# 호스트별 평균 GPU 활용률
avg by (instance) (DCGM_FI_DEV_GPU_UTIL)

# 분당 에러율
sum(rate(http_requests_total{status=~"5.."}[1m]))
  / sum(rate(http_requests_total[1m]))
```

#### 자주 쓰는 함수 5가지

| 함수 | 용도 | 예시 |
|---|---|---|
| `rate()` | 카운터의 초당 증가율 | `rate(requests_total[5m])` |
| `irate()` | 직전 두 점만으로 계산(급변 감지) | `irate(requests_total[5m])` |
| `increase()` | 시간 구간 동안의 총 증가량 | `increase(errors_total[1h])` |
| `histogram_quantile()` | Histogram에서 분위수 추출 | `histogram_quantile(0.99, ...)` |
| `predict_linear()` | 선형 예측 (디스크 용량 등) | `predict_linear(disk_free[1h], 4*3600)` |

> ⚠️ **흔한 함정**: Counter에 직접 `sum()`을 쓰면 안 됩니다. 반드시 `rate()`를 먼저 적용하세요. 카운터는 재시작 시 0으로 리셋되므로 단순 합계는 의미가 없습니다.

### 1.6 알림(Alerting)

Prometheus는 알림 **규칙 평가**까지만 담당하고, 실제 알림 **전송**은 Alertmanager가 합니다.

규칙 예시:

```yaml
groups:
  - name: gpu_alerts
    rules:
      - alert: GPUHighTemperature
        expr: DCGM_FI_DEV_GPU_TEMP > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "GPU {{ $labels.gpu }} 온도가 {{ $value }}°C입니다"
```

`for: 2m`이 핵심입니다. 일시적인 스파이크로 알림이 울리지 않도록 **2분 동안 지속**될 때만 발화시킵니다.

### 1.7 Prometheus의 한계

이 한계들이 곧 Thanos가 등장한 이유입니다.

1. **단일 노드** — 기본적으로 수평 확장이 안 됨
2. **로컬 스토리지** — 보통 15일치만 저장, 장기 보관이 어려움
3. **고가용성(HA) 어려움** — HA 페어를 운영하려면 별도 작업 필요
4. **글로벌 뷰 부재** — 여러 클러스터의 메트릭을 한 번에 쿼리하기 어려움

---

## Part 2: Grafana — 모든 데이터를 한눈에

### 2.1 Grafana가 풀려는 문제

Prometheus도 자체 웹 UI가 있지만, 그래프 하나 그리는 정도입니다. 우리는 보통 다음을 원합니다.

- 여러 메트릭을 **하나의 대시보드**에 모으기
- **메트릭 + 로그 + 트레이스**를 같은 화면에서 보기
- **여러 데이터 소스**(Prometheus, Loki, Postgres 등)를 동시에 쿼리
- 보기 좋고, 공유 가능한 시각화

Grafana는 이 모든 것을 해결합니다. **데이터 소스 중립적인 시각화 플랫폼**입니다.

### 2.2 핵심 개념

#### Data Source

Grafana는 데이터를 **저장하지 않습니다**. 항상 외부 데이터 소스에 쿼리를 던지고 결과를 받아 그립니다.

지원 데이터 소스 예: Prometheus, Loki, Tempo, InfluxDB, Elasticsearch, MySQL, PostgreSQL, CloudWatch, Google Cloud Monitoring, ...

#### Dashboard / Panel / Query

```
Dashboard
  └── Panel (그래프 한 개)
       └── Query (데이터 소스에 던지는 질문)
            └── Visualization (Time series, Bar chart, Stat, ...)
```

#### Variables

대시보드 상단에서 드롭다운으로 선택하는 변수입니다. 예를 들어 `$gpu`라는 변수를 만들고 패널 쿼리에 `{gpu="$gpu"}`라고 쓰면, 사용자가 GPU를 바꿀 때마다 그래프가 다시 그려집니다.

#### Alerting (Unified Alerting)

Grafana 8 이후로는 Grafana 자체에서도 알림을 만들 수 있습니다. Prometheus의 Alertmanager와도 연동됩니다.

### 2.3 사용법 — 첫 대시보드 만들기

1. **Add data source** → Prometheus 선택 → URL 입력 (예: `http://prometheus:9090`)
2. **+ → Dashboard → Add visualization**
3. 데이터 소스 = Prometheus
4. Query 입력: `rate(http_requests_total[5m])`
5. 우측 패널에서 **Time series** 선택, 단위·축 설정
6. **Save**

이게 전부입니다. 5분이면 첫 대시보드가 나옵니다.

### 2.4 Grafana의 강력한 기능

#### Explore 모드

대시보드를 만들기 전, 데이터를 빠르게 탐색할 수 있는 모드입니다. **Loki와 Prometheus를 한 화면에 띄워 놓고**, 메트릭 이상점을 발견했을 때 같은 시간대의 로그를 바로 확인할 수 있습니다.

#### Provisioning (코드형 대시보드)

대시보드를 JSON 또는 YAML로 정의해서 **GitOps 방식**으로 관리할 수 있습니다.

```yaml
# /etc/grafana/provisioning/dashboards/gpu.yaml
apiVersion: 1
providers:
  - name: 'gpu-dashboards'
    folder: 'GPU'
    type: file
    options:
      path: /var/lib/grafana/dashboards/gpu
```

#### 공식 대시보드 라이브러리

[grafana.com/dashboards](https://grafana.com/grafana/dashboards/)에 수천 개의 대시보드가 이미 있습니다. 우리는 NVIDIA의 공식 대시보드 ID **12239**를 곧 사용할 겁니다.

### 2.5 모범 사례

- 대시보드 하나에 패널은 **20개 이하**로 (브라우저가 느려집니다)
- 모든 패널에 **단위(unit)** 설정 (bytes, percent, seconds 등)
- 변수는 **Multi-value + All option**으로 유연하게
- **Annotation**으로 배포·인시던트 시점 표시

---

## Part 3: Loki — 로그를 위한 Prometheus

### 3.1 Loki가 풀려는 문제

전통적인 로그 솔루션(ELK 스택 = Elasticsearch + Logstash + Kibana)은 매우 강력하지만, **비쌉니다**. 왜냐하면 모든 로그 본문을 **풀텍스트 색인**하기 때문입니다.

Grafana Labs는 발상을 뒤집었습니다.

> "로그 본문 전체를 인덱싱하지 말고, **메타데이터(라벨)만** 인덱싱하자. 본문은 그냥 압축 저장하고, 검색은 grep처럼 하자."

이게 Loki의 핵심 철학입니다. 이름도 "Like Prometheus, but for logs"의 줄임말로, **Prometheus와 똑같은 라벨 기반 모델**을 씁니다.

### 3.2 결과: 비용 절감

- Elasticsearch: 모든 토큰을 인덱싱 → CPU/디스크 비쌈
- Loki: 라벨만 인덱싱, 본문은 청크로 압축 → 매우 저렴
- 트레이드오프: 임의 텍스트 검색은 **약간 느림**

LLM/AI 워크로드에서는 보통 로그가 매우 많이 생기지만(GPU 텔레메트리, 추론 요청 로그), 라벨로 잘 분류하면 Loki로 충분합니다.

### 3.3 아키텍처

```
   Application Logs
        │
        ▼
┌───────────────┐    push     ┌──────────────────┐
│   Promtail    │ ─────────►  │   Loki Server    │
│ (Agent)       │  (HTTP)     │                  │
└───────────────┘             ├──────────────────┤
        │                     │  • Distributor   │
        │                     │  • Ingester      │
   File tail / Journal /      │  • Querier       │
   Docker / Kubernetes        │  • Index/Chunks  │
                              └──────────────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │  Object Storage  │
                              │ (S3 / GCS / 로컬)│
                              └──────────────────┘
```

#### 주요 컴포넌트

| 컴포넌트 | 역할 |
|---|---|
| **Promtail** | 로그 수집 에이전트(Loki 전용). 파일·journald·Docker·K8s 로그를 읽어 라벨을 붙여 전송 |
| **Distributor** | 들어온 로그를 검증하고 ingester로 분산 |
| **Ingester** | 인메모리 청크에 쌓다가 일정 크기/시간에 도달하면 오브젝트 스토리지로 플러시 |
| **Querier** | 쿼리(LogQL) 처리 |
| **Compactor** | 인덱스 통합·중복 제거 |

> 💡 **대안 에이전트**: 최근에는 Promtail 대신 **Grafana Alloy**(여러 텔레메트리를 통합 수집)나 **Vector**가 자주 쓰입니다.

### 3.4 LogQL 입문

LogQL은 PromQL과 매우 비슷한 문법을 가집니다.

#### 기본 — 라벨로 스트림 선택

```logql
{app="vllm-server", level="error"}
```

#### 텍스트 필터 (grep처럼)

```logql
{app="vllm-server"} |= "OutOfMemory"          # 포함
{app="vllm-server"} != "DEBUG"                # 미포함
{app="vllm-server"} |~ "OOM|crash"            # 정규식
```

#### 파싱(JSON, logfmt)

```logql
{app="vllm-server"} | json | duration_ms > 1000
```

JSON 로그였다면 위 한 줄로 본문 안의 `duration_ms` 필드를 꺼내서 필터까지 할 수 있습니다.

#### 메트릭으로 변환

LogQL의 진짜 강력함은 로그를 **메트릭으로 변환**할 수 있다는 점입니다.

```logql
# 분당 ERROR 로그 발생 수
sum(rate({app="vllm-server"} |= "ERROR" [1m]))
```

이 쿼리 결과를 Grafana 패널에 넣으면, **로그를 그래프로** 볼 수 있습니다. 알림도 만들 수 있습니다.

### 3.5 모범 사례

#### 라벨 카디널리티 주의!!!

이게 Loki에서 가장 중요한 규칙입니다.

- ❌ **나쁨**: `request_id`, `trace_id`, `user_id` 같은 고유한 값들을 라벨로 → 카디널리티 폭발
- ✅ **좋음**: `app`, `env`, `level`, `cluster`처럼 값의 가짓수가 적은 것을 라벨로
- 💡 고유 값으로 검색하고 싶다면 → **본문에 두고** `|= "request_id=abc123"`로 grep

라벨이 너무 많아지면 인덱스가 망가지고 Loki가 느려집니다.

---

## Part 4: Thanos — Prometheus의 한계를 넘어서

### 4.1 Thanos가 풀려는 문제

Part 1의 끝에서 본 Prometheus의 4가지 한계, 기억나시죠?

1. 단일 노드 (수평 확장 X)
2. 로컬 스토리지 (장기 보관 X)
3. HA 어려움
4. 글로벌 뷰 부재

Thanos는 이 모두를 해결합니다. **Prometheus를 교체하는 게 아니라 옆에 붙여서 확장**합니다. 이게 핵심 디자인 철학입니다.

### 4.2 핵심 아이디어

> "Prometheus는 그대로 둔 채, **오브젝트 스토리지(S3 등)**에 데이터를 추가로 적재하고, 쿼리는 모든 데이터를 통합해서 본다."

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Prometheus #1   │    │  Prometheus #2   │    │  Prometheus #3   │
│  (Cluster A)     │    │  (Cluster B)     │    │  (Cluster C)     │
│  + Sidecar       │    │  + Sidecar       │    │  + Sidecar       │
└────────┬─────────┘    └────────┬─────────┘    └────────┬─────────┘
         │                       │                       │
         │   2시간마다 업로드     │                       │
         ▼                       ▼                       ▼
        ┌────────────────────────────────────────────────┐
        │      Object Storage (S3 / GCS / MinIO)         │
        └────────────────────────────────────────────────┘
                              ▲
                              │ 읽기
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
  ┌──────────┐          ┌──────────┐          ┌──────────┐
  │  Store   │          │ Compactor│          │  Ruler   │
  │ Gateway  │          │          │          │          │
  └──────────┘          └──────────┘          └──────────┘
        ▲
        │
        │     ┌─────────────────────┐
        └─────│      Querier         │  ← 사용자/Grafana
              │  (PromQL 호환 API)   │
              └─────────────────────┘
```

### 4.3 핵심 컴포넌트

| 컴포넌트 | 역할 |
|---|---|
| **Sidecar** | 각 Prometheus 옆에 붙어서, 2시간마다 TSDB 블록을 오브젝트 스토리지에 업로드. 또한 라이브 쿼리 게이트웨이 역할도 함 |
| **Store Gateway** | 오브젝트 스토리지의 데이터를 PromQL로 읽을 수 있게 해주는 컴포넌트 |
| **Querier** | 여러 Sidecar/Store에서 데이터를 모아 통합 쿼리. **사용자는 이 API만 알면 됨** (Prometheus와 호환) |
| **Compactor** | 오래된 데이터를 다운샘플링·압축 (예: 1년 전 데이터는 5분 해상도로) |
| **Ruler** | 알림·기록 규칙을 평가 |
| **Receive** | (대안 모드) Prometheus가 직접 푸시하는 형태 |

### 4.4 Sidecar vs Receive — 두 가지 모드

Thanos는 두 가지 모드로 운영할 수 있습니다.

#### Sidecar 모드 (전통적)
- Prometheus가 **자기 디스크에 쓴 후**, Sidecar가 그걸 S3로 업로드
- 장점: 단순, Prometheus는 그대로 동작
- 단점: 2시간 단위로만 업로드(블록 단위), 로컬 디스크가 충분해야 함

#### Receive 모드
- Prometheus가 remote write로 **실시간으로 Thanos에 푸시**
- 장점: 로컬 스토리지 거의 불필요, 멀티 테넌시 용이
- 단점: 운영 복잡도 ↑

> **선택 기준**: 처음 시작은 Sidecar 모드, 멀티 테넌시·SaaS 형태가 필요하면 Receive.

### 4.5 다운샘플링 — 비용을 절감하는 비결

1년치 메트릭을 15초 해상도로 보관하면 어마어마한 용량이 됩니다. Thanos Compactor는 다음과 같이 다운샘플링합니다.

| 데이터 나이 | 해상도 |
|---|---|
| Raw (최근) | 15초 |
| 5분 다운샘플 | 40시간 후 |
| 1시간 다운샘플 | 10일 후 |

쿼리 시 Querier는 자동으로 적절한 해상도를 선택합니다. 사용자는 이걸 의식할 필요가 없습니다.

### 4.6 언제 Thanos가 필요한가?

- 여러 Prometheus를 통합해서 보고 싶다 → **YES**
- 메트릭을 1년 이상 보관해야 한다 → **YES**
- HA가 필수다 → **YES**
- 단일 작은 환경(예: 개인 DGX Spark) → **굳이 필요 없음**

> **현실적 조언**: 이번 실습에서는 학습 목적으로 Thanos를 켜 보지만, 실제 단일 DGX Spark 환경에서는 Prometheus만으로 충분합니다. 본격적인 클러스터로 확장할 때를 대비한 학습입니다.

### 4.7 Mimir, Cortex — Thanos의 형제들

같은 문제를 푸는 다른 프로젝트도 있습니다.

- **Cortex** — Thanos보다 먼저 시작, multi-tenant SaaS에 적합
- **Mimir** — Grafana Labs가 만든 Cortex 후속작, 운영성 개선

세 가지 모두 Prometheus 호환입니다. **선택 기준은 운영 철학**입니다. Thanos는 "Prometheus와 가까이", Mimir는 "통합된 단일 스택"을 추구합니다.

---

## Part 5: 네 가지 도구의 통합 아키텍처

이제 큰 그림을 봅시다. 네 도구가 어떻게 함께 일하나요?

```
┌─────────────────────────────────────────────────────────────────┐
│                        DGX Spark                                │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ vLLM / Ollama│  │ Node Exporter│  │  DCGM Exporter       │ │
│  │ (앱 메트릭)  │  │  (시스템)    │  │  (GPU 메트릭)        │ │
│  │ /metrics     │  │ /metrics     │  │  /metrics (9400)     │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │
│         │                 │                     │             │
│         └─────────────────┴─────────────────────┘             │
│                           │                                   │
│                  scrape ▼                                     │
│                  ┌────────────────┐                           │
│                  │   Prometheus   │                           │
│                  │   (TSDB 로컬)  │                           │
│                  └────┬───────────┘                           │
│                       │                                       │
│            +──────────┴────────────────+                      │
│            │                           │                      │
│            ▼                           ▼                      │
│     ┌──────────────┐           ┌──────────────┐              │
│     │ Thanos       │           │              │              │
│     │ Sidecar      │           │              │              │
│     └──────┬───────┘           │              │              │
│            │                   │              │              │
│            ▼                   │              │              │
│     ┌──────────────┐           │              │              │
│     │ MinIO (S3)   │           │              │              │
│     └──────┬───────┘           │              │              │
│            │                   │              │              │
│            ▼                   │              │              │
│     ┌──────────────┐           │              │              │
│     │ Thanos Store │           │              │              │
│     └──────┬───────┘           │              │              │
│            │                   │              │              │
│            ▼                   │              │              │
│     ┌──────────────┐           │              │              │
│     │Thanos Querier│           │              │              │
│     └──────┬───────┘           │              │              │
│            │                   │              │              │
│            │                   │              │              │
│            │  ┌──────────────┐ │              │              │
│            │  │  Promtail    │─┘ 로그 수집    │              │
│            │  └──────┬───────┘                │              │
│            │         │                        │              │
│            │         ▼                        │              │
│            │   ┌──────────┐                   │              │
│            │   │   Loki   │                   │              │
│            │   └─────┬────┘                   │              │
│            │         │                        │              │
│            └────┬────┴────────────────────────┘              │
│                 │                                            │
│                 ▼                                            │
│          ┌──────────────┐                                    │
│          │   Grafana    │ ◄── 사용자                         │
│          │  Dashboards  │                                    │
│          └──────────────┘                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 데이터 흐름 정리

1. **메트릭**: Apps/Exporters → Prometheus → (Thanos Sidecar → MinIO → Store) → Thanos Querier → Grafana
2. **로그**: Apps → Promtail → Loki → Grafana
3. **사용자**: Grafana 한 곳만 접속하면 모든 데이터에 접근

이 구조의 우아함은, **Grafana에서 Thanos Querier를 데이터 소스로 등록하면 Prometheus와 똑같이 동작**한다는 점입니다. PromQL 호환이기 때문입니다.

---

## Part 6: NVIDIA DGX Spark 실습

### 6.1 사전 정보 — DGX Spark의 특수성

NVIDIA DGX Spark은 일반 x86 서버와 다른 몇 가지 특징이 있습니다.

| 항목 | 사양 |
|---|---|
| CPU | 20-core ARM (Cortex-X925 ×10 + Cortex-A725 ×10) |
| GPU | NVIDIA GB10 Grace Blackwell (Blackwell 아키텍처) |
| Memory | 128GB LPDDR5X 통합 메모리 (CPU/GPU 공유) |
| OS | DGX OS (Ubuntu 24.04 LTS 기반) |
| **아키텍처** | **ARM64 (aarch64)** |

**핵심 포인트**: DGX Spark은 **ARM64**입니다. 그래서 컨테이너 이미지를 가져올 때 **multi-arch 이미지**(또는 `linux/arm64` 태그)를 써야 합니다. 다행히 우리가 쓸 모든 도구(Prometheus, Grafana, Loki, DCGM Exporter)는 공식 ARM64 이미지를 제공합니다.

### 6.2 실습 시나리오

이런 시나리오를 가정합니다.

> **상황**: DGX Spark에서 Ollama로 로컬 LLM(예: Llama 3 70B)을 서빙하고 있습니다. GPU 활용률, 시스템 자원, 추론 로그를 통합 모니터링해야 합니다.

### 6.3 사전 준비

#### 6.3.1 Docker와 NVIDIA Container Toolkit 확인

DGX OS에는 이미 설치되어 있을 가능성이 높지만, 확인해 봅시다.

```bash
# Docker 동작 확인
docker version

# NVIDIA 런타임 확인
nvidia-smi

# 컨테이너에서 GPU 접근 테스트
docker run --rm --gpus all nvcr.io/nvidia/cuda:12.6.0-base-ubuntu22.04 nvidia-smi
```

마지막 명령이 `nvidia-smi` 출력을 보여 주면 준비 완료입니다.

#### 6.3.2 작업 디렉터리 생성

```bash
mkdir -p ~/observability && cd ~/observability
mkdir -p prometheus grafana/provisioning/datasources grafana/provisioning/dashboards loki promtail thanos minio
```

### 6.4 Step 1: Prometheus + DCGM Exporter (메트릭 기반 닦기)

#### 6.4.1 Prometheus 설정

```bash
cat > ~/observability/prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: dgx-spark-01
    replica: A

scrape_configs:
  # Prometheus 자기 자신
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter - 시스템 메트릭
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # DCGM Exporter - GPU 메트릭
  - job_name: 'dcgm'
    static_configs:
      - targets: ['dcgm-exporter:9400']
EOF
```

> **주의**: `external_labels`는 **Thanos에서 매우 중요**합니다. 여러 Prometheus를 구분하는 키이고, replica 라벨은 HA 페어를 식별합니다.

#### 6.4.2 docker-compose.yml — 기본 메트릭 스택

```bash
cat > ~/observability/docker-compose.yml <<'EOF'
version: '3.8'

networks:
  obs:
    driver: bridge

services:

  # ─── 메트릭 수집 ───
  prometheus:
    image: prom/prometheus:v3.4.1
    container_name: prometheus
    restart: unless-stopped
    networks: [obs]
    ports: ["9090:9090"]
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--storage.tsdb.min-block-duration=2h'
      - '--storage.tsdb.max-block-duration=2h'  # Thanos를 위해 동일하게
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus

  node-exporter:
    image: prom/node-exporter:v1.8.2
    container_name: node-exporter
    restart: unless-stopped
    networks: [obs]
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'

  dcgm-exporter:
    image: nvcr.io/nvidia/k8s/dcgm-exporter:4.5.2-4.8.1-ubuntu22.04
    container_name: dcgm-exporter
    restart: unless-stopped
    networks: [obs]
    runtime: nvidia
    cap_add: [SYS_ADMIN]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

volumes:
  prometheus_data: {}
EOF
```

#### 6.4.3 실행 및 확인

```bash
cd ~/observability
docker compose up -d
docker compose ps

# Prometheus 타겟 상태 확인 (모두 UP이어야 함)
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job:.labels.job, health}'

# DCGM이 GPU 메트릭을 노출하는지 확인
curl -s http://localhost:9400/metrics | grep -E '^DCGM_FI_DEV_GPU_(UTIL|TEMP)' | head
```

> 💡 GB10 GPU의 경우 `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_GPU_TEMP`, `DCGM_FI_DEV_FB_USED` 같은 메트릭이 보여야 합니다.

브라우저에서 `http://<DGX-Spark-IP>:9090`에 접속하면 Prometheus UI가 보입니다. 쿼리창에 다음을 입력해 보세요:

```promql
DCGM_FI_DEV_GPU_UTIL
```

### 6.5 Step 2: Grafana 연결 및 GPU 대시보드

#### 6.5.1 Grafana 데이터 소스 자동 프로비저닝

```bash
cat > ~/observability/grafana/provisioning/datasources/datasources.yml <<'EOF'
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true
  - name: Thanos
    type: prometheus
    access: proxy
    url: http://thanos-querier:10902
    editable: true
EOF
```

#### 6.5.2 NVIDIA 공식 GPU 대시보드 (ID: 12239)

```bash
cat > ~/observability/grafana/provisioning/dashboards/dashboards.yml <<'EOF'
apiVersion: 1
providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
EOF

# NVIDIA 공식 DCGM 대시보드 다운로드
mkdir -p ~/observability/grafana/dashboards
curl -fsSL https://grafana.com/api/dashboards/12239/revisions/3/download \
  -o ~/observability/grafana/dashboards/nvidia-dcgm.json
```

#### 6.5.3 docker-compose.yml에 Grafana 추가

`~/observability/docker-compose.yml`의 `services:` 아래에 다음을 **추가**합니다.

```yaml
  grafana:
    image: grafana/grafana:12.0.1
    container_name: grafana
    restart: unless-stopped
    networks: [obs]
    ports: ["3000:3000"]
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
      - grafana_data:/var/lib/grafana
```

`volumes:` 섹션에도 추가:

```yaml
volumes:
  prometheus_data: {}
  grafana_data: {}
```

재시작:

```bash
docker compose up -d grafana
```

`http://<DGX-Spark-IP>:3000` 접속 → 로그인 (admin/admin) → Dashboards에서 NVIDIA DCGM 대시보드를 확인합니다. 각 GPU의 활용률·온도·전력·메모리 사용량이 실시간으로 보입니다.

#### 6.5.4 Ollama 부하 걸어 보기 (선택)

```bash
# Ollama가 실행 중이라면, 추론 요청을 던져서 GPU가 일하는 모습을 봅시다
curl http://localhost:11434/api/generate -d '{
  "model": "llama3:8b",
  "prompt": "Explain Prometheus in 100 words",
  "stream": false
}'
```

Grafana에서 GPU Utilization이 치솟는 모습을 보세요. 🚀

### 6.6 Step 3: Loki + Promtail (로그 통합)

#### 6.6.1 Loki 설정

```bash
cat > ~/observability/loki/loki-config.yml <<'EOF'
auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /loki

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    chunks_directory: /loki/chunks
    rules_directory: /loki/rules
  tsdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/index_cache

limits_config:
  retention_period: 168h   # 7일
  reject_old_samples: true
  reject_old_samples_max_age: 168h

compactor:
  working_directory: /loki/compactor
  delete_request_store: filesystem
  retention_enabled: true
EOF
```

#### 6.6.2 Promtail 설정

```bash
cat > ~/observability/promtail/promtail-config.yml <<'EOF'
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker-containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'

  - job_name: system
    static_configs:
      - targets: [localhost]
        labels:
          job: syslog
          host: dgx-spark
          __path__: /var/log/syslog
EOF
```

#### 6.6.3 docker-compose.yml에 추가

```yaml
  loki:
    image: grafana/loki:3.5.0
    container_name: loki
    restart: unless-stopped
    networks: [obs]
    ports: ["3100:3100"]
    command: -config.file=/etc/loki/loki-config.yml
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml:ro
      - loki_data:/loki

  promtail:
    image: grafana/promtail:3.5.0
    container_name: promtail
    restart: unless-stopped
    networks: [obs]
    command: -config.file=/etc/promtail/promtail-config.yml
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/log:/var/log:ro
    depends_on: [loki]
```

`volumes:`에 `loki_data: {}` 추가.

```bash
docker compose up -d loki promtail
```

#### 6.6.4 Grafana에서 로그 확인

Grafana → Explore → 데이터 소스를 **Loki**로 변경 → 다음 쿼리:

```logql
{container="dcgm-exporter"}
```

DCGM Exporter의 로그가 보이면 성공입니다. 다음 쿼리로 에러만 필터링:

```logql
{container=~".+"} |= "error" or "ERROR"
```

> 💡 **메트릭 + 로그 상관관계**: Grafana의 강력한 기능 중 하나는 메트릭 그래프에서 시간대를 선택하면 같은 시간대의 로그를 바로 보여 준다는 점입니다. Explore에서 **Split** 버튼으로 Prometheus와 Loki를 좌우에 띄워 보세요.

### 6.7 Step 4: Thanos (장기 보관 + 글로벌 뷰)

#### 6.7.1 MinIO (로컬 S3 호환 스토리지)

```bash
cat > ~/observability/minio/init.sh <<'EOF'
#!/bin/sh
sleep 5
mc alias set local http://minio:9000 minioadmin minioadmin
mc mb -p local/thanos || true
EOF
chmod +x ~/observability/minio/init.sh
```

#### 6.7.2 Thanos S3 설정

```bash
cat > ~/observability/thanos/objstore.yml <<'EOF'
type: s3
config:
  bucket: thanos
  endpoint: minio:9000
  access_key: minioadmin
  secret_key: minioadmin
  insecure: true
EOF
```

#### 6.7.3 docker-compose.yml에 추가

```yaml
  minio:
    image: minio/minio:RELEASE.2025-04-22T22-12-26Z
    container_name: minio
    restart: unless-stopped
    networks: [obs]
    ports: ["9000:9000", "9001:9001"]
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    volumes: [minio_data:/data]

  minio-init:
    image: minio/mc:RELEASE.2025-04-16T18-13-26Z
    container_name: minio-init
    networks: [obs]
    depends_on: [minio]
    entrypoint: /bin/sh -c "/init.sh"
    volumes:
      - ./minio/init.sh:/init.sh:ro

  thanos-sidecar:
    image: quay.io/thanos/thanos:v0.39.0
    container_name: thanos-sidecar
    restart: unless-stopped
    networks: [obs]
    command:
      - sidecar
      - --tsdb.path=/prometheus
      - --prometheus.url=http://prometheus:9090
      - --objstore.config-file=/etc/thanos/objstore.yml
      - --grpc-address=0.0.0.0:10901
      - --http-address=0.0.0.0:10902
    volumes:
      - prometheus_data:/prometheus
      - ./thanos/objstore.yml:/etc/thanos/objstore.yml:ro
    depends_on: [prometheus, minio-init]

  thanos-store:
    image: quay.io/thanos/thanos:v0.39.0
    container_name: thanos-store
    restart: unless-stopped
    networks: [obs]
    command:
      - store
      - --data-dir=/data
      - --objstore.config-file=/etc/thanos/objstore.yml
      - --grpc-address=0.0.0.0:10901
      - --http-address=0.0.0.0:10902
    volumes:
      - thanos_store_data:/data
      - ./thanos/objstore.yml:/etc/thanos/objstore.yml:ro
    depends_on: [minio-init]

  thanos-querier:
    image: quay.io/thanos/thanos:v0.39.0
    container_name: thanos-querier
    restart: unless-stopped
    networks: [obs]
    ports: ["10902:10902"]
    command:
      - query
      - --http-address=0.0.0.0:10902
      - --grpc-address=0.0.0.0:10901
      - --query.replica-label=replica
      - --endpoint=thanos-sidecar:10901
      - --endpoint=thanos-store:10901
    depends_on: [thanos-sidecar, thanos-store]

  thanos-compactor:
    image: quay.io/thanos/thanos:v0.39.0
    container_name: thanos-compactor
    restart: unless-stopped
    networks: [obs]
    command:
      - compact
      - --data-dir=/data
      - --objstore.config-file=/etc/thanos/objstore.yml
      - --wait
      - --retention.resolution-raw=30d
      - --retention.resolution-5m=180d
      - --retention.resolution-1h=2y
    volumes:
      - thanos_compactor_data:/data
      - ./thanos/objstore.yml:/etc/thanos/objstore.yml:ro
    depends_on: [minio-init]
```

`volumes:`에 다음을 추가:

```yaml
  minio_data: {}
  thanos_store_data: {}
  thanos_compactor_data: {}
  loki_data: {}
```

#### 6.7.4 실행 및 확인

```bash
docker compose up -d
docker compose ps

# Thanos Querier UI
# 브라우저: http://<DGX-Spark-IP>:10902

# MinIO 콘솔
# 브라우저: http://<DGX-Spark-IP>:9001 (minioadmin/minioadmin)
```

2시간 후 MinIO 버킷 `thanos`에 첫 블록이 업로드됩니다(혹은 Prometheus를 재시작하면 즉시 업로드됨).

#### 6.7.5 Grafana에서 Thanos 데이터 소스 사용

Grafana → 좌측 메뉴 → Connections → Data sources → Thanos가 이미 등록되어 있을 겁니다.

이제 **Thanos를 데이터 소스로 선택**하면, 동일한 PromQL 쿼리를 던져도 **로컬 + 오브젝트 스토리지의 모든 기간** 데이터를 함께 봅니다.

### 6.8 Step 5: 알림 규칙 한 가지 만들어 보기

GPU가 너무 뜨거우면 알림을 받아 봅시다.

```bash
cat > ~/observability/prometheus/alerts.yml <<'EOF'
groups:
  - name: dgx_spark_gpu
    interval: 30s
    rules:
      - alert: GPUHighTemperature
        expr: DCGM_FI_DEV_GPU_TEMP > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "DGX Spark GPU {{ $labels.gpu }} 온도가 높습니다"
          description: "{{ $value }}°C가 2분 이상 지속됨"

      - alert: GPUMemoryAlmostFull
        expr: |
          (DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)) > 0.95
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "GPU {{ $labels.gpu }} 메모리 95% 이상 사용 중"
EOF
```

`prometheus.yml`의 맨 위쪽에 다음을 추가:

```yaml
rule_files:
  - /etc/prometheus/alerts.yml
```

`docker-compose.yml`의 prometheus 서비스 volumes에 추가:

```yaml
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
```

재시작 후 Prometheus UI의 **Alerts** 탭에서 규칙이 평가되는 걸 볼 수 있습니다.

> 실제 알림을 Slack/이메일로 보내려면 **Alertmanager**를 추가로 띄워야 하는데, 그건 다음 강의 주제로 미루어 둡니다.

### 6.9 정리 (Cleanup)

```bash
cd ~/observability
docker compose down       # 컨테이너만 정지
docker compose down -v    # 데이터까지 모두 삭제
```

---

## 부록 A: 트러블슈팅

### A.1 DCGM Exporter가 GPU를 못 찾을 때

```bash
# NVIDIA Container Toolkit 재설정
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 컨테이너 안에서 GPU 확인
docker exec -it dcgm-exporter nvidia-smi
```

### A.2 ARM64에서 이미지 풀(pull) 실패

`no matching manifest for linux/arm64` 에러가 뜨면, 해당 이미지가 ARM64를 지원하지 않는다는 뜻입니다.

- 다른 태그 시도(예: `:latest` 대신 `:main` 등)
- 공식 이미지인지 확인 (커뮤니티 이미지는 ARM 미지원이 흔함)
- 우리가 쓴 모든 이미지(prom/prometheus, grafana/grafana, grafana/loki, minio/minio, quay.io/thanos/thanos, nvcr.io/nvidia/k8s/dcgm-exporter)는 ARM64 multi-arch 지원합니다

### A.3 Prometheus에서 타겟이 DOWN

- `docker network inspect observability_obs`로 네트워크 확인
- 컨테이너 이름이 hostname (예: `dcgm-exporter:9400`) — 같은 네트워크 안이라면 DNS로 찾아짐
- 방화벽: DGX OS의 `ufw` 상태 확인 (`sudo ufw status`)

### A.4 Loki에서 라벨 카디널리티 경고

`too many active streams` 에러는 라벨 종류가 너무 많다는 신호입니다. Promtail의 `relabel_configs`를 점검해서, 고유 ID 같은 값이 라벨로 들어가지 않도록 합니다.

### A.5 Thanos Sidecar가 블록을 업로드 안 함

- Prometheus의 `--storage.tsdb.min-block-duration`과 `--storage.tsdb.max-block-duration`이 **모두 2h**여야 함 (이게 Thanos의 가장 흔한 함정입니다)
- `external_labels`가 설정되어 있어야 함 (cluster, replica)
- MinIO의 `thanos` 버킷이 존재해야 함

---

## 부록 B: 다음 학습 단계

이 강의를 졸업한 분께 추천하는 다음 주제들입니다.

### B.1 Kubernetes로 옮기기 — kube-prometheus-stack
실서비스에서는 `kube-prometheus-stack` Helm 차트를 씁니다. 이 차트는 Prometheus Operator, Grafana, kube-state-metrics, node-exporter, Alertmanager를 한 번에 설치해 줍니다. NVIDIA의 GPU Operator와 함께 쓰면 GPU 모니터링이 자동화됩니다.

### B.2 분산 트레이싱 — Tempo + OpenTelemetry
LLM 서빙 파이프라인은 보통 여러 서비스로 구성됩니다(라우터 → 토크나이저 → vLLM → 후처리). 각 요청이 어디서 시간을 보내는지 추적하려면 **분산 트레이싱**이 필요합니다. Grafana **Tempo**가 Loki·Prometheus와 같은 철학으로 설계된 트레이싱 백엔드입니다.

### B.3 Alertmanager 심화
- Slack, PagerDuty, Email 라우팅
- Silence(점검 중 알림 억제)
- Inhibition(상위 알림이 울릴 때 하위 알림 억제)
- 그룹화 정책

### B.4 LLM 특화 메트릭 — vLLM, TGI, Ollama
- **vLLM**: `/metrics` 엔드포인트에서 `vllm:num_requests_running`, `vllm:gpu_cache_usage_perc`, `vllm:e2e_request_latency_seconds`
- **TGI**(Text Generation Inference): `tgi_request_inference_duration`
- LLM 특화 대시보드 만들기

### B.5 Grafana Mimir / VictoriaMetrics
Thanos 외에 같은 문제를 푸는 대안 시스템들. 운영 철학과 성능 특성이 다릅니다.

### B.6 SLO와 에러 예산(Error Budget)
- "99.9% 가용성"을 SLI/SLO로 측정
- Multi-burn-rate 알림
- Sloth, Pyrra 같은 SLO 도구

---

## 마무리: 이 강의에서 가져갈 핵심 메시지

1. **관찰가능성은 모니터링의 진화**입니다. 알고 있는 문제만 감시하는 게 아니라, 모르는 문제까지 진단할 수 있어야 합니다.
2. **메트릭(Prometheus)과 로그(Loki)는 보완 관계**입니다. 무슨 일이 일어났는가(메트릭) → 왜 일어났는가(로그).
3. **Grafana는 데이터를 저장하지 않습니다**. 시각화 레이어일 뿐, 데이터는 항상 외부에 있습니다.
4. **Thanos는 Prometheus를 대체하지 않습니다**. 옆에 붙어서 한계를 확장합니다.
5. **DGX Spark은 ARM64**입니다. 이미지를 선택할 때 multi-arch 지원을 확인하는 습관을 기르세요.
6. **카디널리티는 적**입니다. Prometheus 라벨도, Loki 라벨도, 가짓수가 적은 것을 골라야 시스템이 건강합니다.
7. **시작은 단순하게, 필요할 때 확장하세요**. 처음부터 Thanos를 운영할 필요 없습니다. Prometheus 단독 → 필요 시 Thanos.

이 강의가 여러분의 AI 인프라 운영에 단단한 기반이 되길 바랍니다. 즐거운 모니터링 되세요! 🚀

---

*이 문서는 학습용 강의 자료입니다. 실제 프로덕션 환경에 적용하기 전에 보안(인증, TLS, RBAC), 백업, 용량 산정을 반드시 검토하세요.*
