# Apache Spark 완벽 가이드

## ― 기초 개념부터 DGX Spark + ThinkStation Local K8s 실습까지

> **대상 독자**: LLMOps / MLOps / Cloud Engineer
> **실습 환경**: NVIDIA DGX Spark (ARM64 + Blackwell GPU) + Lenovo ThinkStation (x86_64) 기반 2노드 Local Kubernetes
> **Spark 기준 버전**: 4.1.x (LTS 마이그레이션이 필요한 환경은 3.5.x LTS 사용 가능, 2027-11까지 보안 패치 지원)
> **작성일**: 2026-05

---

## 📚 목차

- [Part 1. Apache Spark 입문](#part-1-apache-spark-입문)
- [Part 2. Spark 핵심 개념 정복](#part-2-spark-핵심-개념-정복)
- [Part 3. Spark 아키텍처 심층 이해](#part-3-spark-아키텍처-심층-이해)
- [Part 4. Spark 컴포넌트 라이브러리](#part-4-spark-컴포넌트-라이브러리)
- [Part 5. PySpark 기본 사용법](#part-5-pyspark-기본-사용법)
- [Part 6. Spark on Kubernetes 이해](#part-6-spark-on-kubernetes-이해)
- [Part 7. 실습 환경 구성 — DGX Spark + ThinkStation](#part-7-실습-환경-구성--dgx-spark--thinkstation)
- [Part 8. Spark Operator 설치](#part-8-spark-operator-설치)
- [Part 9. 첫 번째 Spark 작업 실행](#part-9-첫-번째-spark-작업-실행)
- [Part 10. GPU 가속 — RAPIDS Accelerator](#part-10-gpu-가속--rapids-accelerator)
- [Part 11. 실전 ML/LLM 데이터 파이프라인](#part-11-실전-mlllm-데이터-파이프라인)
- [Part 12. 트러블슈팅과 운영 노하우](#part-12-트러블슈팅과-운영-노하우)
- [부록 A. 명령어 치트시트](#부록-a-명령어-치트시트)
- [부록 B. 참고 자료](#부록-b-참고-자료)

---

# Part 1. Apache Spark 입문

## 1.1 Spark, 한 줄로 정의하면?

> **Apache Spark는 "수많은 컴퓨터(=노드)에 데이터를 잘게 쪼개어 동시에 처리하는, 메모리 중심의 분산 데이터 처리 엔진"입니다.**

쉽게 비유하자면 이렇습니다.

- 한 사람이 책 1만 권을 읽고 요약하면 1년이 걸립니다 → 단일 머신 처리 (Pandas 등)
- 100명이 100권씩 나눠 읽고 요약을 합치면 며칠 만에 끝납니다 → **분산 처리 (Spark)**

여기서 중요한 포인트가 두 가지 있습니다.

1. **"잘게 쪼개기"** : 데이터를 *Partition* 단위로 나누어 노드에 분산합니다.
2. **"메모리에서 처리"** : Hadoop MapReduce는 매 단계마다 디스크에 결과를 쓰지만, Spark는 가능한 한 RAM에 올려두고 처리해서 10~100배 빠릅니다.

## 1.2 Spark는 왜 등장했는가? — 역사 한 컷

```
2003 ─ Google File System (GFS) 논문
2004 ─ Google MapReduce 논문
2006 ─ Hadoop 등장 (분산 파일 + MapReduce 오픈소스화)
2009 ─ UC Berkeley AMPLab에서 Spark 시작 (Matei Zaharia)
2014 ─ Apache Top-Level Project 승격
2020 ─ Spark 3.0: Adaptive Query Execution, GPU 지원 강화
2025 ─ Spark 4.0: Spark Connect 안정화, ANSI 모드 기본화
2026 ─ Spark 4.1.x 안정 배포, 4.2 프리뷰
```

핵심 동기는 단순합니다. **"MapReduce는 너무 느리고 너무 어렵다."** 매번 디스크에 쓰고 읽는 구조이기 때문에, 머신러닝처럼 같은 데이터를 반복(iteration) 처리하는 작업에서 비효율이 극심했습니다. Spark는 RDD(Resilient Distributed Dataset)라는 추상화를 통해 **"데이터를 메모리에 올려두고 여러 단계의 변환을 효율적으로 수행"** 하는 길을 열었습니다.

## 1.3 Spark vs MapReduce vs Pandas

| 항목 | Pandas | Hadoop MapReduce | Apache Spark |
|------|--------|------------------|--------------|
| 처리 위치 | 단일 머신 RAM | 분산, 디스크 | **분산, 주로 RAM** |
| 처리 속도 | 작은 데이터 빠름 | 느림 | 매우 빠름 |
| 처리 가능 크기 | RAM 한계 | TB~PB | TB~PB |
| API 친화도 | 매우 높음 | 매우 낮음 | 높음 (DataFrame) |
| 머신러닝 반복 | 가능 | 비효율 | 매우 효율적 |
| 스트리밍 지원 | ✗ | ✗ | ✓ (Structured Streaming) |
| SQL 인터페이스 | 부분적 | Hive 별도 | 내장 (Spark SQL) |

**결론**: 데이터가 RAM을 넘어가는 순간, 또는 분산 환경이 필요한 순간 Spark가 첫 번째 선택지가 됩니다.

## 1.4 Spark는 어디에 쓰이는가? — 실제 활용 시나리오

LLMOps/MLOps 관점에서 특히 중요한 활용 영역입니다.

1. **대규모 ETL (Extract-Transform-Load)** : 원시 로그를 Parquet/Delta로 변환
2. **피처 엔지니어링** : ML 학습 입력 만들기
3. **LLM 학습 데이터 전처리** : Common Crawl 정제, 중복 제거(deduplication), 토크나이즈 전 단계
4. **임베딩 대량 추론(batch inference)** : DataFrame에 UDF로 임베딩 적용
5. **벡터 DB 적재 파이프라인** : Spark → Milvus / Weaviate / pgvector
6. **모델 평가 배치** : 수백만 샘플에 대한 평가 점수 계산

> **현실 사례**: OpenAI GPT-3 학습 데이터 정제, Meta LLaMA 데이터 파이프라인, Databricks의 Mosaic 등 대부분의 LLM 사전학습 데이터 파이프라인은 Spark 또는 Spark 기반 도구(Ray + Spark, Daft + Spark)를 사용합니다.

---

# Part 2. Spark 핵심 개념 정복

이 파트가 이 강의에서 가장 중요합니다. 여기를 완벽히 이해하면 Spark 90%는 정복한 셈입니다.

## 2.1 RDD (Resilient Distributed Dataset) — Spark의 원자(原子)

**RDD**는 "탄력적(resilient) + 분산된(distributed) + 데이터셋(dataset)"의 약자입니다.

세 단어를 풀어보면:

- **분산된**: 데이터가 여러 노드에 파티션으로 나뉘어 있다.
- **탄력적**: 일부 노드가 죽어도 *Lineage(계보)* 를 통해 자동으로 재계산해서 복구할 수 있다.
- **데이터셋**: 그냥 데이터 모음.

```
[원본 데이터: 1억 건]
        │
        ▼ (Partitioning)
┌──────────┬──────────┬──────────┬──────────┐
│Partition0│Partition1│Partition2│Partition3│
│ 2,500만  │ 2,500만  │ 2,500만  │ 2,500만  │
└──────────┴──────────┴──────────┴──────────┘
   Node A    Node B     Node C    Node D
```

### 💡 비유: 도서관 책 정리

- 책 1만 권(데이터)을 4개 책장(파티션)에 나눠 꽂습니다.
- 사서 4명(Executor)이 각자 자기 책장만 봅니다.
- 1번 사서가 갑자기 결근해도, 1번 책장의 원본이 있으면(Lineage) 다른 사람이 그 일을 다시 할 수 있습니다.

> **현재는 RDD를 직접 쓰지 않습니다.** Spark 2.0 이후로는 DataFrame/Dataset이 표준입니다. 다만 **개념적 토대**로서 RDD를 이해하지 못하면 DataFrame의 동작 원리(특히 셔플, 파티셔닝)를 이해할 수 없습니다.

## 2.2 DataFrame과 Dataset — 현대 Spark의 주역

### DataFrame

> **DataFrame = RDD + Schema (컬럼 이름 + 타입)**

Pandas DataFrame을 분산 환경으로 확장한 것이라고 생각하면 됩니다. SQL 테이블과 거의 동일한 구조입니다.

```python
# PySpark DataFrame 예시
+----+-------+-----+
| id |  name | age |
+----+-------+-----+
| 1  | Alice |  30 |
| 2  | Bob   |  25 |
| 3  | Carol |  35 |
+----+-------+-----+
```

**왜 DataFrame이 RDD보다 빠른가?**

1. **Catalyst Optimizer**: SQL 옵티마이저처럼 쿼리 실행 계획을 자동 최적화합니다.
2. **Tungsten Execution Engine**: JVM 객체가 아닌 압축된 바이너리 포맷으로 메모리 사용을 최적화합니다.
3. **Whole-Stage Code Generation**: 여러 연산을 한 덩어리 자바 코드로 생성해 함수 호출 오버헤드를 제거합니다.

### Dataset

Scala/Java에서 사용하는 **타입 안전한 DataFrame**입니다. Python(PySpark)에서는 Dataset 개념이 없고 DataFrame만 있습니다.

```scala
// Scala
case class Person(id: Int, name: String, age: Int)
val ds: Dataset[Person] = spark.read.parquet("...").as[Person]
ds.filter(_.age > 30)  // 컴파일 타임에 타입 체크
```

## 2.3 SparkSession — 모든 것의 시작점

Spark 2.0부터 등장한 **단일 진입점**입니다. SQLContext, HiveContext, SparkContext 등 모든 것을 통합했습니다.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MyApp") \
    .master("local[*]") \
    .config("spark.executor.memory", "4g") \
    .getOrCreate()
```

**핵심 포인트**

- `master`: 어디서 실행할 것인가 (local, yarn, k8s://, spark://)
- `appName`: Spark UI에 표시되는 이름
- `getOrCreate()`: 이미 있으면 재사용, 없으면 생성

## 2.4 Lazy Evaluation — Spark의 마법

> **Spark는 명령을 내려도 즉시 실행하지 않습니다. 결과를 "꼭 봐야 할 때"까지 미룹니다.**

이 동작은 헷갈리면서도 Spark 성능의 핵심입니다.

```python
df = spark.read.csv("data.csv")        # ← 실행 X (계획만 세움)
df = df.filter(df.age > 30)            # ← 실행 X
df = df.groupBy("city").count()        # ← 실행 X
df.show()                              # ← 여기서 처음으로 ⚡ 실행됨!
```

### 왜 미루는가?

전체 그림을 다 본 다음 **최적 실행 계획**을 세우기 위해서입니다. 만약 즉시 실행한다면:

```
filter → groupBy → ... 매 단계 결과를 메모리에 저장 (낭비)
```

지연 실행이라면:

```
"아, filter+groupBy를 한 번에 처리하면 데이터를 한 번만 스캔해도 되겠네!"
→ Catalyst가 단계 병합, 푸시다운, 컬럼 프루닝 등을 자동 적용
```

## 2.5 Transformation vs Action — 절대 헷갈리면 안 되는 이분법

| 분류 | 예시 | 특징 |
|------|------|------|
| **Transformation** | `filter`, `map`, `select`, `groupBy`, `join`, `withColumn` | 새 DataFrame을 반환, **즉시 실행 X** |
| **Action** | `show`, `count`, `collect`, `write`, `take`, `foreach` | 결과 반환 또는 저장, **즉시 실행 ✓** |

### 🎯 외우는 법

> "**T**ransformation은 **T**odo (할 일 목록만 작성), **A**ction은 **A**ttack (즉시 공격 = 실제 실행)"

```python
# 👇 모두 Transformation (실행 안 됨)
df1 = df.filter("age > 30")
df2 = df1.groupBy("city").count()
df3 = df2.orderBy("count", ascending=False)

# 👇 Action — 여기서 위의 모든 변환이 한 번에 실행됨
df3.show(10)
```

### Transformation의 두 종류 — Narrow vs Wide

이건 성능 튜닝의 핵심입니다.

| 종류 | 동작 | 예시 | 비용 |
|------|------|------|------|
| **Narrow** | 자신의 파티션에서만 처리 | `filter`, `map`, `select` | 저렴 |
| **Wide (Shuffle)** | 다른 노드와 데이터 교환 필요 | `groupBy`, `join`, `distinct`, `orderBy` | 매우 비쌈 |

```
Narrow Transformation:
[P0] ──map──→ [P0']
[P1] ──map──→ [P1']
(노드 간 통신 X)

Wide Transformation (Shuffle):
[P0] ─┐    ┌─→ [P0']
[P1] ─┼─🌐─┼─→ [P1']
[P2] ─┘    └─→ [P2']
(네트워크 통해 데이터 재분배)
```

> **튜닝 1법칙**: "Shuffle을 줄여라." Shuffle은 디스크 I/O + 네트워크 I/O + 직렬화 비용이 모두 발생합니다.

## 2.6 Driver와 Executor — 누가 무엇을 하는가

```
┌─────────────────────────────────────────────────────┐
│  Driver Program (= 사령탑)                            │
│  - SparkSession을 만든 곳                             │
│  - DAG를 만들고 Task를 스케줄링                        │
│  - Action 결과를 collect()                            │
└─────────────────┬───────────────────────────────────┘
                  │ Task 분배
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   ┌────────┐┌────────┐┌────────┐
   │Executor││Executor││Executor│
   │ JVM    ││ JVM    ││ JVM    │
   │ 실제   ││ 실제   ││ 실제   │
   │ 작업   ││ 작업   ││ 작업   │
   └────────┘└────────┘└────────┘
```

- **Driver**: "공장장". 코드를 분석하고 계획을 세우고 결과를 받는다.
- **Executor**: "공장 노동자". 실제 데이터 처리를 하는 JVM 프로세스. 각 Executor 안에서 여러 **Task** 가 동시에 돈다(=core 개수).

### 💡 자주 하는 실수

```python
# ❌ Driver로 1억 건을 다 끌어와서 메모리 폭발
data = df.collect()

# ✅ Executor에서 처리하고 결과만 모으기
df.write.parquet("s3://output/")
```

`collect()`를 잘못 쓰는 게 신입 엔지니어 #1 실수입니다. 꼭 `count()`, `take(n)`, `write` 등으로 대체하세요.

## 2.7 DAG와 Stage — 실행 계획의 시각화

Spark는 Action이 호출되면 다음 순서로 실행 계획을 만듭니다.

```
1. Logical Plan      : 사용자 코드를 추상 트리로 변환
2. Optimized Plan    : Catalyst가 최적화 (predicate pushdown 등)
3. Physical Plan     : 실제 실행 가능한 형태로 변환
4. DAG of Stages     : Shuffle 경계에서 Stage 분할
5. Tasks             : 각 Stage가 파티션 수만큼 Task로 분할
```

```
[Stage 0]                    [Stage 1]
 read.csv                     groupBy
   │                           │
 filter ──┐ Shuffle Boundary  count
   │      └──────🌐───────────►│
 select                        orderBy
                               │
                              show ✓ (Action)
```

> **외우세요**: "**Shuffle 경계 = Stage 경계**". 이걸 알면 Spark UI를 읽을 수 있습니다.

---

# Part 3. Spark 아키텍처 심층 이해

## 3.1 전체 아키텍처 한눈에 보기

```
┌──────────────────────────────────────────────────────────────────┐
│                         Cluster Manager                          │
│   (Kubernetes / YARN / Standalone / Mesos[deprecated])          │
└──────────────┬─────────────────────────┬─────────────────────────┘
               │ 리소스 요청              │ 리소스 할당
               ▼                          ▼
       ┌──────────────┐          ┌─────────────────────────┐
       │   Driver     │  ◄────►  │   Worker Nodes          │
       │   Program    │  Task    │  ┌────────┐ ┌────────┐  │
       │              │  분배    │  │Executor│ │Executor│  │
       │ SparkContext │          │  │  JVM   │ │  JVM   │  │
       └──────────────┘          │  │┌──┐┌──┐│ │┌──┐┌──┐│  │
                                 │  ││Tk││Tk││ ││Tk││Tk││  │
                                 │  │└──┘└──┘│ │└──┘└──┘│  │
                                 │  └────────┘ └────────┘  │
                                 └─────────────────────────┘
```

## 3.2 Cluster Manager의 역할

Spark 자체는 **계산 엔진**입니다. 노드를 어떻게 할당받고 관리할지는 Cluster Manager에게 맡깁니다.

| Manager | 특징 | 적합 환경 |
|---------|------|-----------|
| **Standalone** | Spark 자체 내장 | 학습용, 단순 환경 |
| **YARN** | Hadoop 생태계 | 전통적 빅데이터 환경 |
| **Kubernetes** | 컨테이너 오케스트레이션 | **현대 클라우드 / MLOps** |
| Mesos | 범용 | (Spark 3.5에서 deprecated, 4.0에서 제거) |

이 강의의 실습은 모두 **Kubernetes** 기반입니다.

## 3.3 Spark on Kubernetes 동작 흐름

```
1. 사용자: spark-submit --master k8s://https://api-server ...
                              │
2. K8s API Server: Driver Pod 생성 요청 수신
                              │
3. Driver Pod 가 시작됨 (사용자 코드 실행 시작)
                              │
4. Driver 가 Executor Pod들을 K8s에 요청 (Spark 자체가 K8s 클라이언트 역할)
                              │
5. K8s: Executor Pod들을 노드에 스케줄링 + 실행
                              │
6. Driver ←→ Executor 작업 분배 및 결과 수집
                              │
7. 작업 완료 시 Executor Pod 정리, Driver는 옵션에 따라 유지/제거
```

## 3.4 메모리 관리 — Executor 안의 메모리 구조

Executor 1개의 메모리는 다음과 같이 나뉩니다.

```
[Executor JVM Heap]
┌─────────────────────────────────────────────┐
│ Reserved Memory (300MB, 시스템 예약)          │
├─────────────────────────────────────────────┤
│ User Memory (사용자 코드, UDF 등)             │
│ = (Heap - 300MB) × (1 - spark.memory.fraction)│
├─────────────────────────────────────────────┤
│ Spark Memory                                │
│ = (Heap - 300MB) × spark.memory.fraction    │
│ ┌───────────────────────────────────────┐   │
│ │ Storage Memory (cache, broadcast)      │   │
│ │ Execution Memory (shuffle, join, sort)│   │
│ │ → Storage와 Execution이 동적 공유       │   │
│ └───────────────────────────────────────┘   │
└─────────────────────────────────────────────┘

[Off-Heap Memory] — Tungsten 사용 시
- spark.memory.offHeap.enabled = true
- 이 영역은 GC 영향 X
```

### 튜닝 가이드

| 파라미터 | 기본값 | 의미 |
|---------|--------|------|
| `spark.executor.memory` | 1g | Executor JVM Heap |
| `spark.executor.memoryOverhead` | max(384MB, 10%) | JVM 외 영역 (Native, Python 워커) |
| `spark.memory.fraction` | 0.6 | Heap 중 Spark 영역 비율 |
| `spark.memory.storageFraction` | 0.5 | Storage 비율 (나머지는 Execution) |

> **PySpark 주의**: Python 코드는 Executor JVM 옆에 별도 Python 워커 프로세스로 실행됩니다. 이 메모리는 `memoryOverhead`에서 옵니다. PyTorch나 Transformers를 UDF로 쓰면 overhead를 충분히 늘려야 합니다 (예: 4g 이상).

---

# Part 4. Spark 컴포넌트 라이브러리

Spark는 단일 엔진 위에 4개의 큰 라이브러리를 얹은 통합 플랫폼입니다.

```
┌──────────────────────────────────────────────────────────────┐
│  Spark SQL  │ Structured Streaming │  MLlib  │  GraphX/GraphFrames │
├──────────────────────────────────────────────────────────────┤
│            DataFrame / Dataset API (공통 인터페이스)            │
├──────────────────────────────────────────────────────────────┤
│              Spark Core (RDD, 스케줄러, 메모리 관리)            │
└──────────────────────────────────────────────────────────────┘
```

## 4.1 Spark SQL — 가장 많이 쓰는 컴포넌트

SQL을 그대로 쓸 수 있고, DataFrame API와 자유롭게 혼용할 수 있습니다.

```python
df.createOrReplaceTempView("people")
result = spark.sql("""
    SELECT city, COUNT(*) AS cnt
    FROM people
    WHERE age > 30
    GROUP BY city
    ORDER BY cnt DESC
""")
```

지원 포맷: Parquet (권장), ORC, Avro, JSON, CSV, Delta Lake, Iceberg, Hudi 등.

> **현업 팁**: 분석/배치는 99% Parquet 또는 Delta Lake로 갑니다. CSV는 학습용/임시용입니다.

## 4.2 Structured Streaming — 스트림을 테이블처럼

Spark의 스트림 처리는 **"끝없이 늘어나는 테이블에 SQL을 거는 것"** 으로 추상화됩니다. Kafka 토픽을 DataFrame처럼 다룰 수 있습니다.

```python
stream = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka:9092")
    .option("subscribe", "events")
    .load())

# 일반 DataFrame처럼 변환
agg = stream.groupBy("user_id").count()

# 결과 싱크에 출력
query = (agg.writeStream
    .format("console")
    .outputMode("update")
    .start())
```

## 4.3 MLlib — 분산 머신러닝

대용량 데이터에 대한 클래식 ML(로지스틱 회귀, 랜덤 포레스트, K-means, ALS 추천 등)을 분산으로 학습할 수 있습니다.

> **현실**: 딥러닝/LLM 시대에 MLlib의 비중은 줄었습니다. 그러나 **피처 엔지니어링** (Tokenizer, OneHotEncoder, VectorAssembler, StandardScaler)은 여전히 강력합니다. ML 파이프라인의 전처리 단계로 자주 사용됩니다.

## 4.4 GraphX / GraphFrames

그래프 분석(PageRank, Connected Components, BFS 등). LLM 컨텍스트에서는 **지식 그래프 구축, 엔티티 연결 분석** 등에 활용됩니다.

---

# Part 5. PySpark 기본 사용법

## 5.1 SparkSession 생성 — 모든 것의 출발

```python
from pyspark.sql import SparkSession

spark = (SparkSession.builder
    .appName("MyFirstSparkApp")
    .master("local[*]")            # local: 단일 머신, [*]: 모든 코어 사용
    .config("spark.sql.shuffle.partitions", "200")
    .config("spark.executor.memory", "4g")
    .getOrCreate())

# 로그 레벨 조정 (운영 시 INFO → WARN/ERROR 권장)
spark.sparkContext.setLogLevel("WARN")
```

## 5.2 DataFrame 만들기 4가지 방법

```python
# 방법 1: 파이썬 리스트에서
data = [(1, "Alice", 30), (2, "Bob", 25)]
df = spark.createDataFrame(data, schema=["id", "name", "age"])

# 방법 2: 파일에서 (CSV)
df = spark.read.option("header", True).option("inferSchema", True).csv("data.csv")

# 방법 3: 파일에서 (Parquet — 권장)
df = spark.read.parquet("data.parquet")

# 방법 4: SQL 결과
df = spark.sql("SELECT * FROM my_table")
```

## 5.3 DataFrame 핵심 연산 ― 한 페이지로 정리

```python
from pyspark.sql import functions as F

# 보기
df.show(5)
df.printSchema()
df.describe().show()

# Select / Project
df.select("name", "age").show()
df.select(F.col("name"), (F.col("age") + 1).alias("age_plus")).show()

# Filter
df.filter(df.age > 30).show()
df.where("age > 30 AND city = 'Seoul'").show()

# 컬럼 추가/수정
df = df.withColumn("age_group",
    F.when(F.col("age") < 20, "teen")
     .when(F.col("age") < 40, "adult")
     .otherwise("senior"))

# 그룹 집계
(df.groupBy("city")
   .agg(F.count("*").alias("cnt"),
        F.avg("age").alias("avg_age"))
   .orderBy(F.desc("cnt"))
   .show())

# Join
df_users.join(df_orders, on="user_id", how="left")

# Window 함수 (현업 빈출)
from pyspark.sql.window import Window
w = Window.partitionBy("city").orderBy(F.desc("age"))
df.withColumn("rank_in_city", F.row_number().over(w)).show()

# 저장
df.write.mode("overwrite").parquet("output/")
df.write.mode("append").format("delta").save("output_delta/")
```

## 5.4 UDF (User Defined Function) — 주의해서 쓰기

UDF는 강력하지만 **Catalyst 최적화를 우회**해서 느립니다. 가능하면 내장 함수를 쓰세요.

```python
# ❌ 느린 일반 UDF
from pyspark.sql.types import StringType

@F.udf(returnType=StringType())
def my_upper(s):
    return s.upper() if s else None

df.withColumn("name_upper", my_upper("name"))

# ✅ 훨씬 빠른 내장 함수
df.withColumn("name_upper", F.upper("name"))

# ✅ 빠른 UDF가 필요하면 Pandas UDF 사용
@F.pandas_udf("string")
def my_upper_pandas(s: pd.Series) -> pd.Series:
    return s.str.upper()
```

> **기억할 것**: Python UDF < Pandas UDF (=Vectorized UDF) < Built-in 함수 순으로 빠릅니다. LLM 임베딩 추론처럼 **꼭 Python이 필요한 경우만** UDF를 쓰고, 가능하면 Pandas UDF로 작성하세요.

## 5.5 캐싱 ― 같은 데이터 재사용 시

```python
df = spark.read.parquet("huge.parquet").filter("year = 2026")
df.cache()    # 메모리에 저장
df.count()    # Action — 여기서 캐시 채워짐

# 이후 df를 5번 더 사용해도 디스크 I/O 없음
df.groupBy("city").count().show()
df.groupBy("country").count().show()

df.unpersist()   # 끝났으면 해제
```

`cache()`는 `persist(StorageLevel.MEMORY_AND_DISK)`의 별칭입니다. 메모리 부족 시 디스크로 흘러갑니다.

---

# Part 6. Spark on Kubernetes 이해

## 6.1 왜 Kubernetes인가?

전통적인 Hadoop YARN 대신 K8s를 쓰는 이유:

1. **클라우드 네이티브** : 온프레미스, AWS, GCP, Azure 어디서나 동일하게 동작
2. **컨테이너 격리** : 서로 다른 Spark 버전, 다른 라이브러리 버전을 한 클러스터에서 동시 운영
3. **자동 스케일링** : Cluster Autoscaler + KEDA 조합으로 자원 효율 극대화
4. **GitOps** : ArgoCD/Flux로 Spark 작업도 선언형 배포 가능
5. **MLOps 통합** : Kubeflow, KServe, Argo Workflows와 자연스럽게 연동
6. **GPU 지원** : NVIDIA Device Plugin, MIG, Time-Slicing 등 풍부한 GPU 활용

## 6.2 Spark on K8s 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                            │
│                                                                  │
│  ┌─────────────┐      ┌──────────────────────────────────┐      │
│  │             │      │     spark-operator-namespace     │      │
│  │ Control     │      │  ┌────────────────────────────┐  │      │
│  │ Plane       │      │  │  Spark Operator (Pod)      │  │      │
│  │             │      │  │  - SparkApplication CRD 감시 │  │      │
│  └─────────────┘      │  │  - spark-submit 자동 실행    │  │      │
│                       │  └────────────────────────────┘  │      │
│                       └──────────────────────────────────┘      │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              spark-apps namespace                        │    │
│  │  ┌─────────────┐                                         │    │
│  │  │ Driver Pod  │ ◄──── 사용자 코드 실행                    │    │
│  │  │ (sparkpi)   │                                         │    │
│  │  └──────┬──────┘                                         │    │
│  │         │ Executor Pod 요청                               │    │
│  │         ▼                                                │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │    │
│  │  │Executor 1│ │Executor 2│ │Executor 3│ │Executor 4│   │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## 6.3 두 가지 실행 방식

### 방식 1: spark-submit 직접 호출 (Native)

가장 단순합니다. 수동으로 spark-submit을 실행하면 Spark가 K8s API에 직접 Pod를 만듭니다.

```bash
spark-submit \
  --master k8s://https://kubernetes.default.svc:443 \
  --deploy-mode cluster \
  --name spark-pi \
  --class org.apache.spark.examples.SparkPi \
  --conf spark.executor.instances=3 \
  --conf spark.kubernetes.container.image=apache/spark:4.1.1 \
  --conf spark.kubernetes.namespace=spark-apps \
  local:///opt/spark/examples/jars/spark-examples_2.13-4.1.1.jar 1000
```

장점: 단순, 추가 설치 없음
단점: 선언형이 아님, 작업 라이프사이클 관리 어려움

### 방식 2: Spark Operator (CRD 기반) — **이 강의에서 사용**

K8s CRD를 통해 Spark 작업을 **선언형(declarative)** 으로 정의합니다.

```yaml
apiVersion: spark.apache.org/v1alpha1
kind: SparkApplication
metadata:
  name: spark-pi
spec:
  mainClass: org.apache.spark.examples.SparkPi
  jars: "local:///opt/spark/examples/jars/spark-examples.jar"
  driver:
    cores: 1
    memory: 1g
  executor:
    instances: 3
    cores: 1
    memory: 1g
```

장점: GitOps 친화적, Operator가 라이프사이클 관리, 재시도/스케줄링/모니터링 통합
단점: CRD 설치 필요

### 두 Operator의 차이 — 중요!

현재 두 가지 Operator가 존재합니다.

| 항목 | **Apache Spark K8s Operator (공식)** | **Kubeflow Spark Operator** |
|------|-------------------------------------|----------------------------|
| 출처 | Apache Software Foundation 산하 공식 서브 프로젝트 | 원래 Google이 만들고 Kubeflow로 이관 |
| 시작 | 2025년 신규, v0.7.0 (2026-01) | 오래됨, 대규모 배포 실적 다수 |
| API 그룹 | `spark.apache.org/v1alpha1` | `sparkoperator.k8s.io/v1beta2` |
| 요구 사항 | Spark 3.5+, K8s 1.32+ | Spark 2.3+, 더 폭넓은 호환 |
| Helm 차트 | `spark-kubernetes-operator` | `spark-operator/spark-operator` |
| Spark Connect 지원 | ✓ (1순위 기능) | 부분 |
| 권장 | 신규 프로젝트, 4.x 사용 시 | 운영 안정성, 대규모 검증 사례 필요 시 |

> **이 강의에서는 새로운 Apache 공식 Operator를 기본으로 다루고**, 운영 안정성이 중요한 경우 Kubeflow Operator 설치 방법도 함께 안내합니다.

---

# Part 7. 실습 환경 구성 — DGX Spark + ThinkStation

이제 본격적인 실습입니다. 이 파트는 매우 구체적이고 실전적입니다.

## 7.1 하드웨어 사양 정리

### Node 1: NVIDIA DGX Spark (이름과 제품 헷갈리지 말 것!)

| 항목 | 사양 |
|------|------|
| SoC | NVIDIA GB10 Grace Blackwell Superchip |
| CPU | **20-core ARM** (10× Cortex-X925 @ 4.0GHz + 10× Cortex-A725 @ 2.8GHz, Armv9) |
| GPU | NVIDIA Blackwell (5세대 Tensor Core, FP4/FP8 지원) |
| 메모리 | 128GB LPDDR5X (CPU/GPU **통합 메모리, 273GB/s**) |
| 스토리지 | 4TB NVMe SSD |
| 네트워크 | NVIDIA ConnectX-7 (200GbE QSFP56) + 10GbE RJ45 + WiFi 7 |
| OS | **DGX OS (Ubuntu 24.04 LTS 기반)** |
| 아키텍처 | **ARM64 (aarch64)** ← 매우 중요 |
| 성능 | 최대 1 PFLOP @ FP4, 1000 TOPS |

> **⚠️ 명칭 혼동 주의**: 본 강의의 주제는 *"Apache Spark"* 이고, 하드웨어는 *"NVIDIA **DGX** Spark"* 입니다. 둘 다 'Spark'를 쓰지만 서로 무관한 명칭입니다. NVIDIA DGX Spark는 Apache Spark를 위한 머신이 아닙니다 — 본래 LLM 로컬 개발용 워크스테이션이지만, **이 강의에서는 그 위에 K8s를 깔고 Apache Spark를 실행하는 것**이 목표입니다.

### Node 2: Lenovo ThinkStation (예상 사양)

| 항목 | 일반적 사양 |
|------|------------|
| CPU | Intel Xeon W / Core i9, **x86_64** |
| GPU | (모델에 따라) RTX 시리즈 또는 없음 |
| 메모리 | 32~256GB DDR5 |
| OS | Ubuntu 24.04 LTS 권장 |
| 아키텍처 | **x86_64 (amd64)** ← 매우 중요 |

## 7.2 핵심 도전 과제: **이종(Heterogeneous) 아키텍처 클러스터**

이 환경의 가장 큰 특징은 **두 노드의 CPU 아키텍처가 다르다**는 점입니다.

```
DGX Spark    : ARM64 (aarch64)   ← Apple Silicon, AWS Graviton 계열
ThinkStation : x86_64 (amd64)    ← 일반 PC/서버
```

이로 인해 발생하는 문제:

1. **컨테이너 이미지가 멀티 아키텍처여야 함** : 한 이미지가 두 아키텍처 모두에서 실행되어야 함 (Multi-arch manifest)
2. **노드 어피니티(affinity) 설정 필요** : 특정 작업을 특정 아키텍처에만 보낼 수 있어야 함
3. **GPU 작업은 DGX로** : Blackwell GPU는 DGX에만 있음 → GPU 작업은 ARM64 이미지 필요
4. **Java/Python 호환성** : Spark는 ARM64를 공식 지원하지만, **JVM 위에서 도는 일부 native 라이브러리(Snappy, LZ4 등)는 점검 필요** 합니다.

## 7.3 K8s 클러스터 설치 (k3s 권장)

2노드 환경에서는 **k3s**가 가장 편리합니다. 가볍고, etcd 대신 SQLite를 쓰며, 단일 바이너리로 설치됩니다.

### 7.3.1 사전 준비 (양쪽 노드 모두)

```bash
# 양쪽 모두 실행
sudo apt update
sudo apt install -y curl wget vim net-tools

# Swap 비활성화 (K8s 필수)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 호스트명 명확히 설정
# DGX Spark에서:
sudo hostnamectl set-hostname dgx-spark
# ThinkStation에서:
sudo hostnamectl set-hostname thinkstation

# /etc/hosts 양쪽에 추가
sudo tee -a /etc/hosts <<EOF
192.168.1.10  dgx-spark
192.168.1.20  thinkstation
EOF
```

### 7.3.2 컨테이너 런타임 (containerd)

k3s는 containerd를 내장하므로 별도 설치 불필요. 다만 GPU를 쓰려면 nvidia-container-toolkit 설정이 필요합니다.

**DGX Spark에서만:**

```bash
# NVIDIA Container Toolkit 설치 (DGX OS는 보통 이미 설치되어 있음)
distribution=$(. /etc/os-release; echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# GPU 동작 확인
nvidia-smi
```

### 7.3.3 k3s 서버 설치 (Master = DGX Spark 권장)

DGX Spark가 메모리/성능이 우수하므로 컨트롤 플레인으로 두는 것을 추천합니다.

```bash
# DGX Spark에서 실행 (Master)
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --disable traefik \
  --node-label node.kubernetes.io/arch=arm64 \
  --node-label hardware=dgx-spark \
  --node-label nvidia.com/gpu=true

# 토큰 가져오기
sudo cat /var/lib/rancher/k3s/server/node-token
# 출력 예: K10abcdef...::server:xyz...
```

### 7.3.4 k3s 워커 등록 (ThinkStation)

```bash
# ThinkStation에서 실행 (Worker)
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.10:6443 \
  K3S_TOKEN=<위에서 받은 토큰> sh -s - \
  --node-label node.kubernetes.io/arch=amd64 \
  --node-label hardware=thinkstation
```

### 7.3.5 클러스터 확인

```bash
# DGX Spark에서
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config

kubectl get nodes -o wide --show-labels

# 예상 출력:
# NAME          STATUS   ROLES         AGE   VERSION   ...   ARCH
# dgx-spark     Ready    control-plane 5m    v1.32.x   ...   arm64
# thinkstation  Ready    <none>        2m    v1.32.x   ...   amd64
```

## 7.4 GPU Device Plugin 설치 (DGX Spark용)

```bash
# NVIDIA k8s-device-plugin 설치
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.17.0/deployments/static/nvidia-device-plugin.yml

# DaemonSet이 ARM64 노드에도 떠야 합니다.
# 만약 amd64만 지원하는 이미지로 떴다면, NVIDIA가 제공하는 multi-arch 이미지를 명시:
# image: nvcr.io/nvidia/k8s-device-plugin:v0.17.0
```

GPU 노드 확인:

```bash
kubectl describe node dgx-spark | grep -A 5 "Capacity:"
# nvidia.com/gpu: 1 이렇게 보이면 성공
```

## 7.5 노드 라벨링 전략 (이 강의의 핵심)

이종 클러스터에서는 **어떤 작업을 어디로 보낼지 명확히 정의**해야 합니다.

```bash
# 추가 라벨 부여
kubectl label node dgx-spark workload-type=gpu
kubectl label node dgx-spark spark-role=heavy
kubectl label node thinkstation workload-type=cpu
kubectl label node thinkstation spark-role=light

# (선택) DGX의 GPU 자원이 부족하지 않게 일반 작업 차단
# Taint: GPU 작업만 받겠다
kubectl taint nodes dgx-spark dedicated=gpu:NoSchedule
# 일반 Spark 작업은 toleration이 없으면 ThinkStation으로 갑니다.
```

이렇게 해두면 다음 패턴이 가능해집니다.

| 작업 유형 | 배치 노드 | 이유 |
|----------|----------|------|
| Spark Driver | ThinkStation | 가벼운 코디네이션 작업 |
| Spark Executor (CPU) | ThinkStation | 일반 ETL, 그룹바이 |
| Spark Executor (GPU) | DGX Spark | RAPIDS, LLM 임베딩 |
| Spark Operator 자체 | ThinkStation | 항상 살아있어야 함 |

---
# Part 8. Spark Operator 설치

## 8.1 Helm 설치 (양쪽 모두 필요한 것은 아님, 배포 머신에 1번만)

```bash
# Helm 3 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## 8.2 옵션 A: Apache Spark Kubernetes Operator (공식, 신규 - 권장)

```bash
# Helm Repo 추가
helm repo add spark https://apache.github.io/spark-kubernetes-operator
helm repo update

# 네임스페이스 생성
kubectl create namespace spark-operator
kubectl create namespace spark-apps

# 설치
helm install spark spark/spark-kubernetes-operator \
  --namespace spark-operator

# 확인
helm list -n spark-operator
kubectl get pods -n spark-operator
kubectl get crd | grep spark.apache.org
```

설치 직후 다음 CRD가 등록되어야 합니다:
- `sparkapplications.spark.apache.org`
- `sparkclusters.spark.apache.org`

## 8.3 옵션 B: Kubeflow Spark Operator (운영 안정 버전)

```bash
helm repo add spark-operator https://kubeflow.github.io/spark-operator
helm repo update

helm install spark-operator spark-operator/spark-operator \
  --namespace spark-operator \
  --create-namespace \
  --set sparkJobNamespace=spark-apps \
  --set webhook.enable=true

# 설치 확인
kubectl get pods -n spark-operator
kubectl get crd | grep sparkoperator.k8s.io
```

CRD: `sparkapplications.sparkoperator.k8s.io` (v1beta2)

## 8.4 RBAC 설정 (Spark 작업이 Pod를 만들 권한 필요)

```bash
# Service Account 만들기
kubectl create serviceaccount spark -n spark-apps

# 권한 바인딩 (Spark 작업이 Executor Pod를 생성할 수 있게)
kubectl create clusterrolebinding spark-role \
  --clusterrole=edit \
  --serviceaccount=spark-apps:spark
```

## 8.5 멀티아키텍처 컨테이너 이미지 — 가장 중요한 부분

이종 클러스터에서는 한 이미지가 ARM64와 x86_64 양쪽에서 동작해야 합니다. Docker `buildx`를 사용합니다.

### 8.5.1 buildx 설치 및 활성화

```bash
docker buildx create --name multiarch --use --platform linux/amd64,linux/arm64
docker buildx inspect --bootstrap
```

### 8.5.2 베이스 이미지 — Apache 공식이 multi-arch 지원

다행히도 **Apache Spark 공식 도커 이미지는 이미 multi-arch**입니다.

```bash
docker manifest inspect apache/spark:4.1.1
# linux/amd64, linux/arm64 모두 지원함을 확인할 수 있습니다.
```

### 8.5.3 커스텀 이미지 빌드 (예: 패키지 추가)

`Dockerfile`:

```dockerfile
FROM apache/spark:4.1.1

USER root

# Python 패키지 추가
RUN pip install --no-cache-dir \
    pandas==2.2.* \
    pyarrow==17.0.* \
    numpy==1.26.* \
    scikit-learn==1.5.* \
    transformers==4.45.* \
    sentence-transformers==3.1.*

# 작업 디렉터리
WORKDIR /opt/spark/work-dir

USER spark
```

빌드 + 푸시:

```bash
# 로컬 레지스트리를 두면 편합니다 (양쪽 노드에서 접근 가능)
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# 멀티아키 빌드 + 푸시
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t 192.168.1.10:5000/spark-custom:4.1.1 \
  --push .
```

> **DGX Spark의 통합 메모리(128GB)가 빌드에도 큰 장점**입니다. 빌드는 DGX에서 수행하고 ThinkStation은 사용만 하는 패턴을 권장합니다.

### 8.5.4 k3s에서 insecure registry 허용

```bash
# 양쪽 노드의 /etc/rancher/k3s/registries.yaml 작성
sudo tee /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "192.168.1.10:5000":
    endpoint:
      - "http://192.168.1.10:5000"
EOF

# k3s 재시작
sudo systemctl restart k3s        # Master에서
sudo systemctl restart k3s-agent  # Worker에서
```

---

# Part 9. 첫 번째 Spark 작업 실행

## 9.1 Hello, Spark on K8s — Pi 계산

### 9.1.1 SparkApplication YAML 작성

`spark-pi.yaml` (Apache Spark Operator 사용):

```yaml
apiVersion: spark.apache.org/v1alpha1
kind: SparkApplication
metadata:
  name: spark-pi
  namespace: spark-apps
spec:
  runtimeVersions:
    sparkVersion: "4.1.1"
  applicationTolerations:
    instanceConfig:
      initExecutors: 2
      minExecutors: 2
      maxExecutors: 4
  driverSpec:
    podTemplateSpec:
      spec:
        containers:
          - name: spark-kubernetes-driver
            image: apache/spark:4.1.1
            resources:
              requests:
                cpu: "500m"
                memory: "1Gi"
              limits:
                cpu: "1"
                memory: "2Gi"
        # ThinkStation에 강제 배치
        nodeSelector:
          kubernetes.io/arch: amd64
        serviceAccountName: spark
  executorSpec:
    podTemplateSpec:
      spec:
        containers:
          - name: spark-kubernetes-executor
            image: apache/spark:4.1.1
            resources:
              requests:
                cpu: "1"
                memory: "1Gi"
              limits:
                cpu: "1"
                memory: "2Gi"
        nodeSelector:
          kubernetes.io/arch: amd64
  mainClass: "org.apache.spark.examples.SparkPi"
  jars: "local:///opt/spark/examples/jars/spark-examples_2.13-4.1.1.jar"
  sparkConf:
    "spark.executor.instances": "2"
```

> **⚠️ 주의**: Apache 공식 Operator는 신규(v0.7.x)이므로 spec 필드가 향후 변경될 수 있습니다. 정확한 최신 스키마는 `kubectl explain sparkapplication.spec` 로 확인하세요. Kubeflow Operator는 아래 9.1.2의 더 안정된 스키마를 사용합니다.

### 9.1.2 Kubeflow Operator 사용 시 — 안정적이고 검증된 스키마

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi
  namespace: spark-apps
spec:
  type: Scala
  mode: cluster
  image: apache/spark:4.1.1
  imagePullPolicy: IfNotPresent
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples_2.13-4.1.1.jar
  sparkVersion: "4.1.1"
  arguments:
    - "1000"
  driver:
    cores: 1
    memory: "1g"
    serviceAccount: spark
    nodeSelector:
      kubernetes.io/arch: amd64
    labels:
      version: 4.1.1
  executor:
    cores: 1
    instances: 2
    memory: "1g"
    nodeSelector:
      kubernetes.io/arch: amd64
    labels:
      version: 4.1.1
  restartPolicy:
    type: Never
```

### 9.1.3 실행 및 모니터링

```bash
# 제출
kubectl apply -f spark-pi.yaml

# 상태 확인
kubectl get sparkapp -n spark-apps
kubectl describe sparkapp spark-pi -n spark-apps

# Pod 확인 — Driver와 Executor가 차례로 뜸
kubectl get pods -n spark-apps -w

# 로그 확인 (결과: "Pi is roughly 3.1415...")
kubectl logs -n spark-apps spark-pi-driver | grep "Pi is"

# 작업 정리
kubectl delete sparkapp spark-pi -n spark-apps
```

## 9.2 PySpark 작업 작성 — Word Count 실전

`wordcount.py`:

```python
import sys
from pyspark.sql import SparkSession
from pyspark.sql.functions import explode, split, lower, regexp_replace, col

def main():
    spark = (SparkSession.builder
        .appName("WordCount")
        .getOrCreate())

    # 입력: 텍스트 파일 (Driver Pod에서 접근 가능한 경로)
    input_path = sys.argv[1] if len(sys.argv) > 1 else "/opt/spark/data/sample.txt"

    df = spark.read.text(input_path)

    words = (df.select(
                explode(split(lower(regexp_replace("value", r"[^a-zA-Z\s]", "")), r"\s+"))
                .alias("word"))
             .filter(col("word") != ""))

    counts = (words.groupBy("word")
              .count()
              .orderBy(col("count").desc()))

    counts.show(20, truncate=False)

    spark.stop()

if __name__ == "__main__":
    main()
```

### 9.2.1 PySpark용 이미지 만들기

```dockerfile
# Dockerfile.pyspark
FROM apache/spark:4.1.1

USER root
RUN pip install --no-cache-dir pandas pyarrow

# 스크립트 복사
COPY wordcount.py /opt/spark/work-dir/wordcount.py
COPY sample.txt /opt/spark/data/sample.txt

USER spark
```

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t 192.168.1.10:5000/pyspark-wordcount:1.0 \
  --push .
```

### 9.2.2 SparkApplication

`pyspark-wordcount.yaml` (Kubeflow Operator):

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: pyspark-wc
  namespace: spark-apps
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: 192.168.1.10:5000/pyspark-wordcount:1.0
  imagePullPolicy: Always
  mainApplicationFile: local:///opt/spark/work-dir/wordcount.py
  arguments:
    - "/opt/spark/data/sample.txt"
  sparkVersion: "4.1.1"
  driver:
    cores: 1
    memory: "1g"
    serviceAccount: spark
    nodeSelector:
      kubernetes.io/arch: amd64
  executor:
    cores: 2
    instances: 2
    memory: "2g"
    # Executor를 양쪽 노드에 분산하려면 nodeSelector 제거하고
    # multi-arch 이미지 사용
  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
```

```bash
kubectl apply -f pyspark-wordcount.yaml
kubectl logs -f -n spark-apps pyspark-wc-driver
```

## 9.3 Spark UI 접근

```bash
# 포트포워딩으로 Driver의 4040 포트 접근
kubectl port-forward -n spark-apps pyspark-wc-driver 4040:4040
# 브라우저: http://localhost:4040
```

Spark UI에서 확인할 것:
- **Jobs** 탭: 작업 단위 진행 상황
- **Stages** 탭: 셔플 발생 위치
- **Storage** 탭: 캐시된 데이터
- **Executors** 탭: 노드별 자원 사용량
- **SQL/DataFrame** 탭: 쿼리 실행 계획

## 9.4 Driver를 ThinkStation으로, Executor를 양쪽으로 분산

```yaml
# 일부 발췌
spec:
  driver:
    nodeSelector:
      kubernetes.io/arch: amd64    # Driver는 항상 ThinkStation
  executor:
    instances: 4
    # nodeSelector 미지정 → 양쪽 노드 모두에 배포
    # 단, DGX에 taint를 걸어둔 경우 toleration 필요:
    tolerations:
      - key: dedicated
        operator: Equal
        value: gpu
        effect: NoSchedule
```

이렇게 하면 Executor 4개가 ARM64/x86_64 노드에 자동 분산됩니다. **Spark는 노드 아키텍처 차이를 인지하지 않으므로 멀티아치 이미지가 필수**입니다.

---
# Part 10. GPU 가속 — RAPIDS Accelerator

## 10.1 RAPIDS Accelerator for Apache Spark란?

NVIDIA가 만든 **Spark용 GPU 가속 플러그인**입니다. **단 한 줄의 설정 변경**으로 SQL/DataFrame 연산을 GPU에서 실행하게 만들 수 있습니다 — 코드 수정 0.

```
일반 Spark         RAPIDS Accelerator
┌──────────┐       ┌──────────────────┐
│DataFrame │       │ DataFrame        │
│   ↓      │       │   ↓              │
│ Catalyst │       │ Catalyst+RAPIDS  │
│   ↓      │       │   ↓              │
│ JVM/CPU  │  →→→  │ GPU (Blackwell)  │
└──────────┘       └──────────────────┘
                   조인/집계/정렬 → 5~10배 빨라짐
```

DGX Spark의 Blackwell GPU + 통합 메모리 273GB/s는 **CPU-GPU 메모리 복사 오버헤드를 사실상 0**으로 만들기 때문에 RAPIDS 효과가 일반 GPU 환경보다도 큽니다.

## 10.2 RAPIDS 적용 SparkApplication

`pyspark-rapids.yaml`:

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-rapids-demo
  namespace: spark-apps
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: nvcr.io/nvidia/spark-rapids:25.10-spark-4.1
  imagePullPolicy: Always
  mainApplicationFile: local:///opt/spark/work-dir/etl.py
  sparkVersion: "4.1.1"
  driver:
    cores: 1
    memory: "2g"
    serviceAccount: spark
    nodeSelector:
      kubernetes.io/arch: amd64        # Driver는 ThinkStation
  executor:
    cores: 4
    instances: 1                       # GPU는 1개라 보통 1 executor
    memory: "8g"
    gpu:
      name: "nvidia.com/gpu"
      quantity: 1
    nodeSelector:
      kubernetes.io/arch: arm64        # GPU 작업은 DGX Spark
      nvidia.com/gpu: "true"
    tolerations:
      - key: dedicated
        operator: Equal
        value: gpu
        effect: NoSchedule
  sparkConf:
    "spark.plugins": "com.nvidia.spark.SQLPlugin"
    "spark.rapids.sql.enabled": "true"
    "spark.rapids.sql.explain": "ALL"
    "spark.executor.resource.gpu.amount": "1"
    "spark.task.resource.gpu.amount": "0.25"   # 1 GPU를 4 task가 공유
    "spark.executor.resource.gpu.discoveryScript": "/opt/sparkRapidsPlugin/getGpusResources.sh"
  restartPolicy:
    type: Never
```

> 위 이미지 태그는 예시이며, 실제 사용 시 https://catalog.ngc.nvidia.com 에서 ARM64 지원 태그를 확인해 사용하세요. RAPIDS는 ARM64 지원을 점진적으로 확대해 왔으므로 Blackwell + ARM 조합은 사용 전 호환성 확인이 필수입니다.

## 10.3 어떤 연산이 GPU에서 빨라지는가?

| 연산 | GPU 가속 효과 |
|------|---------------|
| `groupBy` 집계 | 매우 빠름 (5~20배) |
| `join` (broadcast/shuffle) | 매우 빠름 |
| `orderBy` / window 함수 | 빠름 |
| 문자열 처리 | 보통 |
| Python UDF | **GPU 사용 안 됨** (JVM Catalyst 외부) |
| `read.parquet` (대용량) | 빠름 (cuDF parser) |

## 10.4 성능 비교 실습 시나리오

같은 작업을 두 번 실행해 비교합니다.

1. **CPU만** (Executor를 ThinkStation에 배치, RAPIDS 비활성화)
2. **GPU 가속** (DGX Spark의 Blackwell GPU, RAPIDS 활성화)

대략적인 기대치 (DGX Spark에서):

```
TPC-H 벤치마크 Q1, 10GB:
  CPU 8-core:        45초
  Blackwell + RAPIDS: 6초  (~7배)
```

> **현실 체크**: 항상 GPU가 빠른 건 아닙니다. 데이터가 작거나(10MB 미만), 단순 필터만 있다면 CPU가 더 빠를 수 있습니다 (커널 런치 오버헤드). 적절한 워크로드 크기에서 진가를 발휘합니다.

---

# Part 11. 실전 ML/LLM 데이터 파이프라인

LLMOps 엔지니어가 실제로 마주치는 시나리오입니다.

## 11.1 시나리오: 대규모 텍스트 코퍼스 정제 + 임베딩

```
[Raw Web Crawl 100GB+]
       │
       ▼
  [Spark on K8s]
   1. Parquet 변환    ← ThinkStation Executor (CPU)
   2. 언어 감지/필터  ← ThinkStation
   3. 중복 제거       ← ThinkStation (대용량 셔플)
   4. PII 마스킹      ← ThinkStation
   5. 임베딩 생성     ← DGX Spark (GPU + Pandas UDF)
       │
       ▼
[정제된 + 임베딩된 Parquet]
```

## 11.2 코드 예시 — `pipeline.py`

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import ArrayType, FloatType
import pandas as pd

spark = (SparkSession.builder
    .appName("LLMDataPipeline")
    .config("spark.sql.adaptive.enabled", "true")
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
    .getOrCreate())

# === Stage 1-3: 정제 (CPU 노드에서) ===
df = (spark.read.parquet("s3a://raw/web-crawl/")
      .filter(F.length("text") > 100)
      .filter(F.length("text") < 100000)
      .dropDuplicates(["url"]))

# 중복 제거 (콘텐츠 해시 기반)
df = df.withColumn("content_hash", F.sha2("text", 256)) \
       .dropDuplicates(["content_hash"])

# PII 마스킹 (간단 정규식 예시)
df = df.withColumn("text",
    F.regexp_replace("text", r"\b\d{3}-\d{2}-\d{4}\b", "[SSN]"))

# === Stage 4: 임베딩 (Pandas UDF — GPU에서 모델 로드) ===
@F.pandas_udf(ArrayType(FloatType()))
def embed(texts: pd.Series) -> pd.Series:
    # 워커 프로세스마다 한 번만 로드되도록 closure 외부에서 캐싱하는 것이 이상적
    # 여기서는 단순화를 위해 함수 내부에 둡니다.
    from sentence_transformers import SentenceTransformer
    import torch
    if not hasattr(embed, "_model"):
        device = "cuda" if torch.cuda.is_available() else "cpu"
        embed._model = SentenceTransformer(
            "sentence-transformers/all-MiniLM-L6-v2",
            device=device)
    embeddings = embed._model.encode(
        texts.tolist(),
        batch_size=64,
        show_progress_bar=False,
        normalize_embeddings=True)
    return pd.Series(embeddings.tolist())

result = df.withColumn("embedding", embed("text"))

# 결과 저장
(result.write
    .mode("overwrite")
    .partitionBy("language")
    .parquet("s3a://processed/web-crawl-embedded/"))

spark.stop()
```

## 11.3 SparkApplication: 다단계 노드 활용

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: llm-pipeline
  namespace: spark-apps
spec:
  type: Python
  mode: cluster
  image: 192.168.1.10:5000/llm-pipeline:1.0
  mainApplicationFile: local:///app/pipeline.py
  sparkVersion: "4.1.1"
  driver:
    cores: 2
    memory: "4g"
    serviceAccount: spark
    nodeSelector:
      kubernetes.io/arch: amd64
  executor:
    cores: 4
    instances: 2
    memory: "8g"
    memoryOverhead: "4g"          # PyTorch + Transformers는 메모리 큼
    gpu:
      name: "nvidia.com/gpu"
      quantity: 1
    nodeSelector:
      kubernetes.io/arch: arm64
      nvidia.com/gpu: "true"
    tolerations:
      - key: dedicated
        operator: Equal
        value: gpu
        effect: NoSchedule
  sparkConf:
    "spark.sql.execution.arrow.pyspark.enabled": "true"
    "spark.sql.execution.arrow.maxRecordsPerBatch": "1000"
    "spark.executor.resource.gpu.amount": "1"
    "spark.task.resource.gpu.amount": "1"
  restartPolicy:
    type: OnFailure
    onFailureRetries: 2
```

## 11.4 핵심 디자인 포인트

1. **Pandas UDF + Arrow** : Python ↔ JVM 직렬화를 0에 가깝게 만듭니다. `spark.sql.execution.arrow.pyspark.enabled=true` 필수.
2. **모델 로드 lazy 캐싱** : UDF 함수 속성에 모델을 저장해 워커당 1회만 로드.
3. **배치 사이즈 튜닝** : `maxRecordsPerBatch`를 GPU 메모리에 맞춰 조정. Blackwell 128GB 통합메모리는 매우 큰 배치를 허용.
4. **GPU 1 task per executor** : 동일 GPU에 너무 많은 task 보내면 OOM. 임베딩 추론은 보통 `task.resource.gpu.amount=1`.
5. **Output Partition** : 너무 많은 작은 파일 방지하기 위해 `coalesce` 또는 `repartition` 사용.

---

# Part 12. 트러블슈팅과 운영 노하우

## 12.1 자주 만나는 에러와 해결법

### 에러 1: `ImagePullBackOff` (이미지 못 가져옴)

```bash
kubectl describe pod <pod-name> -n spark-apps
# 보통: 멀티아치 이미지가 아니거나, registry 인증 실패
```

**해결**: `docker manifest inspect <image>` 로 ARM64 manifest 존재 확인. 사설 레지스트리는 `imagePullSecrets` 설정.

### 에러 2: `OutOfMemoryError: Java heap space`

**원인**: Executor 메모리 부족, 또는 `collect()` 남용
**해결**:
- `spark.executor.memory` 증가 (단, 노드 메모리의 70% 이내)
- `repartition`으로 파티션 수 늘려 파티션당 데이터 줄이기
- `collect()` → `take(n)` 또는 `write.parquet()`

### 에러 3: `Container killed by OOMKiller (137)`

**원인**: K8s가 컨테이너 메모리 제한 초과 시 강제 종료
**해결**: `spark.executor.memoryOverhead` 를 늘리세요 (기본 10%는 PySpark + ML 모델에는 부족합니다. 4g~8g 권장).

```yaml
sparkConf:
  "spark.executor.memoryOverhead": "4g"
```

### 에러 4: `Driver pod stuck in Pending`

```bash
kubectl describe pod <driver-pod> -n spark-apps
# Events 섹션 확인: nodeSelector / taint 미스매치, 자원 부족 등
```

**해결**: nodeSelector / tolerations 매칭 점검, 클러스터 자원 확인 (`kubectl top nodes`).

### 에러 5: ARM64에서 Snappy/LZ4 native 라이브러리 오류

ARM 환경에서 일부 압축 라이브러리가 누락된 경우입니다.

**해결**: Spark 공식 이미지(`apache/spark:4.1.1`)는 이미 ARM64 native 포함. 직접 빌드한 이미지에서 발생 시 zstd 사용 또는 native lib 추가.

```yaml
sparkConf:
  "spark.io.compression.codec": "zstd"
```

## 12.2 성능 튜닝 체크리스트

```
□ Adaptive Query Execution(AQE) 켰는가?
   spark.sql.adaptive.enabled = true
   spark.sql.adaptive.coalescePartitions.enabled = true
   spark.sql.adaptive.skewJoin.enabled = true

□ 셔플 파티션 수가 적절한가? (기본 200, 데이터 크기에 비례)
   spark.sql.shuffle.partitions = (executor cores × instances × 2~4)

□ Broadcast Join을 활용하는가? (10MB 이하 테이블)
   spark.sql.autoBroadcastJoinThreshold = 10485760

□ Parquet/ORC 사용 (CSV 금지)

□ Predicate Pushdown 효과 받게 컬럼 프루닝
   df.select(...).filter(...) ← select를 먼저

□ cache()는 정말 여러 번 쓸 때만

□ Driver collect 금지

□ Pandas UDF + Arrow 사용
   spark.sql.execution.arrow.pyspark.enabled = true
```

## 12.3 모니터링 스택

### Spark UI History Server

작업이 끝난 뒤에도 UI를 볼 수 있도록 이벤트 로그를 PVC/오브젝트 스토리지에 저장합니다.

```yaml
sparkConf:
  "spark.eventLog.enabled": "true"
  "spark.eventLog.dir": "s3a://spark-logs/"
```

History Server를 별도 Deployment로 배포하면 `http://<service>:18080`에서 모든 과거 작업을 볼 수 있습니다.

### Prometheus + Grafana

Kubeflow Spark Operator는 Prometheus 메트릭을 노출합니다.

```yaml
spec:
  monitoring:
    exposeDriverMetrics: true
    exposeExecutorMetrics: true
    prometheus:
      jmxExporterJar: "/prometheus/jmx_prometheus_javaagent-0.20.0.jar"
      port: 8090
```

ServiceMonitor 만들고 Grafana Spark 대시보드(ID 7890 등) 임포트하면 끝.

### NVIDIA DCGM Exporter (DGX Spark용)

GPU 활용률 모니터링은 필수입니다.

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace monitoring --create-namespace
```

Grafana에서 NVIDIA DCGM 대시보드(ID 12239) 사용.

## 12.4 비용/자원 효율화 팁

1. **Dynamic Allocation 사용**

   ```yaml
   sparkConf:
     "spark.dynamicAllocation.enabled": "true"
     "spark.dynamicAllocation.minExecutors": "1"
     "spark.dynamicAllocation.maxExecutors": "10"
     "spark.dynamicAllocation.shuffleTracking.enabled": "true"
   ```

2. **K8s에서 priorityClass 활용** : 중요 작업은 높은 우선순위로

3. **GPU 시간분할(Time-Slicing)** : DGX 1개 GPU를 여러 작업이 공유

4. **Volcano 또는 YuniKorn 스케줄러** : Spark 작업 그룹 스케줄링(gang scheduling)으로 일부 Executor만 뜨는 데드락 방지

   ```bash
   helm repo add yunikorn https://apache.github.io/yunikorn-release
   helm install yunikorn yunikorn/yunikorn \
     --namespace yunikorn --create-namespace \
     --version 1.8.0
   ```

   SparkApplication에 `batchScheduler: yunikorn` 추가.

---

# 부록 A. 명령어 치트시트

## A.1 클러스터 관리

```bash
# 노드 상태
kubectl get nodes -o wide
kubectl top nodes

# Pod 상태
kubectl get pods -n spark-apps -o wide
kubectl logs -f <pod-name> -n spark-apps

# 디버깅
kubectl describe pod <pod-name> -n spark-apps
kubectl exec -it <pod-name> -n spark-apps -- bash

# GPU 확인 (DGX Spark에서)
kubectl describe node dgx-spark | grep -A 3 "Allocatable"
nvidia-smi
```

## A.2 Spark 작업 관리

```bash
# 작업 목록
kubectl get sparkapp -n spark-apps
kubectl get sparkapp -A

# 작업 제출
kubectl apply -f my-spark-app.yaml

# 작업 삭제
kubectl delete sparkapp my-app -n spark-apps

# 재시도
kubectl delete sparkapp my-app -n spark-apps && kubectl apply -f my-spark-app.yaml

# Spark UI 접근
kubectl port-forward -n spark-apps <driver-pod> 4040:4040
```

## A.3 PySpark 디버깅 in Pod

```bash
# Driver Pod 안에서 spark-shell이나 pyspark 실행
kubectl exec -it spark-pi-driver -n spark-apps -- /opt/spark/bin/pyspark
```

## A.4 자주 쓰는 PySpark 코드

```python
# 파티션 수 확인
df.rdd.getNumPartitions()

# 실행 계획 보기 (튜닝 시 필수)
df.explain(extended=True)
df.explain(mode="formatted")

# 스키마 보기
df.printSchema()

# 통계
df.summary().show()
df.describe().show()

# 메모리 사용량
spark.catalog.cacheTable("my_table")
```

---

# 부록 B. 참고 자료

## B.1 공식 문서

- Apache Spark 공식: https://spark.apache.org
- Spark 4.1 문서: https://spark.apache.org/docs/4.1.1/
- Spark on Kubernetes: https://spark.apache.org/docs/latest/running-on-kubernetes.html
- Apache Spark K8s Operator (공식): https://apache.github.io/spark-kubernetes-operator
- Kubeflow Spark Operator: https://www.kubeflow.org/docs/components/spark-operator/
- NVIDIA DGX Spark User Guide: https://docs.nvidia.com/dgx/dgx-spark/
- RAPIDS Accelerator for Spark: https://nvidia.github.io/spark-rapids/

## B.2 추천 도서

- 《러닝 스파크 (Learning Spark, 2nd Edition)》 — Jules Damji 외, O'Reilly
- 《스파크 완벽 가이드 (Spark: The Definitive Guide)》 — Bill Chambers, Matei Zaharia, O'Reilly
- 《Designing Data-Intensive Applications》 — Martin Kleppmann (분산 시스템 일반론)

## B.3 학습 로드맵 (이 강의 이후)

```
1단계 (1~2주):
  □ 이 강의의 모든 실습 완수
  □ Spark UI 모든 탭 의미 이해
  □ 1GB CSV → Parquet 변환 + 집계 직접 작성

2단계 (3~4주):
  □ Delta Lake 또는 Apache Iceberg 도입 (ACID 트랜잭션)
  □ Airflow + SparkKubernetesOperator로 DAG 구성
  □ Prometheus 모니터링 운영

3단계 (1~2개월):
  □ RAPIDS로 TPC-H 벤치 비교
  □ Volcano/YuniKorn 도입
  □ GitOps (ArgoCD) 자동 배포
  □ 멀티테넌시 (네임스페이스별 Quota, NetworkPolicy)

4단계 (LLMOps 심화):
  □ Spark + Ray 통합 (분산 학습 전처리)
  □ Spark + MLflow (모델 추적)
  □ Vector DB(Milvus, Weaviate)와 Spark 파이프라인
  □ Streaming(Structured Streaming) + Kafka로 실시간 추론 파이프라인
```

## B.4 강의 핵심 요약 — 한 페이지

```
[Spark의 본질]
  분산 + 메모리 + 지연 실행 + DAG 최적화

[3대 추상화]
  RDD < DataFrame ≤ Dataset

[2대 연산 분류]
  Transformation (지연) / Action (실행 트리거)
  Narrow (싸다) / Wide=Shuffle (비싸다)

[Driver vs Executor]
  Driver = 사령탑 (계획, 조정)
  Executor = 실행자 (실제 데이터 처리)

[Spark on K8s]
  Native(spark-submit) vs Operator(CRD/선언형)
  → 운영은 Operator 권장

[이종 클러스터 핵심]
  - Multi-arch 이미지 필수
  - nodeSelector / taint+toleration
  - GPU는 DGX Spark, 일반 ETL은 ThinkStation 분담

[GPU 가속]
  RAPIDS Accelerator → 코드 수정 0, 설정 1줄
  Pandas UDF + Arrow + 적절한 batch size

[튜닝 1법칙]
  Shuffle을 줄여라.

[튜닝 2법칙]
  collect() 쓰지 마라.

[튜닝 3법칙]
  AQE를 켜라.
```

---

> **🎓 마지막 한마디**
>
> Apache Spark는 **15년간 사실상 표준의 자리를 지켜온 분산 처리 엔진**이며, LLM 시대의 데이터 파이프라인에서도 그 중요성이 더욱 커지고 있습니다. NVIDIA DGX Spark의 GPU 가속과 결합하면, 한때 데이터센터 규모가 필요했던 작업을 책상 위에서 수행할 수 있습니다.
>
> 이 문서를 따라 직접 손으로 클러스터를 구성하고, 작업을 제출하고, Spark UI에서 stage가 흘러가는 것을 관찰해 보세요. 이론은 1시간, **실습은 10시간 이상** 권장합니다. 막히는 부분은 `kubectl describe`와 `kubectl logs`가 99% 답을 줍니다.
>
> Happy Sparking on K8s! ⚡️🐳
