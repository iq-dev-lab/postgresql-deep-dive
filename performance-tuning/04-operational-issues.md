# 운영 중 발생하는 문제 패턴 — Long Transaction부터 Checkpoint 과부하까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Long Running Transaction이 VACUUM을 막는 정확한 메커니즘은?
- Autovacuum이 따라오지 못할 때 어떻게 진단하는가?
- 테이블 잠금 대기로 인한 연결 누적은 어떻게 해결하는가?
- Checkpoint 과부하는 어떻게 감지하고 조정하는가?
- `pg_cancel_backend`와 `pg_terminate_backend`의 차이와 사용 시점은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

운영 중에는 예상치 못한 문제들이 발생한다. Long Running Transaction이 VACUUM을 차단해 Table Bloat이 급격히 증가하고, ALTER TABLE 하나가 연결 수백 개를 순식간에 차단하며, Checkpoint 과부하가 쿼리 지연을 유발한다. 이런 문제들은 패턴이 있어서 알고 있으면 빠르게 진단하고 해결할 수 있다. 이 문서는 운영 현장에서 가장 자주 만나는 PostgreSQL 문제 패턴과 해결 방법을 다룬다.

---

## 😱 흔한 실수 (Before — 문제 패턴 인지 못함)

```
실수 1: Long Transaction이 VACUUM을 막는 걸 모름

  상황: 특정 테이블이 갑자기 수십 GB로 증가
  "VACUUM ANALYZE 실행했는데 크기가 안 줄어들어"
  → Long Running Transaction이 OldestXmin을 낮게 고정
  → VACUUM이 Dead Tuple 회수 불가

  확인:
  SELECT pid, now() - xact_start AS duration, state, query
  FROM pg_stat_activity
  WHERE backend_xmin IS NOT NULL
  ORDER BY age(backend_xmin) DESC;
  → 8시간짜리 idle in transaction 발견!
  → 해당 연결 종료 → VACUUM 재실행 → 즉시 해결

실수 2: ALTER TABLE이 연결 폭탄을 만드는 걸 모름

  "새벽 2시에 ALTER TABLE로 컬럼 추가하면 괜찮겠지"

  실제 상황:
  ALTER TABLE orders ADD COLUMN note TEXT;
  → AccessExclusiveLock 획득 시도 → 기존 트랜잭션 대기
  → 그 뒤로 오는 모든 SELECT/UPDATE도 대기
  → 연결 수백 개가 Lock 대기 누적
  → "max_connections exceeded" 오류 → 서비스 다운

  올바른 접근:
  lock_timeout 설정 + 트랜잭션이 없는 시간대 선택

실수 3: Checkpoint 과부하 징후 모름

  PostgreSQL 로그:
  LOG: checkpoint starting: time
  LOG: checkpoint complete: wrote 15000 buffers (91.5%); ...
  LOG: checkpoint request for checkpoint of type "time" taking 45.2 s

  → Checkpoint가 45초 걸림 → checkpoint_warning(기본 30초) 초과
  → I/O 스파이크 → 쿼리 지연
  → max_wal_size 증가 + checkpoint_completion_target 조정 필요
```

---

## ✨ 올바른 접근 (After — 패턴별 즉시 대응)

```
운영 대응 체크리스트:

  [DB 응답 느림]
  1. pg_stat_activity → 오래 실행 중인 쿼리/Lock 대기 확인
  2. pg_blocking_pids() → 잠금 체인 확인
  3. 원인 쿼리 pg_cancel_backend() 또는 pg_terminate_backend()

  [테이블 급격히 커짐]
  1. pg_stat_user_tables → n_dead_tup 확인
  2. pg_stat_activity → backend_xmin 오래된 연결 확인
  3. OldestXmin 차단 연결 종료
  4. VACUUM 재실행

  [연결 수 급증]
  1. pg_stat_activity → 대기 상태 쿼리 파악
  2. 잠금 대기 체인 확인
  3. 원인 잠금 보유자 취소/종료
  4. idle_in_transaction_session_timeout 설정 (예방)

  [Checkpoint 경고]
  max_wal_size 증가 (더 긴 Checkpoint 간격)
  checkpoint_completion_target = 0.9 (I/O 분산)
```

---

## 🔬 내부 동작 원리

### 1. Long Running Transaction과 VACUUM 차단

```
VACUUM OldestXmin 메커니즘:

  VACUUM이 Dead Tuple을 회수하려면:
  Dead Tuple의 t_xmax < OldestXmin (모든 활성 트랜잭션의 xmin 중 최솟값)

  Long Running Transaction 효과:
  트랜잭션 A (8시간 전 시작, XID=1000):
    → backend_xmin = 1000 (이 트랜잭션의 스냅샷 xmin)

  OldestXmin = min(현재 모든 backend_xmin) = 1000
  → t_xmax < 1000인 Dead Tuple만 회수 가능
  → 8시간 동안 발생한 Dead Tuple은 회수 불가!

  확인 쿼리:
  SELECT
      pid,
      usename,
      application_name,
      now() - xact_start AS transaction_duration,
      age(backend_xmin) AS xmin_age,
      state,
      wait_event_type,
      left(query, 80) AS query
  FROM pg_stat_activity
  WHERE backend_xmin IS NOT NULL
  ORDER BY age(backend_xmin) DESC
  LIMIT 10;

  xmin_age가 수백만 이상: 심각한 VACUUM 차단

  자동 방지 설정:
  idle_in_transaction_session_timeout = '10min'
  → BEGIN 후 10분 idle이면 자동 종료
  → xact_start 기준 아님, state가 'idle in transaction'인 시간 기준

  또는 statement_timeout으로 쿼리 자체 시간 제한:
  statement_timeout = '30s'  (각 쿼리 최대 30초)
```

### 2. Autovacuum이 따라오지 못하는 경우

```
Autovacuum 처리 용량 계산:

  autovacuum_vacuum_cost_limit = 200 (기본)
  vacuum_cost_page_dirty = 20
  → 초당 처리 가능 페이지: 200/20 = 10 pages
  autovacuum_vacuum_cost_delay = 2ms
  → 실제 처리 속도: 10 pages / (1 + 0.002s) ≈ 5,000 pages/min

  초당 UPDATE 1000건 테이블:
  → 초당 Dead Tuple 1000개, 약 100 pages
  → Autovacuum 처리 속도: 5,000 pages/min = 83 pages/s
  → 처리 가능!

  BUT 초당 UPDATE 10,000건이면:
  → 초당 Dead Tuple 10,000개 ≈ 1000 pages
  → Autovacuum이 따라오지 못함

  진단:
  -- Autovacuum이 실행 중인지 확인
  SELECT pid, now() - xact_start AS duration, query
  FROM pg_stat_activity
  WHERE query LIKE 'autovacuum:%';

  -- Autovacuum 실행 이력 (마지막 실행 시각)
  SELECT relname, last_autovacuum, n_dead_tup, n_live_tup
  FROM pg_stat_user_tables
  WHERE n_dead_tup > 0
  ORDER BY n_dead_tup DESC LIMIT 10;

  해결:
  -- 고빈도 UPDATE 테이블에 Autovacuum 튜닝
  ALTER TABLE hot_table SET (
      autovacuum_vacuum_scale_factor = 0.01,
      autovacuum_vacuum_cost_limit = 1000,
      autovacuum_vacuum_cost_delay = 0
  );

  -- 또는 수동 VACUUM (즉각 처리)
  VACUUM (VERBOSE, ANALYZE) hot_table;
```

### 3. 테이블 잠금 대기 연결 누적

```
잠금 대기 폭발 시나리오:

  T=0: ALTER TABLE orders ADD COLUMN ... (AccessExclusiveLock 요청)
  T=0: 기존 트랜잭션 A가 orders에 활성 쿼리 → ALTER 대기
  T=1: 새 SELECT * FROM orders → ALTER 대기 (A보다 늦게 도착)
  T=2: 또 다른 SELECT → 대기
  ...
  T=60: 연결 1000개 대기 → max_connections 초과 → 장애

  문제의 핵심:
  AccessExclusiveLock이 대기 중이면
  그 뒤로 오는 모든 접근이 차단됨 (이미 잠금 보유한 것도 새 잠금 취득 대기)
  → Lock Queue: A가 Lock 대기 → A 뒤로 온 B, C도 대기

  즉각 진단:
  SELECT
      blocking_pids,
      query AS waiting_query,
      pid AS waiting_pid
  FROM (
      SELECT
          pid,
          pg_blocking_pids(pid) AS blocking_pids,
          query
      FROM pg_stat_activity
      WHERE wait_event_type = 'Lock'
  ) t
  WHERE array_length(blocking_pids, 1) > 0
  ORDER BY array_length(blocking_pids, 1) DESC;

  해결 (잠금 보유자 취소):
  SELECT pg_cancel_backend(blocking_pid);
  -- 또는 강제 종료:
  SELECT pg_terminate_backend(blocking_pid);

  예방:
  -- DDL 실행 전 lock_timeout 설정
  SET lock_timeout = '5s';  -- 5초 내 Lock 못 얻으면 오류 (다음에 재시도)
  ALTER TABLE orders ADD COLUMN note TEXT;
  RESET lock_timeout;

  -- 또는 온라인 DDL 방식 (PostgreSQL ADD COLUMN NOT NULL DEFAULT)
  -- PostgreSQL 11+: NOT NULL DEFAULT는 즉각 처리 (테이블 재작성 없음)
  ALTER TABLE orders ADD COLUMN note TEXT NOT NULL DEFAULT '';
  -- AccessExclusiveLock 시간 최소화 (메타데이터 변경만)
```

### 4. Checkpoint 과부하 진단과 조정

```
Checkpoint 과부하 징후:

  로그 경고:
  LOG: checkpoint starting: time
  LOG: checkpoint complete: wrote 15000 buffers (91.5%); ...
  LOG: checkpoint request for checkpoint of type "time" taking 45.2 s
  → checkpoint_warning (기본 30초) 초과 → 경고

  pg_stat_bgwriter 분석:
  SELECT
      checkpoints_timed,
      checkpoints_req,            -- 강제 Checkpoint (max_wal_size 초과 시)
      buffers_checkpoint,         -- Checkpoint가 쓴 버퍼 수
      buffers_clean,              -- Background Writer가 쓴 버퍼 수
      buffers_backend,            -- Backend가 직접 쓴 버퍼 수 (많으면 문제!)
      round(buffers_checkpoint::numeric / (checkpoints_timed + checkpoints_req), 1) AS avg_buffers
  FROM pg_stat_bgwriter;

  문제 신호:
  checkpoints_req > checkpoints_timed: max_wal_size 너무 작음 → 강제 Checkpoint 多
  buffers_backend > 0 자주 발생: 공유 버퍼 부족 → shared_buffers 증가

  조정:
  -- Checkpoint 간격 늘리기 (강제 Checkpoint 줄이기)
  max_wal_size = 4GB  (기본 1GB)

  -- I/O 분산
  checkpoint_completion_target = 0.9  (기본 0.5)

  -- Background Writer 적극 활용
  bgwriter_lru_maxpages = 200  (기본 100, 더 많은 페이지 미리 기록)
  bgwriter_delay = 50ms        (기본 200ms, 더 자주 실행)

  -- 결과 확인 (통계 초기화 후)
  SELECT pg_stat_reset_shared('bgwriter');
  -- 재측정
```

---

## 💻 실전 실험

### 실험 1: Long Transaction VACUUM 차단 재현

```sql
-- 터미널 1: Long Transaction 시작
BEGIN;
SELECT count(*) FROM pg_class;  -- 스냅샷 획득 및 고정

-- 터미널 2:
-- Dead Tuple 생성
UPDATE pg_class SET relkind = relkind;  -- (권한 있는 테이블 사용)
VACUUM VERBOSE;  -- Dead Tuple 회수 시도

-- Long Transaction이 OldestXmin을 낮추어 VACUUM 차단 확인
SELECT
    pid,
    now() - xact_start AS duration,
    age(backend_xmin) AS xmin_age
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC;

-- 터미널 1: Long Transaction 종료
ROLLBACK;

-- 터미널 2: VACUUM 재시도 → 이제 Dead Tuple 회수 가능
VACUUM VERBOSE;
```

### 실험 2: 잠금 체인 분석

```sql
-- 실험 테이블
CREATE TABLE lock_chain_test (id INT PRIMARY KEY, val TEXT);
INSERT INTO lock_chain_test VALUES (1, 'data');

-- 터미널 1: 잠금 보유
BEGIN;
LOCK TABLE lock_chain_test IN ACCESS EXCLUSIVE MODE;

-- 터미널 2: 대기 (SELECT도 차단됨)
SELECT * FROM lock_chain_test;  -- 대기

-- 터미널 3: 잠금 체인 분석
SELECT
    a.pid AS waiting_pid,
    left(a.query, 50) AS waiting_query,
    a.wait_event_type,
    a.wait_event,
    pg_blocking_pids(a.pid) AS blocking_pids
FROM pg_stat_activity a
WHERE a.wait_event_type = 'Lock'
  OR a.pid = ANY(SELECT unnest(pg_blocking_pids(a2.pid))
                 FROM pg_stat_activity a2 WHERE a2.wait_event_type = 'Lock');

-- 터미널 1: 잠금 해제
ROLLBACK;
```

### 실험 3: Checkpoint 상태 모니터링

```sql
-- Checkpoint 통계 초기화
SELECT pg_stat_reset_shared('bgwriter');

-- 대량 쓰기 후 통계 확인
SELECT
    checkpoints_timed,
    checkpoints_req,
    round(buffers_checkpoint * 8.0 / 1024, 2) AS checkpoint_written_mb,
    round(buffers_clean * 8.0 / 1024, 2) AS bgwriter_written_mb,
    round(buffers_backend * 8.0 / 1024, 2) AS backend_written_mb,
    CASE WHEN checkpoints_req > checkpoints_timed
         THEN '⚠️ max_wal_size 증가 필요'
         ELSE '🟢 정상'
    END AS checkpoint_status
FROM pg_stat_bgwriter;

-- 현재 Checkpoint 정보
SELECT
    checkpoint_lsn,
    checkpoint_time,
    now() - checkpoint_time AS time_since_checkpoint,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), checkpoint_lsn)) AS wal_since_checkpoint
FROM pg_control_checkpoint();
```

---

## 📊 MySQL과 비교

```
MySQL vs PostgreSQL 운영 문제 패턴:

Long Transaction:
  MySQL: 긴 트랜잭션 → Undo Log 증가 (Purge 차단)
  PostgreSQL: 긴 트랜잭션 → Dead Tuple 회수 차단 (VACUUM 차단)
  둘 다: 긴 트랜잭션은 위험 → idle_in_transaction_session_timeout 설정

잠금 문제:
  MySQL: SHOW ENGINE INNODB STATUS → 잠금 정보 (텍스트 파싱 어려움)
  PostgreSQL: pg_locks + pg_blocking_pids() → SQL로 체계적 분석

Checkpoint 과부하:
  MySQL: InnoDB Buffer Pool Flush → innodb_io_capacity 조정
  PostgreSQL: Dirty Page Flush → checkpoint_completion_target, max_wal_size 조정

연결 과부하:
  MySQL: max_connections + Thread Pool (Enterprise)
  PostgreSQL: max_connections + PgBouncer (필수)

모니터링 용이성:
  MySQL: performance_schema (복잡, 별도 설정)
  PostgreSQL: pg_stat_* 뷰 (간단, SQL)
```

---

## ⚖️ 트레이드오프

```
pg_cancel_backend vs pg_terminate_backend:

pg_cancel_backend(pid):
  → SIGINT 전송 → 현재 쿼리만 취소
  → 연결은 유지됨
  → 실행 중인 트랜잭션은 롤백됨
  → 적합: 특정 쿼리만 취소, 연결은 유지

pg_terminate_backend(pid):
  → SIGTERM 전송 → 연결 강제 종료
  → 진행 중인 트랜잭션 자동 롤백
  → 클라이언트에서 "connection terminated" 오류
  → 적합: 연결 자체를 제거해야 할 때

선택 원칙:
  ① 먼저 pg_cancel_backend 시도
  ② 응답 없으면 pg_terminate_backend
  ③ 잠금 보유자 제거: pg_cancel 후 빠르게 확인

lock_timeout vs statement_timeout:
  lock_timeout: Lock 대기 최대 시간 (Lock을 못 얻으면 실패)
  statement_timeout: 전체 쿼리 최대 실행 시간

  DDL 실행 시:
  SET lock_timeout = '5s';  -- 5초 내 Lock 못 얻으면 실패 (재시도)
  SET statement_timeout = 0;  -- DDL 실행 시간 제한 없음
```

---

## 📌 핵심 정리

```
운영 문제 패턴 핵심:

Long Transaction → VACUUM 차단:
  pg_stat_activity.backend_xmin이 오래된 연결 찾기
  age(backend_xmin) 높은 연결 → pg_terminate_backend
  예방: idle_in_transaction_session_timeout = '10min'

Autovacuum 따라오지 못함:
  n_dead_tup 증가 + last_autovacuum 오래됨
  → ALTER TABLE SET (autovacuum_vacuum_scale_factor = 0.01, ...)
  → 또는 수동 VACUUM

잠금 대기 폭발:
  pg_blocking_pids()로 잠금 체인 확인
  원인 쿼리 pg_cancel_backend()
  예방: lock_timeout = '5s' + DDL 시간 주의

Checkpoint 과부하:
  checkpoints_req > checkpoints_timed → max_wal_size 증가
  로그에 Checkpoint 경고 → checkpoint_completion_target = 0.9
  buffers_backend 많음 → shared_buffers 증가

도구:
  pg_cancel_backend(pid): 현재 쿼리만 취소 (연결 유지)
  pg_terminate_backend(pid): 연결 강제 종료
```

---

## 🤔 생각해볼 문제

**Q1.** `lock_timeout = '5s'`를 설정한 후 DDL을 실행했는데 오류가 발생했다. 이 오류가 데이터 일관성에 영향을 주는가?

<details>
<summary>해설 보기</summary>

**아니요**, 데이터 일관성에 영향을 주지 않습니다. `lock_timeout` 초과로 DDL이 실패하면:

1. DDL이 실행되지 않고 롤백됩니다.
2. Lock을 획득하지 못했으므로 Lock이 해제됩니다.
3. 기존 트랜잭션들은 정상적으로 진행됩니다.

`lock_timeout`은 "이 시간 내에 Lock을 얻지 못하면 포기하고 오류를 반환"하는 설정입니다. 실패하더라도 시스템 상태는 변경되지 않습니다.

실패 후 재시도:
```sql
-- 재시도 로직 (Python 예시)
for attempt in range(3):
    try:
        conn.execute("SET lock_timeout = '10s'")
        conn.execute("ALTER TABLE orders ADD COLUMN note TEXT")
        conn.commit()
        break
    except LockNotAvailable:
        time.sleep(random.uniform(1, 5))
        conn.rollback()
```

</details>

---

**Q2.** `idle_in_transaction_session_timeout`과 `statement_timeout`의 차이는? 어떤 상황에 각각 유용한가?

<details>
<summary>해설 보기</summary>

**`idle_in_transaction_session_timeout`**: `BEGIN` 후 트랜잭션이 열려있지만 아무 쿼리도 실행하지 않는 상태('idle in transaction')가 이 시간을 초과하면 연결을 강제 종료합니다.

```
BEGIN;
SELECT * FROM users;  -- 쿼리 완료
-- (이후 10분 동안 아무것도 안 함)
→ idle_in_transaction_session_timeout = '10min' → 연결 종료
```

유용한 경우: VACUUM 차단 방지, Lock 보유 방지. 애플리케이션 버그(BEGIN 후 COMMIT 안 함)를 자동으로 처리.

**`statement_timeout`**: 단일 SQL 문장의 최대 실행 시간. 이 시간을 초과하면 현재 문장이 취소됩니다.

```
SET statement_timeout = '30s';
SELECT * FROM orders;  -- 30초 초과 시 취소
```

유용한 경우: 예상치 못한 SlowQuery 방지, 인덱스 없는 조회로 DB 부하 폭발 방지.

**함께 설정**:
```ini
idle_in_transaction_session_timeout = '5min'  -- idle in transaction 방지
statement_timeout = 0                          -- 쿼리 자체는 제한 없음 (또는 60000)
lock_timeout = '5s'                            -- Lock 획득 대기 제한
```

</details>

---

<div align="center">

**[⬅️ 이전: 인덱스 사용률 분석](./03-index-usage-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring + PostgreSQL 최적화 ➡️](./05-spring-postgresql.md)**

</div>
