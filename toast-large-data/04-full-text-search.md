# 전문 검색(Full Text Search) — tsvector·GIN과 Elasticsearch 비교

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `tsvector`와 `tsquery`는 내부적으로 어떻게 표현되는가?
- `to_tsvector`가 토큰 생성, 불용어 제거, 어간 추출을 어떻게 수행하는가?
- GIN 인덱스가 전문 검색에서 어떻게 역색인으로 동작하는가?
- 한국어 전문 검색(`pg_bigm`)은 어떻게 동작하는가?
- PostgreSQL FTS와 Elasticsearch 중 어떤 경우에 어느 것을 선택해야 하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

"검색 기능이 필요하다"는 요구가 있을 때 곧바로 Elasticsearch를 도입하는 경우가 많다. 하지만 PostgreSQL 자체 전문 검색(FTS)으로도 많은 사용 사례를 커버할 수 있다. FTS 데이터가 이미 PostgreSQL에 있다면, Elasticsearch를 추가하는 것보다 `tsvector` + GIN 인덱스로 시작하고 한계에 도달했을 때 전환하는 것이 운영 복잡도를 크게 낮춘다. 반대로 Elasticsearch가 언제 반드시 필요한지도 이해해야 한다.

---

## 😱 흔한 실수 (Before — FTS 없이 LIKE 검색)

```
실수 1: LIKE 검색으로 전문 검색 구현

  SELECT * FROM articles WHERE content LIKE '%postgresql%';
  → SeqScan + 패턴 매칭 → 100만 건에서 수 초
  → 단어 변형('run', 'running', 'ran') 처리 불가
  → 불용어('the', 'a') 포함 검색 비효율

  올바른 접근:
  tsvector + GIN 인덱스 → 밀리초, 어간 추출 포함

실수 2: tsvector를 저장하지 않고 매번 생성

  -- 인덱스 없이 매 쿼리마다 to_tsvector 호출
  SELECT * FROM articles
  WHERE to_tsvector('english', content) @@ to_tsquery('postgresql');
  → SeqScan + 매 행마다 tsvector 생성 → 매우 느림

  올바른 접근:
  -- GENERATED STORED 컬럼으로 tsvector 미리 계산
  ALTER TABLE articles ADD COLUMN search_vec TSVECTOR
  GENERATED ALWAYS AS (
      to_tsvector('simple', coalesce(title,'') || ' ' || coalesce(content,''))
  ) STORED;
  CREATE INDEX ON articles USING GIN(search_vec);

실수 3: 한국어에 영어 사전 적용

  to_tsvector('english', '안녕하세요 PostgreSQL 데이터베이스')
  → 한국어 토큰화 불가 → 검색 실패

  한국어는 pg_bigm 확장 사용 필요
```

---

## ✨ 올바른 접근 (After — FTS 올바른 구현)

```
영어 FTS (tsvector + GIN):

  -- 1. GENERATED STORED 컬럼 + GIN 인덱스
  ALTER TABLE articles ADD COLUMN search_vec TSVECTOR
  GENERATED ALWAYS AS (
      to_tsvector('english',
          coalesce(title, '') || ' ' || coalesce(content, ''))
  ) STORED;

  CREATE INDEX ON articles USING GIN(search_vec);

  -- 2. 검색 쿼리
  SELECT id, title, ts_rank(search_vec, q) AS rank
  FROM articles, to_tsquery('english', 'postgresql & database') q
  WHERE search_vec @@ q
  ORDER BY rank DESC
  LIMIT 20;

한국어 FTS (pg_bigm):

  -- pg_bigm 설치
  CREATE EXTENSION pg_bigm;

  -- bigm 인덱스
  CREATE INDEX ON articles USING GIN(content gin_bigm_ops);

  -- 검색 (% 연산자 또는 like)
  SELECT * FROM articles
  WHERE content LIKE '%PostgreSQL 데이터베이스%';
  -- GIN bigm 인덱스 사용

  -- 또는 similarity 함수로 관련도 계산
  SELECT *, bigm_similarity(content, '데이터베이스') AS sim
  FROM articles
  WHERE content LIKE '%데이터베이스%'
  ORDER BY sim DESC;
```

---

## 🔬 내부 동작 원리

### 1. tsvector 내부 표현

```
to_tsvector 처리 파이프라인:

입력: "The quick brown foxes are jumping over the lazy dogs"

① 토큰화 (Tokenization):
   ['The', 'quick', 'brown', 'foxes', 'are', 'jumping', 'over', 'the', 'lazy', 'dogs']

② 불용어 제거 (Stop Words):
   영어 불용어: 'the', 'are', 'over'
   → ['quick', 'brown', 'foxes', 'jumping', 'lazy', 'dogs']

③ 어간 추출 (Stemming):
   'foxes' → 'fox'
   'jumping' → 'jump'
   'dogs' → 'dog'
   → ['quick', 'brown', 'fox', 'jump', 'lazy', 'dog']

④ 위치 정보 포함:
   'quick':2 'brown':3 'fox':4 'jump':5 'lazy':9 'dog':10

tsvector 내부 저장:
  ┌────────────────────────────────────────────┐
  │  varlena 헤더                               │
  │  단어 수                                    │
  │  ┌─────────────────────────────────────┐   │
  │  │ WordEntry 배열 (알파벳 정렬):        │   │
  │  │ 'brown':3  'dog':10  'fox':4        │   │
  │  │ 'jump':5   'lazy':9  'quick':2      │   │
  │  └─────────────────────────────────────┘   │
  │  단어 문자열 저장                            │
  └────────────────────────────────────────────┘

SELECT to_tsvector('english', 'The quick brown foxes are jumping over the lazy dogs');
-- 'brown':3 'dog':10 'fox':4 'jump':5 'lazy':9 'quick':2

위치 정보 활용:
  ts_rank_cd(): 단어 간 거리(cover density) 기반 관련도
  위치가 가까울수록 높은 순위
  "postgresql performance" 두 단어가 근접할수록 관련도 높음
```

### 2. tsquery 내부 표현

```
to_tsquery 처리:

  to_tsquery('english', 'postgresql & (database | storage)')
  → 'postgresql' AND ('database' OR 'storage')

  어간 추출 적용:
  to_tsquery('english', 'running')
  → 'run'  (어간 추출됨)

tsquery 연산자:
  &  : AND   to_tsquery('a & b')
  |  : OR    to_tsquery('a | b')
  !  : NOT   to_tsquery('!a')
  <->: PHRASE  to_tsquery('quick <-> brown')  -- 'quick' 바로 다음 'brown'
  <N>: 거리   to_tsquery('a <2> b')          -- a와 b 사이 최대 2단어

plainto_tsquery (단순 입력):
  plainto_tsquery('english', 'postgresql database performance')
  → 'postgresql' & 'databas' & 'perform'
  → 공백을 AND로 처리, 특수 문자 무시

phraseto_tsquery (구문 검색):
  phraseto_tsquery('english', 'quick brown fox')
  → 'quick' <-> 'brown' <-> 'fox'
  → 정확한 구문 순서 일치

websearch_to_tsquery (웹 스타일):
  websearch_to_tsquery('english', 'postgresql -mysql "full text"')
  → 'postgresql' & !'mysql' & 'full' <-> 'text'
  → Google 스타일 검색 문법 지원 (PostgreSQL 11+)
```

### 3. GIN 인덱스와 전문 검색

```
tsvector GIN 인덱스 구조:

  문서들의 tsvector:
  문서1: 'brown':3 'dog':10 'fox':4 'jump':5
  문서2: 'database':2 'postgresql':1 'performance':5
  문서3: 'database':3 'index':4 'postgresql':1

  GIN 역색인:
  'brown'       → [문서1]
  'database'    → [문서2, 문서3]
  'dog'         → [문서1]
  'fox'         → [문서1]
  'index'       → [문서3]
  'jump'        → [문서1]
  'performance' → [문서2]
  'postgresql'  → [문서2, 문서3]

  WHERE search_vec @@ to_tsquery('postgresql & database'):
  ① GIN에서 'postgresql' 포스팅 리스트: [문서2, 문서3]
  ② GIN에서 'database' 포스팅 리스트: [문서2, 문서3]
  ③ 교집합: [문서2, 문서3]
  ④ Heap Fetch → 결과 반환

  위치 정보는 GIN에 저장 안 됨:
  구문 검색(PHRASE)은 GIN 후보 → Recheck 단계에서 위치 확인
```

### 4. 텍스트 검색 설정 (Dictionary)

```
텍스트 검색 설정(Configuration) 선택:

  SELECT * FROM pg_catalog.pg_ts_config;
  -- 'english', 'german', 'french', 'simple', ...

  'english' 설정:
    불용어: 영어 불용어 목록 (the, a, an, ...)
    어간 추출: 영어 Snowball 스테머

  'simple' 설정:
    불용어 없음
    어간 추출 없음 (소문자 변환만)
    → 정확한 단어 일치 (한국어, 고유명사 등에 적합)

  사용자 정의 설정 (커스텀 사전):
  -- 특정 도메인 용어를 불용어에서 제외하거나 어간 추출 규칙 추가

한국어 전문 검색 옵션:

  1. pg_bigm (Bigram 분리):
     2글자씩 분리하여 역색인
     '데이터베이스' → '데이', '이터', '터베', '베이', '이스'
     장점: 별도 형태소 분석기 불필요
     단점: 인덱스 크기 큼, 부분 일치 많음 (False Positive 증가)

  2. pg_morpheme / pg_jieba (형태소 기반):
     언어별 형태소 분석기 활용
     더 정확한 어간 추출
     설치와 설정이 복잡

  3. full_text_search_ko (한국어 형태소):
     오픈소스 PostgreSQL 한국어 형태소 분석기
     프로덕션 사용 시 별도 검토 필요

  실용적 접근:
  한국어 단순 검색 → pg_bigm
  한국어 정확한 검색 → Elasticsearch (형태소 분석기 내장)
```

### 5. PostgreSQL FTS vs Elasticsearch

```
기능 비교:

항목                  | PostgreSQL FTS        | Elasticsearch
─────────────────────┼───────────────────────┼───────────────────────
어간 추출 언어         | 영어, 독일어 등        | 100+ 언어 + 플러그인
한국어 지원            | pg_bigm (제한적)       | nori/seunjeon (우수)
관련도 순위            | ts_rank, ts_rank_cd   | BM25 + 커스텀 스코어링
실시간 인덱싱          | 즉시                  | 준실시간 (1초 기본)
수평 확장             | 제한적 (파티셔닝)       | 자연스러운 샤딩
집계 검색 (Aggregation)| 제한적               | 강력한 Aggregation API
검색 API             | SQL                   | JSON DSL
운영 복잡도           | 낮음 (DB 일원화)       | 높음 (별도 클러스터)
데이터 일관성          | ACID 보장             | eventual consistency
동기화               | 불필요                | 별도 동기화 파이프라인 필요

PostgreSQL FTS가 적합한 경우:
  ✓ 영어/서유럽어 기반 단순 검색
  ✓ 데이터 규모: 수백만 건 이하
  ✓ 운영 복잡도 최소화 우선
  ✓ 검색이 핵심 기능이 아닌 부가 기능
  ✓ 실시간 일관성 중요

Elasticsearch가 필요한 경우:
  ✓ 한국어/일본어 등 형태소 분석 필요
  ✓ 데이터 규모: 수억 건 이상
  ✓ 복잡한 관련도 튜닝 필요
  ✓ Aggregation, Facet, Highlight 기능 필요
  ✓ 검색이 서비스 핵심 기능
  ✓ 자동완성, 오타 교정 (Fuzzy) 필요
```

---

## 💻 실전 실험

### 실험 1: tsvector와 tsquery 기본 실험

```sql
-- tsvector 생성 및 확인
SELECT to_tsvector('english', 'The quick brown foxes are jumping over the lazy dogs');
-- 'brown':3 'dog':10 'fox':4 'jump':5 'lazy':9 'quick':2

-- 다양한 tsquery 형태
SELECT to_tsquery('english', 'running');          -- 'run'
SELECT to_tsquery('english', 'run & jump');        -- 'run' & 'jump'
SELECT to_tsquery('english', 'run | jump');        -- 'run' | 'jump'
SELECT to_tsquery('english', '!stop');             -- !'stop'
SELECT phraseto_tsquery('english', 'quick brown'); -- 'quick' <-> 'brown'
SELECT websearch_to_tsquery('english', 'postgresql -mysql "full text"');

-- @@ 연산자로 매칭 확인
SELECT to_tsvector('english', 'foxes are jumping') @@ to_tsquery('english', 'fox');
-- true (foxes → fox 어간 추출)

SELECT to_tsvector('english', 'foxes') @@ to_tsquery('english', 'running');
-- false
```

### 실험 2: FTS 인덱스 성능 측정

```sql
-- FTS 테이블
CREATE TABLE fts_articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    search_vec TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('english',
            coalesce(title, '') || ' ' || coalesce(content, ''))
    ) STORED
);

CREATE INDEX ON fts_articles USING GIN(search_vec);

-- 샘플 데이터 (1만 건)
INSERT INTO fts_articles (title, content)
SELECT
    'Article about ' || word,
    'This article discusses ' || word || ' in the context of PostgreSQL databases, indexing, and performance optimization techniques.'
FROM (
    SELECT unnest(ARRAY[
        'postgresql', 'database', 'indexing', 'performance',
        'vacuum', 'mvcc', 'transaction', 'replication',
        'backup', 'monitoring'
    ]) AS word
) words,
generate_series(1, 1000) i;

ANALYZE fts_articles;

-- 인덱스 없이 (search_vec 컬럼 미사용, to_tsvector 동적 호출)
EXPLAIN ANALYZE
SELECT id, title
FROM fts_articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'postgresql & performance');

-- 인덱스 활용 (search_vec 컬럼 + GIN 인덱스)
EXPLAIN ANALYZE
SELECT id, title, ts_rank(search_vec, q) AS rank
FROM fts_articles, to_tsquery('english', 'postgresql & performance') q
WHERE search_vec @@ q
ORDER BY rank DESC
LIMIT 10;
```

### 실험 3: ts_rank와 ts_headline

```sql
-- 관련도 순위 + 하이라이트
SELECT
    id,
    title,
    ts_rank(search_vec, q) AS rank,
    ts_rank_cd(search_vec, q) AS cd_rank,
    ts_headline('english', content, q,
        'StartSel=<b>, StopSel=</b>, MaxWords=20, MinWords=10'
    ) AS highlighted
FROM fts_articles, to_tsquery('english', 'postgresql & performance') q
WHERE search_vec @@ q
ORDER BY rank DESC
LIMIT 5;

-- ts_rank vs ts_rank_cd 비교:
-- ts_rank: 단어 빈도 기반
-- ts_rank_cd: cover density (단어 간 거리 반영, 더 정확)
```

---

## 📊 MySQL과 비교

```
MySQL FULLTEXT vs PostgreSQL FTS:

MySQL FULLTEXT INDEX:
  지원 엔진: InnoDB (5.6+), MyISAM
  MATCH ... AGAINST (검색 쿼리, IN BOOLEAN MODE)
  불용어: 자체 목록
  어간 추출: 기본 없음 (플러그인으로 추가)
  최소 단어 길이: ft_min_word_len (기본 4글자)
  N-gram: ft_min_word_len=1 + ngram 파서로 아시아 언어 지원

PostgreSQL FTS:
  지원: 기본 내장 (확장 불필요)
  어간 추출: Snowball 스테머 (영어 등 지원)
  불용어: 사전(configuration)으로 제어
  위치 기반 순위: ts_rank, ts_rank_cd
  구문 검색: PHRASE <->

한국어 지원:
  MySQL: ngram 파서 (ft_min_word_len=2)
  PostgreSQL: pg_bigm (2gram)

비교:
  단순 영어 검색: 유사한 성능
  한국어: MySQL ngram vs PostgreSQL pg_bigm (유사 접근)
  고급 순위: PostgreSQL ts_rank_cd 우수
  대용량/복잡 검색: 둘 다 Elasticsearch 권장
```

---

## ⚖️ 트레이드오프

```
PostgreSQL FTS 장단점:

장점:
  ① 별도 시스템 없이 DB 내에서 전문 검색 가능
  ② ACID 보장 (Elasticsearch는 eventual consistency)
  ③ SQL로 JOIN, 집계와 자연스러운 조합
  ④ 관련도 순위 (ts_rank, ts_rank_cd)
  ⑤ 구문 검색 (PHRASE), 부정 검색

단점:
  ① 한국어/일본어 어간 추출 제한 (pg_bigm은 bigram 기반)
  ② 대용량(수억 건)에서 확장성 제한
  ③ 복잡한 관련도 튜닝 제한 (BM25 없음)
  ④ 자동완성, 오타 교정 기능 약함
  ⑤ Aggregation/Facet API 없음

권장 결정 흐름:
  1단계: 데이터가 이미 PostgreSQL에 있는가? → FTS로 시작
  2단계: 한국어 정확한 검색 필요? → pg_bigm 시도
  3단계: 1억 건 이상 또는 복잡한 관련도? → Elasticsearch 검토
  4단계: 자동완성, 오타 교정, Facet? → Elasticsearch 필요
```

---

## 📌 핵심 정리

```
PostgreSQL FTS 핵심:

tsvector:
  입력 텍스트 → 토큰화 → 불용어 제거 → 어간 추출 → 위치 정보 포함
  GENERATED ALWAYS AS STORED 컬럼으로 미리 계산 권장

tsquery:
  to_tsquery('english', 'run & jump')
  plainto_tsquery: 공백을 AND로 처리
  phraseto_tsquery: 구문 순서 일치
  websearch_to_tsquery: 웹 스타일 (PostgreSQL 11+)

GIN 인덱스:
  tsvector 컬럼에 GIN → @@ 검색 가속
  역색인: 단어 → [문서 TID 목록]

순위:
  ts_rank(): 단어 빈도 기반
  ts_rank_cd(): cover density (단어 근접도 반영)
  ts_headline(): 검색어 하이라이트

한국어:
  pg_bigm: 2그램 분리, 단순 설치
  고급 한국어: Elasticsearch + nori/seunjeon

FTS vs Elasticsearch:
  소규모 + 영어 + DB 일관성 → PostgreSQL FTS
  대규모 + 한국어 + 고급 기능 → Elasticsearch
```

---

## 🤔 생각해볼 문제

**Q1.** `to_tsquery('english', 'running')` 결과가 'run'인데, `to_tsvector('english', 'run fast')` @@ `to_tsquery('english', 'running')`은 true인가?

<details>
<summary>해설 보기</summary>

네, true입니다. 어간 추출은 양방향으로 적용됩니다:
- `to_tsvector('english', 'run fast')` → 'fast':2 'run':1 ('run'은 이미 어간)
- `to_tsquery('english', 'running')` → 'run' ('running' → 'run' 어간 추출)

두 경우 모두 'run'이 되므로 `@@`가 true를 반환합니다.

이것이 FTS의 핵심 장점입니다. 사용자가 'running'으로 검색해도 'run', 'runs', 'ran', 'runner' 등이 포함된 문서를 찾을 수 있습니다. LIKE 검색으로는 불가능한 기능입니다.

</details>

---

**Q2.** 100만 건의 기사에서 GENERATED ALWAYS AS STORED `search_vec` 컬럼을 추가하면 어떤 영향이 있는가?

<details>
<summary>해설 보기</summary>

GENERATED STORED 컬럼 추가 시:
1. **기존 데이터 backfill**: `ALTER TABLE`로 컬럼 추가 시 모든 기존 행에 대해 `to_tsvector()`가 실행됩니다. 100만 건이면 수 분이 소요될 수 있습니다.
2. **테이블 Lock**: `ALTER TABLE ADD COLUMN`은 기본적으로 테이블 전체 스캔이 필요 → 스캔 중 다른 쓰기와의 Lock 충돌 주의.
3. **저장 공간**: 각 행마다 tsvector 크기 증가 (원문의 10~30% 수준).
4. **이후 쓰기**: INSERT/UPDATE 시 자동으로 search_vec 갱신 (추가 CPU 비용).

운영 중 추가 방법:
```sql
-- 1. 일반 컬럼으로 먼저 추가 (Lock 없음)
ALTER TABLE articles ADD COLUMN search_vec TSVECTOR;

-- 2. 배치로 채우기 (Lock 없이)
UPDATE articles SET search_vec = to_tsvector('english', coalesce(title,'') || ' ' || coalesce(content,''))
WHERE id BETWEEN 1 AND 100000;  -- 배치로

-- 3. 트리거로 자동 갱신 설정
CREATE TRIGGER tsvector_update BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION tsvector_update_trigger(search_vec, 'pg_catalog.english', title, content);

-- 4. GIN 인덱스 CONCURRENTLY로 생성
CREATE INDEX CONCURRENTLY ON articles USING GIN(search_vec);
```

</details>

---

<div align="center">

**[⬅️ 이전: 배열 타입](./03-array-type.md)** | **[홈으로 🏠](../README.md)** | **[다음: Large Object vs TOAST ➡️](./05-large-object-vs-toast.md)**

</div>
