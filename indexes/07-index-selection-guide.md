# 인덱스 선택 가이드 — 쿼리 패턴별 최적 인덱스 결정 트리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 쿼리 패턴을 보고 어떤 인덱스 타입을 선택해야 하는가?
- Expression Index(함수 기반 인덱스)는 어떤 쿼리에 사용하는가?
- 인덱스 유지 비용(쓰기 증폭)과 조회 이득을 어떻게 균형 잡는가?
- 복합 인덱스에서 컬럼 순서는 어떻게 결정하는가?
- 미사용 인덱스를 어떻게 찾고 제거하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

"어떤 인덱스를 만들어야 하는가"는 모든 PostgreSQL 운영의 핵심 질문이다. 인덱스가 너무 없으면 쿼리가 느리고, 너무 많으면 쓰기 성능이 저하되고 HOT Update 기회가 사라진다. 각 인덱스 타입(B-Tree, Hash, GiST, GIN, BRIN, Bloom)의 적합한 사용 사례를 이해하고, Expression Index와 Partial Index를 활용해 인덱스 수와 크기를 최소화하는 것이 목표다.

---

## 😱 흔한 실수 (Before — 인덱스 남발 또는 부족)

```
실수 1: 불필요한 인덱스 과다 생성

  CREATE INDEX ON orders(status);          -- 3개 값만 있음 (선택도 낮음)
  CREATE INDEX ON orders(user_id);         -- 이미 (user_id, created_at) 있음 (중복)
  CREATE INDEX ON orders(amount);          -- 실제 쿼리에서 사용 안 됨
  CREATE INDEX ON orders(status, user_id); -- (user_id, status)와 중복

  결과:
  INSERT/UPDATE 시 4개 인덱스 갱신 → 쓰기 성능 저하
  HOT Update 불가 (모든 컬럼에 인덱스)
  인덱스 총 크기: 테이블 크기의 2~3배

실수 2: LIKE 쿼리에 일반 인덱스

  CREATE INDEX ON users(email);
  SELECT * FROM users WHERE email LIKE '%@gmail.com';
  → 인덱스 사용 불가 (접두사가 없는 LIKE)

  올바른 접근:
  -- 접미사 검색이 필요하면 역방향 저장 + text_pattern_ops
  -- 또는 전문 검색 GIN

실수 3: lower() 변환 후 조회에 일반 인덱스

  CREATE INDEX ON users(email);
  SELECT * FROM users WHERE lower(email) = 'alice@example.com';
  → 인덱스 사용 불가 (lower() 변환 후 비교)

  올바른 접근:
  CREATE INDEX ON users(lower(email));  -- Expression Index
```

---

## ✨ 올바른 접근 (After — 체계적 인덱스 설계)

```
인덱스 설계 프로세스:

  Step 1: 쿼리 수집
    pg_stat_statements에서 느린 쿼리 수집
    EXPLAIN ANALYZE로 SeqScan 식별

  Step 2: 쿼리 패턴 분류
    등가 비교 (=): B-Tree 또는 Hash
    범위 비교 (>, <, BETWEEN): B-Tree
    JSONB/배열 포함: GIN
    공간/범위 타입: GiST
    시계열 범위: BRIN
    함수 변환 후 비교: Expression Index

  Step 3: 선택도 평가
    높은 선택도 (1% 이하 반환): 인덱스 효과적
    낮은 선택도 (10% 이상 반환): SeqScan이 더 빠를 수 있음
    → Partial Index로 실제 조회 대상만 인덱싱

  Step 4: 복합 인덱스 컬럼 순서 결정
    선택도 높은 컬럼을 선두에
    범위 조건 컬럼은 뒤에 (선두 등가 후 범위)
    ORDER BY 컬럼을 마지막에 (Index Scan + Sort 제거)

  Step 5: 미사용 인덱스 제거
    pg_stat_user_indexes.idx_scan = 0 → 삭제 검토
    중복/서브셋 인덱스 → 정리
```

---

## 🔬 내부 동작 원리

### 1. 인덱스 타입 선택 결정 트리

```
쿼리 패턴 → 인덱스 타입 결정 트리:

          쿼리의 WHERE 조건은?
                 │
    ┌────────────┼────────────┬─────────────┬──────────────┐
    │            │            │             │              │
  등가 비교     범위 비교    JSON/배열     공간/범위       시계열 범위
  (=)          (>,<,BETWEEN) (containment)  (겹침,포함)    (순차 삽입)
    │            │            │             │              │
  ┌─┴─┐          │           GIN          GiST           BRIN
  │   │          │
짧은 키  긴 키   B-Tree
B-Tree  Hash    복합 인덱스
또는          + 컬럼 순서
Hash          주의

          함수 변환 후 비교?
          → Expression Index

          일부 행만 조회?
          → Partial Index (WHERE 절 추가)

          여러 컬럼 임의 조합?
          → Bloom (쿼리 패턴 불규칙한 경우만)

실제 결정 예시:

  WHERE user_id = ? AND created_at > ?
  → B-Tree (user_id, created_at) — 선두 등가, 뒤에 범위

  WHERE lower(email) = ?
  → Expression Index: CREATE INDEX ON users(lower(email))

  WHERE data @> '{"type": "click"}'
  → GIN: CREATE INDEX ON events USING GIN(data)

  WHERE geom && ST_Buffer(point, 1000)
  → GiST: CREATE INDEX ON locations USING GIST(geom)

  WHERE logged_at BETWEEN ? AND ? (시계열 로그)
  → BRIN: CREATE INDEX ON logs USING BRIN(logged_at)

  WHERE status = 'PENDING' (1% 미만)
  → Partial: CREATE INDEX ON orders(id) WHERE status = 'PENDING'
```

### 2. Expression Index (함수 기반 인덱스)

```
Expression Index 원리:
  컬럼 값 대신 표현식 결과를 인덱싱

사용 사례:

  1. 대소문자 무관 검색:
  CREATE INDEX ON users(lower(email));
  SELECT * FROM users WHERE lower(email) = 'alice@example.com';
  → lower(email) 인덱스 사용

  2. 날짜 함수:
  CREATE INDEX ON orders(date_trunc('day', created_at));
  SELECT * FROM orders WHERE date_trunc('day', created_at) = '2024-01-15';

  3. JSONB 특정 경로:
  CREATE INDEX ON events((data->>'event_type'));
  SELECT * FROM events WHERE data->>'event_type' = 'click';

  4. 계산 컬럼:
  CREATE INDEX ON products(price * (1 - discount_rate));
  SELECT * FROM products WHERE price * (1 - discount_rate) < 10000;

주의사항:
  쿼리의 WHERE 절이 인덱스 표현식과 정확히 일치해야 함
  → CREATE INDEX ON users(lower(email))
  → WHERE lower(email) = ...  (OK)
  → WHERE email = ...  (인덱스 사용 불가)

  함수가 IMMUTABLE이어야 함:
  → VOLATILE 함수 (NOW(), random() 등): Expression Index 불가
  → IMMUTABLE 함수 (lower(), date_trunc() 등): 가능

  Expression Index + Partial Index 조합:
  CREATE INDEX ON orders(lower(customer_email))
  WHERE status = 'PENDING';
  → 미완료 주문의 이메일 대소문자 무관 검색에 최적
```

### 3. 복합 인덱스 컬럼 순서 전략

```
복합 인덱스 최적 컬럼 순서:

원칙 1: 등가 조건 컬럼을 범위 조건 컬럼보다 앞에

  WHERE user_id = 42 AND created_at > '2024-01-01'
  → (user_id, created_at): user_id 등가, created_at 범위 → 최적 ✓
  → (created_at, user_id): created_at 범위 먼저 → user_id 인덱스 활용 불가 ✗

원칙 2: 높은 선택도 컬럼을 앞에

  WHERE status = 'PENDING' AND user_id = 42
  status: 5개 값, user_id: 1만 명
  → (user_id, status): user_id가 더 선택적 → 더 적은 후보 ✓
  → (status, user_id): 큰 집합에서 시작 → 상대적으로 비효율

원칙 3: ORDER BY 컬럼을 마지막에 → Sort 제거

  WHERE user_id = 42 ORDER BY created_at DESC LIMIT 10
  → (user_id, created_at): 인덱스 순서가 ORDER BY와 일치
  → Sort 노드 제거 → 상위 10개만 읽고 종료

  EXPLAIN:
  Index Scan Backward on idx (user_id, created_at)
  → Sort 없음, LIMIT으로 바로 종료

원칙 4: Index-Only Scan을 위한 추가 컬럼

  SELECT user_id, amount FROM orders WHERE user_id = ?
  → (user_id, amount): amount를 인덱스에 포함 → Heap Fetch 없음
  → Covering Index 효과

중복 인덱스 정리:
  (a, b, c) 있으면 (a), (a, b)는 중복 → 삭제 가능
  단, (b) 또는 (c)는 별도 인덱스 필요 (선두 컬럼이 다름)
```

### 4. 인덱스 비용-이득 분석

```
인덱스 추가 비용:
  ① 쓰기 증폭: INSERT/UPDATE/DELETE 시 인덱스 페이지도 수정
     인덱스 1개당 쓰기 비용 약 1.1~1.5배 증가

  ② HOT Update 방해:
     인덱스 컬럼을 UPDATE하면 인덱스 갱신 필수
     → HOT Update 불가 → 인덱스 추가 쓰기

  ③ 디스크 공간: B-Tree는 테이블의 10~30%

  ④ VACUUM 부하: Dead 인덱스 엔트리 정리 비용

인덱스 이득:
  ① SeqScan → Index Scan: 큰 테이블에서 수십~수백 배 빠름
  ② Index-Only Scan: Heap Fetch 제거 → I/O 최소화
  ③ Sort 제거: ORDER BY + LIMIT에서 인덱스 순서 활용

비용-이득 판단 기준:
  읽기 : 쓰기 비율이 높을수록 인덱스가 유리
  테이블 크기가 클수록 인덱스 효과 큼
  선택도가 높을수록 인덱스 효과 큼 (1% 이하 결과)

  공식적 분석:
  SELECT
      schemaname,
      relname,
      indexrelname,
      idx_scan AS scans,
      idx_tup_read AS rows_read,
      pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
      CASE
          WHEN idx_scan = 0 THEN '⚠️ 미사용'
          WHEN idx_scan < 100 THEN '📊 드물게 사용'
          ELSE '✅ 활발히 사용'
      END AS usage_status
  FROM pg_stat_user_indexes
  ORDER BY idx_scan, pg_relation_size(indexrelid) DESC;
```

---

## 💻 실전 실험

### 실험 1: Expression Index 효과

```sql
-- 대소문자 무관 이메일 검색
CREATE TABLE users_test (
    id SERIAL PRIMARY KEY,
    email TEXT,
    status TEXT
);

INSERT INTO users_test (email, status)
SELECT
    'user' || i || '@' || (ARRAY['gmail.com','yahoo.com','naver.com'])[ceil(random()*3)::INT],
    (ARRAY['ACTIVE','INACTIVE','PENDING'])[ceil(random()*3)::INT]
FROM generate_series(1, 500000) i;

-- 일반 인덱스 (lower() 적용 쿼리에 효과 없음)
CREATE INDEX ON users_test(email);

EXPLAIN SELECT * FROM users_test WHERE lower(email) = 'user42@gmail.com';
-- Seq Scan! (lower() 변환으로 인덱스 사용 불가)

-- Expression Index (lower() 포함)
CREATE INDEX ON users_test(lower(email));

EXPLAIN SELECT * FROM users_test WHERE lower(email) = 'user42@gmail.com';
-- Index Scan! (Expression Index 사용)
```

### 실험 2: 복합 인덱스 컬럼 순서 비교

```sql
CREATE TABLE order_search (
    id SERIAL PRIMARY KEY,
    user_id INT,
    status TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    amount NUMERIC
);

INSERT INTO order_search (user_id, status, created_at, amount)
SELECT
    (random() * 10000)::INT,
    (ARRAY['PENDING','PROCESSING','DONE'])[ceil(random()*3)::INT],
    NOW() - (random() * INTERVAL '365 days'),
    (random() * 10000)::NUMERIC
FROM generate_series(1, 1000000);

-- 두 가지 순서로 인덱스 생성
CREATE INDEX idx_user_date ON order_search(user_id, created_at);
CREATE INDEX idx_date_user ON order_search(created_at, user_id);

-- 쿼리: user_id 등가 + created_at 범위
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM order_search
WHERE user_id = 42 AND created_at > NOW() - INTERVAL '30 days';
-- idx_user_date 사용: 효율적 (user_id로 먼저 필터)

-- ORDER BY 포함 쿼리
CREATE INDEX idx_user_date_covering
ON order_search(user_id, created_at, amount);

EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, created_at, amount
FROM order_search
WHERE user_id = 42
ORDER BY created_at DESC LIMIT 10;
-- Index Only Scan: Sort 없음, amount도 인덱스에서
```

### 실험 3: 미사용 인덱스 찾기

```sql
-- 통계 초기화 후 워크로드 실행
SELECT pg_stat_reset();

-- 일정 기간 후 미사용 인덱스 확인
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- 중복 인덱스 찾기 (선두 컬럼이 서브셋인 경우)
SELECT
    a.indexrelname AS index_a,
    b.indexrelname AS index_b,
    a.indkey::text AS cols_a,
    b.indkey::text AS cols_b
FROM pg_index a
JOIN pg_index b ON a.indrelid = b.indrelid
    AND a.indexrelid <> b.indexrelid
    AND b.indkey::int[] @> a.indkey::int[]
    AND array_length(a.indkey::int[], 1) < array_length(b.indkey::int[], 1)
JOIN pg_class ca ON ca.oid = a.indexrelid
JOIN pg_class cb ON cb.oid = b.indexrelid
ORDER BY a.indrelid;
```

---

## 📊 MySQL과 비교

```
인덱스 전략 차이:

MySQL:
  Covering Index: SELECT 컬럼이 인덱스에 포함되면 Extra: Using index
  Prefix Index: TEXT 컬럼의 앞 N자만 인덱싱 (크기 절약)
  Functional Index (8.0+): Expression Index와 유사
  Index Merge: 여러 인덱스 조합 (AND, OR)

PostgreSQL:
  Index-Only Scan: Covering Index 효과 + VM 비트 활용
  Expression Index: 함수 결과 인덱싱
  Partial Index: 조건부 인덱싱 (MySQL에 없음)
  여러 인덱스 타입: B-Tree, Hash, GiST, GIN, BRIN, Bloom

Partial Index (PostgreSQL 강점):
  MySQL: 없음 → 전체 행 인덱싱 필요
  PostgreSQL: WHERE status = 'PENDING' → 1%만 인덱싱

  예:
  1,000만 행, 1%만 PENDING
  MySQL: 전체 1,000만 행 인덱스 = 500MB
  PostgreSQL Partial: 10만 행 인덱스 = 5MB (100배 작음)
```

---

## ⚖️ 트레이드오프

```
인덱스 수와 성능:

인덱스 너무 적음:
  SeqScan 빈번 → 읽기 느림
  대용량 테이블 조회 타임아웃

인덱스 너무 많음:
  쓰기 증폭 심각 → INSERT/UPDATE 느림
  HOT Update 불가
  VACUUM 부하 증가 (더 많은 Index Vacuum)
  메모리/디스크 낭비

황금 균형:
  실제 쿼리 기반 (pg_stat_statements 분석)
  미사용 인덱스 제거 (idx_scan = 0)
  Partial Index로 크기 최소화
  복합 인덱스로 단순 인덱스 대체
  Expression Index로 SeqScan 원인 해결

인덱스 검토 주기:
  월 1회: 미사용 인덱스 확인 및 제거
  분기: 느린 쿼리 분석 → 신규 인덱스 검토
  반기: 전체 인덱스 크기 대비 사용률 리뷰
```

---

## 📌 핵심 정리

```
인덱스 선택 가이드:

타입별 적합 사례:
  B-Tree: 기본 선택, 등가/범위/정렬/LIKE prefix%
  Hash:   긴 키 순수 등가 비교 (UUID, 해시값)
  GiST:   공간(PostGIS), 범위 타입, EXCLUDE 제약
  GIN:    JSONB, 배열, 전문 검색 역색인
  BRIN:   시계열 Append-Only 범위 쿼리 (초경량)
  Bloom:  여러 컬럼 임의 등가 조합 (쿼리 패턴 가변)

최적화 기법:
  Partial Index: WHERE 절로 일부만 인덱싱 (크기 최소화)
  Expression Index: 함수 결과 인덱싱 (lower, date_trunc 등)
  Covering Index: SELECT 컬럼 포함 → Index-Only Scan

복합 인덱스 순서:
  등가 조건 먼저 → 범위 조건 나중
  선택도 높은 컬럼 먼저
  ORDER BY 컬럼 마지막 (Sort 제거)

유지 관리:
  idx_scan = 0 → 미사용 인덱스 삭제 검토
  중복 인덱스 제거
  주기적 인덱스 사용률 검토
```

---

## 🤔 생각해볼 문제

**Q1.** `WHERE status = 'ACTIVE' AND user_id = ?` 쿼리에서 `(status, user_id)`와 `(user_id, status)` 중 어떤 인덱스가 더 효율적인가?

<details>
<summary>해설 보기</summary>

일반적으로 `(user_id, status)`가 더 효율적입니다. 이유:

- `user_id`는 많은 고유값(예: 100만 사용자) → 선택도 높음
- `status`는 적은 값('ACTIVE', 'INACTIVE', 'PENDING' 등) → 선택도 낮음

`(user_id, status)` 인덱스:
1. user_id=42인 행 먼저 찾기 → 몇 개 행
2. 그 중 status='ACTIVE' 필터 → 최종 결과

`(status, user_id)` 인덱스:
1. status='ACTIVE'인 행 먼저 찾기 → 전체의 33% (3개 상태 중 1개)
2. 그 중 user_id=42 필터

user_id로 먼저 필터링하면 훨씬 적은 후보에서 시작하므로 `(user_id, status)`가 유리합니다.

단, `WHERE status = 'ACTIVE'` 단독 쿼리도 있다면 `(status, user_id)`도 있어야 합니다(선두 컬럼이 없으면 인덱스 활용 불가).

</details>

---

**Q2.** 동일한 테이블에 `(a, b)` 복합 인덱스와 `(a)` 단순 인덱스가 모두 있다면 중복인가?

<details>
<summary>해설 보기</summary>

대부분의 경우 `(a)` 인덱스는 중복입니다. B-Tree `(a, b)` 인덱스는 선두 컬럼인 `a`만으로도 인덱스 스캔이 가능합니다(`WHERE a = 42`는 `(a, b)` 인덱스 활용 가능).

단, 예외 상황:
1. `(a)` 인덱스가 훨씬 작아서 캐시 효율이 높은 경우
2. `WHERE a = ?`가 매우 빈번하고 `(a, b)`는 주로 `WHERE a = ? AND b = ?`에만 사용되는 경우
3. `(a)` 인덱스로 Index-Only Scan이 가능한데 `(a, b)` 인덱스는 Heap Fetch 필요한 경우

하지만 일반적으로 `(a)` 인덱스는 제거하고 `(a, b)` 인덱스로 대체합니다:
```sql
-- 중복 인덱스 제거 예시
DROP INDEX CONCURRENTLY idx_table_a;  -- (a, b)가 이 역할을 대신
-- idx_table_a_b는 유지
```

</details>

---

<div align="center">

**[⬅️ 이전: Bloom 필터 인덱스](./06-bloom-filter-index.md)** | **[홈으로 🏠](../README.md)** | **[다음: 실행 계획 분석 심화 ➡️](./08-explain-analyze.md)**

</div>
