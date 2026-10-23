# EDB Postgres® AI Database Essentials v18 — Practice Exam (90 Questions)

> **학습용 모의문제:** 이 자료는 사용자 제공 비공식 연습문제 60문항과 추가 30문항으로 구성한 자체 학습 자료입니다. EDB가 발행한 공식 기출문제나 공식 정답표가 아닙니다.
> **구성:** 총 90문항 · 객관식 4지선다 · 영어 문제/영어 해설 + 한국어 핵심 해설 · 영역별 정답표
> **시험 규칙:** 실제 시험의 문항 수·시간·합격선은 v18 공식 시험 안내에서 확인되지 않아 특정 수치로 단정하지 않습니다. [EDB 공식 시험 카탈로그](https://www.enterprisedb.com/training/certification-exams)를 확인하세요.

## 학습 안내

- 처음에는 **Answer / Explanation** 줄을 가리고 실제 시험처럼 영어 질문과 보기만 읽어 보세요.
- 정답 확인 시 영어 해설과 **한국어 핵심 해설**을 함께 읽고, 키워드를 PostgreSQL 18 또는 EPAS 18 공식 문서에서 확인하세요.
- EPAS Oracle 호환 기능과 커뮤니티 PostgreSQL의 공통 기능은 구별해야 합니다. 추가 EPAS 문제는 Redwood/EPAS 문서에 근거합니다.
- 선택지의 정답 위치는 A/B/C/D로 균등하게 분산했습니다.

## 목차

- Domain 1 — System Architecture & Installation (Q1–Q8)
- Domain 2 — Configuration & Cluster Management (Q9–Q17)
- Domain 3 — Roles, Security & Authentication (Q18–Q26)
- Domain 4 — Database Objects, SQL & EPAS (Q27–Q36)
- Domain 5 — Maintenance, MVCC & Performance (Q37–Q45)
- Domain 6 — Backup, Recovery & Replication (Q46–Q53)
- Domain 7 — PostgreSQL 18 Core Features (Q54–Q60)
- Domain 8 — EPAS Oracle Compatibility (Additional) (Q61–Q75)
- Domain 9 — PostgreSQL 18 Extended Features (Additional) (Q76–Q85)
- Domain 10 — Operational Diagnostics (Additional) (Q86–Q90)

---

## Domain 1 — System Architecture & Installation (Q1–Q8)

### Q1. Which background process in EDB Postgres is responsible for writing modified data pages (dirty buffers) from shared memory to disk asynchronously so that backend processes rarely have to wait for disk writes?

- **A)** `bgwriter`
- **B)** `walwriter`
- **C)** `checkpointer`
- **D)** `autovacuum launcher`

**Answer: A**

**Explanation (EN):** The background writer (`bgwriter`) periodically writes small batches of dirty shared buffers to disk so that server processes handling user queries find clean buffers available without waiting. The `checkpointer` writes all dirty buffers only at checkpoint intervals.

**한국어 핵심 해설:** 백그라운드 라이터는 더티 버퍼를 미리 디스크에 써서 사용자 백엔드의 쓰기 대기를 줄인다. 체크포인터는 체크포인트 과정의 더티 페이지 기록을 관리한다.

---

### Q2. When initializing a new database cluster using `initdb`, which template database is cloned by default whenever a user executes a standard `CREATE DATABASE` command?

- **A)** `template0`
- **B)** `template1`
- **C)** `postgres`
- **D)** `edb`

**Answer: B**

**Explanation (EN):** `CREATE DATABASE` works by copying an existing database. By default, it copies the standard system database named `template1`. `template0` is kept pristine and should be used when you need a clean database without any site-local additions made to `template1`.

**한국어 핵심 해설:** 기본 CREATE DATABASE는 template1을 복제한다. template0은 기본 템플릿의 사용자 변경 내용이 없는 기준 데이터베이스다.

---

### Q3. In an EDB Postgres cluster directory (`$PGDATA`), which subdirectory stores the transaction commit status data (Commit Log)?

- **A)** `pg_wal`
- **B)** `pg_multixact`
- **C)** `pg_xact`
- **D)** `base`

**Answer: C**

**Explanation (EN):** `pg_xact` (formerly `pg_clog` prior to PostgreSQL 10) stores transaction commit status data. `pg_wal` stores Write-Ahead Log files, and `base` contains per-database subdirectories.

**한국어 핵심 해설:** 트랜잭션 커밋 상태는 pg_xact에 저장한다. WAL 기록은 pg_wal이며 서로 목적이 다르다.

---

### Q4. Where are the physical data files for a table stored by default if no specific tablespace is assigned?

- **A)** `$PGDATA/global/`
- **B)** `$PGDATA/pg_tblspc/`
- **C)** `$PGDATA/pg_stat/`
- **D)** `$PGDATA/base/<database_oid>/`

**Answer: D**

**Explanation (EN):** Every database in the `pg_default` tablespace has a subdirectory inside `$PGDATA/base/` named after the database's Object Identifier (OID), where its table and index files reside.

**한국어 핵심 해설:** 기본 테이블스페이스 pg_default의 DB별 릴레이션 파일은 base/<database_oid> 아래에 둔다.

---

### Q5. What is the maximum default size of a single physical segment file for a relation (table or index) before EDB Postgres splits it into multiple files (e.g., `<filenode>.1`, `<filenode>.2`)?

- **A)** 1 GB
- **B)** 16 MB
- **C)** 8 GB
- **D)** 32 GB

**Answer: A**

**Explanation (EN):** When a table or index exceeds 1 GB, Postgres splits it into 1 GB segments named `relfilenode.1`, `relfilenode.2`, etc., to avoid file size limitations on various filesystems.

**한국어 핵심 해설:** 표준 빌드에서 한 릴레이션 세그먼트 파일은 보통 최대 1GB이고, 초과하면 .1, .2 등 추가 세그먼트가 만들어진다.

---

### Q6. What is the role of the Free Space Map (`_fsm` fork) associated with a table's data file?

- **A)** It tracks which pages contain only tuples that are visible to all active transactions.
- **B)** It tracks the available free space within each 8 KB page of the relation to quickly locate space for new tuples.
- **C)** It records the commit status of transactions that modified the table.
- **D)** It caches index pointers in memory.

**Answer: B**

**Explanation (EN):** Each table has a `_fsm` fork that stores information about free space on each page so that `INSERT` and `UPDATE` operations can quickly find an existing page with enough room.

**한국어 핵심 해설:** FSM은 데이터 페이지의 여유 공간을 추적한다. 페이지 전체 가시성은 Visibility Map(_vm)이 담당한다.

---

### Q7. Which process acts as the supervisor for an EDB Postgres instance, listening for incoming client connections and spawning a dedicated backend process for each authenticated connection?

- **A)** `pg_ctl`
- **B)** `syslogger`
- **C)** `postmaster` (main `postgres` / `edb-postgres` process)
- **D)** `archiver`

**Answer: C**

**Explanation (EN):** The `postmaster` (the primary `postgres` or `edb-postgres` daemon) manages shared memory, starts background processes, listens on the TCP/IP port, and forks a child backend process for every client connection.

**한국어 핵심 해설:** 메인 postgres(postmaster) 프로세스가 새 클라이언트 연결에 대한 개별 백엔드를 생성한다.

---

### Q8. When installing EDB Postgres Advanced Server (EPAS) on Linux, which installation mode pre-configures the instance with `redwood` date semantics, data dictionary views (`DBA_*`, `ALL_*`, `USER_*`), and SPL support?

- **A)** PostgreSQL Compatible Mode
- **B)** ANSI Strict Mode
- **C)** Hybrid Enterprise Mode
- **D)** Oracle Compatible Mode (Redwood Mode)

**Answer: D**

**Explanation (EN):** EDB Postgres Advanced Server supports an Oracle-compatible cluster initialization mode (Redwood, e.g. `--redwood-like`) as distinct from PostgreSQL-compatible `--no-redwood-compat`. Oracle mode enables compatibility objects and behavior. Details vary by configuration parameter and installed EPAS features.

**한국어 핵심 해설:** EPAS의 Oracle(Redwood) 호환 초기화 모드는 데이터 사전 뷰, 데이터 형식/함수 등 Oracle 호환 기능을 활성화한다.

---

## Domain 2 — Configuration & Cluster Management (Q9–Q17)

### Q9. If you modify the `shared_buffers` parameter in `postgresql.conf`, how must you apply the change?

- **A)** Restart the database server completely (`pg_ctl restart`)
- **B)** Run `pg_ctl reload`
- **C)** Run `SELECT pg_reload_conf();`
- **D)** Reconnect the client session

**Answer: A**

**Explanation (EN):** Parameters with `context = postmaster` in `pg_settings` (such as `shared_buffers`, `max_connections`, `port`, and `wal_level`) allocate shared memory at startup and require a full server restart to take effect.

**한국어 핵심 해설:** shared_buffers는 서버 시작 때 메모리를 확보하므로 설정 변경 시 전체 인스턴스 재시작이 필요하다.

---

### Q10. Which SQL command modifies a configuration parameter persistently across server restarts by writing the setting to `postgresql.auto.conf`?

- **A)** `SET work_mem = '64MB';`
- **B)** `ALTER SYSTEM SET work_mem = '64MB';`
- **C)** `ALTER CLUSTER SET work_mem = '64MB';`
- **D)** `UPDATE pg_settings SET setting = '64MB' WHERE name = 'work_mem';`

**Answer: B**

**Explanation (EN):** `ALTER SYSTEM` writes the configuration setting to `$PGDATA/postgresql.auto.conf`, which is read after `postgresql.conf` whenever the server starts or reloads.

**한국어 핵심 해설:** ALTER SYSTEM은 postgresql.auto.conf에 값을 기록한다. 적용에는 해당 파라미터의 context에 따라 reload 또는 restart가 필요하다.

---

### Q11. Which system view can you query to determine if a modified configuration parameter is waiting for a server restart before taking effect?

- **A)** `pg_stat_activity` — column `wait_event`
- **B)** `pg_config` — column `requires_restart`
- **C)** `pg_settings` — column `pending_restart`
- **D)** `pg_stat_bgwriter` — column `restart_needed`

**Answer: C**

**Explanation (EN):** In `pg_settings`, the boolean column `pending_restart` is `true` if the value has been changed in the configuration file (and reloaded), but requires a postmaster restart to be applied.

**한국어 핵심 해설:** pg_settings.pending_restart를 조회하면 재시작해야 반영되는 설정 변경 여부를 볼 수 있다.

---

### Q12. What is the difference between the `fast` and `immediate` shutdown modes in `pg_ctl stop -m <mode>`?

- **A)** `fast` waits for all active clients to disconnect voluntarily; `immediate` rolls back active transactions cleanly.
- **B)** There is no difference in version 18.
- **C)** `fast` skips WAL flushing; `immediate` flushes WAL before exiting.
- **D)** `fast` aborts active transactions, disconnects clients, and writes a clean shutdown checkpoint; `immediate` terminates processes abruptly without a checkpoint, requiring crash recovery on the next startup.

**Answer: D**

**Explanation (EN):** `smart` mode waits for clients to disconnect. `fast` mode (the default) terminates open connections, rolls back active transactions, and performs a clean checkpoint. `immediate` kills processes immediately, causing a crash recovery replay of WAL on next boot.

**한국어 핵심 해설:** fast는 기존 연결을 정리하고 정상 종료한다. immediate는 즉시 종료하므로 다음 시작에서 crash recovery가 필요하다.

---

### Q13. Which parameter sets the base memory limit for each sort operation, and also influences hash operation memory usage (which may be multiplied by `hash_mem_multiplier`)?

- **A)** `work_mem`
- **B)** `shared_buffers`
- **C)** `maintenance_work_mem`
- **D)** `effective_cache_size`

**Answer: A**

**Explanation (EN):** `work_mem` sets the base memory budget per sort or similar query operation, not a per-query or cluster-wide limit. Hash operations can use `work_mem * hash_mem_multiplier` before spilling. Multiple plan nodes and sessions may consume memory concurrently.

**한국어 핵심 해설:** work_mem은 정렬 등의 작업 단위 메모리 기준값이다. Hash 연산은 hash_mem_multiplier에 따라 더 많은 메모리를 쓸 수 있다.

---

### Q14. What does the parameter `effective_cache_size` actually do?

- **A)** It allocates the specified amount of RAM from the OS at database startup.
- **B)** It provides an estimate to the query planner of how much memory is available for disk caching by the OS and within the database combined, influencing index vs. sequential scan cost estimates.
- **C)** It caps the total memory used by all active connections.
- **D)** It sets the maximum size of the WAL buffer.

**Answer: B**

**Explanation (EN):** `effective_cache_size` does not allocate any memory. It is purely an advisory hint for the query planner to estimate the probability that data blocks are already cached in `shared_buffers` or the OS page cache.

**한국어 핵심 해설:** effective_cache_size는 실제 메모리 할당량이 아니라 플래너가 추정하는 유효 캐시 크기다.

---

### Q15. How can an administrator configure a specific parameter (e.g., `work_mem = '128MB'`) so that it applies ONLY when a specific user (`report_user`) connects to a specific database (`analytics_db`)?

- **A)** Edit `pg_hba.conf` to include the parameter.
- **B)** Add a conditional block in `postgresql.conf`.
- **C)** Execute `ALTER ROLE report_user IN DATABASE analytics_db SET work_mem = '128MB';`
- **D)** Execute `GRANT SET ON PARAMETER work_mem TO report_user;`

**Answer: C**

**Explanation (EN):** Postgres supports per-role, per-database, and combined role-in-database parameter defaults stored in `pg_db_role_setting` via `ALTER ROLE ... IN DATABASE ... SET`.

**한국어 핵심 해설:** ALTER ROLE ... IN DATABASE ... SET을 사용해 특정 사용자와 특정 DB 조합에만 기본 설정을 지정할 수 있다.

---

### Q16. Which settings govern memory used by maintenance tasks such as manual `VACUUM` and `CREATE INDEX`, and by autovacuum workers when their own override is configured?

- **A)** `work_mem`
- **B)** `wal_buffers`
- **C)** `temp_buffers`
- **D)** `autovacuum_work_mem` (if set) or `maintenance_work_mem`

**Answer: D**

**Explanation (EN):** `maintenance_work_mem` controls memory used by maintenance operations. Autovacuum workers use `autovacuum_work_mem` if set to a value other than -1; otherwise they use `maintenance_work_mem`.

**한국어 핵심 해설:** 수동 VACUUM/CREATE INDEX 등은 주로 maintenance_work_mem을 사용한다. autovacuum 작업자는 autovacuum_work_mem 설정이 있으면 이를 우선한다.

---

### Q17. If the same parameter is set in multiple places, which of the following has the HIGHEST precedence?

- **A)** `SET` command executed interactively inside the current client session
- **B)** `postgresql.auto.conf`
- **C)** `ALTER DATABASE ... SET`
- **D)** `postgresql.conf`

**Answer: A**

**Explanation (EN):** Session-level `SET` (or `SET LOCAL` within a transaction) overrides role/database defaults, which in turn override `postgresql.auto.conf`, which overrides `postgresql.conf`.

**한국어 핵심 해설:** 세션 SET은 비교한 선택지 가운데 우선순위가 가장 높다. 단 해당 파라미터가 세션 변경 가능한 유형이어야 한다.

---

## Domain 3 — Roles, Security & Authentication (Q18–Q26)

### Q18. How does EDB Postgres process rules inside the `pg_hba.conf` file when a client attempts to connect?

- **A)** It evaluates all rules and applies the most restrictive match.
- **B)** It reads rules sequentially from top to bottom and stops at the first record that matches the connection type, client IP address, requested database, and user name.
- **C)** It reads rules from bottom to top.
- **D)** It prioritizes `scram-sha-256` rules over `md5` rules regardless of line order.

**Answer: B**

**Explanation (EN):** `pg_hba.conf` is evaluated strictly from top to bottom. The first matching rule is used to perform authentication; if authentication fails on that rule, subsequent rules are NOT tried.

**한국어 핵심 해설:** pg_hba.conf는 위에서 아래로 첫 번째 일치 항목을 적용하며 인증 실패 시 다음 항목으로 재시도하지 않는다.

---

### Q19. What is the default password encryption algorithm (`password_encryption`) in EDB Postgres v18?

- **A)** `md5`
- **B)** `sha512`
- **C)** `scram-sha-256`
- **D)** `crypt`

**Answer: C**

**Explanation (EN):** `scram-sha-256` has been the default password hashing and authentication method since PostgreSQL 14, offering significantly stronger cryptographic security than `md5`.

**한국어 핵심 해설:** 기본 password_encryption은 scram-sha-256이다. 저장된 비밀번호의 형식과 클라이언트 인증 방식은 구별해야 한다.

---

### Q20. Starting in PostgreSQL 15 and continuing through v18, what are the default privileges on the `public` schema for non-superuser roles in a newly created database?

- **A)** `USAGE` and `CREATE` are granted to `PUBLIC`.
- **B)** All users are automatically owners of the `public` schema.
- **C)** Both `USAGE` and `CREATE` are revoked from `PUBLIC`.
- **D)** Only `USAGE` is granted to `PUBLIC`; `CREATE` is revoked from `PUBLIC` by default.

**Answer: D**

**Explanation (EN):** To prevent security vulnerabilities (CVE-2018-1058), the `public` schema only grants `USAGE` to `PUBLIC` by default. Only the database owner (and superusers) have `CREATE` privileges on `public` unless explicitly granted.

**한국어 핵심 해설:** PostgreSQL 15 이후 신규 DB의 public 스키마에서 PUBLIC은 기본 USAGE 권한을 갖지만 CREATE는 기본적으로 없다.

---

### Q21. Which SQL command ensures that any *future* tables created by user `app_owner` in schema `sales` will automatically grant `SELECT` privileges to the role `readonly_role`?

- **A)** `ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA sales GRANT SELECT ON TABLES TO readonly_role;`
- **B)** `GRANT SELECT ON ALL TABLES IN SCHEMA sales TO readonly_role;`
- **C)** `ALTER SCHEMA sales GRANT DEFAULT SELECT TO readonly_role;`
- **D)** `GRANT FUTURE SELECT ON SCHEMA sales TO readonly_role;`

**Answer: A**

**Explanation (EN):** `GRANT ... ON ALL TABLES` only affects existing tables. To automatically apply permissions to objects created in the future, you must use `ALTER DEFAULT PRIVILEGES` specifying the creator role (`FOR ROLE app_owner`).

**한국어 핵심 해설:** ALTER DEFAULT PRIVILEGES는 지정한 생성자가 앞으로 만드는 객체에 대한 기본 권한을 지정한다.

---

### Q22. When Row-Level Security (RLS) is enabled on a table (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`), which roles bypass the RLS policies by default?

- **A)** Only the `postgres` / `enterprisedb` initialization user.
- **B)** The table owner, superusers, and roles with the `BYPASSRLS` attribute (unless `FORCE ROW LEVEL SECURITY` is enabled on the table for the table owner).
- **C)** Any role that has `DELETE` and `UPDATE` privileges on the table.
- **D)** No roles bypass RLS once enabled.

**Answer: B**

**Explanation (EN):** By default, table owners bypass RLS unless `ALTER TABLE ... FORCE ROW LEVEL SECURITY` is set. Superusers and roles granted `BYPASSRLS` always bypass RLS.

**한국어 핵심 해설:** 슈퍼유저·BYPASSRLS 역할은 RLS를 우회한다. 소유자는 기본 우회하나 FORCE ROW LEVEL SECURITY로 적용 대상이 될 수 있다.

---

### Q23. When a role is created with `NOINHERIT` and is granted membership in `dba_team` with the `SET` membership option enabled, how can it use privileges of `dba_team`?

- **A)** The role cannot log in.
- **B)** The role can never use `dba_team`'s privileges under any circumstance.
- **C)** The role does not automatically have `dba_team`'s privileges upon login, but can activate them by running `SET ROLE dba_team;`.
- **D)** The role inherits table privileges but not schema privileges.

**Answer: C**

**Explanation (EN):** `NOINHERIT` prevents automatic use of privileges granted through memberships. `SET ROLE dba_team` can activate the role if the membership path allows SET. Since PostgreSQL 16, role memberships separately track `INHERIT` and `SET` options.

**한국어 핵심 해설:** NOINHERIT 역할은 멤버 역할 권한을 자동 상속하지 않는다. 다만 SET ROLE 사용은 해당 멤버십의 SET 옵션이 허용되어야 한다.

---

### Q24. In EDB Postgres Advanced Server (EPAS), which feature allows DBAs to enforce password complexity rules, account lockouts after failed login attempts (`FAILED_LOGIN_ATTEMPTS`), and password expiration (`PASSWORD_LIFE_TIME`) similar to Oracle?

- **A)** `pg_ident.conf` mappings
- **B)** `pg_crypto` extension
- **C)** Row-Level Security policies
- **D)** EPAS Password Profiles (`CREATE PROFILE ... LIMIT ...`)

**Answer: D**

**Explanation (EN):** EPAS supports Oracle-compatible `CREATE PROFILE`, `ALTER PROFILE`, and `ALTER ROLE ... PROFILE` commands to manage password lifecycle, reuse history, complexity verification functions, and account lockouts.

**한국어 핵심 해설:** EPAS PROFILE은 로그인 실패 횟수, 비밀번호 만료/잠금 등 Oracle 호환 암호 정책을 정의한다.

---

### Q25. Which predefined (built-in) role can be granted to a monitoring user in v18 so they can read all statistics views (such as full query strings and connection details in `pg_stat_activity`) without granting superuser privileges?

- **A)** `pg_read_all_stats` (or `pg_monitor`)
- **B)** `pg_read_all_data`
- **C)** `pg_stat_scan_tables`
- **D)** `pg_execute_server_program`

**Answer: A**

**Explanation (EN):** `pg_read_all_stats` allows reading all `pg_stat_*` views including other users' queries in `pg_stat_activity`. The umbrella role `pg_monitor` includes `pg_read_all_stats`, `pg_read_all_settings`, and `pg_stat_scan_tables`.

**한국어 핵심 해설:** pg_read_all_stats는 상세 통계를 읽을 수 있으며 pg_monitor는 관련 사전 정의 모니터링 역할을 포함한다.

---

### Q26. What new authentication method was natively introduced in PostgreSQL / EDB Postgres v18 for `pg_hba.conf` to support modern token-based single sign-on (SSO) identity providers?

- **A)** `kerberos5`
- **B)** `oauth` (OAuth 2.0 authentication)
- **C)** `saml2`
- **D)** `jwt-native`

**Answer: B**

**Explanation (EN):** PostgreSQL 18 adds an `oauth` authentication method for `pg_hba.conf`, integrating with OAuth 2.0 token-bearing clients. Actual server-side validation requires configured validator libraries (`oauth_validator_libraries`); no default validator implementation ships with core PostgreSQL.

**한국어 핵심 해설:** PostgreSQL 18은 pg_hba.conf에 oauth 방식을 도입했다. 서버 측 토큰 검증 모듈을 별도로 설정해야 한다.

---

## Domain 4 — Database Objects, SQL & EPAS (Q27–Q36)

### Q27. How does a `MATERIALIZED VIEW` differ from a standard `VIEW`?

- **A)** A standard view cannot contain `JOIN` clauses, whereas a materialized view can.
- **B)** A materialized view updates automatically in real-time on every `INSERT` to the base table.
- **C)** A materialized view physically stores the query result set on disk and can be indexed, requiring `REFRESH MATERIALIZED VIEW` to update its data.
- **D)** A materialized view is dropped automatically when the session ends.

**Answer: C**

**Explanation (EN):** A standard view is just a stored query run dynamically on access. A materialized view persists data on disk like a table, supports indexes, and must be refreshed via `REFRESH MATERIALIZED VIEW [CONCURRENTLY]`.

**한국어 핵심 해설:** 일반 VIEW는 쿼리 정의를 보관하고 MATERIALIZED VIEW는 결과를 저장하며 REFRESH가 필요하다.

---

### Q28. Which requirement must be met before running `REFRESH MATERIALIZED VIEW CONCURRENTLY my_mat_view;` on a populated materialized view?

- **A)** The base tables must have no active transactions.
- **B)** `wal_level` must be set to `logical`.
- **C)** The server must be in single-user mode.
- **D)** The materialized view must have at least one `UNIQUE` index on one or more columns (that does not use a `WHERE` clause and is not an expression index).

**Answer: D**

**Explanation (EN):** The materialized view must already be populated and have at least one suitable UNIQUE index using only column names, indexing all rows (not a partial or expression index).

**한국어 핵심 해설:** CONCURRENTLY 리프레시는 전체 행을 포괄하는 적절한 UNIQUE 인덱스가 필요하며, 뷰가 이미 데이터를 가진 상태여야 한다.

---

### Q29. What is an advantage of creating an `UNLOGGED` table (`CREATE UNLOGGED TABLE ...`)? What is the primary trade-off?

- **A)** Advantage: Faster writes because changes are not written to the Write-Ahead Log (WAL). Trade-off: The table is automatically truncated upon crash recovery and is not replicated to standby servers.
- **B)** Advantage: Data is compressed automatically. Trade-off: The table is read-only.
- **C)** Advantage: It is only visible to the current session. Trade-off: It cannot have indexes.
- **D)** Advantage: It bypasses `shared_buffers`. Trade-off: It cannot exceed 1 GB.

**Answer: A**

**Explanation (EN):** Unlogged tables skip WAL generation, making writes significantly faster. However, they are not crash-safe (automatically truncated if the server crashes or shuts down uncleanly) and are not sent to physical standby replicas.

**한국어 핵심 해설:** UNLOGGED 테이블 데이터 변경에는 일반적인 WAL 기록이 생략된다. 비정상 종료 후 데이터가 비워질 수 있으며 standby에도 복제되지 않는다.

---

### Q30. Which partitioning strategy should you choose when partitioning a table based on a discrete set of state codes (e.g., `'KR'`, `'US'`, `'JP'`)?

- **A)** `PARTITION BY RANGE`
- **B)** `PARTITION BY LIST`
- **C)** `PARTITION BY HASH`
- **D)** `PARTITION BY INTERVAL`

**Answer: B**

**Explanation (EN):** `LIST` partitioning explicitly specifies which discrete key values appear in each partition. `RANGE` is used for continuous intervals (like dates), and `HASH` distributes rows evenly using a modulus/remainder hash function.

**한국어 핵심 해설:** 국가 코드처럼 가능한 값의 목록에 따라 나눌 때 LIST 파티셔닝이 적합하다.

---

### Q31. What happens when a row is inserted into a declarative partitioned table, and its partition key value does NOT match any existing child partition?

- **A)** The row is silently discarded.
- **B)** A new partition is automatically created in standard PostgreSQL.
- **C)** If a `DEFAULT` partition exists, the row is routed there; otherwise, the `INSERT` fails with an error.
- **D)** The row is stored in the parent table's heap file.

**Answer: C**

**Explanation (EN):** Parent partitioned tables have no storage of their own. If no matching child partition exists, the row goes into the `DEFAULT` partition (if defined) or raises an error. (Note: EPAS in Oracle mode also supports `INTERVAL` partitioning which auto-creates range partitions).

**한국어 핵심 해설:** 일치하는 파티션이 없으면 DEFAULT 파티션으로 들어가거나, DEFAULT도 없으면 오류가 발생한다.

---

### Q32. Why is the `JSONB` data type generally preferred over the `JSON` data type for query-heavy applications?

- **A)** `JSONB` preserves exact whitespace and duplicate keys from the input string.
- **B)** `JSONB` uses UTF-16 encoding while `JSON` uses ASCII.
- **C)** `JSON` cannot store arrays.
- **D)** `JSONB` stores data in a decomposed binary format, which makes `INSERT` slightly slower due to parsing overhead, but makes data retrieval, extraction, and indexing (via GIN indexes) much faster.

**Answer: D**

**Explanation (EN):** `JSON` stores an exact text copy of the input and must reparse on every query. `JSONB` parses the JSON into a binary structure on write, supporting fast key lookups and GIN indexing.

**한국어 핵심 해설:** JSONB는 파싱된 바이너리 구조로 저장되어 검색·인덱싱에 유리하고, JSON은 입력 표현을 보존한다.

---

### Q33. Which index type is ideal for very large, naturally ordered append-only tables (such as time-series sensor logs ordered by `created_at`) because it stores only the minimum and maximum values for ranges of physical heap pages, resulting in a tiny index footprint?

- **A)** `BRIN` (Block Range Index)
- **B)** `GIN` (Generalized Inverted Index)
- **C)** `B-Tree`
- **D)** `GiST` (Generalized Search Tree)

**Answer: A**

**Explanation (EN):** `BRIN` stores summary metadata (min/max) for groups of contiguous physical blocks (`pages_per_range`), making it orders of magnitude smaller than a B-Tree when physical storage order correlates strongly with column values.

**한국어 핵심 해설:** BRIN은 물리적 페이지 범위별 요약 정보를 저장하므로 시간순 데이터처럼 저장 순서와 상관도가 높을 때 효율적이다.

---

### Q34. What is the purpose of the `INCLUDE` clause in `CREATE INDEX idx_cust ON orders (customer_id) INCLUDE (order_date, total_amount);`?

- **A)** It filters the index so only non-null rows of `order_date` are indexed.
- **B)** It adds non-key payload columns to the leaf pages of the B-Tree index to enable **Index-Only Scans** without bloating the upper B-Tree search levels or enforcing uniqueness on the included columns.
- **C)** It creates a composite primary key.
- **D)** It encrypts the included columns.

**Answer: B**

**Explanation (EN):** `INCLUDE` adds non-key payload columns to a B-Tree index so some queries are eligible for index-only scans. Whether heap visits can be avoided also depends on the Visibility Map (all-visible heap pages) and the planner. Included columns do not affect uniqueness checks.

**한국어 핵심 해설:** INCLUDE는 비검색 키 컬럼을 인덱스에 보관한다. Index-Only Scan이 실제 heap 접근을 피하려면 Visibility Map의 all-visible 조건 등이 충족되어야 한다.

---

### Q35. In EDB Postgres Advanced Server (EPAS), what is a `PACKAGE`?

- **A)** An operating system RPM or DEB installer file.
- **B)** A compressed backup archive created by `pg_dump`.
- **C)** A schema object that groups logically related SPL procedures, functions, variables, cursors, and exceptions into two parts: a Package Specification (public interface) and a Package Body (implementation).
- **D)** A tablespace container.

**Answer: C**

**Explanation (EN):** EPAS supports Oracle-compatible Packages (`CREATE PACKAGE` and `CREATE PACKAGE BODY`), enabling encapsulation, session-persistent package variables, and overloaded procedures/functions.

**한국어 핵심 해설:** EPAS PACKAGE는 패키지 사양과 구현 본문으로 관련 SPL 루틴과 상태를 묶는다.

---

### Q36. In EPAS (Oracle Compatible Mode), what is the purpose of adding `PRAGMA AUTONOMOUS_TRANSACTION;` inside an SPL procedure or function?

- **A)** It runs the procedure on a standby replica automatically.
- **B)** It executes the query using parallel workers.
- **C)** It bypasses all table triggers.
- **D)** It allows the procedure to execute its SQL statements in an independent transaction that can `COMMIT` or `ROLLBACK` without affecting (rolling back or committing) the calling parent transaction.

**Answer: D**

**Explanation (EN):** Autonomous transactions are commonly used in EPAS for audit logging or error logging so that log records can be committed permanently even if the main business transaction rolls back.

**한국어 핵심 해설:** AUTONOMOUS_TRANSACTION은 호출 측 트랜잭션과 독립적으로 커밋/롤백할 수 있다.

---

## Domain 5 — Maintenance, MVCC & Performance (Q37–Q45)

### Q37. Under EDB Postgres's Multi-Version Concurrency Control (MVCC) architecture, what happens physically when an `UPDATE` statement modifies a row?

- **A)** The existing tuple is marked as expired (setting its `xmax` to the updating transaction's XID), and a brand-new tuple version is inserted into the table (with `xmin` set to the updating XID).
- **B)** The existing row is modified in-place and the old values are written to an Undo Tablespace.
- **C)** The entire table is locked exclusively until the `UPDATE` commits.
- **D)** The row is deleted from disk immediately and re-inserted on commit.

**Answer: A**

**Explanation (EN):** Postgres does not use an external undo log like Oracle. Instead, `UPDATE` inserts a new tuple version into the heap and marks the old tuple with `xmax`, leaving a dead tuple behind once no active transaction needs to see the old version.

**한국어 핵심 해설:** UPDATE는 일반적으로 새로운 튜플 버전을 생성해 MVCC 가시성을 관리한다. 기존 튜플을 곧바로 덮어쓰지 않는다.

---

### Q38. What is the key difference between a standard `VACUUM` and `VACUUM FULL`?

- **A)** Standard `VACUUM` requires an `ACCESS EXCLUSIVE` lock; `VACUUM FULL` runs concurrently with reads and writes.
- **B)** Standard `VACUUM` reclaims dead tuple space internally for reuse by future inserts/updates without shrinking the OS file size (usually) and runs concurrently; `VACUUM FULL` rewrites the entire table into a new disk file, returns free space to the OS, and requires an `ACCESS EXCLUSIVE` lock blocking all reads and writes.
- **C)** Standard `VACUUM` updates query planner statistics; `VACUUM FULL` does not.
- **D)** `VACUUM FULL` only cleans indexes, not tables.

**Answer: B**

**Explanation (EN):** Standard `VACUUM` marks space in the Free Space Map (`_fsm`) for reuse without blocking normal DML. `VACUUM FULL` compacts the table by rewriting it from scratch, taking an `ACCESS EXCLUSIVE` lock and temporarily requiring extra disk space for the new copy.

**한국어 핵심 해설:** 표준 VACUUM은 공간을 재사용하도록 표시하고 동시 DML을 허용하지만, VACUUM FULL은 테이블을 재작성하고 강한 잠금을 요구한다.

---

### Q39. What is a Heap-Only Tuple (HOT) update, and which conditions generally allow it?

- **A)** An update stored entirely in RAM; requires `work_mem > 1GB`.
- **B)** An update performed by the `autovacuum` daemon.
- **C)** An UPDATE can avoid new index entries when no indexed columns referenced by non-summarizing indexes change and the new tuple fits on the same heap page as the old tuple.
- **D)** An update on an unlogged table.

**Answer: C**

**Explanation (EN):** HOT updates avoid new index entries when the new tuple stays on the same heap page and no columns referenced by non-summarizing indexes are changed. BRIN indexes are summarizing indexes and are an exception. Lowering `fillfactor` can leave space for HOT updates.

**한국어 핵심 해설:** HOT는 새 인덱스 엔트리 작성을 피하는 최적화다. 동일 페이지에 신규 튜플이 들어가야 하며 BRIN 같은 요약형 인덱스는 예외다.

---

### Q40. Why does EDB Postgres perform routine **Transaction ID (XID) Freezing** during vacuum operations?

- **A)** To encrypt old historical records.
- **B)** To clear the `pg_stat_statements` view.
- **C)** To move cold data to slower archival tablespaces.
- **D)** Because tuple header transaction IDs (`xmin`/`xmax`) are 32-bit integers (~4 billion values), and without marking old committed tuples as "frozen" (always visible in the past), XID wraparound would cause old data to appear to be in the future and become invisible.

**Answer: D**

**Explanation (EN):** Modulo-$2^{32}$ arithmetic gives roughly 2 billion transactions in the past and 2 billion in the future. Vacuum marks old committed rows as frozen before they reach `autovacuum_freeze_max_age` (default 200 million transactions) to prevent catastrophic transaction ID wraparound.

**한국어 핵심 해설:** 오래된 트랜잭션 ID를 freeze해 wraparound로 인한 가시성 문제를 예방한다.

---

### Q41. What is the purpose of the `ANALYZE` command?

- **A)** It samples rows in tables and updates statistical distribution data in `pg_statistic` (`pg_stats`) so the Query Planner can generate accurate execution plans.
- **B)** It checks the physical integrity of data blocks and repairs corrupted checksums.
- **C)** It reclaims dead tuples from indexes.
- **D)** It rewrites fragmented B-Tree indexes.

**Answer: A**

**Explanation (EN):** The cost-based optimizer relies on statistics (row counts, most common values, histograms, `n_distinct`) stored in `pg_statistic`. `ANALYZE` (and auto-analyze) refreshes these statistics.

**한국어 핵심 해설:** ANALYZE는 데이터 분포 통계를 갱신해 실행계획의 비용 추정을 개선한다.

---

### Q42. Assuming a 10,000-row table, default dead-tuple VACUUM settings (`autovacuum_vacuum_threshold = 50`, `autovacuum_vacuum_scale_factor = 0.2`) and excluding insert-triggered and anti-wraparound VACUUM, what is the minimum number of obsolete tuples that **exceeds** the calculated threshold?

- **A)** 50
- **B)** 2,051
- **C)** 2,000
- **D)** 2,050

**Answer: B**

**Explanation (EN):** The calculated threshold is 50 + 0.2 × 10,000 = 2,050 (assuming the default maximum threshold does not bind). Autovacuum checks whether the obsolete tuple count **exceeds** the threshold, so the minimum integer count is 2,051. Counts are based on collected statistics and scheduling is not instantaneous.

**한국어 핵심 해설:** 기본 공식으로 계산한 임계값은 2,050개다. 실제 일반 VACUUM 발동 기준은 이를 초과하는 수이므로 최소 2,051개(다른 발동 조건 배제 가정)다.

---

### Q43. What happens when you execute `EXPLAIN ANALYZE DELETE FROM orders WHERE status = 'CANCELLED';`?

- **A)** Postgres only estimates the plan and does NOT actually delete any rows.
- **B)** Postgres runs `ANALYZE` on the `orders` table before explaining the query.
- **C)** Postgres **actually executes** the `DELETE` statement, removing the rows and reporting the real execution time and row counts (unless wrapped in `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;`).
- **D)** Postgres throws a syntax error because `EXPLAIN ANALYZE` only works on `SELECT`.

**Answer: C**

**Explanation (EN):** `EXPLAIN ANALYZE` genuinely executes the statement in order to measure actual timings and row counts. For `INSERT`, `UPDATE`, or `DELETE`, always wrap it in `BEGIN; ... ROLLBACK;` if you do not want the data modifications to persist.

**한국어 핵심 해설:** EXPLAIN ANALYZE는 DELETE도 실제 실행한다. 데이터 변경을 남기지 않으려면 별도 트랜잭션에서 실행 후 ROLLBACK해야 한다.

---

### Q44. In an `EXPLAIN (ANALYZE, BUFFERS)` output, what does `Buffers: shared hit=120 read=15` indicate?

- **A)** 120 blocks were written to disk, and 15 blocks were read from WAL.
- **B)** 15 queries are waiting on locks held by 120 sessions.
- **C)** 120 rows were returned out of 15 pages.
- **D)** 120 blocks (8 KB each) were found already in `shared_buffers` (RAM cache), and 15 blocks had to be requested from the operating system / disk.

**Answer: D**

**Explanation (EN):** `shared hit=120` means 120 block accesses were satisfied from PostgreSQL shared buffers; `read=15` means 15 blocks were fetched from outside shared buffers, possibly from the OS cache rather than physical disk. Default PostgreSQL block size is generally 8 KB.

**한국어 핵심 해설:** shared hit는 PostgreSQL 공유 버퍼에서 재사용된 블록이고 read는 OS로부터 읽어온 블록으로, 물리 디스크 입출력과 1:1은 아니다.

---

### Q45. Which built-in function should you use in a production environment to terminate a stuck backend process (canceling its connection completely) vs. only canceling its currently running query while keeping the connection alive?

- **A)** `pg_terminate_backend(pid)` terminates the connection; `pg_cancel_backend(pid)` only cancels the current query.
- **B)** `pg_cancel_backend(pid)` terminates the connection; `pg_terminate_backend(pid)` shuts down the entire database cluster.
- **C)** `pg_kill_pid(pid)` and `pg_stop_query(pid)`.
- **D)** Both functions do the exact same thing.

**Answer: A**

**Explanation (EN):** `pg_cancel_backend(pid)` sends `SIGINT` to abort only the active query. `pg_terminate_backend(pid)` sends `SIGTERM` to close the entire backend session.

**한국어 핵심 해설:** pg_cancel_backend는 실행 쿼리를 취소하고 pg_terminate_backend는 해당 백엔드 연결을 종료한다.

---

## Domain 6 — Backup, Recovery & Replication (Q46–Q53)

### Q46. Which statement accurately describes `pg_dump`?

- **A)** It takes a physical block-level backup of the entire database cluster including all databases and roles.
- **B)** It creates a consistent logical backup of a **single database** without blocking concurrent reads or writes (`SELECT`, `INSERT`, `UPDATE`, `DELETE`), though it does not back up cluster-wide roles or tablespaces.
- **C)** It requires stopping the database server first.
- **D)** It backs up the Write-Ahead Log (`pg_wal`) for Point-In-Time Recovery.

**Answer: B**

**Explanation (EN):** `pg_dump` uses a repeatable-read MVCC snapshot to logically export a single database. Cluster-wide objects like roles (`CREATE ROLE`) and tablespaces must be backed up separately using `pg_dumpall --globals-only` (`-g`).

**한국어 핵심 해설:** pg_dump는 단일 DB의 논리 백업이다. 역할과 테이블스페이스 등 클러스터 전역 객체는 별도로 백업한다.

---

### Q47. Which `pg_dump` output format (`-F`) is the most flexible, allowing parallel dump/restore, selective table restoration, compression by default, and reordering of items during `pg_restore`?

- **A)** Plain-text SQL (`-Fp`)
- **B)** CSV format (`-Fcsv`)
- **C)** Custom format (`-Fc`) or Directory format (`-Fd`, which supports parallel `-j` during `pg_dump`)
- **D)** Binary WAL format (`-Fw`)

**Answer: C**

**Explanation (EN):** While plain format (`-Fp`) outputs a raw SQL script to be run with `psql`, Custom (`-Fc`) and Directory (`-Fd`) formats are used with `pg_restore` to selectively restore specific tables/schemas and run parallel jobs (`-j`). Note that `-Fd` supports parallel jobs during both `pg_dump` and `pg_restore`, whereas `-Fc` supports parallel jobs during `pg_restore`.

**한국어 핵심 해설:** Custom/Directory 포맷은 pg_restore로 선택적·병렬 복원이 가능하며 병렬 덤프는 Directory 포맷이 지원한다.

---

### Q48. What tool is used to take a physical binary backup of a running EDB Postgres cluster to set up a Streaming Replication standby or prepare for Point-In-Time Recovery (PITR)?

- **A)** `pg_dumpall`
- **B)** `pg_controldata`
- **C)** `pg_restore`
- **D)** `pg_basebackup` (or enterprise backup tools like Barman / pgBackRest)

**Answer: D**

**Explanation (EN):** `pg_basebackup` makes a binary copy of the database cluster files over the replication protocol while the server is running and automatically puts the backup into and out of backup mode.

**한국어 핵심 해설:** pg_basebackup은 실행 중인 서버의 물리적 기본 백업을 생성한다.

---

### Q49. For point-in-time recovery using continuous WAL archiving to recover a database to 14:30, which backup and WAL data are required?

- **A)** A usable physical base backup that can be recovered to a point before 14:30, plus an unbroken supply of WAL covering the backup and all changes through the selected target time.
- **B)** A `pg_dump` file and the `pg_hba.conf` file.
- **C)** Only the archived WAL files since the day the cluster was initialized.
- **D)** A logical replication slot and `pg_stat_statements`.

**Answer: A**

**Explanation (EN):** PITR needs a suitable completed physical base backup plus the full sequence of WAL needed from the backup start through the recovery target. Typically archived WAL is supplied through `restore_command`; other valid WAL sources may supplement it. Logical `pg_dump` alone is insufficient.

**한국어 핵심 해설:** PITR에는 목표 시각 이전의 적절한 물리적 기본 백업과 목표 시점까지 연속적으로 이용 가능한 WAL이 필요하다. WAL은 아카이브 등 유효한 공급원에서 제공한다.

---

### Q50. Since PostgreSQL 12 (and in v18), how do you tell an EDB Postgres instance to start up in Streaming Replication Standby mode?

- **A)** Create a file named `recovery.conf` in `$PGDATA`.
- **B)** Create an empty file named `standby.signal` in `$PGDATA` and configure `primary_conninfo` in `postgresql.conf` (or `postgresql.auto.conf`).
- **C)** Set `standby_mode = on` in `pg_hba.conf`.
- **D)** Run `pg_ctl start -m standby`.

**Answer: B**

**Explanation (EN):** `recovery.conf` was removed in version 12. Today, `standby.signal` places the server into standby mode (continuously fetching WAL via `primary_conninfo`), while `recovery.signal` places it into targeted archive recovery mode (PITR).

**한국어 핵심 해설:** standby.signal과 primary_conninfo를 통해 standby 동작과 상위 서버 스트리밍 연결을 구성한다.

---

### Q51. What is the main purpose of configuring a **Replication Slot** on the primary server in a Streaming Replication architecture?

- **A)** It encrypts the replication network traffic.
- **B)** It automatically fails over the VIP when the primary crashes.
- **C)** It ensures that the primary server does not delete or recycle WAL segments from `pg_wal` until the standby has confirmed receiving them (preventing the standby from falling irrevocably behind), though it risks filling up the primary's disk if the standby goes offline for too long (unless mitigated by `max_slot_wal_keep_size`).
- **D)** It converts physical replication into logical replication.

**Answer: C**

**Explanation (EN):** Replication slots track the standby's LSN progress on the primary, preventing premature removal of needed WAL files. `max_slot_wal_keep_size` can be set to protect the primary disk from filling up completely if a replica stays down.

**한국어 핵심 해설:** 복제 슬롯은 필요 WAL의 조기 삭제를 막지만, 지연된 대기 노드 때문에 WAL이 누적될 수 있다.

---

### Q52. In Synchronous Streaming Replication, which setting of `synchronous_commit` guarantees that a transaction committed on the primary is not only flushed to the standby's disk, but also **applied and immediately visible** to `SELECT` queries run on the hot standby?

- **A)** `on`
- **B)** `remote_write`
- **C)** `local`
- **D)** `remote_apply`

**Answer: D**

**Explanation (EN):** `synchronous_commit = on` waits for WAL to be flushed to the standby's disk, but the standby might not have replayed the WAL into its data pages yet. `remote_apply` waits until the standby has both written and replayed (applied) the transaction so read-after-write queries on the standby immediately see the new data.

**한국어 핵심 해설:** remote_apply는 동기 standby가 WAL을 재생해 쿼리에서 보일 때까지 커밋 완료를 대기한다. 적절한 동기 복제 구성이 전제다.

---

### Q53. What is a key capability of **Logical Replication** (Publication / Subscription) compared to Physical Streaming Replication?

- **A)** Logical replication allows replicating a subset of tables (or specific rows/columns), replicating across different major PostgreSQL versions, and permits write operations on other tables on the subscriber node.
- **B)** Logical replication replicates DDL statements (`CREATE TABLE`, `ALTER TABLE`) automatically out of the box in standard core PostgreSQL.
- **C)** Logical replication replicates the entire cluster bit-for-bit including `pg_catalog` and all databases simultaneously.
- **D)** Logical replication requires `wal_level = replica`.

**Answer: A**

**Explanation (EN):** Physical replication is an exact block-for-block clone of the entire cluster (read-only standby, same major version). Logical replication requires `wal_level = logical` and allows selective table replication, row/column filtering, cross-major-version upgrades, and writable subscribers.

**한국어 핵심 해설:** 논리 복제는 테이블/행/열 단위로 선택적 복제가 가능하며 물리 복제와 달리 다른 메이저 버전 간 복제도 지원한다.

---

## Domain 7 — PostgreSQL 18 Core Features (Q54–Q60)

### Q54. Version 18 introduces a major storage subsystem enhancement to overlap disk read latency with CPU processing, replacing traditional synchronous blocking `pread()` calls for sequential scans, bitmap heap scans, and vacuuming. What is this feature called, and what key parameter controls its backend mechanism?

- **A)** Direct Memory Access (`dma_mode`)
- **B)** Asynchronous I/O (AIO), configured via `io_method` (supporting `worker`, `io_uring` on Linux, or `sync`)
- **C)** Kernel Bypass Threading (`kthread_io`)
- **D)** Vectorized Buffer Pool (`vector_buffers`)

**Answer: B**

**Explanation (EN):** PostgreSQL 18 introduces asynchronous I/O to overlap selected disk operations with execution. `io_method` supports `worker`, `io_uring` where available, and `sync`; the `pg_aios` view exposes AIO state. `worker` is the normal default.

**한국어 핵심 해설:** PostgreSQL 18의 비동기 I/O는 io_method 파라미터와 pg_aios 모니터링으로 다룬다.

---

### Q55. In version 18, how does generated column behavior (`GENERATED ALWAYS AS (...)`) change compared to version 17 and earlier?

- **A)** Generated columns are deprecated in v18.
- **B)** Generated columns can now only be created on `TEXT` columns.
- **C)** Version 18 introduces **Virtual Generated Columns** (`GENERATED ALWAYS AS (...) VIRTUAL`) and makes `VIRTUAL` the **default** if neither `VIRTUAL` nor `STORED` is specified, computing values on-the-fly during reads rather than consuming disk space.
- **D)** `STORED` columns are automatically converted into indexes.

**Answer: C**

**Explanation (EN):** Prior to v18, Postgres only supported `STORED` generated columns. Version 18 adds `VIRTUAL` generated columns (computed dynamically at read time like a view column, zero storage overhead) and sets `VIRTUAL` as the default when the storage keyword is omitted.

**한국어 핵심 해설:** 18에서 VIRTUAL 생성 컬럼이 도입됐고 기본값이 됐다. 기존 버전의 STORED와 의미가 다르다.

---

### Q56. Suppose you have a composite B-Tree index on `(region_code, order_date)` where `region_code` has very few distinct values (low cardinality, e.g., only 4 regions), and a user runs `SELECT * FROM sales WHERE order_date = '2026-10-23';` (omitting `region_code`). Which new v18 query optimizer feature allows this query to still efficiently use the composite B-Tree index?

- **A)** Bitmap Hash Join
- **B)** Incremental Sort Pushdown
- **C)** BRIN Bloom Filter
- **D)** B-Tree Index Skip Scan

**Answer: D**

**Explanation (EN):** Version 18 introduces **B-Tree Index Skip Scan**. When the leading column(s) of a composite B-Tree index are omitted from the query predicate and have low cardinality, the executor can dynamically "hop" or skip across each distinct leading key value to search the second column (`order_date`) efficiently instead of falling back to a full index scan or sequential scan.

**한국어 핵심 해설:** B-Tree Skip Scan은 복합 인덱스 선행 열 조건이 없어도 특정 조건에서 효율적인 인덱스 탐색을 가능하게 한다.

---

### Q57. Which new built-in UUID generation function was added in version 18 that generates **timestamp-ordered (time-sortable)** 128-bit UUIDs, significantly reducing B-Tree index page fragmentation and random I/O compared to completely random `uuidv4()` / `gen_random_uuid()`?

- **A)** `uuidv7()`
- **B)** `uuid_generate_v1mc()`
- **C)** `seq_uuid()`
- **D)** `time_uuid_v6()`

**Answer: A**

**Explanation (EN):** Version 18 adds native `uuidv7()` (along with `uuidv4()` as an alias for `gen_random_uuid()`). UUIDv7 encodes a 48-bit Unix epoch millisecond timestamp in the most significant bits followed by random bits, ensuring new rows append near the right-hand side of B-Tree indexes with high cache locality.

**한국어 핵심 해설:** uuidv7()은 시간순 정렬에 유리한 UUID를 생성한다. 모든 삽입 환경에서 성능 이득이 보장되는 것은 아니다.

---

### Q58. In version 18, what new enhancement was added to the `RETURNING` clause of `INSERT`, `UPDATE`, `DELETE`, and `MERGE` statements?

- **A)** `RETURNING` can now write directly to an external CSV file.
- **B)** Support for `OLD` and `NEW` qualifiers (e.g., `UPDATE products SET price = price * 1.1 RETURNING OLD.price AS old_price, NEW.price AS new_price;`) to inspect both before-and-after values in a single DML statement.
- **C)** `RETURNING` can now be used inside a `SELECT` statement.
- **D)** `RETURNING` automatically commits the transaction.

**Answer: B**

**Explanation (EN):** Prior to v18, `RETURNING` in an `UPDATE` only returned the new post-update values (and `DELETE` only returned old values). Version 18 allows referencing `OLD.<col>` and `NEW.<col>` (and aliasing them via `RETURNING WITH (OLD AS o, NEW AS n)`) across `INSERT`, `UPDATE`, `DELETE`, and `MERGE`.

**한국어 핵심 해설:** PostgreSQL 18 RETURNING에서 OLD/NEW를 함께 지정하면 DML 전후 값을 비교할 수 있다.

---

### Q59. What is a PostgreSQL 18 improvement to `pg_upgrade` that reduces the risk of poor query plans immediately after a major-version upgrade?

- **A)** `pg_upgrade` now upgrades B-Tree indexes without copying them.
- **B)** `pg_upgrade` no longer requires stopping the old cluster.
- **C)** It carries over many query planner statistics from the old cluster, although missing statistics and follow-up `ANALYZE` work may still be necessary.
- **D)** `pg_upgrade` automatically converts `md5` passwords to `scram-sha-256`.

**Answer: C**

**Explanation (EN):** PostgreSQL 18 can transfer many optimizer statistics during `pg_upgrade`, reducing initial plan instability. This does not eliminate follow-up work: documentation recommends `vacuumdb --all --analyze-in-stages --missing-stats-only` followed by `vacuumdb --all --analyze-only` after upgrade as appropriate.

**한국어 핵심 해설:** 18의 pg_upgrade는 많은 플래너 통계를 이전하지만, 누락된 통계 보완과 일반 ANALYZE가 여전히 권장된다.

---

### Q60. Which major security and data-integrity feature is enabled **by default** when running `initdb` to initialize a new cluster in version 18 (which previously required explicitly passing `-k` or `--data-checksums`)?

- **A)** Transparent Data Encryption (TDE)
- **B)** SSL client certificate verification
- **C)** Row-Level Security on all system catalogs
- **D)** Data Page Checksums (`data_checksums` enabled by default unless `--no-data-checksums` is specified)

**Answer: D**

**Explanation (EN):** For PostgreSQL 18, `initdb` defaults to enabling page checksums; `--no-data-checksums` disables them. A major upgrade via `pg_upgrade` requires compatible checksum settings for the source and target clusters.

**한국어 핵심 해설:** 18에서 initdb는 데이터 페이지 체크섬을 기본 활성화한다. pg_upgrade 시 이전·새 클러스터의 체크섬 설정 일치도 고려해야 한다.

---

## Domain 8 — EPAS Oracle Compatibility (Additional) (Q61–Q75)

### Q61. When initializing an EDB Postgres Advanced Server 18 cluster, which choice explicitly selects Oracle (Redwood) compatibility mode?

- **A)** Use the `--redwood-like` initialization option.
- **B)** Set `wal_level=logical` before `initdb`.
- **C)** Install the `pg_stat_statements` extension.
- **D)** Create a database named `oracle` after initialization.

**Answer: A**

**Explanation (EN):** EPAS supports Redwood compatibility at cluster initialization using `--redwood-like`; the contrasting PostgreSQL-compatibility option is `--no-redwood-compat`.

**한국어 핵심 해설:** Oracle 호환 모드 선택은 `--redwood-like`와 관련된다. 클러스터 초기화 당시 호환성 설정을 이해해야 한다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/planning/configuration_mode/

---

### Q62. What is the default listening port commonly used by EDB Postgres Advanced Server installations, as distinct from community PostgreSQL?

- **A)** 5433
- **B)** 5444
- **C)** 15432
- **D)** 8080

**Answer: B**

**Explanation (EN):** EPAS packages generally default to port 5444, while community PostgreSQL commonly uses 5432. Always inspect the installed instance settings.

**한국어 핵심 해설:** EPAS의 일반적인 기본 포트는 5444이고 커뮤니티 PostgreSQL은 5432다. 실제 운영 포트는 설정을 확인해야 한다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/planning/planning_prerequisites/epas_known_differences/

---

### Q63. When `edb_redwood_date` is `true` during an EPAS `CREATE TABLE`, how is a declared `DATE` column handled?

- **A)** It becomes a string (`VARCHAR`).
- **B)** It is always stored as a pure date with no time component.
- **C)** It is translated to a timestamp-like type capable of storing time-of-day.
- **D)** It is automatically indexed.

**Answer: C**

**Explanation (EN):** With `edb_redwood_date=true`, EPAS translates DATE column declarations to TIMESTAMP for table definition storage. With false, native PostgreSQL DATE semantics apply to such columns.

**한국어 핵심 해설:** edb_redwood_date=true이면 테이블 DATE 열이 시간 성분을 포함할 수 있는 TIMESTAMP 형태로 매핑된다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/database_administration/01_configuration_parameters/03_configuration_parameters_by_functionality/11_compatibility_options/

---

### Q64. What is the correct statement about access privileges when an EPAS user queries an object through a synonym?

- **A)** Synonym creation automatically grants SELECT on the target.
- **B)** Only superusers may resolve synonyms.
- **C)** A synonym bypasses all underlying object privileges.
- **D)** The user still needs the applicable privileges on the referenced object.

**Answer: D**

**Explanation (EN):** A synonym is an alternate name, not a privilege grant. The querying user requires permissions on the referenced object.

**한국어 핵심 해설:** 동의어(SYNONYM)는 객체 별칭이며 접근 권한 자체를 부여하지 않는다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/reference/oracle_compatibility_reference/epas_compat_sql/35_create_synonym/

---

### Q65. In an EPAS package, where can the implementation and private subprograms that are not exposed in the package specification be defined?

- **A)** In the package body (`CREATE PACKAGE BODY`).
- **B)** Only in `postgresql.conf`.
- **C)** Only in a materialized view.
- **D)** Only in the role catalog.

**Answer: A**

**Explanation (EN):** A package specification defines the public interface; its body contains implementations and can define private functions and procedures.

**한국어 핵심 해설:** PACKAGE 사양은 외부 인터페이스를 선언하고 PACKAGE BODY에는 실제 구현과 비공개 루틴을 둔다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/application_programming/02_packages/01_package_components/

---

### Q66. Which EPAS Oracle-compatible package/procedure is typically used to emit informational lines from SPL code?

- **A)** UTL_FILE.FOPEN
- **B)** DBMS_OUTPUT.PUT_LINE
- **C)** DBMS_RANDOM.VALUE
- **D)** DBMS_LOCK.SLEEP

**Answer: B**

**Explanation (EN):** `DBMS_OUTPUT.PUT_LINE` writes an informational line to the package output buffer (or client display, depending on client settings).

**한국어 핵심 해설:** SPL 실행 중 문자열 메시지를 출력할 때 DBMS_OUTPUT.PUT_LINE을 사용한다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/reference/oracle_compatibility_reference/epas_compat_bip_guide/03_built-in_packages/

---

### Q67. Which additional operational consideration applies to an EPAS autonomous transaction?

- **A)** It never performs database writes.
- **B)** It is limited to SELECT statements only.
- **C)** It consumes a connection slot while active and can increase connection pressure.
- **D)** It forces the outer transaction to COMMIT.

**Answer: C**

**Explanation (EN):** An EPAS autonomous transaction executes independently and uses an extra connection slot while in progress; monitor `max_connections` under load.

**한국어 핵심 해설:** EPAS 자율 트랜잭션은 진행 중 별도의 연결 슬롯을 소비하므로 max_connections 용량에 영향을 줄 수 있다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/application_programming/epas_compat_spl/06_transaction_control/03_pragma_autonomous_transaction/

---

### Q68. In EPAS `CREATE PROFILE ... LIMIT`, what does `FAILED_LOGIN_ATTEMPTS` control?

- **A)** Minimum password length
- **B)** Number of database connections per role
- **C)** Maximum idle transaction duration
- **D)** Number of unsuccessful logins before account lockout

**Answer: D**

**Explanation (EN):** EPAS password profiles can define the number of failed login attempts leading to account lockout, with duration governed by `PASSWORD_LOCK_TIME`.

**한국어 핵심 해설:** FAILED_LOGIN_ATTEMPTS는 로그인 실패에 따른 계정 잠금 기준 횟수를 정의한다.

**Official reference:** https://www.enterprisedb.com/docs/epas/18/reference/oracle_compatibility_reference/epas_compat_sql/29_create_profile/

---

### Q69. What setup step is documented for using the `DBMS_SCHEDULER` package to create and manage Oracle-style scheduled jobs in EPAS 18?

- **A)** Execute `CREATE EXTENSION IF NOT EXISTS dbms_scheduler` and configure the scheduler component.
- **B)** Set `fsync=off`.
- **C)** Install `postgis`.
- **D)** Disable WAL archiving.

**Answer: A**

**Explanation (EN):** The EPAS documentation shows creating the `dbms_scheduler` extension; background execution depends on the associated EDB job scheduler component.

**한국어 핵심 해설:** DBMS_SCHEDULER를 사용하려면 dbms_scheduler 확장 및 스케줄러 실행 구성을 확인한다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/reference/oracle_compatibility_reference/epas_compat_bip_guide/03_built-in_packages/15_dbms_scheduler/

---

### Q70. Which EPAS built-in package supplies Oracle-style server-side file I/O procedures?

- **A)** DBMS_ALERT
- **B)** UTL_FILE
- **C)** DBMS_OUTPUT
- **D)** DBMS_RANDOM

**Answer: B**

**Explanation (EN):** `UTL_FILE` provides compatible file input/output operations. File permissions and directory settings require appropriate controls.

**한국어 핵심 해설:** Oracle 호환 서버 측 파일 읽기/쓰기에는 UTL_FILE 패키지를 사용한다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/reference/oracle_compatibility_reference/epas_compat_bip_guide/03_built-in_packages/

---

### Q71. Which EPAS built-in package is associated with generating random values for Oracle-compatible applications?

- **A)** DBMS_SQL
- **B)** DBMS_LOCK
- **C)** DBMS_RANDOM
- **D)** UTL_MAIL

**Answer: C**

**Explanation (EN):** `DBMS_RANDOM` provides random-value procedures/functions in EPAS Oracle compatibility packages.

**한국어 핵심 해설:** DBMS_RANDOM 패키지는 Oracle 호환 랜덤 값 생성에 사용한다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/reference/oracle_compatibility_reference/epas_compat_bip_guide/03_built-in_packages/

---

### Q72. Which EPAS Oracle-compatible package offers procedures for dynamic SQL statement processing?

- **A)** DBMS_OUTPUT
- **B)** DBMS_ALERT
- **C)** DBMS_JOB
- **D)** DBMS_SQL

**Answer: D**

**Explanation (EN):** `DBMS_SQL` provides Oracle-compatible dynamic SQL facilities, with security considerations for dynamically constructed SQL.

**한국어 핵심 해설:** 동적 SQL 지원 패키지는 DBMS_SQL이며 동적 식별자·값 처리에는 SQL 인젝션 방지 설계가 필요하다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/reference/oracle_compatibility_reference/epas_compat_bip_guide/03_built-in_packages/

---

### Q73. To authorize a non-superuser to execute a restricted built-in EPAS package such as SYS.UTL_FILE, which statement is documented?

- **A)** GRANT EXECUTE ON PACKAGE SYS.UTL_FILE TO john;
- **B)** CREATE EXTENSION role_executor;
- **C)** SET ROLE sys;
- **D)** ALTER SYSTEM SET package_access = on;

**Answer: A**

**Explanation (EN):** The EPAS built-in package reference documents `GRANT EXECUTE ON PACKAGE SYS.UTL_FILE TO john;`. Package execution does not automatically override OS-level file permissions.

**한국어 핵심 해설:** EPAS에서 제한된 내장 패키지 실행 권한은 GRANT EXECUTE ON PACKAGE로 명시적으로 부여한다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/reference/oracle_compatibility_reference/epas_compat_bip_guide/03_built-in_packages/

---

### Q74. Which EPAS configuration setting controls the Oracle-like treatment of concatenating a string with NULL?

- **A)** `autovacuum`
- **B)** `edb_redwood_strings`
- **C)** `hot_standby`
- **D)** `wal_compression`

**Answer: B**

**Explanation (EN):** `edb_redwood_strings` controls Oracle/Redwood style concatenation behavior involving NULL values.

**한국어 핵심 해설:** NULL과 문자열 결합의 Oracle 호환 동작은 edb_redwood_strings 설정의 영향을 받는다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/database_administration/01_configuration_parameters/03_configuration_parameters_by_functionality/11_compatibility_options/

---

### Q75. Which statement best describes DDL transaction behavior in EPAS as compared with Oracle?

- **A)** CREATE TABLE always commits immediately as Oracle does.
- **B)** DDL cannot run inside a transaction.
- **C)** EPAS DDL such as CREATE TABLE is transactional and can be rolled back.
- **D)** DDL is only allowed in autocommit-off sessions.

**Answer: C**

**Explanation (EN):** EPAS does not automatically commit DDL in the Oracle manner; DDL statements can be rolled back as part of a transaction.

**한국어 핵심 해설:** EPAS의 CREATE TABLE 같은 DDL은 Oracle의 암묵적 커밋과 달리 트랜잭션에서 롤백할 수 있다.

**Official reference:** https://www.enterprisedb.com/docs/epas/latest/application_programming/epas_compat_spl/06_transaction_control/about_transactions/

---

## Domain 9 — PostgreSQL 18 Extended Features (Additional) (Q76–Q85)

### Q76. What PostgreSQL 18 syntax can enforce non-overlapping ranges for a temporal `UNIQUE` or `PRIMARY KEY` constraint?

- **A)** CHECK OVERLAPS
- **B)** RANGE EXCLUSIVE
- **C)** NO DATE COLLISION
- **D)** WITHOUT OVERLAPS

**Answer: D**

**Explanation (EN):** PostgreSQL 18 allows `WITHOUT OVERLAPS` on the final range or multirange column of eligible UNIQUE and PRIMARY KEY constraints.

**한국어 핵심 해설:** 시간 범위 중복을 막는 temporal unique/primary key 문법은 WITHOUT OVERLAPS다.

**Official reference:** https://www.postgresql.org/docs/18/sql-createtable.html

---

### Q77. Which PostgreSQL 18 foreign-key feature can require a referenced key to cover the full period of a referencing row?

- **A)** PERIOD
- **B)** MERGE
- **C)** UNLOGGED
- **D)** HOT

**Answer: A**

**Explanation (EN):** Temporal foreign keys use `PERIOD`, checking whether reference periods collectively cover the referencing row period, subject to syntax requirements.

**한국어 핵심 해설:** PostgreSQL 18의 PERIOD 외래키는 참조 대상 기간이 참조 레코드 전체 기간을 포함하는지 검증한다.

**Official reference:** https://www.postgresql.org/docs/18/sql-createtable.html

---

### Q78. Which new PostgreSQL 18 modifier can declare `CHECK` and foreign-key constraints without enforcing them?

- **A)** NO CHECKING
- **B)** NOT ENFORCED
- **C)** IGNORE CONSTRAINT
- **D)** DEFER NEVER

**Answer: B**

**Explanation (EN):** PostgreSQL 18 adds `NOT ENFORCED` for CHECK and foreign key constraints; non-enforcement differs from `NOT VALID`, which still checks future changes.

**한국어 핵심 해설:** NOT ENFORCED는 제약을 정의하지만 검사·강제하지 않는다. 기존 데이터 검사만 건너뛰는 NOT VALID와 다르다.

**Official reference:** https://www.postgresql.org/docs/18/release-18.html

---

### Q79. What is required on the PostgreSQL 18 server before an `oauth` HBA entry can successfully validate client tokens?

- **A)** Setting `password_encryption = md5`
- **B)** A role named `oauth`
- **C)** A configured implementation in `oauth_validator_libraries`
- **D)** An automatic built-in default validator requiring no configuration

**Answer: C**

**Explanation (EN):** Core PostgreSQL 18 provides the OAuth authentication infrastructure but does not ship a default token validator. A validator module must be configured via `oauth_validator_libraries`.

**한국어 핵심 해설:** OAuth 인증은 기본 제공 검증기가 없으므로 별도 validator 모듈과 oauth_validator_libraries 설정이 필요하다.

**Official reference:** https://www.postgresql.org/docs/18/runtime-config-connection.html

---

### Q80. When using `pg_upgrade` to move a cluster without data checksums into PostgreSQL 18, which initialization choice must be considered?

- **A)** Always switch to 16-bit checksums regardless of source.
- **B)** Run `pg_rewind` instead of pg_upgrade.
- **C)** Disable archive_mode permanently.
- **D)** Use `--no-data-checksums` on the new cluster so checksum settings match the source.

**Answer: D**

**Explanation (EN):** PostgreSQL 18 enables checksums by default during initdb, while pg_upgrade requires matching source and target checksum settings. The new cluster may require `--no-data-checksums`.

**한국어 핵심 해설:** 체크섬이 꺼진 기존 클러스터를 pg_upgrade할 때 새 v18 클러스터의 체크섬 설정을 일치시켜야 한다.

**Official reference:** https://www.postgresql.org/docs/18/release-18.html

---

### Q81. In PostgreSQL 18, which option is used with `vacuumdb` to analyze only objects whose optimizer statistics are missing?

- **A)** `--missing-stats-only`
- **B)** `--freeze-only`
- **C)** `--new-tables-only`
- **D)** `--check-only`

**Answer: A**

**Explanation (EN):** `vacuumdb --missing-stats-only` is available in PostgreSQL 18 and is restricted to appropriate analyze-only modes and privileges.

**한국어 핵심 해설:** 통계 누락 객체만 분석하려면 vacuumdb --missing-stats-only를 분석 관련 옵션과 함께 사용한다.

**Official reference:** https://www.postgresql.org/docs/18/release-18.html

---

### Q82. Which PostgreSQL 18 `pg_dump` option enables exporting planner statistics as part of a dump?

- **A)** `--vacuum`
- **B)** `--statistics`
- **C)** `--wal-stats`
- **D)** `--copy-cache`

**Answer: B**

**Explanation (EN):** PostgreSQL 18 adds `pg_dump --statistics` and related statistics-only options for exporting planner statistics.

**한국어 핵심 해설:** PostgreSQL 18에서는 pg_dump --statistics로 플래너 통계를 함께 덤프할 수 있다.

**Official reference:** https://www.postgresql.org/docs/18/release-18.html

---

### Q83. Which PostgreSQL 18 enhancement helps verify backups stored as tar archives?

- **A)** `pg_restore` verifies physical blocks in all backups.
- **B)** `pg_rewind` verifies backup manifests.
- **C)** `pg_verifybackup` supports verification of tar-format base backups.
- **D)** `pg_dump` converts tar backups to WAL automatically.

**Answer: C**

**Explanation (EN):** PostgreSQL 18 extends `pg_verifybackup` to tar-format backups. The tool verifies against a backup manifest; there are restrictions on parsing WAL for tar-format backups.

**한국어 핵심 해설:** PostgreSQL 18의 pg_verifybackup은 tar 형식 백업 검증을 지원하지만 tar 형식의 WAL 파싱에는 제한이 있다.

**Official reference:** https://www.postgresql.org/docs/18/app-pgverifybackup.html

---

### Q84. In PostgreSQL 18, which change applies to `VACUUM` and `ANALYZE` commands issued on a parent relation that has inheritance children?

- **A)** They automatically perform PITR.
- **B)** They always take ACCESS EXCLUSIVE locks on all children.
- **C)** They refuse to run when a child exists.
- **D)** They may also process inheritance children; use `ONLY` to retain the previous parent-only behavior.

**Answer: D**

**Explanation (EN):** PostgreSQL 18 changes VACUUM and ANALYZE behavior to process inherited children when applicable; `ONLY` requests the older behavior.

**한국어 핵심 해설:** 18부터 VACUUM/ANALYZE가 상속 자식을 처리하도록 바뀌었다. ONLY로 종전의 부모 대상 동작을 선택할 수 있다.

**Official reference:** https://www.postgresql.org/docs/18/release-18.html

---

### Q85. Which statement about built-in `uuidv7()` in PostgreSQL 18 is most accurate?

- **A)** It generates 128-bit UUIDs with a timestamp-based ordering component.
- **B)** It generates 64-bit identifiers.
- **C)** It is guaranteed globally monotonically increasing without exception.
- **D)** It produces UUIDs based only on sequential integers.

**Answer: A**

**Explanation (EN):** UUIDv7 is a 128-bit UUID variant with a time-based component that usually improves insertion locality over random UUIDv4. It is not a guaranteed global sequential counter.

**한국어 핵심 해설:** uuidv7()은 타임스탬프 기반 128비트 식별자를 생성하지만 완전한 전역 단조 증가 번호는 아니다.

**Official reference:** https://www.postgresql.org/docs/18/functions-uuid.html

---

## Domain 10 — Operational Diagnostics (Additional) (Q86–Q90)

### Q86. Which system view on a PostgreSQL standby helps inspect the upstream WAL receiver status?

- **A)** `pg_stat_bgwriter`
- **B)** `pg_stat_wal_receiver`
- **C)** `pg_locks`
- **D)** `pg_stat_user_tables`

**Answer: B**

**Explanation (EN):** `pg_stat_wal_receiver` shows the WAL receiver state and connection information on a standby.

**한국어 핵심 해설:** Standby 쪽에서 WAL 수신 상태를 볼 때 pg_stat_wal_receiver를 조회한다.

**Official reference:** https://www.postgresql.org/docs/18/monitoring-stats.html

---

### Q87. On a primary server, which view shows per-WAL-sender replication status and LSN progress for connected standbys?

- **A)** `pg_stat_io`
- **B)** `pg_stat_activity_summary`
- **C)** `pg_stat_replication`
- **D)** `pg_settings`

**Answer: C**

**Explanation (EN):** `pg_stat_replication` shows connected WAL sender processes on the primary, including replication state and LSN positions.

**한국어 핵심 해설:** Primary에서는 pg_stat_replication으로 연결된 복제 대기 노드의 상태와 LSN 진행을 확인한다.

**Official reference:** https://www.postgresql.org/docs/18/monitoring-stats.html

---

### Q88. What does `SELECT pg_is_in_recovery();` return on a running physical standby server?

- **A)** FALSE, because the standby is read-only.
- **B)** An integer replication lag in bytes.
- **C)** NULL until a checkpoint.
- **D)** TRUE, because it is in recovery mode.

**Answer: D**

**Explanation (EN):** `pg_is_in_recovery()` is true while the server is in recovery; a physical streaming standby remains in recovery until promoted.

**한국어 핵심 해설:** 물리 standby는 승격 전까지 recovery 상태이므로 pg_is_in_recovery()가 true다.

**Official reference:** https://www.postgresql.org/docs/18/functions-admin.html

---

### Q89. Which PostgreSQL system view can help find parsing or configuration errors in the effective host-based authentication file?

- **A)** `pg_hba_file_rules`
- **B)** `pg_roles`
- **C)** `pg_stat_progress_vacuum`
- **D)** `pg_stat_database`

**Answer: A**

**Explanation (EN):** `pg_hba_file_rules` displays the interpreted HBA rules and can expose errors. Reload is required to apply edits to the actual authentication rules.

**한국어 핵심 해설:** pg_hba_file_rules에서 HBA 항목과 오류를 점검할 수 있다. 변경 내용을 서버에 적용하려면 reload가 필요하다.

**Official reference:** https://www.postgresql.org/docs/18/auth-pg-hba-conf.html

---

### Q90. Which built-in SQL function helps identify process IDs currently blocking another backend process?

- **A)** `pg_terminate_backend()`
- **B)** `pg_blocking_pids(pid)`
- **C)** `pg_wal_replay_resume()`
- **D)** `pg_relation_size()`

**Answer: B**

**Explanation (EN):** `pg_blocking_pids(pid)` returns the PIDs of backend processes that are blocking the specified process; combine this with `pg_stat_activity` for incident triage.

**한국어 핵심 해설:** 세션의 잠금 대기 원인을 파악하려면 pg_blocking_pids(pid)와 pg_stat_activity를 함께 사용할 수 있다.

**Official reference:** https://www.postgresql.org/docs/18/functions-info.html

---

## 전체 정답표

아래 정답은 현재 문제집에 표시된 **A/B/C/D 선택지 순서**를 기준으로 합니다.

| 문항 | 정답 | 문항 | 정답 | 문항 | 정답 | 문항 | 정답 | 문항 | 정답 |
|---:|:---:|---:|:---:|---:|:---:|---:|:---:|---:|:---:|
| 1 | A | 19 | C | 37 | A | 55 | C | 73 | A |
| 2 | B | 20 | D | 38 | B | 56 | D | 74 | B |
| 3 | C | 21 | A | 39 | C | 57 | A | 75 | C |
| 4 | D | 22 | B | 40 | D | 58 | B | 76 | D |
| 5 | A | 23 | C | 41 | A | 59 | C | 77 | A |
| 6 | B | 24 | D | 42 | B | 60 | D | 78 | B |
| 7 | C | 25 | A | 43 | C | 61 | A | 79 | C |
| 8 | D | 26 | B | 44 | D | 62 | B | 80 | D |
| 9 | A | 27 | C | 45 | A | 63 | C | 81 | A |
| 10 | B | 28 | D | 46 | B | 64 | D | 82 | B |
| 11 | C | 29 | A | 47 | C | 65 | A | 83 | C |
| 12 | D | 30 | B | 48 | D | 66 | B | 84 | D |
| 13 | A | 31 | C | 49 | A | 67 | C | 85 | A |
| 14 | B | 32 | D | 50 | B | 68 | D | 86 | B |
| 15 | C | 33 | A | 51 | C | 69 | A | 87 | C |
| 16 | D | 34 | B | 52 | D | 70 | B | 88 | D |
| 17 | A | 35 | C | 53 | A | 71 | C | 89 | A |
| 18 | B | 36 | D | 54 | B | 72 | D | 90 | B |

## 참고: 핵심 SQL/운영 명령

```sql
-- 설정과 재시작 필요 여부 확인
SELECT name, setting, context, pending_restart
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'effective_cache_size');

-- 복제/복구 상태 확인
SELECT pg_is_in_recovery();
SELECT * FROM pg_stat_replication;       -- primary에서 확인
SELECT * FROM pg_stat_wal_receiver;      -- standby에서 확인

-- 인증 설정 파일의 문법/적용 대상 점검
SELECT * FROM pg_hba_file_rules WHERE error IS NOT NULL;

-- 잠금 차단 세션 확인
SELECT pid, pg_blocking_pids(pid) AS blockers, query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- 자동 VACUUM 임계값(단순 계산 예시)
SELECT 50 + 0.2 * 10000 AS vacuum_threshold;
```

```bash
# PostgreSQL 18 메이저 업그레이드 후 통계 보완 (환경에 따라 수행)
vacuumdb --all --analyze-in-stages --missing-stats-only
vacuumdb --all --analyze-only
```

## 공식 문서

- [EDB — Postgres AI Database Essentials 시험 카탈로그](https://www.enterprisedb.com/training/certification-exams)
- [EDB — EPAS 18 구성 모드](https://www.enterprisedb.com/docs/epas/latest/planning/configuration_mode/)
- [EDB — EPAS 18 Oracle 호환 SQL](https://www.enterprisedb.com/docs/epas/latest/reference/oracle_compatibility_reference/epas_compat_sql/)
- [EDB — EPAS 18 Built-in Packages](https://www.enterprisedb.com/docs/epas/latest/reference/oracle_compatibility_reference/epas_compat_bip_guide/03_built-in_packages/)
- [EDB — EPAS 18 자율 트랜잭션](https://www.enterprisedb.com/docs/epas/latest/application_programming/epas_compat_spl/06_transaction_control/03_pragma_autonomous_transaction/)
- [PostgreSQL 18 Release Notes](https://www.postgresql.org/docs/18/release-18.html)
- [PostgreSQL 18 Routine Vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html)
- [PostgreSQL 18 pg_upgrade](https://www.postgresql.org/docs/18/pgupgrade.html)
- [PostgreSQL 18 Resource Consumption](https://www.postgresql.org/docs/18/runtime-config-resource.html)
- [PostgreSQL 18 Index-Only Scans](https://www.postgresql.org/docs/18/indexes-index-only-scans.html)
- [PostgreSQL 18 HOT](https://www.postgresql.org/docs/18/storage-hot.html)
- [PostgreSQL 18 Backup Verification](https://www.postgresql.org/docs/18/app-pgverifybackup.html)
- [PostgreSQL 18 Continuous Archiving and PITR](https://www.postgresql.org/docs/18/continuous-archiving.html)
- [PostgreSQL 18 OAuth Authentication](https://www.postgresql.org/docs/18/auth-oauth.html)
