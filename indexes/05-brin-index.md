# BRIN(Block Range INdex) — 시계열 데이터를 위한 초경량 인덱스

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- BRIN이 "블록 범위 인덱스"인 이유는 무엇인가?
- 물리적으로 정렬된 데이터에서만 BRIN이 효과적인 이유는?
- 테이블의 0.1% 크기로 어떻게 범위 쿼리를 필터링하는가?
- `pages_per_range`는 무엇이고 어떻게 튜닝하는가?
- B-Tree 대신 BRIN을 선택하는 조건은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

로그 테이블, 이벤트 테이블, 타임시리즈 데이터처럼 **삽입 순서가 곧 시간 순서**인 테이블에서 B-Tree 인덱스는 지나치게 크다. 100GB 테이블에 B-Tree를 만들면 인덱스만 5~10GB다. BRIN은 같은 테이블에 100MB 미만의 인덱스로 날짜 범위 쿼리를 효과적으로 처리한다. "블록 범위마다 최솟값과 최댓값만 저장"하는 단순한 원리로 이를 달성한다. 물리적 정렬이 전제될 때만 효과적이지만, 시계열 데이터에서는 이 전제가 자연스럽게 충족된다.

---

## 😱 흔한 실수 (Before — 정렬 안 된 데이터에 BRIN 사용)

```
실수 1: 랜덤 값 컬럼에 BRIN 사용

  CREATE TABLE users (
      id SERIAL PRIMARY KEY,
      email TEXT,
      score INT  -- 랜덤한 값
  );

  CREATE INDEX ON users USING BRIN(score);

  -- score=500으로 검색
  SELECT * FROM users WHERE score = 500;

  결과: SeqScan (BRIN 무시)
  이유: score 값이 물리적으로 정렬되지 않음
        Block 0: min=1, max=9999 → score=500 가능성 있음
        Block 1: min=2, max=9998 → score=500 가능성 있음
        모든 블록이 해당 범위를 포함 → 필터링 효과 없음 → SeqScan

실수 2: UPDATE 빈번한 테이블에 BRIN (물리 정렬 파괴)

  -- 처음엔 시간 순으로 삽입됨
  INSERT INTO events (created_at, ...) VALUES (NOW(), ...);

  -- 나중에 UPDATE로 created_at 변경
  UPDATE events SET created_at = '2020-01-01' WHERE id = 1000;

  → 물리 페이지의 블록 범위와 created_at 값이 더 이상 일치하지 않음
  → BRIN 효율 저하

올바른 이해:
  BRIN은 "물리적 순서 = 논리적 순서"인 경우에만 효과적
  → INSERT-Only 또는 Append-Only 패턴
  → 자동 증가 ID, 타임스탬프 순 삽입
```

---

## ✨ 올바른 접근 (After — BRIN 적합한 사용 패턴)

```
BRIN이 효과적인 시나리오:

  1. 타임시리즈 로그:
     INSERT INTO access_logs (logged_at, ...)
     VALUES (NOW(), ...);
     → logged_at이 항상 증가 → 물리 정렬 = 시간 정렬
     CREATE INDEX ON access_logs USING BRIN(logged_at);
     → 날짜 범위 쿼리 효율: B-Tree 5GB → BRIN 50MB

  2. 자동 증가 ID:
     id SERIAL (순차 증가)
     → 블록마다 id 범위 단조 증가
     CREATE INDEX ON large_table USING BRIN(id);

  3. 파티셔닝과 조합:
     월별 파티션 + 파티션 내 BRIN
     → 파티션이 범위 필터, BRIN이 파티션 내 블록 필터

  BRIN vs B-Tree 선택:
  ✓ BRIN: 삽입 순 = 정렬 순, 범위 쿼리 위주, 인덱스 크기 중요
  ✓ B-Tree: 랜덤 접근, 단순 등가 비교, 포인트 쿼리, 높은 선택도

  pages_per_range 튜닝:
  128(기본): 128 pages(1MB) 단위 min/max
  더 작게(예: 16): 더 세밀한 필터링 → 인덱스 크기 증가
  더 크게(예: 512): 인덱스 크기 감소 → 필터링 정밀도 감소
```

---

## 🔬 내부 동작 원리

### 1. BRIN 구조 — 블록 범위별 min/max

```
BRIN 개념:

  테이블 파일: 8KB 페이지 1,000개 (8MB 테이블)
  pages_per_range = 128 (기본값)

  Block Range 0 (page 0~127):
    min(created_at) = 2024-01-01 00:00:00
    max(created_at) = 2024-01-05 23:59:59

  Block Range 1 (page 128~255):
    min(created_at) = 2024-01-06 00:00:00
    max(created_at) = 2024-01-10 23:59:59

  Block Range 2 (page 256~383):
    min(created_at) = 2024-01-11 00:00:00
    max(created_at) = 2024-01-15 23:59:59

  ...

  BRIN 인덱스 크기:
    1,000 pages / 128 = 8 범위
    8 범위 × (min + max) = 16 값 → 수 KB
    → 8MB 테이블에 수 KB 인덱스!

쿼리 처리:
  WHERE created_at BETWEEN '2024-01-08' AND '2024-01-12'

  Block Range 0: max=2024-01-05 < 2024-01-08 → 완전 제외!
  Block Range 1: min=2024-01-06, max=2024-01-10
                 범위 겹침 → 이 128페이지는 읽어야 함
  Block Range 2: min=2024-01-11, max=2024-01-15
                 범위 겹침 → 이 128페이지는 읽어야 함
  Block Range 3+: min=2024-01-16 > 2024-01-12 → 완전 제외!

  결과: 1,000페이지 중 256페이지만 읽음 (74% 절약)
```

### 2. pages_per_range 튜닝

```
pages_per_range의 역할:

  작은 값 (예: 16):
  → 범위가 촘촘해짐
  → 불필요한 블록 제외 가능성 높음 → 더 정확한 필터링
  → 인덱스 크기 증가 (더 많은 min/max 저장)

  큰 값 (예: 512):
  → 범위가 넓어짐
  → 여러 블록이 하나의 min/max 범위 공유
  → 인덱스 크기 감소
  → 필터링 정밀도 감소 (더 많은 페이지를 읽어야 함)

예시 계산 (1TB 테이블, 128M pages):

  pages_per_range=128:
    범위 수 = 128M / 128 = 1M 범위
    인덱스 크기 ≈ 1M × 16bytes(2 타임스탬프) = ~16MB

  pages_per_range=16:
    범위 수 = 128M / 16 = 8M 범위
    인덱스 크기 ≈ 8M × 16bytes = ~128MB

  B-Tree 인덱스 (같은 테이블):
    ≈ 50~100GB

  → BRIN이 수백~수천 배 작음

최적 pages_per_range 결정:
  데이터 범위 / pages_per_range = 선택도
  → 쿼리가 전체 데이터의 5% 범위를 조회한다면:
     5% × total_pages / pages_per_range ≈ 읽을 범위 수
  → 범위 수 × pages_per_range ≈ 실제로 읽을 페이지

  실험: 다양한 pages_per_range 값으로 EXPLAIN 비교
```

### 3. BRIN 인덱스 유지 (Summarize)

```
BRIN 인덱스 갱신 방식:

새 데이터 삽입 시:
  새 페이지의 블록 범위가 기존 범위 요약에 포함되면 자동 업데이트
  새 블록 범위가 시작되면 "unsummarized" 표시 → 나중에 요약

자동 요약화:
  autosummarize = on 설정 시 Autovacuum이 미요약 범위 자동 요약
  기본값: off (수동 요약)

수동 요약:
  SELECT brin_summarize_new_values('idx_name'::regclass);
  -- 새로 삽입된 블록의 min/max 계산해 인덱스 업데이트

  SELECT brin_summarize_range('idx_name'::regclass, block_number);
  -- 특정 블록 범위만 요약

  SELECT brin_desummarize_range('idx_name'::regclass, block_number);
  -- 특정 범위 요약 제거 (잘못된 범위 재계산 시)

  -- 전체 재구성
  SELECT brin_summarize_new_values('my_brin_idx'::regclass);

autosummarize 활성화:
  CREATE INDEX ON logs USING BRIN(logged_at)
  WITH (pages_per_range=64, autosummarize=on);

BRIN 인덱스 정보 확인:
  SELECT * FROM brin_page_items(
      get_raw_page('idx_name', 2),
      'idx_name'::regclass
  );
  -- itemoffset, blknum, attnum, allnulls, hasnulls, placeholder, value
```

---

## 💻 실전 실험

### 실험 1: BRIN vs B-Tree 크기와 성능 비교

```sql
-- 시계열 로그 테이블 (타임스탬프 순 삽입)
CREATE TABLE access_logs_btree (
    id BIGSERIAL PRIMARY KEY,
    logged_at TIMESTAMPTZ DEFAULT NOW(),
    user_id INT,
    path TEXT,
    status_code INT
);

CREATE TABLE access_logs_brin (
    id BIGSERIAL PRIMARY KEY,
    logged_at TIMESTAMPTZ DEFAULT NOW(),
    user_id INT,
    path TEXT,
    status_code INT
);

-- 100만 건 삽입 (시간 순)
INSERT INTO access_logs_btree (logged_at, user_id, path, status_code)
SELECT
    NOW() - ((1000000 - i) * INTERVAL '1 second'),
    (random() * 10000)::INT,
    '/api/' || (random() * 100)::INT,
    (ARRAY[200, 200, 200, 404, 500])[ceil(random()*5)::INT]
FROM generate_series(1, 1000000) i;

INSERT INTO access_logs_brin SELECT * FROM access_logs_btree;

-- 인덱스 생성
CREATE INDEX idx_btree ON access_logs_btree(logged_at);
CREATE INDEX idx_brin  ON access_logs_brin USING BRIN(logged_at)
WITH (pages_per_range=128);

-- 크기 비교
SELECT
    'B-Tree' AS type,
    pg_size_pretty(pg_relation_size('idx_btree')) AS index_size
UNION ALL
SELECT
    'BRIN',
    pg_size_pretty(pg_relation_size('idx_brin'))
;

-- 범위 쿼리 성능 비교
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM access_logs_btree
WHERE logged_at BETWEEN NOW()-INTERVAL '1 day' AND NOW();

EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM access_logs_brin
WHERE logged_at BETWEEN NOW()-INTERVAL '1 day' AND NOW();
-- Buffers 읽은 수 비교
```

### 실험 2: 정렬 안 된 데이터에서 BRIN 효과 확인

```sql
-- 랜덤 score 컬럼에 BRIN
CREATE TABLE random_score_test (
    id SERIAL PRIMARY KEY,
    score INT
);

CREATE INDEX ON random_score_test USING BRIN(score);

INSERT INTO random_score_test (score)
SELECT (random() * 10000)::INT
FROM generate_series(1, 100000);

-- BRIN이 실제로 필터링을 못하는지 확인
EXPLAIN SELECT * FROM random_score_test WHERE score = 5000;
-- Bitmap Heap Scan: 거의 모든 블록을 읽음 (필터링 없음)

-- BRIN 인덱스 범위 정보 확인
CREATE EXTENSION pageinspect;
SELECT * FROM brin_page_items(
    get_raw_page('random_score_test_score_idx', 2),
    'random_score_test_score_idx'::regclass
);
-- value: {min_score, max_score}가 거의 동일 → 랜덤이므로
-- 모든 범위의 min≈1, max≈9999 → 필터링 불가
```

### 실험 3: pages_per_range 튜닝

```sql
-- 다양한 pages_per_range로 BRIN 생성
CREATE INDEX brin_16  ON access_logs_brin USING BRIN(logged_at) WITH (pages_per_range=16);
CREATE INDEX brin_128 ON access_logs_brin USING BRIN(logged_at) WITH (pages_per_range=128);
CREATE INDEX brin_512 ON access_logs_brin USING BRIN(logged_at) WITH (pages_per_range=512);

-- 크기 비교
SELECT indexrelname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'access_logs_brin'
ORDER BY pg_relation_size(indexrelid);

-- 같은 쿼리로 Buffers 읽기 비교
SET enable_seqscan = off;
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM access_logs_brin
WHERE logged_at > NOW() - INTERVAL '7 days';
```

---

## 📊 MySQL과 비교

```
MySQL의 BRIN 대응 기능:

MySQL InnoDB:
  BRIN에 해당하는 기능 없음
  시계열 데이터에는 B-Tree 또는 파티셔닝

  파티셔닝 (MySQL):
    PARTITION BY RANGE (YEAR(created_at)) (
        PARTITION p2023 VALUES LESS THAN (2024),
        PARTITION p2024 VALUES LESS THAN (2025)
    )
  → 연도 범위 쿼리: 해당 파티션만 스캔 (BRIN과 유사한 효과)
  → B-Tree 인덱스보다 훨씬 적은 인덱스 크기

PostgreSQL BRIN:
  파티셔닝 없이도 시계열 범위 쿼리 최적화 가능
  파티셔닝과 조합 시 더 강력 (파티션 내 BRIN)

비교:
  소규모 시계열: PostgreSQL BRIN > MySQL B-Tree (크기 측면)
  대규모 시계열: 둘 다 파티셔닝 권장
  PostgreSQL: BRIN + 파티셔닝 조합 가능
  MySQL: 파티셔닝 + 파티션별 B-Tree
```

---

## ⚖️ 트레이드오프

```
BRIN 장단점:

장점:
  ① 극소한 인덱스 크기: B-Tree의 0.1~1%
  ② 삽입 부하 낮음: 블록 범위 min/max만 업데이트
  ③ 시계열 데이터: 물리 정렬이 자연스럽게 유지됨
  ④ 대용량 테이블: B-Tree가 부담스러운 경우

단점:
  ① 물리 정렬 의존: 랜덤 삽입/UPDATE로 효율 저하
  ② 포인트 쿼리 비효율: = 검색은 여전히 많은 블록 스캔
  ③ 높은 선택도 부적합: 1건 조회에 128 페이지 스캔
  ④ UPDATE로 인한 정렬 파괴: BRIN 효율 감소

선택 기준:
  ✓ BRIN이 적합:
    로그/이벤트 테이블 (Append-Only)
    타임시리즈 데이터 (시간 순 삽입)
    범위 쿼리 위주 (WHERE created_at BETWEEN ...)
    인덱스 크기가 제약인 경우

  ✗ BRIN이 부적합:
    랜덤 삽입/UPDATE 많음
    포인트 쿼리 위주 (WHERE id = ?)
    높은 선택도 컬럼 (5% 이하 결과)
```

---

## 📌 핵심 정리

```
BRIN 핵심:

원리:
  블록 범위마다 min/max 값만 저장
  범위 쿼리 시 해당 범위를 포함하지 않는 블록 제외

크기:
  B-Tree의 0.1~1% → 100GB 테이블에 50~100MB

효과 조건:
  물리 저장 순서 = 인덱스 컬럼 순서
  → 시계열(logged_at), 자동 증가 ID에 효과적

pages_per_range:
  기본 128페이지(1MB) 단위 min/max
  낮추면 정밀도 ↑, 크기 ↑
  높이면 정밀도 ↓, 크기 ↓

언제 B-Tree 대신 BRIN:
  Append-Only 패턴 + 범위 쿼리 위주 + 인덱스 크기 중요
  → 대용량 로그, 이벤트, 시계열 테이블
```

---

## 🤔 생각해볼 문제

**Q1.** BRIN 인덱스가 있는 테이블에 과거 날짜로 UPDATE(`SET created_at = '2020-01-01'`)를 대량 실행하면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

BRIN 인덱스의 효율이 크게 저하됩니다. 예를 들어 물리적으로 2024년 데이터가 있는 블록에 `created_at = '2020-01-01'`로 UPDATE하면:

- 해당 블록 범위의 min이 2020-01-01로 갱신됨
- 이후 "2020년 1월 데이터 검색" 쿼리에서 이 블록 범위가 후보로 포함됨
- "2024년 데이터 검색" 쿼리에서도 이 블록이 필요해짐

결과적으로 거의 모든 블록이 여러 날짜 범위를 포함하게 되어 필터링 효과가 사라집니다.

UPDATE가 빈번하게 발생하는 테이블에는 BRIN이 부적합합니다. Append-Only (INSERT만 있는) 패턴 또는 DELETE만 있는 패턴에 사용해야 합니다.

</details>

---

**Q2.** BRIN과 파티셔닝을 함께 사용하면 어떤 이점이 있는가?

<details>
<summary>해설 보기</summary>

파티셔닝과 BRIN의 이중 필터링 효과를 얻습니다:

1. **파티션 프루닝**: 쿼리 조건으로 관련 파티션만 선택 (파티션 레벨 필터)
2. **BRIN 필터**: 선택된 파티션 내에서 관련 블록만 읽음 (블록 레벨 필터)

```sql
-- 월별 파티션 + 파티션 내 BRIN
CREATE TABLE logs (
    id BIGSERIAL,
    logged_at TIMESTAMPTZ NOT NULL,
    data TEXT
) PARTITION BY RANGE (logged_at);

CREATE TABLE logs_2024_01 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE INDEX ON logs_2024_01 USING BRIN(logged_at) WITH (pages_per_range=16);
```

특정 시간대 쿼리: 파티션 프루닝으로 1개 파티션 선택 → 파티션 내 BRIN으로 블록 필터 → 최소 I/O.

</details>

---

<div align="center">

**[⬅️ 이전: GIN 인덱스](./04-gin-index.md)** | **[홈으로 🏠](../README.md)** | **[다음: Bloom 필터 인덱스 ➡️](./06-bloom-filter-index.md)**

</div>
