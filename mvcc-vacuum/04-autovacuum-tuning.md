# Autovacuum 튜닝 — 대형 테이블에서 따라오지 못하는 문제 해결

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Autovacuum이 실행되는 조건(threshold 공식)은 무엇인가?
- 대형 테이블에서 Autovacuum이 왜 따라오지 못하는가?
- 테이블별 Autovacuum 파라미터를 개별 설정하는 방법은?
- `autovacuum_vacuum_cost_limit`과 `vacuum_cost_delay`는 무엇을 제어하는가?
- Autovacuum이 실행 중인지 어떻게 확인하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

Autovacuum은 PostgreSQL의 "자동 청소부"다. 기본 설정으로 대부분의 소형/중형 테이블은 자동으로 관리된다. 하지만 초당 수천 건의 UPDATE가 발생하는 대형 테이블, 또는 수억 건 데이터를 가진 테이블에서는 기본 설정으로 Autovacuum이 따라오지 못한다. 결과는 Table Bloat과 XID Wraparound 위험이다. 테이블 특성에 맞는 Autovacuum 파라미터 조정은 PostgreSQL 운영에서 가장 중요한 튜닝 중 하나다.

---

## 😱 흔한 실수 (Before — Autovacuum 기본값 방치)

```
상황: 일일 활성 사용자 10만 명, users 테이블 1,000만 행
  last_login 컬럼 UPDATE: 초당 500건

Autovacuum 기본값:
  autovacuum_vacuum_threshold = 50
  autovacuum_vacuum_scale_factor = 0.2
  → threshold = 50 + 0.2 × 10,000,000 = 2,000,050

  Dead Tuple이 200만 개가 돼야 Autovacuum 실행!
  초당 500 UPDATE → 200만 개까지: 4,000초 (약 1시간 7분)
  → 1시간마다 한 번 Autovacuum → 각 실행 시 Dead Tuple 200만 개 처리

  문제:
  Autovacuum 1회 실행 시 200만 Dead Tuple 처리 시간:
  vacuum_cost_limit=200, page당 Dead 5개 가정 → 400,000 pages
  초당 약 10 pages (비용 제한) → 40,000초 (11시간!)

  → Autovacuum이 처리를 완료하기 전에 또 200만 Dead Tuple 쌓임
  → Table Bloat 지속 증가
  → 결국 테이블이 수십 GB로 부풀어오름
```

---

## ✨ 올바른 접근 (After — 테이블 특성별 튜닝)

```
대형/고빈도 테이블 전략:

  -- scale_factor를 낮춰 더 자주 Autovacuum 실행
  ALTER TABLE users SET (
      autovacuum_vacuum_scale_factor = 0.01,  -- 1% (기본 20%)
      autovacuum_vacuum_threshold = 1000,      -- (기본 50)
      autovacuum_vacuum_cost_limit = 800,      -- 더 빠르게 처리 (기본 200)
      autovacuum_vacuum_cost_delay = 2         -- 2ms sleep (기본 2ms, 0으로 하면 최속)
  );

  -- 계산:
  threshold = 1000 + 0.01 × 10,000,000 = 101,000
  → Dead Tuple 101,000개마다 Autovacuum 실행 (3.4분마다)
  → 각 실행: 약 101,000 Dead Tuple 처리
  → 처리 시간: cost_limit=800 기준 훨씬 빠름

  소형 테이블 전략 (수십만 행 이하):
  기본값으로 충분
  scale_factor=0.2도 절대량이 적으면 OK

  Insert 전용 테이블 (로그, 이벤트):
  Autovacuum Analyze만 필요 (Dead Tuple 거의 없음)
  autovacuum_vacuum_scale_factor = 1.0  -- 실질적 비활성화
  autovacuum_analyze_scale_factor = 0.05
```

---

## 🔬 내부 동작 원리

### 1. Autovacuum 실행 조건 공식

```
Autovacuum VACUUM 실행 조건:
  n_dead_tup > autovacuum_vacuum_threshold
            + autovacuum_vacuum_scale_factor × n_live_tup

기본값:
  autovacuum_vacuum_threshold = 50
  autovacuum_vacuum_scale_factor = 0.2

예시:
  테이블 행 수  | threshold
  ─────────────┼──────────────────────────────
  1,000행       | 50 + 0.2×1000 = 250
  10만 행       | 50 + 0.2×100000 = 20,050
  100만 행      | 50 + 0.2×1000000 = 200,050
  1,000만 행    | 50 + 0.2×10000000 = 2,000,050

대형 테이블 문제:
  1,000만 행 테이블: 200만 Dead Tuple이 돼야 실행
  → 오랫동안 Dead Tuple 방치 → Bloat

Autovacuum ANALYZE 실행 조건:
  n_mod_since_analyze > autovacuum_analyze_threshold
                      + autovacuum_analyze_scale_factor × n_live_tup
  기본: 50 + 0.2 × n_live_tup

  ANALYZE: 통계 정보 갱신 (플래너 정확도)
  VACUUM과 독립적으로 실행될 수 있음
```

### 2. Autovacuum Worker 구조

```
Autovacuum 아키텍처:

postmaster
  ├── autovacuum launcher  ← 주기적으로 Autovacuum Worker 생성 결정
  │     autovacuum_naptime: Worker 스케줄 주기 (기본 1분)
  │     모든 테이블을 순회하며 threshold 초과 여부 확인
  │
  ├── autovacuum worker 1  ← 특정 테이블 VACUUM 실행
  ├── autovacuum worker 2  ← 다른 테이블 VACUUM 실행
  └── autovacuum worker N  ← max N: autovacuum_max_workers (기본 3)

autovacuum_max_workers = 3 의 의미:
  최대 3개 테이블을 동시에 VACUUM 가능
  DB가 많거나 테이블이 많으면 → 동시 처리 부족
  → autovacuum_max_workers 증가 고려 (단, 메모리/CPU 고려)

Autovacuum Launcher 동작:
  1. autovacuum_naptime마다 깨어남 (기본 1분)
  2. 각 DB의 pg_stat_user_tables 순회
  3. threshold 초과 테이블 발견 → Worker 생성 요청
  4. 이미 max_workers 실행 중이면 → 다음 naptime에 재시도

실행 중 Autovacuum 확인:
  SELECT pid, datname, relid::regclass AS table_name,
         phase, heap_blks_total, heap_blks_scanned
  FROM pg_stat_activity a
  JOIN pg_stat_progress_vacuum v ON v.pid = a.pid
  WHERE a.query LIKE 'autovacuum%';
```

### 3. Cost-Based Delay (속도 제한)

```
Autovacuum이 I/O를 독점하지 않도록 속도 제한:

비용 계산:
  각 페이지 읽기/쓰기 시 비용 누적:
    vacuum_cost_page_hit = 1    ← shared_buffers에서 캐시 히트
    vacuum_cost_page_miss = 2   ← 디스크에서 읽어야 함
    vacuum_cost_page_dirty = 20 ← 페이지 수정 (FSM, Heap)

  누적 비용이 autovacuum_vacuum_cost_limit 초과:
  → autovacuum_vacuum_cost_delay 만큼 sleep (기본 2ms)
  → 다시 진행

VACUUM (수동) 속도:
  vacuum_cost_limit (기본 200), vacuum_cost_delay (기본 0ms = sleep 없음)
  → 수동 VACUUM은 기본적으로 속도 제한 없음

Autovacuum 속도 vs 수동 VACUUM:
  Autovacuum: cost_limit=200, delay=2ms → 초당 약 100페이지
  수동 VACUUM: cost_limit=200, delay=0ms → 초당 수백~수천 페이지

튜닝 예:
  -- 전역 설정 (postgresql.conf)
  autovacuum_vacuum_cost_limit = 800   # 더 빠르게
  autovacuum_vacuum_cost_delay = 2     # 2ms 유지

  -- 특정 테이블만 빠르게
  ALTER TABLE hot_table SET (
      autovacuum_vacuum_cost_limit = 2000,
      autovacuum_vacuum_cost_delay = 0  -- 속도 제한 없음
  );
```

### 4. 테이블별 파라미터 설정

```
테이블별 오버라이드 가능한 파라미터 목록:

  autovacuum_enabled           -- 이 테이블 Autovacuum 활성화 여부
  autovacuum_vacuum_threshold  -- Dead Tuple 최소 수
  autovacuum_vacuum_scale_factor
  autovacuum_analyze_threshold
  autovacuum_analyze_scale_factor
  autovacuum_vacuum_cost_delay -- sleep 시간
  autovacuum_vacuum_cost_limit -- 비용 한도
  autovacuum_freeze_min_age    -- Freeze 최소 XID 나이
  autovacuum_freeze_max_age    -- 이 나이 초과 시 강제 Freeze VACUUM
  fillfactor                   -- 페이지 채우기 비율 (HOT Update 관련)

설정 방법:
  ALTER TABLE table_name SET (param = value, ...);

현재 설정 확인:
  SELECT relname, reloptions
  FROM pg_class
  WHERE relname = 'users';
  -- reloptions: {autovacuum_vacuum_scale_factor=0.01,...}

설정 초기화 (전역값으로 되돌리기):
  ALTER TABLE users RESET (autovacuum_vacuum_scale_factor);

워크로드 유형별 권장 설정:

  고빈도 UPDATE 테이블 (초당 100건 이상):
  ALTER TABLE high_update_table SET (
      autovacuum_vacuum_scale_factor = 0.01,
      autovacuum_vacuum_cost_limit = 1000
  );

  Insert-Only 로그 테이블:
  ALTER TABLE log_table SET (
      autovacuum_vacuum_scale_factor = 1.0,   -- 실질적 비활성화
      autovacuum_analyze_scale_factor = 0.02
  );

  거의 변경 없는 참조 테이블:
  ALTER TABLE country_codes SET (
      autovacuum_vacuum_scale_factor = 0.5,   -- 덜 자주 실행
      autovacuum_vacuum_cost_delay = 10        -- 더 천천히
  );
```

---

## 💻 실전 실험

### 실험 1: Autovacuum 실행 현황 모니터링

```sql
-- Autovacuum 실행 이력 (마지막 실행 시간)
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_ratio,
    last_autovacuum,
    last_autoanalyze,
    autovacuum_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY last_autovacuum DESC NULLS LAST
LIMIT 20;

-- 현재 실행 중인 Autovacuum 프로세스
SELECT
    pid,
    now() - xact_start AS duration,
    query
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%'
ORDER BY xact_start;

-- Autovacuum 진행 상황 (pg_stat_progress_vacuum)
SELECT
    a.pid,
    p.relid::regclass AS table_name,
    p.phase,
    p.heap_blks_total,
    p.heap_blks_scanned,
    round(p.heap_blks_scanned * 100.0 / nullif(p.heap_blks_total, 0), 1) AS pct,
    p.num_dead_tuples,
    p.max_dead_tuples
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a ON a.pid = p.pid;
```

### 실험 2: Autovacuum threshold 계산 쿼리

```sql
-- 각 테이블의 Autovacuum threshold와 현재 Dead Tuple 비교
WITH av_settings AS (
    SELECT
        c.oid,
        c.relname,
        c.reltuples,
        COALESCE(
            (SELECT (option_value::float)
             FROM pg_options_to_table(c.reloptions)
             WHERE option_name = 'autovacuum_vacuum_scale_factor'),
            current_setting('autovacuum_vacuum_scale_factor')::float
        ) AS scale_factor,
        COALESCE(
            (SELECT option_value::int
             FROM pg_options_to_table(c.reloptions)
             WHERE option_name = 'autovacuum_vacuum_threshold'),
            current_setting('autovacuum_vacuum_threshold')::int
        ) AS threshold
    FROM pg_class c
    WHERE c.relkind = 'r'
)
SELECT
    s.relname,
    s.reltuples::bigint AS estimated_rows,
    t.n_dead_tup AS dead_tuples,
    round(s.threshold + s.scale_factor * s.reltuples) AS av_threshold,
    round(t.n_dead_tup * 100.0 / nullif(s.threshold + s.scale_factor * s.reltuples, 0), 1) AS pct_of_threshold
FROM av_settings s
JOIN pg_stat_user_tables t ON t.relid = s.oid
WHERE s.reltuples > 1000
ORDER BY pct_of_threshold DESC NULLS LAST
LIMIT 20;
```

### 실험 3: 테이블 개별 파라미터 설정 및 효과 확인

```sql
-- 실험 테이블
CREATE TABLE av_test (id SERIAL PRIMARY KEY, val TEXT);
INSERT INTO av_test (val) SELECT md5(i::text) FROM generate_series(1, 100000) i;

-- 기본 threshold 확인
SELECT 50 + 0.2 * 100000 AS default_threshold;
-- 20,050: 이만큼 Dead Tuple이 쌓여야 Autovacuum 실행

-- 공격적 설정으로 변경
ALTER TABLE av_test SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 100
);

-- 새 threshold
SELECT 100 + 0.01 * 100000 AS new_threshold;
-- 1,100: 1,100개만 쌓여도 Autovacuum 실행

-- 설정 확인
SELECT relname, reloptions
FROM pg_class WHERE relname = 'av_test';
```

---

## 📊 MySQL과 비교

```
MySQL InnoDB Purge vs PostgreSQL Autovacuum:

MySQL:
  innodb_purge_threads = 4 (기본)
  → 백그라운드에서 자동으로 Undo Log 정리
  → 사용자 개입 거의 불필요
  → Long Transaction이 Purge를 막는 것이 주된 관리 포인트

PostgreSQL:
  autovacuum_max_workers = 3 (기본)
  → 테이블별 threshold 초과 시 실행
  → 대형 테이블은 scale_factor 튜닝 필요
  → DBA가 테이블 특성별 파라미터 관리

운영 복잡도:
  MySQL: innodb_purge_threads 조정 + long transaction 모니터링
  PostgreSQL: Autovacuum 파라미터 테이블별 튜닝 + 진행 모니터링
  → PostgreSQL이 더 세밀한 제어 가능하나 더 많은 관리 필요
```

---

## ⚖️ 트레이드오프

```
Autovacuum 공격적 설정의 트레이드오프:

scale_factor를 매우 낮게 (예: 0.001):
  장점: Dead Tuple 빠르게 정리, Table Bloat 최소화
  단점: Autovacuum이 매우 자주 실행 → I/O 부하 증가

cost_limit을 매우 높게 (예: 10000):
  장점: VACUUM이 빠르게 완료
  단점: I/O 집중 → 운영 쿼리 성능 저하 (I/O 경합)

cost_delay = 0 (속도 제한 없음):
  장점: 가장 빠른 VACUUM
  단점: 디스크 I/O 독점 가능 → 쿼리 지연

균형 전략:
  cost_limit = 400~800 (기본의 2~4배)
  cost_delay = 2ms (기본 유지)
  scale_factor = 0.01~0.05 (대형 테이블)
  → I/O 부하를 제한하면서 Dead Tuple 제때 처리
```

---

## 📌 핵심 정리

```
Autovacuum 튜닝 핵심:

실행 조건:
  n_dead_tup > threshold + scale_factor × n_live_tup
  기본: 50 + 0.2 × n_live_tup
  대형 테이블 문제: scale_factor=0.2이면 threshold가 너무 큼

속도 제어:
  cost_limit / cost_delay: 비용 한도 초과 시 sleep
  기본: limit=200, delay=2ms → 느린 편

테이블별 설정:
  ALTER TABLE t SET (autovacuum_vacuum_scale_factor = 0.01);
  고빈도 UPDATE: scale_factor 낮춤 (자주 실행)
  Insert-Only: scale_factor 높임 (거의 안 실행)

모니터링:
  pg_stat_user_tables: last_autovacuum, n_dead_tup
  pg_stat_progress_vacuum: 실행 중 진행 상황
  pg_stat_activity: autovacuum% 프로세스 확인
```

---

## 🤔 생각해볼 문제

**Q1.** `autovacuum_max_workers = 3`인데, 3개 테이블이 동시에 threshold를 초과했다면 나머지 테이블은 언제 처리되는가?

<details>
<summary>해설 보기</summary>

`autovacuum_max_workers = 3`이면 최대 3개 테이블을 동시에 VACUUM할 수 있습니다. 4번째 테이블은 현재 Worker 중 하나가 완료될 때까지 기다립니다.

정확히는 `autovacuum_naptime`(기본 1분)마다 Launcher가 모든 테이블을 순회하며 threshold 초과 여부를 확인합니다. 빈 Worker 슬롯이 있으면 새 Worker를 생성합니다. 따라서 4번째 테이블은 최대 1분(naptime) 후에 처리될 수 있습니다.

트래픽이 매우 높아 3개 이상의 테이블이 동시에 처리 필요하다면 `autovacuum_max_workers`를 늘리되, 각 Worker가 `maintenance_work_mem`을 소비함을 감안해야 합니다.

</details>

---

**Q2.** Autovacuum을 특정 테이블에서 완전히 비활성화해야 하는 경우가 있는가?

<details>
<summary>해설 보기</summary>

일반적으로 권장하지 않지만, 다음 경우에는 고려할 수 있습니다:

1. **Unlogged Table**: `CREATE UNLOGGED TABLE`로 생성된 테이블은 Autovacuum 대상에서 제외됩니다(복구 불필요이므로).

2. **임시 테이블**: `TEMP TABLE`은 세션 종료 시 삭제되므로 Autovacuum 불필요.

3. **Insert-Only + 주기적 TRUNCATE**: 로그 테이블을 매일 TRUNCATE한다면 Autovacuum이 불필요.

비활성화 방법:
```sql
ALTER TABLE log_events SET (autovacuum_enabled = false);
```

주의: XID Wraparound 방지용 Freeze VACUUM은 `autovacuum_enabled = false`로 막을 수 없습니다. `vacuum_freeze_min_age`와 `autovacuum_freeze_max_age`에 의해 강제로 실행됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: VACUUM FULL vs VACUUM](./03-vacuum-full-vs-vacuum.md)** | **[홈으로 🏠](../README.md)** | **[다음: HOT Update ➡️](./05-hot-update.md)**

</div>
