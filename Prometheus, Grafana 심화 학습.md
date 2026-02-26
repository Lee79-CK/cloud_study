# Prometheus & Grafana: 박사급 심화 가이드
### Senior Cloud Engineer를 위한 완전 참조 문서

---

## Part I. Prometheus 아키텍처 심층 분석

### 1.1 Prometheus의 데이터 모델과 시계열 수학적 기반

Prometheus의 핵심은 **다차원 시계열 데이터 모델(Multi-dimensional Time Series Data Model)**이다. 각 시계열(time series)은 metric 이름과 레이블(label) 집합으로 유일하게 식별되는 벡터 공간의 한 점으로 정의할 수 있다.

수학적으로, 하나의 시계열은 다음과 같이 표현된다:

```
metric_name{label_1="v1", label_2="v2", ..., label_n="vn"} → (timestamp_i, value_i)
```

이 구조는 관계형 DB의 테이블과 근본적으로 다르다. RDBMS는 schema-first이지만, Prometheus는 **레이블 기반 고차원 공간**을 구성하여 쿼리 시점에 집계 차원을 자유롭게 선택한다. 이 설계 철학은 Google의 Borgmon에서 직접 영향을 받았으며, SRE(Site Reliability Engineering)의 USE/RED 방법론과 자연스럽게 결합된다.

**Cardinality(카디널리티) 문제**는 여기서 파생된다. 레이블 값의 고유한 조합 수가 시계열 수를 결정하는데, 카디널리티가 폭발적으로 증가하면(cardinality explosion) TSDB의 메모리와 디스크가 선형이 아닌 기하급수적으로 증가한다. 예를 들어 `user_id`, `request_id` 같은 고유값을 레이블로 사용하는 것은 안티패턴의 교과서적 사례다.

### 1.2 TSDB(Time Series Database) 내부 구조

Prometheus v2.x부터 사용되는 내장 TSDB는 Fabian Reinartz가 설계했으며, 다음의 핵심 개념으로 구성된다.

**Block 구조:** 데이터는 2시간 단위의 불변(immutable) Block으로 저장된다. 각 Block은 chunks 디렉토리(실제 샘플 데이터), index 파일(시계열 메타데이터), tombstones 파일(삭제 마킹), meta.json으로 구성된다. 이 설계는 쓰기 증폭(write amplification)을 최소화하고, 오래된 데이터를 atomic하게 삭제하기 위함이다.

**WAL(Write-Ahead Log):** 메모리의 Head Block에 쓰기 전 WAL에 먼저 기록하여 크래시 복구를 보장한다. WAL은 128MB 세그먼트로 분할되며, checkpoint를 통해 압축된다. WAL replay 시간이 길어지면 Prometheus 재시작이 느려지는 실제 운영 문제로 이어진다.

**Chunk 인코딩:** Prometheus는 XOR 기반의 **Gorilla 압축 알고리즘**(Facebook이 2015년 논문에서 발표)을 사용한다. 이 알고리즘은 연속된 부동소수점 값들이 서로 비슷할 때(가우시안 분포에 가까운 메트릭) 매우 높은 압축률(평균 1.37 bytes/sample)을 달성한다. 타임스탬프도 delta-of-delta 인코딩으로 압축한다.

**Compaction:** 백그라운드에서 여러 Block을 병합하여 더 큰 Block으로 만드는 작업이다. Level-based compaction을 사용하며, Level 0은 2시간, 이후 레벨은 이전 레벨의 5배 크기까지 병합한다. 이 과정에서 삭제된 시계열과 tombstone이 실제로 제거된다.

### 1.3 스크래핑(Scraping) 메커니즘과 Pull 모델의 철학

Prometheus의 Pull 방식은 단순히 구현 선택이 아니라 **관찰 가능성(Observability) 철학**을 반영한다. Push 모델(예: StatsD, InfluxDB)과 비교하면:

Pull 모델의 핵심 장점은 **Prometheus 서버가 모든 scrape 대상의 건강 상태를 알 수 있다**는 점이다. 대상이 응답하지 않으면 `up{job="..."}` 메트릭이 0이 된다. Push 모델에서는 대상이 메트릭을 보내지 않을 때 장애인지 설정 오류인지 구분이 어렵다. 또한 Pull 모델은 네트워크 ACL 설정이 단방향으로 단순화되고, 스크래핑 간격을 Prometheus에서 중앙 통제할 수 있다.

단, **Pushgateway**는 Pull 모델의 예외로, 배치 잡(batch job)처럼 생명주기가 짧아 스크래핑을 받을 수 없는 경우를 위한 중간 집합소다. Pushgateway는 메트릭을 영속적으로 유지하므로, 잡이 실패해도 마지막 값이 남아있다. 이 때문에 `up` 메트릭의 의미가 희석되고, Pushgateway 자체가 SPOF(Single Point of Failure)가 될 수 있다는 운영상 주의가 필요하다.

### 1.4 PromQL 심층 분석: 벡터 연산과 함수형 패러다임

PromQL은 함수형 언어에 가까운 쿼리 언어로, 다음 네 가지 데이터 타입을 다룬다.

**Instant Vector**는 특정 시점의 시계열 집합이고, **Range Vector**는 시간 범위(`[5m]` 등) 내의 샘플 집합이다. Range Vector는 직접 그래프에 표시할 수 없고, `rate()`, `increase()`, `avg_over_time()` 같은 함수의 입력으로만 사용된다.

**`rate()` vs `irate()` vs `increase()`의 미묘한 차이**는 면접에서 자주 등장한다. `rate(counter[5m])`은 5분 윈도우 내 모든 샘플을 사용한 선형 회귀 기반의 초당 평균 증가율이다. 반면 `irate(counter[5m])`은 가장 최근 두 샘플만 사용한 순간 증가율로, 스파이크에 매우 민감하다. `increase(counter[5m])`은 `rate()` × 300초이며, counter reset을 자동으로 처리한다. 중요한 것은 `increase()`가 정수를 반환하지 않는다는 점이다 — 스크래핑 간격이 정확히 맞아떨어지지 않을 때 보간(interpolation)이 발생하므로 소수점 값이 나올 수 있다.

**Staleness Handling:** Prometheus는 스크래핑이 실패하거나 시계열이 사라질 때 **stale marker**를 삽입한다. 이 특수 NaN 값은 다음 스크래핑 이후 5분이 지나면 그래프에서 해당 시계열이 사라지게 한다. 이 동작을 이해하지 못하면 대시보드에서 간헐적으로 그래프가 끊기는 현상의 원인을 찾지 못한다.

**벡터 매칭(Vector Matching):** 두 Instant Vector를 이항 연산(binary operation)할 때 레이블 매칭 규칙을 정확히 이해해야 한다.

```promql
-- Many-to-one 매칭: "on" 절로 매칭 키를 지정하고 "group_left"로 다 대 일 방향을 명시
http_requests_total * on(instance) group_left(version) node_info
```

`on()` 없이 연산하면 양쪽 시계열의 **모든 레이블이 완전히 일치**해야 한다. 이를 모르면 결과가 비어있어 당황하게 된다.

### 1.5 Recording Rules와 Alert Rules의 설계 원칙

Recording Rule은 비용이 큰 PromQL 쿼리를 사전에 계산하여 새로운 시계열로 저장하는 메커니즘이다. 단순히 성능 최적화가 아니라 **집계 계층(aggregation hierarchy)**을 설계하는 도구다.

네이밍 컨벤션은 `level:metric:operations` 형식을 따른다. 예를 들어 `job:http_inprogress_requests:sum`은 job 레벨에서 http_inprogress_requests를 sum 집계한 결과임을 명시한다. 이 컨벤션은 Prometheus 공식 가이드에서 권장하며, Thanos/Cortex 같은 장기 저장소와의 연동 시 관리 복잡도를 낮춘다.

**Alert Rule 설계의 핵심은 `for` 절이다.** `for: 5m`은 조건이 5분 연속으로 충족될 때만 알림을 발송한다. 이 없이 알림을 보내면 순간적인 스파이크나 스크래핑 실패로 인한 노이즈가 폭발한다. 알림 상태는 Inactive → Pending → Firing 세 단계를 거친다.

### 1.6 고가용성(HA)과 장기 저장소 전략: Thanos vs Cortex vs Mimir

단일 Prometheus 인스턴스는 **로컬 저장소의 한계(기본 15일)**와 **단일 장애 지점** 문제를 가진다. 이를 해결하는 아키텍처가 CNCF 생태계에서 여러 갈래로 발전했다.

**Thanos 아키텍처**는 Sidecar 패턴을 기반으로 한다. Thanos Sidecar는 Prometheus와 함께 배포되어, 완료된 Block을 오브젝트 스토리지(S3, GCS, Azure Blob)에 업로드한다. Thanos Querier는 여러 Sidecar와 Store Gateway에 gRPC로 팬아웃(fan-out) 쿼리를 보내고 결과를 병합한다. Thanos Ruler는 Recording/Alert Rule을 분산 환경에서 실행하고, Thanos Compactor는 오브젝트 스토리지의 Block을 장기 압축(downsampling: 5분 해상도, 1시간 해상도)한다.

**Cortex/Grafana Mimir**는 완전히 다른 접근으로, Prometheus를 **마이크로서비스 클러스터**로 분해한다. Distributor(ingestion), Ingester(메모리 저장), Querier(쿼리), Compactor, Store-Gateway 등이 독립 스케일링 가능한 컴포넌트로 분리된다. 내부적으로 일관된 해싱(consistent hashing)을 사용한 링(ring) 아키텍처로 데이터를 분산한다. Mimir는 Cortex의 fork로 Grafana Labs가 Apache 2.0 라이선스로 공개했으며, 현재 가장 성숙한 Prometheus 장기 저장소 솔루션 중 하나다.

**VictoriaMetrics**는 이 두 솔루션과 달리 완전히 재작성된 TSDB로, Prometheus의 remote_write 프로토콜과 호환되며, 단일 바이너리로 Thanos보다 훨씬 낮은 운영 복잡도를 제공한다. 메모리 효율성이 특히 뛰어나다.

---

## Part II. Grafana 아키텍처 심층 분석

### 2.1 Grafana의 데이터 플로우와 Plugin 아키텍처

Grafana는 **데이터 시각화 플랫폼**으로, 데이터 소스와 시각화 사이의 추상화 레이어를 제공한다. 내부 데이터 플로우는 다음과 같다.

브라우저의 패널이 쿼리를 요청하면, Grafana Backend의 **Query Service**가 이를 받아 해당 데이터 소스 플러그인을 통해 실제 백엔드(Prometheus, Loki, Elasticsearch 등)에 쿼리를 보낸다. 결과는 **DataFrame** 형식으로 정규화되어 프론트엔드로 전달되고, 패널 플러그인이 이를 시각화한다.

**Plugin 시스템**은 세 가지 유형이다. 데이터 소스 플러그인(data source plugin), 패널 플러그인(panel plugin), 앱 플러그인(app plugin)으로 나뉜다. Grafana 10.x부터 강화된 **Plugin Sandboxing**은 플러그인을 Web Worker 내에서 실행하여 악성 플러그인이 메인 스레드를 블록킹하거나 전역 상태를 오염시키는 것을 방지한다.

### 2.2 Grafana Loki: 로그 집계 시스템

Loki는 Prometheus 철학을 로그에 적용한 시스템이다. 핵심 차이는 **인덱싱 전략**이다. Elasticsearch는 로그의 전체 내용을 인덱싱하지만, Loki는 **레이블만 인덱싱**하고 로그 내용은 청크(chunk)로 압축 저장한다. 이 설계는 인덱스 크기를 수십 배 줄이지만, 특정 로그 내용으로 검색할 때 모든 청크를 스캔해야 하는 트레이드오프가 있다.

**LogQL**은 두 단계로 구성된다. 먼저 레이블 셀렉터로 관련 스트림을 필터링하고(`{app="nginx"}`), 이후 파이프라인 표현식으로 라인 필터링 및 파싱을 수행한다(`| json | status_code >= 500`). `logfmt`, `json`, `regex`, `pattern` 파서를 통해 구조화되지 않은 로그에서 레이블을 추출하여 메트릭으로 변환할 수 있는 `rate()`와 결합이 가능하다.

**Loki의 분산 아키텍처:** Distributor는 Consistent Hash Ring을 통해 Ingester로 로그를 라우팅한다. Ingester는 인메모리 청크를 보유하다가 플러시 시점에 오브젝트 스토리지에 쓴다. Querier는 인제스터와 오브젝트 스토리지 양쪽을 쿼리하여 결과를 병합한다.

### 2.3 Grafana Tempo: 분산 트레이싱

Tempo는 트레이스를 **TraceID 기반으로 직접 오브젝트 스토리지에 저장**하는 백엔드다. Jaeger, Zipkin과의 핵심 차이는 인덱스가 없다는 점이다. TraceID를 알면 즉시 조회할 수 있지만, 서비스 맵이나 검색은 별도 인덱스(tempodb의 bloom filter 기반)가 필요하다.

**Exemplar**는 Prometheus와 Tempo를 연결하는 핵심 메커니즘이다. Prometheus 메트릭 샘플에 `trace_id` 레이블을 첨부하면, Grafana UI에서 메트릭 스파이크 시점의 트레이스로 원클릭 드릴다운이 가능하다. 이것이 **Grafana의 LGTM 스택(Loki, Grafana, Tempo, Mimir)이 구현하는 세 개의 기둥(Three Pillars of Observability)**이다.

### 2.4 Grafana Alerting Architecture (Unified Alerting)

Grafana 8.x부터 도입된 Unified Alerting은 Grafana 내부에서 **Prometheus-compatible 알림 엔진**을 실행한다. 내부적으로 Alertmanager의 컴포넌트를 임베드하여 라우팅, 그루핑, 억제(inhibition), 사일런싱을 지원한다.

**알림 평가 주기(evaluation interval)**와 `for` 절의 상호작용을 이해하는 것이 중요하다. 예를 들어 evaluation interval이 1분이고 `for: 5m`이면, Grafana는 매 1분마다 조건을 체크하여 5번 연속 충족 시(5분 후) Firing 상태로 전환한다. 이 계산이 틀리면 알림 지연 문제가 발생한다.

**Contact Point와 Notification Policy**의 분리 설계는 알림 라우팅의 유연성을 높인다. 알림의 메타데이터(레이블)에 따라 다른 Contact Point(PagerDuty, Slack, OpsGenie 등)로 라우팅하는 트리 구조를 정의할 수 있다.

### 2.5 Grafana as Code: IaC 패턴

**Grafana Terraform Provider**를 사용하면 대시보드, 데이터 소스, 알림 정책을 코드로 관리할 수 있다. 이 접근의 핵심 과제는 대시보드 JSON의 drift 관리다.

**Grafonnet**은 Jsonnet 기반 Grafana 대시보드 생성 라이브러리로, 재사용 가능한 패널과 변수를 함수로 정의하여 DRY(Don't Repeat Yourself) 원칙을 적용한다.

**Grizzly**는 Grafana Labs가 만든 CLI 도구로, YAML/Jsonnet으로 정의된 Grafana 리소스를 API를 통해 배포한다. CI/CD 파이프라인에 Grafana 대시보드 배포를 통합할 때 사용한다.

---

## Part III. 고급 운영 패턴

### 3.1 멀티 클러스터, 멀티 테넌트 Observability 아키텍처

대기업 환경에서는 수십~수백 개의 Kubernetes 클러스터를 운영한다. 각 클러스터에 Prometheus를 배포하고 Thanos/Mimir로 중앙 집계하는 **Federation 패턴**이 표준이다.

**Prometheus Federation**의 두 가지 방식을 구분해야 한다. 계층적 Federation은 하위 Prometheus가 집계된 메트릭만 상위 Prometheus에 노출하는 방식으로, 원본 레이블이 손실될 수 있다. Remote Write는 원본 시계열을 그대로 중앙 저장소로 전송하며, 현재 권장되는 방식이다.

**멀티 테넌트 격리**는 Mimir의 `X-Scope-OrgID` 헤더 기반 테넌트 분리를 통해 구현한다. 각 팀/서비스가 독립된 메트릭 공간을 가지며, 쿼리 시 교차 조회를 방지한다.

### 3.2 SLO/SLA 기반 알림 설계

Google SRE의 **Error Budget** 개념을 Prometheus에 구현하는 것은 고급 패턴이다. **Sloth**와 **pyrra** 같은 오픈소스 도구는 SLO 정의로부터 Recording Rule과 Alert Rule을 자동 생성한다.

멀티 윈도우, 멀티 번-레이트(multi-window, multi-burn-rate) 알림은 단기 스파이크와 장기 저하를 동시에 감지한다. 예를 들어 1시간 번-레이트가 14.4 이상(월 에러 버짓의 2%를 1시간에 소모) **AND** 5분 번-레이트가 14.4 이상인 경우 즉시 호출(page)하고, 6시간 번-레이트가 6 이상 AND 30분 번-레이트가 6 이상인 경우 티켓을 생성하는 방식이다. 이 접근은 False Positive를 최소화하면서 빠른 감지를 보장한다.

### 3.3 Prometheus Operator와 Kubernetes 통합

**kube-prometheus-stack**은 Helm Chart로 Prometheus Operator, Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics를 일괄 배포한다. Prometheus Operator는 다음 CRD를 통해 Prometheus 설정을 선언적으로 관리한다.

`ServiceMonitor`는 Service 오브젝트를 기반으로 스크래핑 타겟을 자동 발견하며, `PodMonitor`는 Service 없이 Pod를 직접 스크래핑한다. `PrometheusRule`은 Recording Rule과 Alert Rule을 CRD로 정의하여 Operator가 Prometheus에 로드한다. `AlertmanagerConfig`는 Alertmanager 라우팅을 네임스페이스 레벨에서 분리하여 팀별 알림 설정을 가능하게 한다.

**RBAC와 결합한 보안 설계:** Prometheus의 Kubernetes SD(Service Discovery)는 API 서버에 접근해야 한다. 최소 권한 원칙에 따라 `pods`, `services`, `endpoints`, `nodes`에 대한 `get`, `list`, `watch` 권한만 부여하고, namespace 범위를 제한하는 것이 중요하다.

---

## Part IV. 빅테크 면접 Q&A (30문항+)

---

### [Section A: Prometheus 핵심 개념]

**Q1. Prometheus의 Pull 모델이 대규모 마이크로서비스 환경에서 갖는 본질적인 한계는 무엇이며, 어떻게 해결하는가?**

**A:** Pull 모델의 첫 번째 한계는 **네트워크 토폴로지 제약**이다. Prometheus 서버가 스크래핑 대상에 직접 TCP 연결을 열 수 있어야 하므로, 방화벽이나 NAT 뒤의 대상은 직접 스크래핑이 불가능하다. 두 번째는 **스케일 한계**다. 단일 Prometheus 인스턴스는 약 100만~200만 활성 시계열을 처리할 수 있는데, 수천 개의 마이크로서비스 환경에서는 이 한계에 도달한다.

해결책으로는 여러 Prometheus 인스턴스를 **샤딩(sharding)**하여 서비스 세트를 분담하고, Thanos Querier를 통해 통합 쿼리 레이어를 제공한다. 네트워크 제약은 **PushProx**나 **Prometheus Agent Mode**(v2.32 이후)로 해결한다. Agent Mode는 스크래핑만 수행하고 로컬 저장 없이 바로 Remote Write하므로, 메모리와 디스크 사용량이 훨씬 낮다. 에지 환경이나 IoT처럼 리소스가 제한된 환경에 이상적이다.

---

**Q2. `rate()`와 `irate()` 중 어떤 상황에서 무엇을 선택하며, 그 이유는 무엇인가?**

**A:** `rate()`는 지정된 시간 윈도우의 모든 샘플을 사용하는 **시간 가중 평균**이므로 노이즈에 강하고 장기 트렌드 분석에 적합하다. 특히 Alerting Rule에서 `rate()`를 사용해야 하는데, 이는 단기 스파이크로 인한 False Positive를 줄이기 위함이다.

반면 `irate()`는 가장 최근 두 샘플만 사용하므로 **현재 순간의 속도**를 정확히 반영한다. CPU 사용률처럼 빠르게 변하는 메트릭을 실시간 대시보드에 표시할 때 유용하다. 단, 스크래핑 실패로 샘플 하나가 누락되면 완전히 다른 값을 반환할 수 있다.

실무에서 추가로 고려할 것은 **윈도우 크기**다. `rate(http_requests_total[5m])`에서 5분은 스크래핑 간격의 최소 4배 이상이어야 통계적으로 의미 있다. 스크래핑 간격이 30초면 윈도우를 2분 이상으로 설정해야 한다.

---

**Q3. Prometheus에서 Counter가 리셋(reset)될 때 `rate()`는 어떻게 처리하는가? 이 처리 방식의 한계는?**

**A:** Prometheus는 Counter 값이 이전 샘플보다 작아지면 리셋이 발생했다고 판단하고, **리셋 직전 값을 보정치로 더한다.** 예를 들어 샘플 시퀀스가 100, 200, 10(리셋), 50이면 `increase()`는 (200-100) + (200+50-200) = 150으로 계산한다.

단, 이 처리에는 두 가지 한계가 있다. 첫째, **다중 리셋(multiple resets)**이 단일 스크래핑 간격 내에 발생하면 감지하지 못한다. 둘째, **서비스 재시작 시간이 스크래핑 간격보다 짧으면**(즉, 재시작 후 이미 값이 증가한 상태로 스크래핑되면) 리셋 감지 자체가 틀릴 수 있다. 이런 경우 실제보다 낮은 `rate()` 값이 나올 수 있다.

---

**Q4. Prometheus의 카디널리티 폭발(cardinality explosion) 문제를 프로덕션에서 진단하고 해결한 경험을 설명하라.**

**A:** 진단 시작점은 `prometheus_tsdb_head_series` 메트릭으로 현재 활성 시계열 수를 확인하는 것이다. 급증하는 경우 `topk(10, count by(__name__)({__name__=~".+"}))` 쿼리로 어느 메트릭이 가장 많은 시계열을 생성하는지 특정한다.

원인이 확인되면 **relabeling 설정**으로 고카디널리티 레이블을 드롭하거나 값을 해시로 치환한다. Prometheus Operator 환경에서는 `ServiceMonitor`의 `metricRelabelings` 필드에 `action: labeldrop` 또는 `action: replace`를 추가한다. 근본적 해결은 애플리케이션 코드에서 `user_id` 같은 고유값을 레이블로 사용하지 않도록 수정하는 것이다.

예방 차원에서 **`max_samples_per_send`와 `remote_write` 큐 모니터링**, 그리고 CI/CD 파이프라인에 **PromQL 린터(pint)**를 통합하여 고카디널리티 레이블 사용을 빌드 단계에서 차단하는 것이 권장된다.

---

**Q5. Prometheus에서 Recording Rule의 네이밍 컨벤션은 왜 중요한가? `level:metric:operations` 형식의 배경을 설명하라.**

**A:** 네이밍 컨벤션은 단순한 스타일 가이드가 아니라 **쿼리 발견 가능성(discoverability)과 계층적 집계(hierarchical aggregation)**를 위한 계약이다.

`level`은 집계의 단위(예: `job`, `instance`, `cluster`)를, `metric`은 원본 메트릭 이름을, `operations`는 수행된 집계 함수(`sum`, `rate`, `sum_rate`)를 나타낸다.

이 체계의 가장 큰 가치는 **연쇄 Recording Rule**을 만들 때 드러난다. `instance:http_requests:rate5m`을 먼저 계산하고, 이를 기반으로 `job:http_requests:rate5m`을 계산하면, 최종 쿼리가 전체 원시 시계열이 아닌 미리 계산된 소수의 시계열만 참조한다. 이 설계 없이는 Thanos나 Mimir 같은 분산 환경에서 글로벌 집계 쿼리의 지연이 폭발적으로 증가한다.

---

**Q6. Prometheus Alertmanager의 `inhibition rule`과 `silences`의 차이는 무엇인가?**

**A:** 두 개념 모두 알림 억제지만 적용 시나리오가 다르다. **Inhibition Rule**은 **자동화된 인과 관계 기반 억제**다. 예를 들어 `NodeDown` 알림이 발생하면, 해당 노드의 모든 Pod 관련 알림(`PodCrashLooping` 등)을 자동으로 억제한다. 이는 근본 원인(root cause) 알림만 남기고 하위 증상 알림을 제거하는 데 사용한다. 설정은 `inhibit_rules` 블록에 source_matchers와 target_matchers로 정의한다.

**Silences**는 **수동 일시 정지**로, 계획된 유지보수(maintenance window) 동안 모든 관련 알림을 지정된 기간 동안 억제한다. Alertmanager UI나 `amtool` CLI로 생성하며, 만료 시간이 있다. 알림의 구조적 원인을 해결하지 않고 노이즈를 임시로 제거하는 도구이므로 남용을 경계해야 한다.

---

**Q7. Prometheus의 `external_labels` 설정의 역할과 중요성을 설명하라.**

**A:** `external_labels`는 Prometheus가 수집한 모든 시계열에 **전역적으로 추가되는 레이블**로, 주로 `cluster`와 `replica` 레이블을 설정한다.

이것이 중요한 이유는 **Thanos/Mimir 환경의 중복 제거(deduplication)**와 직결되기 때문이다. HA 구성에서 두 Prometheus 인스턴스가 동일한 타겟을 스크래핑하면 데이터가 중복된다. Thanos Querier는 `replica` 레이블을 기준으로 중복을 제거(deduplicate)하는데, 이 레이블이 없으면 동일 데이터가 두 번 표시된다.

또한 Alertmanager로 알림을 보낼 때 `cluster` 레이블이 붙어 있으면, 어느 클러스터에서 발생한 알림인지 즉시 식별할 수 있다.

---

### [Section B: PromQL 심화]

**Q8. 다음 쿼리의 문제점을 찾아라: `sum(http_requests_total) / sum(http_requests_total{status=~"5.."})` 이 쿼리가 에러율을 올바르게 계산하지 못하는 이유는?**

**A:** 이 쿼리에는 두 가지 심각한 문제가 있다. 첫째, **Counter를 직접 나누는 것**은 의미가 없다. Counter는 누적값이므로, 이 쿼리는 "시작 이후 전체 요청 수 대비 5xx 누적 수"를 계산하는데 이는 현재 에러율이 아니다. 올바른 쿼리는 `rate()`를 사용해야 한다.

둘째, **레이블 매칭 문제**가 있다. 두 Vector의 레이블 집합이 달라서(두 번째는 `status` 레이블이 있고 첫 번째는 없음) 나눗셈 결과가 비어있거나 잘못된 값이 나올 수 있다.

올바른 쿼리는 다음과 같다:
```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m]))
```
여기서 분모와 분자 모두 `status` 레이블을 기준으로 sum하므로 레이블이 자연스럽게 제거되어 매칭된다.

---

**Q9. `histogram_quantile(0.99, sum by(le) (rate(http_duration_bucket[5m])))`에서 정확도에 영향을 미치는 요소를 설명하라.**

**A:** Prometheus Histogram은 사전 정의된 버킷(bucket) 경계에 값을 카운팅하므로 **버킷 설계가 정확도의 핵심**이다. `histogram_quantile()`은 버킷 간 선형 보간(linear interpolation)을 사용하므로, p99 값이 실제로 속한 버킷의 상/하한 범위가 정확도의 상한을 결정한다.

예를 들어 버킷이 `[0.1, 0.5, 1.0, 5.0, 10.0]`초이고 p99가 1.5초라면, 1.0~5.0 버킷 내에서 선형 보간하므로 최대 3.5초의 오차가 발생할 수 있다. 반면 버킷이 지수적으로 촘촘하게 설계되어 있으면 오차가 줄어든다.

또한 `sum by(le)`를 사용하면 **여러 인스턴스의 히스토그램을 합산**하는데, 이는 각 인스턴스가 동일한 버킷 경계를 사용할 때만 의미가 있다. 버킷이 다른 히스토그램을 합산하면 완전히 부정확한 결과가 나온다. **Native Histogram(Prometheus 2.40+)**은 이 문제를 해결하여 동적 버킷과 더 높은 정확도를 제공한다.

---

**Q10. Prometheus의 `absent()` 함수와 `absent_over_time()`의 사용 사례와 주의점을 설명하라.**

**A:** `absent(metric_name{labels})` 는 해당 시계열이 **존재하지 않을 때** 값 1을 반환하고, 존재할 때는 아무것도 반환하지 않는다. 이는 **"메트릭 자체가 사라졌을 때"** 알림을 보내는 데 사용한다. 예를 들어 exporter가 다운되면 해당 메트릭이 스크래핑되지 않아 시계열이 사라지는데, `up{job="my-exporter"} == 0`으로는 이를 감지할 수 없다. `absent(up{job="my-exporter"})`를 사용해야 한다.

주의점은 **스크래핑 실패 직후 약 5분의 stale 기간**이 있다는 것이다. 이 기간 동안은 `absent()`가 1을 반환하지 않으므로, 알림의 실제 발생은 스크래핑 실패 후 5분 뒤다. `absent_over_time(metric[5m])`은 지정된 범위 내에 해당 메트릭의 샘플이 하나도 없으면 1을 반환하여 이 지연을 줄일 수 있다.

---

### [Section C: 고가용성 및 스케일링]

**Q11. Thanos와 Cortex/Mimir 중 어떤 아키텍처를 선택하겠는가? 결정 요인은?**

**A:** 이 결정은 **운영 복잡도 대 스케일 요구사항**의 트레이드오프로 귀결된다.

**Thanos**는 기존 Prometheus 클러스터에 사이드카 방식으로 추가하므로 마이그레이션이 점진적이다. 운영팀이 Prometheus에 이미 숙련되어 있다면 Thanos가 학습 곡선이 낮다. 단, Thanos Querier의 fan-out 쿼리는 많은 Sidecar가 있을 때 레이턴시가 증가한다. 또한 Thanos Ruler와 Compactor는 상태가 있어 별도 관리가 필요하다.

**Mimir**는 완전한 멀티 테넌트 지원, 수평 스케일링, 낮은 쿼리 레이턴시(중앙 집중식 인제스터)를 제공하지만, 운영할 마이크로서비스 수가 많아 복잡도가 높다. 수천 개의 팀이 공유하는 중앙 Observability 플랫폼이라면 Mimir가 적합하다.

스타트업이나 중소 규모(<20개 클러스터)라면 **VictoriaMetrics 단일 바이너리**가 최고의 운영 단순성과 성능을 제공한다.

---

**Q12. Remote Write의 신뢰성을 보장하기 위한 설정과 모니터링 방법은?**

**A:** Remote Write는 기본적으로 **WAL(Write-Ahead Log) 기반 재시도**로 내구성을 보장한다. 원격 엔드포인트가 일시적으로 불가 시 WAL에 데이터가 쌓이고 복구 후 전송된다. 단, WAL은 Prometheus의 로컬 보존 기간(기본 2시간) 이상 데이터를 보존하지 않으므로 **장기간 원격 엔드포인트 장애 시 데이터 손실이 발생**한다.

모니터링 핵심 메트릭은 `prometheus_remote_storage_samples_pending`(큐 대기 중인 샘플 수), `prometheus_remote_storage_failed_samples_total`(전송 실패 누적), `prometheus_remote_storage_queue_highest_sent_timestamp_seconds`(마지막으로 성공한 샘플의 타임스탬프)다. 마지막 메트릭과 현재 시간의 차이가 커지면 지연이 발생하고 있다는 신호다.

설정 측면에서 `max_shards`를 적절히 늘려 병렬 전송을 높이고, `capacity`와 `max_samples_per_send`를 튜닝하여 배치 크기를 최적화한다. `min_backoff`와 `max_backoff`는 지수 백오프 재시도 간격을 제어한다.

---

**Q13. Amazon, Google 같은 클라우드 환경에서 Prometheus를 위한 스토리지 비용을 최적화하는 전략은?**

**A:** 스토리지 비용 최적화의 첫 번째 레버는 **데이터 보존 정책의 계층화**다. 모든 데이터를 동일한 해상도로 장기 보존하는 것은 비용 낭비다. Thanos Compactor의 **다운샘플링(downsampling)**을 활성화하면 원본 15초 해상도 데이터를 5분 해상도로 40일 후, 1시간 해상도로 1년 후 자동 다운샘플링하여 스토리지를 90% 이상 줄일 수 있다.

두 번째는 **카디널리티 최적화**다. 불필요한 레이블 제거, 저사용 메트릭의 스크래핑 비활성화, `metric_relabeling` 설정으로 고카디널리티 레이블을 집계 전에 드롭한다.

세 번째는 **S3 스토리지 클래스 자동 전환**이다. S3 Lifecycle Policy를 설정하여 30일 이후 데이터를 S3 Standard-IA, 90일 이후에는 Glacier Instant Retrieval로 이동하면 장기 저장 비용을 크게 절감할 수 있다.

마지막으로 VPC 내에서 S3에 접근할 때 **VPC Gateway Endpoint**를 사용하면 데이터 전송 비용(NAT Gateway 비용)을 제거할 수 있다. AWS 환경에서 Thanos/Mimir를 운영할 때 이 비용이 예상외로 크다.

---

### [Section D: Grafana 심화]

**Q14. Grafana의 `__interval__`과 `__rate_interval__` 변수의 차이를 설명하고, 언제 어떤 것을 사용해야 하는가?**

**A:** `$__interval__`은 Grafana가 현재 시간 범위와 패널의 최대 데이터 포인트 수를 기반으로 **자동 계산한 쿼리 간격**이다. 예를 들어 24시간 범위를 1000포인트로 보려면 약 86초 간격이 된다.

`$__rate_interval__`은 Prometheus 2.3+에서 도입된 변수로, `$__interval__`에 **스크래핑 간격의 4배를 더한 값**이다. 이는 `rate()` 함수에 사용할 때 스크래핑 간격보다 충분히 큰 윈도우를 보장하기 위함이다. `rate(http_requests_total[$__rate_interval__])`처럼 사용하면 어떤 스크래핑 간격 설정에서도 올바르게 동작한다.

**실용 규칙:** `rate()`, `irate()`, `increase()`의 인수에는 항상 `$__rate_interval__`을 사용하고, 단순 집계나 Histogram에서는 `$__interval__`을 사용한다.

---

**Q15. Grafana 대시보드 성능이 느릴 때 어떻게 진단하고 최적화하는가?**

**A:** 진단은 **브라우저 개발자 도구의 Network 탭**에서 각 패널의 쿼리 응답 시간을 확인하는 것부터 시작한다. 느린 패널을 특정했으면 해당 쿼리를 Grafana Explore에서 실행하고, Prometheus의 `/api/v1/query_range` 응답 헤더의 `X-Prometheus-Query-Time`을 확인한다.

최적화 전략은 다음 우선순위로 진행한다. 첫째, 느린 쿼리에 대해 **Recording Rule**을 생성하여 사전 계산한다. 둘째, 쿼리가 불필요하게 많은 레이블 차원을 조회하는지 확인하고 `by()` 절로 필요한 레이블만 남긴다. 셋째, 패널의 **쿼리 캐싱**을 활성화한다(Grafana Enterprise 또는 Mimir/Cortex의 쿼리 프론트엔드 캐시). 넷째, 시간 범위에 맞는 **다운샘플링된 데이터를 사용**하도록 데이터 소스를 Thanos Store Gateway로 향하게 하고, 적절한 해상도의 데이터가 자동 선택되도록 `sampleMilliseconds` 힌트를 활용한다.

---

**Q16. Grafana에서 대시보드를 코드로 관리할 때 발생하는 "dashboard drift" 문제를 어떻게 해결하는가?**

**A:** Dashboard Drift는 누군가가 UI에서 대시보드를 직접 수정하면 Git의 코드와 실제 Grafana 상태가 달라지는 현상이다.

해결책은 **두 가지 레이어**가 필요하다. 기술적 레이어로는 Grafana의 **`editable: false`** 설정으로 프로덕션 대시보드를 읽기 전용으로 만들고, Grafana API를 통한 배포를 CI/CD로만 허용한다(서비스 어카운트 토큰 외 모든 편집 권한 제거). **Grizzly** 또는 **Grafana Terraform Provider**로 모든 대시보드를 코드로 관리한다.

프로세스 레이어로는 "대시보드를 수정하려면 PR을 열어야 한다"는 팀 문화를 정착시킨다. 개발 환경에서는 편집 가능하게 두고, 수정 후 `grafana-dash-gen`이나 `jsonnet-bundler`로 JSON을 export하여 PR로 올리는 워크플로를 만든다.

---

### [Section E: 분산 Observability 아키텍처]

**Q17. The Three Pillars of Observability(메트릭, 로그, 트레이스)를 실제 인시던트 대응에 어떻게 연결하는가?**

**A:** 이론적 연결보다 **실제 워크플로**를 설명하는 것이 중요하다. 일반적인 인시던트 흐름은 다음과 같다.

메트릭이 이상 신호를 감지한다: Grafana 대시보드에서 `p99 latency`가 급증하거나 에러율 알림이 발생한다. 이 시점에서 "언제, 어느 서비스가 영향받는가"를 알 수 있다.

다음으로 Exemplar를 활용한 트레이스 드릴다운을 수행한다: 메트릭의 스파이크 시점 샘플에 붙어있는 `trace_id`를 클릭하면 Grafana Tempo에서 해당 시점의 느린 요청 트레이스를 볼 수 있다. 트레이스는 "어느 컴포넌트/레이어가 느린가"를 보여준다.

마지막으로 로그 상관을 수행한다: 트레이스의 `trace_id`를 Loki에서 검색하면 해당 요청의 전체 로그를 볼 수 있다. 로그는 "정확히 무슨 일이 있었는가"를 설명한다.

이 세 축을 Grafana 단일 UI에서 원클릭으로 이동하는 것이 **Grafana Correlations** 기능의 목적이다.

---

**Q18. SLO 기반 알림(multi-window, multi-burn-rate)을 설계하라. 왜 단순 임계값 알림보다 우수한가?**

**A:** 단순 임계값 알림(예: "에러율 > 1%이면 알림")의 문제는 두 가지다. **느린 감지**와 **높은 False Positive**다. 일시적 스파이크에도 알림이 발생하고, 반대로 에러율이 0.9%로 지속되어 에러 버짓을 완전히 소진해도 알림이 없다.

**번-레이트(burn rate)** 개념은 "현재 속도로 에러가 계속되면 남은 에러 버짓이 언제 소진되는가"를 계산한다. 30일 99.9% SLO의 에러 버짓은 약 43분이다. 번-레이트 1.0은 에러 버짓을 정확히 30일에 소진하는 속도이고, 번-레이트 14.4는 2일 만에 소진하는 속도다(30일/2.08일).

멀티 윈도우 접근은 단기(1h/5m) + 장기(6h/30m) 쌍으로 두 번의 알림을 설계한다. 단기 번-레이트가 높으면 즉시 호출(critical page), 장기 번-레이트가 중간이면 티켓 생성(warning)이다. 두 윈도우를 AND 조건으로 결합하면 순간 스파이크를 필터링하면서도 빠른 감지를 유지한다.

```promql
-- Critical: 에러 버짓을 빠르게 소진 중
(
  job:slo_errors:rate1h{job="api"} > (14.4 * 0.001)
  and
  job:slo_errors:rate5m{job="api"} > (14.4 * 0.001)
)
```

---

**Q19. Prometheus와 OpenTelemetry의 관계를 설명하고, OTLP를 통한 메트릭 수집 아키텍처를 설계하라.**

**A:** OpenTelemetry(OTel)는 메트릭, 로그, 트레이스를 위한 **벤더 중립적 SDK와 프로토콜**을 정의한다. Prometheus는 메트릭 생태계에서 사실상 표준이지만, OTel은 세 신호를 통합하는 상위 추상화 레이어를 제공한다.

OTel Collector를 중간 레이어로 사용하는 권장 아키텍처는 다음과 같다. 애플리케이션은 OTel SDK로 OTLP(OpenTelemetry Protocol)를 사용하여 메트릭을 OTel Collector로 보낸다. Collector는 파이프라인에서 변환, 필터링, 샘플링을 수행하고, Prometheus remote_write exporter로 Prometheus/Mimir에 전달하거나, OTLP exporter로 직접 Grafana Cloud에 보낼 수 있다.

이 아키텍처의 핵심 장점은 **애플리케이션이 백엔드로부터 완전히 분리**된다는 것이다. Prometheus를 다른 백엔드로 교체할 때 애플리케이션 코드 변경이 필요 없다. 단, OTel의 메트릭 모델은 Prometheus와 미묘하게 다르다. Gauge, Counter, Histogram 외에 Summary, ExponentialHistogram이 있으며, Prometheus로 변환 시 일부 정보가 손실될 수 있다.

---

**Q20. Prometheus Operator를 사용하지 않고 Kubernetes에 Prometheus를 배포할 때와 사용할 때의 트레이드오프는?**

**A:** **Operator 없이 배포**하면 Prometheus 설정을 ConfigMap으로 직접 관리한다. 단순하고 Kubernetes 기본 개념만 필요하지만, 새 서비스가 추가될 때마다 ConfigMap을 수동으로 업데이트하고 Prometheus를 reload해야 한다. 수백 개 서비스가 있는 환경에서는 유지보수가 불가능해진다.

**Operator를 사용하면** ServiceMonitor CRD를 통해 서비스 팀이 자신의 네임스페이스에서 독립적으로 스크래핑 설정을 추가할 수 있다. Operator가 CRD 변경을 감지하고 Prometheus 설정을 자동으로 갱신한다. 이는 **GitOps 워크플로와 완벽히 정합**하며 운영 오버헤드를 크게 줄인다.

단점은 Operator 자체를 관리해야 하며, CRD의 버전 업그레이드 시 Breaking Change가 있을 수 있다는 점이다. 또한 Operator가 생성하는 Prometheus 설정은 복잡하고 디버깅이 어려울 수 있다.

---

### [Section F: 회사별 특화 질문]

**Q21. [Amazon/AWS] Amazon Managed Service for Prometheus(AMP)의 아키텍처와 자체 호스팅 Prometheus 대비 트레이드오프는?**

**A:** AMP는 Prometheus-compatible API를 제공하지만 내부적으로 **Cortex 기반**으로 구축되어 있다. 스케일링, HA, 업그레이드를 AWS가 관리하므로 운영 부담이 거의 없다.

주요 트레이드오프는 **비용 모델**이다. AMP는 수집 샘플 수와 쿼리 수를 기준으로 과금하므로, 카디널리티가 높은 메트릭을 대량으로 수집하면 비용이 자체 호스팅보다 훨씬 높아질 수 있다. 반대로 운영 팀의 인건비를 고려하면 소규모 팀에서는 AMP가 경제적이다.

또한 AMP는 **VPC 내에서만 접근 가능**하고, `awscurl`이나 AWS SigV4 서명이 필요하다. Grafana에서 AMP를 데이터 소스로 사용하려면 Grafana가 AWS 자격 증명을 갖거나, Amazon Managed Grafana(AMG)를 함께 사용해야 한다. EKS 환경에서는 **IRSA(IAM Roles for Service Accounts)**로 Prometheus Operator의 remote_write를 AMP에 인증하는 패턴이 권장된다.

---

**Q22. [Google/GCP] Google Cloud Managed Service for Prometheus의 Global Metrics Scope 기능을 설명하고, Multi-cluster 환경에서의 활용 방법은?**

**A:** Google Cloud Managed Service for Prometheus(GMP)는 Google Monarch(내부 메트릭 시스템)를 기반으로 한다. **Global Metrics Scope**는 여러 Google Cloud 프로젝트의 메트릭을 단일 쿼리로 조회할 수 있는 기능이다.

GKE 환경에서 GMP를 사용하면 각 노드에 **Prometheus 수집기(collector)** 가 배포되어 pod annotation 기반으로 자동 스크래핑한다. 이 수집기는 메트릭을 Google Cloud Monitoring API로 전송하며, 보존 기간은 최대 24개월이다.

멀티 클러스터 활용 시 모든 클러스터의 메트릭에 `cluster` 레이블이 자동 추가되고, Cloud Monitoring UI 또는 Grafana의 Cloud Monitoring 데이터 소스를 통해 교차 클러스터 쿼리가 가능하다. 단, PromQL 외에 **Monarch의 MQL(Monitoring Query Language)**도 지원하므로 일부 고급 분석은 MQL이 더 강력하다.

---

**Q23. [Microsoft/Azure] Azure Monitor와 Prometheus의 통합 아키텍처를 설명하라.**

**A:** Azure Monitor Managed Service for Prometheus는 **Azure Monitor Workspace**를 저장소로 사용한다. AKS 클러스터에 **Azure Monitor Agent**를 배포하면 Kubernetes 메트릭을 자동 수집하고 Azure Monitor Workspace에 저장한다.

통합 아키텍처의 핵심은 **Grafana Azure Managed Instance**와의 연동이다. Azure Monitor Workspace를 데이터 소스로 추가하면 표준 PromQL로 쿼리할 수 있다. Prometheus alert rule은 Azure Monitor에서 직접 관리하거나, Alertmanager 통합을 통해 기존 PagerDuty/OpsGenie로 라우팅할 수 있다.

멀티 클러스터 환경에서는 여러 AKS 클러스터를 단일 Azure Monitor Workspace에 연결하여 중앙 집계할 수 있다. Azure Policy를 통해 모든 AKS 클러스터에 자동으로 모니터링 에이전트를 배포하는 **정책 기반 거버넌스**가 기업 환경에서 특히 유용하다.

---

**Q24. [Apple/NVIDIA] GPU 집약적 워크로드(ML 훈련, 추론)에 대한 Prometheus 메트릭 설계와 DCGM Exporter 활용 방법을 설명하라.**

**A:** GPU 워크로드 모니터링은 NVIDIA의 **DCGM(Data Center GPU Manager) Exporter**를 사용한다. DCGM Exporter는 DaemonSet으로 배포되어 각 노드의 GPU 메트릭을 수집한다.

핵심 메트릭은 다음과 같다. `DCGM_FI_DEV_GPU_UTIL`(GPU 연산 사용률), `DCGM_FI_DEV_MEM_COPY_UTIL`(메모리 버스 사용률), `DCGM_FI_DEV_FB_USED` / `DCGM_FI_DEV_FB_FREE`(프레임버퍼 메모리 사용량), `DCGM_FI_DEV_POWER_USAGE`(전력 소비), `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL`(NVLink 대역폭)이다.

ML 훈련에서 중요한 **GPU 활용도 저하 패턴**을 감지하는 알림을 설계하면: GPU 사용률이 30분 이상 40% 이하로 유지되면 데이터 로딩 병목(CPU-GPU 전송 병목) 또는 학습 루프 버그를 의심한다.

추론 서비스에서는 **배치 사이즈 효율성**과 **Model Load Time**을 메트릭으로 노출하고, p99 추론 레이턴시를 SLO로 관리한다. NVIDIA Triton Inference Server는 내장 Prometheus 메트릭 엔드포인트를 제공한다.

---

**Q25. [Tesla] 에지 컴퓨팅 환경(IoT/자동차)에서 Prometheus 메트릭 수집의 아키텍처 도전과 해결책은?**

**A:** 에지 환경의 핵심 도전은 **간헐적 연결, 제한된 리소스, 보안**이다.

간헐적 연결 문제는 **Prometheus Agent Mode + Remote Write**로 해결한다. 로컬 저장 없이 WAL만 유지하여 메모리/디스크를 최소화하고, 연결 복구 시 WAL의 데이터를 자동으로 전송한다. 추가로 **로컬 VictoriaMetrics Agent**가 경량화면에서 우수하다.

보안은 **mTLS(mutual TLS)** 인증으로 에지 디바이스와 중앙 집계 서버 간 통신을 보호한다. 각 디바이스는 고유한 클라이언트 인증서를 가지며, 인증서 갱신은 **cert-manager**나 **Vault Agent**로 자동화한다.

**데이터 우선순위**는 에지에서 집계 및 필터링을 수행하여 중요 메트릭만 전송하는 계층적 수집 구조를 사용한다. 예를 들어 차량 내 이상 감지(anomaly detection)는 에지에서 실행하고, 결과(이벤트)만 클라우드로 전송하여 대역폭을 최소화한다.

---

**Q26. Prometheus의 보안 강화(Security Hardening) 방법을 설명하라. 특히 멀티 테넌트 환경에서 메트릭 분리를 어떻게 구현하는가?**

**A:** Prometheus 자체는 **인증(authentication) 기능이 내장되어 있지 않다**. 이는 설계 철학("단순성 우선")이지만 운영 환경에서는 반드시 보완해야 한다.

첫째, **TLS와 Basic Auth**는 `web.config.file` 설정으로 활성화할 수 있다(Prometheus 2.24+). `/metrics` 엔드포인트에 접근하려면 인증이 필요하도록 설정한다.

둘째, Kubernetes 환경에서는 **NetworkPolicy**로 Prometheus 서버만 exporter의 `/metrics` 포트에 접근할 수 있도록 제한한다.

셋째, **멀티 테넌트 분리**는 Mimir의 테넌트 격리 또는 별도의 Prometheus 인스턴스를 팀별로 배포(Operator를 통해)하는 방식을 사용한다. 팀 A의 알림 규칙이 팀 B의 메트릭을 조회할 수 없도록 AlertmanagerConfig의 namespace 범위를 제한한다.

넷째, **레이블 기반 데이터 필터링**은 지원하지 않으므로 민감한 메트릭(PII 포함 가능 레이블 등)은 수집 단계에서 relabeling으로 제거해야 한다.

---

**Q27. Prometheus 쿼리 성능이 저하될 때(slow queries) 진단 방법과 최적화 전략을 체계적으로 설명하라.**

**A:** 진단의 첫 단계는 **Prometheus Query Log**를 활성화하는 것이다. `--query.timeout`과 `--query.max-samples` 플래그로 쿼리 제한을 설정하고, 느린 쿼리를 로그로 수집한다.

Prometheus 2.x에서 `/api/v1/status/tsdb`는 TSDB의 Head Block 시계열 수, 청크 수, 메모리 사용량을 반환한다. **`/api/v1/query` 응답의 `stats` 필드**에는 쿼리가 평가한 시계열 수, 샘플 수, 처리 시간이 포함되어 있어 쿼리 복잡도를 정량적으로 측정할 수 있다.

최적화 전략의 우선순위는 다음과 같다. 첫째, **Recording Rule로 고비용 쿼리 사전 계산**. 둘째, 쿼리의 **선택자(selector)를 구체화**하여 불필요한 시계열 스캔 최소화(`job`, `instance` 레이블 필터 추가). 셋째, 대시보드의 **쿼리 병렬화 제한**을 확인한다(Grafana `maxDataPoints` 설정). 넷째, **시간 범위 제한**을 추가하여 불필요한 과거 데이터 스캔 방지. 다섯째, Thanos/Mimir의 **쿼리 프론트엔드 캐시(Query Result Cache)**를 활성화한다.

---

**Q28. `kube_state_metrics`와 `node_exporter`의 역할 차이와, 각각의 핵심 메트릭을 설명하라.**

**A:** 두 exporter는 서로 다른 계층을 관찰한다. `node_exporter`는 **노드(물리/가상 머신) 수준**의 시스템 메트릭을 수집한다. CPU, 메모리, 디스크 I/O, 네트워크 통계, 파일시스템 사용량 등 OS 레벨 정보다. Kubernetes를 모르고 순수 Linux 시스템 메트릭을 노출한다.

`kube_state_metrics`는 **Kubernetes 오브젝트의 상태**를 메트릭으로 변환한다. Pod의 phase(Running/Pending/Failed), Deployment의 desired vs available 레플리카 수, Node의 condition, PVC의 바운드 상태 등 Kubernetes API에 있는 정보를 Prometheus가 스크래핑할 수 있는 형태로 노출한다.

실무 활용 예시: "OOM Kill이 발생했는가" → `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}`. "노드 디스크가 가득 찼는가" → `node_filesystem_avail_bytes / node_filesystem_size_bytes`. 두 메트릭을 결합하면 "디스크 부족으로 Pod가 실패하는 패턴"처럼 OS와 Kubernetes 이벤트를 상관 관계로 분석할 수 있다.

---

**Q29. Grafana OnCall과 PagerDuty를 통합할 때 알림 라우팅 설계 전략은? Escalation Policy를 어떻게 설계하는가?**

**A:** 좋은 On-Call 설계의 핵심은 **노이즈 대 중요도의 균형**이다. 모든 알림이 즉시 호출(page)되면 팀은 알림 피로(alert fatigue)에 빠진다.

**Severity 레이블 기반 라우팅**을 Alertmanager에 구현한다. `severity=critical`은 PagerDuty Critical로 즉시 호출, `severity=warning`은 Slack 채널로만, `severity=info`는 로그로만 남긴다. Alertmanager의 `routes` 트리에서 `severity` 레이블을 매처(matcher)로 사용한다.

**Escalation Policy:** P1(서비스 전체 중단)은 On-Call 엔지니어 → 5분 무응답 시 팀 리드 → 15분 무응답 시 관리자로 에스컬레이션한다. P2(성능 저하)는 On-Call만, P3은 자동 티켓 생성으로 설계한다.

**Inhibition Rule**으로 중복 알림을 억제하는 것이 On-Call 품질의 핵심이다. 클러스터 다운 알림이 발생하면 해당 클러스터의 모든 서비스 알림을 억제하여 On-Call 엔지니어가 근본 원인에 집중하게 한다.

---

**Q30. Prometheus에서 메트릭 데이터 백필(backfill)이 필요한 상황과 `promtool tsdb create-blocks-from` 명령의 활용 방법을 설명하라.**

**A:** 백필이 필요한 시나리오는 세 가지다. 첫째, **Prometheus 설정 오류로 일정 기간 데이터가 수집되지 않은 경우**. 둘째, **레거시 시스템에서 Prometheus로 마이그레이션하면서 과거 데이터를 이관**해야 하는 경우. 셋째, **Recording Rule을 새로 만들었는데 과거 데이터도 필요한 경우**다.

`promtool tsdb create-blocks-from openmetrics` 명령은 OpenMetrics 형식의 파일을 읽어 Prometheus TSDB Block을 직접 생성한다. 생성된 Block을 Prometheus의 `--storage.tsdb.path` 디렉토리에 복사하면 Prometheus가 자동으로 인식한다(재시작 불필요, 단 해당 시간 범위와 겹치는 기존 Block이 없어야 함).

중요한 제약 사항이 있다. 백필 데이터는 **Prometheus의 보존 기간 설정에 의해 오래된 것부터 삭제**된다. 또한 Remote Write는 백필 데이터를 소급 전송하지 않으므로, Thanos/Mimir에 과거 데이터를 이관하려면 별도로 해당 Block을 오브젝트 스토리지에 직접 업로드해야 한다.

---

**Q31. OpenMetrics vs Prometheus exposition format의 차이와, 마이그레이션 고려사항은?**

**A:** Prometheus Exposition Format은 Prometheus가 설계한 독자적 텍스트 포맷이다. **OpenMetrics**는 이를 표준화하여 IETF RFC로 만든 포맷으로, 다음 확장 사항을 포함한다.

**Exemplar 지원**이 OpenMetrics의 가장 중요한 차이다. OpenMetrics의 Histogram Counter 라인에 `# {trace_id="abc123"} 0.5 1234567890.123` 형태로 Exemplar를 첨부할 수 있다. 이는 앞서 언급한 메트릭-트레이스 상관 관계의 기반이다.

**Unit 표준화:** OpenMetrics는 메트릭 이름에 단위 접미사를 강제(예: `_bytes`, `_seconds`)하며, `# UNIT` 메타데이터를 추가로 지원한다. 이는 Grafana Units 자동 설정과 연동된다.

마이그레이션 시 주의점은 **Content-Type 협상**이다. Prometheus 2.x는 Accept 헤더로 `application/openmetrics-text`를 요청하면 OpenMetrics 형식으로 응답하는 exporter와 자동으로 협상한다. 구버전 exporter는 이를 지원하지 않으므로, 갑작스러운 format 변경으로 파싱 오류가 발생할 수 있다.

---

**Q32. Alertmanager의 `group_wait`, `group_interval`, `repeat_interval`의 상호작용을 실제 운영 시나리오로 설명하라.**

**A:** 세 파라미터는 알림의 **타이밍 제어 3단계**를 담당한다.

`group_wait`(예: 30초)는 새 알림 그룹이 처음 생성된 후 **첫 번째 알림을 보내기 전 대기 시간**이다. 이 시간 동안 같은 그룹의 다른 알림을 수집하여 배치로 보낸다. 너무 짧으면 개별 알림이 쏟아지고, 너무 길면 첫 통보가 지연된다.

`group_interval`(예: 5분)는 이미 알림이 발송된 그룹에 **새로운 알림이 추가될 때 업데이트 알림을 보내는 간격**이다. 진행 중인 장애에서 영향 범위가 확대될 때 얼마나 빨리 재통보할지를 제어한다.

`repeat_interval`(예: 3시간)는 **Firing 상태가 해결되지 않은 알림을 반복 전송하는 간격**이다. On-Call 엔지니어가 알림을 놓쳤을 때 재인지(re-notify)를 위한 것이다.

시나리오: 오전 2시에 5개 Pod가 동시에 크래시하면 `group_wait` 30초 후 "5개 Pod 크래시" 단일 메시지가 온다(배치). 10분 후 3개 Pod가 추가 크래시하면 `group_interval` 5분 후 업데이트 메시지가 온다. 문제가 해결되지 않으면 `repeat_interval` 3시간 후 다시 알림이 온다.

---

**Q33. Prometheus의 `honor_labels`와 `honor_timestamps` 설정의 의미와 사용 시 주의점은?**

**A:** `honor_labels: true`는 스크래핑 대상이 노출하는 레이블이 Prometheus가 추가하는 레이블과 충돌할 때 **대상의 레이블을 우선**시한다. 예를 들어 Pushgateway는 원본 잡(job)의 레이블을 보존해야 하므로 반드시 `honor_labels: true`를 사용한다. 그렇지 않으면 Pushgateway의 `job` 레이블이 Prometheus가 스크래핑 설정에서 지정한 `job` 레이블로 덮어씌워진다.

그러나 **일반 exporter에는 사용하지 말아야** 한다. 신뢰할 수 없는 exporter가 자신의 메트릭에 임의의 레이블(예: `cluster="production"`)을 추가하면, 이것이 Prometheus 전체 데이터를 오염시킬 수 있다.

`honor_timestamps: true`는 스크래핑된 메트릭의 타임스탬프를 그대로 사용한다. 이는 Pushgateway나 연합(Federation) 시나리오에서 원본 수집 시점을 보존할 때 유용하다. 단, 미래 또는 과거 타임스탬프를 가진 메트릭은 TSDB의 **out-of-order 거부 정책**에 의해 기록되지 않을 수 있다(기본 2시간 허용 범위).

---

## Part V. 추가 고급 토픽 참고 자료

이 문서에서 다루지 않은 고급 토픽으로 추가 학습을 권장하는 영역은 다음과 같다.

**Native Histograms(Prometheus 2.40+):** 기존 Histogram의 버킷 설계 제약을 근본적으로 해결하는 새 데이터 타입이다. DDSketch 알고리즘 기반으로 임의 분위수를 높은 정확도로 계산한다.

**Prometheus Agent Mode(2.32+):** 에지/원격 환경에서의 경량 수집기로, WAL만 유지하고 Remote Write로만 동작한다.

**eBPF 기반 자동 계측:** Grafana Beyla는 eBPF를 사용하여 애플리케이션 코드 수정 없이 RED 메트릭과 트레이스를 자동 수집한다. Kubernetes 환경에서 특히 강력하다.

**Prometheus OTLP Native Ingestion(2.47+):** OTel Collector 없이 Prometheus가 직접 OTLP 형식의 메트릭을 수신할 수 있다.

**Grafana Faro:** 프론트엔드(브라우저) 관찰 가능성을 위한 실사용자 모니터링(RUM) SDK로, 백엔드 메트릭과 연계된 E2E 관찰을 가능하게 한다.
