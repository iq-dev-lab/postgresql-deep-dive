# Bloom 필터 인덱스 — 다중 컬럼 등가 비교의 확률적 필터

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Bloom 필터 인덱스는 어떤 원리로 동작하는가?
- False Positive가 발생하는 이유와 쿼리 결과 정확성에 영향을 주는가?
- 여러 컬럼의 등가 비교에서 B-Tree 복합 인덱스보다 Bloom이 유리한 경우는?
- `CREATE INDEX USING BLOOM`에서 `length`와 `col1`, `col2` 파라미터의 의미는?
- Bloom 필터 인덱스 크기는 B-Tree보다 얼마나 작은가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

테이블에 컬럼이 10개 있고 임의의 2~3개 컬럼 조합으로 등가 비교 쿼리가 발생한다고 가정하자. B-Tree로 모든 조합을 커버하려면 수십 개의 인덱스가 필요하다. Bloom 필터 인덱스는 하나의 인덱스로 여러 컬럼의 임의 조합 등가 비교를 처리한다. 확률적 자료구조라 False Positive(오탐)이 있지만, PostgreSQL이 Heap Recheck로 정확성을 보장하므로 결과는 항상 정확하다. 단, 적합한 상황이 제한적이어서 신중히 선택해야 한다.

---

## 😱 흔한 실수 (Before — Bloom 적합성 오해)

```
실수 1: Bloom 인덱스를 범위/정렬 쿼리에 기대

  CREATE INDEX ON logs USING BLOOM(user_id, event_type, level);

  WHERE user_id = 42 AND event_type = 'click'  → Bloom 인덱스 사용 가능
  WHERE user_id > 100                           → 범위 쿼리, Bloom 사용 불가
  ORDER BY user_id                              → 정렬, Bloom 사용 불가

  Bloom은 오직 등가 비교(=)의 조합에만 적합

실수 2: False Positive를 "결과 오류"로 오해

  "Bloom 필터에서 False Positive가 발생하면 잘못된 행이 반환되지 않는가?"

  올바른 이해:
  False Positive: "이 행이 조건을 만족할 수도 있다"고 잘못 판단
  → 실제로 Heap을 읽어 조건 검사(Recheck) 실행
  → 조건을 만족하지 않으면 제외
  → 최종 결과는 항상 정확

  단, False Positive가 많으면: 불필요한 Heap Fetch 증가 → 성능 저하

실수 3: 선택도 높은 컬럼에 Bloom 기대

  WHERE user_id = 42 (전체의 0.01%)
  → B-Tree: O(log N) + 몇 개 Heap Fetch
  → Bloom: 많은 페이지 스캔 (False Positive 비율에 따라)
  → B-Tree가 훨씬 유리
```

---

## ✨ 올바른 접근 (After — Bloom 적합한 사용)

```
Bloom이 적합한 시나리오:

  여러 컬럼 조합의 등가 비교, 하지만 어떤 조합이 올지 예측 불가:
  -- 10개 컬럼, 任意의 2~4개 조합으로 쿼리
  WHERE col1 = ? AND col3 = ?
  WHERE col2 = ? AND col5 = ? AND col7 = ?
  WHERE col1 = ? AND col4 = ? AND col8 = ?

  B-Tree로 모든 조합: 10C2 + 10C3 + 10C4 = 45 + 120 + 210 = 375개 인덱스 필요
  Bloom: 1개 인덱스로 모든 조합 처리 (단, False Positive 발생)

  적합한 쿼리 특성:
  ✓ 여러 컬럼의 등가 비교 조합
  ✓ 쿼리 패턴이 예측 불가 (임의 조합)
  ✓ 선택도가 낮지 않음 (전체의 1~10% 정도 반환)
  ✓ 쓰기보다 읽기 위주

  부적합:
  ✗ 범위 쿼리 (>, <, BETWEEN)
  ✗ 정렬 필요 (ORDER BY)
  ✗ 높은 선택도 (0.01% 미만) → B-Tree 우월
  ✗ 쿼리 패턴 고정 → 특정 조합 B-Tree 복합 인덱스가 우월
```

---

## 🔬 내부 동작 원리

### 1. Bloom 필터 원리

```
Bloom 필터 자료구조:

  m 비트의 비트 배열 + k개의 해시 함수
  h1, h2, ..., hk

  원소 삽입:
    원소 X의 k개 해시값 계산: h1(X), h2(X), ..., hk(X)
    → 해당 비트 위치를 1로 설정

  원소 존재 확인:
    원소 Y의 k개 해시값 계산
    → 해당 비트가 모두 1이면: "아마 있다" (False Positive 가능)
    → 하나라도 0이면: "확실히 없다" (False Negative 없음)

False Positive 예시:
  X가 비트 [2, 7, 15]를 설정
  Z가 비트 [5, 7, 20]를 설정

  Y를 확인: h1(Y)=2, h2(Y)=7, h3(Y)=20
  → 비트 2는 X가 설정, 비트 7은 X와 Z가 설정, 비트 20은 Z가 설정
  → Y의 모든 비트가 1 → "아마 있다"
  → 실제로 Y는 없음 → False Positive!

  해결: Heap에서 실제 행 확인(Recheck)으로 정확성 보장

PostgreSQL Bloom 인덱스 구조:
  각 Heap 행마다 서명(signature) 생성:
    signature = bitwise OR of hash(col1_val) | hash(col2_val) | ...
  → 서명 크기: length 파라미터 (기본 80비트)

  인덱스 페이지:
    각 Heap TID + 해당 행의 서명(비트맵)

  쿼리 처리:
    쿼리 조건으로 탐색 서명 생성
    → 각 인덱스 페이지 스캔: 서명이 매칭되면 TID 수집
    → 수집된 TID의 Heap 행에서 정확한 조건 확인(Recheck)
    → 조건 불만족이면 제거 (False Positive 처리)
```

### 2. Bloom 인덱스 파라미터

```
CREATE INDEX ON table USING BLOOM(col1, col2, col3)
WITH (length=80, col1=4, col2=4, col3=4);

파라미터 설명:

  length (기본 80):
    각 Heap 행의 서명 비트 수
    클수록: 인덱스 크기 증가, False Positive 감소
    작을수록: 인덱스 크기 감소, False Positive 증가
    범위: 1~4096 (비트)

  col1, col2, ... (기본 2):
    각 컬럼에 사용할 해시 함수 수
    클수록: False Positive 감소, 서명 처리 비용 증가
    작을수록: False Positive 증가, 처리 빠름
    범위: 1~4095

False Positive 확률 (이론):
  FP_rate ≈ (1 - e^(-k*n/m))^k
  k: 해시 함수 수 (col 파라미터)
  n: 원소 수
  m: 서명 비트 수 (length)

  length=80, k=2, n=10 (10개 컬럼 기준):
  FP_rate ≈ (1 - e^(-2*10/80))^2 ≈ 0.037 = 3.7%

  → 100개 후보 중 3~4개가 Heap Recheck 실패
  → 최종 결과에는 포함 안 됨

length 튜닝 가이드:
  낮은 선택도 쿼리 (1% 이하 반환): length=200 이상 (False Positive 최소화)
  중간 선택도 쿼리 (1~10% 반환): length=80~160
  높은 선택도 쿼리 (10% 이상): Bloom이 부적합
```

### 3. B-Tree 복합 인덱스와 Bloom 비교

```
시나리오: 8개 컬럼, 임의 조합 등가 비교

B-Tree 복합 인덱스 (특정 조합):
  (col1, col2, col3) → 이 조합만 최적
  (col1, col4, col5) → 별도 인덱스 필요
  모든 2컬럼 조합: 28개 인덱스 필요 = 8C2
  모든 3컬럼 조합: 56개 인덱스 필요 = 8C3
  → 총 84개 인덱스 (비현실적)

Bloom 단일 인덱스:
  CREATE INDEX ON table USING BLOOM(col1, col2, col3, col4, col5, col6, col7, col8);
  → 모든 임의 등가 조합 처리 가능
  → 단, 각 쿼리에 대해 B-Tree보다 느릴 수 있음

크기 비교 (100만 행, 8컬럼, INT 타입):
  B-Tree (1개): ~35MB
  8개 B-Tree: ~280MB
  Bloom (1개, length=80): ~10MB

  → Bloom 1개가 8개 B-Tree보다 28배 작음
  → 하지만 성능은 특정 조합 B-Tree보다 느림

결론:
  쿼리 패턴 고정: 해당 조합 B-Tree 복합 인덱스
  쿼리 패턴 가변: Bloom 고려 (성능 < 크기 우선시)
```

---

## 💻 실전 실험

### 실험 1: Bloom 인덱스 생성 및 사용

```sql
-- Bloom 인덱스 확장 설치
CREATE EXTENSION bloom;

-- 다중 컬럼 테이블
CREATE TABLE bloom_test (
    id SERIAL PRIMARY KEY,
    col1 INT,
    col2 INT,
    col3 INT,
    col4 INT,
    col5 INT
);

INSERT INTO bloom_test (col1, col2, col3, col4, col5)
SELECT
    (random() * 100)::INT,
    (random() * 100)::INT,
    (random() * 100)::INT,
    (random() * 100)::INT,
    (random() * 100)::INT
FROM generate_series(1, 1000000);

-- Bloom 인덱스 생성
CREATE INDEX bloom_idx ON bloom_test
USING BLOOM(col1, col2, col3, col4, col5)
WITH (length=80, col1=4, col2=4, col3=4, col4=4, col5=4);

-- 인덱스 크기 확인
SELECT pg_size_pretty(pg_relation_size('bloom_idx')) AS bloom_size;

-- 임의 컬럼 조합 쿼리 (Bloom 인덱스 사용)
EXPLAIN ANALYZE
SELECT * FROM bloom_test
WHERE col1 = 42 AND col3 = 17;

EXPLAIN ANALYZE
SELECT * FROM bloom_test
WHERE col2 = 55 AND col4 = 30 AND col5 = 7;
-- Bitmap Heap Scan → Recheck Cond 확인
```

### 실험 2: False Positive 비율 측정

```sql
-- False Positive가 많은 설정 (length 작게)
CREATE INDEX bloom_small ON bloom_test
USING BLOOM(col1, col2, col3)
WITH (length=10, col1=1, col2=1, col3=1);

-- False Positive가 적은 설정 (length 크게)
CREATE INDEX bloom_large ON bloom_test
USING BLOOM(col1, col2, col3)
WITH (length=200, col1=4, col2=4, col3=4);

-- EXPLAIN으로 Rows Removed by Recheck 비교
SET enable_seqscan = off;
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM bloom_test
WHERE col1 = 42 AND col2 = 17 AND col3 = 5;
-- "Rows Removed by Index Recheck": 이 값이 False Positive 수
```

### 실험 3: Bloom vs B-Tree 크기 비교

```sql
-- B-Tree 복합 인덱스들
CREATE INDEX btree_12 ON bloom_test(col1, col2);
CREATE INDEX btree_13 ON bloom_test(col1, col3);
CREATE INDEX btree_23 ON bloom_test(col2, col3);

-- 크기 비교
SELECT indexrelname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'bloom_test'
ORDER BY pg_relation_size(indexrelid);
```

---

## 📊 MySQL과 비교

```
MySQL의 Bloom 필터 대응:

MySQL 8.0+:
  InnoDB Adaptive Hash Index (AHI): 자동으로 자주 조회되는
  B-Tree 항목에 Hash Index 생성 (사용자 제어 불가)

  사용자 정의 Bloom 필터 인덱스: 없음

PostgreSQL Bloom:
  사용자가 명시적으로 생성 가능
  여러 컬럼 임의 조합 등가 비교에 특화
  False Positive → Heap Recheck로 정확성 보장

MySQL에서 다중 컬럼 등가 비교:
  각 조합별 별도 인덱스 필요
  또는 검색 패턴에 따라 가장 선택도 높은 컬럼 단일 인덱스
  → PostgreSQL Bloom의 유연성이 특정 시나리오에서 우월
```

---

## ⚖️ 트레이드오프

```
Bloom 인덱스 장단점:

장점:
  ① 단일 인덱스로 여러 컬럼 임의 조합 처리
  ② B-Tree 대비 매우 작은 크기
  ③ 쓰기 부하: B-Tree보다 가벼움

단점:
  ① False Positive: Heap Recheck 추가 비용
  ② 범위/정렬 불가: 오직 등가 비교(=)만
  ③ 특정 조합 B-Tree보다 느림
  ④ 튜닝 필요: length, col 파라미터 최적화

실용적 평가:
  Bloom이 진짜 필요한 경우는 드물다
  대부분의 경우 특정 쿼리 패턴 파악 후 B-Tree 복합 인덱스가 더 효과적
  "쿼리 패턴이 완전히 가변적이고 인덱스 수를 최소화해야 할 때"만 고려
```

---

## 📌 핵심 정리

```
Bloom 필터 인덱스 핵심:

원리:
  각 행을 비트 서명(signature)으로 변환
  쿼리 조건과 서명 비교 → 가능한 후보 필터
  False Positive: Heap Recheck로 정확성 보장

적합한 경우:
  여러 컬럼의 임의 조합 등가 비교
  쿼리 패턴 예측 불가 → 특정 B-Tree 불가
  인덱스 수/크기 최소화 우선

파라미터:
  length: 서명 비트 수 (기본 80, 클수록 FP 감소/크기 증가)
  col1~N: 컬럼별 해시 함수 수 (기본 2, 클수록 FP 감소)

제한:
  오직 등가(=) 비교만
  범위, 정렬, LIKE, IS NULL 불가

설치:
  CREATE EXTENSION bloom;
  CREATE INDEX ON t USING BLOOM(col1, col2, col3);
```

---

## 🤔 생각해볼 문제

**Q1.** Bloom 인덱스에서 False Positive가 발생하면 최종 쿼리 결과가 틀릴 수 있는가?

<details>
<summary>해설 보기</summary>

아니요, 최종 결과는 항상 정확합니다. False Positive는 "이 행이 조건을 만족할 수 있다"고 **잘못 판단**하는 것이지, 조건을 만족하는 행을 **누락**하는 것이 아닙니다.

PostgreSQL은 Bloom 인덱스로 후보 TID를 수집한 후, 각 Heap 행에서 정확한 조건 검사(Recheck)를 수행합니다. False Positive가 발생한 행은 Recheck에서 탈락합니다.

```
EXPLAIN ANALYZE 출력:
Bitmap Index Scan on bloom_idx
  Index Cond: (col1 = 42 AND col2 = 17)
Bitmap Heap Scan on bloom_test
  Recheck Cond: (col1 = 42 AND col2 = 17)
  Rows Removed by Index Recheck: 5  ← False Positive 5건
```

False Negative(실제 조건 만족 행을 누락)는 Bloom 필터에서 발생하지 않습니다. 비트가 하나라도 0이면 확실히 없으므로, 있는 행을 놓치지 않습니다.

</details>

---

**Q2.** `length` 파라미터를 4096(최대)으로 설정하면 항상 유리한가?

<details>
<summary>해설 보기</summary>

아니요. `length`가 크면 False Positive가 줄어들지만 인덱스 크기가 선형적으로 증가합니다. 극단적으로 크면 Bloom 인덱스가 B-Tree보다 커질 수 있습니다.

또한 각 행의 서명을 처리하는 비용도 증가합니다. 4096비트 서명은 512바이트이므로, 100만 행의 서명만 해도 512MB가 됩니다.

실용적인 접근:
1. 예상 쿼리 선택도(결과 행 비율)를 파악
2. False Positive 허용률 결정 (예: 5% 이하)
3. `length = -m/n × ln(p) / (ln(2))²` 공식으로 최적값 계산
4. 실제 쿼리로 `Rows Removed by Index Recheck` 값 측정 후 튜닝

대부분의 경우 기본값(80) 또는 160 정도가 균형점입니다.

</details>

---

<div align="center">

**[⬅️ 이전: BRIN 인덱스](./05-brin-index.md)** | **[홈으로 🏠](../README.md)** | **[다음: 인덱스 선택 가이드 ➡️](./07-index-selection-guide.md)**

</div>
