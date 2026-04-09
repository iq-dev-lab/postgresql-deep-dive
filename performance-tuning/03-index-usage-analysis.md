# 인덱스 사용률 분석 — 미사용 인덱스 찾기와 Bloat 해소

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `pg_stat_user_indexes`로 미사용 인덱스를 어떻게 찾는가?
- 인덱스 Bloat(부풀음)의 원인과 정량 측정 방법은?
- `REINDEX CONCURRENTLY`와 일반 `REINDEX`의 차이는?
- 중복 인덱스와 서브셋 인덱스를 어떻게 식별하는가?
- 인덱스를 삭제하기 전에 어떤 검증 절차가 필요한가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

"인덱스는 많을수록 좋다"는 오해가 있다. 현실에서는 사용하지 않는 인덱스가 INSERT/UPDATE/DELETE 성능을 떨어뜨리고, HOT Update 기회를 빼앗으며, 디스크 공간을 낭비한다. 많은 서비스에서 인덱스의 30~50%가 실제로는 사용되지 않는다. `pg_stat_user_indexes`와 `pg_statio_user_indexes`로 인덱스 사용률을 측정하고, 미사용 인덱스를 제거하면 쓰기 성능이 크게 향상된다.

---

## 😱 흔한 실수 (Before — 인덱스 과다 생성)

```
실수 1: 모든 컬럼에 인덱스 생성

  "혹시 몰라서" 모든 컬럼에 인덱스:
  CREATE INDEX ON orders(status);
  CREATE INDEX ON orders(user_id);
  CREATE INDEX ON orders(amount);
  CREATE INDEX ON orders(status, user_id);  -- (status)의 슈퍼셋
  CREATE INDEX ON orders(user_id, status);  -- (user_id)의 슈퍼셋

  결과:
  인덱스 5개 → 모든 INSERT/UPDATE에 5개 인덱스 갱신
  중복 인덱스: (status)는 (status, user_id)로 대체 가능
  HOT Update 불가: status, user_id 모두 인덱스에 있음

실수 2: 미사용 인덱스 방치

  서비스 초기에 만든 인덱스가 쿼리 패턴 변경으로 더 이상 사용 안 됨
  pg_stat_user_indexes.idx_scan = 0이 수개월째

  낭비:
  인덱스 크기: 수 GB
  쓰기 성능: 불필요한 인덱스 갱신으로 저하

실수 3: 인덱스 Bloat 방치

  대량 UPDATE/DELETE 후 인덱스 페이지가 비어있는데 파일 크기 유지
  → 인덱스 스캔 시 빈 페이지도 읽음 → 성능 저하

  측정 후 REINDEX CONCURRENTLY로 재구성 필요
```

---

## ✨ 올바른 접근 (After — 인덱스 사용률 기반 관리)

```
인덱스 관리 주기:

  월 1회: 미사용 인덱스 확인 (idx_scan = 0)
  분기:   Bloat 측정 → 심한 경우 REINDEX CONCURRENTLY
  연간:   중복 인덱스 분석 → 정리

  빠른 인덱스 현황 대시보드:
  SELECT
      schemaname,
      relname AS table_name,
      indexrelname AS index_name,
      idx_scan AS scans_since_reset,
      pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
      CASE WHEN idx_scan = 0 THEN '⚠️ 미사용' ELSE '✅ 사용 중' END AS status
  FROM pg_stat_user_indexes
  WHERE schemaname = 'public'
  ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC
  LIMIT 30;

  인덱스 삭제 전 검증:
  1. 통계 기간 충분한지 확인 (최소 2주)
  2. pg_stat_reset() 후 재측정
  3. 쿼리 플랜 확인 (삭제 시 어떤 쿼리가 느려지나)
  4. CONCURRENTLY 옵션으로 운영 중 삭제:
     DROP INDEX CONCURRENTLY idx_orders_amount;
```

---

## 🔬 내부 동작 원리

### 1. 인덱스 통계 뷰

```
pg_stat_user_indexes:

  relname:      테이블 이름
  indexrelname: 인덱스 이름
  idx_scan:     이 인덱스로 시작한 인덱스 스캔 수
  idx_tup_read: 인덱스에서 반환된 튜플 수
  idx_tup_fetch:인덱스 스캔 후 Heap에서 실제 가져온 튜플 수

  주의: idx_scan은 마지막 pg_stat_reset() 이후 누적값
  → 서비스 오픈 후 통계 초기화 없이 오래 유지하는 것이 좋음

pg_statio_user_indexes:

  idx_blks_read: 디스크에서 읽은 인덱스 페이지 (캐시 미스)
  idx_blks_hit:  shared_buffers에서 읽은 인덱스 페이지 (캐시 히트)

  인덱스 캐시 히트율:
  SELECT indexrelname,
         round(idx_blks_hit * 100.0 / nullif(idx_blks_hit + idx_blks_read, 0), 1) AS hit_pct
  FROM pg_statio_user_indexes
  WHERE idx_blks_hit + idx_blks_read > 100;

  인덱스가 shared_buffers에 잘 캐시되는지 확인
  → 자주 사용되는 인덱스: 높은 hit_pct 기대
  → hit_pct 낮으면: shared_buffers 증가 또는 인덱스 크기 최적화
```

### 2. 중복/서브셋 인덱스 찾기

```
중복 인덱스 패턴:

  1. 동일 컬럼 중복:
     CREATE INDEX idx_a ON t(a);
     CREATE INDEX idx_a_2 ON t(a);
     → 완전 중복, 하나 삭제

  2. 선두 컬럼 서브셋:
     CREATE INDEX idx_a ON t(a);
     CREATE INDEX idx_ab ON t(a, b);
     → idx_a는 WHERE a = ? 쿼리에서 idx_ab로 대체 가능
     → WHERE a = ? AND b = ? → idx_ab 필요
     → WHERE a = ? 만 → idx_ab도 사용 가능 (선두 컬럼)
     → idx_a 삭제 검토

  3. PK의 단순 중복:
     PRIMARY KEY (id) → 자동으로 인덱스 생성
     CREATE UNIQUE INDEX ON t(id); → 중복!

중복 인덱스 찾는 쿼리:

  SELECT
      a.indexrelname AS index_a,
      b.indexrelname AS index_b,
      a.tablename,
      a.indexdef AS index_a_def,
      b.indexdef AS index_b_def
  FROM pg_indexes a
  JOIN pg_indexes b
      ON a.tablename = b.tablename
      AND a.indexname != b.indexname
      AND a.indexdef != b.indexdef  -- 완전 동일하지 않은 것
  WHERE a.tablename = 'orders'
  ORDER BY a.tablename, a.indexrelname;

  -- 완전 중복 인덱스 (같은 컬럼, 같은 조건)
  SELECT
      indrelid::regclass AS table_name,
      array_agg(indexrelid::regclass) AS duplicate_indexes,
      pg_get_indexdef(min(indexrelid)) AS index_definition
  FROM pg_index
  GROUP BY indrelid, indkey, indpred, indexprs
  HAVING count(*) > 1;
```

### 3. 인덱스 Bloat 측정

```
인덱스 Bloat 원인:
  UPDATE/DELETE → 인덱스 엔트리가 Dead 상태
  VACUUM으로 Heap Dead Tuple은 정리되지만
  인덱스 페이지의 빈 공간은 완전히 압축되지 않음
  → 인덱스 페이지 내 빈 공간 = Bloat

  특히 단조 증가하는 PK 인덱스에서:
  오래된 데이터 삭제 → 초반 리프 페이지들이 희박해짐
  → 인덱스 파일은 크지만 실제 엔트리 밀도 낮음

pgstattuple로 측정:

  CREATE EXTENSION pgstattuple;

  SELECT * FROM pgstatindex('orders_pkey');
  -- 출력:
  -- version:             4
  -- tree_level:          2              ← B-Tree 높이
  -- index_size:          536870912     ← 인덱스 파일 크기 (bytes)
  -- root_block_no:       3
  -- internal_pages:      15
  -- leaf_pages:          65521         ← 총 리프 페이지 수
  -- empty_pages:         0             ← 완전히 빈 페이지
  -- deleted_pages:       0             ← 삭제된 페이지
  -- avg_leaf_density:    42.37         ← 리프 페이지 평균 채우기율 (%)
  -- leaf_fragmentation:  12.5          ← 조각화율 (%)

  avg_leaf_density < 50%: Bloat 심함 → REINDEX 검토
  avg_leaf_density < 70%: 약간의 Bloat → 모니터링

  간단한 Bloat 추정:
  SELECT indexrelname,
         pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
         -- 이상적 크기 추정
         pg_size_pretty(
             (pg_relation_size(indexrelid)::numeric * avg_leaf_density / 100)::bigint
         ) AS estimated_ideal_size
  FROM pg_stat_user_indexes i
  CROSS JOIN LATERAL pgstatindex(i.indexrelname::regclass) s
  WHERE i.relname = 'orders';
```

### 4. REINDEX CONCURRENTLY

```
일반 REINDEX:
  REINDEX INDEX orders_pkey;
  → AccessExclusiveLock 획득 → 모든 접근 차단
  → 운영 시간대 절대 사용 금지

REINDEX CONCURRENTLY (PostgreSQL 12+):
  REINDEX INDEX CONCURRENTLY orders_pkey;
  → 잠금 최소화 (최종 교체 시 짧은 잠금만)
  → DML 동시 허용

REINDEX CONCURRENTLY 동작:
  1. 새 인덱스 임시 생성 (기존과 병렬)
     → INVALID 상태로 시작 (_ccnew 접미사)
  2. 새 인덱스에 변경사항 지속 반영
  3. 짧은 잠금 후 기존 ↔ 새 인덱스 교체
  4. 기존 인덱스 삭제

  실패 시 INVALID 인덱스 남음:
  SELECT indexrelname, indisvalid
  FROM pg_stat_user_indexes i
  JOIN pg_index pi ON pi.indexrelid = i.indexrelid
  WHERE NOT pi.indisvalid;
  -- INVALID 인덱스 → DROP INDEX CONCURRENTLY 후 재시도

  주의:
  ① 트랜잭션 블록 안에서 실행 불가
  ② 실행 중 다른 REINDEX CONCURRENTLY 병렬 불가
  ③ 오래 걸릴 수 있음 (테이블 크기에 비례)

전체 테이블 인덱스 재구성:
  REINDEX TABLE CONCURRENTLY orders;
  → 테이블의 모든 인덱스를 CONCURRENTLY로 재구성

pg_repack을 통한 테이블+인덱스 재구성:
  pg_repack --table=orders mydb
  → 테이블 + 인덱스를 운영 중에 재구성 (VACUUM FULL 대안)
```

---

## 💻 실전 실험

### 실험 1: 인덱스 사용률 분석

```sql
-- 통계 초기화 (기준점 설정)
SELECT pg_stat_reset();

-- 2주 후 또는 충분한 부하 후:
-- 미사용 인덱스 확인 (idx_scan = 0, 크기 큰 것 우선)
SELECT
    n.nspname AS schema,
    t.relname AS table_name,
    i.relname AS index_name,
    s.idx_scan AS total_scans,
    pg_size_pretty(pg_relation_size(i.oid)) AS index_size,
    -- 인덱스 정의 확인
    pg_get_indexdef(i.oid) AS index_def
FROM pg_class i
JOIN pg_index ix ON ix.indexrelid = i.oid
JOIN pg_class t ON t.oid = ix.indrelid
JOIN pg_namespace n ON n.oid = t.relnamespace
LEFT JOIN pg_stat_user_indexes s ON s.indexrelid = i.oid
WHERE i.relkind = 'i'
  AND n.nspname = 'public'
  AND NOT ix.indisprimary  -- PK 제외
  AND NOT ix.indisunique   -- UNIQUE 제외 (필수)
ORDER BY s.idx_scan ASC NULLS FIRST, pg_relation_size(i.oid) DESC
LIMIT 20;
```

### 실험 2: pgstatindex로 Bloat 측정

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- 전체 인덱스 Bloat 현황
SELECT
    s.relname AS table_name,
    s.indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size,
    round((pi.avg_leaf_density)::numeric, 1) AS leaf_density_pct,
    round((pi.leaf_fragmentation)::numeric, 1) AS fragmentation_pct,
    CASE
        WHEN pi.avg_leaf_density < 50 THEN '🔴 심각한 Bloat'
        WHEN pi.avg_leaf_density < 70 THEN '🟡 약간의 Bloat'
        ELSE '🟢 정상'
    END AS bloat_status
FROM pg_stat_user_indexes s
CROSS JOIN LATERAL pgstatindex(s.indexrelid) pi
WHERE s.schemaname = 'public'
  AND pg_relation_size(s.indexrelid) > 10 * 1024 * 1024  -- 10MB 이상
ORDER BY pi.avg_leaf_density ASC
LIMIT 20;
```

### 실험 3: REINDEX CONCURRENTLY

```sql
-- 1. 현재 인덱스 크기 확인
SELECT pg_size_pretty(pg_relation_size('orders_pkey')) AS before_size;

-- 2. 대량 데이터 삭제 (Bloat 유발)
DELETE FROM orders WHERE created_at < '2023-01-01';

-- 3. Bloat 확인
SELECT round(avg_leaf_density::numeric, 1) AS leaf_density
FROM pgstatindex('orders_pkey');

-- 4. REINDEX CONCURRENTLY (운영 중 가능)
REINDEX INDEX CONCURRENTLY orders_pkey;

-- 5. 재구성 후 크기 및 밀도 확인
SELECT pg_size_pretty(pg_relation_size('orders_pkey')) AS after_size;
SELECT round(avg_leaf_density::numeric, 1) AS leaf_density
FROM pgstatindex('orders_pkey');

-- 6. INVALID 인덱스 없는지 확인
SELECT indexrelname, indisvalid
FROM pg_stat_user_indexes i
JOIN pg_index pi ON pi.indexrelid = i.indexrelid
WHERE NOT pi.indisvalid;
```

---

## 📊 MySQL과 비교

```
MySQL vs PostgreSQL 인덱스 관리:

MySQL:
  SHOW INDEX FROM orders; -- 인덱스 목록
  SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage
  WHERE object_name = 'orders';
  -- INDEX_NAME의 COUNT_FETCH가 0이면 미사용

  인덱스 Bloat:
  OPTIMIZE TABLE orders; -- 인덱스 재구성 (테이블 잠금, 운영 주의)
  ALTER TABLE orders ENGINE=InnoDB; -- Online DDL로 재구성 (8.0)

PostgreSQL:
  pg_stat_user_indexes: idx_scan으로 미사용 인덱스 파악
  pgstatindex: Bloat 정확 측정
  REINDEX CONCURRENTLY: 운영 중 재구성

차이:
  MySQL: performance_schema (설정 필요, 복잡)
  PostgreSQL: pg_stat_* 뷰 (기본 제공, SQL로 분석)

  MySQL OPTIMIZE TABLE: 테이블 잠금 필요 (운영 불가)
  PostgreSQL REINDEX CONCURRENTLY: 운영 중 가능
```

---

## ⚖️ 트레이드오프

```
인덱스 삭제 결정:

삭제 기준:
  ① idx_scan = 0 (충분한 기간 후)
  ② 중복 인덱스 (슈퍼셋이 존재)
  ③ 인덱스 크기 vs 쓰기 비용 불균형

삭제 전 안전 검증:
  ① 통계 기간: 최소 2주 (계절적 피크 포함)
  ② 통계 초기화 후 재측정: pg_stat_reset() → 2주 대기
  ③ 쿼리 확인: 삭제 시 어떤 쿼리가 SeqScan으로 바뀌나?
     SET enable_indexscan = off;
     EXPLAIN SELECT ... -- 인덱스 없을 때 실행 계획

삭제 방법:
  운영 중: DROP INDEX CONCURRENTLY idx_name;
  운영 외: DROP INDEX idx_name;

REINDEX 타이밍:
  avg_leaf_density < 50%: 즉시 REINDEX CONCURRENTLY
  avg_leaf_density 50~70%: 모니터링 후 결정
  avg_leaf_density > 70%: 정상, REINDEX 불필요
```

---

## 📌 핵심 정리

```
인덱스 사용률 분석 핵심:

미사용 인덱스:
  pg_stat_user_indexes.idx_scan = 0
  → 충분한 기간(2주+) 후 확인
  → DROP INDEX CONCURRENTLY

중복 인덱스:
  선두 컬럼이 같은 복합 인덱스 → 서브셋 삭제
  완전 동일 인덱스 → 하나 삭제
  PK와 동일 UNIQUE 인덱스 → 삭제

인덱스 Bloat:
  pgstatindex로 avg_leaf_density 확인
  < 50%: REINDEX CONCURRENTLY 즉시
  50~70%: 모니터링

REINDEX CONCURRENTLY:
  운영 중 실행 가능 (잠금 최소화)
  실패 시 INVALID 인덱스 남음 → DROP 후 재시도
  트랜잭션 블록 안에서 실행 불가

캐시 히트율:
  pg_statio_user_indexes.idx_blks_hit / (hit + read)
  낮으면: 인덱스가 너무 크거나 shared_buffers 증가 필요
```

---

## 🤔 생각해볼 문제

**Q1.** `pg_stat_reset()`을 실행하면 모든 통계가 초기화된다. 미사용 인덱스를 판단할 때 통계 기간이 왜 중요한가?

<details>
<summary>해설 보기</summary>

`idx_scan = 0`이어도 다음 상황에서는 삭제하면 안 됩니다:

1. **계절적 트래픽**: 분기 보고서 쿼리용 인덱스는 분기마다 한 번 사용. 통계 기간이 2주라면 idx_scan = 0으로 나타날 수 있음.

2. **배치 작업 주기**: 월간 정산, 연간 세무 처리 등 주기적 배치에서만 사용하는 인덱스.

3. **비상 쿼리 인덱스**: 장애 발생 시 진단용 쿼리에서만 사용하는 인덱스. 평상 시 idx_scan = 0.

권장 분석 기간: 최소 2주, 이상적으로 1~3개월. 계절적 서비스는 1년 주기도 필요.

안전한 접근:
```sql
-- 최근 통계 초기화 시각 확인
SELECT stats_reset FROM pg_stat_bgwriter;

-- 통계 기간이 짧으면 재측정 예약
```

</details>

---

**Q2.** `REINDEX CONCURRENTLY` 실행 중 실패하면 남겨진 INVALID 인덱스가 쿼리에 영향을 주는가?

<details>
<summary>해설 보기</summary>

INVALID 상태의 인덱스는 **쿼리에 사용되지 않습니다**. 플래너가 INVALID 인덱스를 인식하고 무시합니다. 따라서 쿼리 결과에는 영향이 없지만, 기존 유효한 인덱스는 계속 사용됩니다.

단, INVALID 인덱스도 **쓰기 시에는 갱신을 시도**합니다. INSERT/UPDATE/DELETE 시 INVALID 인덱스에도 변경을 기록하려 하므로 쓰기 성능 저하가 발생합니다.

INVALID 인덱스 정리:
```sql
-- INVALID 인덱스 확인
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes i
JOIN pg_index pi ON pi.indexrelid = i.indexrelid
WHERE NOT pi.indisvalid;

-- INVALID 인덱스 삭제
DROP INDEX CONCURRENTLY idx_name_ccnew;

-- 다시 REINDEX 시도
REINDEX INDEX CONCURRENTLY original_index_name;
```

</details>

---

<div align="center">

**[⬅️ 이전: 쿼리 성능 진단](./02-query-diagnostics.md)** | **[홈으로 🏠](../README.md)** | **[다음: 운영 중 발생하는 문제 패턴 ➡️](./04-operational-issues.md)**

</div>
