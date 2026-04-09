<div align="center">

# 🐘 PostgreSQL Deep Dive

**"PostgreSQL도 SQL 쓰면 되는 거 아닌가 — 와 — Dead Tuple이 왜 생기고 VACUUM이 어떻게 회수하는지, PostgreSQL의 MVCC가 왜 Undo Log가 아닌 Heap 내부 버전으로 구현되는지 아는 것은 다르다"**

<br/>

> *"MySQL만 써왔는데 PostgreSQL로 마이그레이션하면서 테이블이 10배 부풀었다 — VACUUM을 껐기 때문이었다"*

MySQL을 아는 것과 PostgreSQL이 MVCC를 완전히 다른 방식으로 구현하고 그 선택이 어떤 트레이드오프를 만드는지 아는 것의 차이.  
Dead Tuple이 Heap에 공존하는 이유, GIN 인덱스가 JSONB 내부 키를 역색인하는 원리, XID Wraparound가 DB를 강제 종료시키는 이유까지  
**왜 이렇게 설계됐는가** 라는 질문으로 PostgreSQL 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?style=flat-square&logo=postgresql&logoColor=white)](https://www.postgresql.org/docs/current/)
[![Spring](https://img.shields.io/badge/Spring_Data_JPA-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
[![Docs](https://img.shields.io/badge/Docs-41개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

PostgreSQL에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "PostgreSQL도 SQL 문법은 비슷합니다" | MySQL(Undo Log)과 PostgreSQL(Heap 내부 버전)의 MVCC 구현이 근본적으로 다른 이유, 그 차이가 왜 Dead Tuple과 VACUUM을 만드는지 |
| "VACUUM을 주기적으로 실행하세요" | VACUUM이 Dead Tuple을 회수하는 단계별 흐름, FSM/Visibility Map 업데이트, HOT Update로 인덱스 갱신 없이 버전 체인을 구성하는 원리 |
| "JSONB는 JSON보다 빠릅니다" | JSONB가 파싱 후 바이너리로 저장되는 이유, GIN 인덱스가 JSONB 내부 키-값 쌍을 역색인해서 `@>` 연산을 O(log N)으로 만드는 방법 |
| "GIN 인덱스를 쓰세요" | GIN Pending List와 Fastupdate 옵션, GIN이 B-Tree보다 전문 검색에 빠른 이유, GIN 인덱스 크기 트레이드오프 |
| "Autovacuum이 알아서 해줍니다" | XID Wraparound — 32비트 트랜잭션 ID가 40억을 넘으면 과거 트랜잭션이 미래로 보여 DB가 강제 종료되는 이유, `age(datfrozenxid)` 모니터링 |
| "Streaming Replication을 구성하세요" | WAL 기반 복제 내부, Replication Slot이 Standby가 느릴 때 Primary WAL 삭제를 막는 원리, Logical Replication과 CDC(Debezium) 조합 |
| 이론 나열 | 실행 가능한 `psql` 실험 + `EXPLAIN ANALYZE` + `pg_stat_*` 시스템 뷰 + `pageinspect` 확장 + Docker Compose 환경 + Spring 연결 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Architecture](https://img.shields.io/badge/🔹_Architecture-PostgreSQL_vs_MySQL_아키텍처-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](./postgresql-architecture/01-architecture-vs-mysql.md)
[![MVCC](https://img.shields.io/badge/🔹_MVCC-Dead_Tuple_완전_분해-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](./mvcc-vacuum/01-dead-tuple.md)
[![Indexes](https://img.shields.io/badge/🔹_Indexes-B--Tree_인덱스_심화-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](./indexes/01-btree-index.md)
[![TOAST](https://img.shields.io/badge/🔹_TOAST-TOAST_완전_분해-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](./toast-large-data/01-toast-internals.md)
[![SQL](https://img.shields.io/badge/🔹_Advanced_SQL-윈도우_함수_완전_분해-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](./advanced-sql/01-window-functions.md)
[![Replication](https://img.shields.io/badge/🔹_Replication-Streaming_Replication-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](./replication-ha/01-streaming-replication.md)
[![Perf](https://img.shields.io/badge/🔹_Performance-postgresql.conf_핵심_설정-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](./performance-tuning/01-postgresql-conf.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: PostgreSQL 아키텍처와 MySQL과의 근본 차이

> **핵심 질문:** PostgreSQL은 왜 연결당 프로세스를 fork하는가? Heap 페이지는 어떻게 생겼고, MVCC를 MySQL과 완전히 다른 방식으로 구현한 선택이 어떤 결과를 만드는가?

<details>
<summary><b>프로세스 기반 아키텍처부터 XID 가시성 규칙까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. PostgreSQL vs MySQL 아키텍처 비교](./postgresql-architecture/01-architecture-vs-mysql.md) | 연결당 프로세스 fork(PostgreSQL) vs 스레드 기반(MySQL), Shared Memory(shared_buffers)와 OS Page Cache의 이중 캐시 구조, Connection Pooling(PgBouncer)이 필수인 이유, 연결 비용과 max_connections |
| [02. Storage Manager와 페이지 구조](./postgresql-architecture/02-storage-manager-page.md) | 8KB 페이지(Heap Page) 레이아웃, `pd_lower`/`pd_upper`/`pd_special` 의미, 튜플(행) 저장 방식과 Item Pointer 배열, MySQL InnoDB 16KB 페이지와의 구조 비교, `pageinspect`로 페이지 직접 관찰 |
| [03. PostgreSQL MVCC vs MySQL MVCC](./postgresql-architecture/03-mvcc-vs-mysql-mvcc.md) | MySQL: 변경 시 Undo Log에 이전 버전 저장 / PostgreSQL: Heap 안에 이전 버전 튜플을 그대로 보존, Dead Tuple이 Heap에 남는 이유, 각 방식의 장단점, VACUUM이 반드시 필요한 이유 |
| [04. WAL(Write-Ahead Log) 완전 분해](./postgresql-architecture/04-wal-complete.md) | WAL 레코드 구조와 LSN(Log Sequence Number), Checkpoint 동작, WAL 기반 Streaming Replication 흐름, MySQL Binary Log(논리 로그)와의 차이, `pg_waldump`로 WAL 레코드 직접 분석 |
| [05. 트랜잭션 ID(XID)와 가시성 규칙](./postgresql-architecture/05-xid-visibility.md) | 모든 튜플에 `xmin`/`xmax`가 있는 이유, 트랜잭션이 어떤 버전의 튜플을 볼 수 있는지 결정하는 가시성 규칙, Snapshot의 `xmin`/`xmax`/`xip` 3가지 구성요소, XID Wraparound 위험의 원인 |

</details>

<br/>

### 🔹 Chapter 2: MVCC 심화와 VACUUM

> **핵심 질문:** Dead Tuple은 왜 쌓이고 VACUUM은 어떻게 회수하는가? HOT Update는 왜 인덱스 갱신 없이 가능하고, XID Wraparound는 어떻게 DB를 멈추는가?

<details>
<summary><b>Dead Tuple부터 Serializable Snapshot Isolation까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Dead Tuple 완전 분해](./mvcc-vacuum/01-dead-tuple.md) | UPDATE/DELETE 시 기존 튜플을 즉시 삭제하지 않는 이유(다른 트랜잭션이 여전히 읽을 수 있음), Dead Tuple이 쌓이면 발생하는 Table Bloat, `pg_stat_user_tables`의 `n_dead_tup`으로 현황 확인, Full Scan 비용 증가 메커니즘 |
| [02. VACUUM 내부 동작](./mvcc-vacuum/02-vacuum-internals.md) | VACUUM이 Dead Tuple을 회수하는 단계별 흐름(스캔 → FSM 업데이트 → Visibility Map 업데이트 → Index Vacuum), VACUUM이 Lock을 최소화하는 방법, HOT Update와 VACUUM의 상호작용, VACUUM이 OS에 공간을 반환하지 않는 이유 |
| [03. VACUUM FULL vs VACUUM](./mvcc-vacuum/03-vacuum-full-vs-vacuum.md) | VACUUM: Dead Tuple 공간 재사용 가능하게 표시(테이블 크기 유지), VACUUM FULL: 실제 파일 크기 축소(AccessExclusiveLock 필요), 운영 중 VACUUM FULL이 위험한 이유, `pg_repack` 대안 도구 |
| [04. Autovacuum 튜닝](./mvcc-vacuum/04-autovacuum-tuning.md) | Autovacuum 동작 조건(`autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor`), 테이블 크기별 튜닝 전략, 대형 테이블에서 Autovacuum이 따라오지 못하는 문제, 테이블별 `storage_parameter` 오버라이드 |
| [05. HOT(Heap Only Tuple) Update](./mvcc-vacuum/05-hot-update.md) | 인덱스가 있는 컬럼을 UPDATE하지 않을 때 인덱스 갱신 없이 같은 페이지 내에서 버전 체인 구성, HOT 업데이트 조건과 성능 이점, `fillfactor` 설정이 HOT을 늘리는 이유, `pg_stat_user_tables.n_tup_hot_upd`로 HOT 비율 확인 |
| [06. XID Wraparound](./mvcc-vacuum/06-xid-wraparound.md) | 32비트 트랜잭션 ID가 40억 개를 넘으면 과거 트랜잭션이 미래로 보이는 문제, VACUUM이 오래된 튜플을 Freeze하는 이유(`xmin`을 FrozenTransactionId=2로 설정), Wraparound 발생 시 DB 강제 종료, `age(datfrozenxid)` 모니터링 |
| [07. 격리 수준과 스냅샷](./mvcc-vacuum/07-isolation-snapshot.md) | PostgreSQL의 Snapshot Isolation 구현, REPEATABLE READ가 MySQL보다 강한 이유(Phantom Read 방지), Serializable Snapshot Isolation(SSI)으로 직렬화 가능성을 낙관적으로 보장하는 방법, 각 격리 수준 성능 비교 |

</details>

<br/>

### 🔹 Chapter 3: 인덱스 완전 분해

> **핵심 질문:** PostgreSQL은 왜 B-Tree/Hash/GiST/GIN/BRIN/Bloom 6가지 인덱스를 제공하는가? 각각은 어떤 내부 구조를 갖고 어떤 쿼리 패턴에 최적인가?

<details>
<summary><b>B-Tree Index-Only Scan부터 실행 계획 분석까지 (8개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. B-Tree 인덱스 심화](./indexes/01-btree-index.md) | PostgreSQL B-Tree가 MySQL B+Tree와 다른 점, 인덱스 페이지 구조, Index-Only Scan이 가능한 조건(Visibility Map 활용), Partial Index(`WHERE` 절이 있는 인덱스)로 인덱스 크기 줄이기 |
| [02. Hash 인덱스](./indexes/02-hash-index.md) | 등가 비교(`=`)에만 사용, B-Tree보다 빠른 등가 조회, PostgreSQL 10 이전 WAL 로깅 문제(충돌 시 재구성 필요), 범위 쿼리 지원 안 됨, Hash 인덱스가 실무에서 적합한 경우 |
| [03. GiST(Generalized Search Tree)](./indexes/03-gist-index.md) | 기하 데이터(Point, Box, Polygon), 전문 검색, 범위 데이터 인덱싱, R-Tree를 GiST 프레임워크로 구현하는 방법, PostGIS와의 조합으로 공간 쿼리 가속 |
| [04. GIN(Generalized Inverted Index)](./indexes/04-gin-index.md) | JSONB, 배열, 전문 검색의 원소를 역색인하는 방법, GIN이 B-Tree보다 JSONB `@>` 검색에 빠른 이유, GIN Pending List와 Fastupdate 옵션, GIN 인덱스 크기와 쓰기 비용 트레이드오프 |
| [05. BRIN(Block Range INdex)](./indexes/05-brin-index.md) | 물리적으로 정렬된 데이터(타임스탬프, 로그 ID)에 최적화, 블록 범위마다 `min`/`max` 값만 저장하는 초경량 인덱스, 테이블의 0.1% 크기로 효과적인 필터링, BRIN이 B-Tree를 대체할 수 있는 조건 |
| [06. Bloom 필터 인덱스](./indexes/06-bloom-filter-index.md) | 여러 컬럼의 등가 비교를 하나의 인덱스로 처리, 확률적 자료구조(False Positive 있음)의 의미, Bloom 필터 인덱스 크기가 B-Tree보다 작은 이유, 적합한 사용 사례와 False Positive 허용 범위 |
| [07. 인덱스 선택 가이드](./indexes/07-index-selection-guide.md) | 쿼리 패턴별 최적 인덱스 선택 결정 트리, Partial Index로 인덱스 크기 줄이기, Expression Index(함수 기반 인덱스), 인덱스 유지 비용(쓰기 증폭) vs 조회 이득 트레이드오프 |
| [08. 실행 계획 분석 심화](./indexes/08-explain-analyze.md) | `EXPLAIN ANALYZE`의 각 노드(SeqScan/IndexScan/IndexOnlyScan/BitmapHeapScan/NestedLoop/HashJoin/MergeJoin) 의미와 비용 모델, 플래너가 인덱스를 무시하는 조건, `pg_stats` 통계 정보 갱신으로 플래너 정확도 높이기 |

</details>

<br/>

### 🔹 Chapter 4: TOAST와 대용량 데이터

> **핵심 질문:** 8KB 페이지에 1MB짜리 JSONB를 저장하면 어떻게 되는가? TOAST는 왜 애플리케이션에 투명하고, JSONB가 JSON보다 왜 검색에 빠른가?

<details>
<summary><b>TOAST 내부부터 Elasticsearch vs PostgreSQL 전문 검색까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. TOAST 완전 분해](./toast-large-data/01-toast-internals.md) | Oversized-Attribute Storage Technique, 8KB 페이지에 들어가지 않는 큰 값을 별도 TOAST 테이블에 저장하는 방법, 압축(pglz/lz4)과 분리 저장 전략 4가지(PLAIN/EXTENDED/EXTERNAL/MAIN), TOAST가 투명한 이유 |
| [02. JSONB 내부 저장](./toast-large-data/02-jsonb-storage.md) | JSONB가 JSON과 다른 이유(파싱 후 바이너리 형태로 저장), 키 정렬 저장으로 O(log N) 키 접근, JSONB 컬럼 크기와 TOAST 임계값(2KB), GIN 인덱스로 JSONB 내부 검색 가속, `jsonb_path_query`와 JSON Path |
| [03. 배열(Array) 타입](./toast-large-data/03-array-type.md) | PostgreSQL 네이티브 배열 저장 방식, GIN 인덱스로 배열 원소 검색(`ANY`/`ALL`/`@@`), 배열 조작 함수(`unnest`, `array_agg`), 정규화(1NF) vs 배열 컬럼 선택 기준, 배열과 TOAST의 관계 |
| [04. 전문 검색(Full Text Search)](./toast-large-data/04-full-text-search.md) | `tsvector`/`tsquery` 내부 표현, `to_tsvector`로 토큰 생성과 어간 추출, GIN 인덱스로 전문 검색 가속, 한국어 전문 검색(`pg_bigm`), Elasticsearch와의 기능/성능 비교 및 선택 기준 |
| [05. Large Object vs TOAST](./toast-large-data/05-large-object-vs-toast.md) | 파일 저장에 Large Object(`lo_*`) vs TOAST 선택 기준, `pg_largeobject` 테이블 내부 구조, 파일 시스템 vs DB 저장 트레이드오프, Spring에서 Large Object 접근 방법 |

</details>

<br/>

### 🔹 Chapter 5: 고급 SQL과 분석 기능

> **핵심 질문:** 윈도우 함수는 내부에서 어떻게 실행되고, CTE는 왜 Optimization Fence인가? LATERAL JOIN이 필요한 패턴은 무엇이고, Upsert는 동시 충돌을 어떻게 처리하는가?

<details>
<summary><b>Window Function부터 GROUPING SETS까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 윈도우 함수(Window Function) 완전 분해](./advanced-sql/01-window-functions.md) | `OVER(PARTITION BY ... ORDER BY ...)` 내부 실행 방식, 각 행마다 집계하면서 행을 유지하는 원리, `ROW_NUMBER`/`RANK`/`DENSE_RANK`/`LAG`/`LEAD` 동작, 윈도우 프레임(`ROWS vs RANGE`), 실행 계획에서 WindowAgg 노드 |
| [02. CTE(Common Table Expression) 심화](./advanced-sql/02-cte-internals.md) | WITH 절이 Optimization Fence인 이유(PostgreSQL 12 이전), `MATERIALIZED vs NOT MATERIALIZED` 선택, 재귀 CTE로 트리/그래프 순회(`WITH RECURSIVE`), CTE와 서브쿼리 성능 비교, 실행 계획 차이 |
| [03. 파티셔닝 완전 분해](./advanced-sql/03-partitioning.md) | Range/List/Hash 파티셔닝 내부 동작, 파티션 프루닝(Partition Pruning)으로 쿼리가 일부 파티션만 스캔하는 방법, 파티션 인덱스 전략(로컬 vs 글로벌), PostgreSQL 파티셔닝 vs MySQL 파티셔닝 차이 |
| [04. LATERAL JOIN](./advanced-sql/04-lateral-join.md) | 서브쿼리가 왼쪽 테이블의 각 행을 참조하는 방법, 각 사용자의 최근 N개 주문 조회(`LATERAL + LIMIT`) 패턴, 상관 서브쿼리와의 차이, LATERAL이 필요한 패턴과 실행 계획 |
| [05. Upsert와 SKIP LOCKED](./advanced-sql/05-upsert-skip-locked.md) | `INSERT ... ON CONFLICT DO UPDATE`(Upsert) 내부 동작, 동시 Upsert 처리 시 충돌 해결 메커니즘, `SELECT ... FOR UPDATE SKIP LOCKED`로 큐 구현(여러 Worker가 동시에 작업을 가져가는 패턴) |
| [06. 집계 함수 심화](./advanced-sql/06-aggregation-functions.md) | `GROUPING SETS`/`CUBE`/`ROLLUP`으로 다차원 집계, `FILTER` 절로 조건부 집계, 통계 집계 함수(`percentile_cont`, `corr`), 대용량 집계 최적화(HashAgg vs GroupAgg, `work_mem` 영향) |

</details>

<br/>

### 🔹 Chapter 6: 복제와 고가용성

> **핵심 질문:** Streaming Replication은 WAL을 어떻게 전달하고, Logical Replication과 어떻게 다른가? Patroni는 etcd로 어떻게 Split-Brain을 막는가?

<details>
<summary><b>WAL 기반 복제부터 PgBouncer Transaction Mode까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Streaming Replication 완전 분해](./replication-ha/01-streaming-replication.md) | Primary가 WAL을 Standby에 스트리밍하는 방법, 동기(Synchronous) vs 비동기(Asynchronous) 복제 선택 기준, Replication Slot으로 WAL 보존(Standby가 느릴 때 Primary WAL 삭제 방지), `pg_stat_replication`으로 복제 지연 확인 |
| [02. Hot Standby](./replication-ha/02-hot-standby.md) | 복제 중인 Standby에서 읽기 쿼리 처리, 읽기 트래픽 분산, Standby의 쿼리가 Primary의 VACUUM과 충돌하는 문제(`hot_standby_feedback` 설정), Conflict 발생 시 쿼리 취소 메커니즘 |
| [03. 로직 복제(Logical Replication)](./replication-ha/03-logical-replication.md) | WAL 레벨 변경(physical → logical), 테이블 단위 선택적 복제, 이기종 버전 간 복제, CDC(Change Data Capture) 활용(Debezium + PostgreSQL Logical Replication), Publication/Subscription 모델 |
| [04. Patroni 고가용성](./replication-ha/04-patroni-ha.md) | Patroni가 etcd/Consul로 Leader Election을 구현하는 방법, 자동 장애조치(Failover) 절차, Split-Brain 방지(Fencing), Spring 애플리케이션에서 자동 연결 전환(PgBouncer + Patroni 조합) |
| [05. Connection Pooling 심화](./replication-ha/05-connection-pooling.md) | PgBouncer Session Mode vs Transaction Mode vs Statement Mode 차이, Prepared Statement와 Transaction Mode 충돌 문제와 해결, 애플리케이션 연결 풀(HikariCP)과 PgBouncer 조합 아키텍처 |

</details>

<br/>

### 🔹 Chapter 7: 성능 튜닝과 운영

> **핵심 질문:** `shared_buffers`는 얼마로 설정해야 하고, `pg_stat_statements`로 어떻게 병목을 찾는가? Long Running Transaction이 VACUUM을 막는 구조는 어떻게 되는가?

<details>
<summary><b>postgresql.conf 핵심 설정부터 Spring + PostgreSQL 최적화까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. postgresql.conf 핵심 설정](./performance-tuning/01-postgresql-conf.md) | `shared_buffers`(총 RAM의 25%), `effective_cache_size`(OS 캐시 힌트), `work_mem`(정렬/해시 조인 메모리), `maintenance_work_mem`(VACUUM/인덱스 생성 메모리), `max_connections`와 PgBouncer 조합 전략 |
| [02. 쿼리 성능 진단](./performance-tuning/02-query-diagnostics.md) | `pg_stat_statements`로 느린 쿼리 누적 통계 확인, `auto_explain`으로 느린 쿼리 실행 계획 자동 로깅, `pg_stat_activity`로 실행 중인 쿼리 확인, `pg_locks`로 잠금 대기 분석, `log_min_duration_statement` 설정 |
| [03. 인덱스 사용률 분석](./performance-tuning/03-index-usage-analysis.md) | `pg_stat_user_indexes`로 인덱스 스캔 횟수 확인, 사용되지 않는 인덱스 식별과 제거, 인덱스 부풀음(Index Bloat) 측정, `REINDEX CONCURRENTLY`로 운영 중 인덱스 재구성 |
| [04. 운영 중 발생하는 문제 패턴](./performance-tuning/04-operational-issues.md) | Long Running Transaction이 VACUUM을 막는 구조, Autovacuum이 따라오지 못하는 경우 진단, 테이블 잠금 대기로 인한 연결 누적, Checkpoint 과부하(`checkpoint_completion_target` 튜닝), `pg_cancel_backend` / `pg_terminate_backend` |
| [05. Spring + PostgreSQL 최적화](./performance-tuning/05-spring-postgresql.md) | HikariCP 최적 설정(`maximumPoolSize`, `connectionTimeout`, `idleTimeout`), R2DBC PostgreSQL 드라이버, Spring Data JPA와 PostgreSQL 특화 기능(JSONB 쿼리, 배열 타입), Flyway/Liquibase 마이그레이션 전략 |

</details>

---

## 🧪 실험 환경

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: pg_deep_dive
      POSTGRES_PASSWORD: postgres
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C.UTF-8"
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    command: >
      postgres
      -c shared_buffers=256MB
      -c effective_cache_size=1GB
      -c work_mem=16MB
      -c maintenance_work_mem=128MB
      -c log_min_duration_statement=100
      -c log_autovacuum_min_duration=0
      -c track_io_timing=on
      -c track_functions=all
      -c shared_preload_libraries='pg_stat_statements,auto_explain'
      -c pg_stat_statements.max=10000
      -c pg_stat_statements.track=all
      -c auto_explain.log_min_duration=1000

  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    environment:
      DATA_SOURCE_NAME: postgresql://postgres:postgres@postgres:5432/pg_deep_dive?sslmode=disable
    ports:
      - "9187:9187"

volumes:
  pgdata:
```

```bash
# 핵심 진단 쿼리 — 실험 시작 전 확인 세트

# Dead Tuple 현황
psql -c "
SELECT schemaname, tablename,
       n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_ratio,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;"

# 인덱스 사용률
psql -c "
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan;"

# 잠금 대기 분석
psql -c "
SELECT pid, now() - query_start AS duration, query, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE (now() - query_start) > interval '5 seconds';"

# XID Wraparound 위험도
psql -c "SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age DESC;"

# 튜플 가시성 확인 (pageinspect)
psql -c "CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT t_xmin, t_xmax, t_ctid, t_infomask
FROM heap_page_items(get_raw_page('your_table', 0));"
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 PostgreSQL에서 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — MySQL 방식 그대로 적용하거나 PostgreSQL 특성을 무시한 결과 |
| ✨ **올바른 접근** | After — PostgreSQL에 최적화된 설계/운영 방식 |
| 🔬 **내부 동작 원리** | 소스 레벨 분석 + 페이지 구조 ASCII 다이어그램 + 알고리즘 |
| 💻 **실전 실험** | `psql`, `EXPLAIN ANALYZE`, `pg_stat_*`, `pageinspect`로 바로 재현 가능한 실험 |
| 📊 **MySQL과 비교** | 같은 문제를 MySQL은 어떻게 다르게 해결하는가 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "VACUUM을 껐다가 테이블이 10배 부풀었다" — MVCC 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch1-03  PostgreSQL MVCC vs MySQL MVCC → Dead Tuple이 왜 생기는가
       Ch2-01  Dead Tuple 완전 분해 → Table Bloat 원인 이해
Day 2  Ch2-02  VACUUM 내부 동작 → FSM, Visibility Map, Index Vacuum
       Ch2-03  VACUUM FULL vs VACUUM → 운영 중 어떤 선택을 해야 하는가
Day 3  Ch2-04  Autovacuum 튜닝 → 대형 테이블 Autovacuum 따라잡기
       Ch7-04  운영 중 발생하는 문제 패턴 → Long Running Transaction이 VACUUM을 막는 구조
```

</details>

<details>
<summary><b>🟡 "JSONB에 GIN 인덱스를 걸지 않아 쿼리가 느리다" — 인덱스 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-02  Storage Manager와 페이지 구조 → 8KB 페이지 레이아웃
       Ch3-01  B-Tree 인덱스 심화 → Index-Only Scan과 Partial Index
Day 2  Ch4-01  TOAST 완전 분해 → 큰 값이 어떻게 저장되는가
       Ch4-02  JSONB 내부 저장 → 바이너리 저장과 키 정렬
Day 3  Ch3-04  GIN 인덱스 → JSONB 역색인과 Fastupdate
Day 4  Ch3-05  BRIN 인덱스 → 타임스탬프 컬럼 초경량 인덱스
       Ch3-07  인덱스 선택 가이드 → 쿼리 패턴별 결정 트리
Day 5  Ch3-08  실행 계획 분석 심화 → EXPLAIN ANALYZE 노드 해석
Day 6  Ch7-03  인덱스 사용률 분석 → 미사용 인덱스 제거, REINDEX CONCURRENTLY
Day 7  Ch7-02  쿼리 성능 진단 → pg_stat_statements로 병목 찾기
```

</details>

<details>
<summary><b>🔴 "PostgreSQL 소스코드까지 파고들어 내부를 완전히 정복한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — PostgreSQL 아키텍처와 MySQL과의 근본 차이
        → pageinspect로 Heap 페이지 직접 관찰, WAL 레코드 구조 분석

2주차  Chapter 2 전체 — MVCC 심화와 VACUUM
        → Dead Tuple 의도적 생성, VACUUM 전후 FSM 변화 관찰, XID Wraparound 시뮬레이션

3주차  Chapter 3 전체 — 인덱스 완전 분해
        → 각 인덱스 타입별 실제 크기 비교, EXPLAIN ANALYZE로 플래너 결정 관찰

4주차  Chapter 4 전체 — TOAST와 대용량 데이터
        → TOAST 테이블 직접 조회, JSONB GIN 인덱스 성능 비교, pg_bigm 한국어 전문 검색

5주차  Chapter 5 전체 — 고급 SQL과 분석 기능
        → 윈도우 함수 실행 계획, 재귀 CTE로 트리 순회, LATERAL JOIN 패턴

6주차  Chapter 6 전체 — 복제와 고가용성
        → Docker Compose로 Streaming Replication 구성, Patroni Failover 시뮬레이션

7주차  Chapter 7 전체 — 성능 튜닝과 운영
        → postgresql.conf 파라미터별 영향 측정, Long Running Transaction 재현
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [database-internals](https://github.com/dev-book-lab/database-internals) | B-Tree, MVCC, WAL 기초 이론 | Ch1-03(PostgreSQL MVCC 비교의 기초), Ch3-01(B-Tree 인덱스 심화) |
| [mysql-deep-dive](https://github.com/dev-book-lab/mysql-deep-dive) | MySQL InnoDB MVCC, Undo Log, B+Tree | Ch1-03(MVCC 구현 차이), 모든 📊 MySQL과 비교 섹션 |
| [linux-for-backend-deep-dive](https://github.com/dev-book-lab/linux-for-backend-deep-dive) | Page Cache, I/O, 프로세스 | Ch1-01(shared_buffers와 OS Page Cache 이중 캐시), Ch2-02(VACUUM의 fsync) |
| [spring-data-transaction](https://github.com/dev-book-lab/spring-data-transaction) | Spring `@Transactional`, 격리 수준 | Ch2-07(PostgreSQL 격리 수준과 SSI), Ch7-05(Spring + PostgreSQL 최적화) |
| [observability-deep-dive](https://github.com/dev-book-lab/observability-deep-dive) | Prometheus, Grafana, 메트릭 수집 | Ch7-02(postgres-exporter + pg_stat_statements 대시보드) |

> 💡 이 레포는 **PostgreSQL 내부 동작**에 집중합니다. MySQL을 모르더라도 Chapter 1~6을 순서대로 학습할 수 있습니다. 단, 📊 MySQL과 비교 섹션은 mysql-deep-dive를 먼저 학습하면 더 깊이 연결됩니다.

---

## 📚 Reference

- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/current/)
- [The Internals of PostgreSQL — Hironobu Suzuki](https://www.interdb.jp/pg/)
- [PostgreSQL 소스 코드 (GitHub)](https://github.com/postgres/postgres)
- [Use The Index, Luke](https://use-the-index-luke.com/) — SQL 인덱스 완전 분해
- [Cybertec PostgreSQL Blog](https://www.cybertec-postgresql.com/en/blog/)
- [pganalyze Blog](https://pganalyze.com/blog)
- [Bruce Momjian 발표자료](https://momjian.us/main/presentations/) — PostgreSQL 핵심 개발자

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"PostgreSQL도 SQL 쓰면 되는 거 아닌가 — 와 — Dead Tuple이 왜 생기고 VACUUM이 어떻게 회수하는지 아는 것은 다르다"*

</div>
