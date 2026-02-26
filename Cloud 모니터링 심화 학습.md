# AWS · Azure · Kubernetes 모니터링/관제 체계
## 고급 아키텍처, 원리, 사용사례 및 빅테크 면접 Q&A
> 대상 독자: Senior Cloud / MLOps / LLMOps Engineer  
> 작성 수준: 박사급 기술 논문 스타일

---

# 제1부: 모니터링/관제의 이론적 기반

## 1.1 관측가능성(Observability)의 철학적 토대

관측가능성(Observability)은 제어 이론(Control Theory)에서 유래한 개념으로, 시스템의 **내부 상태를 외부 출력만으로 완전히 추론할 수 있는 정도**를 의미합니다. 1960년대 Rudolf Kálmán이 선형 동적 시스템의 관측가능성 조건을 정립한 이래, 이 개념은 현대 분산 시스템 공학에 그대로 이식되었습니다.

분산 클라우드 환경에서 관측가능성은 세 가지 기둥(Three Pillars)으로 구성됩니다. **Metrics(메트릭)**는 시계열 기반의 집계 수치로, 시스템 건강도를 거시적으로 표현합니다. **Logs(로그)**는 이산적 이벤트 기록으로, 인과 관계와 문맥을 제공합니다. **Traces(트레이스)**는 분산 요청의 전체 경로를 추적하여 지연 원인을 특정합니다. 최근에는 여기에 **Profiles(프로파일)** — CPU/메모리 사용의 시계열 플레임 그래프 — 과 **Exceptions(예외/이벤트)** 를 포함한 다섯 기둥(Five Pillars) 모델로 확장되고 있습니다.

그러나 Charity Majors(Honeycomb CTO)가 주창하는 **"Observability 2.0"** 패러다임은 이 세 기둥을 넘어서, 고카디널리티(High Cardinality)·고차원성(High Dimensionality)을 갖춘 **구조화된 이벤트(Structured Event)** 단 하나의 primitive로 모든 관측 요구를 충족할 수 있다고 주장합니다. 이는 특히 LLM/AI 워크로드에서 토큰 단위 지연, 프롬프트 패턴별 오류율 등 초고카디널리티 속성을 추적해야 하는 LLMOps 문제와 직결됩니다.

## 1.2 SLI, SLO, SLA의 수학적 정의와 에러 버짓 관리

**SLI(Service Level Indicator)**는 특정 서비스 동작의 측정 가능한 양적 지표입니다. 가용성 SLI는 다음과 같이 수식으로 정의됩니다.

```
SLI_availability = (good_requests / valid_requests) × 100
```

**SLO(Service Level Objective)**는 SLI에 대한 목표값 범위입니다. 예를 들어 "99.9%의 요청이 200ms 이내에 완료"가 하나의 SLO입니다. SLO는 단순한 목표치가 아니라 **신뢰 구간 내에서의 통계적 약속**이며, 이를 위반하지 않으면서도 충분한 이너바 공간(innovation budget)을 확보하는 것이 SRE 엔지니어링의 핵심 과제입니다.

**에러 버짓(Error Budget)**은 SLO의 역(inverse)으로 계산됩니다. 99.9% SLO라면 한 달(30일) 기준 에러 버짓은 43.2분입니다. Google SRE에서 창안한 이 개념은 **신뢰성 투자와 기능 개발 속도 사이의 정량적 트레이드오프**를 제공합니다. 에러 버짓이 소진될 경우, 새로운 배포를 동결하고 신뢰성 작업에만 집중하는 정책을 자동으로 집행할 수 있습니다.

## 1.3 분산 시스템에서의 관측가능성 난제

**분산 추적(Distributed Tracing)**에서 가장 어려운 문제 중 하나는 **인과성 보존(Causality Preservation)**입니다. 논리 시계(Logical Clock) — Lamport Timestamp, Vector Clock — 를 사용하지 않으면 비동기 이벤트 간의 happens-before 관계를 정확히 복원할 수 없습니다. OpenTelemetry의 W3C TraceContext 표준은 이를 위해 `traceparent` 헤더에 trace-id, parent-span-id, trace-flags를 인코딩하여 컨텍스트를 전파합니다.

**샘플링 딜레마(Sampling Dilemma)**도 중요한 난제입니다. 100% 트레이싱은 Observability 인프라 비용이 프로덕션 워크로드 비용의 30~50%에 달할 수 있습니다. Head-based Sampling은 요청 진입 시점에 샘플 여부를 결정하므로 오류가 발생한 희귀한 긴 꼬리(long tail) 요청을 놓칠 수 있습니다. 이를 극복하기 위해 **Tail-based Sampling** — 요청 완료 후 전체 데이터를 수집한 뒤 이상 패턴(오류, 고지연)을 기준으로 보존 여부를 결정 — 이 등장했으며, OpenTelemetry Collector의 `tail_sampling` 프로세서가 이를 구현합니다.

---

# 제2부: AWS 모니터링/관제 체계

## 2.1 CloudWatch의 아키텍처와 심층 구조

Amazon CloudWatch는 단순한 메트릭 수집 도구가 아닌, AWS의 전사적 관측가능성 패브릭(Observability Fabric)입니다. 내부적으로 CloudWatch는 **시계열 데이터베이스(TSDB)**와 **로그 집계 플랫폼**, 그리고 **이벤트 라우팅 버스**의 세 레이어로 분리 운영됩니다.

CloudWatch Metrics의 해상도는 기본 1분 단위이며, High-Resolution Metrics를 활성화하면 1초 단위까지 수집 가능합니다. 메트릭은 **Namespace → Dimension → MetricName** 의 계층 구조로 조직됩니다. 예를 들어 `AWS/ECS` 네임스페이스 하의 `ServiceName=payment-service, ClusterName=prod` 차원 조합은 서비스 단위 CPU 사용률을 고유하게 식별합니다.

**CloudWatch Metrics Insights**는 메트릭에 대한 SQL 유사 쿼리 언어를 제공하여, 수천 개의 인스턴스에서 상위 N개의 이상 메트릭을 동적으로 집계할 수 있습니다. 예를 들어 다음 쿼리는 CPU 사용률 상위 10개 인스턴스를 반환합니다.

```sql
SELECT AVG(CPUUtilization)
FROM "AWS/EC2"
GROUP BY InstanceId
ORDER BY AVG() DESC
LIMIT 10
```

**CloudWatch Anomaly Detection**은 ML 기반으로 메트릭의 계절성(seasonality), 추세(trend), 잡음(noise)을 자동으로 모델링하여 동적 임계값을 생성합니다. 이는 전통적인 정적 임계값(Static Threshold) 알람의 한계 — 주중/주말, 배포 이전/이후의 메트릭 패턴 차이를 고려하지 못하는 문제 — 를 극복합니다.

## 2.2 CloudWatch Logs Insights와 대규모 로그 분석

CloudWatch Logs Insights는 Presto 기반의 쿼리 엔진을 사용하며, 서버리스 방식으로 로그 그룹을 스캔합니다. 파싱(parse), 필터(filter), 통계(stats), 정렬(sort), 제한(limit) 커맨드를 조합하여 복잡한 로그 분석 파이프라인을 구성할 수 있습니다.

실제 프로덕션에서의 고급 사용 패턴으로, **P99 지연 시간 추이 분석**을 들 수 있습니다.

```
fields @timestamp, @message
| filter @message like /POST \/api\/payment/
| parse @message "duration=* ms" as duration
| stats pct(duration, 99) as p99_latency by bin(5m)
| sort @timestamp asc
```

대용량 로그 환경에서는 CloudWatch Logs를 S3로 내보낸 후 Amazon Athena와 AWS Glue를 결합하는 **로그 레이크(Log Lake)** 아키텍처가 더 비용 효율적입니다. Parquet 형식으로 컬럼형 저장 후, 파티션 프루닝(Partition Pruning)을 활용하면 스캔 비용을 최대 90% 절감할 수 있습니다.

## 2.3 AWS X-Ray와 분산 추적의 실전 구성

AWS X-Ray는 서비스 맵(Service Map)을 통해 마이크로서비스 간 의존관계와 지연 분포를 시각화합니다. X-Ray SDK는 `AWSXRayRecorder`를 통해 코드 레벨 계측을 지원하며, AWS Lambda, ECS, EKS에 사이드카 데몬으로 배포됩니다.

X-Ray의 한계는 **샘플링 제어의 세밀도 부족**과 **커스텀 속성(Annotation)의 카디널리티 제한**입니다. 이를 극복하기 위해 대부분의 엔터프라이즈는 **OpenTelemetry → AWS Distro for OpenTelemetry(ADOT) → X-Ray 또는 Jaeger** 형태의 벤더 중립적 파이프라인을 구성합니다. ADOT Collector는 OTLP 형식으로 수신하여 X-Ray 프로토콜로 변환하거나, Prometheus Remote Write로 CloudWatch로 포워딩하는 이중 경로를 지원합니다.

## 2.4 Amazon Managed Grafana와 통합 대시보드 체계

Amazon Managed Grafana(AMG)는 VPC Private Link를 통해 CloudWatch, X-Ray, Prometheus, Elasticsearch, Timestream, Athena 등 다수의 데이터 소스를 단일 대시보드로 통합합니다. AWS IAM과의 네이티브 통합으로 별도 Grafana 사용자 관리 없이 SSO를 구현할 수 있습니다.

**멀티 어카운트 모니터링 아키텍처**에서는 각 워크로드 어카운트(Spoke Account)에서 CloudWatch 메트릭과 로그를 중앙 모니터링 어카운트(Hub Account)로 집계하는 패턴이 표준입니다. 이를 위해 CloudWatch Cross-Account Observability 기능을 활성화하면, Hub Account의 AMG에서 Spoke Account의 리소스를 직접 쿼리할 수 있습니다.

## 2.5 AWS 고급 알람 및 사고 대응 자동화

**Composite Alarm(복합 알람)**은 여러 알람의 논리 조합(`AND`, `OR`, `NOT`)으로 복잡한 상황 인식을 구현합니다. 예를 들어 "CPU > 80% AND NetworkOut < 임계값 AND 최근 30분 내 배포 없음"을 만족할 때만 PagerDuty를 트리거하면, 배포 중 발생하는 예상된 스파이크로 인한 **알람 피로(Alert Fatigue)**를 크게 줄일 수 있습니다.

**EventBridge + Lambda + SSM Automation**을 연결한 **자가 치유(Self-Healing) 아키텍처**는 다음 흐름으로 구성됩니다. CloudWatch Alarm → EventBridge Rule → Lambda(진단 로직) → SSM Automation Document(치유 작업: 인스턴스 재시작, 오토스케일링 트리거, 캐시 플러시 등). 이때 Lambda의 진단 로직은 단순 조건 분기가 아닌, 최근 배포 이력·의존성 그래프·과거 장애 패턴을 종합하는 **인과 추론(Causal Inference)** 엔진으로 발전하는 추세입니다.

---

# 제3부: Azure 모니터링/관제 체계

## 3.1 Azure Monitor의 데이터 플레인 아키텍처

Azure Monitor는 수집(Collect), 저장(Store), 분석(Analyze), 시각화(Visualize), 대응(Respond)의 다섯 단계 파이프라인으로 구성됩니다. 핵심 저장소는 **Azure Monitor Metrics**(경량 TSDB, 93일 보존)와 **Log Analytics Workspace**(Kusto 기반 로그 데이터베이스, 최대 2년 보존)의 이중 구조입니다.

데이터 수집 경로는 크게 세 가지입니다. Azure 리소스가 자동으로 발행하는 **플랫폼 메트릭(Platform Metrics)**, 에이전트(Azure Monitor Agent, 구 MMA/OMS)가 수집하는 **게스트 메트릭 및 로그**, 그리고 애플리케이션 SDK가 전송하는 **Application Insights 텔레메트리**입니다.

**Data Collection Rules(DCR)**은 Azure Monitor의 최신 수집 구성 메커니즘으로, 어떤 소스에서 어떤 데이터를 어떤 대상으로 흘릴지를 선언적으로 정의합니다. DCR은 ARM 템플릿·Bicep·Terraform으로 관리할 수 있어 GitOps 기반의 Observability-as-Code를 실현합니다.

## 3.2 Kusto Query Language(KQL)의 심화 활용

KQL은 Apache Kafka 및 Azure Data Explorer에서 파생된 파이프라인 기반 쿼리 언어입니다. SQL 대비 시계열 및 스트리밍 데이터 처리에 최적화되어 있으며, `summarize`, `join`, `mv-expand`, `series_decompose_anomalies` 등의 함수가 특히 강력합니다.

다음은 KQL로 **배포 이후 오류율 급등을 감지**하는 쿼리입니다.

```kql
let deploymentTime = datetime(2025-01-15 14:00:00);
let windowBefore = 30m;
let windowAfter = 30m;

let beforeDeploy = requests
    | where timestamp between ((deploymentTime - windowBefore) .. deploymentTime)
    | summarize errorsBefore = countif(success == false), totalBefore = count();

let afterDeploy = requests
    | where timestamp between (deploymentTime .. (deploymentTime + windowAfter))
    | summarize errorsAfter = countif(success == false), totalAfter = count();

beforeDeploy
| join kind=fullouter afterDeploy on $left.$right
| extend errorRateBefore = 100.0 * errorsBefore / totalBefore
| extend errorRateAfter = 100.0 * errorsAfter / totalAfter
| extend rateIncrease = errorRateAfter - errorRateBefore
| project errorRateBefore, errorRateAfter, rateIncrease
```

**`series_decompose_anomalies`** 함수는 시계열에서 계절성과 트렌드를 분리한 후 잔차(residual)에서 이상값을 탐지하는 STL(Seasonal-Trend decomposition using Loess) 알고리즘을 내장합니다.

## 3.3 Application Insights의 분산 추적과 스마트 감지

Application Insights는 OpenTelemetry와의 통합을 강화하여 OTLP 인제스트를 지원합니다. **Application Map**은 X-Ray의 서비스 맵과 유사하게 컴포넌트 간 호출 관계와 SLA 상태를 시각화하며, 특히 실패 의존성(Failed Dependency)을 빨간색으로 강조 표시합니다.

**Smart Detection(스마트 감지)**은 Azure의 독자적인 ML 기반 이상 탐지 기능으로, 장애 이상(Failure Anomalies), 성능 이상(Performance Anomalies), 메모리 누수, 잠재적 보안 문제(비정상 사용 패턴)를 자동으로 학습·탐지합니다. 이는 Rule-based Alerting의 한계를 넘어 **비지도 학습(Unsupervised Learning)** 기반의 자율 모니터링을 실현합니다.

## 3.4 Azure 모니터링의 엔터프라이즈 패턴: AMPLS

대규모 엔터프라이즈에서 Azure Monitor 데이터가 퍼블릭 인터넷을 경유하는 것은 컴플라이언스 위험을 초래합니다. **Azure Monitor Private Link Scope(AMPLS)**는 Log Analytics Workspace와 Application Insights를 프라이빗 엔드포인트(Private Endpoint)를 통해 VNet 내에서만 접근 가능하게 격리합니다. AMPLS는 DNS 오버라이드를 통해 `*.ods.opinsights.azure.com` 등의 엔드포인트를 프라이빗 IP로 해석하게 하며, 이를 통해 데이터 유출(Data Exfiltration) 경로를 원천 차단합니다.

## 3.5 Azure Workbooks와 Dashboards의 고급 구성

Azure Workbooks는 KQL 쿼리, ARM API 호출, 파라미터 연동, 조건부 가시성(Conditional Visibility)을 지원하는 **인터랙티브 리포팅 플랫폼**입니다. 예를 들어 드롭다운으로 환경(dev/staging/prod)과 시간 범위를 선택하면 모든 쿼리가 자동으로 재실행되는 동적 SRE 대시보드를 코드 없이 구성할 수 있습니다. Workbook 템플릿은 ARM 템플릿으로 관리되어 IaC 파이프라인으로 배포·버전 관리가 가능합니다.

---

# 제4부: Kubernetes 모니터링/관제 체계

## 4.1 Kubernetes 메트릭 계층 모델

Kubernetes의 메트릭은 세 개의 계층으로 구성됩니다. **인프라 계층**은 노드의 CPU, 메모리, 디스크, 네트워크를 포함하며 node_exporter가 수집합니다. **오케스트레이션 계층**은 Pod 상태, 컨테이너 리소스 사용, Deployment 상태 등을 포함하며 kube-state-metrics와 metrics-server가 제공합니다. **애플리케이션 계층**은 서비스별 요청률, 오류율, 지연시간(RED 메트릭)으로 구성되며 애플리케이션이 직접 Prometheus 형식으로 노출합니다.

이 세 계층을 연결하는 핵심 개념이 **USE(Utilization, Saturation, Errors)** 메서드(Brendan Gregg 제안)와 **RED(Rate, Errors, Duration)** 메서드(Tom Wilkie 제안)입니다. USE는 인프라 병목 진단에, RED는 서비스 품질 평가에 각각 특화되어 있으며, 현장에서는 두 방법론을 계층적으로 결합합니다.

## 4.2 Prometheus의 아키텍처와 고가용성 설계

Prometheus는 **Pull 기반 수집 모델**을 채택하여, 타겟(Target)의 HTTP `/metrics` 엔드포인트를 주기적으로 스크래핑합니다. 이 모델은 Push 방식 대비 다음과 같은 장점을 가집니다. 먼저, 모니터링 시스템이 타겟의 상태를 직접 통제하므로 타겟 장애 시 즉각 감지가 가능합니다. 또한 타겟이 모니터링 시스템의 주소를 알 필요가 없어 동적 환경에서의 구성 복잡성이 낮습니다.

그러나 단일 Prometheus 인스턴스는 **수평 확장(Horizontal Scaling)**이 불가능하다는 근본적 한계를 갖습니다. 이를 해결하는 아키텍처는 다음과 같이 발전했습니다.

**Thanos 아키텍처**는 Prometheus 사이드카(Thanos Sidecar)가 Prometheus의 TSDB 블록을 S3/GCS/Azure Blob으로 주기적으로 업로드하고, Thanos Query가 여러 Prometheus 인스턴스와 오브젝트 스토리지에 걸쳐 통합 쿼리를 실행합니다. Thanos Compactor는 백그라운드에서 블록을 다운샘플링하여 장기 쿼리 성능을 유지합니다. 이를 통해 사실상 무제한의 메트릭 보존(Unlimited Retention)과 글로벌 뷰(Global View)를 달성합니다.

**Cortex/Mimir 아키텍처**는 완전 수평 확장형 멀티 테넌트 Prometheus 호환 TSDB입니다. Grafana Mimir는 Cortex의 후속으로, Distributor → Ingester → Compactor → Store-Gateway의 파이프라인으로 데이터를 처리합니다. 수십억 개의 활성 시계열(Active Series)을 지원하며, 카산드라나 DynamoDB 대신 오브젝트 스토리지만으로 운영 가능하도록 단순화되었습니다.

## 4.3 PromQL 심화: 레코딩 룰과 이상 탐지

PromQL의 **레코딩 룰(Recording Rules)**은 자주 실행되는 비용이 높은 쿼리를 사전 계산하여 저장하는 메커니즘입니다. 다음은 5분 단위 요청 성공률을 레코딩 룰로 등록하는 예시입니다.

```yaml
groups:
  - name: http_sli_rules
    interval: 30s
    rules:
      - record: job:http_requests:success_rate5m
        expr: |
          sum(rate(http_requests_total{status=~"2.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)
```

**이상 탐지**를 위한 고급 PromQL 기법으로는 `predict_linear` 함수를 사용한 **리소스 소진 예측(Resource Exhaustion Prediction)**이 있습니다.

```promql
# 6시간 후 디스크 풀 여부 예측
predict_linear(node_filesystem_avail_bytes[1h], 6 * 3600) < 0
```

또한 `holt_winters` 함수는 이중 지수 평활법(Double Exponential Smoothing)을 적용하여 계절성이 없는 시계열의 단기 예측값을 생성하며, 예측값과 실제값의 편차가 임계값을 초과할 때 알람을 트리거하는 통계적 이상 탐지에 활용됩니다.

## 4.4 Grafana Loki와 로그 집계 아키텍처

Loki는 "Prometheus for Logs"를 표방하며, 로그 내용을 인덱싱하는 대신 레이블(Label)만 인덱싱하는 설계를 채택합니다. 이는 Elasticsearch 대비 인덱스 스토리지 비용을 90% 이상 절감하지만, 로그 내용 기반의 풀-텍스트 검색은 지원하지 않습니다. 따라서 구조화되지 않은 로그의 임의 텍스트 검색이 중요한 경우에는 여전히 Elasticsearch가 유리합니다.

**LogQL**은 Loki의 쿼리 언어로, 레이블 셀렉터 → 로그 필터 → 파서 → 집계함수의 파이프라인으로 구성됩니다. 다음 예시는 Pod 재시작 원인을 분석합니다.

```logql
{namespace="production", container="payment-api"}
  |= "OOMKilled"
  | json
  | line_format "Pod {{.pod}} was OOMKilled at {{.timestamp}} with limit {{.memory_limit}}"
  | count_over_time[5m]
```

## 4.5 OpenTelemetry Operator와 자동 계측(Auto-Instrumentation)

OpenTelemetry Operator는 Kubernetes Operator 패턴으로 Collector와 Instrumentation 리소스를 선언적으로 관리합니다. 특히 `Instrumentation` CR(Custom Resource)를 통한 **사이드카 자동 주입(Sidecar Auto-Injection)**은, 애플리케이션 코드를 수정하지 않고 Java, Python, Node.js, .NET 애플리케이션의 트레이싱을 활성화할 수 있습니다.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.1"  # 10% head-based sampling
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
```

이 CR이 적용된 네임스페이스의 Pod에 `instrumentation.opentelemetry.io/inject-java: "true"` 어노테이션을 추가하면, Mutation Webhook이 OTEL Java Agent를 자동으로 주입합니다.

## 4.6 Kubernetes 알람 설계의 실전 패턴

알람 설계에서 가장 흔한 실수는 **원인(Cause)** 에 알람을 거는 것입니다. 예를 들어 "CPU > 80%"는 증상이 아닌 원인으로, 사용자 영향이 없는 배치 작업 중에도 발생할 수 있습니다. Google SRE의 권고는 **증상(Symptom) 기반 알람**으로, "사용자가 느끼는 오류율"과 "사용자가 느끼는 지연 시간"에 알람을 거는 것입니다.

**멀티 윈도우 멀티 번 레이트(Multi-Window Multi-Burn-Rate)** 알람은 에러 버짓 소모 속도를 다중 시간 창에서 동시에 모니터링하여 **빠른 에러 버짓 소모**(단시간 내 완전 소진 위기)와 **느린 에러 버짓 소모**(한 달에 걸쳐 조용히 소진) 모두를 감지합니다.

```yaml
# 1시간 창에서 14.4배, 5분 창에서 14.4배 소모 시 긴급 알람(Critical)
- alert: HighErrorBudgetBurnRate
  expr: |
    (
      job:http_requests:error_rate5m > (14.4 * 0.001)
    and
      job:http_requests:error_rate1h > (14.4 * 0.001)
    )
  for: 2m
  labels:
    severity: critical
```

## 4.7 eBPF 기반 차세대 Kubernetes 모니터링

**eBPF(extended Berkeley Packet Filter)**는 커널 공간에서 안전하게 실행되는 샌드박스 프로그램으로, 네트워크 패킷, 시스템 콜, 함수 호출 등 커널 이벤트에 훅을 걸어 극히 낮은 오버헤드로 전체 시스템 동작을 관측할 수 있습니다.

**Cilium**은 eBPF 기반 CNI(Container Network Interface)이며, Hubble을 통해 Pod 간 L3/L4/L7 네트워크 플로우를 서비스 맵으로 시각화합니다. 기존 iptables 기반 네트워킹 대비 eBPF는 패킷 처리 지연을 60~70% 줄이고, 동시에 DNS 쿼리, HTTP 요청, gRPC 호출의 Observability를 커널 레벨에서 제공합니다.

**Pixie(px.dev)**는 eBPF를 사용하여 코드 변경 없이 Kubernetes 워크로드의 L7 트래픽(HTTP, gRPC, MySQL, Redis, Kafka)을 자동으로 캡처하고 분석합니다. 데이터가 클러스터 내에서만 처리되어 프라이버시와 보안 컴플라이언스를 유지합니다.

---

# 제5부: 통합 관측가능성 플랫폼과 고급 사용 사례

## 5.1 Observability Pipeline의 설계 원칙

대규모 환경에서의 Observability 파이프라인은 **수집(Collection) → 처리(Processing) → 라우팅(Routing) → 저장(Storage) → 분석(Analysis)**의 다섯 단계로 구성됩니다. OpenTelemetry Collector는 수집·처리·라우팅의 핵심 컴포넌트로, 레시버(Receiver) → 프로세서(Processor) → 익스포터(Exporter)의 파이프라인 체인으로 구성됩니다.

**벡터(Vector)**(Datadog 인수, 현재 오픈소스 유지)는 Rust로 구현된 고성능 Observability 데이터 파이프라인으로, 초당 수백만 이벤트를 처리하면서 변환(Transform), 집계(Aggregate), 필터링(Filter)을 적용합니다. 특히 **VRL(Vector Remap Language)**은 로그 파싱 및 변환에 특화된 도메인 특화 언어(DSL)로, 복잡한 정규 표현식과 조건 로직을 안전하게 표현합니다.

## 5.2 AIOps와 ML 기반 이상 탐지

**AIOps(AI for IT Operations)**는 Observability 데이터에 ML을 적용하여 노이즈 감소, 이상 탐지, 근본 원인 분석(RCA), 사고 예측을 자동화합니다.

**메트릭 이상 탐지**에서 가장 널리 사용되는 알고리즘은 다음과 같습니다. **SARIMA(Seasonal AutoRegressive Integrated Moving Average)**는 계절성이 강한 KPI(주간 패턴, 일간 패턴)에 적합합니다. **Prophet(Meta)**은 휴일 효과, 변화점(Changepoint)을 자동으로 학습하며 해석 가능성이 높습니다. **Isolation Forest**는 고차원 메트릭의 비지도 이상 탐지에 효과적입니다. **LSTM/Transformer 기반 시계열 모델**은 복잡한 장기 의존성을 캡처하지만, 해석 가능성이 낮고 운영 비용이 높습니다.

**토폴로지 기반 근본 원인 분석(Topological RCA)**에서는 서비스 의존성 그래프를 인과 그래프(Causal Graph)로 모델링하고, 이상이 전파되는 방향을 역추적합니다. Microsoft Azure의 연구팀은 2023년에 그래프 신경망(GNN, Graph Neural Network)을 활용한 **MicroRCA** 알고리즘을 발표하여, 마이크로서비스 환경에서의 근본 원인 특정 정확도를 90% 이상으로 높였습니다.

## 5.3 LLMOps 관측가능성의 특수 과제

LLM(대규모 언어 모델) 기반 서비스의 Observability는 전통적 서비스와 근본적으로 다른 차원의 과제를 제시합니다. **토큰 단위 지연 분포(Token-Level Latency Distribution)**는 Time-to-First-Token(TTFT)과 Inter-Token Latency(ITL), 그리고 Total Generation Time을 별도로 추적해야 합니다. 이는 사용자 인지 지연과 직결되며, P50이 아닌 P99, P999 지연이 UX 품질을 결정합니다.

**비용 추적(Cost Tracking)**은 입력 토큰, 출력 토큰, 캐시 히트 여부별로 분리 집계되어야 합니다. 프롬프트 버전·사용자 세그먼트·기능 플래그별 비용을 분석하려면 고카디널리티 레이블 지원이 필수입니다.

**환각(Hallucination) 감지 및 품질 메트릭**은 자동화된 LLM Judge, RAGAS(Retrieval Augmented Generation Assessment) 스코어, 사람 피드백(RLHF) 신호를 Observability 파이프라인에 통합하여 모델 성능의 시간적 변화(Drift)를 추적해야 합니다.

**LLM Observability 도구 생태계**로는 LangSmith(LangChain), Phoenix(Arize AI), Langfuse(오픈소스), OpenLLMetry(Traceloop)가 OpenTelemetry 기반의 트레이싱 표준을 지원합니다.

---

# 제6부: 빅테크 면접 Q&A 30선

> 대상 기업: Amazon, Apple, Nvidia, Google, Microsoft, Tesla  
> 난이도: Senior Engineer (L5/L6급)

---

## Amazon (AWS/SRE 관련)

**Q1. AWS에서 멀티 리전 Active-Active 서비스의 관측가능성 아키텍처를 설계하라. 특히 두 리전에서 동시에 이상이 발생했을 때 알람 폭풍(Alert Storm)을 어떻게 방지하겠는가?**

A1. 멀티 리전 Active-Active 환경에서는 각 리전의 CloudWatch Alarm이 독립적으로 작동하므로, 리전 장애 시 수백 개의 알람이 동시에 발생하는 Alert Storm이 불가피합니다. 이를 방지하기 위한 핵심 전략은 **알람 계층화(Alarm Layering)**와 **토폴로지 인식 집계(Topology-Aware Aggregation)**입니다.

구체적으로는 먼저 로우레벨 알람(개별 인스턴스 CPU, 메모리)과 하이레벨 알람(서비스 SLO 위반)을 명확히 분리합니다. 하이레벨 알람만 PagerDuty로 라우팅하고, 로우레벨 알람은 ServiceNow 티켓 자동 생성으로만 처리합니다. 다음으로 EventBridge에서 Composite Pattern을 적용하여 "리전 A의 가용성 저하 + 리전 B의 가용성 정상"일 때만 리전 A 장애로 판단하고, "두 리전 모두 가용성 저하"일 때는 단일 글로벌 인프라 장애 알람으로 집약합니다. 마지막으로 AWS Route 53 Health Check를 "알람의 알람"으로 활용하여, 개별 서비스 알람보다 먼저 글로벌 서비스 상태를 판단하는 최상위 알람으로 설계합니다. 이렇게 하면 알람 수를 수백 개에서 3~5개의 행동 가능한 알람으로 압축할 수 있습니다.

---

**Q2. Amazon의 장애 대응(COE, Correction of Errors) 프로세스에서 메트릭과 트레이스가 어떤 역할을 하는지, 그리고 근본 원인 분석(RCA)을 자동화하기 위해 어떤 시스템을 설계하겠는가?**

A2. Amazon의 COE(내부적으로 "5 Whys"를 구조화한 문서)는 장애의 원인이 단순 기술 결함이 아닌 조직·프로세스·도구의 복합적 실패임을 강조합니다. 메트릭과 트레이스는 이 분석에서 두 가지 역할을 합니다. 첫째로 **무엇(What)이 잘못되었나**를 메트릭이 보여주고, **어디서(Where) 그 영향이 전파되었나**를 트레이스가 보여줍니다. 둘째로 장애 타임라인 재구성에서 CloudWatch의 빠른 메트릭과 X-Ray의 트레이스를 시간 정렬(time-correlation)하여 인과 관계를 확립합니다.

자동화된 RCA 시스템 설계로는, OpenSearch에 트레이스와 메트릭을 통합 저장하고, 장애 신호가 감지되면 그래프 알고리즘(BFS/DFS)으로 서비스 의존성 그래프를 역방향 탐색하여 오류 전파 경로를 추적합니다. 여기에 ML 모델이 과거 장애 패턴과 현재 신호를 비교하여 가장 유사한 과거 사례와 해당 해결책을 자동으로 제안합니다. 최종적으로 Slack 또는 내부 인시던트 관리 시스템에 구조화된 "초기 RCA 요약"을 자동 게시하여 온콜 엔지니어의 인지 부하를 줄입니다.

---

**Q3. AWS Lambda의 콜드 스타트 지연을 모니터링하고, 이를 자동으로 완화하는 관측가능성 기반 시스템을 설계하라.**

A3. Lambda 콜드 스타트를 정밀 추적하기 위해서는 먼저 Lambda Extension을 활용하여 Init 단계(런타임 초기화)와 Invoke 단계(실제 핸들러 실행)를 분리 계측합니다. `REPORT` 로그 라인의 `Init Duration` 필드를 CloudWatch Logs Insights로 파싱하여 콜드 스타트 비율(콜드 스타트 발생 호출 수 / 전체 호출 수)과 P99 Init Duration을 시계열로 추적합니다. X-Ray 또는 ADOT를 통해 콜드 스타트가 발생한 트레이스에 `cold_start=true` 속성을 태깅하면, 다운스트림 서비스에서의 지연 증가와 콜드 스타트를 연결 분석할 수 있습니다.

자동 완화 시스템으로는 EventBridge Scheduler가 트래픽 패턴 분석 결과를 기반으로 Provisioned Concurrency를 동적으로 조정합니다. 구체적으로 과거 7일의 트래픽 패턴에서 SageMaker의 시계열 예측 모델이 다음 1시간의 호출 수를 예측하고, 예측 수치에 비례하여 Provisioned Concurrency를 사전 증가시킵니다. 이렇게 하면 콜드 스타트를 제거하면서도 Provisioned Concurrency 비용의 낭비를 최소화할 수 있습니다.

---

**Q4. DynamoDB의 Hot Partition 문제를 관측가능성 도구로 탐지하고 자동으로 대응하는 방법을 설계하라.**

A4. DynamoDB의 Hot Partition은 메트릭 자체에서 직접 드러나지 않는 것이 함정입니다. `SuccessfulRequestLatency`, `ThrottledRequests`, `ConsumedReadCapacityUnits` 메트릭은 테이블 단위로만 제공되며, 파티션 단위 메트릭은 공개되지 않습니다. 따라서 탐지는 간접 신호에서 이루어집니다. 먼저 `ThrottledRequests`의 급등이 Hot Partition의 가장 강력한 간접 신호입니다. 이와 동시에 `ConsumedCapacityUnits`가 `ProvisionedCapacityUnits`의 수치보다 낮음에도 스로틀링이 발생하면 Hot Partition을 강하게 의심합니다. 다음으로 CloudWatch Contributor Insights를 활성화하면 가장 많이 읽히는 파티션 키(Partition Key)를 직접 확인할 수 있어 Hot Partition의 키 분포 문제를 즉각 파악할 수 있습니다.

자동 대응으로는 Lambda가 Hot Partition 알람을 수신하여 DAX(DynamoDB Accelerator) 클러스터를 자동으로 활성화하거나 캐시 TTL을 동적으로 조정합니다. 장기적으로는 AWS Glue Job이 파티션 키 액세스 패턴을 분석하여 키 설계 개선 권고를 자동으로 생성하는 Runbook을 제안합니다.

---

**Q5. AWS에서 100개 이상의 마이크로서비스로 구성된 서비스 메시(Service Mesh)의 Observability를 비용 효율적으로 설계하라. 특히 트레이스 데이터 볼륨과 비용을 어떻게 통제하겠는가?**

A5. 트레이스 비용 통제의 핵심 전략은 **적응형 샘플링(Adaptive Sampling)**입니다. 정상 경로(2xx, 200ms 이하)의 트레이스는 1~5%만 보존하고, 오류(4xx, 5xx)와 고지연(P99 초과) 트레이스는 100% 보존합니다. 이를 위해 AWS App Mesh + ADOT Collector의 Tail-Based Sampling 설정을 조합합니다. ADOT Collector에서 `tail_sampling` 프로세서는 스팬 완료 후 전체 트레이스를 검토하여 오류가 포함된 트레이스를 선별적으로 보존합니다.

비용 구조적으로는 샘플링된 트레이스를 X-Ray(단기 분석용)와 S3(장기 보관용)에 이중 저장하고, S3의 트레이스는 Athena로 임시 분석합니다. 서비스 간 의존성 맵과 에러율 집계는 트레이스의 요약 통계를 CloudWatch Metrics로 내보내어 유지하므로, 높은 샘플링률 없이도 서비스 토폴로지 시각화는 항상 유지됩니다. 이 전략으로 동일한 Observability 품질을 유지하면서 X-Ray 비용을 70~80% 절감할 수 있습니다.

---

## Google (SRE/Cloud 관련)

**Q6. Google의 SRE 철학에서 에러 버짓(Error Budget)이 조직 문화에 미치는 영향을 설명하고, 에러 버짓 정책(Error Budget Policy)을 실제로 집행하는 기술 시스템을 설계하라.**

A6. 에러 버짓은 단순한 기술 메트릭이 아니라 개발팀과 SRE팀 사이의 **협상된 계약(Negotiated Contract)**입니다. 에러 버짓이 풍부할 때는 개발팀이 리스크 있는 빠른 배포를 감행할 수 있고, 에러 버짓이 소진되면 자동으로 배포가 동결(Feature Freeze)됩니다. 이는 신뢰성 투자 동기를 외부 압력이 아닌 **내부 자율 메커니즘**으로 내재화합니다.

기술 집행 시스템으로는 Prometheus 기반의 에러 버짓 계산 엔진이 실시간으로 잔여 에러 버짓을 계산하고, 이를 GitHub Actions의 배포 게이트(Deployment Gate)에 연결합니다. 잔여 에러 버짓이 20% 미만이면 PR 머지를 차단하고, 10% 미만이면 기존 배포도 자동 롤백 모드로 전환합니다. 에러 버짓 대시보드는 제품 매니저와 엔지니어링 매니저에게 공유되어 기술 부채 해소와 기능 개발 사이의 우선순위 결정에 정량적 근거를 제공합니다.

---

**Q7. Google의 분산 추적 시스템 Dapper의 설계 원칙을 설명하고, 이를 현대 OpenTelemetry 생태계와 비교하라. 특히 샘플링 전략에서의 차이점은 무엇인가?**

A7. 2010년 발표된 Dapper 논문은 분산 추적의 세 가지 설계 원칙을 정립했습니다. 첫째, **항상 켜져 있어야 한다(Always-On)**. 개발자가 계측 여부를 선택하는 것이 아니라 인프라 레이어에서 자동으로 계측되어야 합니다. 둘째, **낮은 오버헤드**. Trace 수집이 서비스 성능에 1% 미만의 영향을 주어야 합니다. 셋째, **애플리케이션 투명성(Application Transparency)**. 개발자는 추적 존재를 몰라도 서비스를 개발할 수 있어야 합니다.

Dapper는 Head-Based Sampling을 사용하여 요청 진입 시점에 샘플링을 결정합니다. 반면 OpenTelemetry의 최신 권장 방식은 Tail-Based Sampling으로, 전체 트레이스가 완료된 후 이상 신호를 기준으로 보존 여부를 결정합니다. 이는 Dapper 당시에는 불가능했던 기법으로, OTel Collector의 Tail Sampling Processor가 이를 구현하며, 이를 위해 동일 트레이스의 모든 스팬이 동일한 Collector 인스턴스로 라우팅되어야 한다는 **스팬 집중(Span Concentration)** 요구사항이 생깁니다. Jaeger의 Remote Sampling과 Grafana Tempo의 TraceQL은 이 패러다임을 발전시킨 최신 구현입니다.

---

**Q8. Google Kubernetes Engine(GKE)에서 수만 개의 Pod를 운영하는 환경의 Prometheus 메트릭 수집 아키텍처를 설계하라. Thanos vs Grafana Mimir의 선택 기준은 무엇인가?**

A8. 수만 Pod 규모에서 단일 Prometheus는 메모리 한계(일반적으로 수백만 Active Series가 수십 GB TSDB 메모리를 요구)와 고가용성 부재로 인해 사용 불가능합니다. 따라서 수평 확장형 아키텍처가 필수입니다.

**Thanos 선택 기준**은 다음과 같습니다. 이미 Prometheus를 운영 중이며 이를 재사용하고 싶을 때, 멀티 클러스터/멀티 클라우드 통합 뷰가 필요하지만 완전 관리형 솔루션 대신 운영 유연성을 원할 때, 오브젝트 스토리지 비용 최적화가 중요할 때입니다. Thanos는 기존 Prometheus 인스턴스에 사이드카를 붙이는 방식이라 점진적 마이그레이션이 가능합니다.

**Grafana Mimir 선택 기준**은 다음과 같습니다. 처음부터 스케일 아웃형 아키텍처를 설계할 때, 초당 수백만 개의 메트릭 샘플을 수집해야 할 때, 멀티 테넌시(팀별 메트릭 격리)가 중요할 때입니다. Mimir는 완전 분산형으로 컴포넌트별 독립적 스케일링이 가능하며, GKE 환경에서 GCS를 백엔드로 사용하면 운영 단순성과 비용 효율이 높습니다. 실제로 Grafana Labs는 Weave Works 인수 후 GKE 환경의 표준 스택으로 OpenTelemetry → Mimir + Loki + Tempo → Grafana를 권장합니다.

---

**Q9. Google의 SLO 기반 알람과 전통적인 임계값(Threshold) 기반 알람의 철학적·실용적 차이를 논하라.**

A9. 임계값 기반 알람의 근본적 문제는 **정밀도(Precision)와 재현율(Recall) 사이의 상충(Trade-off)**입니다. 임계값을 낮게 설정하면 거짓 양성(False Positive, 알람 피로)이 증가하고, 높게 설정하면 거짓 음성(False Negative, 실제 장애 미탐지)이 증가합니다. 더 근본적으로, 임계값은 메트릭의 **절대값**에 반응하지만 사용자 영향은 메트릭의 **맥락적 의미**에서 결정됩니다. 예를 들어 배포 직후의 CPU 80%와 안정 운영 중 CPU 80%는 의미가 전혀 다릅니다.

SLO 기반 알람은 "사용자가 나쁜 경험을 하고 있는가?"라는 질문에 직접 답하는 알람입니다. 에러율이 SLO 목표를 초과할 때만 알람이 발생하므로, 사용자 영향이 없는 내부 리소스 변동에 반응하지 않습니다. 다중 번 레이트(Multi-Burn-Rate) 알람은 "지금 얼마나 빨리 에러 버짓이 소모되고 있는가?"를 측정하여 빠른 버짓 소모(긴급 대응 필요)와 느린 버짓 소모(다음 업무일 내 대응)를 구분합니다. 이 철학의 실용적 결과는 **알람 수 대폭 감소(수백 개 → 수 개)**와 **온콜 피로 감소**, 그리고 **알람-사용자 영향 상관관계 증가**입니다.

---

**Q10. eBPF를 사용한 커널 레벨 Kubernetes 네트워크 모니터링의 원리를 설명하고, Cilium/Hubble을 이용한 실제 구성 방법을 논하라.**

A10. eBPF는 Linux 커널의 **안전한 확장 메커니즘**입니다. eBPF 프로그램은 커널 이벤트 훅 포인트(kprobes, tracepoints, XDP, TC)에 부착되어 실행되며, eBPF Verifier가 무한 루프와 메모리 오류를 컴파일 시점에 검증하여 커널 안정성을 보장합니다. 기존 iptables/netfilter 기반 모니터링은 패킷을 사용자 공간으로 복사하는 비용이 컸지만, eBPF는 커널 내에서 처리하여 오버헤드를 10배 이상 줄입니다.

Cilium은 CNI 플러그인으로 배포되며, `helm install cilium cilium/cilium --set hubble.enabled=true --set hubble.relay.enabled=true --set hubble.ui.enabled=true` 로 Hubble UI까지 한 번에 구성합니다. Hubble은 모든 Pod의 네트워크 플로우를 gRPC 스트림으로 노출하며, DNS 쿼리, HTTP 메서드/경로, gRPC 서비스/메서드를 L7 레이어에서 파싱합니다. `hubble observe --namespace production --protocol http --verdict DROPPED` 명령으로 프로덕션 네임스페이스에서 차단된 HTTP 트래픽을 실시간 확인할 수 있습니다. 이를 Grafana와 연동하면 네트워크 정책 위반, 비정상 외부 통신, 서비스 간 지연 분포를 코드 변경 없이 완전한 가시성으로 모니터링할 수 있습니다.

---

## Microsoft (Azure/클라우드 SRE 관련)

**Q11. Azure Monitor와 Log Analytics의 데이터 파이프라인 아키텍처를 심층 설명하고, 수십 개의 Azure 구독(Subscription)을 단일 Observability 플랫폼으로 통합하는 방법을 설계하라.**

A11. 수십 개의 구독 통합을 위한 표준 아키텍처는 **Azure Lighthouse + 중앙 Log Analytics Workspace** 패턴입니다. Azure Lighthouse를 통해 관리 구독(Managing Tenant)에서 모든 고객 구독(Managed Tenant)에 대한 읽기 권한을 위임받고, 각 구독의 진단 설정(Diagnostic Settings)을 중앙 Log Analytics Workspace로 향하게 합니다. 이 설정은 Azure Policy를 통해 새로운 구독에 자동으로 적용됩니다.

데이터 볼륨 관리는 이 아키텍처의 주요 과제입니다. 중앙 Workspace에 모든 로그를 보내면 GB 단위 인제스트 비용이 급증합니다. 이를 위해 **Data Collection Rules(DCR)**를 사용하여 구독별로 필요한 로그만 선택적으로 수집합니다. 예를 들어 AuditLogs는 모든 구독에서 수집하지만, 상세한 ApplicationLog는 프로덕션 구독에서만 수집합니다. 또한 **기본 플랜(Basic Plan)** 로그(저비용, KQL 쿼리 제한)와 **분석 플랜(Analytics Plan)** 로그(고비용, 풀 KQL 지원)를 구분하여 비용을 최적화합니다.

---

**Q12. Azure Application Insights의 샘플링 메커니즘(Adaptive Sampling, Fixed-Rate Sampling, Ingestion Sampling)을 각각 설명하고, 언제 어떤 방식을 선택해야 하는지 논하라.**

A12. 세 가지 샘플링의 동작 레이어가 다릅니다.

**Adaptive Sampling**은 SDK 레이어에서 동작하며, 초당 전송 텔레메트리 아이템 수가 목표값(기본 5 items/sec)을 초과하면 자동으로 샘플링 비율을 낮춥니다. 트래픽 급증 시 Application Insights 인제스트 비용을 자동으로 통제하는 장점이 있지만, 낮은 트래픽 환경에서 드물게 발생하는 오류를 놓칠 수 있습니다.

**Fixed-Rate Sampling**은 SDK와 서버가 동일한 비율(예: 10%)로 샘플링하여 클라이언트-서버 트레이스의 연관성을 보장합니다. 요청 수 기반 통계 분석이 필요할 때 적합합니다.

**Ingestion Sampling**은 Application Insights 인제스트 엔드포인트에서 적용되며 SDK 수정이 필요 없습니다. 그러나 SDK에서 이미 전송된 데이터를 폐기하므로 네트워크 비용은 절감되지 않습니다.

선택 기준으로는, 비용 통제가 최우선이면 Adaptive Sampling, 정확한 샘플링 비율과 통계적 일관성이 중요하면 Fixed-Rate Sampling, 레거시 SDK를 수정할 수 없는 경우 Ingestion Sampling을 선택합니다. 대부분의 프로덕션 환경에서는 Adaptive Sampling을 기본으로 하되, 오류 트레이스는 100% 보존하는 커스텀 TelemetryProcessor를 SDK에 추가하는 하이브리드 방식이 권장됩니다.

---

**Q13. Azure에서 Kubernetes(AKS) 클러스터의 OOMKilled 이벤트를 실시간으로 탐지하고, 자동으로 리소스 한도를 조정하는 Closed-Loop 자동화 시스템을 설계하라.**

A13. 이 시스템은 탐지(Detect) → 분석(Analyze) → 결정(Decide) → 실행(Execute) → 검증(Verify)의 제어 루프(Control Loop)로 설계합니다.

탐지 레이어에서는 kube-state-metrics의 `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` 메트릭을 Prometheus가 수집하고, Alertmanager가 알림을 Azure Event Hub로 발행합니다. 분석 레이어에서는 Azure Function이 이벤트를 수신하여 해당 컨테이너의 과거 메모리 사용 추이(Prometheus의 `container_memory_working_set_bytes`)를 쿼리합니다. P99 메모리 사용량과 현재 메모리 한도를 비교하여 권장 한도를 계산합니다(P99 × 1.5 + 안전 마진). 결정 레이어에서는 권장 조정 값이 전체 노드 메모리의 50%를 초과하지 않는지, 최근 24시간 내 동일 컨테이너의 조정이 3회 이상 발생하지 않았는지 확인하는 가드레일(Guardrail)을 적용합니다. 실행은 Kubernetes API를 통해 Deployment의 resources.limits.memory를 패치합니다. 검증 레이어에서는 조정 후 1시간 동안 OOMKilled 이벤트가 재발하지 않으면 성공으로 기록하고, 재발하면 VPA(Vertical Pod Autoscaler)의 권장값을 참고하여 더 보수적인 재조정을 수행합니다.

---

**Q14. Azure Cost Management와 Observability를 연계하여 클라우드 비용 이상(Cost Anomaly)을 실시간으로 탐지하는 FinOps 관측가능성 시스템을 설계하라.**

A14. FinOps 관측가능성은 기술 메트릭과 비용 메트릭의 상관관계 분석이 핵심입니다. 비용 이상 탐지는 두 단계로 구성됩니다.

첫 번째 단계는 **비용 메트릭 수집**입니다. Azure Cost Management의 Export 기능으로 일별 실제 비용 데이터를 Azure Blob Storage에 저장하고, Azure Data Factory로 정제하여 Log Analytics Workspace에 적재합니다. 메트릭 단위로는 서비스별, 리소스 그룹별, 태그별 일 단위 비용을 수집합니다.

두 번째 단계는 **이상 탐지와 상관관계 분석**입니다. KQL의 `series_decompose_anomalies` 함수로 비용 시계열의 이상값을 탐지하고, 이상이 탐지된 시점의 Azure Activity Log와 리소스 변경 이벤트를 조인하여 비용 급증의 원인을 자동으로 식별합니다. 예를 들어 VM SKU 변경, 스케일 아웃 이벤트, 새로운 서비스 프로비저닝과 비용 급증의 시간적 상관관계를 분석합니다. 탐지된 비용 이상은 Slack 알림과 함께 Workbook 기반의 Cost Drill-Down 링크를 포함하여 팀이 즉시 원인을 탐색할 수 있게 합니다.

---

**Q15. Azure의 AMPLS(Azure Monitor Private Link Scope)가 필요한 시나리오와 그 한계를 설명하고, 금융권 레벨의 컴플라이언스 요구사항을 만족하는 모니터링 아키텍처를 설계하라.**

A15. AMPLS가 필수적인 시나리오는 PCI DSS, ISO 27001, SOC 2 Type II를 요구하는 금융·의료 환경으로, 모니터링 텔레메트리가 인터넷을 경유하면 안 된다는 규정이 있을 때입니다. AMPLS는 Log Analytics Workspace와 Application Insights를 Private Endpoint로 연결하여 모든 데이터 전송이 Microsoft 백본(Azure Private Link)을 통해서만 이루어지도록 보장합니다.

AMPLS의 중요한 한계는 **DNS 오버라이드의 광범위성**입니다. AMPLS를 구성하면 VNet 내의 모든 리소스가 `*.ods.opinsights.azure.com`을 Private IP로 해석하게 됩니다. 따라서 동일 VNet에 AMPLS 외부의 다른 Log Analytics Workspace에 접근해야 하는 경우 DNS 충돌이 발생합니다. 이를 해결하기 위해 AMPLS를 Access Mode "Private Only"로 설정하되, 비컴플라이언스 데이터는 별도 퍼블릭 Workspace를 사용하는 이중 구조를 설계합니다.

금융권 완전 격리 아키텍처에서는 AMPLS 외에도 다음을 추가합니다. Azure Policy로 Diagnostic Setting이 AMPLS 내 Workspace로만 향하도록 강제합니다. Defender for Cloud 알람도 동일 AMPLS를 통해 라우팅합니다. 감사 목적의 Azure Activity Log는 이중으로 Event Hub(AMPLS 내)와 Immutable Storage(변경 불가 Blob)에 저장합니다.

---

## Nvidia (GPU 클러스터/ML 인프라 관련)

**Q16. Nvidia GPU 클러스터에서 DCGM Exporter를 사용한 GPU 모니터링을 설계하라. 특히 멀티 GPU 학습 작업에서 GPU 유틸리제이션 불균형(Imbalanced GPU Utilization)을 탐지하는 방법을 설명하라.**

A16. **DCGM(Data Center GPU Manager) Exporter**는 Prometheus 형식으로 GPU 지표를 노출하는 Nvidia의 공식 도구입니다. 핵심 메트릭으로는 `DCGM_FI_DEV_GPU_UTIL`(GPU 코어 사용률), `DCGM_FI_DEV_MEM_COPY_UTIL`(메모리 대역폭 사용률), `DCGM_FI_DEV_FB_USED`(GPU 메모리 사용량), `DCGM_FI_DEV_SM_CLOCK`(SM 클럭 주파수), `DCGM_FI_DEV_POWER_USAGE`(GPU 전력 소비), `DCGM_FI_DEV_XID_ERRORS`(Xid 오류 코드)가 있습니다.

멀티 GPU 학습에서의 불균형 탐지는 다음 PromQL로 구현합니다. 동일 노드 내 GPU 간 표준편차가 임계값을 초과하면 파이프라인 병목, 데이터 로딩 불균형, AllReduce 통신 병목 등을 의심할 수 있습니다.

```promql
# 동일 Job 내 GPU별 사용률의 표준편차
stddev by (job_id) (
  DCGM_FI_DEV_GPU_UTIL{kubernetes_pod_label_training_job_id!=""}
)
```

이 값이 20% 이상이면 알람을 발생시키고, Grafana 대시보드에 GPU별 사용률을 히트맵으로 표시하여 어떤 GPU가 병목인지 즉시 식별합니다. 추가로 `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL`을 모니터링하여 NVLink 포화(Saturation)가 통신 병목의 원인인지 진단합니다.

---

**Q17. Nvidia H100 클러스터에서 MFU(Model FLOPs Utilization)를 실시간으로 추적하고, 학습 효율 저하를 자동으로 탐지하는 시스템을 설계하라.**

A17. MFU는 실제 처리된 FLOPs를 이론적 최대 FLOPs로 나눈 비율로, 모델 학습 인프라 효율의 핵심 KPI입니다. H100 SXM5의 이론적 피크 FP16 성능은 약 2,000 TFLOPS이며, 현실적인 학습에서 MFU는 30~60% 수준입니다.

MFU 계산은 프레임워크 레벨(PyTorch의 `torch.profiler`)과 하드웨어 레벨(DCGM의 Tensor Core 활성 사이클)을 조합합니다. Kubernetes Job으로 실행되는 학습 작업의 `step_time`과 `tokens_per_step`(혹은 `samples_per_step`)을 Prometheus Custom Metrics API로 노출하고, 이론적 FLOP/step을 모델 아키텍처에서 사전 계산하여 설정합니다. MFU = (실제 throughput tokens/s × FLOPs/token) / (GPU 수 × 이론 FLOPs/GPU).

MFU가 기준치 대비 20% 이상 하락하면 알람을 발생시키고, 동시에 DCGM의 `DCGM_FI_PROF_DRAM_ACTIVE`(메모리 대역폭 포화)와 `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`(Tensor Core 활성율)를 조합하여 병목이 Compute-Bound인지 Memory-Bound인지 자동 진단합니다. 이 결과를 Slack 알림에 포함하면 온콜 엔지니어가 즉시 원인 범주를 파악할 수 있습니다.

---

**Q18. Kubernetes 위에서 분산 학습(DDP, DeepSpeed, Megatron-LM)을 실행하는 환경에서 NCCL 통신 병목을 Observability 도구로 탐지하는 방법을 논하라.**

A18. NCCL(NVIDIA Collective Communications Library)의 병목은 GPU 로그에서 직접 드러나지 않는 경우가 많아 탐지가 어렵습니다. 주요 탐지 신호는 다음과 같습니다.

첫 번째로 **GPU 사용률 간헐적 급락(Utilization Drop)**이 있습니다. 학습 중 모든 GPU의 사용률이 동시에 0~10%로 떨어지는 패턴은 AllReduce 동기화 대기를 강하게 시사합니다. DCGM 메트릭에서 `DCGM_FI_DEV_GPU_UTIL`의 시계열에서 이 패턴을 탐지합니다. 두 번째로 **NVLink/InfiniBand 대역폭 포화**입니다. `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL` 또는 Mellanox 카운터(ibstat, perfquery)에서 이론 최대치 대비 실제 사용률이 낮으면서 GPU 대기가 발생하면 네트워크가 병목이 아님을 확인하고, 높으면 네트워크 포화가 원인입니다. 세 번째로 **NCCL Debug Log 분석**입니다. `NCCL_DEBUG=INFO` 환경 변수로 NCCL 내부 로그를 활성화하고, 이 로그를 Loki로 수집하여 `"Timeout"` 또는 `"collective comm aborted"` 패턴을 실시간 감지합니다.

탐지 자동화를 위해 학습 Job의 `forward_time`과 `backward_time`, `optimizer_step_time`을 별도 Prometheus 게이지로 노출하면, `comm_overhead = step_time - forward_time - backward_time`으로 통신 오버헤드를 직접 계산하여 임계값 알람을 설정할 수 있습니다.

---

## Apple (인프라 관련)

**Q19. Apple 수준의 프라이버시 규제 환경에서 Observability 데이터에 포함될 수 있는 PII(개인 식별 정보)를 자동으로 감지하고 마스킹하는 파이프라인을 설계하라.**

A19. PII 보호는 Observability 파이프라인의 **수집 단계에서 필터링(Shift-Left Privacy)**하는 것이 원칙입니다. 데이터가 일단 중앙 저장소에 도달하면 삭제·수정이 어렵기 때문입니다.

파이프라인 설계로는 OpenTelemetry Collector의 Transform Processor 또는 Vector의 VRL을 사용하여 로그와 스팬의 속성값에서 정규표현식 기반 PII 패턴(이메일, 전화번호, 신용카드 번호, SSN, IP 주소)을 탐지하고 마스킹합니다. 정적 패턴으로 잡기 어려운 PII는 ML 기반 NER(Named Entity Recognition) 모델을 사이드카 서비스로 배포하여 텍스트 필드의 개인 이름, 주소 등을 탐지합니다. 그러나 NER 모델 호출은 레이턴시 오버헤드가 있으므로, 민감 데이터가 흐를 가능성이 높은 경로(사용자 입력이 포함된 API 엔드포인트, 오류 메시지)에만 선택적으로 적용합니다.

감사(Audit) 목적으로는 마스킹 전 원본 데이터를 암호화된 보안 볼트(Azure Key Vault, AWS KMS 봉투 암호화)에 별도 보관하고, 법적 요구 시에만 접근 가능하도록 엄격한 IAM 정책으로 제어합니다. 마스킹 이벤트 자체도 불변 감사 로그(Immutable Audit Log)에 기록합니다.

---

**Q20. 수백만 iOS/macOS 기기에서 수집되는 크래시 리포트와 성능 데이터를 처리하는 대규모 Observability 파이프라인의 아키텍처를 설계하라.**

A20. 이 규모의 파이프라인 설계에서 핵심 도전은 **초당 수백만 이벤트의 인제스트**, **데이터 중요도에 따른 우선순위 처리**, 그리고 **PII 컴플라이언스**입니다.

인제스트 레이어는 AWS Kinesis Data Streams(또는 Apache Kafka on EKS)으로 구성하며, 파티션 키는 디바이스 모델과 OS 버전의 조합으로 설정하여 동일 모델군의 크래시를 동일 샤드로 라우팅합니다. 이를 통해 집계(Aggregation)의 데이터 지역성을 높입니다.

처리 레이어는 Apache Flink로 스트리밍 윈도우 처리를 수행합니다. 크래시 스택 트레이스의 **시그니처 해싱**(스택 프레임의 상위 N개를 해시)으로 동일 근본 원인의 크래시를 그룹화하고, 그룹별 발생률을 1분 윈도우로 집계합니다. 발생률이 기준 대비 10배 이상 급증하면 실시간 알람을 발생시킵니다. 이 패턴은 신규 OS 버전 또는 앱 배포 이후의 회귀(Regression)를 즉시 감지합니다.

장기 분석을 위해 원본 이벤트는 S3에 Parquet 형식으로 저장하고, Athena를 통한 임시 분석과 Amazon Redshift를 통한 트렌드 분석 대시보드를 제공합니다.

---

## Tesla (자동화/실시간 시스템 관련)

**Q21. Tesla의 자율주행 차량 데이터 파이프라인에서 차량 텔레메트리 데이터의 실시간 이상 탐지 시스템을 Kubernetes 위에 설계하라. 특히 Edge-Cloud 하이브리드 관측가능성 아키텍처를 논하라.**

A21. 차량 텔레메트리의 가장 큰 특성은 **산발적 연결성(Intermittent Connectivity)**과 **초저지연 안전 임계 이상(Safety-Critical Anomaly)**의 공존입니다. 안전 임계 이상(센서 오류, 브레이크 이상)은 차량 내부에서 즉각 처리되어야 하며, 학습 데이터 및 성능 분석 데이터는 클라우드로 전송됩니다.

Edge 레이어(차량 내 컴퓨트)에서는 경량 이상 탐지 모델(Isolation Forest의 ONNX 변환 버전 또는 규칙 기반 CEP 엔진)이 센서 데이터의 실시간 이상을 밀리초 단위로 탐지합니다. 이상 이벤트는 로컬 NAND Flash 버퍼에 고우선순위로 기록되고, 다음 셀룰러 연결 시 우선 전송됩니다.

Cloud 레이어(AWS 또는 Tesla 자체 DC)에서는 Kafka로 차량 이벤트를 수신하고, Flink가 차량 ID 기반 키드 스트리밍(Keyed Stream)으로 개별 차량의 이상 패턴 시퀀스를 탐지합니다. 플릿 단위(Fleet-Level) 분석을 위해 동일 차량 모델, 동일 날씨 조건, 동일 주행 시나리오에서의 집계 이상률을 Prometheus Pushgateway를 통해 Grafana 대시보드로 시각화합니다. 차량 소프트웨어 버전별 이상률 비교는 OTA 업데이트의 품질 게이트(Quality Gate)로 활용됩니다.

---

**Q22. Tesla의 Gigafactory 생산 라인에서 IoT 센서 데이터의 실시간 Observability와 예측 유지보수(Predictive Maintenance) 시스템을 Kubernetes 기반으로 설계하라.**

A22. 제조 환경의 IoT Observability는 IT 시스템의 Observability와 구분되는 **OT(Operational Technology) 특성**을 가집니다. 진동 센서(10kHz 샘플링), 온도 센서, 토크 센서의 데이터는 밀리초 해상도가 필요하며, 이는 일반 Prometheus Pull 방식으로는 불가능합니다.

아키텍처는 **MQTT Broker(Eclipse Mosquitto 또는 EMQX on Kubernetes) → Apache Kafka → Flink(실시간 특징 추출) → 예측 모델(KServe on K8s)**로 구성합니다. Flink는 진동 데이터에 FFT(Fast Fourier Transform)를 적용하여 주파수 스펙트럼 특징을 추출하고, 이 특징 벡터를 슬라이딩 윈도우(30초)로 KServe에 배포된 이상 탐지 모델에 전달합니다. 모델은 정상 스펙트럼 패턴에서의 편차를 z-score로 반환하며, z-score > 3이면 장비 이상 알람을 발생시킵니다.

예측 유지보수의 핵심은 **잔여 수명(RUL, Remaining Useful Life) 예측**입니다. LSTM 기반 RUL 모델은 지난 N일의 센서 데이터 시퀀스를 입력으로 받아 다음 정비 필요 시점을 예측합니다. 이 예측은 Prometheus 게이지로 노출되어 Grafana에서 시각화되고, 예측 정비 시점이 임박하면 SAP 연동을 통해 자동으로 정비 오더를 생성합니다.

---

## 공통 고급 Kubernetes/Cloud 질문

**Q23. Kubernetes에서 서비스 메시(Istio/Linkerd)의 mTLS 환경에서 분산 추적이 어떻게 동작하는지 설명하고, 헤더 전파(Header Propagation) 실패로 인한 트레이스 단절(Broken Trace) 문제를 해결하는 방법을 논하라.**

A23. Istio의 사이드카 프록시(Envoy)는 **자동으로 트레이스 스팬을 생성**하지만, 서비스 간 트레이스를 연결하려면 애플리케이션이 **B3/W3C TraceContext 헤더를 다운스트림 요청에 전달**해야 합니다. Envoy가 새 스팬을 생성할 때 `x-request-id`와 Zipkin B3 헤더를 요청에 주입하지만, 애플리케이션이 이 헤더를 읽지 않고 새로운 HTTP 클라이언트로 다운스트림을 호출하면 헤더가 소실되어 트레이스가 단절됩니다.

해결 방법으로는 먼저 OpenTelemetry SDK를 사용하면 Propagator가 자동으로 인바운드 컨텍스트를 추출하고 아웃바운드 요청에 주입합니다. Java 스프링 부트 환경에서는 `spring-cloud-sleuth` 또는 OTEL Spring Auto-Instrumentation이 이를 자동으로 처리합니다. 레거시 애플리케이션에서는 Envoy의 `lua_http_filter`를 사용하여 프록시 레이어에서 헤더를 강제 전파하는 방법을 사용합니다. 트레이스 단절 감지를 위해 Grafana Tempo 또는 Jaeger에서 `root_service != null AND span_count < expected_minimum_span_count` 조건으로 불완전한 트레이스를 주기적으로 스캔하고 레포팅합니다.

---

**Q24. Kubernetes Operator 패턴을 사용하여 자체적인 Observability Operator를 설계한다면, Reconciliation Loop에서 어떤 관측가능성 리소스를 관리하고 어떻게 멱등성(Idempotency)을 보장하겠는가?**

A24. Observability Operator의 주요 관리 대상은 다음과 같습니다. `ServiceMonitor`와 `PodMonitor`(Prometheus Operator CRD)를 통한 스크래핑 설정, `PrometheusRule`을 통한 알람 규칙, `OpenTelemetryCollector`와 `Instrumentation`(OpenTelemetry Operator CRD)을 통한 수집 파이프라인, Grafana Operator를 통한 대시보드와 데이터 소스가 있습니다.

Reconciliation Loop의 멱등성 보장은 **원하는 상태(Desired State) vs 실제 상태(Actual State)** 비교를 항상 수행하는 것으로 달성합니다. 구체적으로 `client.Get()`으로 리소스 존재 여부를 확인하고, 없으면 `client.Create()`, 있으면 현재 Spec을 desired Spec과 비교하여 다를 때만 `client.Update()`를 호출합니다. 리소스 소유권은 `controller-runtime`의 `controllerutil.SetControllerReference()`로 Owner Reference를 설정하여, 상위 CRD 삭제 시 하위 리소스가 가비지 컬렉션되도록 합니다. Reconcile 실패 시 **지수 백오프(Exponential Backoff)**로 재시도하고, 재시도 횟수가 임계값을 초과하면 CRD 상태의 `conditions` 필드에 `Ready: False` 상태를 기록하여 운영자에게 알립니다.

---

**Q25. GitOps 기반의 Observability-as-Code 파이프라인을 설계하라. 알람 규칙, 대시보드, SLO 설정을 코드로 관리하고 배포하는 전체 워크플로우를 설명하라.**

A25. Observability-as-Code의 핵심 원칙은 Observability 설정이 애플리케이션 코드와 **동일한 생명주기(Lifecycle)**를 가져야 한다는 것입니다. 새로운 서비스가 배포될 때 해당 서비스의 알람, 대시보드, SLO가 자동으로 함께 배포되고, 서비스가 삭제될 때 함께 정리되어야 합니다.

워크플로우는 다음과 같이 구성합니다. 먼저 모노레포(Monorepo)에서 각 서비스 디렉토리에 `observability/` 폴더를 두고 `alerts.yaml`, `dashboard.json`, `slo.yaml`을 포함합니다. GitHub Actions CI 파이프라인에서 `promtool check rules` (PromQL 문법 검증), `amtool check-config` (Alertmanager 설정 검증), `terraform validate` (그라파나 대시보드 Terraform 설정 검증)을 실행합니다. ArgoCD ApplicationSet이 모노레포의 서비스 구조를 스캔하여 각 서비스의 관측가능성 설정을 자동으로 ArgoCD Application으로 등록합니다. ArgoCD가 Prometheus Operator CRD(ServiceMonitor, PrometheusRule), Grafana Operator CRD(GrafanaDashboard, GrafanaDatasource), SLO Operator CRD(ServiceLevelObjective)를 클러스터에 적용합니다. 대시보드 변경 PR에서 Grafana Renderer를 사용하여 변경 전후의 대시보드 스크린샷을 자동으로 PR 코멘트에 첨부하면, 비개발자도 시각적으로 변경 내용을 확인하고 승인할 수 있습니다.

---

**Q26. Kubernetes HPA(Horizontal Pod Autoscaler)와 KEDA(Kubernetes Event-Driven Autoscaling)를 Prometheus 메트릭 기반으로 구성하고, 스케일링 결정의 정확도를 높이기 위한 고급 전략을 설명하라.**

A26. HPA의 기본 CPU/메모리 메트릭은 **지연(Lagging) 지표**입니다. CPU가 높아지는 시점은 이미 사용자가 느린 응답을 경험하고 있는 시점이므로, 이 신호로 스케일아웃하면 항상 늦습니다. 따라서 **요청 큐 깊이(Request Queue Depth)**나 **응답 시간 P99 초과율** 같은 **선행(Leading) 지표**를 사용하는 것이 권장됩니다.

KEDA는 Prometheus, Kafka Lag, SQS 큐 길이, Cron 스케줄 등 다양한 외부 메트릭 소스를 스케일링 트리거로 지원합니다. 다음은 Prometheus 쿼리 결과를 기반으로 Deployment를 스케일하는 KEDA ScaledObject 예시입니다.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: payment-api-scaler
spec:
  scaleTargetRef:
    name: payment-api
  minReplicaCount: 3
  maxReplicaCount: 50
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: payment_request_p99_latency
        threshold: "500"   # P99 지연이 500ms를 초과하면 스케일아웃
        query: |
          histogram_quantile(0.99,
            sum(rate(payment_requests_duration_bucket[1m])) by (le)
          )
```

정확도 향상을 위한 고급 전략으로는 **Predictive Autoscaling**이 있습니다. KEDA의 `ScaledJob` 대신, 과거 트래픽 패턴 기반의 예측 모델(Prophet) 결과를 Prometheus에 Pushgateway로 주입하고, KEDA가 이 예측 메트릭을 기반으로 트래픽 도달 전에 미리 스케일아웃합니다. 또한 **스케일인 안정화 윈도우(Stabilization Window)**를 길게 설정(300초)하여 일시적 트래픽 감소 시의 불필요한 스케일인-아웃 반복(Thrashing)을 방지합니다.

---

**Q27. 멀티 클러스터 Kubernetes 환경에서 서비스 A(클러스터 A)가 서비스 B(클러스터 B)를 호출할 때의 End-to-End 트레이스 연결을 어떻게 구현하겠는가? Istio 멀티 클러스터와 OpenTelemetry를 활용하여 설명하라.**

A27. 멀티 클러스터 간 트레이스 연결의 핵심 도전은 **트레이스 컨텍스트가 클러스터 경계를 넘어 보존되어야 한다**는 것입니다. Istio 멀티 클러스터 설정(Primary-Remote 또는 Primary-Primary)에서 클러스터 간 통신은 Istio East-West Gateway를 통해 mTLS로 이루어집니다.

OTel 기반 구현으로는 양쪽 클러스터의 Istio가 Zipkin 트레이스를 각 클러스터의 OpenTelemetry Collector로 전달하도록 구성합니다. OTel Collector는 OTLP 형식으로 변환하여 중앙 Grafana Tempo(또는 Jaeger All-in-One)에 전송합니다. 이때 두 클러스터의 Collector가 동일한 Tempo 백엔드를 가리키면, 동일한 `trace-id`를 공유하는 스팬이 자동으로 단일 트레이스로 조립됩니다. Tempo의 Trace ID 기반 스팬 집계는 스팬 도착 순서나 클러스터 출처에 무관하게 올바르게 동작합니다.

클러스터 정보를 식별하기 위해 OTel Collector의 Resource Processor에서 `k8s.cluster.name` 속성을 모든 스팬에 주입합니다. 이를 통해 Grafana 트레이스 뷰에서 어느 클러스터에서 어느 구간의 지연이 발생했는지 명확히 식별할 수 있습니다.

---

**Q28. Prometheus의 Cardinality Explosion(카디널리티 폭발) 문제를 설명하고, 이를 탐지, 방지, 해결하는 종합적인 전략을 논하라.**

A28. 카디널리티는 시계열 데이터베이스에서 **고유한 레이블 값 조합의 수**입니다. 예를 들어 HTTP 요청 메트릭에 `user_id` 레이블을 추가하면, 사용자 수 × 다른 레이블 조합 수만큼의 시계열이 생성됩니다. 1백만 사용자 × 10개 엔드포인트 × 5개 상태 코드 = 5천만 개의 활성 시계열이 되어, 수십 GB의 메모리를 소비하고 Prometheus 성능을 급격히 저하시킵니다.

탐지는 Prometheus 자체의 메트릭인 `prometheus_tsdb_head_series` 추이를 모니터링하고, `topk(10, count by (__name__)({__name__=~".+"}))` 쿼리로 카디널리티가 가장 높은 메트릭을 식별합니다. Grafana의 Mimirtool `analyse prometheus --address http://prometheus:9090` 명령은 전체 메트릭의 카디널리티 분포와 증가율을 리포팅합니다.

방지 전략으로는 메트릭 네이밍 가이드라인에 "사용자 ID, 요청 ID, 세션 ID, 동적 URL 경로 등 카디널리티가 높은 속성은 레이블로 사용 금지"를 명시합니다. CI 파이프라인에서 `pint`(Prometheus Rule Linter)를 사용하여 고카디널리티 레이블 사용을 PR 단계에서 차단합니다. Prometheus의 `--storage.tsdb.retention.time`과 레코딩 룰을 활용하여 고카디널리티 메트릭을 저카디널리티 집계 메트릭으로 대체합니다.

---

**Q29. OpenTelemetry Collector를 프로덕션에 배포할 때 고가용성(HA), 백프레셔(Backpressure) 처리, 데이터 유실 방지를 어떻게 구성하겠는가?**

A29. 프로덕션 OTel Collector의 배포 아키텍처는 **에이전트(Agent) 계층 + 게이트웨이(Gateway) 계층**의 이중 구조가 표준입니다. 에이전트는 DaemonSet으로 각 노드에 배포되어 로컬 텔레메트리를 수집하고, 게이트웨이는 Deployment(3+ replicas)로 배포되어 집계·변환·라우팅을 담당합니다.

고가용성을 위해 게이트웨이 앞단에 Kubernetes Service(LoadBalancer 또는 Internal NLB)를 두고, 에이전트는 게이트웨이 서비스의 DNS를 가리킵니다. 게이트웨이 파드가 재시작되어도 에이전트는 자동으로 다른 게이트웨이 파드로 연결됩니다.

백프레셔와 데이터 유실 방지를 위해 OTel Collector의 `extensions/file_storage` 익스텐션을 활성화하여 **영속적 큐(Persistent Queue)**를 구성합니다. 이는 Collector 재시작 시에도 큐에 남은 데이터를 유실 없이 재전송합니다. `retry_on_failure` 설정으로 백엔드 불가용 시 지수 백오프 재시도를 구성하고, `sending_queue`의 `queue_size`를 넉넉히 설정하여 일시적 백엔드 장애를 흡수합니다. Collector 자체의 메모리 사용량은 `memory_limiter` 프로세서로 상한을 설정하여 OOM 종료를 방지합니다.

---

**Q30. AIOps와 LLM을 결합한 차세대 자율 운영(Autonomous Operations) 시스템의 아키텍처를 설계하라. 특히 LLM이 알람을 수신하여 RCA를 수행하고 자동 해결 조치를 실행하는 "AI-Powered On-Call Engineer"의 가능성과 한계를 논하라.**

A30. AI-Powered On-Call Engineer의 아키텍처는 **감지 → 컨텍스트 수집 → LLM 추론 → 행동 제안/실행 → 피드백 루프**의 자율 에이전트(Autonomous Agent) 구조로 설계됩니다.

감지 단계에서 PagerDuty 알람이 발생하면 LangChain/LlamaIndex 기반의 에이전트가 활성화됩니다. 컨텍스트 수집 단계에서 에이전트는 다음의 도구(Tool)들을 호출합니다. Prometheus API를 통한 관련 메트릭 쿼리, Loki/CloudWatch Logs API를 통한 오류 로그 수집, Kubernetes API를 통한 파드 상태 및 최근 이벤트 확인, GitHub/Jira API를 통한 최근 배포 및 티켓 확인, 내부 지식베이스(과거 장애 런북) 검색 등을 수행합니다. LLM 추론 단계에서 수집된 컨텍스트를 구조화된 프롬프트로 변환하여 GPT-4/Claude에게 "현재 상황의 가장 가능성 높은 근본 원인 3개와 각각의 권장 대응 조치"를 요청합니다. 행동 실행 단계에서 에이전트는 "안전 조치"(로그 수집, 스냅샷 생성, 알람 억제)는 자율 실행하고, "파괴적 조치"(서비스 재시작, 롤백, 스케일링)는 Slack에 승인 요청을 보낸 후 엔지니어 확인 후 실행합니다.

**현재의 한계**도 솔직하게 논해야 합니다. LLM은 환각(Hallucination)을 생성할 수 있어, 잘못된 RCA를 확신 있게 제시할 위험이 있습니다. 특히 학습 데이터에 없는 신규 유형의 장애에서는 신뢰도가 급격히 하락합니다. 따라서 현 시점에서 AI-Powered On-Call은 **완전 자율 운영보다는 엔지니어의 의사결정을 보조하는 Copilot 수준**이 적절합니다. 행동의 자율성 범위를 명확히 정의하고, 모든 LLM 추론과 실행 이력을 감사 가능(Auditable)하게 기록하며, A/B 테스트를 통해 LLM 제안의 정확도를 지속적으로 측정하고 개선하는 피드백 루프가 필수입니다.

---

## 보너스 질문 (Q31~Q35)

**Q31. Prometheus TSDB의 내부 자료구조(Chunk, Block, WAL)를 설명하고, 쓰기 증폭(Write Amplification) 문제를 최소화하는 운영 방법을 논하라.**

A31. Prometheus TSDB는 시계열 데이터를 두 단계 저장소에 관리합니다. **WAL(Write-Ahead Log)**은 최근 2시간의 수집 데이터를 시간 순서대로 기록하는 순차 로그로, 메모리 내 Head Chunk가 손실될 경우 복구 데이터로 사용됩니다. **TSDB Block**은 2시간 단위로 WAL에서 컴팩션되어 디스크에 기록되는 불변(Immutable) 파일로, `index`(레이블→시계열 역인덱스), `chunks`(샘플 데이터), `meta.json`(블록 메타데이터), `tombstones`(삭제 마커)로 구성됩니다.

쓰기 증폭은 소규모 블록이 반복 병합(Compaction)되는 과정에서 동일 데이터가 여러 번 기록되는 현상입니다. 이를 최소화하기 위해 `--storage.tsdb.min-block-duration`(기본 2h)과 `--storage.tsdb.max-block-duration`(기본 36h)을 조정하여 컴팩션 빈도를 낮춥니다. 대규모 환경에서는 Thanos Compactor가 오브젝트 스토리지의 블록을 2h → 6h → 24h → 2weeks 단위로 점진적 다운샘플링하여 장기 보존 쿼리의 데이터 스캔량을 극적으로 줄입니다.

---

**Q32. Kubernetes Network Policy와 Calico의 정책 적용을 Observability 도구로 감사(Audit)하고, 의도하지 않은 트래픽 차단을 즉각 탐지하는 시스템을 설계하라.**

A32. Network Policy 위반 탐지의 핵심 도전은 **차단된 트래픽은 로그를 생성하지 않는다**는 점입니다. 기본 iptables/Netfilter 기반 Network Policy는 드롭된 패킷에 대한 별도 로그를 생성하지 않습니다.

해결책으로는 세 가지 접근이 있습니다. 첫째, **Calico의 GlobalNetworkPolicy에 Audit 로깅**을 활성화하면 차단된 패킷에 대한 로그를 생성합니다. `action: Log`를 `action: Deny` 앞에 배치하면 차단되는 모든 트래픽이 로깅됩니다. 이 로그를 Fluentd/Fluent Bit로 Elasticsearch 또는 Loki에 전송합니다. 둘째, **eBPF 기반 솔루션(Cilium/Hubble)**은 커널 레벨에서 드롭된 패킷을 네이티브로 가시화합니다. `hubble observe --verdict DROPPED` 명령 또는 Hubble UI에서 실시간으로 차단된 플로우를 확인할 수 있습니다. 셋째, **서비스 관점에서의 간접 탐지**로, TCP 연결 실패율(`tcp_connect_failed_total`)과 DNS 해석 성공률을 Prometheus에서 모니터링하여 Network Policy 강화 이후 급증한 연결 실패를 상관 분석합니다.

---

**Q33. 금융 서비스에서 Kubernetes 클러스터의 CIS Benchmark 컴플라이언스 상태를 실시간으로 모니터링하고, 컴플라이언스 드리프트(Compliance Drift)를 자동으로 복구하는 시스템을 설계하라.**

A33. CIS Kubernetes Benchmark는 API Server 인증, RBAC 최소 권한, 파드 보안 표준(PSS/PSA), 네트워크 암호화, 시크릿 관리 등 200개 이상의 통제 항목으로 구성됩니다. 이를 실시간으로 모니터링하기 위한 도구로 **kube-bench**(CIS 벤치마크 스캐너), **Falco**(런타임 보안 이벤트 탐지), **Kyverno**(정책 엔진)를 조합합니다.

kube-bench는 CronJob으로 매일 실행되어 컴플라이언스 결과를 JSON으로 출력하고, 이를 Prometheus Pushgateway로 `compliance_check_passed{control="1.2.1"}=0/1` 형식의 메트릭으로 노출합니다. Grafana에서 컴플라이언스 항목별 현황을 히트맵으로 시각화하고, 실패 항목은 즉시 알람을 발생시킵니다.

자동 복구(Auto-Remediation)를 위해 Kyverno의 `mutate` 정책을 사용합니다. 예를 들어 `runAsRoot: true`로 배포되는 파드를 자동으로 `runAsNonRoot: true`로 변경하거나, `allowPrivilegeEscalation: true` 설정을 자동으로 `false`로 패치합니다. Kyverno의 `generate` 정책은 새로운 네임스페이스 생성 시 기본 NetworkPolicy(기본 거부 정책)를 자동으로 생성하여 새 네임스페이스의 컴플라이언스를 보장합니다.

---

**Q34. 초대규모 Kubernetes 환경(수천 노드)에서 모니터링 인프라 자체의 비용, 성능, 가용성을 최적화하는 전략을 논하라. "Observability를 모니터링하는 문제(Meta-Observability)"를 어떻게 해결하겠는가?**

A34. Meta-Observability — Observability 인프라 자체의 건강 상태를 모니터링하는 문제 — 는 흔히 간과되지만, Prometheus 자체가 다운되면 아무것도 알 수 없다는 부트스트랩 역설을 가집니다.

이를 해결하기 위해 **이중 Observability 레이어**를 구성합니다. 주 모니터링 스택(Prometheus/Grafana)은 애플리케이션 메트릭을 처리하고, 경량 기반 모니터링 레이어(AWS CloudWatch Agent 또는 Azure Monitor Agent)는 주 스택의 건강 상태(Prometheus 인스턴스의 CPU/메모리/디스크, Grafana Loki 인제스트 지연)를 독립적으로 모니터링합니다. 주 스택이 전부 다운되어도 기반 레이어가 알람을 생성할 수 있습니다.

비용 최적화 전략으로는 메트릭 수집 주기를 중요도에 따라 차등 적용합니다. SLO 메트릭은 15초, 인프라 메트릭은 60초, 비용/용량 메트릭은 300초 주기로 수집합니다. Prometheus의 `--storage.tsdb.retention.time`은 2~4주로 짧게 유지하고, 장기 보관이 필요한 집계 메트릭만 Thanos/Mimir 오브젝트 스토리지로 내보냅니다. 이 전략으로 Prometheus 메모리 사용을 60~70% 절감할 수 있습니다. 레이블 최적화(카디널리티 관리)와 레코딩 룰 활용으로 쿼리 비용을 추가로 50% 줄일 수 있습니다.

---

**Q35. Kubernetes에서 멀티 테넌트 환경의 Observability 격리(Isolation)를 구현하라. 팀 A가 팀 B의 메트릭과 로그를 볼 수 없도록 하면서도, 플랫폼 엔지니어링팀이 전체 클러스터를 관제할 수 있는 아키텍처를 설계하라.**

A35. 멀티 테넌트 Observability 격리는 **데이터 플레인 격리**와 **쿼리 플레인 격리**의 두 레이어에서 구현됩니다.

데이터 플레인 격리를 위해 Grafana Mimir와 Grafana Loki의 **멀티 테넌시(Multi-Tenancy)** 기능을 활성화합니다. 각 요청에 `X-Scope-OrgID: team-a` 헤더를 포함시키면 해당 테넌트의 데이터 격리가 보장됩니다. OTel Collector의 에이전트 레이어에서 네임스페이스 레이블을 기반으로 `X-Scope-OrgID`를 자동으로 설정하여 팀별 텔레메트리가 올바른 테넌트로 라우팅됩니다. Loki는 테넌트별 별도 오브젝트 스토리지 프리픽스를 사용하므로, 데이터 유출은 구조적으로 불가능합니다.

쿼리 플레인 격리를 위해 Grafana의 **Organization 기능** 또는 **Team-based Dashboard 권한**을 활용합니다. 각 팀은 자신의 네임스페이스에 해당하는 Mimir 테넌트와 Loki 테넌트만 데이터 소스로 등록된 Grafana Organization에 접근합니다. 플랫폼 팀은 모든 테넌트를 통합 쿼리할 수 있는 `X-Scope-OrgID: *`(Mimir의 Federation 쿼리) 권한을 가진 별도 Organization을 사용합니다. 이 접근 방식은 RBAC를 Kubernetes 레이어가 아닌 Observability 레이어에서 독립적으로 집행하므로, 새로운 팀이 추가될 때 Kubernetes RBAC 변경 없이 Grafana Organization 추가만으로 격리된 Observability 환경을 프로비저닝할 수 있습니다.

---

# 부록: 핵심 도구 생태계 요약

## AWS 스택
**CloudWatch**(메트릭+로그) → **X-Ray/ADOT**(트레이스) → **Amazon Managed Grafana**(시각화) → **EventBridge+Lambda**(자동화)의 네이티브 스택이 단순하고 유지관리 용이합니다. 대규모 환경에서는 OpenTelemetry → ADOT Collector → CloudWatch + S3(Athena)를 추가합니다.

## Azure 스택
**Azure Monitor + Log Analytics Workspace**(KQL 기반 통합) → **Application Insights**(APM) → **Azure Managed Grafana**(시각화) → **Logic Apps/Azure Automation**(자동화)의 네이티브 스택이 Azure 서비스와 긴밀히 통합됩니다.

## Kubernetes 오픈소스 스택
**Prometheus + Thanos/Mimir**(메트릭) + **Grafana Loki**(로그) + **Grafana Tempo**(트레이스) + **OpenTelemetry Operator**(계측 자동화) + **Grafana**(통합 시각화)의 LGTM 스택이 사실상의 업계 표준입니다. Cilium/Hubble(eBPF 네트워크)과 DCGM Exporter(GPU)를 추가하면 완전한 관측가능성 플랫폼이 구성됩니다.
