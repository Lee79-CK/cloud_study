# Microservices Architecture & Event-Driven Architecture
## Senior Cloud Engineer를 위한 박사급 심층 가이드 + 빅테크 면접 Q&A

---

# PART 1: Microservices Architecture (MSA) — 심층 이론

## 1.1 MSA의 철학적 토대: Conway's Law와 역-Conway 전략

MSA를 단순히 "서비스를 잘게 나누는 것"으로 이해하는 것은 표면적 인식에 불과하다. MSA의 본질은 **소프트웨어 구조가 조직 구조를 반영한다**는 Conway's Law(1967)를 역으로 활용하는 것에 있다. 즉, 원하는 아키텍처를 먼저 설계하고, 그에 맞게 팀 구조를 재편하는 **역-Conway 전략(Inverse Conway Maneuver)**이 MSA 도입의 실질적 출발점이 된다.

Amazon의 "Two-Pizza Team" 원칙은 이 개념의 가장 대표적인 산업 적용 사례다. 피자 두 판으로 먹을 수 있는 규모(6~10명)의 팀이 하나의 서비스를 완전히 소유(full ownership)하며, 설계·개발·배포·운영·장애 대응을 모두 담당한다. 이는 단순한 팀 운영 방식이 아니라, **인지 부하(cognitive load)를 팀 단위로 분산**시켜 복잡성을 관리하는 구조적 메커니즘이다.

## 1.2 서비스 분해 전략: DDD와 Bounded Context

마이크로서비스의 경계를 정의하는 가장 학문적으로 정교한 방법론은 **Domain-Driven Design(DDD)**의 **Bounded Context**다. 단순히 기능별 분리(Function Decomposition)나 데이터베이스 테이블 단위 분리는 수많은 실패 사례에서 보듯 "분산된 모놀리스(Distributed Monolith)"를 만들 뿐이다.

Bounded Context는 특정 도메인 모델이 일관된 의미를 유지하는 언어적·개념적 경계다. 예를 들어 "Customer"라는 개념은 주문 도메인에서는 배송 주소를 가진 구매자지만, 마케팅 도메인에서는 세그먼트와 행동 이력을 가진 타깃이다. 이 개념적 불일치를 무시하고 하나의 `Customer` 서비스로 통합하면, 모든 도메인의 요구사항이 하나의 서비스에 집중되는 **God Service 안티패턴**이 발생한다.

실천적으로는 **Event Storming** 워크숍을 통해 도메인 전문가와 개발자가 협력하여 도메인 이벤트를 식별하고, 이를 기반으로 Aggregate → Bounded Context → 서비스 경계를 도출한다. Alberto Brandolini가 고안한 이 기법은 현재 Netflix, Uber 등 대형 MSA 구현에서 필수 아키텍처 설계 도구로 자리잡았다.

## 1.3 데이터 격리와 Saga 패턴: 분산 트랜잭션의 딜레마

MSA의 핵심 원칙 중 하나는 **"서비스당 하나의 데이터베이스(Database-per-Service)"**다. 이는 서비스 간 결합도를 데이터 계층에서 완전히 분리하기 위함이지만, 동시에 **분산 트랜잭션 문제**를 야기한다.

전통적인 2PC(Two-Phase Commit)는 분산 환경에서 코디네이터 장애 시 전체 시스템이 블로킹되는 치명적 약점과, 각 참여 노드가 락을 유지해야 하는 성능 병목 문제가 있다. CAP 정리에 따르면 네트워크 파티션이 발생하는 분산 시스템에서 CA를 동시에 보장할 수 없으므로, 대부분의 MSA는 **최종 일관성(Eventual Consistency)**을 택하고 그 구현 메커니즘으로 **Saga 패턴**을 채택한다.

Saga는 분산 트랜잭션을 일련의 로컬 트랜잭션 체인으로 분해하고, 각 단계의 실패 시 **보상 트랜잭션(Compensating Transaction)**을 역순으로 실행하여 비즈니스 무결성을 유지한다. 구현 방식은 두 가지로 나뉜다.

**Choreography 방식**에서는 각 서비스가 이벤트를 발행하고, 다음 서비스가 해당 이벤트를 구독하여 자신의 로컬 트랜잭션을 실행한다. 중앙 코디네이터가 없어 단일 장애점(SPOF)이 없지만, 트랜잭션 흐름이 서비스 전반에 분산되어 추적이 어렵고, 서비스 간 순환 의존성 위험이 있다.

**Orchestration 방식**에서는 Saga Orchestrator가 전체 흐름을 명시적으로 지시한다. 워크플로우 가시성이 높고 디버깅이 용이하지만, Orchestrator 자체가 복잡해지고 SPOF가 될 수 있다. AWS Step Functions, Netflix Conductor, Temporal 같은 워크플로우 엔진이 이 역할을 담당한다.

실무에서는 단순한 선형 흐름에는 Choreography를, 복잡한 비즈니스 로직과 예외 처리가 필요한 장기 트랜잭션에는 Orchestration을 선택하는 **하이브리드 전략**이 권장된다.

## 1.4 서비스 메시(Service Mesh): 인프라 계층의 횡단 관심사 처리

MSA 규모가 수십 개에서 수백 개 서비스로 확장될 때, 서비스 간 통신에서 공통으로 필요한 기능—mTLS, 서킷 브레이커, 로드밸런싱, 분산 추적, 재시도 로직—을 각 서비스가 개별 구현하면 막대한 중복과 운영 복잡성이 발생한다.

**서비스 메시(Service Mesh)**는 이러한 횡단 관심사(cross-cutting concerns)를 **데이터 플레인(Data Plane)**의 사이드카 프록시(Envoy, Linkerd-proxy)로 위임하고, **컨트롤 플레인(Control Plane)**(Istio의 istiod, Consul)이 이를 중앙에서 정책으로 제어하는 아키텍처다.

특히 **mTLS 자동화**는 Zero-Trust 네트워크 모델 구현의 핵심이다. 서비스 메시가 없으면 각 서비스 쌍마다 인증서 관리와 TLS 핸드셰이크 로직을 구현해야 하지만, 서비스 메시에서는 인증서 발급과 갱신이 컨트롤 플레인에 의해 자동화되고, 모든 서비스 간 통신이 투명하게 암호화된다.

그러나 서비스 메시는 도입 비용이 상당하다. Sidecar 주입으로 인한 레이턴시 오버헤드(일반적으로 1~5ms, p99에서는 더 높을 수 있음), 리소스 사용량 증가, 컨트롤 플레인의 운영 복잡성이 트레이드오프로 존재한다. 이 때문에 Cilium의 **eBPF 기반 서비스 메시**처럼 사이드카 없이 커널 레벨에서 네트워킹을 처리하는 **Sidecar-less 아키텍처**가 주목받고 있다.

## 1.5 API Gateway와 BFF 패턴: 외부 인터페이스의 계층화

외부 클라이언트와 내부 마이크로서비스 사이에는 반드시 **API Gateway**가 위치해야 한다. API Gateway는 인증/인가, SSL 종료, 요청 라우팅, 속도 제한(Rate Limiting), 요청/응답 변환 등을 처리한다. Netflix의 Zuul, AWS API Gateway, Kong, Apigee가 대표적이다.

그러나 클라이언트 유형이 다양해질수록(모바일 앱, 웹 SPA, IoT 디바이스, 파트너 API) 단일 게이트웨이는 모든 클라이언트의 요구를 충족하기 위해 점점 복잡해지는 "Edge 모놀리스" 문제에 직면한다. **BFF(Backend for Frontend) 패턴**은 클라이언트 유형별로 전용 게이트웨이 레이어를 두어, 각 프론트엔드의 요구에 최적화된 API를 제공하는 방식으로 이 문제를 해결한다.

SoundCloud에서 처음 도입하고 Netflix가 대규모로 적용한 BFF 패턴은, 특히 **GraphQL Federation**과 결합될 때 강력해진다. 각 마이크로서비스가 자신의 GraphQL 스키마를 소유하고, Federation Gateway가 이를 단일 슈퍼그래프(Supergraph)로 조합하여 클라이언트에 제공함으로써, REST의 over-fetching/under-fetching 문제와 BFF의 클라이언트 맞춤화를 동시에 해결한다.

---

# PART 2: Event-Driven Architecture (EDA) — 심층 이론

## 2.1 EDA의 이론적 근거: 시간적 결합의 해체

EDA는 단순히 메시지 큐를 사용하는 것이 아니다. 그 핵심 철학은 **시간적 결합(Temporal Coupling)의 제거**다. 동기 REST 호출에서 서비스 A가 서비스 B를 호출할 때, A는 B의 가용성, 응답 시간, 처리 능력에 완전히 의존한다. B가 느리면 A도 느려지고, B가 다운되면 A도 실패한다. 이는 두 서비스가 시간 축에서 결합되어 있음을 의미한다.

이벤트 기반 아키텍처에서 서비스 A는 "무언가가 일어났다(something happened)"는 사실을 이벤트로 발행하고, B가 그것을 언제 처리할지에 대한 관심을 포기한다. 이 **화자 중심(Fire-and-Forget)** 모델은 두 서비스를 시간적으로 완전히 분리하여 독립적인 확장과 장애 격리를 가능하게 한다.

Greg Young이 정립한 **이벤트의 의미론적 분류**는 EDA 설계에서 핵심적이다.

- **Domain Event**: 비즈니스 도메인에서 의미있는 변화. "OrderPlaced", "PaymentProcessed" — 과거에 일어난 사실을 기술하는 불변(immutable) 레코드.
- **Command**: 특정 행동을 요청. "PlaceOrder", "ProcessPayment" — 실패할 수 있으며 단일 수신자를 대상으로 함.
- **Query**: 상태 조회 요청. CQRS의 Read side에서 처리.

이 구분이 중요한 이유는, 이벤트를 Command처럼 설계하면—즉 이벤트에 "다음에 무엇을 해야 한다"는 지시가 포함되면—EDA의 결합도 이점이 사라지기 때문이다. "OrderPlacedAndInventoryShouldBeDeducted"는 이벤트가 아니라 커맨드다.

## 2.2 이벤트 스트리밍 vs 메시지 큐: 아키텍처 선택의 기준

**메시지 큐(RabbitMQ, AWS SQS, Azure Service Bus)**와 **이벤트 스트리밍 플랫폼(Apache Kafka, AWS Kinesis, Google Pub/Sub)**은 표면적으로 유사해 보이지만 근본적으로 다른 계산 모델을 갖는다.

메시지 큐는 **작업 큐(Work Queue)** 모델로, 메시지는 소비되면 삭제된다. 각 메시지는 정확히 하나의 소비자에게 전달(보통)되며, 소비자의 현재 처리 능력에 맞게 작업을 분산하는 것이 목적이다.

이벤트 스트리밍 플랫폼은 **지속 로그(Durable Log)** 모델로, 이벤트는 소비 후에도 보존(retention period 내)되며 다수의 독립적인 소비자 그룹이 각자의 오프셋으로 이벤트를 처리할 수 있다. Kafka의 파티션 로그는 시간 자체를 데이터 구조로 만들어, 시스템의 전체 상태 변경 이력을 재연(replay)하는 것이 가능하다.

이 차이는 단순한 기술 선택을 넘어 **이벤트 소싱(Event Sourcing)** 패턴의 실현 가능성을 결정한다. Kafka를 **시스템 of Record(진실의 원천)**으로 사용할 때, 모든 서비스의 현재 상태는 이벤트 스트림의 현재 프로젝션(projection)이 된다. 이는 어떤 시점으로도 상태를 되돌릴 수 있는 **시간 여행(Time-Travel Debugging)** 능력과, 새로운 서비스가 히스토리 전체를 처음부터 재처리하여 자신만의 Read Model을 구축하는 능력을 부여한다.

## 2.3 CQRS (Command Query Responsibility Segregation): 읽기와 쓰기의 최적화 분리

**CQRS**는 단일 모델이 읽기와 쓰기 요구사항을 동시에 만족시키려 할 때 발생하는 임피던스 불일치(impedance mismatch)를 해결하기 위해 읽기 모델과 쓰기 모델을 완전히 분리하는 패턴이다.

쓰기 측(Command Side)은 비즈니스 규칙 검증과 도메인 무결성 보장에 최적화된 정규화된(normalized) 모델을 사용한다. 읽기 측(Query Side)은 특정 UI나 리포트에 최적화된 비정규화된(denormalized) 읽기 모델(Read Model 또는 Projection)을 이벤트 스트림으로부터 유지한다. 이 읽기 모델은 용도에 따라 Elasticsearch, Redis, PostgreSQL, Cassandra 등 서로 다른 저장소를 사용할 수 있다.

CQRS를 이벤트 소싱과 결합하면—이른바 CQRS+ES 패턴—쓰기 측은 현재 상태가 아닌 **이벤트 시퀀스**를 저장하고, 읽기 모델은 이 이벤트들을 비동기적으로 처리하여 자신의 프로젝션을 갱신한다. 이는 읽기 성능과 쓰기 성능을 완전히 독립적으로 스케일링하는 것을 가능하게 하며, 읽기 모델의 버그 수정은 이벤트 스트림을 재처리(rehydration)하여 손쉽게 반영할 수 있다.

그러나 CQRS는 도입 복잡성이 상당하다. 최종 일관성으로 인한 읽기 스탈(stale read) 문제, 멀티 모델 동기화 로직, 이벤트 스키마 진화(schema evolution) 문제 등을 감당할 준비가 된 팀과 도메인에서만 도입이 권장된다.

## 2.4 이벤트 스키마 진화와 Schema Registry

이벤트 기반 시스템에서 가장 과소평가되는 운영 과제는 **이벤트 스키마의 하위/상위 호환성 관리**다. 서비스 A가 `OrderPlaced` 이벤트 스키마에 새 필드를 추가할 때, 이미 배포된 수십 개의 소비자 서비스가 즉시 다운될 수 있다.

**Confluent Schema Registry**와 **Apache Avro/Protobuf**의 조합은 이 문제에 대한 산업 표준 솔루션이다. Schema Registry는 모든 이벤트 스키마의 버전화된 저장소 역할을 하며, 생산자가 새 스키마를 등록할 때 **호환성 검사(BACKWARD, FORWARD, FULL)**를 강제한다.

- **BACKWARD 호환성**: 새 스키마로 구버전 이벤트를 읽을 수 있음. 소비자를 먼저 업그레이드하는 전략.
- **FORWARD 호환성**: 구 스키마로 새 이벤트를 읽을 수 있음(새 필드 무시). 생산자를 먼저 업그레이드하는 전략.
- **FULL 호환성**: 양방향 호환. 가장 안전하지만 가장 제약이 많음.

실무에서는 **Expand and Contract 패턴**(일명 Parallel Change 패턴)이 널리 사용된다. 1단계에서 새 필드를 추가하고 구 필드를 deprecated로 표시(expand), 2단계에서 모든 소비자가 새 필드로 마이그레이션 완료 후 구 필드를 제거(contract)하는 두 단계 배포 전략이다.

## 2.5 Outbox 패턴: 이중 쓰기 문제의 원자적 해결

MSA와 EDA의 접점에서 발생하는 가장 치명적인 일관성 문제는 **이중 쓰기(Dual Write)**다. 서비스가 데이터베이스에 상태를 저장하는 것과 이벤트 브로커에 이벤트를 발행하는 것을 하나의 원자적 연산으로 처리할 방법이 없기 때문에, 데이터베이스 커밋은 성공했는데 이벤트 발행이 실패하거나, 이벤트는 발행됐는데 DB 롤백이 발생하는 상황이 생길 수 있다.

**Transactional Outbox 패턴**은 이 문제를 우아하게 해결한다. 서비스는 이벤트를 직접 브로커에 발행하는 대신, **같은 데이터베이스 트랜잭션 내에** `outbox` 테이블에 이벤트를 레코드로 삽입한다. 별도의 **Message Relay 프로세스**(Debezium, Polling Publisher)가 이 테이블을 모니터링하여 이벤트를 브로커로 발행한 후 레코드를 삭제한다. 이로써 DB 상태 변경과 이벤트 발행이 **원자적으로 결합**된다.

**CDC(Change Data Capture)** 기반의 Outbox 구현(Debezium + Kafka)은 DB 트랜잭션 로그(WAL)를 직접 읽기 때문에 Polling 방식보다 레이턴시가 낮고 DB 부하가 적다. 이 패턴은 현재 Uber, Airbnb, LinkedIn 등 대형 플랫폼에서 MSA 이벤트 발행의 사실상 표준으로 자리잡았다.

## 2.6 이벤트 스트리밍의 고급 처리: 스트림 프로세싱과 CEP

단순한 이벤트 발행/구독을 넘어, **스트림 프로세싱(Stream Processing)**은 실시간으로 흐르는 이벤트 스트림에 변환, 집계, 조인을 적용하여 새로운 이벤트나 상태를 생성한다.

**Apache Flink**는 현재 스트림 프로세싱의 아키텍처 표준으로, 진정한 스트리밍(not micro-batch), 이벤트 시간(event time) 처리, 장애 복구를 위한 체크포인트, 정확히 한 번(Exactly-Once) 처리 시맨틱을 지원한다. Alibaba, Uber, Netflix가 실시간 분석, 이상 감지, 특징 엔지니어링에 Flink를 사용한다.

**CEP(Complex Event Processing)**는 단일 이벤트가 아닌 **이벤트 패턴**을 감지하는 기술이다. "5분 내에 동일 사용자의 로그인이 10회 실패한 경우", "A 이벤트 발생 후 B 이벤트 없이 30초 내에 C 이벤트 발생" 같은 시간적·구조적 패턴을 실시간으로 감지하여 사기 탐지, 보안 모니터링, 실시간 추천 등에 활용된다.

**Kafka Streams와 ksqlDB**는 Kafka 에코시스템 내에서 SQL 유사 문법으로 스트림 프로세싱을 구현할 수 있게 하는 경량 솔루션으로, 별도 클러스터 없이 애플리케이션 내에서 스트리밍 로직을 실행할 수 있어 운영 복잡성을 크게 낮춘다.

---

# PART 3: 대규모 시스템 고급 패턴

## 3.1 Strangler Fig 패턴: 모놀리스의 점진적 MSA 전환

실제 엔터프라이즈 환경에서 MSA 전환은 대부분 기존 모놀리스를 유지하면서 점진적으로 수행된다. **Strangler Fig 패턴**(Martin Fowler)은 열대 무화과나무가 숙주 나무를 조금씩 감싸며 결국 대체하듯, 모놀리스의 기능을 하나씩 마이크로서비스로 추출하고 트래픽을 점진적으로 이전하는 전략이다.

API Gateway를 Strangler Facade로 사용하여 특정 엔드포인트의 트래픽을 새 마이크로서비스로 라우팅하고, 기존 모놀리스 코드를 점차 제거한다. 이 과정에서 **Anticorruption Layer(ACL)**는 구 모놀리스의 도메인 모델이 새 서비스의 클린한 도메인 모델을 오염시키지 않도록 번역 계층 역할을 한다.

## 3.2 Circuit Breaker와 Resilience 패턴

분산 시스템에서 하나의 서비스 장애가 전체 시스템으로 전파되는 **Cascading Failure**는 MSA의 가장 치명적인 위험이다. **Circuit Breaker 패턴**(Michael Nygard의 Release It! 에서 정립)은 전기 회로 차단기처럼 동작하여, 다운스트림 서비스의 장애 임계치 도달 시 즉시 요청을 차단하고 빠른 실패(Fail Fast)를 반환한다.

상태 기계로 동작하는 Circuit Breaker는 Closed(정상), Open(차단), Half-Open(탐색) 세 상태를 갖는다. Netflix의 Hystrix가 이 패턴을 대중화했고, 현재는 Resilience4j, AWS SDK의 내장 재시도 로직, 서비스 메시의 Envoy 프록시에서 네이티브로 지원된다.

**Bulkhead 패턴**은 선박의 격벽처럼 서비스 내의 리소스(스레드풀, 커넥션풀)를 격리하여, 하나의 다운스트림 의존성 장애가 전체 서비스 리소스를 소진시키지 않도록 한다.

## 3.3 Observable MSA: 분산 추적과 OpenTelemetry

수백 개 서비스로 구성된 MSA에서 하나의 요청이 수십 개 서비스를 거칠 때, 장애나 성능 저하의 근본 원인을 찾는 것은 로그만으로는 불가능하다.

**OpenTelemetry(OTel)**는 메트릭, 로그, 트레이스라는 **관찰성의 세 기둥(Three Pillars of Observability)**을 위한 벤더 중립적 표준으로, CNCF가 관리하는 산업 표준이다. Trace ID와 Span ID를 서비스 경계를 넘어 전파하여, Jaeger나 Tempo 같은 분산 추적 백엔드에서 전체 요청 경로를 시각화한다.

**SLI(Service Level Indicator), SLO(Service Level Objective), SLA(Service Level Agreement)**의 계층 구조와, **Error Budget**에 기반한 SRE 문화는 MSA에서 안정성과 속도 사이의 균형을 측정 가능하게 만드는 프레임워크다.

---

# PART 4: 빅테크 면접 Q&A 35선

> 각 질문은 회사별로 실제 면접에서 출제되는 스타일과 수준을 반영하였으며, 답변은 인터뷰어가 기대하는 핵심 포인트와 심화 내용을 포함한다.

---

## 🟠 Amazon (AWS) — 시스템 설계 중심, Leadership Principle 연계

---

### Q1. Amazon에서 수백만 TPS를 처리하는 주문 처리 시스템을 MSA로 설계하라. 특히 재고 차감, 결제, 배송 생성이 포함된 분산 트랜잭션을 어떻게 일관성 있게 처리하겠는가?

**핵심 답변:**

먼저 이 시나리오는 전형적인 **Long-Running Transaction** 문제로, 2PC는 성능과 가용성 이유로 배제해야 한다. 대신 **Saga 패턴의 Orchestration 방식**을 AWS Step Functions로 구현하는 것이 Amazon 환경에서 최적이다.

설계는 다음과 같다. 주문 서비스가 `OrderPlaced` 이벤트를 Step Functions 워크플로우 실행으로 전환한다. 1단계에서 재고 서비스가 낙관적 락(Optimistic Locking)을 사용한 재고 예약(soft reservation)을 수행한다. 2단계에서 결제 서비스가 Idempotency Key를 사용한 결제 처리를 실행한다. 3단계에서 배송 서비스가 배송 생성을 한다. 각 단계의 실패 시 보상 트랜잭션(예약 해제, 결제 환불)이 역순으로 실행된다.

이벤트 발행 신뢰성을 위해 **Transactional Outbox + DynamoDB Streams + Lambda** 조합을 사용한다. 재고 서비스는 재고 테이블과 같은 DynamoDB 트랜잭션 내에 outbox 레코드를 작성하고, DynamoDB Streams를 Lambda가 처리하여 SNS/SQS로 발행한다.

인터뷰어는 "낙관적 락이 실패할 때 재시도 전략은?", "결제가 성공했는데 네트워크 타임아웃으로 응답을 못 받은 경우 중복 결제 방지는?"와 같은 엣지 케이스를 반드시 물어본다. 전자는 지수 백오프(exponential backoff)와 재고 재확인, 후자는 Idempotency Key의 외부 생성과 결제 게이트웨이의 멱등 API 활용으로 답한다.

---

### Q2. DynamoDB를 사용하는 마이크로서비스에서 핫 파티션(Hot Partition) 문제를 어떻게 진단하고 해결하는가?

**핵심 답변:**

핫 파티션은 특정 파티션 키에 트래픽이 집중될 때 DynamoDB의 파티션당 3,000 RCU/1,000 WCU 제한을 초과하여 발생한다. 진단은 CloudWatch의 `ConsumedWriteCapacityUnits` 메트릭의 지표와 `ThrottledRequests`를 파티션 키별로 분석하고, DynamoDB Contributor Insights로 핫 키를 직접 식별한다.

해결 전략은 데이터 접근 패턴에 따라 다르다. **Write-heavy 케이스**에서는 파티션 키에 랜덤 접미사(suffix sharding, 예: `userId_1`, `userId_2`)를 추가하여 쓰기를 분산하고, 읽기 시에는 N개의 샤드를 스캐터-게더로 조회 후 집계한다. **Read-heavy 케이스**에서는 DynamoDB Accelerator(DAX)를 인프라 계층에 투명하게 추가하거나, 자주 읽히는 데이터를 ElastiCache에 캐싱하는 패턴을 사용한다. **시간 기반 핫 파티션**(신제품 출시, 이벤트 티케팅)은 예측 가능한 경우 Adaptive Capacity를 활성화하고, 그래도 부족할 경우 Pre-provisioned Capacity와 Application Auto Scaling을 조합한다.

---

### Q3. AWS에서 이벤트 기반 마이크로서비스의 "Exactly-Once" 처리를 구현하는 것이 왜 어렵고, 실용적으로 어떻게 접근하는가?

**핵심 답변:**

분산 시스템에서 진정한 Exactly-Once는 이론적으로 불가능에 가깝다. 두 가지 장애 시나리오가 있다. 첫째로 **At-Least-Once** 시나리오에서는 메시지 처리는 성공했지만 브로커에게 ACK 전송이 실패하면 재전송된다. 둘째로 **At-Most-Once** 시나리오에서는 ACK를 먼저 보내고 처리하다 실패하면 유실된다.

실용적 접근법은 **"At-Least-Once + 소비자 측 멱등성(Idempotency)"**의 조합이다. 모든 메시지에 고유한 `message_id`를 부여하고, 소비자는 처리 전에 해당 ID가 이미 처리된 기록이 있는지 확인한다. 이 **Idempotency Store**로는 처리 중인 트랜잭션과 같은 DB 테이블(Conditional Write), DynamoDB의 조건부 쓰기(Conditional Writes), Redis의 `SET NX EX` 명령을 활용한다.

SQS에서는 **FIFO Queue + Deduplication ID**가 5분 내 중복을 자동 제거하고, Kafka에서는 **Kafka Transactions API**로 Producer 수준의 Exactly-Once를 지원한다. 하지만 소비자가 Kafka 외부 시스템(DB)에 결과를 저장할 때는 Outbox 패턴으로 DB 커밋과 Kafka offset commit을 함께 관리해야 한다.

---

### Q4. MSA에서 서비스 간 계약(Contract)을 어떻게 관리하고, API 버전 관리 전략은 무엇인가?

**핵심 답변:**

API 계약 관리의 핵심은 **"Consumer-Driven Contract Testing"(CDCT)**이다. Pact 프레임워크를 사용하면 소비자 서비스가 생산자에게 기대하는 요청/응답 계약을 코드로 정의하고, 이 계약이 생산자 CI/CD 파이프라인에서 자동으로 검증된다. 이렇게 하면 생산자가 API를 변경할 때 어떤 소비자가 영향받는지를 배포 전에 알 수 있다.

API 버전 관리 전략으로는 URL 경로 버전(`/v1/orders`, `/v2/orders`), HTTP 헤더 버전(`API-Version: 2`), Content Negotiation(`Accept: application/vnd.company.order-v2+json`) 세 방식이 있다. 대형 플랫폼에서는 URL 경로 버전이 캐싱 친화적이고 가시성이 높아 선호된다.

장기적으로는 **Strangler Fig 패턴**과 결합하여, 구버전과 신버전을 API Gateway 레벨에서 동시에 운영하면서 소비자를 점진적으로 마이그레이션한다. 구버전 일몰(sunset)은 `Deprecation` 헤더와 `Sunset` 헤더를 응답에 포함시켜 소비자에게 명시적으로 통보한다.

---

### Q5. 이벤트 순서 보장이 중요한 MSA 시스템(예: 금융 거래 이력)을 Kafka로 설계할 때의 핵심 고려사항은?

**핵심 답변:**

Kafka는 파티션 내에서만 순서를 보장하므로, 순서가 중요한 엔티티는 반드시 같은 파티션에 라우팅해야 한다. 금융 거래에서는 `accountId`를 파티션 키로 사용하여 동일 계좌의 모든 이벤트가 동일 파티션을 통과하도록 보장한다.

주의사항은 세 가지다. 첫째로 **파티션 수 변경 금지**: 파티션 수가 변경되면 키 해싱이 바뀌어 기존 엔티티의 이벤트가 다른 파티션으로 이동할 수 있다. 따라서 초기 파티션 수를 넉넉하게 설계해야 한다. 둘째로 **소비자 수와 파티션 수의 관계**: Consumer Group의 소비자 수는 파티션 수를 초과할 수 없으므로, 처리 병렬성의 상한선이 파티션 수에 의해 결정된다. 셋째로 **재처리(Replay) 전략**: 컨슈머 오프셋을 특정 시점으로 리셋하여 이벤트를 재처리할 때, 소비자가 멱등성을 보장해야 한다.

추가로 **Kafka 트랜잭션 + Fencing Token**을 사용하면, 좀비 프로듀서(네트워크 파티션 이후 되살아난 구 프로듀서)의 중복 쓰기를 방지할 수 있다.

---

## 🍎 Apple — 프라이버시, 보안, 대규모 iOS 생태계 연계

---

### Q6. 수십억 개의 Apple 디바이스가 APNs(Apple Push Notification Service)와 통신하는 인프라를 EDA로 설계한다면, 어떻게 수십억 디바이스의 이벤트를 처리하겠는가?

**핵심 답변:**

이 규모는 단일 이벤트 스트리밍 플랫폼으로는 처리 불가능하므로, 지역별 **Multi-Region Kafka Cluster**와 **계층적 이벤트 파이프라인**이 필요하다.

아키텍처 설계는 다음 5계층으로 구성된다. **Edge 계층**에서는 디바이스와 가장 가까운 PoP(Point of Presence)에서 WebSocket/HTTP/2 연결을 관리하며, 연결 상태를 인메모리로 추적한다. **지역 이벤트 수집 계층**에서 지역별 Kafka 클러스터가 디바이스 이벤트를 수집하고, 지역 내에서 처리 가능한 이벤트(지역 사용자에 대한 알림)를 로컬에서 처리한다. **글로벌 이벤트 복제 계층**에서 **MirrorMaker 2** 또는 **Confluent Replicator**가 글로벌 처리가 필요한 이벤트를 중앙 클러스터로 복제한다. **스트림 프로세싱 계층**에서 Apache Flink가 알림 타겟팅, 개인화, 도달 최적화 로직을 실시간으로 처리한다. **전달 계층**에서 APNs 게이트웨이 서비스가 최종 푸시를 실행하고, 전달 영수증(delivery receipt) 이벤트를 다시 Kafka로 발행한다.

Apple의 프라이버시 원칙(Privacy by Design)에 따라, 모든 이벤트는 디바이스 수준에서 차분 프라이버시(Differential Privacy)로 집계되거나, 엔드투엔드 암호화된 상태로만 파이프라인을 통과해야 한다.

---

### Q7. Apple의 iCloud 데이터 동기화처럼 오프라인 가능 클라이언트와 서버 간의 이벤트 동기화를 어떻게 설계하겠는가?

**핵심 답변:**

이는 **오프라인 퍼스트(Offline-First)** 아키텍처 문제이며, CRDT(Conflict-free Replicated Data Type)와 Lamport 타임스탬프가 핵심이다.

핵심 설계 원칙은, 클라이언트 디바이스가 오프라인 상태에서도 로컬 이벤트 로그에 변경사항을 기록하고, 연결 복구 시 이 이벤트 델타를 서버에 동기화하는 것이다. 충돌(Conflict) 해결 전략으로는 Last-Write-Wins(LWW)이 가장 단순하지만 데이터 유실 위험이 있고, **CRDT**는 자동으로 충돌 없이 병합 가능한 데이터 구조(Counter, Set, Map)를 제공한다. Apple은 실제로 iCloud Notes에 Operation-based CRDT를 사용하는 것으로 알려져 있다.

벡터 클락(Vector Clock)은 각 디바이스의 이벤트 인과관계를 추적하여 "진정한 충돌"과 "인과적으로 선행하는 변경"을 구분할 수 있게 한다. 서버는 각 이벤트의 벡터 클락을 비교하여 충돌을 감지하고, 충돌 해결 정책(자동 병합, 사용자 선택)을 적용한다.

---

### Q8. MSA 환경에서 서비스 간 통신에 Zero-Trust 보안 모델을 구현하는 방법은?

**핵심 답변:**

Zero-Trust는 "네트워크 위치를 신뢰하지 않고 모든 요청을 명시적으로 검증한다"는 원칙이다. MSA 구현에서는 세 계층으로 나뉜다.

**전송 보안 계층**에서는 서비스 메시(Istio/Linkerd)의 **mTLS**를 사용하여 모든 서비스 간 통신을 암호화하고 상호 인증한다. SPIFFE(Secure Production Identity Framework For Everyone) 표준에 따른 **SVID(SPIFFE Verifiable Identity Document)**를 각 서비스에 발급하여, 서비스 자체가 자신의 암호화 정체성을 증명한다.

**인가 정책 계층**에서는 **OPA(Open Policy Agent)**를 Sidecar 또는 중앙 서비스로 배포하여, 모든 서비스 간 요청에 대해 **속성 기반 접근 제어(ABAC)** 정책을 실시간으로 평가한다. "Order Service는 Customer Service의 GET 엔드포인트만 접근 가능하다"는 정책을 Rego 언어로 코드화하여 GitOps로 관리한다.

**비밀 관리 계층**에서는 **HashiCorp Vault** 또는 **AWS Secrets Manager**와 동적 비밀(Dynamic Secret) 패턴을 사용하여, 서비스가 시작될 때만 임시 자격증명을 발급받고 이를 주기적으로 교체한다.

---

## 🟢 Nvidia — GPU 클러스터, AI 인프라, 고성능 컴퓨팅 연계

---

### Q9. GPU 클러스터에서 분산 딥러닝 학습 파이프라인을 EDA로 설계할 때, 이벤트 기반으로 처리해야 할 핵심 이벤트 유형과 그 처리 방식은?

**핵심 답변:**

ML 학습 파이프라인의 EDA 설계는 **데이터 이벤트**, **컴퓨트 이벤트**, **메트릭 이벤트** 세 스트림으로 구성된다.

**데이터 이벤트 스트림**에서는 새 학습 데이터 도착(DatasetVersionCreated), 데이터 검증 완료(DataValidationPassed/Failed), 특징 엔지니어링 완료(FeatureStoreUpdated) 이벤트가 Kafka로 흘러 파이프라인 다음 단계를 트리거한다.

**컴퓨트 이벤트 스트림**에서는 학습 Job이 Kubernetes Job으로 실행되고, 각 에포크 완료(EpochCompleted), 체크포인트 저장(CheckpointSaved), GPU 장애(GPUNodeFailed) 이벤트를 발행한다. Fault-tolerant 학습은 가장 최근 체크포인트에서 재시작하는 방식으로 구현된다.

**메트릭 이벤트 스트림**에서는 각 학습 Step의 손실값, GPU 활용률, 메모리 사용량이 실시간 스트림으로 발행된다. Flink가 이를 처리하여 학습 커브 이상(loss explosion, plateau)을 CEP 패턴으로 감지하고, Hyperparameter 자동 튜닝 서비스(Optuna, Ray Tune)에 피드백한다.

Nvidia의 **NCCL(NVIDIA Collective Communications Library)** 수준의 고속 GPU 간 통신은 이벤트 기반이 아닌 RDMA(Remote Direct Memory Access) 기반 동기 통신이지만, 이를 오케스트레이션하는 상위 레벨 파이프라인 제어는 EDA 패턴으로 설계하는 것이 표준이다.

---

### Q10. 실시간 AI 추론 서비스를 MSA로 구성할 때, 레이턴시와 처리량을 동시에 최적화하기 위한 아키텍처 패턴은?

**핵심 답변:**

AI 추론 서비스의 레이턴시-처리량 트레이드오프 최적화는 **Dynamic Batching**, **모델 캐싱**, **비동기 파이프라인** 세 축으로 접근한다.

**Dynamic Batching**은 NVIDIA Triton Inference Server의 핵심 기능으로, 개별 요청이 도착하면 설정된 최대 지연(예: 1ms) 내에 대기하다가 배치로 묶어 GPU에 한 번에 전송함으로써 GPU 활용률을 극대화한다. 이를 통해 개별 레이턴시는 약간 증가하지만 전체 처리량이 수십 배 향상된다.

**Model Ensemble과 Pipeline Chaining**에서는 전처리(CPU) → 추론(GPU) → 후처리(CPU) 단계를 각각 독립적인 마이크로서비스로 분리하고, gRPC 스트리밍으로 연결하여 파이프라인 병렬성을 확보한다.

**비동기 추론 패턴**에서는 응답 시간이 중요하지 않은 배치 추론(일괄 추천 생성, 오프라인 특징 집계)은 Kafka를 통한 비동기 이벤트 기반 처리로 GPU 리소스를 효율적으로 활용한다.

**Multi-Model Serving**은 하나의 Triton 인스턴스에 수십 개 모델을 로드하고, GPU 메모리 압박 시 LRU로 모델을 언로드/재로드하는 전략으로 GPU 클러스터의 TCO를 최적화한다.

---

## 🔵 Google — 분산 시스템 이론, SRE, GCP 생태계

---

### Q11. Google의 Spanner처럼 글로벌 분산 MSA에서 강한 일관성(Strong Consistency)을 제공하는 것이 가능한가? CAP 정리와 모순되지 않는가?

**핵심 답변:**

이는 CAP 정리에 대한 깊은 이해를 테스트하는 질문이다. 모순이 아니다. CAP 정리는 네트워크 파티션 발생 시 일관성과 가용성 중 하나를 포기해야 한다고 말하지만, 파티션은 항상 발생하는 것이 아니라 예외적 사건이다. **PACELC 정리**가 더 정확한 모델로, 파티션이 없는 정상 상태에서도 레이턴시와 일관성 사이의 트레이드오프가 존재함을 명시한다.

Google Spanner의 혁신은 **TrueTime API**다. GPS와 원자시계를 사용하여 각 데이터센터가 시간의 불확실성 구간([earliest, latest])을 알 수 있게 한다. 트랜잭션 커밋 시 TrueTime의 commit wait(불확실성 구간이 지날 때까지 대기)를 적용함으로써, 전 세계에 분산된 노드들이 물리적 시간 기반의 전역 순서를 보장한다. 이를 **External Consistency**라고 하며, 이는 CAP의 C(Consistency)를 선형 일관성(Linearizability) 수준에서 달성하면서 P(Partition Tolerance)도 유지한다. 단, 이는 파티션 발생 시 가용성(A)을 희생하는 CP 시스템이다.

MSA 설계 관점에서 중요한 함의는, "Eventual Consistency는 MSA의 필수 타협이 아니라, 비용을 지불할 의향이 있다면 Strong Consistency도 분산 환경에서 구현 가능하다"는 것이다. 단 그 비용은 레이턴시(TrueTime의 commit wait는 보통 7~10ms)와 복잡성이다.

---

### Q12. GCP의 Pub/Sub와 Kafka의 아키텍처 차이는 무엇이며, 어떤 기준으로 선택하는가?

**핵심 답변:**

두 시스템은 이벤트 스트리밍이라는 같은 목적을 전혀 다른 아키텍처로 달성한다.

**Kafka**는 **파티셔닝된 분산 로그(Partitioned Distributed Log)**로, 오프셋 기반 소비자 관리, 파티션 수에 비례한 처리량 스케일링, 메시지 보존 기간 내 재처리가 특징이다. 소비자가 직접 오프셋을 관리하므로 풀 기반(Pull-based) 소비다.

**GCP Pub/Sub**는 **관리형 글로벌 메시지 버스**로, 서버리스 자동 스케일링(파티션 관리 불필요), 최소 한 번 전달(At-Least-Once), 최대 7일 메시지 보존(Seek to timestamp으로 재처리 가능)이 특징이다. 풀과 푸시 기반 소비를 모두 지원한다.

선택 기준은 네 가지다. **파티션 수준 순서 보장**이 필요하면 Kafka를 선택한다. **운영 오버헤드 최소화**와 GCP 생태계 통합이 중요하면 Pub/Sub을 선택한다. **낮은 레이턴시**(~5ms 이하)가 중요하면 Kafka (Pub/Sub은 약 100ms). **이벤트 소싱/무한 리플레이**가 필요하면 Kafka + 별도 오브젝트 스토리지 아카이빙을 고려한다. 참고로 GCP Bigtable이나 BigQuery를 이벤트 저장소로 결합하면 Pub/Sub의 보존 기간 한계를 극복할 수 있다.

---

### Q13. 서비스 메시 없이 수천 개의 마이크로서비스에서 관찰성(Observability)을 어떻게 확보하겠는가? OpenTelemetry 기반 설계를 설명하라.

**핵심 답변:**

OpenTelemetry는 SDK(계측), Collector(수집/처리), Exporter(백엔드 전송) 세 컴포넌트로 구성된다.

설계 전략은 **Auto-Instrumentation + 수동 계측의 조합**이다. Java, Python, Go용 OpenTelemetry 에이전트는 애플리케이션 코드 변경 없이 주요 프레임워크(Spring, Express, Django)와 라이브러리를 자동 계측하여 트레이스와 메트릭을 생성한다. 비즈니스 메트릭(주문 금액, 사용자 세그먼트)은 수동 계측(Custom Spans, Custom Metrics)으로 추가한다.

**OTel Collector**는 각 Pod에 Sidecar 또는 DaemonSet으로 배포하여 텔레메트리 데이터를 로컬에서 수집하고, 샘플링(Tail-Based Sampling으로 에러 포함 트레이스만 100% 보존, 정상 트레이스는 1% 샘플링), 필터링, 변환 후 백엔드로 전송한다. 이 계층을 두는 것은 백엔드 변경 시 애플리케이션 재배포 없이 Exporter만 교체할 수 있어 벤더 중립성을 보장한다.

백엔드 스택으로는 **Grafana LGTM 스택(Loki, Grafana, Tempo, Mimir)**이 비용 효율적인 오픈소스 선택이고, GCP 환경에서는 **Cloud Trace + Cloud Monitoring + Cloud Logging**의 네이티브 통합이 운영 편의성을 제공한다.

---

### Q14. EDA 시스템에서 이벤트 스톰(Event Storm) — 이벤트가 이벤트를 과다하게 트리거하는 현상 — 을 어떻게 감지하고 방지하는가?

**핵심 답변:**

이벤트 스톰은 Choreography 방식 Saga에서 특히 위험하며, A가 B를 트리거하고 B가 C를 트리거하고 C가 다시 A를 트리거하는 **순환 이벤트 체인**이나, 하나의 이벤트가 수천 개의 파생 이벤트를 생성하는 **팬아웃 폭발**로 나타난다.

감지는 이벤트 메타데이터의 **causation_id**와 **correlation_id**를 추적하여 Kafka 토픽의 메시지 수 급증, 이벤트 처리 레이턴시 급등, Dead Letter Queue(DLQ) 증가를 Flink CEP 패턴으로 모니터링함으로써 가능하다.

방지 전략은 세 가지다. **이벤트 깊이 제한(Max Event Depth)**으로 causation 체인이 설정된 깊이를 초과하면 처리를 거부한다. **이벤트 타입별 Rate Limiting**을 소비자 그룹에 적용하여 초당 처리 이벤트 수를 제한한다. **Idempotency 보장**으로 같은 이벤트를 여러 번 처리해도 부작용이 없게 하여 순환이 발생해도 무한 루프로 이어지지 않도록 한다. 장기적으로는 순환 의존성을 유발하는 서비스 경계 설계 자체를 DDD Bounded Context 기반으로 재검토해야 한다.

---

### Q15. SRE 관점에서 MSA의 에러 버짓(Error Budget)을 어떻게 정의하고, 마이크로서비스 의존성이 복잡할 때 SLO를 어떻게 계층화하는가?

**핵심 답변:**

**에러 버짓 = 1 - SLO 목표치**다. 99.9% 가용성 SLO라면 월간 43.8분의 다운타임이 허용된다. 에러 버짓이 소진되면 새 기능 배포를 중단하고 신뢰성 개선에 집중한다는 계약이 Engineering과 SRE 간에 명시적으로 존재해야 한다.

MSA에서 복잡한 의존성의 SLO 계층화 원칙은 **종속 서비스의 SLO는 의존하는 서비스의 SLO보다 항상 높아야 한다**는 것이다. Order Service가 99.9% SLO를 목표로 한다면, 그것이 의존하는 Inventory Service는 99.95%, Payment Service는 99.99% SLO를 가져야 한다. 이를 **SLO 체이닝(SLO Chaining)**이라 하며, 합성 가용성(Composite Availability) = A1 × A2 × A3 ... 공식으로 계산된다.

**Critical Path SLO**와 **Non-Critical Path SLO**를 구분하는 것도 중요하다. 주문 흐름의 Critical Path에 있는 서비스는 엄격한 SLO를 적용하고, 비동기 알림이나 추천 서비스는 더 느슨한 SLO를 적용하여 에러 버짓을 효율적으로 분배한다.

---

## 🟦 Microsoft — Azure 생태계, 엔터프라이즈 패턴, .NET 중심

---

### Q16. Azure Service Bus와 Azure Event Hub의 차이는 무엇이며, 어떤 MSA 시나리오에 각각 적합한가?

**핵심 답변:**

**Azure Service Bus**는 엔터프라이즈 메시징 패턴(큐, 토픽/구독, 데드레터 큐, 세션, 트랜잭션)에 최적화된 **신뢰성 중심** 메시지 브로커다. 메시지 크기 최대 100MB, 메시지 잠금과 피크/록 소비 패턴, 브라우저 가능한 메시지 큐를 지원한다. 주문 처리, 청구, 워크플로우 조율처럼 메시지 순서, 중복 감지, 트랜잭션 처리가 중요한 **비즈니스 트랜잭션 시나리오**에 적합하다.

**Azure Event Hub**는 초당 수백만 이벤트를 처리하도록 설계된 **처리량 중심** 이벤트 스트리밍 플랫폼이다. Kafka 호환 API를 제공하여 Kafka 워크로드를 코드 변경 없이 마이그레이션 가능하다. IoT 디바이스 텔레메트리, 클릭스트림 분석, 보안 이벤트 스트리밍처럼 대량의 이벤트를 수집하고 스트림 프로세싱하는 **고처리량 스트리밍 시나리오**에 적합하다.

선택 기준을 한 문장으로 요약하면: "메시지를 놓치지 않고 정확히 처리해야 하면 Service Bus, 엄청난 양의 이벤트를 실시간으로 흘려보내야 하면 Event Hub"다.

---

### Q17. .NET 마이크로서비스에서 MediatR과 이벤트 소싱(Event Sourcing)을 결합할 때의 아키텍처 패턴과 실용적 주의사항은?

**핵심 답변:**

MediatR은 CQRS의 Command/Query 디스패치를 프로세스 내에서 처리하는 인프라 추상화다. MediatR + EventStore(EventStoreDB 또는 Marten)의 조합으로 CQRS+ES를 구현하면, Command Handler가 도메인 이벤트를 EventStore에 저장하고, EventStore의 변경 알림(subscription)이 Projection 업데이트와 외부 이벤트 발행을 트리거한다.

실용적 주의사항은 네 가지다. **이벤트 버전 관리(Schema Evolution)**: 이벤트 스토어에는 버전 1의 이벤트와 버전 2의 이벤트가 영구적으로 공존하므로, 이벤트 업캐스터(Upcaster) 패턴으로 구 버전 이벤트를 현재 버전으로 변환하는 로직이 필수다. **Aggregate 스냅샷팅**: Aggregate의 이벤트 수가 수백~수천 개로 늘어나면 상태 재구성(Rehydration)이 느려지므로, 주기적으로 현재 상태를 스냅샷으로 저장하고 그 이후의 이벤트만 재생하는 최적화가 필요하다. **Projection 관리**: 여러 Read Model의 Projection이 EventStore 구독을 각각 독립적으로 유지하면, 새 Projection 추가 시 전체 이벤트 스트림 재처리 비용을 감수해야 한다. **Outbox와의 통합**: EventStoreDB의 이벤트가 외부 Kafka로 발행되는 시점에 이중 쓰기 문제가 발생하므로, EventStoreDB의 영구 구독(Persistent Subscription)을 Outbox Relay로 사용하는 패턴을 적용한다.

---

### Q18. Kubernetes에서 MSA 배포 시 Canary 배포와 Blue/Green 배포의 차이는 무엇이며, 각각 어떤 상황에서 선택하는가?

**핵심 답변:**

**Blue/Green 배포**는 동일한 용량의 두 환경을 유지하고, 새 버전이 준비되면 트래픽 전체를 즉시 전환하는 전략이다. 롤백이 신속(DNS 또는 LB 전환)하고, 두 버전 간의 인프라 차이(DB 스키마 등)를 테스트하기 좋다. 단, 리소스가 2배 필요하고, 전환 시 일부 진행 중인 요청이 종료될 수 있다.

**Canary 배포**는 신규 버전에 점진적으로 트래픽 비율을 높이는 전략이다(5% → 25% → 50% → 100%). 실제 프로덕션 트래픽으로 새 버전을 검증하면서, 에러율 증가나 레이턴시 저하가 감지되면 자동으로 롤백할 수 있다. Argo Rollouts나 Flagger 같은 Progressive Delivery 도구가 SLO 메트릭에 기반한 자동 Canary 프로모션/롤백을 지원한다.

선택 기준은 **위험도**와 **검증 방식**에 따라 다르다. 데이터베이스 스키마가 변경되어 두 버전이 같은 DB를 공유할 수 없는 경우 Blue/Green이 적합하고, 성능이나 기능 영향을 실제 사용자 트래픽으로 점진 검증하고 싶다면 Canary가 적합하다. MSA에서는 서비스별로 독립적인 배포 전략을 적용하는 것이 가능하며, 중요도가 높은 결제 서비스는 Canary, 레거시 마이그레이션 서비스는 Blue/Green을 선택하는 식이다.

---

### Q19. 분산 시스템에서 멱등성(Idempotency)을 API 수준과 이벤트 소비자 수준에서 각각 어떻게 구현하는가?

**핵심 답변:**

**API 수준 멱등성**은 클라이언트가 `Idempotency-Key` 헤더에 UUID를 포함하여 요청을 전송한다. 서버는 이 키를 Redis나 DB에서 조회하여 이미 처리된 요청이면 저장된 응답을 그대로 반환한다(요청을 재처리하지 않음). 처음 처리되는 요청이면 결과를 저장한 후 응답한다. 이 캐시의 TTL은 클라이언트의 최대 재시도 시간 창보다 길어야 한다. Stripe의 Idempotency Key 구현이 업계 표준 레퍼런스다.

**이벤트 소비자 수준 멱등성**은 두 가지 접근이 있다. **자연적 멱등성(Natural Idempotency)**: 연산 자체가 여러 번 실행해도 결과가 같은 경우(예: `SET price = 1000`은 멱등, `UPDATE price = price - 100`은 비멱등). 비멱등 연산은 **처리 이력 확인(Deduplication Table)** 방식으로 강제 멱등화한다. 소비자는 메시지 ID를 처리 전에 Dedup DB에 삽입(UPSERT/INSERT IGNORE)하고, 성공 시에만 실제 비즈니스 로직을 실행한다. DB 삽입과 비즈니스 로직은 같은 트랜잭션으로 묶어 원자성을 보장한다.

---

### Q20. 마이크로서비스의 서킷 브레이커가 Open 상태로 고착(Stuck Open)될 때 어떻게 진단하고 해결하는가?

**핵심 답변:**

서킷 브레이커 Open 고착의 원인은 보통 세 가지다. 첫 번째는 **다운스트림 서비스의 완전 복구 실패**로, Half-Open 상태에서 테스트 요청이 여전히 실패하면 다시 Open으로 복귀한다. 이때는 Fallback 로직(기본값 반환, 캐시 사용, 기능 저하 모드)이 올바르게 작동하는지 확인한다. 두 번째는 **임계치 설정의 부적절**로, 실패율 임계치가 너무 낮거나 샘플 크기가 너무 작아 정상적인 노이즈에도 Open이 되는 경우다. 롤링 윈도우(시간 기반 vs 카운트 기반)와 임계치를 트래픽 패턴에 맞게 조정한다. 세 번째는 **타임아웃 설정 문제**로, 서킷 브레이커의 슬립 기간이 다운스트림 복구 시간보다 짧아 Half-Open 테스트가 항상 실패한다. 슬립 기간을 다운스트림의 평균 복구 시간의 2~3배로 설정한다.

진단은 Micrometer나 Resilience4j의 상태 지표(state, slow-call-rate, failure-rate)를 Grafana 대시보드에 시각화하고, Actuator 엔드포인트로 각 서킷 브레이커의 현재 상태와 메트릭을 실시간 확인한다. 운영 응급 상황에서는 Circuit Breaker를 수동으로 Closed 상태로 강제 전환하는 관리 API를 제공하고, 이 작업은 변경 관리 프로세스를 거쳐 수행한다.

---

## ⚫ Tesla — 실시간 차량 데이터, IoT, Edge Computing

---

### Q21. Tesla 차량의 센서 데이터(초당 수십 GB 규모)를 클라우드로 실시간 처리하는 EDA 파이프라인을 설계하라.

**핵심 답변:**

Tesla 차량은 8개 카메라, 12개 초음파 센서, 레이더, 1개의 Tesla FSD 컴퓨터(144 TOPS)로 초당 수 GB의 원시 데이터를 생성한다. 이 모든 데이터를 실시간으로 클라우드에 전송하는 것은 비현실적이므로, **Edge-First 아키텍처**가 핵심이다.

**엣지 처리 계층(차량 내)**에서는 FSD 컴퓨터가 실시간 추론을 수행하고, 원시 센서 데이터 대신 **이벤트 델타**(주목할 만한 상황: 급제동, 차선 인식 실패, 이상 기상 조건)와 집계 통계를 로컬 스토리지에 버퍼링한다. 연결이 가능할 때 배치로 클라우드에 업로드하는 **Opportunistic Upload** 패턴을 사용한다.

**클라우드 수집 계층**에서는 Kafka(또는 Kinesis)가 전 세계 수백만 차량의 이벤트를 수집하고, 차량 ID를 파티션 키로 사용한다.

**스트림 프로세싱 계층**에서는 Flink가 실시간으로 이상 감지(급제동 클러스터 = 도로 위험물 신호), 소프트웨어 이상 패턴 감지(OTA 업데이트 후 특정 모델 이상 증가 = 즉각 롤백 신호), Autopilot 실패 패턴 분석을 수행한다.

**배치 학습 계층**에서는 Parquet 포맷으로 S3/GCS에 저장된 이벤트 데이터가 Spark 기반 학습 데이터 파이프라인으로 흘러 FSD 모델 재학습에 활용된다.

---

### Q22. OTA(Over-The-Air) 소프트웨어 업데이트 시스템을 MSA로 설계할 때, 수백만 대 차량에 안전하게 배포하기 위한 전략은?

**핵심 답변:**

자동차 소프트웨어 업데이트는 안전과 직결되므로, 일반 소프트웨어 배포보다 훨씬 보수적인 전략이 필요하다.

**업데이트 서비스 MSA 구성**은 크게 다섯 서비스로 나뉜다. 패키지 관리 서비스(빌드 아티팩트, 서명, 메타데이터), 타겟팅 서비스(어떤 차량 세그먼트에 어떤 버전 배포), 스케줄링 서비스(업데이트 시도 시간 제어 — 충전 중, 주차 중, 새벽 시간 우선), 상태 추적 서비스(다운로드/설치/재부팅 진행 상황), 롤백 서비스(이상 감지 시 자동 롤백 트리거)다.

**Progressive Rollout 전략**은 소프트웨어 Canary 배포와 유사하지만 훨씬 더 단계적이다. 1단계로 내부 Tesla 직원 차량 → 2단계로 Beta 프로그램 참여 차량(0.1%) → 3단계로 지역별 순차 확대(캘리포니아 → 유럽 → 아시아) → 4단계로 전체 롤아웃의 순서로 진행된다.

**Safety Gate**는 각 단계 완료 후 자동 분석을 통해 이상 시그널(충돌률 증가, FCW 실패율 증가, 특정 기능 이상 사용 패턴)이 기준치를 초과하면 롤아웃을 자동 중단하고 알림을 발송한다. 이를 위해 차량 텔레메트리 이벤트와 업데이트 버전 정보를 Kafka로 실시간 스트리밍하고, Flink CEP가 이상 패턴을 감지하는 EDA 파이프라인이 Safety Gate의 핵심 인프라를 구성한다.

---

### Q23. MSA 환경에서 분산 구성 관리(Distributed Configuration Management)를 어떻게 설계하고, Feature Flag와 어떻게 통합하는가?

**핵심 답변:**

분산 구성 관리는 **중앙화된 구성 저장소**와 **동적 구성 갱신** 두 축으로 설계한다.

**구성 저장소**로는 Consul, etcd, AWS AppConfig, HashiCorp Vault(비밀 정보)가 사용된다. 서비스는 시작 시 구성을 로드하고, 구성 변경 시 **long-polling, watch 메커니즘, 또는 이벤트 구독**으로 실시간 갱신을 받는다. 구성은 Git으로 관리되고(GitOps), PR 리뷰를 거쳐 중앙 저장소에 병합되면 자동으로 서비스에 전파된다.

**Feature Flag 통합**은 LaunchDarkly, Split.io, Unleash 같은 플랫폼이나 자체 구현을 통해 코드 배포와 기능 활성화를 분리한다. 새 코드는 Feature Flag로 비활성화된 상태로 배포되고, 운영자가 대시보드에서 점진적으로 활성화한다. 이는 코드를 프로덕션에 먼저 배포하고 준비되면 기능을 켜는 **Feature Branch를 서버 측으로 이동**하는 효과다.

Tesla 차량 시나리오에서는 Feature Flag가 특정 지역, 특정 FSD 버전, 특정 하드웨어 구성의 차량에만 새 Autopilot 기능을 활성화하는 **타겟팅 기반 점진 활성화**에 사용된다.

---

## 🔀 공통 고급 주제 — 모든 빅테크 공통 심화 문제

---

### Q24. 이벤트 기반 아키텍처에서 "Poison Message"(처리를 계속 실패시키는 메시지)를 어떻게 감지하고 처리하는가?

**핵심 답변:**

Poison Message는 소비자가 재시도를 무한히 반복하게 만들어, 그 이후 메시지들의 처리를 막는(head-of-line blocking) 치명적인 운영 문제다.

처리 전략은 **재시도 정책 + DLQ(Dead Letter Queue) 조합**이다. 소비자는 메시지 처리 실패 시 지수 백오프(1s, 2s, 4s, 8s...)로 최대 N회 재시도하고, N회 초과 시 메시지를 DLQ로 이동시킨다. DLQ는 별도 모니터링 대상이며, 운영자 또는 자동화 로직이 분석 후 수동 재처리, 수정 후 재처리, 폐기를 결정한다.

고급 패턴으로는 **재시도 큐 계층화**가 있다. 원본 큐 → Retry-1 큐(5분 지연) → Retry-2 큐(30분 지연) → Retry-3 큐(2시간 지연) → DLQ 구조로, 일시적 오류(네트워크 플래피)는 자동 복구되고 영구적 오류는 DLQ로 격리된다.

Kafka에서는 기본 DLQ 지원이 없으므로, Confluent의 `ErrorHandlingDeserializer`, Spring Kafka의 `SeekToCurrentErrorHandler`, 또는 직접 DLQ 토픽으로 발행하는 로직을 구현해야 한다. **메시지 스키마 불일치**가 Poison Message의 가장 흔한 원인이므로, Schema Registry 기반 호환성 검사가 예방의 첫 번째 방어선이다.

---

### Q25. MSA에서 테스트 전략을 마이크로 수준부터 시스템 수준까지 계층화하라. 특히 이벤트 기반 통합 테스트는 어떻게 접근하는가?

**핵심 답변:**

**테스트 피라미드의 MSA 버전**은 아래서부터 단위 테스트 → 컴포넌트 테스트 → 계약 테스트 → 통합 테스트 → E2E 테스트로 구성된다.

**단위 테스트**는 도메인 로직과 이벤트 핸들러를 인메모리에서 격리 테스트한다. **컴포넌트 테스트**는 단일 서비스를 실제 DB(Testcontainers)와 인메모리 이벤트 브로커(EmbeddedKafka, Spring's TestContext)로 테스트한다. 외부 서비스는 WireMock이나 MockServer로 모킹한다.

**계약 테스트(Consumer-Driven Contract Testing)**는 MSA 테스트 전략의 핵심이다. Pact를 사용하면 소비자 서비스가 생산자에 대한 기대를 Pact 파일로 정의하고, 이 파일이 생산자 CI 파이프라인에서 실행되어 하위 호환성을 자동 검증한다. 이를 통해 통합 환경 없이도 서비스 간 계약을 신뢰성 있게 검증한다.

**이벤트 기반 통합 테스트**에서는 Kafka 토픽을 Testcontainers로 실제 구동하고, 테스트에서 이벤트를 발행한 후 소비자의 처리 완료를 Awaitility(polling with timeout)로 대기하는 비동기 어설션 패턴을 사용한다. **시나리오 기반 E2E 테스트**는 사용자 Journey(주문 → 결제 → 배송 알림)를 실제 환경과 유사한 Staging 환경에서 검증하되, 비용 문제로 최소한의 Happy Path와 핵심 에러 시나리오만 커버한다.

---

### Q26. MSA에서 데이터 조인(Join)이 필요한 복잡한 쿼리(예: 주문 + 고객 + 상품 정보)를 어떻게 처리하는가?

**핵심 답변:**

MSA의 Database-per-Service 원칙은 전통적인 DB JOIN을 불가능하게 만든다. 이를 해결하는 전략은 세 가지다.

**API Composition**은 가장 단순한 접근으로, API Gateway나 BFF가 각 서비스에 병렬로 쿼리하고 결과를 인메모리에서 조합한다. N+1 요청 문제를 DataLoader 패턴이나 GraphQL의 배치 로딩으로 완화한다. 소규모 조합이나 실시간 데이터가 필요한 경우에 적합하다.

**CQRS Read Model**은 이벤트 스트림을 구독하여 비정규화된 조인 뷰를 미리 계산해두는 방식이다. 주문, 고객, 상품 서비스가 각각 이벤트를 발행하면, Order Query Service가 이를 구독하여 `order_with_customer_product` 테이블(또는 Elasticsearch 인덱스)을 실시간으로 유지한다. 조회는 이 단일 테이블에서 이루어지므로 성능이 극히 빠르고, 단점은 최종 일관성과 Read Model 동기화 지연이다.

**스트림 처리 기반 조인**은 Kafka Streams나 Flink의 **스트림-테이블 조인** 기능을 사용하여, 이벤트 스트림과 참조 데이터 테이블(Kafka Compacted Topic 또는 State Store)을 실시간으로 조인하는 방식이다. 이는 중간 복잡성과 실시간성을 동시에 달성하는 균형 잡힌 접근이다.

---

### Q27. MSA 마이그레이션 프로젝트에서 가장 흔한 실패 패턴(Anti-Pattern)은 무엇이며, 어떻게 피하는가?

**핵심 답변:**

**분산 모놀리스(Distributed Monolith)**가 가장 치명적인 안티패턴이다. 서비스들이 런타임에 강하게 결합되어 있어 독립 배포가 불가능하고, MSA의 장점 없이 분산 시스템의 복잡성만 더해진 최악의 상태다. 원인은 대부분 서비스 경계를 DDD가 아닌 기능적 분해나 팀 편의로 정한 것이다.

**너무 잘게 쪼개기(Over-Engineering)**는 팀 규모와 도메인 복잡도에 비해 서비스 수가 너무 많아, 운영 오버헤드가 비즈니스 가치를 초과하는 상태다. Amazon은 "마이크로서비스는 팀이 소유할 수 있는 최대 크기로"를 원칙으로 삼는다.

**공유 데이터베이스(Shared Database)**는 여러 서비스가 같은 DB를 공유하여 사실상 스키마 결합이 생기는 패턴이다. 하나의 서비스가 스키마를 변경하면 다른 서비스가 깨진다.

**계층적 호출 연쇄(Service Chain)**는 요청이 A → B → C → D → E 순서로 동기 체인을 이루어, 하나의 서비스 장애가 전체 체인을 실패시키고 레이턴시가 합산되는 패턴이다. 이는 서비스 경계 재설계와 비동기 이벤트 기반 통신으로 해결한다.

**부재하는 팀 자율성(Absent Team Autonomy)**은 기술적으로 MSA를 구현했지만 모든 배포가 중앙 집중식 승인과 조율이 필요한 경우로, Conway's Law의 역설이다. 역-Conway 전략에 따른 팀 재편 없이는 MSA의 속도 이점을 실현할 수 없다.

---

### Q28. 멀티 클라우드 MSA 환경에서 서비스 디스커버리와 로드밸런싱을 어떻게 설계하는가?

**핵심 답변:**

멀티 클라우드 환경에서 각 클라우드 네이티브 서비스 디스커버리(AWS Cloud Map, GCP Service Directory)는 다른 클라우드를 인식하지 못하므로, **클라우드 중립적 서비스 레지스트리**가 필요하다.

**Consul**은 멀티 클라우드 서비스 디스커버리의 사실상 표준으로, 각 클라우드에 Consul 에이전트가 설치되어 로컬 서비스를 등록하고, Consul Federation을 통해 글로벌 서비스 카탈로그를 유지한다. Consul Connect의 서비스 메시 기능은 클라우드 경계를 넘는 mTLS와 서비스 인텐트 기반 접근 제어를 제공한다.

**로드밸런싱 전략**은 세 계층으로 구성된다. 글로벌 레벨에서는 DNS 기반 지리적 라우팅(Latency-based, Failover)이 트래픽을 최적 클라우드 리전으로 안내한다. 클라우드 레벨에서는 각 클라우드의 네이티브 LB(ALB, GLB)가 처리한다. 서비스 레벨에서는 Envoy의 Locality-aware Load Balancing이 같은 가용 영역(AZ) 내 서비스를 우선 사용하여 크로스-AZ 트래픽 비용을 최소화한다.

**클라우드 간 레이턴시**와 **데이터 이그레스 비용**이 멀티 클라우드의 주요 비용이므로, 서비스 경계를 클라우드 경계와 최대한 일치시키고(클라우드당 하나의 도메인), 불가피한 크로스 클라우드 통신은 최소화 설계해야 한다.

---

### Q29. 이벤트 소싱(Event Sourcing)을 도입했을 때 GDPR의 "잊혀질 권리(Right to Erasure)"를 어떻게 구현하는가?

**핵심 답변:**

이벤트 소싱은 이벤트 스트림을 불변(immutable) 레코드로 관리하기 때문에, 특정 사용자의 데이터를 삭제하는 것이 근본적으로 이 원칙과 충돌한다. 이는 실무에서 자주 간과되는 GDPR 규정 준수 문제다.

해결책은 두 가지 접근이 있다. **Crypto-Shredding(암호화 분쇄)** 방식에서는 각 사용자의 개인 데이터를 이벤트에 직접 저장하는 대신, 사용자별 암호화 키로 암호화한 후 저장한다. "잊혀질 권리" 요청 시 해당 사용자의 암호화 키를 삭제하면, 이벤트 스트림의 개인 데이터가 복호화 불가능한 상태가 되어 사실상 삭제와 동일한 효과를 달성한다. 이벤트 스트림의 불변성을 유지하면서 GDPR을 준수하는 가장 우아한 해결책이다.

**개인 데이터 외부화(PII Externalization)** 방식에서는 이벤트 내에 개인 식별 정보(PII)를 직접 포함하지 않고, 별도의 PII 저장소(암호화된 키-값 스토어)에 저장한 후 이벤트에는 참조 ID만 포함한다. 삭제 요청 시 PII 저장소의 데이터만 삭제하면 된다. 단, 이벤트 재처리 시 PII가 조회되지 않을 수 있다는 점을 시스템 설계에서 명시적으로 처리해야 한다.

두 방법 모두 **초기 설계 단계에서** 결정해야 하며, 사후 적용은 대규모 마이그레이션을 수반한다.

---

### Q30. MSA에서 캐싱 전략을 계층별로 설계하고, 캐시 무효화(Cache Invalidation) 문제를 이벤트 기반으로 어떻게 해결하는가?

**핵심 답변:**

MSA의 캐싱 계층은 클라이언트(CDN, 브라우저 캐시) → API Gateway(응답 캐시) → 서비스 계층(인메모리 캐시, Sidecar 캐시) → 분산 캐시(Redis, Memcached) → 데이터베이스(Query Cache, Buffer Pool)로 구성된다.

**캐시 무효화 전략**은 캐시 정책(Cache-Aside, Write-Through, Write-Behind)에 따라 다르지만, MSA에서 가장 강력한 방법은 **이벤트 기반 캐시 무효화**다. 데이터를 소유한 서비스가 변경 이벤트를 발행하면, 해당 데이터를 캐싱한 다른 서비스들이 이벤트를 구독하여 자신의 캐시를 무효화하거나 갱신한다.

예를 들어 Product Service가 `ProductPriceUpdated` 이벤트를 발행하면, Shopping Cart Service, Order Service, Search Service가 각자의 Redis 캐시에서 해당 상품의 캐시 엔트리를 삭제한다. 이는 폴링 기반 TTL 만료보다 훨씬 빠른 캐시 일관성을 제공한다.

**Cache Stampede(Thundering Herd)** 문제는 캐시 만료와 동시에 대량 요청이 DB로 쏠리는 현상인데, **Probabilistic Early Expiration** 기법(만료 시간이 가까울수록 확률적으로 미리 갱신), **Mutex Lock** 패턴(하나의 요청만 DB 조회하고 나머지는 대기), **Background Refresh** 패턴(만료 전에 백그라운드에서 갱신)으로 해결한다.

---

### Q31. gRPC와 REST를 MSA에서 어떻게 선택하고 조합하는가? gRPC의 스트리밍 모드를 활용하는 사례는?

**핵심 답변:**

REST와 gRPC의 선택은 **통신 패턴, 클라이언트 다양성, 성능 요구사항**에 따라 결정된다.

**REST(HTTP/JSON)**는 외부 클라이언트 API, 브라우저 접근, 파트너 API 연동, 개발자 경험이 중요한 경우에 적합하다. 인간 가독성, 도구 생태계(Swagger, Postman), 방화벽 통과 용이성이 장점이다.

**gRPC(HTTP/2 + Protobuf)**는 내부 서비스 간 통신에서 성능(~10배 높은 처리량, ~30% 낮은 레이턴시)과 엄격한 계약(Protobuf 스키마)이 필요할 때 적합하다. 양방향 스트리밍, 흐름 제어, 멀티플렉싱이 HTTP/2 수준에서 기본 지원된다.

gRPC의 네 가지 스트리밍 모드 활용 사례를 살펴보면 다음과 같다. **단순 RPC**는 일반 요청-응답. **서버 스트리밍**은 실시간 주식 가격, 로그 스트리밍, 이벤트 피드처럼 서버가 연속적으로 데이터를 전송. **클라이언트 스트리밍**은 대용량 파일 업로드, 음성 스트림 전송처럼 클라이언트가 청크로 전송. **양방향 스트리밍**은 실시간 채팅, 협업 도구, 게임 서버처럼 양측이 동시에 전송.

Envoy 프록시는 gRPC-Web 트랜스코딩을 지원하여, 브라우저 클라이언트가 REST/HTTP1.1로 요청하면 서버 측 gRPC 서비스로 자동 변환하는 **Hybrid API Gateway** 구성을 가능하게 한다.

---

### Q32. 마이크로서비스 배포 파이프라인(CI/CD)에서 "Production-Like" 환경을 어떻게 유지하고, 드리프트(Drift)를 방지하는가?

**핵심 답변:**

환경 드리프트는 시간이 지남에 따라 Development, Staging, Production 환경이 서로 달라지는 현상으로, "내 환경에서는 되는데..."의 근본 원인이다.

**GitOps(ArgoCD/Flux)**는 모든 환경의 선언적 상태를 Git에 저장하고, 실제 환경 상태가 Git의 상태에서 벗어나면 자동으로 동기화하는 방식으로 드리프트를 원천 방지한다. Git이 "진실의 원천(Source of Truth)"이 되어, 누가 언제 무엇을 변경했는지 완전한 이력이 남는다.

**Infrastructure as Code(Terraform, Pulumi)**는 클라우드 인프라 자체를 코드로 관리하여, 환경 간 인프라 구성의 일관성을 보장한다. Terraform의 `terraform plan`은 예상 변경사항을 미리 보여주어, 의도하지 않은 드리프트를 사전 감지한다.

**Immutable Infrastructure** 원칙은 서버에 직접 패치하거나 구성을 변경하는 대신, 항상 새 이미지를 빌드하여 교체하는 방식으로 환경의 재현성을 보장한다. 컨테이너 이미지 다이제스트(SHA256) 기반 배포는 Canary에서 사용한 정확히 같은 이미지가 Production에 배포됨을 보장한다.

---

### Q33. 이벤트 기반 MSA에서 백프레셔(Back-Pressure) 처리를 어떻게 설계하는가?

**핵심 답변:**

백프레셔는 소비자의 처리 능력을 초과하는 이벤트가 발행될 때, 시스템이 이를 안전하게 처리하는 메커니즘이다. 제대로 처리되지 않으면 메모리 고갈, 서비스 크래시, 이벤트 유실로 이어진다.

**Reactive Streams와 Project Reactor**는 소비자가 처리 가능한 속도를 생산자에게 신호로 전달하는 **demand-driven 프로토콜**을 제공한다. `request(n)` 메서드로 소비자가 "n개의 이벤트를 처리할 준비가 됐다"고 선언하면, 생산자가 그 속도에 맞춰 발행한다.

Kafka 소비자에서는 `max.poll.records`와 `max.poll.interval.ms` 설정으로 한 번에 처리할 레코드 수와 처리 시간 제한을 제어한다. 처리 시간이 폴링 간격을 초과하면 Consumer Group에서 제외(rebalance)될 수 있으므로, 무거운 처리는 비동기 처리 후 오프셋을 수동 커밋하는 방식으로 설계한다.

**토픽 파티션 수 조정**이 Kafka 기반 백프레셔 관리의 핵심 레버다. 소비자를 수평 확장하려면 파티션 수를 늘려야 하고, 이는 초기 설계 단계에서 충분히 여유롭게 설정해야 함을 의미한다. 파티션 수는 줄일 수 없으므로(줄이면 데이터 재배치 필요), 과소 설정은 영구적 확장성 제한으로 이어진다.

---

### Q34. 서비스 간 통신에서 타임아웃을 어떻게 전파하고 관리하는가? Deadline Propagation의 개념을 설명하라.

**핵심 답변:**

**Deadline Propagation**은 Google의 gRPC에서 정립한 개념으로, 분산 요청 체인에서 **남은 시간(Deadline)**을 각 다운스트림 서비스로 전파하는 패턴이다.

일반적인 타임아웃 문제는 다음과 같다. 클라이언트가 5초 타임아웃으로 A를 호출하고, A가 4초 타임아웃으로 B를 호출하면, 클라이언트가 2초 후 타임아웃으로 연결을 끊어도 A와 B는 각자의 타임아웃이 남아 불필요한 작업을 계속 수행한다.

Deadline Propagation에서는 클라이언트가 "이 요청의 절대 마감 시간(예: T+5s)"을 요청 메타데이터(gRPC deadline, HTTP `Request-Timeout` 헤더)에 포함한다. 각 서비스는 자신에게 남은 시간을 계산(Remaining = Deadline - now)하고, 다운스트림 호출 시 이 남은 시간을 타임아웃으로 사용한다. 만료된 Deadline이 있으면 즉시 요청을 거부하여 불필요한 처리를 방지한다.

이는 Cascading Failure 방지와 시스템 자원 보호에 매우 효과적이며, gRPC는 이를 네이티브로 지원한다. HTTP 기반 서비스에서는 `Request-Timeout` RFC 초안이나 커스텀 헤더로 구현한다. **Sentinel Token** 방식으로 취소 신호를 연쇄 전파하는 Go의 `context.Context`와 Java의 `CompletableFuture` 취소도 같은 철학의 구현체다.

---

### Q35. MSA 아키텍처에서 데이터 메시(Data Mesh)와의 관계는 무엇이며, 어떻게 통합하는가?

**핵심 답변:**

**Data Mesh**는 Zhamak Dehghani가 제안한 분산 데이터 아키텍처 패러다임으로, MSA가 비즈니스 서비스를 도메인별로 분산화했듯이 **데이터도 도메인별로 분산화**하는 것이다.

Data Mesh의 네 가지 원칙은 다음과 같다. **도메인 소유 데이터(Domain Ownership)**: 각 도메인 팀이 자신의 데이터를 소유하고 발행한다. **데이터 제품(Data as a Product)**: 데이터를 내부 사용자가 소비하는 제품으로 취급하여 SLA, 품질, 문서를 제공한다. **자가 서비스 데이터 플랫폼(Self-Serve Data Platform)**: 도메인 팀이 쉽게 데이터를 발행하고 소비할 수 있는 플랫폼 인프라. **연합 거버넌스(Federated Governance)**: 전사적 표준(스키마, 데이터 계약, 접근 정책)을 유지하면서 각 도메인의 자율성을 보장한다.

MSA와 Data Mesh의 통합에서 **이벤트 스트리밍(Kafka)**은 핵심 접착제 역할을 한다. 각 마이크로서비스가 발행하는 도메인 이벤트는 동시에 **데이터 제품**의 원천이 된다. 주문 서비스의 `OrderPlaced` 이벤트는 실시간 운영 이벤트이면서, 주문 도메인 팀이 발행하는 **Order Analytics Data Product**의 원천 스트림이기도 하다.

Data Mesh 구현에서는 각 Kafka 토픽을 **데이터 계약(Data Contract)** 레지스트리에 등록하고, 소비 팀은 이 계약을 구독하여 스키마 변경 알림을 받는다. Confluent의 Stream Catalog, Apache Atlas, DataHub가 이 메타데이터 레지스트리 역할을 한다.

---

# 부록: 핵심 아키텍처 결정 프레임워크

복잡한 아키텍처 결정을 내릴 때 다음 질문 순서로 접근하면 효과적이다.

**1단계 — 문제 정의**: 이 설계가 해결하는 비즈니스 또는 기술 문제는 정확히 무엇인가? 현재 고통이 MSA/EDA를 정당화할 만큼 큰가?

**2단계 — 트레이드오프 명시**: 이 결정이 포기하는 것은 무엇인가? (복잡성 vs 확장성, 일관성 vs 가용성, 개발 속도 vs 운영 성숙도)

**3단계 — 실패 모드 분석**: 이 아키텍처가 어떻게 실패할 수 있는가? 단일 장애점은 없는가? Cascading Failure 경로는?

**4단계 — 운영 준비도 평가**: 이 아키텍처를 운영할 팀의 역량과 도구가 준비되어 있는가? (관찰성, CI/CD, On-call 체계)

**5단계 — 진화 가능성**: 미래에 요구사항이 변할 때 이 아키텍처가 얼마나 유연하게 적응할 수 있는가?

---

*이 문서는 MSA와 EDA에 대한 이론적 깊이와 실무 적용 능력을 동시에 갖춘 Senior Cloud Engineer를 위해 작성되었습니다. 각 개념은 독립적으로 학습하기보다 연결 고리를 이해하며 전체 시스템 사고(Systems Thinking)로 접근하는 것이 중요합니다.*
