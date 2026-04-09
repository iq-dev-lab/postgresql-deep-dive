# VACUUM FULL vs VACUUM — 언제 무엇을 써야 하는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- VACUUM과 VACUUM FULL의 내부 동작 차이는 무엇인가?
- VACUUM FULL이 운영 시간대에 위험한 이유는?
- `pg_repack`은 VACUUM FULL과 어떻게 다른가?
- 어떤 상황에서 VACUUM FULL이 정말 필요한가?
- `CLUSTER` 명령어는 VACUUM FULL과 무엇이 다른가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

"테이블이 너무 크니까 VACUUM FULL을 실행하자"는 생각은 운영 환경에서 서비스 장애로 이어질 수 있다. VACUUM FULL은 테이블 전체를 새로 작성하는 동안 `AccessExclusiveLock`을 유지하기 때문이다. 이 잠금이 걸린 동안 해당 테이블에 대한 모든 SELECT, INSERT, UPDATE, DELETE가 차단된다. 반면 일반 VACUUM은 운영 중에도 DML과 동시에 실행 가능하다. 두 명령어의 차이를 이해해야 올바른 운영 전략을 세울 수 있다.

---

## 😱 흔한 실수 (Before — VACUUM FULL을 가볍게 실행)

```
상황: 배치 작업으로 테이블의 80% 데이터 삭제
  "테이블이 100GB였는데 삭제 후에도 100GB 그대로다"
  "VACUUM FULL로 공간 회수하자"

결과:
  VACUUM FULL 시작
  → AccessExclusiveLock 획득
  → 해당 테이블 접근하는 모든 쿼리 차단
  → 웹서비스 타임아웃 발생 (orders 테이블 접근 불가)

  VACUUM FULL이 새 파일 작성 완료까지: 20분
  → 20분간 서비스 장애

또 다른 실수:
  VACUUM FULL 중 디스크 부족 발생
  새 파일 작성 중 디스크가 꽉 참 (원본 + 새 파일 동시 존재)
  → VACUUM FULL 실패 + 데이터 손상 위험

  올바른 이해:
  VACUUM FULL은 원본 크기만큼 추가 디스크 공간 필요
  100GB 테이블 → VACUUM FULL 중 최대 200GB 필요
```

---

## ✨ 올바른 접근 (After — 차이를 이해한 선택)

```
판단 기준:

  일반 VACUUM (기본 선택):
    Dead Tuple 공간을 재사용 가능으로 표시
    테이블 크기 유지, 운영 중 실행 가능
    → 대부분의 경우 충분

  VACUUM FULL (드물게 사용):
    실제 파일 크기 축소 필요 + 운영 중단 가능한 시간대
    → 정기 점검 시간, 트래픽이 거의 없는 새벽 시간대

  pg_repack (운영 중 VACUUM FULL 대안):
    VACUUM FULL과 동일한 효과 (파일 크기 축소)
    AccessExclusiveLock을 마지막 단계(수 초)만 유지
    운영 중 실행 가능
    → 설치: CREATE EXTENSION pg_repack;

  CLUSTER (인덱스 순서 재정렬 + 크기 축소):
    테이블을 특정 인덱스 순서로 물리적 재정렬
    VACUUM FULL과 유사한 잠금
    Clustered Index가 없는 PostgreSQL에서 읽기 순서 최적화 가능
```

---

## 🔬 내부 동작 원리

### 1. VACUUM (일반) 내부 동작

```
VACUUM 실행 시:

잠금: ShareUpdateExclusiveLock
  → SELECT, INSERT, UPDATE, DELETE 동시 가능
  → ALTER TABLE, VACUUM FULL은 차단됨

처리 방식:
  ① 페이지 스캔: Dead Tuple 식별
  ② Dead Tuple 마킹: ItemId를 LP_DEAD로 표시
  ③ 페이지 내 컴팩션: 살아있는 튜플을 페이지 앞으로 당김
  ④ FSM/VM 업데이트

페이지 파일 변화:
  Before:
  ┌──────────────────────────────────┐
  │  Page 0: [Live][Dead][Live][Dead] │
  │  Page 1: [Dead][Dead][Dead][Dead] │
  │  Page 2: [Live][Live][Live][Live] │
  └──────────────────────────────────┘
  파일 크기: 3 pages × 8KB = 24KB

  After VACUUM:
  ┌──────────────────────────────────┐
  │  Page 0: [Live][Live][Free][Free] │ ← Dead 제거, 여유공간 확보
  │  Page 1: [Free][Free][Free][Free] │ ← 완전히 비워짐 (마지막 페이지면 자름)
  │  Page 2: [Live][Live][Live][Live] │ ← 변화 없음
  └──────────────────────────────────┘
  파일 크기: 3 pages × 8KB = 24KB (그대로)
  BUT: Page 0, 1의 공간은 재사용 가능 (FSM에 등록)

예외: 파일 끝 자르기 (Tail Truncation)
  마지막 페이지들이 완전히 비어있으면 파일에서 제거 가능
  → 테이블 파일이 약간 줄어들 수 있음 (드물게 발생)
```

### 2. VACUUM FULL 내부 동작

```
VACUUM FULL 실행 시:

잠금: AccessExclusiveLock
  → 모든 접근 차단 (SELECT 포함!)
  → 잠금 완료까지: 기존 트랜잭션 완료 대기

처리 방식 (완전히 다른 알고리즘):
  ① 새 파일(heap file) 생성
  ② Live Tuple만 새 파일에 순서대로 복사
  ③ 모든 인덱스 재생성 (DROP + CREATE)
  ④ 원본 파일 삭제
  ⑤ 새 파일로 교체

파일 변화:
  Before:
  ┌──────────────────────────────────┐
  │  orders_16384 (100GB)            │
  │  [Live][Dead][Live][Dead]...     │
  └──────────────────────────────────┘

  VACUUM FULL 중 (임시 상태):
  ┌──────────────────────────────────┐
  │  orders_16384 (100GB) - 원본     │ ← 잠금 중
  │  orders_16384_new (20GB) - 진행  │ ← Live만 복사 중
  └──────────────────────────────────┘
  → 총 120GB 필요!

  After:
  ┌──────────────────────────────────┐
  │  orders_16384 (20GB) - 새 파일   │ ← 원본 삭제, 교체 완료
  └──────────────────────────────────┘

VACUUM FULL 위험 요소:
  ① 추가 디스크 공간: 원본 크기만큼 필요
  ② 인덱스 전체 재생성: 대형 인덱스면 오랜 시간
  ③ 모든 접근 차단: 테이블 크기 × 처리 속도 시간 동안 잠금
  ④ 완료 후 통계 초기화: ANALYZE 필요
```

### 3. CLUSTER 명령어

```
CLUSTER orders USING orders_created_at_idx;

동작:
  VACUUM FULL과 동일한 재작성 방식
  BUT: 데이터를 특정 인덱스 순서로 정렬

잠금: AccessExclusiveLock (VACUUM FULL과 동일)

장점:
  물리적 정렬 → 범위 쿼리 I/O 최소화
  예: created_at 범위 쿼리가 순차 I/O로 처리됨

단점:
  VACUUM FULL과 동일한 위험
  정렬 상태는 이후 INSERT로 점차 흐트러짐 (자동 유지 안 됨)
  → 주기적으로 재실행 필요

  SELECT relname, relhasoids, relhaspkey
  FROM pg_class
  WHERE relname = 'orders';
  -- relispopulated: CLUSTER 후 실행됐는지 확인
```

### 4. pg_repack — 운영 중 테이블 재작성

```
pg_repack 작동 방식:

  ① 임시 테이블 생성 (pg_repack.log)
  ② 원본 테이블의 변경을 트리거로 임시 테이블에 기록
     (이 시점부터 DML 가능!)
  ③ 원본 테이블 Live Tuple을 새 테이블로 복사 (시간 소요)
  ④ 복사 중 발생한 변경을 pg_repack.log에서 적용
  ⑤ 매우 짧은 AccessExclusiveLock (수 초) 후 테이블 교체
  ⑥ 임시 구조 삭제

VACUUM FULL vs pg_repack:
  항목          | VACUUM FULL           | pg_repack
  ─────────────┼───────────────────────┼────────────────────────
  잠금 시간      | 전체 재작성 시간        | 수 초 (마지막 교체만)
  운영 중 사용   | 불가                  | 가능
  DML 허용      | 불가                  | 가능 (복사 중에도)
  추가 공간      | 원본 크기              | 원본 크기 (동일)
  인덱스        | 전체 재생성             | 전체 재생성
  순서 정렬      | 불가                  | 가능 (--order-by 옵션)

설치 및 사용:
  -- 확장 설치
  CREATE EXTENSION pg_repack;

  -- 사용 (psql 외부에서)
  pg_repack -t orders mydb
  pg_repack --table=orders --jobs=2 mydb  -- 병렬 인덱스 재생성
```

---

## 💻 실전 실험

### 실험 1: VACUUM vs VACUUM FULL 크기 비교

```sql
-- 테이블 준비
CREATE TABLE size_test (
    id SERIAL PRIMARY KEY,
    data TEXT DEFAULT repeat('x', 200)
);

INSERT INTO size_test (data) SELECT repeat('x', 200)
FROM generate_series(1, 100000);

SELECT pg_size_pretty(pg_relation_size('size_test')) AS initial_size;

-- 90% 삭제
DELETE FROM size_test WHERE id % 10 != 0;

SELECT pg_size_pretty(pg_relation_size('size_test')) AS after_delete_size;
-- 크기: 그대로 (Dead Tuple)

-- VACUUM
VACUUM size_test;
SELECT pg_size_pretty(pg_relation_size('size_test')) AS after_vacuum_size;
-- 크기: 그대로 (재사용 가능 표시만)

-- VACUUM FULL (테스트 환경에서만)
VACUUM FULL size_test;
SELECT pg_size_pretty(pg_relation_size('size_test')) AS after_vacuum_full_size;
-- 크기: 대폭 감소 (Live Tuple만 남음)
```

### 실험 2: VACUUM FULL 잠금 확인

```sql
-- 터미널 1: 장기 쿼리 실행
BEGIN;
SELECT count(*) FROM size_test;
-- 이 상태 유지

-- 터미널 2: VACUUM FULL 시도
VACUUM FULL size_test;
-- 터미널 1이 완료될 때까지 대기 (잠금 경합!)

-- 터미널 3: pg_locks 확인
SELECT
    l.pid,
    l.locktype,
    l.mode,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.relation = 'size_test'::regclass;
-- AccessExclusiveLock 대기 중임을 확인

-- 터미널 1: 트랜잭션 완료
COMMIT;
-- 이후 터미널 2의 VACUUM FULL 진행됨
```

### 실험 3: VACUUM FULL과 디스크 공간

```bash
# 100GB 테이블이 있을 때 VACUUM FULL 전 디스크 확인
df -h /var/lib/postgresql/

# VACUUM FULL 시 최대 2배 공간 필요
# 테이블 크기 확인
psql -c "SELECT pg_size_pretty(pg_total_relation_size('large_table'));"

# VACUUM FULL 실행 (새벽 점검 시간대에만)
# psql -c "VACUUM FULL ANALYZE large_table;"

# 대안: pg_repack (운영 중 사용 가능)
# pg_repack -t large_table mydb
```

---

## 📊 MySQL과 비교

```
테이블 재구성 방법 비교:

PostgreSQL VACUUM FULL:
  내부: 새 파일 생성 + Live 복사 + 파일 교체
  잠금: AccessExclusiveLock (전체 재작성 시간)
  운영 중: 불가

MySQL OPTIMIZE TABLE:
  InnoDB: ALTER TABLE ... ENGINE=InnoDB와 동일
  내부: 새 테이블 생성 + 복사 + 교체 (VACUUM FULL과 유사)
  잠금: MySQL 5.6 이상 Online DDL 지원 (일부 경우 잠금 최소화)
  Undo 공간 정리도 포함

MySQL Online DDL:
  ALTER TABLE ... ALGORITHM=INPLACE → 잠금 최소화
  복사 중 DML 허용 (pg_repack과 유사한 트리거 방식)

결론:
  PostgreSQL: VACUUM FULL = 잠금, pg_repack = Online
  MySQL: OPTIMIZE TABLE = 잠금, Online DDL = 일부 Online
  둘 다 "운영 중 테이블 재구성"을 위한 추가 도구 필요
```

---

## ⚖️ 트레이드오프

```
VACUUM (일반):
  장점: 운영 중 실행 가능, 잠금 없음
  단점: 파일 크기 유지, 재사용만 가능

VACUUM FULL:
  장점: 파일 크기 실제 축소, 인덱스 재구성
  단점: 운영 중 전체 차단, 추가 디스크 2배 필요

pg_repack:
  장점: 운영 중 실행 가능, 파일 축소, 정렬 옵션
  단점: 별도 설치 필요, 추가 디스크 필요, 복잡도

선택 기준:
  ① 정기적인 Dead Tuple 관리: Autovacuum으로 자동 처리
  ② 대량 DELETE 후 공간 회수 + 서비스 중단 가능: VACUUM FULL
  ③ 대량 DELETE 후 공간 회수 + 서비스 중단 불가: pg_repack
  ④ 읽기 성능 최적화 (물리적 정렬): CLUSTER (서비스 중단) 또는 pg_repack --order-by
```

---

## 📌 핵심 정리

```
VACUUM vs VACUUM FULL:

VACUUM:
  잠금: ShareUpdateExclusiveLock (DML 동시 가능)
  결과: Dead Tuple 공간 재사용 가능 표시
  파일 크기: 유지 (끝 빈 페이지만 제거 가능)
  사용 시기: 일상적 Dead Tuple 관리

VACUUM FULL:
  잠금: AccessExclusiveLock (모든 접근 차단)
  결과: Live Tuple만 새 파일에 복사 → 크기 축소
  추가 공간: 원본 크기만큼 필요
  사용 시기: 정기 점검 시간대, 디스크 공간 실제로 필요할 때

pg_repack:
  잠금: 마지막 교체 단계만 수 초
  결과: VACUUM FULL과 동일한 파일 크기 축소
  추가 공간: 원본 크기만큼 필요
  사용 시기: 운영 중 테이블 재구성이 필요할 때

원칙:
  운영 환경에서는 VACUUM FULL 최대한 자제
  대부분은 Autovacuum이 처리
  파일 축소가 꼭 필요하다면 pg_repack
```

---

## 🤔 생각해볼 문제

**Q1.** VACUUM FULL 실행 중 다른 세션에서 `SELECT * FROM orders LIMIT 1`을 실행하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

VACUUM FULL이 `AccessExclusiveLock`을 보유하고 있으므로, `SELECT`조차 **차단**됩니다. PostgreSQL의 잠금 계층에서 `AccessShareLock`(SELECT가 획득)은 `AccessExclusiveLock`과 호환되지 않습니다. VACUUM FULL이 완료될 때까지 SELECT가 대기합니다.

이것이 VACUUM FULL이 운영 시간대에 위험한 핵심 이유입니다. 조회 쿼리가 많은 서비스에서 VACUUM FULL을 실행하면 해당 테이블에 대한 모든 쿼리가 블로킹됩니다.

```sql
-- VACUUM FULL 대기 중인 쿼리 확인
SELECT pid, query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
  AND wait_event = 'relation';
```

</details>

---

**Q2.** VACUUM FULL 없이 테이블 크기가 줄어드는 경우가 있는가?

<details>
<summary>해설 보기</summary>

일반 VACUUM도 **파일 끝이 빈 페이지인 경우** 파일을 자릅니다(truncation). 테이블의 마지막 페이지들이 완전히 비어있으면 VACUUM이 해당 페이지를 파일에서 제거하여 크기가 약간 줄어들 수 있습니다.

이것은 드문 경우로, 데이터가 테이블 끝에서부터 순서대로 삭제되는 경우에 발생합니다. 예를 들어 파티션 테이블에서 오래된 파티션을 삭제하면 파티션 파일 자체가 제거되어 공간이 즉시 회수됩니다.

일반적으로 삭제는 랜덤하게 발생하므로 Dead Tuple이 전체 파일에 흩어져 있고, VACUUM이 파일 끝을 자를 수 없습니다. 이때는 VACUUM FULL 또는 pg_repack이 필요합니다.

</details>

---

**Q3.** `pg_repack`이 VACUUM FULL과 동일한 효과를 얻으면서도 잠금을 최소화하는 원리는?

<details>
<summary>해설 보기</summary>

pg_repack은 3단계 접근 방식을 사용합니다:

1. **트리거 설치**: 원본 테이블에 트리거를 추가해 이후 모든 INSERT/UPDATE/DELETE를 `pg_repack.log` 임시 테이블에 기록합니다. 이 시점부터 DML은 계속 허용됩니다.

2. **비동기 복사**: 원본 테이블의 Live Tuple을 새 테이블로 복사합니다(AccessShareLock만 사용). 이 과정이 가장 오래 걸리지만 DML은 계속 가능합니다.

3. **델타 적용 + 교체**: 복사 중 쌓인 `pg_repack.log`의 변경사항을 적용한 뒤, 매우 짧은 시간(수 초) 동안 AccessExclusiveLock을 획득해 원본과 새 테이블을 교체합니다.

이 방식의 핵심은 "무거운 작업(데이터 복사)을 잠금 없이 하고, 가벼운 작업(교체)만 잠금과 함께 한다"는 것입니다. 단, pg_repack도 원본 테이블 크기만큼 추가 디스크 공간이 필요합니다.

</details>

---

<div align="center">

**[⬅️ 이전: VACUUM 내부 동작](./02-vacuum-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Autovacuum 튜닝 ➡️](./04-autovacuum-tuning.md)**

</div>
