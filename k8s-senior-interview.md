# 시니어 Kubernetes 엔지니어 면접 질문 · 답변 가이드

> **대상 환경**: 30 클러스터 / 1,000 노드 / 200 서비스 (멀티 테넌트 운영)
> **대상 직급**: Senior SRE / Platform Engineer
> **용도**: 면접관용. 질문 → 기대 답변 → 꼬리 질문 → 평가 포인트 순서로 구성

---

## 0. 사용 가이드

### 이 규모에서 진짜로 검증해야 하는 것

1,000 노드 / 30 클러스터는 "Kubernetes를 아는가"가 아니라 **"Kubernetes가 깨지는 지점을 아는가"**를 물어야 하는 규모입니다. 50 노드짜리 단일 클러스터에서는 절대 만나지 않는 문제들이 지배적입니다.

| 평가 축 | 무엇을 보는가 | 비중 |
|---|---|---|
| 스케일 한계 인식 | etcd/apiserver/CNI/DNS가 어디서 터지는지 | ★★★★★ |
| 플릿 운영 | 30개를 1개처럼 다루는 자동화 설계 | ★★★★★ |
| 장애 진단력 | 증상 → 계층별 가설 → 검증 순서 | ★★★★★ |
| 멀티테넌시/보안 | 고객사 간 격리, 최소 권한, 공급망 | ★★★★☆ |
| 변경 관리 | 업그레이드/롤아웃의 폭발 반경 통제 | ★★★★☆ |
| 비용/효율 | 리소스 낭비와 최적화의 균형 | ★★★☆☆ |
| 커뮤니케이션 | 고객·동료에게 리스크를 설명하는 능력 | ★★★★☆ |

### 권장 라운드 구성 (총 4~5시간)

| 라운드 | 시간 | 내용 | 섹션 |
|---|---|---|---|
| 1. 기술 심화 | 60분 | 컨트롤 플레인 · 네트워킹 · 스케줄링 | 1~6 |
| 2. 라이브 디버깅 | 60분 | 실제 증상 제시, 진단 과정 관찰 | 11 |
| 3. 시스템 설계 | 60분 | 플릿 아키텍처 화이트보드 | 12 |
| 4. 보안/운영 | 45분 | 멀티테넌시 · 업그레이드 · 사고 대응 | 9~10, 13 |
| 5. 컬처핏 | 45분 | 협업 · 의사결정 · MSP 상황 판단 | 13 |

### 채점 기준 (각 질문 공통)

- **1점 (Junior)**: 개념은 알지만 규모가 커질 때의 변화를 모름
- **2점 (Mid)**: 정답은 말하지만 트레이드오프를 못 댐
- **3점 (Senior)**: 한계·실패 모드·대안을 함께 설명
- **4점 (Staff)**: 실제 겪은 사례와 수치, 조직적 해결책까지 제시

---

## 1. 대규모 클러스터 아키텍처

### Q1-1. 1,000 노드를 한 클러스터에 넣을 것인지, 나눌 것인지 결정한다면?

**기대 답변**

- Kubernetes 업스트림 공식 한계는 **노드 5,000 / 전체 Pod 150,000 / 컨테이너 300,000 / 노드당 Pod 110**. 1,000 노드는 기술적으로는 단일 클러스터 범위 안이지만, **한계 근처로 갈수록 운영 비용이 비선형으로 증가**한다.
- 실제 결정 기준은 노드 수가 아니라:
  - **폭발 반경(blast radius)**: 컨트롤 플레인 장애 시 몇 개 워크로드가 죽는가
  - **격리 요구**: 고객사/규제/네트워크 분리가 강제되는가
  - **변경 주기**: 업그레이드·정책 변경을 서로 다른 속도로 가져가야 하는가
  - **리전/AZ 경계**: 클러스터는 리전을 넘지 않는 것이 기본
- MSP 환경이면 **고객사 = 클러스터 경계**가 자연스러움. 대신 클러스터 수가 늘면 "관리 비용"이 새 병목이 되므로 **플릿 자동화가 선행 조건**.
- 30 × 33노드 평균이라면 오히려 **클러스터당 컨트롤 플레인 오버헤드(30세트의 etcd, 30세트의 모니터링 스택)** 가 주요 비용. 소규모 클러스터 통합 여지를 검토해야 함.

**꼬리 질문**
- 클러스터 수를 절반으로 줄이자는 제안이 오면 무엇을 근거로 반대/찬성하겠는가?
- 클러스터를 나눌 때 공유해야 하는 것(레지스트리, 로그, 메트릭, ID)은 어떻게 설계하는가?

**평가 포인트**
- ⭕ 노드 수보다 폭발 반경·변경 주기를 먼저 말하면 시니어
- ❌ "5,000까지 되니까 하나로 하면 됩니다"만 말하면 감점

---

### Q1-2. 노드가 100개에서 1,000개로 늘어날 때 가장 먼저 터지는 컴포넌트는?

**기대 답변** (우선순위 순으로 나와야 함)

1. **etcd** — 쓰기 지연(fsync), DB 크기, watch 팬아웃. 가장 먼저, 가장 치명적으로 터짐.
2. **kube-apiserver 메모리** — 대량 LIST 요청이 들어오면 전체 오브젝트를 메모리에 직렬화. OOM으로 연쇄 장애.
3. **CoreDNS** — 쿼리량 선형 증가 + conntrack 경합.
4. **CNI 데이터플레인** — iptables 규칙 수, 라우팅 테이블, IP 대역 고갈.
5. **모니터링 스택** — Prometheus 카디널리티 폭발.
6. **kube-controller-manager / scheduler** — 기본 QPS 제한에 걸려 반응이 느려짐.

**꼬리 질문**
- apiserver가 OOM으로 죽는 원인을 클라이언트 쪽에서 찾는다면 무엇을 보겠는가?
  - → *기대*: `apiserver_request_total`을 verb=LIST로 필터링, User-Agent별 집계. 잘못 만든 컨트롤러/스크립트가 `kubectl get pods -A`를 주기적으로 도는 경우가 흔함. informer 캐시 미사용, `resourceVersion=0` 미지정.

**평가 포인트**
- ⭕ "LIST 요청"을 지목하면 실전 경험 있음
- ⭕ APF(API Priority and Fairness)를 대응책으로 언급하면 가점

---

### Q1-3. API Priority and Fairness(APF)를 설명하고, 이 환경에서 어떻게 설정하겠는가?

**기대 답변**

- 과거의 `--max-requests-inflight`(기본 400) / `--max-mutating-requests-inflight`(기본 200)은 **전역 단일 카운터**라서, 폭주하는 클라이언트 하나가 전체 대역을 잡아먹고 kubelet의 노드 상태 갱신까지 굶겼다.
- APF는 요청을 **FlowSchema**로 분류하고 **PriorityLevelConfiguration**의 독립된 큐(concurrency share)에 넣어 격리한다. 한 테넌트/컨트롤러의 폭주가 다른 흐름을 굶기지 못하게 하는 **공정 큐잉(fair queuing)**.
- 실무 설정:
  - `system-leader-election`, `node-high`(kubelet) 는 절대 굶기면 안 됨 → 우선순위 보장
  - 배치성/저우선 클라이언트는 별도 낮은 PriorityLevel로 격리
  - 관측: `apiserver_flowcontrol_rejected_requests_total`, `apiserver_flowcontrol_current_inqueue_requests`, `apiserver_flowcontrol_request_wait_duration_seconds`
  - 429 응답이 나오면 클라이언트는 `Retry-After` 존중해야 함
- **주의**: APF는 exempt 레벨(system:masters 등)을 통과시키므로, cluster-admin으로 도는 스크립트는 보호막을 우회한다. 이것 자체가 RBAC 설계 문제로 연결됨.

**평가 포인트**
- ⭕ "exempt 우회" 언급 = 실제로 당해본 사람

---

### Q1-4. kube-apiserver / controller-manager / scheduler 중 대규모에서 튜닝이 필요한 파라미터는?

**기대 답변**

| 컴포넌트 | 파라미터 | 이유 |
|---|---|---|
| apiserver | `--etcd-servers-overrides=/events#...` | **Event를 별도 etcd로 분리**. Event는 쓰기량이 압도적이고 휘발성인데 핵심 데이터와 같은 etcd를 쓰면 동반 사망 |
| apiserver | `--event-ttl` (기본 1h) | 축소해 etcd 부담 감소 |
| apiserver | `--watch-cache-sizes` | 자주 조회되는 리소스 캐시 확대 |
| apiserver | `--goaway-chance` | HTTP/2 롱리브드 커넥션을 주기적으로 재분배해 LB 편중 방지 |
| controller-manager | `--kube-api-qps` / `--kube-api-burst` | 기본값(20/30)은 대규모에서 명백히 부족 |
| controller-manager | `--node-monitor-grace-period`, `--large-cluster-size-threshold`, `--unhealthy-zone-threshold` | 대량 NotReady 시 **eviction 폭주 방지 레이트 리밋** |
| scheduler | `percentageOfNodesToScore` | 전체 노드 채점은 비용이 큼. 기본은 클러스터 크기에 따라 적응적으로 축소(최소 5%). 스케줄링 처리량 ↔ 배치 품질 트레이드오프 |
| scheduler | `--kube-api-qps` | 동일 이유 |
| kubelet | `--node-status-update-frequency`, `--kube-api-qps` | 노드가 많을수록 apiserver 쓰기 압력 직결 |

**평가 포인트**
- ⭕ **Event etcd 분리**를 말하면 큰 가점. 대규모 운영 경험의 지표.

---

### Q1-5. 컨트롤 플레인 앞단 로드밸런서 설계에서 주의할 점은?

**기대 답변**

- apiserver 클라이언트는 **HTTP/2 롱리브드 커넥션(watch)** 을 유지하므로, L4 LB의 라운드로빈이 **초기 연결 시점의 분포**로 고정된다. 컨트롤 플레인 노드 1대를 재시작하면 모든 클라이언트가 남은 2대로 몰리고, 복구 후에도 재분배되지 않아 **영구적 편중**이 발생.
- 대응: `--goaway-chance`(예: 0.001)로 apiserver가 확률적으로 GOAWAY를 보내 재연결 유도.
- 헬스체크는 `/readyz`를 쓸 것. `/healthz`는 셧다운 중에도 200을 줄 수 있음. `/livez`는 프로세스 생존만 의미.
- **graceful shutdown**: `--shutdown-delay-duration`을 둬서 readyz가 먼저 실패 → LB가 뺀 뒤 → 요청 드레인 → 종료. 안 하면 업그레이드마다 순간 에러.
- 부트스트랩 순환 의존성 주의: 클러스터 내부 LB(예: 클러스터 안에서 돌아가는 HAProxy)에 컨트롤 플레인을 의존시키면 콜드 스타트 불가.

---

## 2. etcd

### Q2-1. etcd가 1,000 노드 규모에서 병목이 되는 이유와 튜닝 방법은?

**기대 답변**

**왜 병목인가**
- etcd는 Raft 기반이라 **모든 쓰기가 과반수 노드에 fsync 동기화**되어야 커밋. 디스크 fsync 지연이 그대로 클러스터 쓰기 지연.
- 모든 watch가 etcd에서 시작 → 노드/Pod 수에 비례해 이벤트 팬아웃 증가.
- 기본 DB 사이즈 쿼터는 **2GiB**, 권장 상한은 **8GiB**. 초과 시 클러스터가 **읽기 전용(NOSPACE alarm)** 으로 잠김 — 가장 악명 높은 장애.

**튜닝**
- **디스크**: 로컬 NVMe 필수. 네트워크 스토리지(EBS gp2, NFS) 금지에 가까움. WAL을 별도 디스크로 분리(`--wal-dir`).
- **ionice/우선순위**: etcd 프로세스에 IO 우선순위 부여.
- **압축**: `--auto-compaction-retention`(예: 1h)로 리비전 히스토리 정리. 압축은 공간을 "해제 가능"하게 만들 뿐, **defrag를 해야 실제 반환**.
- **defrag**: 멤버별로 **하나씩** 수행. defrag 중 해당 멤버는 블로킹되므로 동시에 하면 쿼럼 상실.
- **하트비트/선거 타임아웃**: `--heartbeat-interval`은 RTT의 1.5배 정도, `--election-timeout`은 하트비트의 10배. AZ 간 배치 시 RTT 증가분 반영 필요.
- **홀수 멤버 3 또는 5**. 7 이상은 쓰기 지연만 늘어남.
- **Event 분리**(Q1-4 참조).

**관측 지표**
- `etcd_disk_wal_fsync_duration_seconds` (p99 < 10ms 목표)
- `etcd_disk_backend_commit_duration_seconds` (p99 < 25ms)
- `etcd_server_leader_changes_seen_total` (증가하면 불안정)
- `etcd_mvcc_db_total_size_in_bytes` vs `etcd_mvcc_db_total_size_in_use_in_bytes` (차이가 크면 defrag 필요)
- `etcd_network_peer_round_trip_time_seconds`

**평가 포인트**
- ⭕ 압축과 defrag의 차이를 정확히 구분하면 시니어
- ⭕ 2GiB 쿼터 초과 시 읽기 전용 전환을 알면 실전 경험

---

### Q2-2. etcd가 NOSPACE alarm으로 잠겼다. 복구 절차는?

**기대 답변** (순서가 중요)

1. 현상 확인: `etcdctl endpoint status --write-out=table`, `etcdctl alarm list`
2. **먼저 백업**: `etcdctl snapshot save` (복구 실패 시 유일한 안전망)
3. 현재 리비전 확인 후 **압축**: `etcdctl compact <rev>`
4. 멤버별 **defrag**: `etcdctl defrag --endpoints=<멤버1>` → 완료 확인 → 멤버2 → 멤버3 (동시 금지)
5. **알람 해제**: `etcdctl alarm disarm` (이걸 안 하면 공간이 생겨도 계속 읽기 전용)
6. 근본 원인 제거: 자동 압축 설정, Event 분리, 대용량 오브젝트(거대 ConfigMap/Secret, CRD 남용) 색출
7. 재발 방지: DB 사이즈 알람을 쿼터의 60~70% 지점에 설정

**꼬리 질문**
- 백업에서 복구할 때 주의점은?
  - → *기대*: 스냅샷 복구는 **새 클러스터를 만드는 것**. 모든 멤버를 동시에 정지 → 각각 restore → 새 cluster-id로 기동. 하나만 복구해 기존 멤버와 섞으면 데이터 불일치. 복구 시점 이후 생성된 리소스는 사라지고, **실제 워크로드(Pod)는 살아있는데 etcd는 모른다**는 불일치 상태가 되므로 노드 조인/재조정 계획 필요.

---

### Q2-3. etcd 백업 전략을 설계한다면? 30개 클러스터라면?

**기대 답변**

- 주기적 스냅샷(예: 30분~1시간) + 오프사이트(다른 리전/계정) 저장 + 암호화.
- **백업은 복구 테스트를 해야 백업이다.** 분기 1회 이상 실제 복구 훈련(DR drill), RTO/RPO 측정·기록.
- 30 클러스터면 스크립트가 아니라 **선언적 백업 오퍼레이터 + 중앙 상태 대시보드**. "어제 백업이 실패한 클러스터"를 한 화면에서 알 수 있어야 함.
- etcd 스냅샷만으로는 부족: **PV 데이터는 별도**(Velero + CSI VolumeSnapshot).
- GitOps를 쓰면 etcd 복구보다 **클러스터 재생성 + Git 재동기화**가 더 빠르고 안전한 경우가 많음 — 어느 쪽이 주 전략인지 명확히 정해야 함.

**평가 포인트**
- ⭕ "복구 테스트 안 한 백업은 백업이 아니다" + 실제 훈련 경험 = 강한 신호
- ⭕ etcd 복구 vs 클러스터 재생성의 트레이드오프를 논하면 가점

---

## 3. 멀티 클러스터 · 플릿 운영

### Q3-1. 30개 클러스터의 설정을 어떻게 일관되게 관리하겠는가?

**기대 답변**

- **GitOps가 사실상 유일한 답**. 명령형 스크립트/수동 kubectl은 30개에서 반드시 드리프트 발생.
- 계층 구조 설계:
  ```
  base/                 # 모든 클러스터 공통 (CNI 설정, 모니터링, 정책)
  overlays/
    tier/prod, tier/dev # 등급별 차이
    region/kr, region/us
    cluster/cluster-01  # 최소한의 클러스터 고유값만
  ```
  Kustomize 또는 Helm + values 계층. **클러스터 고유 파일은 최대한 얇게.**
- 배포 도구: **Argo CD ApplicationSet**(cluster generator로 30개 자동 팬아웃) 또는 **Flux + cluster bootstrap**.
- **드리프트 감지**: Argo CD의 OutOfSync를 알람으로 연결. "손으로 고친 것"이 반드시 드러나게.
- **정책 강제**: Kyverno / OPA Gatekeeper를 base에 포함 → 모든 클러스터가 동일한 가드레일.
- 클러스터 수명주기: **Cluster API**로 클러스터 자체도 선언적으로. 또는 Terraform + 모듈화.
- 신규 클러스터 온보딩은 **"골든 패스"** 로 자동화 — Git에 클러스터 정의 PR 하나로 프로비저닝 + 베이스 번들 설치 + 모니터링 등록 + 백업 등록까지.

**꼬리 질문**
- 클러스터별로 다를 수밖에 없는 것은 무엇인가?
  - → *기대*: IP 대역, 인증서/도메인, 스토리지 클래스(프로바이더 차이), 노드 인스턴스 타입, 규제 관련 정책. 이 목록이 짧게 유지되도록 관리하는 것이 설계 목표.
- 긴급 상황에 GitOps를 우회해 직접 고쳐야 한다면?
  - → *기대*: break-glass 절차를 미리 정의. 우회 시 자동 알람 + 사후 반드시 Git 반영(reconcile). Argo CD를 수동으로 중지하는 건 마지막 수단.

**평가 포인트**
- ⭕ 드리프트 감지를 알람으로 연결한다는 발상 = 운영 성숙도
- ❌ "Ansible로 30대 돌립니다"만 답하면 규모 감각 부족

---

### Q3-2. 30개 클러스터에 새 정책(예: 특정 레지스트리만 허용)을 배포한다. 롤아웃 전략은?

**기대 답변**

- **절대 한 번에 30개에 적용하지 않는다.** 링(ring) 기반 점진 롤아웃:
  - Ring 0: 내부 샌드박스 클러스터 (1개)
  - Ring 1: 내부 개발 (2~3개)
  - Ring 2: 고객사 스테이징
  - Ring 3: 저위험 프로덕션
  - Ring 4: 핵심 프로덕션
- 각 링 사이에 **베이크 타임(soak time)** 과 자동 중단 조건(에러율, Pod 실패율).
- 정책은 반드시 **Audit 모드 먼저**: Kyverno `validationFailureAction: Audit` → 위반 리포트 수집 → 예외 목록 확정 → `Enforce` 전환. 바로 Enforce하면 배포 전면 중단 사고.
- 위반 현황을 고객사/팀별로 공유하고 마이그레이션 기간 부여.
- 롤백 경로 명확히: Git revert 한 번으로 전 링 되돌릴 수 있는가?
- **어드미션 웹훅의 위험성**: `failurePolicy: Fail`인 웹훅이 죽으면 해당 리소스 생성이 전부 막힘. 네임스페이스 셀렉터로 kube-system 제외, 웹훅 자체는 HA, `timeoutSeconds` 짧게(5초 이하).

**평가 포인트**
- ⭕ Audit → Enforce 2단계를 말하면 실전 경험
- ⭕ failurePolicy로 인한 자기 봉쇄(self-lockout) 위험 언급 = 강한 가점

---

### Q3-3. 멀티 클러스터에서 서비스 디스커버리를 어떻게 할 것인가?

**기대 답변**

- 선택지와 트레이드오프:
  | 방식 | 장점 | 단점 |
  |---|---|---|
  | 클러스터별 독립 + 외부 DNS/LB | 단순, 결합도 낮음 | 지연, 페일오버 수동 |
  | 서비스 메시 (Istio multi-cluster, Linkerd multicluster) | mTLS, 트래픽 정책, 페일오버 | 운영 복잡도 급증, 사이드카 오버헤드 |
  | 전용 도구 (Submariner, Cilium Cluster Mesh) | L3/L4 직결, 오버헤드 낮음 | CNI 종속, IP 대역 중복 금지 |
  | 멀티클러스터 Service API (MCS) | 표준 지향 | 구현체 성숙도 편차 |
- **선행 조건**: 클러스터 간 Pod/Service CIDR 중복이 없어야 함. 30개 클러스터를 만들 때부터 **IPAM을 중앙 관리**하지 않으면 나중에 절대 못 붙임.
- MSP 관점에서는 **고객사 간 연결은 기본적으로 없어야 정상**. 크로스 클러스터 통신은 예외로 취급하고 명시적 허용.

**평가 포인트**
- ⭕ "먼저 IP 대역 계획부터"라고 답하면 크게 가점
- ⭕ 서비스 메시를 "일단 깔자"가 아니라 비용 대비로 판단하면 시니어

---

## 4. 네트워킹

### Q4-1. 1,000 노드 클러스터의 IP 대역을 설계해달라. 어떤 계산이 필요한가?

**기대 답변**

- **Pod CIDR**: `--node-cidr-mask-size`(기본 /24 = 노드당 254개 Pod 주소)로 노드마다 서브넷을 쪼갬.
  - 흔한 실수: 클러스터 CIDR `10.0.0.0/16` + 노드당 `/24` → **최대 256 노드**. 1,000 노드에서 노드 조인이 실패하며 `CIDR not available`.
  - 1,000 노드 + 노드당 /24 → 최소 **/14** (2^(24-14) = 1,024). 여유를 두면 **/13**.
  - 노드당 Pod가 실제로 30~50개면 노드 마스크를 `/25`나 `/26`으로 줄여 주소 절약 가능(단, 노드 재생성 없이 변경 불가).
- **Service CIDR**: 200 서비스면 `/22`(1,022개)로도 충분하지만, 헤드리스/임시 서비스를 감안해 여유. **Service CIDR은 클러스터 생성 후 변경이 매우 어렵다**(최근 버전에서 확장 기능이 들어왔으나 프로바이더 지원 편차 큼).
- **노드 네트워크**: 온프렘이면 VLAN/서브넷 소진 주의. 클라우드면 VPC 서브넷의 가용 IP가 노드 수 + ENI 기반 CNI라면 Pod IP까지 차지(AWS VPC CNI 등).
- **30 클러스터 전체를 아우르는 IPAM 스프레드시트/시스템**이 반드시 필요. 중복 대역은 나중에 클러스터 연결·VPN·피어링을 영구히 불가능하게 만듦.
- 예약 대역, 미래 클러스터 몫, 마이그레이션용 여유 대역을 미리 떼어둘 것.

**평가 포인트**
- ⭕ /16 + /24 = 256노드 함정을 즉시 계산해내면 매우 좋음
- ⭕ "나중에 못 바꾼다"는 비가역성을 인지하면 시니어

---

### Q4-2. kube-proxy의 iptables 모드와 IPVS, eBPF 모드의 차이는? 어떤 걸 고르겠는가?

**기대 답변**

| 모드 | 동작 | 스케일 특성 |
|---|---|---|
| iptables | 서비스/엔드포인트마다 룰 체인 생성, 선형 매칭 | 룰 **갱신**이 O(n) — 서비스·엔드포인트가 많으면 동기화 지연이 초 단위로 증가. 엔드포인트 변경 시 전체 테이블 재작성 부담 |
| IPVS | 커널 L4 로드밸런서 해시 테이블 | 룩업 O(1), 대규모 서비스에 유리. rr/lc/sh 등 알고리즘 선택 가능. 단 conntrack 관련 이슈, graceful termination 처리 주의 |
| eBPF (Cilium kube-proxy replacement) | eBPF 맵 기반, kube-proxy 자체를 제거 | 룩업/갱신 모두 효율적, DSR 지원, 관측성 우수. 커널 버전 요구(5.x+), 운영 학습 곡선 |

- **이 환경(200 서비스)에서는 서비스 수가 병목은 아님.** 진짜 변수는 **엔드포인트 수와 변동률(churn)**: 200 서비스 × 평균 50 Pod = 10,000 엔드포인트, 배포 때마다 대량 변경.
- **EndpointSlice**(1.21+ 기본)가 큰 Endpoints 오브젝트 문제를 해결 — 슬라이스당 기본 100 엔드포인트로 분할. Pod 하나 바뀔 때 전체 오브젝트가 아니라 해당 슬라이스만 갱신 → apiserver/etcd 부담 대폭 감소.
- 실무 판단: 기존 자산과 팀 역량을 고려. **새로 짓는다면 eBPF(Cilium) 우선 검토**, 안정성 우선이면 iptables + EndpointSlice로도 이 규모는 충분.

**꼬리 질문**
- `kube_proxy_sync_proxy_rules_duration_seconds`가 급증하면 어떤 의미인가?
  - → *기대*: 데이터플레인 반영이 늦어져 **배포 직후 트래픽이 죽은 Pod로 감** = 502의 원인.

---

### Q4-3. 서비스 배포 중에 간헐적으로 502/504가 발생한다. 원인은?

**기대 답변** (이 질문은 시니어 판별력이 매우 높음)

Pod를 삭제하면 **두 가지 일이 동시에, 비동기로** 일어난다:
1. Endpoint/EndpointSlice에서 제거 → kube-proxy/Ingress 컨트롤러가 반영 (**네트워크 전파, 수백 ms~수 초**)
2. 컨테이너에 **SIGTERM 즉시 전달**

즉 **Pod가 이미 종료를 시작했는데 아직 트래픽이 오는 구간**이 존재. 이게 502의 주원인.

**해결책**
- `preStop` 훅에 **sleep 5~15초** — SIGTERM을 지연시켜 엔드포인트 전파를 기다림 (앱 수정 없이 가능한 가장 효과적인 조치)
- `terminationGracePeriodSeconds`를 `preStop sleep + 실제 드레인 시간`보다 크게
- 애플리케이션이 **SIGTERM에서 즉시 종료하지 말고** 인플라이트 요청 처리 후 종료 (graceful shutdown)
- **readinessProbe를 실패로 전환**시켜 트래픽 차단 (단, 이것도 전파 시간 필요)
- Ingress 컨트롤러가 kube-proxy를 우회해 **Pod IP로 직접** 보내는 경우(NGINX Ingress 등)가 많음 → 컨트롤러의 엔드포인트 감지 지연도 함께 봐야 함
- Keep-alive 커넥션이 종료 중인 Pod에 붙어있는 경우 → 서버 측 `keepalive_timeout` 조정

**그 외 원인 후보**
- `maxUnavailable` 설정이 커서 동시에 너무 많이 내려감
- readinessProbe가 너무 관대해서 아직 준비 안 된 Pod에 트래픽 유입
- 업스트림 타임아웃 < 애플리케이션 처리 시간 (504)
- conntrack 테이블 가득 참

**평가 포인트**
- ⭕ "엔드포인트 제거와 SIGTERM이 동시에 일어난다"는 경쟁 조건을 정확히 설명하면 **거의 확실한 시니어**
- ❌ "재시작하면 됩니다" 수준이면 탈락

---

### Q4-4. CoreDNS가 1,000 노드에서 문제가 되는 이유와 대응은?

**기대 답변**

**문제**
- 클러스터 내 모든 DNS 쿼리가 소수의 CoreDNS Pod로 집중 → CPU 포화, 지연.
- **`ndots:5` 문제**: Pod의 `/etc/resolv.conf`는 기본 `ndots:5`. `api.example.com`(점 3개 < 5)을 조회하면 search 도메인을 먼저 붙여 **`api.example.com.default.svc.cluster.local` → `...svc.cluster.local` → `...cluster.local` → 실패 → 마지막에 실제 조회**. 외부 도메인 하나당 **4~5회 헛질의**. 트래픽이 5배로 뻥튀기.
- **5초 DNS 타임아웃**: A/AAAA 쿼리를 동일 소켓으로 병렬 전송할 때 커널 conntrack DNAT 경쟁 조건으로 패킷이 유실 → 리졸버 재시도까지 정확히 5초 지연. 대표적인 "가끔 5초 느려요" 증상.
- conntrack 테이블 소진.

**대응**
- **NodeLocal DNSCache** 도입 — 노드마다 캐시 데몬셋. 캐시 히트 + UDP conntrack 회피(TCP로 업스트림) → 5초 문제와 부하를 동시에 해결. **이 규모에서는 사실상 필수.**
- 애플리케이션 매니페스트에 `dnsConfig.options: [{name: ndots, value: "2"}]` 로 헛질의 축소, 또는 외부 도메인에 후행 점(`example.com.`) 사용.
- CoreDNS 리소스: `cluster-proportional-autoscaler`로 노드 수에 비례해 replica 조정, `cache` 플러그인 TTL 조정, `autopath` 플러그인 검토.
- CoreDNS를 kube-system에 두되 **PriorityClass system-cluster-critical** 부여, PDB 설정.
- 관측: `coredns_dns_request_duration_seconds`, `coredns_dns_responses_total{rcode="SERVFAIL"}`, 캐시 히트율.

**평가 포인트**
- ⭕ ndots:5와 5초 타임아웃 둘 다 설명하면 최상위
- ⭕ NodeLocal DNSCache를 즉시 꺼내면 대규모 경험자

---

### Q4-5. NetworkPolicy를 30개 클러스터 · 200 서비스에 적용하는 전략은?

**기대 답변**

- **기본 방향: default deny**. 네임스페이스마다 모든 ingress/egress를 막는 정책을 베이스로 깔고, 필요한 통신만 명시적 허용.
- 단, 한 번에 켜면 전부 끊어짐 → **관측 먼저**: Cilium Hubble / Calico flow log로 실제 통신 그래프를 수집 → 정책 자동 생성 → Audit → Enforce.
- 반드시 허용해야 하는 것: **DNS(kube-dns 53/UDP,TCP)** 를 빠뜨려 전체 장애가 나는 것이 1위 실수. 메트릭 스크레이핑(Prometheus), 헬스체크 경로도 마찬가지.
- 표준 NetworkPolicy의 한계: L7 규칙 없음, egress를 도메인 기준으로 못 씀, 클러스터 외부 CIDR 관리가 번거로움 → CNI 확장(CiliumNetworkPolicy, Calico GlobalNetworkPolicy) 또는 서비스 메시로 보완.
- 정책은 애플리케이션 팀이 작성하되, **플랫폼 팀이 default-deny 베이스와 필수 허용(DNS 등)을 제공**하는 분업.
- 30 클러스터 일관성은 GitOps + Kyverno "네임스페이스 생성 시 default-deny 자동 주입"으로 강제.

---

### Q4-6. Ingress를 200개 서비스에 제공한다. 아키텍처와 주의점은?

**기대 답변**

- **단일 Ingress 컨트롤러에 200개를 다 물리지 않는다.** NGINX Ingress는 설정 변경 시 config 재생성/reload가 발생하고, 규모가 크면 reload가 잦아져 커넥션 드롭·메모리 스파이크.
- 분리 전략: `ingressClassName`으로 컨트롤러를 **테넌트별/등급별로 분리**(public / internal / 고객사별). 폭발 반경도 함께 줄어듦.
- 컨트롤러 선택: NGINX(생태계 성숙) vs Envoy 계열(Contour, Emissary, Istio Gateway) vs Cilium Gateway. Envoy 계열은 **reload 없이 xDS로 동적 갱신**되어 대규모 변경에 유리.
- **Gateway API**로의 전환 검토: Ingress의 어노테이션 난립 문제를 해결하고 **역할 분리**(인프라팀 = GatewayClass/Gateway, 앱팀 = HTTPRoute)가 명확해 멀티테넌시에 적합.
- TLS: cert-manager로 자동 갱신. 200개 인증서면 **ACME 레이트 리밋** 주의, 와일드카드 인증서 + DNS-01 챌린지 조합 검토. 인증서 만료 알람은 만료 30일 전.
- 성능: 컨트롤러 Pod에 충분한 리소스, `keepalive` 튜닝, 노드 `hostNetwork` 또는 외부 LB의 프록시 프로토콜로 클라이언트 IP 보존.

---

## 5. 스케줄링 · 리소스 관리

### Q5-1. requests와 limits를 어떻게 설정하도록 가이드하겠는가? CPU limit은 걸어야 하는가?

**기대 답변**

**기본 개념**
- `requests`: **스케줄링 결정과 cgroup 가중치(cpu.shares)** 에 사용. 노드 용량 계산의 기준.
- `limits`: **런타임 상한**. CPU는 CFS quota로 스로틀링, 메모리는 초과 시 **OOMKill**.
- QoS 클래스: 모든 컨테이너가 `requests == limits`(CPU·메모리 둘 다) → **Guaranteed**, 일부만 설정 → **Burstable**, 아무것도 없음 → **BestEffort**. 노드 압박 시 **BestEffort → Burstable(초과분 큰 순) → Guaranteed** 순으로 축출.

**CPU limit 논쟁 (핵심)**
- CFS quota는 **100ms 주기**로 할당량을 재충전. limit 1 core면 100ms마다 100ms의 CPU 시간. 멀티스레드 앱이 순간적으로 4코어를 쓰면 25ms 만에 소진되고 **나머지 75ms를 강제로 멈춤**. 평균 사용률이 limit의 30%여도 **지연 spike가 발생**.
- 따라서 **지연에 민감한 서비스는 CPU limit을 걸지 않거나 넉넉히** 두고, requests로 공정 배분에 맡기는 것이 일반적 권장.
- 반면 **멀티테넌트 환경(MSP)** 에서는 노이지 네이버 방지를 위해 limit이 필요할 수 있음 → 이 긴장을 인지하고 **ResourceQuota + LimitRange로 네임스페이스 총량을 통제**하는 쪽으로 푸는 것이 더 나은 답.
- **메모리 limit은 반드시 설정**. 안 걸면 노드 전체를 잡아먹고 시스템 OOM으로 다른 Pod까지 죽임. 단 limit == request로 두어 Guaranteed로 만드는 것이 축출 방지에 유리.
- 관측: `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total` 비율이 스로틀링 지표.

**실무 프로세스**
- VPA를 `updateMode: Off`(추천만)로 돌려 **권장값 리포트** → 팀에 제공 → 점진 조정.
- 과다 request가 만연하면 클러스터 사용률이 20%대로 떨어짐. "request 대비 실사용" 대시보드를 팀별로 공개하는 것이 가장 효과적.

**평가 포인트**
- ⭕ CFS 100ms 주기와 스로틀링 메커니즘을 설명하면 강한 가점
- ⭕ CPU limit에 대해 "상황에 따라 다르다 + 이유"를 대면 시니어. "무조건 걸어야 한다"만 말하면 감점

---

### Q5-2. Pod를 여러 AZ·노드에 고르게 분산시키는 방법은? podAntiAffinity의 문제점은?

**기대 답변**

- `podAntiAffinity`(특히 `preferredDuringScheduling`)는 후보 노드마다 **기존 Pod들과의 매칭을 계산**해야 해서 사실상 O(Pods × Nodes). 1,000 노드에서는 **스케줄링 지연이 심각하게 증가**. 대규모에서는 피해야 함.
- 권장: **`topologySpreadConstraints`**
  ```yaml
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway   # DoNotSchedule은 스케줄 불가 위험
      labelSelector: { matchLabels: { app: web } }
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
  ```
- `whenUnsatisfiable` 선택이 핵심 트레이드오프: `DoNotSchedule`은 분산을 보장하지만 **용량 부족 시 Pending**으로 가용성을 해칠 수 있음. 프로덕션 기본은 `ScheduleAnyway` + 알람.
- 클러스터 전체 기본값은 스케줄러 프로파일의 `defaultConstraints`로 설정 가능 → 앱팀이 매번 안 써도 됨.
- 스케일 인/아웃 후 균형이 깨지면 **Descheduler**로 주기적 재배치 (단, 재배치는 Pod 재시작이므로 PDB와 함께 신중히).
- **PodDisruptionBudget**을 반드시 함께: 분산시켜 놓고 드레인 때 한꺼번에 내리면 의미 없음.

**평가 포인트**
- ⭕ antiAffinity의 계산 비용을 지적하면 대규모 경험자
- ⭕ DoNotSchedule의 가용성 리스크를 언급하면 가점

---

### Q5-3. PriorityClass와 선점(preemption)을 어떻게 활용하겠는가?

**기대 답변**

- 계층 예시:
  - `system-cluster-critical` / `system-node-critical` (예약됨) — CoreDNS, CNI, kube-proxy
  - `platform-critical` (2,000,000) — 모니터링, 로깅, 인그레스
  - `prod-high` (1,000,000) — 고객 프로덕션 핵심
  - `prod-default` (100,000)
  - `batch` (1,000) — 배치/분석
  - `best-effort` (0, `preemptionPolicy: Never`) — 유휴 자원 활용
- **선점의 함정**: 높은 우선순위 Pod가 들어오면 낮은 Pod를 죽이는데, 죽은 Pod가 다른 노드로 못 가면 **연쇄 선점(cascading preemption)** 발생. `preemptionPolicy: Never`를 저우선 워크로드에 쓰면 "선점당하지만 선점하지는 않음"으로 안정화.
- **멀티테넌트 주의**: 고객사가 마음대로 높은 PriorityClass를 쓰면 다른 고객사 워크로드를 죽일 수 있음 → **ResourceQuota의 `scopeSelector`로 PriorityClass 사용을 네임스페이스별로 제한**. 이것이 MSP에서 특히 중요.
- 플랫폼 컴포넌트(모니터링, 로깅)가 선점당해서 **장애 상황에 관측이 먼저 죽는** 사고를 막을 것.

**평가 포인트**
- ⭕ ResourceQuota scopeSelector로 PriorityClass 통제를 말하면 멀티테넌시 이해도 높음

---

### Q5-4. 노드 리소스 예약(system-reserved / kube-reserved / eviction)은 어떻게 설정하는가?

**기대 답변**

- 노드의 Allocatable = Capacity − `kube-reserved` − `system-reserved` − `eviction-hard`.
- **예약을 안 하면** Pod가 노드 리소스를 전부 소진 → kubelet/containerd/sshd가 굶어 **노드가 NotReady로 플랩**. 1,000 노드에서 이게 발생하면 대량 축출 폭풍으로 번짐.
- 권장 시작값 (노드 크기에 비례):
  - `kube-reserved`: cpu 100~200m, memory 1~2Gi, ephemeral-storage 1Gi
  - `system-reserved`: cpu 100~200m, memory 1Gi
  - `eviction-hard`: `memory.available<500Mi`, `nodefs.available<10%`, `imagefs.available<15%`
  - `eviction-soft` + `eviction-soft-grace-period`로 하드 축출 전에 완만한 대응 여지
- **`enforceNodeAllocatable`** 에 `pods`(+가능하면 `kube-reserved`, `system-reserved`)를 포함해 cgroup으로 실제 강제.
- 큰 노드(예: 64코어)에 고정값을 쓰면 낭비, 작은 노드에 큰 값을 쓰면 용량 손실 → **인스턴스 타입별 프로파일**로 관리.
- `--max-pods`(기본 110)와 Pod CIDR 마스크의 정합성 확인.
- 관측: `kubelet_node_name` 기준 Allocatable 대비 requests 총합, `kubelet_evictions` 카운터.

---

### Q5-5. 노드 드레인 시 고려사항은? 1,000 노드를 순차 교체한다면?

**기대 답변**

- `kubectl drain`은 cordon + eviction API 호출. **eviction API는 PDB를 존중**하므로 PDB가 잘못되면 드레인이 영원히 멈춤.
  - PDB `minAvailable: 100%` 또는 replica 1 + `minAvailable: 1` → **드레인 불가 데드락**. 매우 흔한 사고.
  - `maxUnavailable: 1`이 일반적으로 안전.
- 예외 처리: `--delete-emptydir-data`, `--ignore-daemonsets`, `--force`(고아 Pod). `--force`는 데이터 손실 가능성 인지하고 사용.
- StatefulSet은 순차적으로만 내려감 → 드레인 시간이 길어짐. 대용량 PV 재연결 시간도 고려.
- **대규모 교체 전략**:
  - 동시 드레인 노드 수를 제한 (예: 전체의 2~5%, 또는 AZ당 1대)
  - 각 노드 사이에 **재스케줄 완료 확인**(Pending Pod 0, 에러율 정상) 후 진행
  - 자동화: Cluster API의 `MachineDeployment` rolling update(maxSurge/maxUnavailable), 또는 노드 그룹 blue/green
  - **먼저 새 노드를 추가(surge)한 뒤 옛 노드를 뺀다** — 용량 부족으로 Pending이 쌓이는 것을 방지
  - Cluster Autoscaler/Karpenter가 동시에 개입하지 않도록 조율
- 사전 체크리스트: 모든 워크로드에 PDB가 있는가, 드레인 불가 PDB가 있는가(사전 스캔), 단일 replica 서비스 목록, 로컬 스토리지 사용 Pod 목록.

**평가 포인트**
- ⭕ PDB 데드락을 사전 스캔한다는 발상 = 대규모 운영 경험
- ⭕ surge-first(추가 후 제거) 순서를 말하면 가점

---

## 6. 오토스케일링

### Q6-1. HPA, VPA, Cluster Autoscaler, Karpenter를 조합할 때 주의점은?

**기대 답변**

- **HPA와 VPA의 충돌**: 둘 다 CPU를 기준으로 하면 서로 상쇄·진동. VPA는 메모리만, HPA는 CPU/커스텀 메트릭으로 역할 분리하거나 VPA를 추천 모드로만 사용.
- **HPA 튜닝**:
  - `behavior.scaleDown.stabilizationWindowSeconds`(기본 300s)를 늘려 플래핑 방지, `scaleUp`은 짧게.
  - CPU 기반은 후행 지표라 반응이 느림 → **큐 길이, RPS, p95 지연** 같은 선행 지표(custom/external metrics, KEDA)가 더 적합.
  - `minReplicas`를 1로 두면 스케일업 순간 콜드 스타트 문제 → 2 이상 권장.
- **Cluster Autoscaler**:
  - 노드 그룹 단위로 동작하고 **그룹 내 노드가 동질적이라고 가정** → 타입이 섞이면 오판.
  - 스케일 다운 차단 요인: PDB, 로컬 스토리지 Pod, `safe-to-evict: false` 어노테이션, kube-system Pod, 미러 Pod. **"왜 스케일 다운이 안 되는가"는 CA 로그가 답을 알려줌.**
  - `scale-down-unneeded-time`, `scale-down-utilization-threshold` 튜닝.
- **Karpenter**: 노드 그룹 없이 Pod 요구사항에 맞는 인스턴스를 직접 프로비저닝 → 빈 패킹과 비용 최적화가 뛰어남. `consolidation`이 노드를 적극적으로 교체하므로 **워크로드 중단 내성(PDB, graceful shutdown)이 전제**. 프로바이더 종속.
- **레이어 순서**: Pod 수(HPA) → 노드 수(CA/Karpenter). Pod가 Pending이어야 노드가 늘어나므로 **스케일업 지연 = HPA 반응 + 노드 프로비저닝 + 이미지 풀**. 총 지연이 SLO를 만족하는지 계산해야 함.
- 급증 대응: **오버프로비저닝 Pod**(낮은 PriorityClass의 pause 컨테이너)로 여유 노드를 미리 확보 → 실제 트래픽이 오면 선점되며 즉시 배치.

**평가 포인트**
- ⭕ 오버프로비저닝 트릭을 아는 것 = 실전 경험
- ⭕ 스케일업 총 지연을 SLO 관점에서 계산하면 최상위

---

### Q6-2. 이미지 풀 시간이 스케일업 지연의 병목이다. 어떻게 줄이겠는가?

**기대 답변**

- 이미지 크기 축소: 멀티스테이지 빌드, distroless/alpine 베이스, 레이어 캐시 최적화.
- **레지스트리 캐시/미러를 리전·클러스터 로컬에 배치** (Harbor proxy cache, ECR pull-through cache). 30개 클러스터가 외부 레지스트리를 동시에 때리면 레이트 리밋과 대역폭 비용 문제.
- 노드 이미지에 **자주 쓰는 베이스 이미지를 미리 굽기**(AMI baking).
- DaemonSet으로 **이미지 프리풀**(pause 컨테이너 + 이미지 사전 다운로드).
- `imagePullPolicy: IfNotPresent` + 태그 불변성(digest 고정).
- 노드당 동시 풀 제한: kubelet `--serialize-image-pulls=false`, `--max-parallel-image-pulls`로 병렬화(단 디스크 IO 고려).
- 이미지 지연 로딩(SOCI, stargz) 검토 — 대형 이미지에서 효과적.

---

## 7. 스토리지

### Q7-1. StatefulSet과 PV 운영에서 대규모일 때 겪는 문제는?

**기대 답변**

- **`volumeBindingMode: WaitForFirstConsumer`** 를 쓰지 않으면 PV가 먼저 특정 AZ에 만들어지고, Pod가 다른 AZ에 스케줄되어 **영원히 Pending**. 멀티 AZ면 사실상 필수 설정.
- **노드당 볼륨 어태치 한도**가 있음(클라우드 인스턴스 타입별로 상이, 예: 25~128). 스토리지 집약 워크로드를 한 노드에 몰면 어태치 실패. CSI의 `CSINode.allocatable.count`로 스케줄러가 인지.
- **Multi-Attach 에러**: 노드가 갑자기 죽으면 볼륨이 detach되지 않은 채로 남아 새 Pod가 attach 못 함. 노드 오브젝트가 삭제되거나 `force detach` 타임아웃(기본 6분)이 지나야 해소 → **StatefulSet 페일오버가 느린 근본 원인**. 노드 장애 시 자동 정리 파이프라인 필요.
- PVC 확장은 지원되지만 **축소는 불가**. 파일시스템 확장은 Pod 재시작이 필요할 수 있음(온라인 확장 지원 여부는 CSI 드라이버에 따라 다름).
- StatefulSet 삭제 시 PVC가 남음 → 고아 PVC가 쌓여 비용 발생. `persistentVolumeClaimRetentionPolicy`로 관리, 또는 주기적 고아 PVC 리포트.
- `ReclaimPolicy: Delete` vs `Retain`의 트레이드오프 — 프로덕션 데이터는 Retain + 별도 정리 프로세스가 안전.
- 백업: Velero + CSI VolumeSnapshot. **스냅샷은 애플리케이션 일관성을 보장하지 않음** → DB는 논리 백업(pg_dump, mysqldump) 또는 freeze 훅 병행.

**평가 포인트**
- ⭕ WaitForFirstConsumer를 즉시 말하면 멀티 AZ 운영 경험
- ⭕ Multi-Attach 에러와 force detach 타임아웃을 알면 장애 경험자

---

### Q7-2. 로컬 NVMe(local PV)를 쓰는 워크로드를 어떻게 운영하겠는가?

**기대 답변**

- Local PV는 **노드에 고정**되므로 노드가 죽으면 데이터도 끝. 복제는 **애플리케이션 레이어**(Kafka, Cassandra, Elasticsearch의 자체 복제)에 맡겨야 함.
- 노드 교체/업그레이드 시 데이터 재구축(rebalance) 시간이 길어 **순차 교체 속도의 상한**이 됨. 한 번에 한 노드, 복제 복구 완료 확인 후 다음.
- `nodeAffinity`가 PV에 박히므로 스케줄링 유연성이 사라짐 → 용량 파편화.
- 대안 비교: 네트워크 스토리지(안전, 느림) vs 로컬(빠름, 취약) vs 분산 스토리지 오퍼레이터(Rook/Ceph, OpenEBS — 운영 부담 큼).
- 실무: local-static-provisioner로 디스크 발견 자동화, 노드 교체 절차를 런북화.

---

## 8. 관측성 (Observability)

### Q8-1. 30개 클러스터 · 1,000 노드의 메트릭 수집 아키텍처를 설계하라.

**기대 답변**

- **클러스터별 Prometheus(수집) + 중앙 장기 저장소(Thanos / Mimir / VictoriaMetrics)** 가 표준 패턴.
  - 로컬 Prometheus는 짧은 보존(2~6시간~2일)만 담당 → 메모리·디스크 부담 최소화
  - `remote_write`로 중앙 전송, 또는 Thanos Sidecar + 오브젝트 스토리지
  - 중앙에서 전 클러스터 크로스 쿼리(글로벌 뷰)
- **Prometheus 페더레이션은 이 규모에서 안티패턴** — 중앙이 모든 클러스터를 스크레이핑하면 중앙이 단일 장애점이자 병목이 됨.
- **카디널리티가 최대의 적**:
  - 고카디널리티 라벨(pod name, request id, user id, container id)이 시계열을 폭발시킴
  - `metric_relabel_configs`로 수집 단계에서 드롭
  - `prometheus_tsdb_head_series` 감시, 팀별 카디널리티 예산 부여
  - cAdvisor 컨테이너 메트릭은 Pod 재생성마다 새 시계열 → 배포가 잦으면 급증
- **kube-state-metrics 샤딩**: 1,000 노드 규모면 단일 인스턴스가 버겁다. `--shard` / `--total-shards`로 수평 분할.
- 스크레이프 설정: `scrape_interval`을 30s~60s로(15s는 카디널리티×4배 비용), 중요 타깃만 짧게.
- **메타 모니터링**: Prometheus 자체를 감시할 별도 인스턴스. "모니터링이 죽었는데 아무도 모름"이 최악.
- 멀티테넌시: 중앙 저장소에서 `cluster`/`tenant` 라벨로 분리, Grafana 데이터소스/대시보드를 고객사별로 권한 격리.

**평가 포인트**
- ⭕ 페더레이션을 안티패턴으로 지적하면 가점
- ⭕ 카디널리티를 "예산"으로 관리한다는 발상 = 성숙한 운영

---

### Q8-2. 30개 클러스터의 알람이 하루 500건 온다. 어떻게 정리하겠는가?

**기대 답변**

- **원인이 아니라 증상(symptom)을 알람한다.** "CPU 80%"는 알람이 아니라 대시보드 지표. 사용자가 아픈 것(에러율, 지연, 가용성)만 페이지를 울려야 함.
- **SLO 기반 + 다중 윈도우 소진율(multi-window multi-burn-rate)**:
  - 빠른 소진(1h 윈도우, 14.4배) → 즉시 페이지
  - 느린 소진(6h/3d) → 티켓
  - 오탐과 탐지 지연을 동시에 줄이는 표준 기법
- **알람 계층 분리**: Page(즉시 호출) / Ticket(업무시간 처리) / Info(대시보드만). 대부분은 Page가 아니어야 함.
- Alertmanager: `group_by`(cluster, alertname, namespace), `inhibit_rules`(노드 다운 시 그 위 Pod 알람 억제), `silence` 절차, 유지보수 창 자동 사일런스.
- **고객사별 라우팅**: MSP이므로 `tenant` 라벨로 담당 팀/채널 분기.
- **정기적 알람 리뷰**: 주 1회 "울렸지만 아무 조치도 하지 않은 알람" 목록 → 삭제 또는 임계값 조정. 이 루틴이 없으면 반드시 다시 500건이 됨.
- 지표: 알람당 실제 조치율, 오탐률, MTTA/MTTR, 온콜 야간 호출 횟수.

**평가 포인트**
- ⭕ 소진율(burn rate) 알람을 알면 SRE 성숙도 높음
- ⭕ "조치하지 않은 알람은 삭제한다"는 원칙을 말하면 가점

---

### Q8-3. 200개 서비스의 로그와 트레이스는 어떻게 다루겠는가?

**기대 답변**

**로그**
- 노드 단위 DaemonSet 수집(Fluent Bit — Fluentd보다 가볍고 C 기반). 애플리케이션은 stdout/stderr만 쓰게 하고 파일 로깅 금지.
- **노드 디스크 보호**: containerd 로그 로테이션(`--container-log-max-size`, `--container-log-max-files`) 미설정 시 노드 디스크 가득 참 → DiskPressure → 축출 폭풍. 흔한 대형 사고.
- 백프레셔: 수집기가 밀리면 버퍼가 쌓이고 결국 유실. 디스크 버퍼 설정 + 유실률 모니터링.
- 비용 통제: 전량 저장은 비현실적. **구조화 로그 + 샘플링 + 등급별 보존**(에러 30일, 인포 7일). 인덱싱 비용이 큰 ELK 대신 Loki 같은 라벨 인덱스 방식 검토.
- 멀티테넌시: 고객사 로그의 **격리·접근 통제·보존 기간이 계약 사항**임을 인지.

**트레이스**
- OpenTelemetry로 계측 표준화, OTel Collector를 게이트웨이로.
- **샘플링 전략이 핵심**: head 샘플링(1~5%)은 싸지만 에러를 놓침 → **tail 샘플링**으로 에러/느린 트레이스는 100% 보존, 정상 트레이스는 낮은 비율.
- 200 서비스면 서비스 의존성 그래프 자체가 산출물 — 장애 시 영향 범위 파악에 직결.
- 로그/메트릭/트레이스를 `trace_id`와 `exemplar`로 상호 연결해야 실제로 쓰임.

---

## 9. 보안 · 멀티테넌시

### Q9-1. MSP 환경에서 고객사 간 격리를 어떻게 설계하겠는가? 네임스페이스로 충분한가?

**기대 답변**

- **소프트 멀티테넌시(네임스페이스 분리)**: 신뢰하는 내부 팀 사이에는 적절. 하지만 **커널을 공유**하므로 컨테이너 탈출 시 격리가 무너짐.
- **하드 멀티테넌시가 필요한 조건**(고객사 간, 규제 대상): 클러스터 분리 또는 노드 풀 분리 + 샌드박스 런타임(gVisor, Kata).
- 네임스페이스 단위로 반드시 세트로 제공해야 하는 것:
  - `ResourceQuota`(CPU/메모리/오브젝트 수/PVC/LoadBalancer 수) + `LimitRange`(기본값·상한)
  - `NetworkPolicy` default-deny
  - `Pod Security Admission` — `restricted` 또는 최소 `baseline` (PSP는 1.25에서 제거됨)
  - RBAC RoleBinding (ClusterRole 금지)
  - PriorityClass 사용 제한(`ResourceQuota.scopeSelector`)
- **네임스페이스로 막을 수 없는 것들**(반드시 언급되어야 함):
  - CRD, CustomResource, ClusterRole 등 **클러스터 스코프 리소스**
  - **노드 리소스 경합**(노이지 네이버) — 쿼터는 스케줄링만 통제
  - 커널 파라미터, 공유 파일시스템 캐시
  - **CoreDNS, Ingress 컨트롤러 등 공유 컴포넌트의 포화** — 한 테넌트가 DNS를 폭주시키면 전체 영향
  - etcd/apiserver 대역(→ APF로 완화)
- 실무 답: **고객사 = 클러스터**를 기본으로 하되, 소규모 고객은 공유 클러스터의 네임스페이스로 수용하고 **계약상 격리 수준을 명시**.

**평가 포인트**
- ⭕ "네임스페이스로 막을 수 없는 것"을 구체적으로 열거하면 매우 높은 평가
- ❌ "네임스페이스 나누면 됩니다"만 답하면 시니어 부적합

---

### Q9-2. RBAC를 어떻게 설계하겠는가? cluster-admin 남발을 어떻게 막는가?

**기대 답변**

- 원칙: **역할 템플릿화**. `tenant-admin` / `tenant-developer` / `tenant-viewer` / `platform-operator` / `auditor`를 표준 ClusterRole로 정의하고, 네임스페이스 단위 RoleBinding으로만 부여.
- 사람에게 직접 바인딩하지 말고 **그룹(IdP 연동)에 바인딩** — OIDC로 SSO, 입퇴사가 IdP에서 자동 반영.
- 위험한 권한을 명시적으로 인지:
  - `escalate`, `bind`, `impersonate` — RBAC 자체를 우회
  - `create pods` + hostPath/privileged — 사실상 노드 루트 (→ PSA로 차단)
  - `get/create secrets` — 전체 자격증명 노출
  - `exec`/`portforward` — 컨테이너 내부 접근. **감사 로그 필수**
  - `patch nodes`, `create clusterrolebindings`
- **break-glass**: 긴급 cluster-admin은 시간 제한 부여(JIT), 사용 즉시 알람, 사후 리뷰 필수. 상시 부여 계정은 0에 수렴해야 함.
- 정기 감사: `kubectl auth can-i --list`, RBAC 분석 도구(rbac-tool, kubectl-who-can)로 **cluster-admin 바인딩 목록을 30개 클러스터에서 주기적으로 수집**해 리포트.
- ServiceAccount: 기본 SA에 권한 주지 말 것, `automountServiceAccountToken: false` 기본화, 바운드 토큰(만료 있는 projected token) 사용, 장기 토큰 Secret 생성 지양.

---

### Q9-3. Secret 관리 전략은?

**기대 답변**

- **기본 Secret은 base64 인코딩일 뿐 암호화가 아니다.** etcd에 평문 저장.
- 최소 조치: **etcd 저장 시 암호화**(`EncryptionConfiguration`) — `aescbc`/`aesgcm`보다 **KMS 프로바이더(v2)** 가 권장(외부 KMS로 키 관리, 키 회전 지원).
- 더 나은 방향: **외부 시크릿 저장소**(Vault, AWS Secrets Manager, GCP Secret Manager) + **External Secrets Operator** 또는 **Secrets Store CSI Driver**로 동기화/마운트. 시크릿의 단일 원천과 감사 추적이 외부에 생김.
- **Git에 시크릿 금지**. GitOps 환경에서는 SOPS(+age/KMS) 또는 Sealed Secrets. 다만 장기적으로는 외부 저장소 참조 방식이 더 깔끔.
- 회전(rotation): 회전해도 Pod가 재시작되지 않으면 옛 값을 계속 씀 → Reloader 같은 컨트롤러로 롤아웃 트리거, 또는 앱이 파일 변경을 감지하도록.
- 유출 탐지: 리포지토리 시크릿 스캐닝(gitleaks), 이미지 레이어 스캐닝.
- RBAC로 Secret `get/list` 권한 최소화 — 네임스페이스 전체 Secret 조회 권한은 사실상 마스터키.

---

### Q9-4. 공급망 보안(supply chain)을 어떻게 구축하겠는가?

**기대 답변**

- **신뢰할 수 있는 레지스트리만 허용** — Kyverno/Gatekeeper 정책으로 이미지 레지스트리 allowlist, `:latest` 태그 금지, **digest 고정** 강제.
- 이미지 서명: **Cosign**으로 서명하고, 어드미션에서 서명 검증(Kyverno `verifyImages`, Sigstore policy-controller). 서명되지 않은 이미지는 배포 불가.
- **SBOM** 생성(syft) + 취약점 스캔(trivy, grype)을 CI에 통합. 레지스트리에서도 지속 스캔(새 CVE는 빌드 후에 발견됨).
- 게이트 기준 정의: Critical/High는 차단, 예외는 만료일 있는 승인으로만.
- 런타임: 읽기 전용 루트 파일시스템, non-root 실행, capability drop ALL, seccomp `RuntimeDefault` — 이걸 **PSA restricted**가 대부분 강제해줌.
- 런타임 위협 탐지(Falco, Tetragon)로 컨테이너 탈출·의심 행위 감지.
- **30 클러스터 CVE 대응 프로세스**: 취약 이미지가 어느 클러스터/네임스페이스에 몇 개 떠 있는지 **한 번의 쿼리로 답할 수 있어야 함**. 이게 안 되면 대응이 며칠 단위로 늘어짐.

**꼬리 질문**
- 심각한 CVE(예: 컨테이너 런타임 탈출)가 공개됐다. 첫 1시간에 무엇을 하겠는가?
  - → *기대*: ① 영향 범위 산정(어떤 버전이 어디에 몇 개) ② 완화책 즉시 적용 가능한지(정책으로 해당 기능 차단, 노드 격리) ③ 패치 롤아웃 링 계획 ④ 고객 커뮤니케이션 초안 ⑤ 탐지 규칙 추가. **"일단 다 패치"가 아니라 영향 범위 산정이 먼저** 나와야 함.

---

### Q9-5. 감사 로그(audit log)를 어떻게 활용하겠는가?

**기대 답변**

- `Policy` 레벨 설계: 대부분 `Metadata`, 민감 리소스(Secret, RBAC)는 `Request`, 읽기 남발(get/list on pods)은 `None`으로 볼륨 통제. 전부 `RequestResponse`로 두면 스토리지가 폭발.
- 수집: 노드 파일 → 로그 파이프라인 → 중앙 저장. **클러스터 밖으로 즉시 반출**해야 침해 시 조작 방지.
- 활용 쿼리 예: `exec`/`portforward` 사용 내역, cluster-admin 권한 행사, Secret 접근, 삭제 이벤트(누가 프로덕션 디플로이먼트를 지웠나), 비정상 시간대 접근.
- 알람: break-glass 계정 사용, RBAC 변경, 어드미션 웹훅 삭제.
- 보존 기간은 **고객 계약·규제 요건**에 따름 — MSP라면 이걸 먼저 확인한다는 답이 나와야 함.

---

## 10. 업그레이드 · 변경 관리

### Q10-1. 30개 클러스터를 1.x에서 1.x+1로 업그레이드한다. 계획을 세워보라.

**기대 답변**

**사전 준비**
1. **API 폐기(deprecation) 스캔**: `kubent`/`pluto`로 제거될 API 사용을 전수 조사. 헬름 릴리스 매니페스트와 CRD도 포함. **CI에도 넣어 재유입 차단.**
2. 컴포넌트 호환성 매트릭스: CNI, CSI, Ingress 컨트롤러, 서비스 메시, 오퍼레이터, 어드미션 웹훅 — 이들이 새 버전을 지원하는지. **이게 실제 병목인 경우가 대부분.**
3. **버전 스큐 정책** 확인: kube-apiserver가 가장 높아야 함. controller-manager/scheduler는 apiserver보다 1 마이너 낮은 것까지, kubelet/kube-proxy는 최근 정책상 최대 3 마이너까지 낮아도 됨(과거 2). **마이너는 한 번에 하나씩만 건너뜀.**
4. etcd 백업 + 복구 검증.
5. 릴리스 노트의 **변경된 기본값**(가장 위험) 확인.

**실행 순서 (클러스터 내부)**
etcd → kube-apiserver → controller-manager/scheduler → kubelet/kube-proxy(노드 순차) → 애드온

**실행 순서 (플릿)**
- Ring 0 샌드박스 → 내부 dev → 스테이징 → 저위험 프로덕션 → 핵심 프로덕션
- 링 사이 베이크 타임(최소 며칠), 중단 조건 명시
- 고객 커뮤니케이션: 유지보수 창 사전 고지, 영향 설명, 롤백 계획

**노드 업그레이드 방식**
- **불변 인프라 방식(권장)**: 새 이미지의 노드를 추가 → 드레인 → 삭제. 롤백이 쉬움.
- In-place 업그레이드: 빠르지만 상태가 누적되고 롤백이 어려움.

**롤백**
- **컨트롤 플레인 다운그레이드는 사실상 불가능**(etcd 스키마/API 변경). 그래서 "롤백"의 실체는 **etcd 스냅샷 복구 또는 클러스터 재생성**. 이 사실을 아는 것이 중요.
- 워크로드는 GitOps로 되돌릴 수 있음 — 컨트롤 플레인 롤백과 분리해서 생각.

**평가 포인트**
- ⭕ "다운그레이드는 안 된다"를 명확히 말하면 큰 가점
- ⭕ 애드온 호환성이 진짜 병목이라고 지적하면 실전 경험
- ❌ 업그레이드 순서만 외워서 말하면 Mid 수준

---

### Q10-2. 업그레이드 후 특정 워크로드만 장애가 났다. 어떻게 접근하는가?

**기대 답변**

- 먼저 **범위 확정**: 한 클러스터인가 전 클러스터인가, 한 네임스페이스인가, 특정 노드에만인가. 범위가 원인의 계층을 말해줌.
- 변경 델타 확인: 릴리스 노트의 동작 변경, 기본값 변경, 폐기 API, 애드온 동시 업그레이드 여부.
- 자주 나오는 원인:
  - 어드미션 웹훅이 새 API 버전을 이해 못 함
  - CRD의 저장 버전/변환 웹훅 문제
  - cgroup v1 → v2 전환으로 메모리 회계·OOM 동작 변화
  - 컨테이너 런타임(containerd) 업그레이드로 인한 동작 차이
  - PSA 기본 정책 강화로 Pod 생성 거부
  - kubelet 기본값 변경(이미지 GC, 축출 임계값)
- **즉시 조치와 근본 원인을 분리**: 고객 영향을 먼저 멈추고(해당 워크로드를 이전 버전 노드 풀로 이동, 정책 임시 예외) 원인 분석은 병행.
- 다른 29개 클러스터로의 확산 차단: **롤아웃 즉시 중단**이 첫 번째 행동.

---

## 11. 장애 시나리오 · 라이브 디버깅

> 면접관 팁: 정답을 맞히는지보다 **가설을 세우고 좁혀가는 순서**를 보라. "무슨 명령어를 치겠습니까?"를 반복해서 물으면 실력이 드러난다.

### Q11-1. Pod가 `Pending` 상태로 계속 머문다. 진단 순서는?

**기대 답변**
1. `kubectl describe pod` → Events에 스케줄러 메시지가 있음 (`0/1000 nodes are available: ...`) — **이 메시지가 원인의 90%를 알려줌**
2. 분기:
   - `Insufficient cpu/memory` → 용량 부족. requests 과다? 오토스케일러가 왜 안 늘었나(CA 로그)
   - `node(s) didn't match node selector/affinity` → 라벨 불일치
   - `node(s) had untolerated taint` → taint/toleration
   - `node(s) didn't find available persistent volumes to bind` → PV/StorageClass/AZ 문제
   - `node(s) didn't match pod topology spread constraints` → `DoNotSchedule` 제약
   - `pod has unbound immediate PersistentVolumeClaims` → PVC Pending, CSI 확인
   - 이벤트 자체가 없음 → **스케줄러가 죽었거나 밀림**. 스케줄러 로그·리더 선출 확인
3. ResourceQuota 초과 여부 (`kubectl describe quota`)
4. 어드미션 웹훅이 막고 있는지 (생성 자체가 거부되면 Pending이 아니라 생성 실패이지만, mutating 웹훅 지연으로 느려질 수 있음)

---

### Q11-2. Pod가 `Terminating`에서 30분째 멈춰 있다.

**기대 답변**
- 원인 후보:
  1. **Finalizer**가 남아 있음 → `kubectl get pod -o yaml`에서 `metadata.finalizers` 확인. 해당 컨트롤러가 죽었거나 오류. **무작정 finalizer를 지우면 리소스 누수/고아 발생** — 컨트롤러를 먼저 복구하는 게 정석.
  2. 애플리케이션이 **SIGTERM을 무시**하고 grace period가 매우 김
  3. **노드가 NotReady** → kubelet이 종료 처리를 못 함. 노드가 살아나거나 노드 오브젝트가 삭제돼야 정리됨
  4. 볼륨 언마운트 실패(NFS 행, CSI 드라이버 문제)
  5. 컨테이너 런타임 행(containerd/shim 프로세스 좀비)
- 진단: `kubectl describe`, kubelet 로그(`journalctl -u kubelet`), 노드에서 `crictl ps`, `ctr` 로 실제 컨테이너 상태 확인.
- **`--force --grace-period=0`은 최후의 수단** — apiserver에서 오브젝트만 지울 뿐 실제 컨테이너는 살아있을 수 있음. StatefulSet에서 쓰면 **스플릿 브레인** 위험.

**평가 포인트**
- ⭕ force delete의 위험성을 지적하면 시니어

---

### Q11-3. 노드 50대가 동시에 `NotReady`가 됐다. 첫 5분에 무엇을 하겠는가?

**기대 답변**
- **가장 먼저: 진짜 노드가 죽은 것인가, 아니면 apiserver/etcd가 느려서 상태 보고가 안 오는 것인가?** 이 구분이 전부를 가른다.
  - 노드에 직접 SSH해서 kubelet과 워크로드가 살아있는지 확인 → 살아있으면 **컨트롤 플레인 문제**
- 컨트롤 플레인 확인: apiserver 지연(`apiserver_request_duration_seconds` p99), etcd fsync/커밋 지연, 리더 변경, APF 거부율.
- 만약 컨트롤 플레인 문제라면 **축출 폭풍 방지가 최우선**. controller-manager의 `--large-cluster-size-threshold` / `--unhealthy-zone-threshold` 덕분에 대량 NotReady 시 축출이 레이트 리밋되지만, 설정을 확인하고 필요시 노드 컨트롤러를 일시 정지하는 것도 고려.
- 노드 자체 문제라면 공통점 찾기: 같은 AZ? 같은 인스턴스 타입? 같은 노드 이미지? 같은 시각에 배포된 DaemonSet? 커널/드라이버 업데이트?
- 흔한 원인: CNI 플러그인 크래시(데몬셋 롤아웃), 디스크 풀(로그/이미지), conntrack 소진, 노드 로컬 DNS 장애, 메모리 고갈로 kubelet 굶김, 클라우드 프로바이더 네트워크 이슈.
- 커뮤니케이션: 영향받은 고객사 파악 및 상태 페이지 업데이트를 **동시에** 시작.

**평가 포인트**
- ⭕ "노드가 죽은 게 아니라 apiserver가 느린 것일 수 있다"를 먼저 말하면 최상위
- ⭕ 축출 폭풍 통제를 떠올리면 대규모 경험자

---

### Q11-4. 특정 서비스만 가끔 5초씩 느리다. 원인은?

**기대 답변**
- **DNS를 가장 먼저 의심**. 정확히 5초는 리졸버 재시도 타임아웃의 시그니처(Q4-4 참조). conntrack DNAT 경쟁 조건.
- 확인: Pod 안에서 `dig`/`time nslookup` 반복, CoreDNS 지연 메트릭, 노드의 `conntrack -S`에서 `insert_failed` 카운터 증가 여부.
- 해결: NodeLocal DNSCache, `ndots` 조정, `single-request-reopen`, TCP 강제.
- 다른 후보: JVM GC stop-the-world, 커넥션 풀 고갈, TCP 재전송(RTO 최소값), 스토리지 IO 스톨, CPU 스로틀링(CFS), 업스트림 keep-alive 만료 후 재연결.
- 방법론: **p50이 아니라 p99를 보고, 어느 홉에서 시간이 소비되는지 트레이스로 분해**한다.

---

### Q11-5. 클러스터 전체가 느려졌다. `kubectl get pods`가 10초 걸린다.

**기대 답변**
- 계층별 분해:
  1. `kubectl get --raw /healthz`, apiserver 지연 메트릭 — apiserver 자체인가?
  2. etcd 지표(fsync, 커밋, DB 크기, 리더 변경) — 대개 여기서 나옴
  3. APF 큐 대기/거부 — 누군가 폭주 중인가? `apiserver_request_total`을 User-Agent별로 집계
  4. apiserver 메모리/CPU, watch 캐시 미스
  5. 네트워크(LB 편중, 커넥션 쏠림)
- **가장 흔한 실제 원인**: 잘못 만든 컨트롤러/CI 스크립트가 전체 리소스를 반복 LIST. 또는 etcd 디스크 성능 저하, 또는 대량 오브젝트(수십만 Event/Secret) 축적.
- 즉시 완화: 문제 클라이언트 차단(APF 낮은 우선순위로 격리, RBAC 회수, 스케일 다운), Event TTL 축소, etcd 압축/defrag.
- 재발 방지: 클라이언트 측 informer 사용 강제, APF 정책 정비, LIST 요청 대시보드 상시화.

---

### Q11-6. 메모리 관련 장애: `OOMKilled`가 났다. 어떻게 구분하고 대응하는가?

**기대 답변**
- 두 가지를 구분해야 함:
  - **cgroup OOM (컨테이너 limit 초과)**: 해당 컨테이너만 죽음. `kubectl describe pod`에 `Reason: OOMKilled`, exit code 137.
  - **노드 시스템 OOM**: 노드 전체 메모리 고갈로 커널 OOM killer가 동작. `dmesg`에서 확인. 무관한 Pod가 같이 죽고, kubelet까지 위험.
  - **kubelet 축출(eviction)**: OOMKill이 아니라 `Evicted` 상태. `memory.available` 임계값 기반. QoS 순서대로 축출.
- 대응:
  - limit 상향 전에 **실제 누수인지 워킹셋 증가인지** 판별 (프로파일링, `container_memory_working_set_bytes` 추세)
  - JVM/Go 등 런타임의 힙 설정이 컨테이너 limit을 인식하는지 확인 (`-XX:MaxRAMPercentage`, `GOMEMLIMIT`)
  - 페이지 캐시가 working set에 포함되어 오해를 부르는 경우 주의
  - 노드 OOM이면 예약값(system-reserved) 미설정이 근본 원인일 가능성
- 예방: 메모리는 `request == limit`(Guaranteed)로 두는 것을 권장, 축출 임계값 설정, 팀별 OOM 발생률 대시보드.

---

### Q11-7. `CrashLoopBackOff`를 체계적으로 진단하는 순서는?

**기대 답변**
1. `kubectl logs <pod> --previous` — **이전 컨테이너의 로그**가 핵심 (현재 컨테이너는 아직 시작 전일 수 있음)
2. `kubectl describe pod` → Last State의 exit code
   - 0: 정상 종료했는데 restartPolicy가 Always (배치를 Deployment로 돌린 실수)
   - 1/2: 애플리케이션 에러
   - 137: SIGKILL (OOMKilled 또는 graceful period 초과)
   - 139: SIGSEGV
   - 143: SIGTERM
3. 설정 문제: ConfigMap/Secret 누락, 환경변수 오타, 마운트 실패
4. 프로브 문제: **livenessProbe가 너무 공격적**이어서 기동 중인 앱을 죽임 → `startupProbe`로 분리하는 것이 정석
5. 의존성: DB/외부 API 미준비 → 앱이 재시도 로직 없이 즉시 종료
6. 권한: PSA/securityContext로 non-root 강제됐는데 앱이 root 필요
7. 리소스: 노드에 자원은 있으나 limit이 너무 낮음
- 디버깅 도구: `kubectl debug`로 **임시 디버그 컨테이너** 주입(이미지에 셸이 없는 distroless에서 특히 유용), `kubectl debug node/<node>`로 노드 접근.

**평가 포인트**
- ⭕ `--previous`와 `startupProbe`를 언급하면 실무 경험
- ⭕ `kubectl debug`(ephemeral container)를 알면 최신 기술 추적 중

---

### Q11-8. 디스크가 가득 차서 노드에 DiskPressure가 발생했다.

**기대 답변**
- 소비 주체 확인: 컨테이너 로그 / 이미지 레이어 / emptyDir / 컨테이너 쓰기 가능 레이어 / 시스템 로그.
- kubelet은 `imagefs.available` 임계값에서 **이미지 GC**를 수행(사용하지 않는 이미지 삭제). 그래도 부족하면 Pod 축출.
- 근본 원인 1위: **컨테이너 로그 로테이션 미설정**. containerd/kubelet의 `container-log-max-size`, `container-log-max-files` 설정.
- 2위: emptyDir에 무제한 쓰기 → `sizeLimit` 설정 + ephemeral-storage requests/limits 강제(LimitRange).
- 3위: 이미지 누적 → GC 임계값 조정, 노드 수명 단축(불변 인프라라면 자연 해소).
- 예방: 노드 디스크 사용률 알람(70/85%), ephemeral-storage 쿼터, 로그 볼륨 상위 네임스페이스 리포트.

---

## 12. 시스템 설계 과제 (화이트보드 60분)

> 정답이 없는 문제. **질문을 던지는가, 트레이드오프를 명시하는가, 실패 모드를 먼저 생각하는가**를 본다.

### 설계 과제 A — 플릿 플랫폼 설계

> "30개 클러스터, 1,000 노드, 200 서비스를 3명이 운영해야 합니다. 플랫폼을 처음부터 설계한다면?"

**좋은 답변에 반드시 등장해야 할 요소**

| 영역 | 기대 내용 |
|---|---|
| 클러스터 수명주기 | Cluster API / Terraform, 클러스터 정의도 Git에 |
| 설정 배포 | GitOps (Argo CD ApplicationSet / Flux), base+overlay 계층 |
| 정책 | Kyverno/Gatekeeper를 base 번들에 포함, Audit→Enforce |
| 관측 | 클러스터 로컬 수집 + 중앙 집계, 메타 모니터링 |
| ID/접근 | OIDC SSO, 그룹 기반 RBAC, JIT break-glass |
| 네트워크 | 중앙 IPAM, 대역 중복 금지 |
| 온보딩 | 신규 클러스터 = PR 1개 → 30분 내 완성 (골든 패스) |
| 셀프서비스 | 앱팀이 플랫폼팀을 거치지 않고 배포·조회할 수 있는 범위 정의 |
| 표준화 | "지원하는 구성"의 목록을 명시하고 그 외는 거절 — 3명이 30개를 운영하는 유일한 방법 |

**반드시 되물어야 할 질문들 (후보자가 물으면 가점)**
- SLA/SLO는? 고객사별로 다른가?
- 클라우드/온프렘 혼합인가?
- 고객사가 클러스터에 직접 접근하는가?
- 기존 자산을 이관하는가, 신규인가?
- 온콜 커버리지 요구(24/7?)와 인력은?

**평가 포인트**
- ⭕ **"3명으로 30개"라는 제약을 설계의 중심에 놓는가** — 표준화·자동화·거절 기준을 말하면 최상위
- ❌ 기술 스택 나열만 하고 인력 제약을 무시하면 감점

---

### 설계 과제 B — 멀티테넌트 온보딩

> "새 고객사가 들어옵니다. 전용 클러스터 1개, 예상 50 노드, 20개 마이크로서비스. 온보딩 설계를 해보세요."

**체크리스트 (후보자가 스스로 만들어야 함)**
- 네트워크 대역 할당(중앙 IPAM에서), VPC/피어링, 인그레스 도메인·인증서
- 클러스터 프로비저닝 + 베이스 번들(CNI, CSI, 모니터링, 로깅, 정책, 백업)
- 고객 계정 RBAC 설계, SSO 연동, 접근 범위 합의
- 쿼터·리밋·정책 수준 합의 (얼마나 엄격하게? 예외 프로세스는?)
- SLO 정의 및 측정 방법 합의 — **"가용성 99.9%"가 무엇을 측정하는지 문서화**
- 알람 라우팅, 온콜 에스컬레이션, 고객 연락 창구
- 백업/DR 요건과 복구 목표(RTO/RPO) 합의 및 훈련 일정
- 런북 작성, 인수인계 세션
- 비용 가시성(쇼백/차지백) 설정
- **이탈(offboarding) 절차도 미리 정의** — 데이터 반환·삭제 증빙

---

### 설계 과제 C — 재해 복구

> "한 고객사의 프로덕션 클러스터가 리전 장애로 완전히 사라졌습니다. 4시간 안에 복구해야 합니다."

**기대 답변**
- 사전 준비가 전부: **복구 계획은 장애 시점에 만드는 것이 아니라 이미 있어야 함**
- 구성 요소별 복구 경로
  - 클러스터 인프라: IaC로 다른 리전에 재생성 (사전에 테스트된 상태여야)
  - 워크로드 정의: **GitOps라면 Git이 원천 — 동기화만 하면 복구**
  - 데이터: PV 스냅샷의 크로스 리전 복제 여부, DB는 별도 복제본
  - 시크릿: 외부 시크릿 저장소면 참조만 복구
  - DNS/트래픽: TTL을 낮게 유지했는가? 전환 절차는?
- **RTO 4시간을 달성하려면** 무엇이 사전에 준비돼야 하는지 역산: 이미지가 다른 리전 레지스트리에 있는가, 쿼터가 확보돼 있는가, 인증서가 발급 가능한가
- 훈련: **게임데이로 실제 복구를 해봤는가**가 핵심 질문
- 복구 중 의사결정 구조: 인시던트 커맨더, 고객 커뮤니케이션 담당 분리

---

## 13. 비용 · 효율

### Q13-1. 클러스터 사용률이 20%다. 어떻게 개선하겠는가?

**기대 답변**
- 먼저 **원인 진단**: 낮은 사용률은 대개 ① request 과다 설정 ② 스케일 다운 차단 ③ 파편화 ④ 의도적 여유(HA/버스트)
- "request 대비 실사용" 리포트를 **팀별·네임스페이스별로 공개** — 데이터 없이 줄이자고 하면 반발만 생김
- VPA 추천 모드로 근거 있는 조정값 제공, 점진 적용
- 스케줄러 `MostAllocated` 스코어링으로 빈 패킹 강화(단, 장애 내성 감소 트레이드오프)
- Karpenter/CA의 consolidation으로 유휴 노드 정리
- 스팟/프리엠티블 인스턴스를 내결함성 워크로드에 적용 + 중단 핸들러
- 배치 워크로드를 낮은 PriorityClass로 유휴 자원에 채움
- **과도한 최적화 경고**: 사용률 80%를 목표로 하면 버스트에 대응 못 하고 장애 시 재스케줄 공간이 없음. **가용성과의 균형**을 명시적으로 말해야 함.
- MSP 관점: 비용 절감이 곧 마진이지만, 고객 SLA 위반 리스크와 비교 형량해야 함. 쇼백/차지백으로 고객이 스스로 최적화하게 만드는 것이 가장 효과적.

**평가 포인트**
- ⭕ "데이터를 팀에 공개"라는 조직적 해법을 말하면 시니어
- ❌ 가용성 트레이드오프를 언급 안 하면 감점

---

## 14. MSP 운영 · 행동 질문

> 기술만큼 중요. 특히 **고객이 있는 운영**에서의 판단력을 본다.

### Q14-1. 본인이 낸 변경으로 프로덕션 장애가 발생한 경험을 말해달라.

**좋은 답변의 구조**
- 무슨 일이 있었는지 구체적으로(수치 포함: 영향 시간, 영향 범위)
- 본인의 역할과 판단 — **책임을 회피하지 않음**
- 즉시 완화와 근본 원인을 어떻게 분리했는지
- 재발 방지책이 **개인의 주의력이 아니라 시스템/프로세스 변경**이었는지 (← 가장 중요한 신호)
- 포스트모템을 비난 없이(blameless) 진행했는지

**레드 플래그**
- 장애 경험이 없다고 하는 것 (규모 있는 운영을 안 해봤거나 솔직하지 않음)
- 다른 팀/사람 탓만 함
- "다음부터 더 조심하겠다"로 끝남

---

### Q14-2. 고객사가 위험한 설정을 강하게 요구한다(예: 모든 Pod에 privileged, cluster-admin 상시 부여). 어떻게 하겠는가?

**기대 답변**
- 먼저 **왜 필요한지 근본 요구를 파악** — 대부분 다른 방법이 있음(특정 capability만, 시간 제한 권한, 전용 노드 풀)
- 리스크를 **고객의 언어로** 설명: 기술 용어가 아니라 "다른 고객 워크로드에 접근 가능해집니다", "감사 대응 시 문제가 됩니다"
- 대안을 반드시 함께 제시. 거절만 하는 것은 답이 아님
- 그래도 요구하면: **의사결정을 문서화**하고 리스크 수용 주체를 명확히(계약·서면 승인), 보상 통제(격리된 노드 풀, 강화된 감사·탐지) 적용
- 회사 내부적으로 **정책 예외 승인 프로세스**가 있어야 하고 그걸 따른다
- 절대 타협 불가 선(다른 고객 데이터 접근 가능성 등)은 명확히 유지

**평가 포인트**
- ⭕ "근본 요구 파악 → 대안 → 문서화된 리스크 수용" 3단계가 나오면 우수
- ❌ 무조건 거절 또는 무조건 수용 둘 다 감점

---

### Q14-3. 새벽 3시에 호출됐다. 고객사 서비스가 다운. 첫 10분의 행동 순서는?

**기대 답변**
1. **영향 범위 확인이 최우선** — 한 서비스인가, 클러스터 전체인가, 여러 고객사인가
2. **커뮤니케이션 시작** — 인지했다는 사실을 알리는 것만으로도 고객 불안이 크게 줄어듦. 기술적 해결과 병행, 지연시키지 말 것
3. **최근 변경 확인** — 배포, 설정 변경, 인프라 변경. 장애의 압도적 다수는 변경에서 옴
4. **완화 우선, 원인은 나중** — 롤백/페일오버/스케일아웃으로 먼저 멈춘다. "원인을 알아야 고친다"는 새벽 3시에 위험한 태도
5. 혼자 오래 붙들지 말고 **에스컬레이션 기준을 지킴** (예: 15분 내 진전 없으면 호출)
6. 타임라인 기록 — 나중 포스트모템과 고객 보고의 근거
7. 인시던트 커맨더 역할 분리(조사 / 커뮤니케이션 / 실행)

**평가 포인트**
- ⭕ "완화 우선"과 "커뮤니케이션 병행"을 말하면 실전 온콜 경험자
- ❌ 바로 로그 뒤지러 들어가면 Mid 수준

---

### Q14-4. 팀에서 특정 기술 도입을 두고 의견이 갈린다(예: 서비스 메시 도입). 어떻게 결론을 내리겠는가?

**기대 답변**
- **해결하려는 문제를 먼저 정의** — 도구가 아니라 문제에서 시작
- 도입 비용을 정직하게 산정: 학습 곡선, 운영 부담, 사이드카 리소스 오버헤드, 디버깅 복잡도, 업그레이드 부담(30 클러스터면 ×30)
- 더 단순한 대안이 문제의 80%를 푸는지 검토
- **작은 범위 PoC + 명시적 성공 기준 + 기한** 설정, 실패 시 철수 조건도 미리 정의
- ADR(Architecture Decision Record)로 결정과 근거를 기록 — 나중에 재논의 비용을 줄임
- 반대 의견을 낸 사람이 실행에서 소외되지 않도록 (disagree and commit)

---

### Q14-5. 반복 운영 업무가 팀 시간의 60%를 차지한다. 어떻게 줄이겠는가?

**기대 답변**
- **먼저 측정**: 어떤 티켓/작업이 얼마나 반복되는지 분류. 감이 아니라 데이터.
- 상위 3개에 집중 — 롱테일 자동화는 ROI가 낮음
- 단계적 접근: 문서화 → 런북 → 스크립트 → 셀프서비스 → 완전 자동화(오퍼레이터). **모든 것을 오퍼레이터로 만들 필요는 없음**
- 셀프서비스로 넘기는 것이 자동화보다 나은 경우가 많음 (앱팀이 직접 하게)
- 알람 정리로 불필요한 개입 자체를 제거
- **토일(toil) 예산**을 명시적으로 관리 — 스프린트의 일정 비율을 반드시 자동화에 배정하지 않으면 영원히 줄지 않음

---

### Q14-6. 주니어 엔지니어가 프로덕션에서 위험한 명령을 실행하려 한다. 어떻게 하겠는가?

**기대 답변**
- 즉시 개입(막는다) + **비난하지 않고** 이유 설명
- 근본 원인은 개인이 아니라 **시스템**: 왜 그 명령이 가능했는가? RBAC가 너무 넓지 않은가? 확인 절차가 없는가? 문서가 부족한가?
- 가드레일 추가: 프로덕션 접근에 승인 필요, 위험 명령 래퍼, 드라이런 기본화
- 학습 기회로 전환 — 페어링, 스테이징에서 재현
- 이런 답에서 **심리적 안전감에 대한 태도**가 드러남. "그 사람 권한을 뺏겠다"만 말하면 감점

---

## 15. 빠른 검증 질문 (워밍업 / 스크리닝, 각 1~2분)

| # | 질문 | 핵심 답변 |
|---|---|---|
| 1 | Deployment / StatefulSet / DaemonSet 선택 기준 | 상태 유무, 안정적 네트워크 ID·순서 필요 여부, 노드마다 1개 필요 여부 |
| 2 | liveness / readiness / startup 프로브 차이 | 재시작 / 트래픽 차단 / 기동 유예. **liveness 과민이 장애 원인 1위** |
| 3 | ConfigMap 변경 시 Pod에 반영되나? | 볼륨 마운트는 결국 갱신(kubelet 동기화 주기, subPath는 갱신 안 됨), env는 절대 안 됨. 재시작 필요 |
| 4 | `kubectl apply`와 `replace`, `patch`의 차이 | 3-way merge(last-applied-configuration) vs 전체 교체 vs 부분 수정. Server-Side Apply의 필드 소유권 |
| 5 | Service의 ClusterIP / NodePort / LoadBalancer / ExternalName | 각각의 용도와 한계 |
| 6 | Headless Service는 왜 쓰는가 | clusterIP: None → DNS가 Pod IP 직접 반환. StatefulSet, 클라이언트 사이드 LB |
| 7 | Init container와 sidecar의 차이 | 순차 완료 vs 병렬 실행. 네이티브 사이드카(restartPolicy: Always인 init container)의 등장 배경 |
| 8 | `Recreate`와 `RollingUpdate` | 다운타임 허용 여부, 리소스 여유, 스키마 호환성 |
| 9 | Job과 CronJob의 함정 | `backoffLimit`, `activeDeadlineSeconds`, `concurrencyPolicy`, `startingDeadlineSeconds`, 실패한 Job 누적 |
| 10 | Taint/Toleration과 NodeAffinity의 차이 | 노드가 밀어내는 것 vs Pod가 끌리는 것. 전용 노드 풀은 taint가 정석 |
| 11 | `hostNetwork: true`의 리스크 | 포트 충돌, 격리 상실, NetworkPolicy 미적용 |
| 12 | CRD와 오퍼레이터 | 선언적 API 확장 + reconcile 루프. 언제 만들지 말아야 하는가도 |
| 13 | Helm과 Kustomize | 템플릿/패키징 vs 오버레이 패치. 함께 쓰는 조합 |
| 14 | Pod가 IP를 얻는 과정 | kubelet → CRI → CNI ADD → IPAM 할당 → veth/라우팅 설정 |
| 15 | 컨테이너 이미지의 레이어와 캐시 | 빌드 최적화, 변경 빈도 낮은 것을 먼저 |

---

## 16. 실습 과제 (선택, 사전 과제 또는 페어링 60~90분)

### 과제 1 — 라이브 디버깅 (가장 변별력 높음)
고장난 클러스터(kind/minikube)를 준비하고 3~4개의 결함을 심는다.
- 예: readinessProbe 경로 오타 / NetworkPolicy가 DNS를 차단 / PDB가 드레인을 막음 / requests가 노드 용량 초과 / ConfigMap 키 이름 불일치
- 관찰 포인트: 가설 → 검증 순서, 어떤 명령을 어떤 순서로 치는지, 막혔을 때 어떻게 방향을 바꾸는지, 소리 내어 생각하는지

### 과제 2 — 매니페스트 리뷰
문제 많은 Deployment YAML을 주고 리뷰하게 한다. 심을 결함:
- `image: myapp:latest`, requests/limits 없음, 프로브 없음, replicas 1
- `runAsUser: 0`, `privileged: true`, hostPath 마운트
- PDB 없음, topologySpread 없음, `terminationGracePeriodSeconds: 0`
- Secret을 환경변수로 평문 주입, `imagePullPolicy: Always` + 태그 고정 안 함
- 관찰 포인트: 몇 개를 찾는가보다 **우선순위를 매기는가**, 리뷰 코멘트의 톤(강압적인지 교육적인지)

### 과제 3 — 포스트모템 작성
장애 타임라인 원자료를 주고 포스트모템 문서를 쓰게 한다.
- 관찰 포인트: 근본 원인을 개인이 아닌 시스템에서 찾는가, 액션 아이템이 검증 가능한가, 타임라인이 정확한가

---

## 17. 레드 플래그 (하나라도 강하게 나오면 재검토)

- ❌ **모든 답이 "그때그때 다릅니다"** — 트레이드오프를 안다는 것과 판단을 못 하는 것은 다름. 시니어는 **조건을 정하고 결론을 낸다**
- ❌ 규모 감각 부재 — 100 노드와 1,000 노드의 차이를 설명하지 못함
- ❌ 장애 경험을 말하지 못하거나 전부 남 탓
- ❌ 최신 도구 이름만 나열하고 왜 쓰는지, 언제 쓰면 안 되는지를 모름
- ❌ 고객/앱팀을 "무지한 사람들"로 취급하는 태도 (MSP에서 치명적)
- ❌ 자동화를 말하지만 롤백·안전장치는 생각 안 함
- ❌ 보안을 "나중에 할 일"로 취급
- ❌ 모르는 것을 아는 척함 — **"모른다, 이렇게 찾아보겠다"가 훨씬 좋은 답**
- ❌ 문서화/커뮤니케이션을 부가 업무로 여김

---

## 18. 채점표 (면접관용)

| 역량 | 관련 질문 | 1 | 2 | 3 | 4 | 비고 |
|---|---|---|---|---|---|---|
| 컨트롤 플레인 · etcd | Q1-2~5, Q2-1~3 | ☐ | ☐ | ☐ | ☐ | |
| 네트워킹 | Q4-1~6 | ☐ | ☐ | ☐ | ☐ | |
| 스케줄링 · 리소스 | Q5-1~5, Q6-1 | ☐ | ☐ | ☐ | ☐ | |
| 플릿 · GitOps | Q3-1~3, 설계 A | ☐ | ☐ | ☐ | ☐ | |
| 관측성 | Q8-1~3 | ☐ | ☐ | ☐ | ☐ | |
| 보안 · 멀티테넌시 | Q9-1~5 | ☐ | ☐ | ☐ | ☐ | |
| 변경 관리 · 업그레이드 | Q10-1~2, Q3-2 | ☐ | ☐ | ☐ | ☐ | |
| 장애 진단력 | Q11-1~8, 실습 1 | ☐ | ☐ | ☐ | ☐ | |
| 비용 · 효율 | Q13-1 | ☐ | ☐ | ☐ | ☐ | |
| 커뮤니케이션 · 판단 | Q14-1~6 | ☐ | ☐ | ☐ | ☐ | |

**채용 기준 제안**
- **장애 진단력**과 **플릿·GitOps**는 3점 미만이면 이 규모에서 부적합
- **보안·멀티테넌시**는 MSP 특성상 2점 미만이면 부적합
- 나머지는 평균 2.5점 이상 + 학습 의지가 확인되면 온보딩으로 보완 가능
- 만점자를 기다리지 말 것. **"이 사람이 없을 때보다 팀이 나아지는가"** 가 실제 기준

---

## 19. 후보자가 우리에게 묻기를 기대하는 질문 (역질문 평가)

좋은 시니어는 다음을 묻습니다. 물어보면 가점:
- 온콜 로테이션은 어떻게 되고, 야간 호출 빈도는 실제로 얼마인가?
- 현재 가장 큰 운영 고통(toil)은 무엇인가?
- 클러스터 업그레이드 주기와 현재 버전 분산은?
- 포스트모템 문화가 있는가? 최근 포스트모템을 하나 보여줄 수 있는가?
- 플랫폼팀과 앱팀의 책임 경계는 어디인가?
- 기술 부채를 갚는 시간이 스프린트에 배정돼 있는가?
- 고객사가 위험한 요구를 할 때 거절할 수 있는 권한이 엔지니어에게 있는가?

**역질문이 전혀 없거나 급여·휴가만 묻는다면** 운영 조직 경험이 얕을 가능성이 있습니다.
