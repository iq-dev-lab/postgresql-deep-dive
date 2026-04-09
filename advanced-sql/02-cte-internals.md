# CTE 심화 — Optimization Fence와 재귀 CTE

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PostgreSQL 12 이전에 CTE가 왜 Optimization Fence인가?
- `MATERIALIZED`와 `NOT MATERIALIZED` 중 어느 것을 언제 선택해야 하는가?
- `WITH RECURSIVE`는 내부적으로 어떤 알고리즘으로 트리/그래프를 순회하는가?
- CTE와 서브쿼리 중 성능 면에서 어느 것이 유리한가?
- 재귀 CTE에서 무한 루프를 방지하는 방법은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

CTE(`WITH` 절)는 쿼리를 읽기 쉽게 만들어주지만, 잘못 사용하면 오히려 성능을 크게 저하시킨다. PostgreSQL 12 이전에는 CTE가 항상 별도로 실행되고 결과를 임시로 저장했기 때문에(Optimization Fence), 옵티마이저가 CTE 안팎의 조건을 연계해서 최적화할 수 없었다. 재귀 CTE는 계층형 데이터(조직도, 카테고리 트리, 경로 탐색)를 SQL 한 번으로 순회할 수 있는 강력한 기능이다.

---

## 😱 흔한 실수 (Before — CTE 최적화 오해)

```
실수 1: PostgreSQL 12 이전에 CTE를 Subquery처럼 사용

  WITH active_orders AS (
      SELECT * FROM orders WHERE status = 'ACTIVE'
  )
  SELECT * FROM active_orders WHERE user_id = 42;

  PostgreSQL 11 이하:
  → CTE가 먼저 전체 실행 → 모든 ACTIVE 주문 임시 저장
  → 그 후 user_id = 42 필터
  → user_id 인덱스 활용 불가! (CTE 결과에 인덱스 없음)

  PostgreSQL 12+:
  → NOT MATERIALIZED가 기본 → CTE가 인라인됨
  → 플래너가 user_id 조건을 CTE 안으로 밀어넣기 가능

실수 2: 재귀 CTE에서 무한 루프

  WITH RECURSIVE org AS (
      SELECT id, parent_id, name FROM departments WHERE id = 1
      UNION ALL
      SELECT d.id, d.parent_id, d.name
      FROM departments d
      JOIN org ON d.parent_id = org.id
  )
  SELECT * FROM org;
  -- 만약 departments 테이블에 순환 참조가 있으면 무한 루프!
  -- id=1 → parent=2 → ... → parent=1 → 무한

  해결: CYCLE 절 (PostgreSQL 14+) 또는 visited 배열 추적

실수 3: 매번 참조되는 무거운 CTE

  WITH expensive AS (
      SELECT ... 복잡한 집계 ...  -- 100만 행 처리
  )
  SELECT * FROM expensive WHERE a = 1
  UNION ALL
  SELECT * FROM expensive WHERE b = 2;
  -- expensive CTE가 2번 참조 → 2번 실행? 1번 실행?
  -- MATERIALIZED이면 1번 (결과 캐시)
  -- NOT MATERIALIZED이면 2번 (인라인됨)
```

---

## ✨ 올바른 접근 (After — CTE 동작 이해 기반 선택)

```
CTE 사용 전략:

  1. 단순 가독성 향상 (PostgreSQL 12+):
     → NOT MATERIALIZED (기본값) 사용
     → 플래너가 최적화 자유롭게 처리
     → WITH not_mat AS NOT MATERIALIZED (...)

  2. 결과를 여러 번 참조, 재실행 비용 높음:
     → MATERIALIZED 명시
     → 첫 실행 결과를 임시 저장, 이후 재사용
     → WITH mat AS MATERIALIZED (...)

  3. Side effect 보장 (DML CTE):
     → MATERIALIZED 자동 적용 (INSERT/UPDATE/DELETE CTE)
     WITH ins AS (INSERT INTO log ... RETURNING id)
     SELECT * FROM ins JOIN orders ON ...
     → INSERT가 반드시 한 번 실행됨

  4. 재귀 CTE:
     WITH RECURSIVE tree AS (
         -- 앵커(비재귀): 시작점
         SELECT id, parent_id, name, 0 AS depth
         FROM categories WHERE parent_id IS NULL
         UNION ALL
         -- 재귀: 이전 결과에서 다음 레벨 조회
         SELECT c.id, c.parent_id, c.name, t.depth + 1
         FROM categories c
         JOIN tree t ON c.parent_id = t.id
         WHERE t.depth < 10  -- 깊이 제한 (무한 루프 방지)
     )
     SELECT * FROM tree;
```

---

## 🔬 내부 동작 원리

### 1. CTE와 Optimization Fence

```
Optimization Fence란:

  옵티마이저가 CTE 안과 밖의 조건을 연계해서 최적화하지 못하는 장벽

PostgreSQL 11 이하 (항상 MATERIALIZED):

  WITH filtered AS (
      SELECT * FROM orders WHERE status = 'DONE'
  )
  SELECT * FROM filtered WHERE user_id = 42;

  실행 계획:
  Hash Join (또는 Nested Loop)
    → CTE Scan on filtered
    → (Hash)
       → Seq Scan on orders
            Filter: (status = 'done')
  CTE Result 후 user_id=42 필터 적용

  문제:
  ① orders에 (user_id, status) 인덱스가 있어도 활용 불가
  ② CTE 결과를 메모리에 임시 저장 (status='DONE' 전체)
  ③ 그 이후에 user_id 필터 → 비효율

PostgreSQL 12+ (기본 NOT MATERIALIZED):

  동일한 쿼리:
  실행 계획:
  Index Scan on orders
    Index Cond: (user_id = 42)
    Filter: (status = 'done')

  → CTE가 인라인됨 → 플래너가 user_id 조건을 orders 스캔에 적용
  → (user_id, status) 인덱스 또는 user_id 인덱스 활용 가능

MATERIALIZED vs NOT MATERIALIZED:

  MATERIALIZED:
  → CTE 결과를 임시 테이블에 저장
  → 여러 번 참조해도 한 번만 실행
  → 옵티마이저가 CTE 내부로 조건 밀어넣기 불가

  NOT MATERIALIZED (PostgreSQL 12+ 기본):
  → CTE를 서브쿼리처럼 인라인 처리
  → 여러 번 참조하면 여러 번 실행 가능
  → 옵티마이저가 조건 밀어넣기 가능 (더 효율적)

명시적 선택:
  WITH mat AS MATERIALIZED (SELECT ...)
  WITH not_mat AS NOT MATERIALIZED (SELECT ...)
```

### 2. WITH RECURSIVE 내부 알고리즘

```
재귀 CTE 실행 알고리즘 (BFS 방식):

WITH RECURSIVE tree AS (
    -- 앵커 (비재귀 파트): 시작점
    SELECT id, parent_id, name, 1 AS level
    FROM categories
    WHERE parent_id IS NULL  -- 루트 노드

    UNION ALL

    -- 재귀 파트: 이전 반복 결과에서 다음 레벨
    SELECT c.id, c.parent_id, c.name, t.level + 1
    FROM categories c
    JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree;

내부 실행 과정:

  WorkTable (임시 작업 테이블):
    반복 1: 앵커 실행 → 루트 노드 → WorkTable에 저장
    반복 2: 재귀 파트 실행 (WorkTable의 결과 참조)
            → 루트의 자식들 → 새 WorkTable에 저장
    반복 3: 재귀 파트 실행 (새 WorkTable 참조)
            → 자식의 자식들 → WorkTable 교체
    ...
    반복 N: WorkTable이 비어있으면 종료 (결과 없음)

  모든 반복 결과를 Result Set에 누적:
  Result Set = 앵커 + 반복1 + 반복2 + ... + 반복N

UNION ALL vs UNION:
  UNION ALL: 중복 허용 (빠름) → 대부분의 트리 탐색
  UNION: 중복 제거 (느림) → 그래프 순환 방지에 사용 가능하지만 CYCLE 절 권장

PostgreSQL 14+ CYCLE 절:
  WITH RECURSIVE path AS (
      SELECT id, parent_id, ARRAY[id] AS visited
      FROM nodes WHERE id = 1
      UNION ALL
      SELECT n.id, n.parent_id, visited || n.id
      FROM nodes n
      JOIN path p ON n.parent_id = p.id
      WHERE NOT n.id = ANY(p.visited)  -- 방문한 노드 제외
  )
  SELECT * FROM path;

  또는 PostgreSQL 14+ CYCLE 절:
  WITH RECURSIVE path AS (
      SELECT id, parent_id FROM nodes WHERE id = 1
      UNION ALL
      SELECT n.id, n.parent_id
      FROM nodes n JOIN path p ON n.parent_id = p.id
  ) CYCLE id SET is_cycle USING path
  SELECT * FROM path WHERE NOT is_cycle;
```

### 3. DML을 포함한 CTE (Data-Modifying CTE)

```
CTE 안에 INSERT/UPDATE/DELETE 포함 가능:

예시: 배치로 이동하며 원자적 처리

  -- orders에서 처리할 항목 가져오고, 동시에 상태 업데이트
  WITH moved AS (
      UPDATE orders
      SET status = 'PROCESSING', processed_at = NOW()
      WHERE id IN (
          SELECT id FROM orders
          WHERE status = 'PENDING'
          LIMIT 10
          FOR UPDATE SKIP LOCKED  -- 동시 Worker 충돌 방지
      )
      RETURNING *
  )
  INSERT INTO processing_log (order_id, started_at)
  SELECT id, processed_at FROM moved;

  → orders 업데이트 + processing_log 삽입이 원자적
  → 하나의 트랜잭션에서 실행

DML CTE 특성:
  ① 항상 MATERIALIZED (DML Side Effect 보장)
  ② RETURNING을 통해 변경된 행을 다음 CTE에 전달
  ③ 같은 트랜잭션 → DML 여러 단계를 원자적으로 처리

snapshot isolation과 DML CTE:
  같은 쿼리 내 여러 DML은 동일한 스냅샷을 사용
  → CTE 안의 UPDATE가 다른 CTE의 UPDATE를 보지 못함
  → 각 DML은 쿼리 시작 시점의 스냅샷 기준으로 대상 결정
```

### 4. CTE vs 서브쿼리 성능 비교

```
PostgreSQL 12+ 기본 동작:

  CTE (NOT MATERIALIZED 기본):
  동일한 서브쿼리와 거의 같은 실행 계획
  → 옵티마이저가 자유롭게 최적화

  서브쿼리:
  항상 인라인 처리 (MATERIALIZED 없음)

  실제 차이가 없는 경우 (PostgreSQL 12+):
  WITH filtered AS (SELECT * FROM orders WHERE status='DONE')
  SELECT * FROM filtered WHERE user_id = 42;

  vs

  SELECT * FROM (SELECT * FROM orders WHERE status='DONE') filtered
  WHERE user_id = 42;

  → 실행 계획 동일 (NOT MATERIALIZED = 인라인)

CTE를 선택하는 이유:
  ① 가독성: 복잡한 쿼리를 단계별로 명명
  ② 재귀: 서브쿼리로는 불가능한 재귀 탐색
  ③ DML: INSERT/UPDATE/DELETE를 중간 단계로
  ④ 의도적 MATERIALIZED: 비용이 큰 연산을 한 번만

MATERIALIZED 선택 기준:
  여러 번 참조 + 매번 실행 비용이 큰 경우:
    WITH heavy AS MATERIALIZED (
        SELECT ..., count(*), ... FROM large_table GROUP BY ...
    )
    SELECT ... FROM heavy h1 JOIN heavy h2 ON ...;
    → heavy를 1번만 실행
```

---

## 💻 실전 실험

### 실험 1: Optimization Fence 효과 비교

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT,
    status TEXT,
    amount NUMERIC
);

CREATE INDEX ON orders(user_id);
CREATE INDEX ON orders(status);
CREATE INDEX ON orders(user_id, status);

INSERT INTO orders (user_id, status, amount)
SELECT
    (random() * 10000)::INT,
    (ARRAY['PENDING','ACTIVE','DONE'])[ceil(random()*3)::INT],
    (random() * 1000)::NUMERIC
FROM generate_series(1, 500000);

ANALYZE orders;

-- PostgreSQL 12+: NOT MATERIALIZED (기본) - 인덱스 활용
EXPLAIN (ANALYZE, BUFFERS)
WITH active_orders AS (
    SELECT * FROM orders WHERE status = 'ACTIVE'
)
SELECT * FROM active_orders WHERE user_id = 42;

-- MATERIALIZED 강제 - 인덱스 활용 불가
EXPLAIN (ANALYZE, BUFFERS)
WITH active_orders AS MATERIALIZED (
    SELECT * FROM orders WHERE status = 'ACTIVE'
)
SELECT * FROM active_orders WHERE user_id = 42;
-- 비교: Seq Scan on CTE vs Index Scan on orders
```

### 실험 2: 재귀 CTE로 조직도 순회

```sql
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name TEXT,
    parent_id INT REFERENCES departments(id)
);

INSERT INTO departments (id, name, parent_id) VALUES
    (1, 'Company', NULL),
    (2, 'Engineering', 1),
    (3, 'Product', 1),
    (4, 'Backend', 2),
    (5, 'Frontend', 2),
    (6, 'Design', 3),
    (7, 'API Team', 4),
    (8, 'DB Team', 4);

-- 전체 계층 구조 순회
WITH RECURSIVE org_tree AS (
    -- 앵커: 루트 노드
    SELECT id, name, parent_id, 0 AS depth,
           name AS path
    FROM departments
    WHERE parent_id IS NULL

    UNION ALL

    -- 재귀: 자식 노드
    SELECT d.id, d.name, d.parent_id,
           t.depth + 1,
           t.path || ' > ' || d.name
    FROM departments d
    JOIN org_tree t ON d.parent_id = t.id
)
SELECT depth, repeat('  ', depth) || name AS indent_name, path
FROM org_tree
ORDER BY path;

-- 특정 부서의 모든 하위 부서 찾기
WITH RECURSIVE subtree AS (
    SELECT id, name, parent_id FROM departments WHERE id = 2
    UNION ALL
    SELECT d.id, d.name, d.parent_id
    FROM departments d
    JOIN subtree s ON d.parent_id = s.id
)
SELECT * FROM subtree;
```

### 실험 3: DML CTE로 원자적 배치 처리

```sql
-- 작업 큐 시뮬레이션
CREATE TABLE task_queue (
    id SERIAL PRIMARY KEY,
    payload TEXT,
    status TEXT DEFAULT 'PENDING',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE task_results (
    task_id INT,
    result TEXT,
    completed_at TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO task_queue (payload)
SELECT 'Task ' || i FROM generate_series(1, 100) i;

-- 원자적: 작업 가져오기 + 처리 중 표시 + 결과 기록
WITH claimed AS (
    UPDATE task_queue
    SET status = 'PROCESSING'
    WHERE id IN (
        SELECT id FROM task_queue
        WHERE status = 'PENDING'
        ORDER BY id
        LIMIT 5
        FOR UPDATE SKIP LOCKED
    )
    RETURNING id, payload
)
INSERT INTO task_results (task_id, result)
SELECT id, 'Processed: ' || payload
FROM claimed;

-- 확인
SELECT status, count(*) FROM task_queue GROUP BY status;
SELECT * FROM task_results LIMIT 5;
```

---

## 📊 MySQL과 비교

```
MySQL CTE 지원 (8.0+):

MySQL 8.0+: WITH 절 지원, WITH RECURSIVE 지원

차이점:
항목                | MySQL 8.0+           | PostgreSQL
───────────────────┼──────────────────────┼────────────────────────
기본 MATERIALIZED  | 옵티마이저 결정        | PostgreSQL 12: NOT MAT
명시적 키워드       | 없음                 | MATERIALIZED / NOT MAT
DML CTE            | 지원 (제한적)         | 완전 지원 (RETURNING)
CYCLE 절           | 없음                 | PostgreSQL 14+ 지원
재귀 깊이 제한      | cte_max_recursion_depth | LIMIT / depth 컬럼

MySQL CTE 제한:
  DML CTE의 RETURNING 없음 → 변경된 행 참조 어려움
  CYCLE 감지 없음 → 순환 그래프 처리 수동 구현 필요

공통:
  재귀 CTE로 트리 탐색 가능
  WITH RECURSIVE ... UNION ALL 문법 동일
```

---

## ⚖️ 트레이드오프

```
CTE 선택 지침:

NOT MATERIALIZED (기본, PostgreSQL 12+):
  장점: 옵티마이저 자유 최적화, 인덱스 활용, 조건 pushdown
  단점: 여러 번 참조 시 여러 번 실행 가능

MATERIALIZED:
  장점: 결과 캐시 (여러 번 참조 시 1번만 실행)
       DML Side Effect 보장
  단점: 인덱스/조건 pushdown 불가
       중간 결과 메모리 저장

결정 기준:
  참조 1번 + 최적화 원함: NOT MATERIALIZED (기본)
  참조 여러 번 + 비용 큰 연산: MATERIALIZED
  DML 포함: 자동 MATERIALIZED
  가독성만: 어느 쪽이든 동일 (PostgreSQL 12+)

재귀 CTE 주의:
  무한 루프 방지: depth 제한, visited 배열, CYCLE 절
  UNION ALL: 빠름 (중복 허용)
  UNION: 느림 (중복 제거, 대형 데이터에서 피해야 함)
```

---

## 📌 핵심 정리

```
CTE 핵심:

Optimization Fence:
  PostgreSQL 11 이하: 항상 MATERIALIZED → 옵티마이저 제한
  PostgreSQL 12+: 기본 NOT MATERIALIZED → 인라인, 자유 최적화

명시적 키워드:
  WITH x AS MATERIALIZED (...): 결과 캐시, 여러 번 참조
  WITH x AS NOT MATERIALIZED (...): 인라인 (기본)

DML CTE:
  INSERT/UPDATE/DELETE + RETURNING → 다음 CTE에 전달
  항상 MATERIALIZED (Side Effect 보장)

WITH RECURSIVE:
  앵커(비재귀) UNION ALL 재귀(앞 결과 참조)
  BFS 방식으로 반복 실행 → WorkTable 비어있으면 종료
  무한 루프 방지: depth 컬럼 + WHERE depth < N
  또는 CYCLE 절 (PostgreSQL 14+)

성능:
  단순 참조: CTE ≈ 서브쿼리 (PostgreSQL 12+)
  여러 참조 + 비용 큰 연산: MATERIALIZED CTE 우세
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 CTE를 3번 참조할 때 `MATERIALIZED`와 `NOT MATERIALIZED` 중 어느 쪽이 유리한가? 조건에 따라 어떻게 다른가?

<details>
<summary>해설 보기</summary>

**MATERIALIZED가 유리한 경우:** CTE 내부 연산이 비용이 크고 결과 집합이 작을 때. 예를 들어 100만 행을 집계해서 100행의 결과를 만드는 CTE를 3번 참조한다면, NOT MATERIALIZED는 집계를 3번 실행하지만 MATERIALIZED는 1번만 실행합니다.

**NOT MATERIALIZED가 유리한 경우:** 각 참조에 서로 다른 WHERE 조건이 있고, 해당 조건으로 인덱스를 활용할 수 있을 때. MATERIALIZED는 CTE 결과 전체를 임시 저장하고 인덱스가 없으므로, 각 참조에서 선형 스캔이 발생합니다.

요약:
- CTE 비용 큼 + 결과 작음 + 여러 번 참조 → MATERIALIZED
- CTE 비용 작음 + 참조별 필터 다양 + 인덱스 활용 가능 → NOT MATERIALIZED

</details>

---

**Q2.** 재귀 CTE에서 `UNION ALL` 대신 `UNION`을 사용하면 무한 루프가 방지되는가?

<details>
<summary>해설 보기</summary>

부분적으로만 맞습니다. `UNION`은 이미 결과에 있는 행과 완전히 동일한 행(모든 컬럼이 같은)을 제거합니다. 순환 그래프에서 같은 노드 ID가 반복되면 중복 제거로 무한 루프가 방지될 수 있습니다.

하지만 한계가 있습니다:
- 중복 비교는 행 전체를 비교하므로 비용이 큼
- 경로 정보(`path` 컬럼 등)가 달라지면 같은 노드도 다른 행으로 취급 → 여전히 무한 루프 가능

PostgreSQL 14+의 `CYCLE` 절이 더 명확한 해결책입니다:
```sql
WITH RECURSIVE path AS (
    SELECT id FROM nodes WHERE id = 1
    UNION ALL
    SELECT n.id FROM nodes n JOIN path p ON n.parent_id = p.id
) CYCLE id SET is_cycle USING cycle_path
SELECT * FROM path WHERE NOT is_cycle;
```

또는 명시적 depth 제한이 실용적입니다: `WHERE t.depth < 20`.

</details>

---

<div align="center">

**[⬅️ 이전: 윈도우 함수](./01-window-functions.md)** | **[홈으로 🏠](../README.md)** | **[다음: 파티셔닝 완전 분해 ➡️](./03-partitioning.md)**

</div>
