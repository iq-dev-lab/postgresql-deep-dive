# Upsert와 SKIP LOCKED — 동시성 처리의 핵심 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `INSERT ... ON CONFLICT DO UPDATE`(Upsert)는 내부적으로 어떻게 동작하는가?
- 동시에 여러 트랜잭션이 같은 키로 Upsert할 때 충돌은 어떻게 처리되는가?
- `EXCLUDED`는 무엇이고 어떻게 사용하는가?
- `SELECT ... FOR UPDATE SKIP LOCKED`로 다중 Worker 큐를 어떻게 구현하는가?
- `FOR SHARE`, `FOR NO KEY UPDATE`, `FOR KEY SHARE`의 차이는?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

"이미 있으면 업데이트, 없으면 삽입"하는 Upsert는 카운터 증가, 세션 갱신, 설정 저장 등 거의 모든 서비스에서 필요하다. 애플리케이션 레벨에서 SELECT → 있으면 UPDATE, 없으면 INSERT를 구현하면 동시성 문제(두 트랜잭션이 동시에 SELECT해서 둘 다 INSERT 시도)가 발생한다. PostgreSQL의 Upsert는 이를 원자적으로 처리한다. SKIP LOCKED는 다중 Worker가 작업 큐에서 중복 없이 작업을 가져가는 패턴의 핵심이다.

---

## 😱 흔한 실수 (Before — Upsert 없이 구현)

```
실수 1: 애플리케이션 레벨 Upsert (경쟁 조건 발생)

  -- Java 코드
  Optional<User> user = userRepo.findById(userId);
  if (user.isPresent()) {
      userRepo.update(userId, newData);
  } else {
      userRepo.insert(userId, newData);
  }

  동시성 문제:
  스레드 A: SELECT → 없음 → INSERT 시도
  스레드 B: SELECT → 없음 → INSERT 시도 (A보다 1ms 늦게)
  스레드 A: INSERT 성공
  스레드 B: INSERT 실패 (PK 충돌)
  → 예외 처리 로직 필요 또는 데이터 손실

  올바른 방법:
  INSERT INTO users (id, data) VALUES (userId, newData)
  ON CONFLICT (id) DO UPDATE SET data = EXCLUDED.data;

실수 2: Upsert를 잘못된 ON CONFLICT 대상으로

  -- 컬럼이 아닌 UNIQUE 제약 이름 사용
  INSERT INTO users (email, name) VALUES ('a@b.com', 'Alice')
  ON CONFLICT ON CONSTRAINT users_email_key DO UPDATE SET name = EXCLUDED.name;
  → OK, 하지만 컬럼명으로 지정하는 것이 더 명확:
  ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;

실수 3: FOR UPDATE 없는 큐에서 중복 처리

  -- Worker 1
  SELECT * FROM tasks WHERE status = 'PENDING' LIMIT 1;
  -- Worker 2 (동시에)
  SELECT * FROM tasks WHERE status = 'PENDING' LIMIT 1;
  → 두 Worker가 같은 task 선택 → 중복 처리

  SKIP LOCKED 사용:
  SELECT * FROM tasks WHERE status = 'PENDING' LIMIT 1 FOR UPDATE SKIP LOCKED;
  → Worker 1이 lock 획득 → Worker 2는 다음 available task 선택
```

---

## ✨ 올바른 접근 (After — Upsert와 SKIP LOCKED 올바른 활용)

```
Upsert 패턴:

  -- 카운터 증가 (페이지뷰 집계)
  INSERT INTO page_views (page_id, view_count, last_viewed)
  VALUES (42, 1, NOW())
  ON CONFLICT (page_id) DO UPDATE
  SET view_count = page_views.view_count + 1,
      last_viewed = EXCLUDED.last_viewed;
  -- page_views.view_count: 기존 값 (테이블 참조)
  -- EXCLUDED.view_count: 새로 삽입하려던 값 (1)

  -- 업데이트 없이 무시 (DO NOTHING)
  INSERT INTO event_log (event_id, processed_at)
  VALUES (evt_id, NOW())
  ON CONFLICT (event_id) DO NOTHING;
  -- 이미 처리된 이벤트는 무시 (멱등성 보장)

SKIP LOCKED 큐 패턴:

  -- Worker에서 다음 작업 가져오기 (원자적 + 잠금)
  BEGIN;
  WITH task AS (
      SELECT id FROM task_queue
      WHERE status = 'PENDING'
      ORDER BY priority DESC, created_at
      LIMIT 1
      FOR UPDATE SKIP LOCKED
  )
  UPDATE task_queue SET status = 'PROCESSING', worker_id = $1
  WHERE id = (SELECT id FROM task)
  RETURNING *;
  -- 처리 후 COMMIT 또는 ROLLBACK (실패 시 status 복원)
```

---

## 🔬 내부 동작 원리

### 1. Upsert 내부 동작 (INSERT ON CONFLICT)

```
Upsert 실행 단계:

INSERT INTO scores (user_id, score)
VALUES (42, 100)
ON CONFLICT (user_id) DO UPDATE
SET score = EXCLUDED.score + scores.score;

Step 1: INSERT 시도
  → (42, 100) 삽입 시도
  → user_id=42에 UNIQUE 인덱스 확인

Step 2: 충돌 감지
  충돌 없음: 그냥 INSERT 완료
  충돌 있음: ON CONFLICT 절 처리 →

Step 3: DO UPDATE 실행
  → 현재 행: scores(user_id=42, score=80)
  → EXCLUDED: 삽입하려던 값 (user_id=42, score=100)
  → SET score = 100 + 80 = 180
  → 기존 행 UPDATE

Step 4: 결과
  → scores(user_id=42, score=180)

EXCLUDED 가상 테이블:
  ON CONFLICT DO UPDATE 절에서 사용 가능
  삽입하려던 새 값을 참조
  → EXCLUDED.column: 삽입 시도한 값
  → table.column: 현재 저장된 값 (충돌 대상)

DO NOTHING:
  충돌 시 아무것도 안 함
  → 멱등성 패턴: 이미 있으면 그냥 넘어감
  → 예: 이미 처리된 이벤트 로그 삽입 무시

Upsert와 RETURNING:
  INSERT ... ON CONFLICT DO UPDATE ... RETURNING *;
  → INSERT 성공 시: 삽입된 행 반환
  → DO UPDATE 발생 시: 업데이트된 행 반환
  → DO NOTHING 발생 시: 아무것도 반환 안 함 (xmax_update 없는 행은 제외)
```

### 2. 동시 Upsert 충돌 처리

```
동시 Upsert 시나리오:

트랜잭션 A                    트랜잭션 B
────────────────────────────────────────────────────
INSERT INTO scores
(user_id, score)
VALUES (42, 100)
ON CONFLICT (user_id)
DO UPDATE SET score = ...;   

(user_id=42 없음 → INSERT 시도)
(user_id 인덱스에 Lock 획득)
                               INSERT INTO scores
                               (user_id, score)
                               VALUES (42, 200)
                               ON CONFLICT (user_id)
                               DO UPDATE SET score = ...;
                               
                               (user_id=42 없음 → INSERT 시도)
                               (user_id 인덱스에 Lock 대기)

COMMIT;
                               (A가 커밋 → B가 Lock 획득)
                               (충돌 감지: user_id=42 이미 있음)
                               (DO UPDATE: 기존 행 업데이트)
                               COMMIT;

결과:
  최종 score = A의 값으로 업데이트 후 B의 값으로 재업데이트
  → 순차적으로 처리됨 (원자적)

주의: 누적 카운터에서 순서 보장 없음
  두 트랜잭션이 각각 +1을 하면 최종 결과 = 기존 + 2 (보장)
  하지만 누가 먼저 커밋했는지에 따라 중간 상태 다름

Upsert가 NOT 보장하는 것:
  ① 특정 순서로 업데이트 보장 없음
  ② "내가 삽입한 행"인지 "다른 트랜잭션이 삽입한 행"인지 구분 불가
    → xmax_update 메타데이터로 확인 가능하지만 복잡
```

### 3. SELECT FOR UPDATE와 잠금 모드

```
Row-Level Lock 모드 (강함 → 약함):

FOR UPDATE:
  가장 강한 잠금
  다른 트랜잭션의 FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE, FOR KEY SHARE 차단
  사용: 행을 수정하거나 삭제할 의도

FOR NO KEY UPDATE:
  FOR UPDATE보다 약함
  다른 트랜잭션의 FOR KEY SHARE 허용
  사용: PK/UNIQUE 변경 없이 UPDATE할 의도

FOR SHARE:
  읽기 잠금 (공유)
  여러 트랜잭션이 동시에 FOR SHARE 가능
  FOR UPDATE 차단, FOR SHARE 허용
  사용: 행이 삭제/변경되지 않도록 보장

FOR KEY SHARE:
  가장 약한 잠금
  FK 참조 검사 등에 사용

충돌 매트릭스:
           |FOR UPDATE|FOR NO KEY|FOR SHARE|FOR KEY SHARE|
FOR UPDATE | ✗ 충돌   | ✗ 충돌   | ✗ 충돌  | ✗ 충돌      |
FOR NO KEY | ✗ 충돌   | ✗ 충돌   | ✗ 충돌  | ✓ 허용      |
FOR SHARE  | ✗ 충돌   | ✗ 충돌   | ✓ 허용  | ✓ 허용      |
FOR KEY SH | ✗ 충돌   | ✓ 허용   | ✓ 허용  | ✓ 허용      |
```

### 4. SKIP LOCKED — 다중 Worker 큐

```
SKIP LOCKED 동작:

  SELECT * FROM tasks
  WHERE status = 'PENDING'
  ORDER BY priority DESC
  LIMIT 1
  FOR UPDATE SKIP LOCKED;

  일반 FOR UPDATE:
  → Lock이 걸린 행에 도달하면 그 행의 Lock이 해제될 때까지 대기

  FOR UPDATE SKIP LOCKED:
  → Lock이 걸린 행은 건너뜀 (Skip)
  → 다음 Lock 가능한 행 선택

  다중 Worker 시나리오:
  Worker 1: SELECT LIMIT 1 FOR UPDATE SKIP LOCKED → task_id=10 Lock
  Worker 2: SELECT LIMIT 1 FOR UPDATE SKIP LOCKED → task_id=10 Skip → task_id=11 Lock
  Worker 3: SELECT LIMIT 1 FOR UPDATE SKIP LOCKED → task_id=10,11 Skip → task_id=12 Lock
  → 각 Worker가 다른 작업 처리 (중복 없음)

실전 구현 패턴:

  -- Worker 코드 (반복 실행)
  BEGIN;
  WITH claimed AS (
      SELECT id, payload
      FROM task_queue
      WHERE status = 'PENDING'
        AND (scheduled_at IS NULL OR scheduled_at <= NOW())
      ORDER BY priority DESC, created_at
      LIMIT 1
      FOR UPDATE SKIP LOCKED
  )
  UPDATE task_queue
  SET status = 'PROCESSING',
      started_at = NOW(),
      worker_id = current_setting('myapp.worker_id')
  WHERE id = (SELECT id FROM claimed)
  RETURNING id, payload;

  -- 결과가 없으면: 처리할 작업 없음 → COMMIT + 대기
  -- 결과가 있으면: payload로 작업 처리 → 성공 시 COMMIT, 실패 시 ROLLBACK

SKIP LOCKED vs 상태 업데이트:

  방법 1: SKIP LOCKED (권장)
    Lock으로 동시성 제어 → 다른 Worker가 같은 행 접근 불가

  방법 2: Status 업데이트 (Optimistic)
    UPDATE task_queue SET status = 'PROCESSING' WHERE id = ?
    AND status = 'PENDING'  -- 조건부 업데이트
    → 업데이트된 행이 없으면 다른 Worker가 먼저 가져감 → 재시도
    → SKIP LOCKED보다 충돌 가능성 있음
```

---

## 💻 실전 실험

### 실험 1: Upsert 카운터 집계

```sql
-- 페이지뷰 집계 테이블
CREATE TABLE page_stats (
    page_id INT PRIMARY KEY,
    view_count INT DEFAULT 0,
    unique_visitors INT DEFAULT 0,
    last_viewed TIMESTAMPTZ
);

-- Upsert: 있으면 카운터 증가, 없으면 삽입
INSERT INTO page_stats (page_id, view_count, unique_visitors, last_viewed)
VALUES (42, 1, 1, NOW())
ON CONFLICT (page_id) DO UPDATE
SET
    view_count = page_stats.view_count + 1,
    -- unique_visitors는 EXCLUDED의 값으로 업데이트 (신규 방문자 시)
    last_viewed = EXCLUDED.last_viewed;

-- EXCLUDED 확인
INSERT INTO page_stats (page_id, view_count, unique_visitors, last_viewed)
VALUES (42, 5, 3, NOW() + INTERVAL '1 hour')
ON CONFLICT (page_id) DO UPDATE
SET
    view_count = page_stats.view_count + EXCLUDED.view_count,
    last_viewed = EXCLUDED.last_viewed
RETURNING
    page_id,
    page_stats.view_count AS new_count,
    EXCLUDED.view_count AS added_count;

-- 조건부 Upsert (최신 데이터만 업데이트)
INSERT INTO page_stats (page_id, view_count, last_viewed)
VALUES (42, 1, NOW())
ON CONFLICT (page_id) DO UPDATE
SET
    view_count = page_stats.view_count + 1,
    last_viewed = EXCLUDED.last_viewed
WHERE page_stats.last_viewed < EXCLUDED.last_viewed;  -- 더 최신일 때만
```

### 실험 2: SKIP LOCKED 큐 구현

```sql
-- 작업 큐
CREATE TABLE job_queue (
    id BIGSERIAL PRIMARY KEY,
    job_type TEXT NOT NULL,
    payload JSONB,
    status TEXT DEFAULT 'PENDING',
    priority INT DEFAULT 5,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    error_message TEXT
);

CREATE INDEX ON job_queue(status, priority DESC, created_at) WHERE status = 'PENDING';

-- 작업 삽입
INSERT INTO job_queue (job_type, payload, priority)
SELECT
    (ARRAY['email', 'sms', 'push'])[ceil(random()*3)::INT],
    jsonb_build_object('user_id', (random()*1000)::INT),
    (random()*10)::INT
FROM generate_series(1, 1000);

-- Worker가 작업을 가져가는 쿼리
BEGIN;
WITH claimed AS (
    SELECT id, job_type, payload
    FROM job_queue
    WHERE status = 'PENDING'
    ORDER BY priority DESC, created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE job_queue
SET status = 'PROCESSING', started_at = NOW()
WHERE id = (SELECT id FROM claimed)
RETURNING id, job_type, payload;
COMMIT;

-- 완료 처리
UPDATE job_queue
SET status = 'DONE', completed_at = NOW()
WHERE id = $1;

-- 실패 처리 (재시도 가능 상태로)
UPDATE job_queue
SET status = 'PENDING', error_message = $2, started_at = NULL
WHERE id = $1;
```

### 실험 3: 동시 Upsert 테스트

```sql
-- 동시 Upsert 시 데이터 무결성 테스트
-- (두 세션에서 동시 실행)

-- 세션 1:
BEGIN;
INSERT INTO page_stats (page_id, view_count, last_viewed)
VALUES (99, 1, NOW())
ON CONFLICT (page_id) DO UPDATE
SET view_count = page_stats.view_count + 1,
    last_viewed = EXCLUDED.last_viewed;
COMMIT;

-- 세션 2 (동시에):
BEGIN;
INSERT INTO page_stats (page_id, view_count, last_viewed)
VALUES (99, 1, NOW())
ON CONFLICT (page_id) DO UPDATE
SET view_count = page_stats.view_count + 1,
    last_viewed = EXCLUDED.last_viewed;
COMMIT;

-- 결과 확인: view_count = 2 (원자적 처리)
SELECT * FROM page_stats WHERE page_id = 99;
```

---

## 📊 MySQL과 비교

```
MySQL Upsert 방식:

MySQL:
  방법 1: INSERT ... ON DUPLICATE KEY UPDATE
    INSERT INTO page_stats (page_id, view_count)
    VALUES (42, 1)
    ON DUPLICATE KEY UPDATE
    view_count = view_count + 1;
    → 행이 있으면 UPDATE (DUPLICATE KEY = UNIQUE 제약 위반 시)

  방법 2: REPLACE INTO
    REPLACE INTO page_stats (page_id, view_count)
    VALUES (42, 100);
    → 충돌 시 DELETE + INSERT (기존 행 완전 교체)
    → 이전 값 참조 불가 (view_count + 1 불가)

PostgreSQL:
  INSERT ... ON CONFLICT DO UPDATE:
    EXCLUDED로 새 값 참조
    기존 값과 새 값 모두 사용 가능
    조건부 UPDATE (WHERE 절)
    DO NOTHING으로 무시 가능

  PostgreSQL이 더 유연:
  ① EXCLUDED.col로 삽입하려던 값 참조
  ② WHERE 조건으로 조건부 업데이트
  ③ DO NOTHING으로 멱등성 패턴
  ④ RETURNING으로 결과 확인

SKIP LOCKED:
  MySQL 8.0+: FOR UPDATE SKIP LOCKED 지원 (PostgreSQL과 동일)
  MySQL 5.7 이하: 지원 안 됨
```

---

## ⚖️ 트레이드오프

```
Upsert 사용 시 주의:

① ON CONFLICT 대상 명확히:
   ON CONFLICT (col) : 특정 컬럼의 UNIQUE 제약
   ON CONFLICT ON CONSTRAINT name: 제약 이름
   ON CONFLICT: 아무 충돌이나 → DO NOTHING에만 사용

② 성능:
   일반 INSERT보다 약간 느림 (충돌 감지 오버헤드)
   대량 Upsert: INSERT ... ON CONFLICT ... COPY보다 느림
   → 대량 데이터는 staging 테이블 → INSERT ... SELECT 또는 COPY 후 처리

③ 동시성:
   같은 키 동시 Upsert → 한 트랜잭션 대기
   Hot Spot: 같은 키를 매우 자주 Upsert → Lock 경합
   해결: 집계 후 Upsert (배치) 또는 파티셔닝으로 분산

SKIP LOCKED 주의:
  ① SKIP LOCKED + LIMIT 조합 필수
     LIMIT 없으면 모든 Lock 가능한 행 가져옴

  ② 트랜잭션 유지:
     FOR UPDATE는 트랜잭션 커밋/롤백 시 Lock 해제
     처리 중 오류 → ROLLBACK → 다른 Worker가 접근 가능

  ③ Starvation:
     우선순위 기반 큐에서 낮은 우선순위 작업이 영원히 밀릴 수 있음
     → created_at 기반 타임아웃 처리 필요
```

---

## 📌 핵심 정리

```
Upsert와 SKIP LOCKED 핵심:

Upsert:
  INSERT INTO t (pk, col) VALUES (val, val2)
  ON CONFLICT (pk) DO UPDATE
  SET col = EXCLUDED.col  -- 삽입하려던 값
      또는
  SET col = t.col + 1     -- 기존 값 기반 업데이트
  [WHERE t.col < EXCLUDED.col]  -- 조건부 업데이트

  DO NOTHING: 멱등성 (이미 있으면 무시)
  RETURNING: 삽입/업데이트 결과 반환

동시 Upsert:
  같은 키 동시 Upsert → Lock 순차 처리 → 원자성 보장
  Hot Spot 주의: 동일 키에 높은 빈도 Upsert → Lock 경합

SKIP LOCKED:
  SELECT ... FOR UPDATE SKIP LOCKED
  Lock이 걸린 행은 건너뜀
  다중 Worker 큐 구현에 필수

Row Lock 모드:
  FOR UPDATE: 가장 강함 (수정/삭제 의도)
  FOR NO KEY UPDATE: PK 변경 없는 UPDATE
  FOR SHARE: 읽기 잠금 (삭제 방지)
  FOR KEY SHARE: 가장 약함 (FK 검사)
```

---

## 🤔 생각해볼 문제

**Q1.** `ON CONFLICT DO UPDATE`의 `WHERE` 절은 언제 사용하고, DO UPDATE가 실행되지 않으면 RETURNING은 무엇을 반환하는가?

<details>
<summary>해설 보기</summary>

`WHERE` 절은 충돌이 발생했을 때 업데이트를 **조건부로 실행**합니다:

```sql
INSERT INTO prices (product_id, price, updated_at)
VALUES (42, 1500, NOW())
ON CONFLICT (product_id) DO UPDATE
SET price = EXCLUDED.price, updated_at = EXCLUDED.updated_at
WHERE prices.price <> EXCLUDED.price;  -- 가격이 다를 때만 업데이트
```

충돌이 있지만 WHERE 조건이 false이면 DO UPDATE가 실행되지 않습니다. 이 경우 행은 변경되지 않고, `RETURNING`은 **아무것도 반환하지 않습니다** (빈 결과). PostgreSQL 15+에서는 `RETURNING`이 항상 행을 반환하도록 하는 방법이 없으므로, "업데이트가 발생했는지"를 알려면 변경 전/후 값을 비교하거나 애플리케이션 레벨에서 처리해야 합니다.

</details>

---

**Q2.** `FOR UPDATE SKIP LOCKED`로 가져온 작업을 처리 중 Worker가 죽으면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Worker가 죽으면 PostgreSQL 연결이 끊어지고 트랜잭션이 **자동으로 ROLLBACK**됩니다. Row-Level Lock이 해제되고, 해당 작업의 `status`는 변경되지 않은 상태('PENDING')로 남습니다. 다른 Worker가 다음에 SELECT할 때 해당 작업을 다시 가져갈 수 있습니다.

이것이 SKIP LOCKED 큐의 핵심 장점입니다. 하지만 몇 가지 주의가 필요합니다:

1. **트랜잭션 내에서 상태 변경**: Worker가 작업을 가져오면서 `status='PROCESSING'`으로 변경하는 UPDATE도 같은 트랜잭션에 포함해야 합니다. 트랜잭션 외부에서 상태를 변경하면 Worker 죽을 때 복원되지 않습니다.

2. **오래된 PROCESSING 항목 모니터링**: Worker가 비정상 종료됐지만 상태가 'PROCESSING'으로 남아있는 경우를 위해 타임아웃 처리가 필요합니다:
```sql
-- N분 이상 PROCESSING 중인 작업을 PENDING으로 복원
UPDATE task_queue SET status = 'PENDING', started_at = NULL
WHERE status = 'PROCESSING' AND started_at < NOW() - INTERVAL '10 minutes';
```

</details>

---

<div align="center">

**[⬅️ 이전: LATERAL JOIN](./04-lateral-join.md)** | **[홈으로 🏠](../README.md)** | **[다음: 집계 함수 심화 ➡️](./06-aggregation-functions.md)**

</div>
