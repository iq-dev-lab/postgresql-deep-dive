# 집계 함수 심화 — GROUPING SETS, FILTER, 통계 집계와 대용량 최적화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `GROUPING SETS`/`CUBE`/`ROLLUP`은 다차원 집계를 어떻게 처리하는가?
- `FILTER` 절로 조건부 집계를 어떻게 표현하는가?
- `percentile_cont`, `corr` 같은 통계 집계 함수는 무엇인가?
- `HashAgg`와 `GroupAgg` 중 어떤 것이 언제 선택되는가?
- `work_mem`이 집계 성능에 어떤 영향을 미치는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

"월별, 카테고리별, 전체 합계를 하나의 쿼리로 가져오라"는 요구를 받으면 UNION ALL로 여러 쿼리를 합치는 방법을 쓰게 된다. `GROUPING SETS`를 사용하면 단일 쿼리로 여러 집계 레벨을 동시에 처리할 수 있어 성능과 가독성 모두 향상된다. `FILTER` 절은 CASE WHEN을 사용한 조건부 집계를 더 명확하게 표현한다. 대용량 집계에서 `HashAgg`와 `GroupAgg`의 차이를 이해하면 `work_mem` 조정으로 성능을 개선할 수 있다.

---

## 😱 흔한 실수 (Before — 비효율적 집계)

```
실수 1: 다차원 집계를 UNION ALL로 구현

  -- 월별 합계
  SELECT 'monthly', month, category, SUM(amount) FROM sales GROUP BY month, category
  UNION ALL
  -- 카테고리별 합계
  SELECT 'by_category', NULL, category, SUM(amount) FROM sales GROUP BY category
  UNION ALL
  -- 전체 합계
  SELECT 'total', NULL, NULL, SUM(amount) FROM sales;

  → 테이블 3번 스캔
  → 코드 중복 (집계 로직 반복)

  GROUPING SETS 사용:
  SELECT month, category, SUM(amount)
  FROM sales
  GROUP BY GROUPING SETS (
      (month, category),  -- 월+카테고리별
      (category),         -- 카테고리별
      ()                  -- 전체
  );
  → 테이블 1번 스캔

실수 2: CASE WHEN으로 조건부 집계 (복잡함)

  SELECT
      SUM(CASE WHEN status = 'DONE' THEN amount ELSE 0 END) AS done_total,
      COUNT(CASE WHEN status = 'FAILED' THEN 1 END) AS failed_count
  FROM orders;

  FILTER 절:
  SELECT
      SUM(amount) FILTER (WHERE status = 'DONE') AS done_total,
      COUNT(*) FILTER (WHERE status = 'FAILED') AS failed_count
  FROM orders;

실수 3: 평균 대신 중앙값이 필요한 경우 AVG 사용

  SELECT AVG(salary) FROM employees;
  → 이상치(outlier)에 영향받는 산술 평균
  → 고연봉자 몇 명이 평균을 크게 왜곡

  중앙값:
  SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
  FROM employees;
```

---

## ✨ 올바른 접근 (After — 고급 집계 활용)

```
다차원 집계 패턴:

  -- 판매 대시보드: 여러 레벨 집계
  SELECT
      COALESCE(year::TEXT, 'ALL') AS year,
      COALESCE(quarter::TEXT, 'ALL') AS quarter,
      COALESCE(region, 'ALL') AS region,
      SUM(revenue) AS total_revenue,
      COUNT(*) AS order_count,
      GROUPING(year) AS is_year_total,
      GROUPING(quarter) AS is_quarter_total,
      GROUPING(region) AS is_region_total
  FROM sales
  GROUP BY ROLLUP(year, quarter, region)
  ORDER BY year NULLS LAST, quarter NULLS LAST, region NULLS LAST;

  → ROLLUP: 오른쪽에서 컬럼을 하나씩 제거한 집계 조합
  → (year, quarter, region), (year, quarter), (year), ()
  → 4가지 집계 레벨을 1번의 테이블 스캔으로

통계 집계:

  SELECT
      job_title,
      percentile_cont(0.25) WITHIN GROUP (ORDER BY salary) AS q1,
      percentile_cont(0.5) WITHIN GROUP (ORDER BY salary) AS median,
      percentile_cont(0.75) WITHIN GROUP (ORDER BY salary) AS q3,
      avg(salary) AS mean,
      stddev(salary) AS std_dev
  FROM employees
  GROUP BY job_title
  ORDER BY median DESC;
```

---

## 🔬 내부 동작 원리

### 1. GROUPING SETS / ROLLUP / CUBE

```
GROUPING SETS — 명시적 집계 조합:

  GROUP BY GROUPING SETS ((a, b), (a), ())
  →
  GROUP BY a, b      -- a+b별
  UNION ALL
  GROUP BY a         -- a별 (b=NULL)
  UNION ALL
  (전체 집계)         -- a=NULL, b=NULL

내부 실행: 단일 패스로 여러 집계 계산
  테이블을 한 번 스캔하면서 각 행을 해당하는 모든 집계 그룹에 기여

ROLLUP — 계층적 집계:

  GROUP BY ROLLUP(a, b, c)
  =
  GROUP BY GROUPING SETS ((a, b, c), (a, b), (a), ())
  → N+1개 조합 (N = 컬럼 수)

  예: ROLLUP(year, quarter, month)
  → (year, quarter, month) ← 가장 세분화
  → (year, quarter)
  → (year)
  → ()               ← 전체 합계

CUBE — 모든 조합:

  GROUP BY CUBE(a, b, c)
  =
  GROUP BY GROUPING SETS ((a,b,c), (a,b), (a,c), (b,c), (a), (b), (c), ())
  → 2^N개 조합 (N = 컬럼 수)

  예: CUBE(year, region, product)
  → 2^3 = 8가지 집계 조합
  → 주의: N이 크면 조합 폭발 (CUBE(a,b,c,d) = 16가지)

GROUPING() 함수:
  어느 컬럼이 집계로 NULL인지 식별
  GROUPING(year) = 1이면: year 기준으로 집계된 행 (year=NULL은 소계)
  GROUPING(year) = 0이면: 실제 year 값이 있는 행

  COALESCE와 조합:
  COALESCE(year::TEXT, 'TOTAL') → 실제 연도 또는 'TOTAL'
  CASE WHEN GROUPING(year) = 1 THEN 'ALL' ELSE year::TEXT END
```

### 2. FILTER 절

```
FILTER 절 — 조건부 집계:

  SELECT
      SUM(amount) FILTER (WHERE status = 'DONE') AS done_sum,
      COUNT(*) FILTER (WHERE status = 'DONE') AS done_count,
      COUNT(*) FILTER (WHERE status = 'FAILED') AS failed_count,
      AVG(amount) FILTER (WHERE amount > 0) AS positive_avg
  FROM orders;

  동등한 CASE WHEN:
  SELECT
      SUM(CASE WHEN status = 'DONE' THEN amount END) AS done_sum,
      COUNT(CASE WHEN status = 'DONE' THEN 1 END) AS done_count,
      COUNT(CASE WHEN status = 'FAILED' THEN 1 END) AS failed_count,
      AVG(CASE WHEN amount > 0 THEN amount END) AS positive_avg
  FROM orders;

  FILTER의 장점:
  ① 더 명확한 의도 표현
  ② 집계 함수의 의미와 조건이 분리되어 읽기 쉬움
  ③ AVG, PERCENTILE 등에서 CASE WHEN 표현이 복잡할 때 특히 유용

  Window Function + FILTER:
  SELECT
      user_id,
      SUM(amount) FILTER (WHERE status = 'DONE')
          OVER (PARTITION BY user_id) AS user_done_total
  FROM orders;

  -- 각 사용자의 완료된 주문 합계 (행 유지)
```

### 3. 통계 집계 함수

```
기술 통계:

  SELECT
      count(*),                              -- 행 수
      sum(value),                            -- 합계
      avg(value),                            -- 산술 평균
      min(value), max(value),               -- 최솟값, 최댓값
      stddev(value),                         -- 표준편차 (표본)
      variance(value),                       -- 분산 (표본)
      stddev_pop(value),                     -- 표준편차 (모집단)
      corr(x, y)                             -- 피어슨 상관계수 (-1 ~ 1)
  FROM measurements;

백분위수 (Ordered-Set Aggregate):

  -- 연속 백분위수 (보간)
  SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY salary) AS median;
  SELECT percentile_cont(ARRAY[0.25, 0.5, 0.75]) WITHIN GROUP (ORDER BY salary)
  AS quartiles;  -- 배열로 여러 백분위수 한 번에

  -- 불연속 백분위수 (실제 값)
  SELECT percentile_disc(0.5) WITHIN GROUP (ORDER BY salary) AS median_actual;

  -- mode (최빈값)
  SELECT mode() WITHIN GROUP (ORDER BY category) AS most_common_category;

  -- top N (빈도 내림차순)
  -- (별도 함수 없음, window + ROW_NUMBER로 구현)

Ordered-Set Aggregate 문법:
  function(args) WITHIN GROUP (ORDER BY sort_col [ASC|DESC])
  → 행을 정렬한 후 집계 (정렬 순서가 결과에 영향)

Hypothetical-Set Aggregate:
  -- "만약 이 값이 있다면 어떤 순위일까?"
  SELECT rank(100000) WITHIN GROUP (ORDER BY salary) FROM employees;
  -- 100000을 기존 데이터에 추가했다면 몇 번째 순위?

  SELECT percent_rank(100000) WITHIN GROUP (ORDER BY salary) FROM employees;
  -- 0.0 ~ 1.0 사이의 백분위
```

### 4. HashAgg vs GroupAgg와 work_mem

```
두 가지 집계 알고리즘:

HashAgg (해시 집계):
  모든 그룹을 해시 테이블에 저장
  입력 데이터가 ORDER BY 없어도 됨
  → work_mem에 해시 테이블이 맞으면 빠름
  → work_mem 초과 시 디스크 Batch 처리 → 성능 저하

  선호: 그룹 수가 많지 않음 (해시 테이블 크기 ≤ work_mem)
  실행 계획: "Hash Aggregate"

GroupAgg (정렬 기반 집계):
  입력을 GROUP BY 키 순으로 정렬 후 순서대로 집계
  → 같은 키가 연속으로 오므로 그룹 하나씩 처리
  → 해시 테이블 없음 → work_mem 영향 덜 받음
  → 하지만 정렬 비용 발생

  선호: 이미 정렬된 입력 (인덱스 스캔 등)
  실행 계획: "Group Aggregate"

work_mem과 HashAgg:

  SET work_mem = '64MB';
  SELECT category, SUM(amount) FROM orders GROUP BY category;

  work_mem이 충분 → HashAgg:
  → 메모리에서 처리 → 빠름

  work_mem 부족 → Disk-based HashAgg (Batches > 1):
  → EXPLAIN (ANALYZE): "Batches: 4"
  → 4번에 나눠서 처리 → 느림

  work_mem 키워서 BatchAgg 제거:
  SET work_mem = '512MB';
  → "Batches: 1" → 메모리 내 처리

EXPLAIN 출력 해석:
  Hash Aggregate  (cost=...) (actual time=...)
    Group Key: category
    Batches: 1  Memory Usage: 128kB
    → Batches=1: 메모리 내 처리
    Batches: 4  Memory Usage: 4096kB
    → Batches>1: 디스크 배치 처리

플래너가 HashAgg vs GroupAgg 선택:
  enable_hashagg = on/off로 강제 선택 가능 (테스트 목적)
  기본: 비용 기반 선택 (통계 기준)
```

---

## 💻 실전 실험

### 실험 1: GROUPING SETS vs UNION ALL 성능

```sql
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    sale_year INT,
    quarter INT,
    region TEXT,
    amount NUMERIC
);

INSERT INTO sales (sale_year, quarter, region, amount)
SELECT
    2022 + (random() * 2)::INT,
    (random() * 4)::INT + 1,
    (ARRAY['North', 'South', 'East', 'West'])[ceil(random()*4)::INT],
    (random() * 10000)::NUMERIC(10,2)
FROM generate_series(1, 500000);

ANALYZE sales;

-- UNION ALL 방식 (3번 스캔)
EXPLAIN (ANALYZE, BUFFERS)
SELECT 'year_quarter_region', sale_year, quarter, region, SUM(amount)
FROM sales GROUP BY sale_year, quarter, region
UNION ALL
SELECT 'year_quarter', sale_year, quarter, NULL, SUM(amount)
FROM sales GROUP BY sale_year, quarter
UNION ALL
SELECT 'total', NULL, NULL, NULL, SUM(amount)
FROM sales;

-- ROLLUP 방식 (1번 스캔)
EXPLAIN (ANALYZE, BUFFERS)
SELECT sale_year, quarter, region,
       SUM(amount) AS total,
       GROUPING(sale_year, quarter, region) AS grouping_level
FROM sales
GROUP BY ROLLUP(sale_year, quarter, region)
ORDER BY sale_year NULLS LAST, quarter NULLS LAST, region NULLS LAST;
```

### 실험 2: FILTER 절 활용

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT,
    status TEXT,
    amount NUMERIC,
    created_at DATE DEFAULT CURRENT_DATE
);

INSERT INTO orders (user_id, status, amount, created_at)
SELECT
    (random() * 1000)::INT,
    (ARRAY['PENDING','DONE','FAILED','CANCELLED'])[ceil(random()*4)::INT],
    (random() * 1000)::NUMERIC(10,2),
    CURRENT_DATE - (random() * 365)::INT
FROM generate_series(1, 100000);

-- FILTER로 조건부 집계
SELECT
    user_id,
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE status = 'DONE') AS completed,
    COUNT(*) FILTER (WHERE status = 'FAILED') AS failed,
    SUM(amount) FILTER (WHERE status = 'DONE') AS revenue,
    AVG(amount) FILTER (WHERE status = 'DONE') AS avg_order_value,
    SUM(amount) FILTER (WHERE created_at >= '2024-01-01') AS this_year
FROM orders
GROUP BY user_id
HAVING COUNT(*) FILTER (WHERE status = 'DONE') > 5
ORDER BY revenue DESC NULLS LAST
LIMIT 20;
```

### 실험 3: 통계 집계 함수

```sql
CREATE TABLE employee_salaries (
    dept TEXT,
    job_title TEXT,
    salary NUMERIC
);

INSERT INTO employee_salaries (dept, job_title, salary)
SELECT
    (ARRAY['Engineering', 'Product', 'Sales', 'HR'])[ceil(random()*4)::INT],
    (ARRAY['Junior', 'Mid', 'Senior', 'Lead', 'Manager'])[ceil(random()*5)::INT],
    (3000000 + random() * 10000000)::NUMERIC(12,0)  -- 300만~1300만원
FROM generate_series(1, 10000);

-- 부서별 통계
SELECT
    dept,
    count(*),
    round(avg(salary)) AS mean_salary,
    round(percentile_cont(0.5) WITHIN GROUP (ORDER BY salary)) AS median_salary,
    round(percentile_cont(0.25) WITHIN GROUP (ORDER BY salary)) AS q1,
    round(percentile_cont(0.75) WITHIN GROUP (ORDER BY salary)) AS q3,
    round(stddev(salary)) AS std_dev
FROM employee_salaries
GROUP BY dept
ORDER BY median_salary DESC;

-- 상관 분석 (salary와 순번의 상관관계 - 의미 없는 예시지만 문법 확인)
SELECT corr(salary, row_number() OVER (ORDER BY salary))
FROM employee_salaries;  -- 1에 가까우면 순위와 완전 상관

-- 백분위수 배열 (분포 확인)
SELECT
    percentile_cont(ARRAY[0.1, 0.25, 0.5, 0.75, 0.9, 0.99])
    WITHIN GROUP (ORDER BY salary) AS salary_distribution
FROM employee_salaries;
```

---

## 📊 MySQL과 비교

```
MySQL 집계 함수 비교:

GROUPING SETS / ROLLUP / CUBE:
  MySQL: ROLLUP 지원 (WITH ROLLUP)
  MySQL: GROUPING SETS, CUBE 미지원
  PostgreSQL: 모두 지원

  MySQL ROLLUP:
  SELECT year, quarter, SUM(amount)
  FROM sales GROUP BY year, quarter WITH ROLLUP;

FILTER 절:
  MySQL: 미지원
  → CASE WHEN 또는 IF()로 대체

  MySQL:
  SELECT SUM(IF(status='DONE', amount, 0)) AS done_sum FROM orders;

통계 함수:
  MySQL: STDDEV, VARIANCE, CORR (일부)
  MySQL: percentile 함수 없음 (복잡한 서브쿼리 필요)
  PostgreSQL: percentile_cont, percentile_disc, mode(), corr() 등 완전 지원

집계 알고리즘:
  MySQL: Hash Aggregate 기반
  PostgreSQL: HashAgg + GroupAgg 비용 기반 선택

결론:
  다차원 분석: PostgreSQL이 GROUPING SETS/CUBE로 우수
  통계 집계: PostgreSQL이 percentile 등 다양한 함수 제공
  기본 집계: 유사한 성능
```

---

## ⚖️ 트레이드오프

```
GROUPING SETS 트레이드오프:

장점:
  단일 스캔 → UNION ALL보다 빠름
  코드 중복 없음
  NULL로 집계 레벨 구분 (GROUPING() 함수로 식별)

단점:
  NULL이 "실제 NULL"인지 "집계로 인한 NULL"인지 구분 필요
  → GROUPING() 함수 필수
  → COALESCE로 레이블 처리 필요

HashAgg vs GroupAgg:

  HashAgg 유리:
  → 입력이 정렬 안 됨
  → 그룹 수 적음 (해시 테이블 크기 작음)
  → work_mem 충분

  GroupAgg 유리:
  → 입력이 이미 정렬됨 (인덱스 스캔)
  → 그룹 수 매우 많음 (해시 테이블 메모리 초과)
  → work_mem 부족 환경

work_mem 조정 시 주의:
  전역 증가: 모든 동시 쿼리에 영향
  SET work_mem = '256MB'; -- 세션 수준 (해당 쿼리에만)
  → 동시 쿼리 수 × work_mem < 총 메모리
```

---

## 📌 핵심 정리

```
고급 집계 핵심:

GROUPING SETS:
  GROUP BY GROUPING SETS ((a,b), (a), ())
  → 명시적 집계 조합, 단일 스캔

ROLLUP(a, b, c):
  → (a,b,c), (a,b), (a), () — N+1 조합

CUBE(a, b, c):
  → 2^N 모든 조합 — 주의: N 크면 폭발

FILTER 절:
  SUM(col) FILTER (WHERE condition)
  → CASE WHEN보다 명확

통계 집계:
  percentile_cont(0.5) WITHIN GROUP (ORDER BY col) — 중앙값
  mode() WITHIN GROUP (ORDER BY col) — 최빈값
  corr(x, y) — 상관계수
  stddev, variance, stddev_pop

HashAgg vs GroupAgg:
  HashAgg: 빠르지만 work_mem 필요
  GroupAgg: 정렬 비용, work_mem 덜 필요
  Batches > 1: work_mem 부족 → 증가 검토

work_mem:
  SET work_mem = 'NMB'; (세션 수준)
  EXPLAIN에서 Batches: 1 확인 → 최적
```

---

## 🤔 생각해볼 문제

**Q1.** `ROLLUP(a, b)`와 `ROLLUP(b, a)`는 같은 집계 조합을 생성하는가?

<details>
<summary>해설 보기</summary>

아니요, 다릅니다.

`ROLLUP(a, b)`:
- (a, b), (a), () — 3가지 조합
- "a 기준 소계 포함, b는 a의 하위 계층"

`ROLLUP(b, a)`:
- (b, a), (b), () — 3가지 조합
- "b 기준 소계 포함, a는 b의 하위 계층"

두 경우 모두 개수는 3가지이지만 (a, b)와 (b, a)는 다른 집계입니다. ROLLUP은 왼쪽에서 오른쪽으로 계층적 관계를 나타내므로, **가장 상위 레벨 컬럼을 왼쪽에** 배치해야 의미 있는 집계가 됩니다.

예: ROLLUP(year, quarter, month) → 연도 > 분기 > 월의 계층 집계

</details>

---

**Q2.** `percentile_cont(0.5)`와 `percentile_disc(0.5)`의 차이는 무엇인가? 각각 언제 사용하는가?

<details>
<summary>해설 보기</summary>

둘 다 중앙값(50번째 백분위수)을 계산하지만 방법이 다릅니다.

**`percentile_cont`(연속)**: 데이터를 보간(interpolation)합니다. 짝수 개의 데이터에서 중앙 두 값의 평균을 반환합니다. 결과가 실제 데이터에 없는 값일 수 있습니다.

```
데이터: [10, 20, 30, 40]
percentile_cont(0.5): (20 + 30) / 2 = 25.0  ← 실제 없는 값
```

**`percentile_disc`(불연속)**: 실제 데이터 중 백분위에 가장 가까운 값을 반환합니다. 결과가 항상 실제 데이터에 있는 값입니다.

```
데이터: [10, 20, 30, 40]
percentile_disc(0.5): 20  ← 실제 데이터 값
```

**사용 기준:**
- 연속 데이터(금액, 시간, 점수): `percentile_cont` (보간으로 더 정확한 추정)
- 이산 데이터(순위, 점수 등급): `percentile_disc` (실제 값만 의미 있음)
- 카테고리형 데이터: `mode()`가 더 적합

</details>

---

<div align="center">

**[⬅️ 이전: Upsert와 SKIP LOCKED](./05-upsert-skip-locked.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 트랜잭션과 잠금 ➡️](../transactions-locks/01-transaction-isolation.md)**

</div>
