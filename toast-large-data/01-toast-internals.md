# TOAST 완전 분해 — 8KB 페이지를 넘는 값은 어디로 가는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 8KB 페이지에 1MB짜리 TEXT 값을 저장하면 정확히 어떤 일이 일어나는가?
- TOAST의 4가지 전략(PLAIN/EXTENDED/EXTERNAL/MAIN)은 각각 언제 사용되는가?
- pglz와 lz4 압축 중 어떤 것이 더 빠르고 어떤 것이 더 작은가?
- TOAST가 애플리케이션에 투명한 이유는 무엇인가?
- TOAST 때문에 쿼리가 예상보다 느려지는 경우는 어떤 패턴인가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL의 모든 행은 8KB 페이지 안에 저장된다. JSONB, TEXT, BYTEA처럼 큰 값이 들어오면 8KB 한계를 어떻게 넘는가? 답이 바로 TOAST(The Oversized-Attribute Storage Technique)다. TOAST는 PostgreSQL이 투명하게 관리하므로 애플리케이션 코드를 수정하지 않아도 되지만, 내부 동작을 모르면 "왜 이 컬럼만 SELECT하는 게 느리지?", "VACUUM이 왜 이렇게 많은 공간을 쓰지?"를 이해하기 어렵다. 특히 JSONB 컬럼이 많은 현대 서비스에서 TOAST 이해는 필수다.

---

## 😱 흔한 실수 (Before — TOAST 모름)

```
실수 1: 모든 컬럼을 SELECT * 로 조회

  SELECT * FROM events WHERE id = 1;
  → events 테이블에 payload JSONB 컬럼 (평균 50KB)
  → TOAST에서 payload를 매번 decompress + Heap Fetch
  → 실제 필요한 컬럼: id, type, created_at (몇 바이트)

  결과:
  10만 건 조회 시 TOAST 해제 비용이 조회 시간의 80%
  SELECT id, type, created_at FROM events WHERE ...;
  → 10배 이상 빠름 (TOAST 컬럼 조회 없음)

실수 2: TOAST 컬럼에 인덱스 없이 GIN 쿼리

  -- payload가 TOAST에 저장된 상태
  SELECT * FROM events WHERE payload @> '{"type": "click"}';
  → GIN 인덱스 없으면: 각 행마다 TOAST decompression + 파싱
  → SeqScan + TOAST 해제 = 극히 느림

  GIN 인덱스를 만들면:
  CREATE INDEX ON events USING GIN(payload);
  → JSONB 원소가 역색인됨 (TOAST와 별도)
  → payload 전체를 가져오지 않아도 검색 가능

실수 3: TOAST 테이블이 비대해지는 현상 방치

  TOAST 테이블도 Dead Tuple 발생
  → VACUUM이 기본 테이블뿐만 아니라 TOAST 테이블도 처리해야 함
  → TOAST 테이블이 크면 VACUUM 시간 급증
  → pg_toast 스키마 테이블 크기 모니터링 필요
```

---

## ✨ 올바른 접근 (After — TOAST 이해 기반 설계)

```
TOAST 친화적 설계:

  1. 꼭 필요한 컬럼만 SELECT:
     SELECT id, event_type, created_at FROM events WHERE ...;
     -- payload (TOAST 컬럼) 제외 → TOAST 접근 없음

  2. TOAST 전략 명시적 설정:
     -- 자주 전체를 읽는 컬럼: EXTENDED (기본, 압축 + 분리)
     -- 자주 일부만 읽는 컬럼: EXTERNAL (압축 안 함, 슬라이싱 가능)
     ALTER TABLE events ALTER COLUMN payload SET STORAGE EXTERNAL;

  3. 압축 알고리즘 선택 (PostgreSQL 14+):
     SET default_toast_compression = 'lz4';  -- 더 빠른 압축/해제
     -- 또는 컬럼별 설정:
     ALTER TABLE events ALTER COLUMN payload SET COMPRESSION lz4;

  4. TOAST 테이블 크기 모니터링:
     SELECT
         c.relname AS main_table,
         t.relname AS toast_table,
         pg_size_pretty(pg_total_relation_size(c.oid)) AS main_size,
         pg_size_pretty(pg_total_relation_size(t.oid)) AS toast_size
     FROM pg_class c
     JOIN pg_class t ON t.oid = c.reltoastrelid
     WHERE c.relkind = 'r' AND c.reltoastrelid != 0
     ORDER BY pg_total_relation_size(t.oid) DESC
     LIMIT 10;
```

---

## 🔬 내부 동작 원리

### 1. TOAST 임계값과 분리 저장

```
TOAST 동작 조건:

  행 크기 임계값: TOAST_TUPLE_THRESHOLD = 2KB (2,048 bytes)
  행이 2KB를 초과하면 TOAST 처리 시작

  목표 크기: TOAST_TUPLE_TARGET = 2KB (페이지 4개 행 이상 유지)

TOAST 처리 순서:

  ① 행 크기 > 2KB → TOAST 후보 컬럼 선택
  ② 우선순위 1: EXTENDED/MAIN 컬럼 압축 시도
     압축 후 2KB 이하이면 → 압축된 값을 Heap에 인라인 저장 (분리 없음)
  ③ 우선순위 2: 압축해도 2KB 초과 → TOAST 테이블로 분리
     컬럼 값을 청크로 분할 → TOAST 테이블에 저장
     Heap에는 TOAST 포인터(18 bytes) 저장

Heap 페이지의 TOAST 포인터:

  ┌─────────────────────────────────────┐
  │  Heap Tuple (2KB 이하)              │
  │  id:     4 bytes                   │
  │  type:   변수 길이 (짧은 경우 인라인) │
  │  payload: 18 bytes (TOAST 포인터)  │
  │    va_tag:    압축/분리 여부 플래그  │
  │    va_rawsize: 원래 값 크기         │
  │    va_toastrelid: TOAST 테이블 OID  │
  │    va_valueid: TOAST 값 고유 ID     │
  └─────────────────────────────────────┘

TOAST 테이블 (pg_toast.pg_toast_XXXXXX):

  chunk_id:  TOAST 값의 고유 ID (va_valueid와 매칭)
  chunk_seq: 청크 순서 번호 (0, 1, 2, ...)
  chunk_data: 실제 데이터 청크 (최대 ~2KB)

  1MB 값 → 약 512개 청크 (각 2KB)
  → TOAST 테이블에 512행으로 저장
```

### 2. TOAST 4가지 전략

```
전략별 동작:

PLAIN (저장 그대로):
  압축 안 함, 분리 안 함
  값이 항상 Heap 페이지 인라인에 저장
  적합: 짧고 고정 길이 값 (INT, SMALLINT, CHAR 등)
  주의: 큰 값이면 행 크기가 8KB 초과 → 오류 발생 가능

EXTENDED (기본값, 대부분의 가변 길이 타입):
  1단계: 압축 시도 (pglz 또는 lz4)
  2단계: 압축해도 크면 → 압축 후 TOAST 테이블로 분리
  장점: 최대한 작게 저장
  단점: 전체 값 읽을 때 decompress 비용

EXTERNAL (압축 없이 분리):
  압축 안 함, 크면 TOAST 테이블로 분리
  장점: 부분 읽기(slicing) 가능 → substring() 등이 전체 읽지 않음
  단점: 압축보다 저장 공간 더 사용
  적합: 대용량이지만 자주 일부만 읽는 컬럼 (파일 내용, 긴 로그)

  예시:
  SELECT substring(content FROM 1 FOR 100) FROM documents WHERE id = 1;
  → EXTENDED: 전체 청크 읽기 + decompress + substring
  → EXTERNAL: 앞 100바이트 청크만 읽기 → I/O 절약

MAIN (압축 우선, 분리 최후):
  1단계: 압축 시도
  2단계: 압축 후에도 크면 → 다른 전략의 컬럼을 먼저 분리
  3단계: 최후에 이 컬럼 분리
  적합: 가능하면 인라인 유지하고 싶은 컬럼

전략 설정:
  ALTER TABLE docs ALTER COLUMN content SET STORAGE EXTERNAL;
  ALTER TABLE events ALTER COLUMN payload SET STORAGE EXTENDED;

  현재 전략 확인:
  SELECT attname, attstorage
  FROM pg_attribute
  WHERE attrelid = 'events'::regclass AND attnum > 0;
  -- p: PLAIN, e: EXTENDED, x: EXTERNAL, m: MAIN
```

### 3. 압축 알고리즘 비교

```
pglz (기본값, PostgreSQL 전통 알고리즘):
  특징: 압축률 높음, 속도 느림
  압축 비율: 텍스트에서 40~70% 압축
  CPU 비용: 높음

lz4 (PostgreSQL 14+):
  특징: 압축률 약간 낮음, 속도 매우 빠름
  압축 비율: 텍스트에서 30~60% 압축
  CPU 비용: 낮음 (pglz의 1/5 수준)

비교 (JSON 100KB 기준):
  pglz: 30KB로 압축, 압축 8ms, 해제 2ms
  lz4:  35KB로 압축, 압축 1ms, 해제 0.5ms

선택 기준:
  디스크 비용 절약 우선 → pglz
  압축/해제 속도 우선 → lz4 (고빈도 TOAST 읽기 시)

설정 방법:
  -- PostgreSQL 14+ 기본값 변경
  SET default_toast_compression = 'lz4';
  ALTER SYSTEM SET default_toast_compression = 'lz4';

  -- 컬럼별 설정
  ALTER TABLE events ALTER COLUMN payload SET COMPRESSION lz4;

  -- 현재 설정 확인
  SELECT attname, attcompression
  FROM pg_attribute
  WHERE attrelid = 'events'::regclass AND attnum > 0;
  -- 빈 문자열: 기본값 사용, 'l': lz4, 'p': pglz
```

### 4. TOAST가 투명한 이유

```
투명성의 구현:

애플리케이션 관점:
  INSERT INTO docs (title, content) VALUES ('제목', '...1MB 텍스트...');
  → PostgreSQL이 자동으로 TOAST 처리
  → 애플리케이션 코드 변경 없음

  SELECT content FROM docs WHERE id = 1;
  → PostgreSQL이 자동으로 TOAST 복원
  → 애플리케이션은 완전한 값을 받음

내부 구현:
  heap_form_tuple(): 행 구성 시 큰 값 감지 → TOAST 저장 호출
  heap_deform_tuple(): 행 분해 시 TOAST 포인터 감지 → TOAST 읽기 호출
  toast_fetch_datum(): TOAST 청크 수집 + 순서 정렬 + 재조합
  → 이 모든 과정이 SQL 레이어 아래에서 투명하게 수행

lazily detoasted:
  SELECT id, type FROM events WHERE id = 1;
  → content 컬럼이 TOAST에 있어도 SELECT 안 하면 읽지 않음
  → WHERE 절에서 TOAST 컬럼 기준 필터링도 마찬가지
  → "필요할 때만" TOAST 접근 (Lazy)

MVCC + TOAST:
  UPDATE events SET payload = '새 값' WHERE id = 1;
  → 기존 TOAST 청크는 삭제 안 됨 (Dead Tuple처럼 유지)
  → 새 payload → 새 TOAST 청크 생성
  → VACUUM이 기존 TOAST 청크 정리
  → TOAST 테이블도 Dead Tuple 발생 → VACUUM 필요
```

---

## 💻 실전 실험

### 실험 1: TOAST 동작 직접 관찰

```sql
-- TOAST가 있는 테이블 생성
CREATE TABLE toast_test (
    id SERIAL PRIMARY KEY,
    small_text TEXT,          -- 짧은 텍스트
    large_text TEXT           -- 큰 텍스트 (TOAST 대상)
);

-- 작은 값 (TOAST 발생 안 함)
INSERT INTO toast_test (small_text, large_text) VALUES
    ('hello', 'hello');

-- 큰 값 (TOAST 발생)
INSERT INTO toast_test (small_text, large_text) VALUES
    ('small', repeat('PostgreSQL TOAST test ', 10000));  -- ~200KB

-- TOAST 테이블 확인
SELECT
    c.relname AS main_table,
    t.relname AS toast_table,
    pg_size_pretty(pg_relation_size(c.oid)) AS main_size,
    pg_size_pretty(pg_relation_size(t.oid)) AS toast_size
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
WHERE c.relname = 'toast_test';

-- TOAST 청크 직접 확인
SELECT
    t.relname AS toast_table,
    count(*) AS chunk_count
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
JOIN pg_toast.pg_toast_0 chunks ON true  -- 실제 TOAST 테이블명 사용
WHERE c.relname = 'toast_test'
GROUP BY t.relname;
```

### 실험 2: TOAST 전략 성능 비교

```sql
CREATE TABLE storage_test_extended (
    id SERIAL PRIMARY KEY,
    content TEXT  -- 기본: EXTENDED (압축 + 분리)
);

CREATE TABLE storage_test_external (
    id SERIAL PRIMARY KEY,
    content TEXT  -- EXTERNAL (압축 없음, 분리)
);

ALTER TABLE storage_test_external ALTER COLUMN content SET STORAGE EXTERNAL;

-- 큰 텍스트 삽입
INSERT INTO storage_test_extended (content)
SELECT repeat('PostgreSQL storage test ', 5000)
FROM generate_series(1, 100);

INSERT INTO storage_test_external (content)
SELECT repeat('PostgreSQL storage test ', 5000)
FROM generate_series(1, 100);

-- 크기 비교 (EXTENDED가 작음 - 압축됨)
SELECT
    'EXTENDED' AS strategy,
    pg_size_pretty(pg_total_relation_size('storage_test_extended')) AS total_size
UNION ALL
SELECT
    'EXTERNAL',
    pg_size_pretty(pg_total_relation_size('storage_test_external'))
;

-- 부분 읽기 성능 비교
-- EXTENDED: 전체 읽고 substring
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, substring(content FROM 1 FOR 100) FROM storage_test_extended;

-- EXTERNAL: 앞 청크만 읽음
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, substring(content FROM 1 FOR 100) FROM storage_test_external;
```

### 실험 3: TOAST 컬럼 미조회 시 성능 차이

```sql
CREATE TABLE perf_test (
    id SERIAL PRIMARY KEY,
    event_type TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    payload JSONB  -- 큰 JSONB (TOAST 대상)
);

INSERT INTO perf_test (event_type, payload)
SELECT
    'click',
    jsonb_build_object(
        'user_id', i,
        'data', repeat('x', 5000),
        'tags', jsonb_build_array('a', 'b', 'c')
    )
FROM generate_series(1, 10000) i;

-- TOAST 컬럼 포함 (느림)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM perf_test WHERE event_type = 'click' LIMIT 100;

-- TOAST 컬럼 제외 (빠름)
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, event_type, created_at FROM perf_test
WHERE event_type = 'click' LIMIT 100;
-- Buffers: shared hit 비교
```

---

## 📊 MySQL과 비교

```
MySQL의 대용량 값 저장:

MySQL InnoDB:
  ROW_FORMAT=DYNAMIC/COMPRESSED (5.6+):
    가변 길이 컬럼이 너무 크면 → 외부 오버플로우 페이지에 저장
    처음 768바이트만 인라인, 나머지 외부 저장 (ROW_FORMAT=COMPACT)
    DYNAMIC: 전체를 외부 저장 (768바이트 인라인 없음)

  TEXT, BLOB, JSON 컬럼:
    작으면 인라인, 크면 오버플로우 페이지
    innodb_page_size=16KB → PostgreSQL 8KB보다 더 많은 인라인 가능

  압축 (ROW_FORMAT=COMPRESSED):
    B-Tree 페이지 단위 압축 (zlib)
    PostgreSQL TOAST는 값 단위 압축

  차이점:
    MySQL: 행 포맷 설정으로 오버플로우 제어
    PostgreSQL: 컬럼별 STORAGE 전략으로 세밀한 제어
    MySQL: 압축은 페이지 단위 (InnoDB 전체)
    PostgreSQL: 압축은 컬럼 값 단위 (TOAST 컬럼만)
```

---

## ⚖️ 트레이드오프

```
TOAST 전략 선택:

EXTENDED (기본):
  장점: 최대한 작게 저장, I/O 절약
  단점: 읽을 때 decompress 비용

EXTERNAL:
  장점: 부분 읽기(substring) 효율적
  단점: 저장 공간 더 사용 (압축 없음)

lz4 압축 (PostgreSQL 14+):
  장점: pglz보다 3~5배 빠른 압축/해제
  단점: 압축률이 pglz보다 약간 낮음

TOAST와 쿼리 설계:
  SELECT *: TOAST 컬럼 모두 읽음 → 주의
  SELECT 필요한 컬럼: TOAST 미접근 → 빠름
  WHERE TOAST_컬럼 = ?: 해당 값을 TOAST에서 읽어 비교 → 부하

  GIN 인덱스의 역할:
  JSONB GIN 인덱스 → TOAST 접근 없이 검색 가능
  → 인덱스 활용 시 TOAST 비용 최소화
```

---

## 📌 핵심 정리

```
TOAST 핵심:

임계값:
  행 크기 > 2KB → TOAST 처리 시작
  목표: 행을 2KB 이하로 유지

전략 (컬럼별 설정):
  PLAIN:    압축·분리 없음 (짧은 고정 타입)
  EXTENDED: 압축 후 분리 (기본, 대부분의 가변 타입)
  EXTERNAL: 분리만 (압축 없음, 부분 읽기 효율)
  MAIN:     압축 우선, 분리 최후

압축 (PostgreSQL 14+):
  pglz: 높은 압축률, 느린 속도
  lz4:  낮은 압축률, 빠른 속도

투명성:
  SQL 레이어에서 자동 처리
  SELECT 안 하면 TOAST 미접근 (lazy)
  UPDATE 시 새 TOAST 청크 생성, VACUUM이 구 청크 정리

성능 영향:
  SELECT *: TOAST 컬럼 모두 decompress → 느림
  필요한 컬럼만: TOAST 미접근 → 빠름
  GIN 인덱스: JSONB/배열 검색 시 TOAST 접근 최소화
```

---

## 🤔 생각해볼 문제

**Q1.** TOAST 테이블이 비대해지는 주요 원인은 무엇이고, 어떻게 모니터링하는가?

<details>
<summary>해설 보기</summary>

TOAST 테이블 비대화의 주요 원인:

1. **UPDATE 빈번한 TOAST 컬럼**: UPDATE 시마다 새 청크 생성, 구 청크가 Dead로 남음. VACUUM이 따라오지 못하면 TOAST 테이블이 커짐.

2. **Autovacuum이 TOAST 테이블을 처리 못함**: 기본 테이블 Autovacuum 설정이 TOAST 테이블에도 적용되지만, TOAST 테이블이 크면 처리 시간이 길어짐.

3. **장기 트랜잭션**: OldestXmin이 낮으면 TOAST 테이블의 Dead 청크도 회수 불가.

모니터링:
```sql
SELECT
    c.relname AS main_table,
    t.relname AS toast_table,
    pg_size_pretty(pg_relation_size(t.oid)) AS toast_size,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
WHERE c.relkind = 'r' AND c.reltoastrelid != 0
  AND pg_relation_size(t.oid) > 100 * 1024 * 1024  -- 100MB 이상
ORDER BY pg_relation_size(t.oid) DESC;
```

해결: 해당 테이블에 Autovacuum 더 자주 실행되도록 파라미터 조정.

</details>

---

**Q2.** EXTERNAL 전략에서 `substring(content FROM 1 FOR 100)`이 EXTENDED보다 빠른 이유는?

<details>
<summary>해설 보기</summary>

EXTERNAL 전략은 압축하지 않고 TOAST 테이블에 청크로 저장합니다. `substring(content FROM 1 FOR 100)`은 앞 100바이트만 필요합니다.

EXTENDED: 값이 압축된 상태로 청크에 저장 → 앞 100바이트를 얻으려면 모든 청크를 읽어서 decompress한 후에야 substring 적용 가능.

EXTERNAL: 압축 없이 청크 저장 → 첫 번째 청크(~2KB)만 읽으면 앞 100바이트 포함 → 나머지 청크 읽기 불필요.

즉, EXTERNAL은 **필요한 청크만 읽는 Partial Detoasting**이 가능합니다. 문서 앞부분만 미리보기로 보여주는 기능처럼, 자주 일부만 읽는 경우에 효과적입니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: 실행 계획 분석 심화](../indexes/08-explain-analyze.md)** | **[홈으로 🏠](../README.md)** | **[다음: JSONB 내부 저장 ➡️](./02-jsonb-storage.md)**

</div>
