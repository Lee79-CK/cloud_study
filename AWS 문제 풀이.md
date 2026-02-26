# Senior Cloud Engineer AWS 주관식 문제집

---

## 1. AWS 서비스 기본 개념

### Q1-1. AWS Region과 Availability Zone(AZ)의 차이를 설명하고, Multi-AZ 아키텍처를 사용해야 하는 이유를 구체적으로 서술하시오.

**모범답안:**

Region은 전 세계에 분포된 물리적 데이터센터 클러스터의 지리적 영역이며, 각 Region은 최소 3개의 Availability Zone(AZ)으로 구성된다. AZ는 하나 이상의 독립된 데이터센터로 구성되며, 각 AZ는 독립된 전력, 네트워크, 냉각 시스템을 갖추고 있다. AZ 간에는 고속 저지연 전용 네트워크로 연결되어 있다.

Multi-AZ 아키텍처를 사용해야 하는 이유는 다음과 같다.

첫째, **고가용성(High Availability)** 확보이다. 단일 AZ에서 장애가 발생하더라도 다른 AZ에서 서비스를 지속할 수 있어 다운타임을 최소화할 수 있다. 예를 들어 RDS Multi-AZ 배포 시 Primary DB 장애 발생 시 Standby로 자동 Failover되며, 일반적으로 60~120초 내에 복구된다.

둘째, **재해복구(Disaster Recovery)** 측면이다. 자연재해나 대규모 인프라 장애에 대비한 물리적 분리를 확보할 수 있다.

셋째, **데이터 내구성** 향상이다. S3의 경우 자동으로 최소 3개 AZ에 데이터를 복제하여 99.999999999%(11 9s)의 내구성을 제공한다.

넷째, **지연시간 최적화**이다. 동일 Region 내 AZ 간 지연시간은 single-digit millisecond 수준으로, Cross-Region 대비 훨씬 낮은 지연시간으로 고가용성을 확보할 수 있다.

---

### Q1-2. AWS Well-Architected Framework의 6가지 Pillar를 나열하고, 각 Pillar의 핵심 설계 원칙을 Senior Engineer 관점에서 설명하시오.

**모범답안:**

AWS Well-Architected Framework의 6가지 Pillar는 다음과 같다.

**① Operational Excellence (운영 우수성):** 운영 프로세스의 자동화, IaC(Infrastructure as Code) 적용, 점진적 변경 수행, 장애 대응 절차 수립이 핵심이다. CloudFormation/Terraform으로 인프라를 코드화하고, CI/CD 파이프라인을 통해 배포를 자동화하며, Runbook/Playbook을 통한 표준화된 운영 절차를 수립해야 한다.

**② Security (보안):** 최소 권한 원칙(Least Privilege), 모든 계층에서의 보안 적용(Defense in Depth), 추적 가능성(Traceability) 확보가 핵심이다. IAM을 통한 세밀한 접근 제어, CloudTrail을 통한 API 감사 로그, KMS를 통한 암호화, Security Group/NACL을 통한 네트워크 보안을 구현해야 한다.

**③ Reliability (안정성):** 장애 자동 복구, 수평 확장을 통한 가용성 향상, 변경 관리 자동화가 핵심이다. Auto Scaling, Multi-AZ 배포, 적절한 백업/복구 전략을 수립해야 한다. Recovery Point Objective(RPO)와 Recovery Time Objective(RTO)를 명확히 정의해야 한다.

**④ Performance Efficiency (성능 효율성):** 워크로드에 적합한 리소스 타입/크기 선택, 서버리스 아키텍처 활용, 글로벌 배포를 통한 지연시간 최소화가 핵심이다. 정기적인 성능 테스트와 모니터링을 통해 최적의 리소스 구성을 유지해야 한다.

**⑤ Cost Optimization (비용 최적화):** 사용량 기반 과금 모델 활용, 불필요한 리소스 제거, Reserved Instance/Savings Plan을 통한 비용 절감이 핵심이다. AWS Cost Explorer, Trusted Advisor를 활용한 지속적인 비용 분석이 필요하다.

**⑥ Sustainability (지속 가능성):** 2021년에 추가된 최신 Pillar로, 환경 영향 최소화를 위한 리소스 효율적 사용, 관리형 서비스 활용, 데이터 보존 정책 수립이 핵심이다.

---

### Q1-3. AWS에서 Shared Responsibility Model을 설명하고, IaaS(EC2)와 관리형 서비스(RDS, Lambda) 각각에서 고객과 AWS의 책임 범위를 구분하시오.

**모범답안:**

Shared Responsibility Model은 AWS와 고객 간 보안 및 컴플라이언스에 대한 책임을 명확히 분리하는 모델이다.

**AWS의 책임 ("Security OF the Cloud"):** 물리적 데이터센터 보안, 하드웨어/소프트웨어 인프라, 네트워크 인프라, 가상화 계층 관리를 담당한다.

**고객의 책임 ("Security IN the Cloud"):** 서비스 유형에 따라 달라진다.

**EC2(IaaS)의 경우:**
- AWS: 물리 서버, 하이퍼바이저, 네트워크 인프라
- 고객: OS 패치, 방화벽 설정(Security Group), 애플리케이션 보안, 데이터 암호화, IAM 관리, OS 레벨 접근 제어

**RDS(PaaS)의 경우:**
- AWS: OS 패치, DB 엔진 패치, 하드웨어, 고가용성 인프라
- 고객: DB 내부 사용자 관리, 데이터 암호화 설정, Security Group 설정, 파라미터 그룹 설정, 백업 정책 결정

**Lambda(Serverless)의 경우:**
- AWS: 런타임 환경, OS, 하드웨어, 스케일링, 고가용성
- 고객: 함수 코드 보안, IAM 실행 역할, 환경변수 내 민감 정보 관리, VPC 설정(필요 시)

핵심은 추상화 수준이 높아질수록 AWS의 책임 범위가 넓어지고, 고객의 책임 범위는 줄어든다는 것이다.

---

### Q1-4. AWS Global Infrastructure에서 Edge Location의 역할과 CloudFront, Route 53, Global Accelerator가 Edge Location을 활용하는 방식을 비교 설명하시오.

**모범답안:**

Edge Location은 전 세계 주요 도시에 분포된 AWS의 캐시 서버 및 네트워크 접점으로, 현재 400개 이상의 PoP(Point of Presence)로 구성되어 있다. 엔드 유저에게 더 가까운 위치에서 콘텐츠를 제공하여 지연시간을 최소화하는 역할을 한다.

**CloudFront:** CDN 서비스로, Edge Location에 정적/동적 콘텐츠를 캐싱한다. Origin(S3, ALB, EC2 등)의 콘텐츠를 Edge에 캐시하여 반복 요청 시 Origin 접근 없이 Edge에서 직접 응답한다. Lambda@Edge나 CloudFront Functions를 통해 Edge에서 요청/응답을 가공할 수 있다. TTL 기반 캐시 관리와 Invalidation을 통한 즉시 캐시 무효화를 지원한다.

**Route 53:** DNS 서비스로, Edge Location에서 DNS 쿼리를 처리한다. Latency-based Routing으로 가장 낮은 지연시간의 리전으로 트래픽을 라우팅하고, Geolocation/Geoproximity Routing으로 지역 기반 라우팅을 수행한다. Health Check와 연계하여 Failover Routing도 가능하다.

**Global Accelerator:** AWS의 글로벌 네트워크를 활용하여 트래픽을 최적 경로로 전달하는 서비스이다. Edge Location에서 Anycast IP를 통해 트래픽을 수신하고, AWS 백본 네트워크를 통해 최적의 엔드포인트(ALB, NLB, EC2, EIP)로 전달한다. CloudFront와의 핵심 차이점은, CloudFront는 HTTP/HTTPS 캐싱에 특화되어 있고, Global Accelerator는 TCP/UDP 트래픽 전반에 대해 네트워크 경로 최적화를 제공한다는 것이다. 게임, IoT, VoIP 등 비HTTP 프로토콜 워크로드에 적합하다.

---

### Q1-5. AWS에서 제공하는 컴퓨팅 서비스(EC2, Lambda, ECS, EKS, Fargate, App Runner, Lightsail)를 워크로드 특성에 따라 어떻게 선택해야 하는지 설명하시오.

**모범답안:**

**EC2:** 완전한 OS 제어가 필요한 경우, GPU 워크로드(ML 학습), 고성능 컴퓨팅(HPC), 특수 소프트웨어 라이선스가 필요한 워크로드에 적합하다. 가장 높은 유연성을 제공하지만 운영 부담이 크다.

**Lambda:** 이벤트 기반, 단기 실행(최대 15분), 간헐적 트래픽 패턴의 워크로드에 적합하다. API Gateway 연동 REST API, S3 이벤트 처리, DynamoDB Streams 처리 등에 활용된다. Cold Start 지연(특히 VPC 연결 시)과 패키지 크기 제한(250MB/컨테이너 이미지 10GB)에 주의해야 한다.

**ECS:** Docker 컨테이너 기반 워크로드로, AWS 에코시스템과의 긴밀한 통합이 필요할 때 적합하다. Task Definition 기반으로 비교적 단순한 컨테이너 오케스트레이션을 제공한다.

**EKS:** Kubernetes 에코시스템 활용, 멀티 클라우드/하이브리드 환경, 복잡한 마이크로서비스 아키텍처에 적합하다. Kubernetes의 풍부한 오픈소스 에코시스템을 활용할 수 있으나, 학습 곡선이 높고 클러스터 관리 비용($0.10/hr per cluster)이 발생한다.

**Fargate:** 서버리스 컨테이너 실행 환경으로, 인프라 관리 부담 없이 컨테이너를 실행할 때 적합하다. ECS/EKS 모두에서 사용 가능하며, 노드 관리가 불필요하지만 EC2 대비 비용이 높고 GPU를 지원하지 않는다.

**App Runner:** 소스 코드나 컨테이너 이미지에서 직접 웹 앱을 빌드/배포하는 완전 관리형 서비스이다. 빠른 프로토타이핑이나 단순한 웹 서비스에 적합하지만 세밀한 제어가 제한적이다.

**Lightsail:** 소규모 프로젝트, 개인 블로그, 간단한 웹사이트에 적합한 VPS 서비스이다. 고정 월 요금으로 예측 가능한 비용을 제공하며, AWS 서비스와의 통합은 제한적이다.

선택 기준은 운영 복잡도, 확장성 요구사항, 비용, 팀의 기술 역량을 종합적으로 고려해야 한다.

---

## 2. AWS 아키텍처 구성 방안

### Q2-1. 일 평균 100만 DAU 규모의 웹 서비스를 AWS에서 구축한다고 가정할 때, 고가용성과 확장성을 갖춘 3-Tier 아키텍처를 설계하시오. 각 계층의 AWS 서비스 선택 근거와 네트워크 설계를 포함하시오.

**모범답안:**

**네트워크 계층(VPC 설계):**
- VPC CIDR: /16 (예: 10.0.0.0/16, 65,536 IPs)
- 최소 2개 AZ, 권장 3개 AZ에 걸쳐 서브넷 배치
- Public Subnet: NAT Gateway, Bastion Host(또는 Session Manager 대체), ALB
- Private Subnet(App): 애플리케이션 서버
- Private Subnet(DB): 데이터베이스
- 서브넷 간 NACL과 Security Group을 조합하여 최소 권한 네트워크 접근 제어

**Presentation Tier (프론트엔드):**
- CloudFront + S3: 정적 콘텐츠(SPA) 배포, OAC(Origin Access Control)로 S3 직접 접근 차단
- Route 53: DNS 관리, Latency-based Routing 또는 Weighted Routing
- AWS WAF: CloudFront/ALB 앞단에서 SQL Injection, XSS 등 방어
- AWS Shield Advanced: DDoS 방어 (비용 대비 필요성 검토)

**Application Tier (백엔드):**
- ALB(Application Load Balancer): Path/Host 기반 라우팅, Sticky Session(필요 시), SSL Termination
- Auto Scaling Group + EC2 또는 ECS/Fargate: 트래픽 기반 자동 확장
- Target Tracking Scaling Policy: CPU 70% 또는 ALB RequestCount 기반
- ElastiCache(Redis): 세션 관리 및 핫 데이터 캐싱, Cluster Mode Enabled로 수평 확장

**Data Tier (데이터):**
- RDS Aurora(MySQL/PostgreSQL): Multi-AZ 배포, Read Replica(최대 15개)로 읽기 부하 분산
- Aurora Serverless v2: 트래픽 변동이 큰 경우 자동 스케일링
- DynamoDB: 고속 키-값 조회가 필요한 데이터(예: 사용자 세션, 장바구니)
- S3: 파일 업로드, 로그 저장, 백업

**부가 인프라:**
- SQS/SNS: 비동기 처리 및 서비스 간 디커플링
- CloudWatch: 모니터링 및 알람
- AWS Backup: 중앙 집중식 백업 관리

---

### Q2-2. 마이크로서비스 아키텍처를 AWS에서 구현할 때, 서비스 간 통신 패턴(동기/비동기)을 설계하고, 각 패턴에 적합한 AWS 서비스를 매핑하시오. 또한 서비스 메시(Service Mesh)의 필요성과 AWS App Mesh의 역할을 설명하시오.

**모범답안:**

**동기 통신 패턴:**
- **API Gateway + Lambda/ECS:** RESTful API 또는 HTTP 기반 동기 호출. API Gateway는 인증, 속도 제한(Throttling), API 버전 관리를 제공한다. 단, 동기 호출은 서비스 간 강결합을 유발하므로 timeout 설정, Circuit Breaker 패턴 적용이 필수적이다.
- **ALB + ECS/EKS:** 내부 서비스 간 직접 HTTP 호출 시 Internal ALB를 활용한다. Service Discovery(Cloud Map)와 연계하여 서비스 엔드포인트를 동적으로 관리한다.
- **gRPC 통신:** NLB를 사용하여 gRPC 기반 동기 통신을 구현할 수 있다. ALB도 gRPC를 지원하므로 path 기반 라우팅이 필요하면 ALB를 선택한다.

**비동기 통신 패턴:**
- **SQS (Simple Queue Service):** Point-to-Point 메시징. 주문 처리, 이메일 발송 등 순차 처리에 적합. Standard Queue(최소 1회 전달, 순서 미보장)와 FIFO Queue(정확히 1회 전달, 순서 보장, 초당 300~3,000 메시지)를 워크로드에 맞게 선택한다. Dead Letter Queue(DLQ)를 반드시 설정하여 처리 실패 메시지를 관리한다.
- **SNS (Simple Notification Service):** Pub/Sub 패턴. 하나의 이벤트를 여러 서비스가 구독하는 Fan-out 패턴에 적합. SNS + SQS 조합으로 안정적인 Fan-out 아키텍처를 구현한다.
- **EventBridge:** 이벤트 기반 아키텍처의 중앙 이벤트 버스. 규칙 기반 라우팅, 스키마 레지스트리, SaaS 통합을 지원한다. 복잡한 이벤트 필터링과 변환이 필요한 경우 SNS보다 적합하다.
- **Kinesis Data Streams:** 실시간 대용량 스트리밍 데이터 처리. 로그 수집, 클릭스트림 분석 등에 적합. Shard 단위 확장이 가능하며, 24시간~365일 데이터 보존을 지원한다.
- **MSK (Managed Kafka):** Apache Kafka 워크로드가 이미 존재하거나, 높은 처리량과 복잡한 토픽 기반 라우팅이 필요한 경우에 적합하다.

**AWS App Mesh의 역할:**
App Mesh는 Envoy 프록시 기반 서비스 메시로, 각 마이크로서비스에 사이드카 프록시를 배포하여 서비스 간 트래픽을 제어한다. 트래픽 라우팅(Canary, Blue/Green), 재시도 정책(Retry Policy), 타임아웃 설정, 서킷 브레이커, mTLS(상호 TLS 인증), Observability(X-Ray 연동)를 애플리케이션 코드 변경 없이 인프라 레벨에서 관리할 수 있다.

다만, 서비스 수가 적거나 통신 패턴이 단순한 경우 App Mesh의 복잡성이 오버엔지니어링이 될 수 있으므로, 서비스 규모와 팀 역량을 고려하여 도입 여부를 결정해야 한다. EKS 환경에서는 Istio를 대안으로 고려할 수 있다.

---

### Q2-3. 글로벌 서비스를 위한 Multi-Region Active-Active 아키텍처를 설계할 때 고려해야 할 사항과 핵심 AWS 서비스를 설명하시오.

**모범답안:**

**데이터 복제 전략:**
- **DynamoDB Global Tables:** Multi-Region 간 Active-Active 복제를 네이티브로 지원한다. 각 Region에서 읽기/쓰기가 가능하며, 일반적으로 1초 이내 Region 간 복제가 완료된다. 다만 Last Writer Wins 충돌 해소 전략을 사용하므로 동일 아이템에 대한 동시 쓰기 시 데이터 충돌 가능성에 주의해야 한다.
- **Aurora Global Database:** Primary Region에서 쓰기, Secondary Region에서 읽기를 처리하며, 복제 지연은 일반적으로 1초 미만이다. Failover 시 Secondary를 Primary로 승격하는 데 약 1분이 소요된다. Active-Active 쓰기는 Write Forwarding 기능으로 제한적 지원된다.
- **S3 Cross-Region Replication (CRR):** 객체 수준의 비동기 복제를 제공한다.
- **ElastiCache Global Datastore:** Redis 기반 글로벌 캐시 복제를 지원한다.

**트래픽 라우팅:**
- Route 53 Latency-based Routing 또는 Geolocation Routing으로 사용자를 가장 가까운 Region으로 라우팅한다. Health Check + Failover Routing으로 Region 장애 시 자동 절체한다. Global Accelerator를 사용하면 Anycast IP를 통해 더 빠른 Failover가 가능하다(DNS TTL 대기 불필요).

**데이터 일관성:**
- CAP 정리에 따라 네트워크 파티션 상황에서 가용성과 일관성 사이의 Trade-off를 명확히 결정해야 한다. 대부분의 글로벌 서비스는 Eventual Consistency를 수용하며, 금융 거래 등 Strong Consistency가 필요한 경우 단일 Region에서 처리 후 비동기로 복제하는 패턴을 사용한다.

**Stateless 설계:**
- 애플리케이션은 반드시 Stateless로 설계하여 어떤 Region에서든 요청을 처리할 수 있어야 한다. 세션 정보는 DynamoDB Global Tables 또는 ElastiCache Global Datastore에 저장한다.

**배포 전략:**
- 각 Region에 동일한 인프라를 IaC(CloudFormation StackSets, Terraform)로 배포하고, CI/CD 파이프라인에서 Multi-Region 동시/순차 배포를 관리한다.

---

### Q2-4. 온프레미스에서 AWS로의 대규모 데이터 마이그레이션(수십 TB 이상) 전략을 수립하시오. 네트워크 대역폭, 다운타임, 데이터 정합성을 모두 고려하시오.

**모범답안:**

**마이그레이션 방법별 비교:**

- **AWS DataSync:** 온프레미스 NFS/SMB 스토리지에서 S3/EFS/FSx로 전송. 에이전트 기반으로 네트워크를 통한 전송을 최적화(압축, 병렬 전송, 무결성 검증)한다. 수 TB 규모, 네트워크 대역폭이 충분한 경우 적합하다.

- **AWS Transfer Family:** SFTP/FTPS/FTP 기반 기존 워크플로우를 유지하면서 S3/EFS로 전송할 때 사용한다.

- **AWS Snow Family:**
  - Snowcone(8~14TB): 엣지/원격 환경의 소규모 데이터
  - Snowball Edge(80~210TB): 수십 TB 규모 오프라인 전송, Storage Optimized/Compute Optimized 선택
  - Snowmobile(100PB): 엑사바이트 규모의 초대량 데이터 전송

- **AWS DMS (Database Migration Service):** 데이터베이스 마이그레이션 전용. 동종(Oracle→RDS Oracle) 및 이종(Oracle→Aurora PostgreSQL) 마이그레이션 지원. SCT(Schema Conversion Tool)로 스키마 변환, CDC(Change Data Capture)로 지속적 복제를 통해 다운타임을 최소화한다.

**권장 전략 (수십 TB, 데이터베이스 포함):**

Phase 1 - **초기 대량 전송:** Snowball Edge로 정적 데이터(파일, 백업) 오프라인 전송. 병렬로 Direct Connect를 설정하여 네트워크 전송 경로도 확보한다.

Phase 2 - **증분 동기화:** DataSync 또는 DMS CDC로 Snowball 전송 이후 변경분을 지속적으로 동기화한다.

Phase 3 - **컷오버:** 최종 동기화 후 DNS 전환 또는 애플리케이션 엔드포인트 변경. 다운타임 윈도우를 최소화하기 위해 사전에 충분한 리허설을 수행한다.

Phase 4 - **검증:** 데이터 정합성 검증(체크섬 비교, 레코드 카운트), 성능 테스트, 롤백 계획 확인.

**네트워크 고려사항:** 1Gbps 회선으로 10TB 전송 시 이론상 약 22시간이 소요된다. Direct Connect(1G/10G/100G) 전용선 설정에 수 주가 걸리므로 사전에 준비해야 한다. VPN over Internet은 대역폭이 불안정하여 대규모 마이그레이션에는 부적합하다.

---

### Q2-5. Serverless 아키텍처로 실시간 이미지/동영상 처리 파이프라인을 설계하시오.

**모범답안:**

**아키텍처 설계:**

1. **업로드 단계:** 클라이언트 → API Gateway → Lambda(Pre-signed URL 생성) → S3 Upload. S3 Multipart Upload를 사용하여 대용량 파일의 안정적 업로드를 보장한다. CloudFront Signed URL을 활용하여 업로드 경로도 최적화할 수 있다.

2. **이벤트 트리거:** S3 Event Notification → EventBridge 또는 SQS → Lambda. EventBridge를 사용하면 이벤트 필터링(파일 확장자, 크기 등)이 용이하다.

3. **이미지 처리:** Lambda(Python/Pillow 또는 Sharp)로 리사이징, 썸네일 생성, 워터마크 적용. Step Functions를 사용하여 여러 처리 단계를 오케스트레이션한다.
   - Lambda 메모리: 최대 10,240MB
   - Lambda 임시 스토리지: 최대 10GB (/tmp)
   - Lambda 실행 시간: 최대 15분

4. **동영상 처리:** AWS Elemental MediaConvert로 트랜스코딩(HLS/DASH 변환, 해상도별 인코딩). Lambda가 MediaConvert Job을 생성하고, EventBridge에서 Job 완료 이벤트를 수신하여 후속 처리를 수행한다.

5. **AI/ML 기반 처리:** Amazon Rekognition으로 객체 감지, 얼굴 인식, 콘텐츠 모더레이션. Amazon Textract로 이미지 내 텍스트 추출.

6. **결과 저장/전달:** 처리된 파일은 S3에 저장, 메타데이터는 DynamoDB에 기록. SNS/WebSocket(API Gateway)으로 클라이언트에 처리 완료 알림.

7. **배포(서빙):** CloudFront를 통해 처리된 이미지/동영상을 엔드유저에게 제공.

**주의사항:** Lambda의 페이로드 크기 제한(동기 6MB, 비동기 256KB)으로 인해, 대용량 파일은 반드시 S3를 통해 전달해야 한다. 동영상 처리는 Lambda의 15분 제한을 초과할 수 있으므로 MediaConvert나 ECS Fargate를 사용해야 한다.

---

## 3. AWS 자격증 대비

### Q3-1. Solutions Architect Professional 시험에서 자주 출제되는 재해복구(DR) 전략 4가지를 RTO/RPO 관점에서 비교하고, 각 전략의 AWS 구현 방법을 설명하시오.

**모범답안:**

**① Backup & Restore (RTO: 수시간, RPO: 수시간)**
- 가장 저비용 전략. AWS Backup으로 EBS 스냅샷, RDS 스냅샷, DynamoDB 백업을 정기적으로 S3에 저장한다. Cross-Region Replication으로 DR Region에 백업을 복제한다. 복구 시 CloudFormation/Terraform으로 인프라를 재생성하고 백업에서 데이터를 복원한다.
- 비용: 매우 낮음 (스토리지 비용만 발생)

**② Pilot Light (RTO: 수십 분, RPO: 수분~수십 분)**
- DR Region에 핵심 인프라(DB 복제본)만 최소한으로 유지한다. Aurora Global Database나 RDS Cross-Region Read Replica로 데이터를 지속 복제한다. 장애 발생 시 Auto Scaling Group을 활성화하고 Read Replica를 Primary로 승격한다. Route 53 Health Check + Failover로 DNS 전환한다.
- 비용: 낮음 (DB 복제 비용 + 최소 인프라)

**③ Warm Standby (RTO: 수분, RPO: 수초~수분)**
- DR Region에 축소된 규모의 전체 시스템을 운영한다. 평소 최소 규모로 동작하다가 Failover 시 Auto Scaling으로 Scale-Out한다. Aurora Global Database + 축소된 ECS/EC2 플릿을 유지한다.
- 비용: 중간 (축소 규모의 상시 운영 비용)

**④ Multi-Site Active-Active (RTO: ~0, RPO: ~0)**
- 두 개 이상의 Region에서 동시에 트래픽을 처리한다. DynamoDB Global Tables, Aurora Global Database(Write Forwarding)를 활용한다. Route 53 Latency-based Routing으로 트래픽을 분산한다.
- 비용: 가장 높음 (복수 Region 전체 인프라 비용)

**시험 Tip:** 시나리오에서 제시된 RTO/RPO 요구사항과 비용 제약을 반드시 확인하고, 가장 비용 효율적이면서 요구사항을 충족하는 전략을 선택해야 한다.

---

### Q3-2. AWS에서 VPC Peering, Transit Gateway, PrivateLink의 차이를 설명하고, 각각의 사용 시나리오를 제시하시오.

**모범답안:**

**VPC Peering:**
- 두 VPC 간 1:1 프라이빗 네트워크 연결이다. 같은 Region 또는 Cross-Region 간 설정 가능하다. Transitive Peering을 지원하지 않아 VPC-A ↔ VPC-B ↔ VPC-C 구성 시 A에서 C로 직접 통신 불가하다(별도 Peering 필요). CIDR 중복 불가하다.
- **사용 시나리오:** VPC 수가 적고(2~3개), 단순한 연결이 필요한 경우. 예: 개발/운영 VPC 간 연결.
- **비용:** 동일 AZ 간 무료, Cross-AZ/Cross-Region은 데이터 전송 비용 발생.

**Transit Gateway (TGW):**
- Hub-and-Spoke 모델의 중앙 집중식 네트워크 허브이다. 수천 개의 VPC, VPN, Direct Connect를 연결할 수 있다. Route Table 기반으로 세밀한 트래픽 라우팅이 가능하다. Transit Gateway Peering으로 Cross-Region 연결도 지원한다. Multicast도 지원한다.
- **사용 시나리오:** 대규모 엔터프라이즈 환경에서 다수의 VPC를 중앙에서 관리할 때. 온프레미스와 다수 VPC를 동시에 연결할 때.
- **비용:** 시간당 요금 + 데이터 처리 요금이 발생하므로 소규모에서는 비용 대비 효율이 낮을 수 있다.

**PrivateLink (VPC Endpoint Service):**
- VPC 간 서비스 단위의 프라이빗 연결이다. 소비자 VPC에 ENI를 생성하여 프라이빗 IP로 서비스에 접근한다. CIDR 중복이 허용되며, 네트워크 전체가 아닌 특정 서비스만 노출한다. NLB 뒤에 있는 서비스를 다른 VPC/계정에 노출할 때 사용한다.
- **사용 시나리오:** SaaS 제공자가 고객 VPC에 서비스를 노출할 때. AWS 서비스(S3, DynamoDB 등)에 프라이빗 접근이 필요할 때(Gateway/Interface Endpoint). 보안 요구사항으로 네트워크 전체 연결이 아닌 서비스 단위 접근 제어가 필요할 때.
- **비용:** 시간당 요금 + 데이터 처리 요금.

**핵심 선택 기준:** VPC 수가 적고 단순한 연결이면 Peering, 대규모이면 Transit Gateway, 서비스 단위 노출이면 PrivateLink를 선택한다.

---

### Q3-3. AWS Organizations, SCP(Service Control Policy), IAM Policy의 관계를 설명하고, 대규모 조직에서 멀티 계정 전략을 수립하시오.

**모범답안:**

**AWS Organizations:**
- 다수의 AWS 계정을 중앙에서 관리하는 서비스이다. Organization Unit(OU) 기반 계층 구조로 계정을 그룹핑한다. 통합 결제(Consolidated Billing)로 볼륨 할인 혜택을 극대화한다.

**SCP (Service Control Policy):**
- Organization 또는 OU 수준에서 적용되는 권한의 상한선(Guardrail)이다. SCP는 권한을 부여하지 않으며, IAM에서 부여한 권한의 범위를 제한하는 역할만 한다. 즉 SCP와 IAM Policy의 교집합이 최종 유효 권한이다. Management Account에는 SCP가 적용되지 않으므로 주의해야 한다.

**IAM Policy:**
- 개별 계정 내에서 사용자, 그룹, 역할에 대한 구체적인 권한을 정의한다.

**멀티 계정 전략 (AWS Control Tower 기반):**

```
Root (Management Account)
├── Security OU
│   ├── Log Archive Account (CloudTrail, Config 로그 중앙 집중)
│   └── Security Audit Account (GuardDuty, Security Hub 중앙 관리)
├── Infrastructure OU
│   ├── Network Account (Transit Gateway, Direct Connect, DNS)
│   └── Shared Services Account (AD, CI/CD, Container Registry)
├── Sandbox OU
│   └── Developer Sandbox Accounts (실험용, SCP로 비용/리소스 제한)
├── Workloads OU
│   ├── Production OU
│   │   ├── App-A Prod Account
│   │   └── App-B Prod Account
│   └── Non-Production OU
│       ├── App-A Dev Account
│       └── App-A Staging Account
```

**SCP 적용 예시:**
- Root 수준: 특정 Region만 허용, CloudTrail 비활성화 차단
- Sandbox OU: 고비용 인스턴스 타입 차단, 특정 서비스만 허용
- Production OU: IAM User 생성 차단(SSO만 허용), VPC 삭제 차단

이 전략은 **Blast Radius 최소화**(장애 영향 범위 제한), **보안 경계 분리**, **비용 추적 용이성**, **규정 준수(Compliance)** 등의 이점을 제공한다.

---

### Q3-4. S3의 스토리지 클래스를 모두 나열하고, 각 클래스의 특성과 적합한 사용 사례를 설명하시오. Lifecycle Policy를 활용한 비용 최적화 전략도 포함하시오.

**모범답안:**

**S3 Standard:** 자주 액세스하는 데이터. 99.99% 가용성, 11 9s 내구성, 최소 3 AZ 복제. 비용이 가장 높지만 검색 비용 없음. 사용 사례: 활성 웹 콘텐츠, 빈번한 데이터 분석.

**S3 Intelligent-Tiering:** 액세스 패턴이 예측 불가한 데이터. 모니터링 및 자동 계층 전환 비용($0.0025/1,000 objects)이 발생하지만, 자동으로 Frequent/Infrequent/Archive Instant/Archive/Deep Archive 계층을 이동한다. 128KB 미만 객체는 항상 Frequent Access 계층에 저장된다.

**S3 Standard-IA (Infrequent Access):** 월 1회 미만 액세스하지만 즉시 검색이 필요한 데이터. Standard 대비 스토리지 비용 약 45% 저렴하지만 검색 비용이 발생한다. 최소 저장 기간 30일, 최소 객체 크기 128KB 과금. 사용 사례: 백업, DR 데이터.

**S3 One Zone-IA:** 단일 AZ에만 저장하여 Standard-IA 대비 약 20% 저렴. 가용성 99.5%. AZ 손실 시 데이터 유실 가능. 사용 사례: 재생성 가능한 데이터, 온프레미스 백업의 Secondary Copy.

**S3 Glacier Instant Retrieval:** 분기 1회 정도 액세스하는 장기 보관 데이터. 밀리초 단위 검색. Standard-IA 대비 스토리지 비용 약 68% 저렴. 최소 저장 기간 90일.

**S3 Glacier Flexible Retrieval:** 연 1~2회 액세스하는 아카이브 데이터. 검색 시간: Expedited(1~5분, 비용 높음), Standard(3~5시간), Bulk(5~12시간, 비용 낮음). 최소 저장 기간 90일. 사용 사례: 미디어 아카이브, 컴플라이언스 데이터.

**S3 Glacier Deep Archive:** 가장 저렴한 스토리지. 검색 시간: Standard(12시간), Bulk(48시간). 최소 저장 기간 180일. 사용 사례: 규정상 7~10년 보관 필요 데이터, 금융/의료 기록.

**Lifecycle Policy 전략 예시:**
```
생성 → 30일: Standard
30일 → 90일: Standard-IA
90일 → 365일: Glacier Instant Retrieval
365일 이후: Glacier Deep Archive
7년 후: 삭제(Expiration)
```

**주의:** Lifecycle 전환 시 최소 저장 기간을 위반하면 조기 삭제 요금이 부과된다. 작은 객체(128KB 미만)가 많은 경우 IA 클래스 전환이 오히려 비용을 증가시킬 수 있다.

---

## 4. AWS 리소스별 특장점, 제약사항, 주의사항

### Q4-1. EC2 인스턴스 유형별 특성을 설명하고, ML 학습, 웹 서버, 인메모리 캐시, 배치 처리 워크로드 각각에 적합한 인스턴스 패밀리를 추천하시오.

**모범답안:**

**범용(General Purpose) - M/T 시리즈:**
- M7i/M7g: 균형 잡힌 CPU, 메모리, 네트워크. 웹 서버, 앱 서버, 소규모 DB에 적합.
- T3/T4g: Burstable 인스턴스. CPU 크레딧 기반으로 기본 성능 이상 사용 시 크레딧 소모. 개발/테스트, 저부하 웹 서버에 적합. **주의:** Unlimited 모드 비활성화 시 크레딧 소진 후 성능 저하 발생.

**컴퓨팅 최적화(Compute Optimized) - C 시리즈:**
- C7i/C7g: 높은 CPU 성능. 배치 처리, 과학 모델링, 게임 서버, 비디오 인코딩에 적합.
- **배치 처리 추천:** C7g(Graviton3 기반, 가성비 우수) 또는 C7i.

**메모리 최적화(Memory Optimized) - R/X/Z 시리즈:**
- R7i/R7g: 높은 메모리:CPU 비율. SAP HANA, 대규모 캐시, 인메모리 DB에 적합.
- X2idn: 최대 3TB 메모리. 초대규모 인메모리 워크로드.
- **인메모리 캐시 추천:** R7g(Graviton3 기반) 또는 ElastiCache for Redis(관리형 서비스).

**가속 컴퓨팅(Accelerated Computing) - P/G/Inf/Trn 시리즈:**
- P5: NVIDIA H100 GPU. 대규모 ML 학습, 고성능 LLM 학습.
- P4d: NVIDIA A100 GPU. ML 학습, HPC.
- G5: NVIDIA A10G GPU. ML 추론, 그래픽 렌더링.
- Inf2: AWS Inferentia2 칩. ML 추론에 최적화, GPU 대비 비용 효율적.
- Trn1: AWS Trainium 칩. ML 학습에 최적화.
- **ML 학습 추천:** P5(최고 성능), Trn1(비용 효율), P4d(범용 학습).

**스토리지 최적화(Storage Optimized) - I/D/H 시리즈:**
- I4i: 높은 IOPS NVMe SSD. NoSQL DB(Cassandra, MongoDB), 데이터 웨어하우스.
- D3: 고밀도 HDD. 대용량 데이터 처리, Hadoop/HDFS.

**웹 서버 추천:** 트래픽이 예측 가능하고 지속적이면 M7g, 간헐적이면 T3/T4g.

**주요 주의사항:**
- Graviton(ARM) 기반 인스턴스는 x86 대비 약 20~40% 가격 대비 성능 우위가 있지만, ARM 호환성을 반드시 검증해야 한다.
- Spot Instance를 활용하면 On-Demand 대비 최대 90% 절감이 가능하지만, 2분 전 중단 알림에 대한 Graceful Shutdown 처리가 필요하다.

---

### Q4-2. EBS 볼륨 유형별 특성과 적합한 워크로드를 설명하고, EBS 성능 최적화 방안을 서술하시오.

**모범답안:**

**gp3 (General Purpose SSD):**
- 기본 성능: 3,000 IOPS, 125 MiB/s throughput (무료)
- 최대: 16,000 IOPS, 1,000 MiB/s (추가 비용으로 독립 조정 가능)
- gp2 대비 약 20% 저렴하며, IOPS와 throughput을 독립적으로 프로비저닝 가능
- 사용 사례: 범용 워크로드, 부트 볼륨, 개발/테스트 환경
- **gp2 → gp3 전환을 적극 권장** (대부분의 워크로드에서 비용 절감)

**gp2 (General Purpose SSD):**
- IOPS가 볼륨 크기에 비례 (3 IOPS/GB, 최소 100 ~ 최대 16,000)
- Burst Credit 메커니즘: 1TB 미만 볼륨은 3,000 IOPS로 버스트, 크레딧 소진 시 기본 IOPS로 하락
- **주의:** 크레딧 소진으로 인한 성능 저하를 CloudWatch의 `BurstBalance` 메트릭으로 모니터링해야 함

**io2 Block Express (Provisioned IOPS SSD):**
- 최대 256,000 IOPS, 4,000 MiB/s throughput
- 99.999% 내구성 (다른 EBS는 99.8~99.9%)
- Sub-millisecond 지연시간
- 사용 사례: 대규모 RDBMS(Oracle, SQL Server), 지연시간에 민감한 트랜잭션 워크로드

**io2 (Provisioned IOPS SSD):**
- 최대 64,000 IOPS
- IOPS:GB 비율 최대 500:1
- 사용 사례: 중규모 DB, I/O 집약적 워크로드

**st1 (Throughput Optimized HDD):**
- 최대 500 MiB/s throughput
- 부트 볼륨 불가
- 사용 사례: 빅데이터, 데이터 웨어하우스, 로그 처리 (순차 읽기/쓰기)

**sc1 (Cold HDD):**
- 최대 250 MiB/s throughput, 가장 저렴
- 사용 사례: 자주 접근하지 않는 대용량 데이터

**성능 최적화 방안:**
1. **EBS-Optimized Instance 사용:** 전용 네트워크 대역폭으로 EBS 트래픽과 일반 네트워크 트래픽을 분리한다.
2. **RAID 0 구성:** 여러 EBS 볼륨을 스트라이핑하여 IOPS와 throughput을 합산한다. 단, 내구성이 줄어들므로 RAID 1이나 정기 스냅샷과 병행한다.
3. **스냅샷 워밍:** 스냅샷에서 복원한 볼륨은 첫 접근 시 S3에서 데이터를 가져오므로 지연이 발생한다. `fio` 도구로 사전에 모든 블록을 읽어 워밍할 수 있다. 또는 Fast Snapshot Restore(FSR)를 활성화한다(추가 비용 발생).
4. **볼륨 크기 적정화:** gp2는 볼륨 크기에 따라 IOPS가 결정되므로, 필요한 IOPS에 맞게 볼륨 크기를 조정하거나 gp3로 전환한다.

---

### Q4-3. VPC의 Security Group과 NACL(Network ACL)의 차이를 설명하고, 실제 운영 환경에서의 설계 모범 사례를 제시하시오.

**모범답안:**

| 구분 | Security Group | NACL |
|------|---------------|------|
| 적용 단위 | ENI (인스턴스 레벨) | 서브넷 레벨 |
| 상태 | Stateful (반환 트래픽 자동 허용) | Stateless (인바운드/아웃바운드 별도 규칙 필요) |
| 규칙 유형 | 허용(Allow)만 가능 | 허용(Allow) + 거부(Deny) 가능 |
| 규칙 평가 | 모든 규칙을 평가하여 허용 여부 결정 | 규칙 번호 순서대로 평가, 첫 매칭 규칙 적용 |
| 기본 정책 | 인바운드 모두 거부, 아웃바운드 모두 허용 | 기본 NACL은 모두 허용, 커스텀 NACL은 모두 거부 |

**설계 모범 사례:**

**Security Group 설계:**
- 역할 기반으로 Security Group을 분리한다 (예: sg-web, sg-app, sg-db).
- Source에 IP 대신 Security Group ID를 참조하여 체이닝한다: sg-web → sg-app → sg-db. 이렇게 하면 인스턴스 IP 변경에 무관하게 동작한다.
- 0.0.0.0/0 인바운드는 ALB/NLB의 Security Group에만 허용하고, 나머지는 필요한 소스만 지정한다.
- 아웃바운드도 필요한 포트/대상만 허용하여 데이터 유출을 방지한다.
- Security Group 규칙 수 제한(기본 60 인바운드 + 60 아웃바운드)에 주의하고, Prefix List를 활용하여 규칙 수를 줄인다.

**NACL 설계:**
- NACL은 서브넷 경계의 추가 방어 계층으로 사용한다 (Defense in Depth).
- 특정 IP 대역 차단(예: 알려진 악성 IP)에 NACL의 Deny 규칙을 활용한다.
- Stateless이므로 Ephemeral Port(1024-65535) 아웃바운드를 반드시 허용해야 응답 트래픽이 반환된다.
- 규칙 번호는 100 단위로 증가시켜 나중에 규칙 삽입 여지를 남긴다 (100, 200, 300...).
- 과도하게 복잡한 NACL 규칙은 관리 부담을 증가시키므로, 1차 방어는 Security Group에서 처리하고 NACL은 보조적으로 사용한다.

---

### Q4-4. ALB, NLB, GWLB(Gateway Load Balancer)의 차이를 설명하고, 각각의 적합한 사용 시나리오를 제시하시오.

**모범답안:**

**ALB (Application Load Balancer) - Layer 7:**
- HTTP/HTTPS/gRPC/WebSocket 프로토콜 지원
- Path 기반 라우팅 (/api/* → 서비스A, /web/* → 서비스B)
- Host 기반 라우팅 (api.example.com → 서비스A)
- HTTP 헤더, 쿼리 스트링, 소스 IP 기반 고급 라우팅
- SSL Termination, Sticky Session, 사용자 인증(Cognito/OIDC 연동)
- 대상 그룹: EC2, IP, Lambda, ECS 컨테이너
- **사용 시나리오:** 마이크로서비스 라우팅, 웹 애플리케이션, REST API, gRPC 서비스
- **제약:** TCP/UDP 직접 로드밸런싱 불가, NLB 대비 지연시간 약간 높음

**NLB (Network Load Balancer) - Layer 4:**
- TCP, UDP, TLS 프로토콜 지원
- 초저지연(~100μs), 초고성능(수백만 RPS)
- Static IP / Elastic IP 할당 가능 (화이트리스트 기반 방화벽 환경에서 유용)
- Client IP 보존 (ALB는 X-Forwarded-For 헤더 사용)
- Cross-zone Load Balancing이 기본적으로 비활성화
- PrivateLink와 연동하여 서비스를 다른 VPC에 노출
- **사용 시나리오:** 게임 서버, IoT, 금융 거래 시스템, gRPC(비HTTP), VoIP, 고정 IP 필요 시, PrivateLink 제공자
- **제약:** Layer 7 라우팅(Path, Host 기반) 불가

**GWLB (Gateway Load Balancer) - Layer 3:**
- GENEVE 프로토콜(포트 6081) 사용
- 트래픽을 투명하게 3rd Party 네트워크 어플라이언스(방화벽, IDS/IPS, DPI)로 전달
- 패킷의 원본 소스/대상 IP를 보존하며 bump-in-the-wire 방식으로 동작
- Gateway Load Balancer Endpoint를 통해 VPC 간 트래픽 검사
- **사용 시나리오:** Palo Alto, Fortinet 등 3rd Party 방화벽을 통한 트래픽 검사, 중앙 집중식 네트워크 보안 어플라이언스 관리
- **제약:** 어플라이언스가 GENEVE 프로토콜을 지원해야 함

**Cross-zone Load Balancing 주의:**
- ALB: 기본 활성화, 비활성화 가능
- NLB: 기본 비활성화, 활성화 시 Cross-AZ 데이터 전송 비용 발생
- GWLB: 기본 비활성화

---

### Q4-5. NAT Gateway와 NAT Instance의 차이를 설명하고, 비용 최적화 관점에서의 NAT Gateway 대안을 제시하시오.

**모범답안:**

**NAT Gateway (관리형):**
- AWS 관리형 서비스로 고가용성 내장(단일 AZ 내, Multi-AZ는 AZ별 배포 필요)
- 최대 100Gbps 대역폭으로 자동 확장
- Security Group 연결 불가, NACL로만 트래픽 제어
- 비용: 시간당 요금($0.045/hr, us-east-1) + 데이터 처리 요금($0.045/GB)
- 포트 포워딩 불가, Bastion Host로 사용 불가

**NAT Instance (자체 관리):**
- EC2 인스턴스 기반, Source/Destination Check 비활성화 필요
- Security Group 적용 가능, 포트 포워딩 가능
- 단일 인스턴스이므로 SPOF(Single Point of Failure)
- 수동 확장, 인스턴스 크기에 따른 대역폭 제한
- 비용: EC2 인스턴스 비용만 발생 (소규모 시 NAT GW보다 저렴)

**비용 최적화 대안:**

1. **VPC Endpoint (Gateway/Interface):** S3, DynamoDB 등 AWS 서비스 접근 시 NAT Gateway를 경유하지 않도록 Gateway Endpoint(무료)를 설정한다. ECR, CloudWatch Logs 등 자주 사용하는 서비스에 Interface Endpoint를 설정하여 NAT Gateway 데이터 처리 비용을 절감한다.

2. **NAT Instance(t4g.nano):** 개발/테스트 환경에서 트래픽이 적은 경우, t4g.nano Spot Instance를 NAT Instance로 사용하면 월 $1~3 수준으로 운영 가능하다 (NAT GW는 월 $32+).

3. **IPv6 활용:** Egress-Only Internet Gateway(무료)를 통해 IPv6 아웃바운드 트래픽을 처리한다. IPv6를 지원하는 서비스와의 통신에서 NAT Gateway 비용을 절감할 수 있다.

4. **VPC 아키텍처 최적화:** 불필요한 Cross-AZ NAT Gateway 트래픽을 줄이기 위해 각 AZ에 NAT Gateway를 배치하고, 라우팅 테이블에서 같은 AZ의 NAT Gateway를 사용하도록 설정한다.

---

## 5. AWS RBAC, IAM, Security, AI 기능

### Q5-1. IAM Policy의 평가 로직(Evaluation Logic)을 상세히 설명하고, 명시적 거부(Explicit Deny), 묵시적 거부(Implicit Deny), 허용(Allow)의 관계를 서술하시오.

**모범답안:**

AWS IAM Policy 평가 로직은 다음 순서로 결정된다.

**Step 1 - 명시적 거부(Explicit Deny) 확인:** 적용 가능한 모든 정책에서 하나라도 명시적 Deny가 있으면, 다른 Allow와 관계없이 최종 결과는 **거부**이다. 이것이 최우선 원칙이다.

**Step 2 - SCP(Organizations) 확인:** Organizations SCP에서 Allow가 없으면 **거부**. SCP는 권한의 상한선 역할을 한다.

**Step 3 - Resource-based Policy 확인:** S3 Bucket Policy, KMS Key Policy 등 리소스 기반 정책에서 Allow가 있으면, Identity-based Policy의 Allow 없이도 접근이 허용될 수 있다 (같은 계정 내, IAM 역할은 예외).

**Step 4 - Identity-based Policy 확인:** IAM User/Role에 연결된 정책에서 Allow를 확인한다.

**Step 5 - Permission Boundary 확인:** Permission Boundary가 설정된 경우, Identity-based Policy와 Permission Boundary의 교집합만 허용된다.

**Step 6 - Session Policy 확인:** AssumeRole 시 전달된 Session Policy가 있으면, Role의 정책과 Session Policy의 교집합만 허용된다.

**최종 결정:** 위 모든 단계에서 Allow를 통과하지 못하면 **묵시적 거부(Implicit Deny)**로 최종 거부된다.

**실무 예시:**
```json
// SCP: EC2 허용, IAM Policy: EC2 + S3 허용 → 최종: EC2만 허용 (교집합)
// IAM Policy: S3 Allow, S3 Bucket Policy: Deny → 최종: 거부 (명시적 Deny 우선)
// Permission Boundary: S3 + EC2, IAM Policy: S3 + RDS → 최종: S3만 허용 (교집합)
```

**Cross-Account 접근 시:** Resource-based Policy에서 Allow하더라도, 호출자의 계정 IAM Policy에서도 Allow가 있어야 접근이 허용된다 (양쪽 모두 Allow 필요).

---

### Q5-2. AWS에서 최소 권한 원칙(Least Privilege)을 실현하기 위한 구체적인 방법과 도구를 설명하시오.

**모범답안:**

**1. IAM Access Analyzer 활용:**
- Policy Generation: CloudTrail 로그를 분석하여 실제 사용된 서비스/액션만 포함하는 최소 권한 정책을 자동 생성한다.
- Policy Validation: IAM 정책의 문법 오류, 보안 경고, 과도한 권한을 자동 검출한다.
- External Access Finding: S3 버킷, IAM 역할 등이 외부 엔터티에 접근 가능한 경우 알림을 제공한다.
- Unused Access Finding: 미사용 역할, 미사용 권한, 미사용 Access Key를 탐지한다.

**2. Permission Boundary 활용:**
- 개발자에게 IAM 역할 생성 권한을 위임하되, 생성되는 역할의 최대 권한을 Permission Boundary로 제한한다. 이를 통해 권한 상승(Privilege Escalation)을 방지한다.

**3. IAM 정책 설계 원칙:**
- 와일드카드(`*`)를 최소화하고, 구체적인 Action과 Resource ARN을 명시한다.
- Condition 키를 활용하여 접근 조건을 세분화한다. 예: `aws:SourceIp`, `aws:RequestedRegion`, `aws:PrincipalTag`, `ec2:ResourceTag`.
- AWS 관리형 정책보다 고객 관리형 정책을 사용하여 필요한 권한만 포함한다.

**4. IAM Role 우선 사용:**
- IAM User의 장기 자격 증명(Access Key) 대신 IAM Role의 임시 자격 증명(STS)을 사용한다.
- EC2에는 Instance Profile, Lambda에는 Execution Role, ECS에는 Task Role을 사용한다.
- 사람이 AWS에 접근할 때는 IAM Identity Center(SSO)를 통해 임시 자격 증명을 발급한다.

**5. 정기적 감사:**
- IAM Credential Report로 모든 사용자의 자격 증명 상태를 확인한다.
- Access Advisor로 각 서비스의 마지막 접근 시간을 확인하고, 미사용 권한을 제거한다.
- CloudTrail 로그를 Athena로 분석하여 실제 API 호출 패턴을 파악한다.

**6. Condition 키 활용 고급 예시:**
```json
{
  "Condition": {
    "StringEquals": {
      "aws:RequestedRegion": ["ap-northeast-2", "us-east-1"],
      "ec2:ResourceTag/Environment": "Production"
    },
    "Bool": { "aws:MultiFactorAuthPresent": "true" },
    "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
  }
}
```

---

### Q5-3. AWS에서 데이터 암호화 전략을 설명하시오. KMS(Key Management Service)의 CMK 유형, 키 계층 구조, Envelope Encryption의 동작 원리를 포함하시오.

**모범답안:**

**데이터 암호화 유형:**
- **전송 중 암호화(Encryption in Transit):** TLS 1.2/1.3을 사용하여 클라이언트-서버, 서비스 간 통신을 암호화한다. ACM(Certificate Manager)으로 SSL/TLS 인증서를 관리하고 ALB/CloudFront에 적용한다.
- **저장 시 암호화(Encryption at Rest):** S3 SSE, EBS Encryption, RDS Encryption 등 스토리지 레벨 암호화. KMS 또는 CloudHSM으로 키를 관리한다.

**KMS Key 유형:**
- **AWS Owned Keys:** AWS가 내부적으로 관리하는 키. 고객이 직접 관리/확인 불가. 무료. (예: S3 SSE-S3의 기본 암호화)
- **AWS Managed Keys (aws/service-name):** 서비스별 자동 생성 키. 매년 자동 로테이션. 고객이 키 정책 수정 불가. 비용: 무료(사용 요금만 발생).
- **Customer Managed Keys (CMK):** 고객이 생성/관리하는 키. 키 정책, 로테이션 주기, 활성화/비활성화 제어 가능. 연 $1/key + API 호출 비용.
- **CloudHSM:** FIPS 140-2 Level 3 인증 HSM. 고객이 전용 HSM 하드웨어를 완전히 제어. 금융, 의료 등 높은 컴플라이언스 요구 시 사용.

**Envelope Encryption 동작 원리:**
1. KMS에 `GenerateDataKey` API를 호출하면 평문 Data Key와 암호화된 Data Key가 반환된다.
2. 평문 Data Key로 데이터를 암호화(AES-256)한다.
3. 평문 Data Key는 즉시 메모리에서 삭제한다.
4. 암호화된 Data Key를 암호화된 데이터와 함께 저장한다.
5. 복호화 시 암호화된 Data Key를 KMS에 `Decrypt` API로 보내 평문 Data Key를 얻고, 이를 사용하여 데이터를 복호화한다.

**Envelope Encryption을 사용하는 이유:** KMS는 4KB 이하 데이터만 직접 암호화할 수 있으므로, 대용량 데이터는 로컬에서 Data Key로 암호화하고 Data Key만 KMS로 보호하여 네트워크 오버헤드를 최소화한다.

**키 로테이션:**
- AWS Managed Key: 자동 1년 주기 로테이션
- Customer Managed Key: 자동 로테이션 활성화 가능(1년~). 수동 로테이션 시 새 Key 생성 후 Alias를 전환하는 방식 사용.
- 이전 키 자료(Backing Key)는 자동 보관되어 과거 데이터 복호화가 가능하다.

---

### Q5-4. Amazon Bedrock과 SageMaker의 차이를 설명하고, 기업에서 생성형 AI를 도입할 때의 아키텍처를 설계하시오.

**모범답안:**

**Amazon Bedrock:**
- Foundation Model(FM)에 대한 API 기반 서비스(Serverless)
- Claude(Anthropic), Llama(Meta), Titan(Amazon), Stable Diffusion 등 다양한 FM 제공
- 인프라 관리 불필요, API 호출로 즉시 사용
- Knowledge Base(RAG), Agents, Guardrails, Fine-tuning, Model Evaluation 기능 제공
- 데이터가 모델 학습에 사용되지 않으며, VPC 내에서 프라이빗 접근 가능(PrivateLink)
- **적합한 경우:** FM을 그대로 또는 경량 커스터마이징(Fine-tuning, RAG)하여 사용할 때, 빠른 프로토타이핑, 인프라 관리 부담 최소화

**Amazon SageMaker:**
- 커스텀 ML 모델의 전체 생명주기 관리 플랫폼 (데이터 준비 → 학습 → 배포 → 모니터링)
- SageMaker Studio, Notebooks, Processing Jobs, Training Jobs, Endpoints
- 자체 모델을 처음부터 학습하거나, 오픈소스 모델(Hugging Face 등)을 커스텀 학습
- GPU/Inf/Trn 인스턴스 직접 선택하여 리소스 완전 제어
- **적합한 경우:** 독자적 모델 학습, 복잡한 ML 파이프라인, 정밀한 리소스 제어가 필요할 때

**기업용 생성형 AI 아키텍처 (RAG 패턴):**

```
사용자 → API Gateway → Lambda → Bedrock (Embedding Model)
                                        ↓
                          OpenSearch Serverless / Aurora pgvector
                          (Vector DB - 사내 문서 임베딩 저장)
                                        ↓
                          Bedrock (Claude) - 검색 결과 + 질문으로 답변 생성
                                        ↓
                          Bedrock Guardrails (부적절한 응답 필터링)
                                        ↓
                          응답 반환
```

**데이터 파이프라인:**
- S3에 사내 문서(PDF, 문서) 저장
- Bedrock Knowledge Base 또는 Lambda + Bedrock Embedding API로 문서를 벡터화
- OpenSearch Serverless 또는 Aurora PostgreSQL(pgvector)에 벡터 저장
- 문서 업데이트 시 EventBridge → Step Functions로 자동 재인덱싱

**보안 고려사항:**
- PrivateLink를 통한 Bedrock API 프라이빗 접근
- IAM 기반 API 호출 인가
- CloudTrail을 통한 모델 호출 감사 로그
- Bedrock Guardrails로 유해 콘텐츠, 민감 정보(PII) 필터링
- 데이터 소유권: Bedrock에 전송된 데이터는 모델 학습에 사용되지 않음

---

### Q5-5. AWS GuardDuty, Security Hub, Inspector, Detective의 역할을 각각 설명하고, 통합 보안 운영 아키텍처를 설계하시오.

**모범답안:**

**GuardDuty (위협 탐지):**
- CloudTrail 로그, VPC Flow Logs, DNS 로그, EKS Audit 로그, S3 데이터 이벤트, RDS 로그인 활동, Lambda 네트워크 활동, Runtime Monitoring을 분석하여 위협을 탐지한다.
- ML 기반 이상 탐지: 비정상 API 호출, 암호화폐 마이닝, 무차별 대입 공격, C&C 서버 통신 등을 감지한다.
- 에이전트 설치 불필요(Agentless), 활성화만 하면 동작한다.

**Security Hub (보안 상태 통합 대시보드):**
- GuardDuty, Inspector, IAM Access Analyzer, Firewall Manager 등의 보안 발견 사항을 중앙에서 집계한다.
- AWS Foundational Security Best Practices, CIS Benchmark, PCI DSS 등 보안 표준 대비 자동 평가를 수행한다.
- Multi-Account, Multi-Region 보안 상태를 단일 뷰로 제공한다.

**Inspector (취약성 평가):**
- EC2 인스턴스, Lambda 함수, ECR 컨테이너 이미지의 소프트웨어 취약성(CVE)과 네트워크 노출을 자동 스캔한다.
- SSM Agent 기반으로 동작하며, 패키지 매니저의 설치된 소프트웨어 목록을 분석한다.
- 심각도별 분류와 AWS Lambda 함수의 코드 취약성 스캔도 지원한다.

**Detective (보안 조사):**
- GuardDuty 탐지 결과의 근본 원인을 조사하기 위한 도구이다.
- CloudTrail, VPC Flow Logs, GuardDuty 탐지 결과를 자동으로 수집하여 그래프 기반 분석을 제공한다.
- IP 주소, IAM 역할, EC2 인스턴스의 활동 이력을 시각화하여 보안 인시던트를 분석한다.

**통합 보안 운영 아키텍처:**
```
전체 계정 → GuardDuty (위협 탐지)  ─┐
         → Inspector (취약성 스캔) ─┤
         → IAM Access Analyzer    ─┤→ Security Hub (중앙 집계)
         → Config Rules           ─┘         │
                                              ↓
                                     EventBridge (자동 대응 트리거)
                                       ↓           ↓
                                  Lambda         SNS
                              (자동 격리/차단)   (알림)
                                                  ↓
                                     Slack/PagerDuty/이메일
                                              │
                              Detective ← (수동 조사 필요 시)
```

Security Hub의 Findings를 EventBridge로 전달하여 자동 대응을 구현한다. 예를 들어 GuardDuty에서 Compromised EC2 인스턴스를 탐지하면, Lambda가 자동으로 해당 인스턴스의 Security Group을 격리용 SG로 교체하고, EBS 스냅샷을 생성하여 포렌식용으로 보존한다.

---

## 6. EKS & Kubernetes

### Q6-1. EKS에서 Worker Node의 3가지 옵션(Managed Node Group, Self-Managed Node, Fargate)을 비교하고, 각각의 적합한 사용 시나리오를 설명하시오.

**모범답안:**

**Managed Node Group:**
- AWS가 EC2 인스턴스의 프로비저닝과 생명주기를 관리한다.
- Amazon Linux 2/Bottlerocket AMI 기반, AMI 업데이트 시 Rolling Update를 자동 수행한다.
- Auto Scaling Group이 자동 구성되며, Cluster Autoscaler/Karpenter와 연동 가능하다.
- GPU 인스턴스, Custom Launch Template 지원
- **적합한 시나리오:** 대부분의 일반적인 워크로드. 관리 부담을 줄이면서 EC2 인스턴스 수준의 제어가 필요한 경우.

**Self-Managed Node:**
- 고객이 직접 EC2 인스턴스를 프로비저닝하고 EKS 클러스터에 조인시킨다.
- Custom AMI(특수 커널, 보안 에이전트 등) 사용이 필요한 경우.
- Windows 컨테이너 노드, Bottlerocket 이외의 OS 사용 시.
- AMI 업데이트, 패치, 드레인을 직접 관리해야 한다.
- **적합한 시나리오:** 특수 AMI 요구사항, 엄격한 보안/컴플라이언스 요건으로 OS 레벨 커스터마이징이 필수인 경우.

**Fargate:**
- 서버리스 컨테이너 실행. 노드 관리가 완전히 불필요하다.
- Pod별 전용 microVM에서 실행되어 Pod 수준 격리를 제공한다.
- DaemonSet 미지원 (로그 수집은 Fluent Bit sidecar로 대체).
- EBS 볼륨 미지원 (EFS만 지원), GPU 미지원.
- Pod별 최대 4 vCPU, 30GB 메모리 제한.
- Cold Start 지연(수십 초)이 발생할 수 있다.
- **적합한 시나리오:** 배치 작업, 간헐적 워크로드, 보안 격리가 중요한 워크로드, 인프라 관리 인력이 부족한 경우.

**하이브리드 구성 권장:** 상시 실행 워크로드는 Managed Node Group(Reserved Instance/Savings Plan 적용), 간헐적 배치 워크로드는 Fargate, GPU 워크로드는 Managed Node Group(GPU 인스턴스)으로 구성하는 것이 비용 최적화에 유리하다.

---

### Q6-2. EKS에서 Karpenter와 Cluster Autoscaler의 차이를 설명하고, Karpenter의 동작 원리와 장점을 서술하시오.

**모범답안:**

**Cluster Autoscaler (CA):**
- Kubernetes 공식 오토스케일러로, Auto Scaling Group(ASG)을 조정하여 노드를 확장/축소한다.
- Node Group 단위로 동작하며, 각 Node Group에 인스턴스 타입, AZ 등이 사전 정의되어야 한다.
- Pending Pod를 감지하면 적합한 Node Group의 ASG Desired Count를 증가시킨다.
- 확장 속도가 상대적으로 느리다 (ASG → EC2 시작 → 노드 조인).
- 다양한 인스턴스 타입을 지원하려면 여러 Node Group을 미리 구성해야 한다.

**Karpenter:**
- AWS에서 개발한 오픈소스 Kubernetes 노드 프로비저너이다.
- ASG를 사용하지 않고 EC2 Fleet API를 직접 호출하여 노드를 프로비저닝한다.
- **핵심 개념:** NodePool(구 Provisioner)과 EC2NodeClass(구 AWSNodeTemplate)로 구성.

**Karpenter 동작 원리:**
1. Pending Pod를 감지하면 Pod의 리소스 요청(CPU, 메모리, GPU), nodeSelector, Affinity, Tolerations를 분석한다.
2. EC2 인스턴스 타입 목록에서 Pod 요구사항에 최적인 인스턴스를 자동 선택한다 (right-sizing).
3. EC2 Fleet API로 인스턴스를 직접 생성한다.
4. Spot Instance를 자동으로 활용하며, Spot 중단 시 빠르게 대체 노드를 프로비저닝한다.

**Karpenter의 장점:**
- **빠른 확장:** ASG를 거치지 않아 노드 프로비저닝이 빠르다 (일반적으로 60초 이내).
- **효율적 Bin-packing:** Pod 요구사항에 정확히 맞는 인스턴스를 선택하여 리소스 낭비를 최소화한다.
- **자동 Consolidation:** 유휴 노드 또는 비효율적으로 사용되는 노드를 감지하여 워크로드를 재배치하고 노드를 제거한다.
- **Drift Detection:** AMI, Security Group 등의 변경을 감지하여 노드를 자동으로 교체한다.
- **Just-in-Time 프로비저닝:** 사전에 Node Group을 정의할 필요 없이, 워크로드 요구에 따라 동적으로 인스턴스 타입을 선택한다.

**주의사항:**
- Karpenter 자체는 중요 시스템 컴포넌트이므로, Karpenter Pod는 Fargate 또는 소규모 Managed Node Group에서 실행하여 자기 자신이 관리하는 노드에 의존하지 않도록 해야 한다.
- Spot Instance 비율이 높은 경우 대규모 회수(Reclaim) 이벤트에 대비한 PDB(Pod Disruption Budget) 설정이 필요하다.

---

### Q6-3. EKS에서의 네트워킹을 설명하시오. VPC CNI Plugin의 동작 원리, Pod IP 할당 방식, IP 주소 고갈 문제와 해결 방안을 포함하시오.

**모범답안:**

**VPC CNI Plugin 동작 원리:**
- AWS VPC CNI는 EKS의 기본 네트워크 플러그인으로, 각 Pod에 VPC 서브넷의 실제 IP 주소를 할당한다.
- 각 EC2 인스턴스의 ENI(Elastic Network Interface)에 Secondary IP를 할당하고, 이를 Pod에 부여한다.
- Pod IP가 VPC IP이므로 VPC 내 다른 리소스(RDS, ElastiCache 등)와 NAT 없이 직접 통신할 수 있다.

**Pod IP 할당 방식:**
- 인스턴스 타입에 따라 연결 가능한 ENI 수와 ENI당 IP 수가 결정된다.
- 예: m5.large = 3 ENIs × 10 IPs/ENI - 3(Primary IPs) = 27 Pods/Node (kube-proxy, aws-node 등 시스템 Pod 포함)
- 공식: (ENI 수 × ENI당 IP 수) - ENI 수 = 최대 Pod 수

**IP 주소 고갈 문제:**
- 대규모 클러스터에서 수천 개의 Pod가 VPC IP를 사용하면 서브넷의 IP가 고갈될 수 있다.
- /24 서브넷은 251개 IP만 사용 가능하므로, 대규모 워크로드에는 매우 부족하다.

**해결 방안:**

1. **Prefix Delegation 모드:** ENI에 /28 prefix(16 IPs)를 할당하여 노드당 Pod 수를 대폭 증가시킨다. 예: m5.large에서 27 → 110+ Pods. `ENABLE_PREFIX_DELEGATION=true` 환경변수로 활성화한다.

2. **Secondary CIDR 추가:** VPC에 추가 CIDR 블록(100.64.0.0/16 등)을 연결하고, 이 대역을 Pod 전용으로 사용한다. `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true`로 활성화한다.

3. **IPv6 활용:** EKS에서 IPv6 모드를 활성화하면 거의 무한한 IP 주소를 사용할 수 있다. 단, 모든 워크로드가 IPv6를 지원하는지 검증이 필요하다.

4. **대안 CNI:** Calico, Cilium 등 오버레이 네트워크 기반 CNI를 사용하면 VPC IP를 소모하지 않는다. 다만 VPC 네이티브 통합(Security Group for Pods, VPC Flow Logs 등)의 이점을 잃게 된다.

**Security Group for Pods:**
- VPC CNI의 `ENABLE_POD_ENI=true` 설정으로 개별 Pod에 Security Group을 적용할 수 있다.
- Pod가 RDS 등 VPC 리소스에 접근할 때 Security Group으로 네트워크 접근을 제어할 수 있어 보안이 강화된다.

---

### Q6-4. EKS에서 IRSA(IAM Roles for Service Accounts)의 동작 원리를 설명하고, Pod Identity와의 차이를 서술하시오.

**모범답안:**

**IRSA (IAM Roles for Service Accounts) 동작 원리:**

1. EKS 클러스터에 OIDC Provider를 연결한다 (클러스터 생성 시 자동 또는 수동 설정).
2. IAM Role을 생성하고, Trust Policy에 EKS OIDC Provider를 지정한다. 특정 Namespace/ServiceAccount만 AssumeRole 할 수 있도록 조건을 설정한다.
3. Kubernetes ServiceAccount에 `eks.amazonaws.com/role-arn` 어노테이션으로 IAM Role ARN을 지정한다.
4. Pod가 해당 ServiceAccount로 실행되면, EKS가 자동으로 AWS STS 토큰을 Pod에 마운트한다 (`/var/run/secrets/eks.amazonaws.com/serviceaccount/token`).
5. AWS SDK가 이 토큰을 사용하여 `sts:AssumeRoleWithWebIdentity`를 호출하고 임시 자격 증명을 획득한다.

```yaml
# ServiceAccount 예시
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
```

**EKS Pod Identity (2023년 출시):**
- IRSA의 단순화된 대안으로, OIDC Provider 설정이 불필요하다.
- EKS Pod Identity Agent(DaemonSet)가 클러스터에 설치된다.
- AWS Console/CLI에서 ServiceAccount와 IAM Role의 매핑을 직접 설정한다 (IAM Trust Policy 수정 불필요).
- `eks:AssumeRoleForPodIdentity` 액션을 사용한다.

**IRSA vs Pod Identity 비교:**

| 구분 | IRSA | Pod Identity |
|------|------|-------------|
| OIDC Provider | 필요 | 불필요 |
| IAM Trust Policy | 수동 관리 (Namespace/SA 조건 추가) | AWS에서 자동 관리 |
| Cross-Account | Trust Policy 수정 | 간단한 Association 설정 |
| Role Chaining | 미지원 | 지원 (세션 태그 전파) |
| 기존 호환성 | 모든 AWS SDK 지원 | 최신 SDK 필요 |
| 설정 복잡도 | 높음 | 낮음 |

**권장 사항:** 신규 클러스터에서는 Pod Identity를 사용하는 것이 관리가 간편하다. 기존 IRSA를 사용 중인 경우 점진적으로 Pod Identity로 마이그레이션을 고려한다. 두 방식을 동시에 사용할 수 있다.

---

### Q6-5. EKS 클러스터 업그레이드 전략을 설명하시오. In-place 업그레이드와 Blue/Green 업그레이드의 장단점, 업그레이드 시 주의사항을 포함하시오.

**모범답안:**

**Kubernetes 버전 정책:**
- EKS는 동시에 최소 4개의 Kubernetes 마이너 버전을 지원한다.
- EKS는 EOL(End of Life) 이후 자동으로 Extended Support 모드에 진입하며, 추가 비용($0.60/hr per cluster)이 발생한다.
- 한 번에 1개 마이너 버전씩만 업그레이드 가능 (1.28 → 1.29 → 1.30).

**In-place 업그레이드:**

*절차:*
1. Control Plane 업그레이드 (EKS 관리형, ~25분 소요, 다운타임 없음)
2. Add-on 업그레이드 (VPC CNI, CoreDNS, kube-proxy, EBS CSI Driver 등)
3. Worker Node 업그레이드 (Rolling Update)
   - Managed Node Group: `eksctl upgrade nodegroup` 또는 새 Launch Template 적용
   - Karpenter: NodePool의 AMI 변경 → Drift Detection으로 자동 교체

*장점:* 단일 클러스터 관리, DNS/엔드포인트 변경 불필요
*단점:* 롤백이 어려움(특히 Control Plane), 업그레이드 중 호환성 이슈 시 영향 범위가 큼

**Blue/Green 업그레이드:**

*절차:*
1. 새 버전의 EKS 클러스터(Green)를 생성한다.
2. 동일한 워크로드를 Green 클러스터에 배포한다.
3. 테스트 완료 후 Route 53 또는 ALB 가중치 라우팅으로 트래픽을 점진적으로 전환한다.
4. 문제 발생 시 즉시 Blue 클러스터로 롤백한다.
5. 안정화 후 Blue 클러스터를 삭제한다.

*장점:* 안전한 롤백, 충분한 테스트 시간 확보
*단점:* 일시적으로 2배의 인프라 비용, 상태 저장 워크로드(PV)의 마이그레이션 복잡성, CI/CD 파이프라인 복잡성 증가

**업그레이드 주의사항:**
- **API Deprecation 확인:** `kubectl convert` 또는 `pluto` 도구로 deprecated API 사용 여부를 사전 점검한다.
- **Add-on 호환성:** VPC CNI, CoreDNS 등의 호환 버전을 EKS 문서에서 확인한다.
- **PDB(Pod Disruption Budget):** 업그레이드 시 동시에 종료되는 Pod 수를 제한하여 서비스 가용성을 보장한다.
- **Helm Chart / Operator 호환성:** 사용 중인 3rd Party Helm Chart, Operator의 새 Kubernetes 버전 호환성을 확인한다.
- **IRSA/Pod Identity:** Control Plane 업그레이드 시 OIDC Provider는 유지되므로 IRSA에 영향 없다.
- **Staging 환경에서 먼저 업그레이드:** Production 전에 반드시 비프로덕션 환경에서 전체 프로세스를 검증한다.

---

## 7. AWS 비용 최적화

### Q7-1. EC2 구매 옵션(On-Demand, Reserved Instance, Savings Plan, Spot Instance, Dedicated Host)을 비교하고, 워크로드 유형별 최적의 구매 전략을 수립하시오.

**모범답안:**

**On-Demand:**
- 시간/초 단위 과금, 약정 없음, 즉시 사용 가능.
- 가장 비쌈. 단기, 예측 불가능, 중단 불가 워크로드에 적합.

**Reserved Instance (RI):**
- 1년/3년 약정으로 On-Demand 대비 최대 72% 할인.
- Standard RI: 인스턴스 타입/Region 고정, 할인율 높음. Marketplace에서 매매 가능.
- Convertible RI: 인스턴스 타입/OS/Tenancy 변경 가능, Standard 대비 할인율 낮음.
- 결제 옵션: 전액 선결제(최대 할인) > 부분 선결제 > 선결제 없음(최소 할인).
- **주의:** Savings Plan이 RI보다 유연하므로, 신규 약정 시 Savings Plan을 먼저 검토한다.

**Savings Plan:**
- 시간당 일정 금액($X/hr)의 컴퓨팅 사용을 약정. RI보다 유연.
- Compute Savings Plan: EC2, Fargate, Lambda에 적용. Region, 인스턴스 패밀리, OS 변경 자유.
- EC2 Instance Savings Plan: 특정 Region/인스턴스 패밀리 고정. Compute SP보다 할인율 높음.
- **추천:** 안정적 기본 사용량은 EC2 Instance SP, 유연성이 필요한 부분은 Compute SP로 커버.

**Spot Instance:**
- On-Demand 대비 최대 90% 할인. 2분 전 중단 알림.
- **적합한 워크로드:** 배치 처리, CI/CD 빌드, 데이터 분석, ML 학습(체크포인트 저장), 스테이트리스 웹 서버(Auto Scaling과 함께).
- **전략:** Spot Fleet으로 다양한 인스턴스 타입/AZ에 분산 요청. Capacity-optimized 할당 전략 사용. On-Demand 인스턴스와 혼합 구성.

**Dedicated Host:**
- 물리 서버를 전용으로 할당. 소켓/코어 기반 소프트웨어 라이선스(Oracle, Windows Server 등)가 필요한 경우.
- 규정 준수(컴플라이언스)로 물리적 격리가 필요한 경우.
- 가장 비쌈.

**워크로드별 최적 전략:**
- **상시 운영 Production 서버:** Savings Plan(기본 용량 70~80%) + On-Demand(피크 시 추가 용량)
- **배치 처리/데이터 분석:** Spot Instance(Primary) + On-Demand(Fallback)
- **개발/테스트 환경:** Spot Instance + 비업무 시간 자동 중지(Instance Scheduler)
- **라이선스 서버:** Dedicated Host + RI
- **간헐적 이벤트 트래픽:** Auto Scaling + On-Demand + Spot 혼합

---

### Q7-2. AWS 비용 가시성(Cost Visibility)을 확보하기 위한 전략과 도구를 설명하시오.

**모범답안:**

**태깅 전략 (Tagging Strategy):**
- 비용 배분 태그(Cost Allocation Tags)를 조직 표준으로 정의한다.
- 필수 태그 예시: `Environment` (prod/staging/dev), `Team`, `Project`, `CostCenter`, `Owner`
- AWS Organizations의 Tag Policy로 태그 값을 표준화한다.
- SCP로 태그가 없는 리소스 생성을 차단할 수 있다 (`aws:RequestTag` 조건 활용).
- Config Rules로 태그 미준수 리소스를 탐지한다.

**AWS Cost Explorer:**
- 비용/사용량을 서비스, 계정, 태그, Region별로 시각화한다.
- RI/Savings Plan 커버리지 및 활용률을 확인한다.
- Rightsizing Recommendations으로 과대 프로비저닝된 EC2 인스턴스를 식별한다.
- 비용 예측(Forecasting) 기능으로 향후 비용을 예측한다.

**AWS Budgets:**
- 월별/분기별 비용 예산을 설정하고, 임계값(80%, 100%, 예측 초과) 도달 시 알림을 전송한다.
- Budget Actions으로 자동 대응: IAM 정책 적용(리소스 생성 차단), SCP 적용 등.

**AWS Cost and Usage Report (CUR):**
- 가장 상세한 비용 데이터를 S3로 내보낸다.
- Athena, QuickSight, 또는 3rd Party 도구(Kubecost, CloudHealth)로 분석한다.
- 시간 단위, 리소스 레벨의 세부 비용 데이터를 제공한다.

**AWS Trusted Advisor:**
- 비용 최적화, 보안, 성능, 서비스 한도, 내결함성 관련 권장 사항을 제공한다.
- 유휴 RDS, 미사용 EIP, 과소 활용 EC2 등을 탐지한다.
- Business/Enterprise Support Plan에서 전체 점검 항목 사용 가능.

**Kubecost (EKS 환경):**
- Kubernetes Pod/Namespace/Label 단위로 비용을 추적한다.
- 유휴 클러스터 리소스, 비효율적 리소스 요청을 식별한다.

---

### Q7-3. 다음 시나리오에서 비용을 최적화하기 위한 방안을 최소 5가지 이상 제시하시오: "월 AWS 비용이 $50,000인 중견기업이 비용을 30% 절감해야 한다."

**모범답안:**

**1. Rightsizing (즉각적 효과):**
- AWS Cost Explorer Rightsizing Recommendations 또는 Compute Optimizer를 활용하여 과대 프로비저닝된 EC2, RDS 인스턴스를 식별한다.
- CloudWatch CPU, 메모리(Custom Metric) 사용률이 평균 40% 미만인 인스턴스를 다운사이징한다.
- 예상 절감: 10~30%

**2. Savings Plan / Reserved Instance 도입:**
- 상시 운영 인스턴스의 기본 사용량에 대해 Compute Savings Plan(3년 부분 선결제)을 적용한다.
- Cost Explorer의 Savings Plan Recommendations를 활용하여 최적의 약정 금액을 결정한다.
- 예상 절감: 30~40% (적용 리소스 기준)

**3. Spot Instance 활용 확대:**
- 개발/테스트 환경, 배치 처리, CI/CD 빌드 워크로드를 Spot으로 전환한다.
- EKS 환경에서 Karpenter를 도입하여 Spot 인스턴스를 자동으로 활용한다.
- 예상 절감: 60~90% (적용 워크로드 기준)

**4. 스토리지 최적화:**
- S3 Intelligent-Tiering 적용 또는 Lifecycle Policy로 비활성 데이터를 저비용 스토리지 클래스로 전환한다.
- gp2 → gp3 EBS 볼륨 전환 (약 20% 절감, IOPS/Throughput 동일 또는 향상).
- 미사용/분리된 EBS 볼륨, 오래된 EBS 스냅샷 삭제.
- 예상 절감: 10~30% (스토리지 비용 기준)

**5. 비업무 시간 리소스 중지:**
- 개발/테스트 환경의 EC2, RDS 인스턴스를 비업무 시간에 자동 중지한다.
- AWS Instance Scheduler 또는 EventBridge + Lambda로 자동화한다.
- 주중 10시간 운영 기준 약 70% 절감 가능.

**6. NAT Gateway 비용 절감:**
- VPC Gateway Endpoint(S3, DynamoDB)를 설정하여 NAT GW 데이터 처리 비용을 절감한다.
- ECR, CloudWatch Logs 등 빈번히 사용하는 서비스에 Interface Endpoint를 설정한다.
- VPC Flow Logs로 NAT GW 트래픽 패턴을 분석하여 불필요한 외부 트래픽을 식별한다.

**7. 데이터 전송 비용 최적화:**
- 동일 AZ 내 통신을 최대화하도록 서비스를 배치한다.
- CloudFront를 활용하여 Origin에서의 Data Transfer Out을 줄인다 (CloudFront의 Data Transfer 비용이 EC2/S3 직접 전송보다 저렴).

**8. 서버리스 전환 검토:**
- 간헐적 트래픽의 API를 Lambda + API Gateway로 전환하면 유휴 시간 비용을 제거할 수 있다.
- RDS를 Aurora Serverless v2로 전환하여 비활성 시간에 자동 스케일다운한다.

---

## 8. AWS 모니터링

### Q8-1. CloudWatch의 주요 기능(Metrics, Logs, Alarms, Dashboards, Insights, Synthetics, RUM)을 설명하고, 효과적인 모니터링 전략을 수립하시오.

**모범답안:**

**CloudWatch Metrics:**
- AWS 서비스의 성능 지표를 자동 수집한다 (EC2: CPU, 네트워크; RDS: Connections, IOPS 등).
- 기본 모니터링: 5분 간격 (무료). 상세 모니터링: 1분 간격 (추가 비용).
- Custom Metrics: CloudWatch Agent 또는 PutMetricData API로 메모리, 디스크, 애플리케이션 지표를 전송한다.
- Metric Math: 여러 메트릭을 조합한 계산식 생성 (예: 에러율 = 5xx/(2xx+5xx)*100).
- Anomaly Detection: ML 기반 비정상 패턴 자동 감지.

**CloudWatch Logs:**
- EC2(CloudWatch Agent), Lambda(자동), ECS, VPC Flow Logs 등의 로그를 수집한다.
- Log Groups → Log Streams 계층 구조. 보존 기간을 설정하여 비용을 관리한다 (기본: 무기한).
- Metric Filters: 로그에서 패턴을 매칭하여 Custom Metric을 생성한다 (예: ERROR 로그 발생 횟수).
- Subscription Filters: 로그를 실시간으로 Lambda, Kinesis, OpenSearch로 스트리밍한다.
- **비용 주의:** 로그 수집(Ingestion) 비용이 상당할 수 있으므로, 필요한 로그만 수집하고 보존 기간을 적절히 설정한다.

**CloudWatch Logs Insights:**
- 로그 데이터에 대한 대화형 쿼리 엔진. 전용 쿼리 문법을 사용한다.
- 예: `fields @timestamp, @message | filter @message like /ERROR/ | stats count(*) by bin(1h)`

**CloudWatch Alarms:**
- Metric 또는 Metric Math 기반 알람. Static Threshold 또는 Anomaly Detection Band 사용.
- 상태: OK → ALARM → INSUFFICIENT_DATA.
- 액션: SNS 알림, Auto Scaling, EC2 액션(정지/종료/재부팅), Systems Manager OpsItem/Incident.
- Composite Alarms: 여러 알람을 AND/OR로 조합하여 알람 노이즈를 줄인다.

**CloudWatch Dashboards:**
- 커스텀 대시보드로 주요 메트릭, 로그, 알람을 시각화한다.
- Cross-Account, Cross-Region 대시보드 지원.

**CloudWatch Synthetics (Canary):**
- 합성 모니터링. 스크립트(Node.js/Python)로 API 엔드포인트, 웹 페이지를 주기적으로 테스트한다.
- 실 사용자 트래픽 없이도 서비스 가용성과 지연시간을 모니터링한다.

**CloudWatch RUM (Real User Monitoring):**
- 실 사용자의 브라우저에서 성능 데이터(페이지 로드 시간, JS 에러 등)를 수집한다.

**효과적인 모니터링 전략:**
- **Golden Signals 기반:** Latency, Traffic, Errors, Saturation 4가지 핵심 지표를 모든 서비스에 대해 모니터링한다.
- **계층별 모니터링:** 인프라(EC2, RDS) → 플랫폼(EKS, Lambda) → 애플리케이션(Custom Metrics) → 비즈니스(매출, 주문 수).
- **알람 피로도 관리:** Composite Alarms, 적절한 임계값, SNS → Lambda → PagerDuty/Slack 경로로 알람 라우팅.

---

### Q8-2. AWS X-Ray를 활용한 분산 추적(Distributed Tracing) 아키텍처를 설명하고, 마이크로서비스 환경에서의 성능 문제 진단 방법을 서술하시오.

**모범답안:**

**X-Ray의 핵심 개념:**
- **Trace:** 하나의 요청이 여러 서비스를 거치는 전체 경로. 고유 Trace ID로 식별.
- **Segment:** 각 서비스가 생성하는 작업 단위. 서비스 이름, 호스트, 시작/종료 시간, HTTP 상태 코드를 포함한다.
- **Subsegment:** Segment 내의 세부 작업. 외부 API 호출, DB 쿼리, AWS SDK 호출 등을 추적한다.
- **Service Map:** 서비스 간 의존성과 통신 패턴을 시각적으로 표시한다. 지연시간, 에러율을 색상으로 표시하여 병목점을 즉시 식별할 수 있다.

**X-Ray 연동 방법:**
- **Lambda:** 활성화 옵션만 켜면 자동으로 추적 (Managed Integration).
- **EC2/ECS/EKS:** X-Ray Daemon을 사이드카 또는 DaemonSet으로 실행. 애플리케이션에 X-Ray SDK를 통합한다.
- **API Gateway:** 스테이지 설정에서 X-Ray Tracing 활성화.
- **ALB:** Access Log에서 X-Ray Trace ID 확인 가능.

**OpenTelemetry 통합:**
- AWS Distro for OpenTelemetry(ADOT)를 사용하면 벤더 종속 없이 X-Ray로 트레이스를 전송할 수 있다.
- ADOT Collector를 사이드카 또는 DaemonSet으로 배포하고, OTLP Exporter로 X-Ray에 전송한다.
- 동시에 Jaeger, Zipkin 등 다른 백엔드로도 전송 가능.

**성능 문제 진단 방법:**

1. **Service Map 분석:** 에러율이 높거나 지연시간이 긴 서비스를 색상으로 즉시 식별한다.

2. **Trace 분석:** 느린 요청의 Trace를 열어 각 Segment/Subsegment의 소요 시간을 확인한다. 예: API Gateway(10ms) → Lambda(200ms) → DynamoDB(50ms) → 외부API(2000ms) → 병목: 외부 API 호출.

3. **X-Ray Analytics:** 필터 표현식으로 특정 조건의 Trace를 검색한다.
   - `service("order-service") AND responsetime > 5`
   - `fault = true AND http.url CONTAINS "/api/payment"`
   - Response time 분포 히스토그램으로 P50, P90, P99 지연시간을 분석한다.

4. **X-Ray Insights:** ML 기반으로 비정상적인 지연시간 증가, 에러율 급등을 자동 탐지하고 알림을 전송한다.

**샘플링 전략:**
- 기본 샘플링: 초당 1개 요청 + 이후 5% 샘플링.
- 커스텀 샘플링 규칙으로 특정 경로/서비스의 샘플링 비율을 조정할 수 있다.
- 모든 요청을 추적하면 비용이 급증하므로, 프로덕션에서는 적절한 샘플링 비율을 설정해야 한다.

---

### Q8-3. EKS 환경에서 Observability Stack을 설계하시오. Metrics, Logs, Traces 각각에 대한 수집/저장/시각화 방안을 포함하시오.

**모범답안:**

**AWS 관리형 스택:**

**Metrics:**
- Amazon Managed Service for Prometheus (AMP): Prometheus 호환 메트릭 수집/저장.
- ADOT Collector 또는 Prometheus Server를 DaemonSet으로 배포하여 Pod/Node 메트릭을 수집하고 AMP로 Remote Write한다.
- Amazon Managed Grafana (AMG): AMP를 데이터 소스로 연결하여 대시보드를 구성한다.
- Container Insights: CloudWatch 기반 EKS 모니터링. 클러스터, 노드, Pod, 컨테이너 수준 메트릭과 로그를 제공한다.

**Logs:**
- Fluent Bit DaemonSet으로 컨테이너 로그를 수집한다 (/var/log/containers/).
- 목적지: CloudWatch Logs, OpenSearch, S3.
- Fargate의 경우: Fluent Bit sidecar로 로그를 전송한다.
- 구조화 로그(JSON) 형식을 표준화하여 검색/분석을 용이하게 한다.

**Traces:**
- ADOT Collector를 사이드카 또는 DaemonSet으로 배포.
- X-Ray 또는 Jaeger/Tempo로 트레이스를 전송한다.
- 애플리케이션에 OpenTelemetry SDK를 통합하여 자동/수동 계측을 수행한다.

**오픈소스 스택 (대안):**

**Metrics:**
- Prometheus (자체 관리) + Thanos/Cortex (장기 보존, 고가용성).
- Grafana (자체 관리) + Prometheus 데이터 소스.

**Logs:**
- Fluent Bit → OpenSearch 또는 Loki.
- Loki는 Prometheus와 유사한 레이블 기반 쿼리로, 인덱싱 비용이 OpenSearch 대비 낮다.

**Traces:**
- Jaeger 또는 Grafana Tempo.
- Tempo는 오브젝트 스토리지(S3) 백엔드를 사용하여 비용 효율적이다.

**통합 시각화:**
- Grafana에서 Prometheus(Metrics) + Loki(Logs) + Tempo(Traces)를 연결하면 Exemplar를 통해 Metric → Trace → Log 간 자동 연결이 가능하다 (Correlation).

**권장 아키텍처 (비용/운영 밸런스):**
- Metrics: AMP + AMG (관리 부담 최소화)
- Logs: Fluent Bit → CloudWatch Logs (기본) + S3 (장기 보관)
- Traces: ADOT → X-Ray (AWS 통합) 또는 ADOT → Tempo (오픈소스 선호 시)

---

## 9. 오픈소스 연동

### Q9-1. Terraform과 CloudFormation을 비교하고, 각각의 장단점과 적합한 사용 시나리오를 설명하시오. Terraform 운영 시 State 관리 모범 사례를 포함하시오.

**모범답안:**

**CloudFormation:**
- AWS 네이티브 IaC 서비스. YAML/JSON 템플릿.
- 장점: AWS 서비스와의 완벽한 통합 (새 서비스/기능 즉시 지원), Drift Detection, Change Sets(변경 사항 미리보기), StackSets(멀티 계정/리전 배포), CDK(Python/TypeScript 등으로 작성 가능).
- 단점: AWS 전용(멀티 클라우드 불가), 복잡한 템플릿의 디버깅이 어려움, 상태를 AWS가 관리(직접 접근/수정 어려움), 조건부 로직이 제한적.
- **적합한 시나리오:** 순수 AWS 환경, CDK 활용, AWS Service Catalog 통합, AWS 관리형 상태 관리를 선호하는 경우.

**Terraform:**
- HashiCorp의 오픈소스 IaC. HCL(HashiCorp Configuration Language) 사용.
- 장점: 멀티 클라우드/하이브리드 지원(AWS, GCP, Azure, K8s, Datadog 등), 풍부한 Provider 에코시스템, 강력한 모듈 시스템, Plan 명령으로 변경 사항 미리보기, 활발한 커뮤니티.
- 단점: State 파일 관리 책임이 사용자에게 있음, 새 AWS 서비스 지원이 CloudFormation보다 느릴 수 있음, HCL 학습 필요, Provider 업데이트 시 Breaking Change 가능.
- **적합한 시나리오:** 멀티 클라우드 환경, 기존 Terraform 역량 보유, 비AWS 리소스(GitHub, PagerDuty 등) 함께 관리.

**Terraform State 관리 모범 사례:**

1. **Remote Backend 사용:**
   - S3 + DynamoDB로 원격 State 저장 및 State Locking을 구현한다.
   - S3 버킷: 버전관리 활성화, 서버 측 암호화(SSE-S3/SSE-KMS), 접근 로깅 활성화.
   - DynamoDB: State Lock을 위한 테이블 (동시 수정 방지).
   ```hcl
   backend "s3" {
     bucket         = "my-terraform-state"
     key            = "env/prod/terraform.tfstate"
     region         = "ap-northeast-2"
     dynamodb_table = "terraform-locks"
     encrypt        = true
   }
   ```

2. **State 분리 (State Isolation):**
   - 환경별(dev/staging/prod), 계층별(네트워크/컴퓨팅/데이터)로 State를 분리한다.
   - Blast Radius를 줄이고, 각 팀이 독립적으로 인프라를 관리할 수 있다.
   - `terraform_remote_state` data source로 다른 State의 Output을 참조한다.

3. **State 잠금:** DynamoDB Lock으로 동시 `terraform apply` 실행을 방지한다. CI/CD 파이프라인에서 직렬화(Serialization)를 보장한다.

4. **민감 데이터 보호:** State 파일에는 리소스 속성이 평문으로 저장된다(DB 비밀번호 등). S3 암호화를 반드시 활성화하고, State 파일 접근 권한을 최소화한다.

5. **Terraform Cloud/Enterprise 또는 Atlantis:** 팀 환경에서 PR 기반 `plan/apply` 워크플로우, State 관리, 비용 예측 등을 통합 관리한다.

---

### Q9-2. AWS 환경에서 GitOps를 구현하기 위한 ArgoCD/Flux 아키텍처를 설계하고, 기존 CI/CD 파이프라인과의 통합 방안을 설명하시오.

**모범답안:**

**GitOps 원칙:**
- Git을 Single Source of Truth로 사용하여 선언적(Declarative) 인프라/애플리케이션 배포를 관리한다.
- 시스템의 실제 상태를 Git에 선언된 상태와 자동으로 동기화(Reconciliation)한다.
- 모든 변경은 Git Commit/PR을 통해 이루어지며, 감사 추적이 자동으로 제공된다.

**ArgoCD 기반 아키텍처:**

```
[App Repo] → CI Pipeline → Build/Test → Container Image → ECR
                                                             ↓
                                                   Image Tag 업데이트
                                                             ↓
[Config Repo] ← PR (자동/수동) → Review → Merge
      ↓
  ArgoCD (EKS에 설치)
      ↓
  EKS Cluster (자동 동기화)
```

**CI 파이프라인 (AWS CodePipeline/GitHub Actions/Jenkins):**
1. 소스 코드 변경 → CI 파이프라인 트리거
2. 빌드, 테스트, 보안 스캔(Trivy, Snyk)
3. Docker 이미지 빌드 → ECR에 Push (태그: Git SHA 또는 Semantic Version)
4. Config Repo의 Kubernetes manifest/Helm values에서 이미지 태그를 업데이트 (자동 PR 생성)

**CD 파이프라인 (ArgoCD):**
1. ArgoCD가 Config Repo를 감시 (Polling 또는 Webhook)
2. Git 상태와 클러스터 상태의 차이(Diff)를 감지
3. 자동(Auto Sync) 또는 수동 승인 후 동기화
4. Sync Strategy: Prune(불필요 리소스 삭제), Self-Heal(수동 변경 자동 롤백)

**ArgoCD on EKS 구성:**
- ArgoCD를 EKS 클러스터의 `argocd` Namespace에 설치한다.
- IRSA를 통해 ECR 이미지 Pull 권한을 부여한다.
- ArgoCD ApplicationSet으로 여러 환경(dev/staging/prod) 배포를 자동화한다.
- SSO 통합: ArgoCD의 OIDC 설정을 통해 AWS IAM Identity Center 또는 Okta와 연동한다.
- RBAC: ArgoCD Project로 팀별 접근 범위를 제한한다.

**배포 전략 (ArgoCD + Argo Rollouts):**
- **Canary:** 새 버전을 소량 트래픽으로 시작하여 점진적으로 확대. AnalysisRun으로 Prometheus 메트릭 기반 자동 롤백.
- **Blue/Green:** 새 버전을 Preview Service로 배포 → 테스트 → 트래픽 전환.

**Config Repo 구조 예시:**
```
config-repo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── prod/
│       └── kustomization.yaml
└── argocd/
    └── applications.yaml
```

---

### Q9-3. AWS에서 오픈소스 모니터링 스택(Prometheus, Grafana, OpenTelemetry)을 운영할 때의 관리형 서비스(AMP, AMG, ADOT)와 자체 운영의 Trade-off를 분석하시오.

**모범답안:**

**Amazon Managed Service for Prometheus (AMP):**

*장점:*
- 서버 관리, 스토리지 확장, 고가용성을 AWS가 관리한다.
- PromQL 100% 호환, 기존 Prometheus 워크플로우 유지 가능.
- 자동 스케일링으로 대규모 메트릭 처리.
- Multi-AZ 내구성, 150일 기본 보존 기간.
- IRSA 기반 인증으로 보안 강화.

*단점:*
- 비용이 자체 운영 대비 높을 수 있다 (수집 비용 + 저장 비용 + 쿼리 비용).
- Alert Manager가 별도이므로 SNS 연동이 필요하다.
- Remote Write만 지원 (Pull 모델 아님), 기존 Recording Rules/Alert Rules는 AMP Workspace에 업로드해야 한다.

**Amazon Managed Grafana (AMG):**

*장점:*
- Grafana 서버 관리 불필요. 자동 업데이트, 고가용성.
- IAM Identity Center(SSO) 통합으로 사용자 인증 관리.
- AMP, CloudWatch, X-Ray, OpenSearch 등 다양한 AWS 데이터 소스와 원클릭 연동.
- Grafana Enterprise 플러그인 포함.

*단점:*
- 커스텀 플러그인 설치 제한.
- 비용: 사용자당 $9/월 (Editor), $5/월 (Viewer).
- 세밀한 Grafana 설정(grafana.ini) 조정이 제한적.

**AWS Distro for OpenTelemetry (ADOT):**

*장점:*
- 벤더 중립 오픈소스 계측 프레임워크.
- X-Ray, AMP, CloudWatch 등 다양한 AWS 백엔드로 동시 전송 가능.
- Lambda Layer로 서버리스 환경에서도 쉽게 사용 가능.
- Auto-instrumentation 지원 (Java, Python, Node.js 등).

*단점:*
- Collector 관리(배포, 확장, 모니터링)가 필요하다.
- 일부 Exporter/Processor의 안정성이 커뮤니티 버전과 다를 수 있다.

**자체 운영 (Prometheus + Grafana + Thanos):**

*장점:*
- 완전한 제어: 버전, 설정, 플러그인, 보존 기간 등.
- 비용 최적화: EC2/EBS 비용만 발생, 대규모에서는 관리형 서비스보다 저렴할 수 있다.
- Thanos/Cortex를 사용한 장기 보존 및 글로벌 뷰.

*단점:*
- 운영 부담: 고가용성 구성, 업그레이드, 스토리지 관리, 모니터링의 모니터링(meta-monitoring).
- 인력 비용: 전담 SRE/DevOps 인력 필요.
- 스케일링: 대규모 환경에서 Prometheus의 메모리 사용량, WAL 관리가 복잡.

**의사결정 기준:**
- **팀 규모 5인 이하, 빠른 구축:** AMP + AMG 추천
- **대규모(수만 Time Series 이상), 비용 민감:** 자체 Prometheus + Thanos(S3 백엔드)
- **멀티 클라우드/하이브리드:** 자체 운영 + ADOT
- **컴플라이언스 요건:** 데이터 보존 정책, 접근 제어 등을 고려하여 관리형/자체 운영 선택

---

### Q9-4. Helm을 사용한 EKS 애플리케이션 패키징 및 배포 전략을 설명하고, Helm Chart 작성 시의 모범 사례를 서술하시오.

**모범답안:**

**Helm 기본 구조:**
```
my-chart/
├── Chart.yaml          # 차트 메타데이터 (name, version, appVersion)
├── values.yaml         # 기본 설정값
├── values-dev.yaml     # 환경별 오버라이드
├── values-prod.yaml
├── templates/
│   ├── _helpers.tpl    # 공통 템플릿 헬퍼 함수
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   └── configmap.yaml
└── charts/             # 의존 차트
```

**Helm Chart 작성 모범 사례:**

1. **values.yaml 설계:**
   - 환경별로 달라지는 값만 values.yaml에 노출한다 (이미지 태그, 리소스 요청/제한, 레플리카 수).
   - 합리적인 기본값을 제공하여 최소 설정으로 배포 가능하게 한다.
   - 중첩 구조로 관련 설정을 그룹핑한다.
   ```yaml
   replicaCount: 2
   image:
     repository: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app
     tag: "latest"
     pullPolicy: IfNotPresent
   resources:
     requests:
       cpu: 100m
       memory: 128Mi
     limits:
       cpu: 500m
       memory: 512Mi
   serviceAccount:
     create: true
     annotations:
       eks.amazonaws.com/role-arn: ""
   ```

2. **템플릿 모범 사례:**
   - `_helpers.tpl`에 공통 레이블, 이름 생성 로직을 정의한다.
   - `{{ include "my-chart.labels" . }}` 로 표준 레이블(app.kubernetes.io/name, version, managed-by)을 일관되게 적용한다.
   - `{{- if .Values.ingress.enabled }}` 조건부 리소스 생성으로 유연성을 확보한다.
   - `{{ required "image.tag is required" .Values.image.tag }}` 로 필수 값 검증을 수행한다.

3. **버전 관리:**
   - Chart.yaml의 `version`(차트 버전)과 `appVersion`(애플리케이션 버전)을 분리 관리한다.
   - Semantic Versioning을 엄격히 적용한다.
   - Breaking Change 시 Major 버전을 올린다.

4. **차트 저장소:**
   - ECR (OCI Registry): EKS와 동일한 AWS 에코시스템, IRSA 인증.
   - ChartMuseum (자체 운영): S3 백엔드.
   - `helm push/pull` OCI 프로토콜 사용 권장.

5. **테스트:**
   - `helm lint`: 문법 검증.
   - `helm template`: 렌더링 결과 확인.
   - `helm test`: 배포 후 연결 테스트 (Pod로 실행).
   - Conftest/OPA: 보안 정책 검증 (privileged 컨테이너 차단 등).
   - CI 파이프라인에서 자동화.

6. **Secrets 관리:**
   - 민감 정보를 values.yaml에 직접 포함하지 않는다.
   - AWS Secrets Manager + External Secrets Operator로 Kubernetes Secret을 자동 동기화한다.
   - Sealed Secrets 또는 SOPS로 암호화된 Secret을 Git에 저장하는 방법도 있다.

---

### Q9-5. AWS에서 Apache Kafka(MSK)와 Amazon Kinesis를 비교하고, 오픈소스 스트리밍 아키텍처를 설계하시오.

**모범답안:**

**Amazon MSK (Managed Streaming for Apache Kafka):**
- Apache Kafka 완전 관리형 서비스. 오픈소스 Kafka API 100% 호환.
- Broker, ZooKeeper(또는 KRaft 모드) 관리를 AWS가 담당.
- MSK Serverless: 용량 자동 관리, Kafka 클러스터 관리 불필요.
- MSK Connect: Kafka Connect 관리형 실행 환경.

*장점:* 기존 Kafka 워크로드 마이그레이션 용이, 풍부한 Kafka 에코시스템(Kafka Streams, KSQL, Connect, Schema Registry) 활용, 세밀한 파티션/토픽 제어, 무제한에 가까운 데이터 보존.

*단점:* Kafka 운영 지식 필요, Broker 수/인스턴스 타입 선택 필요(Serverless 제외), Serverless 모드의 비용이 높을 수 있음.

**Amazon Kinesis Data Streams:**
- AWS 네이티브 실시간 스트리밍 서비스.
- Shard 단위 확장 (Shard당 1MB/s 쓰기, 2MB/s 읽기).
- On-Demand 모드: 자동 Shard 관리.

*장점:* AWS 서비스와의 네이티브 통합 (Lambda, Firehose, Analytics), 운영 부담 매우 낮음, IAM 기반 인증/인가, Encryption at Rest/Transit 기본 제공.

*단점:* Shard당 소비자 수 제한(Fan-Out 사용 시 완화), 최대 365일 보존, Kafka 에코시스템 미지원, Vendor Lock-in.

**비교 요약:**

| 구분 | MSK | Kinesis |
|------|-----|---------|
| 프로토콜 | Kafka (오픈소스) | AWS 독자 API |
| 데이터 보존 | 무제한 | 최대 365일 |
| 소비자 모델 | Consumer Group (다수 가능) | Shard당 제한 (Enhanced Fan-Out) |
| 확장 | 브로커/파티션 추가 | Shard 추가 (On-Demand 모드) |
| 에코시스템 | Kafka Streams, Connect, KSQL | Lambda, Firehose, Analytics |
| 적합한 경우 | Kafka 기존 워크로드, 복잡한 스트리밍 | AWS 네이티브, 빠른 구축 |

**오픈소스 스트리밍 아키텍처 (MSK 기반):**

```
데이터 소스 → MSK Connect (Debezium CDC) → MSK Cluster
                                                 ↓
                              ┌──────────────────┼──────────────────┐
                              ↓                  ↓                  ↓
                    Kafka Streams App      Flink on EMR       MSK Connect
                    (실시간 처리/변환)     (복잡한 이벤트 처리)  (S3 Sink)
                              ↓                  ↓                  ↓
                        OpenSearch           Iceberg/Hudi        S3 Data Lake
                        (검색/대시보드)      (분석용 테이블)     (원본 데이터 보관)
```

**Schema Registry:** MSK에 Confluent Schema Registry 또는 AWS Glue Schema Registry를 연동하여 스키마 진화(Schema Evolution)를 관리한다. Avro/Protobuf/JSON Schema 형식을 표준화한다.

**핵심 설계 고려사항:**
- **파티션 설계:** Key 기반 파티션으로 순서 보장이 필요한 데이터를 동일 파티션에 배치한다. 파티션 수는 처리량 요구사항과 소비자 수를 고려하여 결정한다.
- **Exactly-Once Semantics:** Kafka 트랜잭션(`enable.idempotence=true`, `transactional.id`)으로 중복 메시지를 방지한다.
- **모니터링:** MSK의 CloudWatch 메트릭(UnderReplicatedPartitions, OfflinePartitionsCount) + Prometheus Exporter(Kafka Exporter, JMX Exporter)로 상세 모니터링을 구현한다.
