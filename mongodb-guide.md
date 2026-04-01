# MongoDB 완벽 가이드: 기본 개념부터 실전 운영까지

> **대상**: 백엔드/클라우드/MLOps 엔지니어  
> **목표**: MongoDB의 철학을 이해하고, 설계 → 개발 → 운영 → 최적화까지 전 과정을 익힌다.

---

## 목차

1. [MongoDB란 무엇인가](#1-mongodb란-무엇인가)
2. [핵심 개념: Document Model](#2-핵심-개념-document-model)
3. [설치 및 환경 구성](#3-설치-및-환경-구성)
4. [CRUD 완전 정복](#4-crud-완전-정복)
5. [스키마 설계 전략](#5-스키마-설계-전략)
6. [인덱싱 심화](#6-인덱싱-심화)
7. [Aggregation Framework](#7-aggregation-framework)
8. [트랜잭션과 데이터 일관성](#8-트랜잭션과-데이터-일관성)
9. [Replica Set — 고가용성](#9-replica-set--고가용성)
10. [Sharding — 수평 확장](#10-sharding--수평-확장)
11. [보안 설정](#11-보안-설정)
12. [백업과 복구](#12-백업과-복구)
13. [모니터링과 성능 튜닝](#13-모니터링과-성능-튜닝)
14. [실전 활용 사례](#14-실전-활용-사례)
15. [클라우드 환경 운영 (Atlas & K8s)](#15-클라우드-환경-운영-atlas--k8s)
16. [MLOps/LLMOps에서의 MongoDB 활용](#16-mlopsllmops에서의-mongodb-활용)

---

## 1. MongoDB란 무엇인가

### 1.1 정의

MongoDB는 **문서 지향(Document-Oriented) NoSQL 데이터베이스**이다. 2009년 MongoDB Inc.(구 10gen)에서 처음 출시되었으며, JSON과 유사한 **BSON(Binary JSON)** 형식으로 데이터를 저장한다.

### 1.2 RDBMS와의 근본적 차이

| 관점 | RDBMS | MongoDB |
|------|-------|---------|
| 데이터 모델 | 테이블 + 행 + 열 (정규화) | 컬렉션 + 도큐먼트 (비정규화) |
| 스키마 | 고정 스키마 (DDL 필수) | 유연한 스키마 (Schema-less) |
| 조인 | SQL JOIN이 기본 | `$lookup`으로 제한적 지원, 임베딩 선호 |
| 확장 방식 | 수직 확장(Scale-Up) 중심 | 수평 확장(Sharding) 네이티브 지원 |
| 트랜잭션 | ACID 완전 지원 | 4.0부터 Multi-Document ACID 지원 |
| 쿼리 언어 | SQL | MQL (MongoDB Query Language) |

### 1.3 왜 MongoDB를 선택하는가

- **빠른 개발 속도**: 스키마 마이그레이션 없이 필드 추가/삭제 가능
- **자연스러운 데이터 매핑**: 애플리케이션 객체 ↔ 도큐먼트가 1:1 매핑
- **수평 확장**: 샤딩으로 페타바이트급 데이터 처리
- **풍부한 쿼리**: 인덱스, 집계, 전문 검색, 지리공간 쿼리, 벡터 검색

### 1.4 MongoDB가 적합하지 않은 경우

- 복잡한 JOIN이 빈번한 ERP/금융 원장 시스템
- 엄격한 스키마 무결성이 법적으로 요구되는 도메인
- 단일 노드에서 충분한 소규모 정형 데이터

---

## 2. 핵심 개념: Document Model

### 2.1 용어 매핑

```
RDBMS          →  MongoDB
─────────────────────────────
Database       →  Database
Table          →  Collection
Row            →  Document
Column         →  Field
Primary Key    →  _id (자동 생성)
Index          →  Index
JOIN           →  $lookup / Embedding
```

### 2.2 BSON 도큐먼트

```json
{
  "_id": ObjectId("664a1f2e3b4c5d6e7f8a9b0c"),
  "username": "mlops_engineer",
  "email": "eng@example.com",
  "profile": {
    "department": "AI Platform",
    "skills": ["Python", "Kubernetes", "Terraform"],
    "level": "Senior"
  },
  "projects": [
    {
      "name": "LLM Serving Pipeline",
      "status": "active",
      "created_at": ISODate("2025-03-15T09:00:00Z")
    }
  ],
  "created_at": ISODate("2025-01-10T00:00:00Z")
}
```

**핵심 포인트:**
- `_id`는 모든 도큐먼트에 자동 부여되는 12바이트 고유 식별자(ObjectId)
- 도큐먼트 내에 **배열**과 **중첩 객체**를 자유롭게 포함 가능
- 하나의 컬렉션 안에 서로 다른 구조의 도큐먼트가 공존 가능

### 2.3 ObjectId 구조

```
|-- 4 bytes --|-- 5 bytes --|-- 3 bytes --|
   Timestamp    Random Value   Counter
```

- 타임스탬프가 포함되어 있어 **생성 시각을 역추출** 가능: `ObjectId.getTimestamp()`
- 별도의 `created_at` 필드 없이도 삽입 순서를 알 수 있다

### 2.4 BSON 주요 데이터 타입

| 타입 | 설명 | 예시 |
|------|------|------|
| String | UTF-8 문자열 | `"hello"` |
| Int32 / Int64 | 정수 | `NumberInt(42)` |
| Double | 부동소수점 | `3.14` |
| Decimal128 | 고정밀 소수 (금융) | `NumberDecimal("19.99")` |
| Boolean | 참/거짓 | `true` |
| Date | 밀리초 정밀도 날짜 | `ISODate(...)` |
| ObjectId | 12바이트 고유 ID | `ObjectId(...)` |
| Array | 배열 | `[1, 2, 3]` |
| Object | 중첩 도큐먼트 | `{ key: value }` |
| Binary | 바이너리 데이터 | 파일, 이미지 등 |
| Null | null 값 | `null` |

---

## 3. 설치 및 환경 구성

### 3.1 로컬 설치 (Ubuntu/Debian)

```bash
# 1. GPG 키 등록
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# 2. 저장소 추가
echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# 3. 설치
sudo apt-get update
sudo apt-get install -y mongodb-org

# 4. 서비스 시작
sudo systemctl start mongod
sudo systemctl enable mongod

# 5. 상태 확인
sudo systemctl status mongod
```

### 3.2 macOS 설치 (Homebrew)

```bash
brew tap mongodb/brew
brew install mongodb-community@7.0
brew services start mongodb-community@7.0
```

### 3.3 Docker로 실행 (권장 — 개발 환경)

```bash
# 단일 인스턴스
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  -v mongodb_data:/data/db \
  mongo:7.0

# 접속 확인
docker exec -it mongodb mongosh \
  -u admin -p secret --authenticationDatabase admin
```

### 3.4 Docker Compose — Replica Set 개발 환경

```yaml
# docker-compose.yml
version: "3.8"
services:
  mongo1:
    image: mongo:7.0
    command: mongod --replSet rs0 --bind_ip_all
    ports:
      - "27017:27017"
    volumes:
      - mongo1_data:/data/db

  mongo2:
    image: mongo:7.0
    command: mongod --replSet rs0 --bind_ip_all
    ports:
      - "27018:27017"
    volumes:
      - mongo2_data:/data/db

  mongo3:
    image: mongo:7.0
    command: mongod --replSet rs0 --bind_ip_all
    ports:
      - "27019:27017"
    volumes:
      - mongo3_data:/data/db

volumes:
  mongo1_data:
  mongo2_data:
  mongo3_data:
```

```bash
docker compose up -d

# Replica Set 초기화
docker exec -it $(docker ps -qf "name=mongo1") mongosh --eval '
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})'
```

### 3.5 mongosh 기본 명령어

```javascript
// 데이터베이스 목록
show dbs

// 데이터베이스 선택/생성
use myapp

// 컬렉션 목록
show collections

// 서버 상태
db.serverStatus()

// 현재 연결 정보
db.runCommand({ connectionStatus: 1 })
```

### 3.6 주요 설정 파일 (mongod.conf)

```yaml
# /etc/mongod.conf
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2          # 전체 RAM의 50% 또는 256MB 중 큰 값

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 0.0.0.0            # 프로덕션에서는 특정 IP로 제한

security:
  authorization: enabled       # 인증 활성화

replication:
  replSetName: rs0             # Replica Set 이름
```

---

## 4. CRUD 완전 정복

### 4.1 Create (삽입)

```javascript
// 단일 삽입
db.users.insertOne({
  name: "김엔지니어",
  role: "MLOps Engineer",
  skills: ["Python", "K8s", "MongoDB"],
  created_at: new Date()
})

// 다중 삽입 (ordered: false → 하나 실패해도 나머지 계속 삽입)
db.users.insertMany([
  { name: "이개발", role: "Backend", skills: ["Go", "PostgreSQL"] },
  { name: "박데이터", role: "Data Engineer", skills: ["Spark", "Airflow"] }
], { ordered: false })
```

### 4.2 Read (조회)

```javascript
// 기본 조회
db.users.find({ role: "MLOps Engineer" })

// 프로젝션 (필요한 필드만)
db.users.find(
  { role: "MLOps Engineer" },
  { name: 1, skills: 1, _id: 0 }
)

// 비교 연산자
db.users.find({ age: { $gte: 25, $lte: 35 } })

// 배열 쿼리
db.users.find({ skills: { $all: ["Python", "K8s"] } })        // AND
db.users.find({ skills: { $in: ["Python", "Go"] } })          // OR
db.users.find({ skills: { $size: 3 } })                       // 배열 길이

// 중첩 도큐먼트 쿼리 (Dot Notation)
db.users.find({ "profile.department": "AI Platform" })

// 정규식
db.users.find({ name: { $regex: /^김/, $options: "i" } })

// 정렬 + 페이징
db.users.find()
  .sort({ created_at: -1 })
  .skip(20)
  .limit(10)

// 존재 여부
db.users.find({ email: { $exists: true } })

// 논리 연산자
db.users.find({
  $or: [
    { role: "MLOps Engineer" },
    { skills: "Terraform" }
  ]
})
```

### 4.3 Update (수정)

```javascript
// 단일 수정
db.users.updateOne(
  { name: "김엔지니어" },
  {
    $set: { role: "Senior MLOps Engineer" },
    $push: { skills: "Ray" },
    $currentDate: { updated_at: true }
  }
)

// 다중 수정
db.users.updateMany(
  { role: "Backend" },
  { $set: { department: "Platform" } }
)

// 배열 조작 연산자
db.users.updateOne(
  { name: "김엔지니어" },
  {
    $addToSet: { skills: "Triton" },           // 중복 방지 추가
    $pull: { skills: "Legacy Tool" },           // 특정 요소 제거
  }
)

// Upsert (없으면 삽입, 있으면 수정)
db.configs.updateOne(
  { key: "model_version" },
  { $set: { value: "v2.1.0", updated_at: new Date() } },
  { upsert: true }
)

// replaceOne (도큐먼트 전체 교체)
db.users.replaceOne(
  { name: "이개발" },
  { name: "이개발", role: "Senior Backend", skills: ["Go", "Rust"], replaced: true }
)

// findOneAndUpdate (수정 후 결과 반환)
db.counters.findOneAndUpdate(
  { _id: "request_seq" },
  { $inc: { value: 1 } },
  { returnDocument: "after", upsert: true }
)
```

### 4.4 Delete (삭제)

```javascript
// 단일 삭제
db.users.deleteOne({ name: "퇴사자" })

// 다중 삭제
db.logs.deleteMany({ created_at: { $lt: ISODate("2024-01-01") } })

// 컬렉션 전체 삭제 (주의!)
db.temp_collection.drop()
```

### 4.5 Bulk Write (배치 연산)

```javascript
db.products.bulkWrite([
  {
    insertOne: {
      document: { name: "GPU A100", price: 10000 }
    }
  },
  {
    updateOne: {
      filter: { name: "GPU V100" },
      update: { $set: { price: 5000 } }
    }
  },
  {
    deleteOne: {
      filter: { name: "GPU K80" }
    }
  }
], { ordered: false })
```

---

## 5. 스키마 설계 전략

### 5.1 핵심 원칙 — "함께 조회되는 데이터는 함께 저장한다"

MongoDB 스키마 설계의 황금률이다. RDBMS처럼 정규화하면 `$lookup`(JOIN)이 빈번해지고 성능이 급락한다.

### 5.2 임베딩(Embedding) vs 참조(Referencing)

#### 임베딩이 적합한 경우

```javascript
// 블로그 포스트 + 댓글 (1:Few)
{
  _id: ObjectId("..."),
  title: "MongoDB 스키마 설계",
  author: "김엔지니어",
  comments: [
    { user: "이개발", text: "좋은 글!", date: ISODate("...") },
    { user: "박데이터", text: "감사합니다", date: ISODate("...") }
  ]
}
```

**적합한 상황:**
- 1:1 또는 1:Few 관계
- 자식 데이터가 부모 없이 독립적으로 조회되지 않는 경우
- 도큐먼트 크기가 16MB를 넘지 않는 경우
- 자식 데이터의 변경 빈도가 낮은 경우

#### 참조가 적합한 경우

```javascript
// 주문 → 상품 (Many:Many)
// orders 컬렉션
{
  _id: ObjectId("order_001"),
  user_id: ObjectId("user_123"),
  product_ids: [
    ObjectId("prod_a"),
    ObjectId("prod_b")
  ],
  total: 25000
}

// products 컬렉션
{
  _id: ObjectId("prod_a"),
  name: "GPU 인스턴스 1시간",
  price: 15000
}
```

**적합한 상황:**
- 1:Many에서 "Many"가 수천 이상
- 자식 데이터가 독립적으로 조회되어야 하는 경우
- 여러 부모가 같은 자식을 공유하는 경우 (M:N)
- 도큐먼트 크기 제한(16MB)에 근접하는 경우

### 5.3 설계 패턴 (Building with Patterns)

#### Bucket Pattern — 시계열/로그 데이터

```javascript
// 나쁜 예: 측정값마다 1 도큐먼트 → 도큐먼트 폭발
{ sensor_id: "s1", temp: 22.5, ts: ISODate("2025-03-15T10:00:00Z") }
{ sensor_id: "s1", temp: 22.7, ts: ISODate("2025-03-15T10:00:01Z") }

// 좋은 예: 1시간 단위 Bucket
{
  sensor_id: "s1",
  bucket_start: ISODate("2025-03-15T10:00:00Z"),
  bucket_end: ISODate("2025-03-15T10:59:59Z"),
  count: 3600,
  measurements: [
    { temp: 22.5, ts: ISODate("2025-03-15T10:00:00Z") },
    { temp: 22.7, ts: ISODate("2025-03-15T10:00:01Z") }
    // ...
  ],
  summary: {
    avg_temp: 22.6,
    min_temp: 22.0,
    max_temp: 23.1
  }
}
```

#### Polymorphic Pattern — 다형성 데이터

```javascript
// 다양한 모델 메타데이터를 하나의 컬렉션에서 관리
{ type: "llm", name: "GPT-4o", params: "1.8T", context_window: 128000 }
{ type: "embedding", name: "text-embedding-3", dimensions: 3072 }
{ type: "image", name: "DALL-E 3", resolution: "1024x1024" }
```

#### Computed Pattern — 사전 계산

```javascript
// 조회 시마다 집계하지 않고 Write 시점에 누적
db.products.updateOne(
  { _id: productId },
  {
    $inc: { review_count: 1, rating_sum: 4.5 },
    $set: { avg_rating: 4.3 }  // 앱 로직에서 계산
  }
)
```

#### Outlier Pattern — 이상치 분리

```javascript
// 인기 게시물의 좋아요가 배열 한도를 넘을 때
{
  _id: "post_viral",
  title: "...",
  likes: [ /* 처음 1000개 */ ],
  has_overflow: true
}

// overflow 컬렉션에 나머지 저장
{
  post_id: "post_viral",
  page: 2,
  likes: [ /* 1001~2000 */ ]
}
```

### 5.4 Schema Validation

```javascript
db.createCollection("models", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "version", "framework"],
      properties: {
        name: {
          bsonType: "string",
          description: "모델 이름 — 필수"
        },
        version: {
          bsonType: "string",
          pattern: "^v[0-9]+\\.[0-9]+\\.[0-9]+$",
          description: "시맨틱 버전 형식 (v1.0.0)"
        },
        framework: {
          enum: ["pytorch", "tensorflow", "jax", "onnx"],
          description: "지원 프레임워크"
        },
        metrics: {
          bsonType: "object",
          properties: {
            accuracy: { bsonType: "double", minimum: 0, maximum: 1 },
            latency_ms: { bsonType: "int", minimum: 0 }
          }
        }
      }
    }
  },
  validationLevel: "strict",      // strict | moderate
  validationAction: "error"        // error | warn
})
```

---

## 6. 인덱싱 심화

### 6.1 인덱스의 중요성

인덱스가 없으면 MongoDB는 **Collection Scan**(전체 탐색)을 수행한다. 데이터가 100만 건이면 100만 건을 모두 읽는다. 인덱스는 B-Tree 기반으로 O(log n) 탐색을 가능하게 한다.

### 6.2 인덱스 종류

```javascript
// 1. 단일 필드 인덱스
db.users.createIndex({ email: 1 })                    // 오름차순
db.users.createIndex({ created_at: -1 })               // 내림차순

// 2. 복합 인덱스 (Compound Index) — 순서가 중요!
db.logs.createIndex({ service: 1, timestamp: -1 })

// 3. 유니크 인덱스
db.users.createIndex({ email: 1 }, { unique: true })

// 4. TTL 인덱스 — 자동 만료 삭제
db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 86400 }    // 24시간 후 자동 삭제
)

// 5. 텍스트 인덱스
db.articles.createIndex({ title: "text", body: "text" })
db.articles.find({ $text: { $search: "mongodb 성능 튜닝" } })

// 6. 해시 인덱스 (샤딩용)
db.events.createIndex({ user_id: "hashed" })

// 7. 지리공간 인덱스
db.stores.createIndex({ location: "2dsphere" })
db.stores.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [126.978, 37.566] },
      $maxDistance: 5000    // 5km 이내
    }
  }
})

// 8. Wildcard 인덱스 — 동적 필드 대응
db.events.createIndex({ "metadata.$**": 1 })

// 9. 부분 인덱스 (Partial Index)
db.orders.createIndex(
  { status: 1, created_at: -1 },
  { partialFilterExpression: { status: "active" } }
)
```

### 6.3 복합 인덱스 설계 법칙 — ESR Rule

**E**quality → **S**ort → **R**ange 순서로 인덱스 필드를 배치한다.

```javascript
// 쿼리: status가 "active"이고, created_at 범위 조건, score 내림차순 정렬
db.tasks.find({
  status: "active",                    // Equality
  created_at: { $gte: ISODate("2025-01-01") }  // Range
}).sort({ score: -1 })                 // Sort

// 최적 인덱스: E → S → R
db.tasks.createIndex({ status: 1, score: -1, created_at: 1 })
```

### 6.4 Covered Query

인덱스만으로 결과를 반환할 수 있으면 디스크 I/O가 제로가 된다.

```javascript
// 인덱스: { email: 1, name: 1 }
// 쿼리에서 _id를 제외하고 인덱스 필드만 프로젝션
db.users.find(
  { email: "user@example.com" },
  { email: 1, name: 1, _id: 0 }
)
// explain의 totalDocsExamined: 0 → Covered Query 달성
```

### 6.5 explain()으로 쿼리 분석

```javascript
db.users.find({ role: "MLOps Engineer" }).explain("executionStats")
```

주요 확인 지표:

| 지표 | 의미 | 이상적 값 |
|------|------|-----------|
| `stage` | COLLSCAN이면 위험 | IXSCAN |
| `totalKeysExamined` | 인덱스 스캔 수 | ≈ nReturned |
| `totalDocsExamined` | 도큐먼트 스캔 수 | ≈ nReturned |
| `executionTimeMillis` | 실행 시간 | 최소화 |

### 6.6 인덱스 관리

```javascript
// 인덱스 목록 확인
db.users.getIndexes()

// 인덱스 삭제
db.users.dropIndex("email_1")

// 인덱스 사용 통계
db.users.aggregate([{ $indexStats: {} }])

// 사용되지 않는 인덱스 식별 → 삭제 대상
```

---

## 7. Aggregation Framework

Aggregation은 MongoDB의 **데이터 처리 파이프라인**이다. UNIX 파이프(`|`)처럼 스테이지를 연결한다.

### 7.1 기본 구조

```
입력 컬렉션 → [$match] → [$group] → [$sort] → [$project] → 결과
```

### 7.2 주요 스테이지

```javascript
// 종합 예제: 서비스별 월간 에러 통계
db.logs.aggregate([
  // 1단계: 필터링
  {
    $match: {
      level: "ERROR",
      timestamp: {
        $gte: ISODate("2025-03-01"),
        $lt: ISODate("2025-04-01")
      }
    }
  },

  // 2단계: 필드 추출/변환
  {
    $project: {
      service: 1,
      day: { $dateToString: { format: "%Y-%m-%d", date: "$timestamp" } },
      message: 1
    }
  },

  // 3단계: 그룹핑
  {
    $group: {
      _id: { service: "$service", day: "$day" },
      error_count: { $sum: 1 },
      sample_messages: { $push: "$message" }
    }
  },

  // 4단계: 배열 슬라이스 (샘플 3개만)
  {
    $project: {
      error_count: 1,
      sample_messages: { $slice: ["$sample_messages", 3] }
    }
  },

  // 5단계: 정렬
  { $sort: { error_count: -1 } },

  // 6단계: 상위 20개
  { $limit: 20 }
])
```

### 7.3 고급 스테이지

```javascript
// $lookup — LEFT OUTER JOIN
db.orders.aggregate([
  {
    $lookup: {
      from: "products",
      localField: "product_id",
      foreignField: "_id",
      as: "product_info"
    }
  },
  { $unwind: "$product_info" }
])

// $facet — 하나의 파이프라인에서 다중 집계
db.products.aggregate([
  {
    $facet: {
      "by_category": [
        { $group: { _id: "$category", count: { $sum: 1 } } }
      ],
      "price_stats": [
        {
          $group: {
            _id: null,
            avg: { $avg: "$price" },
            min: { $min: "$price" },
            max: { $max: "$price" }
          }
        }
      ],
      "top5_expensive": [
        { $sort: { price: -1 } },
        { $limit: 5 }
      ]
    }
  }
])

// $graphLookup — 재귀적 그래프 탐색
db.employees.aggregate([
  {
    $graphLookup: {
      from: "employees",
      startWith: "$manager_id",
      connectFromField: "manager_id",
      connectToField: "_id",
      as: "management_chain",
      maxDepth: 5,
      depthField: "level"
    }
  }
])

// $unionWith — 컬렉션 합치기 (UNION ALL)
db.logs_2025_03.aggregate([
  { $unionWith: "logs_2025_04" },
  { $match: { level: "ERROR" } },
  { $count: "total_errors" }
])

// $merge — 집계 결과를 다른 컬렉션에 저장
db.raw_metrics.aggregate([
  { $group: { _id: "$model_id", avg_latency: { $avg: "$latency_ms" } } },
  {
    $merge: {
      into: "model_performance_summary",
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

### 7.4 Window Functions (v5.0+)

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { date: 1 },
      output: {
        running_total: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }
        },
        moving_avg_7d: {
          $avg: "$amount",
          window: { range: [-7, 0], unit: "day" }
        }
      }
    }
  }
])
```

---

## 8. 트랜잭션과 데이터 일관성

### 8.1 Write Concern

```javascript
// w: 1 — Primary 확인만 (기본값)
db.orders.insertOne({ ... }, { writeConcern: { w: 1 } })

// w: "majority" — 과반 노드 기록 확인 (권장)
db.orders.insertOne({ ... }, { writeConcern: { w: "majority" } })

// w: "majority" + j: true — 과반 노드의 저널 기록까지 확인 (가장 안전)
db.orders.insertOne(
  { ... },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
```

### 8.2 Read Concern

| Level | 설명 |
|-------|------|
| `local` | 로컬 데이터 반환 (롤백 가능성 있음) |
| `available` | Sharded에서 가장 빠름 (고아 도큐먼트 가능) |
| `majority` | 과반 커밋된 데이터만 반환 |
| `linearizable` | 가장 최신 majority 데이터 보장 |
| `snapshot` | 트랜잭션 내 일관된 스냅샷 |

### 8.3 Read Preference

```javascript
// 읽기 대상 노드 설정
db.getMongo().setReadPref("secondaryPreferred")
```

| Mode | 용도 |
|------|------|
| `primary` | 최신 데이터 필수 (기본값) |
| `primaryPreferred` | Primary 우선, 장애 시 Secondary |
| `secondary` | 리포트/분석 쿼리 분산 |
| `secondaryPreferred` | Secondary 우선 |
| `nearest` | 레이턴시 최소 노드 |

### 8.4 Multi-Document 트랜잭션 (v4.0+)

```javascript
const session = db.getMongo().startSession()
session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
})

try {
  const accounts = session.getDatabase("bank").accounts

  // 계좌 이체
  accounts.updateOne(
    { _id: "account_A" },
    { $inc: { balance: -50000 } },
    { session }
  )

  accounts.updateOne(
    { _id: "account_B" },
    { $inc: { balance: 50000 } },
    { session }
  )

  session.commitTransaction()
  print("트랜잭션 성공")
} catch (error) {
  session.abortTransaction()
  print("트랜잭션 롤백: " + error.message)
} finally {
  session.endSession()
}
```

**주의사항:**
- 트랜잭션은 60초 제한 (기본값)
- 가능하면 단일 도큐먼트 원자적 연산으로 설계하는 것이 성능상 유리
- 트랜잭션 남용은 MongoDB의 장점을 상쇄한다

---

## 9. Replica Set — 고가용성

### 9.1 아키텍처

```
┌─────────────────────────────────────────────────┐
│                   Replica Set                    │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ PRIMARY  │  │SECONDARY │  │SECONDARY │      │
│  │          │  │          │  │          │      │
│  │ Read +   │──│ Repl.    │──│ Repl.    │      │
│  │ Write    │  │ (읽기분산)│  │ (읽기분산)│      │
│  └──────────┘  └──────────┘  └──────────┘      │
│       │              │              │            │
│       └──── Heartbeat (2초 간격) ────┘            │
│                                                  │
│  Primary 장애 시 → 자동 선출 (Election)           │
│  복구 시간: 보통 10~12초                          │
└─────────────────────────────────────────────────┘
```

### 9.2 Replica Set 구성

```javascript
// mongosh에서 실행
rs.initiate({
  _id: "rs-production",
  members: [
    { _id: 0, host: "mongo-0.example.com:27017", priority: 2 },   // Primary 우선
    { _id: 1, host: "mongo-1.example.com:27017", priority: 1 },
    { _id: 2, host: "mongo-2.example.com:27017", priority: 1 }
  ]
})

// 상태 확인
rs.status()

// 설정 확인
rs.conf()
```

### 9.3 특수 멤버 역할

```javascript
// Arbiter — 투표만, 데이터 미저장 (비용 절감)
rs.addArb("arbiter.example.com:27017")

// Hidden + Priority 0 — 백업/분석 전용
rs.add({
  host: "analytics.example.com:27017",
  priority: 0,
  hidden: true
})

// Delayed — 지연 복제 (재해 복구)
rs.add({
  host: "delayed.example.com:27017",
  priority: 0,
  hidden: true,
  secondaryDelaySecs: 3600    // 1시간 지연
})
```

### 9.4 Oplog

Oplog(Operation Log)는 Primary의 모든 쓰기 연산을 기록하는 Capped Collection이다. Secondary는 이를 폴링하여 복제한다.

```javascript
// Oplog 크기 확인
use local
db.oplog.rs.stats().maxSize

// Oplog 내용 조회
db.oplog.rs.find().sort({ $natural: -1 }).limit(5)
```

---

## 10. Sharding — 수평 확장

### 10.1 아키텍처

```
┌────────────────────────────────────────────────────────────┐
│                    Sharded Cluster                          │
│                                                            │
│  ┌──────────┐                                              │
│  │  mongos   │ ← 라우터 (여러 대 가능)                      │
│  │ (Router)  │                                              │
│  └─────┬─────┘                                              │
│        │                                                    │
│  ┌─────┴────────────────────────────┐                      │
│  │        Config Servers             │                      │
│  │   (메타데이터/청크 매핑 관리)       │                      │
│  │   3-노드 Replica Set 필수         │                      │
│  └─────┬────────────────────────────┘                      │
│        │                                                    │
│  ┌─────┼──────────┬──────────┬──────────┐                  │
│  │ Shard 1       │ Shard 2  │ Shard 3   │                  │
│  │ (Replica Set) │ (RS)     │ (RS)      │                  │
│  │ chunk A,B     │ chunk C,D│ chunk E,F │                  │
│  └───────────────┴──────────┴───────────┘                  │
└────────────────────────────────────────────────────────────┘
```

### 10.2 Shard Key 선택 전략

Shard Key는 한번 설정하면 변경이 매우 어렵다. **가장 중요한 설계 결정**이다.

| Shard Key 유형 | 장점 | 단점 |
|----------------|------|------|
| 해시 (Hashed) | 균등 분배 | 범위 쿼리 비효율 |
| 범위 (Ranged) | 범위 쿼리 효율적 | 핫스팟 위험 |
| 복합 (Compound) | 균형 잡힌 접근 | 설계 복잡도 |

**좋은 Shard Key 조건:**
- 높은 카디널리티 (고유값이 많을수록 좋음)
- 균등한 분포 (특정 값에 쏠리지 않음)
- 쿼리 패턴에 포함됨 (Targeted Query 가능)

```javascript
// 샤딩 활성화
sh.enableSharding("myapp")

// 해시 기반 샤딩
sh.shardCollection("myapp.events", { user_id: "hashed" })

// 복합 키 범위 기반 샤딩
sh.shardCollection("myapp.logs", { service: 1, timestamp: 1 })
```

### 10.3 Chunk 관리

```javascript
// 청크 상태 확인
sh.status()

// 수동 청크 분할
sh.splitAt("myapp.events", { user_id: NumberLong("5000000") })

// Balancer 상태
sh.getBalancerState()
sh.isBalancerRunning()

// Balancer 윈도우 설정 (오프피크에만 이동)
db.settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "02:00", stop: "06:00" } } },
  { upsert: true }
)
```

### 10.4 Targeted vs Broadcast Query

```javascript
// Targeted Query — shard key 포함 → 특정 샤드만 조회 (빠름)
db.events.find({ user_id: "user_123" })

// Broadcast Query — shard key 미포함 → 모든 샤드 조회 (느림)
db.events.find({ event_type: "login" })
```

---

## 11. 보안 설정

### 11.1 인증 활성화

```javascript
// admin 사용자 생성
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),     // 인터랙티브 입력
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    "clusterAdmin"
  ]
})

// 애플리케이션 전용 사용자
use myapp
db.createUser({
  user: "app_service",
  pwd: "strong_password_here",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})

// 읽기 전용 분석 사용자
db.createUser({
  user: "analyst",
  pwd: "analyst_password",
  roles: [
    { role: "read", db: "myapp" }
  ]
})
```

### 11.2 주요 내장 역할

| 역할 | 권한 |
|------|------|
| `read` | 읽기 전용 |
| `readWrite` | 읽기 + 쓰기 |
| `dbAdmin` | 인덱스, 통계, 컬렉션 관리 |
| `userAdmin` | 사용자/역할 관리 |
| `clusterAdmin` | 클러스터 레벨 관리 |
| `root` | 모든 권한 (최소한으로 사용) |

### 11.3 TLS/SSL 암호화

```yaml
# mongod.conf
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongodb/server.pem
    CAFile: /etc/mongodb/ca.pem
```

```bash
# TLS로 접속
mongosh --tls \
  --tlsCertificateKeyFile /etc/mongodb/client.pem \
  --tlsCAFile /etc/mongodb/ca.pem \
  --host mongo.example.com
```

### 11.4 네트워크 보안

```yaml
# mongod.conf — IP 제한
net:
  bindIp: 127.0.0.1,10.0.1.0/24
  port: 27017
```

### 11.5 필드 수준 암호화 (CSFLE)

```javascript
// 클라이언트 사이드 필드 레벨 암호화
// 드라이버에서 설정 (Python 예시)
const encryptedFieldsMap = {
  "myapp.users": {
    fields: [
      {
        path: "ssn",
        bsonType: "string",
        keyId: UUID("..."),
        queries: { queryType: "equality" }
      }
    ]
  }
}
```

### 11.6 감사 로깅 (Enterprise)

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ atype: { $in: ["authenticate", "createUser", "dropDatabase"] } }'
```

---

## 12. 백업과 복구

### 12.1 mongodump / mongorestore

```bash
# 전체 백업
mongodump \
  --uri="mongodb://admin:secret@localhost:27017" \
  --authenticationDatabase=admin \
  --gzip \
  --out=/backup/$(date +%Y%m%d)

# 특정 DB만 백업
mongodump --db=myapp --gzip --archive=/backup/myapp.gz

# 복원
mongorestore \
  --uri="mongodb://admin:secret@localhost:27017" \
  --gzip \
  /backup/20250315/

# 특정 컬렉션 복원
mongorestore \
  --db=myapp \
  --collection=users \
  --gzip \
  /backup/20250315/myapp/users.bson.gz
```

### 12.2 mongoexport / mongoimport (JSON/CSV)

```bash
# JSON 내보내기
mongoexport --db=myapp --collection=users \
  --query='{"role":"MLOps Engineer"}' \
  --out=mlops_users.json

# CSV 내보내기
mongoexport --db=myapp --collection=metrics \
  --type=csv --fields=model_id,accuracy,timestamp \
  --out=metrics.csv

# JSON 가져오기
mongoimport --db=myapp --collection=configs \
  --file=configs.json --jsonArray
```

### 12.3 파일시스템 스냅샷

```bash
# WiredTiger + Journaling 환경 (LVM Snapshot 예시)

# 1. fsync lock (쓰기 일시 중지)
mongosh --eval 'db.fsyncLock()'

# 2. LVM 스냅샷 생성
lvcreate --size 10G --snapshot --name mongo_snap /dev/vg0/mongo_data

# 3. 잠금 해제
mongosh --eval 'db.fsyncUnlock()'

# 4. 스냅샷 마운트 & 복사
mount /dev/vg0/mongo_snap /mnt/snapshot
cp -a /mnt/snapshot/* /backup/snapshot/
```

### 12.4 Ops Manager / Atlas 자동 백업

Atlas(클라우드 매니지드)에서는 자동 연속 백업(Continuous Backup)과 Point-in-Time Recovery를 제공한다. 별도 스크립트 없이 UI에서 설정 가능하다.

---

## 13. 모니터링과 성능 튜닝

### 13.1 기본 모니터링 명령어

```javascript
// 서버 상태 (전반적 헬스)
db.serverStatus()

// 현재 실행 중인 오퍼레이션
db.currentOp({ "active": true, "secs_running": { $gt: 5 } })

// 느린 쿼리 킬
db.killOp(<opid>)

// 데이터베이스 통계
db.stats()

// 컬렉션 통계
db.users.stats()

// 커넥션 풀 상태
db.serverStatus().connections
```

### 13.2 프로파일러

```javascript
// 프로파일링 활성화 (느린 쿼리 기록)
db.setProfilingLevel(1, { slowms: 100 })    // 100ms 이상 기록

// 프로파일링 결과 조회
db.system.profile.find().sort({ ts: -1 }).limit(10)

// 가장 느린 쿼리 Top 5
db.system.profile.find().sort({ millis: -1 }).limit(5)

// 프로파일링 끄기
db.setProfilingLevel(0)
```

### 13.3 mongostat & mongotop

```bash
# 실시간 서버 통계 (1초 간격)
mongostat --rowcount=20

# 컬렉션별 읽기/쓰기 시간
mongotop 10    # 10초 간격
```

### 13.4 주요 성능 지표

| 지표 | 확인 방법 | 주의 기준 |
|------|-----------|-----------|
| 페이지 폴트 | `serverStatus().extra_info.page_faults` | 지속적 증가 |
| 커넥션 수 | `serverStatus().connections.current` | max의 80% 이상 |
| 큐 대기 | `serverStatus().globalLock.currentQueue` | total > 10 |
| Oplog 윈도우 | `rs.printReplicationInfo()` | < 24시간이면 위험 |
| 캐시 히트율 | WiredTiger cache stats | < 95%이면 메모리 부족 |
| Replication Lag | `rs.printSecondaryReplicationInfo()` | > 10초 |

### 13.5 Prometheus + Grafana 연동

```yaml
# docker-compose.yml (모니터링 스택)
services:
  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    environment:
      MONGODB_URI: "mongodb://monitor:password@mongo1:27017"
    ports:
      - "9216:9216"

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']
```

### 13.6 성능 튜닝 체크리스트

1. **인덱스 최적화**: `explain()`으로 COLLSCAN 제거, 미사용 인덱스 삭제
2. **Working Set ≤ RAM**: 자주 접근하는 데이터가 메모리에 있도록 캐시 크기 조정
3. **Connection Pooling**: 드라이버에서 적절한 풀 크기 설정 (기본 100)
4. **배치 처리**: 개별 연산 대신 `bulkWrite()` 활용
5. **프로젝션 사용**: 필요한 필드만 가져와 네트워크 대역폭 절약
6. **적절한 Write Concern**: 데이터 중요도에 따라 조절
7. **읽기 분산**: `readPreference: secondaryPreferred`로 Secondary 활용

---

## 14. 실전 활용 사례

### 14.1 실시간 로그 수집 파이프라인

```javascript
// Time Series Collection (v5.0+)
db.createCollection("service_logs", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"
  },
  expireAfterSeconds: 2592000     // 30일 자동 삭제
})

// 로그 삽입
db.service_logs.insertOne({
  timestamp: new Date(),
  metadata: { service: "inference-api", pod: "pod-abc123", region: "ap-northeast-2" },
  level: "INFO",
  latency_ms: 45,
  model: "gpt-4o",
  tokens: { input: 150, output: 320 }
})

// 서비스별 평균 레이턴시 (최근 1시간)
db.service_logs.aggregate([
  { $match: { timestamp: { $gte: new Date(Date.now() - 3600000) } } },
  {
    $group: {
      _id: "$metadata.service",
      avg_latency: { $avg: "$latency_ms" },
      p99_latency: { $percentile: { input: "$latency_ms", p: [0.99], method: "approximate" } },
      total_requests: { $sum: 1 }
    }
  }
])
```

### 14.2 실시간 알림/이벤트 시스템 — Change Streams

```javascript
// 특정 컬렉션의 변경 사항을 실시간 감시
const pipeline = [
  { $match: { "fullDocument.status": "critical" } }
]

const changeStream = db.alerts.watch(pipeline, {
  fullDocument: "updateLookup"
})

changeStream.on("change", (event) => {
  console.log("Alert triggered:", event.operationType)
  console.log("Document:", JSON.stringify(event.fullDocument))
  // Slack/PagerDuty 알림 전송 로직
})
```

### 14.3 상품 카탈로그 (이커머스)

```javascript
// Polymorphic 패턴으로 다양한 상품 타입 관리
{
  _id: ObjectId("..."),
  name: "NVIDIA A100 80GB",
  type: "hardware",
  category: ["GPU", "Data Center"],
  price: { amount: NumberDecimal("15000.00"), currency: "USD" },
  specs: {
    memory: "80GB HBM2e",
    tdp: 300,
    tensor_cores: 432
  },
  inventory: { warehouse_kr: 50, warehouse_us: 120 },
  tags: ["ai", "training", "inference"],
  reviews_summary: { count: 234, avg_rating: 4.8 }
}

// Atlas Search를 이용한 전문 검색
db.products.aggregate([
  {
    $search: {
      index: "product_search",
      compound: {
        must: [{ text: { query: "GPU training", path: ["name", "tags"] } }],
        filter: [{ range: { path: "price.amount", lte: 20000 } }]
      }
    }
  },
  { $limit: 10 },
  {
    $project: {
      name: 1, price: 1, score: { $meta: "searchScore" }
    }
  }
])
```

### 14.4 IoT 센서 데이터

```javascript
// Bucket + Time Series 조합
db.createCollection("sensor_readings", {
  timeseries: {
    timeField: "ts",
    metaField: "device",
    granularity: "minutes"
  }
})

// 기기별 이상 탐지 쿼리
db.sensor_readings.aggregate([
  { $match: { "device.type": "temperature", ts: { $gte: new Date(Date.now() - 86400000) } } },
  {
    $setWindowFields: {
      partitionBy: "$device.id",
      sortBy: { ts: 1 },
      output: {
        avg_temp: { $avg: "$value", window: { range: [-1, 0], unit: "hour" } },
        std_temp: { $stdDevPop: "$value", window: { range: [-1, 0], unit: "hour" } }
      }
    }
  },
  {
    $match: {
      $expr: {
        $gt: [{ $abs: { $subtract: ["$value", "$avg_temp"] } }, { $multiply: ["$std_temp", 3] }]
      }
    }
  }
])
```

### 14.5 세션/캐시 저장소

```javascript
// TTL 인덱스를 활용한 자동 만료 세션
db.createCollection("sessions")
db.sessions.createIndex({ "last_access": 1 }, { expireAfterSeconds: 1800 })

db.sessions.updateOne(
  { _id: sessionId },
  {
    $set: {
      user_id: "user_123",
      data: { cart: [...], preferences: {...} },
      last_access: new Date()
    }
  },
  { upsert: true }
)
```

---

## 15. 클라우드 환경 운영 (Atlas & K8s)

### 15.1 MongoDB Atlas

Atlas는 MongoDB의 완전 관리형 클라우드 서비스이다. AWS, GCP, Azure에서 운영 가능하다.

**핵심 기능:**
- 자동 스케일링 (Cluster Tier Auto-Scaling)
- 연속 백업 + Point-in-Time Recovery
- Atlas Search (Lucene 기반 전문 검색)
- Atlas Vector Search (벡터 유사도 검색)
- Data Federation (S3, Atlas 데이터를 SQL로 쿼리)
- Charts (내장 시각화 도구)

**연결 문자열:**

```python
# Python 예시
from pymongo import MongoClient

client = MongoClient(
    "mongodb+srv://user:password@cluster0.xxxxx.mongodb.net/"
    "?retryWrites=true&w=majority&appName=myApp"
)
db = client["myapp"]
```

### 15.2 Kubernetes에서 MongoDB 운영

#### MongoDB Community Operator

```yaml
# operator 설치 (Helm)
# helm repo add mongodb https://mongodb.github.io/helm-charts
# helm install community-operator mongodb/community-operator

# MongoDBCommunity CR 정의
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: mongodb-rs
  namespace: mongodb
spec:
  members: 3
  type: ReplicaSet
  version: "7.0.0"
  security:
    authentication:
      modes: ["SCRAM"]
  users:
    - name: app-user
      db: admin
      passwordSecretRef:
        name: mongodb-password
      roles:
        - name: readWrite
          db: myapp
      scramCredentialsSecretName: app-user-scram
  statefulSet:
    spec:
      template:
        spec:
          containers:
            - name: mongod
              resources:
                requests:
                  cpu: "1"
                  memory: "4Gi"
                limits:
                  cpu: "2"
                  memory: "8Gi"
      volumeClaimTemplates:
        - metadata:
            name: data-volume
          spec:
            accessModes: ["ReadWriteOnce"]
            storageClassName: gp3-encrypted
            resources:
              requests:
                storage: 100Gi
```

#### PVC 및 StorageClass 최적화

```yaml
# AWS gp3 StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "6000"
  throughput: "250"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### 15.3 Terraform으로 Atlas 프로비저닝

```hcl
# main.tf
terraform {
  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.15"
    }
  }
}

provider "mongodbatlas" {
  public_key  = var.atlas_public_key
  private_key = var.atlas_private_key
}

resource "mongodbatlas_project" "ml_platform" {
  name   = "ml-platform"
  org_id = var.atlas_org_id
}

resource "mongodbatlas_advanced_cluster" "production" {
  project_id   = mongodbatlas_project.ml_platform.id
  name         = "prod-cluster"
  cluster_type = "REPLICASET"

  replication_specs {
    region_configs {
      electable_specs {
        instance_size = "M30"
        node_count    = 3
      }
      analytics_specs {
        instance_size = "M30"
        node_count    = 1
      }
      provider_name = "AWS"
      region_name   = "AP_NORTHEAST_2"     # 서울
      priority      = 7
    }
  }

  advanced_configuration {
    javascript_enabled           = false
    oplog_size_mb               = 2048
    default_read_concern        = "majority"
    default_write_concern       = "majority"
  }
}

resource "mongodbatlas_database_user" "app_user" {
  project_id         = mongodbatlas_project.ml_platform.id
  auth_database_name = "admin"
  username           = "app_service"
  password           = var.db_password

  roles {
    role_name     = "readWrite"
    database_name = "ml_platform"
  }

  scopes {
    name = mongodbatlas_advanced_cluster.production.name
    type = "CLUSTER"
  }
}
```

---

## 16. MLOps/LLMOps에서의 MongoDB 활용

### 16.1 ML 모델 메타데이터 저장소 (Model Registry)

```javascript
// 모델 레지스트리 컬렉션
db.model_registry.insertOne({
  name: "fraud-detection-v3",
  version: "v3.2.1",
  framework: "pytorch",
  artifact_uri: "s3://models/fraud-detection/v3.2.1/model.pt",
  created_by: "ml-pipeline",
  created_at: new Date(),
  status: "staging",           // staging → production → archived
  metrics: {
    accuracy: 0.967,
    precision: 0.945,
    recall: 0.932,
    f1: 0.938,
    auc_roc: 0.991
  },
  hyperparameters: {
    learning_rate: 0.001,
    batch_size: 256,
    epochs: 50,
    optimizer: "AdamW"
  },
  training: {
    dataset: "transactions_2024_q4",
    dataset_version: "v2.1",
    duration_seconds: 3600,
    gpu: "A100-80GB",
    framework_version: "2.2.0"
  },
  tags: ["fraud", "production-ready", "approved-by-risk-team"]
})

// 프로덕션 모델 자동 승격
db.model_registry.updateOne(
  { name: "fraud-detection-v3", version: "v3.2.1" },
  {
    $set: {
      status: "production",
      promoted_at: new Date(),
      promoted_by: "ci-cd-pipeline"
    }
  }
)

// 모델 성능 비교 쿼리
db.model_registry.aggregate([
  { $match: { name: "fraud-detection-v3" } },
  { $sort: { "metrics.f1": -1 } },
  {
    $project: {
      version: 1,
      status: 1,
      "metrics.f1": 1,
      "metrics.auc_roc": 1,
      created_at: 1
    }
  }
])
```

### 16.2 실험 추적 (Experiment Tracking)

```javascript
// MLflow 스타일의 실험 추적을 MongoDB로 구현
db.experiments.insertOne({
  experiment_id: "exp_20250315_001",
  run_id: UUID(),
  name: "llm-fine-tuning-koalpaca",
  status: "completed",
  start_time: ISODate("2025-03-15T09:00:00Z"),
  end_time: ISODate("2025-03-15T15:30:00Z"),
  params: {
    base_model: "meta-llama/Llama-3-8B",
    method: "LoRA",
    lora_r: 16,
    lora_alpha: 32,
    learning_rate: 2e-4,
    dataset: "koalpaca-v1.1",
    max_seq_length: 2048,
    num_epochs: 3,
    per_device_batch_size: 4,
    gradient_accumulation_steps: 8
  },
  metrics: {
    train_loss: [2.45, 1.82, 1.34, 0.98],
    eval_loss: [2.31, 1.75, 1.41, 1.12],
    bleu_score: 0.342,
    rouge_l: 0.456
  },
  artifacts: {
    model: "s3://experiments/exp_001/model/",
    logs: "s3://experiments/exp_001/logs/",
    config: "s3://experiments/exp_001/config.yaml"
  },
  infrastructure: {
    gpu_type: "A100-80GB",
    gpu_count: 4,
    cloud: "AWS",
    instance: "p4d.24xlarge",
    cost_usd: 45.6
  },
  tags: ["llm", "korean", "lora", "production-candidate"]
})
```

### 16.3 Vector Search — RAG 파이프라인

```javascript
// Atlas Vector Search 인덱스 생성 (Atlas UI 또는 API)
/*
{
  "mappings": {
    "dynamic": true,
    "fields": {
      "embedding": {
        "dimensions": 1536,
        "similarity": "cosine",
        "type": "knnVector"
      }
    }
  }
}
*/

// 문서 저장 (임베딩 포함)
db.knowledge_base.insertOne({
  content: "MongoDB Atlas Vector Search는 벡터 유사도 검색을 지원합니다...",
  source: "mongodb-docs",
  chunk_id: 42,
  metadata: {
    page: 15,
    section: "Vector Search",
    last_updated: new Date()
  },
  embedding: [0.023, -0.041, 0.067, ...]    // 1536차원 벡터
})

// Vector Search 쿼리 (RAG의 Retrieval 단계)
db.knowledge_base.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: [0.019, -0.038, 0.071, ...],   // 쿼리 임베딩
      numCandidates: 200,
      limit: 5,
      filter: { "metadata.section": "Vector Search" }
    }
  },
  {
    $project: {
      content: 1,
      source: 1,
      score: { $meta: "vectorSearchScore" }
    }
  }
])
```

### 16.4 피처 스토어 (Feature Store)

```javascript
// 온라인 피처 서빙용 컬렉션
db.createCollection("feature_store")
db.feature_store.createIndex({ entity_id: 1, feature_group: 1 }, { unique: true })

// 피처 저장
db.feature_store.updateOne(
  { entity_id: "user_12345", feature_group: "user_behavior" },
  {
    $set: {
      features: {
        avg_session_duration: 342.5,
        login_frequency_7d: 12,
        purchase_count_30d: 3,
        last_active_category: "electronics",
        churn_risk_score: 0.23
      },
      updated_at: new Date()
    }
  },
  { upsert: true }
)

// 실시간 피처 조회 (모델 서빙 시)
db.feature_store.findOne(
  { entity_id: "user_12345", feature_group: "user_behavior" },
  { features: 1, _id: 0 }
)
```

### 16.5 LLM 프롬프트/응답 로깅

```javascript
// LLM 호출 로그 (비용 추적, 품질 모니터링, 디버깅)
db.createCollection("llm_logs", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"
  },
  expireAfterSeconds: 7776000    // 90일 보관
})

db.llm_logs.insertOne({
  timestamp: new Date(),
  metadata: {
    service: "chatbot-api",
    environment: "production",
    model: "claude-sonnet-4-20250514"
  },
  request: {
    prompt_template: "customer_support_v3",
    input_tokens: 1250,
    user_id: "user_hash_abc"
  },
  response: {
    output_tokens: 450,
    latency_ms: 890,
    finish_reason: "end_turn"
  },
  cost: {
    input_cost_usd: 0.00375,
    output_cost_usd: 0.00675,
    total_cost_usd: 0.0105
  },
  quality: {
    user_rating: null,           // 사용자 피드백 대기
    auto_eval_score: 0.87        // 자동 평가 점수
  }
})

// 모델별 일간 비용 분석
db.llm_logs.aggregate([
  {
    $match: {
      timestamp: { $gte: new Date(Date.now() - 86400000) }
    }
  },
  {
    $group: {
      _id: "$metadata.model",
      total_cost: { $sum: "$cost.total_cost_usd" },
      total_requests: { $sum: 1 },
      avg_latency: { $avg: "$response.latency_ms" },
      total_input_tokens: { $sum: "$request.input_tokens" },
      total_output_tokens: { $sum: "$response.output_tokens" }
    }
  },
  { $sort: { total_cost: -1 } }
])
```

---

## 부록: 빠른 참조

### 자주 사용하는 업데이트 연산자

| 연산자 | 설명 | 예시 |
|--------|------|------|
| `$set` | 필드 값 설정 | `{ $set: { status: "active" } }` |
| `$unset` | 필드 삭제 | `{ $unset: { temp: "" } }` |
| `$inc` | 숫자 증감 | `{ $inc: { count: 1 } }` |
| `$push` | 배열에 추가 | `{ $push: { tags: "new" } }` |
| `$pull` | 배열에서 제거 | `{ $pull: { tags: "old" } }` |
| `$addToSet` | 배열에 중복 없이 추가 | `{ $addToSet: { tags: "unique" } }` |
| `$rename` | 필드 이름 변경 | `{ $rename: { "old": "new" } }` |
| `$min` / `$max` | 조건부 갱신 | `{ $min: { low_score: 50 } }` |
| `$currentDate` | 현재 시각 설정 | `{ $currentDate: { updated: true } }` |

### 자주 사용하는 쿼리 연산자

| 연산자 | 설명 |
|--------|------|
| `$eq`, `$ne` | 같음 / 다름 |
| `$gt`, `$gte`, `$lt`, `$lte` | 비교 |
| `$in`, `$nin` | 포함 / 미포함 |
| `$and`, `$or`, `$not`, `$nor` | 논리 |
| `$exists` | 필드 존재 여부 |
| `$type` | BSON 타입 체크 |
| `$regex` | 정규 표현식 |
| `$all` | 배열 내 모든 요소 포함 |
| `$elemMatch` | 배열 요소 복합 조건 |
| `$size` | 배열 길이 |
| `$expr` | 집계 표현식 사용 |

### 연결 문자열 템플릿

```
# Standalone
mongodb://user:password@host:27017/dbname

# Replica Set
mongodb://user:password@host1:27017,host2:27017,host3:27017/dbname?replicaSet=rs0

# Atlas (SRV)
mongodb+srv://user:password@cluster.xxxxx.mongodb.net/dbname?retryWrites=true&w=majority

# 전체 옵션 예시
mongodb://user:password@host:27017/dbname?
  authSource=admin&
  replicaSet=rs0&
  w=majority&
  readPreference=secondaryPreferred&
  maxPoolSize=200&
  connectTimeoutMS=10000&
  retryWrites=true
```

---

> **이 문서는 MongoDB 7.x 기준으로 작성되었습니다.**  
> 최신 기능이나 변경 사항은 [MongoDB 공식 문서](https://www.mongodb.com/docs/)를 참고하세요.
