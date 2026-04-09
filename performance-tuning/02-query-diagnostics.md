# 쿼리 성능 진단 — pg_stat_statements와 실시간 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `pg_stat_statements`로 누적 슬로우 쿼리를 어떻게 찾는가?
- `auto_explain`은 어떻게 설정하고 어떤 정보를 제공하는가?
- `pg_stat_activity`에서 현재 실행 중인 쿼리를 어떻게 분석하는가?
- `pg_locks`로 잠금 대기를 어떻게 진단하는가?
- `log_min_duration_statement` 설정의 적정값과 로그 분석 방법은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

"왜 갑자기 응답이 느려졌지?"라는 상황에서 어디서부터 분석을 시작해야 할지 모르면 손을 쓸 수 없다. `pg_stat_statements`는 서비스 오픈 후 누적된 쿼리 통계를 제공해 "어떤 쿼리가 가장 많은 시간을 소비했는가"를 즉시 파악할 수 있게 해준다. `pg_stat_activity`는 현재 실행 중인 쿼리를 실시간으로 보여주며, `pg_locks`는 잠금 대기로 인한 연결 누적의 원인을 찾는다. 이 세 가지 뷰를 활용하면 대부분의 성능 문제를 진단할 수 있다.

---

## 😱 흔한 실수 (Before — 진단 도구 미활용)

```
실수 1: 느린 쿼리를 애플리케이션 로그로만 추적

  애플리케이션 로그: "쿼리 X가 2초 걸렸다"
  → 몇 번 실행됐는지 모름
  → 평균 실행 시간 모름
  → 총 소비 시간 모름

  pg_stat_statements 설치 후:
  SELECT query, calls, total_exec_time, mean_exec_time, rows
  FROM pg_stat_statements
  ORDER BY total_exec_time DESC LIMIT 20;
  → "이 쿼리가 하루에 10만 번 실행, 평균 50ms, 총 83분 소비"
  → 이 쿼리를 최적화하면 가장 큰 효과

실수 2: Lock 대기 원인 모름

  서비스 응답 지연 → DB 연결 수 급증
  이유를 모르고 서버 재시작 → 임시 해결 → 재발

  pg_locks 분석으로:
  SELECT ...  -- 잠금 대기 체인 확인
  → "ALTER TABLE이 AccessExclusiveLock 보유 중"
  → "다른 모든 쿼리가 그 잠금 대기 중"
  → ALTER TABLE 취소 → 즉시 해결

실수 3: auto_explain 없이 운영 슬로우 쿼리 실행 계획 모름

  운영 환경에서 쿼리가 2초 걸렸는데:
  "왜 느린지 실행 계획을 봤어야 했는데..."
  EXPLAIN ANALYZE는 다시 실행해야 함 → 데이터/부하가 다를 수 있음

  auto_explain 설정:
  auto_explain.log_min_duration = '1s'
  auto_explain.log_analyze = true
  → 1초 이상 걸린 쿼리의 EXPLAIN ANALYZE 결과 자동 로깅
```

---

## ✨ 올바른 접근 (After — 체계적 성능 진단)

```
성능 진단 우선순위:

  1. pg_stat_statements: 누적 통계로 "병목 쿼리" 파악
  2. pg_stat_activity: 현재 문제 파악
  3. pg_locks: 잠금 대기 체인 파악
  4. auto_explain 로그: 슬로우 쿼리 실행 계획
  5. EXPLAIN ANALYZE: 특정 쿼리 심층 분석

빠른 진단 체크리스트:

  -- 1. 현재 실행 중인 느린 쿼리 (30초 이상)
  SELECT pid, now() - query_start AS duration, state, query
  FROM pg_stat_activity
  WHERE state != 'idle' AND query_start < now() - INTERVAL '30 seconds'
  ORDER BY duration DESC;

  -- 2. 잠금 대기 체인
  SELECT waiting.pid, waiting.query, blocking.pid AS blocking_pid, blocking.query
  FROM pg_stat_activity waiting
  JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(waiting.pid))
  WHERE waiting.wait_event_type = 'Lock';

  -- 3. 누적 슬로우 쿼리 TOP 10
  SELECT query, calls, round(total_exec_time/1000) AS total_sec,
         round(mean_exec_time) AS avg_ms
  FROM pg_stat_statements
  ORDER BY total_exec_time DESC LIMIT 10;
```

---

## 🔬 내부 동작 원리

### 1. pg_stat_statements

```
설치 및 활성화:

  postgresql.conf:
  shared_preload_libraries = 'pg_stat_statements'  # 재시작 필요
  pg_stat_statements.max = 10000                   # 추적할 쿼리 수
  pg_stat_statements.track = all                   # top(기본): 상위 레벨만, all: 중첩 함수 포함

  활성화 후:
  CREATE EXTENSION pg_stat_statements;

주요 컬럼:

  SELECT * FROM pg_stat_statements LIMIT 1;

  userid:          실행한 사용자
  dbid:            데이터베이스 ID
  queryid:         쿼리 식별자 (파라미터 제외한 구조 해시)
  query:           쿼리 텍스트 (파라미터는 $1, $2로 정규화)
  calls:           실행 횟수
  total_exec_time: 총 실행 시간 (ms)
  mean_exec_time:  평균 실행 시간 (ms)
  stddev_exec_time:실행 시간 표준편차 (불안정한 쿼리 감지)
  rows:            반환된 총 행 수
  shared_blks_hit: shared_buffers 캐시 히트
  shared_blks_read:디스크에서 읽은 페이지 수
  local_blks_hit/read: 로컬 버퍼 (임시 테이블)
  temp_blks_read/written: 임시 파일 (정렬 디스크 spill)
  wal_bytes:       생성된 WAL 바이트 (쓰기 집중 쿼리 파악)

파라미터 정규화:
  SELECT * FROM users WHERE id = 42  → SELECT * FROM users WHERE id = $1
  SELECT * FROM users WHERE id = 100 → 동일 queryid (같은 쿼리 구조)
  → 다양한 파라미터 값을 하나의 항목으로 집계

유용한 분석 쿼리:

  -- 총 실행 시간 TOP 10 (최고 비용 쿼리)
  SELECT
      left(query, 80) AS query_preview,
      calls,
      round(total_exec_time::numeric / 1000, 2) AS total_sec,
      round(mean_exec_time::numeric, 2) AS avg_ms,
      round(stddev_exec_time::numeric, 2) AS stddev_ms,
      rows
  FROM pg_stat_statements
  ORDER BY total_exec_time DESC
  LIMIT 10;

  -- 가장 느린 평균 쿼리 (빈도 적고 느린 쿼리)
  SELECT left(query, 80), calls, round(mean_exec_time::numeric, 2) AS avg_ms
  FROM pg_stat_statements
  WHERE calls > 100
  ORDER BY mean_exec_time DESC LIMIT 10;

  -- I/O 집중 쿼리 (캐시 히트율 낮음)
  SELECT left(query, 80), shared_blks_read, shared_blks_hit,
         round(shared_blks_hit * 100.0 / nullif(shared_blks_hit + shared_blks_read, 0), 1) AS hit_pct
  FROM pg_stat_statements
  WHERE shared_blks_read + shared_blks_hit > 1000
  ORDER BY shared_blks_read DESC LIMIT 10;

  -- 통계 초기화
  SELECT pg_stat_statements_reset();
```

### 2. auto_explain

```
auto_explain 설정:

  postgresql.conf:
  shared_preload_libraries = 'pg_stat_statements,auto_explain'

  # 전역 설정
  auto_explain.log_min_duration = 1000  # 1초 이상 쿼리 자동 로깅 (ms)
  auto_explain.log_analyze = true       # 실제 실행 통계 포함
  auto_explain.log_buffers = true       # I/O 통계 포함
  auto_explain.log_verbose = false      # 상세 출력 (보통 false)
  auto_explain.log_nested_statements = false  # 중첩 함수 내부 쿼리
  auto_explain.sample_rate = 1.0       # 100% 샘플링 (프로덕션에서 0.01 = 1%)

  세션 수준 설정:
  LOAD 'auto_explain';  # 세션에서만 활성화
  SET auto_explain.log_min_duration = 0;  # 모든 쿼리 로깅
  SET auto_explain.log_analyze = true;

  auto_explain 로그 예시:
  LOG:  duration: 2345.678 ms  plan:
        Query Text: SELECT * FROM orders JOIN users ON ...
        Gather  (cost=...) (actual time=... rows=... loops=1)
          -> Nested Loop  (cost=...)
               -> Seq Scan on orders  (cost=...) ← 문제!
               -> Index Scan on users (cost=...)

  sample_rate 조정:
  프로덕션에서 log_analyze = true는 쿼리를 두 번 실행 (계획 + 실행)하는 것과 유사
  → sample_rate = 0.01~0.1로 부하 감소
```

### 3. pg_stat_activity 실시간 분석

```
pg_stat_activity 주요 컬럼:

  pid:              백엔드 프로세스 ID
  usename:          실행 사용자
  application_name: 애플리케이션 이름 (HikariCP 등)
  client_addr:      클라이언트 IP
  state:            'active'/'idle'/'idle in transaction'/'waiting'
  wait_event_type:  대기 원인 타입 (Lock/IO/IPC 등)
  wait_event:       구체적 대기 원인
  query_start:      쿼리 시작 시각
  xact_start:       트랜잭션 시작 시각
  query:            현재 실행 중인 쿼리

상태별 분석:

  -- 'active': 실제 쿼리 실행 중
  SELECT pid, now() - query_start AS duration, query
  FROM pg_stat_activity WHERE state = 'active'
  ORDER BY duration DESC;

  -- 'idle in transaction': 트랜잭션 열고 쿼리 안 하는 상태 (위험!)
  SELECT pid, now() - xact_start AS txn_duration, query
  FROM pg_stat_activity WHERE state = 'idle in transaction'
  ORDER BY txn_duration DESC;
  -- idle in transaction이 오래되면: VACUUM 차단, Lock 보유 가능
  -- 설정: idle_in_transaction_session_timeout = '5min'

  -- Lock 대기 중인 쿼리
  SELECT pid, wait_event_type, wait_event, query
  FROM pg_stat_activity WHERE wait_event_type = 'Lock';

  -- 쿼리 종료
  SELECT pg_cancel_backend(pid);   -- SIGINT (현재 쿼리만 취소)
  SELECT pg_terminate_backend(pid); -- SIGTERM (연결 종료)
```

### 4. pg_locks 잠금 분석

```
pg_locks 주요 컬럼:

  locktype:     잠금 대상 타입 (relation/tuple/transactionid 등)
  relation:     테이블 OID
  pid:          Lock 보유/대기 중인 프로세스
  mode:         Lock 모드 (AccessShareLock ~ AccessExclusiveLock)
  granted:      true=보유, false=대기 중

잠금 대기 체인 시각화:

  SELECT
      waiting.pid AS waiting_pid,
      waiting_activity.query AS waiting_query,
      blocking.pid AS blocking_pid,
      blocking_activity.query AS blocking_query,
      waiting_activity.wait_event_type,
      waiting_activity.wait_event
  FROM pg_stat_activity waiting_activity
  JOIN pg_stat_activity blocking_activity
      ON blocking_activity.pid = ANY(pg_blocking_pids(waiting_activity.pid))
  JOIN pg_locks waiting ON waiting.pid = waiting_activity.pid AND NOT waiting.granted
  JOIN pg_locks blocking ON blocking.pid = blocking_activity.pid AND blocking.granted
  WHERE waiting_activity.wait_event_type = 'Lock';

  -- 출력 예시:
  -- waiting_pid | waiting_query              | blocking_pid | blocking_query
  -- 1234        | SELECT * FROM orders       | 5678         | ALTER TABLE orders ADD ...
  -- 2345        | UPDATE orders SET ...      | 5678         | ALTER TABLE orders ADD ...
  -- (ALTER TABLE가 AccessExclusiveLock 보유 → 모든 쿼리 대기)

  해결:
  SELECT pg_cancel_backend(5678);  -- ALTER TABLE 취소
  → 대기 중인 쿼리들 즉시 진행

테이블 잠금 현황:

  SELECT
      c.relname AS table_name,
      l.mode,
      l.granted,
      a.pid,
      now() - a.query_start AS lock_duration,
      a.query
  FROM pg_locks l
  JOIN pg_class c ON c.oid = l.relation
  JOIN pg_stat_activity a ON a.pid = l.pid
  WHERE c.relname = 'orders'
  ORDER BY l.granted DESC, lock_duration DESC;
```

---

## 💻 실전 실험

### 실험 1: pg_stat_statements 설치 및 분석

```sql
-- pg_stat_statements 설치 (shared_preload_libraries 설정 후 재시작)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 통계 초기화 (분석 시작점)
SELECT pg_stat_statements_reset();

-- 부하 생성 (다양한 쿼리)
SELECT count(*) FROM pg_class;  -- 여러 번
SELECT * FROM pg_stat_activity LIMIT 10;
SELECT now();

-- 분석
SELECT
    left(query, 100) AS query_preview,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 4) AS avg_ms,
    rows
FROM pg_stat_statements
WHERE query NOT LIKE 'SELECT pg_stat%'
ORDER BY total_exec_time DESC
LIMIT 10;
```

### 실험 2: 잠금 대기 시뮬레이션

```sql
-- 실험 테이블
CREATE TABLE lock_test (id INT PRIMARY KEY, val TEXT);
INSERT INTO lock_test VALUES (1, 'data');

-- 터미널 1: 잠금 보유
BEGIN;
SELECT * FROM lock_test FOR UPDATE;  -- 행 잠금 보유 (대기)

-- 터미널 2: 잠금 대기 발생
BEGIN;
UPDATE lock_test SET val = 'new' WHERE id = 1;  -- 대기!

-- 터미널 3: 잠금 대기 체인 확인
SELECT
    waiting.pid AS waiting_pid,
    left(w.query, 50) AS waiting_query,
    blocking.pid AS blocking_pid,
    left(b.query, 50) AS blocking_query
FROM pg_stat_activity w
JOIN pg_stat_activity b ON b.pid = ANY(pg_blocking_pids(w.pid))
WHERE w.wait_event_type = 'Lock';

-- 터미널 1: 잠금 해제
ROLLBACK;
```

### 실험 3: idle in transaction 감지

```sql
-- 위험한 패턴 시뮬레이션
-- 터미널 1:
BEGIN;
SELECT * FROM lock_test;
-- 이 상태에서 아무것도 안 함 (idle in transaction)

-- 터미널 2: idle in transaction 감지
SELECT
    pid,
    now() - xact_start AS transaction_duration,
    state,
    left(query, 50) AS last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;

-- 심각한 경우 강제 종료
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > INTERVAL '5 minutes';
```

---

## 📊 MySQL과 비교

```
MySQL 성능 진단 도구 vs PostgreSQL:

MySQL:
  SHOW PROCESSLIST: 현재 실행 중인 쿼리 (pg_stat_activity 대응)
  SHOW ENGINE INNODB STATUS: InnoDB 잠금 상태 포함
  performance_schema.events_statements_summary_by_digest:
    pg_stat_statements 대응 (누적 쿼리 통계)
  information_schema.innodb_locks: 잠금 정보 (pg_locks 대응)
  slow_query_log: 슬로우 쿼리 로그 (log_min_duration_statement 대응)

PostgreSQL:
  pg_stat_statements: 강력한 쿼리 통계 (표준화된 쿼리 정규화)
  pg_stat_activity: 실행 중 쿼리 상태
  pg_locks: 잠금 분석
  auto_explain: 슬로우 쿼리 자동 실행 계획 로깅
  pg_blocking_pids(): 잠금 대기 체인 즉시 파악

차이점:
  MySQL: slow_query_log는 파일 기반, pg는 DB 뷰로 SQL 분석
  PostgreSQL pg_stat_statements: queryid로 파라미터 정규화 → 집계 정확
  MySQL 잠금 분석: SHOW ENGINE INNODB STATUS (텍스트), 파싱 어려움
  PostgreSQL: pg_locks + pg_blocking_pids() SQL로 체계적 분석
```

---

## ⚖️ 트레이드오프

```
진단 도구 사용 비용:

pg_stat_statements:
  오버헤드: 쿼리당 통계 기록 → 약 1~5% 성능 영향
  → 프로덕션에서 항상 활성화 권장 (이점 >> 비용)

auto_explain:
  log_analyze = true: 실행 계획 수집 비용
  → sample_rate = 0.01~0.1로 부하 감소
  log_min_duration = 1000: 1초 이상 쿼리만 → 일반적으로 낮은 오버헤드

pg_stat_activity 조회:
  시스템 카탈로그 락 필요 → 매우 빠른 조회 → 오버헤드 무시

pg_locks 조회:
  잠금 테이블 스캔 → 빠름
  잠금 대기 체인 조인 → 비교적 빠름

운영 권장:
  pg_stat_statements: 항상 on
  log_min_duration_statement: 1000~5000ms
  auto_explain: sample_rate 낮게 + log_analyze on
  pg_stat_activity: 필요 시 즉시 조회
```

---

## 📌 핵심 정리

```
쿼리 성능 진단 핵심:

pg_stat_statements:
  shared_preload_libraries = 'pg_stat_statements'
  누적 쿼리 통계: total_exec_time, mean_exec_time, calls
  → 총 소비 시간 TOP N 쿼리가 최우선 최적화 대상

auto_explain:
  shared_preload_libraries에 추가
  log_min_duration: 임계값 (ms)
  log_analyze: 실제 실행 통계 포함
  sample_rate: 프로덕션에서 0.01~0.1

pg_stat_activity:
  state: active/idle/idle in transaction
  idle in transaction: VACUUM 차단, Lock 보유 위험
  pg_blocking_pids(): 잠금 대기 원인 찾기

pg_locks:
  잠금 대기 체인 시각화
  blocking 쿼리 → pg_cancel_backend(pid)

로그:
  log_min_duration_statement = 1000 (1초 이상)
  log_checkpoints = on (Checkpoint 모니터링)
```

---

## 🤔 생각해볼 문제

**Q1.** `pg_stat_statements`의 `queryid`가 같은 쿼리는 어떤 경우인가? 다른 파라미터 값도 같은 `queryid`가 될 수 있는가?

<details>
<summary>해설 보기</summary>

`queryid`는 쿼리의 **파싱 트리 구조**를 해시한 값입니다. 파라미터 값과 무관하게 동일한 SQL 구조면 같은 `queryid`입니다.

```sql
-- 모두 같은 queryid (파라미터만 다름)
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;
SELECT * FROM users WHERE id = 100;
→ 정규화: SELECT * FROM users WHERE id = $1

-- 다른 queryid (구조가 다름)
SELECT * FROM users WHERE id = 1 AND name = 'Alice';
→ SELECT * FROM users WHERE id = $1 AND name = $2
```

같은 `queryid`가 아닌 경우:
- 컬럼 순서가 다른 경우 (`SELECT a, b` vs `SELECT b, a`)
- JOIN 순서가 다른 경우
- WHERE 조건 구조가 다른 경우

이 정규화 덕분에 "이 쿼리 패턴이 하루에 얼마나 실행됐는가"를 파라미터와 무관하게 집계할 수 있습니다.

</details>

---

**Q2.** `auto_explain.log_analyze = true` 설정이 운영 환경에서 위험할 수 있는 이유는?

<details>
<summary>해설 보기</summary>

`log_analyze = true`는 쿼리를 **실제로 실행**하면서 각 노드의 실제 실행 시간, 행 수, I/O 통계를 수집합니다. 이것이 추가 오버헤드를 만듭니다:

1. **타이밍 정보 수집**: 각 Plan 노드의 시작/종료 시각 기록 → 약간의 CPU 오버헤드
2. **샘플링 없이 전체 적용**: `sample_rate = 1.0`(기본)이면 모든 슬로우 쿼리에 적용

완화 방법:
```ini
auto_explain.sample_rate = 0.01  # 1%만 샘플링
auto_explain.log_min_duration = 5000  # 5초 이상만 (임계값 높임)
```

DML 쿼리(`UPDATE`, `DELETE`)도 `log_analyze`가 실행하므로 Side Effect가 있는 쿼리도 두 번 처리되는 것처럼 보일 수 있습니다. 하지만 실제로 DML은 한 번만 실행되며, auto_explain은 실행 계획 노드의 통계를 수집하는 것입니다.

결론: `log_min_duration`을 충분히 높게 설정하고 `sample_rate`을 낮추면 프로덕션에서도 안전하게 사용할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: postgresql.conf 핵심 설정](./01-postgresql-conf.md)** | **[홈으로 🏠](../README.md)** | **[다음: 인덱스 사용률 분석 ➡️](./03-index-usage-analysis.md)**

</div>
