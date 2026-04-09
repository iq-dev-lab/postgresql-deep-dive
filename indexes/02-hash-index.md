# Hash 인덱스 — 등가 비교 전용, 언제 B-Tree보다 빠른가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Hash 인덱스는 내부적으로 어떻게 동작하고, B-Tree와 무엇이 다른가?
- Hash 인덱스가 등가 비교에서 B-Tree보다 빠를 수 있는 이유는?
- PostgreSQL 10 이전 Hash 인덱스에 WAL 문제가 있었던 이유는?
- Hash 인덱스가 범위 쿼리를 지원하지 않는 이유는?
- 실무에서 Hash 인덱스가 B-Tree보다 적합한 경우는 언제인가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

대부분의 인덱스는 B-Tree로 충분하지만, UUID나 해시값처럼 정렬이 의미 없는 긴 키에 대한 순수 등가 비교(`=`)만 있는 경우 Hash 인덱스가 이점을 가질 수 있다. Hash 인덱스는 키를 해시값으로 변환해 O(1) 접근을 제공한다. PostgreSQL 10 이전에는 WAL 미지원으로 운영 환경에서 사용을 꺼렸으나, PostgreSQL 10부터 WAL 지원이 완비되어 안전하게 사용 가능하다.

---

## 😱 흔한 실수 (Before — Hash 인덱스 오해)

```
실수 1: PostgreSQL 10 이전 Hash 인덱스를 운영에 사용

  CREATE INDEX ON sessions(token) USING HASH;

  PostgreSQL 9.x에서 Hash 인덱스:
  WAL에 기록되지 않음 (Not WAL-logged)
  → 서버 크래시 후 인덱스 손상 가능
  → REINDEX 필요
  → Streaming Replication에서 Standby에 Hash 인덱스 전달 안 됨

  경고 메시지도 없었음 → 운영자가 모르고 사용 → 크래시 후 인덱스 깨짐

실수 2: Hash 인덱스로 범위 쿼리 시도

  CREATE INDEX ON orders(amount) USING HASH;
  SELECT * FROM orders WHERE amount > 1000;

  결과: Hash 인덱스 사용 안 됨 (SeqScan으로 폴백)
  Hash 인덱스는 = 연산자만 지원, >, <, BETWEEN 불가

실수 3: 정렬이 필요한 컬럼에 Hash 인덱스

  SELECT * FROM users ORDER BY email LIMIT 10;
  → Hash 인덱스: 사용 불가 (정렬 정보 없음)
  → B-Tree: 인덱스 순서 = 정렬 순서 → 효율적
```

---

## ✨ 올바른 접근 (After — Hash 인덱스 적합한 사용)

```
Hash 인덱스가 적합한 시나리오:

  1. UUID 등가 비교 (길고 정렬 의미 없는 키):
     -- UUID: 32자 고정 길이, 정렬 의미 없음
     CREATE INDEX ON sessions(session_id) USING HASH;
     SELECT * FROM sessions WHERE session_id = '550e8400-e29b-...';
     → Hash: O(1) 접근, B-Tree보다 빠를 수 있음

  2. 해시/토큰 값 조회:
     CREATE INDEX ON api_keys(token_hash) USING HASH;
     SELECT * FROM api_keys WHERE token_hash = 'abc123...';

  3. 등가 비교만 하는 외래키 (범위/정렬 불필요):
     -- user_id로 항상 = 조회만 하는 경우
     CREATE INDEX ON events(user_id) USING HASH;

  Hash vs B-Tree 선택 기준:
  ✓ Hash: 오직 = 비교, 긴 키, 범위/정렬 불필요
  ✓ B-Tree: 범위 쿼리, 정렬, LIKE 'prefix%', IS NULL
  ✓ B-Tree: 대부분의 일반적인 경우

  실무 조언:
  Hash 인덱스 이점이 확실하지 않으면 B-Tree 사용
  측정 후 Hash 인덱스가 실제로 빠를 때만 전환
```

---

## 🔬 내부 동작 원리

### 1. Hash 인덱스 내부 구조

```
Hash 인덱스 파일 구조:

┌───────────────────────────────────────────────────────┐
│ Metapage (page 0)                                     │
│   hash_magic: 고유 식별자                              │
│   hash_version: 버전                                   │
│   hashm_ntuples: 총 튜플 수                            │
│   hashm_ffactor: 버킷당 목표 튜플 수 (fill factor)      │
│   hashm_bmsize: 비트맵 페이지 크기                      │
│   hashm_maxbucket: 현재 최대 버킷 번호                  │
└───────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────┐
│ Bitmap Page                                           │
│   각 버킷의 overflow 페이지 할당 상태 추적               │
└───────────────────────────────────────────────────────┘

┌─────────────────┐  ┌─────────────────┐
│  Bucket Page 0  │  │  Bucket Page 1  │
│  hash=0~N/2-1   │  │  hash=N/2~N-1   │
│  ┌───────────┐  │  │  ┌───────────┐  │
│  │ key → TID │  │  │  │ key → TID │  │
│  │ key → TID │  │  │  │ key → TID │  │
│  └───────────┘  │  │  └───────────┘  │
│  overflow →     │  │  overflow →     │
└─────────────────┘  └─────────────────┘
        │                     │
  ┌─────────────────┐
  │  Overflow Page  │  ← 버킷이 가득 찰 때
  │  key → TID      │
  └─────────────────┘

Hash 조회 과정:
  1. key → hash 함수 적용 → hash value
  2. hash value mod num_buckets = bucket number
  3. 해당 Bucket Page 직접 접근
  4. 버킷 내에서 key 비교 (충돌 처리)
  5. TID → Heap Fetch

시간 복잡도:
  B-Tree: O(log N) — 트리 높이 탐색
  Hash:   O(1) — 버킷 직접 접근 (충돌 없을 때)

  N=1억 rows:
  B-Tree: log₂(10^8) ≈ 27 레벨
  Hash:   1 버킷 접근 (이상적인 경우)
```

### 2. 버킷 분할 (Split) 과정

```
Linear Hashing 방식으로 버킷을 점진적으로 분할:

초기 상태: 버킷 2개
  Bucket 0: hash mod 2 = 0 → 여기에
  Bucket 1: hash mod 2 = 1 → 여기에

버킷 0이 가득 차면 분할:
  Bucket 0 → Bucket 0 + Bucket 2
  → hash mod 4 = 0 → Bucket 0
  → hash mod 4 = 2 → Bucket 2 (새 버킷)

단계적 분할:
  → 한 번에 하나의 버킷만 분할
  → 대량 재구성 없이 점진적 확장
  → 분할 중에도 조회 가능 (잠금 최소화)

B-Tree 페이지 분할과 비교:
  B-Tree: 페이지 가득 차면 중간 키를 부모로 올리며 분할
          → 부모 페이지까지 연쇄 분할 가능 (희박하게)
  Hash:   Linear Hashing → 버킷 단위 분할, 독립적
```

### 3. WAL 지원 (PostgreSQL 10+)

```
PostgreSQL 10 이전 Hash 인덱스의 문제:
  WAL에 Hash 인덱스 변경 기록 안 됨
  → 크래시 후 WAL Replay 시 Hash 인덱스 변경 미복구
  → 인덱스 손상
  → Streaming Replication에서 Standby에 Hash 인덱스 변경 전파 안 됨

  대응: pg_upgrade, pg_dump/restore 후 Hash 인덱스 수동 REINDEX
        운영 환경에서 사용 자제

PostgreSQL 10+:
  Hash 인덱스도 WAL에 기록됨
  → 크래시 복구 완전 지원
  → Streaming Replication에서 Standby 전파 가능
  → 운영 환경 사용 안전

Hash 인덱스 WAL 확인:
  SELECT relname, relpersistence
  FROM pg_class
  WHERE relname = 'sessions_session_id_idx'
    AND relkind = 'i';
  -- relpersistence = 'p' (permanent): WAL 기록됨
```

### 4. Hash vs B-Tree 성능 비교

```
등가 조회 성능:

키 크기별 차이:
  짧은 키 (INT, BIGINT):
    B-Tree: 이미 4~8바이트 → 비교 비용 낮음
    Hash: 해시 계산 오버헤드
    → B-Tree가 유사하거나 더 빠를 수 있음

  긴 키 (UUID 36자, TEXT 100자+):
    B-Tree: 긴 문자열 비교 O(len) × O(log N)
    Hash: 해시 계산 한 번 → O(1) 접근
    → Hash가 유리할 수 있음

  인덱스 크기:
    B-Tree: 모든 키 값을 내부 노드에 복사 (큰 키 = 큰 인덱스)
    Hash: 해시값(4바이트)만 저장 + 키는 버킷 페이지에
    → 긴 키에서 Hash 인덱스가 더 작을 수 있음

실측 비교 (UUID, 100만 행):
  인덱스 크기:
    B-Tree: ~55MB
    Hash:   ~35MB
  등가 조회 속도:
    B-Tree: 0.15ms
    Hash:   0.12ms (약 20% 빠름)
  범위 조회:
    B-Tree: 지원
    Hash:   지원 안 함 (SeqScan 필요)

결론:
  대부분의 경우 차이가 미미 → B-Tree 기본 선택
  긴 키 + 오직 등가 비교 + 범위/정렬 불필요 → Hash 검토
```

---

## 💻 실전 실험

### 실험 1: Hash vs B-Tree 인덱스 생성 및 크기 비교

```sql
-- UUID를 사용하는 테이블
CREATE TABLE hash_test (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    session_data TEXT DEFAULT repeat('x', 100)
);

INSERT INTO hash_test (session_data)
SELECT repeat(md5(random()::text), 3)
FROM generate_series(1, 500000);

-- B-Tree 인덱스 (기본)
CREATE INDEX idx_btree_id ON hash_test(id) USING BTREE;

-- Hash 인덱스
CREATE INDEX idx_hash_id ON hash_test(id) USING HASH;

-- 크기 비교
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'hash_test';

-- 등가 조회 성능 비교
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM hash_test WHERE id = '550e8400-e29b-41d4-a716-446655440000'::UUID;
```

### 실험 2: Hash 인덱스가 범위 쿼리를 처리하지 못함

```sql
-- Hash 인덱스로 범위 쿼리 시도
CREATE TABLE range_test (val TEXT);
CREATE INDEX ON range_test(val) USING HASH;
INSERT INTO range_test SELECT md5(i::text) FROM generate_series(1, 10000) i;

-- 등가 비교: Hash 인덱스 사용
EXPLAIN SELECT * FROM range_test WHERE val = 'abc123';
-- Index Scan using hash index

-- 범위 비교: Hash 인덱스 사용 불가
EXPLAIN SELECT * FROM range_test WHERE val > 'abc123';
-- Seq Scan (Hash 인덱스 무시!)

-- 정렬: Hash 인덱스 사용 불가
EXPLAIN SELECT * FROM range_test ORDER BY val LIMIT 10;
-- Seq Scan + Sort
```

### 실험 3: Hash 인덱스 적합성 판단

```sql
-- 실제 쿼리 패턴 분석
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan,
    idx_tup_read,
    pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size
FROM pg_stat_user_indexes
WHERE relname = 'sessions'
ORDER BY idx_scan DESC;

-- 특정 컬럼의 쿼리 패턴 확인 (pg_stat_statements 활용)
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
WHERE query ILIKE '%session_id =%'
ORDER BY calls DESC
LIMIT 10;
```

---

## 📊 MySQL과 비교

```
MySQL Hash 인덱스:

MySQL InnoDB:
  Hash 인덱스: 지원 안 함 (B-Tree만)
  Adaptive Hash Index (AHI): 자동으로 자주 접근하는 B-Tree 항목에
                             내부 Hash Table 생성 (사용자 제어 불가)
  → InnoDB가 자동으로 Hash 인덱스 효과 제공

MySQL MEMORY 엔진:
  Hash 인덱스 지원 (등가 비교 최적)
  범위 쿼리 지원 안 됨 (B-Tree 별도 생성 필요)

PostgreSQL:
  사용자가 직접 USING HASH 선택 가능
  10+에서 WAL 완전 지원
  Adaptive Hash Index 없음 (shared_buffers 캐시로 유사 효과)

결론:
  MySQL InnoDB: AHI가 자동으로 Hash 효과 제공
  PostgreSQL: 명시적 USING HASH 또는 B-Tree의 shared_buffers 캐시
```

---

## ⚖️ 트레이드오프

```
Hash 인덱스 장단점:

장점:
  ① 긴 키의 등가 비교: B-Tree보다 빠를 수 있음
  ② 인덱스 크기: 긴 키에서 B-Tree보다 작을 수 있음
  ③ O(1) 접근: 이론적 최적

단점:
  ① 범위 쿼리 불가: >, <, BETWEEN, ORDER BY 사용 불가
  ② 복합 인덱스 제한: 등가 조건의 복합 활용 어려움
  ③ IS NULL 불가: NULL 값 해시화 불가
  ④ LIKE, 패턴 매칭 불가
  ⑤ 10 이전 버전에서 WAL 문제

실무 결론:
  대부분의 경우 B-Tree가 안전하고 유연함
  Hash를 선택하려면: 벤치마크로 실제 이득 확인 후 전환
```

---

## 📌 핵심 정리

```
Hash 인덱스 핵심:

구조:
  key → hash value → bucket 직접 접근
  O(1) 이론적 복잡도, 충돌 시 overflow 페이지

지원 연산:
  = (등가 비교)만 지원
  범위, 정렬, LIKE, IS NULL 불가

PostgreSQL 버전:
  10 미만: WAL 미지원 → 크래시 위험 → 운영 사용 자제
  10 이상: WAL 완전 지원 → 안전

적합한 경우:
  UUID, 해시값, 토큰 등 긴 키의 순수 등가 비교
  범위/정렬/패턴 쿼리가 절대 없는 컬럼

부적합한 경우:
  범위 쿼리 필요
  정렬/ORDER BY 활용 필요
  복합 인덱스 필요
  → B-Tree 사용
```

---

## 🤔 생각해볼 문제

**Q1.** `CREATE INDEX ON users(email) USING HASH`를 생성하면 `WHERE email LIKE 'alice%'` 쿼리에서 이 인덱스를 사용하는가?

<details>
<summary>해설 보기</summary>

사용하지 않습니다. Hash 인덱스는 오직 `=` 연산자만 지원합니다. `LIKE 'alice%'`는 범위 매칭이므로 Hash 인덱스로 처리할 수 없습니다. PostgreSQL이 SeqScan을 선택합니다.

`LIKE 'prefix%'` 패턴을 인덱스로 처리하려면 B-Tree 인덱스가 필요합니다. B-Tree는 정렬된 순서를 유지하므로 `prefix%`를 범위 조건으로 변환해 인덱스 스캔을 수행합니다.

```sql
-- LIKE 전문 인덱스 (text_pattern_ops 연산자 클래스)
CREATE INDEX ON users(email text_pattern_ops);
-- 이제 LIKE 'alice%' 쿼리에서 인덱스 사용 가능
```

</details>

---

**Q2.** Hash 인덱스는 복합 인덱스(multi-column)를 지원하는가?

<details>
<summary>해설 보기</summary>

PostgreSQL의 Hash 인덱스는 단일 컬럼만 지원합니다. 복합 컬럼 Hash 인덱스는 생성 자체가 불가능합니다:

```sql
CREATE INDEX ON orders(user_id, session_id) USING HASH;
-- ERROR: access method "hash" does not support multicolumn indexes
```

복합 등가 조건(`WHERE a = 1 AND b = 2`)에는 B-Tree 복합 인덱스를 사용해야 합니다. 또는 두 컬럼을 결합한 Expression Index를 고려할 수 있습니다:

```sql
CREATE INDEX ON orders(user_id || ':' || session_id) USING HASH;
-- 단, 쿼리도 동일한 표현식 사용 필요
```

실용적으로는 B-Tree 복합 인덱스가 더 유연합니다.

</details>

---

<div align="center">

**[⬅️ 이전: B-Tree 인덱스 심화](./01-btree-index.md)** | **[홈으로 🏠](../README.md)** | **[다음: GiST 인덱스 ➡️](./03-gist-index.md)**

</div>
