# 윈도우 함수(Window Function) 완전 분해 — 집계하면서 행을 유지하는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `OVER(PARTITION BY ... ORDER BY ...)` 내부에서 정확히 어떤 순서로 실행되는가?
- `GROUP BY`와 Window Function의 근본적인 차이는 무엇인가?
- `ROW_NUMBER`, `RANK`, `DENSE_RANK`는 언제 다른 값을 반환하는가?
- `LAG`, `LEAD`로 이전/다음 행 값에 접근하는 방법은?
- `ROWS BETWEEN`과 `RANGE BETWEEN` 프레임의 차이는?
- 실행 계획에서 `WindowAgg` 노드는 무엇을 의미하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

"각 사용자별 가장 최근 주문 1개를 가져오라"는 요구는 서비스 개발에서 매우 흔하다. `GROUP BY`로는 집계값만 얻을 수 있고 원본 행의 다른 컬럼을 함께 얻기 어렵다. 상관 서브쿼리로 구현하면 N+1 문제가 생긴다. 윈도우 함수는 이 패턴을 단일 패스로 처리한다. 순위 계산, 전일 대비 증감, 이동 평균, 누적 합계 같은 분석 쿼리를 애플리케이션 코드 없이 SQL 한 줄로 표현할 수 있다.

---

## 😱 흔한 실수 (Before — 윈도우 함수 모름)

```
실수 1: 각 카테고리 최고 판매 상품 조회 (서브쿼리 남발)

  SELECT p.id, p.name, p.category, p.sales
  FROM products p
  WHERE p.sales = (
      SELECT max(sales) FROM products
      WHERE category = p.category
  );
  → 상관 서브쿼리: 외부 쿼리 행마다 서브쿼리 실행 → N+1

  윈도우 함수:
  SELECT id, name, category, sales
  FROM (
      SELECT id, name, category, sales,
             ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rn
      FROM products
  ) t
  WHERE rn = 1;
  → 단일 스캔, 정렬 한 번

실수 2: 전일 대비 매출 증감을 애플리케이션에서 계산

  -- Python/Java에서 날짜별 매출 목록을 가져와 루프로 계산
  rows = db.query("SELECT date, revenue FROM daily_sales ORDER BY date")
  for i, row in enumerate(rows):
      if i > 0:
          row['delta'] = row['revenue'] - rows[i-1]['revenue']

  SQL 한 줄로:
  SELECT date, revenue,
         revenue - LAG(revenue) OVER (ORDER BY date) AS delta
  FROM daily_sales;

실수 3: RANK와 ROW_NUMBER 혼동

  -- 동점자 처리를 ROW_NUMBER로 하면 임의 순서로 1, 2, 3 부여
  -- 동점자가 같은 순위를 받아야 하면 RANK 사용
```

---

## ✨ 올바른 접근 (After — 윈도우 함수 기반 분석)

```
윈도우 함수로 해결하는 주요 패턴:

  1. 각 그룹의 상위 N개:
     ROW_NUMBER() OVER (PARTITION BY group_col ORDER BY sort_col DESC)
     → 서브쿼리로 감싼 후 WHERE rn <= N

  2. 전/후 행 참조:
     LAG(col, n, default) OVER (ORDER BY date) -- n행 이전
     LEAD(col, n, default) OVER (ORDER BY date) -- n행 이후

  3. 누적 합계:
     SUM(revenue) OVER (ORDER BY date
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

  4. 이동 평균 (7일):
     AVG(value) OVER (ORDER BY date
                      ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)

  5. 전체 대비 비율:
     revenue / SUM(revenue) OVER () * 100

  6. 순위:
     RANK() OVER (PARTITION BY dept ORDER BY salary DESC)
```

---

## 🔬 내부 동작 원리

### 1. 윈도우 함수 실행 순서

```
SQL 논리적 실행 순서:

  FROM → WHERE → GROUP BY → HAVING → SELECT → WINDOW → ORDER BY → LIMIT

주의: WINDOW 함수는 WHERE/GROUP BY/HAVING 이후에 적용됨
  → WHERE 절에서 윈도우 함수 사용 불가
  → GROUP BY 후 집계된 결과에 윈도우 함수 적용 가능

실행 단계별 예시:

  SELECT
      user_id,
      order_id,
      amount,
      SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
  FROM orders
  WHERE status = 'DONE';

  Step 1 (FROM + WHERE):
  → status='DONE'인 행 필터 → 결과 집합 확보

  Step 2 (WINDOW 처리):
  → user_id별로 파티션 분할
  → 각 파티션 내 created_at으로 정렬
  → 각 행의 running_total 계산

  Step 3 (SELECT 컬럼 반환):
  → 원본 행 + 윈도우 함수 결과 함께 반환

핵심: 행이 제거되지 않음 (GROUP BY와의 차이)
  GROUP BY: N개 행 → K개 행 (그룹 수)
  WINDOW:   N개 행 → N개 행 (행 수 유지, 각 행에 집계값 추가)
```

### 2. OVER 절 구성 요소

```
OVER (
    PARTITION BY col1, col2   -- 파티션 분할 (없으면 전체가 하나의 파티션)
    ORDER BY col3 DESC        -- 파티션 내 정렬
    ROWS BETWEEN ... AND ...  -- 프레임 범위 (집계 범위)
)

각 요소의 의미:

PARTITION BY:
  → GROUP BY와 유사하게 데이터를 분할
  → 각 파티션 내에서 독립적으로 윈도우 함수 계산
  → 파티션 없으면 전체 결과 집합이 하나의 파티션

ORDER BY:
  → 파티션 내 행 순서 결정
  → LAG/LEAD, 누적 집계(SUM, AVG)에 필수
  → ORDER BY 없으면 프레임이 전체 파티션

Frame (ROWS/RANGE BETWEEN):
  → 현재 행 기준 집계 범위 지정

  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  → 파티션 시작부터 현재 행까지 (누적 합계)

  ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  → 현재 행 포함 이전 7행 (7일 이동 평균)

  ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
  → 이전 1행, 현재, 다음 1행 (3행 이동 평균)

  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  → 파티션 전체 (파티션 집계)

  ORDER BY 있는 경우 기본 프레임:
  RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

### 3. ROWS vs RANGE 프레임

```
ROWS: 물리적 행 수 기준

  ORDER BY date
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW

  date  | value | SUM (ROWS 2 PRECEDING)
  ------+-------+------------------------
  1/1   |  10   |  10       (행 1개: 현재만)
  1/2   |  20   |  30       (행 2개: 1/1, 1/2)
  1/3   |  30   |  60       (행 3개: 1/1, 1/2, 1/3)
  1/4   |  40   |  90       (행 3개: 1/2, 1/3, 1/4)
  1/5   |  50   | 120       (행 3개: 1/3, 1/4, 1/5)

RANGE: 값 범위 기준 (ORDER BY 값과 관련)

  ORDER BY date
  RANGE BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING

  중요 차이: RANGE에서 ORDER BY 값이 같은 행은 같은 프레임으로 처리

  date  | group | value | SUM (RANGE CURRENT ROW)
  ------+-------+-------+-------------------------
  1/1   |  A    |  10   |  10
  1/2   |  A    |  20   |  50  ← 1/2 A, 1/2 B 같은 날짜
  1/2   |  B    |  30   |  50  ← 두 행 모두 50 (같은 RANGE)
  1/3   |  A    |  40   |  40

  ROWS는 동일 날짜 행도 물리적으로 다른 행으로 취급
  RANGE는 동일 ORDER BY 값 행을 같은 범위로 취급

실용적 선택:
  이동 평균, 누적 합계 → ROWS (명확한 행 수)
  동점자 처리가 있는 통계 → RANGE 검토
```

### 4. 주요 윈도우 함수

```
순위 함수:

  ROW_NUMBER():
    동점 무시, 항상 고유 순위 (1, 2, 3, 4, 5)
    arbitrary한 tie-breaking → 동점자 처리가 불필요할 때

  RANK():
    동점자에게 같은 순위, 다음 순위 건너뜀 (1, 2, 2, 4, 5)
    스포츠 순위표처럼 "2등이 두 명이면 4등으로 바로"

  DENSE_RANK():
    동점자에게 같은 순위, 다음 순위 건너뛰지 않음 (1, 2, 2, 3, 4)
    연속된 순위 번호가 필요할 때

  NTILE(n):
    전체를 n개 버킷으로 균등 분할 (1~n)
    상위 10% = NTILE(10) WHERE ntile = 1

값 접근 함수:

  LAG(col, offset=1, default=NULL):
    현재 행에서 offset 행 이전 값
    이전 달 매출: LAG(revenue, 1) OVER (ORDER BY month)

  LEAD(col, offset=1, default=NULL):
    현재 행에서 offset 행 이후 값
    다음 달 예측: LEAD(revenue, 1) OVER (ORDER BY month)

  FIRST_VALUE(col):
    파티션(또는 프레임)에서 첫 번째 값

  LAST_VALUE(col):
    파티션(또는 프레임)에서 마지막 값
    주의: 기본 프레임이 CURRENT ROW까지라서
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING 지정 필요

  NTH_VALUE(col, n):
    파티션에서 n번째 값

집계 함수 (윈도우 모드):
  SUM, AVG, COUNT, MIN, MAX
  → OVER() 절과 함께 사용하면 윈도우 함수로 동작
```

### 5. 실행 계획의 WindowAgg 노드

```
EXPLAIN ANALYZE:

  WindowAgg  (cost=...) (actual time=... rows=... loops=1)
    Run Condition: ...
    ->  Sort  (cost=...) (actual time=... rows=...)
          Sort Key: user_id, created_at
          ->  Seq Scan on orders

동작 순서:
  1. 기본 데이터 스캔 (Seq Scan or Index Scan)
  2. Sort: PARTITION BY + ORDER BY 기준 정렬
  3. WindowAgg: 정렬된 데이터를 한 번 순서대로 읽으며
     파티션 경계 감지 + 프레임 내 집계 계산

정렬 비용이 핵심:
  윈도우 함수 비용 대부분 = 정렬 비용
  → work_mem 부족 시 외부 정렬(디스크) → 성능 저하

여러 윈도우 함수 최적화:
  같은 PARTITION BY + ORDER BY → 단일 Sort + 단일 WindowAgg
  다른 PARTITION BY → 별도 Sort + 별도 WindowAgg
  → 가능하면 같은 OVER 절을 재사용

  -- 비효율 (정렬 2번)
  SELECT
      ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC),
      SUM(salary) OVER (PARTITION BY dept)  -- 다른 OVER
  FROM employees;

  -- 효율 (같은 PARTITION BY, ORDER BY 없는 SUM은 별도)
  SELECT
      ROW_NUMBER() OVER w,
      RANK() OVER w,
      SUM(salary) OVER (PARTITION BY dept)
  FROM employees
  WINDOW w AS (PARTITION BY dept ORDER BY salary DESC);
  -- WINDOW 별칭으로 재사용
```

---

## 💻 실전 실험

### 실험 1: 각 카테고리별 상위 3개 상품

```sql
-- 실험 테이블
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    category TEXT,
    sales INT,
    price NUMERIC
);

INSERT INTO products (name, category, sales, price)
SELECT
    'Product ' || i,
    (ARRAY['Electronics', 'Clothing', 'Books', 'Food'])[ceil(random()*4)::INT],
    (random() * 10000)::INT,
    (random() * 500 + 10)::NUMERIC(10,2)
FROM generate_series(1, 1000) i;

-- 각 카테고리별 판매량 상위 3개
SELECT id, name, category, sales, rn
FROM (
    SELECT id, name, category, sales,
           ROW_NUMBER() OVER (
               PARTITION BY category
               ORDER BY sales DESC
           ) AS rn
    FROM products
) t
WHERE rn <= 3
ORDER BY category, rn;

-- EXPLAIN으로 WindowAgg 노드 확인
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, category, sales
FROM (
    SELECT id, name, category, sales,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rn
    FROM products
) t WHERE rn <= 3;
```

### 실험 2: 누적 합계와 이동 평균

```sql
CREATE TABLE daily_sales (
    sale_date DATE,
    revenue NUMERIC
);

INSERT INTO daily_sales (sale_date, revenue)
SELECT
    '2024-01-01'::DATE + (i-1),
    (random() * 1000 + 100)::NUMERIC(10,2)
FROM generate_series(1, 90) i;

-- 누적 합계 + 7일 이동 평균 + 전일 대비 증감
SELECT
    sale_date,
    revenue,
    SUM(revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_sum,
    AVG(revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d,
    revenue - LAG(revenue) OVER (ORDER BY sale_date) AS day_over_day,
    round(
        (revenue - LAG(revenue) OVER (ORDER BY sale_date))
        / LAG(revenue) OVER (ORDER BY sale_date) * 100, 1
    ) AS pct_change
FROM daily_sales
ORDER BY sale_date;
```

### 실험 3: ROWS vs RANGE 프레임 차이

```sql
CREATE TABLE score_test (
    player TEXT,
    score INT
);

INSERT INTO score_test VALUES
    ('A', 100), ('B', 100), ('C', 90), ('D', 80), ('E', 80);

-- ROWS: 물리적 행 순서
SELECT player, score,
       RANK() OVER (ORDER BY score DESC) AS rank_score,
       ROW_NUMBER() OVER (ORDER BY score DESC) AS rn,
       DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rn,
       SUM(score) OVER (
           ORDER BY score DESC
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS cumsum_rows,
       SUM(score) OVER (
           ORDER BY score DESC
           RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS cumsum_range
FROM score_test;
-- cumsum_rows vs cumsum_range: 동점자(100, 100)에서 차이 발생
```

---

## 📊 MySQL과 비교

```
MySQL 8.0 이상 Window Function 지원:

MySQL 8.0+:
  ROW_NUMBER(), RANK(), DENSE_RANK(), NTILE()
  LAG(), LEAD(), FIRST_VALUE(), LAST_VALUE(), NTH_VALUE()
  SUM, AVG, COUNT, MIN, MAX over OVER()

  MySQL 8.0 이전: 윈도우 함수 없음
  → 사용자 변수(@var)로 복잡하게 구현

PostgreSQL vs MySQL Window Function:
  기능: 거의 동일 (둘 다 SQL 표준 지원)
  성능: 유사 (정렬 기반)
  WINDOW 별칭: 둘 다 지원
  EXCLUDE: PostgreSQL만 지원 (EXCLUDE CURRENT ROW 등)

MySQL에서 지원 안 되는 기능:
  GROUPS 프레임 (PostgreSQL 11+)
  EXCLUDE CURRENT ROW/GROUP/TIES (PostgreSQL 14+)
  일부 집계 함수의 윈도우 모드
```

---

## ⚖️ 트레이드오프

```
윈도우 함수 비용:

정렬 비용:
  윈도우 함수의 대부분 비용 = PARTITION BY + ORDER BY 정렬
  대용량 데이터 + work_mem 부족 → 디스크 정렬 → 성능 저하

  최적화:
  SET work_mem = '256MB' 전에 실행 또는
  미리 정렬된 인덱스를 활용해 Sort 노드 제거

여러 윈도우 함수:
  같은 OVER 절 → 정렬 1회 공유 (효율)
  다른 OVER 절 → 정렬 N회 (비효율)
  → WINDOW 별칭으로 재사용하거나 쿼리 재구성

윈도우 함수 vs 서브쿼리:
  상관 서브쿼리로 구현 가능하지만 N+1 문제
  윈도우 함수: 단일 패스, 훨씬 빠름

윈도우 함수 결과를 WHERE 절에 쓰려면:
  → 서브쿼리/CTE로 감싼 후 외부에서 WHERE 적용
  WHERE ROW_NUMBER() OVER (...) = 1  -- 오류!
  → FROM (... ROW_NUMBER() ...) t WHERE rn = 1  -- 정상
```

---

## 📌 핵심 정리

```
Window Function 핵심:

핵심 개념:
  행을 유지하면서 집계 (GROUP BY는 행 감소)
  OVER(PARTITION BY ... ORDER BY ... ROWS/RANGE BETWEEN ...)

순위 함수:
  ROW_NUMBER(): 동점 무시, 항상 고유 (1,2,3,4)
  RANK():       동점 같은 순위, 건너뜀 (1,2,2,4)
  DENSE_RANK(): 동점 같은 순위, 연속 (1,2,2,3)

값 접근:
  LAG(col, n): n행 이전 값
  LEAD(col, n): n행 이후 값
  FIRST_VALUE/LAST_VALUE: 프레임 첫/마지막 값

프레임:
  ROWS: 물리적 행 수 기준
  RANGE: ORDER BY 값 기준 (동점 처리 방식 다름)

실행 계획:
  Sort → WindowAgg
  정렬 비용이 핵심 → work_mem 충분히 확보
  같은 OVER 절 → WINDOW 별칭으로 재사용
```

---

## 🤔 생각해볼 문제

**Q1.** `LAST_VALUE(col) OVER (PARTITION BY dept ORDER BY salary DESC)`가 예상과 다른 값을 반환한다. 왜인가?

<details>
<summary>해설 보기</summary>

ORDER BY가 있을 때 기본 프레임이 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`이기 때문입니다. `LAST_VALUE`는 프레임의 마지막 값을 반환하는데, 기본 프레임이 "처음부터 현재 행까지"이므로 현재 행 자체가 마지막 값이 됩니다.

파티션의 실제 마지막 값을 원하면 프레임을 명시해야 합니다:
```sql
LAST_VALUE(col) OVER (
    PARTITION BY dept
    ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

이것이 `LAST_VALUE`가 `FIRST_VALUE`보다 자주 실수가 발생하는 이유입니다.

</details>

---

**Q2.** 같은 `PARTITION BY dept ORDER BY salary`를 가진 윈도우 함수 3개를 사용할 때 정렬은 몇 번 발생하는가?

<details>
<summary>해설 보기</summary>

WINDOW 별칭을 사용하거나 동일한 OVER 절을 반복해도, PostgreSQL 옵티마이저는 동일한 PARTITION BY + ORDER BY를 가진 윈도우 함수를 **단일 Sort + 단일 WindowAgg**로 처리합니다. 정렬은 1번만 발생합니다.

```sql
SELECT
    ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC),
    RANK() OVER (PARTITION BY dept ORDER BY salary DESC),
    DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC)
FROM employees;
```

EXPLAIN에서 Sort 1개 + WindowAgg 1개가 나타납니다. WINDOW 별칭을 쓰면 더 명시적으로 표현할 수 있지만 성능 차이는 없습니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: Large Object vs TOAST](../toast-large-data/05-large-object-vs-toast.md)** | **[홈으로 🏠](../README.md)** | **[다음: CTE 심화 ➡️](./02-cte-internals.md)**

</div>
