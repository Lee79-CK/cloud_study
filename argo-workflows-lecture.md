# Argo Workflows 완전 정복: 개념부터 DGX Spark + ThinkStation 이종 클러스터 실습까지

> **대상 독자**: MLOps / LLMOps / Cloud Engineer
> **실습 환경**: 2-Node Kubernetes Cluster (NVIDIA DGX Spark + Lenovo ThinkStation)
> **기준 버전**: Argo Workflows v4.0.5 (2026년 4월 기준 최신 stable), Kubernetes v1.30+
> **문서 목적**: 처음 접하는 사람도 따라올 수 있는 강의식 가이드

---

## 목차

1. [강의 개요 - 왜 Argo Workflows를 배워야 하는가](#1-강의-개요---왜-argo-workflows를-배워야-하는가)
2. [Argo Workflows의 정체성과 아키텍처](#2-argo-workflows의-정체성과-아키텍처)
3. [핵심 개념 7가지 - 이것만 알면 90%](#3-핵심-개념-7가지---이것만-알면-90)
4. [실습 환경의 특수성 - 이종(Heterogeneous) 클러스터 이해하기](#4-실습-환경의-특수성---이종heterogeneous-클러스터-이해하기)
5. [2-Node 로컬 K8s 클러스터 구축](#5-2-node-로컬-k8s-클러스터-구축)
6. [Argo Workflows 설치](#6-argo-workflows-설치)
7. [실습 1 - Hello World부터 DAG까지](#7-실습-1---hello-world부터-dag까지)
8. [실습 2 - GPU 워크로드와 노드 어피니티](#8-실습-2---gpu-워크로드와-노드-어피니티)
9. [실습 3 - 진짜 ML 파이프라인 만들기](#9-실습-3---진짜-ml-파이프라인-만들기)
10. [실습 4 - 하이퍼파라미터 스윕(LLM Fine-tuning 시나리오)](#10-실습-4---하이퍼파라미터-스윕llm-fine-tuning-시나리오)
11. [운영 베스트 프랙티스](#11-운영-베스트-프랙티스)
12. [트러블슈팅 자주 만나는 함정](#12-트러블슈팅-자주-만나는-함정)
13. [부록 - 다음 단계 학습 로드맵](#13-부록---다음-단계-학습-로드맵)

---

## 1. 강의 개요 - 왜 Argo Workflows를 배워야 하는가

### 1.1 MLOps 엔지니어가 마주치는 현실

여러분이 다음과 같은 작업을 매일 한다고 상상해 봅시다.

- 매일 새벽 3시에 데이터 레이크에서 학습 데이터를 추출
- 추출한 데이터를 전처리(클렌징, 토크나이징, 임베딩)
- 전처리된 데이터로 LLM 파인튜닝 (보통 GPU 필요)
- 학습된 모델을 평가셋으로 벤치마크
- 평가 점수가 임계값을 넘으면 자동으로 모델 레지스트리에 등록
- Slack으로 결과 알림

이걸 `cron + bash script`로 운영하면 어떻게 될까요? 한 단계만 실패해도 전체가 멈추고, 어디서 실패했는지 추적이 어렵고, GPU 자원은 놀고, 재시도 로직은 매번 직접 짜야 합니다.

**Argo Workflows는 이 모든 문제를 Kubernetes 위에서 우아하게 풀어주는 도구입니다.**

### 1.2 다른 도구와의 비교 - 한눈에 정리

| 도구 | 실행 모델 | K8s 친화도 | ML 특화 | 학습 곡선 |
|------|-----------|-----------|---------|----------|
| **Apache Airflow** | Python DAG, Worker가 작업 실행 | 중간(K8sExecutor 필요) | 보통 | 중간 |
| **Argo Workflows** | 컨테이너 기반, 각 step = Pod | 매우 높음 (CRD 네이티브) | 높음 | 낮음~중간 |
| **Kubeflow Pipelines** | Argo 위에 빌드된 ML 전용 추상화 | 높음 | 매우 높음 | 높음 |
| **Prefect / Dagster** | Python 중심 | 중간 | 중간 | 중간 |

> 💡 **포인트**: Kubeflow Pipelines는 사실상 내부적으로 Argo Workflows를 사용합니다. 즉 Argo를 알면 Kubeflow의 동작도 자연스럽게 이해됩니다.

### 1.3 이 강의에서 얻을 수 있는 것

1. Argo의 **개념 모델**을 머릿속에 정확히 그리는 것
2. ARM64 + x86_64 이종 클러스터에서 발생하는 **현실적인 함정** 회피법
3. GPU 워크로드를 **어떤 노드에 어떻게 배치하는지** 실전 패턴
4. 진짜 LLM 파이프라인을 **YAML로 표현**하는 법

---

## 2. Argo Workflows의 정체성과 아키텍처

### 2.1 한 줄 정의

> **Argo Workflows = Kubernetes 위에서 컨테이너 기반의 병렬/순차 작업을 정의하고 실행하는 워크플로우 엔진**

핵심은 두 가지입니다.

1. **컨테이너 네이티브**: 모든 작업 단계가 Pod로 실행됨 → 언어/프레임워크 독립적
2. **Kubernetes CRD**: `Workflow`라는 새로운 리소스 타입을 K8s에 추가 → `kubectl get workflows`로 조회 가능

### 2.2 아키텍처 구성요소

```
┌────────────────────────────────────────────────────────┐
│                  Kubernetes API Server                 │
│  (Workflow CRD, WorkflowTemplate CRD, CronWorkflow CRD)│
└──────────────────┬────────────────────────────┬────────┘
                   │                            │
                   ▼                            ▼
        ┌─────────────────────┐      ┌─────────────────────┐
        │ Workflow Controller │      │     Argo Server     │
        │  (실제 오케스트레이션)│      │  (UI / REST / gRPC) │
        └──────────┬──────────┘      └─────────────────────┘
                   │
                   ▼
        ┌─────────────────────┐
        │  Pod (각 step 실행)  │
        │  ┌───────────────┐  │
        │  │ main          │  │  ← 사용자 컨테이너
        │  │ wait (사이드카)│  │  ← 결과 수집/아티팩트 업로드
        │  └───────────────┘  │
        └─────────────────────┘
                   │
                   ▼
        ┌─────────────────────┐
        │ Artifact Repository │  ← MinIO / S3 / GCS
        └─────────────────────┘
```

각 컴포넌트의 역할을 풀어보면:

- **Workflow Controller**: 실제로 워크플로우 실행을 책임지는 두뇌. `Workflow` CRD를 watch하다가 새로운 게 생기면 다음 실행할 step을 결정하고 Pod를 띄움.
- **Argo Server**: UI와 API를 제공. 컨트롤러와 분리되어 있어서 서버가 죽어도 실행은 멈추지 않음.
- **Executor (사이드카)**: Pod 내부에서 main 컨테이너 옆에 붙어 있으며, 결과 캡처, 아티팩트 업로드, 시그널 처리를 담당.
- **Artifact Repository**: step 간에 큰 파일을 주고받을 때 사용하는 외부 저장소. 로컬 환경에서는 보통 MinIO를 사용.

### 2.3 워크플로우 실행 흐름

1. 사용자가 `kubectl apply -f workflow.yaml` 또는 `argo submit`으로 워크플로우 생성
2. K8s API 서버가 etcd에 `Workflow` 객체 저장
3. Workflow Controller가 객체 변경 감지
4. 의존성 그래프 분석 후 실행 가능한 step에 대한 Pod 스펙 생성
5. K8s Scheduler가 Pod를 적절한 노드에 배치
6. Pod 실행 완료 시 사이드카가 결과를 컨트롤러에 보고
7. 컨트롤러가 다음 step 진행 또는 완료 처리

---

## 3. 핵심 개념 7가지 - 이것만 알면 90%

### 3.1 Workflow

워크플로우 자체를 나타내는 **최상위 리소스**. 한 번 실행되면 사라지지 않고 K8s 안에 남아 있어서 결과를 나중에 조회할 수 있습니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-      # 실제 이름은 hello-world-abc12 처럼 자동 생성됨
spec:
  entrypoint: main                 # 시작 template 이름
  templates:
  - name: main
    container:
      image: alpine:3.19
      command: [sh, -c]
      args: ["echo 안녕 Argo!"]
```

### 3.2 Template - 작업 단위의 정의

Template은 "어떤 작업을 어떻게 실행할지" 정의하는 **레시피**입니다. 5가지 종류가 있습니다.

| 종류 | 설명 | 사용 시점 |
|------|------|----------|
| `container` | 단일 컨테이너 실행 | 가장 기본, 90% 이 형태 |
| `script` | 인라인 스크립트(bash/python) 실행 | 간단한 변환 작업 |
| `resource` | K8s 리소스(Job, ConfigMap 등) 직접 생성/관리 | KFServing 모델 배포 등 |
| `dag` | 다른 template들을 그래프로 연결 | 복잡한 의존성 |
| `steps` | 다른 template들을 순차적으로 실행 | 단순 파이프라인 |

### 3.3 Steps vs DAG - 둘 다 흐름 제어인데 뭐가 다른가?

**Steps**는 위에서 아래로 순차 실행. 같은 단계 안에 여러 작업을 두면 병렬 실행됨.

```yaml
- name: pipeline
  steps:
  - - name: download
      template: download-data
  - - name: train         # download가 끝나야 시작
      template: train-model
    - name: validate      # train과 동시에 실행됨 (같은 들여쓰기)
      template: validate-data
```

**DAG**는 명시적으로 의존성을 선언하는 그래프. 더 유연합니다.

```yaml
- name: pipeline
  dag:
    tasks:
    - name: download
      template: download-data
    - name: preprocess-train
      template: preprocess
      dependencies: [download]
    - name: preprocess-eval
      template: preprocess
      dependencies: [download]
    - name: train
      template: train-model
      dependencies: [preprocess-train]
    - name: evaluate
      template: evaluate-model
      dependencies: [train, preprocess-eval]
```

> 💡 **실전 팁**: 작은 파이프라인은 `steps`가 읽기 쉽고, ML 파이프라인처럼 분기/합류가 많으면 `dag`가 자연스럽습니다.

### 3.4 Parameters - step 간 작은 값 전달

문자열이나 숫자처럼 **작은 데이터**를 step 사이로 흘려보낼 때 사용합니다.

```yaml
spec:
  arguments:
    parameters:
    - name: model-name
      value: "qwen2.5-7b"
  templates:
  - name: train
    inputs:
      parameters:
      - name: model-name
    container:
      image: pytorch/pytorch:2.4.0
      command: [python, train.py]
      args: ["--model", "{{inputs.parameters.model-name}}"]
```

### 3.5 Artifacts - step 간 큰 파일 전달

학습 데이터, 모델 가중치처럼 **큰 데이터**는 Artifact로 주고받습니다. 내부적으로 MinIO/S3에 저장됐다가 다음 step의 Pod에 자동 마운트됩니다.

```yaml
- name: train
  outputs:
    artifacts:
    - name: model-weights
      path: /workspace/output/model.safetensors

- name: evaluate
  inputs:
    artifacts:
    - name: model-weights
      path: /workspace/input/model.safetensors
```

### 3.6 WorkflowTemplate vs ClusterWorkflowTemplate

매번 같은 워크플로우를 정의하기 귀찮으면 템플릿화할 수 있습니다.

- **`WorkflowTemplate`**: 네임스페이스 범위의 재사용 가능한 템플릿
- **`ClusterWorkflowTemplate`**: 클러스터 전체에서 재사용 가능

조직에서 "표준 학습 파이프라인"을 만들어 두면 데이터 사이언티스트가 가져다 쓸 수 있습니다.

### 3.7 CronWorkflow - 스케줄 실행

cron 형식으로 정기 실행. v4.0부터는 복수의 스케줄을 지원합니다(`schedule` 단수형은 deprecated).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: nightly-retrain
spec:
  schedules:                  # v4.x 부터 plural
  - "0 3 * * *"               # 매일 새벽 3시
  - "0 15 * * 0"              # 일요일 오후 3시
  timezone: "Asia/Seoul"
  workflowSpec:
    entrypoint: train
    templates:
    - name: train
      container:
        image: my-trainer:latest
```

---

## 4. 실습 환경의 특수성 - 이종(Heterogeneous) 클러스터 이해하기

이 부분이 일반적인 Argo 튜토리얼과 가장 다른 부분입니다. **반드시 이해하고 넘어가야** 나중에 디버깅 지옥에 빠지지 않습니다.

### 4.1 두 노드의 정체

| 항목 | DGX Spark | ThinkStation (예: P3 / P5 / PX 시리즈) |
|------|-----------|--------------------------------------|
| **CPU** | NVIDIA Grace (ARM Cortex-X925/A725 20코어) | Intel Xeon / Core i9 (x86_64) |
| **아키텍처** | `arm64` / `aarch64` | `amd64` / `x86_64` |
| **GPU** | Blackwell GPU (GB10 통합, 1 PFLOPS FP4) | Optional (NVIDIA RTX A4000/A6000 등) |
| **메모리** | 128GB LPDDR5X 통합 메모리 | 일반 DDR5 RAM |
| **OS (전제)** | DGX OS (Ubuntu 24.04 LTS 기반) | Ubuntu 22.04/24.04 |
| **역할 (권장)** | GPU 워크로드, LLM 학습/추론 | 컨트롤 플레인, 데이터 전처리 |

### 4.2 발생할 수 있는 5가지 함정

#### 함정 1: 컨테이너 이미지 아키텍처 불일치

`docker pull pytorch/pytorch:latest`를 그냥 하면 보통 `linux/amd64` 이미지가 받아집니다. 이걸 DGX Spark(ARM64)에서 실행하면 즉시 `exec format error`로 죽습니다.

**해결**: 멀티 아키텍처 이미지를 사용하거나, 노드별로 다른 이미지를 쓰도록 분기.

```bash
# 멀티 아키 이미지인지 확인
docker manifest inspect nvcr.io/nvidia/pytorch:24.10-py3 | grep architecture
# "architecture": "amd64",
# "architecture": "arm64",   ← 둘 다 있어야 함
```

NVIDIA NGC의 PyTorch 컨테이너는 24.x 버전부터 멀티 아키텍처를 지원하므로 DGX Spark에서도 그대로 쓸 수 있습니다.

#### 함정 2: GPU가 없는 노드에 GPU Pod가 스케줄링되는 문제

기본적으로 K8s는 GPU 자원을 요청하지 않은 Pod를 GPU 노드에도 배치할 수 있습니다. GPU 노드의 자원이 일반 작업으로 점유되면 정작 GPU 작업이 펜딩됩니다.

**해결**: `nodeSelector`, `tolerations`, `affinity` 적극 활용.

#### 함정 3: 아티팩트 저장소가 한 노드에만 있을 때

MinIO를 ThinkStation에만 띄웠는데 DGX Spark의 GPU Pod가 큰 모델을 업로드할 때 네트워크 병목이 생길 수 있습니다. 두 노드는 빠른 네트워크로 연결되어 있어야 합니다(권장 10GbE 이상, DGX Spark의 ConnectX-7을 활용하면 200GbE까지 가능).

#### 함정 4: 호스트 경로 PV 사용 시 노드 종속

`hostPath` 볼륨은 그 노드에서만 유효합니다. Pod가 다른 노드에 스케줄되면 데이터를 못 찾습니다.

**해결**: 로컬 클러스터에서는 `local-path-provisioner`(Rancher) 또는 `OpenEBS LocalPV`를 사용하고, 데이터 공유는 NFS 또는 MinIO로.

#### 함정 5: NVIDIA Container Toolkit 미설치

DGX OS에는 기본 설치되어 있지만, ThinkStation에 GPU가 있고 거기서도 GPU 워크로드를 돌릴 거라면 별도 설치가 필요합니다.

### 4.3 노드 라벨링 전략

실습 시작 전에 노드에 의미 있는 라벨을 붙여 두면 워크플로우가 훨씬 깔끔해집니다.

```bash
# DGX Spark 노드 (gpu-node로 가정)
kubectl label node dgx-spark workload-type=gpu
kubectl label node dgx-spark hardware=dgx-spark
kubectl label node dgx-spark accelerator=nvidia-blackwell

# ThinkStation 노드
kubectl label node thinkstation workload-type=cpu
kubectl label node thinkstation hardware=thinkstation
```

K8s에는 표준 라벨도 자동으로 붙어 있습니다.

```bash
kubectl get nodes -L kubernetes.io/arch -L kubernetes.io/os
# NAME           ARCH    OS
# dgx-spark      arm64   linux
# thinkstation   amd64   linux
```

이걸 활용해서 워크로드를 보낼 곳을 정확히 지정할 수 있습니다.

---

## 5. 2-Node 로컬 K8s 클러스터 구축

이 섹션은 클러스터가 이미 구성되어 있다면 건너뛰어도 됩니다. 실습은 `kubeadm` 기반 클러스터를 가정하지만, k3s/microk8s/RKE2도 동일한 개념이 적용됩니다.

### 5.1 사전 준비 (두 노드 공통)

```bash
# 1) 스왑 비활성화
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2) 커널 모듈 및 sysctl
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# 3) containerd 설치 및 SystemdCgroup 활성화
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# 4) kubeadm/kubelet/kubectl 설치 (v1.30 라인 권장)
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 5.2 컨트롤 플레인 초기화 (ThinkStation에서)

ThinkStation을 컨트롤 플레인으로 두는 이유는 GPU 노드인 DGX Spark가 학습 작업으로 바쁠 때도 클러스터가 안정적으로 운영되기 때문입니다.

```bash
# Pod CIDR은 사용하려는 CNI에 맞게 (Calico 기본: 192.168.0.0/16)
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<ThinkStation의 내부 IP>

# kubeconfig 설정
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# CNI 설치 (Calico)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

`kubeadm init` 마지막에 출력되는 `kubeadm join ...` 명령어를 복사해 둡니다.

### 5.3 워커 노드 조인 (DGX Spark에서)

```bash
# kubeadm init 결과로 받은 명령어 그대로
sudo kubeadm join <thinkstation-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

조인 후 ThinkStation에서 확인:

```bash
kubectl get nodes -o wide
# NAME           STATUS   ROLES           AGE   VERSION   ...    OS-IMAGE                                ARCH
# thinkstation   Ready    control-plane   5m    v1.30.x   ...    Ubuntu 24.04 LTS                        amd64
# dgx-spark      Ready    <none>          1m    v1.30.x   ...    DGX OS 7.x (Ubuntu 24.04 LTS based)     arm64
```

### 5.4 NVIDIA GPU Operator 설치

DGX Spark의 GPU를 K8s에서 인식시키려면 GPU Operator가 필요합니다. DGX OS에는 드라이버와 NVIDIA Container Toolkit이 이미 설치되어 있으므로 이를 활용하는 옵션을 줍니다.

```bash
# Helm 설치
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | \
  sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update && sudo apt-get install -y helm

# GPU Operator 설치 (DGX는 드라이버/툴킷 사전 설치되어 있음)
helm repo add nvidia https://nvidia.github.io/gpu-operator
helm repo update
kubectl create namespace gpu-operator

helm install --wait gpu-operator nvidia/gpu-operator \
  -n gpu-operator \
  --set driver.enabled=false \
  --set toolkit.enabled=false

# 확인
kubectl get pods -n gpu-operator
kubectl describe node dgx-spark | grep -A5 "Allocatable"
# nvidia.com/gpu: 1   ← 이게 보이면 성공
```

### 5.5 노드 라벨링

```bash
kubectl label node dgx-spark workload-type=gpu accelerator=nvidia-blackwell
kubectl label node thinkstation workload-type=cpu
```

### 5.6 MinIO 설치 (Artifact Repository)

```bash
kubectl create namespace argo

# MinIO를 ThinkStation에 고정
cat <<EOF | kubectl apply -n argo -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
spec:
  replicas: 1
  selector:
    matchLabels: {app: minio}
  template:
    metadata:
      labels: {app: minio}
    spec:
      nodeSelector:
        workload-type: cpu
      containers:
      - name: minio
        image: quay.io/minio/minio:latest
        args: ["server", "/data", "--console-address", ":9001"]
        env:
        - name: MINIO_ROOT_USER
          value: admin
        - name: MINIO_ROOT_PASSWORD
          value: password123
        ports:
        - containerPort: 9000
        - containerPort: 9001
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        emptyDir: {}            # 실습용. 운영은 PV 사용
---
apiVersion: v1
kind: Service
metadata:
  name: minio
spec:
  selector: {app: minio}
  ports:
  - name: api
    port: 9000
  - name: console
    port: 9001
EOF

# 버킷 생성
kubectl run -n argo --rm -it minio-mc --image=minio/mc --restart=Never -- \
  /bin/sh -c "mc alias set local http://minio:9000 admin password123 && mc mb local/argo-artifacts"
```

---

## 6. Argo Workflows 설치

### 6.1 설치

```bash
ARGO_VERSION="v4.0.5"
kubectl apply -n argo -f \
  https://github.com/argoproj/argo-workflows/releases/download/${ARGO_VERSION}/install.yaml

# 컨트롤러와 서버 모두 ThinkStation(amd64)에 두는 게 안정적
kubectl patch deployment workflow-controller -n argo --patch '
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/arch: amd64
        workload-type: cpu'

kubectl patch deployment argo-server -n argo --patch '
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/arch: amd64
        workload-type: cpu'
```

> ⚠️ **참고**: 공식 install.yaml의 컨트롤러/서버 이미지는 멀티 아키텍처를 지원하지만, 안정성을 위해 컨트롤러는 한 아키텍처에 고정하는 것이 권장됩니다.

### 6.2 Argo CLI 설치

각 노드 아키텍처에 맞는 바이너리를 설치합니다.

```bash
# ThinkStation (amd64)
ARGO_VERSION="v4.0.5"
curl -sLO "https://github.com/argoproj/argo-workflows/releases/download/${ARGO_VERSION}/argo-linux-amd64.gz"
gunzip argo-linux-amd64.gz
chmod +x argo-linux-amd64
sudo mv argo-linux-amd64 /usr/local/bin/argo
argo version

# DGX Spark (arm64)에서 작업하려면 argo-linux-arm64.gz 다운로드
```

### 6.3 Artifact Repository 설정

```bash
# MinIO 시크릿
kubectl create secret -n argo generic minio-credentials \
  --from-literal=accesskey=admin \
  --from-literal=secretkey=password123

# Artifact Repository ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: artifact-repositories
  namespace: argo
  annotations:
    workflows.argoproj.io/default-artifact-repository: default-v1
data:
  default-v1: |
    s3:
      bucket: argo-artifacts
      endpoint: minio.argo.svc.cluster.local:9000
      insecure: true
      accessKeySecret:
        name: minio-credentials
        key: accesskey
      secretKeySecret:
        name: minio-credentials
        key: secretkey
EOF
```

### 6.4 UI 접속

```bash
# 인증 모드를 server로 변경(실습용. 운영은 SSO 권장)
kubectl patch deployment argo-server -n argo --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/args","value":["server","--auth-mode=server"]}]'

# 포트 포워딩
kubectl port-forward -n argo svc/argo-server 2746:2746
```

브라우저에서 `https://localhost:2746` 접속 (자체 서명 인증서 경고는 무시).

### 6.5 RBAC 설정 (워크플로우 실행 권한)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workflow-executor
  namespace: argo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: workflow-executor
  namespace: argo
rules:
- apiGroups: ["argoproj.io"]
  resources: ["workflowtaskresults", "workflowartifactgctasks"]
  verbs: ["create", "patch", "get", "list", "watch", "update"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: workflow-executor
  namespace: argo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: workflow-executor
subjects:
- kind: ServiceAccount
  name: workflow-executor
  namespace: argo
EOF
```

이후 모든 워크플로우는 `serviceAccountName: workflow-executor`를 명시합니다.

---

## 7. 실습 1 - Hello World부터 DAG까지

### 7.1 첫 워크플로우: Hello World

`01-hello.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-
  namespace: argo
spec:
  serviceAccountName: workflow-executor
  entrypoint: main
  templates:
  - name: main
    container:
      image: alpine:3.19              # alpine은 멀티아키 이미지
      command: [sh, -c]
      args: ["echo 안녕! 나는 $(uname -m) 아키텍처의 노드에서 실행 중!"]
```

실행:

```bash
argo submit -n argo --watch 01-hello.yaml
```

실행 후 어떤 노드에 떴는지 확인:

```bash
argo logs -n argo @latest
kubectl get pods -n argo -l workflows.argoproj.io/workflow=<workflow-name> -o wide
```

> 🎯 **관찰 포인트**: 별다른 nodeSelector가 없으면 K8s가 임의 노드에 배치합니다. `uname -m` 결과가 `aarch64`이면 DGX, `x86_64`이면 ThinkStation에 떴다는 뜻입니다.

### 7.2 파라미터 받기

`02-params.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: params-
  namespace: argo
spec:
  serviceAccountName: workflow-executor
  entrypoint: main
  arguments:
    parameters:
    - name: greeting
      value: "Hello from Argo"
    - name: count
      value: "3"
  templates:
  - name: main
    inputs:
      parameters:
      - name: greeting
      - name: count
    container:
      image: alpine:3.19
      command: [sh, -c]
      args:
      - |
        for i in $(seq 1 {{inputs.parameters.count}}); do
          echo "[$i] {{inputs.parameters.greeting}}"
        done
```

실행 시 파라미터 오버라이드:

```bash
argo submit -n argo --watch 02-params.yaml \
  -p greeting="안녕 MLOps" -p count=5
```

### 7.3 Steps - 순차 실행

`03-steps.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: steps-
  namespace: argo
spec:
  serviceAccountName: workflow-executor
  entrypoint: pipeline
  templates:
  - name: pipeline
    steps:
    - - name: step-a
        template: print-msg
        arguments:
          parameters: [{name: msg, value: "1단계 - 데이터 다운로드"}]
    - - name: step-b
        template: print-msg
        arguments:
          parameters: [{name: msg, value: "2단계 - 전처리"}]
    - - name: step-c1                  # step-b 종료 후 c1, c2 동시 실행
        template: print-msg
        arguments:
          parameters: [{name: msg, value: "3a단계 - 학습 (병렬)"}]
      - name: step-c2
        template: print-msg
        arguments:
          parameters: [{name: msg, value: "3b단계 - 검증 (병렬)"}]

  - name: print-msg
    inputs:
      parameters: [{name: msg}]
    container:
      image: alpine:3.19
      command: [echo, "{{inputs.parameters.msg}}"]
```

UI(`https://localhost:2746`)에서 보면 step-c1, step-c2가 나란히 실행되는 그래프를 확인할 수 있습니다.

### 7.4 DAG - 진짜 그래프

`04-dag.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-
  namespace: argo
spec:
  serviceAccountName: workflow-executor
  entrypoint: ml-pipeline
  templates:
  - name: ml-pipeline
    dag:
      tasks:
      - name: ingest
        template: echo
        arguments: {parameters: [{name: msg, value: "데이터 수집"}]}

      - name: clean
        template: echo
        dependencies: [ingest]
        arguments: {parameters: [{name: msg, value: "데이터 클렌징"}]}

      - name: feature-eng
        template: echo
        dependencies: [clean]
        arguments: {parameters: [{name: msg, value: "피처 엔지니어링"}]}

      - name: split-train
        template: echo
        dependencies: [feature-eng]
        arguments: {parameters: [{name: msg, value: "학습 셋 분리"}]}

      - name: split-test
        template: echo
        dependencies: [feature-eng]
        arguments: {parameters: [{name: msg, value: "테스트 셋 분리"}]}

      - name: train
        template: echo
        dependencies: [split-train]
        arguments: {parameters: [{name: msg, value: "모델 학습"}]}

      - name: evaluate
        template: echo
        dependencies: [train, split-test]
        arguments: {parameters: [{name: msg, value: "모델 평가"}]}

  - name: echo
    inputs: {parameters: [{name: msg}]}
    container:
      image: alpine:3.19
      command: [echo, "{{inputs.parameters.msg}}"]
```

UI에서 보면 **다이아몬드 모양의 분기**가 그려집니다. ML 파이프라인의 전형적인 모습입니다.

---

## 8. 실습 2 - GPU 워크로드와 노드 어피니티

### 8.1 GPU 노드에 명시적 배치

`05-gpu-check.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: gpu-check-
  namespace: argo
spec:
  serviceAccountName: workflow-executor
  entrypoint: nvidia-smi
  templates:
  - name: nvidia-smi
    nodeSelector:
      workload-type: gpu                    # GPU 노드만
      kubernetes.io/arch: arm64             # DGX Spark 특정
    container:
      image: nvcr.io/nvidia/cuda:12.6.0-base-ubuntu24.04
      command: [nvidia-smi]
      resources:
        limits:
          nvidia.com/gpu: 1                 # GPU 1개 요청
```

실행 후 로그를 보면 Blackwell GPU의 정보가 출력됩니다.

```bash
argo submit -n argo --watch 05-gpu-check.yaml
argo logs -n argo @latest
```

### 8.2 노드별 작업 분배

CPU 작업은 ThinkStation, GPU 작업은 DGX Spark로 보내는 패턴입니다.

`06-mixed-workload.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: mixed-
  namespace: argo
spec:
  serviceAccountName: workflow-executor
  entrypoint: pipeline
  templates:
  - name: pipeline
    dag:
      tasks:
      - name: preprocess
        template: cpu-preprocess
      - name: train
        template: gpu-train
        dependencies: [preprocess]
      - name: report
        template: cpu-report
        dependencies: [train]

  # CPU 작업: ThinkStation에 배치
  - name: cpu-preprocess
    nodeSelector:
      workload-type: cpu
    container:
      image: python:3.11-slim
      command: [python, -c]
      args:
      - |
        import platform
        print(f"전처리 실행 중: {platform.machine()} on {platform.node()}")
        # 실제로는 데이터 클렌징, 토크나이징 등 수행
        print("전처리 완료")

  # GPU 작업: DGX Spark에 배치
  - name: gpu-train
    nodeSelector:
      workload-type: gpu
    container:
      image: nvcr.io/nvidia/pytorch:24.10-py3
      command: [python, -c]
      args:
      - |
        import torch, platform
        print(f"학습 실행 중: {platform.machine()} on {platform.node()}")
        print(f"CUDA 사용 가능: {torch.cuda.is_available()}")
        print(f"GPU: {torch.cuda.get_device_name(0)}")
        # 실제 학습 코드는 여기에
      resources:
        limits:
          nvidia.com/gpu: 1

  - name: cpu-report
    nodeSelector:
      workload-type: cpu
    container:
      image: alpine:3.19
      command: [sh, -c]
      args: ["echo '리포트 생성 완료. Slack 알림을 여기서 보냄.'"]
```

> 💡 **검증 팁**: 실행 후 `kubectl get pods -n argo -o wide`로 각 step이 정확히 어느 노드에 떴는지 확인하세요.

### 8.3 Tolerations - GPU 노드 보호

GPU 노드에 일반 작업이 들어오는 걸 막으려면 **taint**를 걸어 두고, GPU 작업만 **toleration**을 갖게 합니다.

```bash
# DGX Spark에 taint 적용
kubectl taint node dgx-spark accelerator=nvidia:NoSchedule
```

이제 Pod에 toleration이 없으면 DGX Spark에 스케줄되지 않습니다. GPU Pod에는 다음을 추가합니다.

```yaml
- name: gpu-train
  nodeSelector:
    workload-type: gpu
  tolerations:
  - key: accelerator
    operator: Equal
    value: nvidia
    effect: NoSchedule
  container: ...
```

---

## 9. 실습 3 - 진짜 ML 파이프라인 만들기

이제 실제로 의미 있는 워크플로우를 만들어 봅시다. 시나리오는 **간단한 LLM 추론 평가 파이프라인**입니다.

### 9.1 시나리오

1. **준비** (CPU): 평가 데이터셋 다운로드
2. **추론** (GPU): 작은 LLM(예: Qwen2.5-0.5B)으로 답변 생성
3. **평가** (CPU): 생성된 답변을 자동 채점
4. **리포트** (CPU): 결과를 MinIO에 저장

### 9.2 워크플로우 정의

`07-llm-eval.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: llm-eval-
  namespace: argo
spec:
  serviceAccountName: workflow-executor
  entrypoint: eval-pipeline
  arguments:
    parameters:
    - name: model-name
      value: "Qwen/Qwen2.5-0.5B-Instruct"
    - name: sample-size
      value: "20"

  templates:
  # ───────────────── 메인 DAG ─────────────────
  - name: eval-pipeline
    inputs:
      parameters:
      - name: model-name
      - name: sample-size
    dag:
      tasks:
      - name: prepare-data
        template: prepare-data
        arguments:
          parameters:
          - {name: sample-size, value: "{{inputs.parameters.sample-size}}"}

      - name: run-inference
        template: run-inference
        dependencies: [prepare-data]
        arguments:
          parameters:
          - {name: model-name, value: "{{inputs.parameters.model-name}}"}
          artifacts:
          - name: dataset
            from: "{{tasks.prepare-data.outputs.artifacts.dataset}}"

      - name: score-results
        template: score-results
        dependencies: [run-inference]
        arguments:
          artifacts:
          - name: predictions
            from: "{{tasks.run-inference.outputs.artifacts.predictions}}"
          - name: dataset
            from: "{{tasks.prepare-data.outputs.artifacts.dataset}}"

      - name: report
        template: report
        dependencies: [score-results]
        arguments:
          parameters:
          - {name: model-name, value: "{{inputs.parameters.model-name}}"}
          artifacts:
          - name: scores
            from: "{{tasks.score-results.outputs.artifacts.scores}}"

  # ───────────────── Step 1: 데이터 준비 (CPU) ─────────────────
  - name: prepare-data
    inputs:
      parameters:
      - name: sample-size
    nodeSelector:
      workload-type: cpu
    container:
      image: python:3.11-slim
      command: [sh, -c]
      args:
      - |
        pip install --quiet datasets
        python <<'PY'
        import json
        from datasets import load_dataset
        n = int("{{inputs.parameters.sample-size}}")
        ds = load_dataset("openai/gsm8k", "main", split=f"test[:{n}]")
        with open("/tmp/dataset.jsonl", "w") as f:
            for row in ds:
                f.write(json.dumps({"q": row["question"], "a": row["answer"]}) + "\n")
        print(f"{n} 샘플 준비 완료")
        PY
    outputs:
      artifacts:
      - name: dataset
        path: /tmp/dataset.jsonl

  # ───────────────── Step 2: LLM 추론 (GPU, DGX Spark) ─────────────────
  - name: run-inference
    inputs:
      parameters:
      - name: model-name
      artifacts:
      - name: dataset
        path: /workspace/dataset.jsonl
    nodeSelector:
      workload-type: gpu
    tolerations:
    - {key: accelerator, operator: Equal, value: nvidia, effect: NoSchedule}
    container:
      image: nvcr.io/nvidia/pytorch:24.10-py3
      command: [sh, -c]
      args:
      - |
        pip install --quiet transformers accelerate
        python <<'PY'
        import json, torch
        from transformers import AutoModelForCausalLM, AutoTokenizer

        model_name = "{{inputs.parameters.model-name}}"
        tok = AutoTokenizer.from_pretrained(model_name)
        model = AutoModelForCausalLM.from_pretrained(
            model_name, torch_dtype=torch.bfloat16, device_map="cuda"
        )

        results = []
        with open("/workspace/dataset.jsonl") as f:
            for line in f:
                row = json.loads(line)
                msgs = [{"role": "user", "content": row["q"]}]
                prompt = tok.apply_chat_template(msgs, tokenize=False, add_generation_prompt=True)
                ids = tok(prompt, return_tensors="pt").to("cuda")
                out = model.generate(**ids, max_new_tokens=256, do_sample=False)
                ans = tok.decode(out[0][ids["input_ids"].shape[1]:], skip_special_tokens=True)
                results.append({"q": row["q"], "predicted": ans})

        with open("/workspace/predictions.jsonl", "w") as f:
            for r in results:
                f.write(json.dumps(r, ensure_ascii=False) + "\n")
        print(f"{len(results)}개 추론 완료")
        PY
      resources:
        limits:
          nvidia.com/gpu: 1
    outputs:
      artifacts:
      - name: predictions
        path: /workspace/predictions.jsonl

  # ───────────────── Step 3: 채점 (CPU) ─────────────────
  - name: score-results
    inputs:
      artifacts:
      - name: predictions
        path: /workspace/predictions.jsonl
      - name: dataset
        path: /workspace/dataset.jsonl
    nodeSelector:
      workload-type: cpu
    container:
      image: python:3.11-slim
      command: [sh, -c]
      args:
      - |
        python <<'PY'
        import json, re

        def extract(text):
            m = re.findall(r"-?\d+\.?\d*", text.replace(",", ""))
            return m[-1] if m else None

        gold = [json.loads(l) for l in open("/workspace/dataset.jsonl")]
        pred = [json.loads(l) for l in open("/workspace/predictions.jsonl")]

        correct = 0
        for g, p in zip(gold, pred):
            g_ans = extract(g["a"].split("####")[-1])
            p_ans = extract(p["predicted"])
            if g_ans and p_ans and g_ans == p_ans:
                correct += 1

        score = correct / len(gold) if gold else 0
        with open("/workspace/scores.json", "w") as f:
            json.dump({"correct": correct, "total": len(gold), "accuracy": score}, f)
        print(f"정확도: {score*100:.1f}% ({correct}/{len(gold)})")
        PY
    outputs:
      artifacts:
      - name: scores
        path: /workspace/scores.json

  # ───────────────── Step 4: 리포트 (CPU) ─────────────────
  - name: report
    inputs:
      parameters:
      - name: model-name
      artifacts:
      - name: scores
        path: /workspace/scores.json
    nodeSelector:
      workload-type: cpu
    container:
      image: python:3.11-slim
      command: [sh, -c]
      args:
      - |
        python <<'PY'
        import json
        s = json.load(open("/workspace/scores.json"))
        print("=" * 50)
        print(f"모델: {{inputs.parameters.model-name}}")
        print(f"정확도: {s['accuracy']*100:.2f}%")
        print(f"정답/전체: {s['correct']}/{s['total']}")
        print("=" * 50)
        PY
```

### 9.3 실행 및 관찰

```bash
argo submit -n argo --watch 07-llm-eval.yaml
```

UI에서 보면 4개 step이 차례로 실행되고, **각 step의 아티팩트(`dataset`, `predictions`, `scores`)가 자동으로 MinIO에 업로드/다운로드**되는 걸 확인할 수 있습니다.

`run-inference` step만 GPU 노드에서 실행되고 나머지는 CPU 노드에서 실행됩니다.

---

## 10. 실습 4 - 하이퍼파라미터 스윕(LLM Fine-tuning 시나리오)

`withItems` 또는 `withSequence`를 쓰면 동일한 template을 여러 파라미터로 **병렬** 실행할 수 있습니다. 하이퍼파라미터 스윕에 매우 유용합니다.

### 10.1 워크플로우

`08-hp-sweep.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hp-sweep-
  namespace: argo
spec:
  serviceAccountName: workflow-executor
  entrypoint: sweep
  parallelism: 2          # 동시 실행 최대 2개 (GPU 1장이라도 순차 처리)
  templates:
  - name: sweep
    steps:
    - - name: train-with-hp
        template: train
        arguments:
          parameters:
          - {name: lr,    value: "{{item.lr}}"}
          - {name: bs,    value: "{{item.bs}}"}
          - {name: epoch, value: "{{item.epoch}}"}
        withItems:
        - {lr: "1e-4", bs: "16", epoch: "1"}
        - {lr: "1e-4", bs: "32", epoch: "1"}
        - {lr: "5e-5", bs: "16", epoch: "1"}
        - {lr: "5e-5", bs: "32", epoch: "1"}

  - name: train
    inputs:
      parameters:
      - {name: lr}
      - {name: bs}
      - {name: epoch}
    nodeSelector:
      workload-type: gpu
    tolerations:
    - {key: accelerator, operator: Equal, value: nvidia, effect: NoSchedule}
    container:
      image: nvcr.io/nvidia/pytorch:24.10-py3
      command: [sh, -c]
      args:
      - |
        echo "[학습 시작] lr={{inputs.parameters.lr}} bs={{inputs.parameters.bs}} epoch={{inputs.parameters.epoch}}"
        # 실제 학습 스크립트 호출
        # python finetune.py --lr {{inputs.parameters.lr}} --bs {{inputs.parameters.bs}} ...
        echo "[학습 완료]"
      resources:
        limits:
          nvidia.com/gpu: 1
```

### 10.2 핵심 포인트

- **`parallelism: 2`**: GPU가 1개뿐이어도 큐에 쌓아두고 처리됨. 단, 동시에 GPU를 점유하려면 GPU 수를 늘리거나 MIG 분할이 필요.
- **`withItems`**: 리스트의 각 원소마다 별도의 Pod 생성
- **DGX Spark의 통합 메모리(128GB)** 덕분에 일반 GPU 대비 큰 모델/배치 사이즈를 다룰 수 있음

### 10.3 결과 집계 패턴

각 학습의 결과를 모아서 비교하려면 다음과 같이 합니다.

```yaml
# 각 train의 outputs로 metric 결과를 받고
# aggregation step에서 모음

- name: train
  outputs:
    parameters:
    - name: val-loss
      valueFrom:
        path: /workspace/val_loss.txt    # 컨테이너 안에서 학습 후 기록

# steps 또는 dag 다음 단계에서
- name: pick-best
  inputs:
    parameters:
    - name: results            # JSON 배열로 받기
  script:
    image: python:3.11-slim
    command: [python]
    source: |
      import json, sys
      results = json.loads("""{{inputs.parameters.results}}""")
      best = min(results, key=lambda x: float(x['val-loss']))
      print(f"최적 하이퍼파라미터: {best}")
```

---

## 11. 운영 베스트 프랙티스

### 11.1 리소스 관리

| 항목 | 권장 |
|------|------|
| `requests`/`limits` | 모든 컨테이너에 명시 (특히 GPU). 미설정 시 노드 자원 고갈 위험 |
| `parallelism` | 워크플로우와 controller config 모두에서 제한 (같은 시점에 너무 많은 Pod 방지) |
| `activeDeadlineSeconds` | 무한정 도는 작업 방지용 타임아웃 |
| `retryStrategy` | 일시적 실패에 대비 |

```yaml
spec:
  parallelism: 4
  activeDeadlineSeconds: 7200        # 2시간 제한
  templates:
  - name: train
    retryStrategy:
      limit: 3
      retryPolicy: "Always"
      backoff:
        duration: "1m"
        factor: 2
        maxDuration: "10m"
```

### 11.2 보안

- **`serviceAccountName` 명시 필수**: 기본 `default` SA는 권한이 없거나 너무 많을 수 있음
- **Pod Security Standards(PSS)** 적용: `restricted` 프로파일 권장
- **시크릿 마운트**: 하드코딩 금지. 항상 K8s Secret을 환경변수로
- **Argo Server는 SSO 연동**: `--auth-mode=server`는 실습 전용

### 11.3 아티팩트 정리

큰 모델 가중치를 매번 저장하면 MinIO가 빨리 가득 찹니다.

```yaml
spec:
  artifactGC:
    strategy: OnWorkflowDeletion        # 워크플로우 삭제 시 아티팩트도 삭제
  ttlStrategy:
    secondsAfterCompletion: 86400       # 24시간 후 자동 정리
```

### 11.4 관찰성(Observability)

- **Prometheus 메트릭**: workflow-controller가 `/metrics` 노출
- **로그**: `argo logs <wf-name>` 또는 Loki 연동
- **이벤트**: `kubectl get events -n argo`
- **알림**: Slack/Webhook은 `exit handler` template으로

```yaml
spec:
  onExit: notify
  templates:
  - name: notify
    container:
      image: curlimages/curl
      command: [sh, -c]
      args:
      - |
        STATUS="{{workflow.status}}"
        curl -X POST $SLACK_WEBHOOK -d "{\"text\":\"Workflow {{workflow.name}} ended: $STATUS\"}"
```

### 11.5 멀티 아키텍처 이미지 빌드

직접 만든 이미지는 `docker buildx`로 두 아키텍처 모두 빌드:

```bash
docker buildx create --name multi --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myregistry/my-trainer:1.0 \
  --push .
```

이렇게 만든 이미지는 어느 노드에서 pull해도 자동으로 맞는 아키텍처 레이어가 받아집니다.

---

## 12. 트러블슈팅 자주 만나는 함정

### 12.1 `exec format error`

**원인**: ARM 노드에 amd64 이미지가 떨어졌거나 그 반대.

**확인**:
```bash
kubectl describe pod <pod-name> -n argo | grep "Image:"
docker manifest inspect <image>
```

**해결**: 멀티아키 이미지 사용 또는 `nodeSelector: kubernetes.io/arch: amd64`로 강제.

### 12.2 GPU Pod가 Pending에서 안 움직임

**원인 후보**:
1. `nvidia.com/gpu` 리소스를 GPU Operator가 제공하지 않음
2. taint는 있는데 toleration이 없음
3. 다른 Pod가 이미 GPU를 점유 중

**확인**:
```bash
kubectl describe pod <pod-name> -n argo | tail -20      # Events 섹션 확인
kubectl get nodes -o json | jq '.items[].status.allocatable | with_entries(select(.key | contains("gpu")))'
```

### 12.3 아티팩트 업로드 실패

**원인**:
- MinIO 시크릿 이름 오타
- 버킷이 존재하지 않음
- Pod에서 MinIO 서비스 DNS 해석 실패

**확인**:
```bash
kubectl logs <pod-name> -c wait -n argo                 # wait 사이드카 로그
kubectl run -n argo --rm -it test --image=minio/mc --restart=Never -- \
  mc alias set local http://minio:9000 admin password123
```

### 12.4 워크플로우가 영원히 Running

**원인**: Pod는 끝났는데 controller가 인지 못 함. v4.0의 write-back informer 비활성화 영향일 수 있음.

**확인**:
```bash
kubectl logs -n argo deployment/workflow-controller | grep <wf-name>
```

**해결**: controller 재시작 또는 `argo retry` / `argo terminate` 후 재실행.

### 12.5 DGX Spark가 SchedulingDisabled

DGX OS 업데이트 등으로 노드가 cordon된 경우.

```bash
kubectl get nodes
kubectl uncordon dgx-spark
```

---

## 13. 부록 - 다음 단계 학습 로드맵

### 13.1 깊이 가기 위한 주제들

1. **Hera (Python SDK)**: YAML 대신 Python으로 워크플로우 작성
   ```python
   from hera.workflows import Workflow, Container
   with Workflow(generate_name="hello-") as w:
       Container(name="main", image="alpine:3.19", command=["echo", "hi"])
   w.create()
   ```
2. **Workflow of Workflows**: 큰 파이프라인을 작은 워크플로우 조합으로
3. **Sensors (Argo Events)**: 외부 이벤트(Kafka, S3, Webhook)로 워크플로우 트리거
4. **Memoization**: 같은 입력에 대해 step 결과를 캐싱
5. **Database-backed archive**: PostgreSQL에 워크플로우 이력 영속화

### 13.2 Argo와 함께 쓰면 좋은 도구

| 영역 | 도구 |
|------|------|
| **CD/GitOps** | Argo CD - 워크플로우 정의를 Git으로 관리 |
| **이벤트 기반** | Argo Events - 외부 트리거 |
| **롤아웃** | Argo Rollouts - 모델 카나리 배포 |
| **추적** | MLflow, Weights & Biases - 실험 추적 |
| **모델 서빙** | KServe, BentoML - 학습된 모델 서빙 |
| **데이터** | MinIO, DVC - 데이터/모델 버전 관리 |

### 13.3 프로덕션 체크리스트

- [ ] `serviceAccountName` 모든 워크플로우에 명시
- [ ] `resources.requests/limits` 모든 컨테이너에 설정
- [ ] `activeDeadlineSeconds`로 타임아웃 설정
- [ ] `retryStrategy` 정의
- [ ] `artifactGC` 또는 `ttlStrategy`로 정리 정책
- [ ] 멀티 아키텍처 이미지 사용 또는 nodeSelector 명시
- [ ] GPU 노드 taint와 toleration 쌍 설정
- [ ] Prometheus 메트릭 수집 활성화
- [ ] Argo Server SSO 인증 (운영 환경)
- [ ] PostgreSQL 기반 archive로 이력 보존

---

## 마무리

여기까지 따라오셨다면, 이제 다음을 할 수 있어야 합니다.

1. ✅ Argo Workflows의 핵심 개념(Workflow, Template, Steps/DAG, Parameters, Artifacts)을 설명할 수 있다
2. ✅ DGX Spark + ThinkStation 이종 클러스터에 Argo를 설치하고 실행할 수 있다
3. ✅ ARM/x86 아키텍처 차이로 인한 함정을 피할 수 있다
4. ✅ GPU 워크로드를 정확한 노드에 배치할 수 있다
5. ✅ 진짜 LLM 평가 파이프라인을 YAML로 작성하고 운영할 수 있다
6. ✅ 하이퍼파라미터 스윕을 병렬로 돌릴 수 있다

**핵심 정신**: Argo는 "각 단계는 컨테이너, 단계 간 연결은 그래프"라는 단순한 모델을 끝까지 밀고 갑니다. 처음에는 YAML이 길어 보이지만, 한번 익숙해지면 어떤 ML 워크플로우도 같은 패턴으로 표현할 수 있다는 걸 깨닫게 됩니다.

**즐거운 MLOps 여정 되세요!** 🚀

---

## 참고 자료

- 공식 문서: https://argo-workflows.readthedocs.io/
- GitHub: https://github.com/argoproj/argo-workflows
- 예제 모음: https://github.com/argoproj/argo-workflows/tree/main/examples
- DGX Spark 사용자 가이드: https://docs.nvidia.com/dgx/dgx-spark/
- NVIDIA GPU Operator: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/
