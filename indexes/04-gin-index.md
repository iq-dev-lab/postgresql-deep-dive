# GIN(Generalized Inverted Index) — JSONB·배열·전문 검색의 역색인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- GIN이 JSONB `@>` 검색에서 B-Tree보다 빠른 이유는?
- GIN Pending List와 Fastupdate 옵션은 무엇이고 왜 필요한가?
- JSONB, 배열, 전문 검색에서 GIN을 각각 어떻게 사용하는가?
- GIN 인덱스의 크기와 쓰기 비용 트레이드오프는?
- GIN과 GiST 중 전문 검색에 어느 것이 더 적합한가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

JSONB 컬럼에 `WHERE data @> '{"status": "active"}'` 같은 containment 쿼리를 B-Tree로 처리하는 것은 불가능하다. GIN(Generalized Inverted Index)은 복합 값(JSONB, 배열, tsvector)의 각 **원소**를 역색인하여, "이 원소를 포함하는 문서는 무엇인가"를 빠르게 찾는다. 전형적인 검색 엔진의 inverted index와 같은 원리다. PostgreSQL JSONB 검색, 배열 포함 검색, 전문 검색(Full-Text Search)의 표준 인덱스가 GIN이다.

---

## 😱 흔한 실수 (Before — JSONB에 인덱스 없이 쿼리)

```
실수 1: JSONB 컬럼에 인덱스 없이 containment 쿼리

  CREATE TABLE events (
      id SERIAL PRIMARY KEY,
      data JSONB
  );
  -- 100만 건

  SELECT * FROM events WHERE data @> '{"user_id": 42}';
  → SeqScan: 100만 개 JSONB 파싱 → 수 초 소요

  GIN 인덱스 추가 후:
  CREATE INDEX ON events USING GIN(data);
  → Index Scan: user_id=42를 포함하는 문서 즉시 조회

실수 2: Fastupdate 미인지로 쿼리 지연

  -- GIN 인덱스 있는 테이블에 초당 1000건 INSERT
  INSERT INTO events (data) VALUES ('{"type": "click", ...}');

  Fastupdate = on (기본값):
  → 변경사항을 Pending List에 일단 모았다가 나중에 적용
  → INSERT는 빠름
  → 특정 순간에 Pending List 병합 발생 → 그 순간 쿼리 지연

  해결: fastupdate = off (쓰기 느려지지만 지연 없음)
       또는 gin_pending_list_limit 조정

실수 3: 전문 검색에 GiST 선택

  CREATE INDEX ON articles USING GIST(to_tsvector('english', content));
  → GiST 전문 검색: 쓰기 빠름, 조회 느림

  올바른 선택:
  CREATE INDEX ON articles USING GIN(to_tsvector('english', content));
  → GIN 전문 검색: 조회 훨씬 빠름 (역색인 최적)
```

---

## ✨ 올바른 접근 (After — GIN 인덱스 적절한 적용)

```
JSONB GIN 인덱스:

  -- 전체 JSONB 키/값 인덱스
  CREATE INDEX ON events USING GIN(data);
  → @>, ?, ?|, ?& 연산자 지원

  -- 특정 경로만 인덱스 (더 작은 인덱스)
  CREATE INDEX ON events USING GIN((data -> 'tags'));
  → data->'tags' 배열 containment 쿼리에만

  -- jsonb_path_ops (더 작고 빠름, @> 전용)
  CREATE INDEX ON events USING GIN(data jsonb_path_ops);
  → @>만 사용하면 jsonb_path_ops가 기본보다 작고 빠름

배열 GIN 인덱스:

  CREATE INDEX ON posts USING GIN(tags);
  SELECT * FROM posts WHERE tags @> ARRAY['postgresql', 'database'];
  SELECT * FROM posts WHERE tags && ARRAY['postgresql', 'nosql'];

전문 검색 GIN:

  -- 컬럼에 tsvector 생성 후 인덱스
  ALTER TABLE articles ADD COLUMN search_vec tsvector
  GENERATED ALWAYS AS (
      to_tsvector('simple', coalesce(title, '') || ' ' || coalesce(content, ''))
  ) STORED;

  CREATE INDEX ON articles USING GIN(search_vec);

  SELECT * FROM articles
  WHERE search_vec @@ to_tsquery('simple', 'postgresql & database');
```

---

## 🔬 내부 동작 원리

### 1. GIN 역색인 구조

```
역색인(Inverted Index) 원리:

일반 인덱스:
  문서 1 → [단어A, 단어B, 단어C]
  문서 2 → [단어B, 단어D]

역색인 (GIN):
  단어A → [문서1]
  단어B → [문서1, 문서2]
  단어C → [문서1]
  단어D → [문서2]

GIN 파일 구조:

  ┌─────────────────────────────────────────┐
  │  B-Tree of Keys (항목 키)                │
  │  key1 → posting list TID 배열           │
  │  key2 → posting list TID 배열           │
  │  ...                                    │
  └─────────────────────────────────────────┘

  각 키(원소)마다 그 원소를 가진 Heap 튜플의 TID 목록:
  "postgresql" → [(page1, item3), (page2, item7), (page5, item1), ...]

JSONB 역색인 예시:
  data = {"user_id": 42, "tags": ["a", "b"], "active": true}

  GIN이 추출하는 키:
    "user_id" = 42
    "tags" ∋ "a"
    "tags" ∋ "b"
    "active" = true

  각 키 → 이 JSONB를 포함하는 Heap TID 목록

쿼리 처리:
  WHERE data @> '{"user_id": 42, "active": true}'

  Step 1: "user_id"=42의 posting list → TID 집합 A
  Step 2: "active"=true의 posting list → TID 집합 B
  Step 3: A ∩ B → 두 조건을 모두 만족하는 TID
  Step 4: 각 TID의 Heap 행 반환

  → 수백만 JSONB에서 두 조건 교집합을 비트 연산으로 처리
  → SeqScan + JSONB 파싱보다 훨씬 빠름
```

### 2. GIN Pending List와 Fastupdate

```
GIN 쓰기 비용 문제:
  GIN 인덱스 업데이트 = 각 원소 키에 TID 추가
  JSONB가 10개 키를 가지면 → INSERT 1건 = GIN 업데이트 10건
  → 쓰기 집중 워크로드에서 심각한 부하

Fastupdate 최적화 (기본 on):
  ┌──────────────────────────────────────────────────────┐
  │  Pending List (pg_gin_pending_list 테이블)            │
  │  INSERT → 바로 GIN에 쓰지 않고 여기 임시 저장          │
  │  [TID1, key_a], [TID1, key_b], [TID2, key_a], ...   │
  └──────────────────────────────────────────────────────┘
         ↓ (자동 병합 조건 달성 시)
  ┌──────────────────────────────────────────────────────┐
  │  GIN B-Tree (실제 역색인)                             │
  │  key_a → [TID1, TID2, ...]                          │
  │  key_b → [TID1, ...]                                │
  └──────────────────────────────────────────────────────┘

Pending List 병합 조건:
  ① gin_pending_list_limit (기본 4MB) 초과
  ② Autovacuum이 테이블 처리할 때
  ③ SELECT 쿼리 실행 시 (병합 후 검색)

병합 시 동작:
  Pending List 전체를 GIN B-Tree에 병합
  → 병합 중 쿼리: Pending List + GIN B-Tree 모두 검색

Fastupdate 단점:
  병합이 발생하는 시점에 쿼리가 느려질 수 있음
  (병합이 완료될 때까지 기다리거나 병합 중 양쪽 검색)

Fastupdate = off:
  즉시 GIN B-Tree에 업데이트
  → 쓰기 느림, 조회 일관된 속도

설정 방법:
  CREATE INDEX ON events USING GIN(data) WITH (fastupdate = off);
  또는
  ALTER INDEX idx_events_data SET (fastupdate = off);

  gin_pending_list_limit 조정 (병합 주기 조정):
  ALTER INDEX idx_events_data SET (gin_pending_list_limit = 32768);  -- 32MB
```

### 3. jsonb_ops vs jsonb_path_ops

```
기본 GIN (jsonb_ops):
  CREATE INDEX ON events USING GIN(data);

  모든 키와 값을 개별적으로 인덱싱:
  {"a": {"b": 1}} →
    "a" (키)
    "b" (키)
    1 (값)
    {"b": 1} (중첩 값)

  지원 연산자:
    @>: containment
    ?:  키 존재 여부
    ?|: 키 집합 중 하나 존재
    ?&: 키 집합 모두 존재

jsonb_path_ops:
  CREATE INDEX ON events USING GIN(data jsonb_path_ops);

  경로+값 쌍을 해시로 인덱싱:
  {"a": {"b": 1}} →
    hash("a", "b", 1)  ← 경로 전체

  지원 연산자: @> 만 지원
  인덱스 크기: jsonb_ops보다 작음 (해시값 저장)
  조회 속도: @>에서 더 빠름

선택 기준:
  @> 만 사용: jsonb_path_ops (더 작고 빠름)
  ?, ?|, ?& 도 사용: jsonb_ops (기본값)

인덱스 크기 예시 (100만 JSONB, 평균 5 키):
  jsonb_ops: ~200MB
  jsonb_path_ops: ~120MB
```

### 4. 전문 검색 GIN

```
tsvector + GIN:

  ts_document = to_tsvector('english', 'PostgreSQL is fast and reliable')
  → 'fast':3 'postgreSql':1 'reliabl':5
  (불용어 제거, 어간 추출, 위치 정보)

  GIN 역색인:
  'fast'     → [doc1, doc5, doc9, ...]
  'postgreSql' → [doc1, doc2, ...]
  'reliabl'  → [doc1, doc3, ...]

쿼리:
  WHERE ts_doc @@ to_tsquery('english', 'postgresql & fast')

  Step 1: 'postgreSql' posting list ∩ 'fast' posting list
  Step 2: 교집합 TID 반환
  → 수백만 문서에서 밀리초 내 결과

Ranking (순위):
  SELECT title, ts_rank(search_vec, query) AS rank
  FROM articles, to_tsquery('postgresql') query
  WHERE search_vec @@ query
  ORDER BY rank DESC;

  ts_rank: 단어 빈도 기반 관련도 점수
  ts_rank_cd: 커버 밀도 기반 (단어 간 거리 고려)
```

---

## 💻 실전 실험

### 실험 1: JSONB GIN 인덱스 효과 측정

```sql
-- JSONB 테이블
CREATE TABLE json_events (
    id SERIAL PRIMARY KEY,
    data JSONB
);

-- 100만 건 삽입
INSERT INTO json_events (data)
SELECT jsonb_build_object(
    'user_id', (random() * 10000)::INT,
    'event_type', (ARRAY['click','view','purchase'])[ceil(random()*3)::INT],
    'amount', (random() * 1000)::NUMERIC(10,2),
    'tags', jsonb_build_array(
        (ARRAY['a','b','c','d'])[ceil(random()*4)::INT],
        (ARRAY['x','y','z'])[ceil(random()*3)::INT]
    )
)
FROM generate_series(1, 1000000);

-- 인덱스 없이 쿼리
EXPLAIN ANALYZE
SELECT count(*) FROM json_events
WHERE data @> '{"event_type": "purchase"}';
-- Seq Scan: 수 초

-- GIN 인덱스 생성
CREATE INDEX ON json_events USING GIN(data);

-- 인덱스 후 쿼리
EXPLAIN ANALYZE
SELECT count(*) FROM json_events
WHERE data @> '{"event_type": "purchase"}';
-- GIN Index Scan: 밀리초

-- jsonb_path_ops 비교
CREATE INDEX ON json_events USING GIN(data jsonb_path_ops);
SELECT pg_size_pretty(pg_relation_size('json_events_data_idx')) AS gin_ops_size;
SELECT pg_size_pretty(pg_relation_size('json_events_data_idx1')) AS gin_path_ops_size;
```

### 실험 2: 전문 검색 GIN

```sql
-- 전문 검색 테이블
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    search_vec TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('simple',
            coalesce(title, '') || ' ' || coalesce(content, ''))
    ) STORED
);

CREATE INDEX ON articles USING GIN(search_vec);

-- 샘플 데이터
INSERT INTO articles (title, content) VALUES
    ('PostgreSQL Performance', 'Learn about indexes and query optimization in PostgreSQL'),
    ('MySQL vs PostgreSQL', 'Comparing two popular open source databases'),
    ('Database Indexing', 'B-Tree and GIN indexes explained'),
    ('JSONB in PostgreSQL', 'Working with JSON data using GIN indexes');

-- 전문 검색 (AND)
SELECT title, ts_rank(search_vec, q) AS rank
FROM articles, to_tsquery('simple', 'postgresql & index') q
WHERE search_vec @@ q
ORDER BY rank DESC;

-- 전문 검색 (OR)
SELECT title
FROM articles
WHERE search_vec @@ to_tsquery('simple', 'mysql | postgresql');

-- EXPLAIN으로 GIN 인덱스 사용 확인
EXPLAIN SELECT title FROM articles
WHERE search_vec @@ to_tsquery('simple', 'postgresql');
```

### 실험 3: Pending List 확인

```sql
-- Pending List 크기 확인
SELECT indexrelname, pending_heap_blks, pending_list_size
FROM pg_stat_user_indexes
WHERE indexrelname LIKE '%gin%';

-- gin_pending_list_limit 확인
SELECT *
FROM pg_indexes
WHERE indexname = 'json_events_data_idx';

-- Pending List 즉시 병합
SELECT gin_clean_pending_list('json_events_data_idx'::regclass);
```

---

## 📊 MySQL과 비교

```
전문 검색 지원 비교:

MySQL:
  FULLTEXT 인덱스 (MyISAM, InnoDB 5.6+)
  MATCH ... AGAINST 쿼리
  기본 어간 추출: 없음 (단어 그대로)
  BOOLEAN MODE: +term -term 검색

PostgreSQL GIN + tsvector:
  to_tsvector, to_tsquery
  어간 추출 (english: 'running' → 'run')
  불용어 제거 (a, the, in...)
  ts_rank, ts_rank_cd: 관련도 순위
  ARRAY 지원, JSONB 지원

JSONB 검색:
  MySQL: JSON_CONTAINS, JSON_SEARCH (인덱스: InnoDB 기능 인덱스)
          Virtual Column + Index로 특정 경로 인덱싱
  PostgreSQL: GIN으로 전체 JSONB 역색인
              containment, key exists 모두 인덱스 활용

배열 검색:
  MySQL: 배열 타입 없음 (JSON 배열로 처리, 인덱스 제한적)
  PostgreSQL: 배열 타입 + GIN = @>, &&, @< 인덱스 지원
```

---

## ⚖️ 트레이드오프

```
GIN 장단점:

장점:
  ① JSONB @>, ?, ?| 검색: 최적
  ② 배열 @>, && 검색: 최적
  ③ 전문 검색 @@: GiST보다 조회 빠름
  ④ 다중 원소 AND/OR 처리: Posting List 교집합

단점:
  ① 쓰기 비용: 원소 수 × 인덱스 업데이트 (Fastupdate로 완화)
  ② 인덱스 크기: 큰 JSONB는 큰 GIN 인덱스
  ③ Pending List 병합: 특정 시점 지연 발생 가능

GIN vs GiST 선택:
  읽기 집중 + 전문 검색/JSONB: GIN
  쓰기 빈번 + 전문 검색: GiST (쓰기 더 빠름)
  공간 데이터: GiST
  범위 타입: GiST
```

---

## 📌 핵심 정리

```
GIN 핵심:

구조:
  원소 키 → Heap TID 포스팅 리스트 (역색인)
  쿼리: 여러 키의 포스팅 리스트 교집합/합집합

적합한 데이터:
  JSONB: @>, ?, ?|, ?& 연산
  배열: @>, &&, <@ 연산
  전문 검색: tsvector @@ tsquery

Fastupdate:
  on(기본): Pending List에 모았다가 병합 → 쓰기 빠름, 병합 시 지연
  off: 즉시 GIN 업데이트 → 쓰기 느리지만 일관된 성능

jsonb_path_ops:
  @>만 사용하면 기본 jsonb_ops보다 작고 빠름

전문 검색:
  tsvector 컬럼 (GENERATED ALWAYS AS STORED 권장)
  GIN 인덱스 → ts_rank로 관련도 정렬
```

---

## 🤔 생각해볼 문제

**Q1.** `CREATE INDEX ON events USING GIN(data)` 생성 후 `WHERE data->>'event_type' = 'purchase'` 쿼리도 인덱스를 사용하는가?

<details>
<summary>해설 보기</summary>

사용하지 않습니다. GIN 인덱스는 `@>`, `?`, `?|`, `?&` 등 GIN 전용 연산자에만 적용됩니다. `->>` 연산자는 JSONB를 TEXT로 추출하는 연산으로, GIN 인덱스가 지원하지 않습니다.

인덱스를 활용하려면:
1. `WHERE data @> '{"event_type": "purchase"}'` 형태로 변경 (GIN 인덱스 사용)
2. 또는 특정 경로에 Expression Index 추가: `CREATE INDEX ON events((data->>'event_type'))`

```sql
-- GIN 인덱스 사용 가능
SELECT * FROM events WHERE data @> '{"event_type": "purchase"}';

-- GIN 인덱스 사용 불가 (SeqScan)
SELECT * FROM events WHERE data->>'event_type' = 'purchase';
```

</details>

---

**Q2.** Fastupdate를 사용 중일 때 `SELECT` 쿼리가 Pending List에 있는 데이터도 검색하는가?

<details>
<summary>해설 보기</summary>

네, 검색합니다. Fastupdate가 활성화된 경우, SELECT 쿼리는 **GIN B-Tree + Pending List 모두**를 검색합니다. 따라서 결과의 정확성은 보장됩니다.

다만 성능 영향이 있습니다:
- Pending List가 클수록 SELECT 시 Pending List 스캔 비용 증가
- 심한 경우 SELECT 쿼리가 Pending List 병합을 트리거하여 해당 SELECT가 병합을 기다림

Pending List 크기를 낮게 유지하거나(`gin_pending_list_limit`), 쿼리 전 명시적으로 병합하면 성능이 안정됩니다:
```sql
SELECT gin_clean_pending_list('idx_name'::regclass);
```

</details>

---

<div align="center">

**[⬅️ 이전: GiST 인덱스](./03-gist-index.md)** | **[홈으로 🏠](../README.md)** | **[다음: BRIN 인덱스 ➡️](./05-brin-index.md)**

</div>
