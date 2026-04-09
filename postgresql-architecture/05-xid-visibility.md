# 트랜잭션 ID(XID)와 가시성 규칙 — 어떤 버전의 튜플이 보이는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PostgreSQL의 모든 튜플에 `xmin`/`xmax`가 있는 이유는 무엇인가?
- 트랜잭션 스냅샷은 어떤 구조이고, 이것으로 어떤 버전의 튜플을 볼지 어떻게 결정하는가?
- `xmin`, `xmax`, `xip` 세 값이 스냅샷에서 각각 무엇을 의미하는가?
- `REPEATABLE READ`와 `READ COMMITTED`는 스냅샷을 어떻게 다르게 다루는가?
- XID Wraparound는 왜 DB를 강제로 멈추게 하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL MVCC의 핵심 질문은 "어떤 트랜잭션이 어떤 버전의 튜플을 볼 수 있는가"다. 이를 결정하는 메커니즘이 **스냅샷(Snapshot)**과 **XID 기반 가시성 규칙**이다. 이 규칙을 이해하지 못하면 "왜 방금 커밋한 데이터가 안 보이지?", "왜 VACUUM이 이 Dead Tuple을 지우지 못하지?", "XID Wraparound 경고가 왜 심각한가?"에 답하기 어렵다. 특히 XID Wraparound는 운영 실수 중 가장 치명적인 것 중 하나로, 처리하지 않으면 PostgreSQL이 스스로 강제 종료된다.

---

## 😱 흔한 실수 (Before — 가시성 규칙을 모를 때)

```
흔한 실수 1: REPEATABLE READ에서 왜 새 데이터가 안 보이지?

  트랜잭션 A (REPEATABLE READ):
    BEGIN;
    SELECT count(*) FROM orders;  -- 1,000

  트랜잭션 B:
    INSERT INTO orders ...1,000건 커밋

  트랜잭션 A (계속):
    SELECT count(*) FROM orders;  -- 여전히 1,000 (왜?!)

  혼란: "커밋된 데이터인데 왜 안 보이지?"

  이유: REPEATABLE READ는 트랜잭션 시작 시점의 스냅샷을 고정
        스냅샷 이후 커밋된 트랜잭션은 보이지 않음 → 정상 동작

흔한 실수 2: XID Wraparound 경고 무시

  경고 메시지:
  WARNING: database "mydb" must be vacuumed within 177009986 transactions
  HINT:  To avoid a database shutdown, execute a database-wide VACUUM in "mydb".

  "그냥 경고겠지" → 무시

  결과:
  ERROR: database is not accepting commands to avoid wraparound data loss in database "mydb"
  HINT: Stop the postmaster and vacuum that database in single-user mode.
  → 모든 쓰기 쿼리 거부!
  → 단일 사용자 모드에서 VACUUM FREEZE 실행해야 복구

흔한 실수 3: 가시성 오해로 잘못된 격리 수준 선택

  "SELECT를 빠르게 하려고 READ UNCOMMITTED 사용"
  PostgreSQL: READ UNCOMMITTED = READ COMMITTED로 동작
  Dirty Read를 허용하지 않음 (구현 자체가 불가)
  → 격리 수준 변경이 의도한 효과 없음
```

---

## ✨ 올바른 접근 (After — 가시성 규칙 이해 후)

```
격리 수준 선택 전략:

  READ COMMITTED (기본값):
    쿼리마다 새 스냅샷 → 커밋된 데이터는 항상 보임
    → 같은 트랜잭션 내에서도 외부 커밋이 보임
    → 적합: 대부분의 OLTP (항상 최신 데이터 필요)

  REPEATABLE READ:
    트랜잭션 시작 시 스냅샷 고정 → 트랜잭션 종료까지 일관된 뷰
    PostgreSQL: Phantom Read도 방지 (MySQL REPEATABLE READ보다 강함)
    → 적합: 보고서, 배치 처리, 일관된 읽기가 필요한 경우

  SERIALIZABLE:
    SSI(Serializable Snapshot Isolation)로 직렬화 가능성 보장
    → 적합: 복잡한 비즈니스 로직, 데이터 무결성 최우선

XID Wraparound 예방:

  일상 모니터링 (크론 또는 모니터링 시스템에 추가):
    SELECT datname, age(datfrozenxid) AS xid_age
    FROM pg_database
    ORDER BY xid_age DESC;

  임계값:
    age < 100,000,000: 정상
    age > 200,000,000: Autovacuum 적극 모니터링
    age > 1,000,000,000: 즉시 수동 VACUUM FREEZE 실행
    age > 2,000,000,000: PostgreSQL 강제 종료 임박!
```

---

## 🔬 내부 동작 원리

### 1. 트랜잭션 ID(XID) 구조

```
PostgreSQL XID:
  32비트 정수: 1 ~ 4,294,967,295 (약 40억)
  순차적으로 증가: 각 트랜잭션 시작 시 하나씩 할당

특수 XID:
  0: InvalidTransactionId (유효하지 않음)
  1: BootstrapTransactionId (PostgreSQL 초기화)
  2: FrozenTransactionId (Freeze된 튜플, 항상 가시)
  3~: 일반 트랜잭션

XID 조회:
  -- 현재 트랜잭션 XID
  SELECT txid_current();  -- 예: 501

  -- 현재 트랜잭션 XID (트랜잭션 없으면 NULL)
  SELECT txid_current_if_assigned();

XID 할당 시점:
  SELECT, SHOW: XID 할당 없음 (읽기 전용)
  INSERT, UPDATE, DELETE: XID 할당됨

Backend XID vs Virtual XID:
  Virtual XID: 읽기 전용 트랜잭션도 갖는 내부 식별자
               Backend ID + Local Counter (XID 소모 없음)
  Real XID: 데이터 변경 시 할당, WAL에 기록
  → 읽기 전용 트랜잭션은 Real XID를 소모하지 않음 (Wraparound에 유리)

  pg_stat_activity에서 확인:
  SELECT pid, backend_xid, backend_xmin FROM pg_stat_activity;
  -- backend_xid: 현재 트랜잭션의 Real XID
  -- backend_xmin: 이 backend가 필요로 하는 최소 XID (VACUUM에 영향)
```

### 2. 스냅샷(Snapshot) 구조

```
PostgreSQL 스냅샷의 3가지 구성요소:

  Snapshot {
      xmin: 이 값보다 작은 XID의 트랜잭션은 모두 완료됨
            (커밋 or 롤백) → 해당 튜플은 이 스냅샷에서 가시 판단 가능
      xmax: 이 값 이상의 XID는 아직 시작도 안 됨
            → 해당 튜플은 이 스냅샷에서 보이지 않음
      xip:  [xmin, xmax) 사이에서 아직 진행 중인 트랜잭션 XID 목록
            → 이 목록에 있는 XID가 만든 튜플은 보이지 않음
  }

예시:
  현재 시스템 상태:
    완료된 트랜잭션: XID 100, 101, 102, 103, 105, 106
    진행 중인 트랜잭션: XID 104, 107
    다음 할당될 XID: 108

  이 시점에 생성된 스냅샷:
    xmin = 104  (100~103은 모두 완료)
    xmax = 108  (108 이상은 아직 없음)
    xip = [104, 107]  (진행 중)

  가시성 판단:
    xmin=100인 튜플: 100 < 104 → 스냅샷 xmin보다 작음 → 완료된 것 확실 → 커밋 여부 확인
    xmin=104인 튜플: 104는 xip에 있음 → 아직 진행 중 → 보이지 않음
    xmin=105인 튜플: [104,108) 사이, xip에 없음 → 완료됨 → 커밋이면 보임
    xmin=108인 튜플: 108 >= xmax → 스냅샷 이후 생성 → 보이지 않음
```

### 3. 튜플 가시성 판단 알고리즘

```
튜플 T가 스냅샷 S에서 보이는 조건 (단순화):

function isTupleVisible(T, S):

  // xmin 판단: 이 튜플을 만든 트랜잭션이 커밋됐는가?
  if T.xmin >= S.xmax:
      return false  // 스냅샷 이후 생성
  if T.xmin in S.xip:
      return false  // 아직 진행 중
  if not isCommitted(T.xmin):
      return false  // 롤백됨

  // xmax 판단: 이 튜플을 삭제/갱신한 트랜잭션이 커밋됐는가?
  if T.xmax == 0:
      return true   // 삭제/갱신 없음 → 살아있음
  if T.xmax >= S.xmax:
      return true   // 스냅샷 이후 삭제 → 아직 살아있음
  if T.xmax in S.xip:
      return true   // 삭제 트랜잭션이 아직 진행 중 → 살아있음
  if not isCommitted(T.xmax):
      return true   // 삭제 트랜잭션이 롤백됨 → 살아있음

  return false  // xmax 커밋됨 → Dead Tuple

실전 예시:

트랜잭션 T1(XID=500)이 스냅샷 획득:
  S = {xmin=500, xmax=503, xip=[501]}

Heap 페이지에 있는 튜플들:

  튜플 A: xmin=499, xmax=0
    → 499 < 500 (xmin) → 커밋 확인 → 커밋됨
    → xmax=0 → 삭제 안 됨
    → 보임 ✓

  튜플 B: xmin=501, xmax=0
    → 501 in xip=[501] → 진행 중
    → 보이지 않음 ✗

  튜플 C: xmin=499, xmax=502
    → xmin=499: 커밋됨
    → xmax=502: 502 < 503, not in xip, 커밋됨
    → Dead → 보이지 않음 ✗

  튜플 D: xmin=499, xmax=501
    → xmin=499: 커밋됨
    → xmax=501: 501 in xip=[501] → 삭제 트랜잭션 진행 중
    → 살아있음 ✓ (아직 삭제 완료 안 됨)
```

### 4. 격리 수준과 스냅샷 획득 시점

```
READ COMMITTED:
  각 SQL 문장 실행 시마다 새 스냅샷 획득
  → 같은 트랜잭션 내에서도 다른 트랜잭션의 커밋이 보임

  BEGIN;                  -- 스냅샷 없음
  SELECT * FROM orders;   -- 스냅샷 1 획득, 실행, 폐기
  -- (외부에서 INSERT 1000건 커밋)
  SELECT * FROM orders;   -- 스냅샷 2 획득 (1000건 포함!)
  COMMIT;

REPEATABLE READ:
  트랜잭션 첫 번째 SQL 문장 실행 시 스냅샷 획득, 이후 고정
  → 트랜잭션 종료까지 동일한 스냅샷 사용

  BEGIN;                  -- 스냅샷 없음
  SELECT * FROM orders;   -- 스냅샷 획득, 이후 고정
  -- (외부에서 INSERT 1000건 커밋)
  SELECT * FROM orders;   -- 같은 스냅샷 → 1000건 안 보임!
  COMMIT;

PostgreSQL REPEATABLE READ의 특별한 점:
  MySQL REPEATABLE READ: Phantom Read 허용
  PostgreSQL REPEATABLE READ: Phantom Read 방지 (스냅샷 격리로 자동 보장)

  Phantom Read란:
    T1이 "SELECT WHERE price > 100" → 10건
    T2가 price=200인 행 삽입 후 커밋
    T1이 다시 "SELECT WHERE price > 100" → 11건 (MySQL에서)
    PostgreSQL: REPEATABLE READ에서도 10건 (스냅샷 고정이므로)

SERIALIZABLE (SSI):
  Serializable Snapshot Isolation
  REPEATABLE READ보다 강한 격리
  쓰기-쓰기 의존성, 읽기-쓰기 의존성을 추적
  직렬화 불가능한 패턴 감지 시 → ERROR: could not serialize access
  애플리케이션이 재시도 처리 필요
```

### 5. XID Wraparound 문제와 VACUUM Freeze

```
32비트 XID의 한계:
  최대값: 4,294,967,295 (약 40억)
  초당 1,000 트랜잭션 → 약 49일 만에 절반(20억) 소모
  활발한 서비스: 수개월 만에 40억 소모 가능

Wraparound 문제:
  PostgreSQL의 XID 비교는 모듈 산술(modular arithmetic) 사용
  32비트 XID 공간을 절반으로 나눠 "과거/미래" 판단

  정상 상태:
    현재 XID = 3,000,000,000
    XID = 1,000,000,000 → "20억 전" → 과거 (보임)
    XID = 3,500,000,000 → "5억 후" → 미래 (보이지 않음)

  Wraparound 발생:
    현재 XID = 4,294,967,295 (최대)
    다음 트랜잭션: XID = 1 (다시 시작)
    XID = 3,000,000,000 → "20억 후" → 미래로 보임!
    → XID=3,000,000,000이 삽입한 데이터가 사라짐!
    → 데이터 손상!

VACUUM Freeze:
  오래된 튜플의 xmin을 FrozenTransactionId(2)로 설정
  → Freeze된 튜플은 어떤 XID보다도 "과거"로 판단
  → Wraparound 영향 없음

  vacuum_freeze_min_age (기본 50,000,000):
    XID age가 이 값 이상인 튜플을 Freeze 대상으로 처리

  vacuum_freeze_table_age (기본 150,000,000):
    datfrozenxid의 age가 이 값 이상이면 테이블 전체 스캔하며 Freeze

PostgreSQL 안전 장치:
  age가 1,100,000,000 도달 → 경고 로그
  age가 1,500,000,000 도달 → ERROR, 쓰기 거부 (Wraparound 예방)
  → 이 시점에서 VACUUM FREEZE 필수

  SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age DESC;
  → age > 200,000,000: Autovacuum 상태 점검
  → age > 1,000,000,000: 즉시 수동 조치
```

---

## 💻 실전 실험

### 실험 1: 스냅샷으로 가시성 직접 관찰

```sql
-- 두 세션에서 실행하여 가시성 확인

-- 세션 1:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT txid_current_snapshot();
-- 출력: 501:501: (진행 중 트랜잭션 없음, 다음 XID=501)

SELECT count(*) FROM orders;  -- 현재 count

-- 세션 2 (별도 세션):
INSERT INTO orders (status) VALUES ('NEW');
COMMIT;

-- 세션 1 (계속):
SELECT count(*) FROM orders;  -- 여전히 같은 count (REPEATABLE READ)
COMMIT;

-- 세션 1 (새 트랜잭션, READ COMMITTED):
SELECT count(*) FROM orders;  -- 이제 +1 보임
```

### 실험 2: 스냅샷 정보 직접 조회

```sql
-- 현재 트랜잭션의 스냅샷 확인
BEGIN;
SELECT txid_current_snapshot();
-- 출력 형식: xmin:xmax:xip_list
-- 예: 500:503:501,502

-- 스냅샷 파싱
SELECT
    txid_snapshot_xmin(txid_current_snapshot()) AS snap_xmin,
    txid_snapshot_xmax(txid_current_snapshot()) AS snap_xmax,
    txid_snapshot_xip(txid_current_snapshot()) AS snap_xip;

-- 특정 XID가 스냅샷에서 보이는지 확인
SELECT txid_visible_in_snapshot(500, txid_current_snapshot()) AS is_visible;
COMMIT;
```

### 실험 3: XID Age 모니터링

```sql
-- 데이터베이스별 XID Age
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    CASE
        WHEN age(datfrozenxid) > 1000000000 THEN '🔴 즉시 조치 필요'
        WHEN age(datfrozenxid) > 200000000  THEN '🟡 Autovacuum 점검'
        ELSE '🟢 정상'
    END AS status
FROM pg_database
ORDER BY xid_age DESC;

-- 테이블별 XID Age (오래된 순)
SELECT
    schemaname,
    relname,
    age(relfrozenxid) AS table_xid_age,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(relfrozenxid) DESC
LIMIT 20;

-- 튜플의 xmin/xmax 직접 확인 (pageinspect)
CREATE EXTENSION pageinspect;
SELECT t_xmin, t_xmax, t_ctid,
       age(t_xmin::text::xid) AS xmin_age
FROM heap_page_items(get_raw_page('orders', 0))
WHERE t_xmin IS NOT NULL
ORDER BY t_xmin;
```

### 실험 4: VACUUM FREEZE 효과 관찰

```sql
-- Freeze 전 테이블 XID Age
SELECT relname, age(relfrozenxid) FROM pg_class WHERE relname = 'orders';

-- VACUUM FREEZE 실행
VACUUM FREEZE VERBOSE orders;

-- Freeze 후 XID Age (크게 감소)
SELECT relname, age(relfrozenxid) FROM pg_class WHERE relname = 'orders';

-- Freeze된 튜플 확인 (t_infomask에 HEAP_XMIN_FROZEN 비트)
SELECT t_xmin, t_xmax,
       (t_infomask & 256) > 0 AS is_xmin_frozen  -- 256 = HEAP_XMIN_FROZEN
FROM heap_page_items(get_raw_page('orders', 0))
WHERE t_xmin IS NOT NULL;
```

---

## 📊 MySQL과 비교

```
격리 수준별 Phantom Read 비교:

시나리오:
  T1: SELECT count(*) FROM orders WHERE amount > 100; -- 10건
  T2: INSERT INTO orders (amount) VALUES (200); COMMIT;
  T1: SELECT count(*) FROM orders WHERE amount > 100; -- ?

REPEATABLE READ:
  MySQL:      11건 (Phantom Read 허용 — 넥스트 키 락으로 일부 방지 가능)
  PostgreSQL: 10건 (Phantom Read 방지 — 스냅샷 격리로 자연스럽게)

XID vs MySQL Transaction ID:
  MySQL: 64비트 트랜잭션 ID → 사실상 Wraparound 없음
  PostgreSQL: 32비트 XID → Wraparound 관리 필요

  PostgreSQL이 32비트를 유지하는 이유:
    모든 튜플 헤더에 t_xmin, t_xmax가 있음
    64비트로 바꾸면 튜플 헤더 크기 증가 → 저장 공간 증가
    VACUUM Freeze로 문제를 회피하는 방식 선택

Long-running Transaction 영향:
  MySQL: 오래된 트랜잭션 → Undo Log 증가 (Purge 차단)
  PostgreSQL: 오래된 트랜잭션 → backend_xmin이 낮아짐
              → VACUUM이 Dead Tuple 회수 못함 (OldestXmin 차단)
              → XID Freeze도 불가 → Wraparound 위험 가속
```

---

## ⚖️ 트레이드오프

```
스냅샷 격리의 장단점:

장점:
  ① 읽기가 쓰기를 차단하지 않음:
     읽기 트랜잭션은 잠금 없이 일관된 버전 읽기
     → 높은 동시성

  ② 일관된 읽기 보장:
     REPEATABLE READ에서 Phantom Read 자동 방지
     MySQL의 넥스트 키 락 없이도 보장

  ③ SERIALIZABLE SSI:
     잠금 기반 직렬화보다 동시성 높음
     충돌이 감지될 때만 오류 → 낙관적 접근

단점:
  ① Dead Tuple 누적:
     MVCC를 위해 이전 버전 보존 → Table Bloat
     스냅샷이 살아있는 한 VACUUM이 청소 못함

  ② XID 소모:
     DML이 있는 트랜잭션마다 XID 소모
     → Wraparound 모니터링 필수

  ③ Long-running Transaction 부작용:
     장기 트랜잭션이 OldestXmin을 낮춤
     → VACUUM 차단 + Freeze 차단
     → 운영 중 장기 트랜잭션 모니터링 필요

실무 권장:
  idle_in_transaction_session_timeout 설정:
    SET idle_in_transaction_session_timeout = '5min';
    → BEGIN 후 5분 이상 idle인 트랜잭션 자동 종료
    → Wraparound 위험과 VACUUM 차단 방지
```

---

## 📌 핵심 정리

```
XID와 가시성 핵심:

XID 구조:
  32비트 순차 증가 (1~4,294,967,295)
  DML 트랜잭션만 소모
  특수값: 0=Invalid, 1=Bootstrap, 2=Frozen

스냅샷 구성:
  xmin: 이 값 미만의 XID는 모두 완료됨
  xmax: 이 값 이상의 XID는 아직 없음
  xip: [xmin, xmax) 사이의 활성 트랜잭션 목록

튜플 가시성:
  xmin 커밋됨 AND (xmax=0 OR xmax 진행 중 OR xmax 롤백) → 보임
  그 외 → 보이지 않음

격리 수준과 스냅샷:
  READ COMMITTED: 문장마다 스냅샷 갱신
  REPEATABLE READ: 트랜잭션 첫 문장에서 고정
  SERIALIZABLE: SSI로 직렬화 불가 패턴 감지 시 오류

XID Wraparound:
  age(datfrozenxid) > 200M → 점검
  age > 1B → 즉시 VACUUM FREEZE
  age > 2.1B → PostgreSQL 강제 종료!
  VACUUM Freeze: 오래된 xmin을 FrozenXID(2)로 변환
```

---

## 🤔 생각해볼 문제

**Q1.** 읽기 전용 트랜잭션(`BEGIN; SELECT ...; COMMIT;`)은 XID를 소모하는가? 장기 읽기 트랜잭션이 Wraparound에 어떤 영향을 미치는가?

<details>
<summary>해설 보기</summary>

읽기 전용 트랜잭션은 **Real XID를 소모하지 않습니다**. Virtual Transaction ID(Backend ID + Local Counter)만 사용합니다. 따라서 읽기만 하는 트랜잭션 자체는 XID 소모에 기여하지 않습니다.

그러나 장기 읽기 트랜잭션은 **Wraparound에 간접적으로 영향**을 미칩니다. 해당 트랜잭션의 `backend_xmin`이 낮게 유지되어 OldestXmin이 앞으로 나아가지 못합니다. 결과적으로:
- VACUUM이 Dead Tuple을 회수하지 못함
- VACUUM FREEZE가 오래된 튜플을 Freeze하지 못함
- `age(datfrozenxid)`가 증가하지만 VACUUM이 줄일 수 없음

따라서 Wraparound 위험은 DML 트랜잭션의 빈도뿐만 아니라 **장기 읽기 트랜잭션 여부**도 포함해서 모니터링해야 합니다.

```sql
-- 장기 트랜잭션 + xmin이 오래된 것 찾기
SELECT pid, usename, xact_start, backend_xmin,
       age(backend_xmin) AS xmin_age,
       state
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC;
```

</details>

---

**Q2.** PostgreSQL의 `SERIALIZABLE` 격리 수준은 MySQL과 어떻게 다르고, 직렬화 오류가 발생하면 애플리케이션이 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

**MySQL SERIALIZABLE**: 모든 `SELECT`에 암묵적으로 `FOR SHARE` 잠금을 추가합니다. 강한 잠금 기반으로 동시성이 크게 떨어질 수 있습니다.

**PostgreSQL SERIALIZABLE (SSI)**: Serializable Snapshot Isolation을 사용합니다. 잠금 없이 읽기를 수행하되, 트랜잭션 간의 **읽기-쓰기 의존성**을 추적합니다. 직렬화 불가능한 사이클이 감지되면 오류를 반환합니다.

```
오류: ERROR: could not serialize access due to read/write dependencies among transactions
DETAIL: Reason code: Canceled on identification as a pivot, during commit attempt.
HINT: The transaction might succeed if retried.
```

애플리케이션 처리:
```java
// Spring @Transactional + 재시도 로직
@Transactional(isolation = Isolation.SERIALIZABLE)
public void criticalOperation() {
    // 비즈니스 로직
}

// 재시도 로직 (예: @Retryable)
@Retryable(value = {SQLException.class},
           maxAttempts = 3,
           backoff = @Backoff(delay = 100))
public void retryableSerializableOperation() {
    criticalOperation();
}
```

PostgreSQL SERIALIZABLE은 잠금 기반보다 동시성이 높지만 재시도 처리가 필수입니다.

</details>

---

**Q3.** `pg_stat_activity`에서 `backend_xmin`이 매우 낮은 값으로 고정되어 있다면 어떤 문제가 발생하고 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

`backend_xmin`이 매우 낮다는 것은 해당 세션이 오래된 스냅샷을 유지 중임을 의미합니다. 이로 인해:

1. **VACUUM 차단**: OldestXmin = min(all backend_xmin)이 낮아져 VACUUM이 Dead Tuple을 회수하지 못함
2. **Freeze 차단**: 오래된 튜플의 xmin이 Freeze되지 못함
3. **Table Bloat**: Dead Tuple이 계속 쌓임

진단:
```sql
-- 문제 세션 찾기
SELECT pid, usename, application_name, state,
       xact_start, query_start,
       age(backend_xmin) AS xmin_age,
       left(query, 80) AS query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC
LIMIT 5;
```

해결:
1. 애플리케이션 수준: `idle_in_transaction_session_timeout` 설정으로 자동 종료
2. 수동 종료: `SELECT pg_terminate_backend(pid);`
3. 예방: 커넥션 풀에서 idle 트랜잭션 감지 로직 추가

```sql
-- idle 설정 (postgresql.conf 또는 ALTER DATABASE)
ALTER DATABASE mydb SET idle_in_transaction_session_timeout = '5min';
```

</details>

---

<div align="center">

**[⬅️ 이전: WAL 완전 분해](./04-wal-complete.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Dead Tuple 완전 분해 ➡️](../mvcc-vacuum/01-dead-tuple.md)**

</div>
