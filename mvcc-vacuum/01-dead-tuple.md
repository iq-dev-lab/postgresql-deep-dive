# Dead Tuple 완전 분해 — Table Bloat의 원인과 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Dead Tuple은 정확히 어떤 상태의 튜플인가? 언제 생성되는가?
- UPDATE/DELETE가 즉시 삭제하지 않고 Dead Tuple을 남기는 이유는?
- Dead Tuple이 쌓이면 쿼리 성능에 어떤 영향이 나타나는가?
- `pg_stat_user_tables`의 `n_dead_tup`은 어떻게 계산되는가?
- Table Bloat을 정량적으로 측정하는 방법은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL에서 UPDATE를 1만 번 실행하면 테이블 크기가 2배가 된다. DELETE를 1만 건 실행해도 테이블 파일은 줄어들지 않는다. 이유는 PostgreSQL의 MVCC 구현에 있다. 이전 버전의 튜플(Dead Tuple)을 즉시 삭제하지 않고 페이지에 그대로 남겨두기 때문이다. Dead Tuple이 페이지를 얼마나 채우느냐에 따라 SeqScan 비용이 직접 비례해서 증가한다. 이것이 VACUUM이 반드시 필요한 근본 이유다.

---

## 😱 흔한 실수 (Before — Dead Tuple 개념 없이 운영)

```
상황: 매일 새벽 배치로 30일 이상 된 orders 삭제
  DELETE FROM orders WHERE created_at < NOW() - INTERVAL '30 days';
  → 매일 약 10만 건 삭제

한 달 후:
  "orders 테이블 조회가 점점 느려진다"
  "인덱스도 있는데 왜 SeqScan으로 가는 거지?"

원인:
  10만 건 × 30일 = 300만 Dead Tuple이 페이지에 쌓임
  Live Tuple은 100만 건인데 Dead Tuple이 300만 건
  → dead_ratio = 75%
  → 테이블 전체 스캔 시 읽어야 할 페이지가 4배
  → 인덱스가 있어도 플래너가 SeqScan 선택 가능
    (n_dead_tup이 통계에 반영되어 비용 추정 왜곡)

또 다른 상황:
  UPDATE users SET last_login = NOW() WHERE id = ?;
  → 초당 500건, 매일 4,300만 건 UPDATE

  VACUUM이 따라오지 못하면:
  → 테이블 크기 매일 수십 GB 증가
  → 결국 디스크 풀
```

---

## ✨ 올바른 접근 (After — Dead Tuple 모니터링 기반 운영)

```
Dead Tuple 현황 상시 모니터링:

  SELECT
      schemaname,
      relname,
      n_live_tup,
      n_dead_tup,
      round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_ratio,
      last_autovacuum,
      last_autoanalyze,
      pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS total_size
  FROM pg_stat_user_tables
  WHERE n_dead_tup > 0
  ORDER BY n_dead_tup DESC
  LIMIT 20;

  dead_ratio > 20% → Autovacuum 설정 재검토
  dead_ratio > 50% → 즉시 수동 VACUUM
  last_autovacuum이 수 시간 전 → Autovacuum이 따라오는 중

배치 DELETE 전략:
  -- 한 번에 전체 삭제 대신 청크 단위 삭제
  DO $$
  DECLARE
    deleted INT;
  BEGIN
    LOOP
      DELETE FROM orders
      WHERE id IN (
        SELECT id FROM orders
        WHERE created_at < NOW() - INTERVAL '30 days'
        LIMIT 1000
      );
      GET DIAGNOSTICS deleted = ROW_COUNT;
      EXIT WHEN deleted = 0;
      PERFORM pg_sleep(0.1);  -- VACUUM이 따라올 시간
    END LOOP;
  END $$;
```

---

## 🔬 내부 동작 원리

### 1. Dead Tuple 생성 과정

```
=== INSERT ===

INSERT INTO orders (id, status) VALUES (1, 'PENDING');
→ Heap 페이지에 새 튜플 삽입
→ t_xmin = 현재 트랜잭션 XID, t_xmax = 0

Heap Page:
┌────────────────────────────────────────┐
│ ItemId[1] → Tuple 1                   │
│   t_xmin=500, t_xmax=0               │ ← ALIVE
│   status='PENDING'                    │
└────────────────────────────────────────┘

=== DELETE ===

DELETE FROM orders WHERE id = 1;
(트랜잭션 XID=501, 이후 커밋)

After DELETE + COMMIT:
┌────────────────────────────────────────┐
│ ItemId[1] → Tuple 1                   │
│   t_xmin=500, t_xmax=501             │ ← DEAD (xmax 커밋됨)
│   status='PENDING'                    │ ← 물리적으로 여전히 존재
└────────────────────────────────────────┘

→ 물리적 삭제 없음! Dead Tuple로만 표시됨

=== UPDATE ===

UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
(트랜잭션 XID=502, 이후 커밋)

After UPDATE + COMMIT:
┌────────────────────────────────────────┐
│ ItemId[1] → Tuple 1 (OLD)             │
│   t_xmin=500, t_xmax=502             │ ← DEAD
│   status='PENDING'                    │

│ ItemId[2] → Tuple 2 (NEW)             │
│   t_xmin=502, t_xmax=0               │ ← ALIVE
│   status='SHIPPED'                    │
└────────────────────────────────────────┘

→ OLD 버전은 Dead Tuple, NEW 버전이 공존

"Dead Tuple"의 정확한 정의:
  t_xmax가 설정되어 있고, 그 트랜잭션이 커밋된 튜플
  (t_xmax 설정 + 커밋 = 어떤 새 트랜잭션도 이 버전을 볼 수 없음)

  "아직 삭제하지 않은 이유":
  Long Running Transaction이 있으면 이 Dead Tuple을 참조할 수 있음
  → 모든 트랜잭션이 더 이상 필요 없을 때까지 유지
  → OldestXmin: 현재 활성 트랜잭션 중 가장 오래된 xmin
  → t_xmax < OldestXmin → 회수 가능
```

### 2. Dead Tuple이 성능에 미치는 영향

```
SeqScan 비용 계산:

  Dead Tuple 없을 때:
  Table: 1,000 pages (Live 10,000 tuples, 10/page)
  SeqScan: 1,000 page reads

  Dead Tuple 50% 일 때:
  Table: 2,000 pages (Live 10,000 + Dead 10,000 tuples, 10/page)
  SeqScan: 2,000 page reads → 2배 비용

  Dead Tuple 90%:
  Table: 10,000 pages (Live 10,000 + Dead 90,000 tuples)
  SeqScan: 10,000 page reads → 10배 비용

Index Scan에도 영향:
  인덱스 → ItemId → Heap Page Fetch → Dead 확인 → 버려짐
  → Index Scan이 많은 페이지를 읽고도 결과 없는 경우 발생

  예시: status='DONE' 인덱스 스캔
  인덱스에 1만 개 엔트리 → Heap Fetch → 9천 개가 Dead
  → 9천 번의 헛된 I/O

플래너 통계 왜곡:
  pg_statistic (ANALYZE 결과)에는 Dead Tuple 포함된 n_live_tup 기준 통계
  실제 페이지 수(relpages)는 Dead 포함 크기
  → 플래너가 비용을 잘못 추정할 수 있음
  → ANALYZE 주기도 Autovacuum과 맞춰야 함

페이지 캐시 효율 저하:
  shared_buffers에 Dead Tuple을 포함한 페이지가 캐싱됨
  → 유효한 데이터보다 Dead Tuple 캐싱에 메모리 낭비
```

### 3. n_dead_tup 카운터의 작동 방식

```
pg_stat_user_tables의 n_dead_tup:

  정확한 값이 아닌 추정치:
  UPDATE/DELETE 시마다 카운터 증가
  VACUUM 후 카운터 감소 (실제 회수된 만큼)

  카운터 증가 시점:
    UPDATE → n_dead_tup += 1 (OLD 버전)
    DELETE → n_dead_tup += 1
    롤백 → 카운터 조정 없음 (약간 부정확할 수 있음)

  카운터 감소 시점:
    VACUUM/Autovacuum 완료 후 감소

  주의: Dead Tuple이 0이 아니어도 즉시 VACUUM 불필요
  → Autovacuum이 threshold를 넘을 때 자동 처리
  → threshold = autovacuum_vacuum_threshold(50)
               + autovacuum_vacuum_scale_factor(0.2) × n_live_tup

  확인 쿼리:
  SELECT
      relname,
      n_live_tup,
      n_dead_tup,
      n_mod_since_analyze,
      last_autovacuum::date,
      last_autoanalyze::date
  FROM pg_stat_user_tables
  WHERE relname = 'orders';
```

### 4. Table Bloat 정량 측정

```
방법 1: pg_stat_user_tables 활용 (추정)

  SELECT
      relname,
      pg_size_pretty(pg_relation_size(schemaname||'.'||relname)) AS heap_size,
      n_live_tup,
      n_dead_tup,
      round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_ratio,
      -- 예상 낭비 공간 (rough estimate)
      pg_size_pretty(
          pg_relation_size(schemaname||'.'||relname)
          * n_dead_tup / nullif(n_live_tup + n_dead_tup, 1)
      ) AS estimated_bloat
  FROM pg_stat_user_tables
  ORDER BY n_dead_tup DESC;

방법 2: pgstattuple 확장 (정확)

  CREATE EXTENSION pgstattuple;

  SELECT * FROM pgstattuple('orders');
  -- 출력:
  -- table_len         : 전체 테이블 바이트
  -- tuple_count       : Live Tuple 수
  -- tuple_len         : Live Tuple 총 바이트
  -- tuple_percent     : Live Tuple 점유율
  -- dead_tuple_count  : Dead Tuple 수
  -- dead_tuple_len    : Dead Tuple 총 바이트
  -- dead_tuple_percent: Dead Tuple 점유율 ← 이게 bloat 비율
  -- free_space        : 사용 가능한 여유 공간
  -- free_percent      : 여유 공간 점유율

방법 3: bloat 쿼리 (pgBadger/check_postgres 스타일)

  -- 실제 예상 행 크기 vs 실제 테이블 크기 비교
  SELECT
      schemaname,
      tablename,
      pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS actual_size,
      pg_size_pretty(
          (SELECT reltuples * avg_width
           FROM pg_stats s
           JOIN pg_class c ON c.relname = s.tablename
           WHERE s.tablename = pg_tables.tablename
           GROUP BY reltuples, avg_width
           LIMIT 1)::bigint
      ) AS estimated_data_size
  FROM pg_tables
  WHERE schemaname = 'public';
```

---

## 💻 실전 실험

### 실험 1: Dead Tuple 발생과 Table Bloat 관찰

```sql
-- 실험 테이블
CREATE TABLE bloat_test (
    id SERIAL PRIMARY KEY,
    status TEXT DEFAULT 'PENDING',
    payload TEXT DEFAULT repeat('x', 100)
);

-- 초기 데이터 10,000건
INSERT INTO bloat_test (status)
SELECT 'PENDING' FROM generate_series(1, 10000);

-- 초기 크기와 Dead Tuple
SELECT
    pg_size_pretty(pg_relation_size('bloat_test')) AS size,
    n_live_tup, n_dead_tup
FROM pg_stat_user_tables WHERE relname = 'bloat_test';

-- UPDATE 10,000건 (Dead Tuple 생성)
UPDATE bloat_test SET status = 'PROCESSING';

-- 크기와 Dead Tuple 변화
SELECT
    pg_size_pretty(pg_relation_size('bloat_test')) AS size,
    n_live_tup, n_dead_tup
FROM pg_stat_user_tables WHERE relname = 'bloat_test';
-- size: 약 2배, n_dead_tup: ~10,000

-- pgstattuple로 정확한 bloat 측정
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT
    pg_size_pretty(table_len) AS table_size,
    tuple_count AS live_tuples,
    dead_tuple_count AS dead_tuples,
    round(dead_tuple_percent, 1) AS dead_pct,
    round(free_percent, 1) AS free_pct
FROM pgstattuple('bloat_test');
```

### 실험 2: Dead Tuple이 SeqScan 비용에 미치는 영향

```sql
-- EXPLAIN ANALYZE로 비용 측정
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT count(*) FROM bloat_test WHERE status = 'PROCESSING';
-- Seq Scan: rows=10000 (실제), Pages Read: 많음

-- VACUUM 후 비용 재측정
VACUUM bloat_test;

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT count(*) FROM bloat_test WHERE status = 'PROCESSING';
-- 같은 쿼리인데 Buffers hit 감소 (Dead Tuple 공간 재사용 가능 표시됨)

-- 파이지 비교
SELECT
    pg_size_pretty(pg_relation_size('bloat_test')) AS size_after_vacuum,
    n_live_tup, n_dead_tup
FROM pg_stat_user_tables WHERE relname = 'bloat_test';
-- n_dead_tup = 0, 하지만 size는 그대로 (OS 반환 안 함)
```

### 실험 3: 장기 트랜잭션이 Dead Tuple 청소를 막는 실험

```sql
-- 터미널 1: 장기 트랜잭션 열기
BEGIN;
SELECT count(*) FROM bloat_test; -- 스냅샷 획득

-- 터미널 2: UPDATE + VACUUM
UPDATE bloat_test SET status = 'DONE';
VACUUM VERBOSE bloat_test;
-- 출력에서: "0 dead row versions cannot be removed yet"
-- Dead Tuple이 있지만 터미널1 트랜잭션 때문에 회수 불가

SELECT n_dead_tup FROM pg_stat_user_tables WHERE relname = 'bloat_test';
-- n_dead_tup > 0 (VACUUM 했는데도!)

-- 터미널 1: 트랜잭션 종료
ROLLBACK;

-- 터미널 2: 다시 VACUUM
VACUUM VERBOSE bloat_test;
-- 이제 Dead Tuple 회수됨
SELECT n_dead_tup FROM pg_stat_user_tables WHERE relname = 'bloat_test';
-- n_dead_tup = 0
```

---

## 📊 MySQL과 비교

```
동일 워크로드: UPDATE 10,000건 실행

PostgreSQL:
  Heap 페이지: OLD 버전(Dead) + NEW 버전 공존
  테이블 크기: 약 2배 증가
  별도 Undo 영역: 없음
  정리 방법: VACUUM (명시적 실행 또는 Autovacuum)

MySQL InnoDB:
  Clustered Index 페이지: 최신 버전만
  테이블 크기: 변화 없음 (Undo Log에 이전 버전 저장)
  Undo Tablespace: 이전 버전으로 증가할 수 있음
  정리 방법: Purge Thread 자동 처리

어느 방식이 더 나은가:

  읽기 성능:
    PostgreSQL: dead_ratio가 낮을 때 유사, 높을 때 MySQL 유리
    MySQL: 항상 최신 버전만 → 일관된 읽기 성능

  쓰기 성능:
    PostgreSQL: Heap에만 쓰기 (단일 위치)
    MySQL: 페이지 수정 + Undo Log (이중 쓰기)

  운영 복잡도:
    PostgreSQL: Autovacuum 튜닝, Table Bloat 모니터링 필요
    MySQL: Purge Thread 자동, Undo Log 크기 모니터링 필요

  결론: 둘 다 MVCC의 "비용"을 다른 방식으로 지불
        PostgreSQL: 공간(Table Bloat) + VACUUM 운영 비용
        MySQL: CPU(Purge) + Undo 공간
```

---

## ⚖️ 트레이드오프

```
Dead Tuple 즉시 삭제하지 않는 이유:

장점:
  ① 잠금 없는 읽기:
     이전 버전이 Heap에 있어 다른 트랜잭션이 잠금 없이 읽기 가능
  ② 빠른 쓰기:
     추가 위치(Undo Log)에 쓸 필요 없음
  ③ 롤백 비용 없음:
     물리적 undo 없이 xmax 설정 취소만으로 롤백 완료

단점:
  ① Table Bloat:
     VACUUM 없이 테이블 크기 무한 증가
  ② VACUUM 필요:
     명시적 실행 또는 Autovacuum 튜닝 필요
  ③ 장기 트랜잭션 부작용:
     Dead Tuple 정리 차단 → 더 큰 Bloat

Dead Tuple 관리 원칙:
  ① dead_ratio > 20%: 경계
  ② Autovacuum이 처리: 대부분의 경우 자동 관리
  ③ Autovacuum이 못 따라갈 때: 테이블별 파라미터 튜닝
  ④ 청크 DELETE: 한 번에 대량 삭제 대신 소량씩
  ⑤ fillfactor: UPDATE 빈번한 테이블에 HOT Update 유도
```

---

## 📌 핵심 정리

```
Dead Tuple 핵심:

생성 조건:
  DELETE: 튜플의 t_xmax에 트랜잭션 XID 설정
  UPDATE: OLD 버전의 t_xmax 설정 + NEW 버전 삽입
  커밋 완료 후 → Dead Tuple

물리적 위치:
  Heap 페이지에 그대로 존재 (즉시 삭제 안 함)
  MVCC: 다른 트랜잭션이 이전 버전을 참조할 수 있으므로

회수 조건:
  t_xmax < OldestXmin (모든 활성 트랜잭션보다 오래됨)
  → VACUUM이 재사용 가능으로 표시

성능 영향:
  dead_ratio에 비례해 SeqScan 비용 증가
  Index Scan도 헛된 Heap Fetch 증가
  50% 이상이면 쿼리 성능 심각 저하

모니터링:
  SELECT n_dead_tup, n_live_tup FROM pg_stat_user_tables;
  CREATE EXTENSION pgstattuple;
  SELECT dead_tuple_percent FROM pgstattuple('table_name');
```

---

## 🤔 생각해볼 문제

**Q1.** `DELETE FROM orders`로 전체 삭제 후 `SELECT count(*) FROM orders`가 0을 반환한다. Dead Tuple은 얼마나 남아있는가? 이를 즉시 회수하는 방법은?

<details>
<summary>해설 보기</summary>

`DELETE FROM orders`는 전체 행을 Dead Tuple로 만들지만 물리적으로 삭제하지 않습니다. Live Tuple = 0이고 Dead Tuple = 기존 전체 행 수입니다. 테이블 파일 크기는 그대로입니다.

회수 방법:
- `VACUUM orders;` → Dead Tuple 공간을 재사용 가능으로 표시 (크기 유지)
- `VACUUM FULL orders;` → 파일 크기 축소 (AccessExclusiveLock 필요)
- `TRUNCATE orders;` → 테이블 파일을 처음부터 다시 생성 (가장 빠름, 단 조건 없이 전체 삭제)

전체 행 삭제 시 `TRUNCATE`가 `DELETE + VACUUM FULL`보다 빠르고 락 시간도 짧습니다.

```sql
-- TRUNCATE: 파일 자체를 교체 → 즉시 공간 회수
TRUNCATE orders;

-- vs DELETE: Dead Tuple 생성 후 VACUUM 필요
DELETE FROM orders;
VACUUM FULL orders; -- AccessExclusiveLock 필요
```

</details>

---

**Q2.** 초당 1,000건의 UPDATE가 발생하는 테이블에서 Autovacuum 기본 설정(scale_factor=0.2)으로 충분한가? 어떻게 계산하는가?

<details>
<summary>해설 보기</summary>

Autovacuum 실행 조건:
```
threshold = autovacuum_vacuum_threshold(50)
          + autovacuum_vacuum_scale_factor(0.2) × n_live_tup
```

예: 테이블에 100만 Live Tuple
- threshold = 50 + 0.2 × 1,000,000 = **200,050 Dead Tuple** 이상일 때 Autovacuum 실행

초당 1,000 UPDATE면 200,050 Dead Tuple 생성까지 **약 200초(3.3분)**마다 Autovacuum 실행됩니다.

Autovacuum이 200초마다 20만 건을 처리해야 합니다. `autovacuum_vacuum_cost_limit`(기본 200)과 `vacuum_cost_page_dirty`(기본 20) 기준으로 초당 약 10 페이지 처리 → 심각한 경우 따라오지 못합니다.

해결:
```sql
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- 1%로 낮춤 → 10,000건마다 실행
    autovacuum_vacuum_cost_limit = 1000      -- 더 빠르게 처리
);
```

</details>

---

**Q3.** `n_dead_tup`과 `pgstattuple`의 `dead_tuple_count`가 다른 값을 보여준다. 어느 것이 정확한가?

<details>
<summary>해설 보기</summary>

`pgstattuple`이 더 정확합니다. `n_dead_tup`은 테이블 통계 카운터로, 실제 카운트가 아닌 **추정치**입니다. UPDATE/DELETE 시 증가하지만 롤백 시 정확하게 감소하지 않을 수 있습니다. 또한 오래된 Dead Tuple이 힌트 비트 업데이트 없이 남아있으면 오차가 발생합니다.

`pgstattuple`은 테이블 파일을 실제로 스캔하여 각 튜플의 `t_xmin`, `t_xmax`와 CLOG를 직접 확인합니다. 정확하지만 **테이블 전체 읽기** 비용이 발생하므로 대형 테이블에서는 `pgstattuple_approx` 사용을 권장합니다.

```sql
-- 정확하지만 느림
SELECT dead_tuple_count FROM pgstattuple('orders');

-- 빠른 근사치
SELECT dead_tuple_count FROM pgstattuple_approx('orders');
```

</details>

---

<div align="center">

**[⬅️ 이전 챕터: XID와 가시성 규칙](../postgresql-architecture/05-xid-visibility.md)** | **[홈으로 🏠](../README.md)** | **[다음: VACUUM 내부 동작 ➡️](./02-vacuum-internals.md)**

</div>
