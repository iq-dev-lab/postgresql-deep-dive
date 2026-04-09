# VACUUM 내부 동작 — Dead Tuple 회수의 단계별 흐름

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- VACUUM은 내부적으로 어떤 단계를 거쳐 Dead Tuple을 처리하는가?
- Free Space Map(FSM)과 Visibility Map(VM)은 무엇이고 VACUUM이 어떻게 업데이트하는가?
- VACUUM이 실행 중에도 다른 쿼리가 가능한 이유(최소 Lock)는?
- Index Vacuum이 왜 필요하고 어떤 순서로 실행되는가?
- VACUUM이 OS에 공간을 반환하지 않는 이유는?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

VACUUM은 PostgreSQL에서 가장 중요한 백그라운드 작업이다. "VACUUM이 Dead Tuple을 청소한다"는 것은 알지만, 내부에서 정확히 무슨 일이 일어나는지 모르면 튜닝 방향을 잡기 어렵다. FSM이 왜 필요한지, VM이 Index-Only Scan과 어떤 관계인지, VACUUM이 왜 테이블 크기를 줄이지 않는지를 이해해야 Autovacuum 튜닝과 VACUUM FULL 사용 시점을 정확히 판단할 수 있다.

---

## 😱 흔한 실수 (Before — VACUUM 내부 모름)

```
실수 1: VACUUM 후 크기가 안 줄었다고 VACUUM FULL 남발

  "VACUUM 실행했는데 테이블 크기가 그대로다"
  → "VACUUM이 효과가 없나 보다, VACUUM FULL 실행"
  → VACUUM FULL: AccessExclusiveLock 획득
  → 운영 시간대 전체 테이블 잠금 → 서비스 장애

  올바른 이해:
  VACUUM: Dead Tuple 공간을 재사용 가능으로 표시 (크기 유지, 잠금 없음)
  VACUUM FULL: 파일 재작성으로 크기 축소 (잠금 필요, 운영 중 위험)

실수 2: VACUUM 중 쿼리 차단 걱정으로 VACUUM을 기피

  "VACUUM이 실행되면 쿼리가 느려지지 않을까?"
  → VACUUM 실행 자제
  → Dead Tuple 누적 → Table Bloat

  올바른 이해:
  VACUUM은 ShareUpdateExclusiveLock (매우 약한 잠금)
  일반 SELECT, INSERT, UPDATE, DELETE와 동시 실행 가능
  DDL(ALTER TABLE, DROP 등)만 차단
```

---

## ✨ 올바른 접근 (After — VACUUM 동작 이해 후)

```
VACUUM 실행 및 모니터링:

  -- VACUUM 상태 확인
  SELECT
      pid,
      query,
      pg_size_pretty(
          (SELECT pg_relation_size(relid))
      ) AS table_size,
      phase,
      heap_blks_total,
      heap_blks_scanned,
      heap_blks_vacuumed,
      index_vacuum_count
  FROM pg_stat_progress_vacuum;

  -- VACUUM 실행 (자세한 진행 로그)
  VACUUM VERBOSE orders;

  VACUUM 결과 해석:
  INFO: vacuuming "public.orders"
  INFO: scanned index "orders_pkey" to remove 5000 row versions
  INFO: "orders": removed 5000 row versions in 100 pages
  INFO: index scan bypassed: 1000 pages from table (10% of total) had 0 dead item identifiers
  INFO: "orders": found 5000 removable, 10000 nonremovable row versions
  INFO: "orders": page count: 200, estimated free space: 50%
```

---

## 🔬 내부 동작 원리

### 1. VACUUM 단계별 흐름

```
VACUUM 전체 흐름:

Phase 1: Heap Scan (스캔 단계)
  ───────────────────────────────────
  테이블의 8KB 페이지를 순서대로 읽기
  각 페이지에서 Dead Tuple 식별:
    t_xmax가 설정된 튜플 찾기
    해당 xmax가 OldestXmin보다 작은지 확인
    → 작으면: 회수 가능 Dead Tuple (dead_item)
    → 크면: 아직 누군가 읽을 수 있음 (유지)

  dead_item 목록을 메모리에 수집
  (maintenance_work_mem 한도 내에서)

Phase 2: Index Vacuum (인덱스 청소)
  ───────────────────────────────────
  수집한 dead_item 목록으로 각 인덱스 스캔
  Dead Tuple을 가리키는 인덱스 엔트리 제거
  (인덱스 먼저 청소해야 Heap에서 제거 가능)

  인덱스 수만큼 반복:
    B-Tree 인덱스: amvacuumcleanup() 호출
    GIN 인덱스: 별도 처리
    ...

Phase 3: Heap Vacuum (Heap 청소)
  ───────────────────────────────────
  dead_item이 있는 페이지로 돌아가
  해당 튜플의 ItemId를 LP_DEAD 또는 LP_UNUSED로 표시
  페이지 내 공간 압축 (컴팩션): 살아있는 튜플을 앞으로 당김
  pd_lower, pd_upper 포인터 업데이트

Phase 4: FSM 업데이트 (Free Space Map)
  ───────────────────────────────────
  각 페이지의 새 여유 공간을 FSM에 기록
  → 다음 INSERT/UPDATE 시 여유 공간 있는 페이지 빠르게 찾기

Phase 5: Visibility Map 업데이트
  ───────────────────────────────────
  페이지 내 모든 튜플이 "모든 트랜잭션에게 보임" 상태이면
  → VM 비트 설정 (All-Visible)
  → 이후 Index-Only Scan 시 Heap Fetch 생략 가능

Phase 6: 통계 업데이트
  ───────────────────────────────────
  pg_stat_user_tables 카운터 업데이트
  (n_dead_tup 감소, last_autovacuum 갱신)
```

### 2. Free Space Map (FSM) 구조

```
FSM 파일: relation_oid_fsm
  각 페이지의 여유 공간을 1바이트로 표현 (0~255, 32바이트 단위)

FSM 계층 구조 (B-Tree 형태):
  ┌─────────────────────────────────┐
  │  Root: 전체 최대 여유 공간        │
  ├─────────────────────────────────┤
  │  Lower: 구역별 최대 여유 공간     │
  ├─────────────────────────────────┤
  │  Leaf: 각 페이지 여유 공간 (1B)  │
  │  Page 0: 200  Page 1: 0         │
  │  Page 2: 150  Page 3: 255       │
  └─────────────────────────────────┘

INSERT 시 FSM 활용:
  필요한 튜플 크기를 FSM에서 검색
  → 충분한 여유 공간이 있는 페이지 반환
  → 해당 페이지에 삽입

VACUUM 후 FSM:
  Dead Tuple 회수된 페이지의 여유 공간 증가
  FSM 업데이트 → 다음 INSERT가 이 공간 활용

FSM 확인:
  CREATE EXTENSION pg_freespacemap;
  SELECT blkno, avail
  FROM pg_freespace('orders')
  ORDER BY avail DESC
  LIMIT 10;

FSM 생성/재구성:
  새로 생성된 테이블: FSM 없음 (처음엔 VACUUM이 생성)
  VACUUM: FSM 업데이트
  VACUUM FULL: FSM 재생성
```

### 3. Visibility Map (VM) 구조

```
VM 파일: relation_oid_vm
  각 페이지당 2비트:
    Bit 0 (All-Visible): 이 페이지의 모든 튜플이 모든 트랜잭션에게 보임
    Bit 1 (All-Frozen): 이 페이지의 모든 튜플이 Freeze됨

VM의 역할 1: Index-Only Scan 최적화
  Index Scan → 인덱스 엔트리 → Heap Page 필요
  Index-Only Scan → 인덱스에서 바로 값 반환
    조건: 인덱스가 필요한 컬럼을 모두 포함 (Covering Index)
    AND: 해당 Heap 페이지가 All-Visible (VM 비트 = 1)
    → Heap Fetch 생략!

  VM 비트가 0이면:
    페이지에 아직 보이지 않는 튜플이 있을 수 있음
    → Heap Fetch 필요 (가시성 확인)

VM의 역할 2: Autovacuum Freeze 최적화
  All-Frozen 비트가 설정된 페이지는 Freeze 불필요
  → VACUUM Freeze 시 이 페이지 건너뜀 → 빠른 Freeze

VM 비트 설정/해제:
  설정: VACUUM이 페이지 내 모든 튜플 확인 후 설정
  해제: 해당 페이지에 INSERT/UPDATE/DELETE 발생 시 자동 해제

VM 확인:
  CREATE EXTENSION pg_visibility;
  SELECT * FROM pg_visibility_map('orders', 0);  -- 0번 페이지
  -- all_visible | all_frozen

  -- 테이블 전체 VM 통계
  SELECT * FROM pg_visibility_map_summary('orders');
  -- all_visible | all_frozen (페이지 수)
```

### 4. VACUUM의 잠금 전략

```
VACUUM이 획득하는 잠금:

ShareUpdateExclusiveLock (약한 잠금):
  동시 허용: SELECT, INSERT, UPDATE, DELETE
  차단됨: ALTER TABLE, VACUUM FULL, CLUSTER, CREATE INDEX CONCURRENTLY

→ 일반 DML 쿼리는 VACUUM 중에도 계속 실행 가능!

예외: Truncate 작업 중 특정 페이지에 AccessExclusiveLock 짧게
  (페이지 레벨 핀 처리, 밀리초 단위)

Index Vacuum 중의 잠금:
  각 인덱스에 대해 IndexScanLock (읽기와 호환)
  인덱스 페이지 수정 시 잠깐 버퍼 핀

cost-based delay (vacuum_cost_delay):
  VACUUM이 I/O 집중 시 잠시 sleep
  autovacuum_vacuum_cost_limit: 초당 처리할 비용 상한
  vacuum_cost_delay: 한도 초과 시 sleep 시간 (기본 0ms, Autovacuum은 2ms)
  → VACUUM이 시스템 I/O를 독점하지 않도록 속도 제한

  cost 계산:
    페이지 읽기(캐시 미스): vacuum_cost_page_miss = 2
    페이지 읽기(캐시 히트): vacuum_cost_page_hit = 1
    페이지 쓰기(더티): vacuum_cost_page_dirty = 20
```

### 5. maintenance_work_mem과 다중 패스

```
VACUUM 메모리 한계:

  dead_item 목록을 maintenance_work_mem에 수집
  기본값: 64MB
  dead_item 하나당 6 bytes (TID: page + offset)
  → 64MB / 6B = 약 1,000만 TID 저장 가능

  테이블에 Dead Tuple이 2,000만 개라면:
  → 1번째 패스: 1,000만 개 수집 → Index Vacuum → Heap Vacuum
  → 2번째 패스: 나머지 1,000만 개 수집 → Index Vacuum → Heap Vacuum
  → 인덱스를 두 번 스캔 (패스 수만큼)

  maintenance_work_mem을 늘리면:
  → 한 번에 더 많은 Dead TID 처리 → 인덱스 스캔 횟수 감소
  → VACUUM 속도 향상 (인덱스가 많을수록 효과 큼)

  SET maintenance_work_mem = '256MB';  -- VACUUM 전 세션 설정
  VACUUM orders;
```

---

## 💻 실전 실험

### 실험 1: VACUUM 진행 상황 실시간 관찰

```sql
-- 터미널 1: 대용량 테이블에서 VACUUM 실행 (백그라운드로)
-- 먼저 Dead Tuple 대량 생성
CREATE TABLE vacuum_test AS
SELECT i::int AS id, md5(i::text) AS val
FROM generate_series(1, 1000000) i;

CREATE INDEX ON vacuum_test(id);

UPDATE vacuum_test SET val = md5(random()::text);
-- Dead Tuple 100만 개 생성

-- 터미널 2: 진행 상황 모니터링
SELECT
    phase,
    heap_blks_total,
    heap_blks_scanned,
    round(heap_blks_scanned * 100.0 / nullif(heap_blks_total, 0), 1) AS scan_pct,
    heap_blks_vacuumed,
    index_vacuum_count,
    num_dead_tuples,
    max_dead_tuples
FROM pg_stat_progress_vacuum
WHERE relid = 'vacuum_test'::regclass;
-- Phase가 순서대로 변화: scanning heap → vacuuming indexes → vacuuming heap → ...
```

### 실험 2: VM 비트와 Index-Only Scan 관계

```sql
CREATE EXTENSION pg_visibility;

CREATE TABLE vm_test (id INT, amount NUMERIC);
CREATE INDEX ON vm_test(id);

INSERT INTO vm_test SELECT i, random() * 1000
FROM generate_series(1, 10000) i;

-- VACUUM 전: VM 비트 없음
SELECT all_visible, all_frozen
FROM pg_visibility_map('vm_test', 0);
-- all_visible = false

-- VACUUM 실행
VACUUM vm_test;

-- VACUUM 후: All-Visible 설정됨
SELECT all_visible, all_frozen
FROM pg_visibility_map('vm_test', 0);
-- all_visible = true

-- Index-Only Scan 실행 계획 확인
EXPLAIN SELECT id FROM vm_test WHERE id BETWEEN 1 AND 100;
-- Index Only Scan (Heap Fetch 없음!)

-- UPDATE 하나 실행 (VM 비트 해제)
UPDATE vm_test SET amount = 999 WHERE id = 1;

-- VM 비트 해제됨 (UPDATE된 페이지)
SELECT all_visible FROM pg_visibility_map('vm_test', 0);
-- all_visible = false

-- 이제 Index Scan (Index Only Scan 아님)
EXPLAIN SELECT id FROM vm_test WHERE id BETWEEN 1 AND 100;
```

### 실험 3: FSM 확인

```sql
CREATE EXTENSION pg_freespacemap;

-- VACUUM 전 여유 공간
SELECT blkno, avail
FROM pg_freespace('vacuum_test')
WHERE avail > 0
ORDER BY avail DESC LIMIT 5;
-- UPDATE 직후: 여유 공간 거의 없음

VACUUM vacuum_test;

-- VACUUM 후 여유 공간
SELECT blkno, avail
FROM pg_freespace('vacuum_test')
WHERE avail > 0
ORDER BY avail DESC LIMIT 5;
-- Dead Tuple이 정리된 페이지의 여유 공간 증가
```

---

## 📊 MySQL과 비교

```
VACUUM vs MySQL Purge Thread:

PostgreSQL VACUUM:
  실행 방법: 명시적 VACUUM 또는 Autovacuum
  잠금 수준: ShareUpdateExclusiveLock (약함)
  처리 대상: Dead Tuple이 있는 페이지
  인덱스 처리: 별도 Index Vacuum 단계 필요
  OS 공간 반환: 없음 (VACUUM FULL만 가능)
  속도 제어: vacuum_cost_delay, vacuum_cost_limit
  메모리 사용: maintenance_work_mem

MySQL InnoDB Purge Thread:
  실행 방법: 백그라운드 스레드 자동 실행
  잠금 수준: 없음 (Undo Log만 처리)
  처리 대상: Undo Log 레코드 삭제
  인덱스 처리: Purge 중 Secondary Index 정리 포함
  OS 공간 반환: Undo Tablespace 자동 관리 (innodb_undo_log_truncate)
  속도 제어: innodb_purge_threads
  메모리 사용: innodb_purge_batch_size

운영 차이:
  PostgreSQL: Autovacuum 튜닝이 DBA의 중요 작업
  MySQL: Purge 는 자동, Long Transaction 관리가 핵심
```

---

## ⚖️ 트레이드오프

```
VACUUM 설계의 장단점:

장점:
  ① 일반 쿼리와 동시 실행 가능 (약한 잠금)
  ② 부분 처리 가능: maintenance_work_mem 한도 내 배치 처리
  ③ cost-based delay: I/O 집중 방지

단점:
  ① OS에 공간 반환 불가: 파일 크기는 유지 (재사용만 가능)
  ② 인덱스 별도 스캔: 인덱스가 많을수록 느림
  ③ 장기 트랜잭션: OldestXmin이 낮으면 Dead Tuple 회수 불가

VACUUM이 OS에 공간을 반환하지 않는 이유:
  Heap 페이지의 마지막 일부만 비어있을 때만 파일 끝을 자를 수 있음
  중간에 Dead Tuple이 비워진 페이지는 "재사용 가능"으로 표시만
  → 이 공간은 다음 INSERT/UPDATE가 활용
  → 물리적 파일 크기는 유지되나 실제 데이터 공간은 효율적

VACUUM FULL이 필요한 경우:
  테이블이 영구적으로 줄어드는 경우 (대량 DELETE 후 INSERT 없음)
  디스크 공간이 실제로 부족한 경우
  → 단, 운영 중 AccessExclusiveLock → 서비스 중단 위험
  → 대안: pg_repack (운영 중 사용 가능)
```

---

## 📌 핵심 정리

```
VACUUM 내부 흐름:

단계:
  1. Heap Scan: Dead Tuple 식별 및 TID 수집 (maintenance_work_mem)
  2. Index Vacuum: 인덱스에서 Dead 엔트리 제거
  3. Heap Vacuum: Dead Tuple 공간을 LP_UNUSED로 표시, 페이지 컴팩션
  4. FSM 업데이트: 여유 공간 정보 갱신
  5. VM 업데이트: All-Visible 비트 설정 (Index-Only Scan 지원)

잠금:
  ShareUpdateExclusiveLock → SELECT/DML 동시 실행 가능

FSM:
  각 페이지 여유 공간 추적 → INSERT 시 빠른 페이지 선택

VM:
  All-Visible 비트 → Index-Only Scan 시 Heap Fetch 생략 가능

OS 공간 반환:
  VACUUM: 안 함 (재사용 가능 표시만)
  VACUUM FULL: 함 (AccessExclusiveLock, 운영 중 주의)
```

---

## 🤔 생각해볼 문제

**Q1.** VACUUM이 "Index Vacuum → Heap Vacuum" 순서로 처리하는 이유는? 순서가 바뀌면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

인덱스가 Dead Tuple을 가리키고 있는 상태에서 Heap의 Dead Tuple을 먼저 제거하면, 인덱스 엔트리가 **더 이상 존재하지 않는 Heap 위치를 가리키는 dangling pointer** 상태가 됩니다. 이후 Index Scan 시 인덱스 엔트리를 따라 Heap에 접근하면 잘못된 데이터를 읽을 수 있습니다.

올바른 순서:
1. Index Vacuum: 인덱스에서 Dead Tuple 가리키는 엔트리 제거
2. Heap Vacuum: 이제 아무도 가리키지 않는 Dead Tuple 제거

이렇게 하면 Dead Tuple이 존재하는 동안 인덱스에서 언제든지 접근 가능하고, 정리 후에는 인덱스도 깨끗한 상태가 됩니다.

</details>

---

**Q2.** `maintenance_work_mem`을 매우 크게 설정하면 VACUUM 성능에 어떤 영향이 있는가? 무한정 키우면 되는가?

<details>
<summary>해설 보기</summary>

`maintenance_work_mem`이 클수록 Dead Tuple TID를 더 많이 한 번에 수집할 수 있어, 인덱스를 더 적게 스캔합니다. 인덱스가 10개이고 2번 패스가 필요하던 것이 1번 패스로 끝나면, 인덱스 스캔이 10번 → 10번 아닌 경우의 절반으로 줄어듭니다.

무한정 키우면 안 되는 이유:
- 여러 Autovacuum Worker가 동시 실행 중이면 각각 `maintenance_work_mem`을 소비
- `max_parallel_maintenance_workers`개의 병렬 VACUUM도 각각 소비
- 총 메모리 = autovacuum_max_workers × maintenance_work_mem 최대 가능

권장:
```
maintenance_work_mem = min(총 RAM × 0.1 / autovacuum_max_workers, 1GB)
```

세션별로 VACUUM 전에만 높게 설정하는 방법도 유효합니다:
```sql
SET maintenance_work_mem = '512MB';
VACUUM ANALYZE large_table;
RESET maintenance_work_mem;
```

</details>

---

**Q3.** VM의 All-Visible 비트가 해제되는 조건은 정확히 무엇인가? UPDATE 하나로 페이지 전체의 All-Visible이 해제되는가?

<details>
<summary>해설 보기</summary>

네, 해당 페이지에 **INSERT, UPDATE, DELETE가 하나라도 발생하면** 그 페이지의 All-Visible 비트가 해제됩니다. 이유는 새로 삽입된 튜플은 아직 모든 트랜잭션에게 보이지 않을 수 있기 때문입니다(새 트랜잭션의 xmin이 다른 트랜잭션의 스냅샷보다 클 수 있음).

비트 해제는 즉각적이고 해당 페이지 전체에 적용됩니다. 따라서 UPDATE가 빈번한 테이블은 VM 비트가 자주 해제되어 Index-Only Scan을 잘 활용하지 못합니다. 이런 테이블에서 Index-Only Scan을 기대하기 어렵고, VACUUM이 빈번하게 실행되어야 VM 비트가 복원됩니다.

`fillfactor`를 낮게 설정하면 HOT Update로 같은 페이지에서 처리되더라도 VM 비트 해제는 피할 수 없습니다. VM 비트 해제는 페이지 레벨에서 발생하므로 HOT Update와 무관합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Dead Tuple 완전 분해](./01-dead-tuple.md)** | **[홈으로 🏠](../README.md)** | **[다음: VACUUM FULL vs VACUUM ➡️](./03-vacuum-full-vs-vacuum.md)**

</div>
