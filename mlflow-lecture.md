# MLflow 완벽 가이드 — 기본 개념부터 실무 적용까지

> 대상: MLOps / LLMOps / Cloud Engineer
> 형식: 강의 노트 (총 8개 모듈)

---

## Module 1. MLflow 개요와 아키텍처

### 1-1. MLflow란?

MLflow는 Databricks가 2018년에 오픈소스로 공개한 **엔드투엔드 ML 라이프사이클 관리 플랫폼**이다. 실험 추적, 모델 패키징, 배포, 모델 레지스트리를 하나의 통합 인터페이스로 제공하며, 프레임워크에 종속되지 않는(framework-agnostic) 설계가 핵심 철학이다.

### 1-2. 핵심 컴포넌트 (4+1)

| 컴포넌트 | 역할 | 핵심 키워드 |
|---|---|---|
| **Tracking** | 실험 파라미터·메트릭·아티팩트 기록 | `mlflow.log_param`, `mlflow.log_metric` |
| **Projects** | 재현 가능한 코드 패키징 | `MLproject` 파일, Conda/Docker 환경 |
| **Models** | 모델 직렬화 및 서빙 포맷 통합 | `mlflow.pyfunc`, flavor 시스템 |
| **Model Registry** | 모델 버저닝·스테이지 관리 | Staging → Production → Archived |
| **Evaluate** (2.x+) | LLM 평가 파이프라인 | `mlflow.evaluate()`, custom metrics |

### 1-3. 아키텍처 개요

```
┌─────────────┐       REST / gRPC        ┌──────────────────┐
│  Client SDK  │ ◄─────────────────────► │  Tracking Server │
│ (Python/Java/R)                        │  (Flask-based)   │
└─────────────┘                          └────────┬─────────┘
                                                  │
                              ┌────────────────────┼────────────────────┐
                              ▼                    ▼                    ▼
                     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
                     │ Backend Store│    │ Artifact Store│    │ Model Registry│
                     │ (MySQL/PG/   │    │ (S3/GCS/ADLS/│    │    DB         │
                     │  SQLite)     │    │  HDFS/local) │    │              │
                     └──────────────┘    └──────────────┘    └──────────────┘
```

**Backend Store**는 실험 메타데이터(파라미터, 메트릭, 태그)를 저장하고, **Artifact Store**는 모델 바이너리, 이미지, CSV 등 대용량 파일을 저장한다. 이 둘을 분리하는 것이 프로덕션 설계의 핵심이다.

---

## Module 2. 설치 및 환경 구성

### 2-1. 로컬 개발 환경 (Quick Start)

```bash
# 기본 설치
pip install mlflow[extras]    # extras: scikit-learn, boto3 등 포함

# 버전 확인
mlflow --version

# UI 실행 (로컬 기본 포트 5000)
mlflow ui --port 5000
```

### 2-2. 프로덕션 서버 구성 (Docker Compose)

```yaml
# docker-compose.yml
version: "3.8"
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: mlflow
      POSTGRES_USER: mlflow
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mlflow"]
      interval: 5s

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_PASSWORD}
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"

  mlflow:
    image: ghcr.io/mlflow/mlflow:v2.15.0
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      MLFLOW_S3_ENDPOINT_URL: http://minio:9000
      AWS_ACCESS_KEY_ID: ${MINIO_USER}
      AWS_SECRET_ACCESS_KEY: ${MINIO_PASSWORD}
    command: >
      mlflow server
        --backend-store-uri postgresql://mlflow:${DB_PASSWORD}@postgres:5432/mlflow
        --default-artifact-root s3://mlflow-artifacts/
        --host 0.0.0.0
        --port 5000
    ports:
      - "5000:5000"

volumes:
  pgdata:
  minio_data:
```

### 2-3. Kubernetes 배포 (Helm)

```bash
# Bitnami MLflow Chart 사용
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mlflow bitnami/mlflow \
  --set tracking.auth.enabled=true \
  --set tracking.persistence.storageClass=gp3 \
  --set postgresql.enabled=true \
  --set minio.enabled=true \
  --namespace mlops --create-namespace
```

### 2-4. AWS 네이티브 구성

```bash
# Tracking Server를 ECS/Fargate로, Backend를 RDS Aurora, Artifact를 S3로
mlflow server \
  --backend-store-uri mysql+pymysql://admin:${PW}@mlflow-rds.cluster-xxx.rds.amazonaws.com/mlflow \
  --default-artifact-root s3://my-mlflow-bucket/artifacts \
  --host 0.0.0.0 --port 5000

# 클라이언트 측 설정
export MLFLOW_TRACKING_URI=https://mlflow.internal.company.com
```

---

## Module 3. MLflow Tracking — 실험 추적 심화

### 3-1. 기본 로깅

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, f1_score

# ── 서버 연결 ──
mlflow.set_tracking_uri("http://mlflow-server:5000")

# ── 실험 생성/선택 ──
mlflow.set_experiment("iris-classification")

# ── 데이터 준비 ──
X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# ── 학습 + 로깅 ──
with mlflow.start_run(run_name="rf-baseline") as run:
    # 파라미터 로깅
    params = {"n_estimators": 100, "max_depth": 5, "random_state": 42}
    mlflow.log_params(params)

    # 학습
    model = RandomForestClassifier(**params)
    model.fit(X_train, y_train)
    preds = model.predict(X_test)

    # 메트릭 로깅
    mlflow.log_metric("accuracy", accuracy_score(y_test, preds))
    mlflow.log_metric("f1_weighted", f1_score(y_test, preds, average="weighted"))

    # 모델 로깅 (sklearn flavor)
    mlflow.sklearn.log_model(
        model,
        artifact_path="model",
        input_example=X_test[:3],
        registered_model_name="iris-rf"   # 자동으로 Registry에 등록
    )

    # 추가 아티팩트 (confusion matrix 이미지 등)
    import matplotlib.pyplot as plt
    from sklearn.metrics import ConfusionMatrixDisplay
    fig, ax = plt.subplots()
    ConfusionMatrixDisplay.from_predictions(y_test, preds, ax=ax)
    fig.savefig("confusion_matrix.png")
    mlflow.log_artifact("confusion_matrix.png")

    print(f"Run ID: {run.info.run_id}")
```

### 3-2. Autolog — 자동 로깅

```python
# scikit-learn 자동 로깅 활성화
mlflow.sklearn.autolog(
    log_input_examples=True,
    log_model_signatures=True,
    log_models=True,
    max_tuning_runs=10       # GridSearchCV 시 상위 10개 trial만 기록
)

# 이후 일반적으로 학습하면 자동 기록
model = RandomForestClassifier(n_estimators=200)
model.fit(X_train, y_train)
# → 파라미터, 메트릭, 모델 자동 기록됨
```

지원 프레임워크: `sklearn`, `tensorflow`, `pytorch`, `xgboost`, `lightgbm`, `spark`, `fastai`, `transformers` 등 20+개.

### 3-3. 시계열 메트릭 (Step Logging)

```python
# 에폭 단위 loss 기록 (딥러닝 학습 곡선)
for epoch in range(100):
    train_loss = train_one_epoch(model)
    val_loss = validate(model)
    mlflow.log_metrics({
        "train_loss": train_loss,
        "val_loss": val_loss
    }, step=epoch)
```

### 3-4. Nested Runs (하이퍼파라미터 서치)

```python
with mlflow.start_run(run_name="hyperopt-search") as parent:
    for trial_params in param_grid:
        with mlflow.start_run(run_name=f"trial-{trial_params}", nested=True):
            mlflow.log_params(trial_params)
            score = train_and_evaluate(trial_params)
            mlflow.log_metric("val_auc", score)
```

### 3-5. 태그와 검색

```python
# 태그 설정
mlflow.set_tag("team", "ml-platform")
mlflow.set_tag("dataset_version", "v2.3")
mlflow.set_tag("purpose", "baseline")

# 프로그래밍 방식 검색
client = mlflow.MlflowClient()
runs = client.search_runs(
    experiment_ids=["1"],
    filter_string="metrics.accuracy > 0.95 AND params.n_estimators = '200'",
    order_by=["metrics.accuracy DESC"],
    max_results=5
)
for r in runs:
    print(f"{r.info.run_id}: accuracy={r.data.metrics['accuracy']:.4f}")
```

---

## Module 4. MLflow Models — 모델 패키징과 서빙

### 4-1. Flavor 시스템

MLflow의 핵심 추상화이다. 하나의 모델을 여러 "flavor"로 저장해 다양한 환경에서 로드할 수 있다.

```
model/
├── MLmodel              # 메타데이터 (flavor 정보, signature)
├── conda.yaml           # 재현 가능한 의존성
├── requirements.txt
├── python_env.yaml
├── model.pkl            # sklearn flavor용 바이너리
└── input_example.json   # 입력 예시
```

**MLmodel 파일 예시:**
```yaml
artifact_path: model
flavors:
  python_function:
    env:
      conda: conda.yaml
      virtualenv: python_env.yaml
    loader_module: mlflow.sklearn
    model_path: model.pkl
    python_version: 3.11.5
  sklearn:
    code: null
    pickled_model: model.pkl
    serialization_format: cloudpickle
    sklearn_version: 1.4.0
signature:
  inputs: '[{"type": "double", "name": "sepal_length"}, ...]'
  outputs: '[{"type": "long"}]'
```

### 4-2. Custom PyFunc Model

```python
import mlflow.pyfunc
import pandas as pd

class PreprocessAndPredict(mlflow.pyfunc.PythonModel):
    """전처리 + 예측을 하나의 서빙 단위로 캡슐화"""

    def load_context(self, context):
        """모델 로드 시 1회 실행"""
        import joblib
        self.scaler = joblib.load(context.artifacts["scaler"])
        self.model = joblib.load(context.artifacts["model"])

    def predict(self, context, model_input: pd.DataFrame, params=None):
        """예측 요청마다 실행"""
        scaled = self.scaler.transform(model_input)
        predictions = self.model.predict(scaled)
        probabilities = self.model.predict_proba(scaled)
        return pd.DataFrame({
            "prediction": predictions,
            "confidence": probabilities.max(axis=1)
        })

# 저장
artifacts = {
    "scaler": "artifacts/scaler.joblib",
    "model": "artifacts/rf_model.joblib"
}

with mlflow.start_run():
    mlflow.pyfunc.log_model(
        artifact_path="custom_model",
        python_model=PreprocessAndPredict(),
        artifacts=artifacts,
        pip_requirements=["scikit-learn==1.4.0", "pandas>=2.0"],
        input_example=X_test[:3],
        registered_model_name="iris-custom-pipeline"
    )
```

### 4-3. 모델 서빙

```bash
# 로컬 REST API 서빙
mlflow models serve \
  -m "models:/iris-rf/Production" \
  --port 8080 \
  --env-manager virtualenv

# Docker 이미지 빌드
mlflow models build-docker \
  -m "models:/iris-rf/3" \
  -n iris-rf-serving \
  --enable-mlserver    # Seldon MLServer 기반 (gRPC + REST)

# 추론 요청
curl -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"dataframe_split": {
    "columns": ["sepal_length","sepal_width","petal_length","petal_width"],
    "data": [[5.1, 3.5, 1.4, 0.2]]
  }}'
```

---

## Module 5. Model Registry — 거버넌스와 워크플로

### 5-1. 모델 등록 및 버전 관리

```python
from mlflow import MlflowClient

client = MlflowClient()

# 방법 1: log_model 시 자동 등록 (위에서 이미 사용)
# 방법 2: 기존 run의 모델을 수동 등록
result = client.create_model_version(
    name="iris-rf",
    source=f"runs:/{run_id}/model",
    run_id=run_id,
    description="RandomForest with depth=5, n_estimators=200"
)
print(f"Version: {result.version}")
```

### 5-2. 스테이지 전환 (Aliases 방식 — 2.x 권장)

MLflow 2.x에서는 기존 Stage(Staging/Production/Archived) 대신 **Alias** 시스템을 권장한다.

```python
# Alias 설정 (2.x 방식)
client.set_registered_model_alias("iris-rf", "champion", version=3)
client.set_registered_model_alias("iris-rf", "challenger", version=4)

# Alias로 모델 로드
champion = mlflow.pyfunc.load_model("models:/iris-rf@champion")
challenger = mlflow.pyfunc.load_model("models:/iris-rf@challenger")

# 레거시 Stage 방식 (1.x 호환)
client.transition_model_version_stage(
    name="iris-rf",
    version=4,
    stage="Production",
    archive_existing_versions=True   # 기존 Production 자동 아카이브
)
```

### 5-3. 모델 승인 워크플로 (Webhooks + CI/CD)

```python
# MLflow에서 모델 버전 변경 감지 → CI/CD 트리거
# GitHub Actions / Jenkins 등과 연동

import requests

def promote_model_if_passed(model_name, version, metrics_threshold):
    """검증 통과 시 Production으로 승격"""
    client = MlflowClient()

    # 1. 챌린저 모델의 메트릭 확인
    mv = client.get_model_version(model_name, version)
    run = client.get_run(mv.run_id)
    accuracy = run.data.metrics.get("accuracy", 0)

    # 2. 기준 충족 여부 판단
    if accuracy >= metrics_threshold:
        # 3. Shadow 테스트 결과 확인 (외부 시스템)
        shadow_result = check_shadow_test(model_name, version)

        if shadow_result["passed"]:
            client.set_registered_model_alias(model_name, "champion", version)
            notify_slack(f"✅ {model_name} v{version} promoted to champion")
            return True

    notify_slack(f"❌ {model_name} v{version} did not meet criteria")
    return False
```

---

## Module 6. LLM 관련 기능 (MLflow 2.x+)

### 6-1. LLM Tracking

```python
import mlflow
import openai

mlflow.set_experiment("llm-experiments")

# OpenAI autolog
mlflow.openai.autolog()

# 자동으로 프롬프트, 응답, 토큰 사용량, 지연시간 기록
client = openai.OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Explain MLOps in 3 sentences."}],
    temperature=0.7
)
```

### 6-2. MLflow Evaluate (LLM 평가)

```python
import mlflow
import pandas as pd

eval_data = pd.DataFrame({
    "inputs": [
        "What is MLflow?",
        "How does model registry work?",
        "Explain feature stores."
    ],
    "ground_truth": [
        "MLflow is an open-source ML lifecycle platform...",
        "Model registry provides versioning and stage management...",
        "Feature stores centralize feature computation and serving..."
    ]
})

with mlflow.start_run():
    results = mlflow.evaluate(
        model="openai:/gpt-4o",
        data=eval_data,
        targets="ground_truth",
        model_type="question-answering",
        evaluators="default",
        extra_metrics=[
            mlflow.metrics.latency(),
            mlflow.metrics.token_count(),
            mlflow.metrics.toxicity(),
            mlflow.metrics.relevance(),         # LLM-as-judge
            mlflow.metrics.faithfulness(),       # RAG 충실도
        ]
    )

    print(f"Metrics: {results.metrics}")
    print(results.tables["eval_results_table"])
```

### 6-3. Prompt Engineering UI

MLflow 2.7+에서는 UI를 통해 프롬프트 템플릿을 관리하고 A/B 비교할 수 있다.

```python
# 프로그래밍 방식 프롬프트 버저닝
from mlflow.models import set_model

prompt_template = """
You are a helpful assistant specialized in {domain}.
Answer the following question concisely:
{question}
"""

with mlflow.start_run():
    mlflow.log_param("prompt_version", "v3")
    mlflow.log_param("prompt_template", prompt_template)
    mlflow.log_param("domain", "MLOps")
```

### 6-4. RAG 평가 파이프라인

```python
# 커스텀 RAG 메트릭 정의
from mlflow.metrics.genai import make_genai_metric, EvaluationExample

context_precision = make_genai_metric(
    name="context_precision",
    definition="Measures whether the retrieved context is relevant to the question.",
    grading_prompt=(
        "Score from 1-5 how relevant the retrieved context is.\n"
        "Context: {context}\nQuestion: {inputs}\nAnswer: {targets}"
    ),
    examples=[
        EvaluationExample(
            input="What is MLflow?",
            output="MLflow is a database tool.",
            score=1,
            justification="Incorrect — MLflow is an ML lifecycle platform."
        )
    ],
    model="openai:/gpt-4o",
    parameters={"temperature": 0},
    greater_is_better=True
)

results = mlflow.evaluate(
    model=my_rag_chain,
    data=eval_data,
    extra_metrics=[context_precision]
)
```

---

## Module 7. 다른 MLOps 도구와의 연계

### 7-1. MLflow + Airflow (워크플로 오케스트레이션)

```python
# airflow DAG
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def train_and_register():
    import mlflow
    mlflow.set_tracking_uri("http://mlflow:5000")
    mlflow.set_experiment("scheduled-training")

    with mlflow.start_run():
        model = train_model()
        metrics = evaluate_model(model)
        mlflow.log_metrics(metrics)
        mlflow.sklearn.log_model(model, "model",
                                 registered_model_name="prod-model")

def validate_and_promote():
    from mlflow import MlflowClient
    client = MlflowClient("http://mlflow:5000")
    latest = client.get_latest_versions("prod-model", stages=["None"])[0]

    if run_validation_suite(latest):
        client.set_registered_model_alias("prod-model", "champion", latest.version)

dag = DAG("ml_pipeline", schedule_interval="@weekly",
          start_date=datetime(2025, 1, 1), catchup=False)

t1 = PythonOperator(task_id="train", python_callable=train_and_register, dag=dag)
t2 = PythonOperator(task_id="validate", python_callable=validate_and_promote, dag=dag)
t1 >> t2
```

### 7-2. MLflow + Kubeflow Pipelines

```python
# Kubeflow 컴포넌트로 MLflow 통합
from kfp import dsl
from kfp.dsl import component, Output, Model

@component(
    base_image="python:3.11",
    packages_to_install=["mlflow==2.15.0", "scikit-learn", "boto3"]
)
def train_component(tracking_uri: str, experiment: str, run_id: Output[str]):
    import mlflow
    mlflow.set_tracking_uri(tracking_uri)
    mlflow.set_experiment(experiment)

    with mlflow.start_run() as run:
        # ... 학습 로직 ...
        mlflow.sklearn.log_model(model, "model")
        run_id = run.info.run_id

@component(packages_to_install=["mlflow==2.15.0"])
def deploy_component(tracking_uri: str, model_name: str, version: int):
    from mlflow import MlflowClient
    client = MlflowClient(tracking_uri)
    client.set_registered_model_alias(model_name, "champion", version)

@dsl.pipeline(name="mlflow-kfp-pipeline")
def ml_pipeline():
    train_task = train_component(
        tracking_uri="http://mlflow:5000",
        experiment="kfp-experiment"
    )
    deploy_task = deploy_component(
        tracking_uri="http://mlflow:5000",
        model_name="my-model",
        version=1
    ).after(train_task)
```

### 7-3. MLflow + DVC (데이터 버저닝)

```python
# DVC로 데이터 버전 관리, MLflow로 실험 추적
import dvc.api
import mlflow

# DVC에서 데이터 경로 및 버전 가져오기
data_url = dvc.api.get_url("data/train.csv", repo=".", rev="v2.0")
params = dvc.api.params_show()  # dvc.yaml의 파라미터

with mlflow.start_run():
    # 데이터 리니지 기록
    mlflow.set_tag("dvc.data_version", "v2.0")
    mlflow.set_tag("dvc.data_url", data_url)
    mlflow.log_params(params)

    data = pd.read_csv("data/train.csv")
    # ... 학습 ...
```

### 7-4. MLflow + Feast (Feature Store)

```python
from feast import FeatureStore
import mlflow

store = FeatureStore(repo_path="./feature_repo")

# Feast에서 학습 데이터셋 생성
training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "user_features:age",
        "user_features:lifetime_value",
        "transaction_features:avg_amount_7d",
    ]
).to_df()

with mlflow.start_run():
    mlflow.set_tag("feature_store", "feast")
    mlflow.set_tag("feature_service", "user_features,transaction_features")

    # 피처 목록을 파라미터로 기록
    mlflow.log_param("features", list(training_df.columns))

    model = train(training_df)
    mlflow.sklearn.log_model(model, "model")
```

### 7-5. MLflow + Seldon Core / KServe (서빙)

```yaml
# seldon-deployment.yaml
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: iris-model
spec:
  predictors:
    - name: default
      replicas: 2
      graph:
        name: classifier
        implementation: MLFLOW_SERVER
        modelUri: s3://mlflow-artifacts/1/abc123/artifacts/model
        envSecretRefName: seldon-s3-secret
      componentSpecs:
        - spec:
            containers:
              - name: classifier
                resources:
                  requests:
                    cpu: "500m"
                    memory: "1Gi"
                  limits:
                    cpu: "2"
                    memory: "4Gi"
```

```python
# KServe InferenceService
# MLflow 모델을 직접 KServe로 서빙
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: iris-mlflow
spec:
  predictor:
    model:
      modelFormat:
        name: mlflow
      storageUri: s3://mlflow-artifacts/1/abc123/artifacts/model
```

### 7-6. MLflow + Prometheus + Grafana (모니터링)

```python
# 커스텀 서빙 서버에서 Prometheus 메트릭 노출
from prometheus_client import Counter, Histogram, start_http_server
import mlflow.pyfunc
import time

REQUEST_COUNT = Counter("prediction_requests_total", "Total predictions", ["model_name", "version"])
LATENCY = Histogram("prediction_latency_seconds", "Prediction latency", ["model_name"])
DRIFT_SCORE = Histogram("feature_drift_score", "Feature drift", ["feature_name"])

model = mlflow.pyfunc.load_model("models:/iris-rf@champion")

def predict_with_monitoring(input_data, model_name="iris-rf", version="3"):
    start = time.time()
    result = model.predict(input_data)
    latency = time.time() - start

    REQUEST_COUNT.labels(model_name, version).inc()
    LATENCY.labels(model_name).observe(latency)

    # 드리프트 감지 (PSI / KS-test 등)
    detect_and_report_drift(input_data)

    return result

start_http_server(8001)  # Prometheus scrape endpoint
```

### 7-7. 도구 비교 및 포지셔닝

| 영역 | MLflow 역할 | 보완 도구 | 연계 방식 |
|---|---|---|---|
| 실험 추적 | ✅ 핵심 기능 | W&B (더 풍부한 시각화) | W&B는 별도 사용 또는 MLflow로 통합 |
| 데이터 버저닝 | ❌ 미지원 | DVC, LakeFS, Delta Lake | 태그로 데이터 버전 추적 |
| 피처 스토어 | ❌ 미지원 | Feast, Tecton, SageMaker FS | Feast에서 데이터 추출 → MLflow로 학습 기록 |
| 파이프라인 오케스트레이션 | 🔸 Projects (기본) | Airflow, Kubeflow, Prefect | DAG 내에서 MLflow API 호출 |
| 모델 서빙 | 🔸 기본 제공 | Seldon, KServe, BentoML | MLflow model format을 직접 배포 |
| 모니터링/드리프트 | ❌ 미지원 | Evidently, WhyLabs, NannyML | 서빙 레이어에서 별도 연동 |
| CI/CD | ❌ 미지원 | GitHub Actions, Jenkins | Registry webhook → CI/CD 트리거 |

---

## Module 8. 고급 기법 및 프로덕션 베스트 프랙티스

### 8-1. 멀티 테넌시 & 접근 제어

```bash
# MLflow 2.5+ 기본 인증
mlflow server \
  --app-name basic-auth \
  --backend-store-uri postgresql://... \
  --default-artifact-root s3://...
```

```python
# 프로그래밍 방식 사용자/권한 관리
from mlflow.server import get_app_client

auth_client = get_app_client("basic-auth", tracking_uri="http://mlflow:5000")
auth_client.create_user(username="data-scientist-1", password="secure-pw")
auth_client.update_experiment_permission(
    experiment_id="1",
    username="data-scientist-1",
    permission="MANAGE"    # READ, EDIT, MANAGE, NO_PERMISSIONS
)
```

### 8-2. A/B 테스트 패턴 (Champion-Challenger)

```python
import random
import mlflow.pyfunc

# 두 모델 로드
champion = mlflow.pyfunc.load_model("models:/my-model@champion")
challenger = mlflow.pyfunc.load_model("models:/my-model@challenger")

def route_prediction(input_data, user_id: str):
    """트래픽 라우팅: 90% champion, 10% challenger"""
    # 결정적 라우팅 (동일 사용자는 항상 같은 모델)
    use_challenger = (hash(user_id) % 100) < 10

    if use_challenger:
        result = challenger.predict(input_data)
        log_prediction("challenger", user_id, input_data, result)
    else:
        result = champion.predict(input_data)
        log_prediction("champion", user_id, input_data, result)

    return result
```

### 8-3. 대규모 실험 관리 전략

```python
# 1. 실험 네이밍 컨벤션
# {팀}/{프로젝트}/{태스크} 형식
mlflow.set_experiment("ml-platform/recommendation/ctr-prediction")

# 2. 환경별 태그 자동 부여
import os
import subprocess

with mlflow.start_run():
    # Git 정보 자동 기록
    git_hash = subprocess.check_output(["git", "rev-parse", "HEAD"]).strip().decode()
    mlflow.set_tag("mlflow.source.git.commit", git_hash)
    mlflow.set_tag("environment", os.getenv("ENV", "dev"))
    mlflow.set_tag("docker_image", os.getenv("DOCKER_IMAGE", "local"))

    # 데이터셋 핑거프린트
    import hashlib
    data_hash = hashlib.md5(open("data/train.csv", "rb").read()).hexdigest()
    mlflow.set_tag("data_md5", data_hash)
```

### 8-4. 모델 시그니처 & 입력 검증

```python
from mlflow.models import infer_signature, ModelSignature
from mlflow.types.schema import Schema, ColSpec

# 자동 추론
signature = infer_signature(X_train, model.predict(X_train))

# 수동 정의 (엄격한 제어)
input_schema = Schema([
    ColSpec("double", "sepal_length"),
    ColSpec("double", "sepal_width"),
    ColSpec("double", "petal_length"),
    ColSpec("double", "petal_width"),
])
output_schema = Schema([ColSpec("long", "prediction")])
signature = ModelSignature(inputs=input_schema, outputs=output_schema)

mlflow.sklearn.log_model(model, "model", signature=signature)
# → 서빙 시 signature 불일치 요청은 자동 거부됨
```

### 8-5. 가비지 컬렉션 & 정리

```python
from mlflow import MlflowClient
from datetime import datetime, timedelta

client = MlflowClient()

# 30일 이상 된 삭제 상태 Run 영구 제거
cutoff = datetime.now() - timedelta(days=30)

experiments = client.search_experiments()
for exp in experiments:
    deleted_runs = client.search_runs(
        experiment_ids=[exp.experiment_id],
        filter_string="",
        run_view_type=mlflow.entities.ViewType.DELETED_ONLY
    )
    for run in deleted_runs:
        if run.info.end_time and run.info.end_time < cutoff.timestamp() * 1000:
            client.delete_run(run.info.run_id)  # 영구 삭제
            print(f"Permanently deleted: {run.info.run_id}")
```

```bash
# CLI로 아티팩트 가비지 컬렉션 (2.x)
mlflow gc --backend-store-uri postgresql://... --older-than 30d
```

### 8-6. 전체 아키텍처 레퍼런스 (프로덕션)

```
Developer/DS Workstation
    │
    ▼
┌────────────────────────────────────────────────────────┐
│  Orchestration Layer (Airflow / Kubeflow / Prefect)    │
│    ┌──────┐  ┌──────────┐  ┌───────────┐  ┌────────┐  │
│    │ Data │→ │ Feature  │→ │  Training │→ │ Eval & │  │
│    │ Ingest│  │ Engineer │  │  (GPU K8s)│  │ Promote│  │
│    └──────┘  └──────────┘  └───────────┘  └────────┘  │
└───────┬────────────┬──────────────┬──────────────┬─────┘
        │            │              │              │
        ▼            ▼              ▼              ▼
   ┌─────────┐ ┌──────────┐ ┌────────────┐ ┌───────────────┐
   │  DVC /  │ │  Feast   │ │  MLflow    │ │  MLflow       │
   │  Delta  │ │  Feature │ │  Tracking  │ │  Registry     │
   │  Lake   │ │  Store   │ │  Server    │ │  (Alias mgmt) │
   └─────────┘ └──────────┘ └────────────┘ └───────┬───────┘
                                                    │
                                                    ▼
                                          ┌──────────────────┐
                                          │ Serving Layer     │
                                          │ Seldon / KServe / │
                                          │ SageMaker Endpoint│
                                          └────────┬─────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────┐
                                          │ Monitoring        │
                                          │ Evidently + Prom  │
                                          │ + Grafana         │
                                          └──────────────────┘
```

---

## 부록: 자주 하는 실수와 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `RESOURCE_DOES_NOT_EXIST` | 실험 이름 오타 또는 삭제된 실험 | `mlflow.set_experiment()` 재확인, `search_experiments()` 조회 |
| 아티팩트 업로드 느림 | 대용량 파일을 default file store에 기록 | S3/GCS artifact store 분리 구성 |
| 모델 로드 시 의존성 에러 | conda.yaml / requirements.txt 불일치 | `pip_requirements` 명시적 지정, Docker 환경 통일 |
| Tracking Server OOM | 동시 다수 Run의 대량 메트릭 기록 | `mlflow.log_metrics()`로 배치 기록, 비동기 로깅 활성화 |
| Registry 동시 접근 충돌 | 여러 파이프라인이 같은 모델을 동시 업데이트 | Alias 시스템 사용, 버전 번호 명시적 참조 |
| `mlflow.autolog()` 중복 기록 | 여러 프레임워크의 autolog 동시 활성화 | `mlflow.autolog(exclusive=True)` 또는 프레임워크별 개별 설정 |

---

## 핵심 요약

1. **MLflow = 실험추적 + 모델패키징 + 레지스트리** — 이 세 축을 중심으로 ML 라이프사이클을 관리한다.
2. **Backend Store와 Artifact Store를 반드시 분리**하여 프로덕션을 구성한다.
3. **Flavor 시스템과 PyFunc**를 이해하면 어떤 프레임워크의 모델이든 통일된 방식으로 서빙할 수 있다.
4. **MLflow 2.x의 Alias 시스템**이 기존 Stage 방식을 대체하며, 더 유연한 모델 라이프사이클 관리를 지원한다.
5. **MLflow는 "전체" MLOps가 아니다** — 데이터 버저닝(DVC), 피처 스토어(Feast), 오케스트레이션(Airflow), 모니터링(Evidently)과 조합하여 완전한 플랫폼을 구성한다.
6. **LLM 시대에도 유효** — `mlflow.evaluate()`와 LLM-as-judge 패턴으로 LLM/RAG 파이프라인의 실험 관리가 가능하다.
