# Langfuse 실전 강의: 설치부터 오픈소스 LLM 관찰까지

> **대상 환경**
> - **노드 1**: Lenovo ThinkStation Workstation (x86_64) — 컨트롤 플레인 + CPU 워크로드
> - **노드 2**: NVIDIA DGX Spark (ARM64/aarch64, GB10 Grace Blackwell, 128GB 통합 메모리) — GPU 워크로드 (vLLM 등)
> - **K8s**: 로컬 2노드 클러스터
>
> **이 강의에서 다루는 것**
> 1. Langfuse가 무엇이고, 왜 LLM 관찰성(Observability)이 필요한가
> 2. Langfuse v3 아키텍처 완전 분해
> 3. Helm 차트 기반 설치 (이기종 아키텍처 고려)
> 4. 첫 트레이스 송신 테스트
> 5. DGX Spark에 vLLM(Qwen2.5-7B-Instruct) 배포 후 Langfuse로 관찰
> 6. LangChain · 프롬프트 관리 · 데이터셋 평가 (LLM-as-a-Judge)
> 7. 운영 시 트러블슈팅 가이드

---

## 제1장. Langfuse란 무엇인가 — LLM Observability 기초

### 1.1 왜 "관찰성"이 필요한가

LLM 애플리케이션은 전통적인 소프트웨어와 본질적으로 다릅니다.

| 구분 | 전통적 소프트웨어 | LLM 애플리케이션 |
|---|---|---|
| 동작 결정성 | 같은 입력 → 같은 출력 (deterministic) | 같은 입력 → 다른 출력 (stochastic) |
| 디버깅 단위 | 함수, 스택 트레이스 | 프롬프트, 토큰, 컨텍스트, 모델 응답 |
| 성능 지표 | latency, RPS, error rate | + token usage, cost, quality score, hallucination rate |
| 버그의 형태 | 예외, 잘못된 값 | "그럴듯하지만 틀린 답", 환각, 형식 오류 |

LLM 시스템은 **"왜 이런 답을 했는가?"** 를 추적하지 못하면 디버깅이 사실상 불가능합니다.
사용자가 "어제는 정상이었는데 오늘은 이상해요"라고 했을 때, 우리는 다음 모두를 알아야 합니다.

- 실제로 모델에 어떤 **프롬프트**가 들어갔는가? (시스템 + 유저 + 컨텍스트)
- RAG라면 어떤 **문서**가 검색되어 들어갔는가?
- 에이전트라면 어떤 **도구**가 어떤 순서로 호출되었는가?
- 응답의 **토큰 사용량**과 **지연시간**, **비용**은?
- 어떤 **모델 버전**이 어떤 **온도/파라미터**로 호출되었나?

Langfuse는 이 모든 것을 시간순/계층순으로 기록하고 분석하는 **LLM 전용 관찰성 플랫폼**입니다.

### 1.2 핵심 개념 모델 — Trace · Observation · Score

Langfuse의 데이터 모델을 이해하면 나머지가 쉽습니다.

```
Trace (사용자 1회 요청 = 전체 흐름)
 └── Observation (트레이스 안의 단위 행위)
      ├── SPAN     : 작업 단위 (예: "벡터 검색", "프롬프트 빌드")
      ├── GENERATION : LLM 호출 (입력/출력, 토큰, 비용)
      └── EVENT    : 시점 이벤트 (로그성)
 └── Score (트레이스나 옵저베이션에 붙는 평가 점수)
      └── 종류: 사용자 피드백, LLM-as-a-Judge, 휴먼 어노테이션, 룰 기반
```

- **Trace**: 사용자가 "안녕"이라고 입력한 후 답이 반환될 때까지의 *모든 내부 활동*을 묶은 한 묶음.
- **Observation**: 트레이스 내부의 한 단계. 트리 구조로 중첩 가능. 즉, LangChain의 "체인 안의 체인" 같은 구조도 자연스럽게 표현됩니다.
- **GENERATION**은 특수한 Observation으로, LLM 호출에 특화된 필드(`model`, `usage`, `cost`, `prompt`, `completion`)를 가집니다.
- **Score**는 *나중에 붙는* 평가입니다. 즉, 트레이스가 끝난 뒤에 사용자가 👍/👎를 누르거나, LLM-as-a-Judge가 비동기로 평가한 결과를 같은 트레이스에 결합할 수 있습니다.

### 1.3 다른 도구와의 비교 (간단히)

| 도구 | 라이선스 | 설치 형태 | 특징 |
|---|---|---|---|
| **Langfuse** | OSS (MIT, 일부 EE) | Self-host 가능 (K8s/Docker) | 본격적인 self-host, 프롬프트 관리, 데이터셋, 평가까지 통합 |
| LangSmith | 상용 | 클라우드 (self-host는 엔터프라이즈 한정) | LangChain 친화적 |
| Arize Phoenix | OSS | Self-host 가능 | OpenTelemetry 기반, eval 강점 |
| Helicone | OSS/상용 | Self-host 가능 | 프록시 기반 |

KT 사내망/온프레미스 환경에서 **데이터 주권**을 유지하면서 LangSmith급 기능을 쓰고 싶을 때 Langfuse가 가장 자연스러운 선택입니다.

---

## 제2장. Langfuse v3 아키텍처 완전 분해

Langfuse v3는 v2와 비교해 *구조가 완전히 달라졌습니다*. 단순한 Postgres 단일 DB 구조에서, **다중 데이터스토어 + 비동기 큐** 기반으로 진화했습니다.

### 2.1 컴포넌트 개요

```
                ┌──────────────────────────────────────┐
                │           Langfuse 클라이언트           │
                │ (Python SDK / JS SDK / OpenTelemetry) │
                └──────────────┬───────────────────────┘
                               │ HTTPS (POST /api/public/ingestion)
                               ▼
        ┌──────────────────────────────────────────────┐
        │            langfuse-web (Next.js)             │
        │  - UI 서빙                                    │
        │  - Public API                                 │
        │  - 이벤트를 즉시 S3에 저장 + Redis 큐에 참조 등록  │
        └─────┬─────────────────┬──────────────────┬───┘
              │                 │                  │
              ▼                 ▼                  ▼
         ┌─────────┐      ┌──────────┐       ┌──────────┐
         │ Postgres│      │  Redis   │       │ S3/MinIO │
         │ (메타,  │      │ (큐,캐시) │      │(원본 이벤트│
         │ 인증,프 │      │          │       │대용량 페이│
         │ 로젝트) │      │          │       │ 로드 저장)│
         └─────────┘      └────┬─────┘       └────┬─────┘
                               │                  │
                               ▼                  ▼
                   ┌─────────────────────────────────────┐
                   │      langfuse-worker (Node.js)       │
                   │  - 큐에서 작업을 꺼내                  │
                   │  - S3에서 원본 이벤트 로드              │
                   │  - 파싱/토크나이즈/병합 후 ClickHouse 적재│
                   │  - 비용 계산, eval 실행                │
                   └─────────────┬───────────────────────┘
                                 │
                                 ▼
                          ┌──────────────┐
                          │  ClickHouse  │
                          │ (OLAP: 트레  │
                          │  이스/관측/   │
                          │  스코어 저장) │
                          └──────────────┘
```

### 2.2 각 컴포넌트의 역할과 리소스 특성

| 컴포넌트 | 역할 | 아키텍처 | CPU/Mem | 디스크 | 노드 배치 권장 |
|---|---|---|---|---|---|
| **langfuse-web** | UI + Public API. SDK 이벤트 수신 후 S3+Redis로 빠르게 전달 | x86, ARM 모두 OK | 중 | 거의 없음 | ThinkStation |
| **langfuse-worker** | 큐 소비, 파싱, ClickHouse 적재, eval 실행, 토크나이저 사용 (CPU 집약) | x86, ARM 모두 OK | **중~높음** | 거의 없음 | ThinkStation |
| **PostgreSQL** | 메타데이터: 유저, 프로젝트, API 키, 프롬프트, 데이터셋 정의 | 멀티아키 | 낮음 | 작음 (수 GB) | ThinkStation |
| **ClickHouse** | OLAP: 트레이스/옵저베이션/스코어 (수십억 행 가능) | x86 권장, ARM도 가능 | 중~높음 | **크다** (관찰 데이터의 90%) | ThinkStation |
| **Redis/Valkey** | 큐(BullMQ) + 캐시 (`maxmemory-policy=noeviction` **필수**) | 멀티아키 | 낮음 | 작음 | ThinkStation |
| **S3/MinIO** | 원본 이벤트 페이로드, 멀티모달 입력 저장 | 멀티아키 | 낮음 | **크다** | ThinkStation |
| (선택) Zookeeper/Keeper | ClickHouse 클러스터 조율 (단일 노드면 생략) | 멀티아키 | 낮음 | 작음 | ThinkStation |

> **DGX Spark는 왜 안 쓰는가?**
> Langfuse 자체는 **GPU를 사용하지 않습니다**. DGX Spark의 128GB 통합 메모리와 GB10 GPU는 vLLM, LoRA 학습, 임베딩 같은 **워크로드 측**에 써야 합니다. 즉, **DGX Spark = "관찰 대상"**, **ThinkStation = "관찰자"** 라는 구도가 가장 자연스럽습니다.

### 2.3 데이터 흐름 — 한 번의 트레이스가 어떻게 흘러가는가

1. **클라이언트(Python SDK)** 가 `langfuse.trace(...)`, `langfuse.generation(...)`을 호출하면, 내부 버퍼에 쌓이고 백그라운드로 일괄 전송(batched POST).
2. **langfuse-web**이 `/api/public/ingestion`으로 받음. 페이로드를 **S3(MinIO)에 즉시 저장**, **참조만 Redis 큐에 enqueue**.
   - 이렇게 함으로써 *순간적인 트래픽 폭발에도 API가 응답을 빨리 돌려줄 수 있음*.
3. **langfuse-worker**가 Redis 큐를 polling. job을 받아 S3에서 원본을 로드 → 파싱/토크나이즈/병합 → ClickHouse에 upsert.
4. UI에서 조회할 때는 langfuse-web이 **ClickHouse에 직접 쿼리** + **Postgres에서 메타데이터 조회**.

이 비동기 구조 덕분에 v3는 v2 대비 *수십 배 높은 ingestion 처리량*을 갖습니다.

### 2.4 이기종 아키텍처(amd64 + arm64) 고려사항

다행스럽게도 Langfuse의 주요 컴포넌트는 **모두 멀티아키 이미지를 공식 제공**합니다.
- `langfuse/langfuse:3` → `linux/amd64`, `linux/arm64`
- `langfuse/langfuse-worker:3` → 동일
- `bitnami/clickhouse`, `bitnami/postgresql`, `bitnami/valkey`, `bitnami/minio` → 모두 멀티아키

따라서 *이미지 빌드를 따로 할 필요는 없지만*, **노드 배치 전략(nodeSelector / nodeAffinity)** 은 명시적으로 지정해야 합니다. 그래야:
- ClickHouse가 실수로 DGX Spark에 스케줄링되어 GPU 워크로드의 메모리/디스크와 충돌하는 일을 막을 수 있고,
- DGX Spark는 vLLM·임베딩 등 GPU 일에만 집중할 수 있습니다.

---

## 제3장. 사전 준비 — 클러스터 점검과 노드 라벨링

### 3.1 노드 상태 확인

```bash
# 노드 목록과 아키텍처/OS 확인
kubectl get nodes -o wide

# 아키텍처 라벨 확인
kubectl get nodes -L kubernetes.io/arch,kubernetes.io/hostname,node.kubernetes.io/instance-type
```

기대하는 출력:

```
NAME            STATUS   ROLES           ARCH    OS
thinkstation    Ready    control-plane   amd64   linux
dgx-spark       Ready    <none>          arm64   linux
```

### 3.2 노드 라벨링 (Langfuse용)

Langfuse는 모두 ThinkStation에 떨어뜨릴 것이므로, 일관된 라벨을 부여합니다.

```bash
# ThinkStation: 관찰성 플랫폼이 떨어질 노드
kubectl label node thinkstation \
  workload-type=observability \
  langfuse=enabled \
  --overwrite

# DGX Spark: GPU 워크로드 전용 (관찰 대상)
kubectl label node dgx-spark \
  workload-type=gpu \
  accelerator=nvidia-gb10 \
  --overwrite
```

### 3.3 DGX Spark에 GPU 워크로드만 떨어지도록 Taint (이미 적용했다면 패스)

```bash
# 일반 파드가 실수로 들어가지 못하도록 taint
kubectl taint node dgx-spark nvidia.com/gpu=true:NoSchedule --overwrite
```

이제 GPU를 명시적으로 요청하지 않는 파드는 DGX Spark에 들어가지 않습니다.

### 3.4 필수 인프라 확인

```bash
# 1. StorageClass — ClickHouse, Postgres, MinIO가 PVC를 사용함
kubectl get storageclass
# 기본 StorageClass가 있어야 함 (없다면 local-path-provisioner나 rook-ceph 등 설치 필요)

# 2. Ingress Controller (nginx 권장)
kubectl get pods -n ingress-nginx
# 없다면:
# helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace

# 3. (선택) cert-manager — TLS 자동화
kubectl get pods -n cert-manager
```

> **로컬 환경의 StorageClass 권장**: `local-path-provisioner` (Rancher) 가 가장 간단합니다.
> 설치: `kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml`

### 3.5 Helm 준비

```bash
helm version    # v3.12 이상 권장

# Langfuse 차트 레포 추가
helm repo add langfuse https://langfuse.github.io/langfuse-k8s
helm repo update
helm search repo langfuse
```

---
## 제4장. Helm으로 Langfuse 설치하기

### 4.1 시크릿 값 생성

설치 전에 보안에 필요한 랜덤 값들을 먼저 생성해 두겠습니다. *이 값들은 한 번 정하면 절대로 바꾸면 안 됩니다* (특히 `encryptionKey`는 DB에 저장된 데이터를 푸는 키이기 때문).

```bash
mkdir -p ~/langfuse && cd ~/langfuse

# 1) NextAuth 시크릿 (세션 암호화)
openssl rand -base64 32 > nextauth_secret.txt

# 2) Salt (API 키 해싱)
openssl rand -base64 32 > salt.txt

# 3) 데이터 암호화 키 — 반드시 64자 hex
openssl rand -hex 32 > encryption_key.txt

# 4) Postgres / ClickHouse / Redis / MinIO 패스워드
openssl rand -base64 24 > pg_password.txt
openssl rand -base64 24 > ch_password.txt
openssl rand -base64 24 > redis_password.txt
openssl rand -base64 24 > minio_password.txt

ls -la
```

### 4.2 네임스페이스 생성

```bash
kubectl create namespace langfuse
```

### 4.3 values.yaml 작성 — 두 노드 환경에 최적화된 버전

다음은 **이 강의의 핵심 산출물**입니다. 주석을 천천히 읽으며 한 줄 한 줄의 의미를 이해해 주세요.

```yaml
# ~/langfuse/values.yaml
# 2노드 (ThinkStation x86 + DGX Spark ARM) 환경용 Langfuse v3 values.yaml

# ─────────────────────────────────────────────
# 모든 Langfuse 컴포넌트를 ThinkStation에 고정
# ─────────────────────────────────────────────
langfuse:
  # web/worker는 멀티아키 이미지이므로 노드 선택만 잘 해주면 OK
  nodeSelector:
    workload-type: observability

  # 시크릿 값 (위에서 만든 값으로 채우기)
  nextauth:
    url: http://langfuse.local      # Ingress 호스트와 일치시킬 것
    secret:
      value: "여기에 nextauth_secret.txt 내용"
  salt:
    value: "여기에 salt.txt 내용"
  encryptionKey:
    value: "여기에 encryption_key.txt 내용 (64자 hex)"

  # 로그인 설정 — 자체 가입 비활성 + 첫 관리자 자동 생성
  additionalEnv:
    - name: LANGFUSE_INIT_ORG_NAME
      value: "KT-LLMOps"
    - name: LANGFUSE_INIT_PROJECT_NAME
      value: "default"
    - name: LANGFUSE_INIT_USER_NAME
      value: "admin"
    - name: LANGFUSE_INIT_USER_EMAIL
      value: "admin@kt.com"
    - name: LANGFUSE_INIT_USER_PASSWORD
      value: "ChangeMe!2026"
    - name: AUTH_DISABLE_SIGNUP
      value: "true"
    # 타임존 강제 — Langfuse는 모든 컴포넌트가 UTC여야 함 (공식 권고)
    - name: TZ
      value: "UTC"

  # 리소스 (워크로드 규모에 따라 조정)
  web:
    replicas: 1
    resources:
      requests: { cpu: "500m", memory: "1Gi" }
      limits:   { cpu: "2",    memory: "4Gi" }
  worker:
    replicas: 1
    resources:
      requests: { cpu: "500m", memory: "1Gi" }
      limits:   { cpu: "2",    memory: "4Gi" }

  # Ingress
  ingress:
    enabled: true
    className: "nginx"
    annotations:
      nginx.ingress.kubernetes.io/proxy-body-size: "50m"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    hosts:
      - host: langfuse.local
        paths:
          - path: /
            pathType: Prefix
      # TLS를 붙일 거면 아래 주석 해제 + cert-manager 사용
      # tls:
      #   - hosts: [ langfuse.local ]
      #     secretName: langfuse-tls

# ─────────────────────────────────────────────
# PostgreSQL  (Bitnami 차트의 sub-chart로 같이 떨어짐)
# ─────────────────────────────────────────────
postgresql:
  deploy: true
  auth:
    username: langfuse
    password: "여기에 pg_password.txt 내용"
    database: langfuse
  primary:
    nodeSelector:
      workload-type: observability
    persistence:
      enabled: true
      size: 20Gi
    resources:
      requests: { cpu: "500m", memory: "1Gi" }
      limits:   { cpu: "2",    memory: "4Gi" }
    extraEnvVars:
      - name: TZ
        value: "UTC"

# ─────────────────────────────────────────────
# ClickHouse  (관찰 데이터의 본진)
# ─────────────────────────────────────────────
clickhouse:
  deploy: true
  # Langfuse는 멀티샤드 미지원. 샤드는 1 고정.
  shards: 1
  # 로컬 2노드 환경에선 단일 레플리카로 시작 (개발/소규모용)
  replicaCount: 1
  resourcesPreset: "large"
  auth:
    username: default
    password: "여기에 ch_password.txt 내용"
  persistence:
    enabled: true
    size: 100Gi    # 트레이스가 쌓이면 빨리 커진다. 넉넉히.
  # zookeeper는 단일 레플리카 모드에선 어차피 안 띄움
  zookeeper:
    enabled: false
  # 모든 ClickHouse 파드를 ThinkStation에 고정
  nodeSelector:
    workload-type: observability
  # ClickHouse 데이터도 UTC
  defaultConfigurationOverrides: |
    <clickhouse>
      <timezone>UTC</timezone>
    </clickhouse>

# ─────────────────────────────────────────────
# Redis (실제론 Valkey 차트 사용)
# ─────────────────────────────────────────────
redis:
  deploy: true
  architecture: standalone
  auth:
    password: "여기에 redis_password.txt 내용"
  master:
    nodeSelector:
      workload-type: observability
    # Langfuse 큐 처리를 위해 반드시 noeviction
    extraFlags:
      - "--maxmemory-policy"
      - "noeviction"
    persistence:
      enabled: true
      size: 8Gi
    resources:
      requests: { cpu: "200m", memory: "512Mi" }
      limits:   { cpu: "1",    memory: "1.5Gi" }

# ─────────────────────────────────────────────
# S3 호환 스토리지 (MinIO)
# ─────────────────────────────────────────────
s3:
  deploy: true
  auth:
    rootUser: "minioadmin"
    rootPassword: "여기에 minio_password.txt 내용"
  persistence:
    enabled: true
    size: 50Gi
  resources:
    requests: { cpu: "500m", memory: "1Gi" }
    limits:   { cpu: "2",    memory: "4Gi" }
  nodeSelector:
    workload-type: observability
```

> **주의: 시크릿 값 채우기**
> 위 `여기에 ...` 부분을 실제 파일 내용으로 치환해야 합니다. 한 번에 처리하려면 아래 스크립트:
>
> ```bash
> sed -i "s|여기에 nextauth_secret.txt 내용|$(cat nextauth_secret.txt)|g" values.yaml
> sed -i "s|여기에 salt.txt 내용|$(cat salt.txt)|g" values.yaml
> sed -i "s|여기에 encryption_key.txt 내용 (64자 hex)|$(cat encryption_key.txt)|g" values.yaml
> sed -i "s|여기에 pg_password.txt 내용|$(cat pg_password.txt)|g" values.yaml
> sed -i "s|여기에 ch_password.txt 내용|$(cat ch_password.txt)|g" values.yaml
> sed -i "s|여기에 redis_password.txt 내용|$(cat redis_password.txt)|g" values.yaml
> sed -i "s|여기에 minio_password.txt 내용|$(cat minio_password.txt)|g" values.yaml
> ```

### 4.4 설치 실행

```bash
helm upgrade --install langfuse langfuse/langfuse \
  -n langfuse \
  -f ~/langfuse/values.yaml \
  --wait --timeout 15m
```

> 첫 설치는 **5~10분** 정도 걸립니다. Postgres 마이그레이션 → ClickHouse 마이그레이션이 차례로 돌기 때문에 web/worker 파드는 몇 번 재시작하는 게 정상입니다.

### 4.5 설치 상태 확인

```bash
# 파드 상태
kubectl get pods -n langfuse

# 기대 출력 (모두 Running)
# NAME                                READY   STATUS    AGE
# langfuse-web-xxx                    1/1     Running   3m
# langfuse-worker-xxx                 1/1     Running   3m
# langfuse-postgresql-0               1/1     Running   5m
# langfuse-clickhouse-shard0-0        1/1     Running   5m
# langfuse-valkey-master-0            1/1     Running   5m
# langfuse-s3-xxx                     1/1     Running   5m
```

문제가 있으면:

```bash
# 잘 못 떠 있는 파드 로그
kubectl logs -n langfuse <pod-name> --tail=100

# 모든 이벤트
kubectl get events -n langfuse --sort-by='.lastTimestamp' | tail -30
```

### 4.6 hosts 파일 또는 DNS 등록

`langfuse.local`을 ingress IP로 해석시켜야 합니다.

```bash
# Ingress 외부 IP 확인
kubectl get svc -n ingress-nginx ingress-nginx-controller

# 워크스테이션 /etc/hosts에 한 줄 추가 (예: 192.168.0.50)
echo "192.168.0.50  langfuse.local" | sudo tee -a /etc/hosts
```

브라우저에서 `http://langfuse.local` 접속 → 위 values.yaml에 넣은 `admin@kt.com` / `ChangeMe!2026` 로 로그인.

---

## 제5장. 첫 접속 — 조직, 프로젝트, API 키

설치 시 `LANGFUSE_INIT_*` 환경변수로 조직과 프로젝트, 관리자 계정까지 자동 생성됩니다.

### 5.1 API 키 발급

1. 좌측 메뉴 → **Settings** → **API Keys**
2. **Create new API keys** 클릭
3. 화면에 표시되는 값을 잘 보관 (Secret Key는 이때만 보임):

```bash
LANGFUSE_PUBLIC_KEY=pk-lf-xxxxxxxxxxxx
LANGFUSE_SECRET_KEY=sk-lf-xxxxxxxxxxxx
LANGFUSE_HOST=http://langfuse.local
```

> 운영 환경이면 이 값을 K8s Secret으로 관리하세요:
> ```bash
> kubectl create secret generic langfuse-creds -n llm-apps \
>   --from-literal=public_key=$LANGFUSE_PUBLIC_KEY \
>   --from-literal=secret_key=$LANGFUSE_SECRET_KEY \
>   --from-literal=host=$LANGFUSE_HOST
> ```

---
## 제6장. Python SDK로 첫 트레이스 보내기

### 6.1 가장 짧은 Hello World

```bash
# Python 가상환경 (워크스테이션에서)
python3 -m venv ~/venv-lf && source ~/venv-lf/bin/activate
pip install --upgrade langfuse
```

```python
# hello_langfuse.py
import os
from langfuse import Langfuse, observe

os.environ["LANGFUSE_HOST"]       = "http://langfuse.local"
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-lf-..."   # 위에서 받은 값
os.environ["LANGFUSE_SECRET_KEY"] = "sk-lf-..."

# Langfuse 클라이언트 초기화
lf = Langfuse()

# 연결 테스트 — True면 OK
print("auth_check:", lf.auth_check())

@observe()
def greet(name: str) -> str:
    return f"안녕하세요, {name}님!"

print(greet("철규"))

# 종료 전 버퍼 flush
lf.flush()
```

실행 후 Langfuse UI → **Tracing** → 새 트레이스가 보이면 성공입니다.

### 6.2 LLM 호출 자체를 추적하기 (수동 GENERATION)

@observe() 데코레이터만으로도 호출 흐름은 잡히지만, *모델 호출의 토큰/비용/지연*까지 잡으려면 **GENERATION**을 명시해야 합니다.

```python
from langfuse import Langfuse
import time, random

lf = Langfuse()

with lf.start_as_current_span(name="qa-pipeline", input={"question": "지구의 자전 주기는?"}) as span:
    # ── 1단계: 모의 RAG 검색
    with lf.start_as_current_span(name="retrieve") as ret:
        time.sleep(0.05)
        ret.update(output={"docs": ["지구는 약 23시간 56분에 한 번 자전한다..."]})

    # ── 2단계: LLM 호출 (가짜 응답)
    with lf.start_as_current_generation(
        name="answer-llm",
        model="qwen2.5-7b-instruct",
        model_parameters={"temperature": 0.2, "max_tokens": 200},
        input=[
            {"role": "system", "content": "너는 정확한 천문학 조교다."},
            {"role": "user",   "content": "지구의 자전 주기는?"},
        ],
    ) as gen:
        time.sleep(0.3)
        answer = "지구의 자전 주기(항성일)는 약 23시간 56분 4초입니다."
        gen.update(
            output=answer,
            usage_details={
                "input": 42,        # prompt tokens
                "output": 28,       # completion tokens
                "total": 70,
            },
        )
        span.update(output=answer)

lf.flush()
print("Trace written.")
```

UI에서 트레이스를 클릭하면:
- 부모 span (`qa-pipeline`) 아래에 자식 span(`retrieve`)과 generation(`answer-llm`)이 트리로 보입니다.
- generation을 클릭하면 토큰 사용량, 모델 파라미터, 지연시간이 표시됩니다.

---

## 제7장. DGX Spark에 vLLM으로 오픈소스 LLM 배포

이제 실제로 **관찰 대상**을 만들 차례입니다. DGX Spark의 GPU에 vLLM을 띄워 Qwen2.5-7B-Instruct를 서빙합니다.

### 7.1 vLLM Helm 배포용 매니페스트

vLLM은 공식 컨테이너 이미지 `vllm/vllm-openai:latest`를 제공하며, **CUDA(arm64)** 빌드가 ARM 기반 DGX Spark에서 동작합니다.

> **DGX Spark 주의**: GB10은 Blackwell 세대(아키텍처 `sm_100`)이므로, vLLM 이미지는 CUDA 12.6+ / sm_100을 포함한 최신 버전을 사용해야 합니다. 필요 시 `vllm/vllm-openai:v0.7.x`처럼 명시 태그를 고정하세요. ARM 빌드는 `vllm/vllm-openai:<tag>-aarch64` 식으로 별도 제공되는 경우가 있으니, `docker manifest inspect`로 멀티아키 여부를 확인 후 사용하시면 됩니다.

```yaml
# vllm-qwen.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: llm-apps
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache
  namespace: llm-apps
spec:
  accessModes: [ ReadWriteOnce ]
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-qwen
  namespace: llm-apps
spec:
  replicas: 1
  selector:
    matchLabels: { app: vllm-qwen }
  template:
    metadata:
      labels: { app: vllm-qwen }
    spec:
      # DGX Spark에만 떨어지도록
      nodeSelector:
        workload-type: gpu
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: vllm
          image: vllm/vllm-openai:latest    # 실 환경에선 버전 고정 권장
          args:
            - "--model"
            - "Qwen/Qwen2.5-7B-Instruct"
            - "--served-model-name"
            - "qwen2.5-7b-instruct"
            - "--host"
            - "0.0.0.0"
            - "--port"
            - "8000"
            - "--max-model-len"
            - "8192"
            - "--gpu-memory-utilization"
            - "0.85"
          ports:
            - containerPort: 8000
          env:
            - name: HF_HOME
              value: /models
            - name: HUGGING_FACE_HUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hf-token
                  key: token
                  optional: true   # 공개 모델이면 없어도 됨
          volumeMounts:
            - { name: model-cache, mountPath: /models }
            - { name: shm,         mountPath: /dev/shm }
          resources:
            limits:
              nvidia.com/gpu: "1"
      volumes:
        - name: model-cache
          persistentVolumeClaim: { claimName: model-cache }
        - name: shm
          emptyDir:
            medium: Memory
            sizeLimit: 8Gi
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-qwen
  namespace: llm-apps
spec:
  selector: { app: vllm-qwen }
  ports:
    - name: http
      port: 80
      targetPort: 8000
```

배포:

```bash
kubectl apply -f vllm-qwen.yaml

# 다운로드 및 워밍업까지 시간이 좀 걸림 — 로그 확인
kubectl logs -n llm-apps -l app=vllm-qwen -f
# 'Uvicorn running on http://0.0.0.0:8000' 가 나오면 OK
```

### 7.2 동작 확인

```bash
# 포트포워딩
kubectl port-forward -n llm-apps svc/vllm-qwen 8000:80 &

# OpenAI 호환 API 호출
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5-7b-instruct",
    "messages": [{"role":"user","content":"안녕? 간단히 자기소개 해줘"}],
    "max_tokens": 128
  }'
```

응답이 오면 모델 서빙 성공.

---

## 제8장. Langfuse × vLLM 연동 — 진짜 LLM 호출 추적하기

### 8.1 방법 A — OpenAI SDK 래퍼 (가장 간단)

Langfuse는 `openai` 패키지를 *드롭인 교체*해 주는 래퍼를 제공합니다. **한 줄 import만 바꾸면** 모든 호출이 자동 트레이싱됩니다.

```bash
pip install langfuse openai
```

```python
# vllm_with_langfuse.py
import os
# Langfuse가 OpenAI SDK를 monkey-patch — 이 한 줄이 핵심!
from langfuse.openai import openai

os.environ["LANGFUSE_HOST"]       = "http://langfuse.local"
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-lf-..."
os.environ["LANGFUSE_SECRET_KEY"] = "sk-lf-..."

# vLLM의 OpenAI 호환 엔드포인트로 연결
client = openai.OpenAI(
    base_url="http://vllm-qwen.llm-apps.svc.cluster.local/v1",  # 클러스터 내부면 이걸로
    # base_url="http://localhost:8000/v1",                       # 포트포워딩이면 이걸로
    api_key="EMPTY",  # vLLM은 키 검증을 하지 않음 (기본 설정)
)

resp = client.chat.completions.create(
    model="qwen2.5-7b-instruct",
    messages=[
        {"role": "system", "content": "너는 KT의 LLMOps 어시스턴트다. 간결히 한국어로 답해."},
        {"role": "user",   "content": "Kubernetes에서 Helm 차트의 values.yaml은 어떤 역할이지?"},
    ],
    temperature=0.3,
    max_tokens=300,
    # Langfuse용 메타데이터를 그대로 끼워넣기
    name="qa-helm-question",                       # 트레이스 이름
    metadata={"environment": "lab", "user": "cheolkyu"},
    tags=["qna", "lab-test"],
)
print(resp.choices[0].message.content)

# 종료 시 flush
from langfuse import Langfuse
Langfuse().flush()
```

실행하면 Langfuse UI **Tracing** 메뉴에 `qa-helm-question`이 뜨고, 입력 메시지, 출력, 토큰 사용량, 지연 시간이 자동으로 잡힙니다.

### 8.2 방법 B — 함수 전체 추적 (`@observe`) + 모델 호출

비즈니스 로직이 더 복잡해지면, 함수 자체를 트레이스로 묶고 그 안에서 LLM을 호출하는 식이 깔끔합니다.

```python
from langfuse import observe
from langfuse.openai import openai
from langfuse import Langfuse

client = openai.OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

@observe()
def summarize(text: str) -> str:
    """긴 문서를 3줄 요약."""
    resp = client.chat.completions.create(
        model="qwen2.5-7b-instruct",
        messages=[
            {"role": "system", "content": "다음 글을 한국어로 정확히 3줄로 요약하라."},
            {"role": "user",   "content": text},
        ],
        temperature=0.2,
    )
    return resp.choices[0].message.content

@observe()
def pipeline(text: str) -> dict:
    summary = summarize(text)
    return {"length": len(text), "summary": summary}

print(pipeline(open("some_long_doc.txt").read()))
Langfuse().flush()
```

UI에서 `pipeline → summarize → openai.chat.completions` 의 호출 트리가 그대로 보입니다.

---

## 제9장. LangChain 통합

KT에서 LangChain을 쓰는 경우, CallbackHandler 하나만 끼우면 됩니다.

```bash
pip install langchain langchain-openai langfuse
```

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langfuse.langchain import CallbackHandler

# 콜백 핸들러 — 같은 env vars를 자동 인식
lf_handler = CallbackHandler()

llm = ChatOpenAI(
    model="qwen2.5-7b-instruct",
    openai_api_base="http://localhost:8000/v1",
    openai_api_key="EMPTY",
    temperature=0.3,
)

prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 친절한 한국어 비서다."),
    ("user",   "{question}"),
])

chain = prompt | llm

result = chain.invoke(
    {"question": "DGX Spark의 GB10 칩 특징을 3줄로 설명해줘."},
    config={"callbacks": [lf_handler]},
)
print(result.content)
```

UI에는 `ChatPromptTemplate → ChatOpenAI` 의 체인 구조가 그대로 표시됩니다.

---

## 제10장. 프롬프트 관리 (Prompt Management)

프롬프트가 *코드에 하드코딩되어 있으면* 변경 시마다 배포가 필요합니다. Langfuse의 Prompt Management는 프롬프트를 **외부화**하고 버전을 관리합니다.

### 10.1 UI에서 프롬프트 생성

1. 좌측 메뉴 → **Prompts** → **Create**
2. 이름: `qa/system-prompt`
3. Type: **Chat**
4. 내용:
   ```
   [system]
   너는 {{persona}}다. 답변은 반드시 한국어, 3줄 이내로 한다.
   [user]
   {{question}}
   ```
5. **Save**. v1로 저장됨.
6. 운영에 올릴 준비가 되면 라벨 **production**을 부여.

### 10.2 코드에서 가져와 쓰기

```python
from langfuse import Langfuse
from langfuse.openai import openai

lf = Langfuse()
client = openai.OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

# production 라벨이 붙은 최신 버전을 가져옴
prompt = lf.get_prompt("qa/system-prompt", label="production")

# 변수 바인딩 — Chat 프롬프트는 list[dict]가 반환됨
messages = prompt.compile(
    persona="KT의 LLMOps 어시스턴트",
    question="Argo Workflows의 DAG와 Steps 차이?",
)

resp = client.chat.completions.create(
    model="qwen2.5-7b-instruct",
    messages=messages,
    # 어떤 프롬프트 버전이 쓰였는지 트레이스에 연결
    langfuse_prompt=prompt,
)
print(resp.choices[0].message.content)
lf.flush()
```

이렇게 하면 트레이스 페이지에서 **어떤 프롬프트의 어떤 버전이 어떤 응답을 만들었는지** 가 영구 기록됩니다. *A/B 테스트, 회귀 추적이 가능해지는 출발점.*

---
## 제11장. 데이터셋과 평가 — 회귀 방지의 핵심

LLM 시스템은 **"고치는 순간 다른 곳이 망가지는"** 문제가 흔합니다. Langfuse의 Datasets + Experiments는 이를 막아줍니다.

### 11.1 데이터셋 만들기

UI → **Datasets** → **New dataset** → 이름: `qa-eval-v1`

```python
# Python SDK로 일괄 등록도 가능
from langfuse import Langfuse
lf = Langfuse()

dataset_items = [
    {
        "input":           {"question": "지구의 자전 주기는?"},
        "expected_output": "약 23시간 56분 4초 (항성일)",
    },
    {
        "input":           {"question": "vLLM의 PagedAttention이 뭐야?"},
        "expected_output": "KV cache를 메모리 페이지처럼 나눠 관리해 메모리 낭비를 줄이는 기법",
    },
    {
        "input":           {"question": "Kubernetes Helm 차트의 values.yaml은?"},
        "expected_output": "차트의 기본 설정값을 사용자 환경에 맞게 오버라이드하기 위한 파일",
    },
]

# 데이터셋 생성 (이미 있으면 skip)
lf.create_dataset(name="qa-eval-v1", description="기본 Q&A 회귀 테스트")

for item in dataset_items:
    lf.create_dataset_item(
        dataset_name="qa-eval-v1",
        input=item["input"],
        expected_output=item["expected_output"],
    )
```

### 11.2 실험(Experiment) 실행

같은 데이터셋을 *같은 코드로 여러 모델/프롬프트로* 돌려보고 비교하는 게 핵심입니다.

```python
from langfuse import Langfuse
from langfuse.openai import openai

lf = Langfuse()
client = openai.OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

def my_qa_chain(question: str) -> str:
    resp = client.chat.completions.create(
        model="qwen2.5-7b-instruct",
        messages=[
            {"role": "system", "content": "정확한 한국어로 한 문장으로 답하라."},
            {"role": "user",   "content": question},
        ],
        temperature=0.0,
    )
    return resp.choices[0].message.content

# Run 이름 — 비교를 위해 변경 가능 (모델/프롬프트별로 구분)
RUN_NAME = "qwen2.5-7b-v1"

dataset = lf.get_dataset("qa-eval-v1")
for item in dataset.items:
    # item.run() 컨텍스트가 트레이스를 자동으로 데이터셋 실행과 연결
    with item.run(run_name=RUN_NAME) as root_trace:
        answer = my_qa_chain(item.input["question"])
        root_trace.update(output=answer)

lf.flush()
```

UI → **Datasets → qa-eval-v1 → Runs**에서 한 페이지에 모든 데이터셋 항목에 대해 어떤 답을 했는지 한눈에 비교 가능. 모델 바꿔서 `RUN_NAME="llama3-8b-v1"`로 다시 돌리면 *나란히 비교*됩니다.

### 11.3 LLM-as-a-Judge — 자동 평가

수십, 수백 개 답을 사람이 다 보긴 힘듭니다. *다른 LLM에게 채점*시키는 방법을 등록할 수 있습니다.

1. **Settings → LLM Connections** → "Local vLLM" 추가 (BaseURL: `http://vllm-qwen.llm-apps.svc.cluster.local/v1`, Model: `qwen2.5-7b-instruct`).
   - 채점은 자기보다 큰 모델로 하는 게 일반적이지만, 데모는 같은 모델로 충분.
2. **Evaluators → New** → "Correctness" 템플릿 선택 → 데이터셋 `qa-eval-v1`에 연결.
3. 다음 Experiment부터는 각 항목마다 자동으로 0~1 점수가 매겨집니다.

> 평가의 핵심은 *점수 자체가 절대적이지 않다는 것*. **변화 추세**(릴리즈 A→B에서 점수가 떨어지면 즉시 알림)를 추적하는 게 목적입니다.

### 11.4 휴먼 피드백 (Score) 추가

운영 환경에서 사용자 👍/👎를 트레이스에 결합:

```python
from langfuse import Langfuse
lf = Langfuse()

# UI에서 트레이스 클릭 → 우측 상단의 trace_id 복사
lf.create_score(
    trace_id="6f1b...d3",      # 또는 from .observation_id="..."
    name="user_feedback",
    value=1,                   # 1=좋아요, 0=싫어요
    comment="정확하고 빠름",
    data_type="NUMERIC",
)
lf.flush()
```

이렇게 모인 점수는 추후 **fine-tuning 데이터의 소스**로도 활용할 수 있습니다 (DPO/RLHF 등).

---

## 제12장. 운영 — 모니터링, 비용, 알림

### 12.1 대시보드

좌측 **Dashboards** 메뉴:
- **Traces over time**: 시간대별 트레이스 수
- **Token consumption**: 모델별 입력/출력 토큰
- **Cost**: 모델별 누적 비용 (모델 가격은 **Settings → Models**에서 직접 등록 가능 — 셀프 호스팅 모델의 단가 추정에 유용)
- **Latency p50/p95/p99**: 응답 지연 분포
- **Errors**: 실패한 트레이스

### 12.2 모델 단가 등록 (셀프호스팅 모델용)

UI → **Settings → Models → Add model**
- Match Pattern: `qwen2.5-7b-instruct` (정확 일치 또는 regex)
- Token pricing: `input: 0.0000001 USD/token`, `output: 0.0000003 USD/token` (운영비 추산용)

이렇게 하면 sefl-hosted 모델도 *명목상 비용 추적*이 가능해 부서별/프로젝트별 어카운팅에 활용됩니다.

### 12.3 K8s 레벨 모니터링과 연동

Prometheus를 이미 쓰고 있다면, Langfuse 컴포넌트들의 메트릭(요청 처리량, 큐 길이 등)도 같이 보는 게 좋습니다.

```yaml
# Langfuse values.yaml에 추가
langfuse:
  web:
    podAnnotations:
      prometheus.io/scrape: "true"
      prometheus.io/port:   "3000"
      prometheus.io/path:   "/api/public/metrics"
```

ClickHouse 자체도 system 테이블이 풍부하므로, ClickHouse Exporter를 띄워 디스크 사용량, 머지 큐 길이를 추적하시면 좋습니다.

### 12.4 데이터 보존 정책

LLM 트레이스는 *예상보다 훨씬 빠르게* 디스크를 잡아먹습니다. 입력/출력 페이로드가 크기 때문입니다.

- **S3/MinIO**: 90일 lifecycle 정책으로 오래된 원본 페이로드 제거
- **ClickHouse**: 트레이스 테이블에 TTL 설정 (UI → Project Settings → Data Retention)

> 보관 기간 정책은 *비즈니스 요구사항 + 개인정보보호 + 디스크 비용*의 삼각 균형으로 정해야 합니다. KT 보안 가이드라인이 있다면 그쪽을 우선.

---

## 제13장. 트러블슈팅 가이드

### 13.1 잘 만나는 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| `lf.auth_check()`가 False | API 키 오타 / HOST 오타 | 마지막 슬래시 제거, http vs https 확인 |
| 트레이스가 UI에 안 보임 | worker가 큐를 비우지 못함 | `kubectl logs ... langfuse-worker`, Redis 접속 확인 |
| 트레이스가 *느리게* 보임 | 정상. 워커가 비동기 처리 | 평소 1~5초 지연. 1분 이상이면 worker 스케일 업 |
| ClickHouse 파드 OOMKilled | 메모리 부족 | `resourcesPreset: xlarge` 또는 limits 상향 |
| Postgres 마이그레이션 실패 | UTC 미설정 / 권한 | `TZ=UTC` 환경변수, postgres 유저 권한 확인 |
| `encryptionKey` 관련 에러 | 64자 hex 아님 | `openssl rand -hex 32`로 정확히 재생성 |
| Redis "Cannot perform LMOVE" | maxmemory-policy가 noeviction 아님 | `redis.master.extraFlags`로 강제 |
| DGX Spark에 LF 파드가 떨어짐 | nodeSelector 빠짐 | values.yaml의 모든 컴포넌트에 `workload-type: observability` |
| vLLM 모델 다운로드 실패 | HF 토큰 / 네트워크 | `HUGGING_FACE_HUB_TOKEN` Secret 등록, egress 정책 확인 |

### 13.2 디버깅 명령 모음

```bash
# 모든 Langfuse 파드 한 화면 보기
kubectl get pods -n langfuse -o wide

# worker 로그 실시간
kubectl logs -n langfuse -l app.kubernetes.io/component=worker -f

# Redis 큐 상태 — 큐가 막혔는지 확인
kubectl exec -it -n langfuse langfuse-valkey-master-0 -- \
  valkey-cli -a "$(cat ~/langfuse/redis_password.txt)" \
  --no-auth-warning LLEN bull:ingestion-queue:wait

# ClickHouse 상태
kubectl exec -it -n langfuse langfuse-clickhouse-shard0-0 -- \
  clickhouse-client --password "$(cat ~/langfuse/ch_password.txt)" \
  --query "SELECT count() FROM langfuse.traces"

# S3(MinIO) 버킷 확인
kubectl port-forward -n langfuse svc/langfuse-s3 9001:9001
# 브라우저 http://localhost:9001 (minioadmin / 위에서 만든 password)
```

### 13.3 백업 전략 (최소)

1. **Postgres**: 매일 `pg_dump` → S3로 푸시 (CronJob).
2. **ClickHouse**: `clickhouse-backup` 또는 disk snapshot.
3. **S3/MinIO**: 또 다른 S3 버킷으로 mirror 백업 (rclone).
4. **values.yaml + 시크릿 텍스트 파일**: *반드시 별도 안전한 곳에* 보관.
   특히 `encryptionKey`가 사라지면 기존 데이터를 복호화할 수 없습니다.

---

## 제14장. 운영 베스트 프랙티스 요약

### 데이터 모델 측면
- **모든 사용자 요청을 트레이스 1개로** 묶는다. 트레이스가 너무 잘게 쪼개지면 분석이 어렵다.
- 외부 호출(LLM, 임베딩, 벡터 검색)은 **각각 별도 span/generation**으로 잡는다.
- **user_id, session_id**를 트레이스에 항상 붙인다 (sampling, debugging의 출발점).
- 민감 정보는 SDK 레벨에서 **마스킹**한다 (PII 자동 마스킹은 `langfuse` SDK의 `mask` 기능 또는 사전 후처리).

### 인프라 측면
- **ClickHouse 디스크는 빨리 찬다**. 모니터링 + TTL 정책 필수.
- Redis `maxmemory-policy=noeviction`은 *절대 양보 불가*.
- 모든 컴포넌트 **UTC** 통일.
- 단일 노드(이 강의 환경)에선 ClickHouse `replicaCount: 1`이지만, 운영 클러스터는 **3 이상** 권장.

### 보안 측면
- **NEXTAUTH_URL**과 **Ingress 호스트**는 반드시 일치. 안 그러면 OAuth/CSRF 오류.
- **AUTH_DISABLE_SIGNUP=true**로 사내 인원만 가입 허용.
- LDAP/SSO를 쓴다면 SAML 또는 OAuth (Keycloak, Azure AD) 연동 추천 — 환경변수로 설정 가능.
- 운영 트래픽이 흐르는 경로엔 반드시 **TLS** (cert-manager + Let's Encrypt 또는 사내 CA).

### MLOps 통합 측면
- **Airflow LoRA 파인튜닝 파이프라인**에서 fine-tuning 데이터 후보를 추출할 때, Langfuse Score가 높은 트레이스만 필터링해서 가져오면 자연스러운 DPO 데이터셋이 됩니다.
- **Argo Workflows 하이퍼파라미터 스윕**에서 각 실험을 Langfuse Experiment에 매핑하면 결과 비교가 직관적입니다.
- **KServe로 모델을 배포**할 때, 모델 이름을 Langfuse에 등록된 모델 패턴과 일치시키면 비용/지연 추적이 자동으로 됩니다.

---

## 부록 A. 자주 쓰는 환경변수 치트시트

| 변수 | 의미 |
|---|---|
| `LANGFUSE_HOST` | UI/API URL (예: `http://langfuse.local`) |
| `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` | 프로젝트별 API 키 |
| `LANGFUSE_TRACING_ENABLED` | 트레이싱 on/off (기본 true) |
| `LANGFUSE_FLUSH_AT` / `LANGFUSE_FLUSH_INTERVAL` | 배치 크기/주기 |
| `LANGFUSE_SAMPLE_RATE` | 0.0~1.0, 트레이스 샘플링 비율 (운영 시 트래픽 폭증 대비) |
| `LANGFUSE_DEBUG` | SDK 디버그 로그 |

## 부록 B. 디렉터리 구조 권장

```
~/langfuse/
├── values.yaml              # 메인 설정
├── nextauth_secret.txt      # 시크릿 (git 제외!)
├── salt.txt
├── encryption_key.txt
├── pg_password.txt
├── ch_password.txt
├── redis_password.txt
└── minio_password.txt

~/llm-apps/
├── vllm-qwen.yaml           # vLLM 디플로이먼트
├── hf-token-secret.yaml     # HF 토큰 (옵션)
└── samples/
    ├── hello_langfuse.py
    ├── vllm_with_langfuse.py
    ├── langchain_with_langfuse.py
    └── dataset_experiment.py
```

## 부록 C. 다음 단계 (추천 학습 경로)

1. **OpenTelemetry 연동** — Langfuse는 OTLP를 지원합니다. 기존 OTel 기반 백엔드(Jaeger 등)와 *병행* 송신하여 통합 가시성을 확보하는 패턴을 시험해 보세요.
2. **에이전트 트레이싱** — LangGraph, AutoGen 등의 에이전트 프레임워크에 Langfuse 콜백을 끼우면 **도구 호출 트리**가 그대로 시각화됩니다.
3. **사용자 단위 비용 어카운팅** — `user_id` + 모델 단가로 부서별/팀별 비용 차징.
4. **회귀 게이트(Regression Gate)** — Argo Workflows의 파이프라인 마지막에 "최근 7일 평균 score > 0.8"을 만족할 때만 production 라벨을 전환하는 자동 가드 추가.
5. **Garak/NeMo Guardrails 결과**를 Langfuse Score로 자동 푸시 — 보안/안전성 회귀 게이팅.

---

## 마무리

Langfuse는 단순한 "LLM 로그 뷰어"가 아닙니다.
**프롬프트 관리 + 평가 + 데이터셋 + 회귀 추적**까지 갖춘, *LLM 시대의 통합 관찰성 + 실험 플랫폼*입니다.

이 강의의 마지막 권고:
> *"트레이스 없는 LLM 시스템은 운영 가능한 시스템이 아니다."*

DGX Spark + ThinkStation 2노드 구성에서 시작해, 사내 트래픽이 늘면 ClickHouse 클러스터 확장 + worker replica 증설로 자연스럽게 스케일 아웃되므로, 지금 시점에 한 번 제대로 깔아두는 것이 향후 KT의 LLMOps 성숙도를 가장 빠르게 끌어올리는 길입니다.

수고하셨습니다. 🚀
