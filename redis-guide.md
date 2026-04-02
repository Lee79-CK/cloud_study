# Redis In-Memory Database 완벽 가이드

> 기본 개념부터 고급 활용, 실전 운영까지 — 실무 중심 강의 노트

---

## 목차

1. [Redis 개요](#1-redis-개요)
2. [핵심 아키텍처](#2-핵심-아키텍처)
3. [데이터 타입과 명령어](#3-데이터-타입과-명령어)
4. [고급 개념](#4-고급-개념)
5. [영속성(Persistence)](#5-영속성persistence)
6. [복제와 고가용성](#6-복제와-고가용성)
7. [Redis Cluster](#7-redis-cluster)
8. [성능 최적화](#8-성능-최적화)
9. [보안](#9-보안)
10. [설치 및 운영](#10-설치-및-운영)
11. [실전 활용 사례](#11-실전-활용-사례)
12. [MLOps/LLMOps에서의 Redis 활용](#12-mlopsllmops에서의-redis-활용)
13. [모니터링과 트러블슈팅](#13-모니터링과-트러블슈팅)
14. [Redis vs 다른 솔루션 비교](#14-redis-vs-다른-솔루션-비교)

---

## 1. Redis 개요

### 1.1 Redis란?

Redis(Remote Dictionary Server)는 오픈소스 인메모리 데이터 구조 저장소로, 데이터베이스·캐시·메시지 브로커·스트리밍 엔진으로 활용된다. Salvatore Sanfilippo가 2009년에 처음 릴리스했으며, 현재는 Redis Ltd.가 관리하고 있다.

핵심 특징을 정리하면 다음과 같다.

- **인메모리 저장**: 모든 데이터를 RAM에 저장하여 마이크로초~밀리초 단위의 응답 속도를 보장한다.
- **싱글 스레드 이벤트 루프**: 하나의 메인 스레드가 모든 명령을 순차 처리하므로 락(lock) 경합이 없다.
- **다양한 데이터 구조**: String, Hash, List, Set, Sorted Set, Stream, Bitmap, HyperLogLog 등을 네이티브로 지원한다.
- **영속성 옵션**: RDB 스냅샷과 AOF(Append Only File)를 통해 디스크에 데이터를 기록할 수 있다.
- **클러스터링/복제**: 수평 확장(Cluster)과 마스터-레플리카 복제를 지원한다.

### 1.2 왜 Redis인가?

전통적인 RDBMS가 디스크 I/O에 의존하는 반면, Redis는 모든 연산을 메모리에서 수행한다. 단순 GET/SET 기준으로 초당 수십만 건의 요청을 단일 인스턴스에서 처리할 수 있다. 이러한 특성 때문에 "핫 데이터" 캐싱, 실시간 분석, 세션 관리, 리더보드, 메시지 큐 등 지연 시간에 민감한 워크로드에서 사실상 표준으로 자리잡았다.

### 1.3 라이선스 변경 이슈

2024년 3월부터 Redis는 기존 BSD 라이선스에서 SSPL + RSALv2 듀얼 라이선스로 전환했다. 이에 대응하여 Linux Foundation 산하에서 **Valkey**라는 포크 프로젝트가 시작되었고, AWS ElastiCache 등 주요 클라우드 서비스도 Valkey 지원을 추가했다. 운영 환경에서 라이선스 조건을 반드시 검토해야 한다.

---

## 2. 핵심 아키텍처

### 2.1 싱글 스레드 모델

Redis의 메인 이벤트 루프는 단일 스레드로 동작한다. 클라이언트 요청을 순차적으로 처리하기 때문에 원자성(atomicity)이 자연스럽게 보장된다.

```
┌─────────────────────────────────────────────┐
│               Redis Process                  │
│                                              │
│  ┌──────────┐    ┌──────────────────────┐   │
│  │  I/O     │───▶│  Event Loop          │   │
│  │ Threads  │    │  (Single Thread)     │   │
│  │ (6.0+)   │◀───│  - 명령 파싱/실행     │   │
│  └──────────┘    │  - 결과 반환          │   │
│                  └──────────────────────┘   │
│                          │                   │
│                  ┌───────▼────────┐          │
│                  │  Memory (RAM)  │          │
│                  └───────┬────────┘          │
│                          │                   │
│                  ┌───────▼────────┐          │
│                  │  Disk (RDB/AOF)│          │
│                  └────────────────┘          │
└─────────────────────────────────────────────┘
```

Redis 6.0부터 I/O 멀티스레딩이 도입되었지만, 이는 네트워크 읽기/쓰기에만 적용되며 실제 명령 실행은 여전히 싱글 스레드이다.

### 2.2 메모리 관리

Redis의 메모리 구조를 이해하는 것은 운영에 매우 중요하다.

**메모리 할당자**: Redis는 기본적으로 jemalloc을 사용한다. jemalloc은 메모리 단편화를 줄이고 멀티코어 환경에서의 성능을 최적화한다.

**메모리 오버헤드**: 각 키-값 쌍은 실제 데이터 외에도 redisObject 헤더(16바이트), dictEntry(24바이트), SDS 헤더 등의 메타데이터를 포함한다. 작은 값을 대량으로 저장할 때는 이 오버헤드가 무시할 수 없다.

**maxmemory 설정**: 메모리 한계를 설정하고, 한계에 도달했을 때의 축출 정책(eviction policy)을 지정한다.

```
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

### 2.3 축출 정책 (Eviction Policy)

| 정책 | 설명 |
|------|------|
| `noeviction` | 메모리 초과 시 쓰기 명령에 에러 반환 (기본값) |
| `allkeys-lru` | 모든 키 중 가장 최근에 사용되지 않은 키 축출 |
| `allkeys-lfu` | 모든 키 중 가장 적게 사용된 키 축출 |
| `volatile-lru` | TTL이 설정된 키 중 LRU 축출 |
| `volatile-lfu` | TTL이 설정된 키 중 LFU 축출 |
| `volatile-ttl` | TTL이 가장 짧은 키 우선 축출 |
| `allkeys-random` | 무작위 축출 |
| `volatile-random` | TTL 설정된 키 중 무작위 축출 |

캐시 용도라면 `allkeys-lru` 또는 `allkeys-lfu`가 적합하고, 영속 데이터와 캐시가 혼재된 경우 `volatile-lru`를 고려한다.

---

## 3. 데이터 타입과 명령어

### 3.1 String

가장 기본적인 타입으로, 최대 512MB의 텍스트나 바이너리 데이터를 저장할 수 있다.

```bash
# 기본 SET/GET
SET user:1001:name "Alice"
GET user:1001:name               # "Alice"

# TTL 설정 (초 단위)
SET session:abc123 "data" EX 3600

# 조건부 설정
SET lock:resource1 "owner1" NX   # 키가 없을 때만 설정 (분산 락)
SET counter:page "100" XX        # 키가 있을 때만 설정

# 숫자 연산 (원자적)
INCR counter:visits              # 1 증가
INCRBY counter:visits 10         # 10 증가
INCRBYFLOAT price:item1 0.5      # 실수 증가

# 여러 키 한번에 처리
MSET k1 "v1" k2 "v2" k3 "v3"
MGET k1 k2 k3
```

**내부 인코딩**: 값이 정수이면 `int` 인코딩, 44바이트 이하 문자열이면 `embstr`, 그 이상이면 `raw`로 저장된다. `OBJECT ENCODING key`로 확인 가능하다.

### 3.2 Hash

필드-값 쌍의 집합으로, 하나의 키 아래 여러 속성을 저장할 때 이상적이다.

```bash
# 해시 설정
HSET user:1001 name "Alice" age 30 email "alice@example.com"

# 개별 필드 조회
HGET user:1001 name              # "Alice"

# 전체 필드 조회
HGETALL user:1001
# 1) "name"    2) "Alice"
# 3) "age"     4) "30"
# 5) "email"   6) "alice@example.com"

# 숫자 필드 증감
HINCRBY user:1001 age 1          # 31

# 필드 존재 여부
HEXISTS user:1001 phone          # 0 (없음)

# 필드 삭제
HDEL user:1001 email
```

**최적화 포인트**: 필드 수가 `hash-max-ziplist-entries`(기본 128) 이하이고 각 값이 `hash-max-ziplist-value`(기본 64바이트) 이하이면 ziplist(listpack) 인코딩을 사용하여 메모리를 절약한다.

### 3.3 List

삽입 순서가 보장되는 연결 리스트이다.

```bash
# 양쪽 삽입
LPUSH queue:tasks "task3" "task2" "task1"   # 왼쪽에 삽입
RPUSH queue:tasks "task4"                    # 오른쪽에 삽입

# 양쪽 추출
LPOP queue:tasks                # "task1"
RPOP queue:tasks                # "task4"

# 범위 조회 (0-based, -1은 마지막)
LRANGE queue:tasks 0 -1

# 블로킹 팝 (메시지 큐 패턴)
BLPOP queue:tasks 30            # 최대 30초 대기

# 리스트 길이
LLEN queue:tasks

# 특정 인덱스 조회
LINDEX queue:tasks 0
```

### 3.4 Set

중복 없는 문자열 집합이다.

```bash
# 멤버 추가
SADD tags:post:1001 "redis" "database" "nosql"

# 전체 멤버 조회
SMEMBERS tags:post:1001

# 멤버 존재 여부
SISMEMBER tags:post:1001 "redis"   # 1 (있음)

# 집합 연산
SADD tags:post:1002 "redis" "cache" "performance"
SINTER tags:post:1001 tags:post:1002     # 교집합: {"redis"}
SUNION tags:post:1001 tags:post:1002     # 합집합
SDIFF tags:post:1001 tags:post:1002      # 차집합

# 멤버 수
SCARD tags:post:1001

# 랜덤 멤버
SRANDMEMBER tags:post:1001 2
```

### 3.5 Sorted Set (ZSet)

각 멤버에 점수(score)가 부여되어 자동 정렬되는 집합이다. 리더보드, 랭킹, 시계열 인덱스에 핵심적이다.

```bash
# 멤버 추가 (score member)
ZADD leaderboard 1500 "player:alice"
ZADD leaderboard 2300 "player:bob"
ZADD leaderboard 1800 "player:charlie"

# 순위 조회 (0-based, 오름차순)
ZRANK leaderboard "player:bob"           # 2 (1등 = index 2, 내림차순 시)
ZREVRANK leaderboard "player:bob"        # 0 (1등)

# 상위 N명 (내림차순)
ZREVRANGE leaderboard 0 2 WITHSCORES

# 점수 범위 조회
ZRANGEBYSCORE leaderboard 1000 2000 WITHSCORES

# 점수 증가
ZINCRBY leaderboard 200 "player:alice"   # 1700

# 멤버 수
ZCARD leaderboard

# 점수 범위 내 멤버 수
ZCOUNT leaderboard 1500 2000
```

### 3.6 Stream

Redis 5.0에서 도입된 append-only 로그 구조로, Kafka와 유사한 메시지 스트리밍을 지원한다.

```bash
# 메시지 추가 (* = 자동 ID 생성)
XADD events * type "click" user "alice" page "/home"
XADD events * type "purchase" user "bob" item "widget"

# 범위 조회
XRANGE events - +                # 전체 조회
XRANGE events - + COUNT 10      # 최근 10개

# 길이
XLEN events

# Consumer Group 생성
XGROUP CREATE events mygroup $ MKSTREAM

# Consumer Group으로 읽기
XREADGROUP GROUP mygroup consumer1 COUNT 5 BLOCK 2000 STREAMS events >

# 처리 완료 확인
XACK events mygroup "1234567890-0"

# 미처리 메시지 확인
XPENDING events mygroup
```

### 3.7 기타 특수 데이터 타입

**Bitmap**: 비트 단위 연산으로 일별 활성 사용자 추적 등에 활용한다.

```bash
SETBIT user:active:2026-04-01 1001 1   # 사용자 1001 활성 표시
GETBIT user:active:2026-04-01 1001     # 1
BITCOUNT user:active:2026-04-01        # 활성 사용자 수
BITOP AND result key1 key2             # 비트 AND 연산
```

**HyperLogLog**: 고유 원소 수를 추정하는 확률적 자료구조로, 최대 12KB만 사용하면서 표준 오차 0.81%의 카디널리티 추정이 가능하다.

```bash
PFADD visitors:2026-04 "user1" "user2" "user3"
PFCOUNT visitors:2026-04              # 약 3
PFMERGE visitors:q1 visitors:2026-01 visitors:2026-02 visitors:2026-03
```

**Geospatial**: 좌표 기반 검색을 지원한다.

```bash
GEOADD stores 126.978 37.566 "store:gangnam"
GEOADD stores 126.972 37.556 "store:sinsa"
GEOSEARCH stores FROMLONLAT 126.975 37.560 BYRADIUS 2 km ASC
```

---

## 4. 고급 개념

### 4.1 트랜잭션 (MULTI/EXEC)

Redis 트랜잭션은 여러 명령을 하나의 원자적 단위로 실행한다.

```bash
MULTI
SET account:alice:balance 900
SET account:bob:balance 1100
EXEC
```

주의: Redis 트랜잭션은 RDBMS의 ROLLBACK을 지원하지 않는다. EXEC 이후 개별 명령이 실패하더라도 나머지 명령은 계속 실행된다.

**WATCH를 이용한 낙관적 락(Optimistic Locking)**:

```bash
WATCH account:alice:balance
val = GET account:alice:balance       # "1000"
MULTI
SET account:alice:balance 900         # 1000 - 100
SET account:bob:balance 1100          # 1000 + 100
EXEC
# WATCH 이후 다른 클라이언트가 키를 변경했으면 EXEC는 nil 반환
```

### 4.2 Lua 스크립팅

서버 측에서 원자적으로 복잡한 로직을 실행할 수 있다. 네트워크 라운드트립을 줄이고 원자성을 보장한다.

```lua
-- rate_limiter.lua
-- 슬라이딩 윈도우 Rate Limiter
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- 윈도우 밖의 요소 제거
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- 현재 카운트 확인
local count = redis.call('ZCARD', key)

if count < limit then
    redis.call('ZADD', key, now, now .. '-' .. math.random())
    redis.call('EXPIRE', key, window)
    return 1  -- 허용
else
    return 0  -- 거부
end
```

```bash
# 실행
EVALSHA <sha1> 1 rate:api:user1001 100 60 1711929600
```

Redis 7.0+에서는 `FUNCTION` 명령으로 Lua 함수를 서버에 영구 등록할 수 있다.

```bash
# 함수 등록
FUNCTION LOAD "#!lua name=mylib\nredis.register_function('my_func', function(keys, args) ... end)"

# 호출
FCALL my_func 1 mykey arg1 arg2
```

### 4.3 Pub/Sub

발행-구독 메시징 패턴이다.

```bash
# 구독자
SUBSCRIBE news:tech news:science

# 패턴 구독
PSUBSCRIBE news:*

# 발행자
PUBLISH news:tech "Redis 8.0 released!"
```

Pub/Sub은 메시지를 영속하지 않으며, 구독자가 없으면 메시지가 소실된다. 영속이 필요하면 Stream을 사용한다.

### 4.4 파이프라이닝 (Pipelining)

여러 명령을 한 번에 전송하여 네트워크 라운드트립을 줄인다.

```python
import redis

r = redis.Redis()
pipe = r.pipeline(transaction=False)

for i in range(10000):
    pipe.set(f"key:{i}", f"value:{i}")

pipe.execute()  # 한 번에 전송, RTT 1회
```

파이프라이닝 없이 10,000개를 개별 전송하면 RTT × 10,000이 발생하지만, 파이프라이닝을 쓰면 사실상 RTT 1회로 줄어든다.

### 4.5 Redis Modules

Redis의 기능을 확장하는 동적 라이브러리이다.

| 모듈 | 설명 |
|------|------|
| **RediSearch** | 풀텍스트 검색 엔진, 인덱싱, 집계 |
| **RedisJSON** | 네이티브 JSON 저장/조회/인덱싱 |
| **RedisTimeSeries** | 시계열 데이터 저장, 다운샘플링, 집계 |
| **RedisBloom** | Bloom Filter, Cuckoo Filter, Count-Min Sketch, Top-K |
| **RedisGraph** | Cypher 쿼리 기반 그래프 DB (deprecated) |
| **RedisAI** | 텐서 저장 및 모델 추론 실행 |

```bash
# RedisJSON 예시
JSON.SET user:1001 $ '{"name":"Alice","scores":[90,85,92]}'
JSON.GET user:1001 $.scores[0]   # 90
JSON.NUMINCRBY user:1001 $.scores[0] 5  # 95

# RediSearch 예시
FT.CREATE idx:users ON JSON PREFIX 1 user: SCHEMA $.name AS name TEXT $.age AS age NUMERIC
FT.SEARCH idx:users "@name:Alice"
```

### 4.6 키 만료(TTL) 메커니즘

Redis의 키 만료는 두 가지 방식으로 작동한다.

**수동적(Passive) 만료**: 클라이언트가 만료된 키에 접근할 때 삭제한다.

**능동적(Active) 만료**: 주기적으로(초당 10회) 만료 키 샘플을 추출하여 삭제한다. 전체 키 중 만료된 비율이 25% 이상이면 반복 수행한다.

```bash
SET temp:data "value"
EXPIRE temp:data 300              # 300초 후 만료
TTL temp:data                     # 남은 시간 확인
PERSIST temp:data                 # TTL 제거 (영구 보존)
EXPIREAT temp:data 1711929600     # Unix timestamp 기준 만료
PEXPIRE temp:data 300000          # 밀리초 단위 만료
```

---

## 5. 영속성(Persistence)

### 5.1 RDB (Redis Database Snapshot)

특정 시점의 전체 데이터셋을 바이너리 파일로 저장한다.

```
# redis.conf
save 900 1       # 900초 동안 1개 이상 변경 시 저장
save 300 10      # 300초 동안 10개 이상 변경 시 저장
save 60 10000    # 60초 동안 10000개 이상 변경 시 저장

dbfilename dump.rdb
dir /var/lib/redis
```

**동작 원리**: `fork()`를 통해 자식 프로세스를 생성하고, Copy-on-Write(CoW) 메커니즘으로 메모리를 효율적으로 공유하면서 스냅샷을 작성한다. 부모 프로세스는 계속 요청을 처리한다.

**장점**: 컴팩트한 바이너리 파일, 빠른 복구 속도, 재해 복구에 적합하다.

**단점**: fork() 시 메모리 사용량 증가(최악의 경우 2배), 마지막 스냅샷 이후 데이터 유실 가능하다.

### 5.2 AOF (Append Only File)

모든 쓰기 명령을 로그 형태로 기록한다.

```
# redis.conf
appendonly yes
appendfilename "appendonly.aof"

# fsync 정책
appendfsync always     # 매 명령마다 (가장 안전, 가장 느림)
appendfsync everysec   # 1초마다 (권장, 기본값)
appendfsync no         # OS에 맡김 (가장 빠름, 위험)
```

**AOF Rewrite**: AOF 파일이 커지면 현재 데이터셋을 최소 명령으로 재작성한다.

```
# 자동 리라이트 조건
auto-aof-rewrite-percentage 100   # AOF 크기가 100% 증가 시
auto-aof-rewrite-min-size 64mb    # 최소 64MB 이상일 때
```

Redis 7.0+에서는 Multi Part AOF를 지원하여 리라이트 중 서비스 영향을 최소화한다.

### 5.3 RDB + AOF 하이브리드 모드

Redis 4.0+에서 도입된 방식으로, AOF 리라이트 시 RDB 형식의 프리앰블과 이후 변경분의 AOF 명령을 결합한다.

```
aof-use-rdb-preamble yes   # 기본 활성화 (Redis 4.0+)
```

이 방식은 빠른 로딩 속도(RDB의 장점)와 높은 내구성(AOF의 장점)을 동시에 확보한다. **프로덕션 환경에서 권장되는 설정**이다.

### 5.4 영속성 전략 비교

| 기준 | RDB Only | AOF Only | RDB + AOF |
|------|----------|----------|-----------|
| 데이터 안전성 | 낮음 (분 단위 유실) | 높음 (초 단위) | 가장 높음 |
| 복구 속도 | 빠름 | 느림 | 빠름 |
| 파일 크기 | 작음 | 큼 | 중간 |
| fork() 빈도 | 설정에 따라 | 리라이트 시 | 리라이트 시 |
| 프로덕션 권장 | 캐시 전용 | 단독 사용 비권장 | **권장** |

---

## 6. 복제와 고가용성

### 6.1 마스터-레플리카 복제

Redis 복제는 비동기 방식이 기본이다.

```
# replica 서버의 redis.conf
replicaof 192.168.1.100 6379
masterauth "your-master-password"

# 또는 런타임에서
REPLICAOF 192.168.1.100 6379
```

**복제 과정**:
1. 레플리카가 마스터에 `PSYNC` 명령 전송
2. 최초 연결 시: 마스터가 RDB 스냅샷을 생성하여 전송 (Full Sync)
3. 이후: 마스터의 쓰기 명령이 복제 버퍼를 통해 실시간 전파 (Partial Sync)
4. 연결 끊김 후 재연결 시: replication backlog를 이용한 부분 동기화 시도

```
# replication backlog 설정
repl-backlog-size 256mb    # 기본 1mb, 프로덕션에서는 크게 설정
repl-backlog-ttl 3600
```

**주의사항**: 비동기 복제이므로 마스터 장애 시 아직 복제되지 않은 데이터가 유실될 수 있다. `min-replicas-to-write`와 `min-replicas-max-lag` 설정으로 최소한의 동기 보장을 할 수 있다.

```
min-replicas-to-write 1      # 최소 1개 레플리카에 복제되어야 쓰기 허용
min-replicas-max-lag 10       # 레플리카 지연이 10초 이내여야 함
```

### 6.2 Redis Sentinel

자동 장애 감지(failover)와 서비스 디스커버리를 제공한다.

```
# sentinel.conf
sentinel monitor mymaster 192.168.1.100 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
sentinel auth-pass mymaster "your-password"
```

**Sentinel의 역할**:
- **모니터링**: 마스터와 레플리카의 상태를 주기적으로 확인한다.
- **알림**: 장애 발생 시 관리자/외부 시스템에 통보한다.
- **자동 페일오버**: 마스터 장애 시 레플리카를 새 마스터로 승격한다.
- **서비스 디스커버리**: 클라이언트에 현재 마스터의 주소를 제공한다.

**Quorum**: `sentinel monitor`의 마지막 인자(위 예시에서 2)는 마스터 다운 판정에 필요한 Sentinel 동의 수이다. 최소 3개의 Sentinel 인스턴스를 서로 다른 장애 도메인(AZ, 호스트)에 배치해야 한다.

```python
# Python 클라이언트 연결 예시
from redis.sentinel import Sentinel

sentinel = Sentinel(
    [('sentinel1', 26379), ('sentinel2', 26379), ('sentinel3', 26379)],
    socket_timeout=0.5
)
master = sentinel.master_for('mymaster', socket_timeout=0.5)
replica = sentinel.slave_for('mymaster', socket_timeout=0.5)

master.set('key', 'value')
value = replica.get('key')
```

---

## 7. Redis Cluster

### 7.1 개요

Redis Cluster는 데이터를 자동으로 여러 노드에 분산(샤딩)하여 수평 확장을 가능하게 한다.

**핵심 개념**: 16,384개의 해시 슬롯(Hash Slot)을 노드들에 분배한다. 키의 해시 슬롯은 `CRC16(key) % 16384`로 결정된다.

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Node A     │  │  Node B     │  │  Node C     │
│ Slots 0-5460│  │ Slots 5461- │  │ Slots 10923-│
│  + Replica  │  │ 10922       │  │ 16383       │
│             │  │  + Replica  │  │  + Replica  │
└─────────────┘  └─────────────┘  └─────────────┘
```

### 7.2 클러스터 생성

```bash
# 최소 6개 노드 (마스터 3 + 레플리카 3) 준비
# 각 노드의 redis.conf
port 7000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000
appendonly yes

# 클러스터 생성 (Redis CLI)
redis-cli --cluster create \
  192.168.1.101:7000 192.168.1.102:7000 192.168.1.103:7000 \
  192.168.1.104:7000 192.168.1.105:7000 192.168.1.106:7000 \
  --cluster-replicas 1
```

### 7.3 Hash Tag

같은 슬롯에 배치해야 하는 키들은 Hash Tag `{}`를 사용한다.

```bash
# {user:1001} 부분만 해시 계산에 사용됨
SET {user:1001}:profile "data"
SET {user:1001}:settings "data"
SET {user:1001}:cart "data"

# 같은 슬롯이므로 멀티키 연산 가능
MGET {user:1001}:profile {user:1001}:settings
```

### 7.4 클러스터 리샤딩과 관리

```bash
# 슬롯 마이그레이션 (리샤딩)
redis-cli --cluster reshard 192.168.1.101:7000

# 노드 추가
redis-cli --cluster add-node new-host:7000 existing-host:7000

# 노드 제거
redis-cli --cluster del-node host:7000 <node-id>

# 클러스터 상태 확인
redis-cli --cluster check 192.168.1.101:7000
redis-cli --cluster info 192.168.1.101:7000
```

### 7.5 클러스터의 제약사항

- 멀티키 연산은 같은 슬롯의 키에만 가능하다 (Hash Tag 필요).
- 데이터베이스는 0번만 사용 가능하다 (`SELECT` 불가).
- Lua 스크립트 내에서 접근하는 모든 키는 같은 슬롯이어야 한다.
- `KEYS`, `SCAN` 등은 해당 노드의 키만 반환한다.

---

## 8. 성능 최적화

### 8.1 키 설계 원칙

키 이름은 가독성과 효율성을 모두 고려해야 한다.

```bash
# 권장: 콜론으로 구분, 의미 있는 계층 구조
user:1001:profile
order:2026:04:01:00001
cache:api:v2:products:list

# 비권장: 너무 길거나 의미 없는 키
this_is_a_very_long_key_name_that_wastes_memory_for_no_reason
```

### 8.2 빅키(Big Key) 방지

하나의 키에 과도하게 큰 값을 저장하면 다음 문제가 발생한다.

- 해당 키 접근 시 다른 요청이 블로킹된다.
- 삭제 시 순간적인 지연이 발생한다 (`UNLINK` 사용으로 비동기 삭제 가능).
- 클러스터에서 데이터 편중(hot spot)이 생긴다.

```bash
# 빅키 탐지
redis-cli --bigkeys

# MEMORY USAGE로 개별 키 크기 확인
MEMORY USAGE user:1001:followers

# 대안: 데이터를 분할
# Before (빅키)
SADD user:1001:followers "user1" "user2" ... (100만 멤버)

# After (분할)
SADD user:1001:followers:bucket:0 ...
SADD user:1001:followers:bucket:1 ...
```

### 8.3 SCAN 패턴 (KEYS 대체)

`KEYS *` 명령은 전체 키를 순회하므로 프로덕션에서 절대 사용하면 안 된다.

```bash
# SCAN: 커서 기반 점진적 순회
SCAN 0 MATCH "user:*" COUNT 100
# 반환: (다음커서, 매칭된 키 목록)

# 타입별 SCAN
HSCAN user:1001 0 MATCH "field*" COUNT 10
SSCAN myset 0 MATCH "prefix*" COUNT 10
ZSCAN myzset 0 MATCH "*" COUNT 10
```

### 8.4 Connection Pooling

클라이언트 측에서 커넥션 풀을 반드시 사용해야 한다.

```python
import redis

# 커넥션 풀 (권장)
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    db=0,
    max_connections=50,
    decode_responses=True,
    socket_timeout=5,
    socket_connect_timeout=5,
    retry_on_timeout=True
)
r = redis.Redis(connection_pool=pool)
```

### 8.5 Slow Log 분석

```bash
# 실행 시간이 10ms 이상인 명령 기록
CONFIG SET slowlog-log-slower-than 10000  # 마이크로초 단위
CONFIG SET slowlog-max-len 128

# Slow Log 조회
SLOWLOG GET 10
SLOWLOG LEN
SLOWLOG RESET
```

### 8.6 메모리 최적화 테크닉

**작은 값에는 내부 인코딩 최적화를 활용한다**:

```
# ziplist/listpack 임계값 조정
hash-max-ziplist-entries 128
hash-max-ziplist-value 64
list-max-ziplist-size -2
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
set-max-intset-entries 512
```

**메모리 단편화 관리**:

```bash
# 단편화 비율 확인
INFO memory
# mem_fragmentation_ratio > 1.5이면 단편화 심각

# 활성 단편화 정리 (Redis 4.0+)
CONFIG SET activedefrag yes
```

---

## 9. 보안

### 9.1 인증

```
# redis.conf (Redis 6.0+ ACL)
requirepass "strong-password-here"

# ACL 사용자 관리
ACL SETUSER readonly on >password123 ~cache:* +get +mget +scan +hgetall -@write
ACL SETUSER admin on >admin-secret ~* +@all
ACL LIST
ACL WHOAMI
```

### 9.2 네트워크 보안

```
# 바인딩 제한
bind 127.0.0.1 192.168.1.100
protected-mode yes

# TLS 암호화 (Redis 6.0+)
tls-port 6380
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
tls-auth-clients optional
```

### 9.3 위험한 명령 비활성화

```
# redis.conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG "CONFIG_b840fc02"
rename-command DEBUG ""
rename-command KEYS ""
```

### 9.4 보안 체크리스트

프로덕션 환경에서 최소한 다음 항목을 확인해야 한다.

- `protected-mode yes` 활성화
- 강력한 비밀번호 설정
- 불필요한 네트워크 바인딩 제거
- 방화벽으로 Redis 포트(6379) 제한
- ACL로 사용자별 최소 권한 부여
- TLS 암호화 활성화
- `KEYS`, `FLUSHALL` 등 위험 명령 비활성화 또는 이름 변경
- Redis 프로세스를 비루트 사용자로 실행

---

## 10. 설치 및 운영

### 10.1 설치 방법

**Ubuntu/Debian**:

```bash
# 공식 저장소에서 설치
sudo apt update
sudo apt install -y lsb-release curl gpg

curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

sudo apt update
sudo apt install -y redis
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

**소스 컴파일**:

```bash
wget https://download.redis.io/redis-stable.tar.gz
tar xzf redis-stable.tar.gz
cd redis-stable
make
make test
sudo make install

# 설정 파일 복사
sudo mkdir -p /etc/redis
sudo cp redis.conf /etc/redis/redis.conf
```

**Docker**:

```bash
# 단일 인스턴스
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7.4 \
  redis-server --appendonly yes --requirepass "your-password"

# Docker Compose
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  redis:
    image: redis:7.4
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    deploy:
      resources:
        limits:
          memory: 4g
    restart: unless-stopped

volumes:
  redis-data:
```

**Kubernetes (Helm)**:

```bash
# Bitnami Redis Helm Chart
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis bitnami/redis \
  --set auth.password="your-password" \
  --set replica.replicaCount=3 \
  --set master.persistence.size=10Gi \
  --set master.resources.requests.memory=4Gi \
  --set master.resources.limits.memory=4Gi
```

### 10.2 프로덕션 redis.conf 핵심 설정

```
# === 네트워크 ===
bind 0.0.0.0
port 6379
tcp-backlog 511
timeout 300
tcp-keepalive 300

# === 메모리 ===
maxmemory 4gb
maxmemory-policy allkeys-lru

# === 영속성 (하이브리드 모드) ===
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# === 복제 ===
repl-backlog-size 256mb
repl-backlog-ttl 3600
min-replicas-to-write 1
min-replicas-max-lag 10

# === 성능 ===
io-threads 4                    # CPU 코어 수에 따라 조정
io-threads-do-reads yes
lazyfree-lazy-eviction yes      # 비동기 축출
lazyfree-lazy-expire yes        # 비동기 만료 삭제
lazyfree-lazy-server-del yes

# === 보안 ===
requirepass "your-strong-password"
protected-mode yes
rename-command FLUSHALL ""
rename-command FLUSHDB ""

# === 로깅 ===
loglevel notice
logfile /var/log/redis/redis.log
slowlog-log-slower-than 10000
slowlog-max-len 128
```

### 10.3 Linux 커널 튜닝

Redis가 최적으로 동작하기 위해 OS 수준의 튜닝이 필요하다.

```bash
# /etc/sysctl.conf
vm.overcommit_memory = 1          # fork() 시 OOM Killer 방지
net.core.somaxconn = 65535        # TCP 백로그
net.ipv4.tcp_max_syn_backlog = 65535

# Transparent Huge Pages 비활성화 (필수!)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 적용
sysctl -p

# systemd 서비스 파일에서 ulimit 설정
# /etc/systemd/system/redis.service.d/limit.conf
[Service]
LimitNOFILE=65535
```

**THP(Transparent Huge Pages) 비활성화는 매우 중요하다**. THP가 활성화되면 fork() 시 Copy-on-Write 오버헤드가 극적으로 증가하여 지연 시간 스파이크가 발생한다.

### 10.4 클라우드 관리형 서비스

직접 운영 대신 클라우드 관리형 서비스를 고려할 수 있다.

| 서비스 | 제공사 | 특징 |
|--------|--------|------|
| **Amazon ElastiCache** | AWS | Redis/Valkey 호환, 자동 페일오버, 클러스터 모드 |
| **Amazon MemoryDB** | AWS | 내구성 보장, 마이크로초 읽기/밀리초 쓰기 |
| **Azure Cache for Redis** | Azure | 엔터프라이즈 티어(Redis Enterprise 기반) |
| **Google Memorystore** | GCP | 완전 관리형, VPC 피어링 |
| **Redis Cloud** | Redis Ltd. | 멀티 클라우드, Active-Active Geo 복제 |

---

## 11. 실전 활용 사례

### 11.1 세션 스토어

```python
import redis
import json
import uuid

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def create_session(user_id, user_data):
    session_id = str(uuid.uuid4())
    session_key = f"session:{session_id}"
    r.hset(session_key, mapping={
        "user_id": user_id,
        "data": json.dumps(user_data),
        "created_at": "2026-04-01T09:00:00Z"
    })
    r.expire(session_key, 3600)  # 1시간 만료
    return session_id

def get_session(session_id):
    session_key = f"session:{session_id}"
    session = r.hgetall(session_key)
    if session:
        r.expire(session_key, 3600)  # 접근 시 TTL 갱신
    return session
```

### 11.2 분산 락 (Distributed Lock)

Redlock 알고리즘을 활용한 분산 환경 락이다.

```python
import redis
import time
import uuid

class RedisLock:
    def __init__(self, client, resource, ttl=10):
        self.client = client
        self.resource = f"lock:{resource}"
        self.ttl = ttl
        self.token = str(uuid.uuid4())

    def acquire(self, timeout=30):
        end = time.time() + timeout
        while time.time() < end:
            if self.client.set(self.resource, self.token, nx=True, ex=self.ttl):
                return True
            time.sleep(0.1)
        return False

    def release(self):
        # Lua 스크립트로 원자적 해제 (본인 토큰만 삭제)
        script = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
        """
        self.client.eval(script, 1, self.resource, self.token)

# 사용
r = redis.Redis()
lock = RedisLock(r, "critical-section", ttl=10)
if lock.acquire():
    try:
        # 크리티컬 섹션 작업 수행
        pass
    finally:
        lock.release()
```

### 11.3 Rate Limiter

```python
def is_rate_limited(r, user_id, limit=100, window=60):
    """슬라이딩 윈도우 Rate Limiter"""
    key = f"rate:{user_id}"
    now = time.time()

    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)  # 윈도우 밖 제거
    pipe.zcard(key)                               # 현재 카운트
    pipe.zadd(key, {f"{now}-{uuid.uuid4()}": now}) # 현재 요청 추가
    pipe.expire(key, window)                       # TTL 설정
    results = pipe.execute()

    current_count = results[1]
    if current_count >= limit:
        r.zrem(key, f"{now}-{uuid.uuid4()}")  # 초과 시 롤백
        return True  # 제한됨
    return False  # 허용
```

### 11.4 캐싱 패턴

**Cache-Aside (Lazy Loading)**:

```python
def get_user(user_id):
    cache_key = f"cache:user:{user_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # 캐시 미스: DB에서 조회
    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    r.set(cache_key, json.dumps(user), ex=300)
    return user
```

**Write-Through**:

```python
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    cache_key = f"cache:user:{user_id}"
    r.set(cache_key, json.dumps(data), ex=300)
```

**Write-Behind (Write-Back)**:

```python
def update_user_async(user_id, data):
    cache_key = f"cache:user:{user_id}"
    r.set(cache_key, json.dumps(data), ex=300)
    # 변경 사항을 큐에 저장, 백그라운드 워커가 DB에 반영
    r.xadd("stream:db-writes", {
        "table": "users",
        "id": user_id,
        "data": json.dumps(data)
    })
```

### 11.5 실시간 리더보드

```python
class Leaderboard:
    def __init__(self, r, name):
        self.r = r
        self.key = f"leaderboard:{name}"

    def update_score(self, player, score):
        self.r.zadd(self.key, {player: score})

    def increment_score(self, player, delta):
        return self.r.zincrby(self.key, delta, player)

    def get_rank(self, player):
        rank = self.r.zrevrank(self.key, player)
        return rank + 1 if rank is not None else None

    def get_top(self, n=10):
        return self.r.zrevrange(self.key, 0, n - 1, withscores=True)

    def get_around(self, player, n=5):
        rank = self.r.zrevrank(self.key, player)
        if rank is None:
            return []
        start = max(0, rank - n)
        end = rank + n
        return self.r.zrevrange(self.key, start, end, withscores=True)
```

### 11.6 실시간 알림 시스템

```python
# Producer
def send_notification(user_id, notification):
    stream_key = f"notifications:{user_id}"
    r.xadd(stream_key, notification, maxlen=1000)  # 최대 1000개 유지

# Consumer (Consumer Group)
def process_notifications(user_id, consumer_name):
    stream_key = f"notifications:{user_id}"
    group = "notification-processors"

    try:
        r.xgroup_create(stream_key, group, id='0', mkstream=True)
    except redis.exceptions.ResponseError:
        pass  # 그룹 이미 존재

    while True:
        messages = r.xreadgroup(group, consumer_name,
                                {stream_key: '>'},
                                count=10, block=5000)
        for stream, msgs in messages:
            for msg_id, data in msgs:
                process(data)
                r.xack(stream_key, group, msg_id)
```

---

## 12. MLOps/LLMOps에서의 Redis 활용

### 12.1 Feature Store 캐시

모델 서빙 시 실시간 피처를 빠르게 조회하기 위해 Redis를 Feature Store의 온라인 저장소로 활용한다.

```python
# 피처 저장 (배치 파이프라인에서)
def store_features(user_id, features: dict):
    key = f"features:user:{user_id}"
    r.hset(key, mapping={
        k: json.dumps(v) if isinstance(v, (list, dict)) else str(v)
        for k, v in features.items()
    })
    r.expire(key, 86400)  # 24시간 TTL

# 피처 조회 (추론 시)
def get_features(user_id, feature_names: list):
    key = f"features:user:{user_id}"
    values = r.hmget(key, feature_names)
    return dict(zip(feature_names, values))
```

Feast 등의 Feature Store 프레임워크에서도 Redis를 온라인 스토어 백엔드로 공식 지원한다.

### 12.2 LLM 응답 캐싱 (Semantic Cache)

동일하거나 유사한 프롬프트에 대한 LLM 응답을 캐싱하여 비용과 지연 시간을 절감한다.

```python
import hashlib
import numpy as np

def get_prompt_hash(prompt):
    return hashlib.sha256(prompt.strip().lower().encode()).hexdigest()

# 정확한 매칭 캐시
def cached_llm_call(prompt, model="claude-sonnet-4-20250514"):
    cache_key = f"llm:cache:{get_prompt_hash(prompt)}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    response = call_llm(prompt, model)
    r.set(cache_key, json.dumps(response), ex=3600)
    return response

# 시맨틱 캐시 (Redis + Vector Search)
# RediSearch의 벡터 유사도 검색 활용
def create_semantic_cache_index():
    r.execute_command(
        'FT.CREATE', 'idx:llm_cache',
        'ON', 'HASH', 'PREFIX', '1', 'llm:semantic:',
        'SCHEMA',
        'prompt', 'TEXT',
        'embedding', 'VECTOR', 'FLAT', '6',
            'TYPE', 'FLOAT32',
            'DIM', '1536',
            'DISTANCE_METRIC', 'COSINE',
        'response', 'TEXT',
        'created_at', 'NUMERIC'
    )
```

### 12.3 모델 서빙 큐

```python
# 추론 요청을 Redis Stream으로 큐잉
def enqueue_inference(request_id, input_data):
    r.xadd("stream:inference:requests", {
        "request_id": request_id,
        "input": json.dumps(input_data),
        "timestamp": str(time.time())
    })

# GPU 워커에서 배치 처리
def inference_worker(batch_size=32, timeout=100):
    """배치 추론 워커 - 요청을 모아서 GPU에서 한번에 처리"""
    group = "inference-workers"
    consumer = f"worker-{os.getpid()}"
    stream = "stream:inference:requests"

    try:
        r.xgroup_create(stream, group, id='0', mkstream=True)
    except redis.exceptions.ResponseError:
        pass

    while True:
        messages = r.xreadgroup(
            group, consumer,
            {stream: '>'},
            count=batch_size,
            block=timeout
        )

        if not messages:
            continue

        batch_inputs = []
        batch_ids = []
        for _, msgs in messages:
            for msg_id, data in msgs:
                batch_inputs.append(json.loads(data[b'input']))
                batch_ids.append((msg_id, data[b'request_id']))

        # 배치 추론
        results = model.predict(batch_inputs)

        # 결과 저장 및 ACK
        pipe = r.pipeline()
        for (msg_id, req_id), result in zip(batch_ids, results):
            pipe.set(f"inference:result:{req_id.decode()}", json.dumps(result), ex=300)
            pipe.xack(stream, group, msg_id)
        pipe.execute()
```

### 12.4 A/B 테스트 / 모델 라우팅

```python
# 모델 트래픽 비율 설정
def set_model_weights(experiment_id, weights: dict):
    """weights = {"model_a": 70, "model_b": 30}"""
    key = f"experiment:{experiment_id}:weights"
    r.hset(key, mapping=weights)

# 요청 라우팅
def route_request(experiment_id, user_id):
    key = f"experiment:{experiment_id}:weights"
    weights = r.hgetall(key)

    # 사용자별 일관된 라우팅 (sticky assignment)
    assignment_key = f"experiment:{experiment_id}:assignment:{user_id}"
    assigned = r.get(assignment_key)
    if assigned:
        return assigned

    # 가중 랜덤 선택
    models = list(weights.keys())
    probs = [int(w) for w in weights.values()]
    total = sum(probs)
    probs = [p / total for p in probs]

    chosen = np.random.choice(models, p=probs)
    r.set(assignment_key, chosen, ex=86400)
    return chosen
```

### 12.5 Embedding Vector 캐시

LLM 애플리케이션에서 동일 텍스트의 임베딩 재계산을 방지한다.

```python
def get_embedding_cached(text, model="text-embedding-3-small"):
    text_hash = hashlib.md5(text.encode()).hexdigest()
    cache_key = f"embedding:{model}:{text_hash}"

    cached = r.get(cache_key)
    if cached:
        return np.frombuffer(cached, dtype=np.float32)

    embedding = openai.embeddings.create(input=text, model=model)
    vector = np.array(embedding.data[0].embedding, dtype=np.float32)

    r.set(cache_key, vector.tobytes(), ex=604800)  # 7일
    return vector
```

---

## 13. 모니터링과 트러블슈팅

### 13.1 INFO 명령

가장 기본적인 모니터링 도구이다.

```bash
# 전체 정보
INFO

# 섹션별 조회
INFO memory
INFO stats
INFO replication
INFO clients
INFO keyspace
```

**핵심 모니터링 지표**:

| 지표 | 설명 | 위험 기준 |
|------|------|-----------|
| `used_memory` | 사용 중인 메모리 | maxmemory의 80% 이상 |
| `mem_fragmentation_ratio` | 메모리 단편화 비율 | 1.5 이상이면 정리 필요 |
| `connected_clients` | 연결된 클라이언트 수 | maxclients에 근접 |
| `blocked_clients` | 블로킹 명령 대기 중 | 지속적으로 높으면 문제 |
| `instantaneous_ops_per_sec` | 초당 처리 명령 수 | 기준선 대비 급변 |
| `keyspace_hits / misses` | 캐시 히트/미스 | 히트율 95% 이상 목표 |
| `rdb_last_bgsave_status` | 마지막 RDB 저장 상태 | "ok" 이외이면 즉시 확인 |
| `master_link_status` | 레플리카 연결 상태 | "down"이면 즉시 조치 |
| `used_memory_rss` | OS 기준 실제 메모리 | used_memory 대비 과다 |

### 13.2 Prometheus + Grafana 모니터링

```yaml
# docker-compose에 redis-exporter 추가
redis-exporter:
  image: oliver006/redis_exporter:latest
  environment:
    REDIS_ADDR: redis://redis:6379
    REDIS_PASSWORD: "your-password"
  ports:
    - "9121:9121"
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

### 13.3 주요 알림 규칙

```yaml
# Prometheus Alert Rules
groups:
  - name: redis
    rules:
      - alert: RedisHighMemoryUsage
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis 메모리 사용률 85% 초과"

      - alert: RedisHighLatency
        expr: redis_slowlog_length > 10
        for: 1m
        labels:
          severity: warning

      - alert: RedisReplicationBroken
        expr: redis_connected_slaves < 1
        for: 1m
        labels:
          severity: critical

      - alert: RedisHighFragmentation
        expr: redis_mem_fragmentation_ratio > 1.5
        for: 10m
        labels:
          severity: warning
```

### 13.4 트러블슈팅 체크리스트

**지연 시간 증가 시**:
1. `SLOWLOG GET 10`으로 느린 명령 확인
2. `CLIENT LIST`로 블로킹 클라이언트 확인
3. `INFO commandstats`로 명령별 통계 분석
4. `LATENCY LATEST` / `LATENCY HISTORY` 확인 (Redis 2.8.13+)
5. 빅키 존재 여부 확인: `redis-cli --bigkeys`

**메모리 급증 시**:
1. `INFO memory`에서 `used_memory`, `used_memory_rss`, `mem_fragmentation_ratio` 확인
2. `MEMORY DOCTOR`로 자동 진단
3. `MEMORY STATS`로 상세 메모리 분포 확인
4. `DEBUG OBJECT key`로 개별 키 인코딩 확인

**복제 문제 시**:
1. `INFO replication`에서 `master_link_status`, `master_last_io_seconds_ago` 확인
2. `repl-backlog-size`가 충분한지 확인
3. 네트워크 대역폭 및 지연 시간 점검

---

## 14. Redis vs 다른 솔루션 비교

### 14.1 Redis vs Memcached

| 기준 | Redis | Memcached |
|------|-------|-----------|
| 데이터 구조 | 다양 (String, Hash, List, Set...) | String only |
| 영속성 | RDB, AOF 지원 | 없음 |
| 복제 | 마스터-레플리카 | 없음 (클라이언트 분산) |
| 클러스터링 | Redis Cluster | 클라이언트 샤딩 |
| 메모리 효율 | 오버헤드 있음 | 단순 캐시에서는 더 효율적 |
| Pub/Sub | 지원 | 미지원 |
| 적합한 용도 | 다목적 (캐시, DB, 큐, ...) | 순수 캐시 |

### 14.2 Redis vs DynamoDB DAX

| 기준 | Redis | DAX |
|------|-------|----|
| 데이터 모델 | 다양한 구조 | DynamoDB 호환 |
| 관리 | 셀프 or 관리형 | 완전 관리형 |
| 지연 시간 | 마이크로초 | 마이크로초 |
| 적합한 용도 | 범용 | DynamoDB 전용 캐시 |

### 14.3 Redis vs Apache Kafka (메시징 관점)

| 기준 | Redis Streams | Kafka |
|------|---------------|-------|
| 처리량 | 중간 (수십만/초) | 매우 높음 (수백만/초) |
| 영속성 | 메모리 기반 + AOF | 디스크 기반 |
| 보존 기간 | 제한적 (메모리) | 장기 보존 가능 |
| Consumer Group | 지원 | 지원 |
| 적합한 용도 | 실시간 경량 스트리밍 | 대규모 이벤트 스트리밍 |

### 14.4 Redis vs Valkey

Valkey는 Redis 7.2.4에서 포크된 프로젝트로, BSD 라이선스를 유지한다. 기능적으로 호환되며 Linux Foundation이 관리한다. AWS ElastiCache, Google Cloud 등 주요 클라우드에서 채택하고 있다. 라이선스가 중요한 환경이라면 Valkey를 고려할 만하다.

---

## 부록: 자주 사용하는 CLI 명령 모음

```bash
# 서버 상태
redis-cli INFO
redis-cli PING
redis-cli DBSIZE

# 디버깅
redis-cli MONITOR              # 실시간 명령 모니터링 (프로덕션 주의!)
redis-cli CLIENT LIST           # 연결된 클라이언트 목록
redis-cli SLOWLOG GET 25        # 느린 쿼리 로그
redis-cli LATENCY LATEST        # 지연 이벤트

# 메모리
redis-cli MEMORY USAGE mykey
redis-cli MEMORY DOCTOR
redis-cli MEMORY STATS
redis-cli --bigkeys             # 빅키 스캔
redis-cli --memkeys             # 메모리 사용량 기준 키 스캔

# 운영
redis-cli BGSAVE               # RDB 스냅샷 수동 생성
redis-cli BGREWRITEAOF          # AOF 리라이트 수동 실행
redis-cli CONFIG GET maxmemory
redis-cli CONFIG SET maxmemory 8gb
redis-cli CONFIG REWRITE        # 변경 사항을 redis.conf에 반영

# 클러스터
redis-cli CLUSTER INFO
redis-cli CLUSTER NODES
redis-cli CLUSTER SLOTS
```

---

> **마지막 조언**: Redis는 단순해 보이지만, 프로덕션에서 안정적으로 운영하려면 메모리 관리, 영속성 설정, 복제/클러스터 구성, 보안, 모니터링 등 모든 측면을 꼼꼼히 다뤄야 한다. 캐시로만 쓰더라도 축출 정책과 메모리 한계를 반드시 설정하고, 데이터 저장소로 쓴다면 하이브리드 영속성 + 복제를 기본으로 구성하라. `KEYS *`는 절대 쓰지 말고, `MONITOR`는 디버깅 목적으로만 짧게 사용하라.
