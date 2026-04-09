# 파티셔닝 완전 분해 — Partition Pruning과 인덱스 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Range/List/Hash 파티셔닝은 내부적으로 어떻게 동작하는가?
- Partition Pruning은 어떤 조건에서 작동하고, 어떻게 확인하는가?
- 로컬(Local) 인덱스와 글로벌(Global) 인덱스의 차이는?
- 파티션 테이블에서 `UPDATE`가 파티션을 이동시키는 상황은?
- PostgreSQL 파티셔닝과 MySQL 파티셔닝의 핵심 차이는?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

수억 건의 로그, 이벤트, 주문 데이터가 단일 테이블에 쌓이면 VACUUM, 인덱스 스캔, 통계 수집 모두 느려진다. 파티셔닝은 데이터를 논리적으로는 하나의 테이블로 보이지만 물리적으로는 여러 파티션으로 분산 저장하는 기법이다. Partition Pruning으로 필요한 파티션만 스캔하고, 오래된 파티션을 `DROP PARTITION`으로 즉시 제거하며, VACUUM이 파티션별로 독립 실행된다. 운영 중인 대용량 테이블을 파티셔닝으로 전환하는 것은 복잡하지만, 처음부터 파티셔닝을 적용하면 수명 내내 혜택을 받는다.

---

## 😱 흔한 실수 (Before — 파티셔닝 설계 실수)

```
실수 1: 파티셔닝 키가 WHERE 절에 없어서 Pruning 실패

  -- 월별 Range 파티셔닝 (파티션 키: created_at)
  CREATE TABLE orders (id, user_id, created_at, ...)
  PARTITION BY RANGE (created_at);

  SELECT * FROM orders WHERE user_id = 42;
  → 파티션 키(created_at)가 WHERE 절에 없음
  → 모든 파티션 스캔 (Pruning 없음)
  → 단일 테이블보다 더 느릴 수 있음

  Pruning이 작동하는 쿼리:
  SELECT * FROM orders WHERE created_at >= '2024-01-01'
    AND created_at < '2024-02-01';
  → 해당 월 파티션만 스캔

실수 2: 너무 많은 파티션 생성

  일별 파티션 × 5년 = 1,825개 파티션
  → 플래너가 모든 파티션을 계획에 포함하는 데 시간 소요
  → 실행 계획 생성 자체가 느려짐
  → 일반적으로 파티션 수: 100~1,000개 이내 권장

실수 3: 파티션 경계를 넘는 UPDATE 무시

  -- range 파티셔닝: 2024년 파티션
  UPDATE orders SET created_at = '2023-12-31' WHERE id = 1;
  → created_at 변경으로 파티션 이동 필요
  → PostgreSQL: 자동으로 DELETE + INSERT (파티션 간 이동)
  → 성능 저하 + HOT Update 불가
```

---

## ✨ 올바른 접근 (After — 파티셔닝 올바른 설계)

```
파티셔닝 설계 원칙:

  1. 파티션 키 선택:
     → 항상 WHERE 절에 포함되는 컬럼 (Pruning 보장)
     → 시계열: created_at (Range)
     → 지역: region (List)
     → 균등 분산: id % N (Hash)

  2. 파티션 크기:
     → 파티션당 약 1~10GB (VACUUM이 효율적으로 처리)
     → 파티션 수: 100~1,000개 이내

  3. 오래된 데이터 관리:
     → 파티션 DROP: TRUNCATE + DROP TABLE (매우 빠름)
     → vs DELETE: 수 시간 소요, Dead Tuple 발생
     ALTER TABLE orders DETACH PARTITION orders_2022;
     DROP TABLE orders_2022;

  4. 파티션 자동 생성 (월별):
     → pg_partman 확장 활용 (자동 파티션 생성/삭제)
     CREATE EXTENSION pg_partman;
```

---

## 🔬 내부 동작 원리

### 1. 파티셔닝 유형별 내부 구조

```
Range Partitioning (범위 파티셔닝):

  CREATE TABLE orders (
      id BIGSERIAL,
      user_id INT,
      created_at TIMESTAMPTZ NOT NULL,
      amount NUMERIC
  ) PARTITION BY RANGE (created_at);

  CREATE TABLE orders_2024_01 PARTITION OF orders
      FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
  CREATE TABLE orders_2024_02 PARTITION OF orders
      FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

  내부 구조:
  orders (부모, 데이터 없음)
  ├── orders_2024_01 (독립된 Heap 파일)
  ├── orders_2024_02 (독립된 Heap 파일)
  └── orders_default (기본 파티션, 나머지 값)

  INSERT 라우팅:
  INSERT INTO orders (created_at='2024-01-15')
  → PostgreSQL이 created_at 값으로 파티션 결정
  → orders_2024_01.Heap에 실제 삽입

  SELECT 라우팅 (Partition Pruning):
  SELECT * FROM orders WHERE created_at >= '2024-01-01'
    AND created_at < '2024-02-01';
  → 파티션 경계와 비교: orders_2024_01만 해당
  → orders_2024_02 이후는 스캔 안 함

List Partitioning (목록 파티셔닝):

  CREATE TABLE events (...) PARTITION BY LIST (region);
  CREATE TABLE events_us PARTITION OF events FOR VALUES IN ('US', 'CA');
  CREATE TABLE events_eu PARTITION OF events FOR VALUES IN ('DE', 'FR', 'UK');
  CREATE TABLE events_ap PARTITION OF events FOR VALUES IN ('KR', 'JP', 'CN');

  → region 값으로 파티션 라우팅
  → WHERE region = 'KR' → events_ap만 스캔

Hash Partitioning (해시 파티셔닝):

  CREATE TABLE users (...) PARTITION BY HASH (id);
  CREATE TABLE users_0 PARTITION OF users FOR VALUES WITH (modulus 4, remainder 0);
  CREATE TABLE users_1 PARTITION OF users FOR VALUES WITH (modulus 4, remainder 1);
  CREATE TABLE users_2 PARTITION OF users FOR VALUES WITH (modulus 4, remainder 2);
  CREATE TABLE users_3 PARTITION OF users FOR VALUES WITH (modulus 4, remainder 3);

  → id % 4 == 0 → users_0, id % 4 == 1 → users_1 ...
  → 균등 분산, 특정 파티션 제거 불가 (모든 파티션에 모든 기간 데이터)
  → Pruning: WHERE id = 42 → 42 % 4 = 2 → users_2만 스캔
```

### 2. Partition Pruning 상세

```
Pruning이 작동하는 조건:

  파티션 키가 WHERE 절에 있고 상수/파라미터로 비교될 때

  Range 파티션 Pruning 예시:
  파티션 경계: [2024-01-01, 2024-02-01), [2024-02-01, 2024-03-01), ...

  WHERE created_at = '2024-01-15':
  → 2024-01 파티션에만 해당 → 1개 파티션만 스캔

  WHERE created_at >= '2024-01-01' AND created_at < '2024-03-01':
  → 2024-01, 2024-02 파티션 해당 → 2개 파티션 스캔

  WHERE created_at > '2024-01-01':
  → 2024-01 이상 모든 파티션 스캔 (상한 없음)

Pruning이 작동하지 않는 경우:

  WHERE date_trunc('month', created_at) = '2024-01-01':
  → 함수로 변환: created_at 값 직접 비교 불가
  → 모든 파티션 스캔

  WHERE created_at = NOW() - INTERVAL '30 days':
  → NOW()는 플래닝 시점 상수로 평가됨 → Pruning 가능

  동적 파라미터 (PREPARE):
  → Execution-time Pruning (실행 시 Pruning)
  → PostgreSQL 12+ enable_partition_pruning = on (기본)

EXPLAIN으로 Pruning 확인:
  EXPLAIN SELECT * FROM orders
  WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01';

  -> Append (실제 실행된 파티션만 표시)
       -> Seq Scan on orders_2024_01
  (다른 파티션: "Pruned" 또는 표시 안 됨)
```

### 3. 파티션 인덱스 전략

```
로컬(Local) 인덱스:
  각 파티션에 독립적으로 생성

  CREATE INDEX ON orders_2024_01 (user_id);
  CREATE INDEX ON orders_2024_02 (user_id);
  -- 또는 부모 테이블에 생성 → 각 파티션에 자동 생성
  CREATE INDEX ON orders (user_id);  -- 모든 파티션에 자동 생성

  특성:
  ① 파티션 단위로 REINDEX CONCURRENTLY 가능
  ② 파티션 DROP 시 인덱스도 함께 삭제 (간단)
  ③ UNIQUE 제약: 파티션 키 포함 시만 가능
  ④ 파티션 키가 WHERE에 없으면 모든 파티션 인덱스 스캔

글로벌(Global) 인덱스:
  PostgreSQL 17+에서 지원 시작 (이전에는 미지원)

  특성:
  ① 파티션 키 없이도 전체 테이블 단위 UNIQUE 가능
  ② 파티션 추가/제거 시 글로벌 인덱스 갱신 필요
  ③ 구현 복잡성 높음

현재 일반적 접근 (PostgreSQL 16 이하):
  UNIQUE 없이 user_id 인덱스 → 로컬 인덱스
  파티션 키(created_at) + user_id 복합 인덱스 → Pruning + user_id 검색

파티션 + 인덱스 전략 예시:
  -- 기본: 부모에 생성 → 모든 파티션에 자동
  CREATE INDEX idx_orders_user_id ON orders(user_id);

  -- Pruning 활용한 복합 인덱스
  CREATE INDEX idx_orders_created_user ON orders(created_at, user_id);
  -- WHERE created_at BETWEEN ... AND ... AND user_id = 42 → 특정 파티션 + 인덱스
```

### 4. 파티션 관리

```
파티션 추가:
  CREATE TABLE orders_2024_03 PARTITION OF orders
  FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

  기존 파티션 연결:
  ALTER TABLE orders ATTACH PARTITION orders_2024_03
  FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

파티션 분리:
  -- 분리 후에도 독립된 테이블로 접근 가능
  ALTER TABLE orders DETACH PARTITION orders_2022;
  -- orders_2022는 독립 테이블로 남음
  -- 보관 또는 아카이브 후 삭제 가능

파티션 삭제:
  DROP TABLE orders_2022;  -- 즉시 삭제 (DELETE보다 훨씬 빠름!)

vs DELETE:
  DELETE FROM orders WHERE created_at < '2023-01-01';
  → 수 시간 소요, Dead Tuple 발생, VACUUM 필요

  DROP TABLE orders_2022;  (DETACH 후)
  → 즉각 (파일 삭제)
  → Dead Tuple 없음

파티션 간 UPDATE:
  UPDATE orders SET created_at = '2023-12-31'
  WHERE id = 1 AND created_at = '2024-01-15';
  → created_at 변경 → 파티션 경계 교차 → 자동 DELETE + INSERT
  → 성능 주의: 파티션 이동이 빈번하면 설계 재검토
```

---

## 💻 실전 실험

### 실험 1: Range 파티셔닝 생성 및 Pruning 확인

```sql
-- 월별 파티션 테이블
CREATE TABLE log_events (
    id BIGSERIAL,
    user_id INT NOT NULL,
    event_type TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    payload JSONB
) PARTITION BY RANGE (created_at);

-- 월별 파티션 생성 (2024년)
DO $$
DECLARE
    m INT;
    start_date DATE;
    end_date DATE;
BEGIN
    FOR m IN 1..12 LOOP
        start_date := DATE '2024-01-01' + ((m-1) || ' month')::INTERVAL;
        end_date := start_date + INTERVAL '1 month';
        EXECUTE format(
            'CREATE TABLE log_events_%s PARTITION OF log_events '
            'FOR VALUES FROM (%L) TO (%L)',
            to_char(start_date, 'YYYY_MM'), start_date, end_date
        );
    END LOOP;
END $$;

-- 인덱스 (모든 파티션에 자동 생성)
CREATE INDEX ON log_events (user_id);
CREATE INDEX ON log_events (created_at);

-- 데이터 삽입 (자동 라우팅)
INSERT INTO log_events (user_id, event_type, created_at)
SELECT
    (random() * 10000)::INT,
    (ARRAY['click','view','purchase'])[ceil(random()*3)::INT],
    '2024-01-01'::TIMESTAMPTZ + (random() * 365 * 24 * 3600)::INT * INTERVAL '1 second'
FROM generate_series(1, 100000);

ANALYZE log_events;

-- Partition Pruning 확인
EXPLAIN SELECT count(*) FROM log_events
WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
-- 1개 파티션만 스캔

-- Pruning 없는 쿼리
EXPLAIN SELECT count(*) FROM log_events WHERE user_id = 42;
-- 모든 파티션 스캔 (파티션 키 없음)
```

### 실험 2: 파티션 삭제 vs DELETE 성능 비교

```sql
-- 파티션 삭제 (빠름)
ALTER TABLE log_events DETACH PARTITION log_events_2024_01;
-- \timing
DROP TABLE log_events_2024_01;  -- 즉각 완료

-- 같은 데이터를 DELETE로 삭제하면
-- DELETE FROM log_events WHERE created_at < '2024-02-01';
-- → 수만 건 * 각 행 처리 → Dead Tuple → VACUUM 필요
```

### 실험 3: 파티션 정보 조회

```sql
-- 파티션 목록 및 크기
SELECT
    parent.relname AS parent_table,
    child.relname AS partition_name,
    pg_get_expr(child.relpartbound, child.oid) AS partition_range,
    pg_size_pretty(pg_relation_size(child.oid)) AS size,
    pg_size_pretty(pg_total_relation_size(child.oid)) AS total_size
FROM pg_class parent
JOIN pg_inherits i ON i.inhparent = parent.oid
JOIN pg_class child ON child.oid = i.inhrelid
WHERE parent.relname = 'log_events'
ORDER BY child.relname;

-- 파티션별 행 수
SELECT
    tableoid::regclass AS partition,
    count(*) AS row_count
FROM log_events
GROUP BY tableoid
ORDER BY partition;
```

---

## 📊 MySQL과 비교

```
MySQL 파티셔닝 vs PostgreSQL 파티셔닝:

항목               | MySQL                  | PostgreSQL
──────────────────┼────────────────────────┼──────────────────────────
파티션 유형         | RANGE, LIST, HASH, KEY  | RANGE, LIST, HASH
파티션 테이블 타입   | 단일 테이블 파일 분할    | 독립 테이블 (상속)
글로벌 인덱스       | 지원 안 됨              | PostgreSQL 17+ 지원
로컬 인덱스         | 지원                   | 지원
외래키              | 파티션 테이블에 FK 불가  | 파티션 테이블에 FK 불가
파티션 제거 방법    | TRUNCATE PARTITION      | DETACH + DROP TABLE
자동 파티션         | 없음                   | pg_partman 확장
Pruning             | 지원                   | 지원 (12+: 실행 시)
INSERT 라우팅       | 자동                   | 자동

MySQL TRUNCATE PARTITION:
  ALTER TABLE orders TRUNCATE PARTITION p2022;
  → 해당 파티션 데이터 즉시 삭제 (DROP 없이)

PostgreSQL:
  ALTER TABLE orders DETACH PARTITION orders_2022;
  DROP TABLE orders_2022;
  또는 DELETE + VACUUM (느림)

MySQL의 제한:
  InnoDB의 파티션 테이블에 외래키 불가 → 관련 테이블 파티셔닝 제한
  글로벌 인덱스 없음 → 파티션 키 없는 컬럼 UNIQUE 제약 어려움
```

---

## ⚖️ 트레이드오프

```
파티셔닝의 장단점:

장점:
  ① 오래된 데이터 빠른 제거 (DROP TABLE)
  ② VACUUM이 파티션 단위로 독립 실행 → 부하 분산
  ③ 파티션 키 조건 쿼리: Pruning으로 일부만 스캔
  ④ 통계가 파티션 단위로 더 정확
  ⑤ 파티션 단위 REINDEX CONCURRENTLY

단점:
  ① 파티션 키 없는 쿼리: 모든 파티션 스캔 (더 느릴 수 있음)
  ② 파티션 간 UPDATE 비용 (DELETE + INSERT)
  ③ 파티션 수 많으면 실행 계획 생성 오버헤드
  ④ Cross-partition JOIN 비용
  ⑤ 설계 변경 어려움 (파티션 키 변경은 테이블 재생성 필요)

적합한 시나리오:
  ✓ 시계열 데이터 (로그, 이벤트, 주문)
  ✓ 주기적인 오래된 데이터 제거 (월별 DROP)
  ✓ 파티션 키 기반 쿼리가 대부분

부적합한 시나리오:
  ✗ 파티션 키가 없는 WHERE 절 쿼리가 대부분
  ✗ 자주 파티션 경계를 넘는 UPDATE
  ✗ 소규모 데이터 (오버헤드가 이득보다 큼)
```

---

## 📌 핵심 정리

```
파티셔닝 핵심:

유형:
  RANGE: 시계열, 날짜 범위 (매월/매년 파티션)
  LIST:  지역, 상태 등 카테고리 기반
  HASH:  균등 분산, Pruning은 = 조건만

Partition Pruning:
  파티션 키가 WHERE에 있고 상수 비교 → 해당 파티션만 스캔
  함수 변환 (date_trunc 등) → Pruning 실패
  EXPLAIN으로 확인: 실행된 파티션 수

인덱스:
  부모 테이블에 CREATE INDEX → 모든 파티션 자동 생성
  UNIQUE: 파티션 키 포함 시만 가능
  글로벌 인덱스: PostgreSQL 17+ (이전엔 미지원)

파티션 관리:
  추가: CREATE TABLE ... PARTITION OF ...
  분리: ALTER TABLE DETACH PARTITION
  삭제: DROP TABLE (즉각, DELETE보다 훨씬 빠름)
  파티션 간 UPDATE → 자동 DELETE + INSERT

운영:
  pg_partman으로 자동 파티션 생성/만료
  파티션 수: 100~1,000개 이내 권장
```

---

## 🤔 생각해볼 문제

**Q1.** `CREATE INDEX ON orders(user_id)`를 파티션된 부모 테이블에 생성하면, 각 파티션의 인덱스는 언제 어떻게 생성되는가?

<details>
<summary>해설 보기</summary>

부모 테이블에 `CREATE INDEX ON orders(user_id)`를 실행하면 PostgreSQL은 **모든 기존 파티션에 즉시 동일한 인덱스를 생성**합니다. 이후 새로 생성되거나 ATTACH되는 파티션에도 자동으로 동일한 인덱스가 생성됩니다.

중요한 점:
- 부모 테이블 인덱스는 메타데이터로만 존재 (실제 인덱스 엔트리 없음)
- 실제 인덱스는 각 파티션에 독립적으로 생성

`CREATE INDEX CONCURRENTLY ON orders(user_id)`는 파티션된 테이블에서 동작하지 않습니다. 대신 각 파티션에 개별적으로 `CONCURRENTLY` 인덱스를 생성한 후, 부모에 인덱스를 생성하는 방법을 사용합니다.

</details>

---

**Q2.** 100개 파티션이 있는 테이블에 파티션 키 없이 SELECT를 실행하면 실행 계획 생성 시간도 늘어나는가?

<details>
<summary>해설 보기</summary>

네, 늘어납니다. PostgreSQL 옵티마이저는 각 파티션을 계획에 포함시키는 과정에서 모든 파티션을 검토합니다. 파티션 수가 많을수록 **Planning Time(계획 생성 시간)**이 증가합니다.

```sql
EXPLAIN (ANALYZE, FORMAT TEXT)
SELECT count(*) FROM log_events WHERE user_id = 42;
-- Planning Time: X.XXX ms  ← 파티션 수에 비례
-- Execution Time: Y.YYY ms
```

대응 방법:
1. 파티션 수를 100~1,000개 이내로 유지
2. `enable_partition_pruning = on`으로 Pruning 활성화 (기본)
3. 파티션이 매우 많으면 쿼리에 파티션 키 조건 필수화

PostgreSQL 12+에서 Execution-time Pruning 개선으로 PREPARE 문도 실행 시점에 Pruning이 가능해져 Planning Time 일부 절감이 가능합니다.

</details>

---

<div align="center">

**[⬅️ 이전: CTE 심화](./02-cte-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: LATERAL JOIN ➡️](./04-lateral-join.md)**

</div>
