# HOT(Heap Only Tuple) Update — 인덱스 갱신 없는 버전 체인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HOT Update가 일반 UPDATE와 어떻게 다른가?
- HOT Update가 가능한 두 가지 조건은 무엇인가?
- `fillfactor`가 HOT Update 발생률에 어떤 영향을 주는가?
- HOT Update는 인덱스를 어떻게 갱신하지 않는가? 내부 체인 구조는?
- `pg_stat_user_tables.n_tup_hot_upd`로 HOT 비율을 어떻게 분석하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL에서 UPDATE는 NEW 버전 삽입 + 인덱스 갱신을 동반한다. 테이블에 인덱스가 10개 있으면 하나의 UPDATE에 최소 10번의 인덱스 페이지 수정이 발생한다. HOT Update는 이 인덱스 갱신 비용을 제거하는 최적화다. 조건만 맞으면 인덱스를 전혀 건드리지 않고 같은 페이지 내 버전 체인만으로 UPDATE를 완료한다. 초당 수천 건의 UPDATE가 있는 테이블에서 HOT Update 비율이 높으면 I/O와 CPU 부하가 크게 줄어든다.

---

## 😱 흔한 실수 (Before — HOT Update 모름)

```
상황: users 테이블에 인덱스 5개 존재
  users(id PK, email UNIQUE, status, last_login, score, profile_data)
  인덱스: id, email, status, last_login, score

  UPDATE users SET last_login = NOW(), score = score + 1 WHERE id = ?;
  → 초당 1,000건

  HOT 조건 미충족 경우:
  last_login에 인덱스 있음 → 인덱스 갱신 필요
  score에 인덱스 있음 → 인덱스 갱신 필요
  → 매 UPDATE마다 5개 인덱스 수정
  → 초당 5,000번 인덱스 페이지 수정

  개선 방법:
  last_login, score 인덱스가 실제로 쿼리에 사용되는가?
  사용 안 된다면 삭제 → HOT Update 가능
  또는 profile_data만 업데이트하도록 설계 변경

잘못된 fillfactor 설정 이해:
  "fillfactor를 낮추면 저장 공간이 낭비되니 100%로 유지하자"
  → 페이지가 가득 차 HOT Update 불가능
  → 모든 UPDATE가 새 페이지에 삽입 → 인덱스 갱신 폭증
```

---

## ✨ 올바른 접근 (After — HOT Update 최적화)

```
HOT Update 최적화 전략:

  1. 인덱스 컬럼 분리:
     자주 업데이트되는 컬럼(last_login, score, status)에 인덱스 지양
     조회에 필수가 아니라면 인덱스 제거

  2. fillfactor 설정:
     UPDATE 빈번한 테이블: fillfactor = 70~80
     → 페이지 20~30% 여유 확보 → 같은 페이지 내 NEW 버전 삽입 가능

  3. HOT 비율 확인:
     SELECT
         relname,
         n_tup_upd AS total_updates,
         n_tup_hot_upd AS hot_updates,
         round(n_tup_hot_upd * 100.0 / nullif(n_tup_upd, 0), 1) AS hot_pct
     FROM pg_stat_user_tables
     WHERE n_tup_upd > 0
     ORDER BY n_tup_upd DESC;

  HOT 비율 해석:
    80% 이상: 매우 좋음 (인덱스 갱신 최소화)
    50~80%: 양호
    50% 미만: fillfactor 조정 또는 인덱스 검토
    0%: 인덱스 컬럼 업데이트 중 (HOT 불가)
```

---

## 🔬 내부 동작 원리

### 1. 일반 UPDATE vs HOT Update

```
테이블 구조:
  users(id INT, email TEXT, status TEXT, last_login TIMESTAMPTZ)
  인덱스: idx_users_id (id), idx_users_status (status)

=== 일반 UPDATE (인덱스 컬럼 변경) ===

UPDATE users SET status = 'ACTIVE' WHERE id = 1;
(status 컬럼에 인덱스 있음)

① Heap 페이지에 NEW 버전 삽입 (같은 또는 다른 페이지)
② OLD 버전: t_xmax 설정
③ idx_users_status 인덱스: 'INACTIVE' 엔트리 → Dead 표시
                              'ACTIVE' 엔트리 → 새로 추가
④ idx_users_id 인덱스: 변경 없음 (id 컬럼 변경 안 했으므로)

인덱스 변경 비용:
  인덱스 페이지 읽기 + 수정 + WAL 기록 (각 인덱스마다)

=== HOT Update (비인덱스 컬럼 변경 + 같은 페이지 여유) ===

UPDATE users SET last_login = NOW() WHERE id = 1;
(last_login 컬럼에 인덱스 없음)

조건 1: 변경된 컬럼이 어떤 인덱스에도 포함되지 않음 ✓
조건 2: NEW 버전을 같은 Heap 페이지에 삽입 가능 (여유 공간) ✓

처리:
① OLD 버전: t_xmax 설정 + HEAP_HOT_UPDATED 플래그
② NEW 버전: 같은 페이지의 빈 공간에 삽입 + HEAP_ONLY_TUPLE 플래그
③ 인덱스: 전혀 변경 없음!

버전 체인:
  idx_users_id → (page=0, item=1) → OLD Tuple (ItemId[1])
                                           t_ctid = (0, 2)  ← NEW 위치
                                         ↓
                                     NEW Tuple (ItemId[2])
                                           t_ctid = (0, 2)  ← 자기 자신 (최신)

HOT 체인 탐색:
  Index Scan: idx_users_id에서 (0, 1) 찾음
  → ItemId[1] → t_ctid = (0, 2) → HOT 체인 따라가기
  → ItemId[2] → t_ctid = (0, 2) (자기 자신) → 현재 버전 반환
  → 인덱스는 OLD 버전을 가리키지만 t_ctid 체인으로 최신 버전 찾음
```

### 2. fillfactor와 HOT Update의 관계

```
fillfactor = 100 (기본값):
  INSERT 시 페이지를 100% 채움
  UPDATE 시 새 버전 삽입 공간 없음
  → NEW 버전을 다른 페이지에 삽입
  → 다른 페이지 = HOT Update 불가! (같은 페이지 조건 미충족)
  → 인덱스 갱신 필요

fillfactor = 70:
  INSERT 시 페이지의 70%만 채움
  30% 여유 공간 확보
  UPDATE 시 NEW 버전을 같은 페이지 여유 공간에 삽입 가능
  → HOT Update 가능!

페이지 상태 비교:

fillfactor=100 (UPDATE 전):
┌───────────────────────────────────────┐
│ [T1][T2][T3][T4][T5][T6][T7][T8][T9] │ ← 빈 공간 없음
└───────────────────────────────────────┘

fillfactor=100 (UPDATE 후):
┌───────────────────────────────────────┐
│ [T1*][T2][T3][T4][T5][T6][T7][T8][T9]│ ← T1이 Dead (*), 새 버전은 다른 페이지
└───────────────────────────────────────┘
→ 인덱스 갱신 필요

fillfactor=70 (UPDATE 전):
┌───────────────────────────────────────┐
│ [T1][T2][T3][T4][T5][T6][ ][ ][ ]    │ ← 30% 여유
└───────────────────────────────────────┘

fillfactor=70 (UPDATE 후):
┌───────────────────────────────────────┐
│ [T1*][T2][T3][T4][T5][T6][T1'][  ][  ]│ ← T1 Dead, T1' 새 버전이 같은 페이지에
└───────────────────────────────────────┘
→ HOT Update! 인덱스 갱신 없음

fillfactor 트레이드오프:
  낮은 fillfactor: HOT Update ↑, 저장 공간 효율 ↓ (더 많은 페이지)
  높은 fillfactor: HOT Update ↓, 저장 공간 효율 ↑ (더 적은 페이지)

권장:
  자주 UPDATE되는 컬럼이 있는 테이블: fillfactor = 70~80
  Insert-Only 또는 UPDATE 거의 없는 테이블: fillfactor = 100 (기본값)
```

### 3. HOT 체인과 VACUUM의 정리

```
VACUUM과 HOT 체인:

HOT Update가 반복되면 체인이 길어짐:
  ItemId[1] → OLD_v1 (Dead, t_ctid→ItemId[2])
  ItemId[2] → OLD_v2 (Dead, t_ctid→ItemId[3])
  ItemId[3] → LIVE_v3 (Alive, t_ctid→자기자신)

VACUUM 처리:
  Dead 튜플 (v1, v2) 제거
  ItemId[1] → LP_REDIRECT → ItemId[3] (바로 최신으로)
  ItemId[2] 제거 (LP_UNUSED)

VACUUM 후:
  ItemId[1] (LP_REDIRECT → [3])
  ItemId[3] → LIVE_v3

인덱스: ItemId[1]을 여전히 가리킴
  → LP_REDIRECT 덕분에 [1]→[3]으로 바로 이동
  → 인덱스 갱신 없이 최신 버전 접근 가능

Prune (Pruning):
  VACUUM 없이도 페이지를 읽을 때 HOT 체인 정리 발생
  → heap_page_prune_opt(): 읽기 중 Dead 제거 가능한 경우 제거
  → "Lazy VACUUM" 역할
  → pg_stat_user_tables.n_tup_hot_upd에 반영
```

### 4. HOT Update 불가 조건

```
HOT Update가 불가능한 경우:

조건 1 위반: 변경된 컬럼이 인덱스에 포함
  UPDATE users SET status = 'ACTIVE' WHERE id = 1;
  status 컬럼에 인덱스 있음 → HOT 불가
  → 이유: 인덱스가 status 값으로 정렬되어 있으므로
          값이 바뀌면 인덱스 내 위치도 바뀌어야 함

조건 2 위반: 같은 페이지에 공간 없음
  fillfactor=100이거나 페이지가 꽉 찬 경우
  → NEW 버전을 같은 페이지에 삽입 불가
  → HOT 체인이 페이지를 넘어갈 수 없음

Expression Index (함수 기반 인덱스):
  CREATE INDEX ON users (lower(email));
  UPDATE users SET email = 'NEW@EXAMPLE.COM' WHERE id = 1;
  → lower(email)이 변경됨 → 인덱스에 영향 → HOT 불가

Partial Index:
  CREATE INDEX ON users (id) WHERE status = 'ACTIVE';
  UPDATE users SET status = 'INACTIVE' WHERE id = 1;
  → Partial Index 조건(status)이 변경 → HOT 불가

확인 방법:
  EXPLAIN (VERBOSE) UPDATE users SET last_login = NOW() WHERE id = 1;
  -- 출력에 "HOT update" 여부 포함 없음
  -- 실제 HOT 여부는 실행 후 n_tup_hot_upd로 확인
```

---

## 💻 실전 실험

### 실험 1: HOT Update 발생 확인

```sql
-- 실험 테이블 (fillfactor=70)
CREATE TABLE hot_test (
    id SERIAL PRIMARY KEY,
    email TEXT UNIQUE,
    status TEXT,
    last_login TIMESTAMPTZ,
    score INT
) WITH (fillfactor = 70);

-- 인덱스: id(PK), email(UNIQUE), status
CREATE INDEX ON hot_test(status);

-- 초기 데이터
INSERT INTO hot_test (email, status, last_login, score)
SELECT
    'user' || i || '@example.com',
    'ACTIVE',
    NOW(),
    0
FROM generate_series(1, 10000) i;

-- 통계 초기화
SELECT pg_stat_reset();

-- HOT 가능한 UPDATE (비인덱스 컬럼 + 같은 페이지 여유)
UPDATE hot_test SET last_login = NOW(), score = score + 1 WHERE id = 1;
UPDATE hot_test SET last_login = NOW(), score = score + 1 WHERE id = 2;
UPDATE hot_test SET last_login = NOW(), score = score + 1 WHERE id = 3;
-- 1,000건 연속
UPDATE hot_test SET last_login = NOW(), score = score + 1;

-- HOT 비율 확인
SELECT
    relname,
    n_tup_upd AS total_upd,
    n_tup_hot_upd AS hot_upd,
    round(n_tup_hot_upd * 100.0 / nullif(n_tup_upd, 0), 1) AS hot_pct
FROM pg_stat_user_tables
WHERE relname = 'hot_test';
-- hot_pct 높을수록 HOT Update 효과적

-- HOT 불가 UPDATE (인덱스 컬럼 변경)
UPDATE hot_test SET status = 'INACTIVE' WHERE id = 1;
-- n_tup_hot_upd 증가 없음
```

### 실험 2: fillfactor 효과 비교

```sql
-- fillfactor=100 (기본값) 테이블
CREATE TABLE hot_ff100 (id SERIAL PRIMARY KEY, val TEXT)
WITH (fillfactor = 100);

-- fillfactor=70 테이블
CREATE TABLE hot_ff70 (id SERIAL PRIMARY KEY, val TEXT)
WITH (fillfactor = 70);

INSERT INTO hot_ff100 (val) SELECT md5(i::text) FROM generate_series(1, 10000) i;
INSERT INTO hot_ff70  (val) SELECT md5(i::text) FROM generate_series(1, 10000) i;

SELECT pg_stat_reset();

-- 두 테이블에 동일한 UPDATE
UPDATE hot_ff100 SET val = md5(random()::text);
UPDATE hot_ff70  SET val = md5(random()::text);

-- HOT 비율 비교
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(n_tup_hot_upd * 100.0 / nullif(n_tup_upd, 0), 1) AS hot_pct
FROM pg_stat_user_tables
WHERE relname IN ('hot_ff100', 'hot_ff70');
-- hot_ff70의 hot_pct가 높음
```

### 실험 3: HOT 체인을 pageinspect로 관찰

```sql
CREATE EXTENSION pageinspect;

-- 소규모 테이블에서 HOT 관찰
CREATE TABLE hot_chain (id INT, val TEXT) WITH (fillfactor = 50);
CREATE INDEX ON hot_chain(id);

INSERT INTO hot_chain VALUES (1, 'v1');

-- HOT Update
UPDATE hot_chain SET val = 'v2' WHERE id = 1;
UPDATE hot_chain SET val = 'v3' WHERE id = 1;

-- 페이지 내부 확인
SELECT
    lp AS item_id,
    lp_flags,
    t_xmin, t_xmax,
    t_ctid,
    CASE
        WHEN (t_infomask2 & 32768) > 0 THEN 'HOT_UPDATED'
        WHEN (t_infomask2 & 16384) > 0 THEN 'HEAP_ONLY_TUPLE'
        ELSE 'NORMAL'
    END AS hot_flag
FROM heap_page_items(get_raw_page('hot_chain', 0));
-- HOT_UPDATED: 이 튜플이 HOT Update의 원본
-- HEAP_ONLY_TUPLE: 이 튜플이 HOT Update로 삽입된 새 버전
```

---

## 📊 MySQL과 비교

```
MySQL InnoDB UPDATE vs PostgreSQL HOT Update:

MySQL:
  UPDATE users SET last_login = NOW() WHERE id = 1;
  → Clustered Index 페이지의 레코드를 직접 수정 (In-Place)
  → Secondary Index(last_login에 없다면): 변경 없음
  → last_login에 Secondary Index 있다면: 해당 Index 갱신

  결과: "인덱스 없는 컬럼 UPDATE" = In-Place 수정
        별도의 버전 체인 없음, 추가 페이지 필요 없음

PostgreSQL HOT Update:
  UPDATE users SET last_login = NOW() WHERE id = 1;
  → 새 버전 같은 페이지에 삽입 (In-Place 아님)
  → 인덱스: 갱신 없음 (t_ctid 체인으로 해결)
  → Dead Tuple 발생 → VACUUM 필요

MySQL In-Place 업데이트의 장점:
  추가 공간 불필요, Dead Tuple 없음
  단, Undo Log에 이전 버전 저장 (공간 상쇄)

PostgreSQL HOT의 장점:
  In-Place 수정 불가하지만 인덱스 갱신 비용 제거
  인덱스가 많은 테이블에서 특히 효과적
```

---

## ⚖️ 트레이드오프

```
HOT Update 최적화 적용 판단:

fillfactor 낮추기:
  장점: HOT Update 비율 증가 → 쓰기 성능 향상
  단점: 테이블 크기 증가 (빈 공간 예약)
        SeqScan 시 더 많은 페이지 읽음

인덱스 제거:
  장점: HOT Update 비율 증가 + 인덱스 유지 비용 제거
  단점: 해당 컬럼으로 조회 시 SeqScan

적합한 시나리오:
  ✓ UPDATE 컬럼이 조회에 거의 사용 안 됨 (last_login, score 등 집계용)
  ✓ 동일 행을 자주 업데이트 (상태 추적, 카운터)
  ✓ 인덱스가 많은 테이블

부적합한 시나리오:
  ✗ 인덱스 컬럼을 업데이트해야 할 때 (status, category 등 필터링 기준)
  ✗ 데이터가 INSERT되고 거의 UPDATE가 없는 경우
  ✗ fillfactor 낮추는 것이 저장 공간 부담되는 경우
```

---

## 📌 핵심 정리

```
HOT Update 핵심:

조건 (둘 다 만족해야):
  ① 변경된 컬럼이 어떤 인덱스에도 포함 안 됨
  ② NEW 버전을 같은 Heap 페이지에 삽입 가능 (여유 공간)

내부 구조:
  OLD 튜플: HEAP_HOT_UPDATED 플래그 + t_ctid → NEW 위치
  NEW 튜플: HEAP_ONLY_TUPLE 플래그 (인덱스 엔트리 없음)
  인덱스: 변경 없음 (t_ctid 체인으로 최신 버전 탐색)

fillfactor의 역할:
  낮은 fillfactor → 페이지 여유 확보 → 조건 2 충족 가능성 ↑
  fillfactor = 70~80: UPDATE 빈번한 테이블 권장

모니터링:
  SELECT n_tup_upd, n_tup_hot_upd,
         round(n_tup_hot_upd * 100.0 / nullif(n_tup_upd, 0), 1) AS hot_pct
  FROM pg_stat_user_tables WHERE relname = 'my_table';

VACUUM과의 관계:
  VACUUM이 HOT 체인 정리 (LP_REDIRECT로 단축)
  Page-level Pruning: 읽기 시점에 HOT 체인 정리
```

---

## 🤔 생각해볼 문제

**Q1.** HOT Update가 발생해도 Dead Tuple은 여전히 생성된다. 그렇다면 HOT Update의 핵심 이점은 무엇인가?

<details>
<summary>해설 보기</summary>

HOT Update의 핵심 이점은 **인덱스 갱신 비용 제거**입니다. Dead Tuple은 여전히 생성되지만, 인덱스 페이지의 수정이 없습니다.

인덱스 5개가 있는 테이블에서 UPDATE 1건의 비용 비교:
- 일반 UPDATE: Heap 쓰기 1 + 인덱스 쓰기 5 = 최소 6번 페이지 수정
- HOT Update: Heap 쓰기 1 = 1번 페이지 수정 (인덱스 갱신 없음)

이는 WAL 크기에도 영향을 줍니다. 인덱스 페이지 수정이 없으면 관련 WAL 레코드도 생성되지 않아 WAL 생성량이 줄어듭니다. 동시에 인덱스 페이지 Lock 경합도 줄어들어 동시성이 향상됩니다.

Dead Tuple 자체는 VACUUM이 처리하므로, HOT Update의 이점은 "쓰기 시점의 인덱스 I/O 제거"에 있습니다.

</details>

---

**Q2.** `fillfactor = 50`으로 설정하면 HOT Update 비율은 올라가겠지만 어떤 부작용이 있는가?

<details>
<summary>해설 보기</summary>

`fillfactor = 50`이면 페이지의 절반만 데이터로 채우고 나머지는 비워둡니다. 부작용:

1. **테이블 크기 2배**: 같은 데이터가 2배 많은 페이지를 사용. SeqScan 비용 2배.
2. **shared_buffers 효율 저하**: 실제 데이터보다 2배 많은 페이지를 캐시해야 함.
3. **백업 크기 증가**: pg_dump 크기 증가.
4. **Cold Start I/O 증가**: 캐시 워밍업 시 더 많은 페이지 읽기 필요.

실용적 권장:
- UPDATE가 많고 읽기도 많은 테이블: `fillfactor = 70~80`
- UPDATE가 매우 빈번하고 읽기는 인덱스로만 하는 테이블: `fillfactor = 60~70`
- `fillfactor = 50`은 극단적으로 UPDATE가 빈번하고 SeqScan이 거의 없는 경우만

```sql
-- 현재 fillfactor 확인
SELECT relname, reloptions FROM pg_class WHERE relname = 'users';

-- fillfactor 변경 (즉시 적용, 기존 페이지는 점진적으로 반영)
ALTER TABLE users SET (fillfactor = 75);
-- 기존 가득 찬 페이지는 VACUUM 또는 데이터 재삽입 후 효과
```

</details>

---

<div align="center">

**[⬅️ 이전: Autovacuum 튜닝](./04-autovacuum-tuning.md)** | **[홈으로 🏠](../README.md)** | **[다음: XID Wraparound ➡️](./06-xid-wraparound.md)**

</div>
