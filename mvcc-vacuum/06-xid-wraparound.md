# XID Wraparound — PostgreSQL을 멈추게 하는 32비트 한계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- XID Wraparound가 왜 데이터 손상을 일으키는가?
- VACUUM Freeze는 Wraparound를 어떻게 예방하는가?
- PostgreSQL은 Wraparound 직전에 어떤 안전 장치를 발동하는가?
- `age(datfrozenxid)`를 어떻게 모니터링하고 어느 수치에서 행동해야 하는가?
- Wraparound 위기 상황을 어떻게 복구하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

XID Wraparound는 PostgreSQL에서 발생할 수 있는 가장 심각한 운영 장애 중 하나다. VACUUM이 장기간 실행되지 않으면(장기 트랜잭션, Autovacuum 비활성화, 레플리케이션 슬롯 정체 등) `age(datfrozenxid)`가 증가한다. 약 21억을 초과하면 PostgreSQL이 **스스로 모든 쓰기 쿼리를 거부**한다. "이 시점을 넘기면 데이터 손상이 발생할 수 있으니 복구를 위해 수동 개입하라"는 의미다. 미리 모니터링하지 않으면 예고 없이 서비스 장애가 발생한다.

---

## 😱 흔한 실수 (Before — 경고 무시)

```
PostgreSQL 로그에 나타나는 경고:
  WARNING: database "production" must be vacuumed within 177009986 transactions
  HINT: To avoid a database shutdown, execute a database-wide VACUUM in "production".

팀의 반응:
  "경고 메시지이니까 나중에 보자"
  → 무시

며칠 후:
  ERROR: database is not accepting commands to avoid wraparound data loss
         in database "production"
  HINT: Stop the postmaster and vacuum that database in single-user mode.

  → 모든 INSERT, UPDATE, DELETE 거부!
  → SELECT는 가능
  → 즉각적 서비스 장애

복구 과정:
  1. PostgreSQL 중지
  2. 단일 사용자 모드로 시작:
     postgres --single -D /var/lib/postgresql/data production
  3. VACUUM FREEZE 실행 (수 시간 소요)
  4. PostgreSQL 재시작

  → 서비스 다운 수 시간

Wraparound를 유발한 원인들:
  ① 장기 트랜잭션: BEGIN 후 수십 시간 방치 → Autovacuum Freeze 차단
  ② Replication Slot 정체: Standby가 오래 오프라인 → WAL + Freeze 차단
  ③ Autovacuum 비활성화: autovacuum=off 설정 방치
  ④ 과도한 Lock: VACUUM FREEZE가 필요한데 테이블 Lock으로 실행 못함
```

---

## ✨ 올바른 접근 (After — 사전 모니터링과 예방)

```
XID Age 모니터링 (Prometheus Alert 등에 추가):

  -- DB 레벨 모니터링
  SELECT
      datname,
      age(datfrozenxid) AS xid_age,
      CASE
          WHEN age(datfrozenxid) > 1500000000 THEN '🔴 CRITICAL: 즉시 조치'
          WHEN age(datfrozenxid) > 1000000000 THEN '🟠 WARNING: 수동 VACUUM 실행'
          WHEN age(datfrozenxid) > 500000000  THEN '🟡 NOTICE: Autovacuum 점검'
          ELSE '🟢 정상'
      END AS status
  FROM pg_database
  ORDER BY age(datfrozenxid) DESC;

  -- 테이블 레벨 모니터링
  SELECT
      n.nspname, c.relname,
      age(c.relfrozenxid) AS table_xid_age,
      pg_size_pretty(pg_total_relation_size(c.oid)) AS size
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND age(c.relfrozenxid) > 500000000
  ORDER BY age(c.relfrozenxid) DESC;

  임계값 행동 지침:
  age > 200M: Autovacuum이 제때 실행되는지 확인
  age > 500M: 수동 VACUUM FREEZE 스케줄링
  age > 1B:   즉시 수동 VACUUM FREEZE 실행
  age > 1.5B: CRITICAL - 즉시 조치 없으면 DB 자동 쓰기 거부 임박
```

---

## 🔬 내부 동작 원리

### 1. 32비트 XID와 Wraparound 원리

```
XID 범위: 0 ~ 4,294,967,295 (32비트)

PostgreSQL의 XID 비교 방식 (모듈 산술):
  32비트 공간을 "과거 20억" + "미래 20억"으로 분할
  현재 XID로부터 ±2,147,483,648(21억)이 "인식 가능한 범위"

정상 상태 (현재 XID = 3,000,000,000):
  3,000,000,000 - 2,147,483,648 = 852,516,352 이전은 "과거"
  3,000,000,000 + 2,147,483,648 = 5,147,483,648 → 오버플로 → 852,516,352 이후는 "미래"
  
  XID = 1,000,000,000 → "20억 이전" → 과거 → 가시
  XID = 3,500,000,000 → "5억 미래" → 아직 없음 → 불가시

Wraparound 발생 (XID가 최대치를 넘어 1로 돌아옴):
  현재 XID: 4,294,967,295 (최대)
  다음 트랜잭션: XID = 3 (0, 1, 2는 특수값)

  이제 XID = 3,000,000,000 는?
  3 + 2,147,483,648 = 2,147,483,651
  3,000,000,000 > 2,147,483,651 → "미래"로 인식!
  
  → XID=3,000,000,000 트랜잭션이 삽입한 튜플이 "미래에 생성된 것"으로 인식
  → 모든 현재 트랜잭션에게 보이지 않음!
  → 데이터가 사라진 것처럼 보임 → 심각한 데이터 손상
```

### 2. VACUUM Freeze 메커니즘

```
Freeze의 원리:
  오래된 튜플의 t_xmin을 FrozenTransactionId(2)로 교체
  → 어떤 XID와 비교해도 항상 "과거"로 인식
  → Wraparound 영향 없음

Freeze 조건 (VACUUM이 자동으로 적용):
  vacuum_freeze_min_age = 50,000,000 (기본값)
  → 현재 XID - t_xmin > 50,000,000인 튜플을 Freeze

Full Table Freeze (강제 전체 스캔):
  vacuum_freeze_table_age = 150,000,000 (기본값)
  → table의 relfrozenxid의 age > 150,000,000이면
  → VACUUM이 테이블 전체를 스캔해서 Freeze

datfrozenxid:
  pg_database의 datfrozenxid = 해당 DB에서 가장 오래된 미Freeze XID
  → Wraparound 위험도의 실질적 지표

relfrozenxid:
  pg_class의 relfrozenxid = 해당 테이블에서 가장 오래된 미Freeze XID

age(xid):
  현재 XID - xid = XID 나이 (얼마나 오래됐는지)

Freeze 과정:

  Before:
  Tuple: t_xmin = 500, t_xmax = 0
  age(500) = 현재XID - 500 = 50,000,001 → Freeze 대상

  After VACUUM FREEZE:
  Tuple: t_xmin = 2 (FrozenTransactionId)
  → age = "항상 과거" → Wraparound 영향 없음
  → t_infomask에 HEAP_XMIN_FROZEN 비트 설정
```

### 3. PostgreSQL 안전 장치 단계

```
Wraparound 방지 안전 장치 (4단계):

age(datfrozenxid) 기준:

  1단계: Autovacuum 자동 Freeze 실행
    age > autovacuum_freeze_max_age (기본 200,000,000)
    → Autovacuum이 해당 테이블을 강제로 전체 스캔 + Freeze
    → 정상적으로 관리되는 DB는 이 단계에서 처리됨

  2단계: 경고 로그
    age > 1,100,000,000 (약 11억)
    LOG: database "mydb" must be vacuumed within X transactions
    → 로그 모니터링 시 감지하고 수동 VACUUM FREEZE 실행

  3단계: 강제 Autovacuum
    age > 1,400,000,000 (약 14억)
    → Autovacuum이 비용 제한 무시하고 최대 속도로 Freeze 시도

  4단계: 쓰기 거부 (안전 종료)
    age > 2,100,000,000 (약 21억)
    ERROR: database is not accepting commands to avoid wraparound
    → INSERT, UPDATE, DELETE 모두 거부
    → SELECT는 허용
    → 단일 사용자 모드에서 VACUUM FREEZE 실행해야 복구

  복구 절차 (4단계 도달 시):
    1. 서비스 중단
    2. postgresql.conf에서 max_connections = 1 임시 설정 (선택)
    3. 단일 사용자 모드: postgres --single -D $PGDATA mydb
    4. 내부에서: VACUUM FREEZE;  -- 시간 소요
    5. 정상 재시작
```

### 4. Wraparound를 유발하는 원인들

```
원인 1: 장기 트랜잭션
  BEGIN; (XID = 1000)
  -- 수십 시간 방치

  VACUUM FREEZE는 OldestXmin 이전 튜플만 Freeze 가능
  OldestXmin = min(모든 활성 트랜잭션의 xmin) = 1000
  → XID >= 1000인 튜플은 Freeze 불가
  → age가 계속 증가

  해결: idle_in_transaction_session_timeout 설정
    SET idle_in_transaction_session_timeout = '30min';

원인 2: Replication Slot 정체
  pg_replication_slots에 inactive slot이 있는 경우
  → Slot의 restart_lsn이 낮으면 WAL 삭제 못함
  → Slot의 catalog_xmin이 낮으면 pg_catalog 테이블 VACUUM Freeze 불가
  → datfrozenxid 증가

  확인:
  SELECT slot_name, active, restart_lsn,
         age(catalog_xmin) AS catalog_xmin_age
  FROM pg_replication_slots
  ORDER BY age(catalog_xmin) DESC;

  미사용 Slot 삭제:
  SELECT pg_drop_replication_slot('old_slot_name');

원인 3: Autovacuum 비활성화
  autovacuum = off (postgresql.conf)
  또는 특정 테이블: autovacuum_enabled = false
  → VACUUM이 실행 안 됨 → age 무한 증가

  해결: Autovacuum 활성화 또는 정기적 수동 VACUUM FREEZE 스케줄링

원인 4: VACUUM FREEZE 실행 실패
  테이블에 Lock이 걸려 VACUUM이 대기
  → 오랫동안 Freeze 못함

  확인:
  SELECT pid, query, wait_event_type, wait_event
  FROM pg_stat_activity WHERE query LIKE 'autovacuum%';
```

---

## 💻 실전 실험

### 실험 1: XID Age 모니터링 쿼리 세트

```sql
-- 1. DB 레벨 XID Age 전체 확인
SELECT
    datname,
    datfrozenxid,
    age(datfrozenxid) AS xid_age,
    pg_size_pretty(
        pg_database_size(datname)
    ) AS db_size
FROM pg_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY age(datfrozenxid) DESC;

-- 2. 가장 오래된 XID를 가진 테이블 TOP 20
SELECT
    n.nspname || '.' || c.relname AS full_table_name,
    age(c.relfrozenxid) AS table_age,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS size,
    last_autovacuum,
    last_autoanalyze
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_stat_user_tables t ON t.relid = c.oid
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;

-- 3. Replication Slot이 Freeze를 막는지 확인
SELECT
    slot_name,
    slot_type,
    active,
    age(catalog_xmin) AS catalog_xmin_age,
    age(xmin) AS xmin_age,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal
FROM pg_replication_slots
ORDER BY age(catalog_xmin) DESC;

-- 4. 장기 트랜잭션 확인
SELECT
    pid,
    usename,
    age(backend_xmin) AS xmin_age,
    now() - xact_start AS duration,
    state,
    left(query, 80) AS query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC
LIMIT 10;
```

### 실험 2: VACUUM FREEZE 수동 실행

```sql
-- 특정 테이블 Freeze
VACUUM FREEZE VERBOSE orders;

-- VERBOSE 출력 예시:
-- INFO: aggressively vacuuming "public.orders"
-- INFO: scanned index "orders_pkey" to remove 0 row versions
-- INFO: "orders": found 0 removable, 10000 nonremovable row versions
-- INFO: "orders": page count: 200, estimated free space: 0 bytes
-- INFO: "orders": frozen 10000 tuples

-- 전체 DB Freeze (주의: 오래 걸림)
-- VACUUM FREEZE;

-- Freeze 후 age 확인
SELECT relname, age(relfrozenxid) FROM pg_class WHERE relname = 'orders';
-- age가 크게 감소

-- Freeze 상태 확인 (pageinspect)
CREATE EXTENSION pageinspect;
SELECT
    t_xmin,
    (t_infomask & 256) > 0 AS xmin_frozen,  -- HEAP_XMIN_FROZEN
    t_xmax
FROM heap_page_items(get_raw_page('orders', 0))
WHERE t_xmin IS NOT NULL
LIMIT 5;
```

---

## 📊 MySQL과 비교

```
MySQL vs PostgreSQL: 트랜잭션 ID 관리

MySQL:
  트랜잭션 ID: 64비트 (8바이트)
  최대값: 2^64 - 1 ≈ 18 × 10^18 (매우 크다)
  Wraparound 걱정: 사실상 없음
  → 초당 100만 트랜잭션에도 584,542년 소요

PostgreSQL:
  트랜잭션 ID: 32비트 (4바이트)
  최대값: ~42억
  Wraparound 위험: 현실적으로 존재
  → 초당 1,000 트랜잭션에도 약 49일이면 절반 소모

왜 PostgreSQL은 32비트를 유지하는가:
  ① 모든 튜플 헤더(HeapTupleHeader)에 t_xmin, t_xmax 포함
     64비트로 바꾸면 튜플당 8바이트 추가 → 테이블 크기 증가
  ② 기존 데이터 파일과 호환성 유지
  ③ VACUUM Freeze로 Wraparound 예방 가능

Epoch-based 확장:
  PostgreSQL 내부에서 "epoch" 필드로 실질적 64비트 사용 가능 (txid_snapshot 등)
  하지만 Heap 튜플의 t_xmin/t_xmax는 여전히 32비트
```

---

## ⚖️ 트레이드오프

```
32비트 XID 설계의 트레이드오프:

장점:
  ① 튜플 헤더 공간 효율: 32비트 × 2(xmin+xmax) = 8바이트
  ② 모든 기존 데이터와 호환
  ③ VACUUM Freeze로 예방 가능 (운영 가능)

단점:
  ① Wraparound 위험: 초당 1000 트랜잭션에 49일이면 절반 소모
  ② VACUUM 필수: Freeze를 위해 정기적 VACUUM 필요
  ③ 운영 복잡도: Age 모니터링, Autovacuum 튜닝 필요

예방 전략:
  ① Autovacuum 정상 작동 확인 (가장 중요)
  ② idle_in_transaction_session_timeout 설정
  ③ Replication Slot 정기 점검
  ④ age(datfrozenxid) 모니터링 Alert 설정 (1B 초과 시)
  ⑤ 정기적 VACUUM FREEZE 스케줄링 (배치)
```

---

## 📌 핵심 정리

```
XID Wraparound 핵심:

원인:
  32비트 XID가 40억을 넘으면
  모듈 산술에 의해 과거 XID가 미래로 인식
  → 해당 XID 트랜잭션의 데이터가 보이지 않음

예방:
  VACUUM Freeze: 오래된 t_xmin을 FrozenXID(2)로 교체
  → Freeze된 튜플은 항상 "과거"로 인식

모니터링:
  age(datfrozenxid): 데이터베이스 레벨 위험도
  age(relfrozenxid): 테이블 레벨 위험도
  임계값: 200M(점검), 500M(VACUUM FREEZE 스케줄), 1B(즉시 조치), 1.5B(CRITICAL)

PostgreSQL 안전 장치:
  age > 2.1B → 쓰기 거부 (데이터 손상 방지 목적)
  → 단일 사용자 모드에서 VACUUM FREEZE 후 복구

Wraparound 유발 원인:
  장기 트랜잭션, 비활성 Replication Slot, Autovacuum 비활성화
```

---

## 🤔 생각해볼 문제

**Q1.** Autovacuum이 정상 작동 중임에도 `age(datfrozenxid)`가 계속 증가할 수 있는 경우는?

<details>
<summary>해설 보기</summary>

가능한 경우:

1. **Inactive Replication Slot**: `catalog_xmin`이 낮은 슬롯이 있으면 `pg_catalog` 테이블의 VACUUM Freeze가 차단됩니다. `datfrozenxid`는 `pg_catalog` 포함 전체 DB의 최솟값이므로 영향을 받습니다.

2. **매우 큰 테이블의 Freeze 지연**: Autovacuum이 대형 테이블의 Freeze를 완료하기 전에 새 Dead Tuple이 생겨 계속 뒤처지는 경우.

3. **autovacuum_freeze_max_age 미달**: 아직 Freeze가 강제 실행되는 임계값에 도달하지 않아 일반 VACUUM만 실행 중인 경우. Freeze가 필요한 테이블에서 vacuum_freeze_min_age를 낮추면 해결.

4. **VACUUM FREEZE 실패**: Lock 경합으로 VACUUM이 장시간 대기하다 타임아웃으로 종료.

```sql
-- Autovacuum이 Freeze를 시도 중인지 확인
SELECT pid, query, now() - xact_start AS duration
FROM pg_stat_activity
WHERE query LIKE 'autovacuum: VACUUM%FREEZE%';
```

</details>

---

**Q2.** 단일 사용자 모드(postgres --single)에서 VACUUM FREEZE를 실행할 때 주의사항은?

<details>
<summary>해설 보기</summary>

1. **PostgreSQL 완전 중지 필요**: 일반 모드의 PostgreSQL이 실행 중이면 단일 사용자 모드를 시작할 수 없습니다.

2. **시간 계획**: 테이블 크기에 따라 수 시간이 걸릴 수 있습니다. 100GB 테이블이라면 IO 속도에 따라 1~수 시간.

3. **메모리 설정**: 단일 사용자 모드에서는 `maintenance_work_mem`을 높게 설정하면 Freeze가 빠릅니다.
   ```
   postgres --single -D /var/lib/postgresql/data \
   -c maintenance_work_mem=512MB production
   VACUUM FREEZE;
   ```

4. **pg_catalog 포함**: `VACUUM FREEZE`는 일반 테이블뿐만 아니라 `pg_catalog` 스키마도 포함합니다.

5. **복구 확인**: 완료 후 재시작 전 `SELECT age(datfrozenxid) FROM pg_database;`로 age 감소 확인.

</details>

---

<div align="center">

**[⬅️ 이전: HOT Update](./05-hot-update.md)** | **[홈으로 🏠](../README.md)** | **[다음: 격리 수준과 스냅샷 ➡️](./07-isolation-snapshot.md)**

</div>
