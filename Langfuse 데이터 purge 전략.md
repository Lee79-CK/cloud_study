# Langfuse v3 데이터 Purge 전략: PostgreSQL · ClickHouse · S3 종합 가이드

## TL;DR
- **결론 — 직접 SQL/Lifecycle을 다루기 전에, 가능한 한 Langfuse의 내장 Data Retention 기능(Project Settings의 retention days, 최소 3일)을 1차 수단으로 써라.** 이 기능은 매일 밤 ClickHouse(`traces`/`observations`/`scores`/`events`)와 PostgreSQL `media` 테이블, S3 미디어 객체를 한 트랜잭션처럼 묶어 정합성을 유지하며 삭제한다. 단, Langfuse Pro/Enterprise/Self-Hosted EE에서만 사용 가능하다(Hobby/Core/OSS는 비활성).
- **OSS self-hosted라면 3-레벨 직접 운영이 필수다.** ① ClickHouse `traces/observations/scores/event_log`에 TTL 또는 K8s CronJob 기반 `DELETE` 실행, ② S3 `event/` prefix에 7–14일 Expiration lifecycle rule(미디어 버킷은 절대 금지), ③ PostgreSQL은 `audit_logs`, `pending_deletions(WHERE is_deleted=true)`, `job_executions`, `batch_exports`, NextAuth `Session` 같은 append-only 테이블만 날짜 기반으로 prune. `traces`/`observations`/`scores`는 v3에서 더 이상 PostgreSQL에 없다.
- **7–14일 보존은 충분히 공격적이지만 안전하게 가능하다.** ClickHouse TTL은 `event_ts(=수정 타임스탬프)`가 아닌 `timestamp`/`start_time` 기준으로 잡고, S3 event 버킷 lifecycle은 ClickHouse retention 보다 같거나 길게(권장: ClickHouse N일이면 S3 ≥ N일) 두어야 “지연된 update”가 새 row를 중복 생성하지 않는다. ClickHouse 25.7 이상이라면 공식 25.7 Release Call에서 발표된 `SET lightweight_delete_mode = 'lightweight_update'` (Developer: Anton Popov)를 활용해 mutation 부담을 크게 줄일 수 있고, 그 미만이면 partition-drop 또는 야간 배치 DELETE 전략을 우선 검토하라.

## Key Findings

### 1) v3 아키텍처: 무엇이 어디에 살고, 왜 그렇게 됐는가
Langfuse v3는 2024-12-09 GA(공식 changelog `langfuse.com/changelog/2024-12-09-Langfuse-v3-stable-release`: *“December 9, 2024 · Langfuse v3 is now stable and ready for production use when self-hosting Langfuse”*)로, v2 대비 다음과 같이 저장소가 분리된다(공식 self-hosting 문서 “Architecture Overview” 기준):

| 데이터 종류 | 저장소 | 비고 |
|---|---|---|
| traces, observations, scores | **ClickHouse** | `ReplacingMergeTree`, sort key (project_id, …, event_ts) |
| dataset_run_items (denormalized 사본) | ClickHouse + PostgreSQL | PostgreSQL의 `datasets/dataset_runs/dataset_items` 와 dual-write |
| `event_log`, 새로운 `events` 테이블(>=3.155) | ClickHouse | dual-write 마이그레이션 진행 중 |
| 원본 ingestion event JSON, multi-modal blob, batch export 파일 | **S3 / Blob Storage** | `{projectId}/{type}/{entityId}/{eventId}.json` 키 구조 |
| organizations, projects, users, api_keys, prompts, models, prices, datasets*, score_configs, eval_templates, job_configurations, job_executions, annotation_queues, memberships, audit_logs, batch_exports, **pending_deletions**, **media / trace_media / observation_media**, blob_storage_integrations, background_migrations, NextAuth Session | **PostgreSQL** | Prisma 관리, 389+ migrations |
| BullMQ 큐, API key/prompt 캐시 | Redis/Valkey | purge 대상 아님 |

이 분리의 핵심 함의는 **“traces/observations/scores를 지우려면 PostgreSQL이 아니라 ClickHouse를 건드려야 한다”**는 점이다. v3 migrate 이후에도 PostgreSQL에는 v2 잔재인 `LegacyPrismaTrace/Observation/Score` 모델이 남아 있지만, 정상 운영에서는 사용되지 않는다.

S3 키 구조(DeepWiki “System Architecture”):
```
s3://<bucket>/<LANGFUSE_S3_EVENT_UPLOAD_PREFIX>/
  └── {projectId}/
        └── {trace|observation|score}/
              └── {entityId}/
                    └── {eventId}.json
```
미디어와 batch export는 각각 별도 prefix(`LANGFUSE_S3_MEDIA_UPLOAD_PREFIX`, `LANGFUSE_S3_BATCH_EXPORT_PREFIX`)로 분리할 수 있고, 보통 같은 버킷 안에서 prefix만 달리한다.

ClickHouse가 ingestion에서 “해당 객체의 이전 row를 읽어 머지” 하므로 S3 event blob 은 **단순한 raw 백업이 아니라 ingestion path의 일부**다. 따라서 S3 retention은 “지연된 update를 어디까지 받을지”와 직결된다.

### 2) Langfuse 내장 Data Retention(권장 1차 솔루션)
공식 문서 `docs/administration/data-retention`:
> *“On a nightly basis, Langfuse selects traces, observations, scores, and media assets that are older than the configured retention period and deletes them.”*

- 적용 단위: **프로젝트별**(전역 설정 불가; 이슈 #9590에서 feature request 진행 중)
- 기준 필드: traces=`timestamp`, observations=`start_time`, scores=`timestamp`, media assets=`created_at`
- 최소값: **3일**
- 대상 저장소: ClickHouse + PostgreSQL `media` + S3 미디어/이벤트
- 가용 플랜: **Pro, Enterprise, Self-Hosted EE 전용** (OSS Hobby/Core는 UI 없음 — 그래서 OSS 사용자는 아래 직접 purge가 필요)
- self-hosted 요구사항: IAM 정책에 `s3:DeleteObject` 포함, `LANGFUSE_CLICKHOUSE_DELETION_TIMEOUT_MS`(기본 600,000ms=10분) 조정 가능

EE 환경이라면 **이 한 가지만 켜면 끝**이다. 아래의 모든 수동 작업은 OSS를 가정한다.

### 3) ClickHouse Purge 전략 (가장 중요한 부분)

**스키마와 엔진 특성**
- `traces`, `observations`, `scores`, `event_log`, `dataset_run_items` 모두 `ReplacingMergeTree`(클러스터 모드에서는 `ReplicatedReplacingMergeTree`) 엔진을 사용한다. version 컬럼은 `event_ts`(write timestamp), sort key는 `(project_id, ..., event_ts)`이며, 모든 read 쿼리는 `FINAL` 또는 `ORDER BY event_ts LIMIT 1 BY id, project_id` 로 read-time deduplication을 한다.
- Langfuse Engineering Handbook(`langfuse.com/handbook/product-engineering/infrastructure/clickhouse`, “ClickHouse Cloud” 페이지) 인용: *“All tracing tables use the ReplacingMergeTree engine. … The deduplication in ReplacingMergeTrees happens eventually and there is no guarantee for it to ever happen.”*
- 파티셔닝: 정식 production migration의 `traces`/`observations`/`scores` 테이블은 컬럼 단위 PARTITION BY 없이(또는 매우 거친 단위로) 정의되어 있다 — 즉 `DROP PARTITION` 으로 깔끔한 일자별 삭제는 어렵고, 보조적 staging 테이블(`observations_batch_staging`)만 3분 단위 파티션이다. 따라서 직접 일자 단위 파티션을 추가하려면 마이그레이션을 작성해야 한다.

**Option A — 자동 TTL (가장 깔끔)**

공식 docs `self-hosting/configuration/scaling` 가 명시적으로 허용하는 방법:
> *“To automatically remove data within ClickHouse, you can use the TTL feature … This is applicable for the `traces`, `observations`, `scores`, and `event_log` table within ClickHouse.”*

14일 보존 예시:
```sql
-- 기준 컬럼: traces=timestamp, observations=start_time, scores=timestamp
ALTER TABLE traces       ON CLUSTER default MODIFY TTL toDate(timestamp)  + INTERVAL 14 DAY DELETE;
ALTER TABLE observations ON CLUSTER default MODIFY TTL toDate(start_time) + INTERVAL 14 DAY DELETE;
ALTER TABLE scores       ON CLUSTER default MODIFY TTL toDate(timestamp)  + INTERVAL 14 DAY DELETE;
ALTER TABLE event_log    ON CLUSTER default MODIFY TTL toDate(created_at) + INTERVAL 14 DAY DELETE;
```
주의사항(Altinity KB 인용):
1. `ALTER TABLE ... MODIFY TTL` 는 메타데이터만 바꾸지만, 기본값으로 기존 part 전체를 다시 읽고 TTL 을 재계산한다. 큰 테이블에서 OOM 가능 — 사전에 `SET materialize_ttl_after_modify = 0;` 로 비활성화하거나 `SET max_table_size_to_drop = 0;` 후 `TRUNCATE` 가능한 경우만 실행.
2. TTL은 머지 시점에 동작한다. 공식 ClickHouse 가이드(`clickhouse.com/docs/guides/developer/ttl`) 인용: *“Default value: 14400 seconds (4 hours). So by default, your TTL rules will be applied to your table at least once every 4 hours.”* 즉시 효과를 보려면 `merge_with_ttl_timeout`을 줄이거나, 작은 테이블에서만 `OPTIMIZE TABLE … FINAL` 로 강제. **production traces 테이블에 OPTIMIZE FINAL은 금지** — 모든 part를 재작성해 디스크와 IO를 폭발시킨다.
3. `ttl_only_drop_parts = 1` 설정을 고려하라(파티션이 있을 때만 의미). 부분 row만 TTL로 지우면 머지가 매우 비싸진다.

**중요한 정합성 제약 — 공식 문서(`self-hosting/deployment/infrastructure/blobstorage`)**:
> *“Langfuse uses raw event data from the bucket to merge delta-updates into existing traces, observations, and scores. The bucket lifecycle expiration period determines how long these updates can be merged into existing records. Once the raw events are deleted by the lifecycle policy, delta-updates will create duplicate entries instead of merging.”*

즉 **S3 event 버킷의 TTL과 ClickHouse TTL은 서로 맞춰야 한다.** 권장 패턴은 `S3 event TTL = ClickHouse TTL` 또는 `S3 ≥ ClickHouse`(S3가 ingestion source-of-truth이기 때문). 7–14일 보존 정책이라면 양쪽 모두 같은 값을 잡는 것이 가장 안전하다.

**Option B — 명시적 DELETE(mutation) by cron**

OSS이며 TTL을 운영팀이 직접 통제하고 싶을 때 가장 흔한 방식. kyma-project의 실제 운영 예시(issue `kyma-project/kyma-companion#1076`)를 그대로 차용 가능하다:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: langfuse-clickhouse-retention
data:
  retention.sh: |
    #!/bin/sh
    set -e
    HOST="http://langfuse-clickhouse:8123"
    DAYS=${RETENTION_DAYS:-14}
    for T in \
      "DELETE FROM traces       WHERE timestamp  < now() - interval ${DAYS} day" \
      "DELETE FROM observations WHERE start_time < now() - interval ${DAYS} day" \
      "DELETE FROM scores       WHERE timestamp  < now() - interval ${DAYS} day" \
      "DELETE FROM event_log    WHERE created_at < now() - interval ${DAYS} day"
    do
      curl -sf -u "${CLICKHOUSE_USER}:${CLICKHOUSE_PASSWORD}" "${HOST}/" --data "$T"
    done
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: langfuse-ch-retention
spec:
  schedule: "0 3 * * *"     # 매일 새벽 3시 (KST)
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 1
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: curlimages/curl:8.10.1
            command: ["/scripts/retention.sh"]
            env:
            - name: RETENTION_DAYS
              value: "14"
            - name: CLICKHOUSE_USER
              value: default
            - name: CLICKHOUSE_PASSWORD
              valueFrom: { secretKeyRef: { name: langfuse-clickhouse, key: admin-password } }
            volumeMounts:
            - { name: scripts, mountPath: /scripts }
          volumes:
          - name: scripts
            configMap: { name: langfuse-clickhouse-retention, defaultMode: 0755 }
```

DELETE 동작은 두 가지 모드가 있다:
- **Default (`alter_update`)**: 내부적으로 `ALTER TABLE … DELETE WHERE …` mutation. 영향받는 모든 part를 재작성한다. 대량 데이터에서 매우 비싸다.
- **`lightweight_update` (ClickHouse 25.7+)**: hidden `_row_exists` 마스킹. 즉시 끝나며 background merge 때 실제 제거. 공식 25.7 Release Call(`presentations.clickhouse.com/2025-release-25.7/`) verbatim: *“SET lightweight_delete_mode = 'lightweight_update'; Developer: Anton Popov.”* Langfuse는 같은 기능을 다음 환경변수로 노출한다.
  - `CLICKHOUSE_LIGHTWEIGHT_DELETE_MODE=lightweight_update` (기본 `alter_update`)
  - `CLICKHOUSE_USE_LIGHTWEIGHT_UPDATE=true` (trace upsert에도 적용)
  - 7-14일 daily delete 같은 “자주, 가볍게” 시나리오에 강력 추천.

mutation 모니터링:
```sql
SELECT database, table, command, create_time, is_done, latest_fail_reason
FROM system.mutations
WHERE NOT is_done
ORDER BY create_time DESC;
```

**Option C — Partition drop**
Langfuse stock 스키마는 production migration에 PARTITION BY 가 없으므로 단순 적용 불가. 자체 fork 운영이 가능한 팀만, 마이그레이션을 추가해 `PARTITION BY toYYYYMMDD(timestamp)` 등으로 바꾸고 `ALTER TABLE traces DROP PARTITION 'YYYYMMDD'` 를 cron 으로 돌리는 것이 이론상 가장 빠르고 부담 없는 방법이다. 7–14일 보존이라면 일자 단위 파티션이 적합. 단, Langfuse 기본 마이그레이션과 충돌 가능성이 높아 권장하지 않는다.

**시스템 테이블도 잊지 말 것 — 가장 흔한 디스크 폭증 원인**

공식 docs는 system log table들이 디스크를 잠식하는 사례를 명시적으로 언급한다(`scaling#clickhouse-system-log-tables`, GitHub issue `langfuse/langfuse-k8s#333`). Helm 또는 config.d 로 다음을 적용:

```xml
<clickhouse>
  <trace_log remove="1"/>
  <text_log remove="1"/>
  <opentelemetry_span_log remove="1"/>
  <asynchronous_metric_log remove="1"/>
  <metric_log remove="1"/>
  <latency_log remove="1"/>
</clickhouse>
```
또는 짧은 TTL만 부여:
```sql
SET max_table_size_to_drop = 0;
TRUNCATE TABLE system.trace_log;
ALTER TABLE system.trace_log MODIFY TTL event_date + INTERVAL 7 DAY;
```
실제 사용자 사례(Discussion `langfuse/discussions/5687`, user Pratish315) verbatim: *“Its 2 days later and I still only have a single trace and my clickhouse EFS storage has grown from 400MB to 1.2GB.”* EKS + RDS + ElastiCache + EFS + Zookeeper 구성에서 trace 거의 없음에도 ClickHouse PV가 부풀어 오른 사례 — 원인은 Zookeeper/system log였다.

**최대 테이블 진단(공식 권장 쿼리)**:
```sql
SELECT table, formatReadableSize(size) AS size, rows
FROM (
  SELECT table, database, sum(bytes) AS size, sum(rows) AS rows
  FROM system.parts WHERE active GROUP BY table, database
  ORDER BY size DESC
);
```

`blob_storage_file_log` 테이블 동기화: S3 lifecycle을 켰다면 ClickHouse 메타데이터도 같이 줄여야 한다(`self-hosting/deployment/infrastructure/blobstorage`):
```sql
ALTER TABLE blob_storage_file_log MODIFY TTL created_at + INTERVAL 14 DAY DELETE;
```
또는 `LANGFUSE_ENABLE_BLOB_STORAGE_FILE_LOG=false` 로 추적 자체를 끈다(S3 lifecycle만 단독으로 쓸 때).

권한 grants (공식):
```sql
GRANT INSERT, SELECT, ALTER UPDATE, ALTER DELETE, ALTER DROP INDEX, CREATE, DROP TABLE
  ON default.* TO 'langfuse';
```

### 4) S3 / Object Storage Purge 전략

**환경 변수 3종(`.env.prod.example` 기준)**:
- `LANGFUSE_S3_EVENT_UPLOAD_BUCKET` / `_PREFIX` (기본 `events/`) — **필수**, 모든 ingestion raw event JSON
- `LANGFUSE_S3_MEDIA_UPLOAD_BUCKET` / `_PREFIX` (기본 `media/`) — optional, multimodal blob
- `LANGFUSE_S3_BATCH_EXPORT_BUCKET` / `_PREFIX` (기본 `exports/`) — optional, CSV/JSONL exports

**핵심 운영 규칙(공식 문서 verbatim)**:
- ClickHouse가 backing store로 S3를 쓰는 disk-as-S3 버킷에 대해서는 *“Do not enable bucket versioning … Do not enable lifecycle policies for deletion: Avoid deletion lifecycle policies as this may break ClickHouse's internal consistency model.”* — **ClickHouse data 디스크용 S3 버킷에는 lifecycle을 절대 걸지 마라**(Langfuse 이벤트 버킷과 다른 버킷이다).
- 미디어 버킷에도 lifecycle 금지(공식): *“Setting a retention policy on this bucket is not recommended because: (1) Referenced media files in traces would break, (2) Future uploads of the same file would fail since file upload status is tracked by hash in Postgres.”*
- 따라서 **lifecycle 대상은 오직 `events/` prefix와 (optionally) `exports/` prefix.**

**AWS S3 Lifecycle — 14일 expiration(JSON)**:
```json
{
  "Rules": [
    {
      "ID": "langfuse-events-14d",
      "Status": "Enabled",
      "Filter": { "Prefix": "events/" },
      "Expiration": { "Days": 14 },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 1 }
    },
    {
      "ID": "langfuse-exports-14d",
      "Status": "Enabled",
      "Filter": { "Prefix": "exports/" },
      "Expiration": { "Days": 14 }
    }
  ]
}
```
적용:
```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket langfuse-data \
  --lifecycle-configuration file://lifecycle.json
aws s3api get-bucket-lifecycle-configuration --bucket langfuse-data
```
versioning이 켜져 있다면 `NoncurrentVersionExpiration: { NoncurrentDays: 1 }`을 추가해 delete marker / non-current version도 정리한다.

**MinIO (자체 호스팅)**:
```bash
mc alias set lf https://minio.internal:9000 ACCESS SECRET
mc ilm rule add lf/langfuse --prefix "events/" --expire-days 14
mc ilm rule add lf/langfuse --prefix "exports/" --expire-days 14
mc ilm rule ls lf/langfuse
```
또는 JSON import:
```bash
mc ilm import lf/langfuse <<EOF
{ "Rules": [
  { "ID": "events14",  "Status": "Enabled", "Filter": { "Prefix": "events/" },  "Expiration": { "Days": 14 } },
  { "ID": "exports14", "Status": "Enabled", "Filter": { "Prefix": "exports/" }, "Expiration": { "Days": 14 } }
] }
EOF
```

**ClickHouse–S3 정합성**: 위 ClickHouse 섹션의 인용을 재강조 — S3 event TTL은 ClickHouse 보다 **짧으면 안 된다**. 7일 보존을 원하면 양쪽 7일, 14일이면 양쪽 14일. SDK 기본 동작이 즉시 전송이라면 보통 영향 없지만 OpenTelemetry 배치/지연 ingestion 환경에서는 S3 측을 1-2일 더 길게 두는 것이 안전.

**수동 일회성 삭제**:
```bash
aws s3 rm s3://langfuse-data/events/ --recursive \
  --exclude "*" --include "*$(date -d '15 days ago' +%Y-%m-%d)*"
```
대량 삭제는 S3 Batch Operations 또는 `aws s3api list-objects-v2 --query 'Contents[?LastModified<\`2025-04-29\`].Key'` 로 매니페스트 만들어 일괄 처리.

### 5) PostgreSQL Purge 전략 — 무엇을 건드리고 무엇을 건드리면 안 되는지

`packages/shared/prisma/schema.prisma` (~389 Prisma migrations, Prisma 6.19.3 기준) 분석을 토대로:

**❌ 절대 직접 DELETE 금지(설정/관계 데이터)**: `organizations`, `projects`, `users`, `organization_memberships`, `project_memberships`, `membership_invitations`, `api_keys`, `llm_api_keys`, `sso_configs`, `prompts`, `prompt_dependencies`, `prompt_protected_labels`, `models`, `prices`, `score_configs`, `eval_templates`, `job_configurations`, `datasets`, `dataset_items`, `dataset_runs`, `dataset_run_items`, `annotation_queues`, `annotation_queue_items`, `posthog_integrations`, `blob_storage_integrations`, `slack_integrations`, `comments`, `dashboards`, `dashboard_widgets`, `table_view_presets`, `actions`, `triggers`, `automations`, `default_llm_models`, `llm_schemas`, `llm_tools`, `trace_sessions`, `background_migrations`, `_prisma_migrations`, `Account`/`Session`/`VerificationToken`(NextAuth).

특히 datasets류는 공식적으로 *“datasets are not affected by data retention and are kept indefinitely”* — retention 정책의 사정거리 밖이다.

**⚠️ 절대 수동 purge 금지(미디어 join 테이블)**: `media`, `trace_media`, `observation_media` — 이 세 테이블은 S3 미디어 객체와 1:1 매핑이며 PostgreSQL에 SHA256 해시로 dedup 정보가 저장돼 있다. 수동 삭제 시 ClickHouse trace에서 깨진 참조 발생 + 향후 같은 파일 업로드 실패. **Langfuse Data Retention(EE) 또는 “해당 trace 자체를 delete API로 지움” 으로만 정리.**

**✅ 안전하게 날짜 기반 prune 가능(append-only, FK 없음)**:

1. **`audit_logs`** — 가장 흔한 비대화 원인. 스키마 verbatim 주석: `// No FK constraints to preserve audit logs`. `audit_logs.projectId` / `orgId`는 단순 String 컬럼이라 cascade도 없다. OSS에서는 UI에서 못 보고 EE에서만 조회 가능. 실제 사례(`github.com/orgs/langfuse/discussions/10446`) verbatim: *“I'm using the open source version of langfuse (currently v3.130.0), and I'm seeing the audit_logs table in postgres growing to more than 6GB.”*
```sql
DELETE FROM audit_logs WHERE created_at < now() - INTERVAL '14 days';
```

2. **`pending_deletions`** — v3에서 trace 비동기 삭제용 ledger 테이블. 스키마 verbatim:
```prisma
model PendingDeletion {
  id String @id @default(cuid())
  projectId String
  project   Project @relation(... onDelete: Cascade)
  object    String   // "trace" 등
  objectId  String
  isDeleted Boolean @default(false)
  createdAt DateTime @default(now())
  ...
}
```
Discussion `langfuse/discussions/10049` verbatim: *“the deletion flow does not clean it up. You can check how many rows are in this table, and clean it up if required.”* 정기 청소 권장:
```sql
DELETE FROM pending_deletions
WHERE is_deleted = true AND updated_at < now() - INTERVAL '7 days';
```

3. **`job_executions`** — eval(LLM-as-judge) 실행 이력. `jobInputTraceId` 등은 ClickHouse trace를 가리키는 단순 String. 안전하게 prune:
```sql
DELETE FROM job_executions WHERE created_at < now() - INTERVAL '14 days';
```

4. **`batch_exports`** — 비동기 export job 메타. 14일 이상은 prune 가능.

5. **NextAuth `Session`** — `expires < now()` 인 row 삭제 가능.

**CASCADE 체인** — `Project` row 삭제 시 자동 cascade되는 테이블: `api_keys`, `llm_api_keys`, `prompts`, `score_configs`, `eval_templates`, `job_configurations`, `job_executions`(via JobConfiguration), `annotation_queues→items`, `datasets→items→run_items`, `dataset_runs`, `comments`, integration 테이블, `trace_sessions`, `media`(→`trace_media`/`observation_media` 재cascade), `dashboards`, `pending_deletions`, `prices` 등. **`audit_logs`는 cascade 대상이 아님** — 의도된 보존 정책.

`Organization → OrganizationMembership → ProjectMembership` 도 모두 `onDelete: Cascade`. `User` 삭제 역시 같은 체인을 타고 내려간다(공식 `data-deletion` 문서: *“Remove the corresponding user record from the users table and drop all foreign keys to it using cascade.”*).

**유지보수(VACUUM/REINDEX)** — 대량 DELETE 후 dead tuple 정리 필수:
```sql
VACUUM (ANALYZE, VERBOSE) audit_logs;
-- 디스크 공간을 즉시 OS에 반환해야 한다면(잠금 주의):
VACUUM FULL audit_logs;
REINDEX TABLE CONCURRENTLY audit_logs;
```
정기 운영이라면 `autovacuum_vacuum_scale_factor`를 audit_logs 같은 테이블에 한해 낮추는 것을 고려:
```sql
ALTER TABLE audit_logs SET (autovacuum_vacuum_scale_factor = 0.05);
```

### 6) Kubernetes 통합 운영 패턴

운영 순서(데이터 정합성 우선):
1. **백업 먼저** — PostgreSQL pg_dump (managed PG라면 자동 snapshot), ClickHouse `BACKUP TABLE … TO Disk(...)` 혹은 PV snapshot, S3 cross-region replication. Langfuse 공식 `self-hosting/configuration/backups` 가이드의 CronJob 예시 사용.
2. **Langfuse 내장 Data Retention 활성화(EE)** — 모든 프로젝트에 nightly 자동화.
3. **OSS라면 K8s CronJob 3종**:
   - `langfuse-ch-retention` (위 Option B의 ClickHouse DELETE, daily 03:00)
   - `langfuse-pg-retention` (audit_logs / pending_deletions / job_executions / batch_exports prune + VACUUM, daily 03:30)
   - S3는 lifecycle policy이므로 별도 cron 불필요
4. **모니터링** — Prometheus + statsd:
   - `langfuse.queue.ingestion.length` (worker가 throughput을 못 따라가는지)
   - `system.mutations WHERE NOT is_done` (ClickHouse mutation backlog)
   - PG `pg_stat_user_tables` 의 `n_dead_tup`, `last_autovacuum`
   - S3 `BucketSizeBytes`, `NumberOfObjects` CloudWatch metric (prefix별로 별도 metrics filter 필요)
5. **알림** — DELETE timeout(`LANGFUSE_CLICKHOUSE_DELETION_TIMEOUT_MS` 초과), mutation queue 100+, pending_deletions 행수 폭증.

### 7) GDPR / 컴플라이언스

- **트레이스 단위 “erasure right”**: Langfuse는 UI에서 single/batch/query-by-filter delete 지원(`docs/administration/data-deletion`). 내부적으로 `pending_deletions` 큐에 넣고 ClickHouse + S3 정합성 있게 삭제. 회원 탈퇴 시 “Delete user account” 도 self-host에서 가능.
- 단, 공식 docs는 명시한다: *“The retention policy applies to the respective data, independent of any references. For example, if a dataset references a trace but the trace is e.g. deleted after 30 days, the dataset run item will point to a non-existent trace.”* — 7-14일 보존을 잡았다면 dataset run의 trace 링크가 끊긴다는 점을 evaluation 팀과 합의해야 한다.
- 미디어 객체는 PostgreSQL `media` 테이블 + S3 객체 + ClickHouse `trace_media` 참조의 3중 정합성을 유지해야 하므로 **반드시 Langfuse API/Retention으로만 삭제**.

## Details

### 직접 쿼리/명령 모음

ClickHouse 디스크 분포 진단(공식 권장):
```sql
SELECT table, formatReadableSize(sum(bytes)) AS size, sum(rows) AS rows
FROM system.parts WHERE active GROUP BY table ORDER BY sum(bytes) DESC LIMIT 20;
```

ClickHouse mutation queue 모니터링:
```sql
SELECT database, table, mutation_id, command,
       create_time, latest_fail_reason
FROM system.mutations WHERE NOT is_done ORDER BY create_time;
```

ClickHouse TTL 즉시 강제(소규모 테이블에서만):
```sql
ALTER TABLE event_log MATERIALIZE TTL;
```

PostgreSQL 비대 테이블 진단:
```sql
SELECT schemaname, relname,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC LIMIT 20;
```

PostgreSQL 안전한 일자 기반 chunked purge(긴 잠금 회피):
```sql
-- 14일 기준
DO $$ BEGIN
  LOOP
    DELETE FROM audit_logs
    WHERE id IN (
      SELECT id FROM audit_logs
      WHERE created_at < now() - INTERVAL '14 days'
      LIMIT 10000
    );
    EXIT WHEN NOT FOUND;
    COMMIT;
  END LOOP;
END $$;
```

AWS CLI 일회성 cleanup(예: 14일 이전):
```bash
aws s3api list-objects-v2 --bucket langfuse-data --prefix events/ \
  --query "Contents[?LastModified<='$(date -u -d '14 days ago' +%Y-%m-%dT%H:%M:%SZ)'].Key" \
  --output text | tr '\t' '\n' \
  | xargs -I{} -P10 aws s3 rm "s3://langfuse-data/{}"
```

### 환경변수 요약(purge 관련)
| 변수 | 기본값 | 용도 |
|---|---|---|
| `LANGFUSE_CLICKHOUSE_DELETION_TIMEOUT_MS` | 600000 | retention/delete HTTP timeout |
| `CLICKHOUSE_LIGHTWEIGHT_DELETE_MODE` | `alter_update` | `lightweight_update` 추천 (CH 25.7+) |
| `CLICKHOUSE_USE_LIGHTWEIGHT_UPDATE` | false | trace upsert에도 lightweight |
| `LANGFUSE_ENABLE_BLOB_STORAGE_FILE_LOG` | true | S3 lifecycle 단독 운영 시 false |
| `LANGFUSE_S3_*_PREFIX` | 위 표 참조 | bucket 내 prefix |
| `LANGFUSE_INIT_PROJECT_RETENTION` | (unset) | headless init 시만 적용, UI 생성엔 미반영 |

## Recommendations

다음 순서로 단계적으로 진행할 것을 권장한다.

**Phase 0 (즉시, 1일 이내)**
1. ClickHouse `system.*_log` 비활성화 or 7일 TTL 적용. 가장 흔한 “이유 없는 디스크 폭증” 원인을 제거.
2. `LANGFUSE_ENABLE_BLOB_STORAGE_FILE_LOG=true` 그대로 두고 `blob_storage_file_log` 에 TTL 14일 매칭 설정.
3. `audit_logs`, `pending_deletions(is_deleted=true)`, `job_executions` 최초 1회 cleanup + VACUUM.

**Phase 1 (1주일 이내)**
4. **EE 라이선스 보유** → 모든 프로젝트 retention=14일로 설정하고 끝.
5. **OSS** → 위 K8s CronJob 2종(ClickHouse + Postgres) 배포 + S3 lifecycle 14일. ClickHouse 25.7+ 면 `CLICKHOUSE_LIGHTWEIGHT_DELETE_MODE=lightweight_update` 활성화.

**Phase 2 (정착, 2주~)**
6. 모니터링/알림 임계치 설정: mutation backlog>100, deletion timeout 발생, S3 event prefix 객체 수 일 단위 추세 그래프.
7. 보존 기간 튜닝: 7일 vs 14일은 “dataset run trace 링크가 끊겨도 되는가” + “디버깅 윈도우” 결정에 달림. 권장 시작값은 14일.

**보존 정책 변경(threshold)이 필요한 신호**
- 사용자가 “2주 전 trace를 보여달라”는 요구가 분기마다 1회 이상 발생 → 30일로 확장
- ClickHouse 디스크 사용량이 14일 보존에도 90% 도달 → input/output 크기 mask·truncation(server-side ingestion masking, `LANGFUSE_INGESTION_MASKING_CALLBACK_URL`) 검토
- pending_deletions 행수가 1000 이상 정체 → BullMQ worker 스케일업, `LANGFUSE_TRACE_DELETE_DELAY_MS` 조정

## Caveats

1. **공식적으로 자동 retention 기능은 OSS에서 빠져 있다.** Discussion `langfuse/discussions/11508` verbatim: *“the Data Retention feature is not available in the OSS (open source) version … only available in the Pro, Enterprise, and Self-Hosted Enterprise Edition plans.”* OSS 운영자가 위 가이드를 따르는 것은 사실상 “직접 만든 retention” 임을 인식해야 한다.
2. **`OPTIMIZE TABLE … FINAL` 은 production traces 테이블에 절대 정기 실행하지 마라.** Altinity KB(`kb.altinity.com/altinity-kb-queries-and-syntax/altinity-kb-optimize-vs-optimize-final/`) verbatim: *“It creates a huge CPU / Disk load if the table (XYZ) is huge. ClickHouse reads / uncompress / merge / compress / writes all data in the table. If this table has size 1TB it could take around 3 hours to complete. So we don't recommend running OPTIMIZE TABLE xyz FINAL against tables with more than 10 million rows.”* Langfuse 공식도 read 시 deduplication을 코드 단에서 처리한다고 명시하므로 강제 머지가 불필요.
3. **PARTITION DROP 기반 전략은 stock Langfuse 스키마에서는 즉시 사용 불가.** stock migrations의 `traces`/`observations`/`scores` 에는 일자 단위 파티션이 없다. 임의 마이그레이션 추가는 향후 공식 마이그레이션과 충돌 위험 — 비권장.
4. **S3 event 버킷 lifecycle을 ClickHouse retention보다 짧게 잡으면 update가 중복 row를 만든다.** 공식 verbatim 재인용: *“Once the raw events are deleted by the lifecycle policy, delta-updates will create duplicate entries instead of merging.”* SDK 기본 동작이라면 큰 문제는 없으나 OpenTelemetry batched export 환경에서는 critical.
5. **미디어/S3 media 버킷에 lifecycle 적용 금지.** 깨진 trace 참조 + 같은 hash 재업로드 실패. 미디어 정리는 무조건 Langfuse Retention(EE) 또는 trace delete를 경유.
6. **`OPTIMIZE TABLE … FINAL CLEANUP` 및 `ReplacingMergeTree(version, is_deleted)` 의 깊은 사용은 Langfuse 표준 외**다. Altinity KB는 23.2+에서 가능하다고 하지만, Langfuse는 `event_ts` 하나의 version만 사용한다.
7. **`LANGFUSE_INIT_PROJECT_RETENTION` 은 headless init에서만 적용**된다(Discussion `langfuse/discussions/11123`). UI에서 생성되는 신규 프로젝트에는 기본 retention이 자동 부여되지 않는다 — 새 프로젝트마다 수동 설정 또는 Projects API 사용 필요. 조직 단위 전역 기본값은 feature request 상태(`langfuse/discussions/9590`).
8. **ClickHouse 단일 노드 vs 클러스터**: `CLICKHOUSE_CLUSTER_ENABLED=true` 인 경우 위 DDL은 `ON CLUSTER <name>` 가 필요. 단일 인스턴스에서 `ON CLUSTER`를 쓰면 에러.
9. **백업 권장**: 모든 retention 변경(특히 TTL 최초 적용) 전에 ClickHouse `BACKUP`, PG `pg_dump`, S3 cross-region replication 스냅샷을 확보. 한 번 지워진 데이터는 복구 불가(공식 docs verbatim: *“Deleted assets cannot be recovered.”*).
10. **`schema.prisma` 직접 추적 한계**: 본 보고서의 PostgreSQL 테이블/CASCADE 분석은 v3.170+ 시점의 schema 단편을 기반으로 합성한 것이다. 운영 자동화에 들어가기 전에 자신의 deployed 버전의 `packages/shared/prisma/schema.prisma` 와 `prisma/migrations/*` 를 직접 확인해, `audit_logs` 등의 FK가 의도 그대로 유지되어 있는지(특히 `batch_exports` 의 project cascade 여부 — 마이그레이션에 따라 다름) 검증할 것.