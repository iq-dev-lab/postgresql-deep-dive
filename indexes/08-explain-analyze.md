# 실행 계획 분석 심화 — EXPLAIN ANALYZE 완전 해독

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `EXPLAIN ANALYZE`의 각 노드(SeqScan, IndexScan, BitmapHeapScan, Hash Join, Merge Join, Nested Loop)의 의미와 비용은?
- 플래너가 인덱스를 무시하는 조건은 무엇인가?
- `rows=100 actual rows=50000` 같은 추정치 오류를 어떻게 수정하는가?
- Buffers, WAL, Planning Time vs Execution Time을 어떻게 해석하는가?
- `pg_stats` 통계 정보로 플래너 정확도를 어떻게 높이는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

쿼리가 왜 느린지 파악하는 핵심 도구는 `EXPLAIN ANALYZE`다. 인덱스를 만들었는데도 SeqScan이 발생하거나, 예상보다 훨씬 많은 행이 반환되는 경우, 플래너의 비용 추정이 틀렸다는 신호다. `EXPLAIN ANALYZE` 출력의 각 노드를 정확히 해석하고, 통계 갱신과 힌트로 플래너를 조정하는 방법을 아는 것이 PostgreSQL 성능 튜닝의 핵심이다.

---

## 😱 흔한 실수 (Before — EXPLAIN 결과 오해)

```
실수 1: EXPLAIN만 보고 EXPLAIN ANALYZE를 생략

  EXPLAIN SELECT * FROM orders WHERE user_id = 42;
  → rows=10 (플래너 추정)

  실제:
  EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
  → rows=10 actual rows=500000 (추정 오류!)

  EXPLAIN은 실제 실행 없이 계획만 보여줌
  실제 실행 시간과 행 수는 ANALYZE 필요

실수 2: cost 값을 절대적 시간으로 오해

  cost=0.00..1234.56 → "1234초?" → 아님!
  cost는 페이지 I/O 비용을 기준으로 한 상대적 단위
  실제 시간은 actual time으로 확인 (ms 단위)

실수 3: 인덱스가 없어서 SeqScan이라고 단정

  SeqScan 원인:
  ① 인덱스 없음 → 인덱스 생성
  ② 선택도 낮음 → SeqScan이 더 빠름 (정상)
  ③ 통계 부족 → 플래너가 잘못 판단 → ANALYZE 실행
  ④ enable_indexscan = off → 설정 확인
  ⑤ 함수 변환 후 비교 → Expression Index 필요
```

---

## ✨ 올바른 접근 (After — EXPLAIN ANALYZE 올바른 활용)

```
완전한 EXPLAIN 옵션 활용:

  EXPLAIN (
      ANALYZE,    -- 실제 실행 (실제 행 수, 시간 포함)
      BUFFERS,    -- I/O 통계 (shared/local hit/read)
      VERBOSE,    -- 컬럼 목록, 스키마 이름 등 상세 정보
      FORMAT TEXT -- 또는 JSON (프로그래밍 파싱 시)
  )
  SELECT * FROM orders WHERE user_id = 42;

  Buffers 해석:
    shared hit: shared_buffers에서 읽음 (빠름, 디스크 I/O 없음)
    shared read: 디스크에서 읽음 (느림, 캐시 미스)
    shared dirtied: 수정된 페이지 수
    shared written: 디스크에 쓴 페이지 수

  이상적인 상태:
    shared hit >> shared read → 캐시 적중률 높음

  성능 병목 신호:
    shared read 많음 → 캐시 부족 (shared_buffers 증가 검토)
    rows 추정 >> actual rows → ANALYZE 필요
    actual rows 0인데 오래 걸림 → Lock 대기 가능성
```

---

## 🔬 내부 동작 원리

### 1. 주요 노드 해석

```
=== Seq Scan (순차 스캔) ===

Seq Scan on orders  (cost=0.00..18343.00 rows=1000000 width=45)
                    (actual time=0.015..245.023 rows=1000000 loops=1)
  Buffers: shared hit=8343

해석:
  cost=0.00: 시작 비용 (첫 행 반환 전 비용)
  ..18343.00: 전체 완료 비용
  rows=1000000: 예상 반환 행 수
  width=45: 예상 행 크기 (bytes)
  actual time=0.015ms (첫 행) .. 245ms (마지막 행)
  rows=1000000: 실제 반환 행 수 (추정과 일치)
  loops=1: 이 노드 실행 횟수

SeqScan이 적절한 경우:
  → 전체 데이터의 10% 이상을 반환하는 쿼리
  → 인덱스 있어도 SeqScan이 빠를 수 있음 (비용 모델 기준)

=== Index Scan ===

Index Scan using idx_user_id on orders
  (cost=0.43..8.45 rows=1 width=45)
  (actual time=0.052..0.055 rows=1 loops=1)
  Index Cond: (user_id = 42)
  Buffers: shared hit=4

해석:
  cost=0.43: B-Tree 탐색 시작 비용
  ..8.45: 완료 비용 (페이지 접근 포함)
  Buffers hit=4: 인덱스 2페이지 + Heap 2페이지

Index Scan vs Seq Scan 선택:
  rows가 적으면 Index Scan (낮은 선택도)
  rows가 많으면 Seq Scan (높은 선택도)
  임계값: 약 10~20% (default_statistics_target에 따라 변동)

=== Index Only Scan ===

Index Only Scan using idx_user_amount on orders
  (cost=0.43..5.45 rows=1 width=12)
  (actual time=0.031..0.033 rows=1 loops=1)
  Index Cond: (user_id = 42)
  Heap Fetches: 0

해석:
  Heap Fetches: 0 → Heap을 전혀 읽지 않음!
  → 인덱스에 필요한 컬럼이 모두 있음 + VM All-Visible

=== Bitmap Index Scan + Bitmap Heap Scan ===

Bitmap Heap Scan on orders
  (cost=234.15..5892.23 rows=10000 width=45)
  (actual time=1.234..45.678 rows=9876 loops=1)
  Recheck Cond: (user_id = 42)
  Heap Blocks: exact=320
  Buffers: shared hit=345 shared read=17

  →  Bitmap Index Scan on idx_user_id
       (cost=0.00..231.65 rows=10000 width=0)
       (actual time=0.987..0.987 rows=9876 loops=1)
       Index Cond: (user_id = 42)
       Buffers: shared hit=25

해석:
  1단계 (Bitmap Index Scan):
    인덱스를 스캔하여 TID 비트맵 생성
    정렬된 순서로 페이지 목록 생성 (랜덤 I/O → 순차 I/O)
  2단계 (Bitmap Heap Scan):
    비트맵의 페이지 순서대로 Heap 읽기
    Recheck: 정확한 조건 재확인

  Heap Blocks: exact=320 → 320개 페이지를 정확하게 읽음
  (lossy가 있으면: lossy=X → 비트맵이 손실됐음)

  언제 Bitmap Scan:
  Index Scan보다 많은 행 (수백~수천)
  단일 페이지를 여러 TID가 가리킬 때 효율적
```

### 2. 조인 노드 해석

```
=== Nested Loop Join ===

Nested Loop  (cost=...) (actual time=... rows=... loops=1)
  ->  Seq Scan on users  (... rows=100 loops=1)
  ->  Index Scan on orders  (... rows=5 loops=100)
       Index Cond: (user_id = users.id)

해석:
  외부 루프: users 100행
  내부 루프: 각 user마다 orders를 Index Scan (100번)
  total loops: 100 (내부 노드의 loops=100)

  적합한 경우: 외부 테이블이 작고, 내부 인덱스가 있을 때
  비효율적: 외부 테이블이 크고, 내부에 인덱스 없을 때

=== Hash Join ===

Hash Join  (cost=...) (actual time=... rows=... loops=1)
  Hash Cond: (orders.user_id = users.id)
  ->  Seq Scan on orders  (... rows=1000000)
  ->  Hash  (actual time=12.345..12.345 rows=10000 loops=1)
         Buckets: 16384  Batches: 1  Memory Usage: 512kB
        ->  Seq Scan on users  (... rows=10000)

해석:
  1단계: users를 읽어 Hash Table 생성 (내부 테이블)
  2단계: orders를 읽으면서 Hash Table 조회 (외부 테이블)
  Memory Usage: 512kB → work_mem 내에서 처리됨
  Batches: 1 → 메모리 내 처리 (Batches > 1이면 디스크 spill!)

  Batches > 1 경우:
  work_mem 부족 → 디스크에 일부 해시 저장 → 성능 저하
  해결: work_mem 증가 또는 쿼리 최적화

=== Merge Join ===

Merge Join  (cost=...) (actual time=... rows=...)
  Merge Cond: (orders.user_id = users.id)
  ->  Sort  (... actual rows=... sorted=true)
        Sort Key: orders.user_id
  ->  Index Scan using idx_user_id on users

해석:
  양쪽 입력이 조인 키로 정렬된 경우 효율적
  인덱스가 있으면 Sort 생략 가능
  주로 양쪽이 인덱스로 정렬 상태일 때 플래너가 선택
```

### 3. 플래너가 인덱스를 무시하는 이유

```
인덱스가 있어도 SeqScan을 선택하는 경우:

① 선택도가 낮음 (너무 많은 행 반환):
   WHERE status = 'DONE' (90% 행)
   → SeqScan이 더 빠름 (인덱스 + Heap Fetch 오버헤드 > SeqScan)

② 통계 부족 (추정치 오류):
   rows=100 actual rows=50000 → 플래너가 잘못 판단
   해결: ANALYZE table_name;

③ 함수 변환:
   WHERE lower(email) = ...  (일반 email 인덱스로 불가)
   해결: Expression Index

④ 타입 불일치:
   WHERE user_id = '42'  (INT 컬럼에 TEXT 값)
   → 타입 캐스팅으로 인덱스 사용 불가
   해결: WHERE user_id = 42 (타입 일치)

⑤ enable_indexscan = off:
   SHOW enable_indexscan;
   SET enable_indexscan = on;

⑥ 인덱스 카디널리티 통계 오래됨:
   ANALYZE table_name;

⑦ 부등호 방향이 인덱스 순서와 반대:
   (드문 경우) 플래너가 Backward Scan 비용 과대 추정

강제로 인덱스 사용 (테스트 목적):
  SET enable_seqscan = off;
  EXPLAIN SELECT ...;
  SET enable_seqscan = on;  -- 반드시 복원
```

### 4. 통계 정보와 플래너 정확도

```
pg_statistics (pg_stats 뷰):
  각 컬럼의 통계 정보 → 플래너가 비용 추정에 활용

주요 통계 컬럼:
  null_frac:     NULL 비율
  avg_width:     평균 값 크기 (bytes)
  n_distinct:    고유 값 수 (음수면 비율)
  most_common_vals:   가장 흔한 값 목록
  most_common_freqs:  각 값의 빈도
  histogram_bounds:   값 분포 히스토그램

통계 확인:
  SELECT
      attname,
      n_distinct,
      most_common_vals,
      most_common_freqs,
      histogram_bounds
  FROM pg_stats
  WHERE tablename = 'orders' AND attname = 'status';

통계 갱신:
  ANALYZE orders;  -- 특정 테이블
  ANALYZE;         -- 전체

default_statistics_target (기본 100):
  히스토그램 버킷 수 = 이 값
  높을수록 정확한 통계 → 플래너 정확도 향상
  낮을수록 빠른 ANALYZE

특정 컬럼 통계 품질 높이기:
  ALTER TABLE orders ALTER COLUMN amount
  SET STATISTICS 500;  -- 이 컬럼만 500 버킷
  ANALYZE orders;

왜 statistics_target을 늘려야 하는가:
  쿼리가 특정 범위의 amount 값 조회
  → 히스토그램 버킷이 적으면 추정 부정확
  → rows=100 actual rows=5000 오류
  → statistics_target 증가 → 더 정확한 히스토그램
```

---

## 💻 실전 실험

### 실험 1: 주요 노드 관찰

```sql
-- 실험 테이블
CREATE TABLE explain_test (
    id SERIAL PRIMARY KEY,
    user_id INT,
    status TEXT,
    amount NUMERIC,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON explain_test(user_id);
CREATE INDEX ON explain_test(status);
CREATE INDEX ON explain_test(user_id, amount);

INSERT INTO explain_test (user_id, status, amount, created_at)
SELECT
    (random() * 10000)::INT,
    (ARRAY['PENDING','DONE','FAILED'])[ceil(random()*3)::INT],
    (random() * 10000)::NUMERIC,
    NOW() - (random() * INTERVAL '365 days')
FROM generate_series(1, 500000);

ANALYZE explain_test;

-- SeqScan (선택도 낮음: 33%)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM explain_test WHERE status = 'DONE';

-- Index Scan (선택도 높음: 0.01%)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM explain_test WHERE user_id = 42;

-- Index Only Scan
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, amount FROM explain_test WHERE user_id = 42;

-- Bitmap Heap Scan (중간 선택도)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM explain_test WHERE user_id BETWEEN 100 AND 200;
```

### 실험 2: 통계 오류로 인한 잘못된 실행 계획

```sql
-- 편향된 데이터 (90%가 user_id=1)
CREATE TABLE skewed_data (id SERIAL, user_id INT, val TEXT);
INSERT INTO skewed_data (user_id, val)
SELECT
    CASE WHEN i <= 900000 THEN 1 ELSE (random() * 9999 + 2)::INT END,
    md5(i::text)
FROM generate_series(1, 1000000) i;

CREATE INDEX ON skewed_data(user_id);

-- ANALYZE 없이 통계 오래됨
EXPLAIN SELECT * FROM skewed_data WHERE user_id = 1;
-- rows 추정이 부정확할 수 있음

-- ANALYZE 후 개선
ANALYZE skewed_data;
EXPLAIN SELECT * FROM skewed_data WHERE user_id = 1;
-- 더 정확한 rows 추정

-- 특정 컬럼 statistics_target 증가
ALTER TABLE skewed_data ALTER COLUMN user_id SET STATISTICS 500;
ANALYZE skewed_data;

-- pg_stats 확인
SELECT n_distinct, most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'skewed_data' AND attname = 'user_id';
```

### 실험 3: 조인 노드 관찰

```sql
-- 소규모 테이블 (Nested Loop 유도)
CREATE TABLE small_users AS
SELECT generate_series(1, 100) AS id, md5(random()::text) AS name;

-- 대규모 테이블 (Hash Join 유도)
CREATE TABLE large_orders AS
SELECT
    generate_series(1, 1000000) AS id,
    (random() * 100)::INT AS user_id,
    (random() * 1000)::NUMERIC AS amount;

CREATE INDEX ON small_users(id);
CREATE INDEX ON large_orders(user_id);

ANALYZE small_users;
ANALYZE large_orders;

-- Nested Loop (소규모 × 인덱스)
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.name, o.amount
FROM small_users u
JOIN large_orders o ON o.user_id = u.id
WHERE u.id <= 10;

-- Hash Join (대규모)
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.name, sum(o.amount)
FROM small_users u
JOIN large_orders o ON o.user_id = u.id
GROUP BY u.name;
```

---

## 📊 MySQL과 비교

```
실행 계획 분석 도구 비교:

MySQL:
  EXPLAIN: 기본 실행 계획 (실제 실행 없음)
  EXPLAIN ANALYZE (8.0.18+): 실제 실행 포함 (PostgreSQL과 유사)
  EXPLAIN FORMAT=JSON: JSON 형식 출력
  SHOW PROFILE: 구식, deprecated
  performance_schema: 상세 쿼리 프로파일링

  MySQL EXPLAIN 컬럼:
    type: ALL(SeqScan), index, ref, eq_ref, const
    key: 사용된 인덱스
    rows: 예상 검사 행 수
    Extra: Using index(커버링), Using filesort(정렬), Using temporary

PostgreSQL EXPLAIN:
  노드 기반 트리 구조
  actual time, actual rows, loops 포함 (ANALYZE 시)
  Buffers: I/O 통계 (BUFFERS 옵션)
  WAL: WAL 생성 통계 (WAL 옵션)

PostgreSQL 강점:
  Buffers 통계로 캐시 효율 직접 확인 가능
  actual rows vs rows 비교로 통계 오류 즉시 파악
  노드별 실행 시간 → 병목 노드 정확한 식별
```

---

## ⚖️ 트레이드오프

```
EXPLAIN ANALYZE 사용 시 주의:

  ANALYZE는 실제로 쿼리를 실행:
  → SELECT: 실행해도 안전
  → INSERT/UPDATE/DELETE: ANALYZE 후 롤백 필요
    BEGIN;
    EXPLAIN ANALYZE DELETE FROM orders WHERE id = 1;
    ROLLBACK;

  실행 계획은 데이터와 통계에 따라 변한다:
  → 개발 환경 실행 계획 ≠ 운영 환경 실행 계획
  → 데이터 분포, shared_buffers 상태, work_mem 설정 등에 영향

  플래너 파라미터 강제 (테스트 목적):
    SET enable_seqscan = off;  -- SeqScan 비활성화
    SET enable_hashjoin = off; -- Hash Join 비활성화
    → 특정 플랜 강제 후 비교
    → 운영 환경에서는 반드시 복원

auto_explain 확장 (운영 환경):
  운영에서 느린 쿼리 자동으로 EXPLAIN ANALYZE 기록
  CREATE EXTENSION auto_explain;
  SET auto_explain.log_min_duration = '1s';
  SET auto_explain.log_analyze = true;
  → 1초 이상 걸린 쿼리를 PostgreSQL 로그에 자동 기록
```

---

## 📌 핵심 정리

```
EXPLAIN ANALYZE 핵심:

옵션:
  ANALYZE: 실제 실행 (actual time, actual rows)
  BUFFERS: I/O 통계 (shared hit/read)
  VERBOSE: 상세 정보

주요 노드:
  Seq Scan: 전체 스캔
  Index Scan: 인덱스 + Heap Fetch
  Index Only Scan: 인덱스만 (Heap Fetch 없음)
  Bitmap Heap Scan: 비트맵으로 정렬 후 Heap 읽기
  Nested Loop: 외부 × 내부 반복 조인
  Hash Join: 해시 테이블 빌드 후 조인
  Merge Join: 정렬 후 병합 조인

비용 해석:
  cost=시작..완료: 상대적 단위 (절대 시간 아님)
  rows=N: 플래너 예상 행 수
  actual rows=M: 실제 행 수 (N≪M이면 통계 오류!)
  Buffers hit/read: 캐시 적중률 확인

플래너 오류 수정:
  rows 오차 → ANALYZE 실행
  statistics_target 높이기 (특정 컬럼)
  Expression Index (함수 변환 컬럼)
```

---

## 🤔 생각해볼 문제

**Q1.** `EXPLAIN ANALYZE`에서 `actual rows=0 loops=1000`인 Nested Loop 노드는 어떤 의미인가?

<details>
<summary>해설 보기</summary>

Nested Loop 내부 노드에서 `loops=1000`은 외부 테이블의 1000개 행에 대해 내부 노드가 1000번 실행됐다는 의미입니다. `actual rows=0`은 각 실행에서 평균 0개 행이 반환됐다는 뜻입니다.

이 상황이 문제가 되는 경우:
1. 내부 노드가 인덱스 없이 SeqScan을 1000번 실행 → 심각한 성능 문제
2. 조인 조건이 맞는 행이 없어서 결과가 0개이지만, 여전히 1000번 스캔

해결 방법:
- 조인 조건 컬럼에 인덱스 생성
- 또는 Hash Join이 더 효율적인지 검토 (`SET enable_nestloop = off`로 비교)

`total actual rows = actual rows × loops`임을 항상 기억하세요.

</details>

---

**Q2.** Hash Join에서 `Batches: 4`가 나타났다. 어떤 문제이고 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

`Batches: 4`는 Hash Table이 `work_mem`에 맞지 않아 **4번에 나눠서 처리**했다는 의미입니다. 해시 테이블의 일부를 디스크에 저장하고(spill), 여러 패스로 조인을 완료합니다. 성능이 크게 저하됩니다.

해결 방법:
1. `work_mem` 증가:
```sql
SET work_mem = '256MB';  -- 세션 수준
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- Batches: 1로 변경되는지 확인
```

2. 필요한 최소 `work_mem` 추정:
```
Memory Usage: 128MB × 4 batches = ~512MB 필요
work_mem = 512MB로 설정 → Batches: 1
```

3. `work_mem`을 전역으로 올리면 모든 쿼리에 영향 → 동시 연결 수 × work_mem 주의.

4. 대안: 쿼리 리팩토링으로 Hash Table 크기 줄이기 (불필요한 컬럼 제거).

</details>

---

<div align="center">

**[⬅️ 이전: 인덱스 선택 가이드](./07-index-selection-guide.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 쿼리 플래너와 통계 ➡️](../query-planner/01-cost-model.md)**

</div>
