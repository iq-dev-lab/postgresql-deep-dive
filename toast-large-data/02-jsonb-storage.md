# JSONB 내부 저장 — 바이너리 형식이 JSON보다 빠른 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JSONB는 JSON과 내부 저장 방식이 어떻게 다른가?
- JSONB의 키 정렬 저장이 O(log N) 키 접근을 가능하게 하는 방법은?
- JSONB 컬럼의 2KB TOAST 임계값은 실제로 어떻게 작동하는가?
- GIN 인덱스 없이 JSONB를 검색하면 왜 느린가?
- `jsonb_path_query`와 JSON Path 표준은 무엇이며 어떻게 활용하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

현대 웹 서비스에서 JSONB 컬럼은 거의 필수다. 스키마 유연성, 이벤트 페이로드, 설정 데이터를 JSONB로 저장하는 패턴이 일반화됐다. 하지만 JSONB가 JSON보다 "빠르다"는 말을 들어도 왜 빠른지 모르면, GIN 인덱스를 언제 써야 하는지, `jsonb_path_ops`와 `jsonb_ops`의 차이가 왜 중요한지, 왜 `data->>'key'`보다 `data @> '{"key": "val"}'`가 인덱스를 더 잘 활용하는지 이해하기 어렵다.

---

## 😱 흔한 실수 (Before — JSONB와 JSON 혼동)

```
실수 1: JSON vs JSONB 잘못 선택

  CREATE TABLE events (
      id SERIAL PRIMARY KEY,
      payload JSON  -- 검색이 필요한데 JSON 선택
  );

  SELECT * FROM events WHERE payload->>'type' = 'click';
  → SeqScan: 매번 JSON 파싱 + 텍스트 비교
  → JSONB였다면: GIN 인덱스로 O(log N) 검색

  JSON은 원본 텍스트 그대로 저장 (공백, 키 순서 보존)
  JSONB는 파싱 후 바이너리 저장 (INSERT 느림, 검색/수정 빠름)

  JSON이 적합한 경우:
  ① 원본 텍스트 그대로 보존해야 할 때 (키 순서, 공백 보존)
  ② 읽기만 하고 검색/수정이 없을 때
  ③ INSERT 성능이 SELECT보다 훨씬 중요할 때

실수 2: GIN 인덱스 없이 JSONB 검색

  CREATE TABLE events (id SERIAL, data JSONB);
  -- 인덱스 없음
  SELECT * FROM events WHERE data @> '{"user_id": 42}';
  → 1억 건에서 SeqScan + JSONB decompression = 수 분

  GIN 인덱스 추가:
  CREATE INDEX ON events USING GIN(data);
  → 즉시 Index Scan → 밀리초

실수 3: ->> 연산자로 인덱스 활용 기대

  CREATE INDEX ON events USING GIN(data);
  SELECT * FROM events WHERE data->>'event_type' = 'click';
  → GIN 인덱스 사용 불가! (->> 연산자는 GIN 미지원)

  올바른 방법:
  SELECT * FROM events WHERE data @> '{"event_type": "click"}';
  → GIN 인덱스 사용
```

---

## ✨ 올바른 접근 (After — JSONB 최적 활용)

```
JSONB 설계 원칙:

  1. 검색 필요 → JSONB + GIN 인덱스
     CREATE INDEX ON events USING GIN(data);           -- 모든 키/값
     -- 또는
     CREATE INDEX ON events USING GIN(data jsonb_path_ops);  -- @>만 사용 시

  2. @> 연산자로 일관된 검색 패턴 사용:
     -- GIN 활용 (빠름)
     WHERE data @> '{"type": "click", "user_id": 42}'

     -- GIN 미활용 (SeqScan)
     WHERE data->>'type' = 'click' AND (data->>'user_id')::INT = 42

  3. 자주 필터링하는 키는 일반 컬럼으로 분리:
     -- 안티패턴: data->>'user_id'로 필터 (GIN 비효율)
     -- 올바른 패턴: user_id 컬럼 별도 + B-Tree 인덱스
     CREATE TABLE events (
         id SERIAL PRIMARY KEY,
         user_id INT,         -- 분리된 컬럼 (B-Tree 인덱스)
         event_type TEXT,     -- 분리된 컬럼
         metadata JSONB       -- 나머지 동적 데이터
     );

  4. JSON Path 활용 (PostgreSQL 12+):
     SELECT jsonb_path_query(data, '$.tags[*] ? (@ starts with "pg")')
     FROM events WHERE data @@ '$.amount > 1000';
```

---

## 🔬 내부 동작 원리

### 1. JSON vs JSONB 내부 표현

```
JSON 저장:
  원본 텍스트 그대로 저장
  {"name": "Alice",   "age": 30}  ← 공백, 키 순서 보존

  읽기 시: 필드 접근마다 전체 파싱 필요
  검색 시: SeqScan + 전체 파싱 → 매우 느림

JSONB 저장:
  INSERT 시 한 번 파싱 → 바이너리 포맷으로 변환하여 저장
  변환 내용:
    ① 키 알파벳 정렬 (이진 탐색 가능)
    ② 중복 키 제거 (마지막 값 유지)
    ③ 공백 제거
    ④ 숫자는 수치형으로 변환 (문자열 아님)

JSONB 바이너리 포맷 (JsonbValue):

  ┌────────────────────────────────────────────────┐
  │  Header                                        │
  │  vl_len: 전체 크기                              │
  │  JB_FLAG_IS_JSONB: 식별 플래그                  │
  ├────────────────────────────────────────────────┤
  │  Container Header                              │
  │  JB_FTYPE_OBJECT or JB_FTYPE_ARRAY             │
  │  nElems: 원소 수                                │
  ├────────────────────────────────────────────────┤
  │  JEntry 배열 (각 키/값의 오프셋 + 타입 정보)     │
  │  [key1_entry][val1_entry][key2_entry][val2_entry]│
  ├────────────────────────────────────────────────┤
  │  실제 데이터 (키 정렬 순서)                       │
  │  "age" | 30 | "name" | "Alice"                 │
  │  (알파벳 순: age < name)                        │
  └────────────────────────────────────────────────┘

키 정렬의 이점:
  이진 탐색 (Binary Search) 가능
  data->'name' 접근: O(log N) where N = 키 수
  JSON: 선형 탐색 O(N)

JSONB 삽입 비용:
  JSON보다 INSERT 느림 (파싱 + 변환 + 정렬)
  SELECT/검색: JSON보다 훨씬 빠름 (이진 탐색 + GIN)
```

### 2. JSONB 연산자 분류

```
JSONB 주요 연산자:

객체 접근 (GIN 인덱스 활용 안 됨):
  ->   : JSON 값 반환      data->'user'  → {"id": 1} (JSONB)
  ->>  : 텍스트 반환       data->>'user' → '{"id": 1}' (TEXT)
  #>   : 경로 접근         data#>'{a,b}' → data["a"]["b"]
  #>>  : 경로 → 텍스트    data#>>'{a,b}' → TEXT

포함/존재 (GIN 인덱스 활용 됨):
  @>   : containment      data @> '{"type": "click"}'
  <@   : contained by     '{"type": "click"}' <@ data
  ?    : 키 존재           data ? 'user_id'
  ?|   : 키 중 하나 존재   data ?| ARRAY['a', 'b']
  ?&   : 키 모두 존재      data ?& ARRAY['a', 'b']

JSON Path (PostgreSQL 12+):
  @@   : JSON Path 조건   data @@ '$.amount > 100'
  @?   : JSON Path 존재   data @? '$.tags[*] ? (@ == "pg")'

연결/수정:
  ||   : 병합             '{"a":1}'::jsonb || '{"b":2}' → {"a":1,"b":2}
  -    : 키 제거          '{"a":1,"b":2}'::jsonb - 'a' → {"b":2}
  #-   : 경로 제거        data #- '{a,b}'

GIN 인덱스 활용 여부 요약:
  @>, <@, ?, ?|, ?& → GIN 인덱스 활용 ✓
  ->, ->>, #>, #>>   → GIN 인덱스 미활용 ✗ (Expression Index 필요)
  @@ , @?            → jsonb_path_ops GIN으로 활용 가능
```

### 3. JSONB 크기와 TOAST

```
JSONB와 TOAST 임계값:

JSONB 저장 크기:
  입력 JSON 크기 ≈ JSONB 크기 (공백 제거 + 인코딩 오버헤드)
  큰 JSON → 큰 JSONB → TOAST 대상

  2KB 임계값:
  JSONB 컬럼이 2KB 초과 → TOAST 처리 시작
  기본 전략: EXTENDED (압축 후 분리)

JSONB TOAST 최적화:
  pglz로 JSONB 압축: 구조화된 JSON은 60~80% 압축률
  50KB JSON → ~15KB TOAST 청크
  → 15KB를 몇 개 청크로 분할

JSONB 업데이트와 TOAST:
  data = data || '{"new_key": "value"}'  (키 추가)
  → 전체 JSONB를 새로 TOAST에 저장
  → 구 TOAST 청크 Dead 처리 → VACUUM 대상
  → 빈번한 JSONB 업데이트 = 빈번한 TOAST 생성

  최적화:
  자주 업데이트되는 필드 → 별도 컬럼으로 분리
  JSONB는 변경이 드문 "설정/메타데이터"에 적합
  자주 변경되는 "카운터, 상태"는 일반 컬럼
```

### 4. JSON Path (PostgreSQL 12+)

```
JSON Path 표준 (SQL/JSON):
  SQL:2016 표준의 JSON 탐색 언어
  XPath의 JSON 버전과 유사

기본 문법:
  $.key:          최상위 키 접근     $.user_id
  $.a.b:          중첩 접근          $.user.name
  $.arr[0]:       배열 인덱스        $.tags[0]
  $.arr[*]:       배열 모든 원소     $.tags[*]
  ?(...):         필터 조건          $.amount ? (@ > 100)

연산자:
  data @@ '$.amount > 1000'         -- BOOLEAN 조건
  data @? '$.tags[*] ? (@ == "pg")' -- 경로 존재 여부

jsonb_path_query 함수:
  SELECT jsonb_path_query(data, '$.tags[*]')
  FROM events;
  → 각 태그를 별도 행으로 반환

  SELECT jsonb_path_query_array(data, '$.tags[*]')
  FROM events;
  → 배열로 반환

  SELECT jsonb_path_query_first(data, '$.tags[0]')
  FROM events;
  → 첫 번째 결과만 반환

JSON Path와 GIN 인덱스:
  jsonb_path_ops GIN 인덱스 + @@ 연산자 조합 가능
  CREATE INDEX ON events USING GIN(data jsonb_path_ops);
  SELECT * FROM events WHERE data @@ '$.amount > 100';
  -- 단, 범위 조건(@@ 내 >, <)은 인덱스 활용 제한적

JSONB 함수 목록 (주요):
  jsonb_each(jsonb): 키-값 쌍을 행으로
  jsonb_object_keys(jsonb): 키 목록
  jsonb_array_elements(jsonb): 배열 원소를 행으로
  jsonb_set(target, path, val): 특정 경로 값 수정
  jsonb_insert(target, path, val): 배열에 원소 삽입
  jsonb_strip_nulls(jsonb): null 값 키 제거
  jsonb_agg(value): 행들을 JSONB 배열로 집계
  jsonb_build_object(k1, v1, k2, v2, ...): JSONB 객체 생성
```

---

## 💻 실전 실험

### 실험 1: JSON vs JSONB 저장 및 성능 비교

```sql
-- JSON 테이블
CREATE TABLE test_json (
    id SERIAL PRIMARY KEY,
    data JSON
);

-- JSONB 테이블
CREATE TABLE test_jsonb (
    id SERIAL PRIMARY KEY,
    data JSONB
);

-- 동일 데이터 삽입 (10만 건)
INSERT INTO test_json (data)
SELECT jsonb_build_object(
    'user_id', i,
    'event_type', (ARRAY['click','view','purchase'])[ceil(random()*3)::INT],
    'amount', (random() * 1000)::NUMERIC(10,2),
    'tags', '["a", "b", "c"]'
)::TEXT  -- JSON은 텍스트로 삽입
FROM generate_series(1, 100000) i;

INSERT INTO test_jsonb (data)
SELECT jsonb_build_object(
    'user_id', i,
    'event_type', (ARRAY['click','view','purchase'])[ceil(random()*3)::INT],
    'amount', (random() * 1000)::NUMERIC(10,2),
    'tags', '["a", "b", "c"]'
)
FROM generate_series(1, 100000) i;

-- 검색 성능 비교 (인덱스 없음)
EXPLAIN ANALYZE
SELECT * FROM test_json WHERE data->>'event_type' = 'click';

EXPLAIN ANALYZE
SELECT * FROM test_jsonb WHERE data->>'event_type' = 'click';
-- JSONB가 이미 파싱됐으므로 빠름

-- GIN 인덱스 후 JSONB 성능
CREATE INDEX ON test_jsonb USING GIN(data);
EXPLAIN ANALYZE
SELECT * FROM test_jsonb WHERE data @> '{"event_type": "click"}';
-- GIN Index Scan: 밀리초
```

### 실험 2: GIN 인덱스 활용 여부 비교

```sql
-- GIN 인덱스가 있는 테이블
CREATE TABLE jsonb_search (id SERIAL PRIMARY KEY, data JSONB);
CREATE INDEX ON jsonb_search USING GIN(data);

INSERT INTO jsonb_search (data)
SELECT jsonb_build_object(
    'user_id', (random()*10000)::INT,
    'type', (ARRAY['click','view'])[ceil(random()*2)::INT]
)
FROM generate_series(1, 500000);

-- GIN 활용 (빠름)
EXPLAIN SELECT * FROM jsonb_search WHERE data @> '{"type": "click"}';
-- Bitmap GIN Index Scan ✓

-- GIN 미활용 (느림)
EXPLAIN SELECT * FROM jsonb_search WHERE data->>'type' = 'click';
-- Seq Scan ✗

-- Expression Index로 해결
CREATE INDEX ON jsonb_search((data->>'type'));
EXPLAIN SELECT * FROM jsonb_search WHERE data->>'type' = 'click';
-- Index Scan ✓
```

### 실험 3: JSON Path 탐색

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    data JSONB
);

INSERT INTO orders (data) VALUES
    ('{"user_id": 42, "items": [{"name": "book", "price": 15000}, {"name": "pen", "price": 2000}], "total": 17000}'),
    ('{"user_id": 43, "items": [{"name": "laptop", "price": 1500000}], "total": 1500000}'),
    ('{"user_id": 44, "items": [{"name": "mouse", "price": 30000}], "total": 30000}');

-- JSON Path로 고액 주문 검색
SELECT id, data
FROM orders
WHERE data @@ '$.total > 100000';

-- 아이템 가격 목록 추출
SELECT id, jsonb_path_query_array(data, '$.items[*].price') AS prices
FROM orders;

-- 비싼 아이템이 있는 주문
SELECT id
FROM orders
WHERE data @? '$.items[*] ? (@.price > 50000)';

-- JSON Path로 복잡한 필터
SELECT
    id,
    jsonb_path_query_first(data, '$.items[0].name') AS first_item
FROM orders
WHERE data @@ '$.items.size() > 1';
```

---

## 📊 MySQL과 비교

```
MySQL JSON vs PostgreSQL JSONB:

MySQL JSON (5.7.8+):
  저장 방식: 파싱 후 바이너리 형식 (PostgreSQL JSONB와 유사)
  검색 인덱스: Virtual Column + B-Tree 인덱스 (특정 경로)
  포함 검색: JSON_CONTAINS() 함수 (GIN과 유사하나 덜 유연)
  JSON Path: JSON_EXTRACT($, '$.key') 또는 -> 연산자

  MySQL JSON_CONTAINS:
  WHERE JSON_CONTAINS(data, '{"type": "click"}')
  → 인덱스: Virtual Column + Index 필요 (자동 아님)
  → GIN 인덱스처럼 단일 CREATE로 전체 인덱싱 불가

PostgreSQL JSONB:
  저장 방식: 파싱 + 바이너리 + 키 정렬 (O(log N) 접근)
  검색 인덱스: GIN 인덱스 하나로 @>, ?, ?|, ?& 모두 지원
  포함 검색: @> 연산자 (간결하고 인덱스 친화적)
  JSON Path: @@ , @? (PostgreSQL 12+, SQL/JSON 표준)

비교 결론:
  MySQL: 특정 경로를 Virtual Column으로 만들어 인덱싱 (세밀)
  PostgreSQL: GIN 인덱스로 전체 JSONB 내부를 역색인 (편리)
  검색 유연성: PostgreSQL GIN > MySQL JSON Virtual Column
  특정 경로 B-Tree: MySQL이 더 간단한 경우 있음
```

---

## ⚖️ 트레이드오프

```
JSONB 사용 결정 기준:

JSONB 적합:
  ① 스키마가 동적으로 변하는 데이터 (이벤트 페이로드, 사용자 설정)
  ② 키/값 검색이 필요한 경우 (GIN 인덱스)
  ③ 구조가 다양한 여러 타입 이벤트를 단일 테이블로 관리

JSONB 부적합:
  ① 항상 동일한 필드를 조회하고 업데이트 → 일반 컬럼이 더 효율적
  ② 자주 특정 값을 범위 검색 → B-Tree 인덱스가 더 효율적
  ③ 조인이 필요한 관계형 데이터 → 정규화된 테이블이 적합

JSONB 성능 주의:
  자주 업데이트되는 JSONB 컬럼:
  → 전체 JSONB를 새로 생성 (부분 업데이트 없음)
  → TOAST 청크 재생성 → VACUUM 부하 증가

  해결책:
  업데이트 빈도 높은 필드 → 별도 컬럼으로 분리
  JSONB는 "쓰기 드물고 읽기 많은" 데이터에 최적
```

---

## 📌 핵심 정리

```
JSONB 핵심:

JSON vs JSONB:
  JSON: 텍스트 그대로 저장, 매번 파싱, 원본 형식 보존
  JSONB: 파싱 후 바이너리, 키 정렬, 빠른 검색/수정

JSONB 바이너리 포맷:
  키 알파벳 정렬 → O(log N) 이진 탐색
  JEntry 배열 → 오프셋 직접 접근

GIN 인덱스:
  @>, ?, ?|, ?& 연산자 지원
  jsonb_path_ops: @>만 (더 작고 빠름)
  jsonb_ops: 기본, ?, ?|, ?& 모두 지원

TOAST와 JSONB:
  2KB 초과 → TOAST 분리 + 압축
  UPDATE 시 전체 JSONB 재생성 → TOAST 청크 교체

JSON Path (PostgreSQL 12+):
  @@ : 조건 필터      data @@ '$.amount > 100'
  @? : 경로 존재      data @? '$.tags[*] ? (@ == "pg")'
  jsonb_path_query(): 경로 탐색 결과 행으로 반환
```

---

## 🤔 생각해볼 문제

**Q1.** `data @> '{"event_type": "click"}'`과 `data->>'event_type' = 'click'`은 결과가 같은데 왜 성능이 다른가?

<details>
<summary>해설 보기</summary>

결과는 같지만 GIN 인덱스 활용 여부가 다릅니다.

`@>`는 GIN 인덱스가 지원하는 **containment** 연산자입니다. GIN 인덱스는 `{"event_type": "click"}`이라는 키-값 쌍을 역색인으로 추적합니다. 쿼리 시 GIN 인덱스에서 직접 해당 항목을 찾아 Heap TID를 반환합니다.

`->>`는 GIN 인덱스가 지원하지 않는 **element extraction** 연산자입니다. GIN 인덱스가 있어도 인덱스를 사용할 수 없어 SeqScan이 발생하며, 모든 행의 JSONB에서 'event_type' 키를 꺼내 텍스트 비교를 수행합니다.

대안으로 Expression Index를 사용할 수 있습니다:
```sql
CREATE INDEX ON events((data->>'event_type'));
-- 이제 data->>'event_type' = 'click' 도 인덱스 활용 가능
```
하지만 이 경우 특정 경로 하나에 대한 B-Tree 인덱스이므로, 다른 경로 검색에는 별도 인덱스가 필요합니다.

</details>

---

**Q2.** JSONB에서 중복 키가 있으면 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

PostgreSQL JSONB는 중복 키를 허용하지 않습니다. 중복 키가 있으면 **마지막 값**만 유지됩니다:

```sql
SELECT '{"a": 1, "a": 2}'::jsonb;
-- 결과: {"a": 2}  (마지막 값만 유지)
```

JSON(텍스트)은 중복 키를 그대로 저장합니다:
```sql
SELECT '{"a": 1, "a": 2}'::json;
-- 결과: {"a": 1, "a": 2}  (원본 텍스트 보존)
```

이것이 JSON과 JSONB의 중요한 차이 중 하나입니다. JSON에서 중복 키를 처리하는 방식은 라이브러리마다 다르지만(일부는 첫 번째, 일부는 마지막 값 사용), JSONB는 일관되게 마지막 값을 사용합니다.

</details>

---

<div align="center">

**[⬅️ 이전: TOAST 완전 분해](./01-toast-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: 배열 타입 ➡️](./03-array-type.md)**

</div>
