# 배열(Array) 타입 — PostgreSQL 네이티브 배열의 활용과 한계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PostgreSQL 배열은 내부적으로 어떻게 저장되는가?
- GIN 인덱스로 배열 원소 검색(`@>`, `&&`, `ANY`)을 어떻게 가속하는가?
- `unnest`, `array_agg`는 어떻게 동작하고 언제 유용한가?
- 1NF(정규화)를 따른 별도 테이블 vs 배열 컬럼 — 어느 것을 선택해야 하는가?
- 배열이 클 경우 TOAST와 어떻게 상호작용하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

MySQL에는 없는 PostgreSQL의 강력한 기능 중 하나가 네이티브 배열 타입이다. 태그(tags TEXT[]), 권한(permissions INT[]), 이미지 목록(image_urls TEXT[]) 같은 데이터를 정규화된 별도 테이블 없이 단일 컬럼으로 저장할 수 있다. GIN 인덱스와 조합하면 `WHERE tags @> ARRAY['postgresql']` 같은 검색을 효율적으로 처리한다. 하지만 "언제 배열을, 언제 별도 테이블을 써야 하는가"를 판단하지 못하면 성능 문제가 발생한다.

---

## 😱 흔한 실수 (Before — 배열 타입 오용)

```
실수 1: 인덱스 없이 배열 검색

  CREATE TABLE posts (
      id SERIAL PRIMARY KEY,
      title TEXT,
      tags TEXT[]  -- 인덱스 없음
  );

  SELECT * FROM posts WHERE 'postgresql' = ANY(tags);
  → SeqScan: 매 행마다 배열 스캔 → 10만 건에서 수 초

  GIN 인덱스 추가:
  CREATE INDEX ON posts USING GIN(tags);
  SELECT * FROM posts WHERE tags @> ARRAY['postgresql'];
  → GIN Index Scan: 밀리초

실수 2: 배열을 조인 대신 사용

  -- 게시글-카테고리 관계를 배열로 저장
  posts(id, category_ids INT[])

  SELECT p.* FROM posts p
  WHERE 10 = ANY(p.category_ids);

  vs 정규화:
  post_categories(post_id, category_id) + B-Tree 인덱스

  배열 방식 문제:
  ① category_id 10의 게시글 수 카운트: SeqScan + 배열 스캔
  ② 카테고리 정보(이름 등) 조인: 불가 (인라인 저장)
  ③ FK 제약: 불가
  ④ UPDATE (카테고리 추가): 전체 배열 재작성

실수 3: 매우 큰 배열을 자주 업데이트

  -- 사용자 행동 로그를 배열에 누적
  users(id, action_log TEXT[])
  UPDATE users SET action_log = action_log || '{"action"}'::TEXT[]
  WHERE id = 1;

  → 매 UPDATE마다 전체 배열 TOAST 재생성
  → 배열이 커질수록 UPDATE 비용 선형 증가
  → 올바른 접근: 별도 user_actions 테이블에 INSERT
```

---

## ✨ 올바른 접근 (After — 배열 타입 올바른 활용)

```
배열이 적합한 시나리오:

  1. 태그 시스템:
     tags TEXT[]  → GIN 인덱스
     조회: WHERE tags @> ARRAY['postgresql']
     검색 유연 + 정규화 테이블보다 단순

  2. 권한/역할 목록:
     permissions TEXT[]  → 정적 데이터, 자주 변경 안 됨
     WHERE 'admin' = ANY(permissions)

  3. 고정 크기 옵션:
     favorite_colors TEXT[]  → 항상 짧고, 전체가 함께 사용됨

  배열 사용 결정 기준:
  ✓ 배열: 원소가 단순 스칼라 값, 조인 불필요, 읽기 위주
  ✓ 별도 테이블: FK 관계 필요, 원소에 추가 속성, 빈번한 단독 업데이트

GIN 인덱스 활용:
  CREATE INDEX ON posts USING GIN(tags);
  -- @>: 이 태그들을 모두 포함
  WHERE tags @> ARRAY['postgresql', 'database']
  -- &&: 이 태그들 중 하나라도 포함
  WHERE tags && ARRAY['mysql', 'postgresql']
  -- @<: 이 태그들에 완전히 포함됨
  WHERE tags <@ ARRAY['a', 'b', 'c', 'd']
```

---

## 🔬 내부 동작 원리

### 1. 배열 내부 저장 구조

```
PostgreSQL 배열 저장 형식:

Array Header (varlena):
  ┌─────────────────────────────────────────────────┐
  │  vl_len:     전체 크기                           │
  │  ndim:       차원 수 (1: 1차원, 2: 2차원 등)     │
  │  dataoffset: 데이터 시작 오프셋 (NULL 비트맵 크기) │
  │  elemtype:   원소 타입 OID                        │
  │  dim[]:      각 차원의 크기                       │
  │  lbound[]:   각 차원의 하한 (기본 1)              │
  ├─────────────────────────────────────────────────┤
  │  NULL 비트맵 (NULL 있을 때만):                    │
  │  1비트 per 원소: 1=NOT NULL, 0=NULL              │
  ├─────────────────────────────────────────────────┤
  │  원소 데이터 (순서대로):                           │
  │  고정 크기 타입: 바로 저장                         │
  │  가변 크기 타입: 각 원소 앞에 길이 정보            │
  └─────────────────────────────────────────────────┘

예시: ARRAY['hello', 'world', 'postgresql']

  ndim = 1, dim[0] = 3, lbound[0] = 1
  NULL 비트맵: 없음 (모두 NOT NULL)
  데이터: [5|hello][5|world][10|postgresql]

배열 인덱스 (1-based):
  arr[1] = 'hello', arr[2] = 'world', arr[3] = 'postgresql'
  arr[-1]:  마지막 원소 (PostgreSQL 음수 인덱스 지원)
  arr[1:2]: 슬라이싱 → ARRAY['hello', 'world']

다차원 배열:
  ARRAY[[1,2],[3,4]] → 2차원, dim={2,2}
  접근: arr[1][1] = 1, arr[2][2] = 4

TOAST와 배열:
  배열 전체가 하나의 값으로 TOAST 처리
  배열이 2KB 초과 → TOAST 테이블로 분리
  → 큰 배열을 자주 읽으면 TOAST decompression 비용 발생
```

### 2. 배열 연산자와 함수

```
검색 연산자:
  @>  : 포함 (superset)     ARRAY[1,2,3] @> ARRAY[1,2]  → true
  <@  : 포함됨 (subset)     ARRAY[1,2] <@ ARRAY[1,2,3]  → true
  &&  : 겹침 (overlap)      ARRAY[1,2] && ARRAY[2,3]    → true
  =   : 동일                ARRAY[1,2] = ARRAY[1,2]     → true

원소 확인:
  ANY: 배열 중 하나와 일치  WHERE 1 = ANY(arr)
  ALL: 배열 모두와 일치     WHERE 1 = ALL(arr)

배열 조작 함수:
  array_append(arr, elem):  끝에 원소 추가  → {1,2,3,4}
  array_prepend(elem, arr): 앞에 원소 추가  → {0,1,2,3}
  array_cat(arr1, arr2):    두 배열 연결    → {1,2,3,4,5}
  ||: 연결 연산자            arr || 4       → {1,2,3,4}

집합 함수:
  array_length(arr, dim):   차원별 길이
  array_dims(arr):          차원 문자열
  array_lower(arr, dim):    하한
  array_upper(arr, dim):    상한
  cardinality(arr):         전체 원소 수 (모든 차원)

unnest (배열 → 행):
  SELECT unnest(ARRAY['a', 'b', 'c']);
  → 'a', 'b', 'c' (각각 별도 행)

  SELECT id, unnest(tags) AS tag FROM posts;
  → 각 태그를 별도 행으로 확장

  PostgreSQL 9.4+: WITH ORDINALITY
  SELECT id, tag, ord
  FROM posts, unnest(tags) WITH ORDINALITY AS t(tag, ord);
  → 순서 번호(ord) 포함

array_agg (행 → 배열):
  SELECT user_id, array_agg(tag ORDER BY tag) AS tags
  FROM post_tags
  GROUP BY user_id;
  → 각 user_id의 태그를 배열로 집계
```

### 3. GIN 인덱스와 배열 검색

```
배열 GIN 인덱스 구조:

  tags = ARRAY['postgresql', 'database', 'index']인 행이 있다면:

  GIN 역색인:
    'database'   → [TID1, TID5, TID9, ...]
    'index'      → [TID1, TID3, ...]
    'postgresql' → [TID1, TID2, TID7, ...]

  WHERE tags @> ARRAY['postgresql', 'database']:
    'postgresql' 포스팅 리스트 ∩ 'database' 포스팅 리스트
    → TID1, TID5 (공통)
    → Heap Fetch → 결과 반환

  WHERE tags && ARRAY['mysql', 'postgresql']:
    'mysql' 포스팅 리스트 ∪ 'postgresql' 포스팅 리스트
    → 합집합 TID

GIN과 ANY:
  WHERE 'postgresql' = ANY(tags)  → GIN 인덱스 사용 가능
  WHERE tags @> ARRAY['postgresql']  → 동일, GIN 사용
  두 쿼리는 동일하지만 쿼리 형식이 다름

  ANY와 GIN 제한:
  WHERE expensive_function(tag) = ... FROM unnest(tags) AS tag
  → GIN 미활용 → SeqScan

성능 비교:
  10만 행, 각 행에 태그 5개 가정

  인덱스 없이 WHERE 'tag' = ANY(tags):
  → SeqScan + 배열 스캔: ~100ms

  GIN 인덱스 후 WHERE tags @> ARRAY['tag']:
  → GIN Index Scan: ~1ms

  → 100배 이상 차이
```

### 4. 배열 vs 별도 테이블 — 설계 판단

```
정규화(별도 테이블) vs 배열 컬럼:

시나리오 1: 게시글-태그

  정규화:
  posts(id, title)
  tags(id, name)
  post_tags(post_id, tag_id)  -- 연결 테이블
  → FK 제약 가능
  → 태그에 추가 속성(color, category) 추가 가능
  → 특정 태그의 게시글 수: SELECT count(*) FROM post_tags WHERE tag_id = 1
  → 단점: 조인 필요, 쿼리 복잡

  배열:
  posts(id, title, tags TEXT[])
  → 단순한 쿼리: WHERE tags @> ARRAY['postgresql']
  → 태그 이름이 중복 저장됨 (정규화 깨짐)
  → 태그 이름 변경: 모든 게시글의 배열 UPDATE 필요
  → 단점: FK 없음, 태그 추가 속성 불가

  판단:
  단순 레이블/태그 → 배열 OK
  태그에 속성 필요, FK 참조, 자주 단독 업데이트 → 정규화

시나리오 2: 사용자 권한

  배열:
  users(id, roles TEXT[])
  WHERE 'admin' = ANY(roles)
  → 역할 수가 적고 단순 레이블 → 배열 적합

  정규화:
  user_roles(user_id, role_id)
  → 역할에 상세 권한 속성 필요 → 정규화 필요

결론:
  ✓ 배열: 단순 레이블, FK 불필요, 읽기 위주, 배열 전체를 함께 사용
  ✗ 배열: 관계형 데이터, FK 참조, 빈번한 단독 원소 업데이트, 추가 속성
```

---

## 💻 실전 실험

### 실험 1: 배열 GIN 인덱스 효과

```sql
-- 태그 시스템
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    tags TEXT[]
);

CREATE INDEX ON articles USING GIN(tags);

-- 샘플 데이터 (10만 건)
INSERT INTO articles (title, tags)
SELECT
    'Article ' || i,
    ARRAY[
        (ARRAY['postgresql','mysql','redis','mongodb'])[ceil(random()*4)::INT],
        (ARRAY['performance','indexing','vacuum','mvcc'])[ceil(random()*4)::INT],
        (ARRAY['backend','frontend','devops'])[ceil(random()*3)::INT]
    ]
FROM generate_series(1, 100000) i;

ANALYZE articles;

-- @> (superset): 이 태그들 모두 포함
EXPLAIN ANALYZE
SELECT count(*) FROM articles
WHERE tags @> ARRAY['postgresql', 'performance'];

-- && (overlap): 이 태그들 중 하나라도
EXPLAIN ANALYZE
SELECT count(*) FROM articles
WHERE tags && ARRAY['mysql', 'redis'];

-- ANY: 특정 태그 포함 여부
EXPLAIN ANALYZE
SELECT count(*) FROM articles
WHERE 'postgresql' = ANY(tags);
```

### 실험 2: unnest로 배열 평탄화

```sql
-- 배열 원소별 집계
SELECT tag, count(*) AS article_count
FROM articles, unnest(tags) AS t(tag)
GROUP BY tag
ORDER BY count(*) DESC;

-- 순서 번호 포함
SELECT id, tag, ord
FROM articles, unnest(tags) WITH ORDINALITY AS t(tag, ord)
WHERE id <= 5
ORDER BY id, ord;

-- array_agg: 행을 배열로
SELECT
    tags[1] AS primary_tag,
    array_agg(id ORDER BY id) AS article_ids,
    count(*) AS article_count
FROM articles
GROUP BY tags[1]
ORDER BY article_count DESC;
```

### 실험 3: 배열 업데이트 방법

```sql
-- 원소 추가 (안티패턴: 큰 배열 매번 재작성)
UPDATE articles SET tags = tags || ARRAY['new-tag']
WHERE id = 1;

-- 원소 제거 (array_remove)
UPDATE articles SET tags = array_remove(tags, 'postgresql')
WHERE id = 1;

-- 원소 존재 여부 확인 후 추가 (중복 방지)
UPDATE articles
SET tags = CASE
    WHEN 'new-tag' = ANY(tags) THEN tags
    ELSE tags || ARRAY['new-tag']::TEXT[]
END
WHERE id = 1;

-- array_agg 활용
SELECT
    array_agg(DISTINCT tag ORDER BY tag) AS unique_tags
FROM (
    SELECT unnest(tags) AS tag
    FROM articles
    WHERE id IN (1, 2, 3)
) t;
```

---

## 📊 MySQL과 비교

```
MySQL vs PostgreSQL 배열 지원:

MySQL:
  네이티브 배열 타입: 없음
  대안 1: VARCHAR로 콤마 구분 저장 ('tag1,tag2,tag3')
    → FIND_IN_SET('postgresql', tags) 검색
    → 인덱스 활용 불가, 느림
  대안 2: JSON 배열 타입 (5.7.8+)
    → JSON_CONTAINS(tags, '"postgresql"')
    → Virtual Column + Index로 특정 인덱싱 가능
  대안 3: 정규화된 별도 테이블 (표준 관계형 방식)

PostgreSQL:
  네이티브 배열 타입 지원 (INT[], TEXT[], JSONB[] 등)
  GIN 인덱스로 @>, &&, ANY 등 효율적 검색
  unnest, array_agg 등 풍부한 배열 함수
  다차원 배열 지원

결론:
  MySQL: 배열 기능 제한 → 정규화 테이블 강제
  PostgreSQL: 배열 + GIN = 유연한 레이블/태그 시스템 가능
  단순 태그 시스템: PostgreSQL 배열 방식이 MySQL 별도 테이블보다 쿼리 단순
```

---

## ⚖️ 트레이드오프

```
배열 컬럼의 장단점:

장점:
  ① 단순한 쿼리: WHERE tags @> ARRAY['pg'] (조인 불필요)
  ② 원자적 읽기: 행 하나에서 전체 배열 읽음
  ③ GIN 인덱스: 효율적인 포함/겹침 검색
  ④ 유연한 길이: 원소 수 제한 없음 (TOAST로 대형 배열 처리)

단점:
  ① 정규화 부재: 중복 값 저장 가능 (태그 이름 변경 시 모든 행 업데이트)
  ② FK 제약 불가: 배열 원소에 외래키 제약 없음
  ③ 빈번한 단독 업데이트 비효율: 원소 하나 추가도 전체 배열 재작성
  ④ 복잡한 집계: unnest + GROUP BY 필요
  ⑤ TOAST 비용: 큰 배열 → TOAST 분리 → 읽기 비용

권장 패턴:
  단순 레이블 (태그, 권한 역할): 배열 ✓
  관계형 데이터 (카테고리, 조직): 정규화 ✓
  빈번한 업데이트: 정규화 ✓
  자주 함께 읽는 데이터: 배열 ✓
```

---

## 📌 핵심 정리

```
PostgreSQL 배열 핵심:

저장:
  배열 헤더 + NULL 비트맵 + 원소 데이터
  2KB 초과 → TOAST 분리

연산자:
  @>: 슈퍼셋  ARRAY[1,2,3] @> ARRAY[1,2]  → true
  <@: 서브셋  ARRAY[1,2] <@ ARRAY[1,2,3]  → true
  &&: 겹침    ARRAY[1,2] && ARRAY[2,3]    → true
  ANY/ALL: 원소 비교

GIN 인덱스:
  CREATE INDEX ON table USING GIN(arr_col);
  @>, &&, ANY 검색 가속

핵심 함수:
  unnest(): 배열 → 행 (평탄화)
  array_agg(): 행 → 배열 (집계)
  array_remove(): 특정 원소 제거
  array_append()/prepend(): 원소 추가

설계 기준:
  단순 레이블 + 전체를 함께 사용: 배열
  FK 필요 + 추가 속성 + 빈번한 단독 업데이트: 정규화 테이블
```

---

## 🤔 생각해볼 문제

**Q1.** `WHERE 'postgresql' = ANY(tags)`와 `WHERE tags @> ARRAY['postgresql']`은 동일한 결과를 반환하는가? GIN 인덱스 활용 여부는 같은가?

<details>
<summary>해설 보기</summary>

결과는 동일합니다. 두 쿼리 모두 tags 배열에 'postgresql'이 포함된 행을 반환합니다.

GIN 인덱스 활용에서는 **둘 다 GIN 인덱스를 사용할 수 있습니다**. PostgreSQL 옵티마이저는 `ANY`를 GIN 호환 형태로 변환할 수 있습니다. 다만 쿼리 계획이 미묘하게 다를 수 있으므로, 확실하게 GIN을 활용하려면 `@>` 연산자를 사용하는 것이 권장됩니다.

`EXPLAIN`으로 직접 확인하는 것이 가장 확실합니다:
```sql
EXPLAIN SELECT * FROM articles WHERE 'postgresql' = ANY(tags);
EXPLAIN SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];
```
두 쿼리 모두 "Bitmap Index Scan" 또는 "Bitmap GIN Index Scan"이 나오면 GIN을 활용하는 것입니다.

</details>

---

**Q2.** 배열 컬럼에서 NULL 원소는 어떻게 처리되는가? `ARRAY[1, NULL, 3]`이 가능한가?

<details>
<summary>해설 보기</summary>

네, PostgreSQL 배열은 NULL 원소를 지원합니다:
```sql
SELECT ARRAY[1, NULL, 3];         -- {1,NULL,3}
SELECT ARRAY[1, NULL, 3][2];      -- NULL
SELECT cardinality(ARRAY[1, NULL, 3]);  -- 3 (NULL도 카운트)
```

NULL 원소 처리 주의사항:
- `= ANY(ARRAY[1, NULL, 3])`: `= 1` → true, `= 2` → false, `= NULL` → NULL (세 값 비교)
- `array_remove(arr, NULL)`: NULL 원소 제거
- GIN 인덱스: NULL 원소는 인덱싱되지 않음

NULL 비트맵이 배열 헤더에 포함됩니다. NULL 원소가 없으면 비트맵 없음(최적화). NULL 있으면 원소당 1비트로 NULL 여부 추적.

실용적으로는 배열 내 NULL 원소는 혼란을 야기할 수 있으므로, 가능하면 피하는 것이 좋습니다.

</details>

---

<div align="center">

**[⬅️ 이전: JSONB 내부 저장](./02-jsonb-storage.md)** | **[홈으로 🏠](../README.md)** | **[다음: 전문 검색 ➡️](./04-full-text-search.md)**

</div>
