# Azure 추가 예상문제

---

## 1. Azure 서비스 기본 개념 (30문항)

### 문항 1

박대리는 Azure 리전 선택 시 고려해야 할 사항을 검토하고 있다. 다음 중 Azure 리전에 대한 설명으로 틀린 것은?

A. 모든 Azure 리전은 가용성 영역(Availability Zone)을 지원한다.  
B. Paired Region은 재해 복구를 위해 동일 지리적 영역 내 다른 리전과 쌍을 이룬다.  
C. Sovereign Region(예: Azure Government)은 일반 상용 리전과 물리적으로 분리된다.  
D. 특정 서비스는 일부 리전에서만 사용 가능하다.

**정답: A**  
**해설:** 모든 Azure 리전이 가용성 영역을 지원하는 것은 아닙니다. 가용성 영역은 선택된 리전에서만 제공되며, 각 리전은 최소 3개의 독립된 데이터센터로 구성됩니다.

---

### 문항 2

김과장은 Azure에서 제공하는 컴퓨팅 서비스를 비교하고 있다. 다음 중 서버리스 컴퓨팅에 해당하지 않는 것은?

A. Azure Functions  
B. Azure Logic Apps  
C. Azure Virtual Machine Scale Sets  
D. Azure Container Apps

**정답: C**  
**해설:** VM Scale Sets는 IaaS 기반으로 인프라를 직접 관리해야 합니다. Azure Functions, Logic Apps, Container Apps는 서버리스 또는 서버리스에 가까운 서비스입니다.

---

### 문항 3

이선임은 Azure App Service Plan을 선택하고 있다. 다음 중 Free/Shared 계층의 제약사항으로 맞는 것은?

A. 사용자 정의 도메인을 사용할 수 없다.  
B. 자동 스케일링이 가능하다.  
C. VNET 통합을 지원한다.  
D. 배포 슬롯을 사용할 수 있다.

**정답: A**  
**해설:** Free 계층은 사용자 정의 도메인을 지원하지 않으며, 자동 스케일링, VNET 통합, 배포 슬롯은 Standard 이상에서 지원됩니다. (Shared 계층은 사용자 정의 도메인은 지원하지만 SSL 바인딩은 제한적)

---

### 문항 4

송팀장은 Azure Cosmos DB의 일관성 수준을 선택하려고 한다. 다음 중 가장 강한 일관성에서 가장 약한 일관성 순서로 올바른 것은?

A. Strong → Bounded Staleness → Session → Consistent Prefix → Eventual  
B. Strong → Session → Bounded Staleness → Eventual → Consistent Prefix  
C. Eventual → Consistent Prefix → Session → Bounded Staleness → Strong  
D. Strong → Consistent Prefix → Session → Bounded Staleness → Eventual

**정답: A**  
**해설:** Cosmos DB는 5단계 일관성 모델을 제공하며, Strong이 가장 강하고 Eventual이 가장 약합니다. 순서는 Strong → Bounded Staleness → Session → Consistent Prefix → Eventual입니다.

---

### 문항 5

장전임은 Azure SQL Database와 Azure SQL Managed Instance의 차이를 비교하고 있다. 다음 중 SQL Managed Instance에서만 지원되는 기능은?

A. 자동 백업  
B. 교차 데이터베이스 쿼리  
C. 지역 복제  
D. 동적 데이터 마스킹

**정답: B**  
**해설:** SQL Managed Instance는 SQL Server와 거의 100% 호환성을 제공하며, 교차 데이터베이스 쿼리, SQL Agent, CLR, Linked Server 등을 지원합니다. 이는 단일 데이터베이스인 Azure SQL Database에서는 지원되지 않습니다.

---

### 문항 6

박대리는 Azure Event Grid와 Event Hub의 차이를 검토하고 있다. 다음 중 Azure Event Grid의 특징으로 맞는 것은?

A. 대용량 데이터 스트리밍에 최적화되어 있다.  
B. 이벤트 기반의 반응형 프로그래밍 모델을 제공한다.  
C. Apache Kafka 프로토콜을 지원한다.  
D. 이벤트 데이터를 장기간 보존한다.

**정답: B**  
**해설:** Event Grid는 이벤트 기반 아키텍처를 위한 라우팅 서비스로, 게시-구독 모델을 사용합니다. 대용량 스트리밍과 Kafka 프로토콜은 Event Hub의 영역입니다.

---

### 문항 7

김과장은 Azure Service Bus와 Azure Queue Storage를 비교하고 있다. 다음 중 Service Bus에서만 제공되는 기능은?

A. 메시지 큐잉  
B. 세션 기반 FIFO 보장  
C. REST API 지원  
D. 메시지 크기 제한

**정답: B**  
**해설:** Service Bus는 세션 기반 FIFO, 중복 감지, 트랜잭션, Dead Letter Queue 등 엔터프라이즈급 기능을 제공합니다. Queue Storage는 간단한 큐잉만 지원합니다.

---

### 문항 8

이선임은 Azure Cognitive Services(현 Azure AI Services)를 검토하고 있다. 다음 중 Azure AI Services에 포함되지 않는 것은?

A. Azure OpenAI Service  
B. Computer Vision  
C. Azure Databricks  
D. Speech Service

**정답: C**  
**해설:** Azure Databricks는 빅데이터 분석 플랫폼이며, AI Services에 포함되지 않습니다. Azure AI Services는 Vision, Speech, Language, Decision, OpenAI 등을 포함합니다.

---

### 문항 9

송팀장은 Azure 데이터베이스 서비스를 선택하고 있다. 다음 중 완전 관리형 오픈소스 데이터베이스 서비스가 아닌 것은?

A. Azure Database for MySQL  
B. Azure Database for PostgreSQL  
C. Azure Database for MariaDB  
D. Azure SQL Database

**정답: D**  
**해설:** Azure SQL Database는 Microsoft SQL Server 기반이며 오픈소스가 아닙니다. MySQL, PostgreSQL, MariaDB는 오픈소스 데이터베이스의 관리형 서비스입니다.

---

### 문항 10

장전임은 Azure Functions의 호스팅 플랜을 선택하고 있다. 다음 중 Consumption Plan의 제약사항으로 맞는 것은?

A. VNET 통합이 가능하다.  
B. 기본 실행 시간 제한이 5분이다.  
C. 항상 실행(Always On) 상태를 유지한다.  
D. 전용 인스턴스에서 실행된다.

**정답: B**  
**해설:** Consumption Plan은 기본 5분(최대 10분) 실행 시간 제한이 있으며, 콜드 스타트가 발생할 수 있습니다. Premium Plan이나 Dedicated Plan은 이러한 제약이 없습니다.

---

### 문항 11

박대리는 Azure Cache for Redis를 구성하려고 한다. 다음 중 Enterprise 계층에서만 제공되는 기능은?

A. 데이터 지속성  
B. Active Geo-Replication  
C. 클러스터링  
D. 인증

**정답: B**  
**해설:** Active Geo-Replication(양방향 지역 복제)은 Enterprise 계층에서만 지원됩니다. Premium 계층은 수동 Geo-Replication만 지원합니다.

---

### 문항 12

김과장은 Azure Notification Hubs를 구성하고 있다. 다음 중 지원되는 플랫폼이 아닌 것은?

A. iOS (APNs)  
B. Android (FCM)  
C. Windows (WNS)  
D. SMS 문자 메시지

**정답: D**  
**해설:** Notification Hubs는 푸시 알림 서비스로 APNs, FCM, WNS, Baidu 등을 지원합니다. SMS는 Azure Communication Services를 사용해야 합니다.

---

### 문항 13

이선임은 Azure Batch를 사용하여 대규모 병렬 작업을 실행하려고 한다. 다음 중 Azure Batch의 특징으로 틀린 것은?

A. 수천 개의 VM을 자동으로 프로비저닝할 수 있다.  
B. 작업 스케줄링과 큐 관리를 제공한다.  
C. Low-priority VM으로 비용을 절감할 수 있다.  
D. 실시간 스트리밍 처리에 최적화되어 있다.

**정답: D**  
**해설:** Azure Batch는 대규모 배치 작업과 HPC(High Performance Computing)에 최적화되어 있으며, 실시간 스트리밍은 Stream Analytics나 Event Hub를 사용해야 합니다.

---

### 문항 14

송팀장은 Azure Static Web Apps를 구성하고 있다. 다음 중 Static Web Apps의 기능이 아닌 것은?

A. GitHub/Azure DevOps CI/CD 통합  
B. Azure Functions 기반 API  
C. 사용자 정의 인증 프로바이더  
D. 서버 측 렌더링 (SSR) 완전 지원

**정답: D**  
**해설:** Static Web Apps는 정적 프론트엔드와 서버리스 API에 최적화되어 있으며, 전통적인 서버 측 렌더링은 App Service 등을 사용해야 합니다.

---

### 문항 15

장전임은 Azure Kubernetes Service와 Azure Container Instances를 비교하고 있다. 다음 중 ACI가 AKS보다 적합한 시나리오는?

A. 마이크로서비스 아키텍처 운영  
B. 단기 실행 배치 작업  
C. 복잡한 오케스트레이션  
D. 장기 실행 서비스

**정답: B**  
**해설:** ACI는 빠른 시작과 단기 실행 컨테이너에 적합합니다. 복잡한 오케스트레이션이나 장기 서비스는 AKS가 적합합니다.

---

### 문항 16

박대리는 Azure SignalR Service를 구성하려고 한다. 다음 중 SignalR Service의 용도로 적절하지 않은 것은?

A. 실시간 채팅 애플리케이션  
B. 실시간 대시보드  
C. 배치 데이터 처리  
D. 라이브 알림

**정답: C**  
**해설:** SignalR은 실시간 양방향 통신을 위한 서비스이며, 배치 처리는 Azure Batch나 Data Factory를 사용해야 합니다.

---

### 문항 17

김과장은 Azure Data Factory를 구성하고 있다. 다음 중 Data Factory의 Integration Runtime 유형이 아닌 것은?

A. Azure Integration Runtime  
B. Self-hosted Integration Runtime  
C. Azure-SSIS Integration Runtime  
D. Kubernetes Integration Runtime

**정답: D**  
**해설:** Data Factory는 Azure IR, Self-hosted IR, Azure-SSIS IR 세 가지만 지원합니다.

---

### 문항 18

이선임은 Azure API Management의 정책(Policy)을 구성하고 있다. 다음 중 정책 적용 시점이 아닌 것은?

A. inbound  
B. backend  
C. outbound  
D. deployment

**정답: D**  
**해설:** API Management 정책은 inbound, backend, outbound, on-error 네 가지 시점에 적용됩니다. deployment는 정책 시점이 아닙니다.

---

### 문항 19

송팀장은 Azure Container Registry의 SKU를 선택하고 있다. 다음 중 Premium SKU에서만 제공되는 기능은?

A. Webhook  
B. Geo-replication  
C. Docker 이미지 저장  
D. Azure AD 인증

**정답: B**  
**해설:** Geo-replication, Content Trust, Private Link, Customer-managed Key는 Premium SKU에서만 지원됩니다.

---

### 문항 20

장전임은 Azure Application Gateway v2와 v1의 차이를 비교하고 있다. 다음 중 v2에서만 지원되는 기능은?

A. SSL 종료  
B. Autoscaling  
C. 상태 프로브  
D. URL 기반 라우팅

**정답: B**  
**해설:** Application Gateway v2는 Autoscaling, 가용성 영역, 정적 VIP, Header Rewrite 등을 추가로 지원합니다.

---

### 문항 21

박대리는 Azure Service Fabric과 AKS를 비교하고 있다. 다음 중 Service Fabric의 고유 기능은?

A. 컨테이너 오케스트레이션  
B. Reliable Services/Actors 프로그래밍 모델  
C. 자동 스케일링  
D. 로드 밸런싱

**정답: B**  
**해설:** Service Fabric은 Reliable Services와 Reliable Actors라는 고유한 상태 저장 프로그래밍 모델을 제공합니다.

---

### 문항 22

김과장은 Azure Logic Apps Standard와 Consumption을 비교하고 있다. 다음 중 Standard 계획의 이점이 아닌 것은?

A. VNET 통합 지원  
B. 로컬 개발 및 디버깅  
C. 사용한 만큼만 과금  
D. 상태 저장/무상태 워크플로우 선택

**정답: C**  
**해설:** 사용한 만큼만 과금되는 것은 Consumption 계획의 특징입니다. Standard는 App Service Plan 기반으로 고정 비용이 발생합니다.

---

### 문항 23

이선임은 Azure Managed Grafana를 구성하려고 한다. 다음 중 Managed Grafana가 기본으로 통합되는 데이터 소스가 아닌 것은?

A. Azure Monitor  
B. Azure Data Explorer  
C. AWS CloudWatch  
D. Prometheus (Azure Monitor managed)

**정답: C**  
**해설:** Azure Managed Grafana는 Azure Monitor, Azure Data Explorer, Prometheus 등 Azure 네이티브 데이터 소스와 기본 통합됩니다. AWS CloudWatch는 별도 구성이 필요합니다.

---

### 문항 24

송팀장은 Azure의 SLA(Service Level Agreement)를 검토하고 있다. 다음 중 SLA가 가장 높은 구성은?

A. 단일 VM (Premium SSD)  
B. 가용성 집합 내 VM  
C. 가용성 영역에 분산된 VM  
D. 단일 VM (Standard HDD)

**정답: C**  
**해설:** 가용성 영역 분산 VM은 99.99% SLA를 제공합니다. 가용성 집합은 99.95%, 단일 VM(Premium SSD)은 99.9%입니다.

---

### 문항 25

장전임은 Azure의 글로벌 인프라 구조를 설명하려고 한다. 다음 중 올바른 계층 구조는?

A. Region → Geography → Availability Zone  
B. Geography → Region → Availability Zone  
C. Availability Zone → Region → Geography  
D. Geography → Availability Zone → Region

**정답: B**  
**해설:** Azure 인프라는 Geography(지리적 영역) → Region(리전) → Availability Zone(가용성 영역) 순으로 계층 구조를 가집니다.

---

### 문항 26

박대리는 Azure Chaos Studio를 사용하여 복원력 테스트를 수행하려고 한다. 다음 중 Chaos Studio가 지원하지 않는 결함 유형은?

A. VM 종료  
B. 네트워크 지연 주입  
C. DNS 장애  
D. 물리적 서버 전원 차단

**정답: D**  
**해설:** Chaos Studio는 소프트웨어 수준의 결함만 주입하며, 물리적 하드웨어 조작은 불가능합니다.

---

### 문항 27

김과장은 Azure 리소스의 이름 지정 규칙을 검토하고 있다. 다음 중 Storage Account 이름 규칙으로 틀린 것은?

A. 3~24자 길이  
B. 소문자와 숫자만 허용  
C. 하이픈(-) 사용 가능  
D. 전역적으로 고유해야 함

**정답: C**  
**해설:** Storage Account 이름은 소문자 영문과 숫자만 허용되며, 하이픈, 밑줄 등 특수문자는 사용할 수 없습니다.

---

### 문항 28

이선임은 Azure의 Compute Gallery(구 Shared Image Gallery)를 구성하고 있다. 다음 중 Compute Gallery의 기능이 아닌 것은?

A. 사용자 정의 이미지 버전 관리  
B. 여러 리전에 이미지 복제  
C. VM 실시간 마이그레이션  
D. RBAC 기반 이미지 공유

**정답: C**  
**해설:** Compute Gallery는 이미지 관리 및 배포를 위한 서비스이며, VM 실시간 마이그레이션은 제공하지 않습니다.

---

### 문항 29

송팀장은 Azure Spring Apps를 검토하고 있다. 다음 중 Azure Spring Apps의 주요 대상 워크로드는?

A. .NET Framework 레거시 앱  
B. Java Spring Boot 마이크로서비스  
C. PHP 웹 애플리케이션  
D. Python 데이터 파이프라인

**정답: B**  
**해설:** Azure Spring Apps는 VMware와 협력하여 Java Spring Boot/Spring Cloud 애플리케이션을 위한 관리형 플랫폼입니다.

---

### 문항 30

장전임은 Azure Digital Twins를 검토하고 있다. 다음 중 Digital Twins의 용도로 적절하지 않은 것은?

A. IoT 디바이스의 디지털 모델링  
B. 건물 에너지 시뮬레이션  
C. 공장 생산 라인 최적화  
D. 일반 웹 호스팅

**정답: D**  
**해설:** Digital Twins는 물리적 환경의 디지털 모델을 생성하는 IoT 플랫폼이며, 웹 호스팅과는 무관합니다.

---

## 2. Azure 아키텍처 구성 방안 (25문항)

### 문항 31

박대리는 고가용성 웹 애플리케이션 아키텍처를 설계하고 있다. 다음 중 단일 리전 내 고가용성 구성으로 가장 적절한 것은?

A. 단일 VM에 모든 서비스 배포  
B. 가용성 영역에 분산된 VM + Zone-redundant Load Balancer + ZRS Storage  
C. 동일 랙에 여러 VM 배치  
D. 단일 가용성 영역에 모든 리소스 집중

**정답: B**  
**해설:** 가용성 영역 분산, Zone-redundant 부하 분산기, ZRS 스토리지를 조합하면 단일 리전 내에서 최고 수준의 가용성을 확보할 수 있습니다.

---

### 문항 32

김과장은 마이크로서비스 간 비동기 통신을 설계하고 있다. 다음 중 이벤트 기반 아키텍처에 가장 적합한 서비스 조합은?

A. Azure Functions + SQL Database  
B. Event Grid + Service Bus + Azure Functions  
C. VM + File Share  
D. App Service + Blob Storage

**정답: B**  
**해설:** Event Grid(이벤트 라우팅) + Service Bus(메시지 큐잉) + Functions(이벤트 처리)는 이벤트 기반 마이크로서비스 아키텍처의 핵심 조합입니다.

---

### 문항 33

이선임은 재해 복구(DR) 전략을 설계하고 있다. 다음 중 RPO(Recovery Point Objective)가 가장 짧은 DR 전략은?

A. Cold Site  
B. Warm Standby  
C. Hot Standby (Active-Active)  
D. Backup & Restore

**정답: C**  
**해설:** Active-Active 구성은 실시간 데이터 동기화로 RPO가 거의 0에 가깝습니다. Cold Site는 RPO가 가장 길고, Backup & Restore는 마지막 백업 시점까지의 데이터만 복구됩니다.

---

### 문항 34

송팀장은 멀티 리전 배포 아키텍처를 설계하고 있다. 다음 중 글로벌 트래픽 분산에 적합하지 않은 서비스는?

A. Azure Front Door  
B. Azure Traffic Manager  
C. Azure Internal Load Balancer  
D. Azure CDN

**정답: C**  
**해설:** Internal Load Balancer는 VNET 내부 트래픽 분산용으로, 글로벌 트래픽 분산에는 Front Door, Traffic Manager, CDN을 사용해야 합니다.

---

### 문항 35

장전임은 N-tier 아키텍처를 Azure에 구현하려고 한다. 다음 중 3-tier 아키텍처의 올바른 구성은?

A. Presentation → Application → Database  
B. Network → Compute → Storage  
C. Frontend → Middleware → Cache  
D. Web → Queue → Batch

**정답: A**  
**해설:** 전통적 3-tier 아키텍처는 Presentation(웹 프론트엔드), Application(비즈니스 로직), Database(데이터) 계층으로 구성됩니다.

---

### 문항 36

박대리는 Landing Zone 아키텍처를 설계하고 있다. 다음 중 Azure Landing Zone의 핵심 구성 요소가 아닌 것은?

A. Management Group 계층 구조  
B. 네트워크 토폴로지  
C. 개별 VM 스펙 정의  
D. ID 및 액세스 관리

**정답: C**  
**해설:** Landing Zone은 거버넌스, 네트워크, 보안, ID 관리 등의 기반 인프라 설계이며, 개별 VM 스펙은 워크로드 수준에서 결정합니다.

---

### 문항 37

김과장은 CQRS(Command Query Responsibility Segregation) 패턴을 Azure에서 구현하려고 한다. 다음 중 가장 적합한 서비스 조합은?

A. 쓰기: Cosmos DB + 읽기: Azure SQL + 동기화: Event Hub  
B. 단일 SQL Database  
C. Blob Storage + Table Storage  
D. Redis Cache만 사용

**정답: A**  
**해설:** CQRS 패턴은 쓰기와 읽기 모델을 분리하며, 이벤트 기반 동기화를 통해 두 모델 간 데이터 일관성을 유지합니다.

---

### 문항 38

이선임은 Azure에서 Strangler Fig 패턴을 적용하여 레거시 시스템을 현대화하려고 한다. 다음 중 이 패턴의 핵심 요소는?

A. 전체 시스템을 한 번에 교체  
B. API Gateway를 통해 점진적으로 기능을 새 시스템으로 전환  
C. 레거시 시스템을 그대로 유지  
D. 데이터베이스만 먼저 마이그레이션

**정답: B**  
**해설:** Strangler Fig 패턴은 API Gateway(예: API Management)를 파사드로 사용하여 기능별로 점진적으로 새 시스템으로 트래픽을 전환합니다.

---

### 문항 39

송팀장은 Azure에서 멀티 테넌트 SaaS 아키텍처를 설계하고 있다. 다음 중 데이터 격리 수준이 가장 높은 방법은?

A. 테넌트별 별도 데이터베이스  
B. 공유 데이터베이스의 Row-level Security  
C. 공유 테이블의 TenantID 컬럼  
D. 공유 Blob Container

**정답: A**  
**해설:** 테넌트별 별도 데이터베이스는 가장 높은 수준의 데이터 격리를 제공하지만, 비용과 관리 부담이 증가합니다.

---

### 문항 40

장전임은 Azure에서 API 기반 마이크로서비스 아키텍처를 설계하고 있다. 다음 중 API Gateway 패턴의 이점이 아닌 것은?

A. 인증/인가 중앙화  
B. 요청 속도 제한  
C. 마이크로서비스 간 직접 통신 제거  
D. 데이터베이스 성능 향상

**정답: D**  
**해설:** API Gateway는 트래픽 관리, 보안, 변환을 담당하며, 데이터베이스 성능은 별개의 영역입니다.

---

### 문항 41

박대리는 Azure에서 Circuit Breaker 패턴을 구현하려고 한다. 다음 중 이 패턴의 주요 목적은?

A. 네트워크 대역폭 증가  
B. 장애가 발생한 서비스로의 반복적인 호출 방지  
C. 데이터 암호화  
D. 로그 수집

**정답: B**  
**해설:** Circuit Breaker 패턴은 실패한 서비스에 대한 연쇄 장애를 방지하고, 시스템 복원력을 높이는 패턴입니다.

---

### 문항 42

김과장은 Azure 환경에서 Blue-Green 배포와 Canary 배포를 비교하고 있다. 다음 중 Canary 배포의 특징은?

A. 전체 트래픽을 한 번에 전환  
B. 소량의 트래픽만 먼저 새 버전으로 라우팅  
C. 두 환경을 동시에 운영하지 않음  
D. 롤백이 불가능

**정답: B**  
**해설:** Canary 배포는 소량(예: 5-10%)의 트래픽을 새 버전으로 보내 안정성을 확인한 후 점진적으로 전환합니다.

---

### 문항 43

이선임은 Azure에서 데이터 레이크하우스 아키텍처를 설계하고 있다. 다음 중 핵심 구성 요소의 조합으로 가장 적절한 것은?

A. Azure Data Lake Storage Gen2 + Azure Synapse Analytics + Azure Databricks  
B. Blob Storage + SQL Database + Power BI  
C. File Share + Cosmos DB + Logic Apps  
D. Table Storage + Event Hub + Functions

**정답: A**  
**해설:** 데이터 레이크하우스는 Data Lake Storage Gen2(저장), Synapse Analytics(분석), Databricks(데이터 처리)를 핵심으로 합니다.

---

### 문항 44

송팀장은 Azure에서 IoT 솔루션 아키텍처를 설계하고 있다. 다음 중 올바른 데이터 흐름 순서는?

A. 디바이스 → IoT Hub → Stream Analytics → Storage/Dashboard  
B. 디바이스 → Blob Storage → IoT Hub → Functions  
C. 디바이스 → API Management → SQL Database  
D. 디바이스 → CDN → Event Grid

**정답: A**  
**해설:** IoT 아키텍처의 일반적 흐름은 디바이스에서 IoT Hub로 데이터 수집, Stream Analytics로 실시간 처리, Storage/Dashboard로 저장 및 시각화입니다.

---

### 문항 45

장전임은 Azure에서 Zero Trust 네트워크 아키텍처를 설계하고 있다. 다음 중 Zero Trust의 핵심 원칙이 아닌 것은?

A. 명시적으로 확인 (Verify Explicitly)  
B. 최소 권한 액세스 (Least Privilege)  
C. 위반 가정 (Assume Breach)  
D. 내부 네트워크 무조건 신뢰

**정답: D**  
**해설:** Zero Trust는 "절대 신뢰하지 않고 항상 확인"이 원칙이며, 내부 네트워크도 신뢰하지 않습니다.

---

### 문항 46

박대리는 Azure에서 Bulkhead 패턴을 적용하려고 한다. 다음 중 이 패턴의 구현 방법은?

A. 모든 서비스를 단일 리소스 풀에서 실행  
B. 서비스별로 리소스를 격리하여 장애 전파 방지  
C. 모든 트래픽을 단일 경로로 라우팅  
D. 데이터베이스 커넥션 풀 제거

**정답: B**  
**해설:** Bulkhead 패턴은 선박의 격벽처럼 리소스를 격리하여 하나의 서비스 장애가 전체 시스템에 영향을 미치지 않도록 합니다.

---

### 문항 47

김과장은 Azure에서 Saga 패턴을 구현하여 분산 트랜잭션을 관리하려고 한다. 다음 중 Saga 패턴의 특징으로 틀린 것은?

A. 각 서비스가 로컬 트랜잭션을 실행  
B. 실패 시 보상 트랜잭션 실행  
C. 강한 일관성(Strong Consistency) 보장  
D. Choreography 또는 Orchestration 방식 선택

**정답: C**  
**해설:** Saga 패턴은 최종 일관성(Eventual Consistency)을 제공하며, 강한 일관성은 분산 2PC(Two-Phase Commit)가 필요합니다.

---

### 문항 48

이선임은 Azure Well-Architected Framework의 신뢰성 기둥에 따라 복원력을 설계하고 있다. 다음 중 복원력 향상에 도움이 되지 않는 것은?

A. 상태 프로브 구성  
B. 재시도 정책 구현  
C. 단일 장애 지점(SPOF) 유지  
D. 비동기 메시징 사용

**정답: C**  
**해설:** 단일 장애 지점은 시스템 전체 장애를 유발할 수 있으므로 제거해야 합니다.

---

### 문항 49

송팀장은 Azure에서 Geode 패턴을 적용하려고 한다. 다음 중 Geode 패턴의 설명으로 맞는 것은?

A. 단일 리전에서만 운영  
B. 여러 리전에 동일한 스택을 배포하여 사용자에게 가장 가까운 인스턴스가 요청을 처리  
C. 데이터를 단일 중앙 위치에만 저장  
D. 글로벌 트래픽 분산 없이 운영

**정답: B**  
**해설:** Geode 패턴은 여러 지역에 동일한 백엔드 스택을 배포하여 사용자 가까이에서 요청을 처리하고 지연 시간을 최소화합니다.

---

### 문항 50

장전임은 Azure에서 Sidecar 패턴을 구현하려고 한다. 다음 중 Sidecar 패턴의 일반적인 사용 사례가 아닌 것은?

A. 로깅 에이전트  
B. 프록시/서비스 메시  
C. 주 애플리케이션 비즈니스 로직 실행  
D. 인증서 관리

**정답: C**  
**해설:** Sidecar는 주 애플리케이션의 보조 기능(로깅, 프록시, 모니터링 등)을 담당하며, 비즈니스 로직은 주 컨테이너에서 실행합니다.

---

### 문항 51

박대리는 Hub-Spoke 네트워크 토폴로지에서 Spoke 간 통신을 설계하고 있다. 다음 중 가장 확장성 있는 방법은?

A. 모든 Spoke 간 직접 Peering  
B. Hub에 Azure Firewall 배치 + UDR로 트래픽 라우팅  
C. 모든 Spoke에 Public IP 할당  
D. 각 Spoke에 별도 VPN Gateway 배치

**정답: B**  
**해설:** Hub의 Azure Firewall을 중앙 라우팅/필터링 포인트로 사용하고 UDR로 트래픽을 제어하는 것이 가장 관리하기 쉽고 확장성 있는 방법입니다.

---

### 문항 52

김과장은 Azure에서 API Versioning 전략을 설계하고 있다. 다음 중 Azure API Management에서 지원되는 API 버전 관리 방식이 아닌 것은?

A. URL Path 기반 (예: /v1/users)  
B. Query String 기반 (예: ?api-version=1.0)  
C. Header 기반 (예: Api-Version: 1.0)  
D. Body Payload 기반

**정답: D**  
**해설:** API Management는 URL Path, Query String, Header 방식의 버전 관리를 지원하며, Body 기반은 지원하지 않습니다.

---

### 문항 53

이선임은 Azure에서 Event Sourcing 패턴을 구현하려고 한다. 다음 중 이벤트 저장소로 가장 적합한 서비스는?

A. Azure Cosmos DB (Append-only)  
B. Azure Cache for Redis  
C. Azure CDN  
D. Azure Front Door

**정답: A**  
**해설:** Event Sourcing은 모든 상태 변경을 이벤트로 저장하므로, Cosmos DB의 Change Feed와 함께 사용하면 효과적입니다.

---

### 문항 54

송팀장은 Azure에서 Compute 선택 가이드를 작성하고 있다. 다음 중 "관리 부담 최소 + 이벤트 기반 + 짧은 실행 시간"에 가장 적합한 서비스는?

A. Azure Virtual Machines  
B. Azure App Service  
C. Azure Functions (Consumption)  
D. Azure Kubernetes Service

**정답: C**  
**해설:** Functions Consumption Plan은 서버리스, 이벤트 기반, 자동 스케일링으로 관리 부담이 최소이며 짧은 실행에 최적화되어 있습니다.

---

### 문항 55

장전임은 Azure에서 다중 리전 Active-Active 데이터베이스 구성을 설계하고 있다. 다음 중 네이티브로 Active-Active 전역 분산을 지원하는 서비스는?

A. Azure SQL Database  
B. Azure Cosmos DB  
C. Azure Database for PostgreSQL  
D. Azure Database for MySQL

**정답: B**  
**해설:** Cosmos DB는 multi-region writes를 네이티브로 지원하여 여러 리전에서 동시 읽기/쓰기가 가능한 유일한 Azure 데이터베이스 서비스입니다.

---

## 3. Azure 자격증 대비 (25문항)

### 문항 56

AZ-104 시험 범위에서, 다음 중 Azure에서 VM 가용성을 높이기 위한 방법으로 틀린 것은?

A. 가용성 집합(Availability Set)에 VM 배치  
B. 가용성 영역(Availability Zone)에 VM 분산  
C. VM Scale Set 사용  
D. 동일 Fault Domain에 모든 VM 배치

**정답: D**  
**해설:** 동일 Fault Domain에 모든 VM을 배치하면 해당 도메인 장애 시 모든 VM이 영향받습니다. 가용성을 높이려면 Fault Domain을 분산해야 합니다.

---

### 문항 57

다음 중 Azure AD(현 Microsoft Entra ID)의 Conditional Access 정책 조건으로 사용할 수 없는 것은?

A. 사용자 위치  
B. 디바이스 플랫폼  
C. 로그인 위험 수준  
D. 사용자의 월급

**정답: D**  
**해설:** Conditional Access는 사용자, 위치, 디바이스, 앱, 위험 수준 등의 신호를 기반으로 합니다. 급여 정보는 Azure AD의 범위가 아닙니다.

---

### 문항 58

다음 중 Azure에서 VM을 다른 리전으로 이동하는 올바른 방법은?

A. Azure Portal에서 리전 드롭다운을 변경  
B. Azure Site Recovery를 사용하여 VM 복제 후 장애 조치  
C. VM의 IP 주소만 변경  
D. Resource Group을 다른 리전으로 이동

**정답: B**  
**해설:** Azure Site Recovery를 사용하여 VM을 대상 리전으로 복제한 후 장애 조치를 통해 이동합니다. VM은 직접 리전을 변경할 수 없습니다.

---

### 문항 59

다음 중 Azure Virtual Network에서 기본적으로 허용되는 트래픽은?

A. 인터넷 인바운드 트래픽  
B. 동일 VNET 내 서브넷 간 트래픽  
C. VNET 간 트래픽  
D. 온프레미스에서의 트래픽

**정답: B**  
**해설:** 동일 VNET 내 서브넷 간 트래픽은 기본적으로 허용됩니다. 인터넷 인바운드는 NSG로 차단되며, VNET 간은 Peering이 필요합니다.

---

### 문항 60

다음 중 Azure DNS에서 CNAME 레코드를 생성할 수 없는 위치는?

A. 서브도메인 (예: www.contoso.com)  
B. Zone Apex (예: contoso.com)  
C. 다른 서브도메인 (예: mail.contoso.com)  
D. 긴 서브도메인 (예: app.dev.contoso.com)

**정답: B**  
**해설:** RFC 표준에 따라 Zone Apex(루트 도메인)에는 CNAME 레코드를 생성할 수 없습니다. Azure는 이를 위해 Alias 레코드를 제공합니다.

---

### 문항 61

다음 중 Azure Backup의 Recovery Services Vault에서 지원하지 않는 백업 대상은?

A. Azure VM  
B. Azure Files  
C. Azure Cosmos DB  
D. SQL Server in Azure VM

**정답: C**  
**해설:** Cosmos DB는 자체 연속 백업 기능을 제공하며, Recovery Services Vault를 사용하지 않습니다.

---

### 문항 62

다음 중 Azure AD의 Multi-Factor Authentication 방법이 아닌 것은?

A. Microsoft Authenticator 앱  
B. SMS 인증  
C. 하드웨어 FIDO2 보안 키  
D. 보안 질문 (What is your pet's name?)

**정답: D**  
**해설:** 보안 질문은 셀프 서비스 암호 재설정(SSPR)에서 사용되며, MFA 방법이 아닙니다.

---

### 문항 63

다음 중 Azure에서 Network Security Group Flow Log를 저장할 수 있는 위치는?

A. Azure Blob Storage  
B. Azure Files  
C. Azure Table Storage  
D. Azure Queue Storage

**정답: A**  
**해설:** NSG Flow Log는 JSON 형식으로 Azure Blob Storage에 저장됩니다. Log Analytics로 전송하여 분석할 수도 있습니다.

---

### 문항 64

다음 중 Azure VM의 Custom Script Extension에 대한 설명으로 틀린 것은?

A. VM 프로비저닝 후 스크립트를 실행할 수 있다.  
B. PowerShell 또는 Bash 스크립트를 지원한다.  
C. 스크립트 실행 시간 제한이 없다.  
D. Azure Storage나 GitHub에서 스크립트를 다운로드할 수 있다.

**정답: C**  
**해설:** Custom Script Extension은 기본 90분의 실행 시간 제한이 있습니다.

---

### 문항 65

다음 중 Azure Resource Manager(ARM) 배포 모드에 대한 설명으로 맞는 것은?

A. Incremental 모드는 템플릿에 없는 리소스를 삭제한다.  
B. Complete 모드는 템플릿에 없는 리소스를 삭제한다.  
C. 두 모드 모두 기존 리소스를 수정하지 않는다.  
D. Complete 모드가 기본 배포 모드이다.

**정답: B**  
**해설:** Complete 모드는 템플릿에 정의되지 않은 리소스 그룹 내 리소스를 삭제합니다. Incremental(기본값)은 기존 리소스를 유지합니다.

---

### 문항 66

다음 중 Azure Storage Account에서 지원하는 인증 방법이 아닌 것은?

A. Azure AD (Microsoft Entra ID)  
B. Shared Key  
C. Shared Access Signature (SAS)  
D. OAuth 1.0

**정답: D**  
**해설:** Azure Storage는 Azure AD, Shared Key, SAS를 지원합니다. OAuth 2.0은 Azure AD를 통해 지원되지만, OAuth 1.0은 지원하지 않습니다.

---

### 문항 67

다음 중 Azure Monitor의 Log Analytics에서 사용되는 쿼리 언어는?

A. SQL  
B. KQL (Kusto Query Language)  
C. GraphQL  
D. LINQ

**정답: B**  
**해설:** Log Analytics는 KQL(Kusto Query Language)을 사용하며, 이는 Azure Data Explorer에서도 사용되는 쿼리 언어입니다.

---

### 문항 68

다음 중 Azure VM의 Availability Set에서 Fault Domain과 Update Domain의 기본 최대 수가 올바른 것은?

A. Fault Domain: 2, Update Domain: 5  
B. Fault Domain: 3, Update Domain: 20  
C. Fault Domain: 5, Update Domain: 10  
D. Fault Domain: 10, Update Domain: 3

**정답: B**  
**해설:** Availability Set은 기본 Fault Domain 최대 3개, Update Domain 최대 20개를 지원합니다.

---

### 문항 69

다음 중 Azure에서 VM의 OS 디스크를 교체(Swap)하는 것이 가능한 상태는?

A. VM이 실행 중인 상태에서만 가능  
B. VM이 중지(할당 해제)된 상태에서만 가능  
C. 어떤 상태에서든 가능  
D. OS 디스크는 교체할 수 없다

**정답: B**  
**해설:** OS 디스크 교체는 VM이 할당 해제(Deallocated) 상태에서만 가능합니다.

---

### 문항 70

다음 중 Azure SQL Database의 DTU 기반 모델과 vCore 기반 모델의 차이점으로 맞는 것은?

A. DTU 모델은 CPU, 메모리, IO를 독립적으로 조정할 수 있다.  
B. vCore 모델은 CPU, 메모리를 독립적으로 조정할 수 있다.  
C. DTU 모델이 항상 더 저렴하다.  
D. vCore 모델은 Serverless 옵션을 지원하지 않는다.

**정답: B**  
**해설:** vCore 모델은 컴퓨트와 스토리지를 독립적으로 조정할 수 있으며, Azure Hybrid Benefit 적용도 가능합니다. DTU는 번들형입니다.

---

### 문항 71

다음 중 Azure에서 VM Scale Set의 Orchestration Mode로 지원되는 것은?

A. Uniform과 Flexible  
B. Standard와 Premium  
C. Basic과 Advanced  
D. Manual과 Automatic

**정답: A**  
**해설:** VM Scale Set은 Uniform(동일 VM 구성)과 Flexible(다양한 VM 구성 혼합) 두 가지 오케스트레이션 모드를 지원합니다.

---

### 문항 72

다음 중 Azure에서 Point-to-Site VPN이 지원하는 프로토콜이 아닌 것은?

A. OpenVPN  
B. IKEv2  
C. SSTP  
D. L2TP

**정답: D**  
**해설:** Azure P2S VPN은 OpenVPN, IKEv2, SSTP를 지원하며, L2TP는 지원하지 않습니다.

---

### 문항 73

다음 중 Azure Disk Encryption과 Server-Side Encryption의 차이점으로 맞는 것은?

A. 둘 다 같은 기술을 사용한다.  
B. Azure Disk Encryption은 OS 내부에서 BitLocker/DM-Crypt를 사용한다.  
C. Server-Side Encryption은 OS 내부에서 암호화한다.  
D. Azure Disk Encryption만 Customer-managed Key를 지원한다.

**정답: B**  
**해설:** Azure Disk Encryption은 OS 내부에서 BitLocker(Windows)/DM-Crypt(Linux)를 사용하며, Server-Side Encryption(SSE)은 Azure 플랫폼 수준에서 저장 시 암호화합니다.

---

### 문항 74

다음 중 Azure에서 리소스를 다른 리소스 그룹으로 이동할 때 발생하는 것은?

A. 리소스의 리전이 변경된다.  
B. 리소스 ID가 변경된다.  
C. 리소스에 적용된 태그가 삭제된다.  
D. 리소스의 성능이 향상된다.

**정답: B**  
**해설:** 리소스 그룹이 변경되면 리소스 ID의 경로가 변경됩니다. 리전과 태그는 유지됩니다.

---

### 문항 75

다음 중 Azure에서 스냅샷으로 Managed Disk를 생성할 때의 특징으로 틀린 것은?

A. 다른 리전에 디스크를 생성할 수 있다.  
B. 디스크 크기를 원본보다 줄일 수 있다.  
C. 다른 디스크 유형으로 변환할 수 있다.  
D. 스냅샷은 증분 방식으로 저장된다.

**정답: B**  
**해설:** 스냅샷에서 생성하는 디스크 크기는 원본 디스크와 같거나 더 커야 합니다. 줄일 수는 없습니다.

---

### 문항 76

다음 중 Azure App Service의 Deployment Slot에 대한 설명으로 틀린 것은?

A. 프로덕션과 스테이징 슬롯 간 트래픽을 분할할 수 있다.  
B. 슬롯 간 Swap 시 다운타임이 발생하지 않는다.  
C. Free 계층에서도 배포 슬롯을 사용할 수 있다.  
D. 각 슬롯은 고유한 호스트 이름을 가진다.

**정답: C**  
**해설:** Deployment Slot은 Standard 계층 이상에서만 사용 가능합니다.

---

### 문항 77

다음 중 Azure에서 Network Interface Card(NIC)에 대한 설명으로 틀린 것은?

A. 하나의 VM에 여러 NIC를 연결할 수 있다.  
B. NIC는 반드시 하나의 서브넷에 속해야 한다.  
C. NIC에 여러 Private IP를 할당할 수 있다.  
D. 모든 VM 크기가 동일한 수의 NIC를 지원한다.

**정답: D**  
**해설:** VM 크기에 따라 지원되는 최대 NIC 수가 다릅니다. 소규모 VM은 NIC 수가 제한됩니다.

---

### 문항 78

다음 중 Azure에서 Proximity Placement Group의 용도는?

A. VM을 서로 다른 리전에 분산  
B. VM을 물리적으로 가까운 위치에 배치하여 네트워크 지연 최소화  
C. VM 비용 절감  
D. VM 보안 강화

**정답: B**  
**해설:** Proximity Placement Group은 VM들을 동일 데이터센터 내 물리적으로 가까운 위치에 배치하여 네트워크 지연을 최소화합니다.

---

### 문항 79

다음 중 Azure에서 Virtual Machine의 Accelerated Networking에 대한 설명으로 틀린 것은?

A. SR-IOV(Single Root I/O Virtualization)를 사용한다.  
B. 네트워크 처리량과 지연 시간을 개선한다.  
C. 모든 VM 크기에서 지원된다.  
D. 추가 비용 없이 사용 가능하다.

**정답: C**  
**해설:** Accelerated Networking은 특정 VM 크기(주로 4 vCPU 이상)에서만 지원됩니다.

---

### 문항 80

다음 중 Azure에서 VM에 대한 Just-In-Time(JIT) 액세스의 동작 방식은?

A. VM에 영구적으로 관리 포트를 열어둔다.  
B. 요청 시 지정된 시간 동안만 관리 포트를 열고 이후 자동으로 닫는다.  
C. VM을 필요 시에만 시작한다.  
D. VM의 성능을 일시적으로 향상시킨다.

**정답: B**  
**해설:** JIT는 NSG 규칙을 자동으로 관리하여 요청된 시간 동안만 특정 IP에서 관리 포트(SSH/RDP)에 접근을 허용합니다.

---

## 4. Azure 리소스별 특장점, 제약사항, 주의사항 (25문항)

### 문항 81

박대리는 Azure SQL Database의 Serverless 계층을 검토하고 있다. 다음 중 Serverless 계층의 특징으로 틀린 것은?

A. 사용하지 않을 때 자동으로 일시 중지된다.  
B. vCore당 초 단위로 과금된다.  
C. 자동 일시 중지 지연 시간을 구성할 수 있다.  
D. 항상 최대 vCore로 실행된다.

**정답: D**  
**해설:** Serverless는 워크로드에 따라 최소~최대 vCore 범위 내에서 자동으로 스케일링되며, 항상 최대가 아닙니다.

---

### 문항 82

김과장은 Azure Application Gateway에 WAF를 구성하려고 한다. 다음 중 WAF Policy의 모드가 아닌 것은?

A. Detection  
B. Prevention  
C. Monitoring  
D. (Detection과 Prevention만 존재)

**정답: C**  
**해설:** WAF는 Detection(감지 후 로깅만)과 Prevention(감지 후 차단) 두 가지 모드만 제공합니다.

---

### 문항 83

이선임은 Azure Cosmos DB의 파티셔닝 전략을 설계하고 있다. 다음 중 파티션 키 선택 시 고려사항으로 틀린 것은?

A. 높은 카디널리티(다양한 값)를 가진 속성 선택  
B. 읽기 및 쓰기가 균등하게 분산되는 속성 선택  
C. 파티션 키는 생성 후 변경 가능  
D. 자주 필터 조건으로 사용되는 속성 선택

**정답: C**  
**해설:** Cosmos DB의 파티션 키는 컨테이너 생성 후 변경할 수 없으므로, 초기 설계가 매우 중요합니다.

---

### 문항 84

송팀장은 Azure Event Hub의 처리량 단위(Throughput Unit)를 조정하고 있다. 다음 중 1 TU의 처리 용량으로 맞는 것은?

A. 수신: 1 MB/s, 송신: 2 MB/s  
B. 수신: 2 MB/s, 송신: 1 MB/s  
C. 수신: 1 MB/s, 송신: 1 MB/s  
D. 수신: 5 MB/s, 송신: 5 MB/s

**정답: A**  
**해설:** Event Hub의 1 Throughput Unit은 수신 1 MB/s (또는 1000 이벤트/s), 송신 2 MB/s를 제공합니다.

---

### 문항 85

장전임은 Azure Key Vault의 Soft Delete와 Purge Protection을 구성하고 있다. 다음 중 Purge Protection에 대한 설명으로 맞는 것은?

A. 삭제된 키/비밀을 즉시 영구 삭제할 수 있다.  
B. 보존 기간 동안 영구 삭제를 방지한다.  
C. Soft Delete 없이 단독으로 활성화 가능하다.  
D. 기본적으로 비활성화되어 있으며 활성화 후 다시 비활성화 가능하다.

**정답: B**  
**해설:** Purge Protection이 활성화되면 보존 기간 동안 관리자도 영구 삭제할 수 없습니다. 활성화 후 비활성화가 불가능합니다.

---

### 문항 86

박대리는 Azure Load Balancer의 Health Probe를 구성하고 있다. 다음 중 Health Probe 유형이 아닌 것은?

A. TCP  
B. HTTP  
C. HTTPS  
D. ICMP

**정답: D**  
**해설:** Azure Load Balancer는 TCP, HTTP, HTTPS 프로브를 지원하며, ICMP(Ping)는 지원하지 않습니다.

---

### 문항 87

김과장은 Azure Functions에서 Durable Functions를 사용하고 있다. 다음 중 Durable Functions의 패턴이 아닌 것은?

A. Function Chaining  
B. Fan-out/Fan-in  
C. Async HTTP API  
D. Direct Database Access

**정답: D**  
**해설:** Durable Functions의 주요 패턴은 Function Chaining, Fan-out/Fan-in, Async HTTP API, Monitor, Human Interaction입니다.

---

### 문항 88

이선임은 Azure Container Apps를 검토하고 있다. 다음 중 Container Apps의 제약사항은?

A. HTTP 트래픽만 수신 가능  
B. 사용자 정의 도메인 불가  
C. Windows 컨테이너 미지원  
D. 환경 변수 설정 불가

**정답: C**  
**해설:** Azure Container Apps는 현재 Linux 컨테이너만 지원하며, Windows 컨테이너는 지원하지 않습니다.

---

### 문항 89

송팀장은 Azure Front Door의 Rules Engine을 구성하고 있다. 다음 중 Rules Engine으로 수행할 수 없는 작업은?

A. URL 리다이렉트  
B. 요청/응답 헤더 수정  
C. 백엔드 데이터베이스 쿼리 실행  
D. 캐시 동작 재정의

**정답: C**  
**해설:** Rules Engine은 트래픽 라우팅과 헤더 수정 등 네트워크 수준 규칙을 처리하며, 데이터베이스 접근은 범위 밖입니다.

---

### 문항 90

장전임은 Azure Monitor의 Action Group을 구성하고 있다. 다음 중 Action Group에서 지원하지 않는 알림 유형은?

A. 이메일  
B. SMS  
C. 팩스  
D. Webhook

**정답: C**  
**해설:** Action Group은 이메일, SMS, Push 알림, Voice, Webhook, Logic App, Azure Function, ITSM 등을 지원하며, 팩스는 지원하지 않습니다.

---

### 문항 91

박대리는 Azure App Service의 VNET 통합을 구성하고 있다. 다음 중 Regional VNET Integration의 제약사항은?

A. 동일 리전의 VNET에만 연결 가능  
B. 인바운드 트래픽 제어 가능  
C. Free 계층에서도 사용 가능  
D. NSG와 UDR을 사용할 수 없다.

**정답: A**  
**해설:** Regional VNET Integration은 동일 리전의 VNET에만 연결 가능합니다. 다른 리전은 Gateway Required VNET Integration이 필요합니다.

---

### 문항 92

김과장은 Azure Cosmos DB의 Request Unit(RU)에 대해 검토하고 있다. 다음 중 RU 소비에 영향을 주지 않는 것은?

A. 문서 크기  
B. 인덱싱 정책  
C. 쿼리 복잡도  
D. 컨테이너 이름 길이

**정답: D**  
**해설:** 컨테이너 이름 길이는 RU 소비에 영향을 미치지 않습니다. 문서 크기, 인덱싱, 쿼리 복잡도, 일관성 수준이 영향을 줍니다.

---

### 문항 93

이선임은 Azure Virtual Machine의 Availability Zone 제약사항을 검토하고 있다. 다음 중 맞는 설명은?

A. 모든 VM 크기가 모든 Availability Zone에서 사용 가능하다.  
B. VM 생성 후 Availability Zone을 변경할 수 있다.  
C. Availability Zone 간 데이터 전송 비용이 발생할 수 있다.  
D. Availability Zone은 모든 Azure 리전에서 제공된다.

**정답: C**  
**해설:** Availability Zone 간 데이터 전송에는 소량의 네트워크 비용이 발생합니다. VM 생성 후 Zone 변경은 불가능합니다.

---

### 문항 94

송팀장은 Azure Service Bus의 Premium 계층을 검토하고 있다. 다음 중 Premium 계층에서만 제공되는 기능은?

A. 큐 기능  
B. 토픽/구독  
C. 메시지 전달  
D. VNET Service Endpoint 및 Private Link

**정답: D**  
**해설:** VNET 통합(Service Endpoint, Private Link)과 전용 리소스는 Service Bus Premium에서만 지원됩니다.

---

### 문항 95

장전임은 Azure Blob Storage의 Access Tier 변경에 대해 검토하고 있다. 다음 중 주의사항으로 맞는 것은?

A. Hot에서 Cool로 이동 시 비용이 발생하지 않는다.  
B. Cool/Archive 계층은 조기 삭제 수수료가 있다.  
C. Archive에서 Hot으로 즉시 전환된다.  
D. 계층 변경은 하루에 한 번만 가능하다.

**정답: B**  
**해설:** Cool 계층은 30일, Archive 계층은 180일의 최소 보존 기간이 있으며, 조기 삭제/이동 시 수수료가 부과됩니다.

---

### 문항 96

박대리는 Azure VPN Gateway의 Site-to-Site 연결을 구성하고 있다. 다음 중 필수 온프레미스 요구사항이 아닌 것은?

A. 공용 IPv4 주소를 가진 VPN 디바이스  
B. BGP 지원 (선택사항이지만 권장)  
C. Azure와 동일한 OS  
D. IKEv2 또는 IKEv1 지원

**정답: C**  
**해설:** VPN 디바이스의 OS는 Azure와 동일할 필요가 없으며, IPsec/IKE 호환 VPN 디바이스면 됩니다.

---

### 문항 97

김과장은 Azure SQL Database의 Failover Group을 구성하고 있다. 다음 중 Failover Group의 특징으로 틀린 것은?

A. 자동 장애 조치를 지원한다.  
B. 읽기 전용 리스너 엔드포인트를 제공한다.  
C. 동일 리전 내 서버 간에만 구성 가능하다.  
D. Grace Period를 구성할 수 있다.

**정답: C**  
**해설:** Failover Group은 서로 다른 리전의 서버 간에 구성하여 지역 재해 복구를 제공합니다.

---

### 문항 98

이선임은 Azure Container Registry의 Tasks를 사용하고 있다. 다음 중 ACR Tasks의 트리거가 아닌 것은?

A. 소스 코드 커밋  
B. 베이스 이미지 업데이트  
C. 타이머 스케줄  
D. VM CPU 사용률

**정답: D**  
**해설:** ACR Tasks는 코드 커밋, 베이스 이미지 업데이트, 타이머로 트리거되며, VM 메트릭은 트리거가 아닙니다.

---

### 문항 99

송팀장은 Azure Application Insights의 Sampling을 구성하고 있다. 다음 중 Sampling의 목적은?

A. 데이터 정확도 향상  
B. 텔레메트리 데이터 양과 비용 절감  
C. 응답 시간 단축  
D. 보안 강화

**정답: B**  
**해설:** Sampling은 수집되는 텔레메트리 데이터의 양을 줄여 비용을 절감하면서도 통계적으로 유의미한 분석이 가능하게 합니다.

---

### 문항 100

장전임은 Azure Virtual Network의 서브넷에 대한 제약사항을 검토하고 있다. 다음 중 맞는 설명은?

A. 서브넷은 생성 후 주소 범위를 변경할 수 없다.  
B. 리소스가 연결된 서브넷의 주소 범위는 확장만 가능하다.  
C. 서브넷 간 주소 범위가 겹칠 수 있다.  
D. 서브넷 삭제 시 연결된 리소스도 자동 삭제된다.

**정답: B**  
**해설:** 리소스가 연결된 서브넷은 주소 범위를 확장할 수 있지만, 축소하려면 리소스를 먼저 제거해야 합니다. 서브넷 간 주소 범위는 겹칠 수 없습니다.

---

### 문항 101

박대리는 Azure Managed Disk의 Bursting 기능을 검토하고 있다. 다음 중 On-demand Bursting과 Credit-based Bursting의 차이점으로 맞는 것은?

A. Credit-based는 Premium SSD 512GB 이상에서만 사용 가능하다.  
B. On-demand Bursting은 추가 비용 없이 제공된다.  
C. Credit-based Bursting은 유휴 시간에 크레딧이 축적된다.  
D. 두 방식 모두 Ultra Disk에서만 지원된다.

**정답: C**  
**해설:** Credit-based Bursting은 유휴 시간에 I/O 크레딧이 축적되어 필요 시 버스트에 사용됩니다. On-demand는 P20 이상에서 추가 비용으로 제공됩니다.

---

### 문항 102

김과장은 Azure API Management의 Rate Limiting 정책을 구성하고 있다. 다음 중 rate-limit과 rate-limit-by-key의 차이점으로 맞는 것은?

A. rate-limit은 개별 키별로 제한한다.  
B. rate-limit-by-key는 전체 API에 대한 전역 제한이다.  
C. rate-limit-by-key는 사용자, IP 등 특정 키별로 제한할 수 있다.  
D. 두 정책은 동일하다.

**정답: C**  
**해설:** rate-limit은 전체 구독/API에 대한 전역 제한이고, rate-limit-by-key는 사용자, IP, 헤더 값 등 특정 키별로 세분화된 제한이 가능합니다.

---

### 문항 103

이선임은 Azure Data Lake Storage Gen2의 ACL(Access Control List)을 구성하고 있다. 다음 중 POSIX ACL에 대한 설명으로 틀린 것은?

A. 파일과 디렉터리에 개별 권한을 설정할 수 있다.  
B. Access ACL과 Default ACL이 있다.  
C. ACL은 RBAC보다 항상 우선한다.  
D. Owner, Owning Group, Other에 대해 권한 설정이 가능하다.

**정답: C**  
**해설:** RBAC과 ACL은 함께 평가되며, RBAC이 허용하면 ACL은 무시됩니다. RBAC이 더 넓은 범위의 권한을 제어합니다.

---

### 문항 104

송팀장은 Azure ExpressRoute의 Circuit SKU를 선택하고 있다. 다음 중 Local SKU의 특징으로 맞는 것은?

A. 전역 모든 리전에 연결 가능  
B. 동일 Metro 내의 Azure 리전에만 연결 가능하며 Egress 무료  
C. Premium 기능이 모두 포함  
D. 가장 비용이 높은 SKU

**정답: B**  
**해설:** ExpressRoute Local은 피어링 위치와 동일 Metro의 Azure 리전에만 연결되지만, 데이터 전송(Egress)이 무료라는 비용 이점이 있습니다.

---

### 문항 105

장전임은 Azure Cosmos DB의 Consistency Level 변경에 대해 검토하고 있다. 다음 중 맞는 설명은?

A. 일관성 수준은 계정 수준에서만 설정 가능하다.  
B. 요청별로 계정 기본값보다 더 강한 일관성으로 변경 가능하다.  
C. 요청별로 계정 기본값보다 더 약한 일관성으로 변경 가능하다.  
D. 일관성 수준은 변경할 수 없다.

**정답: C**  
**해설:** Cosmos DB는 계정 수준 기본 일관성을 설정하고, 개별 요청에서 더 약한 수준으로만 변경 가능합니다. 더 강한 수준으로는 변경할 수 없습니다.

---

## 5. Azure RBAC, Security, AI 기능 (25문항)

### 문항 106

박대리는 Microsoft Entra ID(구 Azure AD)의 Conditional Access에서 Named Location을 구성하고 있다. 다음 중 Named Location의 유형이 아닌 것은?

A. IP 범위 기반  
B. 국가/지역 기반  
C. GPS 기반  
D. 호스트 이름 기반

**정답: D**  
**해설:** Named Location은 IP 범위, 국가/지역, GPS 좌표(Compliant Network)를 지원하며, 호스트 이름 기반은 지원하지 않습니다.

---

### 문항 107

김과장은 Azure RBAC에서 Deny Assignment에 대해 검토하고 있다. 다음 중 Deny Assignment의 특징으로 맞는 것은?

A. 사용자가 직접 Deny Assignment를 생성할 수 있다.  
B. Deny Assignment는 Allow 할당보다 우선한다.  
C. Deny Assignment는 Azure Blueprint에서만 생성된다.  
D. Deny Assignment는 Owner 역할에 영향을 주지 않는다.

**정답: B**  
**해설:** Deny Assignment는 Allow보다 우선하여 특정 작업을 명시적으로 차단합니다. Azure Blueprint나 관리형 앱에서 생성되며, 사용자가 직접 생성할 수는 없습니다.

---

### 문항 108

이선임은 Microsoft Defender for Cloud의 CSPM(Cloud Security Posture Management) 기능을 검토하고 있다. 다음 중 CSPM 기능이 아닌 것은?

A. 보안 점수 계산  
B. 규정 준수 대시보드  
C. 공격 경로 분석  
D. 자동 코드 배포

**정답: D**  
**해설:** CSPM은 보안 평가, 권장사항, 규정 준수 모니터링, 공격 경로 분석을 제공하며, 코드 배포는 DevOps 영역입니다.

---

### 문항 109

송팀장은 Azure Key Vault의 Access Policy와 RBAC 모델을 비교하고 있다. 다음 중 RBAC 모델의 이점이 아닌 것은?

A. Azure 리소스와 동일한 권한 관리 체계  
B. 더 세분화된 권한 제어  
C. Conditional Access 통합  
D. Key Vault 수준에서만 권한 설정 가능

**정답: D**  
**해설:** RBAC 모델은 개별 키, 비밀, 인증서 수준까지 세분화된 권한 설정이 가능합니다. Access Policy는 Key Vault 수준에서만 설정됩니다.

---

### 문항 110

장전임은 Azure OpenAI Service를 구성하려고 한다. 다음 중 Azure OpenAI의 특징으로 틀린 것은?

A. 데이터가 다른 고객과 공유되지 않는다.  
B. Microsoft의 엔터프라이즈 보안 및 규정 준수 적용  
C. 모든 Azure 리전에서 사용 가능하다.  
D. Managed Identity를 통한 인증 지원

**정답: C**  
**해설:** Azure OpenAI Service는 선택된 리전에서만 사용 가능하며, 모든 모델이 모든 리전에서 제공되지는 않습니다.

---

### 문항 111

박대리는 Microsoft Entra ID Protection을 구성하고 있다. 다음 중 ID Protection이 감지하지 않는 위험은?

A. 유출된 자격 증명  
B. 익명 IP 주소 사용  
C. 비정상적 이동 (Atypical Travel)  
D. 하드웨어 고장

**정답: D**  
**해설:** ID Protection은 ID 관련 위험(유출 자격 증명, 익명 IP, 비정상 이동, 악성 IP 등)을 감지하며, 하드웨어 관련은 범위 밖입니다.

---

### 문항 112

김과장은 Azure RBAC에서 사용자 지정 역할의 범위를 설정하고 있다. 다음 중 AssignableScopes로 지정할 수 없는 것은?

A. Management Group  
B. Subscription  
C. Resource Group  
D. 개별 리소스 (예: 특정 VM)

**정답: D**  
**해설:** Custom Role의 AssignableScopes는 Management Group, Subscription, Resource Group 수준까지만 지정 가능하며, 개별 리소스는 불가합니다.

---

### 문항 113

이선임은 Azure AI Search(구 Cognitive Search)를 구성하고 있다. 다음 중 AI Search의 기능이 아닌 것은?

A. 전체 텍스트 검색  
B. AI 기반 인덱싱 (Skillset)  
C. Semantic Search  
D. 실시간 데이터베이스 동기화

**정답: D**  
**해설:** AI Search는 인덱서를 통해 주기적으로 데이터를 가져오며, 실시간 동기화는 제공하지 않습니다. Change Tracking을 통한 점진적 인덱싱은 가능합니다.

---

### 문항 114

송팀장은 Microsoft Sentinel의 Data Connector를 구성하고 있다. 다음 중 Sentinel에 연결할 수 없는 데이터 소스는?

A. Microsoft 365  
B. Azure Activity Log  
C. AWS CloudTrail  
D. 물리적 종이 문서

**정답: D**  
**해설:** Sentinel은 디지털 로그 및 이벤트 데이터만 수집하며, 물리적 문서는 범위 밖입니다.

---

### 문항 115

장전임은 Azure AI Document Intelligence(구 Form Recognizer)를 사용하려고 한다. 다음 중 지원되는 기능이 아닌 것은?

A. 영수증 데이터 추출  
B. 신분증 정보 추출  
C. 레이아웃 분석  
D. 실시간 음성 인식

**정답: D**  
**해설:** Document Intelligence는 문서/이미지에서 텍스트와 구조를 추출하는 서비스이며, 음성 인식은 Azure Speech Service의 영역입니다.

---

### 문항 116

박대리는 Azure에서 Microsoft Entra Privileged Identity Management(PIM)의 Access Review를 구성하고 있다. 다음 중 Access Review의 대상이 아닌 것은?

A. Azure AD 역할 할당  
B. Azure 리소스 역할 할당  
C. 그룹 멤버십  
D. Storage Account의 데이터 내용

**정답: D**  
**해설:** Access Review는 역할 할당과 그룹 멤버십을 검토하는 것이며, 데이터 내용 검토는 범위 밖입니다.

---

### 문항 117

김과장은 Azure OpenAI Service에서 RAG(Retrieval-Augmented Generation) 패턴을 구현하려고 한다. 다음 중 RAG 구현에 필요한 핵심 구성 요소가 아닌 것은?

A. Azure AI Search (검색 인덱스)  
B. Azure OpenAI (언어 모델)  
C. 데이터 소스 (Blob Storage, SQL 등)  
D. Azure DevOps Pipeline

**정답: D**  
**해설:** RAG는 데이터 소스 → 검색 인덱스(AI Search) → 언어 모델(OpenAI)로 구성되며, CI/CD 파이프라인은 배포용으로 RAG 자체의 핵심 요소는 아닙니다.

---

### 문항 118

이선임은 Azure의 DDoS Protection에서 Standard와 Basic의 차이를 검토하고 있다. 다음 중 Basic에서 제공되지 않는 기능은?

A. 항상 활성화된 트래픽 모니터링  
B. 자동 공격 완화  
C. 맞춤형 DDoS 정책 튜닝  
D. 네트워크 계층 보호

**정답: C**  
**해설:** Basic은 기본적인 인프라 수준 보호만 제공하며, 맞춤형 정책 튜닝, 공격 메트릭, 비용 보호 등은 Standard에서만 지원됩니다.

---

### 문항 119

송팀장은 Azure에서 Customer Lockbox를 구성하려고 한다. 다음 중 Customer Lockbox의 용도는?

A. 고객 데이터 암호화  
B. Microsoft 지원 엔지니어의 고객 데이터 접근 시 명시적 승인 요구  
C. VM 자동 잠금  
D. 네트워크 트래픽 차단

**정답: B**  
**해설:** Customer Lockbox는 Microsoft 지원 요청 처리 중 엔지니어가 고객 데이터에 접근해야 할 때 고객의 명시적 승인을 요구합니다.

---

### 문항 120

장전임은 Azure AI Content Safety를 검토하고 있다. 다음 중 Content Safety가 감지하는 범주가 아닌 것은?

A. 혐오 콘텐츠  
B. 성적 콘텐츠  
C. 폭력  
D. 저작권 침해

**정답: D**  
**해설:** Azure AI Content Safety는 혐오, 성적 콘텐츠, 폭력, 자해 관련 콘텐츠를 감지합니다. 저작권 침해는 별도의 법적 영역입니다.

---

### 문항 121

박대리는 Microsoft Entra ID의 B2C(Business to Consumer) 테넌트를 구성하고 있다. 다음 중 B2C의 기능이 아닌 것은?

A. 소셜 로그인 (Google, Facebook 등)  
B. 사용자 흐름(User Flow) 커스터마이징  
C. 조직 내부 직원 관리  
D. 다중 테넌트 앱 등록

**정답: C**  
**해설:** B2C는 외부 소비자/고객 대상 ID 관리 서비스이며, 내부 직원 관리는 일반 Entra ID(구 Azure AD)의 역할입니다.

---

### 문항 122

김과장은 Azure에서 Microsoft Copilot for Azure를 사용하고 있다. 다음 중 Copilot for Azure가 도움을 줄 수 있는 작업이 아닌 것은?

A. 리소스 정보 조회  
B. KQL 쿼리 생성 지원  
C. 비용 분석 및 최적화 권장  
D. 물리적 서버 하드웨어 교체

**정답: D**  
**해설:** Copilot for Azure는 Azure Portal에서 AI 기반 지원을 제공하며, 물리적 하드웨어 관리는 범위 밖입니다.

---

### 문항 123

이선임은 Azure에서 Managed HSM(Hardware Security Module)을 검토하고 있다. 다음 중 Managed HSM의 특징으로 틀린 것은?

A. FIPS 140-2 Level 3 인증  
B. 단일 테넌트 HSM  
C. Key Vault Standard와 동일한 가격  
D. 고객만 HSM에 대한 관리 권한 보유

**정답: C**  
**해설:** Managed HSM은 전용 하드웨어를 사용하므로 Key Vault Standard보다 비용이 상당히 높습니다.

---

### 문항 124

송팀장은 Azure에서 Microsoft Entra Workload ID를 구성하고 있다. 다음 중 Workload ID의 대상이 아닌 것은?

A. 애플리케이션  
B. 서비스 주체  
C. 관리 ID  
D. 최종 사용자

**정답: D**  
**해설:** Workload ID는 애플리케이션, 서비스 주체, 관리 ID 등 비인간 ID를 관리하며, 최종 사용자 ID는 별도로 관리됩니다.

---

### 문항 125

장전임은 Azure AI Studio를 사용하여 AI 프로젝트를 구성하고 있다. 다음 중 AI Studio의 기능이 아닌 것은?

A. 모델 카탈로그 탐색  
B. 프롬프트 플로우 설계  
C. 모델 평가 및 벤치마킹  
D. 하드웨어 GPU 직접 구매

**정답: D**  
**해설:** AI Studio는 AI 모델 개발, 배포, 평가를 위한 플랫폼이며, 하드웨어 구매는 범위 밖입니다.

---

### 문항 126

박대리는 Azure에서 Service Principal과 Managed Identity를 비교하고 있다. 다음 중 Managed Identity의 이점이 아닌 것은?

A. 자격 증명 관리 불필요  
B. 자동 회전  
C. 온프레미스 애플리케이션에서 사용 가능  
D. Azure 서비스와 직접 통합

**정답: C**  
**해설:** Managed Identity는 Azure 리소스에서만 사용 가능하며, 온프레미스 애플리케이션은 Service Principal을 사용해야 합니다.

---

### 문항 127

김과장은 Azure에서 Data Encryption at Rest에 대해 검토하고 있다. 다음 중 맞는 설명은?

A. Azure Storage의 저장 데이터 암호화는 선택 사항이다.  
B. 모든 Azure Storage 데이터는 기본적으로 AES-256으로 암호화된다.  
C. 암호화를 비활성화하면 성능이 향상된다.  
D. 암호화 키는 고객만 관리할 수 있다.

**정답: B**  
**해설:** Azure Storage는 모든 데이터를 기본적으로 AES-256으로 암호화하며, 이를 비활성화할 수 없습니다. Microsoft-managed 또는 Customer-managed 키를 선택할 수 있습니다.

---

### 문항 128

이선임은 Azure에서 Microsoft Entra ID의 Administrative Units를 구성하고 있다. 다음 중 Administrative Unit의 용도는?

A. 네트워크 서브넷 분리  
B. 관리 범위를 특정 사용자/그룹으로 제한하여 위임  
C. VM 크기 제한  
D. Storage 용량 관리

**정답: B**  
**해설:** Administrative Unit은 조직의 부서나 지역별로 관리 범위를 제한하여 Helpdesk 등에 세분화된 관리 권한을 위임할 수 있게 합니다.

---

### 문항 129

송팀장은 Azure AI Service의 Responsible AI 원칙을 검토하고 있다. 다음 중 Microsoft의 Responsible AI 원칙이 아닌 것은?

A. 공정성 (Fairness)  
B. 신뢰성 및 안전 (Reliability & Safety)  
C. 수익 극대화 (Revenue Maximization)  
D. 투명성 (Transparency)

**정답: C**  
**해설:** Microsoft의 Responsible AI 6대 원칙은 공정성, 신뢰성/안전, 개인 정보 보호/보안, 포용성, 투명성, 책임입니다.

---

### 문항 130

장전임은 Azure에서 Network Security Group의 Service Tag를 사용하고 있다. 다음 중 Service Tag의 이점으로 틀린 것은?

A. Microsoft가 IP 범위를 자동으로 관리  
B. 개별 IP를 직접 관리할 필요 없음  
C. 모든 타사 서비스의 IP를 포함  
D. NSG 규칙에서 직접 사용 가능

**정답: C**  
**해설:** Service Tag는 Azure 서비스의 IP 범위만 포함하며, 타사 서비스의 IP는 포함하지 않습니다.

---

## 6. AKS (25문항)

### 문항 131

박대리는 AKS에서 Kubernetes RBAC과 Azure RBAC의 차이를 검토하고 있다. 다음 중 맞는 설명은?

A. Kubernetes RBAC은 Azure 리소스를 관리한다.  
B. Azure RBAC은 Kubernetes 네임스페이스 수준 권한을 제어한다.  
C. Kubernetes RBAC은 클러스터 내부 리소스(Pod, Service 등)를 제어하고, Azure RBAC은 AKS 인프라를 관리한다.  
D. 두 RBAC은 동일하다.

**정답: C**  
**해설:** Kubernetes RBAC은 클러스터 내부 리소스(Pod, Deployment, Service 등)를 제어하고, Azure RBAC은 AKS 클러스터 자체의 관리(생성, 삭제, 스케일링 등)를 담당합니다.

---

### 문항 132

김과장은 AKS에서 Pod 스케줄링을 최적화하려고 한다. 다음 중 특정 Node에 Pod를 배치하기 위한 방법이 아닌 것은?

A. Node Selector  
B. Node Affinity  
C. Taints and Tolerations  
D. Pod Priority Class

**정답: D**  
**해설:** Pod Priority Class는 리소스 부족 시 Pod 축출 우선순위를 결정하며, Node 배치와는 직접 관련이 없습니다.

---

### 문항 133

이선임은 AKS에서 Horizontal Pod Autoscaler의 스케일링 메트릭을 구성하고 있다. 다음 중 HPA가 기본적으로 지원하는 메트릭 유형이 아닌 것은?

A. CPU Utilization  
B. Memory Utilization  
C. Custom Metrics  
D. Node Disk Usage

**정답: D**  
**해설:** HPA는 Pod 수준 메트릭(CPU, Memory)과 Custom/External Metrics를 지원합니다. Node 수준 디스크 사용량은 HPA 메트릭이 아닙니다.

---

### 문항 134

송팀장은 AKS에서 Init Container를 사용하고 있다. 다음 중 Init Container의 특징으로 틀린 것은?

A. 메인 컨테이너가 시작되기 전에 실행된다.  
B. 순차적으로 실행된다.  
C. 메인 컨테이너와 동시에 실행된다.  
D. 설정 파일 준비, 의존성 확인 등에 사용된다.

**정답: C**  
**해설:** Init Container는 메인 컨테이너 전에 순차적으로 실행되며, 모든 Init Container가 완료되어야 메인 컨테이너가 시작됩니다.

---

### 문항 135

장전임은 AKS에서 Kubernetes Service 유형을 선택하고 있다. 다음 중 ClusterIP Service의 특징은?

A. 외부 인터넷에서 직접 접근 가능  
B. 클러스터 내부에서만 접근 가능  
C. 각 Node의 포트로 노출  
D. Azure Load Balancer와 자동 연결

**정답: B**  
**해설:** ClusterIP는 기본 Service 유형으로 클러스터 내부에서만 접근 가능합니다. 외부 접근은 NodePort, LoadBalancer, Ingress를 사용해야 합니다.

---

### 문항 136

박대리는 AKS에서 ConfigMap과 Secret의 차이를 검토하고 있다. 다음 중 Secret에 대한 설명으로 틀린 것은?

A. Base64로 인코딩되어 저장된다.  
B. etcd에 암호화되어 저장된다(encryption at rest).  
C. 환경 변수나 볼륨으로 마운트 가능  
D. 무제한 크기의 데이터를 저장할 수 있다.

**정답: D**  
**해설:** Kubernetes Secret은 1MB 크기 제한이 있습니다. 대용량 비밀 데이터는 Key Vault 등 외부 저장소를 사용해야 합니다.

---

### 문항 137

김과장은 AKS에서 Liveness Probe와 Readiness Probe의 차이를 검토하고 있다. 다음 중 맞는 설명은?

A. Liveness Probe 실패 시 Pod가 Service에서 제거된다.  
B. Readiness Probe 실패 시 컨테이너가 재시작된다.  
C. Liveness Probe 실패 시 컨테이너가 재시작되고, Readiness Probe 실패 시 Pod가 Service Endpoint에서 제거된다.  
D. 두 Probe는 동일한 동작을 한다.

**정답: C**  
**해설:** Liveness Probe 실패 → 컨테이너 재시작, Readiness Probe 실패 → Service Endpoint에서 제거(트래픽 중단). Startup Probe는 초기화가 완료될 때까지 다른 프로브를 비활성화합니다.

---

### 문항 138

이선임은 AKS에서 Network Policy를 구성하고 있다. 다음 중 기본 동작으로 맞는 것은?

A. 모든 Pod 간 트래픽이 차단된다.  
B. 모든 Pod 간 트래픽이 허용된다.  
C. 같은 Namespace의 Pod만 통신 가능하다.  
D. 외부 트래픽만 차단된다.

**정답: B**  
**해설:** Network Policy가 없으면 모든 Pod 간 트래픽이 기본적으로 허용됩니다. Network Policy를 적용하면 해당 Pod에 대해 명시적으로 허용된 트래픽만 통과합니다.

---

### 문항 139

송팀장은 AKS에서 DaemonSet을 사용하고 있다. 다음 중 DaemonSet의 일반적인 사용 사례가 아닌 것은?

A. 로그 수집 에이전트  
B. 모니터링 에이전트  
C. 웹 API 서버  
D. 네트워크 플러그인

**정답: C**  
**해설:** DaemonSet은 모든 Node에서 하나씩 실행되어야 하는 시스템 수준 Pod에 사용됩니다. 웹 API는 Deployment로 배포합니다.

---

### 문항 140

장전임은 AKS에서 Ingress와 LoadBalancer Service의 차이를 검토하고 있다. 다음 중 Ingress의 이점은?

A. TCP/UDP 수준 부하 분산  
B. 여러 서비스를 하나의 IP로 노출하고 URL 경로 기반 라우팅  
C. 자동 DNS 등록  
D. DDoS 방어

**정답: B**  
**해설:** Ingress는 Layer 7에서 호스트/경로 기반 라우팅을 제공하여 여러 서비스를 하나의 IP/도메인으로 노출할 수 있습니다.

---

### 문항 141

박대리는 AKS에서 StatefulSet을 사용하려고 한다. 다음 중 StatefulSet이 Deployment와 다른 점은?

A. StatefulSet의 Pod는 고유한 이름과 안정적인 네트워크 ID를 가진다.  
B. StatefulSet은 Rolling Update를 지원하지 않는다.  
C. StatefulSet은 ReplicaSet을 사용한다.  
D. Deployment보다 항상 성능이 좋다.

**정답: A**  
**해설:** StatefulSet의 Pod는 pod-0, pod-1 등 순서가 있는 고유 이름과 안정적인 네트워크 ID, Persistent Volume 바인딩을 가집니다.

---

### 문항 142

김과장은 AKS에서 Kubernetes Namespace의 용도를 검토하고 있다. 다음 중 Namespace에 대한 설명으로 틀린 것은?

A. 리소스를 논리적으로 분리한다.  
B. Resource Quota를 Namespace별로 설정할 수 있다.  
C. Network Policy를 Namespace 단위로 적용할 수 있다.  
D. Namespace 간 완전한 네트워크 격리를 기본 제공한다.

**정답: D**  
**해설:** Namespace는 논리적 분리만 제공하며, 네트워크 격리는 Network Policy를 별도로 구성해야 합니다.

---

### 문항 143

이선임은 AKS에서 Container Storage Interface(CSI) 드라이버를 사용하고 있다. 다음 중 AKS에서 기본 지원되는 CSI 드라이버가 아닌 것은?

A. Azure Disk CSI Driver  
B. Azure Files CSI Driver  
C. Azure Blob CSI Driver  
D. AWS EBS CSI Driver

**정답: D**  
**해설:** AKS는 Azure Disk, Azure Files, Azure Blob CSI 드라이버를 기본 지원합니다. AWS EBS는 EKS용입니다.

---

### 문항 144

송팀장은 AKS에서 Pod Anti-Affinity를 구성하고 있다. 다음 중 Pod Anti-Affinity의 용도는?

A. 동일 Label의 Pod를 같은 Node에 배치  
B. 동일 Label의 Pod를 서로 다른 Node에 분산 배치  
C. Pod의 CPU 제한 설정  
D. Pod의 네트워크 대역폭 제한

**정답: B**  
**해설:** Pod Anti-Affinity는 동일 서비스의 Pod를 여러 Node에 분산하여 단일 Node 장애 시 전체 서비스 중단을 방지합니다.

---

### 문항 145

장전임은 AKS에서 Horizontal Pod Autoscaler와 Vertical Pod Autoscaler의 차이를 검토하고 있다. 다음 중 맞는 설명은?

A. HPA는 Pod의 리소스 요청/제한을 조정한다.  
B. VPA는 Pod 수를 조정한다.  
C. HPA는 Pod 수를 조정하고, VPA는 Pod의 CPU/메모리 요청/제한을 조정한다.  
D. HPA와 VPA는 항상 함께 사용해야 한다.

**정답: C**  
**해설:** HPA는 수평 스케일링(Pod 수 조정), VPA는 수직 스케일링(Pod 리소스 조정)입니다. 같은 메트릭에 대해 동시 사용은 권장되지 않습니다.

---

### 문항 146

박대리는 AKS에서 Kubernetes Job과 CronJob을 비교하고 있다. 다음 중 CronJob의 특징으로 틀린 것은?

A. 스케줄에 따라 주기적으로 Job 생성  
B. Cron 표현식으로 스케줄 정의  
C. 실패 시 자동으로 무한 재시도  
D. 동시성 정책을 설정할 수 있다.

**정답: C**  
**해설:** CronJob의 재시도 횟수는 backoffLimit으로 제한되며, 무한 재시도는 기본 동작이 아닙니다.

---

### 문항 147

김과장은 AKS에서 CoreDNS를 커스터마이징하려고 한다. 다음 중 CoreDNS ConfigMap 수정 시 주의사항으로 맞는 것은?

A. CoreDNS는 User Node Pool에서 실행된다.  
B. ConfigMap 수정 후 CoreDNS Pod를 반드시 수동으로 재시작해야 한다.  
C. 잘못된 구성은 전체 클러스터의 DNS 확인을 중단시킬 수 있다.  
D. CoreDNS는 사용자가 삭제할 수 있다.

**정답: C**  
**해설:** CoreDNS는 클러스터의 핵심 DNS 서비스이므로, 잘못된 구성은 모든 Pod의 이름 확인을 중단시킬 수 있습니다. System Node Pool에서 실행됩니다.

---

### 문항 148

이선임은 AKS에서 Node 수준 보안을 강화하려고 한다. 다음 중 권장 보안 설정이 아닌 것은?

A. SSH 접근 제한  
B. Node 이미지 자동 업그레이드  
C. 모든 Pod에 root 권한 부여  
D. Kubernetes API Server 접근 제한

**정답: C**  
**해설:** Pod에 root 권한을 부여하는 것은 보안 위험입니다. 최소 권한 원칙에 따라 non-root로 실행해야 합니다.

---

### 문항 149

송팀장은 AKS에서 Kubernetes의 Resource Request와 Limit의 차이를 설명하고 있다. 다음 중 맞는 설명은?

A. Request는 Pod가 보장받는 최소 리소스이고, Limit는 사용 가능한 최대 리소스이다.  
B. Request와 Limit는 동일한 값이어야 한다.  
C. Limit만 설정하면 Request는 자동으로 0이 된다.  
D. Request가 Limit보다 클 수 있다.

**정답: A**  
**해설:** Request는 스케줄링 시 보장되는 최소 리소스, Limit는 사용 가능한 최대 리소스입니다. Request는 Limit 이하여야 합니다.

---

### 문항 150

장전임은 AKS에서 Azure CNI Overlay 네트워킹 모드를 검토하고 있다. 다음 중 CNI Overlay의 이점은?

A. 각 Pod에 VNET IP를 할당  
B. IP 주소 소비를 줄이면서 Azure CNI의 성능 유지  
C. Kubenet보다 낮은 네트워크 성능  
D. Network Policy 미지원

**정답: B**  
**해설:** Azure CNI Overlay는 Pod에 오버레이 네트워크 IP를 할당하여 VNET IP 소비를 줄이면서도 Azure CNI 수준의 네트워크 성능을 제공합니다.

---

### 문항 151

박대리는 AKS에서 Kubernetes의 Topology Spread Constraints를 구성하고 있다. 다음 중 이 기능의 용도는?

A. Pod를 특정 Node에만 고정  
B. Pod를 Availability Zone, Node 등에 균등하게 분산  
C. Node의 CPU 용량 증가  
D. 네트워크 대역폭 분산

**정답: B**  
**해설:** Topology Spread Constraints는 Pod를 Zone, Node, Region 등의 토폴로지 도메인에 균등하게 분산하여 고가용성을 확보합니다.

---

### 문항 152

김과장은 AKS에서 Kubernetes의 StorageClass를 구성하고 있다. 다음 중 StorageClass의 reclaimPolicy로 유효하지 않은 값은?

A. Retain  
B. Delete  
C. Recycle  
D. Archive

**정답: D**  
**해설:** reclaimPolicy는 Retain(보존), Delete(삭제), Recycle(재사용, 비권장)만 유효합니다. Archive는 존재하지 않습니다.

---

### 문항 153

이선임은 AKS에서 Istio Service Mesh의 핵심 기능을 검토하고 있다. 다음 중 Istio가 제공하지 않는 기능은?

A. 트래픽 관리 (Traffic Management)  
B. mTLS 기반 서비스 간 보안  
C. 분산 추적 (Distributed Tracing)  
D. 데이터베이스 자동 백업

**정답: D**  
**해설:** Istio는 트래픽 관리, 보안(mTLS), 관찰성(분산 추적, 메트릭)을 제공하며, 데이터베이스 관리는 범위 밖입니다.

---

### 문항 154

송팀장은 AKS에서 Pod의 Quality of Service(QoS) 클래스를 검토하고 있다. 다음 중 Guaranteed QoS를 받기 위한 조건은?

A. Request만 설정  
B. Limit만 설정  
C. Request와 Limit이 모든 컨테이너에서 동일한 값으로 설정  
D. 아무 설정도 하지 않음

**정답: C**  
**해설:** Guaranteed QoS는 모든 컨테이너의 CPU/Memory Request = Limit일 때 적용됩니다. 리소스 압박 시 가장 마지막에 축출됩니다.

---

### 문항 155

장전임은 AKS에서 kubectl drain 명령어를 실행하고 있다. 다음 중 drain 시 발생하는 동작으로 틀린 것은?

A. Node에 Cordon(스케줄링 차단)이 적용된다.  
B. 기존 Pod가 안전하게 퇴거(Evict)된다.  
C. DaemonSet Pod도 퇴거된다.  
D. PodDisruptionBudget이 존중된다.

**정답: C**  
**해설:** drain은 기본적으로 DaemonSet Pod를 무시합니다. --ignore-daemonsets 플래그 없이는 DaemonSet Pod 때문에 drain이 차단될 수 있습니다.

---

## 7. 오픈소스 연동 (25문항)

### 문항 156

박대리는 Azure에서 Terraform을 사용하여 인프라를 관리하고 있다. 다음 중 Terraform의 핵심 워크플로우 순서로 맞는 것은?

A. Plan → Init → Apply  
B. Init → Plan → Apply  
C. Apply → Plan → Init  
D. Init → Apply → Plan

**정답: B**  
**해설:** Terraform 워크플로우는 Init(초기화) → Plan(변경 사항 미리 보기) → Apply(적용) 순서입니다.

---

### 문항 157

김과장은 Azure에서 Ansible을 사용하여 구성 관리를 수행하고 있다. 다음 중 Ansible의 특징으로 틀린 것은?

A. 에이전트리스(Agentless) 아키텍처  
B. YAML 기반 Playbook  
C. 반드시 Azure VM에서 실행해야 한다.  
D. 멱등성(Idempotent) 지원

**정답: C**  
**해설:** Ansible은 로컬, 온프레미스, 클라우드 어디서든 실행 가능하며, SSH를 통해 대상 호스트를 관리합니다.

---

### 문항 158

이선임은 Azure에서 HashiCorp Vault를 Azure Key Vault 대신 사용하려고 한다. 다음 중 HashiCorp Vault의 이점이 아닌 것은?

A. 멀티 클라우드 지원  
B. 동적 비밀 생성  
C. Azure 네이티브 통합  
D. 다양한 시크릿 엔진

**정답: C**  
**해설:** Azure 네이티브 통합은 Azure Key Vault의 강점입니다. HashiCorp Vault는 멀티 클라우드와 온프레미스 지원이 주요 장점입니다.

---

### 문항 159

송팀장은 Azure에서 Prometheus와 Grafana 스택을 구성하고 있다. 다음 중 Azure Monitor와 비교한 오픈소스 모니터링 스택의 이점이 아닌 것은?

A. 벤더 종속성 없음  
B. 커뮤니티의 풍부한 대시보드 템플릿  
C. Azure와의 네이티브 통합  
D. PromQL 쿼리 언어

**정답: C**  
**해설:** Azure 네이티브 통합은 Azure Monitor의 강점입니다. 오픈소스 스택은 커스터마이징 자유도와 멀티 클라우드 지원이 장점입니다.

---

### 문항 160

장전임은 Azure에서 ArgoCD를 사용하여 GitOps를 구현하려고 한다. 다음 중 ArgoCD와 Flux의 차이점으로 맞는 것은?

A. ArgoCD는 웹 UI를 기본 제공한다.  
B. Flux만 Helm을 지원한다.  
C. ArgoCD는 Pull 기반이 아니다.  
D. Flux는 Git 저장소와 연동되지 않는다.

**정답: A**  
**해설:** ArgoCD는 웹 기반 대시보드를 기본 제공하여 배포 상태를 시각적으로 확인할 수 있습니다. Flux는 CLI 중심입니다.

---

### 문항 161

박대리는 Azure에서 Jenkins를 CI/CD 도구로 사용하고 있다. 다음 중 Azure Plugin for Jenkins가 지원하는 기능이 아닌 것은?

A. Azure VM 에이전트 프로비저닝  
B. Azure Container Instances 에이전트  
C. Azure Storage에 아티팩트 저장  
D. Azure 인프라 자동 설계

**정답: D**  
**해설:** Jenkins Azure Plugin은 에이전트 프로비저닝, 스토리지 통합, 배포 등을 지원하지만, 인프라 자동 설계는 제공하지 않습니다.

---

### 문항 162

김과장은 Azure에서 Helm Chart를 사용하여 애플리케이션을 배포하고 있다. 다음 중 Helm의 이점이 아닌 것은?

A. 패키지 관리  
B. 버전 관리 및 롤백  
C. 템플릿 기반 배포  
D. 런타임 성능 최적화

**정답: D**  
**해설:** Helm은 Kubernetes 패키지 관리 도구로, 배포와 구성 관리를 간소화합니다. 런타임 성능은 애플리케이션 자체에 달려있습니다.

---

### 문항 163

이선임은 Azure에서 Kafka를 사용하려고 한다. 다음 중 Azure에서 Kafka를 사용하는 방법이 아닌 것은?

A. Azure Event Hub (Kafka 호환 프로토콜)  
B. Azure HDInsight Kafka  
C. Confluent Cloud on Azure (마켓플레이스)  
D. Azure Functions에 내장된 Kafka

**정답: D**  
**해설:** Azure Functions 자체에 Kafka가 내장되어 있지는 않습니다. Kafka 트리거/바인딩은 사용 가능하지만, Kafka 클러스터는 별도 서비스가 필요합니다.

---

### 문항 164

송팀장은 Azure에서 PostgreSQL을 운영하고 있다. 다음 중 Azure Database for PostgreSQL Flexible Server의 특징으로 틀린 것은?

A. 고가용성 구성 선택 가능  
B. 유지 관리 기간 사용자 정의  
C. PgBouncer 내장 지원  
D. 모든 PostgreSQL Extension 무제한 지원

**정답: D**  
**해설:** Flexible Server는 많은 Extension을 지원하지만, 보안 및 운영상의 이유로 일부 Extension은 제한되거나 지원되지 않습니다.

---

### 문항 165

장전임은 Azure에서 Redis를 오픈소스로 직접 운영하는 것과 Azure Cache for Redis를 비교하고 있다. 다음 중 관리형 서비스의 이점이 아닌 것은?

A. 자동 패치 및 업데이트  
B. 고가용성 내장  
C. 완전한 Redis 모듈 커스터마이징  
D. SLA 보장

**정답: C**  
**해설:** 관리형 서비스는 일부 Redis 모듈 커스터마이징에 제약이 있을 수 있습니다. 완전한 커스터마이징은 자체 운영 시 가능합니다.

---

### 문항 166

박대리는 Azure에서 Nginx를 Ingress Controller로 사용하고 있다. 다음 중 Azure Application Gateway Ingress Controller(AGIC) 대비 NGINX의 이점은?

A. Azure 네이티브 WAF 통합  
B. 더 넓은 커뮤니티 지원과 커스터마이징  
C. Azure Monitor 자동 통합  
D. Azure 관리형 서비스로 제공

**정답: B**  
**해설:** NGINX Ingress Controller는 오픈소스 커뮤니티의 방대한 지원과 높은 커스터마이징 자유도가 장점입니다. AGIC는 Azure 네이티브 통합이 강점입니다.

---

### 문항 167

김과장은 Azure에서 Terraform Provider for Azure(azurerm)의 버전 관리를 하고 있다. 다음 중 Provider 버전 고정의 이점은?

A. 최신 기능을 항상 사용  
B. 재현 가능한 인프라 배포 보장  
C. Provider 다운로드 속도 향상  
D. Azure API 변경 자동 반영

**정답: B**  
**해설:** Provider 버전을 고정하면 팀 전체에서 동일한 동작을 보장하고, 예기치 않은 변경을 방지하여 재현 가능한 배포가 가능합니다.

---

### 문항 168

이선임은 Azure에서 Pulumi를 IaC 도구로 사용하고 있다. 다음 중 Terraform 대비 Pulumi의 차별점은?

A. HCL 전용 언어 사용  
B. Python, TypeScript, Go 등 범용 프로그래밍 언어 사용  
C. Azure만 지원  
D. 상태 파일 없음

**정답: B**  
**해설:** Pulumi는 Python, TypeScript, Go, C#, Java 등 범용 프로그래밍 언어를 사용하여 IaC를 작성할 수 있습니다.

---

### 문항 169

송팀장은 Azure에서 Cert-Manager를 AKS에 설치하고 있다. 다음 중 Cert-Manager의 주요 기능은?

A. 네트워크 트래픽 암호화  
B. TLS 인증서 자동 발급 및 갱신  
C. 코드 서명  
D. 사용자 인증

**정답: B**  
**해설:** Cert-Manager는 Let's Encrypt 등의 CA에서 TLS 인증서를 자동으로 발급하고 갱신하는 Kubernetes 네이티브 도구입니다.

---

### 문항 170

장전임은 Azure에서 Dapr(Distributed Application Runtime)를 AKS에 통합하고 있다. 다음 중 Dapr가 제공하지 않는 Building Block은?

A. Service Invocation  
B. State Management  
C. Pub/Sub Messaging  
D. Machine Learning Training

**정답: D**  
**해설:** Dapr는 Service Invocation, State Management, Pub/Sub, Bindings, Secrets Management 등을 제공합니다. ML Training은 범위 밖입니다.

---

### 문항 171

박대리는 Azure에서 Open Policy Agent(OPA)와 Gatekeeper를 AKS에 구성하고 있다. 다음 중 Gatekeeper의 용도는?

A. Pod 네트워크 정책  
B. 컨테이너 런타임 보안  
C. Kubernetes Admission Control을 통한 정책 적용  
D. 모니터링 데이터 수집

**정답: C**  
**해설:** Gatekeeper는 OPA 기반의 Kubernetes Admission Controller로, 리소스 생성/수정 시 정책을 검증합니다.

---

### 문항 172

김과장은 Azure에서 MinIO를 S3 호환 Object Storage로 사용하려고 한다. 다음 중 Azure Blob Storage 대비 MinIO의 이점은?

A. Azure 네이티브 통합  
B. S3 API 완벽 호환으로 멀티 클라우드/온프레미스 이식성  
C. Azure SLA 보장  
D. Azure Portal에서 관리

**정답: B**  
**해설:** MinIO는 S3 API 완벽 호환으로 AWS, Azure, 온프레미스 간 애플리케이션 이식성이 뛰어납니다.

---

### 문항 173

이선임은 Azure에서 Crossplane을 사용하여 인프라를 관리하려고 한다. 다음 중 Crossplane의 특징으로 맞는 것은?

A. Terraform과 동일한 HCL 언어 사용  
B. Kubernetes Custom Resource로 클라우드 인프라 관리  
C. Azure Portal에서만 사용 가능  
D. 컨테이너 런타임으로만 사용

**정답: B**  
**해설:** Crossplane은 Kubernetes CRD(Custom Resource Definition)를 사용하여 클라우드 인프라를 Kubernetes 네이티브 방식으로 관리합니다.

---

### 문항 174

송팀장은 Azure에서 Velero를 사용하여 AKS 백업을 구성하고 있다. 다음 중 Velero가 백업하는 대상이 아닌 것은?

A. Kubernetes 리소스 (Deployment, Service 등)  
B. Persistent Volume 데이터  
C. Node OS 설정  
D. Namespace 구성

**정답: C**  
**해설:** Velero는 Kubernetes 리소스와 PV 데이터를 백업하며, Node OS 수준 설정은 백업하지 않습니다.

---

### 문항 175

장전임은 Azure에서 Keycloak을 Identity Provider로 사용하려고 한다. 다음 중 Keycloak과 Microsoft Entra ID를 함께 사용하는 시나리오는?

A. Keycloak을 Entra ID의 External Identity Provider로 연동  
B. Keycloak이 Entra ID를 완전히 대체  
C. 두 서비스는 함께 사용할 수 없다.  
D. Keycloak이 Azure VM을 직접 관리

**정답: A**  
**해설:** Keycloak은 SAML/OIDC를 통해 Entra ID와 연동하여 외부 사용자 인증이나 레거시 시스템 통합에 사용할 수 있습니다.

---

### 문항 176

박대리는 Azure에서 ELK Stack(Elasticsearch, Logstash, Kibana)을 구성하고 있다. 다음 중 Azure에서 ELK를 관리형으로 사용하는 방법은?

A. Azure Marketplace의 Elastic Cloud  
B. Azure에서 공식 ELK 서비스 제공  
C. Azure Functions에 내장  
D. Azure Monitor가 ELK를 완전히 대체

**정답: A**  
**해설:** Elastic은 Azure Marketplace를 통해 Elastic Cloud를 관리형 서비스로 제공합니다. Azure Monitor는 별도의 로그 분석 솔루션입니다.

---

### 문항 177

김과장은 Azure에서 GitLab과 Azure DevOps를 비교하고 있다. 다음 중 GitLab의 이점으로 맞는 것은?

A. Azure 네이티브 통합이 더 뛰어남  
B. 단일 플랫폼에서 소스 코드 관리부터 CI/CD, 보안까지 통합 제공  
C. Azure Boards와 직접 연동  
D. Azure Pipelines와 동일한 YAML 구문

**정답: B**  
**해설:** GitLab은 SCM, CI/CD, 보안 스캔, 패키지 레지스트리 등을 단일 플랫폼에서 제공하는 DevOps 올인원 솔루션입니다.

---

### 문항 178

이선임은 Azure에서 Harbor를 Private Container Registry로 사용하려고 한다. 다음 중 ACR 대비 Harbor의 이점은?

A. Azure Portal 통합  
B. 멀티 클라우드/온프레미스 이식성과 이미지 취약점 스캐닝 내장  
C. Azure Managed Identity 지원  
D. Azure 관리형 서비스

**정답: B**  
**해설:** Harbor는 오픈소스 레지스트리로 멀티 클라우드/온프레미스에서 사용 가능하며, Trivy 기반 이미지 취약점 스캐닝을 내장합니다.

---

### 문항 179

송팀장은 Azure에서 Kustomize를 사용하여 Kubernetes 매니페스트를 관리하고 있다. 다음 중 Kustomize와 Helm의 차이점으로 맞는 것은?

A. Kustomize는 템플릿 엔진을 사용한다.  
B. Kustomize는 기존 YAML을 패치/오버레이 방식으로 수정한다.  
C. Helm은 패치 방식을 사용한다.  
D. 두 도구는 함께 사용할 수 없다.

**정답: B**  
**해설:** Kustomize는 템플릿 없이 기존 YAML 파일에 패치/오버레이를 적용하는 방식이고, Helm은 Go 템플릿 기반입니다. 두 도구는 함께 사용 가능합니다.

---

### 문항 180

장전임은 Azure에서 Backstage를 개발자 포털로 구성하려고 한다. 다음 중 Backstage의 기능이 아닌 것은?

A. 서비스 카탈로그  
B. TechDocs  
C. 소프트웨어 템플릿  
D. 클라우드 인프라 직접 프로비저닝

**정답: D**  
**해설:** Backstage는 서비스 카탈로그, 문서화, 템플릿을 통한 개발자 경험 향상 플랫폼이며, 인프라 프로비저닝은 Terraform 등 IaC 도구를 통합하여 수행합니다.

---

**총 180문항 생성 완료**

카테고리별 분포:
- Azure 서비스 기본 개념: 30문항
- Azure 아키텍처 구성 방안: 25문항
- Azure 자격증 대비: 25문항
- Azure 리소스별 특장점/제약사항/주의사항: 25문항
- Azure RBAC, Security, AI 기능: 25문항
- AKS: 25문항
- 오픈소스 연동: 25문항
