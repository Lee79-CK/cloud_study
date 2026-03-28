# Kubeflow 종합 강의: 기본 개념부터 실무 적용까지

> **대상**: MLOps / LLMOps / Cloud Engineer  
> **수준**: 중급 → 고급  
> **구성**: 8개 모듈, 실습 코드 포함

---

## 목차

1. [Module 1] Kubeflow 개요와 아키텍처
2. [Module 2] 설치 및 환경 구성
3. [Module 3] Kubeflow Pipelines 기초
4. [Module 4] Kubeflow Pipelines 심화
5. [Module 5] 핵심 컴포넌트 Deep Dive
6. [Module 6] 고급 기법 및 최적화
7. [Module 7] 다른 MLOps 도구와의 연계
8. [Module 8] 실무 프로젝트: End-to-End ML 시스템 구축

---

# Module 1. Kubeflow 개요와 아키텍처

## 1.1 Kubeflow란?

Kubeflow는 Kubernetes 위에서 ML 워크플로를 배포, 확장, 관리하기 위한 오픈소스 플랫폼입니다. Google이 내부적으로 사용하던 TFX(TensorFlow Extended)의 경험을 바탕으로 2017년에 시작되었으며, ML 워크플로의 전체 라이프사이클(데이터 전처리 → 모델 학습 → 서빙 → 모니터링)을 Kubernetes-native 방식으로 관리합니다.

**핵심 철학**: "Make deployments of ML workflows on Kubernetes simple, portable, and scalable."

## 1.2 왜 Kubeflow인가?

전통적인 ML 개발 방식에서는 로컬 환경에서 Jupyter로 모델을 개발한 뒤 수동으로 배포하는 과정에서 환경 불일치, 재현 불가능, 확장성 부재 등의 문제가 발생합니다. Kubeflow는 이를 해결합니다.

**Kubeflow가 풀어주는 문제들**:

- **환경 일관성**: 컨테이너 기반이므로 개발/스테이징/프로덕션 환경이 동일
- **재현성(Reproducibility)**: 파이프라인 정의, 파라미터, 아티팩트가 모두 버전 관리됨
- **확장성(Scalability)**: Kubernetes의 오토스케일링, GPU 스케줄링을 그대로 활용
- **이식성(Portability)**: 온프레미스, AWS, GCP, Azure 어디서든 동일하게 실행
- **협업**: 멀티 테넌시 지원, 실험(Experiment) 기반 관리

## 1.3 전체 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                    Kubeflow Platform                     │
│                                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐  │
│  │ Notebooks│ │Pipelines │ │  KServe  │ │  Katib    │  │
│  │ (JupyterH│ │ (Argo/   │ │ (Model   │ │ (AutoML/  │  │
│  │  ub)     │ │  Tekton) │ │  Serving)│ │  HPO)     │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐  │
│  │Training  │ │ Feature  │ │ Metadata │ │ Tensorboa │  │
│  │Operators │ │  Store   │ │  Store   │ │  rd       │  │
│  │(TFJob,   │ │ (Feast)  │ │ (MLMD)   │ │           │  │
│  │PyTorchJob│ │          │ │          │ │           │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────┘  │
│                                                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Istio Service Mesh                     │ │
│  │         (Traffic Management, Auth, mTLS)            │ │
│  └────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Kubernetes Cluster                     │ │
│  │    (Nodes, Pods, Services, PV/PVC, RBAC)           │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## 1.4 핵심 컴포넌트 요약

| 컴포넌트 | 역할 | 핵심 기술 |
|----------|------|----------|
| **Kubeflow Pipelines (KFP)** | ML 워크플로 오케스트레이션 | Argo Workflows / Tekton |
| **Notebook Servers** | 대화형 개발 환경 | JupyterHub, VS Code Server |
| **KServe** | 모델 서빙 (추론) | Knative, Istio |
| **Katib** | 하이퍼파라미터 튜닝 / NAS | Bayesian, TPE, ENAS |
| **Training Operators** | 분산 학습 | TFJob, PyTorchJob, MPIJob |
| **ML Metadata (MLMD)** | 메타데이터/리니지 추적 | gRPC, SQLite/MySQL |
| **Tensorboard** | 학습 시각화 | TensorBoard Controller |

---

# Module 2. 설치 및 환경 구성

## 2.1 설치 방식 비교

| 방식 | 난이도 | 용도 | 특징 |
|------|--------|------|------|
| **Minikube + Kustomize** | 중 | 로컬 개발/학습 | 리소스 제한적 |
| **Kind + Manifests** | 중 | CI/CD 테스트 | 경량, 빠른 시작 |
| **kubeflow/manifests** | 상 | 프로덕션 (온프레미스) | 가장 유연한 커스터마이징 |
| **AWS (Terraform/EKS)** | 중상 | AWS 프로덕션 | Cognito, RDS, S3 연동 |
| **GCP (Config Connector)** | 중 | GCP 프로덕션 | IAP, CloudSQL 자동 구성 |
| **Azure (AKS)** | 중상 | Azure 프로덕션 | AAD, Blob Storage 연동 |
| **Charmed Kubeflow** | 하 | 빠른 프로덕션 셋업 | Canonical/Juju 기반 |

## 2.2 로컬 환경 설치 (Kind 기반)

```bash
# 1. 사전 요구사항 설치
# Docker, kubectl, kind, kustomize 설치 확인
docker --version        # >= 20.10
kubectl version --client # >= 1.26
kind version            # >= 0.20
kustomize version       # >= 5.0

# 2. Kind 클러스터 생성
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: ClusterConfiguration
        apiServer:
          extraArgs:
            "service-account-issuer": "kubernetes.default.svc"
            "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"
    extraPortMappings:
      - containerPort: 31380
        hostPort: 8080   # Kubeflow Dashboard
        protocol: TCP
  - role: worker
    extraMounts:
      - hostPath: /tmp/kf-data
        containerPath: /data
EOF

kind create cluster --name kubeflow --config kind-config.yaml

# 3. Kubeflow Manifests 클론 및 설치
KF_VERSION="v1.8.1"
git clone https://github.com/kubeflow/manifests.git
cd manifests
git checkout ${KF_VERSION}

# 전체 설치 (시간 소요: ~15-20분)
while ! kustomize build example | kubectl apply -f -; do
  echo "Retrying to apply resources..."
  sleep 10
done

# 4. 설치 확인
kubectl get pods -n kubeflow --watch
# 모든 Pod가 Running/Completed 될 때까지 대기

# 5. 포트포워딩으로 대시보드 접근
kubectl port-forward svc/istio-ingressgateway \
  -n istio-system 8080:80

# 브라우저: http://localhost:8080
# 기본 계정: user@example.com / 12341234
```

## 2.3 AWS EKS 기반 프로덕션 설치

```hcl
# terraform/main.tf - EKS 클러스터 + Kubeflow 인프라

terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "kubeflow-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-northeast-2a", "ap-northeast-2b", "ap-northeast-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = false
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.17.0"

  cluster_name    = "kubeflow-prod"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    # 시스템 노드 (Kubeflow 컴포넌트)
    system = {
      instance_types = ["m5.2xlarge"]
      min_size       = 2
      max_size       = 4
      desired_size   = 3

      labels = { role = "system" }
      taints = []
    }

    # GPU 학습 노드
    gpu_training = {
      instance_types = ["g5.2xlarge"]
      min_size       = 0
      max_size       = 10
      desired_size   = 0  # 스케일 투 제로

      ami_type = "AL2_x86_64_GPU"
      labels   = { role = "training", "nvidia.com/gpu" = "true" }
      taints = [{
        key    = "nvidia.com/gpu"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }

    # 서빙 노드
    serving = {
      instance_types = ["c5.2xlarge"]
      min_size       = 2
      max_size       = 20
      desired_size   = 2

      labels = { role = "serving" }
    }
  }

  # IRSA (IAM Roles for Service Accounts)
  enable_irsa = true
}

# S3 버킷 (파이프라인 아티팩트 저장소)
resource "aws_s3_bucket" "pipeline_artifacts" {
  bucket = "kubeflow-artifacts-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "artifacts" {
  bucket = aws_s3_bucket.pipeline_artifacts.id
  versioning_configuration { status = "Enabled" }
}

# RDS (Kubeflow 메타데이터/파이프라인 DB)
resource "aws_db_instance" "kubeflow_db" {
  identifier     = "kubeflow-metadata"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.r5.large"

  allocated_storage     = 100
  max_allocated_storage = 500

  db_name  = "kubeflow"
  username = "admin"
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  db_subnet_group_name   = aws_db_subnet_group.kubeflow.name

  backup_retention_period = 7
  multi_az               = true
  storage_encrypted      = true
}

data "aws_caller_identity" "current" {}
```

```bash
# Kubeflow on AWS 설치 (Terraform 적용 후)
# 1. AWS 전용 Kubeflow 매니페스트
git clone https://github.com/awslabs/kubeflow-manifests.git
cd kubeflow-manifests

# 2. Cognito + RDS + S3 프로파일 적용
export CLUSTER_NAME="kubeflow-prod"
export CLUSTER_REGION="ap-northeast-2"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Kustomize 오버레이 적용
kustomize build deployments/cognito-rds-s3 | kubectl apply -f -

# 3. GPU 노드에 NVIDIA Device Plugin 설치
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.0/nvidia-device-plugin.yml
```

## 2.4 네임스페이스 및 멀티테넌시 구성

```yaml
# user-profile.yaml
# Kubeflow Profile은 네임스페이스 + RBAC + 리소스 쿼터를 패키징
apiVersion: kubeflow.org/v1
kind: Profile
metadata:
  name: team-ml-engineering
spec:
  owner:
    kind: User
    name: lead@company.com
  resourceQuotaSpec:
    hard:
      requests.cpu: "32"
      requests.memory: 64Gi
      requests.nvidia.com/gpu: "4"
      limits.cpu: "64"
      limits.memory: 128Gi
      limits.nvidia.com/gpu: "8"
      persistentvolumeclaims: "20"
  # 플러그인으로 추가 설정
  plugins:
    - kind: WorkloadIdentity
      spec:
        gcpServiceAccount: ml-team@project.iam.gserviceaccount.com
```

```bash
kubectl apply -f user-profile.yaml

# 프로파일 기반 네임스페이스 확인
kubectl get ns team-ml-engineering
kubectl get resourcequota -n team-ml-engineering
```

---

# Module 3. Kubeflow Pipelines (KFP) 기초

## 3.1 KFP v2 개요

KFP v2는 Kubeflow의 핵심 컴포넌트로, ML 워크플로를 DAG(Directed Acyclic Graph) 형태로 정의하고 실행합니다. v2에서는 IR(Intermediate Representation) 기반으로 바뀌면서 Argo Workflows와 Tekton 백엔드를 모두 지원합니다.

**핵심 개념:**

- **Component**: 파이프라인의 최소 실행 단위. 하나의 컨테이너로 실행됨
- **Pipeline**: Component들의 DAG
- **Run**: 파이프라인의 한 번의 실행 인스턴스
- **Experiment**: Run들을 논리적으로 그룹화하는 단위
- **Artifact**: 파이프라인 스텝 간에 전달되는 데이터(모델, 데이터셋 등)

## 3.2 첫 번째 파이프라인

```python
# 01_basic_pipeline.py
"""
KFP v2 기본 파이프라인 예제
데이터 로드 → 전처리 → 학습 → 평가의 기본 워크플로
"""

from kfp import dsl, compiler
from kfp.dsl import Input, Output, Dataset, Model, Metrics, ClassificationMetrics

# ──────────────────────────────────────────────
# Component 1: 데이터 수집
# ──────────────────────────────────────────────
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas==2.1.0", "scikit-learn==1.3.0"]
)
def load_data(
    dataset_url: str,
    output_dataset: Output[Dataset]
):
    """데이터를 다운로드하고 CSV로 저장"""
    import pandas as pd
    from sklearn.datasets import load_iris

    if dataset_url == "iris":
        data = load_iris(as_frame=True)
        df = data.frame
    else:
        df = pd.read_csv(dataset_url)

    df.to_csv(output_dataset.path, index=False)
    output_dataset.metadata["num_rows"] = len(df)
    output_dataset.metadata["num_cols"] = len(df.columns)
    print(f"Loaded dataset: {len(df)} rows, {len(df.columns)} columns")


# ──────────────────────────────────────────────
# Component 2: 데이터 전처리
# ──────────────────────────────────────────────
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas==2.1.0", "scikit-learn==1.3.0"]
)
def preprocess_data(
    input_dataset: Input[Dataset],
    test_size: float,
    train_dataset: Output[Dataset],
    test_dataset: Output[Dataset]
):
    """데이터 분할 및 스케일링"""
    import pandas as pd
    from sklearn.model_selection import train_test_split
    from sklearn.preprocessing import StandardScaler
    import json

    df = pd.read_csv(input_dataset.path)

    # 특성과 타겟 분리
    X = df.iloc[:, :-1]
    y = df.iloc[:, -1]

    # Train/Test 분할
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=test_size, random_state=42, stratify=y
    )

    # 스케일링
    scaler = StandardScaler()
    X_train_scaled = pd.DataFrame(
        scaler.fit_transform(X_train), columns=X_train.columns
    )
    X_test_scaled = pd.DataFrame(
        scaler.transform(X_test), columns=X_test.columns
    )

    # 저장
    train_df = pd.concat(
        [X_train_scaled, y_train.reset_index(drop=True)], axis=1
    )
    test_df = pd.concat(
        [X_test_scaled, y_test.reset_index(drop=True)], axis=1
    )

    train_df.to_csv(train_dataset.path, index=False)
    test_df.to_csv(test_dataset.path, index=False)

    train_dataset.metadata["rows"] = len(train_df)
    test_dataset.metadata["rows"] = len(test_df)


# ──────────────────────────────────────────────
# Component 3: 모델 학습
# ──────────────────────────────────────────────
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas==2.1.0",
        "scikit-learn==1.3.0",
        "joblib==1.3.0"
    ]
)
def train_model(
    train_dataset: Input[Dataset],
    n_estimators: int,
    max_depth: int,
    model_artifact: Output[Model]
):
    """RandomForest 모델 학습"""
    import pandas as pd
    from sklearn.ensemble import RandomForestClassifier
    import joblib

    df = pd.read_csv(train_dataset.path)
    X = df.iloc[:, :-1]
    y = df.iloc[:, -1]

    model = RandomForestClassifier(
        n_estimators=n_estimators,
        max_depth=max_depth,
        random_state=42,
        n_jobs=-1
    )
    model.fit(X, y)

    # 모델 저장
    joblib.dump(model, model_artifact.path)

    # 메타데이터 기록
    model_artifact.metadata["framework"] = "sklearn"
    model_artifact.metadata["algorithm"] = "RandomForestClassifier"
    model_artifact.metadata["n_estimators"] = n_estimators
    model_artifact.metadata["max_depth"] = max_depth


# ──────────────────────────────────────────────
# Component 4: 모델 평가
# ──────────────────────────────────────────────
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas==2.1.0",
        "scikit-learn==1.3.0",
        "joblib==1.3.0"
    ]
)
def evaluate_model(
    test_dataset: Input[Dataset],
    model_artifact: Input[Model],
    threshold: float,
    metrics: Output[Metrics],
    classification_metrics: Output[ClassificationMetrics]
) -> bool:
    """모델 평가 및 승인 여부 결정"""
    import pandas as pd
    from sklearn.metrics import (
        accuracy_score, precision_score, recall_score,
        f1_score, confusion_matrix
    )
    import joblib

    df = pd.read_csv(test_dataset.path)
    X = df.iloc[:, :-1]
    y_true = df.iloc[:, -1]

    model = joblib.load(model_artifact.path)
    y_pred = model.predict(X)

    accuracy = accuracy_score(y_true, y_pred)
    precision = precision_score(y_true, y_pred, average="weighted")
    recall = recall_score(y_true, y_pred, average="weighted")
    f1 = f1_score(y_true, y_pred, average="weighted")

    # 메트릭 로깅
    metrics.log_metric("accuracy", accuracy)
    metrics.log_metric("precision", precision)
    metrics.log_metric("recall", recall)
    metrics.log_metric("f1_score", f1)

    # Confusion Matrix 로깅
    cm = confusion_matrix(y_true, y_pred)
    labels = sorted(y_true.unique().tolist())
    classification_metrics.log_confusion_matrix(
        categories=[str(l) for l in labels],
        matrix=cm.tolist()
    )

    print(f"Accuracy: {accuracy:.4f} (threshold: {threshold})")
    is_approved = accuracy >= threshold
    return is_approved


# ──────────────────────────────────────────────
# Pipeline 정의
# ──────────────────────────────────────────────
@dsl.pipeline(
    name="iris-classification-pipeline",
    description="Iris 데이터셋 분류 파이프라인 (기초 예제)"
)
def ml_pipeline(
    dataset_url: str = "iris",
    test_size: float = 0.2,
    n_estimators: int = 100,
    max_depth: int = 5,
    accuracy_threshold: float = 0.9
):
    # Step 1: 데이터 로드
    load_task = load_data(dataset_url=dataset_url)

    # Step 2: 전처리
    preprocess_task = preprocess_data(
        input_dataset=load_task.outputs["output_dataset"],
        test_size=test_size
    )

    # Step 3: 학습
    train_task = train_model(
        train_dataset=preprocess_task.outputs["train_dataset"],
        n_estimators=n_estimators,
        max_depth=max_depth
    )
    # GPU 리소스 요청
    train_task.set_cpu_limit("4")
    train_task.set_memory_limit("8Gi")

    # Step 4: 평가
    eval_task = evaluate_model(
        test_dataset=preprocess_task.outputs["test_dataset"],
        model_artifact=train_task.outputs["model_artifact"],
        threshold=accuracy_threshold
    )


# ──────────────────────────────────────────────
# 컴파일 및 실행
# ──────────────────────────────────────────────
if __name__ == "__main__":
    # 1) YAML로 컴파일
    compiler.Compiler().compile(
        pipeline_func=ml_pipeline,
        package_path="iris_pipeline.yaml"
    )
    print("Pipeline compiled to iris_pipeline.yaml")

    # 2) Kubeflow에 직접 제출
    import kfp
    client = kfp.Client(host="http://localhost:8080/pipeline")

    run = client.create_run_from_pipeline_func(
        ml_pipeline,
        arguments={
            "dataset_url": "iris",
            "n_estimators": 200,
            "max_depth": 10,
            "accuracy_threshold": 0.92
        },
        experiment_name="iris-experiment",
        run_name="iris-run-001"
    )
    print(f"Run submitted: {run.run_id}")
```

## 3.3 커스텀 컨테이너 기반 Component

```python
# components/data_validation/component.py
"""
사전 빌드된 Docker 이미지를 사용하는 Component
대규모 의존성이 필요한 경우 사용
"""

from kfp import dsl
from kfp.dsl import Input, Output, Dataset, Artifact

@dsl.container_component
def great_expectations_validation(
    input_dataset: Input[Dataset],
    expectation_suite: str,
    validation_report: Output[Artifact]
):
    """Great Expectations로 데이터 품질 검증"""
    return dsl.ContainerSpec(
        image="my-registry.com/data-validation:v1.2.0",
        command=["python", "validate.py"],
        args=[
            "--input-path", input_dataset.path,
            "--suite", expectation_suite,
            "--output-path", validation_report.path
        ]
    )
```

```dockerfile
# components/data_validation/Dockerfile
FROM python:3.11-slim

RUN pip install great-expectations==0.18.0 pandas==2.1.0

COPY validate.py /app/validate.py
WORKDIR /app

ENTRYPOINT ["python", "validate.py"]
```

---

# Module 4. Kubeflow Pipelines 심화

## 4.1 조건부 실행 및 반복

```python
# 02_advanced_pipeline.py
"""
조건부 실행, 반복, 병렬 처리를 포함하는 심화 파이프라인
"""

from kfp import dsl, compiler
from kfp.dsl import Input, Output, Dataset, Model, Metrics

@dsl.component(base_image="python:3.11-slim")
def check_data_quality(
    input_dataset: Input[Dataset],
    metrics: Output[Metrics]
) -> float:
    """데이터 품질 점수를 반환"""
    import pandas as pd

    df = pd.read_csv(input_dataset.path)
    null_ratio = df.isnull().sum().sum() / (df.shape[0] * df.shape[1])
    duplicate_ratio = df.duplicated().sum() / len(df)

    quality_score = 1.0 - (null_ratio * 0.5 + duplicate_ratio * 0.5)

    metrics.log_metric("null_ratio", null_ratio)
    metrics.log_metric("duplicate_ratio", duplicate_ratio)
    metrics.log_metric("quality_score", quality_score)

    return quality_score


@dsl.component(base_image="python:3.11-slim")
def clean_data(input_dataset: Input[Dataset], output_dataset: Output[Dataset]):
    """결측치 및 중복 제거"""
    import pandas as pd
    df = pd.read_csv(input_dataset.path)
    df = df.dropna().drop_duplicates()
    df.to_csv(output_dataset.path, index=False)


@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["scikit-learn==1.3.0", "pandas==2.1.0", "joblib==1.3.0"]
)
def train_single_model(
    train_data: Input[Dataset],
    algorithm: str,
    model_output: Output[Model],
    metrics: Output[Metrics]
) -> float:
    """단일 알고리즘으로 학습"""
    import pandas as pd
    from sklearn.ensemble import (
        RandomForestClassifier, GradientBoostingClassifier
    )
    from sklearn.svm import SVC
    from sklearn.model_selection import cross_val_score
    import joblib

    df = pd.read_csv(train_data.path)
    X, y = df.iloc[:, :-1], df.iloc[:, -1]

    models = {
        "random_forest": RandomForestClassifier(n_estimators=100, random_state=42),
        "gradient_boosting": GradientBoostingClassifier(n_estimators=100, random_state=42),
        "svm": SVC(kernel="rbf", random_state=42),
    }

    model = models[algorithm]
    scores = cross_val_score(model, X, y, cv=5, scoring="accuracy")
    model.fit(X, y)

    mean_score = scores.mean()

    joblib.dump(model, model_output.path)
    model_output.metadata["algorithm"] = algorithm

    metrics.log_metric("cv_mean_accuracy", mean_score)
    metrics.log_metric("cv_std", scores.std())

    return mean_score


@dsl.component(base_image="python:3.11-slim")
def select_best_model(
    scores: list,
    algorithms: list
) -> str:
    """최고 성능 모델 선택"""
    best_idx = scores.index(max(scores))
    print(f"Best model: {algorithms[best_idx]} (score: {scores[best_idx]:.4f})")
    return algorithms[best_idx]


@dsl.component(base_image="python:3.11-slim")
def deploy_notification(model_name: str, score: float):
    """배포 알림 (Slack/이메일 등)"""
    print(f"🚀 Deploying {model_name} with score {score:.4f}")


@dsl.pipeline(name="advanced-ml-pipeline")
def advanced_pipeline(
    dataset_url: str = "iris",
    quality_threshold: float = 0.8,
    algorithms: list = ["random_forest", "gradient_boosting", "svm"]
):
    # Step 1: 데이터 로드
    load_task = load_data(dataset_url=dataset_url)

    # Step 2: 데이터 품질 검사
    quality_task = check_data_quality(
        input_dataset=load_task.outputs["output_dataset"]
    )

    # Step 3: 조건부 실행 - 품질이 낮으면 클리닝
    with dsl.If(
        quality_task.output < quality_threshold,
        name="data-quality-gate"
    ):
        clean_task = clean_data(
            input_dataset=load_task.outputs["output_dataset"]
        )

    # Step 4: 여러 알고리즘 병렬 학습 (ParallelFor)
    with dsl.ParallelFor(
        items=algorithms,
        parallelism=3
    ) as algorithm:
        train_task = train_single_model(
            train_data=load_task.outputs["output_dataset"],
            algorithm=algorithm
        )

    # Step 5: 최고 모델 선택
    select_task = select_best_model(
        scores=dsl.Collected(train_task.output),
        algorithms=algorithms
    )


# 컴파일
compiler.Compiler().compile(
    pipeline_func=advanced_pipeline,
    package_path="advanced_pipeline.yaml"
)
```

## 4.2 캐싱 및 재사용

```python
# 파이프라인 레벨 캐싱 설정
@dsl.pipeline(name="cached-pipeline")
def cached_pipeline(data_url: str = "iris"):
    load_task = load_data(dataset_url=data_url)
    # 캐싱 활성화 (기본값: True)
    # 동일 입력이면 이전 결과를 재사용
    load_task.set_caching_options(enable_caching=True)

    # 학습은 캐싱 비활성화 (매번 실행)
    train_task = train_model(
        train_dataset=load_task.outputs["output_dataset"],
        n_estimators=100,
        max_depth=5
    )
    train_task.set_caching_options(enable_caching=False)
```

## 4.3 Recurring Run (스케줄링)

```python
# 주기적 파이프라인 실행 설정
import kfp

client = kfp.Client(host="http://localhost:8080/pipeline")

# 매일 자정 UTC에 실행
recurring_run = client.create_recurring_run(
    experiment_id="exp-001",
    job_name="daily-retraining",
    pipeline_package_path="iris_pipeline.yaml",
    params={
        "dataset_url": "s3://my-bucket/daily-data/latest.csv",
        "accuracy_threshold": 0.92
    },
    cron_expression="0 0 * * *",  # 매일 00:00 UTC
    max_concurrency=1,
    enabled=True
)
```

## 4.4 파이프라인 구성 고급 패턴

```python
# 리소스 관리, 볼륨 마운트, 환경변수, 시크릿 등
@dsl.pipeline(name="production-pipeline")
def production_pipeline():
    train_task = train_model(...)

    # GPU 리소스 요청
    train_task.set_gpu_limit(2)
    train_task.set_cpu_limit("8")
    train_task.set_memory_limit("32Gi")

    # Node Selector로 특정 노드 그룹 지정
    train_task.add_node_selector_constraint(
        "node.kubernetes.io/instance-type", "p3.8xlarge"
    )

    # Tolerations (GPU 노드의 Taint 허용)
    train_task.add_toleration({
        "key": "nvidia.com/gpu",
        "operator": "Equal",
        "value": "true",
        "effect": "NoSchedule"
    })

    # 환경변수
    train_task.set_env_variable("CUDA_VISIBLE_DEVICES", "0,1")
    train_task.set_env_variable("NCCL_DEBUG", "INFO")

    # Kubernetes Secret에서 환경변수 주입
    from kubernetes.client import V1EnvVar, V1EnvVarSource, V1SecretKeySelector
    train_task.container.add_env_variable(
        V1EnvVar(
            name="WANDB_API_KEY",
            value_from=V1EnvVarSource(
                secret_key_ref=V1SecretKeySelector(
                    name="wandb-secret",
                    key="api-key"
                )
            )
        )
    )

    # PVC 마운트 (대용량 데이터셋)
    train_task.add_pvolumes({
        "/mnt/datasets": dsl.PipelineVolume(pvc="shared-datasets-pvc"),
        "/mnt/models": dsl.PipelineVolume(pvc="model-store-pvc")
    })

    # Retry 설정
    train_task.set_retry(
        num_retries=3,
        policy="Always",
        backoff_duration="30s",
        backoff_factor=2.0,
        backoff_max_duration="3600s"
    )

    # 타임아웃
    train_task.set_timeout(seconds=7200)  # 2시간
```

---

# Module 5. 핵심 컴포넌트 Deep Dive

## 5.1 Katib - 하이퍼파라미터 튜닝

Katib은 Kubernetes-native AutoML 시스템으로, 하이퍼파라미터 최적화(HPO)와 Neural Architecture Search(NAS)를 지원합니다.

```yaml
# katib-experiment.yaml
apiVersion: kubeflow.org/v1beta1
kind: Experiment
metadata:
  name: pytorch-hpo-experiment
  namespace: team-ml-engineering
spec:
  # 목표: validation accuracy 최대화
  objective:
    type: maximize
    goal: 0.98
    objectiveMetricName: val_accuracy
    additionalMetricNames:
      - train_loss
      - val_loss

  # 탐색 알고리즘
  algorithm:
    algorithmName: tpe  # Tree-structured Parzen Estimator
    # 대안: random, grid, bayesianoptimization, hyperband, enas, darts
    algorithmSettings:
      - name: "n_startup_trials"
        value: "10"

  # Early Stopping
  earlyStopping:
    algorithmName: medianstop
    algorithmSettings:
      - name: "min_trials_required"
        value: "3"
      - name: "start_step"
        value: "5"

  # 병렬 실행 설정
  parallelTrialCount: 4
  maxTrialCount: 50
  maxFailedTrialCount: 5

  # 탐색 공간 정의
  parameters:
    - name: learning_rate
      parameterType: double
      feasibleSpace:
        min: "0.0001"
        max: "0.1"
        step: ""  # 연속 값
    - name: batch_size
      parameterType: int
      feasibleSpace:
        min: "16"
        max: "256"
        step: "16"
    - name: num_layers
      parameterType: int
      feasibleSpace:
        min: "2"
        max: "8"
    - name: optimizer
      parameterType: categorical
      feasibleSpace:
        list:
          - adam
          - sgd
          - adamw
    - name: dropout_rate
      parameterType: double
      feasibleSpace:
        min: "0.1"
        max: "0.5"

  # Trial 템플릿 (각 시행의 학습 작업 정의)
  trialTemplate:
    primaryContainerName: training-container
    trialParameters:
      - name: learningRate
        description: "Learning rate"
        reference: learning_rate
      - name: batchSize
        description: "Batch size"
        reference: batch_size
      - name: numLayers
        description: "Number of layers"
        reference: num_layers
      - name: optimizer
        description: "Optimizer"
        reference: optimizer
      - name: dropoutRate
        description: "Dropout rate"
        reference: dropout_rate
    trialSpec:
      apiVersion: batch/v1
      kind: Job
      spec:
        template:
          metadata:
            annotations:
              sidecar.istio.io/inject: "false"
          spec:
            containers:
              - name: training-container
                image: my-registry.com/pytorch-trainer:v1.0
                command:
                  - "python"
                  - "/app/train.py"
                  - "--lr=${trialParameters.learningRate}"
                  - "--batch-size=${trialParameters.batchSize}"
                  - "--num-layers=${trialParameters.numLayers}"
                  - "--optimizer=${trialParameters.optimizer}"
                  - "--dropout=${trialParameters.dropoutRate}"
                  - "--epochs=50"
                resources:
                  limits:
                    nvidia.com/gpu: 1
                    memory: "16Gi"
                    cpu: "4"
            restartPolicy: Never
```

```python
# katib_trainer.py - Katib Trial에서 실행되는 학습 스크립트
"""
주의: Katib은 stdout에서 메트릭을 파싱하므로
정해진 형식으로 출력해야 합니다.
"""

import argparse
import torch
import torch.nn as nn

def build_model(num_layers, dropout_rate):
    layers = []
    in_dim = 784
    for i in range(num_layers):
        out_dim = max(64, 784 // (2 ** (i + 1)))
        layers.extend([
            nn.Linear(in_dim, out_dim),
            nn.ReLU(),
            nn.Dropout(dropout_rate)
        ])
        in_dim = out_dim
    layers.append(nn.Linear(in_dim, 10))
    return nn.Sequential(*layers)

def train(args):
    model = build_model(args.num_layers, args.dropout)
    optimizer_cls = {
        "adam": torch.optim.Adam,
        "sgd": torch.optim.SGD,
        "adamw": torch.optim.AdamW
    }[args.optimizer]
    optimizer = optimizer_cls(model.parameters(), lr=args.lr)

    for epoch in range(args.epochs):
        # ... 학습 루프 ...
        train_loss = 0.1  # 예시
        val_accuracy = 0.95  # 예시

        # ⚠️ Katib 메트릭 수집기가 인식하는 형식으로 출력
        # StdOut 메트릭 수집기 기본 형식
        print(f"epoch={epoch+1}")
        print(f"train_loss={train_loss:.6f}")
        print(f"val_accuracy={val_accuracy:.6f}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--lr", type=float, default=0.001)
    parser.add_argument("--batch-size", type=int, default=32)
    parser.add_argument("--num-layers", type=int, default=3)
    parser.add_argument("--optimizer", type=str, default="adam")
    parser.add_argument("--dropout", type=float, default=0.3)
    parser.add_argument("--epochs", type=int, default=50)
    train(parser.parse_args())
```

## 5.2 Training Operators - 분산 학습

```yaml
# pytorch-distributed-training.yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: bert-finetuning
  namespace: team-ml-engineering
spec:
  elasticPolicy:
    rdzvBackend: c10d
    minReplicas: 2
    maxReplicas: 8
    maxRestarts: 3
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 80
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers:
            - name: pytorch
              image: my-registry.com/bert-trainer:v2.1
              command:
                - torchrun
                - --nproc_per_node=2
                - --nnodes=$(WORLD_SIZE)
                - --node_rank=$(RANK)
                - --master_addr=$(MASTER_ADDR)
                - --master_port=$(MASTER_PORT)
                - train_bert.py
                - --model_name=bert-base-uncased
                - --dataset=squad
                - --epochs=3
                - --per_device_batch_size=16
              resources:
                limits:
                  nvidia.com/gpu: 2
                  memory: "64Gi"
                  cpu: "16"
              volumeMounts:
                - name: dshm
                  mountPath: /dev/shm
                - name: dataset
                  mountPath: /data
          volumes:
            - name: dshm
              emptyDir:
                medium: Memory
                sizeLimit: "16Gi"
            - name: dataset
              persistentVolumeClaim:
                claimName: training-datasets
          tolerations:
            - key: "nvidia.com/gpu"
              operator: "Exists"
              effect: "NoSchedule"
    Worker:
      replicas: 3
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: my-registry.com/bert-trainer:v2.1
              command:
                - torchrun
                - --nproc_per_node=2
                - --nnodes=$(WORLD_SIZE)
                - --node_rank=$(RANK)
                - --master_addr=$(MASTER_ADDR)
                - --master_port=$(MASTER_PORT)
                - train_bert.py
                - --model_name=bert-base-uncased
                - --dataset=squad
                - --epochs=3
                - --per_device_batch_size=16
              resources:
                limits:
                  nvidia.com/gpu: 2
                  memory: "64Gi"
                  cpu: "16"
              volumeMounts:
                - name: dshm
                  mountPath: /dev/shm
          volumes:
            - name: dshm
              emptyDir:
                medium: Memory
                sizeLimit: "16Gi"
          tolerations:
            - key: "nvidia.com/gpu"
              operator: "Exists"
              effect: "NoSchedule"
```

## 5.3 KServe - 모델 서빙

```yaml
# kserve-inference-service.yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-iris
  namespace: team-ml-engineering
  annotations:
    # Canary 배포: 신규 버전에 20% 트래픽
    serving.kserve.io/canaryTrafficPercent: "20"
spec:
  predictor:
    # 오토스케일링 설정
    minReplicas: 1
    maxReplicas: 10
    scaleTarget: 10      # 동시 요청 수 기준
    scaleMetric: concurrency

    # 모델 서버 (sklearn 기본 지원)
    sklearn:
      storageUri: "s3://model-store/sklearn/iris/v2"
      protocolVersion: v2
      resources:
        requests:
          cpu: "500m"
          memory: "1Gi"
        limits:
          cpu: "2"
          memory: "4Gi"

  # 전처리 Transformer
  transformer:
    containers:
      - name: feature-transformer
        image: my-registry.com/iris-transformer:v1.0
        resources:
          requests:
            cpu: "200m"
            memory: "512Mi"

  # 후처리 Explainer (SHAP/LIME)
  explainer:
    alibi:
      type: AnchorTabular
      storageUri: "s3://model-store/explainers/iris-anchor"
      resources:
        requests:
          cpu: "500m"
          memory: "1Gi"
---
# GPU 기반 PyTorch 모델 서빙
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: bert-qa-model
  namespace: team-ml-engineering
spec:
  predictor:
    minReplicas: 2
    maxReplicas: 20
    pytorch:
      storageUri: "s3://model-store/pytorch/bert-qa/v1"
      protocolVersion: v2
      resources:
        limits:
          nvidia.com/gpu: 1
          memory: "16Gi"
          cpu: "4"
    # 배치 추론 설정
    batcher:
      maxBatchSize: 32
      maxLatency: 500  # ms
      timeout: 60
```

```python
# kserve_client.py - KServe 추론 요청 예제
import requests
import json

# REST API 추론
KSERVE_URL = "http://sklearn-iris.team-ml-engineering.svc.cluster.local"

# v2 프로토콜
payload = {
    "inputs": [
        {
            "name": "input-0",
            "shape": [1, 4],
            "datatype": "FP32",
            "data": [[5.1, 3.5, 1.4, 0.2]]
        }
    ]
}

response = requests.post(
    f"{KSERVE_URL}/v2/models/sklearn-iris/infer",
    json=payload
)
result = response.json()
print(f"Prediction: {result['outputs'][0]['data']}")

# 설명 요청 (Explainer)
explain_response = requests.post(
    f"{KSERVE_URL}/v1/models/sklearn-iris:explain",
    json={"instances": [[5.1, 3.5, 1.4, 0.2]]}
)
```

---

# Module 6. 고급 기법 및 최적화

## 6.1 파이프라인에서 LLM Fine-tuning

```python
# 03_llm_finetuning_pipeline.py
"""
LLM Fine-tuning 파이프라인
QLoRA를 사용한 효율적 파인튜닝
"""

from kfp import dsl, compiler
from kfp.dsl import Input, Output, Dataset, Model, Artifact

@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["datasets==2.14.0", "huggingface-hub==0.17.0"]
)
def prepare_instruction_dataset(
    dataset_name: str,
    max_samples: int,
    output_dataset: Output[Dataset]
):
    """HuggingFace 데이터셋을 Instruction 형식으로 변환"""
    from datasets import load_dataset
    import json

    ds = load_dataset(dataset_name, split=f"train[:{max_samples}]")

    formatted = []
    for row in ds:
        formatted.append({
            "instruction": row.get("instruction", ""),
            "input": row.get("input", ""),
            "output": row.get("output", ""),
        })

    with open(output_dataset.path, "w") as f:
        json.dump(formatted, f)

    output_dataset.metadata["num_samples"] = len(formatted)


@dsl.container_component
def qlora_finetune(
    train_dataset: Input[Dataset],
    base_model: str,
    lora_r: int,
    lora_alpha: int,
    learning_rate: float,
    num_epochs: int,
    output_model: Output[Model],
    training_logs: Output[Artifact]
):
    """QLoRA Fine-tuning (사전 빌드 이미지 필요)"""
    return dsl.ContainerSpec(
        image="my-registry.com/llm-trainer:v1.0-cuda12",
        command=["python", "/app/qlora_train.py"],
        args=[
            "--dataset-path", train_dataset.path,
            "--base-model", base_model,
            "--lora-r", str(lora_r),
            "--lora-alpha", str(lora_alpha),
            "--lr", str(learning_rate),
            "--epochs", str(num_epochs),
            "--output-dir", output_model.path,
            "--log-dir", training_logs.path,
            "--bf16",
            "--gradient-checkpointing",
            "--per-device-batch-size", "4",
            "--gradient-accumulation-steps", "4",
        ]
    )


@dsl.container_component
def merge_lora_weights(
    base_model: str,
    lora_model: Input[Model],
    merged_model: Output[Model]
):
    """LoRA 어댑터를 베이스 모델에 병합"""
    return dsl.ContainerSpec(
        image="my-registry.com/llm-trainer:v1.0-cuda12",
        command=["python", "/app/merge_lora.py"],
        args=[
            "--base-model", base_model,
            "--lora-path", lora_model.path,
            "--output-path", merged_model.path
        ]
    )


@dsl.container_component
def evaluate_llm(
    model_path: Input[Model],
    eval_dataset: Input[Dataset],
    eval_results: Output[Artifact]
):
    """LLM 평가 (perplexity, BLEU, ROUGE 등)"""
    return dsl.ContainerSpec(
        image="my-registry.com/llm-evaluator:v1.0",
        command=["python", "/app/evaluate.py"],
        args=[
            "--model-path", model_path.path,
            "--eval-data", eval_dataset.path,
            "--output", eval_results.path
        ]
    )


@dsl.container_component
def deploy_vllm_serving(
    model_path: Input[Model],
    model_name: str,
    gpu_memory_utilization: float
):
    """vLLM으로 모델 서빙 배포"""
    return dsl.ContainerSpec(
        image="my-registry.com/vllm-deployer:v1.0",
        command=["python", "/app/deploy.py"],
        args=[
            "--model-path", model_path.path,
            "--model-name", model_name,
            "--gpu-memory-utilization", str(gpu_memory_utilization),
            "--kserve-namespace", "serving"
        ]
    )


@dsl.pipeline(
    name="llm-finetuning-pipeline",
    description="QLoRA 기반 LLM Fine-tuning End-to-End 파이프라인"
)
def llm_finetuning_pipeline(
    dataset_name: str = "tatsu-lab/alpaca",
    max_samples: int = 10000,
    base_model: str = "meta-llama/Llama-2-7b-hf",
    lora_r: int = 16,
    lora_alpha: int = 32,
    learning_rate: float = 2e-4,
    num_epochs: int = 3,
    deploy_model: bool = True
):
    # Step 1: 데이터 준비
    data_task = prepare_instruction_dataset(
        dataset_name=dataset_name,
        max_samples=max_samples
    )

    # Step 2: QLoRA Fine-tuning
    train_task = qlora_finetune(
        train_dataset=data_task.outputs["output_dataset"],
        base_model=base_model,
        lora_r=lora_r,
        lora_alpha=lora_alpha,
        learning_rate=learning_rate,
        num_epochs=num_epochs
    )
    train_task.set_gpu_limit(2)
    train_task.set_memory_limit("64Gi")
    train_task.set_cpu_limit("16")
    train_task.add_node_selector_constraint(
        "node.kubernetes.io/instance-type", "g5.12xlarge"
    )

    # Step 3: LoRA 어댑터 병합
    merge_task = merge_lora_weights(
        base_model=base_model,
        lora_model=train_task.outputs["output_model"]
    )
    merge_task.set_gpu_limit(1)
    merge_task.set_memory_limit("32Gi")

    # Step 4: 평가
    eval_task = evaluate_llm(
        model_path=merge_task.outputs["merged_model"],
        eval_dataset=data_task.outputs["output_dataset"]
    )
    eval_task.set_gpu_limit(1)

    # Step 5: 조건부 배포
    with dsl.If(deploy_model == True, name="deploy-gate"):
        deploy_vllm_serving(
            model_path=merge_task.outputs["merged_model"],
            model_name="llama2-7b-custom",
            gpu_memory_utilization=0.9
        )


compiler.Compiler().compile(
    pipeline_func=llm_finetuning_pipeline,
    package_path="llm_finetuning_pipeline.yaml"
)
```

## 6.2 모니터링 및 Observability

```yaml
# prometheus-servicemonitor.yaml
# Kubeflow 메트릭 수집 설정
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kubeflow-pipelines-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: ml-pipeline
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
  namespaceSelector:
    matchNames:
      - kubeflow
---
# KServe 추론 메트릭
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kserve-inference-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      serving.kserve.io/inferenceservice: "*"
  endpoints:
    - port: http-usermetric
      path: /metrics
      interval: 15s
```

```python
# model_monitoring.py
"""
모델 드리프트 감지 파이프라인 컴포넌트
"""

from kfp import dsl
from kfp.dsl import Input, Output, Dataset, Artifact, Metrics

@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "evidently==0.4.0",
        "pandas==2.1.0"
    ]
)
def detect_data_drift(
    reference_data: Input[Dataset],
    production_data: Input[Dataset],
    drift_report: Output[Artifact],
    metrics: Output[Metrics],
    drift_threshold: float = 0.05
) -> bool:
    """Evidently로 데이터 드리프트 감지"""
    import pandas as pd
    from evidently.report import Report
    from evidently.metric_preset import DataDriftPreset
    import json

    ref_df = pd.read_csv(reference_data.path)
    prod_df = pd.read_csv(production_data.path)

    report = Report(metrics=[DataDriftPreset()])
    report.run(reference_data=ref_df, current_data=prod_df)

    # 리포트 저장
    report_dict = report.as_dict()
    with open(drift_report.path, "w") as f:
        json.dump(report_dict, f)

    # 드리프트 메트릭 로깅
    drift_result = report_dict["metrics"][0]["result"]
    drift_share = drift_result["share_of_drifted_columns"]

    metrics.log_metric("drift_share", drift_share)
    metrics.log_metric("num_drifted_columns", drift_result["number_of_drifted_columns"])

    is_drifted = drift_share > drift_threshold
    if is_drifted:
        print(f"⚠️ Data drift detected! {drift_share:.2%} columns drifted.")
    return is_drifted
```

## 6.3 비용 최적화 전략

```yaml
# spot-instance-training.yaml
# Spot/Preemptible 인스턴스로 학습 비용 절감
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: cost-optimized-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          # Master는 On-Demand (안정성 보장)
          nodeSelector:
            node.kubernetes.io/lifecycle: on-demand
          containers:
            - name: pytorch
              image: my-registry.com/trainer:v1
              resources:
                limits:
                  nvidia.com/gpu: 1
    Worker:
      replicas: 4
      template:
        spec:
          # Worker는 Spot (비용 절감)
          nodeSelector:
            node.kubernetes.io/lifecycle: spot
          tolerations:
            - key: "kubernetes.io/spot"
              operator: "Exists"
              effect: "NoSchedule"
          containers:
            - name: pytorch
              image: my-registry.com/trainer:v1
              resources:
                limits:
                  nvidia.com/gpu: 1
          # Spot 중단 시 Graceful Shutdown
          terminationGracePeriodSeconds: 120
---
# Karpenter Provisioner (AWS) - GPU 노드 자동 프로비저닝
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: gpu-training
spec:
  requirements:
    - key: "karpenter.sh/capacity-type"
      operator: In
      values: ["spot", "on-demand"]
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["g5.xlarge", "g5.2xlarge", "g5.4xlarge", "p3.2xlarge"]
  limits:
    resources:
      nvidia.com/gpu: "32"
  ttlSecondsAfterEmpty: 300  # 5분 유휴 후 노드 제거
  providerRef:
    name: default
```

---

# Module 7. 다른 MLOps 도구와의 연계

## 7.1 아키텍처 개요: Kubeflow 중심 MLOps 생태계

```
┌─────────────────────────────────────────────────────────────┐
│                    MLOps Platform Architecture               │
│                                                             │
│  ┌─── Data Layer ───┐  ┌── Experiment ──┐  ┌── Deploy ──┐  │
│  │                   │  │    Tracking    │  │            │  │
│  │  Apache Airflow   │  │               │  │  ArgoCD    │  │
│  │  (데이터 파이프라인)│  │  MLflow /     │  │  (GitOps)  │  │
│  │       │           │  │  W&B          │  │     │      │  │
│  │       ▼           │  │     ▲         │  │     ▼      │  │
│  │  Feast            │  │     │         │  │  KServe    │  │
│  │  (Feature Store)  │  │     │         │  │  /vLLM     │  │
│  │       │           │  │     │         │  │     │      │  │
│  └───────┼───────────┘  └─────┼─────────┘  └─────┼──────┘  │
│          │                    │                    │         │
│  ┌───────▼────────────────────▼────────────────────▼──────┐ │
│  │              Kubeflow Pipelines (오케스트레이터)         │ │
│  │   데이터 수집 → 전처리 → 학습 → 평가 → 등록 → 배포     │ │
│  └──────────────────────┬─────────────────────────────────┘ │
│                         │                                   │
│  ┌──────────────────────▼─────────────────────────────────┐ │
│  │              Kubernetes + Monitoring                    │ │
│  │   Prometheus │ Grafana │ Seldon/Evidently │ Jaeger      │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 7.2 MLflow 연계

```python
# 04_kfp_mlflow_integration.py
"""
Kubeflow Pipelines + MLflow 통합
MLflow로 실험 추적, 모델 레지스트리 활용
"""

from kfp import dsl
from kfp.dsl import Input, Output, Dataset, Model

@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "mlflow==2.8.0",
        "scikit-learn==1.3.0",
        "pandas==2.1.0",
        "boto3==1.29.0"
    ]
)
def train_with_mlflow(
    train_dataset: Input[Dataset],
    mlflow_tracking_uri: str,
    experiment_name: str,
    n_estimators: int,
    max_depth: int,
    model_output: Output[Model]
) -> str:
    """MLflow 추적과 함께 모델 학습"""
    import mlflow
    import mlflow.sklearn
    import pandas as pd
    from sklearn.ensemble import RandomForestClassifier
    from sklearn.model_selection import cross_val_score
    import joblib
    import json

    # MLflow 서버 연결
    mlflow.set_tracking_uri(mlflow_tracking_uri)
    mlflow.set_experiment(experiment_name)

    df = pd.read_csv(train_dataset.path)
    X, y = df.iloc[:, :-1], df.iloc[:, -1]

    with mlflow.start_run() as run:
        # 하이퍼파라미터 로깅
        mlflow.log_params({
            "n_estimators": n_estimators,
            "max_depth": max_depth,
            "algorithm": "RandomForestClassifier"
        })

        # 학습
        model = RandomForestClassifier(
            n_estimators=n_estimators,
            max_depth=max_depth,
            random_state=42
        )

        scores = cross_val_score(model, X, y, cv=5)
        model.fit(X, y)

        # 메트릭 로깅
        mlflow.log_metrics({
            "cv_mean_accuracy": scores.mean(),
            "cv_std_accuracy": scores.std()
        })

        # 모델 로깅 및 레지스트리 등록
        mlflow.sklearn.log_model(
            model,
            artifact_path="model",
            registered_model_name="iris-classifier"
        )

        # 로컬에도 저장 (KFP 아티팩트)
        joblib.dump(model, model_output.path)

        run_id = run.info.run_id
        print(f"MLflow Run ID: {run_id}")
        return run_id


@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["mlflow==2.8.0", "boto3==1.29.0"]
)
def promote_model(
    mlflow_tracking_uri: str,
    model_name: str,
    run_id: str,
    accuracy_threshold: float
) -> str:
    """MLflow Model Registry에서 모델 승격"""
    import mlflow
    from mlflow.tracking import MlflowClient

    mlflow.set_tracking_uri(mlflow_tracking_uri)
    client = MlflowClient()

    # 최신 버전 가져오기
    versions = client.search_model_versions(f"name='{model_name}'")
    latest = max(versions, key=lambda v: int(v.version))

    # 메트릭 확인
    run = client.get_run(run_id)
    accuracy = run.data.metrics.get("cv_mean_accuracy", 0)

    if accuracy >= accuracy_threshold:
        # Production 단계로 승격
        client.transition_model_version_stage(
            name=model_name,
            version=latest.version,
            stage="Production",
            archive_existing_versions=True
        )
        return f"Model v{latest.version} promoted to Production"
    else:
        client.transition_model_version_stage(
            name=model_name,
            version=latest.version,
            stage="Staging"
        )
        return f"Model v{latest.version} moved to Staging (accuracy: {accuracy:.4f})"
```

```yaml
# mlflow-deployment.yaml (Kubernetes에 MLflow 서버 배포)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-server
  namespace: mlops
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mlflow
  template:
    metadata:
      labels:
        app: mlflow
    spec:
      containers:
        - name: mlflow
          image: ghcr.io/mlflow/mlflow:2.8.0
          command:
            - mlflow
            - server
            - --backend-store-uri=mysql://mlflow:$(DB_PASSWORD)@mysql:3306/mlflow
            - --default-artifact-root=s3://mlflow-artifacts/
            - --host=0.0.0.0
            - --port=5000
          ports:
            - containerPort: 5000
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mlflow-secrets
                  key: db-password
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: access-key
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secret-key
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow
  namespace: mlops
spec:
  selector:
    app: mlflow
  ports:
    - port: 5000
      targetPort: 5000
```

## 7.3 Weights & Biases (W&B) 연계

```python
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["wandb==0.16.0", "torch==2.1.0"]
)
def train_with_wandb(
    config_json: str,
    wandb_project: str
):
    """W&B 추적과 함께 PyTorch 학습"""
    import wandb
    import torch
    import json
    import os

    config = json.loads(config_json)

    # W&B 초기화 (API Key는 K8s Secret으로 주입)
    wandb.init(
        project=wandb_project,
        config=config,
        tags=["kubeflow", "production"]
    )

    # 학습 루프
    model = build_model(config)
    optimizer = torch.optim.Adam(model.parameters(), lr=config["lr"])

    for epoch in range(config["epochs"]):
        train_loss = train_one_epoch(model, optimizer)
        val_loss, val_acc = validate(model)

        # W&B 로깅
        wandb.log({
            "epoch": epoch,
            "train_loss": train_loss,
            "val_loss": val_loss,
            "val_accuracy": val_acc,
            "learning_rate": optimizer.param_groups[0]["lr"]
        })

        # 모델 체크포인트를 W&B Artifact로 저장
        if epoch % 5 == 0:
            artifact = wandb.Artifact(
                f"model-checkpoint-{epoch}",
                type="model"
            )
            torch.save(model.state_dict(), f"/tmp/model_epoch_{epoch}.pt")
            artifact.add_file(f"/tmp/model_epoch_{epoch}.pt")
            wandb.log_artifact(artifact)

    wandb.finish()
```

## 7.4 Feast (Feature Store) 연계

```python
# 05_feast_integration.py
"""
Kubeflow + Feast Feature Store 통합
"""

from kfp import dsl
from kfp.dsl import Output, Dataset

@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["feast==0.34.0", "pandas==2.1.0"]
)
def materialize_features(
    feast_repo_path: str,
    start_date: str,
    end_date: str
):
    """Feast Feature Store 물리화 (오프라인 → 온라인)"""
    from feast import FeatureStore
    from datetime import datetime

    store = FeatureStore(repo_path=feast_repo_path)

    store.materialize(
        start_date=datetime.fromisoformat(start_date),
        end_date=datetime.fromisoformat(end_date)
    )
    print("Feature materialization complete")


@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["feast==0.34.0", "pandas==2.1.0"]
)
def get_training_features(
    feast_repo_path: str,
    entity_csv_path: str,
    feature_service_name: str,
    output_dataset: Output[Dataset]
):
    """Feast에서 학습용 피처 데이터셋 생성"""
    from feast import FeatureStore
    import pandas as pd

    store = FeatureStore(repo_path=feast_repo_path)

    # 엔터티 DataFrame (학습 대상 + timestamp)
    entity_df = pd.read_csv(entity_csv_path)
    entity_df["event_timestamp"] = pd.to_datetime(entity_df["event_timestamp"])

    # Feature Service로 피처 조회
    training_df = store.get_historical_features(
        entity_df=entity_df,
        features=store.get_feature_service(feature_service_name)
    ).to_df()

    training_df.to_csv(output_dataset.path, index=False)
    output_dataset.metadata["num_features"] = len(training_df.columns)
    output_dataset.metadata["num_rows"] = len(training_df)
```

```python
# feast_repo/feature_store.yaml
project: ml_platform
registry: s3://feast-registry/registry.db
provider: aws
online_store:
  type: redis
  connection_string: redis://redis-cluster.mlops.svc:6379
offline_store:
  type: redshift
  cluster_id: feast-offline
  region: ap-northeast-2
  database: feast
  user: feast_admin
```

## 7.5 Apache Airflow 연계

```python
# airflow_dag.py
"""
Airflow에서 Kubeflow Pipeline을 트리거하는 DAG
데이터 파이프라인(Airflow) → ML 파이프라인(Kubeflow)
"""

from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import (
    KubernetesPodOperator
)
from datetime import datetime, timedelta
import kfp

default_args = {
    "owner": "ml-team",
    "depends_on_past": False,
    "start_date": datetime(2024, 1, 1),
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
}

dag = DAG(
    "ml_retraining_pipeline",
    default_args=default_args,
    schedule_interval="0 2 * * *",  # 매일 02:00 UTC
    catchup=False,
    tags=["ml", "retraining"]
)


def trigger_kubeflow_pipeline(**context):
    """Kubeflow Pipeline 실행을 트리거"""
    client = kfp.Client(
        host="http://ml-pipeline.kubeflow.svc.cluster.local:8888"
    )

    execution_date = context["ds"]

    run = client.create_run_from_pipeline_package(
        pipeline_file="/opt/airflow/pipelines/ml_pipeline.yaml",
        arguments={
            "dataset_date": execution_date,
            "dataset_url": f"s3://data-lake/processed/{execution_date}/",
            "accuracy_threshold": "0.92"
        },
        run_name=f"retraining-{execution_date}",
        experiment_name="daily-retraining"
    )

    # Run ID를 XCom에 저장
    context["ti"].xcom_push(key="kfp_run_id", value=run.run_id)
    return run.run_id


def wait_for_pipeline(**context):
    """Kubeflow Pipeline 완료 대기"""
    client = kfp.Client(
        host="http://ml-pipeline.kubeflow.svc.cluster.local:8888"
    )
    run_id = context["ti"].xcom_pull(key="kfp_run_id")

    result = client.wait_for_run_completion(run_id, timeout=7200)

    if result.run.status != "Succeeded":
        raise Exception(f"Pipeline failed: {result.run.status}")
    return "Pipeline completed successfully"


# 데이터 품질 검사 (Airflow에서)
data_quality = KubernetesPodOperator(
    task_id="data_quality_check",
    name="data-quality",
    namespace="airflow",
    image="my-registry.com/data-quality:v1",
    cmds=["python", "check.py"],
    arguments=["--date={{ ds }}"],
    dag=dag
)

# Kubeflow Pipeline 트리거
trigger_kfp = PythonOperator(
    task_id="trigger_kubeflow",
    python_callable=trigger_kubeflow_pipeline,
    dag=dag
)

# Pipeline 완료 대기
wait_kfp = PythonOperator(
    task_id="wait_for_kubeflow",
    python_callable=wait_for_pipeline,
    dag=dag
)

# 알림
notify = PythonOperator(
    task_id="send_notification",
    python_callable=lambda: print("Training complete!"),
    dag=dag
)

data_quality >> trigger_kfp >> wait_kfp >> notify
```

## 7.6 ArgoCD를 통한 GitOps 배포

```yaml
# argocd-application.yaml
# KServe InferenceService를 GitOps로 관리
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ml-model-serving
  namespace: argocd
spec:
  project: ml-platform
  source:
    repoURL: https://github.com/company/ml-model-configs.git
    targetRevision: main
    path: serving/production

  destination:
    server: https://kubernetes.default.svc
    namespace: serving

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```python
# 파이프라인에서 Git Push → ArgoCD 자동 배포 트리거
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["gitpython==3.1.40", "pyyaml==6.0"]
)
def update_serving_config(
    model_name: str,
    model_version: str,
    model_uri: str,
    git_repo: str,
    git_branch: str
):
    """모델 서빙 설정을 Git에 업데이트 → ArgoCD가 감지하여 배포"""
    import git
    import yaml
    import os

    repo_dir = "/tmp/ml-configs"
    repo = git.Repo.clone_from(git_repo, repo_dir, branch=git_branch)

    # InferenceService YAML 업데이트
    serving_path = f"{repo_dir}/serving/production/{model_name}.yaml"

    with open(serving_path) as f:
        config = yaml.safe_load(f)

    config["spec"]["predictor"]["sklearn"]["storageUri"] = model_uri
    config["metadata"]["labels"]["model-version"] = model_version

    with open(serving_path, "w") as f:
        yaml.dump(config, f, default_flow_style=False)

    # Git Commit & Push
    repo.index.add([serving_path])
    repo.index.commit(
        f"chore: update {model_name} to version {model_version}"
    )
    repo.remote("origin").push()
    print(f"Pushed serving config update for {model_name}:{model_version}")
```

## 7.7 Seldon Core 연계 (대안 서빙)

```yaml
# seldon-deployment.yaml
# KServe 대신 Seldon Core를 사용하는 경우
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: iris-model
  namespace: serving
spec:
  predictors:
    - name: default
      replicas: 2
      graph:
        name: classifier
        implementation: SKLEARN_SERVER
        modelUri: s3://models/iris/v2
        children: []
      componentSpecs:
        - spec:
            containers:
              - name: classifier
                resources:
                  requests:
                    cpu: "500m"
                    memory: "1Gi"
      traffic: 80
      # Shadow 배포 (신규 버전 테스트)
    - name: canary
      replicas: 1
      graph:
        name: classifier
        implementation: SKLEARN_SERVER
        modelUri: s3://models/iris/v3
      traffic: 20
```

## 7.8 DVC (Data Version Control) 연계

```python
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["dvc[s3]==3.30.0"]
)
def pull_versioned_data(
    dvc_repo: str,
    data_version: str,
    output_dataset: Output[Dataset]
):
    """DVC로 버전 관리된 데이터를 가져오기"""
    import subprocess
    import shutil

    # DVC repo 클론 및 특정 버전 체크아웃
    subprocess.run(["git", "clone", dvc_repo, "/tmp/data-repo"], check=True)
    subprocess.run(
        ["git", "checkout", data_version],
        cwd="/tmp/data-repo", check=True
    )
    subprocess.run(
        ["dvc", "pull"],
        cwd="/tmp/data-repo", check=True
    )

    # 데이터 복사
    shutil.copy("/tmp/data-repo/data/processed/train.csv", output_dataset.path)
    output_dataset.metadata["dvc_version"] = data_version
```

---

# Module 8. 실무 프로젝트: End-to-End ML 시스템

## 8.1 프로젝트 구조

```
ml-platform/
├── infrastructure/
│   ├── terraform/
│   │   ├── main.tf              # EKS/GKE 클러스터
│   │   ├── kubeflow.tf          # Kubeflow 설치
│   │   ├── monitoring.tf        # Prometheus/Grafana
│   │   └── variables.tf
│   └── kubernetes/
│       ├── namespaces.yaml
│       ├── resource-quotas.yaml
│       └── network-policies.yaml
│
├── components/                   # 재사용 가능한 KFP Components
│   ├── data_ingestion/
│   │   ├── component.py
│   │   ├── Dockerfile
│   │   └── tests/
│   ├── preprocessing/
│   ├── training/
│   ├── evaluation/
│   ├── model_registry/
│   └── serving/
│
├── pipelines/                    # 파이프라인 정의
│   ├── training_pipeline.py
│   ├── inference_pipeline.py
│   ├── monitoring_pipeline.py
│   └── retraining_pipeline.py
│
├── serving/                      # 모델 서빙 설정 (GitOps)
│   ├── production/
│   │   ├── model-a.yaml
│   │   └── model-b.yaml
│   └── staging/
│
├── monitoring/
│   ├── grafana-dashboards/
│   ├── alerting-rules.yaml
│   └── drift-detection/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── ci-cd/
│   ├── .github/workflows/
│   │   ├── component-build.yaml
│   │   ├── pipeline-test.yaml
│   │   └── model-deploy.yaml
│   └── argocd/
│       └── applications.yaml
│
├── Makefile
├── pyproject.toml
└── README.md
```

## 8.2 CI/CD 파이프라인

```yaml
# .github/workflows/ml-pipeline-ci.yaml
name: ML Pipeline CI/CD

on:
  push:
    branches: [main, develop]
    paths:
      - 'components/**'
      - 'pipelines/**'
  pull_request:
    branches: [main]

env:
  REGISTRY: ${{ secrets.ECR_REGISTRY }}
  KFP_HOST: ${{ secrets.KFP_ENDPOINT }}

jobs:
  # 1. Component 이미지 빌드
  build-components:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        component:
          - data_ingestion
          - preprocessing
          - training
          - evaluation
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-2

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push component image
        run: |
          IMAGE_TAG="${REGISTRY}/${{ matrix.component }}:${GITHUB_SHA::8}"
          docker build -t ${IMAGE_TAG} components/${{ matrix.component }}/
          docker push ${IMAGE_TAG}

  # 2. 파이프라인 테스트
  test-pipeline:
    needs: build-components
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install kfp==2.4.0 pytest

      - name: Compile pipeline
        run: |
          python pipelines/training_pipeline.py
          # 컴파일 성공 여부 확인
          test -f training_pipeline.yaml

      - name: Validate pipeline YAML
        run: python -c "
          import yaml
          with open('training_pipeline.yaml') as f:
              spec = yaml.safe_load(f)
          assert 'pipelineSpec' in spec
          print('Pipeline validation passed')
          "

  # 3. 스테이징 배포
  deploy-staging:
    needs: test-pipeline
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          pip install kfp==2.4.0
          python -c "
          import kfp
          client = kfp.Client(host='${KFP_HOST}')
          client.upload_pipeline(
              pipeline_package_path='training_pipeline.yaml',
              pipeline_name='training-pipeline-staging',
              description='Staging: ${GITHUB_SHA::8}'
          )
          "

  # 4. 프로덕션 배포
  deploy-production:
    needs: test-pipeline
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # GitHub Environment 승인 필요
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to production
        run: |
          pip install kfp==2.4.0
          python -c "
          import kfp
          client = kfp.Client(host='${KFP_HOST}')
          pipeline = client.upload_pipeline(
              pipeline_package_path='training_pipeline.yaml',
              pipeline_name='training-pipeline-prod',
              description='Production: ${GITHUB_SHA::8}'
          )
          # 자동 실행 (스모크 테스트)
          client.run_pipeline(
              experiment_id='prod-smoke-test',
              job_name=f'smoke-test-${GITHUB_SHA::8}',
              pipeline_id=pipeline.id
          )
          "
```

## 8.3 Grafana 대시보드 설정

```json
{
  "dashboard": {
    "title": "ML Platform Dashboard",
    "panels": [
      {
        "title": "Pipeline Run Status",
        "type": "stat",
        "targets": [{
          "expr": "sum by (status) (kfp_pipeline_runs_total)"
        }]
      },
      {
        "title": "Model Inference Latency (p99)",
        "type": "timeseries",
        "targets": [{
          "expr": "histogram_quantile(0.99, rate(kserve_inference_duration_seconds_bucket[5m]))"
        }]
      },
      {
        "title": "GPU Utilization (Training Nodes)",
        "type": "gauge",
        "targets": [{
          "expr": "avg(DCGM_FI_DEV_GPU_UTIL{kubernetes_node=~'.*gpu.*'})"
        }]
      },
      {
        "title": "Model Drift Score",
        "type": "timeseries",
        "targets": [{
          "expr": "model_drift_score{model_name=~'.*'}"
        }],
        "thresholds": [
          { "value": 0.05, "color": "green" },
          { "value": 0.1, "color": "yellow" },
          { "value": 0.2, "color": "red" }
        ]
      }
    ]
  }
}
```

## 8.4 운영 체크리스트

### 보안

- [ ] Istio mTLS 활성화 (Pod 간 암호화 통신)
- [ ] RBAC 및 NetworkPolicy 설정
- [ ] Secret은 External Secrets Operator 또는 Vault 사용
- [ ] 컨테이너 이미지 스캔 (Trivy)
- [ ] PodSecurityPolicy / Pod Security Standards 적용

### 가용성

- [ ] Kubeflow 핵심 컴포넌트 HA 구성 (replicas >= 2)
- [ ] 메타데이터 DB (MySQL/PostgreSQL) Multi-AZ
- [ ] 아티팩트 스토리지 (S3/GCS) 버전 관리 활성화
- [ ] PDB(PodDisruptionBudget) 설정
- [ ] Velero로 클러스터 백업

### 모니터링

- [ ] Prometheus + Grafana 대시보드
- [ ] 파이프라인 실패 알림 (Slack/PagerDuty)
- [ ] 모델 성능 드리프트 알림
- [ ] 리소스 사용량 알림 (GPU, 메모리, 디스크)
- [ ] 비용 모니터링 (Kubecost)

### 성능

- [ ] Node Affinity/Taint로 워크로드 격리
- [ ] GPU 노드 Scale-to-Zero (Karpenter/Cluster Autoscaler)
- [ ] 파이프라인 캐싱 전략 수립
- [ ] 아티팩트 저장소 라이프사이클 정책

---

## 부록: 주요 참고 자료

| 자료 | URL |
|------|-----|
| Kubeflow 공식 문서 | https://www.kubeflow.org/docs/ |
| KFP SDK v2 API | https://kubeflow-pipelines.readthedocs.io/ |
| KServe 공식 문서 | https://kserve.github.io/website/ |
| Katib 공식 문서 | https://www.kubeflow.org/docs/components/katib/ |
| Kubeflow Manifests | https://github.com/kubeflow/manifests |
| Feast 공식 문서 | https://docs.feast.dev/ |
| MLflow 공식 문서 | https://mlflow.org/docs/latest/index.html |

---

> **강의 요약**: Kubeflow는 Kubernetes-native ML 플랫폼으로, 파이프라인 오케스트레이션(KFP), 하이퍼파라미터 튜닝(Katib), 분산 학습(Training Operators), 모델 서빙(KServe)을 통합 제공합니다. MLflow, Feast, Airflow, ArgoCD 등과의 연계를 통해 완전한 MLOps 시스템을 구축할 수 있으며, LLM 시대에도 QLoRA 파인튜닝과 vLLM 서빙 파이프라인으로 확장 가능합니다.
