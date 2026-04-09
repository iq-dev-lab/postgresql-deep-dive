# LATERAL JOIN — 각 행을 참조하는 서브쿼리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- LATERAL JOIN은 일반 JOIN과 어떻게 다른가? 왜 "각 행을 참조"할 수 있는가?
- `LATERAL + LIMIT`으로 각 사용자의 최근 N개 주문을 가져오는 패턴은?
- 상관 서브쿼리(Correlated Subquery)와 LATERAL JOIN의 차이는?
- LATERAL이 반드시 필요한 쿼리 패턴은 무엇인가?
- LATERAL JOIN의 실행 계획에서 어떤 노드가 나타나는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

"각 사용자별 최근 주문 3개를 조회하라" 같은 요구는 서비스 개발에서 빈번하다. 윈도우 함수로 해결할 수 있지만, 서브쿼리에서 외부 테이블의 컬럼을 참조해야 하는 경우 LATERAL JOIN이 필요하다. LATERAL은 서브쿼리가 왼쪽(외부) 행의 값을 참조할 수 있게 해주는 SQL 표준 기능이다. 함수 호출, JSON 분해, Top-N 조회 패턴에서 LATERAL은 다른 방법으로 대체하기 어렵다.

---

## 😱 흔한 실수 (Before — LATERAL 없이 복잡한 쿼리)

```
실수 1: 각 사용자별 최근 주문 3개 — 비효율적 방법

  방법 1: 상관 서브쿼리 (N+1 문제)
  SELECT u.id, u.name,
      (SELECT o.id FROM orders o WHERE o.user_id = u.id
       ORDER BY o.created_at DESC LIMIT 1) AS last_order
  FROM users u;
  → 사용자 1명당 서브쿼리 1회 → N명이면 N번 실행

  방법 2: 윈도우 함수 (전체 스캔)
  SELECT u.id, u.name, o.id AS order_id
  FROM users u
  JOIN (
      SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
      FROM orders
  ) o ON o.user_id = u.id AND o.rn <= 3;
  → orders 전체 스캔 후 ROW_NUMBER → 많은 메모리/정렬 비용
  → 특정 사용자만 조회해도 전체 처리

  LATERAL 사용:
  SELECT u.id, u.name, recent_orders.*
  FROM users u
  LEFT JOIN LATERAL (
      SELECT id, amount, created_at
      FROM orders
      WHERE user_id = u.id          ← u.id 참조!
      ORDER BY created_at DESC
      LIMIT 3
  ) recent_orders ON true;
  → 각 사용자마다 최신 3개만 조회 → 효율적

실수 2: 함수 반환 결과를 JOIN할 때 LATERAL 몰라서 실패

  -- jsonb_array_elements: JSONB 배열을 행으로 변환
  SELECT id, element
  FROM events, jsonb_array_elements(payload->'tags') AS t(element);
  -- 이 경우 FROM 절의 함수는 암묵적으로 LATERAL처럼 동작

  -- 명시적 LATERAL:
  SELECT e.id, tag
  FROM events e
  JOIN LATERAL jsonb_array_elements_text(e.payload->'tags') AS tag ON true;
```

---

## ✨ 올바른 접근 (After — LATERAL 활용 패턴)

```
LATERAL이 필요한 핵심 패턴:

  1. 각 행의 Top-N 조회:
     SELECT u.id, o.*
     FROM users u
     LEFT JOIN LATERAL (
         SELECT * FROM orders
         WHERE user_id = u.id
         ORDER BY created_at DESC
         LIMIT 3
     ) o ON true;

  2. 각 행의 함수 호출 결과:
     SELECT id, stats.*
     FROM products p
     JOIN LATERAL calculate_discount(p.price, p.category) AS stats ON true;

  3. JSON 배열 분해 후 필터:
     SELECT e.id, tag
     FROM events e
     JOIN LATERAL jsonb_array_elements_text(e.data->'tags') AS tag ON true
     WHERE tag LIKE 'pg%';

  4. 조건부 최근 관련 데이터:
     SELECT c.id, c.name,
            latest_order.amount AS last_purchase
     FROM customers c
     LEFT JOIN LATERAL (
         SELECT amount FROM orders
         WHERE customer_id = c.id AND status = 'DONE'
         ORDER BY created_at DESC LIMIT 1
     ) latest_order ON true;
```

---

## 🔬 내부 동작 원리

### 1. LATERAL의 의미와 일반 서브쿼리와의 차이

```
일반 서브쿼리 (외부 참조 불가):

  SELECT * FROM users u
  JOIN (
      SELECT * FROM orders WHERE amount > 100  -- u.id 참조 불가!
  ) o ON o.user_id = u.id;

  → 서브쿼리는 외부 FROM 절의 다른 항목을 참조할 수 없음
  → orders를 한 번에 전체 스캔, 그 후 u.id로 JOIN

LATERAL 서브쿼리 (외부 행 참조 가능):

  SELECT * FROM users u
  JOIN LATERAL (
      SELECT * FROM orders WHERE user_id = u.id  -- u.id 참조 가능!
      ORDER BY created_at DESC LIMIT 3
  ) o ON true;

  → "users의 각 행마다 별도로 서브쿼리 실행"
  → 각 u.id를 서브쿼리에 전달 → orders에서 해당 user의 최신 3개 조회

  핵심: LATERAL 서브쿼리는 왼쪽(FROM 앞쪽) 항목의 컬럼을 참조 가능

FROM 절에서 함수도 암묵적 LATERAL:
  SELECT * FROM users u, generate_series(1, u.order_count) AS gs;
  -- generate_series(1, u.order_count)에서 u.order_count 참조 = 암묵적 LATERAL
  -- PostgreSQL에서 FROM 절의 함수는 자동으로 LATERAL 처리
```

### 2. LATERAL JOIN 실행 방식

```
실행 모델: Nested Loop 방식

  SELECT u.id, u.name, o.id, o.amount
  FROM users u
  LEFT JOIN LATERAL (
      SELECT id, amount FROM orders
      WHERE user_id = u.id
      ORDER BY created_at DESC
      LIMIT 3
  ) o ON true;

  내부 실행:
    For each row in users:
        u = {id=1, name='Alice'}
        실행: SELECT id, amount FROM orders
              WHERE user_id = 1  ← u.id = 1 전달
              ORDER BY created_at DESC LIMIT 3
        → 3행 반환 (또는 그 이하)
        결과에 u + o 조합하여 추가

        u = {id=2, name='Bob'}
        실행: SELECT id, amount FROM orders WHERE user_id = 2 LIMIT 3
        ...

  실행 계획:
  Nested Loop Left Join
    -> Seq Scan on users
    -> Limit
         -> Index Scan on orders using orders_user_id_created_at_idx
              Index Cond: (user_id = u.id)  ← 외부 행의 값

  인덱스 (user_id, created_at DESC)가 있으면:
  → 각 사용자마다 인덱스로 빠르게 최신 3개 조회 → 효율적

인덱스 없으면:
  Nested Loop + Seq Scan on orders (users 수만큼 반복)
  → N * M 비용 → 매우 느림
  → LATERAL 사용 시 관련 인덱스 필수
```

### 3. 상관 서브쿼리 vs LATERAL JOIN 비교

```
상관 서브쿼리:

  SELECT u.id, u.name,
      (SELECT max(amount) FROM orders WHERE user_id = u.id) AS max_order
  FROM users u;

  특성:
  ① SELECT 절에서 단일 값만 반환 가능
  ② 스칼라 서브쿼리: 2개 이상 컬럼 반환 불가
  ③ 각 행마다 서브쿼리 실행 (Nested Loop과 유사)

LATERAL JOIN:

  SELECT u.id, u.name, stats.*
  FROM users u
  LEFT JOIN LATERAL (
      SELECT max(amount) AS max_order,
             avg(amount) AS avg_order,
             count(*) AS order_count
      FROM orders WHERE user_id = u.id
  ) stats ON true;

  특성:
  ① 여러 컬럼 반환 가능
  ② LIMIT 사용 가능 (상관 서브쿼리: 불가)
  ③ 외부 행마다 실행 (Nested Loop)
  ④ 인덱스 활용 가능

언제 LATERAL이 필수인가:
  ✓ LIMIT이 필요한 경우: "각 사용자의 최신 N개"
  ✓ 여러 컬럼을 함께 반환: "첫 번째 주문의 id, amount, date"
  ✓ 집합 반환 함수: generate_series, jsonb_array_elements
  ✓ 외부 행의 값으로 함수 호출: calculate_stats(u.segment)

언제 윈도우 함수가 대안인가:
  ✓ 전체 데이터에 대한 순위/집계: ROW_NUMBER() OVER (PARTITION BY ...)
  ✓ 특정 사용자 집합이 아닌 전체 테이블 처리
  ✗ LIMIT이 각 파티션마다 다른 경우 → LATERAL 필요
```

### 4. LATERAL과 집합 반환 함수

```
PostgreSQL의 집합 반환 함수 (Set-Returning Functions):
  generate_series, jsonb_array_elements, unnest, regexp_matches 등

  FROM 절에서 이 함수들은 암묵적으로 LATERAL 동작:

  -- 각 이벤트의 태그를 행으로 분해
  SELECT e.id, tag
  FROM events e,
       jsonb_array_elements_text(e.data->'tags') AS tag;
  -- jsonb_array_elements_text(e.data->'tags')는 e 참조 → 자동 LATERAL

  명시적 LATERAL으로 같은 결과:
  SELECT e.id, tag
  FROM events e
  JOIN LATERAL jsonb_array_elements_text(e.data->'tags') AS tag ON true;

  generate_series와 LATERAL:
  -- 각 사용자의 subscription_months만큼 행 생성
  SELECT u.id, u.name, month_num
  FROM users u
  CROSS JOIN LATERAL generate_series(1, u.subscription_months) AS month_num;
  -- u.subscription_months를 generate_series에 전달 → 각 사용자마다 다른 범위

ON true:
  LATERAL의 결과를 JOIN할 때 JOIN 조건 없이 Cartesian Product처럼 결합
  LEFT JOIN LATERAL ... ON true: 결과가 없는 외부 행도 NULL로 포함 (LEFT JOIN)
  JOIN LATERAL ... ON true: 결과가 없으면 외부 행 제외 (INNER JOIN)
```

---

## 💻 실전 실험

### 실험 1: 각 사용자별 최신 주문 3개

```sql
-- 실험 데이터
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name TEXT
);

CREATE TABLE purchases (
    id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    product TEXT,
    amount NUMERIC,
    purchased_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON purchases(customer_id, purchased_at DESC);

INSERT INTO customers (name) SELECT 'Customer ' || i FROM generate_series(1, 100) i;

INSERT INTO purchases (customer_id, product, amount, purchased_at)
SELECT
    (random() * 100)::INT + 1,
    'Product ' || (random() * 50)::INT,
    (random() * 1000)::NUMERIC(10,2),
    NOW() - (random() * 365 * 24 * 3600)::INT * INTERVAL '1 second'
FROM generate_series(1, 5000);

ANALYZE purchases;

-- LATERAL: 각 고객의 최신 3개 주문
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.id, c.name, p.product, p.amount, p.purchased_at
FROM customers c
LEFT JOIN LATERAL (
    SELECT product, amount, purchased_at
    FROM purchases
    WHERE customer_id = c.id
    ORDER BY purchased_at DESC
    LIMIT 3
) p ON true
ORDER BY c.id, p.purchased_at DESC;
-- Nested Loop + Index Scan on purchases 확인
```

### 실험 2: JSONB 배열 분해 + 필터

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    tags JSONB  -- ["postgresql", "database", "indexing"]
);

INSERT INTO articles (title, tags) VALUES
    ('PostgreSQL FTS', '["postgresql", "fts", "gin"]'),
    ('MySQL vs PostgreSQL', '["mysql", "postgresql", "comparison"]'),
    ('Redis Caching', '["redis", "caching", "performance"]');

-- LATERAL로 JSONB 배열 분해
SELECT a.id, a.title, tag
FROM articles a
JOIN LATERAL jsonb_array_elements_text(a.tags) AS tag ON true
WHERE tag LIKE 'p%';

-- 태그별 아티클 수 집계
SELECT tag, count(*) AS article_count
FROM articles a
JOIN LATERAL jsonb_array_elements_text(a.tags) AS tag ON true
GROUP BY tag
ORDER BY article_count DESC;
```

### 실험 3: LATERAL vs 상관 서브쿼리 성능 비교

```sql
-- 상관 서브쿼리 (스칼라만)
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.id, c.name,
    (SELECT max(amount) FROM purchases WHERE customer_id = c.id) AS max_amount,
    (SELECT count(*) FROM purchases WHERE customer_id = c.id) AS total_orders
FROM customers c;

-- LATERAL (여러 컬럼, 한 번 실행)
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.id, c.name, stats.max_amount, stats.total_orders
FROM customers c
LEFT JOIN LATERAL (
    SELECT max(amount) AS max_amount,
           count(*) AS total_orders
    FROM purchases
    WHERE customer_id = c.id
) stats ON true;

-- 성능 비교: LATERAL이 customers × 2번 서브쿼리 vs customers × 1번 LATERAL
```

---

## 📊 MySQL과 비교

```
MySQL의 LATERAL 지원:

MySQL 8.0.14+: LATERAL 서브쿼리 지원 (공식 지원)

  SELECT c.id, recent.*
  FROM customers c
  LEFT JOIN LATERAL (
      SELECT * FROM orders
      WHERE customer_id = c.id
      ORDER BY created_at DESC
      LIMIT 3
  ) recent ON true;
  → MySQL 8.0.14+에서 동작

MySQL 8.0 이전:
  LATERAL 미지원 → 상관 서브쿼리 또는 복잡한 우회 방법

차이점:
  PostgreSQL: FROM 절의 집합 반환 함수가 암묵적 LATERAL
  MySQL: 명시적 LATERAL 키워드 필요 (함수는 JOIN 형태)

공통점:
  Top-N 패턴, 외부 행 참조 서브쿼리 동일하게 지원 (8.0.14+)
```

---

## ⚖️ 트레이드오프

```
LATERAL JOIN 장단점:

장점:
  ① 각 외부 행마다 독립적인 서브쿼리 실행 (Top-N 패턴)
  ② 서브쿼리에서 여러 컬럼 반환 가능
  ③ LIMIT을 각 파티션별로 적용 가능
  ④ 집합 반환 함수와 자연스러운 조합

단점:
  ① Nested Loop 방식 → 외부 행 수 × 서브쿼리 비용
  ② 인덱스가 없으면 극히 느림 (필수)
  ③ 외부 집합이 크면 전체 비용 증가

vs 윈도우 함수:
  LATERAL: 특정 외부 조건에 맞는 Top-N, 외부 값 의존 로직
  Window: 전체 집합에 순위/집계 → 그 결과로 필터

적합한 상황:
  ✓ "각 X에 대해 최근 N개 Y" 패턴
  ✓ 외부 행의 값으로 함수를 호출해야 할 때
  ✓ JSON/배열을 행으로 분해하면서 필터 필요
  ✓ 복잡한 조건의 첫 번째 관련 항목 조회
```

---

## 📌 핵심 정리

```
LATERAL JOIN 핵심:

개념:
  LATERAL 서브쿼리는 왼쪽 FROM 항목의 컬럼을 참조 가능
  각 외부 행마다 서브쿼리가 별도 실행 (Nested Loop)

문법:
  FROM left_table
  [LEFT] JOIN LATERAL (
      SELECT ... FROM right_table
      WHERE right_table.col = left_table.col  -- 외부 참조
      LIMIT N
  ) alias ON true

핵심 패턴:
  Top-N per group: LATERAL + LIMIT
  JSON 분해: JOIN LATERAL jsonb_array_elements(col) AS elem ON true
  함수 결과: JOIN LATERAL my_function(outer.col) AS result ON true

실행 계획:
  Nested Loop → 외부 행마다 Index Scan on right_table
  인덱스 (JOIN 컬럼, ORDER BY 컬럼) 필수

vs 상관 서브쿼리:
  상관 서브쿼리: 스칼라 단일 값만
  LATERAL: 여러 컬럼, LIMIT 가능

FROM 절 함수:
  암묵적 LATERAL → 외부 행의 값으로 함수 호출 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `LEFT JOIN LATERAL ... ON true`에서 `ON true`는 무엇을 의미하고, `ON some_condition`으로 바꾸면 어떤 효과가 있는가?

<details>
<summary>해설 보기</summary>

`ON true`는 항상 참인 JOIN 조건으로, LATERAL 서브쿼리의 결과를 조건 없이 결합합니다. 외부 행 당 0~N개의 결과를 모두 결합합니다.

`ON some_condition`으로 바꾸면:
```sql
LEFT JOIN LATERAL (
    SELECT id, amount FROM orders WHERE user_id = u.id LIMIT 5
) o ON o.amount > 100;
```
LATERAL 서브쿼리를 먼저 실행해서 최신 5개를 가져온 후, `o.amount > 100`인 것만 결합합니다. 서브쿼리 내부의 WHERE 조건으로 처리하는 것과 동일하지 않습니다.

실용적으로는 LATERAL의 필터를 서브쿼리 내부에 넣는 것이 더 명확합니다:
```sql
LEFT JOIN LATERAL (
    SELECT id, amount FROM orders WHERE user_id = u.id AND amount > 100 LIMIT 5
) o ON true;
```

</details>

---

**Q2.** LATERAL JOIN이 Nested Loop으로 실행될 때, 외부 테이블에 10만 건이 있고 내부 Index Scan이 평균 3건을 반환한다면 실제 I/O는 얼마인가?

<details>
<summary>해설 보기</summary>

- 외부 테이블 스캔: 10만 건 처리 → Seq Scan이면 테이블 크기에 비례
- 내부 Index Scan: 10만 번 실행 × 각 3건 = 30만 건 index access

Index Scan 한 번당 비용:
- 인덱스 리프 페이지 2~3개 + Heap 페이지 3개 = 5~6 페이지
- 10만 번 × 5~6 = **50~60만 페이지 I/O**

그러나 shared_buffers 캐시 효과:
- 자주 접근하는 인덱스 페이지 캐싱 → 실제 디스크 I/O는 훨씬 적음
- 인덱스가 `(user_id, created_at DESC)` 복합이면 리프 탐색이 빠름

결론: 적절한 인덱스 + 충분한 shared_buffers가 있으면 LATERAL이 효율적. 캐시 미스가 많으면 느림.

`EXPLAIN (ANALYZE, BUFFERS)` 출력의 `Buffers: shared hit=X read=Y`로 실제 캐시 효율을 확인하세요.

</details>

---

<div align="center">

**[⬅️ 이전: 파티셔닝 완전 분해](./03-partitioning.md)** | **[홈으로 🏠](../README.md)** | **[다음: Upsert와 SKIP LOCKED ➡️](./05-upsert-skip-locked.md)**

</div>
