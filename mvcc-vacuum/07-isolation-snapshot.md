# 격리 수준과 스냅샷 — Snapshot Isolation과 SSI

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PostgreSQL의 4가지 격리 수준은 내부적으로 어떻게 구현되는가?
- PostgreSQL REPEATABLE READ가 MySQL REPEATABLE READ보다 강한 이유는?
- Serializable Snapshot Isolation(SSI)은 직렬화를 어떻게 낙관적으로 보장하는가?
- 각 격리 수준의 성능과 동시성 트레이드오프는?
- Spring `@Transactional(isolation=...)` 설정이 실제로 어떻게 동작하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

트랜잭션 격리 수준은 "동시에 실행되는 트랜잭션이 서로 어디까지 영향을 주는가"를 결정한다. 너무 낮은 격리 수준은 데이터 이상 현상(Phantom Read, Non-Repeatable Read)을 허용하고, 너무 높은 격리 수준은 동시성을 떨어뜨린다. PostgreSQL은 SQL 표준의 4가지 격리 수준을 지원하지만, 내부 구현은 MySQL과 다르다. 특히 REPEATABLE READ에서 Phantom Read를 자동으로 방지하고, SERIALIZABLE에서 잠금 없이 직렬화를 보장하는 방식이 다르다.

---

## 😱 흔한 실수 (Before — 격리 수준 개념 혼동)

```
실수 1: "READ UNCOMMITTED로 Dirty Read 성능 향상"

  SET default_transaction_isolation = 'READ UNCOMMITTED';
  → PostgreSQL: READ COMMITTED로 동작 (Dirty Read 구현 자체 없음)
  → 격리 수준 변경이 의도한 효과 없음

  올바른 이해:
  PostgreSQL은 MVCC로 인해 커밋되지 않은 데이터를 읽는 것 자체가 불가
  READ UNCOMMITTED = READ COMMITTED로 동작

실수 2: SERIALIZABLE이 항상 안전할 것이라 믿고 재시도 로직 미구현

  @Transactional(isolation = Isolation.SERIALIZABLE)
  public void transfer(Long from, Long to, BigDecimal amount) {
      // 계좌 잔액 확인 및 이체
  }

  직렬화 충돌 시 오류 발생:
  ERROR: could not serialize access due to read/write dependencies

  재시도 없으면: 사용자에게 오류 노출
  → SERIALIZABLE 사용 시 반드시 재시도 로직 필요

실수 3: MySQL REPEATABLE READ 경험으로 PostgreSQL 동작 예상

  MySQL REPEATABLE READ: Phantom Read 허용 (Next-Key Lock으로 일부 방지)
  PostgreSQL REPEATABLE READ: Phantom Read 방지 (스냅샷으로 자연스럽게)
  → "PostgreSQL에서도 Phantom Read가 발생하겠지" → 틀린 가정
```

---

## ✨ 올바른 접근 (After — 격리 수준 이해 기반 선택)

```
격리 수준 선택 전략:

  READ COMMITTED (기본값):
    적합: 대부분의 OLTP 서비스
          항상 최신 커밋 데이터가 필요한 경우
    특징: 각 문장마다 스냅샷 갱신
    주의: 같은 트랜잭션 내에서 같은 쿼리가 다른 결과 반환 가능

  REPEATABLE READ:
    적합: 일관된 읽기가 필요한 보고서/배치
          SELECT 결과를 기반으로 여러 UPDATE가 있는 경우
    특징: 트랜잭션 시작 시 스냅샷 고정
          Phantom Read 자동 방지 (MySQL보다 강함)

  SERIALIZABLE:
    적합: 복잡한 비즈니스 로직, 잔액 이체 등 정확성 최우선
    특징: SSI로 낙관적 직렬화 보장
    주의: 직렬화 오류 시 재시도 필수

Spring 설정:
  @Transactional  -- 기본: READ COMMITTED
  @Transactional(isolation = Isolation.REPEATABLE_READ)  -- 일관된 읽기
  @Transactional(isolation = Isolation.SERIALIZABLE)     -- 최고 격리
```

---

## 🔬 내부 동작 원리

### 1. 4가지 격리 수준과 SQL 표준 이상 현상

```
SQL 표준 이상 현상:

  Dirty Read: 커밋 안 된 데이터 읽기
  Non-Repeatable Read: 같은 행을 두 번 읽으면 다른 결과
  Phantom Read: 같은 조건으로 두 번 조회하면 행 수가 다름

격리 수준별 허용 이상 현상:

  격리 수준         | Dirty Read | Non-Repeatable | Phantom
  ─────────────────┼────────────┼────────────────┼─────────
  READ UNCOMMITTED  | 허용       | 허용            | 허용
  READ COMMITTED    | 방지       | 허용            | 허용
  REPEATABLE READ   | 방지       | 방지            | 허용(SQL)
  SERIALIZABLE      | 방지       | 방지            | 방지

PostgreSQL 실제 구현:

  격리 수준         | Dirty Read | Non-Repeatable | Phantom
  ─────────────────┼────────────┼────────────────┼─────────
  READ UNCOMMITTED  | 방지(*)    | 허용            | 허용
  READ COMMITTED    | 방지       | 허용            | 허용
  REPEATABLE READ   | 방지       | 방지            | 방지(**)
  SERIALIZABLE      | 방지       | 방지            | 방지

(*): PostgreSQL은 READ UNCOMMITTED를 READ COMMITTED처럼 처리
(**): SQL 표준보다 강함 (스냅샷 격리로 Phantom Read도 방지)
```

### 2. READ COMMITTED — 문장별 스냅샷

```
READ COMMITTED 동작:

  각 SQL 문장 실행 시 새로운 스냅샷 획득
  → 다른 트랜잭션의 커밋이 즉시 반영

예시:
  T1 (READ COMMITTED):   T2:
  BEGIN;                  
  SELECT val FROM t       -- val = 'A'
  WHERE id = 1;           
                          UPDATE t SET val = 'B' WHERE id = 1;
                          COMMIT;
  SELECT val FROM t       -- val = 'B' (T2 커밋 후 새 스냅샷!)
  WHERE id = 1;           
  COMMIT;

장점:
  항상 최신 커밋 데이터 반환
  Lock 대기 없는 읽기 (MVCC)

단점:
  Non-Repeatable Read: 같은 트랜잭션에서 같은 쿼리가 다른 결과
  → UPDATE 기반 로직에서 일관성 깨질 수 있음

Spring JPA 주의사항:
  기본 READ COMMITTED + 영속성 컨텍스트의 1차 캐시
  → EntityManager.find()는 1차 캐시에서 반환 (DB 재조회 안 함)
  → JPQL은 DB에서 읽고 1차 캐시에 병합
  → READ COMMITTED + JPA 1차 캐시 = 예상치 못한 데이터 불일치 가능
```

### 3. REPEATABLE READ — 트랜잭션 스냅샷

```
REPEATABLE READ 동작:

  트랜잭션 첫 번째 문장 실행 시 스냅샷 획득 (이후 고정)

예시:
  T1 (REPEATABLE READ):  T2:
  BEGIN;                  
  SELECT val FROM t       -- val = 'A', 스냅샷 고정
  WHERE id = 1;           
                          UPDATE t SET val = 'B' WHERE id = 1;
                          COMMIT;
  SELECT val FROM t       -- val = 'A' (스냅샷 고정이므로!)
  WHERE id = 1;           
  COMMIT;

PostgreSQL REPEATABLE READ의 특별한 강점:

  Phantom Read 방지:
  T1 (REPEATABLE READ):        T2:
  BEGIN;
  SELECT count(*) FROM orders
  WHERE amount > 100;           -- 10건
                                INSERT INTO orders (amount) VALUES (200);
                                COMMIT;
  SELECT count(*) FROM orders
  WHERE amount > 100;           -- 10건 (Phantom Read 방지!)
  COMMIT;

  이유: 스냅샷이 고정되어 T2의 INSERT가 T1에 보이지 않음

MySQL REPEATABLE READ와 비교:
  MySQL: Next-Key Lock으로 일부 Phantom Read 방지
          잠금 기반 → 동시성 일부 저하
  PostgreSQL: 스냅샷으로 Phantom Read 완전 방지
              잠금 없음 → 동시성 유지

쓰기 충돌 시 (Write-Write Conflict):
  T1: UPDATE t SET val = 'B' WHERE id = 1;
  T2: UPDATE t SET val = 'C' WHERE id = 1; (T1이 커밋된 후)
  → T2: ERROR: could not serialize access due to concurrent update
  → 이미 업데이트된 행을 다시 업데이트하면 오류 (REPEATABLE READ 이상에서)
```

### 4. SERIALIZABLE — SSI 낙관적 직렬화

```
Serializable Snapshot Isolation (SSI):

기본 원리:
  Snapshot Isolation + 읽기-쓰기 의존성 추적
  충돌 감지 시 트랜잭션 중 하나를 롤백 (낙관적)

읽기-쓰기 의존성 (rw-anti-dependency):
  T1이 X를 읽음 → T2가 X를 씀 (T1 read, T2 write: rw 의존)
  T1이 Y를 씀 → T2가 Y를 읽음 (T2 read after T1 write: wr 의존)

직렬화 불가 사이클 감지:
  T1 →rw→ T2 →rw→ T1
  → 순환 의존: 직렬 실행 불가 → 하나 롤백

실제 예시 (직렬화 위반):
  T1: SELECT sum(amount) FROM accounts WHERE type = 'A'; -- 1000
  T2: SELECT sum(amount) FROM accounts WHERE type = 'A'; -- 1000

  T1: INSERT INTO accounts (type, amount) VALUES ('A', 100);  -- 합계를 B에 기록
  T2: INSERT INTO accounts (type, amount) VALUES ('A', 200);  -- 합계를 B에 기록

  두 트랜잭션이 각각 SELECT 후 INSERT하면:
  → T1은 T2의 삽입을 보지 못하고, T2는 T1의 삽입을 보지 못함
  → 실제 합계 1300인데, 둘 다 1100을 기록
  → 데이터 이상!

  SERIALIZABLE에서:
  → SSI가 rw-anti-dependency 사이클 감지
  → 하나의 트랜잭션이 rollback됨:
     ERROR: could not serialize access due to read/write dependencies

SSI 성능 특징:
  MySQL SERIALIZABLE: 모든 SELECT에 LOCK IN SHARE MODE
  PostgreSQL SERIALIZABLE: Lock 없음, 의존성 추적만
  → PostgreSQL이 MySQL보다 동시성 높음 (충돌이 드문 경우)
  → 충돌이 많으면 재시도 overhead 발생

SSI 내부 자료구조:
  pg_serialize_transaction: 각 트랜잭션의 의존성 그래프
  SIREAD Lock: 읽은 데이터를 추적하는 가상 잠금 (실제 잠금 아님)
  → 메모리 오버헤드: max_pred_locks_per_transaction (기본 64)
```

### 5. 격리 수준과 VACUUM의 관계

```
격리 수준이 Dead Tuple 유지에 미치는 영향:

READ COMMITTED:
  각 문장마다 새 스냅샷 → 오래된 스냅샷 빨리 해제
  → VACUUM이 빠르게 Dead Tuple 회수 가능

REPEATABLE READ / SERIALIZABLE:
  트랜잭션 종료까지 스냅샷 고정
  → 장기 REPEATABLE READ 트랜잭션:
     OldestXmin이 낮아짐 → VACUUM 차단
  → 배치 보고서 트랜잭션이 1시간 실행 중:
     그 1시간 동안 생성된 Dead Tuple 회수 불가

해결책:
  배치 보고서: 가능하면 짧은 트랜잭션으로 분리
  또는 READ COMMITTED로 실행 (일관성이 크게 중요하지 않다면)

  idle_in_transaction_session_timeout:
  SET idle_in_transaction_session_timeout = '30min';
  → 트랜잭션 내에서 30분 이상 대기 시 자동 종료
  → Dead Tuple 차단 방지
```

---

## 💻 실전 실험

### 실험 1: 격리 수준별 Non-Repeatable Read 동작 비교

```sql
-- 터미널 1 (READ COMMITTED, 기본)
BEGIN;
SELECT amount FROM accounts WHERE id = 1;  -- 1000

-- 터미널 2:
UPDATE accounts SET amount = 2000 WHERE id = 1;
COMMIT;

-- 터미널 1 (계속):
SELECT amount FROM accounts WHERE id = 1;  -- 2000 (Non-Repeatable Read 발생)
COMMIT;

-- 터미널 1 (REPEATABLE READ):
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT amount FROM accounts WHERE id = 1;  -- 1000

-- 터미널 2:
UPDATE accounts SET amount = 3000 WHERE id = 1;
COMMIT;

-- 터미널 1 (계속):
SELECT amount FROM accounts WHERE id = 1;  -- 1000 (스냅샷 고정!)
COMMIT;
```

### 실험 2: SERIALIZABLE 충돌 재현

```sql
-- 의사 코드 (두 터미널 동시 실행)

-- 터미널 1:
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM orders WHERE status = 'PENDING';

-- 터미널 2:
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM orders WHERE status = 'PENDING';

-- 터미널 1:
INSERT INTO orders (status, amount) VALUES ('PENDING', 100);

-- 터미널 2:
INSERT INTO orders (status, amount) VALUES ('PENDING', 200);

-- 터미널 1:
COMMIT;  -- 성공

-- 터미널 2:
COMMIT;  -- ERROR: could not serialize access due to read/write dependencies
         -- → 재시도 필요
```

### 실험 3: 격리 수준 확인 및 설정

```sql
-- 현재 격리 수준 확인
SHOW transaction_isolation;
SHOW default_transaction_isolation;

-- 세션 기본값 변경
SET default_transaction_isolation = 'REPEATABLE READ';

-- 트랜잭션별 설정
BEGIN ISOLATION LEVEL SERIALIZABLE;
SHOW transaction_isolation;  -- serializable
COMMIT;

-- Spring JPA에서 격리 수준 확인
-- @Transactional(isolation = Isolation.REPEATABLE_READ)
-- → JDBC: conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ)
-- → PostgreSQL: SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
```

---

## 📊 MySQL과 비교

```
격리 수준 구현 비교:

REPEATABLE READ Phantom Read:
  MySQL: 허용 (단, INSERT에 Next-Key Lock으로 일부 방지)
  PostgreSQL: 완전 방지 (스냅샷으로)

  MySQL에서 Phantom Read 발생 시나리오:
  T1: SELECT * FROM t WHERE id > 10;  -- 0건
  T2: INSERT INTO t VALUES (15, ...); COMMIT;
  T1: SELECT * FROM t WHERE id > 10;  -- 1건 (Phantom!)
  → MySQL REPEATABLE READ에서 발생 가능

  PostgreSQL REPEATABLE READ:
  T1: SELECT * FROM t WHERE id > 10;  -- 0건, 스냅샷 고정
  T2: INSERT INTO t VALUES (15, ...); COMMIT;
  T1: SELECT * FROM t WHERE id > 10;  -- 0건 (스냅샷 덕분에 Phantom 없음)

SERIALIZABLE:
  MySQL: 모든 SELECT에 암묵적 LOCK IN SHARE MODE
         → Lock 기반 → 높은 잠금 경합
  PostgreSQL: SSI (잠금 없음, 의존성 추적)
              → 충돌이 드물면 성능 우수
              → 충돌이 많으면 재시도 overhead

결론:
  PostgreSQL REPEATABLE READ ≥ MySQL SERIALIZABLE (Phantom Read 측면)
  PostgreSQL SERIALIZABLE: 잠금 기반보다 동시성 우수
```

---

## ⚖️ 트레이드오프

```
격리 수준별 동시성과 정확성:

  격리 수준       | 동시성  | 정확성 | 재시도 필요
  ───────────────┼────────┼───────┼──────────
  READ COMMITTED  | 높음   | 낮음  | 없음
  REPEATABLE READ | 중간   | 높음  | 쓰기 충돌 시
  SERIALIZABLE    | 낮음   | 최고  | 직렬화 충돌 시

SSI 사용 시 재시도 로직:
  @Retryable(
      value = {org.springframework.dao.CannotSerializeTransactionException.class},
      maxAttempts = 3,
      backoff = @Backoff(delay = 100, multiplier = 2)
  )
  @Transactional(isolation = Isolation.SERIALIZABLE)
  public void criticalOperation() { ... }

  또는 JDBC 수준:
  while (retries-- > 0) {
      try {
          conn.setAutoCommit(false);
          conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
          // 비즈니스 로직
          conn.commit();
          break;
      } catch (PSQLException e) {
          if ("40001".equals(e.getSQLState())) {
              conn.rollback();
              // 재시도
          }
      }
  }
```

---

## 📌 핵심 정리

```
PostgreSQL 격리 수준 핵심:

READ COMMITTED (기본):
  문장마다 새 스냅샷 → 커밋된 최신 데이터 반환
  Non-Repeatable Read 허용

REPEATABLE READ:
  트랜잭션 첫 문장에서 스냅샷 고정
  Phantom Read까지 방지 (SQL 표준보다 강함)
  쓰기 충돌 시 오류

SERIALIZABLE (SSI):
  읽기-쓰기 의존성 추적, 사이클 감지 시 롤백
  잠금 없음 → 동시성 높음
  직렬화 오류 시 재시도 필수

MySQL과 차이:
  REPEATABLE READ: PostgreSQL이 Phantom Read도 방지
  SERIALIZABLE: MySQL은 Lock 기반, PostgreSQL은 SSI(낙관적)

VACUUM과의 관계:
  장기 REPEATABLE READ/SERIALIZABLE 트랜잭션 → Dead Tuple 청소 차단
  idle_in_transaction_session_timeout 설정으로 예방
```

---

## 🤔 생각해볼 문제

**Q1.** PostgreSQL REPEATABLE READ에서 자신이 같은 트랜잭션에서 삽입한 행은 볼 수 있는가?

<details>
<summary>해설 보기</summary>

네, 볼 수 있습니다. 스냅샷 격리는 **다른 트랜잭션**의 변경을 숨기는 것입니다. 자신의 트랜잭션(같은 XID)이 만든 변경은 항상 보입니다.

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
INSERT INTO test VALUES (999, 'mine');
SELECT * FROM test WHERE id = 999;  -- 보임! (자신이 삽입)
COMMIT;
```

PostgreSQL의 가시성 규칙에서 `t_xmin = 현재 트랜잭션 XID`인 경우 항상 가시입니다. 스냅샷의 `xip` 목록에 자기 자신이 포함되어 있어도, 자신의 변경은 특별 처리로 보이도록 합니다.

</details>

---

**Q2.** SERIALIZABLE에서 "write skew" 문제란 무엇이고, SSI는 이를 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

Write Skew는 REPEATABLE READ까지 허용되는 이상 현상입니다:

```
시나리오: 의사 당직 시스템
  T1: SELECT count(*) FROM on_duty; -- 2명 (최소 1명 필요)
  T2: SELECT count(*) FROM on_duty; -- 2명
  T1: DELETE FROM on_duty WHERE doctor = 'A'; -- 1명 남음
  T2: DELETE FROM on_duty WHERE doctor = 'B'; -- 0명 남음!
  → 둘 다 커밋 → 당직 의사 0명 (규칙 위반)
```

SERIALIZABLE SSI 방지:
- T1이 `on_duty`를 읽음 → SIREAD Lock
- T2가 `on_duty`를 읽음 → SIREAD Lock
- T1이 `on_duty` 행 삭제 (T2의 읽기 결과에 영향)
- T2가 `on_duty` 행 삭제 (T1의 읽기 결과에 영향)
- 사이클 감지: T1 →rw→ T2 →rw→ T1
- 하나 롤백: `ERROR: could not serialize access`

REPEATABLE READ에서는 이 문제를 방지하지 못합니다. SERIALIZABLE이 필요한 이유입니다.

</details>

---

<div align="center">

**[⬅️ 이전: XID Wraparound](./06-xid-wraparound.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: B-Tree 인덱스 심화 ➡️](../indexes/01-btree-index.md)**

</div>
