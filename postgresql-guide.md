# PostgreSQL 완벽 가이드: 기본부터 고급, 실전 운영까지

> **대상:** 클라우드 엔지니어, MLOps/LLMOps 실무자  
> **목표:** PostgreSQL의 핵심 개념을 체계적으로 이해하고, 실전 환경에서 안정적으로 운영할 수 있는 역량을 갖춘다.

---

## 목차

1. [PostgreSQL 소개](#1-postgresql-소개)
2. [설치 및 초기 설정](#2-설치-및-초기-설정)
3. [기본 개념](#3-기본-개념)
4. [SQL 기초](#4-sql-기초)
5. [고급 개념](#5-고급-개념)
6. [인덱스와 성능 최적화](#6-인덱스와-성능-최적화)
7. [트랜잭션과 동시성 제어](#7-트랜잭션과-동시성-제어)
8. [백업과 복구](#8-백업과-복구)
9. [복제(Replication)와 고가용성](#9-복제replication와-고가용성)
10. [실전 활용 사례](#10-실전-활용-사례)
11. [클라우드 환경에서의 PostgreSQL](#11-클라우드-환경에서의-postgresql)
12. [모니터링과 운영](#12-모니터링과-운영)
13. [보안](#13-보안)
14. [PostgreSQL과 AI/ML 파이프라인](#14-postgresql과-aiml-파이프라인)

---

## 1. PostgreSQL 소개

### 1.1 PostgreSQL이란?

PostgreSQL(포스트그레스큐엘)은 30년 이상의 역사를 가진 오픈소스 객체-관계형 데이터베이스 관리 시스템(ORDBMS)입니다. UC Berkeley의 POSTGRES 프로젝트에서 출발하여 현재는 세계에서 가장 진보된 오픈소스 관계형 데이터베이스로 자리매김했습니다.

### 1.2 핵심 특징

- **ACID 완벽 준수:** 트랜잭션의 원자성(Atomicity), 일관성(Consistency), 격리성(Isolation), 지속성(Durability)을 완벽하게 보장합니다.
- **MVCC(Multi-Version Concurrency Control):** 읽기와 쓰기가 서로를 블로킹하지 않아 높은 동시성을 제공합니다.
- **확장성:** 사용자 정의 타입, 함수, 연산자, 인덱스 메서드, 절차 언어를 추가할 수 있습니다.
- **풍부한 데이터 타입:** JSON/JSONB, Array, HStore, Range, Geometric, Network Address 등 다양한 내장 타입을 지원합니다.
- **표준 SQL 준수:** SQL:2016 표준의 대부분을 지원하며, 윈도우 함수, CTE, LATERAL JOIN 등 고급 SQL 기능을 제공합니다.

### 1.3 MySQL과의 비교

| 항목 | PostgreSQL | MySQL |
|------|-----------|-------|
| 라이선스 | PostgreSQL License (BSD계열) | GPL v2 (Oracle 소유) |
| MVCC | 네이티브 지원 | InnoDB에서만 지원 |
| JSON 지원 | JSONB (바이너리, 인덱싱 가능) | JSON (텍스트 기반) |
| 전문 검색 | 내장 지원 (tsvector/tsquery) | 제한적 내장 지원 |
| 확장성 | 매우 높음 (Extension 시스템) | 제한적 |
| 복잡한 쿼리 | 옵티마이저 우수 | 상대적으로 단순 |
| GIS 지원 | PostGIS (업계 표준) | 기본적 수준 |

---

## 2. 설치 및 초기 설정

### 2.1 운영체제별 설치

#### Ubuntu/Debian

```bash
# 공식 PostgreSQL APT 저장소 추가
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# 저장소 서명 키 가져오기
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# 패키지 목록 업데이트 및 설치
sudo apt-get update
sudo apt-get install -y postgresql-16 postgresql-contrib-16
```

#### CentOS/RHEL

```bash
# PostgreSQL 공식 저장소 설치
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# 기본 모듈 비활성화 후 설치
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib

# 데이터베이스 초기화 및 서비스 시작
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable --now postgresql-16
```

#### macOS (Homebrew)

```bash
brew install postgresql@16
brew services start postgresql@16
```

#### Docker (권장 — 개발 및 테스트 환경)

```bash
docker run -d \
  --name postgres-dev \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secure_password \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2.0'

volumes:
  pgdata:
```

### 2.2 초기 접속 및 사용자 설정

```bash
# postgres 시스템 유저로 전환하여 접속
sudo -u postgres psql

# 또는 직접 접속 (비밀번호 인증 설정 후)
psql -h localhost -U admin -d myapp
```

```sql
-- 새로운 사용자(Role) 생성
CREATE ROLE app_user WITH LOGIN PASSWORD 'strong_password_here';

-- 데이터베이스 생성
CREATE DATABASE production_db
    OWNER = app_user
    ENCODING = 'UTF8'
    LC_COLLATE = 'ko_KR.UTF-8'
    LC_CTYPE = 'ko_KR.UTF-8'
    TEMPLATE = template0;

-- 권한 부여
GRANT CONNECT ON DATABASE production_db TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;

-- 앞으로 생성될 테이블에도 자동 권한 부여
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;
```

### 2.3 핵심 설정 파일

PostgreSQL은 두 개의 핵심 설정 파일로 운영됩니다.

#### postgresql.conf — 서버 동작 설정

```ini
# ===== 접속 설정 =====
listen_addresses = '*'          # 모든 IP에서 접속 허용 (프로덕션에서는 제한 권장)
port = 5432
max_connections = 200           # 최대 동시 접속 수

# ===== 메모리 설정 =====
shared_buffers = '4GB'          # 전체 메모리의 25% 권장
effective_cache_size = '12GB'   # 전체 메모리의 50~75%
work_mem = '64MB'               # 정렬/해시 조인시 사용 (세션 단위 주의)
maintenance_work_mem = '1GB'    # VACUUM, CREATE INDEX 등에 사용

# ===== WAL 설정 =====
wal_level = replica             # 복제를 위해 replica 이상 필요
max_wal_size = '4GB'
min_wal_size = '1GB'
wal_compression = on

# ===== 쿼리 플래너 =====
random_page_cost = 1.1          # SSD 환경에 적합한 값
effective_io_concurrency = 200  # SSD: 200, HDD: 2

# ===== 로깅 =====
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_min_duration_statement = 1000   # 1초 이상 걸리는 쿼리 로깅
log_line_prefix = '%t [%p] %u@%d '
```

#### pg_hba.conf — 클라이언트 인증 설정

```
# TYPE  DATABASE    USER        ADDRESS         METHOD
local   all         postgres                    peer
local   all         all                         md5
host    all         all         127.0.0.1/32    scram-sha-256
host    all         all         10.0.0.0/8      scram-sha-256
host    replication repl_user   10.0.1.0/24     scram-sha-256
```

인증 방법 설명: `peer`는 OS 사용자와 DB 사용자가 동일할 때 비밀번호 없이 접속을 허용합니다. `scram-sha-256`은 가장 안전한 비밀번호 인증 방식으로, 프로덕션 환경에서 권장됩니다. `md5`는 이전 버전 호환용이며, 새 설치에서는 `scram-sha-256`을 사용하는 것이 좋습니다.

---

## 3. 기본 개념

### 3.1 아키텍처 개요

PostgreSQL은 **프로세스 기반 아키텍처**를 사용합니다. 클라이언트가 접속할 때마다 `postmaster` 프로세스가 새로운 `backend` 프로세스를 fork합니다.

주요 프로세스 구성:

- **Postmaster:** 메인 프로세스. 클라이언트 연결 요청을 수신하고 backend 프로세스를 생성합니다.
- **Backend Process:** 각 클라이언트 연결당 하나씩 생성됩니다. SQL 파싱, 실행, 결과 반환을 담당합니다.
- **Background Writer:** shared buffer의 dirty page를 디스크에 주기적으로 기록합니다.
- **WAL Writer:** WAL(Write-Ahead Log) 버퍼를 디스크에 기록합니다.
- **Checkpointer:** 체크포인트를 수행하여 데이터 일관성을 보장합니다.
- **Autovacuum Launcher:** 자동 VACUUM 프로세스를 관리합니다.
- **Stats Collector:** 테이블, 인덱스 사용 통계를 수집합니다.

### 3.2 메모리 구조

PostgreSQL의 메모리는 공유 메모리 영역과 프로세스별 로컬 메모리 영역으로 나뉩니다.

**공유 메모리(Shared Memory):**

- **Shared Buffers:** 테이블과 인덱스의 데이터 페이지를 캐싱합니다. `shared_buffers` 파라미터로 크기를 설정하며, 보통 전체 RAM의 25%를 권장합니다.
- **WAL Buffers:** WAL 레코드를 디스크에 쓰기 전에 임시 저장합니다. `wal_buffers` 파라미터로 설정합니다.
- **CLOG(Commit Log):** 각 트랜잭션의 커밋 상태를 기록합니다.

**프로세스 로컬 메모리:**

- **work_mem:** ORDER BY, DISTINCT, Hash Join 등에 사용됩니다. 쿼리 내 각 정렬/해시 노드마다 개별 할당되므로, 복잡한 쿼리에서는 `work_mem × 노드 수`만큼 메모리를 사용할 수 있습니다.
- **maintenance_work_mem:** VACUUM, CREATE INDEX, ALTER TABLE ADD FOREIGN KEY 등 유지보수 작업에 사용됩니다.
- **temp_buffers:** 임시 테이블 접근에 사용됩니다.

### 3.3 저장 구조

PostgreSQL의 데이터는 다음과 같은 물리적 구조로 저장됩니다.

```
$PGDATA/
├── base/                   # 데이터베이스별 디렉토리
│   ├── 1/                  # template1 데이터베이스
│   ├── 13067/              # template0 데이터베이스
│   └── 16384/              # 사용자 데이터베이스
│       ├── 16385           # 테이블 파일 (OID 기반)
│       ├── 16385_fsm       # Free Space Map
│       └── 16385_vm        # Visibility Map
├── global/                 # 클러스터 전역 테이블
├── pg_wal/                 # WAL 세그먼트 파일
├── pg_xact/                # 트랜잭션 커밋 상태
├── postgresql.conf         # 메인 설정 파일
└── pg_hba.conf             # 인증 설정 파일
```

각 테이블은 하나 이상의 **페이지(Page)**로 구성됩니다. 기본 페이지 크기는 8KB이며, 각 페이지에는 여러 개의 **튜플(Tuple, Row)**이 저장됩니다.

### 3.4 데이터 타입 총정리

```sql
-- === 숫자형 ===
smallint        -- 2바이트, -32768 ~ 32767
integer         -- 4바이트, 약 ±21억
bigint          -- 8바이트, 약 ±922경
numeric(p,s)    -- 가변 길이, 정확한 소수 계산 (금융 데이터에 필수)
real            -- 4바이트, 부동소수점 (6자리 정밀도)
double precision -- 8바이트, 부동소수점 (15자리 정밀도)
serial          -- 자동 증가 정수 (내부적으로 시퀀스 사용)
bigserial       -- 자동 증가 큰 정수

-- === 문자형 ===
varchar(n)      -- 가변 길이 문자열 (최대 n자)
char(n)         -- 고정 길이 문자열 (n자, 빈 공간은 공백으로 채움)
text            -- 무제한 가변 길이 문자열 (varchar와 성능 동일)

-- === 날짜/시간 ===
date            -- 날짜만 (2025-01-15)
time            -- 시간만 (14:30:00)
timestamp       -- 날짜 + 시간 (타임존 없음)
timestamptz     -- 날짜 + 시간 + 타임존 (프로덕션 권장)
interval        -- 시간 간격 ('1 year 2 months 3 days')

-- === 불리언 ===
boolean         -- true / false / null

-- === JSON ===
json            -- 텍스트로 저장 (입력 시마다 파싱)
jsonb           -- 바이너리로 저장 (인덱싱 가능, 권장)

-- === 배열 ===
integer[]       -- 정수 배열
text[]          -- 문자열 배열

-- === 네트워크 ===
inet            -- IPv4/IPv6 주소 + 서브넷
cidr            -- IPv4/IPv6 네트워크 주소
macaddr         -- MAC 주소

-- === UUID ===
uuid            -- 128비트 고유 식별자

-- === 범위(Range) ===
int4range       -- 정수 범위
tstzrange       -- 타임스탬프(타임존) 범위
daterange       -- 날짜 범위

-- === 기하학 ===
point           -- 2차원 좌표
line            -- 무한 직선
polygon         -- 다각형

-- === 기타 ===
bytea           -- 바이너리 데이터
tsvector        -- 전문 검색용 문서 벡터
tsquery         -- 전문 검색 쿼리
```

---

## 4. SQL 기초

### 4.1 DDL (Data Definition Language)

```sql
-- 스키마 생성 (네임스페이스 분리)
CREATE SCHEMA ml_pipeline;

-- 테이블 생성 (실전 예시: ML 모델 메타데이터 테이블)
CREATE TABLE ml_pipeline.models (
    id              bigserial PRIMARY KEY,
    model_name      varchar(255) NOT NULL,
    version         varchar(50) NOT NULL,
    framework       varchar(50) NOT NULL CHECK (framework IN ('pytorch', 'tensorflow', 'sklearn', 'onnx')),
    parameters      jsonb DEFAULT '{}',
    metrics         jsonb DEFAULT '{}',
    artifact_path   text NOT NULL,
    status          varchar(20) DEFAULT 'registered'
                    CHECK (status IN ('registered', 'staging', 'production', 'archived')),
    created_by      varchar(100) NOT NULL,
    created_at      timestamptz DEFAULT now(),
    updated_at      timestamptz DEFAULT now(),

    -- 복합 유니크 제약조건
    UNIQUE (model_name, version)
);

-- 인덱스 생성
CREATE INDEX idx_models_status ON ml_pipeline.models (status);
CREATE INDEX idx_models_metrics ON ml_pipeline.models USING GIN (metrics);
CREATE INDEX idx_models_created ON ml_pipeline.models (created_at DESC);

-- updated_at 자동 갱신 트리거
CREATE OR REPLACE FUNCTION update_modified_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON ml_pipeline.models
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_column();

-- 테이블 수정
ALTER TABLE ml_pipeline.models ADD COLUMN description text;
ALTER TABLE ml_pipeline.models ALTER COLUMN model_name SET NOT NULL;
ALTER TABLE ml_pipeline.models DROP COLUMN IF EXISTS description;
```

### 4.2 DML (Data Manipulation Language)

```sql
-- INSERT
INSERT INTO ml_pipeline.models (model_name, version, framework, parameters, metrics, artifact_path, created_by)
VALUES (
    'sentiment-classifier',
    'v2.1.0',
    'pytorch',
    '{"hidden_size": 768, "num_layers": 12, "learning_rate": 0.00003}'::jsonb,
    '{"accuracy": 0.945, "f1_score": 0.938, "latency_ms": 12.5}'::jsonb,
    's3://ml-artifacts/sentiment-classifier/v2.1.0/',
    'ml-team'
);

-- UPSERT (INSERT ... ON CONFLICT)
INSERT INTO ml_pipeline.models (model_name, version, framework, artifact_path, created_by, status)
VALUES ('sentiment-classifier', 'v2.1.0', 'pytorch', 's3://...', 'ml-team', 'staging')
ON CONFLICT (model_name, version)
DO UPDATE SET
    status = EXCLUDED.status,
    updated_at = now();

-- UPDATE with JOIN
UPDATE ml_pipeline.models m
SET status = 'archived'
FROM ml_pipeline.models newer
WHERE m.model_name = newer.model_name
  AND m.version < newer.version
  AND m.status = 'production'
  AND newer.status = 'production';

-- DELETE with RETURNING
DELETE FROM ml_pipeline.models
WHERE status = 'archived'
  AND updated_at < now() - interval '90 days'
RETURNING id, model_name, version;
```

### 4.3 SELECT 심화

```sql
-- JSONB 쿼리: accuracy가 0.9 이상인 프로덕션 모델
SELECT
    model_name,
    version,
    metrics->>'accuracy' AS accuracy,
    metrics->>'f1_score' AS f1_score,
    created_at
FROM ml_pipeline.models
WHERE status = 'production'
  AND (metrics->>'accuracy')::float > 0.9
ORDER BY (metrics->>'accuracy')::float DESC;

-- 집계 함수
SELECT
    framework,
    COUNT(*) AS model_count,
    AVG((metrics->>'accuracy')::float) AS avg_accuracy,
    MAX((metrics->>'f1_score')::float) AS best_f1
FROM ml_pipeline.models
WHERE status IN ('staging', 'production')
GROUP BY framework
HAVING COUNT(*) >= 2
ORDER BY avg_accuracy DESC;

-- 서브쿼리 vs JOIN
-- 서브쿼리 방식
SELECT * FROM ml_pipeline.models
WHERE model_name IN (
    SELECT model_name FROM ml_pipeline.models
    WHERE status = 'production'
);

-- JOIN 방식 (보통 더 효율적)
SELECT DISTINCT m.*
FROM ml_pipeline.models m
JOIN ml_pipeline.models prod
    ON m.model_name = prod.model_name
WHERE prod.status = 'production';
```

---

## 5. 고급 개념

### 5.1 Window Functions

윈도우 함수는 현재 행과 관련된 행 집합에 대해 계산을 수행하면서도 행을 축소하지 않습니다. GROUP BY와 달리 개별 행을 유지한 채 집계 결과를 함께 볼 수 있습니다.

```sql
-- 각 모델별 최신 버전 찾기 (ROW_NUMBER)
SELECT * FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY model_name
            ORDER BY created_at DESC
        ) AS rn
    FROM ml_pipeline.models
) ranked
WHERE rn = 1;

-- 모델별 accuracy 추이 (LAG/LEAD)
SELECT
    model_name,
    version,
    (metrics->>'accuracy')::float AS accuracy,
    LAG((metrics->>'accuracy')::float) OVER (
        PARTITION BY model_name ORDER BY created_at
    ) AS prev_accuracy,
    (metrics->>'accuracy')::float - LAG((metrics->>'accuracy')::float) OVER (
        PARTITION BY model_name ORDER BY created_at
    ) AS accuracy_delta
FROM ml_pipeline.models
ORDER BY model_name, created_at;

-- 누적 모델 수 (Running Total)
SELECT
    date_trunc('month', created_at) AS month,
    COUNT(*) AS monthly_models,
    SUM(COUNT(*)) OVER (ORDER BY date_trunc('month', created_at)) AS cumulative_models
FROM ml_pipeline.models
GROUP BY date_trunc('month', created_at)
ORDER BY month;

-- 프레임워크별 순위 (RANK vs DENSE_RANK)
SELECT
    model_name,
    framework,
    (metrics->>'accuracy')::float AS accuracy,
    RANK() OVER (PARTITION BY framework ORDER BY (metrics->>'accuracy')::float DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY framework ORDER BY (metrics->>'accuracy')::float DESC) AS dense_rank,
    NTILE(4) OVER (ORDER BY (metrics->>'accuracy')::float DESC) AS quartile
FROM ml_pipeline.models
WHERE metrics->>'accuracy' IS NOT NULL;
```

### 5.2 CTE (Common Table Expressions)

CTE는 복잡한 쿼리를 논리적 단계로 분해하여 가독성과 유지보수성을 높여줍니다.

```sql
-- 일반 CTE: 모델 성능 분석 보고서
WITH model_stats AS (
    SELECT
        model_name,
        COUNT(*) AS version_count,
        MAX((metrics->>'accuracy')::float) AS best_accuracy,
        MIN(created_at) AS first_created,
        MAX(created_at) AS last_updated
    FROM ml_pipeline.models
    GROUP BY model_name
),
production_models AS (
    SELECT model_name, version, metrics
    FROM ml_pipeline.models
    WHERE status = 'production'
)
SELECT
    ms.model_name,
    ms.version_count,
    ms.best_accuracy,
    pm.version AS production_version,
    pm.metrics->>'accuracy' AS production_accuracy,
    ms.last_updated - ms.first_created AS development_span
FROM model_stats ms
LEFT JOIN production_models pm ON ms.model_name = pm.model_name
ORDER BY ms.best_accuracy DESC;

-- 재귀 CTE: 카테고리 계층 구조 탐색
CREATE TABLE categories (
    id serial PRIMARY KEY,
    name text NOT NULL,
    parent_id integer REFERENCES categories(id)
);

WITH RECURSIVE category_tree AS (
    -- 앵커 멤버: 루트 카테고리
    SELECT id, name, parent_id, 0 AS depth, name::text AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- 재귀 멤버: 자식 카테고리
    SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;
```

### 5.3 JSONB 심화

PostgreSQL의 JSONB는 NoSQL 수준의 유연성과 관계형 DB의 안정성을 동시에 제공합니다.

```sql
-- JSONB 연산자 총정리
SELECT
    -- 키로 값 추출 (JSON 반환)
    metrics->'accuracy' AS accuracy_json,
    -- 키로 값 추출 (텍스트 반환)
    metrics->>'accuracy' AS accuracy_text,
    -- 중첩 경로 접근
    parameters#>'{training, epochs}' AS epochs_json,
    parameters#>>'{training, epochs}' AS epochs_text,
    -- 키 존재 여부 확인
    metrics ? 'f1_score' AS has_f1,
    -- 포함 여부 확인
    metrics @> '{"accuracy": 0.945}' AS exact_match,
    -- 경로 존재 여부
    metrics ? 'accuracy' AND metrics ? 'f1_score' AS has_both
FROM ml_pipeline.models;

-- JSONB 배열 처리
-- 학습 로그에서 에포크별 손실값 추출
SELECT
    model_name,
    idx AS epoch,
    (value->>'loss')::float AS loss,
    (value->>'val_loss')::float AS val_loss
FROM ml_pipeline.models,
    jsonb_array_elements(parameters->'training_log') WITH ORDINALITY AS t(value, idx)
WHERE model_name = 'sentiment-classifier';

-- JSONB 업데이트 (부분 업데이트)
UPDATE ml_pipeline.models
SET metrics = metrics || '{"latency_p99": 25.3}'::jsonb     -- 키 추가/업데이트
WHERE model_name = 'sentiment-classifier';

UPDATE ml_pipeline.models
SET metrics = metrics - 'deprecated_metric'                   -- 키 삭제
WHERE model_name = 'sentiment-classifier';

UPDATE ml_pipeline.models
SET parameters = jsonb_set(
    parameters,
    '{training, learning_rate}',                              -- 경로 지정
    '0.00001'::jsonb                                          -- 새 값
)
WHERE model_name = 'sentiment-classifier';
```

### 5.4 파티셔닝

대용량 테이블을 물리적으로 분할하여 쿼리 성능과 관리 효율을 높입니다.

```sql
-- 범위 파티셔닝: 로그 테이블
CREATE TABLE inference_logs (
    id              bigserial,
    model_name      varchar(255) NOT NULL,
    request_payload jsonb,
    response_payload jsonb,
    latency_ms      float,
    status_code     int,
    created_at      timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- 월별 파티션 생성
CREATE TABLE inference_logs_2025_01
    PARTITION OF inference_logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE inference_logs_2025_02
    PARTITION OF inference_logs
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- 자동 파티션 생성 (pg_partman 확장 사용 권장)
-- 또는 cron + 스크립트로 자동화
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
    next_month date := date_trunc('month', now()) + interval '1 month';
    partition_name text;
    start_date text;
    end_date text;
BEGIN
    partition_name := 'inference_logs_' || to_char(next_month, 'YYYY_MM');
    start_date := to_char(next_month, 'YYYY-MM-DD');
    end_date := to_char(next_month + interval '1 month', 'YYYY-MM-DD');

    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF inference_logs FOR VALUES FROM (%L) TO (%L)',
        partition_name, start_date, end_date
    );
END;
$$ LANGUAGE plpgsql;

-- 리스트 파티셔닝
CREATE TABLE model_registry (
    id bigserial,
    model_name varchar(255),
    environment varchar(20) NOT NULL,
    data jsonb
) PARTITION BY LIST (environment);

CREATE TABLE model_registry_dev PARTITION OF model_registry FOR VALUES IN ('dev');
CREATE TABLE model_registry_staging PARTITION OF model_registry FOR VALUES IN ('staging');
CREATE TABLE model_registry_prod PARTITION OF model_registry FOR VALUES IN ('production');

-- 해시 파티셔닝 (균등 분산)
CREATE TABLE embeddings (
    id bigserial,
    doc_id bigint NOT NULL,
    vector float8[]
) PARTITION BY HASH (doc_id);

CREATE TABLE embeddings_p0 PARTITION OF embeddings FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE embeddings_p1 PARTITION OF embeddings FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE embeddings_p2 PARTITION OF embeddings FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE embeddings_p3 PARTITION OF embeddings FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### 5.5 Stored Procedures & Functions

```sql
-- 함수: 모델 배포 검증
CREATE OR REPLACE FUNCTION ml_pipeline.validate_model_for_deployment(
    p_model_name varchar,
    p_version varchar,
    p_min_accuracy float DEFAULT 0.9
)
RETURNS TABLE (
    is_valid boolean,
    reason text,
    current_accuracy float
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_model record;
BEGIN
    SELECT * INTO v_model
    FROM ml_pipeline.models
    WHERE model_name = p_model_name AND version = p_version;

    IF NOT FOUND THEN
        RETURN QUERY SELECT false, '모델을 찾을 수 없습니다'::text, 0.0::float;
        RETURN;
    END IF;

    IF (v_model.metrics->>'accuracy')::float < p_min_accuracy THEN
        RETURN QUERY SELECT
            false,
            format('정확도 미달: %.3f < %.3f', (v_model.metrics->>'accuracy')::float, p_min_accuracy),
            (v_model.metrics->>'accuracy')::float;
        RETURN;
    END IF;

    RETURN QUERY SELECT
        true,
        '배포 가능'::text,
        (v_model.metrics->>'accuracy')::float;
END;
$$;

-- 프로시저: 모델 프로모션 (staging → production)
CREATE OR REPLACE PROCEDURE ml_pipeline.promote_model(
    p_model_name varchar,
    p_version varchar
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- 기존 production 모델을 archived로 변경
    UPDATE ml_pipeline.models
    SET status = 'archived'
    WHERE model_name = p_model_name
      AND status = 'production';

    -- 대상 모델을 production으로 변경
    UPDATE ml_pipeline.models
    SET status = 'production'
    WHERE model_name = p_model_name
      AND version = p_version;

    IF NOT FOUND THEN
        RAISE EXCEPTION '모델 %s 버전 %s를 찾을 수 없습니다', p_model_name, p_version;
    END IF;

    COMMIT;
END;
$$;

-- 사용
SELECT * FROM ml_pipeline.validate_model_for_deployment('sentiment-classifier', 'v2.1.0');
CALL ml_pipeline.promote_model('sentiment-classifier', 'v2.1.0');
```

### 5.6 LATERAL JOIN

LATERAL JOIN은 서브쿼리가 외부 쿼리의 각 행을 참조할 수 있게 해주는 강력한 기능입니다.

```sql
-- 각 모델의 최근 3개 버전만 가져오기
SELECT m.model_name, v.*
FROM (SELECT DISTINCT model_name FROM ml_pipeline.models) m,
LATERAL (
    SELECT version, status, metrics, created_at
    FROM ml_pipeline.models
    WHERE model_name = m.model_name
    ORDER BY created_at DESC
    LIMIT 3
) v;
```

---

## 6. 인덱스와 성능 최적화

### 6.1 인덱스 유형

```sql
-- B-Tree (기본, 등호/범위 비교에 최적)
CREATE INDEX idx_models_name ON ml_pipeline.models (model_name);
CREATE INDEX idx_models_created_desc ON ml_pipeline.models (created_at DESC);

-- 복합 인덱스 (컬럼 순서가 중요: 선택도가 높은 컬럼을 앞에)
CREATE INDEX idx_models_name_status ON ml_pipeline.models (model_name, status);

-- Partial Index (조건부 인덱스: 특정 조건의 행만 인덱싱)
CREATE INDEX idx_models_production ON ml_pipeline.models (model_name, version)
WHERE status = 'production';

-- GIN (Generalized Inverted Index: JSONB, 배열, 전문 검색에 최적)
CREATE INDEX idx_models_params_gin ON ml_pipeline.models USING GIN (parameters);
CREATE INDEX idx_models_metrics_gin ON ml_pipeline.models USING GIN (metrics jsonb_path_ops);

-- GiST (Generalized Search Tree: 범위, 기하학, 전문 검색에 최적)
CREATE INDEX idx_logs_tsrange ON inference_logs USING GiST (
    tstzrange(created_at, created_at + interval '1 hour')
);

-- BRIN (Block Range Index: 물리적 정렬과 상관관계가 높은 컬럼에 최적)
-- 시계열 데이터의 타임스탬프 컬럼에 특히 유용 (인덱스 크기가 B-Tree의 1/100 수준)
CREATE INDEX idx_logs_created_brin ON inference_logs USING BRIN (created_at);

-- 커버링 인덱스 (INCLUDE: 인덱스만으로 쿼리 완료 = Index-Only Scan)
CREATE INDEX idx_models_covering ON ml_pipeline.models (model_name, status)
INCLUDE (version, created_at);

-- 표현식 인덱스
CREATE INDEX idx_models_lower_name ON ml_pipeline.models (lower(model_name));
CREATE INDEX idx_models_accuracy ON ml_pipeline.models (((metrics->>'accuracy')::float));
```

### 6.2 EXPLAIN ANALYZE

쿼리 실행 계획을 분석하는 것은 성능 최적화의 출발점입니다.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT model_name, version, metrics->>'accuracy' AS accuracy
FROM ml_pipeline.models
WHERE status = 'production'
  AND (metrics->>'accuracy')::float > 0.9
ORDER BY (metrics->>'accuracy')::float DESC;
```

실행 계획 읽는 법:

- **Seq Scan:** 전체 테이블 스캔. 소규모 테이블이 아니라면 인덱스 추가를 고려합니다.
- **Index Scan:** 인덱스를 사용한 스캔. 소수의 행을 가져올 때 효율적입니다.
- **Index Only Scan:** 인덱스만으로 결과 반환 (커버링 인덱스). 가장 효율적입니다.
- **Bitmap Index Scan:** 여러 인덱스를 결합하거나, 중간 규모의 결과를 가져올 때 사용됩니다.
- **Hash Join / Merge Join / Nested Loop:** 조인 전략입니다. 데이터 크기와 분포에 따라 옵티마이저가 선택합니다.
- **actual time:** 실제 소요 시간(ms)입니다. `actual time=0.015..0.018`은 첫 행까지 0.015ms, 마지막 행까지 0.018ms를 의미합니다.
- **rows:** 실제 처리한 행 수입니다. 추정값과 차이가 크면 통계가 오래됐을 수 있습니다.
- **Buffers: shared hit/read:** hit은 캐시에서 읽은 블록, read는 디스크에서 읽은 블록입니다.

### 6.3 VACUUM과 AUTOVACUUM

PostgreSQL은 MVCC를 구현하기 위해 UPDATE/DELETE 시 이전 버전의 튜플(Dead Tuple)을 즉시 제거하지 않습니다. VACUUM은 이 Dead Tuple을 정리하여 공간을 재활용합니다.

```sql
-- 수동 VACUUM
VACUUM ml_pipeline.models;               -- 일반 VACUUM: Dead Tuple 정리
VACUUM FULL ml_pipeline.models;           -- 테이블 재작성 (잠금 발생, 주의!)
VACUUM ANALYZE ml_pipeline.models;        -- VACUUM + 통계 업데이트

-- 현재 Dead Tuple 상태 확인
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_ratio,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

AUTOVACUUM 튜닝 (postgresql.conf):

```ini
autovacuum = on
autovacuum_max_workers = 5               # 동시 autovacuum 워커 수
autovacuum_vacuum_threshold = 50          # 최소 Dead Tuple 수
autovacuum_vacuum_scale_factor = 0.1      # 테이블의 10%가 Dead Tuple이면 실행
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 2          # I/O 부하 제한 (ms)
autovacuum_vacuum_cost_limit = 1000       # 높일수록 VACUUM이 빠르게 진행
```

대용량 테이블에 대한 개별 튜닝:

```sql
-- 특정 테이블의 AUTOVACUUM을 더 공격적으로 설정
ALTER TABLE inference_logs SET (
    autovacuum_vacuum_scale_factor = 0.01,    -- 1%만 변경되어도 VACUUM
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_vacuum_cost_delay = 0           -- I/O 제한 없음
);
```

---

## 7. 트랜잭션과 동시성 제어

### 7.1 트랜잭션 격리 수준

PostgreSQL은 4가지 격리 수준을 지원합니다.

```sql
-- Read Uncommitted (PostgreSQL에서는 Read Committed와 동일하게 동작)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Read Committed (기본값)
-- 각 쿼리문이 실행 시작 시점의 스냅샷을 봅니다.
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Repeatable Read
-- 트랜잭션 시작 시점의 스냅샷을 봅니다. 같은 쿼리를 반복해도 같은 결과를 보장합니다.
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Serializable
-- 가장 높은 격리 수준. 직렬 실행과 동일한 결과를 보장합니다.
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### 7.2 잠금(Lock) 메커니즘

```sql
-- 명시적 행 잠금
SELECT * FROM ml_pipeline.models
WHERE model_name = 'sentiment-classifier' AND status = 'production'
FOR UPDATE;                    -- 배타적 잠금 (다른 트랜잭션 대기)

SELECT * FROM ml_pipeline.models
WHERE model_name = 'sentiment-classifier'
FOR UPDATE SKIP LOCKED;        -- 잠긴 행은 건너뛰기 (큐 처리 패턴)

SELECT * FROM ml_pipeline.models
WHERE model_name = 'sentiment-classifier'
FOR UPDATE NOWAIT;             -- 잠금 즉시 실패 (대기하지 않음)

-- Advisory Lock (애플리케이션 수준 잠금)
-- 분산 환경에서 리소스 접근 직렬화에 유용
SELECT pg_advisory_lock(hashtext('deploy:sentiment-classifier'));
-- ... 배포 작업 수행 ...
SELECT pg_advisory_unlock(hashtext('deploy:sentiment-classifier'));

-- 잠금 대기 상태 모니터링
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation = blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### 7.3 데드락 방지 패턴

```sql
-- 패턴 1: 항상 동일한 순서로 잠금 획득
-- 나쁜 예: 트랜잭션 A는 테이블1→테이블2, 트랜잭션 B는 테이블2→테이블1
-- 좋은 예: 모든 트랜잭션이 테이블1→테이블2 순서로 잠금

-- 패턴 2: 짧은 트랜잭션 유지
BEGIN;
    -- 필요한 작업만 수행
    UPDATE ml_pipeline.models SET status = 'production' WHERE id = 42;
COMMIT;

-- 패턴 3: NOWAIT 또는 lock_timeout 사용
SET lock_timeout = '5s';
UPDATE ml_pipeline.models SET status = 'production' WHERE id = 42;
```

---

## 8. 백업과 복구

### 8.1 논리적 백업 (pg_dump)

```bash
# 단일 데이터베이스 백업 (커스텀 포맷, 압축 포함)
pg_dump -h localhost -U admin -d production_db \
    -Fc --compress=9 \
    -f /backup/production_db_$(date +%Y%m%d_%H%M%S).dump

# 스키마만 백업
pg_dump -h localhost -U admin -d production_db \
    --schema-only -f /backup/schema_only.sql

# 특정 테이블만 백업
pg_dump -h localhost -U admin -d production_db \
    -t 'ml_pipeline.models' -Fc \
    -f /backup/models_table.dump

# 전체 클러스터 백업 (모든 데이터베이스 + 역할)
pg_dumpall -h localhost -U admin \
    -f /backup/full_cluster_$(date +%Y%m%d).sql

# 복원
pg_restore -h localhost -U admin -d production_db \
    --clean --if-exists --no-owner \
    /backup/production_db_20250115_030000.dump

# 병렬 복원 (대규모 데이터베이스)
pg_restore -h localhost -U admin -d production_db \
    --jobs=4 --clean --if-exists \
    /backup/production_db_20250115_030000.dump
```

### 8.2 물리적 백업 (pg_basebackup)

WAL(Write-Ahead Log) 기반 연속 아카이빙으로 PITR(Point-in-Time Recovery)을 지원합니다.

```bash
# WAL 아카이빙 설정 (postgresql.conf)
archive_mode = on
archive_command = 'cp %p /archive/wal/%f'
# 또는 S3로 직접 아카이빙
# archive_command = 'aws s3 cp %p s3://pg-backup/wal/%f'

# 기본 백업 수행
pg_basebackup -h localhost -U repl_user \
    -D /backup/base \
    --wal-method=stream \
    --checkpoint=fast \
    --progress \
    --verbose
```

### 8.3 PITR (Point-in-Time Recovery)

```bash
# 1. 서비스 중지
sudo systemctl stop postgresql

# 2. 현재 데이터 디렉토리 이동
mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main_old

# 3. 기본 백업에서 복원
cp -r /backup/base /var/lib/postgresql/16/main

# 4. recovery 설정
cat > /var/lib/postgresql/16/main/postgresql.auto.conf << EOF
restore_command = 'cp /archive/wal/%f %p'
recovery_target_time = '2025-01-15 14:30:00+09'
recovery_target_action = 'promote'
EOF

# 5. recovery 시그널 파일 생성
touch /var/lib/postgresql/16/main/recovery.signal

# 6. 서비스 시작 (복구 진행)
sudo systemctl start postgresql
```

### 8.4 자동 백업 스크립트

```bash
#!/bin/bash
# /opt/scripts/pg_backup.sh
set -euo pipefail

BACKUP_DIR="/backup/postgresql"
RETENTION_DAYS=30
DB_HOST="localhost"
DB_USER="backup_user"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/pg_backup.log"

log() { echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"; }

# 백업 실행
log "백업 시작: production_db"
pg_dump -h "$DB_HOST" -U "$DB_USER" -d production_db \
    -Fc --compress=9 \
    -f "${BACKUP_DIR}/production_db_${TIMESTAMP}.dump" 2>> "$LOG_FILE"

# 백업 검증 (복원 테스트)
pg_restore --list "${BACKUP_DIR}/production_db_${TIMESTAMP}.dump" > /dev/null 2>&1
if [ $? -eq 0 ]; then
    log "백업 검증 성공"
else
    log "ERROR: 백업 검증 실패!"
    exit 1
fi

# 오래된 백업 삭제
find "$BACKUP_DIR" -name "*.dump" -mtime +${RETENTION_DAYS} -delete
log "백업 완료 (${RETENTION_DAYS}일 이전 파일 삭제)"
```

```bash
# cron 등록 (매일 새벽 3시)
0 3 * * * /opt/scripts/pg_backup.sh
```

---

## 9. 복제(Replication)와 고가용성

### 9.1 스트리밍 복제 (Streaming Replication)

Primary 서버의 WAL을 Standby 서버에 실시간으로 전송하여 데이터를 복제합니다.

#### Primary 서버 설정

```ini
# postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = '1GB'
synchronous_standby_names = ''    # 비동기 복제 (빈 문자열)
# synchronous_standby_names = 'standby1'   # 동기 복제
```

```
# pg_hba.conf
host replication repl_user 10.0.1.0/24 scram-sha-256
```

```sql
-- 복제 전용 사용자 생성
CREATE ROLE repl_user WITH REPLICATION LOGIN PASSWORD 'repl_password';
```

#### Standby 서버 설정

```bash
# Primary에서 기본 백업 수행
pg_basebackup -h primary-host -U repl_user \
    -D /var/lib/postgresql/16/main \
    --wal-method=stream \
    -R    # standby.signal 파일과 primary_conninfo 자동 생성
```

```ini
# postgresql.conf (Standby)
hot_standby = on
hot_standby_feedback = on
```

### 9.2 논리적 복제 (Logical Replication)

특정 테이블만 선택적으로 복제하거나, 서로 다른 PostgreSQL 버전 간 복제가 가능합니다.

```sql
-- Publisher (원본 서버)
CREATE PUBLICATION ml_models_pub
    FOR TABLE ml_pipeline.models, inference_logs;

-- Subscriber (대상 서버)
CREATE SUBSCRIPTION ml_models_sub
    CONNECTION 'host=primary-host dbname=production_db user=repl_user password=...'
    PUBLICATION ml_models_pub;

-- 복제 상태 확인
SELECT * FROM pg_stat_replication;       -- Primary에서 실행
SELECT * FROM pg_stat_subscription;      -- Subscriber에서 실행
```

### 9.3 고가용성 아키텍처

프로덕션 환경에서의 고가용성은 보통 다음과 같은 도구 조합으로 구현합니다.

**Patroni + etcd + HAProxy** 구성이 가장 널리 사용됩니다.

```yaml
# patroni.yml
scope: pg-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.1:8008

etcd:
  hosts: 10.0.2.1:2379,10.0.2.2:2379,10.0.2.3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        shared_buffers: 4GB
        wal_level: replica
        max_wal_senders: 5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.1:5432
  data_dir: /var/lib/postgresql/16/main
  authentication:
    superuser:
      username: postgres
      password: ${POSTGRES_PASSWORD}
    replication:
      username: repl_user
      password: ${REPL_PASSWORD}
```

```
# HAProxy 설정 (읽기/쓰기 분리)
frontend pg_write
    bind *:5432
    default_backend pg_primary

frontend pg_read
    bind *:5433
    default_backend pg_replicas

backend pg_primary
    option httpchk GET /primary
    http-check expect status 200
    server node1 10.0.1.1:5432 check port 8008
    server node2 10.0.1.2:5432 check port 8008
    server node3 10.0.1.3:5432 check port 8008

backend pg_replicas
    option httpchk GET /replica
    http-check expect status 200
    balance roundrobin
    server node1 10.0.1.1:5432 check port 8008
    server node2 10.0.1.2:5432 check port 8008
    server node3 10.0.1.3:5432 check port 8008
```

---

## 10. 실전 활용 사례

### 10.1 전문 검색 (Full-Text Search)

PostgreSQL에 내장된 전문 검색 기능으로 별도의 Elasticsearch 없이도 검색 기능을 구현할 수 있습니다.

```sql
-- 문서 테이블
CREATE TABLE documents (
    id bigserial PRIMARY KEY,
    title text NOT NULL,
    content text NOT NULL,
    search_vector tsvector,
    created_at timestamptz DEFAULT now()
);

-- 검색 벡터 자동 업데이트 트리거
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.content, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trig_search_vector
    BEFORE INSERT OR UPDATE OF title, content
    ON documents
    FOR EACH ROW
    EXECUTE FUNCTION update_search_vector();

-- GIN 인덱스 생성
CREATE INDEX idx_documents_search ON documents USING GIN (search_vector);

-- 검색 쿼리 (랭킹 포함)
SELECT
    id,
    title,
    ts_rank_cd(search_vector, query) AS rank,
    ts_headline('english', content, query,
        'StartSel=<mark>, StopSel=</mark>, MaxFragments=3') AS highlighted
FROM documents,
    to_tsquery('english', 'machine & learning & model') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- 퍼지 검색 (오타 허용)
-- pg_trgm 확장 필요
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_documents_title_trgm ON documents USING GIN (title gin_trgm_ops);

SELECT title, similarity(title, 'machin lerning') AS sim
FROM documents
WHERE title % 'machin lerning'
ORDER BY sim DESC;
```

### 10.2 작업 큐 (Job Queue) 패턴

```sql
CREATE TABLE job_queue (
    id bigserial PRIMARY KEY,
    queue_name varchar(50) NOT NULL DEFAULT 'default',
    payload jsonb NOT NULL,
    status varchar(20) DEFAULT 'pending'
        CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    priority int DEFAULT 0,
    attempts int DEFAULT 0,
    max_attempts int DEFAULT 3,
    locked_by varchar(100),
    locked_at timestamptz,
    scheduled_at timestamptz DEFAULT now(),
    completed_at timestamptz,
    error_message text,
    created_at timestamptz DEFAULT now()
);

CREATE INDEX idx_queue_fetch ON job_queue (queue_name, priority DESC, scheduled_at)
WHERE status = 'pending';

-- 작업 가져오기 (동시성 안전)
WITH next_job AS (
    SELECT id
    FROM job_queue
    WHERE queue_name = 'ml-training'
      AND status = 'pending'
      AND scheduled_at <= now()
    ORDER BY priority DESC, scheduled_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE job_queue
SET
    status = 'processing',
    locked_by = 'worker-01',
    locked_at = now(),
    attempts = attempts + 1
FROM next_job
WHERE job_queue.id = next_job.id
RETURNING job_queue.*;

-- LISTEN/NOTIFY로 실시간 알림
-- Worker 측
LISTEN new_job;

-- Producer 측 (트리거로 자동화)
CREATE OR REPLACE FUNCTION notify_new_job()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('new_job', json_build_object(
        'id', NEW.id,
        'queue', NEW.queue_name
    )::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trig_notify_job
    AFTER INSERT ON job_queue
    FOR EACH ROW
    EXECUTE FUNCTION notify_new_job();
```

### 10.3 시계열 데이터 관리

```sql
-- TimescaleDB 확장 (시계열 데이터에 특화)
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE metrics (
    time        timestamptz NOT NULL,
    host        varchar(100) NOT NULL,
    metric_name varchar(100) NOT NULL,
    value       double precision,
    tags        jsonb DEFAULT '{}'
);

-- 하이퍼테이블로 변환 (자동 시간 기반 파티셔닝)
SELECT create_hypertable('metrics', 'time', chunk_time_interval => INTERVAL '1 day');

-- 연속 집계 (자동 갱신되는 집계 뷰)
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    host,
    metric_name,
    AVG(value) AS avg_val,
    MAX(value) AS max_val,
    MIN(value) AS min_val,
    COUNT(*) AS sample_count
FROM metrics
GROUP BY bucket, host, metric_name
WITH NO DATA;

-- 데이터 보존 정책
SELECT add_retention_policy('metrics', INTERVAL '90 days');
SELECT add_compression_policy('metrics', INTERVAL '7 days');
```

### 10.4 멀티테넌트 아키텍처

```sql
-- Row-Level Security (RLS)를 활용한 멀티테넌시
CREATE TABLE tenant_data (
    id bigserial PRIMARY KEY,
    tenant_id uuid NOT NULL,
    data jsonb,
    created_at timestamptz DEFAULT now()
);

-- RLS 활성화
ALTER TABLE tenant_data ENABLE ROW LEVEL SECURITY;

-- 정책 생성: 현재 세션의 tenant_id와 일치하는 행만 접근 가능
CREATE POLICY tenant_isolation ON tenant_data
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- 애플리케이션에서 사용
SET app.current_tenant = 'a1b2c3d4-e5f6-7890-abcd-ef1234567890';
SELECT * FROM tenant_data;  -- 해당 tenant의 데이터만 반환됨
```

---

## 11. 클라우드 환경에서의 PostgreSQL

### 11.1 관리형 서비스 비교

| 항목 | AWS RDS/Aurora | GCP Cloud SQL | Azure Database |
|------|---------------|---------------|----------------|
| 자동 백업 | 35일 보존 | 자동 구성 | 35일 보존 |
| 복제 | Aurora Replica (최대 15개) | Read Replica | Read Replica |
| 확장성 | Aurora Serverless v2 | 자동 스토리지 확장 | Hyperscale (Citus) |
| 가용성 | Multi-AZ (99.99%) | Regional (99.95%) | Zone-redundant |
| 특이사항 | Aurora는 자체 스토리지 레이어 | AlloyDB (AI 최적화) | Citus 통합 |

### 11.2 Kubernetes에서의 PostgreSQL

```yaml
# CloudNativePG Operator 사용 예시
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster
  namespace: database
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:16

  postgresql:
    parameters:
      shared_buffers: "2GB"
      effective_cache_size: "6GB"
      work_mem: "64MB"
      max_connections: "200"
      log_min_duration_statement: "1000"

  storage:
    size: 100Gi
    storageClass: gp3-encrypted

  resources:
    requests:
      memory: "8Gi"
      cpu: "4"
    limits:
      memory: "8Gi"
      cpu: "4"

  backup:
    barmanObjectStore:
      destinationPath: "s3://pg-backups/cluster/"
      s3Credentials:
        accessKeyId:
          name: aws-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-creds
          key: ACCESS_SECRET_KEY
    retentionPolicy: "30d"

  monitoring:
    enablePodMonitor: true

  affinity:
    podAntiAffinityType: required    # 각 인스턴스를 다른 노드에 배치
```

### 11.3 Connection Pooling (PgBouncer)

PostgreSQL은 프로세스 기반이므로, 연결마다 약 5-10MB의 메모리를 소비합니다. 수백 개의 마이크로서비스가 접속하는 환경에서는 Connection Pooler가 필수입니다.

```ini
# pgbouncer.ini
[databases]
production_db = host=localhost port=5432 dbname=production_db

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# 풀링 모드
pool_mode = transaction    # 트랜잭션 단위 풀링 (권장)
# pool_mode = session      # 세션 단위 풀링 (PREPARE 사용 시)
# pool_mode = statement    # 쿼리 단위 풀링 (가장 공격적)

# 풀 크기 설정
default_pool_size = 25
max_client_conn = 1000
max_db_connections = 100

# 타임아웃
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 30
```

---

## 12. 모니터링과 운영

### 12.1 핵심 모니터링 쿼리

```sql
-- 1. 현재 활성 쿼리 및 세션 상태
SELECT
    pid,
    usename,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS query_duration,
    left(query, 100) AS query_preview
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid != pg_backend_pid()
ORDER BY query_start;

-- 2. 느린 쿼리 강제 종료
SELECT pg_cancel_backend(pid);     -- 쿼리만 취소
SELECT pg_terminate_backend(pid);  -- 세션 종료

-- 3. 테이블 크기 확인
SELECT
    schemaname || '.' || tablename AS table_name,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
    pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- 4. 인덱스 사용률 (사용되지 않는 인덱스 찾기)
SELECT
    schemaname || '.' || relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog')
ORDER BY pg_relation_size(indexrelid) DESC;

-- 5. 캐시 히트율 (99% 이상이 이상적)
SELECT
    sum(heap_blks_hit) AS cache_hit,
    sum(heap_blks_read) AS disk_read,
    round(
        sum(heap_blks_hit)::numeric /
        NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2
    ) AS cache_hit_ratio
FROM pg_statio_user_tables;

-- 6. 복제 지연 확인 (Primary에서 실행)
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

-- 7. pg_stat_statements (쿼리 성능 통계, 확장 설치 필요)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT
    left(query, 80) AS query,
    calls,
    round(total_exec_time::numeric, 2) AS total_time_ms,
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_total,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### 12.2 Prometheus + Grafana 모니터링

```yaml
# postgres_exporter 설정 (docker-compose)
services:
  postgres-exporter:
    image: quay.io/prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor:password@postgres:5432/production_db?sslmode=disable"
    ports:
      - "9187:9187"

# Prometheus scrape 설정
scrape_configs:
  - job_name: 'postgresql'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

핵심 알림 규칙:

```yaml
# prometheus_rules.yml
groups:
  - name: postgresql_alerts
    rules:
      - alert: PostgreSQLHighConnections
        expr: pg_stat_activity_count > 180
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL 연결 수가 높습니다 ({{ $value }}/200)"

      - alert: PostgreSQLReplicationLag
        expr: pg_replication_lag > 30
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "복제 지연이 30초를 초과했습니다"

      - alert: PostgreSQLDeadTupleRatio
        expr: pg_stat_user_tables_n_dead_tup / (pg_stat_user_tables_n_live_tup + 1) > 0.1
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Dead Tuple 비율이 10%를 초과했습니다"
```

---

## 13. 보안

### 13.1 인증 및 권한 관리

```sql
-- 역할(Role) 기반 접근 제어
-- 읽기 전용 역할
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE production_db TO readonly;
GRANT USAGE ON SCHEMA ml_pipeline TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA ml_pipeline TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA ml_pipeline GRANT SELECT ON TABLES TO readonly;

-- 읽기/쓰기 역할
CREATE ROLE readwrite;
GRANT readonly TO readwrite;  -- readonly 권한 상속
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA ml_pipeline TO readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA ml_pipeline
    GRANT INSERT, UPDATE, DELETE ON TABLES TO readwrite;

-- 사용자에게 역할 부여
CREATE ROLE data_scientist WITH LOGIN PASSWORD 'strong_pass';
GRANT readonly TO data_scientist;

CREATE ROLE ml_engineer WITH LOGIN PASSWORD 'strong_pass';
GRANT readwrite TO ml_engineer;

-- 컬럼 수준 권한 제한
REVOKE ALL ON ml_pipeline.models FROM data_scientist;
GRANT SELECT (model_name, version, framework, metrics, status) ON ml_pipeline.models TO data_scientist;
-- artifact_path, parameters 등 민감 정보는 접근 불가
```

### 13.2 SSL/TLS 설정

```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'
ssl_min_protocol_version = 'TLSv1.3'
```

```
# pg_hba.conf (SSL 필수)
hostssl all all 0.0.0.0/0 scram-sha-256
```

### 13.3 데이터 암호화

```sql
-- pgcrypto 확장으로 컬럼 수준 암호화
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- 데이터 암호화
INSERT INTO secrets (name, encrypted_value)
VALUES ('api_key', pgp_sym_encrypt('sk-abc123...', 'encryption-key'));

-- 데이터 복호화
SELECT name, pgp_sym_decrypt(encrypted_value, 'encryption-key') AS value
FROM secrets
WHERE name = 'api_key';
```

### 13.4 감사 로깅

```ini
# postgresql.conf (pgAudit 확장 사용)
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl, role'
pgaudit.log_catalog = off
pgaudit.log_level = 'log'
```

---

## 14. PostgreSQL과 AI/ML 파이프라인

### 14.1 pgvector — 벡터 유사도 검색

LLM 애플리케이션에서 임베딩 저장 및 유사도 검색에 핵심적으로 사용됩니다.

```sql
-- pgvector 확장 설치
CREATE EXTENSION IF NOT EXISTS vector;

-- 임베딩 테이블 생성
CREATE TABLE document_embeddings (
    id bigserial PRIMARY KEY,
    content text NOT NULL,
    metadata jsonb DEFAULT '{}',
    embedding vector(1536),       -- OpenAI text-embedding-3-small 차원
    created_at timestamptz DEFAULT now()
);

-- HNSW 인덱스 (권장: 빠른 근사 최근접 이웃 검색)
CREATE INDEX idx_embeddings_hnsw ON document_embeddings
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- IVFFlat 인덱스 (메모리 효율적)
CREATE INDEX idx_embeddings_ivfflat ON document_embeddings
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- 유사도 검색 (RAG 파이프라인의 Retrieval 단계)
SELECT
    id,
    content,
    metadata,
    1 - (embedding <=> $1::vector) AS similarity   -- 코사인 유사도
FROM document_embeddings
WHERE metadata->>'category' = 'technical'
ORDER BY embedding <=> $1::vector
LIMIT 10;

-- 하이브리드 검색 (키워드 + 시맨틱)
WITH keyword_results AS (
    SELECT id, ts_rank_cd(search_vector, query) AS keyword_score
    FROM document_embeddings, to_tsquery('english', 'machine & learning') query
    WHERE search_vector @@ query
),
semantic_results AS (
    SELECT id, 1 - (embedding <=> $1::vector) AS semantic_score
    FROM document_embeddings
    ORDER BY embedding <=> $1::vector
    LIMIT 100
)
SELECT
    d.id,
    d.content,
    COALESCE(k.keyword_score, 0) * 0.3 + COALESCE(s.semantic_score, 0) * 0.7 AS combined_score
FROM document_embeddings d
LEFT JOIN keyword_results k ON d.id = k.id
LEFT JOIN semantic_results s ON d.id = s.id
WHERE k.id IS NOT NULL OR s.id IS NOT NULL
ORDER BY combined_score DESC
LIMIT 10;
```

### 14.2 Feature Store 패턴

```sql
-- 피처 저장소 테이블
CREATE TABLE feature_store (
    entity_id varchar(255) NOT NULL,
    feature_group varchar(100) NOT NULL,
    feature_name varchar(100) NOT NULL,
    feature_value jsonb NOT NULL,
    event_time timestamptz NOT NULL,
    created_at timestamptz DEFAULT now(),

    PRIMARY KEY (entity_id, feature_group, feature_name, event_time)
) PARTITION BY RANGE (event_time);

-- 최신 피처 조회 뷰
CREATE MATERIALIZED VIEW latest_features AS
SELECT DISTINCT ON (entity_id, feature_group, feature_name)
    entity_id,
    feature_group,
    feature_name,
    feature_value,
    event_time
FROM feature_store
ORDER BY entity_id, feature_group, feature_name, event_time DESC;

-- 피처 벡터 조합 (모델 서빙 시)
SELECT
    entity_id,
    jsonb_object_agg(feature_name, feature_value) AS feature_vector
FROM latest_features
WHERE entity_id = 'user_12345'
  AND feature_group = 'user_behavior'
GROUP BY entity_id;
```

### 14.3 실험 추적 (Experiment Tracking)

```sql
CREATE TABLE experiments (
    id bigserial PRIMARY KEY,
    experiment_name varchar(255) NOT NULL,
    run_id uuid DEFAULT gen_random_uuid(),
    hyperparameters jsonb NOT NULL,
    metrics jsonb DEFAULT '{}',
    artifacts jsonb DEFAULT '{}',       -- S3 경로 등
    tags text[] DEFAULT '{}',
    git_commit varchar(40),
    status varchar(20) DEFAULT 'running',
    started_at timestamptz DEFAULT now(),
    finished_at timestamptz,

    UNIQUE (experiment_name, run_id)
);

CREATE INDEX idx_experiments_tags ON experiments USING GIN (tags);
CREATE INDEX idx_experiments_metrics ON experiments USING GIN (metrics jsonb_path_ops);

-- 최고 성능 실험 찾기
SELECT
    experiment_name,
    run_id,
    hyperparameters,
    metrics,
    finished_at - started_at AS duration
FROM experiments
WHERE experiment_name = 'sentiment-v2'
  AND status = 'completed'
  AND (metrics->>'accuracy')::float > 0.9
ORDER BY (metrics->>'accuracy')::float DESC
LIMIT 5;
```

---

## 부록: 운영 체크리스트

### A. 프로덕션 배포 전 체크리스트

- [ ] `shared_buffers`를 RAM의 25%로 설정했는가?
- [ ] `effective_cache_size`를 RAM의 50~75%로 설정했는가?
- [ ] `work_mem`을 워크로드에 맞게 조정했는가?
- [ ] `max_connections`를 적절히 설정하고 Connection Pooler를 도입했는가?
- [ ] `wal_level = replica` 이상으로 설정했는가?
- [ ] SSL/TLS를 활성화했는가?
- [ ] `scram-sha-256` 인증을 사용하는가?
- [ ] AUTOVACUUM이 활성화되어 있는가?
- [ ] `log_min_duration_statement`로 느린 쿼리를 기록하는가?
- [ ] `pg_stat_statements`가 활성화되어 있는가?
- [ ] 자동 백업이 설정되어 있는가?
- [ ] 백업 복원 테스트를 수행했는가?
- [ ] 모니터링과 알림이 구성되어 있는가?
- [ ] Standby 서버가 준비되어 있는가?
- [ ] Failover 절차가 문서화되어 있는가?

### B. 정기 운영 작업

- **일간:** 백업 상태 확인, 복제 지연 확인, 느린 쿼리 리뷰
- **주간:** 인덱스 사용률 분석, Dead Tuple 비율 확인, 디스크 사용량 점검
- **월간:** pg_stat_statements 리뷰 및 초기화, 불필요한 인덱스 정리, 파티션 생성/삭제
- **분기:** 백업 복원 테스트, Failover 훈련, PostgreSQL 마이너 버전 업그레이드 검토

### C. 유용한 psql 명령어

```
\l                  -- 데이터베이스 목록
\dt                 -- 테이블 목록
\dt+                -- 테이블 목록 (크기 포함)
\di                 -- 인덱스 목록
\d tablename        -- 테이블 구조 상세 보기
\du                 -- 역할(사용자) 목록
\dn                 -- 스키마 목록
\df                 -- 함수 목록
\x                  -- 확장 출력 토글
\timing             -- 쿼리 실행 시간 표시
\e                  -- 외부 편집기로 쿼리 작성
\watch 5            -- 5초마다 마지막 쿼리 반복 실행
\copy               -- CSV 가져오기/내보내기
```

---

> **마지막 한마디:** PostgreSQL은 단순한 데이터베이스를 넘어, 벡터 검색(pgvector), 시계열(TimescaleDB), 지리정보(PostGIS), 그래프(Apache AGE) 등 다양한 확장으로 무한히 확장 가능한 데이터 플랫폼입니다. 특히 LLMOps/MLOps 파이프라인에서 모델 레지스트리, 피처 스토어, 벡터 DB, 실험 추적을 하나의 PostgreSQL로 통합할 수 있다는 점은 운영 복잡도를 크게 낮춰줍니다. "Just use Postgres"라는 커뮤니티의 격언에는 충분한 이유가 있습니다.
