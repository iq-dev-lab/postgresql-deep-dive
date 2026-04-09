# B-Tree 인덱스 심화 — Index-Only Scan과 Partial Index

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PostgreSQL B-Tree와 MySQL B+Tree의 구체적인 차이는?
- Index-Only Scan이 가능한 정확한 조건은 무엇이고 Visibility Map과 어떤 관계인가?
- Partial Index는 인덱스 크기와 성능을 어떻게 개선하는가?
- B-Tree 인덱스 페이지의 내부 구조는 어떻게 생겼는가?
- 인덱스 Bloat이 발생하는 원인과 `REINDEX CONCURRENTLY`는 언제 사용하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL에서 가장 많이 사용되는 인덱스는 B-Tree다. `CREATE INDEX`의 기본값이 B-Tree이고, Primary Key와 Unique Constraint도 내부적으로 B-Tree 인덱스를 생성한다. B-Tree를 깊이 이해하면 Index-Only Scan으로 Heap Fetch를 제거하거나, Partial Index로 인덱스 크기를 수십 배 줄이는 최적화가 가능해진다. PostgreSQL에 Clustered Index가 없다는 점도 MySQL 사용자가 반드시 이해해야 하는 차이다.

---

## 😱 흔한 실수 (Before — B-Tree 내부 모름)

```
실수 1: "인덱스만 추가하면 빨라지겠지" → 인덱스 남발

  CREATE INDEX ON orders(status);
  CREATE INDEX ON orders(created_at);
  CREATE INDEX ON orders(user_id);
  CREATE INDEX ON orders(amount);
  CREATE INDEX ON orders(status, user_id);  -- (status)와 중복

  결과:
  인덱스 5개 → UPDATE 시마다 5개 인덱스 갱신
  → 쓰기 성능 저하
  HOT Update 불가 (모든 컬럼이 인덱스에 있음)
  인덱스 총 크기: 테이블의 2~3배

실수 2: Partial Index를 몰라서 불필요하게 큰 인덱스 유지

  -- 전체 orders에 status 인덱스
  CREATE INDEX ON orders(status);
  -- 99%가 'DONE' 상태 → 인덱스 99%가 쓸모없음
  -- 실제로 'PENDING', 'PROCESSING' 1%만 조회

  올바른 접근:
  CREATE INDEX ON orders(status) WHERE status IN ('PENDING', 'PROCESSING');
  → 인덱스 크기 100분의 1, 성능은 동일
```

---

## ✨ 올바른 접근 (After — B-Tree 이해 기반 인덱스 설계)

```
효과적인 인덱스 설계 원칙:

  1. Index-Only Scan 유도:
     SELECT id, amount FROM orders WHERE user_id = 1;
     → 인덱스: (user_id, amount) 포함 → Heap Fetch 없음

  2. Partial Index 적극 활용:
     -- 미완료 주문만 조회한다면:
     CREATE INDEX ON orders(created_at)
     WHERE status = 'PENDING';
     → 1%의 인덱스 크기로 동일한 성능

  3. 인덱스 사용 여부 확인:
     SELECT indexrelname, idx_scan, idx_tup_read,
            pg_size_pretty(pg_relation_size(indexrelid)) AS size
     FROM pg_stat_user_indexes
     WHERE relname = 'orders'
     ORDER BY idx_scan;
     → idx_scan = 0이면 미사용 인덱스 → 삭제 검토

  4. 복합 인덱스 선두 컬럼 원칙:
     WHERE user_id = ? AND status = ?
     → (user_id, status): user_id 단독 조회도 활용 가능
     → (status, user_id): status 단독은 활용 가능, user_id 단독은 불가
```

---

## 🔬 내부 동작 원리

### 1. PostgreSQL B-Tree 구조

```
B-Tree 인덱스 파일 구조:

Level 0 (Root):
  ┌──────────────────────────────────────────┐
  │  Metapage (page 0): 루트 페이지 번호 등 메타 │
  └──────────────────────────────────────────┘

Level 1 (Root Node):
  ┌─────────────────────────────────────────────────────────┐
  │  Internal Page                                           │
  │  [<100 → page 2] [100~999 → page 3] [≥1000 → page 4]   │
  └─────────────────────────────────────────────────────────┘

Level 2 (Leaf Nodes):
  ┌──────────────────────┐  ┌──────────────────────┐
  │  Leaf Page 2         │  │  Leaf Page 3         │
  │  key=1  → (page,item)│  │  key=100 → (p,i)    │
  │  key=5  → (page,item)│  │  key=200 → (p,i)    │
  │  key=50 → (page,item)│  │  key=999 → (p,i)    │
  │  next_page → Page 3  │←→│  next_page → Page 4  │
  └──────────────────────┘  └──────────────────────┘

PostgreSQL B-Tree vs MySQL B+Tree:

공통점:
  리프 노드끼리 연결 (범위 스캔 지원)
  높이 균일 (Balanced)

차이점:
  항목                | PostgreSQL             | MySQL B+Tree
  ───────────────────┼────────────────────────┼─────────────────────
  테이블과의 관계      | 별도 Heap 파일 참조     | Clustered (데이터=인덱스)
  리프 노드 포함 내용  | key + TID(heap 위치)   | key + 실제 행 데이터 (PK)
  PK 조회             | Index Scan + Heap Fetch | 단일 B-Tree Scan
  Secondary Index     | key + TID              | key + PK 값 (이중 조회)
  NULL 처리           | NULL 인덱싱 가능         | NULL 인덱싱 가능
  Clustering          | CLUSTER 명령 (수동)     | 자동 (PK = Clustered Index)

PostgreSQL 리프 페이지 내부:
  ┌──────────────────────────────────────────────────┐
  │  Page Header                                     │
  │  Item Pointer 배열: [1][2][3]...[N]              │
  │                                                  │
  │  IndexTuple 1: key_value | tid(page, item)       │
  │  IndexTuple 2: key_value | tid(page, item)       │
  │  ...                                             │
  │  left_link → previous leaf page                  │
  │  right_link → next leaf page                     │
  └──────────────────────────────────────────────────┘
```

### 2. Index-Only Scan과 Visibility Map

```
Heap Fetch가 필요한 이유:
  인덱스에서 값을 찾아도 "이 튜플이 내 스냅샷에서 보이는가?" 확인 필요
  → 인덱스에는 MVCC 정보(xmin, xmax) 없음
  → Heap 페이지를 읽어 가시성 확인 필요 (Heap Fetch)

Index-Only Scan 조건:
  ① 필요한 모든 컬럼이 인덱스에 있음 (Covering Index)
  ② Heap 페이지가 "All-Visible" 상태 (Visibility Map 비트 = 1)

  조건 ②가 없으면: Heap Fetch 필요 (Index Scan으로 폴백)

Visibility Map 연계:
  SELECT id, amount FROM orders WHERE user_id = 1;
  → 인덱스 (user_id, id, amount)가 있는 경우

  인덱스 리프 페이지에서 (user_id=1)인 엔트리 찾기
  → 각 엔트리의 TID → Heap 페이지 번호 추출
  → VM에서 해당 페이지의 All-Visible 비트 확인:
    All-Visible = 1: Heap Fetch 없이 인덱스 값 반환
    All-Visible = 0: Heap Fetch 필요 (가시성 확인)

  → 모든 페이지가 All-Visible이면: 완전한 Index-Only Scan!

실행 계획 확인:
  EXPLAIN SELECT id, amount FROM orders WHERE user_id = 1;

  Index Only Scan (최적):
  → Heap Fetches: 0
  → 인덱스만 읽음

  Index Scan (Heap Fetch 있음):
  → Heap Fetches: N
  → 인덱스 + Heap 읽음

  VACUUM 후 Index-Only Scan 가능성 증가:
  VACUUM orders;  → All-Visible 비트 설정
  → 이후 Index-Only Scan으로 전환될 수 있음
```

### 3. Partial Index — 조건부 인덱스

```
Partial Index 기본:
  CREATE INDEX idx_orders_pending
  ON orders(created_at)
  WHERE status = 'PENDING';

  → status = 'PENDING'인 행만 인덱스에 포함
  → 나머지 status는 인덱스에서 완전히 제외

사용 조건:
  WHERE 절이 인덱스 조건과 일치할 때만 인덱스 사용
  SELECT * FROM orders WHERE status = 'PENDING' AND created_at > '2024-01-01';
  → idx_orders_pending 사용 가능 ✓

  SELECT * FROM orders WHERE status = 'DONE' AND created_at > '2024-01-01';
  → idx_orders_pending 사용 불가 (status='DONE'은 인덱스에 없음) ✗

실용적 사용 사례:

  1. 소수의 활성 상태만 조회:
  CREATE INDEX ON orders(created_at) WHERE status IN ('PENDING', 'PROCESSING');
  → 99%가 DONE인 테이블에서 인덱스 크기 100분의 1

  2. NULL 제외 인덱스:
  CREATE INDEX ON users(email) WHERE email IS NOT NULL;
  → email이 NULL인 사용자가 많을 때 인덱스 크기 감소

  3. Soft Delete 패턴:
  CREATE INDEX ON posts(user_id, created_at) WHERE deleted_at IS NULL;
  → 삭제된 글 제외 → 실제 활성 데이터만 인덱스

  4. Unique 제약 (조건부):
  CREATE UNIQUE INDEX ON orders(user_id)
  WHERE status = 'PENDING';
  → "한 사용자당 활성 주문 1개" 제약

인덱스 크기 비교:
  CREATE INDEX idx_full ON orders(created_at);            -- 100% 포함
  CREATE INDEX idx_partial ON orders(created_at)
  WHERE status = 'PENDING';                               -- 1% 포함

  SELECT pg_size_pretty(pg_relation_size('idx_full')) AS full_size,
         pg_size_pretty(pg_relation_size('idx_partial')) AS partial_size;
  -- full_size: 500MB
  -- partial_size: 5MB (99% 감소!)
```

### 4. 인덱스 Bloat과 REINDEX

```
인덱스 Bloat 발생 원인:
  UPDATE/DELETE로 인덱스 엔트리가 Dead 상태가 됨
  VACUUM이 Heap Dead Tuple 제거 + Index 정리
  BUT: 인덱스 페이지 내 공간 재배치는 제한적
  → 인덱스 페이지에 빈 공간이 쌓임 (Bloat)

특히 심한 경우:
  Hot Spot 페이지 (최신 데이터가 항상 마지막 리프):
  예: id SERIAL (단조 증가 PK)
  → 오래된 데이터 삭제 → 초반 리프 페이지들이 희박해짐
  → 인덱스 크기는 유지되나 실제 데이터 밀도 낮음

인덱스 Bloat 측정:
  CREATE EXTENSION pgstattuple;
  SELECT * FROM pgstatindex('orders_pkey');
  -- leaf_pages: 전체 리프 페이지 수
  -- leaf_fragmentation: 조각화율
  -- avg_leaf_density: 리프 페이지 평균 채우기율 (낮으면 Bloat)

REINDEX 방법:
  -- 전통적 (잠금 있음): AccessExclusiveLock
  REINDEX INDEX orders_pkey;

  -- 운영 중 사용 가능 (PostgreSQL 12+): 잠금 최소화
  REINDEX INDEX CONCURRENTLY orders_pkey;

REINDEX CONCURRENTLY 동작:
  1. 새 인덱스 임시 생성 (기존과 병렬)
  2. 새 인덱스에 변경사항 반영 (트리거 방식)
  3. 잠깐 잠금 후 교체
  → VACUUM FULL과 유사한 방식으로 인덱스 재구성

  단, 오래 걸릴 수 있고 실패 시 수동 정리 필요:
  SELECT indexrelname, indisvalid
  FROM pg_index i JOIN pg_class c ON c.oid = i.indexrelid
  WHERE NOT indisvalid;
  -- CONCURRENTLY 실패 시 "INVALID" 상태 인덱스 남음
  -- → DROP INDEX 후 재시도
```

---

## 💻 실전 실험

### 실험 1: Index-Only Scan 조건 실험

```sql
-- 실험 테이블 및 인덱스
CREATE TABLE io_scan_test (
    user_id INT,
    amount  NUMERIC,
    status  TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Covering Index (user_id + amount 포함)
CREATE INDEX ON io_scan_test(user_id, amount);

INSERT INTO io_scan_test (user_id, amount, status)
SELECT (random() * 1000)::INT, random() * 10000, 'DONE'
FROM generate_series(1, 100000);

-- ANALYZE 실행 (통계 갱신)
ANALYZE io_scan_test;

-- Index-Only Scan 확인 (VACUUM 전)
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, sum(amount)
FROM io_scan_test
WHERE user_id = 42
GROUP BY user_id;
-- "Index Only Scan"이지만 Heap Fetches > 0 (VM 비트 없음)

-- VACUUM으로 VM 비트 설정
VACUUM io_scan_test;

-- VACUUM 후 Index-Only Scan 확인
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, sum(amount)
FROM io_scan_test
WHERE user_id = 42
GROUP BY user_id;
-- Heap Fetches: 0 → 완전한 Index-Only Scan!
```

### 실험 2: Partial Index 크기 비교

```sql
-- 테이블: 100만 행, 99%가 'DONE', 1%가 'PENDING'
CREATE TABLE partial_test (
    id SERIAL PRIMARY KEY,
    status TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO partial_test (status, created_at)
SELECT
    CASE WHEN i % 100 = 0 THEN 'PENDING' ELSE 'DONE' END,
    NOW() - (random() * INTERVAL '365 days')
FROM generate_series(1, 1000000) i;

-- 전체 인덱스
CREATE INDEX idx_full_status ON partial_test(created_at);

-- Partial 인덱스
CREATE INDEX idx_partial_pending ON partial_test(created_at)
WHERE status = 'PENDING';

-- 크기 비교
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan
FROM pg_stat_user_indexes
WHERE relname = 'partial_test';

-- 쿼리 성능 비교
EXPLAIN ANALYZE
SELECT * FROM partial_test
WHERE status = 'PENDING' AND created_at > NOW() - INTERVAL '7 days';
-- idx_partial_pending 사용 (작은 인덱스)
```

### 실험 3: 인덱스 사용률 확인

```sql
-- 인덱스 사용 현황 (미사용 인덱스 찾기)
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    CASE WHEN idx_scan = 0 THEN '⚠️ 미사용' ELSE '✅ 사용중' END AS status
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan, pg_relation_size(indexrelid) DESC;
```

---

## 📊 MySQL과 비교

```
인덱스 구조 근본 차이:

PostgreSQL:
  모든 테이블 = Heap (순서 없음)
  PRIMARY KEY = B-Tree 인덱스 (Heap의 TID 참조)
  Secondary Index = B-Tree 인덱스 (Heap의 TID 참조)
  PK 조회: Index Scan + Heap Fetch (2단계)
  Secondary 조회: Index Scan + Heap Fetch (2단계)

MySQL InnoDB:
  모든 테이블 = Clustered Index (PK 순서로 저장)
  PRIMARY KEY = 데이터 자체 (추가 Heap 없음)
  Secondary Index = B-Tree (PK 값 참조)
  PK 조회: 단일 B-Tree Scan (1단계)
  Secondary 조회: Secondary Index + PK Index (2단계, 이중 조회)

어느 방식이 유리한가:

  PK 기반 조회:
  MySQL 유리 (단일 B-Tree Scan vs PostgreSQL의 Index+Heap)

  Secondary Index 기반 조회:
  유사 (둘 다 2단계, PostgreSQL의 Index-Only Scan = MySQL Covering Index와 유사)

  대량 INSERT 순서:
  PostgreSQL: 순서 무관 (Heap에 append) → 빠름
  MySQL: PK 순서가 아니면 B-Tree 재정렬 → 느릴 수 있음

  UPDATE 패턴:
  PostgreSQL: HOT Update로 인덱스 갱신 없음 가능
  MySQL: In-Place 업데이트 (PK 포함 시 B-Tree 재정렬 필요)
```

---

## ⚖️ 트레이드오프

```
B-Tree 인덱스의 장단점:

장점:
  ① 범위 쿼리, 정렬, 등가 비교 모두 지원
  ② Index-Only Scan으로 Heap Fetch 제거 가능
  ③ 복합 인덱스 (선두 컬럼 부분 활용 가능)
  ④ Partial Index로 크기 최소화

단점:
  ① JSONB 내부 검색: GIN이 우월
  ② 전문 검색: GIN이 우월
  ③ 공간 데이터: GiST가 우월
  ④ 시계열 대용량 데이터: BRIN이 효율적

인덱스 유지 비용:
  인덱스 추가 → UPDATE 시 인덱스 페이지도 수정 → WAL 생성
  HOT Update 불가 → 더 많은 I/O
  → 인덱스는 "읽기 이득 vs 쓰기 비용"의 트레이드오프
  → 실제로 사용되는 쿼리 기준으로 최소한의 인덱스
```

---

## 📌 핵심 정리

```
B-Tree 인덱스 핵심:

구조:
  리프 노드: key + TID (Heap 위치)
  리프 노드끼리 연결 → 범위 스캔 지원
  MySQL과 달리 Heap과 분리 → PK도 별도 인덱스

Index-Only Scan:
  조건: 컬럼이 인덱스에 포함 + 페이지 All-Visible
  VACUUM → All-Visible 설정 → Index-Only Scan 가능
  Heap Fetch 없음 → I/O 최소화

Partial Index:
  WHERE 절로 일부 행만 인덱스에 포함
  소수 활성 상태 조회: 인덱스 크기 수십~수백분의 1
  활용: soft delete, 상태 기반 필터링

인덱스 Bloat:
  VACUUM으로 일부 정리, 심하면 REINDEX CONCURRENTLY
  운영 중 재구성 가능 (PostgreSQL 12+)

미사용 인덱스:
  pg_stat_user_indexes.idx_scan = 0 → 삭제 검토
  쓰기 비용 절감 + HOT Update 가능성 증가
```

---

## 🤔 생각해볼 문제

**Q1.** 복합 인덱스 `(a, b, c)`가 있을 때, `WHERE b = 1 AND c = 2` 쿼리는 이 인덱스를 사용할 수 있는가?

<details>
<summary>해설 보기</summary>

일반적으로 사용할 수 없습니다. B-Tree 복합 인덱스는 **선두 컬럼부터 순서대로** 사용해야 합니다. `(a, b, c)` 인덱스는:
- `WHERE a = 1`: 사용 가능
- `WHERE a = 1 AND b = 2`: 사용 가능
- `WHERE a = 1 AND b = 2 AND c = 3`: 사용 가능
- `WHERE b = 1 AND c = 2`: **사용 불가** (a 없음)
- `WHERE a = 1 AND c = 3`: a에 대해서는 사용, c는 Heap Fetch 후 필터링

예외: PostgreSQL의 `Index Skip Scan`은 PostgreSQL 16에서 실험적으로 지원되기 시작했습니다. 일부 경우 선두 컬럼 없이도 인덱스를 활용할 수 있습니다.

`WHERE b = 1 AND c = 2` 쿼리가 필요하다면 `(b, c)` 인덱스를 별도로 생성하거나 Partial Index를 활용하는 것이 올바른 접근입니다.

</details>

---

**Q2.** `CREATE INDEX CONCURRENTLY`와 `REINDEX INDEX CONCURRENTLY`의 차이는?

<details>
<summary>해설 보기</summary>

**`CREATE INDEX CONCURRENTLY`**: 새 인덱스를 **처음 생성**할 때 잠금 없이 생성. 테이블에 여러 번 스캔을 수행하고 그 사이에 발생한 변경도 처리. 완료까지 오래 걸리지만 DML 차단 없음.

**`REINDEX INDEX CONCURRENTLY`**: 기존 인덱스를 새로 **재구성**. 인덱스 Bloat 해소나 파라미터 변경이 목적. `CREATE INDEX CONCURRENTLY`로 임시 인덱스를 만든 뒤 교체하는 방식으로 동작. 잠금 최소화.

공통 주의사항:
- 실패 시 INVALID 상태 인덱스가 남을 수 있음 → 확인 후 삭제 재시도
- 트랜잭션 블록 안에서 실행 불가
- 실행 중에는 해당 테이블에 다른 `CONCURRENTLY` 작업 불가

```sql
-- INVALID 인덱스 확인
SELECT indexrelname FROM pg_stat_user_indexes i
JOIN pg_index pi ON pi.indexrelid = i.indexrelid
WHERE NOT pi.indisvalid;

-- INVALID 인덱스 제거 후 재생성
DROP INDEX CONCURRENTLY idx_invalid;
CREATE INDEX CONCURRENTLY idx_new ON orders(user_id);
```

</details>

---

<div align="center">

**[⬅️ 이전 챕터: 격리 수준과 스냅샷](../mvcc-vacuum/07-isolation-snapshot.md)** | **[홈으로 🏠](../README.md)** | **[다음: Hash 인덱스 ➡️](./02-hash-index.md)**

</div>
