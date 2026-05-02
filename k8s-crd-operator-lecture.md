# Kubernetes CRD와 Operator 패턴 완벽 가이드

> **대상 독자**: Kubernetes를 처음 접하거나, 기본 리소스(Pod, Deployment, Service)는 사용해봤지만 CRD와 Operator는 처음인 분들
>
> **학습 목표**: CRD와 Operator의 개념을 이해하고, 왜 필요한지, 어떻게 활용할 수 있는지 파악하기

---

## 📚 목차

1. [들어가며: 왜 이걸 배워야 할까?](#1-들어가며-왜-이걸-배워야-할까)
2. [사전 지식 복습: Kubernetes의 핵심 동작 원리](#2-사전-지식-복습-kubernetes의-핵심-동작-원리)
3. [CRD(Custom Resource Definition)란 무엇인가?](#3-crdcustom-resource-definition란-무엇인가)
4. [Controller 패턴: Kubernetes의 심장](#4-controller-패턴-kubernetes의-심장)
5. [Operator 패턴: CRD와 Controller의 만남](#5-operator-패턴-crd와-controller의-만남)
6. [실전 예제: 나만의 CRD와 Operator 만들기](#6-실전-예제-나만의-crd와-operator-만들기)
7. [실무 활용 사례 (특히 MLOps/LLMOps)](#7-실무-활용-사례-특히-mlopsllmops)
8. [Operator 개발 도구 소개](#8-operator-개발-도구-소개)
9. [베스트 프랙티스와 주의사항](#9-베스트-프랙티스와-주의사항)
10. [다음 단계: 더 깊이 학습하기](#10-다음-단계-더-깊이-학습하기)

---

## 1. 들어가며: 왜 이걸 배워야 할까?

### 🤔 이런 경험 있으신가요?

여러분이 PostgreSQL 데이터베이스를 Kubernetes에 배포한다고 생각해 봅시다.
단순히 Pod 하나 띄우는 게 아니라, 다음과 같은 작업이 필요합니다:

- 마스터-슬레이브 복제 설정
- 백업 스케줄링
- 장애 발생 시 자동 페일오버(failover)
- 스토리지 볼륨 관리
- 비밀번호와 인증서 관리

이 모든 것을 매번 YAML로 직접 작성하면 어떻게 될까요?
**수백 줄짜리 매니페스트가 만들어지고, 운영 중 문제가 생기면 사람이 직접 개입해야 합니다.**

만약 이렇게 할 수 있다면 어떨까요?

```yaml
apiVersion: postgresql.example.com/v1
kind: PostgresCluster
metadata:
  name: my-database
spec:
  replicas: 3
  storageSize: 100Gi
  backupSchedule: "0 2 * * *"
```

단 **10줄짜리 YAML**로 클러스터 구성, 백업, 페일오버까지 모든 것이 자동화된다면?

이것이 바로 **CRD와 Operator 패턴이 해결하는 문제**입니다.

### 💡 한 줄 요약

> **CRD**는 Kubernetes에 "새로운 종류의 리소스"를 가르치는 방법이고,
> **Operator**는 그 리소스를 "어떻게 관리할지 아는 운영 전문가 봇(bot)"입니다.

---

## 2. 사전 지식 복습: Kubernetes의 핵심 동작 원리

CRD를 이해하려면 먼저 Kubernetes가 어떻게 동작하는지 핵심만 짚고 가야 합니다.

### 2.1 선언적(Declarative) 모델

Kubernetes의 가장 중요한 철학은 **"원하는 상태(Desired State)를 선언하면, Kubernetes가 알아서 그 상태를 유지한다"** 입니다.

여러분이 이런 YAML을 작성하면:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3   # ← "Pod 3개를 유지해줘"
```

Kubernetes는 "Pod를 3개 유지한다"는 **목표 상태(Desired State)** 를 기억하고, 현재 상태가 다르면 (예: Pod가 2개만 있으면) 차이를 감지해 새로 1개를 띄웁니다.

### 2.2 핵심 구성 요소

```
┌─────────────────────────────────────────────────┐
│                  Control Plane                  │
│                                                 │
│  ┌──────────────┐    ┌────────────────────┐    │
│  │  API Server  │◄──►│  etcd (저장소)     │    │
│  └──────┬───────┘    └────────────────────┘    │
│         │                                       │
│  ┌──────▼───────┐    ┌────────────────────┐    │
│  │  Scheduler   │    │  Controller Manager │   │
│  └──────────────┘    └────────────────────┘    │
└─────────────────────────────────────────────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
       ┌────────┐   ┌────────┐   ┌────────┐
       │ Node 1 │   │ Node 2 │   │ Node 3 │
       └────────┘   └────────┘   └────────┘
```

핵심 포인트:
- **API Server**: 모든 요청의 관문. YAML을 받으면 검증하고 etcd에 저장.
- **etcd**: 클러스터의 모든 상태가 저장되는 데이터베이스.
- **Controller Manager**: 여러 컨트롤러들의 집합. "감시자"이자 "조정자".

### 2.3 기본으로 제공되는 리소스 종류

Kubernetes는 기본적으로 다음과 같은 리소스를 제공합니다:

| 리소스 | 역할 |
|--------|------|
| Pod | 컨테이너의 실행 단위 |
| Deployment | Pod 관리 + 롤링 업데이트 |
| Service | 네트워크 접근 추상화 |
| ConfigMap / Secret | 설정과 비밀 정보 |
| PersistentVolume | 영구 스토리지 |
| Namespace | 논리적 격리 공간 |

**그런데...** 여러분이 운영하는 애플리케이션이 위 리소스로 표현하기 어렵다면? 🤔

예를 들어:
- 머신러닝 학습 작업 (`TrainingJob`)
- LLM 모델 서빙 (`ModelServer`)
- Kafka 클러스터 (`KafkaCluster`)
- Redis 복제 그룹 (`RedisReplicationGroup`)

이런 것들을 **Kubernetes의 일급 시민(first-class citizen)** 처럼 다루고 싶을 때 등장하는 것이 CRD입니다.

---

## 3. CRD(Custom Resource Definition)란 무엇인가?

### 3.1 비유로 이해하기

Kubernetes를 **"레고 블록 시스템"** 이라고 생각해봅시다.

- 기본 블록: Pod, Deployment, Service... (Kubernetes 제공)
- 만약 "내 프로젝트에는 우주선 블록이 필요해!" 라고 한다면?

**CRD = 레고 시스템에 "우주선이라는 새로운 블록 종류를 등록하는 설명서"** 입니다.

CRD를 등록하는 순간, Kubernetes는 "아, 이제 우주선이라는 블록도 있구나"라고 인식하게 됩니다.

### 3.2 CRD의 정확한 정의

> **Custom Resource Definition(CRD)** 은 Kubernetes API를 확장해서 새로운 종류의 리소스를 추가할 수 있게 해주는 메커니즘입니다.

CRD를 등록하면:
1. ✅ `kubectl get <my-resource>` 같은 명령어를 사용할 수 있음
2. ✅ YAML로 새 리소스를 선언할 수 있음
3. ✅ etcd에 자동으로 저장됨
4. ✅ RBAC, Webhook 등 Kubernetes의 모든 기능을 활용할 수 있음

### 3.3 CRD의 구조 살펴보기

가장 간단한 CRD 예제를 보겠습니다. "데이터베이스 백업 작업"을 표현하는 새로운 리소스를 만들어볼까요?

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 반드시 <plural>.<group> 형식이어야 함
  name: databasebackups.example.com
spec:
  group: example.com           # API 그룹
  versions:
    - name: v1
      served: true             # API로 제공할지 여부
      storage: true            # etcd에 저장할 버전인지
      schema:
        openAPIV3Schema:       # 어떤 필드를 가질 수 있는지 정의
          type: object
          properties:
            spec:
              type: object
              properties:
                databaseName:
                  type: string
                schedule:
                  type: string
                retentionDays:
                  type: integer
                  minimum: 1
                  maximum: 365
              required: ["databaseName", "schedule"]
  scope: Namespaced            # 또는 Cluster
  names:
    plural: databasebackups    # kubectl get databasebackups
    singular: databasebackup
    kind: DatabaseBackup       # YAML의 kind 필드에 사용
    shortNames:
      - dbbackup               # kubectl get dbbackup
```

### 3.4 CRD 등록 후 사용하기

위 CRD를 클러스터에 적용한 후:

```bash
$ kubectl apply -f databasebackup-crd.yaml
customresourcedefinition.apiextensions.k8s.io/databasebackups.example.com created
```

이제 새로운 리소스를 만들 수 있습니다:

```yaml
apiVersion: example.com/v1
kind: DatabaseBackup
metadata:
  name: my-daily-backup
  namespace: default
spec:
  databaseName: production-db
  schedule: "0 3 * * *"
  retentionDays: 30
```

```bash
$ kubectl apply -f my-backup.yaml
databasebackup.example.com/my-daily-backup created

$ kubectl get databasebackups
NAME              AGE
my-daily-backup   5s

$ kubectl get dbbackup           # shortName 사용 가능!
NAME              AGE
my-daily-backup   10s
```

### 3.5 ⚠️ 중요한 깨달음

> **CRD만으로는 아무 일도 일어나지 않습니다!**

위 예제에서 `DatabaseBackup` 리소스를 만들었지만, **실제로 백업이 실행되진 않습니다.**

CRD는 단지 "이런 모양의 리소스가 있을 수 있다"는 **정의**만 추가했을 뿐입니다.
누군가 이 리소스를 보고 **"실제 작업을 수행"** 해야 합니다.

그 누군가가 바로 **Controller(또는 Operator)** 입니다. 👇

---

## 4. Controller 패턴: Kubernetes의 심장

### 4.1 Controller란?

Controller는 **"특정 리소스를 감시하고, 원하는 상태(spec)와 실제 상태(status)를 비교해서 차이를 메우는 프로그램"** 입니다.

### 4.2 Reconciliation Loop (조정 루프)

Controller의 핵심 동작 방식을 **"무한 반복 비교 작업"** 이라고 부르며, 이를 **Reconciliation Loop**라고 합니다.

```
       ┌────────────────────────────────┐
       │                                │
       ▼                                │
  ┌─────────┐    ┌──────────────┐      │
  │ Watch   │───►│  Compare     │      │
  │ (감시)  │    │  desired vs  │      │
  └─────────┘    │  actual      │      │
                 └──────┬───────┘      │
                        │              │
                        ▼              │
                 ┌──────────────┐      │
                 │  Take Action │──────┘
                 │  (조치)      │
                 └──────────────┘
```

**예시: Deployment Controller의 동작**

1. **Watch**: API Server를 감시하여 Deployment 변경 이벤트를 받음
2. **Compare**: `spec.replicas: 3`이지만 현재 Pod는 2개만 실행 중
3. **Action**: 누락된 Pod 1개를 새로 생성
4. **Loop**: 다시 1번으로 돌아가 계속 감시

### 4.3 핵심 통찰: Spec과 Status

모든 Kubernetes 리소스는 두 가지 핵심 필드를 가집니다:

```yaml
spec:        # ← 사용자가 원하는 상태 (Desired State)
  replicas: 3

status:      # ← 현재 실제 상태 (Actual State, Controller가 업데이트)
  readyReplicas: 2
  availableReplicas: 2
```

> Controller의 임무: **`status`가 `spec`을 따라가도록 만드는 것**

### 4.4 비유: 온도 조절기(Thermostat)

Controller는 **온도 조절기**와 매우 비슷합니다:

| 온도 조절기 | Kubernetes Controller |
|-------------|----------------------|
| 설정 온도 22도 (`spec`) | `replicas: 3` |
| 현재 온도 측정 (`status`) | 현재 Pod 개수 확인 |
| 차이가 있으면 히터/에어컨 작동 | Pod 생성/삭제 |
| 계속 반복 | Reconciliation Loop |

---

## 5. Operator 패턴: CRD와 Controller의 만남

### 5.1 드디어 등장한 Operator

이제 모든 퍼즐 조각이 모였습니다:

> **Operator = CRD + Custom Controller**

```
┌──────────────────────────────────────────────────┐
│                                                  │
│   CRD (새 리소스 종류 정의)                      │
│      +                                           │
│   Custom Controller (그 리소스를 관리하는 로직)  │
│      =                                           │
│   Operator                                       │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 5.2 Operator라는 이름의 의미

"Operator"는 **"운영자(operator)"** 라는 영어 단어에서 왔습니다.

원래 운영자(SRE, DevOps 엔지니어)가 머릿속에 가지고 있던 **운영 노하우**를 코드로 옮겨 자동화한 것입니다.

> 사람 운영자가 PostgreSQL을 다룰 때:
> - "마스터가 죽으면 슬레이브를 승격시킨다"
> - "디스크가 80% 차면 알림을 보낸다"
> - "백업은 매일 새벽 3시에 실행한다"
>
> Operator는 **이런 노하우를 코드로 구현해 무한 반복 실행**합니다.

### 5.3 Operator의 동작 흐름 (전체 그림)

```
┌──────────────┐
│ 사용자       │
│              │
│ kubectl apply│
└──────┬───────┘
       │ (1) DatabaseBackup YAML 제출
       ▼
┌─────────────────┐
│   API Server    │
└──────┬──────────┘
       │ (2) etcd에 저장
       ▼
┌─────────────────┐
│      etcd       │
└─────────────────┘
       ▲
       │ (3) Operator가 변경 감지 (Watch)
       │
┌──────┴──────────┐
│  Custom         │  (4) Reconciliation 시작
│  Operator       │  (5) 실제 작업 수행:
│                 │      - CronJob 생성
│  (Pod로 실행)   │      - Secret 생성
│                 │      - 백업 스토리지 준비
└─────────────────┘
       │
       │ (6) status 업데이트
       ▼
┌─────────────────┐
│   API Server    │
└─────────────────┘
```

### 5.4 Operator Maturity Model (성숙도 모델)

CoreOS(현 Red Hat)는 Operator의 능력 수준을 5단계로 정의했습니다:

| Level | 이름 | 설명 |
|-------|------|------|
| 1 | Basic Install | 기본 설치 자동화 |
| 2 | Seamless Upgrades | 버전 업그레이드 자동화 |
| 3 | Full Lifecycle | 백업, 페일오버, 복구 등 전체 라이프사이클 |
| 4 | Deep Insights | 메트릭, 알림, 분석 |
| 5 | Auto Pilot | 자동 스케일링, 자동 튜닝, 자동 복구 |

**좋은 Operator일수록 사람의 개입 없이 시스템을 운영할 수 있습니다.**

---

## 6. 실전 예제: 나만의 CRD와 Operator 만들기

개념만으로는 와닿지 않으니, 가상의 시나리오를 통해 살펴봅시다.

### 6.1 시나리오: WebApp Operator 만들기

목표: `WebApp`이라는 CRD를 만들어, 한 번에 다음을 자동 생성하는 Operator 만들기
- Deployment
- Service
- Ingress
- HorizontalPodAutoscaler

### 6.2 사용자가 작성하는 YAML

```yaml
apiVersion: webapp.example.com/v1
kind: WebApp
metadata:
  name: my-blog
spec:
  image: nginx:1.21
  replicas: 2
  domain: blog.example.com
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUPercent: 70
```

### 6.3 Operator가 내부에서 자동 생성하는 리소스들

위의 단순한 YAML 하나로 Operator는 다음과 같은 리소스들을 자동으로 만들어냅니다:

```yaml
# 1. Deployment 자동 생성
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-blog
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: app
        image: nginx:1.21
---
# 2. Service 자동 생성
apiVersion: v1
kind: Service
metadata:
  name: my-blog
spec:
  selector:
    app: my-blog
  ports:
  - port: 80
---
# 3. Ingress 자동 생성
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-blog
spec:
  rules:
  - host: blog.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: my-blog
---
# 4. HPA 자동 생성
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-blog
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 6.4 Reconcile 로직 의사코드(Pseudocode)

Operator의 Controller 부분 핵심 로직을 의사코드로 표현하면:

```python
def reconcile(webapp):
    """
    WebApp 리소스가 변경될 때마다 호출됨
    """
    # 1. 현재 WebApp의 spec 읽기
    desired_image = webapp.spec.image
    desired_replicas = webapp.spec.replicas

    # 2. 관련된 Deployment가 있는지 확인
    deployment = get_deployment(webapp.name)

    if deployment is None:
        # Deployment 없으면 새로 생성
        create_deployment(webapp)
    else:
        # 있으면 spec과 일치하는지 확인하고 업데이트
        if deployment.spec.image != desired_image:
            update_deployment(deployment, desired_image)

    # 3. Service, Ingress, HPA도 동일하게 처리
    ensure_service(webapp)
    ensure_ingress(webapp)

    if webapp.spec.autoscaling.enabled:
        ensure_hpa(webapp)
    else:
        delete_hpa_if_exists(webapp)

    # 4. status 업데이트
    webapp.status.ready = check_all_resources_ready(webapp)
    update_status(webapp)

    # 5. 다음 reconcile 예약 (또는 이벤트 대기)
    return RequeueAfter(30 seconds)
```

### 6.5 Owner Reference: 자동 가비지 컬렉션

Operator가 만든 리소스들은 **Owner Reference**를 통해 원본 CR(Custom Resource)에 연결됩니다.

```yaml
# Operator가 생성한 Deployment에 자동으로 추가됨
metadata:
  name: my-blog
  ownerReferences:
  - apiVersion: webapp.example.com/v1
    kind: WebApp
    name: my-blog
    uid: abc-123-...
```

이렇게 하면:
- `WebApp`을 삭제하면 → Deployment, Service, Ingress, HPA 모두 자동 삭제
- 마치 "부모-자식" 관계처럼 동작

---

## 7. 실무 활용 사례 (특히 MLOps/LLMOps)

### 7.1 데이터베이스 Operator

- **PostgreSQL Operator** (CrunchyData, Zalando): 마스터-슬레이브 복제, 자동 페일오버, 백업
- **MySQL Operator** (Oracle): InnoDB Cluster 관리
- **MongoDB Operator**: 샤딩, 복제 셋 자동 구성
- **Redis Operator**: 클러스터 모드, 센티넬 관리

### 7.2 모니터링/관측성 Operator

- **Prometheus Operator**: `Prometheus`, `ServiceMonitor`, `AlertmanagerConfig` 같은 CRD로 모니터링 스택 선언적 관리
- **Grafana Operator**: Grafana 대시보드, 데이터소스를 CRD로 관리

### 7.3 메시징 시스템 Operator

- **Strimzi (Kafka Operator)**: `Kafka`, `KafkaTopic`, `KafkaUser` CRD 제공
- **RabbitMQ Cluster Operator**: 클러스터 자동 구성

### 7.4 🚀 MLOps/LLMOps에서의 Operator (핵심!)

**ML/AI 워크로드는 Operator의 진가가 가장 잘 드러나는 영역입니다.** 모델 학습, 분산 처리, 서빙, 자동 스케일링 등 복잡한 운영 패턴이 많기 때문입니다.

#### 🔹 Kubeflow Training Operator

다양한 분산 학습 프레임워크를 CRD로 추상화:

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pytorch-distributed-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          containers:
          - name: pytorch
            image: my-training:latest
            resources:
              limits:
                nvidia.com/gpu: 1
    Worker:
      replicas: 4
      template:
        spec:
          containers:
          - name: pytorch
            image: my-training:latest
            resources:
              limits:
                nvidia.com/gpu: 1
```

이 한 장의 YAML로:
- 마스터-워커 Pod 생성
- 분산 학습 환경 변수 자동 주입 (`MASTER_ADDR`, `WORLD_SIZE` 등)
- 학습 완료 시 자동 정리
- 실패 시 재시도 로직

지원하는 CRD: `PyTorchJob`, `TFJob`, `MPIJob`, `XGBoostJob`, `PaddleJob` 등

#### 🔹 KServe (모델 서빙 Operator)

LLM/ML 모델 서빙을 단순한 YAML로:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama-service
spec:
  predictor:
    model:
      modelFormat:
        name: huggingface
      storageUri: s3://my-bucket/llama-7b/
      resources:
        limits:
          nvidia.com/gpu: 1
    minReplicas: 1
    maxReplicas: 10
    scaleTarget: 70
```

자동으로 처리해주는 것들:
- 모델 다운로드 및 로딩
- HTTP/gRPC 엔드포인트 생성
- 트래픽 기반 오토스케일링 (Knative 활용)
- 카나리 배포, A/B 테스트
- GPU 리소스 관리

#### 🔹 Ray Operator (KubeRay)

분산 컴퓨팅과 LLM 추론을 위한 Ray 클러스터를 관리:

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  name: llm-inference-cluster
spec:
  headGroupSpec:
    template:
      spec:
        containers:
        - name: ray-head
          image: rayproject/ray:latest
  workerGroupSpecs:
  - replicas: 3
    minReplicas: 1
    maxReplicas: 10
    template:
      spec:
        containers:
        - name: ray-worker
          resources:
            limits:
              nvidia.com/gpu: 2
```

#### 🔹 Volcano (배치 스케줄링 Operator)

ML 학습 작업의 갱 스케줄링(gang scheduling), 큐잉, 우선순위를 관리:

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: distributed-training
spec:
  minAvailable: 4    # 4개 Pod이 모두 준비되어야 시작 (gang scheduling)
  schedulerName: volcano
  queue: high-priority-queue
  tasks:
  - replicas: 4
    name: trainer
    template:
      spec:
        containers:
        - name: trainer
          image: my-training:v1
          resources:
            limits:
              nvidia.com/gpu: 8
```

#### 🔹 NVIDIA GPU Operator

GPU 노드를 위한 모든 것을 자동화:
- NVIDIA 드라이버 설치
- CUDA 런타임 설정
- Device Plugin 배포
- DCGM(모니터링) 설정
- MIG(Multi-Instance GPU) 관리

#### 🔹 LLMOps 특화: 새로운 Operator들

- **vLLM Production Stack**: vLLM 서빙 자동화
- **Text Generation Inference Operator**: HuggingFace TGI 배포 관리
- **Run.ai / Anyscale Operator**: GPU 리소스 풀링과 가상화

### 7.5 클라우드 리소스 관리 Operator

- **AWS Controllers for Kubernetes (ACK)**: S3 버킷, RDS, Lambda를 K8s YAML로 관리
- **Crossplane**: 멀티 클라우드 리소스를 K8s에서 통합 관리
- **External Secrets Operator**: AWS Secrets Manager, Vault 등의 비밀을 K8s Secret으로 동기화

### 7.6 활용 사례 정리

| 영역 | 대표 Operator | 해결하는 문제 |
|------|--------------|---------------|
| 데이터베이스 | PostgreSQL Operator | 복제, 백업, 페일오버 자동화 |
| 모니터링 | Prometheus Operator | 모니터링 스택 선언적 관리 |
| 메시징 | Strimzi (Kafka) | 클러스터, 토픽, 사용자 관리 |
| ML 학습 | Kubeflow Training Op. | 분산 학습 작업 추상화 |
| 모델 서빙 | KServe | LLM/ML 서빙 자동화 |
| 배치 작업 | Volcano | 갱 스케줄링, 큐잉 |
| GPU 관리 | NVIDIA GPU Operator | GPU 노드 자동 구성 |
| 클라우드 통합 | Crossplane, ACK | 클라우드 리소스를 K8s로 관리 |

---

## 8. Operator 개발 도구 소개

직접 Operator를 만들어야 한다면, 처음부터 다 짤 필요는 없습니다. 다음 도구들이 도움을 줍니다.

### 8.1 Kubebuilder (Go 기반, 가장 표준적)

- Kubernetes SIG에서 공식 관리
- Go 언어 기반
- `controller-runtime` 라이브러리 활용
- 프로젝트 스캐폴딩, CRD 자동 생성, 테스트 프레임워크 제공

```bash
$ kubebuilder init --domain example.com --repo example.com/webapp-operator
$ kubebuilder create api --group webapp --version v1 --kind WebApp
```

### 8.2 Operator SDK (Red Hat)

- Kubebuilder 위에 추가 기능 제공
- **Go, Ansible, Helm** 세 가지 방식 지원
- Helm Operator: 기존 Helm Chart를 Operator로 변환
- Ansible Operator: Ansible Playbook으로 Operator 작성

```bash
# Helm Chart를 그대로 Operator로 만들기
$ operator-sdk init --plugins=helm --domain=example.com --group=webapp \
    --version=v1 --kind=WebApp --helm-chart=./my-chart
```

### 8.3 Kopf (Python 기반)

- Python으로 Operator를 작성하고 싶다면 추천
- Kubebuilder보다 훨씬 간단함
- 데이터 사이언스/ML팀에 친숙

```python
import kopf

@kopf.on.create('webapp.example.com', 'v1', 'webapps')
def create_fn(spec, name, namespace, **kwargs):
    # WebApp이 생성될 때 실행되는 로직
    image = spec.get('image')
    replicas = spec.get('replicas', 1)
    # ... Deployment 생성 등
    return {'message': f'{name} created'}
```

### 8.4 Metacontroller (간단한 Operator용)

- Hook 기반의 가벼운 Operator 프레임워크
- 어떤 언어로든 HTTP 핸들러만 작성하면 됨
- 복잡하지 않은 Operator에 적합

### 8.5 도구 선택 가이드

```
┌────────────────────────────────────────────────────┐
│ Q: 어떤 도구를 선택해야 할까?                      │
├────────────────────────────────────────────────────┤
│                                                    │
│  Helm Chart가 이미 있고 단순한 자동화만 원해요    │
│   → Operator SDK (Helm 모드)                       │
│                                                    │
│  Python을 좋아하고 Operator를 빠르게 만들고 싶어요│
│   → Kopf                                           │
│                                                    │
│  복잡한 로직이 필요하고 프로덕션 수준이 목표예요   │
│   → Kubebuilder 또는 Operator SDK (Go 모드)        │
│                                                    │
│  Ansible 자동화 자산을 활용하고 싶어요            │
│   → Operator SDK (Ansible 모드)                    │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## 9. 베스트 프랙티스와 주의사항

### 9.1 ✅ DO (이렇게 하세요)

#### 1) **Idempotent(멱등성)을 보장하라**

같은 reconcile을 여러 번 호출해도 결과가 동일해야 합니다.

```python
# 나쁜 예: 매번 새로 생성
def reconcile(cr):
    create_deployment(cr)  # ❌ 두 번째 호출에서 에러!

# 좋은 예: 있는지 확인 후 처리
def reconcile(cr):
    if not deployment_exists(cr.name):
        create_deployment(cr)
    else:
        update_deployment_if_needed(cr)
```

#### 2) **Status를 충실히 업데이트하라**

사용자가 `kubectl describe`로 현재 상태를 알 수 있어야 합니다.

```yaml
status:
  phase: Running
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2026-05-02T10:00:00Z"
  - type: Synced
    status: "True"
  observedGeneration: 5
```

#### 3) **Owner Reference를 반드시 설정하라**

자식 리소스에 부모 CR의 Owner Reference를 설정하면, 부모 삭제 시 자동으로 자식도 정리됩니다.

#### 4) **Finalizer를 활용하라**

리소스가 삭제되기 전에 정리 작업이 필요할 때 사용합니다.

```yaml
metadata:
  finalizers:
  - webapp.example.com/cleanup-storage
```

Operator는 finalizer가 있으면 정리 작업을 수행한 후 finalizer를 제거합니다.

#### 5) **Reconcile은 짧고 빠르게**

오래 걸리는 작업은 비동기로 처리하고, reconcile 함수는 빠르게 리턴되도록 합니다.

### 9.2 ❌ DON'T (이러면 안 됩니다)

#### 1) **Reconcile 안에서 무한 대기하지 마세요**

```python
# 나쁜 예
def reconcile(cr):
    while not all_pods_ready(cr):
        time.sleep(10)  # ❌ Operator 멈춤!
```

```python
# 좋은 예
def reconcile(cr):
    if not all_pods_ready(cr):
        return RequeueAfter(seconds=10)  # ✅ 다시 호출되도록 예약
```

#### 2) **Spec을 Operator가 변경하지 마세요**

`spec`은 사용자가 정의하는 영역, `status`만 Operator가 업데이트합니다.

#### 3) **Cluster 권한을 과도하게 요구하지 마세요**

가능하면 Namespace 범위로 권한을 제한하세요.

#### 4) **CRD 버전을 함부로 변경하지 마세요**

`v1alpha1` → `v1` 업그레이드 시 마이그레이션 전략(conversion webhook 등)이 필요합니다.

### 9.3 ⚠️ 알아두면 좋은 함정들

| 함정 | 설명 | 해결책 |
|------|------|--------|
| Race Condition | 여러 reconcile이 동시에 실행 | Workqueue가 자동 처리 (controller-runtime 사용 시) |
| Ghost Resource | CR은 삭제됐는데 자식 리소스가 남음 | Owner Reference + Finalizer |
| Reconcile Storm | 변경 한 번에 reconcile 폭주 | Debounce, RateLimiter 활용 |
| 메모리 누수 | 캐시가 너무 커짐 | Field Selector, Label Selector로 필터링 |

---

## 10. 다음 단계: 더 깊이 학습하기

### 10.1 학습 로드맵

```
[1단계: 사용자] 
  └─ 기존 Operator (Prometheus, KServe 등) 사용해보기
       │
       ▼
[2단계: 분석가]
  └─ 오픈소스 Operator의 코드 읽어보기
       │
       ▼
[3단계: 개발자]
  └─ Kubebuilder/Kopf로 간단한 Operator 직접 작성
       │
       ▼
[4단계: 전문가]
  └─ 프로덕션급 Operator 설계 (HA, 업그레이드 전략, 멀티 테넌시)
```

### 10.2 추천 학습 자료

**공식 문서**
- Kubernetes 공식 CRD 문서: kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources
- Kubebuilder Book: book.kubebuilder.io
- Operator SDK: sdk.operatorframework.io

**실습 가능한 프로젝트**
- OperatorHub.io: 다양한 Operator 카탈로그
- Kubeflow: MLOps Operator의 종합 세트
- KServe: LLM 서빙 Operator

**강력 추천 도서**
- "Programming Kubernetes" (O'Reilly) - CRD/Operator의 바이블
- "Kubernetes Operators" (O'Reilly)

### 10.3 직접 만들어볼 만한 첫 Operator 아이디어

**난이도: 쉬움**
- 자동 ConfigMap 동기화 Operator
- Ingress 자동 TLS 인증서 갱신 Operator (이미 cert-manager가 있지만 학습용)

**난이도: 중간**
- 사용자 정의 백업 작업 Operator
- 멀티 환경 (dev/staging/prod) 배포 Operator

**난이도: 어려움**
- LLM 추론 비용 최적화 Operator (트래픽에 따라 GPU 인스턴스 자동 전환)
- ML 파이프라인 Orchestration Operator

---

## 🎯 마치며: 핵심 정리

오늘 배운 내용을 한 페이지로 정리해봅시다:

### 한 줄 정의

| 개념 | 한 줄 정의 |
|------|----------|
| **CRD** | Kubernetes에 새로운 종류의 리소스를 가르치는 정의서 |
| **Custom Resource (CR)** | CRD에 의해 정의된 새로운 리소스의 인스턴스 |
| **Controller** | 리소스를 감시하고 desired state ↔ actual state를 맞추는 프로그램 |
| **Operator** | CRD + Custom Controller. 도메인 전문 지식을 코드화한 운영 자동화 |

### 핵심 통찰

1. 🔄 **Reconciliation Loop**가 모든 것의 핵심입니다. "감시 → 비교 → 조치"의 무한 반복.

2. 📦 CRD는 단지 **정의**일 뿐. **Controller가 있어야 의미**가 있습니다.

3. 🤖 Operator의 본질은 **운영 노하우의 코드화**입니다. 사람이 하던 일을 봇이 하는 것.

4. 🚀 MLOps/LLMOps에서 Operator는 **필수**입니다. 분산 학습, 모델 서빙, GPU 관리 모두 Operator로 해결됩니다.

5. 🛠️ 직접 만들 때는 **Kubebuilder, Operator SDK, Kopf** 같은 도구를 활용하세요.

### 사고 전환

이제 여러분이 Kubernetes 위에서 무언가를 운영할 때 이렇게 생각해보세요:

> **"내가 매번 손으로 하는 이 작업, Operator로 만들 수 있을까?"**

이 질문이 여러분을 진정한 클라우드 엔지니어로 성장시킬 것입니다. 🚀

---

**🎓 학습 완료!**

이 강의를 통해 CRD와 Operator의 기초를 다지셨다면, 다음 단계로 실제 Operator를 직접 만들어보거나, 운영 중인 클러스터의 Operator들을 분석해보세요. 직접 만져보는 것이 가장 빠른 학습입니다.

**Happy Operating! ⚙️**
