# Apache Airflow 완전 정복: 개념부터 Local K8s 실습까지

> **대상 독자**: MLOps / LLMOps / Cloud Engineer를 지향하는 실무자
> **실습 환경**: NVIDIA DGX Spark (ARM64) + Lenovo ThinkStation (x86_64) 2-노드 Local Kubernetes
> **학습 시간**: 약 6~8시간 (이론 2h + 실습 4~6h)

---

## 목차

1. [Airflow란 무엇인가](#1-airflow란-무엇인가)
2. [핵심 개념 완전 정복](#2-핵심-개념-완전-정복)
3. [Airflow 아키텍처](#3-airflow-아키텍처)
4. [Executor의 종류와 선택 기준](#4-executor의-종류와-선택-기준)
5. [실습 환경 분석: DGX Spark + ThinkStation](#5-실습-환경-분석-dgx-spark--thinkstation)
6. [실습 1: 2-노드 K8s 클러스터 구성](#6-실습-1-2-노드-k8s-클러스터-구성)
7. [실습 2: Helm으로 Airflow 설치](#7-실습-2-helm으로-airflow-설치)
8. [실습 3: 첫 번째 DAG 작성](#8-실습-3-첫-번째-dag-작성)
9. [실습 4: KubernetesPodOperator로 GPU 잡 실행](#9-실습-4-kubernetespodoperator로-gpu-잡-실행)
10. [실습 5: LLM Fine-tuning 파이프라인](#10-실습-5-llm-fine-tuning-파이프라인)
11. [운영 팁과 베스트 프랙티스](#11-운영-팁과-베스트-프랙티스)
12. [트러블슈팅 가이드](#12-트러블슈팅-가이드)

---

## 1. Airflow란 무엇인가

### 1.1 한 줄 정의

> **Apache Airflow는 "워크플로우(workflow)를 코드로 작성, 스케줄링, 모니터링하는 오픈소스 플랫폼"입니다.**

### 1.2 왜 Airflow가 필요한가?

여러분이 매일 아침 9시에 다음 작업을 해야 한다고 가정해 봅시다.

1. S3에서 어제 생성된 학습 데이터를 다운로드
2. 데이터 전처리 (결측치 처리, 토크나이징)
3. GPU 노드에서 모델 Fine-tuning
4. 검증 데이터로 평가
5. 평가 점수가 임계값 이상이면 모델 레지스트리에 등록
6. Slack으로 결과 알림

이걸 `cron`과 bash 스크립트로 묶으면 어떻게 될까요?

- **3번 단계가 실패하면?** → 4, 5, 6번이 무의미하게 실행됨
- **2번 단계 결과를 3번에 어떻게 전달하지?** → 임시 파일과 환경변수의 지옥
- **어제 어떤 작업이 성공했고 실패했는지 어떻게 알지?** → 로그를 일일이 확인
- **재시도는?** → 직접 구현해야 함
- **실행 이력 분석은?** → 불가능

Airflow는 이 모든 문제를 해결합니다.

### 1.3 Airflow가 잘하는 일 vs 못하는 일

| 잘하는 일 | 못하는 일 |
|---|---|
| 배치 워크플로우 오케스트레이션 | 실시간 스트리밍 처리 |
| 의존성이 있는 작업의 순차 실행 | 마이크로초 단위 응답 |
| 스케줄링 + 백필(backfill) | 데이터 자체의 처리 (Spark/Flink가 담당) |
| 작업 모니터링 / 재시도 / 알림 | 사용자 트리거 API 서버 |
| ML 파이프라인 (학습/배포) | 모델 서빙 (KServe/Triton이 담당) |

> **중요한 멘탈 모델**: Airflow는 **오케스트레이터(Orchestrator)** 입니다. 데이터를 직접 처리하기보다, "누가 무엇을 언제 할지" 지휘하는 지휘자입니다.

---

## 2. 핵심 개념 완전 정복

Airflow를 처음 배우는 사람이 가장 헷갈려하는 7가지 개념을 차근차근 짚어봅니다.

### 2.1 DAG (Directed Acyclic Graph)

**방향이 있고 순환하지 않는 그래프**라는 뜻입니다. 어렵게 들리지만 그림으로 보면 간단합니다.

```
[데이터 다운로드] ──▶ [전처리] ──▶ [학습] ──▶ [평가] ──▶ [배포]
                                            └──▶ [Slack 알림]
```

- **방향이 있음(Directed)**: 화살표 방향대로 실행
- **순환하지 않음(Acyclic)**: A → B → A 같은 무한루프 불가

DAG는 **파이썬 파일** 하나로 정의합니다. 가장 간단한 예:

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

with DAG(
    dag_id="hello_world",
    start_date=datetime(2026, 1, 1),
    schedule="@daily",
    catchup=False,
) as dag:
    task1 = BashOperator(task_id="say_hello", bash_command="echo 'Hello'")
    task2 = BashOperator(task_id="say_world", bash_command="echo 'World'")
    
    task1 >> task2  # task1 실행 후 task2 실행
```

### 2.2 Task와 Operator의 차이

이 둘은 자주 혼용되지만 엄밀히 다릅니다.

- **Operator (오퍼레이터)**: "무엇을 할지"에 대한 **템플릿/클래스**. 예: `BashOperator`, `PythonOperator`, `KubernetesPodOperator`
- **Task (태스크)**: Operator를 인스턴스화한 **DAG 안의 실제 노드**. 즉, "구체적 작업 한 개".

```
Operator (설계도) ─── 인스턴스화 ──▶ Task (실제 작업)
```

비유하자면, `BashOperator`는 "오븐"이고, `BashOperator(task_id="bake_cake", bash_command="...")`는 "오늘 구울 케이크 한 개"입니다.

### 2.3 Task Instance와 DAG Run

- **DAG Run**: 특정 시점에 DAG가 실행된 한 회차. 예: "2026-05-04 09:00에 실행된 fine_tuning_dag"
- **Task Instance**: 그 DAG Run 안에서 개별 Task의 실행 인스턴스. 상태(state)를 가집니다.

상태(State)의 종류:
- `queued`: 실행 대기열
- `running`: 실행 중
- `success`: 성공
- `failed`: 실패
- `skipped`: 건너뜀
- `up_for_retry`: 재시도 대기
- `upstream_failed`: 상위 태스크 실패로 인해 실행 안 됨

### 2.4 Scheduler와 Executor

이 둘이 Airflow의 심장입니다.

- **Scheduler**: DAG 파일을 **계속 읽으면서**, "지금 실행해야 할 Task가 무엇인가?"를 판단해서 실행 큐(queue)에 넣는 역할.
- **Executor**: Scheduler가 큐에 넣은 Task를 **실제로 어디서 어떻게 실행할지** 결정하는 엔진.

> **헷갈리지 않기**: Scheduler는 "결정자", Executor는 "실행자". 둘은 한 몸이 아니고 분리된 컴포넌트입니다.

### 2.5 XCom (Cross-Communication)

태스크 간에 작은 데이터를 주고받는 메커니즘입니다.

```python
def extract():
    return {"user_count": 1234}  # 자동으로 XCom에 저장됨

def transform(**context):
    data = context["ti"].xcom_pull(task_ids="extract")
    print(data["user_count"])  # 1234
```

> **주의**: XCom은 메타데이터 DB에 저장되므로 **큰 데이터 전송에 부적합**합니다. 큰 데이터는 S3/MinIO 등 외부 스토리지에 저장하고, XCom으로는 **경로(path)만** 전달하세요.

### 2.6 Connection과 Variable

- **Connection**: 외부 시스템 접속 정보 (DB, S3, K8s API 등). UI나 환경변수로 등록.
- **Variable**: 전역 설정 값 (key-value). 환경별 분기, 임계치 등에 사용.

```python
from airflow.models import Variable
threshold = Variable.get("model_accuracy_threshold", default_var=0.85)
```

### 2.7 Hook과 Sensor

- **Hook**: 외부 시스템과의 **저수준 인터페이스**. Operator 내부에서 사용.
- **Sensor**: 특정 조건이 **충족될 때까지 기다리는** 특수한 Operator. 예: "S3에 파일이 도착할 때까지 대기".

```python
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

wait_for_data = S3KeySensor(
    task_id="wait_for_data",
    bucket_key="s3://my-bucket/today/data.parquet",
    poke_interval=60,  # 60초마다 체크
    timeout=3600,
)
```

---

## 3. Airflow 아키텍처

### 3.1 컴포넌트 구성도

```
        ┌─────────────────────────────────────────────┐
        │              Web Server (UI)                │
        │  ─ DAG 시각화, 로그 확인, 수동 트리거         │
        └─────────────┬───────────────────────────────┘
                      │
                      ▼
        ┌─────────────────────────────────────────────┐
        │         Metadata Database (Postgres)        │
        │  ─ DAG/Task 상태, XCom, Connection 등 저장   │
        └─────┬───────────────────────────┬───────────┘
              │                           │
              ▼                           ▼
        ┌──────────┐               ┌──────────────┐
        │Scheduler │──── 큐잉 ───▶  │   Executor   │
        │          │               │              │
        └──────────┘               └──────┬───────┘
              │                           │
              ▼                           ▼
        ┌──────────┐               ┌──────────────┐
        │ DAG 파일 │               │   Workers    │
        │ (.py)    │               │ (실제 실행)   │
        └──────────┘               └──────────────┘
```

### 3.2 각 컴포넌트의 역할

| 컴포넌트 | 역할 | K8s 환경에서 |
|---|---|---|
| **Web Server** | UI 제공, FastAPI 기반 | Deployment + Service |
| **Scheduler** | DAG 파싱, 스케줄 결정 | Deployment (HA를 위해 2개 권장) |
| **Triggerer** | 비동기 Deferrable Task 처리 | Deployment |
| **Worker** | Task 실제 실행 | KubernetesExecutor 사용 시 Pod로 동적 생성 |
| **Metadata DB** | 상태 저장 (Postgres 권장) | StatefulSet 또는 외부 RDS |
| **DAG 저장소** | DAG 파일 보관 | PersistentVolume / Git-Sync |

---

## 4. Executor의 종류와 선택 기준

Executor 선택은 Airflow 운영의 **가장 중요한 결정**입니다.

### 4.1 주요 Executor 비교

| Executor | 동작 방식 | 적합한 환경 | 비고 |
|---|---|---|---|
| **SequentialExecutor** | 한 번에 하나씩 직렬 실행 | 로컬 테스트만 | 운영 금지 |
| **LocalExecutor** | 한 머신 안에서 멀티프로세스 | 단일 서버 소규모 | DB는 Postgres 필요 |
| **CeleryExecutor** | Redis/RabbitMQ로 워커 분산 | 전통적 분산 환경 | 운영 복잡도 높음 |
| **KubernetesExecutor** | Task마다 Pod를 동적 생성 | K8s 환경 | **본 강의 권장** |
| **CeleryKubernetesExecutor** | 하이브리드 | 대규모 혼합 환경 | 고급 |

### 4.2 왜 KubernetesExecutor를 쓸까?

1. **자원 격리**: Task별로 Pod가 생성되어 격리됨
2. **자원 효율**: 유휴 워커 없음 (필요할 때만 Pod 생성)
3. **이종 자원 활용**: GPU Task는 GPU 노드에, CPU Task는 CPU 노드에 스케줄링
4. **이미지별 실행**: Task마다 다른 컨테이너 이미지 사용 가능

> **핵심 인사이트**: 우리 환경처럼 GPU 노드(DGX Spark)와 CPU 노드(ThinkStation)가 혼재된 경우, **KubernetesExecutor가 사실상 유일한 선택**입니다.

---

## 5. 실습 환경 분석: DGX Spark + ThinkStation

### 5.1 두 노드의 특성

| 항목 | NVIDIA DGX Spark | Lenovo ThinkStation |
|---|---|---|
| **CPU 아키텍처** | ARM64 (Grace 20-core) | x86_64 (Intel/AMD) |
| **GPU** | Blackwell (1 PFLOP FP4) | 모델별 상이 (RTX/None) |
| **메모리** | 128GB Unified | 모델별 상이 |
| **OS** | DGX OS (Ubuntu 24.04 기반) | Ubuntu 권장 |
| **역할 (본 실습)** | GPU Worker / ML Compute | Control Plane + CPU Worker |

### 5.2 ⚠️ 가장 중요한 함정: 멀티 아키텍처

**DGX Spark는 ARM64, ThinkStation은 x86_64입니다.** 이는 다음을 의미합니다.

- 하나의 Docker 이미지가 두 노드에서 모두 실행되지 않을 수 있음
- 컨테이너 이미지를 빌드할 때 **멀티 아키텍처 빌드**(buildx)가 필요
- 일부 공식 이미지는 ARM64 미지원일 수 있음 (확인 필수)

### 5.3 권장 노드 역할 분담

```
┌──────────────────────────────────────┐    ┌──────────────────────────────┐
│  Lenovo ThinkStation (x86_64)        │    │  NVIDIA DGX Spark (ARM64)    │
│                                      │    │                              │
│  ─ Kubernetes Control Plane          │    │  ─ Kubernetes Worker          │
│  ─ Airflow Scheduler / Webserver     │    │  ─ GPU Task 전용 Worker       │
│  ─ Postgres (Metadata DB)            │    │  ─ NVIDIA Device Plugin       │
│  ─ MinIO (Artifact Storage)          │    │  ─ Blackwell GPU 노출         │
│  ─ CPU 기반 전처리 Task               │    │  ─ Fine-tuning / Inference   │
└──────────────────────────────────────┘    └──────────────────────────────┘
        ──────────────  Network (예: 200GbE 또는 일반 1GbE)  ──────────────
```

> **이유**: Airflow 컨트롤 플레인 컴포넌트들의 컨테이너 이미지가 x86_64 위주로 풍부합니다. GPU 작업은 Pod를 노드 셀렉터로 DGX Spark에 강제 스케줄링합니다.

---

## 6. 실습 1: 2-노드 K8s 클러스터 구성

### 6.1 사전 준비

두 노드 모두에서:

```bash
# 1. swap 비활성화 (K8s 필수)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# 2. 커널 모듈 로드
sudo modprobe br_netfilter
sudo modprobe overlay

# 3. sysctl 설정
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# 4. containerd 설치
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### 6.2 kubeadm / kubelet / kubectl 설치

```bash
# 두 노드 모두에서 (Kubernetes 1.31 기준 — 실제 환경에 맞게 수정)
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 6.3 ThinkStation에서 Control Plane 초기화

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<ThinkStation IP>

# 일반 사용자에서 kubectl 사용 설정
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# CNI 설치 (Flannel)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### 6.4 DGX Spark를 Worker로 Join

ThinkStation에서 출력된 `kubeadm join ...` 명령어를 DGX Spark에서 실행합니다.

```bash
# DGX Spark에서
sudo kubeadm join <ThinkStation IP>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### 6.5 노드 상태 확인 및 라벨링

```bash
# ThinkStation에서
kubectl get nodes -o wide

# 출력 예시:
# NAME             STATUS   ROLES           ARCH    GPU
# thinkstation     Ready    control-plane   amd64
# dgx-spark        Ready    <none>          arm64

# 노드 라벨 추가 (Pod 스케줄링에 활용)
kubectl label node dgx-spark accelerator=nvidia-blackwell
kubectl label node dgx-spark node-role/gpu=true
kubectl label node thinkstation node-role/cpu=true
```

### 6.6 NVIDIA GPU Operator 설치 (DGX Spark)

GPU를 Pod에서 사용하려면 NVIDIA Device Plugin 또는 GPU Operator가 필요합니다.

```bash
# Helm 추가
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# GPU Operator 설치
helm install --wait --generate-name \
  -n gpu-operator --create-namespace \
  nvidia/gpu-operator \
  --set driver.enabled=false  # DGX OS는 드라이버 이미 포함

# GPU 인식 확인
kubectl get nodes -o json | jq '.items[].status.capacity'
# nvidia.com/gpu: "1" 이 보여야 성공
```

---

## 7. 실습 2: Helm으로 Airflow 설치

### 7.1 네임스페이스 및 스토리지 준비

```bash
kubectl create namespace airflow

# DAG 저장용 PVC (NFS 또는 hostPath)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: airflow-dags-pv
spec:
  capacity:
    storage: 10Gi
  accessModes: [ReadWriteMany]
  hostPath:
    path: /mnt/airflow-dags  # ThinkStation의 실제 경로
  storageClassName: manual
EOF
```

### 7.2 values.yaml 작성 (핵심)

```yaml
# airflow-values.yaml
executor: "KubernetesExecutor"

# 메타데이터 DB (내장 Postgres 사용)
postgresql:
  enabled: true
  primary:
    nodeSelector:
      node-role/cpu: "true"  # ThinkStation에 배치

# Webserver / Scheduler는 ThinkStation에
webserver:
  nodeSelector:
    node-role/cpu: "true"
  service:
    type: NodePort
    nodePort: 30808

scheduler:
  nodeSelector:
    node-role/cpu: "true"

# Worker Pod는 기본적으로 어디든 가능, 단 GPU Task는 노드셀렉터로 GPU 노드에
workers:
  nodeSelector:
    node-role/cpu: "true"

# DAG는 PVC에서 로드
dags:
  persistence:
    enabled: true
    existingClaim: airflow-dags-pvc

# 멀티 아키텍처 대응: 이미지 풀 정책
images:
  airflow:
    repository: apache/airflow
    tag: 2.10.3-python3.11
    pullPolicy: IfNotPresent

# 추가 라이브러리 설치를 위한 환경변수 (선택)
env:
  - name: AIRFLOW__CORE__LOAD_EXAMPLES
    value: "False"
```

### 7.3 설치

```bash
helm repo add apache-airflow https://airflow.apache.org
helm repo update

helm install airflow apache-airflow/airflow \
  --namespace airflow \
  -f airflow-values.yaml \
  --version 1.15.0  # 실제 최신 버전 확인 필요

# 설치 확인
kubectl get pods -n airflow

# Web UI 접속
# http://<ThinkStation IP>:30808
# 기본 계정: admin / admin
```

---

## 8. 실습 3: 첫 번째 DAG 작성

### 8.1 DAG 파일 배치

`/mnt/airflow-dags/` (PVC 마운트 경로)에 파이썬 파일을 작성합니다.

```python
# /mnt/airflow-dags/hello_mlops.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator

default_args = {
    "owner": "mlops-team",
    "depends_on_past": False,
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
}

def extract_data(**context):
    """데이터 추출 (예시)"""
    print("Extracting data from source...")
    return {"row_count": 10000, "path": "/tmp/raw_data.parquet"}

def validate_data(**context):
    """데이터 검증 + XCom 사용"""
    ti = context["ti"]
    upstream = ti.xcom_pull(task_ids="extract")
    if upstream["row_count"] < 100:
        raise ValueError("데이터가 너무 적습니다!")
    return "validated"

with DAG(
    dag_id="hello_mlops",
    description="첫 번째 MLOps DAG",
    default_args=default_args,
    start_date=datetime(2026, 5, 1),
    schedule="0 9 * * *",  # 매일 09:00
    catchup=False,
    tags=["tutorial", "mlops"],
) as dag:

    start = BashOperator(
        task_id="start",
        bash_command="echo 'Pipeline 시작: $(date)'"
    )

    extract = PythonOperator(
        task_id="extract",
        python_callable=extract_data,
    )

    validate = PythonOperator(
        task_id="validate",
        python_callable=validate_data,
    )

    end = BashOperator(
        task_id="end",
        bash_command="echo 'Pipeline 완료'"
    )

    # 의존성 설정
    start >> extract >> validate >> end
```

### 8.2 DAG 인식 확인

```bash
# Scheduler가 DAG를 파싱했는지 확인 (1~2분 소요)
kubectl exec -n airflow deploy/airflow-scheduler -- airflow dags list | grep hello_mlops

# UI에서 DAG 활성화 (토글 ON) → 수동 트리거 가능
```

### 8.3 TaskFlow API (현대적 스타일)

Airflow 2.0+ 부터는 데코레이터 기반으로 더 깔끔하게 쓸 수 있습니다.

```python
from airflow.decorators import dag, task
from datetime import datetime

@dag(
    schedule="@daily",
    start_date=datetime(2026, 5, 1),
    catchup=False,
    tags=["taskflow", "modern"],
)
def modern_mlops_pipeline():

    @task
    def extract() -> dict:
        return {"row_count": 10000}

    @task
    def transform(data: dict) -> dict:
        # XCom 자동 처리됨
        data["transformed"] = True
        return data

    @task
    def load(data: dict):
        print(f"적재 완료: {data}")

    load(transform(extract()))

modern_mlops_pipeline()
```

> **Tip**: 신규 작성은 무조건 TaskFlow API. 더 짧고, 타입힌트가 자연스럽고, XCom 처리가 명시적입니다.

---

## 9. 실습 4: KubernetesPodOperator로 GPU 잡 실행

이제 본격적으로 **DGX Spark의 GPU**를 활용해 봅시다.

### 9.1 KubernetesPodOperator의 강력함

`KubernetesPodOperator`는 Task를 **임의의 컨테이너 이미지로 실행**할 수 있게 해줍니다. 즉:

- Airflow 환경에 PyTorch 설치 불필요
- GPU 노드에 직접 Pod를 띄워서 학습
- 작업 종료 후 Pod 자동 삭제

### 9.2 GPU 인식 테스트 DAG

```python
# /mnt/airflow-dags/gpu_check.py
from datetime import datetime
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.pod import KubernetesPodOperator
from kubernetes.client import models as k8s

with DAG(
    dag_id="gpu_check",
    start_date=datetime(2026, 5, 1),
    schedule=None,  # 수동 트리거 전용
    catchup=False,
    tags=["gpu", "test"],
) as dag:

    nvidia_smi = KubernetesPodOperator(
        task_id="run_nvidia_smi",
        name="nvidia-smi-check",
        namespace="airflow",
        image="nvcr.io/nvidia/cuda:12.6.0-base-ubuntu22.04",
        cmds=["nvidia-smi"],
        # 핵심 1: GPU 노드에만 스케줄
        node_selector={"accelerator": "nvidia-blackwell"},
        # 핵심 2: GPU 자원 요청
        container_resources=k8s.V1ResourceRequirements(
            limits={"nvidia.com/gpu": "1"},
        ),
        # 핵심 3: ARM64 노드이므로 ARM 이미지 필요
        # NVIDIA 공식 CUDA 이미지는 multi-arch 지원
        get_logs=True,
        is_delete_operator_pod=True,
    )
```

### 9.3 실행 후 확인

UI에서 트리거 → Task 로그 확인. 다음과 같은 출력이 보이면 성공:

```
+-----------------------------------------------------------------------+
| NVIDIA-SMI 565.xx           Driver Version: 565.xx     CUDA: 12.6     |
|-----------------------------------------------------------------------|
| GPU  Name                  | Persistence-M | Bus-Id        Disp.A     |
|   0  NVIDIA GB10 Blackwell |  ...                                     |
+-----------------------------------------------------------------------+
```

---

## 10. 실습 5: LLM Fine-tuning 파이프라인

이제 모든 것을 종합해서 **실전 LLM Fine-tuning DAG**를 작성합니다.

### 10.1 시나리오

> 매주 일요일 자정, 신규 수집된 고객 문의 데이터로 sLLM을 LoRA Fine-tuning하고, 평가 점수가 임계치를 넘으면 모델 레지스트리에 등록한다.

### 10.2 전체 DAG 구조

```
[데이터 수집] ──▶ [전처리] ──▶ [학습 데이터셋 검증]
                                       │
                                       ▼
                            [LoRA Fine-tuning (GPU)]
                                       │
                                       ▼
                            [평가 (GPU)] ──▶ [점수 체크 (Branch)]
                                                    │
                          ┌─────────────────────────┴─────────────────────────┐
                          ▼                                                   ▼
                  [모델 레지스트리 등록]                                   [실패 알림]
                          │                                                   │
                          └────────────▶ [Slack 성공 알림] ◀──────────────────┘
```

### 10.3 코드

```python
# /mnt/airflow-dags/llm_finetune_pipeline.py
from datetime import datetime, timedelta
from airflow.decorators import dag, task
from airflow.providers.cncf.kubernetes.operators.pod import KubernetesPodOperator
from airflow.operators.python import BranchPythonOperator
from airflow.operators.empty import EmptyOperator
from kubernetes.client import models as k8s

GPU_NODE_SELECTOR = {"accelerator": "nvidia-blackwell"}
GPU_RESOURCES = k8s.V1ResourceRequirements(limits={"nvidia.com/gpu": "1"})

# MinIO/S3 등 외부 스토리지 마운트 (예시)
SHARED_VOLUME = k8s.V1Volume(
    name="shared",
    persistent_volume_claim=k8s.V1PersistentVolumeClaimVolumeSource(
        claim_name="ml-shared-pvc"
    ),
)
SHARED_MOUNT = k8s.V1VolumeMount(name="shared", mount_path="/data")


@dag(
    dag_id="llm_finetune_weekly",
    description="주간 sLLM LoRA Fine-tuning 파이프라인",
    start_date=datetime(2026, 5, 1),
    schedule="0 0 * * 0",  # 매주 일요일 자정
    catchup=False,
    default_args={"retries": 1, "retry_delay": timedelta(minutes=10)},
    tags=["llm", "finetuning", "production"],
)
def llm_pipeline():

    @task
    def collect_data() -> str:
        """지난 주 신규 데이터 수집 → /data/raw/<run_id>.jsonl"""
        # 실제로는 DB 쿼리, S3 다운로드 등
        return "/data/raw/2026-05-04.jsonl"

    @task
    def preprocess(raw_path: str) -> str:
        """토크나이징, 포맷팅"""
        # 실제로는 datasets, transformers 사용
        return "/data/processed/2026-05-04.jsonl"

    finetune = KubernetesPodOperator(
        task_id="lora_finetune",
        name="lora-finetune",
        namespace="airflow",
        image="myregistry.local/llm-trainer:0.1.0-arm64",  # ARM64 빌드 필수
        cmds=["python", "/app/train.py"],
        arguments=[
            "--model", "Qwen/Qwen2.5-7B",
            "--data", "{{ ti.xcom_pull(task_ids='preprocess') }}",
            "--output", "/data/checkpoints/{{ ds }}",
            "--lora-rank", "16",
            "--epochs", "3",
        ],
        node_selector=GPU_NODE_SELECTOR,
        container_resources=GPU_RESOURCES,
        volumes=[SHARED_VOLUME],
        volume_mounts=[SHARED_MOUNT],
        get_logs=True,
        is_delete_operator_pod=True,
        # GPU Task는 길어질 수 있으므로 충분한 timeout
        execution_timeout=timedelta(hours=6),
    )

    evaluate = KubernetesPodOperator(
        task_id="evaluate",
        name="evaluate",
        namespace="airflow",
        image="myregistry.local/llm-evaluator:0.1.0-arm64",
        cmds=["python", "/app/eval.py"],
        arguments=[
            "--checkpoint", "/data/checkpoints/{{ ds }}",
            "--out", "/data/eval/{{ ds }}/score.json",
        ],
        node_selector=GPU_NODE_SELECTOR,
        container_resources=GPU_RESOURCES,
        volumes=[SHARED_VOLUME],
        volume_mounts=[SHARED_MOUNT],
        do_xcom_push=True,  # /airflow/xcom/return.json 자동 push
        get_logs=True,
        is_delete_operator_pod=True,
    )

    @task.branch
    def check_score(**context) -> str:
        result = context["ti"].xcom_pull(task_ids="evaluate")
        score = result.get("rouge_l", 0)
        threshold = 0.45
        return "register_model" if score >= threshold else "notify_failure"

    @task
    def register_model():
        # MLflow / 자체 레지스트리 등록
        print("모델 등록 완료")

    @task
    def notify_failure():
        # Slack Webhook
        print("성능 기준 미달 알림")

    notify_success = EmptyOperator(task_id="notify_success", trigger_rule="none_failed_min_one_success")

    # 의존성 정의
    raw = collect_data()
    processed = preprocess(raw)
    processed >> finetune >> evaluate
    branch = check_score()
    evaluate >> branch
    branch >> [register_model(), notify_failure()] >> notify_success


llm_pipeline()
```

### 10.4 멀티 아키텍처 이미지 빌드 (중요!)

DGX Spark가 ARM64이므로 트레이너 이미지를 ARM64로 빌드해야 합니다.

```bash
# buildx 활성화 (한 번만)
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# 이미지 빌드 + 푸시 (멀티아키)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myregistry.local/llm-trainer:0.1.0 \
  --push .
```

---

## 11. 운영 팁과 베스트 프랙티스

### 11.1 DAG 작성 원칙

1. **멱등성(Idempotency)**: 같은 입력으로 여러 번 실행해도 같은 결과
2. **원자성(Atomicity)**: 한 Task는 한 가지 일만
3. **결정론적 시간**: `datetime.now()` 대신 `{{ ds }}`, `{{ data_interval_start }}` 사용
4. **작은 XCom**: 큰 데이터는 외부 스토리지 + 경로만 XCom

### 11.2 K8s 환경 최적화

| 항목 | 권장값 / 방법 |
|---|---|
| Scheduler 개수 | 2 (HA) |
| Worker Pod 자원 요청 | 명시적으로 requests/limits 설정 |
| 이미지 풀 정책 | `IfNotPresent` (이미지 캐싱 활용) |
| DAG 동기화 | Git-Sync 사이드카 권장 (PVC보다 안정적) |
| 로그 저장소 | S3/MinIO Remote Logging 설정 |
| Metadata DB | 외부 매니지드 Postgres 권장 (운영 시) |

### 11.3 Git-Sync 설정 예시 (values.yaml)

```yaml
dags:
  gitSync:
    enabled: true
    repo: git@github.com:my-org/airflow-dags.git
    branch: main
    subPath: "dags"
    sshKeySecret: airflow-ssh-secret
    period: 60s
```

### 11.4 모니터링

- **Prometheus + Grafana**: Airflow는 statsd 메트릭 노출 → Prometheus exporter 연동
- **로그**: Remote Logging (S3/MinIO/GCS)
- **알림**: `on_failure_callback`으로 Slack/PagerDuty 연동

```python
def slack_alert(context):
    from airflow.providers.slack.hooks.slack_webhook import SlackWebhookHook
    msg = f"❌ DAG {context['dag'].dag_id} - Task {context['task_instance'].task_id} 실패"
    SlackWebhookHook(slack_webhook_conn_id="slack_default").send(text=msg)

default_args = {"on_failure_callback": slack_alert}
```

---

## 12. 트러블슈팅 가이드

### 12.1 Pod가 Pending 상태로 멈춤

```bash
kubectl describe pod <pod-name> -n airflow
```

흔한 원인:
- **노드 셀렉터 불일치**: `kubectl get nodes --show-labels`로 라벨 확인
- **GPU 부족**: `nvidia.com/gpu`가 이미 할당됨
- **PVC 미바인딩**: `kubectl get pvc -n airflow`

### 12.2 ARM64 이미지 미지원 에러

```
exec format error
```

→ 이미지가 x86_64만 지원됩니다. multi-arch 이미지로 다시 빌드하거나, 해당 Task를 ThinkStation으로 강제 스케줄(`node-role/cpu: "true"`).

### 12.3 DAG가 UI에 안 보임

체크리스트:
1. 파일이 PVC 또는 Git-Sync 경로에 정확히 있는가?
2. 파이썬 import 에러가 없는가? → `kubectl logs deploy/airflow-scheduler -n airflow | grep -i error`
3. DAG 파일에 `dag_id`가 유니크한가?
4. Scheduler가 healthy한가? → `kubectl get pods -n airflow`

### 12.4 Task가 한참 동안 queued 상태

- KubernetesExecutor가 Pod를 만들지 못하는 경우
- ServiceAccount 권한 부족: Airflow가 Pod 생성 권한이 있는지 RBAC 확인
- 클러스터 자원 부족

### 12.5 GPU 메모리 부족

DGX Spark의 128GB는 unified memory입니다. CPU와 GPU가 메모리를 공유하므로, **호스트 프로세스가 메모리를 많이 쓰면 GPU 작업도 영향**을 받습니다.

→ 학습 Pod에 메모리 limit을 명시하거나, 동시에 큰 작업이 돌지 않도록 `max_active_tasks` 제한.

---

## 마치며: 다음 학습 단계

이 강의를 완료했다면, 여러분은 다음을 할 수 있습니다.

✅ Airflow 핵심 개념 설명 가능
✅ KubernetesExecutor 기반 클러스터 운영
✅ 이종 아키텍처(ARM64+x86_64) K8s 노드 활용
✅ GPU Task를 특정 노드로 스케줄링
✅ 실전 LLM Fine-tuning 파이프라인 작성

### 다음 학습 추천 순서

1. **Airflow Providers 깊이 학습**: `airflow-providers-amazon`, `airflow-providers-google`, MLflow Provider 등
2. **MLflow + Airflow 연동**: 실험 트래킹과 모델 레지스트리 통합
3. **KServe 연동**: 학습된 모델을 K8s에서 서빙
4. **Airflow + Argo Workflows / Kubeflow Pipelines 비교**
5. **Data-aware Scheduling (Datasets)**: Airflow 2.4+ 신기능

### 참고 자료

- 공식 문서: https://airflow.apache.org/docs/
- Helm Chart: https://airflow.apache.org/docs/helm-chart/stable/index.html
- KubernetesPodOperator: https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/
- DGX Spark User Guide: https://docs.nvidia.com/dgx/dgx-spark/

> 💡 **마지막 조언**: Airflow는 "DAG를 잘 짜는 것"이 80%, "운영을 잘하는 것"이 20%입니다. 처음에는 단순한 DAG부터 시작해서, 점진적으로 복잡한 워크플로우로 확장하세요. 그리고 항상 **멱등성과 관측 가능성(observability)** 을 최우선으로 두세요.
