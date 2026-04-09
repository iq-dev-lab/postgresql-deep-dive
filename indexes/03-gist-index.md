# GiST(Generalized Search Tree) 인덱스 — 기하·범위·전문 검색의 기반

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- GiST 인덱스는 B-Tree와 어떻게 다른가? "Generalized"의 의미는?
- R-Tree가 GiST 프레임워크로 어떻게 구현되는가?
- PostGIS의 공간 쿼리 최적화에 GiST가 어떻게 기여하는가?
- 범위 타입(tsrange, int4range)에 GiST를 사용하는 이유는?
- GiST와 GIN의 차이는? 전문 검색에 GiST를 쓰는 경우는?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

B-Tree는 1차원 순서가 있는 데이터에 최적화되어 있다. 하지만 2차원 공간 좌표, 겹치는 시간 범위, 전문 검색 어휘는 B-Tree로 효율적으로 인덱싱할 수 없다. GiST는 이런 "다차원 또는 비선형 데이터"를 위한 **확장 가능한 인덱스 프레임워크**다. PostgreSQL의 기하 타입(point, box, circle), PostGIS의 geometry, 범위 타입(tsrange), `tsvector` 전문 검색 모두 GiST가 기반이다.

---

## 😱 흔한 실수 (Before — 공간/범위 쿼리에 B-Tree 시도)

```
실수 1: 위치 기반 서비스에 B-Tree 사용

  -- users 테이블: latitude, longitude 컬럼
  CREATE INDEX ON users(latitude);
  CREATE INDEX ON users(longitude);

  -- 반경 1km 내 사용자 조회
  SELECT * FROM users
  WHERE latitude BETWEEN 37.4 AND 37.6
    AND longitude BETWEEN 127.0 AND 127.2;

  문제:
  B-Tree로 2차원 공간을 인덱싱할 수 없음
  → latitude로 먼저 필터, longitude는 이후 필터
  → 직사각형 범위 쿼리만 가능 (원형 반경 불가)
  → 인덱스 2개를 BitmapAnd로 조합 → 효율 낮음

  올바른 접근:
  Point 타입 + GiST 인덱스 사용 (또는 PostGIS)

실수 2: 예약 시스템에서 겹치는 시간 범위 B-Tree로 처리

  -- 예약: 특정 시간대와 겹치는 예약 조회
  SELECT * FROM reservations
  WHERE start_time < '2024-01-15 18:00'
    AND end_time > '2024-01-15 14:00';

  B-Tree로는 start_time 또는 end_time 하나만 인덱스 활용
  → 나머지 조건은 필터링 → 많은 행 스캔

  올바른 접근:
  tsrange 컬럼 + GiST 인덱스
  WHERE reservation_range && '[2024-01-15 14:00, 2024-01-15 18:00)'::tsrange
  → GiST로 겹치는 범위 효율적 검색
```

---

## ✨ 올바른 접근 (After — GiST 인덱스 적용)

```
공간 데이터 (PostGIS 없이, 기본 기하 타입):

  ALTER TABLE users ADD COLUMN location POINT;
  UPDATE users SET location = POINT(longitude, latitude);
  CREATE INDEX ON users USING GIST(location);

  -- 특정 좌표 기준 반경 내 검색
  SELECT *, location <-> POINT(127.1, 37.5) AS distance
  FROM users
  WHERE location <@ CIRCLE(POINT(127.1, 37.5), 0.01)  -- 약 1km
  ORDER BY distance
  LIMIT 10;

범위 데이터 (예약 시스템):

  ALTER TABLE reservations ADD COLUMN time_range TSTZRANGE;
  UPDATE reservations
  SET time_range = TSTZRANGE(start_time, end_time, '[)');

  CREATE INDEX ON reservations USING GIST(time_range);

  -- 겹치는 예약 조회 (GiST 인덱스 활용)
  SELECT * FROM reservations
  WHERE time_range && '[2024-01-15 14:00, 2024-01-15 18:00)'::TSTZRANGE;

  -- 중복 예약 방지 (EXCLUDE 제약)
  ALTER TABLE reservations
  ADD CONSTRAINT no_overlap
  EXCLUDE USING GIST (room_id WITH =, time_range WITH &&);
```

---

## 🔬 내부 동작 원리

### 1. GiST 프레임워크 구조

```
GiST = Generalized Search Tree

"Generalized"의 의미:
  B-Tree 알고리즘의 핵심 골격을 유지하면서
  사용자가 7가지 함수를 구현하여 어떤 데이터 타입도 인덱싱 가능

필수 구현 함수 (Operator Class):
  consistent(E, q):   엔트리 E가 쿼리 q와 일치 가능한지 (리프에서는 정확 판단)
  union(P):           엔트리 집합 P를 포함하는 최소 키 반환
  compress(v):        값 v를 인덱스 저장 형식으로 변환
  decompress(v):      인덱스 저장 형식을 원래 형식으로 복원
  penalty(E1, E2):    E1에 E2를 삽입할 때의 비용 (트리 균형)
  picksplit(P):       오버플로 시 페이지를 두 그룹으로 분할
  same(E1, E2):       두 엔트리가 같은지 비교

이 함수들을 구현하면 어떤 데이터 타입도 GiST 인덱스 가능:
  기하 타입: box, circle, point, polygon
  범위 타입: int4range, tsrange, numrange
  네트워크: inet, cidr
  전문 검색: tsvector
  PostGIS: geometry, geography
```

### 2. R-Tree가 GiST로 구현되는 방법

```
R-Tree (Rectangle Tree) 원리:
  2차원 공간을 계층적 Bounding Box로 구성

GiST R-Tree 구조:

Level 0 (Root):
  ┌──────────────────────────────────────┐
  │  BBox: 전체 공간 커버                  │
  │  child1 → Left BBox | child2 → Right │
  └──────────────────────────────────────┘

Level 1 (Internal):
  ┌──────────────────┐  ┌──────────────────┐
  │  BBox: 좌측 영역   │  │  BBox: 우측 영역   │
  │  child → ...     │  │  child → ...     │
  └──────────────────┘  └──────────────────┘

Level 2 (Leaf):
  ┌───────────────┐  ┌───────────────┐
  │  Box(0,0,10,10) │  │  Box(50,50,60,60)│
  │  TID: (1, 5)  │  │  TID: (2, 3)  │
  └───────────────┘  └───────────────┘

공간 쿼리 탐색:
  WHERE location @ Box(5,5,15,15)  (5,5 ~ 15,15 영역에 포함되는 것)

  Root: BBox(전체) contains Box(5,5,15,15)? → 가능 → 내려가
  Child1: BBox(0,0,50,50) && Box(5,5,15,15)? → 겹침 → 내려가
  Child2: BBox(60,60,100,100) && Box(5,5,15,15)? → 겹치지 않음 → 가지치기!
  Leaf: Box(0,0,10,10) @ Box(5,5,15,15)? → 부분 → 정확 판단

union 함수:
  여러 기하 객체의 Bounding Box를 하나로 합침
  → 내부 노드는 자식들의 최소 포함 BBox 저장
  → 겹치는 객체가 많으면 BBox 중복 → 쿼리 시 여러 브랜치 탐색

penalty 함수:
  새 객체를 어느 브랜치에 삽입할지 결정
  → 추가 공간이 최소인 브랜치 선택 (면적 증가 최소화)
```

### 3. 범위 타입과 GiST

```
범위 타입 (Range Types):
  int4range: 정수 범위 [1, 10)
  int8range: 큰 정수 범위
  numrange:  숫자 범위
  tsrange:   타임스탬프 범위 (timezone 없음)
  tstzrange: 타임스탬프 범위 (timezone 있음)
  daterange: 날짜 범위

GiST 인덱스로 지원하는 연산자:
  &&:  겹침 (overlap)     [1,5) && [3,8) → true
  @>:  포함 (contains)    [1,10) @> [2,5) → true
  <@:  포함됨 (containedby) [2,5) <@ [1,10) → true
  =:   동일               [1,5) = [1,5) → true
  >>:  오른쪽에            [6,10) >> [1,5) → true
  <<:  왼쪽에              [1,5) << [6,10) → true

예약 시스템 EXCLUDE 제약:
  ALTER TABLE reservations ADD CONSTRAINT no_double_book
  EXCLUDE USING GIST (
      room_id WITH =,       -- 같은 방이고
      time_range WITH &&    -- 시간이 겹치면 안 됨
  );

  → 같은 room_id + 겹치는 time_range 삽입 시 오류
  → GiST 인덱스가 효율적으로 충돌 검사
```

### 4. PostGIS와 GiST

```
PostGIS geometry 타입 + GiST:

  CREATE EXTENSION postgis;
  ALTER TABLE locations ADD COLUMN geom geometry(Point, 4326);
  CREATE INDEX ON locations USING GIST(geom);

  -- 반경 1km 내 조회 (ST_DWithin)
  SELECT name, ST_Distance(geom, ST_MakePoint(127.1, 37.5)::geography) AS dist
  FROM locations
  WHERE ST_DWithin(
      geom::geography,
      ST_MakePoint(127.1, 37.5)::geography,
      1000  -- 미터 단위
  )
  ORDER BY dist
  LIMIT 10;

  GiST R-Tree가 하는 일:
  1. 쿼리 포인트 주변 Bounding Box로 후보 필터
  2. 정확한 거리 계산은 힙에서 수행 (Recheck)

  BRIN vs GiST (PostGIS):
  BRIN: 공간 데이터에 부적합 (공간 정렬이 없음)
  GiST: 공간 데이터 표준 선택
  SP-GiST: 특정 유형(사분 트리)에 더 효율적일 수 있음

  geography vs geometry:
  geometry: 평면 좌표계 (빠름, 큰 거리에서 오차)
  geography: 구면 좌표계 (정확, 더 느림)
  → 서울 같은 도시 수준: geometry로 충분
  → 대륙 간 거리: geography 필요
```

---

## 💻 실전 실험

### 실험 1: 범위 타입 + GiST 인덱스

```sql
-- 회의실 예약 시스템
CREATE TABLE room_reservations (
    id SERIAL PRIMARY KEY,
    room_id INT NOT NULL,
    user_name TEXT,
    time_range TSTZRANGE NOT NULL,
    EXCLUDE USING GIST (room_id WITH =, time_range WITH &&)
);

CREATE INDEX ON room_reservations USING GIST(time_range);
CREATE INDEX ON room_reservations USING GIST(room_id, time_range);

-- 데이터 삽입
INSERT INTO room_reservations (room_id, user_name, time_range) VALUES
    (1, 'Alice', '[2024-01-15 09:00, 2024-01-15 11:00)'),
    (1, 'Bob',   '[2024-01-15 13:00, 2024-01-15 15:00)'),
    (2, 'Carol', '[2024-01-15 09:00, 2024-01-15 17:00)');

-- 중복 예약 시도 (EXCLUDE 제약 동작)
INSERT INTO room_reservations (room_id, user_name, time_range) VALUES
    (1, 'Dave', '[2024-01-15 10:00, 2024-01-15 12:00)');
-- ERROR: conflicting key value violates exclusion constraint

-- 특정 시간대와 겹치는 예약 조회
SELECT room_id, user_name, time_range
FROM room_reservations
WHERE time_range && '[2024-01-15 10:00, 2024-01-15 14:00)'::TSTZRANGE;
```

### 실험 2: 기하 타입 + GiST

```sql
-- 위치 기반 서비스 (기본 기하 타입)
CREATE TABLE points_of_interest (
    id SERIAL PRIMARY KEY,
    name TEXT,
    location POINT  -- (longitude, latitude)
);

CREATE INDEX ON points_of_interest USING GIST(location);

-- 샘플 데이터 (서울 주변 랜덤 포인트)
INSERT INTO points_of_interest (name, location)
SELECT
    'POI_' || i,
    POINT(127.0 + random() * 0.2, 37.4 + random() * 0.2)
FROM generate_series(1, 10000) i;

-- 반경 내 검색 (Box 사용)
EXPLAIN ANALYZE
SELECT name, location
FROM points_of_interest
WHERE location <@ CIRCLE(POINT(127.1, 37.5), 0.05)
ORDER BY location <-> POINT(127.1, 37.5)
LIMIT 10;
-- GiST Index Scan 사용 확인
```

### 실험 3: GiST vs B-Tree 범위 쿼리 비교

```sql
-- 날짜 범위: GiST vs B-Tree
CREATE TABLE events_btree (
    id SERIAL PRIMARY KEY,
    start_at TIMESTAMPTZ,
    end_at TIMESTAMPTZ
);

CREATE TABLE events_gist (
    id SERIAL PRIMARY KEY,
    event_range TSTZRANGE
);

-- 동일 데이터 삽입 (10만 건)
INSERT INTO events_btree (start_at, end_at)
SELECT
    NOW() - (random() * INTERVAL '365 days'),
    NOW() - (random() * INTERVAL '365 days') + INTERVAL '2 hours'
FROM generate_series(1, 100000);

INSERT INTO events_gist (event_range)
SELECT TSTZRANGE(start_at, end_at, '[)')
FROM events_btree;

CREATE INDEX ON events_btree(start_at, end_at);
CREATE INDEX ON events_gist USING GIST(event_range);

-- 겹치는 이벤트 조회
EXPLAIN ANALYZE
SELECT count(*) FROM events_btree
WHERE start_at < '2024-06-01' AND end_at > '2024-05-01';

EXPLAIN ANALYZE
SELECT count(*) FROM events_gist
WHERE event_range && '[2024-05-01, 2024-06-01)'::TSTZRANGE;
```

---

## 📊 MySQL과 비교

```
공간 인덱스 지원 비교:

MySQL InnoDB:
  SPATIAL INDEX: R-Tree 기반 공간 인덱스
  지원 타입: GEOMETRY, POINT, LINESTRING, POLYGON
  지원 함수: ST_Within, ST_Distance, MBRContains
  제한: NOT NULL 컬럼만 SPATIAL INDEX 가능
         FOREIGN KEY와 함께 사용 불가

PostgreSQL GiST:
  지원 타입: 기본 기하(point, box, circle), PostGIS geometry/geography
  지원 연산: @>, <@, &&, <-> (거리 순 정렬), ST_DWithin 등
  EXCLUDE 제약: 겹침 방지 제약에 GiST 활용 가능
  범위 타입: MySQL에 없는 기능 (tsrange, int4range 등)

범위 타입 (Range Types):
  MySQL: 별도 지원 없음
         시작/종료 날짜 컬럼으로 처리
  PostgreSQL: 기본 제공 tsrange, int4range 등
               EXCLUDE USING GIST로 중복 방지

공간 기능 성숙도:
  PostgreSQL + PostGIS: GIS 표준 (ST_Distance, ST_DWithin, 투영 변환 등)
  MySQL SPATIAL: 기본 공간 기능 (복잡한 GIS는 제한적)
```

---

## ⚖️ 트레이드오프

```
GiST 장단점:

장점:
  ① 다차원 데이터 인덱싱 (B-Tree로 불가한 영역)
  ② 범위, 공간, 전문 검색 등 다양한 타입 지원
  ③ 확장 가능: 새로운 타입도 Operator Class로 추가 가능
  ④ EXCLUDE 제약 지원 (예약 시스템 등)

단점:
  ① B-Tree보다 느린 단순 등가 비교
  ② 인덱스 크기: B-Tree보다 클 수 있음
  ③ Lossy 필터링: 내부 노드는 근사값(BBox) → 리프에서 정확 판단
                  → CPU Recheck 비용
  ④ 공간 중복이 많으면(많이 겹치는 BBox): 탐색 브랜치 증가

GiST vs GIN (겹치는 사용 사례: 전문 검색):
  GiST: 전문 검색 가능, 업데이트 빠름, 조회 느림
  GIN:  전문 검색 최적, 업데이트 느림(Pending List), 조회 빠름
  → 쓰기 빈번하면 GiST, 읽기 집중이면 GIN
```

---

## 📌 핵심 정리

```
GiST 핵심:

특징:
  확장 가능한 트리 인덱스 프레임워크
  Operator Class로 어떤 데이터 타입도 인덱싱 가능

적합한 데이터:
  공간 데이터: POINT, BOX, CIRCLE, PostGIS geometry
  범위 데이터: tsrange, int4range, daterange
  전문 검색: tsvector (GIN보다 쓰기 친화적)

핵심 연산:
  &&: 겹침, @>: 포함, <@: 포함됨, <->: 거리

EXCLUDE 제약:
  EXCLUDE USING GIST (room_id WITH =, time_range WITH &&)
  → 같은 방 + 겹치는 시간 중복 예약 방지

PostGIS:
  GiST R-Tree로 공간 인덱싱
  ST_DWithin, ST_Within 등 공간 함수에 자동 활용

GiST vs GIN 선택:
  쓰기 빈번: GiST
  읽기 집중 (전문 검색): GIN
```

---

## 🤔 생각해볼 문제

**Q1.** `EXCLUDE USING GIST`로 중복 예약을 방지할 때, 다른 room_id에 같은 시간대 예약은 허용되는가?

<details>
<summary>해설 보기</summary>

네, 허용됩니다. `EXCLUDE USING GIST (room_id WITH =, time_range WITH &&)`에서 두 조건이 **모두** 만족해야 충돌이 됩니다:
- `room_id WITH =`: 같은 방이고
- `time_range WITH &&`: 시간이 겹치는 경우

room_id가 다르면 `room_id WITH =`가 false이므로 EXCLUDE 조건이 충족되지 않습니다. 서로 다른 방에는 동일 시간대 예약이 허용됩니다.

```sql
-- 이것은 가능 (다른 room_id)
INSERT INTO room_reservations (room_id, user_name, time_range) VALUES
    (1, 'Alice', '[2024-01-15 09:00, 2024-01-15 11:00)'),
    (2, 'Bob',   '[2024-01-15 09:00, 2024-01-15 11:00)');  -- 다른 방 → OK
```

</details>

---

**Q2.** GiST 인덱스의 "Lossy" 특성은 쿼리 결과의 정확성에 영향을 주는가?

<details>
<summary>해설 보기</summary>

결과의 정확성에는 영향을 주지 않지만 **성능**에 영향을 줍니다.

GiST의 내부 노드는 자식들의 Bounding Box만 저장합니다. 쿼리 시 BBox가 겹치는 노드를 탐색하지만, BBox가 겹쳐도 실제 데이터가 조건을 만족하지 않을 수 있습니다(False Positive). 이런 경우 Heap에서 정확한 조건 확인(Recheck)을 수행합니다.

PostgreSQL의 `EXPLAIN ANALYZE`에서 이를 확인할 수 있습니다:
```
Index Cond: (location <@ ...)
Rows Removed by Index Recheck: 15
```
"Index Recheck"으로 제거된 행이 많다면 인덱스의 Lossy 비율이 높은 것입니다. 공간적으로 겹치는 BBox가 많을수록 Recheck 비용이 커집니다.

최종 결과는 항상 정확합니다. Recheck가 정확성을 보장합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Hash 인덱스](./02-hash-index.md)** | **[홈으로 🏠](../README.md)** | **[다음: GIN 인덱스 ➡️](./04-gin-index.md)**

</div>
