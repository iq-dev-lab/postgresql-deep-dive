# PostgreSQL MVCC vs MySQL MVCC — Heap 내부 버전이 만드는 Dead Tuple

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- MySQL은 UPDATE 시 Undo Log에 이전 버전을 저장하는데, PostgreSQL은 왜 Heap에 이전 버전을 그대로 두는가?
- Dead Tuple이 정확히 어떤 상태의 튜플을 가리키는가?
- 두 MVCC 구현 방식 중 어느 것이 읽기에 유리하고, 어느 것이 쓰기에 유리한가?
- PostgreSQL의 MVCC 방식은 어떤 문제를 야기하고, MySQL은 그 문제가 없는가?
- Dead Tuple을 방치하면 쿼리 성능에 어떤 영향이 나타나는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL을 운영하면서 VACUUM 없이 방치된 테이블이 수십 배로 커지는 경험을 하면, 반드시 "왜 MySQL에서는 이런 일이 없었나?"라는 질문이 생긴다. 그 답은 두 데이터베이스가 MVCC(Multi-Version Concurrency Control)를 근본적으로 다른 방식으로 구현하기 때문이다. 이 차이를 이해하지 못하면 VACUUM의 존재 이유를 납득할 수 없고, Autovacuum 튜닝의 방향도 잡기 어렵다.

---

## 😱 흔한 실수 (Before — MySQL 방식으로 생각할 때)

```
흔한 오해:
  "DELETE FROM orders WHERE created_at < NOW() - INTERVAL '30 days';"
  → "1백만 건 삭제했으니 공간이 1백만 건만큼 줄어들겠지"

실제 결과:
  테이블 파일 크기: 그대로
  Dead Tuple: 1백만 개 (아직 페이지에 존재)
  → Autovacuum이 처리하거나 수동 VACUUM이 필요

또 다른 오해:
  "VACUUM을 실행하면 테이블 크기가 줄어들겠지"

실제 결과:
  VACUUM: Dead Tuple 공간을 "재사용 가능"으로 표시
  파일 크기: 변화 없음 (OS에 반환 안 함)
  → 테이블 크기를 실제로 줄이려면 VACUUM FULL (운영 중 위험)
     또는 pg_repack (운영 중 사용 가능, 별도 도구)

운영 중 발생한 실제 장애 사례:
  OLTP 서비스, 초당 1,000 UPDATE
  Autovacuum 기본 설정으로 방치
  Dead Tuple 비율: 80% 이상
  → 테이블 전체 스캔 시 페이지 대부분이 Dead Tuple
  → SeqScan 비용 5~10배 증가
  → 쿼리 타임아웃 발생
```

---

## ✨ 올바른 접근 (After — MVCC 차이를 이해한 후)

```
PostgreSQL MVCC 특성을 반영한 운영:

  Dead Tuple 모니터링:
    SELECT relname, n_dead_tup, n_live_tup,
           round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_ratio,
           last_autovacuum
    FROM pg_stat_user_tables
    WHERE n_dead_tup > 0
    ORDER BY n_dead_tup DESC;

  Autovacuum 적극 튜닝:
    -- UPDATE 빈번한 테이블: Autovacuum 조건 낮춤
    ALTER TABLE orders SET (
        autovacuum_vacuum_scale_factor = 0.01,  -- 1% 변경 시 실행 (기본 20%)
        autovacuum_vacuum_threshold = 100        -- 최소 100개 Dead Tuple 시 실행
    );

  UPDATE 설계 개선:
    -- 인덱스가 있는 컬럼만 업데이트 → HOT Update 불가 (인덱스 갱신 필요)
    -- 인덱스가 없는 컬럼만 업데이트 → HOT Update 가능 (같은 페이지 내)
    -- 컬럼 업데이트 전략이 VACUUM 부하에 직접 영향
```

---

## 🔬 내부 동작 원리

### 1. MySQL MVCC — Undo Log 방식

```
MySQL InnoDB MVCC 구조:

UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
(트랜잭션 T1, XID=200)

=== 현재 데이터 페이지 ===

Before:
┌──────────────────────────────────┐
│  Page (Clustered Index)          │
│  Record: id=1, status='PENDING'  │
│          DB_TRX_ID=100           │
│          DB_ROLL_PTR ─────────→  │ (이전 Undo 가리킴)
└──────────────────────────────────┘

After:
┌──────────────────────────────────┐
│  Page (Clustered Index)          │
│  Record: id=1, status='SHIPPED'  │ ← 현재 페이지의 레코드를 직접 수정
│          DB_TRX_ID=200           │
│          DB_ROLL_PTR ──────────→ │ ← Undo Log를 가리킴
└──────────────────────────────────┘

=== Undo Log (별도 파일: ibdata1 또는 undo tablespace) ===

Undo Record:
  이전 상태: id=1, status='PENDING', DB_TRX_ID=100
  ← 롤백 시 또는 다른 트랜잭션이 오래된 버전 읽을 때 사용

=== 트랜잭션 T2가 이전 버전을 읽으려 할 때 ===

T2 스냅샷 기준: XID 150 이하 버전만 읽어야 함
  → 현재 레코드: DB_TRX_ID=200 → 내 스냅샷보다 새로움 → 안 됨
  → DB_ROLL_PTR 따라가 Undo Log 조회
  → 이전 버전: DB_TRX_ID=100 → OK → 'PENDING' 반환

Purge Thread:
  모든 트랜잭션이 더 이상 참조하지 않는 Undo Record를 주기적 삭제
  → 테이블 페이지는 항상 최신 버전만 존재
  → Undo Tablespace가 커지는 경우: 오래된 트랜잭션이 Purge를 막음

MySQL MVCC 흐름:
  1. UPDATE → 현재 페이지 수정 + DB_ROLL_PTR 설정
  2. 이전 버전 → Undo Log에 저장
  3. 읽기 → 현재 버전이 너무 새로우면 Undo Log 체인 순회
  4. Purge Thread → 불필요한 Undo Record 삭제
```

### 2. PostgreSQL MVCC — Heap 내부 버전 방식

```
PostgreSQL MVCC 구조:

UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
(트랜잭션 T1, XID=200)

=== Heap 페이지 (변화) ===

Before:
┌──────────────────────────────────────────┐
│  Heap Page 0                             │
│  ItemId[1] → offset=7980                │
│  ItemId[2] (없음)                        │
│                                          │
│  Tuple 1: id=1, status='PENDING'        │
│    t_xmin=100, t_xmax=0                 │ ← 살아있음
│    t_ctid=(0,1)                         │
└──────────────────────────────────────────┘

After UPDATE (같은 페이지 내 공간이 있는 경우):
┌──────────────────────────────────────────┐
│  Heap Page 0                             │
│  ItemId[1] → offset=7980                │
│  ItemId[2] → offset=7930                │ ← 새 Item Pointer 추가
│                                          │
│  NEW Tuple 2: id=1, status='SHIPPED'    │ ← 새 버전 삽입
│    t_xmin=200, t_xmax=0                 │
│    t_ctid=(0,2)                         │
│                                          │
│  OLD Tuple 1: id=1, status='PENDING'    │ ← Dead Tuple!
│    t_xmin=100, t_xmax=200              │ ← T1이 커밋되면 Dead
│    t_ctid=(0,2)                         │ ← 새 버전 가리킴
└──────────────────────────────────────────┘

핵심 차이:
  MySQL: 현재 페이지 = 최신 버전만 / Undo Log = 이전 버전들
  PostgreSQL: Heap 페이지 = 여러 버전 공존 / Undo Log 없음

=== T2가 이전 버전을 읽으려 할 때 ===

T2 스냅샷 기준: XID 150 이하 버전만 읽어야 함
  → Tuple 2: t_xmin=200 → 내 스냅샷보다 새로움 → 안 됨
  → Tuple 1: t_xmin=100 → OK → t_xmax=200은 아직 커밋 전? → YES → 'PENDING' 반환

T1이 커밋된 후 T2가 같은 조회:
  → Tuple 2: t_xmin=200 → 내 스냅샷보다 새로움 → 안 됨
  → Tuple 1: t_xmin=100 OK, t_xmax=200 커밋됨 → Dead → 안 됨
  → 찾지 못함 → MVCC 스냅샷 격리 정상 동작 (REPEATABLE READ)
```

### 3. Dead Tuple이 쌓이는 과정과 영향

```
UPDATE 1만 번 실행 시나리오:

초기 상태:
  테이블: 10,000행, 각 50 bytes → 약 5MB

UPDATE 1만 번 (각 행의 status 컬럼 변경):

  각 UPDATE마다:
    OLD 버전: t_xmax 설정 (Dead Tuple)
    NEW 버전: Heap에 삽입

  결과:
    Live Tuples: 10,000개 (최신 버전)
    Dead Tuples: 10,000개 (이전 버전)
    테이블 크기: ~10MB (2배)

  Dead Tuple이 더 쌓이면:
    2만 번 UPDATE → Dead 2만, Live 1만 → ~15MB (3배)

Dead Tuple이 성능에 미치는 영향:

  SeqScan (전체 테이블 스캔):
    8KB 페이지를 순서대로 읽음
    페이지마다 Dead Tuple도 스캔 → 가시성 확인 후 제외
    → 실제 필요 튜플보다 훨씬 많은 페이지 I/O 발생

  예시:
    Live 10,000개 + Dead 90,000개 (dead_ratio=90%)
    → SeqScan이 10배 많은 페이지 읽음
    → Seq Scan 비용 10배 증가

  Index Scan도 영향:
    인덱스 엔트리 → Heap Fetch → Dead 확인 → 버려짐
    → 인덱스가 있어도 Dead Tuple이 많으면 느려짐

  Dead Tuple 비율과 성능:
  dead_ratio | SeqScan 영향
  ───────────┼─────────────
  0~5%       | 무시 가능
  5~20%      | 약간 느려짐
  20~50%     | 눈에 띄게 느려짐
  50~80%     | 심각하게 느려짐
  80% 이상    | 쿼리 타임아웃 위험
```

### 4. 두 방식의 근본 트레이드오프

```
┌────────────────────────────────────────────────────────────┐
│            MVCC 구현 방식 비교                               │
├────────────────────┬───────────────────┬───────────────────┤
│ 항목               │ MySQL (Undo Log)   │ PostgreSQL (Heap) │
├────────────────────┼───────────────────┼───────────────────┤
│ 이전 버전 저장 위치  │ Undo Tablespace   │ Heap 페이지       │
│ 현재 페이지 상태    │ 항상 최신 버전만   │ 여러 버전 공존     │
│ 이전 버전 읽기      │ Undo Log 체인 순회 │ Heap 직접 읽기    │
│ 이전 버전 정리      │ Purge Thread 자동  │ VACUUM 필요       │
│ 크래시 복구         │ Undo 기반 롤백     │ WAL 기반 복구     │
│ 테이블 공간 관리    │ 안정적 크기        │ Bloat 관리 필요   │
│ 읽기 성능 (최신)    │ 빠름              │ 힌트 비트로 빠름   │
│ 읽기 성능 (오래된)  │ Undo 체인 비용    │ 같은 페이지 접근  │
│ 쓰기 성능           │ 이중 쓰기(페이지+Undo) │ 단일 쓰기(Heap) │
│ WAL 크기            │ Binlog + Redo Log │ WAL 단일           │
└────────────────────┴───────────────────┴───────────────────┘

PostgreSQL이 Heap 방식을 선택한 이유:
  ① 구현 단순성: Undo Log 관리 시스템이 불필요
                 → 크래시 복구도 WAL만으로 처리
  ② 오래된 버전 접근 비용: 같은 Heap 페이지에 있어 I/O 최소
                           MySQL: Undo 체인 길면 I/O 많음
  ③ 트랜잭션 롤백: WAL로 물리적 undo (페이지 원복)
                   Undo Log 없이도 롤백 가능

MySQL이 Undo 방식을 선택한 이유:
  ① 현재 버전 접근 최적: 현재 페이지는 항상 최신 버전
                         → 대부분의 쿼리가 최신 데이터를 읽음
  ② 테이블 공간 예측 가능: UPDATE/DELETE가 테이블 크기에 영향 없음
  ③ 자동 정리: Purge Thread가 비동기로 처리
```

---

## 💻 실전 실험

### 실험 1: Dead Tuple 발생 과정 관찰

```sql
-- 실험 테이블
CREATE TABLE mvcc_test (
    id SERIAL PRIMARY KEY,
    status TEXT,
    data TEXT
);

-- 초기 데이터
INSERT INTO mvcc_test (status, data)
SELECT 'PENDING', md5(i::text)
FROM generate_series(1, 1000) i;

-- Dead Tuple 현황 (초기)
SELECT n_live_tup, n_dead_tup,
       round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_ratio
FROM pg_stat_user_tables
WHERE relname = 'mvcc_test';
-- n_dead_tup = 0

-- UPDATE 1,000번 실행
UPDATE mvcc_test SET status = 'PROCESSING';

-- Dead Tuple 현황 (UPDATE 후)
SELECT n_live_tup, n_dead_tup,
       round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_ratio
FROM pg_stat_user_tables
WHERE relname = 'mvcc_test';
-- n_dead_tup ≈ 1000, dead_ratio ≈ 50%

-- 테이블 크기 확인
SELECT pg_size_pretty(pg_relation_size('mvcc_test')) AS table_size;
-- UPDATE 전보다 약 2배

-- VACUUM 실행
VACUUM mvcc_test;

-- VACUUM 후 Dead Tuple
SELECT n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'mvcc_test';
-- n_dead_tup = 0

-- 크기 확인 (VACUUM 후에도 크기 변화 없음)
SELECT pg_size_pretty(pg_relation_size('mvcc_test')) AS table_size;
-- 여전히 큼 (OS에 반환 안 함)
```

### 실험 2: pageinspect로 Dead Tuple 직접 관찰

```sql
CREATE EXTENSION pageinspect;

-- 단순 테이블로 실험
CREATE TABLE version_test (id INT, val TEXT);
INSERT INTO version_test VALUES (1, 'original');

-- UPDATE 실행
BEGIN;
UPDATE version_test SET val = 'updated' WHERE id = 1;
COMMIT;

-- 페이지 내부의 두 버전 확인
SELECT
    lp AS item_id,
    t_ctid,
    t_xmin,
    t_xmax,
    CASE
        WHEN t_xmax = 0 THEN 'ALIVE'
        ELSE 'DEAD (or locked)'
    END AS state,
    t_data
FROM heap_page_items(get_raw_page('version_test', 0))
WHERE t_xmin IS NOT NULL;
-- item_id=1: t_xmax=<XID>, state=DEAD (OLD 버전)
-- item_id=2: t_xmax=0, state=ALIVE (NEW 버전)
```

### 실험 3: 장기 트랜잭션이 Dead Tuple 청소를 막는 시뮬레이션

```sql
-- 터미널 1: 장기 트랜잭션 시작 (스냅샷 고정)
BEGIN;
-- (이 상태에서 오래 유지)

-- 터미널 2: UPDATE 반복
UPDATE mvcc_test SET status = 'DONE';

-- Dead Tuple 현황
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'mvcc_test';
-- n_dead_tup 증가

-- VACUUM 실행
VACUUM mvcc_test;

-- Dead Tuple이 여전히 남아있음 확인 (장기 트랜잭션이 참조할 수 있으므로 청소 불가)
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'mvcc_test';
-- n_dead_tup = 0이 아님!

-- 터미널 1: 트랜잭션 종료
ROLLBACK;

-- 다시 VACUUM 실행
VACUUM mvcc_test;
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'mvcc_test';
-- 이제 n_dead_tup = 0
```

---

## 📊 MySQL과 비교

```
동일 워크로드: 1만 행 UPDATE 100번 반복 (총 100만 UPDATE)

PostgreSQL:
  테이블 크기 변화:
    초기: 5MB
    UPDATE 후: 최대 50~100MB (Autovacuum이 따라오지 못하면)
    Autovacuum 정상 작동 시: 10~20MB 수준 유지

  Undo Log: 없음 (WAL만 존재)
  WAL 생성: 각 UPDATE마다 OLD + NEW 튜플 변경 기록
  VACUUM 필요: 반드시

MySQL InnoDB:
  테이블 크기 변화:
    초기: 5MB
    UPDATE 후: ~5MB (테이블 파일은 안정적)
    Undo Tablespace: 증가할 수 있음 (긴 트랜잭션 유지 시)

  Undo Log 크기:
    짧은 트랜잭션: 빠른 Purge → 소량
    긴 트랜잭션: Purge 차단 → Undo Log 대량 누적 가능
    mysql 5.7+: 별도 Undo Tablespace 관리

  정리 비용:
    PostgreSQL: VACUUM이 명시적으로 실행해야 함
    MySQL: Purge Thread가 자동으로 처리 (CPU 사용)

결론:
  PostgreSQL: 테이블 공간 관리가 운영 부담이나, 구조 단순
  MySQL: 테이블 공간 안정적이나, Undo Log 별도 모니터링 필요
```

---

## ⚖️ 트레이드오프

```
PostgreSQL Heap MVCC의 장단점:

장점:
  ① 오래된 버전 접근 빠름:
     같은 페이지에 있어 추가 I/O 없이 이전 버전 읽기
     MySQL: 긴 Undo 체인은 여러 번 I/O 필요

  ② 구현 단순:
     Undo Log 관리 시스템 없음
     크래시 복구도 WAL만으로 처리 (Undo 재실행 불필요)

  ③ 쓰기 증폭 낮음:
     Heap 한 곳에만 쓰기 (MySQL: 페이지 + Undo Log 이중 쓰기)

단점:
  ① Table Bloat:
     Dead Tuple이 Heap에 쌓임 → 테이블 비대화
     VACUUM 없이는 계속 증가

  ② VACUUM 운영 부담:
     Autovacuum 튜닝, 모니터링, 장기 트랜잭션 관리 필요

  ③ 현재 버전 읽기:
     최신 버전도 Heap에서 읽되 힌트 비트로 최적화
     (MySQL은 현재 페이지가 항상 최신 버전이라 단순)

실무 결론:
  PostgreSQL MVCC의 단점은 Autovacuum 올바른 설정으로 대부분 완화 가능
  문제는 Autovacuum을 기본값으로 방치하는 것
  특히 대형 테이블, 높은 DML 워크로드에서 적극 튜닝 필요
```

---

## 📌 핵심 정리

```
PostgreSQL vs MySQL MVCC 핵심:

MySQL MVCC (Undo Log 방식):
  UPDATE → 현재 페이지 직접 수정 + Undo Log에 이전 버전 저장
  이전 버전 조회 → DB_ROLL_PTR 따라 Undo 체인 순회
  정리 → Purge Thread 자동 처리
  테이블 파일: 안정적

PostgreSQL MVCC (Heap 내부 버전 방식):
  UPDATE → OLD 튜플 t_xmax 설정 (Dead) + NEW 튜플 삽입
  이전 버전 조회 → 같은 Heap 페이지에서 직접 읽기
  정리 → VACUUM 명시적 실행 필요
  테이블 파일: Dead Tuple 누적 시 Table Bloat 발생

Dead Tuple:
  t_xmax가 설정된 튜플 (아직 페이지에 물리적으로 존재)
  VACUUM이 "재사용 가능"으로 표시 후 공간 회수

모니터링:
  SELECT n_dead_tup, n_live_tup,
         round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 1) dead_ratio
  FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;
  → dead_ratio > 20%이면 Autovacuum 튜닝 또는 수동 VACUUM 필요
```

---

## 🤔 생각해볼 문제

**Q1.** PostgreSQL에서 `ROLLBACK`된 트랜잭션의 INSERT로 생성된 튜플은 어떻게 처리되는가? Undo Log 없이 어떻게 이전 상태로 돌아가는가?

<details>
<summary>해설 보기</summary>

PostgreSQL에서 트랜잭션이 ROLLBACK되면, 해당 트랜잭션이 삽입한 튜플의 `t_xmin`은 "ROLLBACK된 XID"로 기록됩니다. 다른 트랜잭션이 이 튜플을 읽을 때 `pg_xact`(CLOG)를 확인하면 해당 XID가 ROLLBACK 상태이므로, 이 튜플은 보이지 않습니다.

물리적으로 삭제하지는 않습니다. `t_infomask`에 `HEAP_XMIN_INVALID` 힌트 비트가 설정되어 이후 읽기 시 CLOG 조회 없이 즉시 무시됩니다. VACUUM이 이후에 이 튜플도 Dead Tuple처럼 정리합니다.

즉, PostgreSQL의 "롤백"은 실제로 물리적 undo가 아니라 **"이 XID가 커밋되지 않았으므로 보이지 않는다"**는 가시성 규칙을 활용하는 것입니다. WAL에 기록된 변경은 그대로이지만, 트랜잭션이 커밋 상태가 아니면 해당 변경이 보이지 않습니다.

</details>

---

**Q2.** PostgreSQL에서 `SELECT ... FOR UPDATE`가 Dead Tuple을 만드는가? 잠금(Lock)과 MVCC는 어떻게 상호작용하는가?

<details>
<summary>해설 보기</summary>

`SELECT ... FOR UPDATE`는 Dead Tuple을 만들지 **않습니다**. 이 명령어는 튜플에 "Row Lock" 표시를 추가하지만 `t_xmax`에 현재 트랜잭션 XID를 임시로 기록합니다. 트랜잭션이 커밋 또는 롤백되면 `t_xmax`의 의미가 "잠금"이었는지 "삭제"였는지는 `t_infomask`의 플래그로 구분합니다.

`HEAP_XMAX_LOCK_ONLY` 플래그가 설정된 `t_xmax`는 삭제가 아닌 잠금을 의미하므로, 트랜잭션이 끝나면 이 튜플은 Dead가 아닌 Live 상태로 남습니다.

MVCC와 Lock의 상호작용:
- MVCC: **읽기 격리** — 어떤 버전을 볼 수 있는지 결정 (잠금 없이 읽기 가능)
- Lock: **쓰기 충돌 방지** — 동시에 같은 행을 수정하는 것을 방지

PostgreSQL은 읽기에는 MVCC를 사용하므로 읽기와 쓰기가 서로 차단하지 않습니다.

</details>

---

**Q3.** 긴 트랜잭션(Long Running Transaction)이 VACUUM을 방해하는 정확한 메커니즘은 무엇인가?

<details>
<summary>해설 보기</summary>

VACUUM은 Dead Tuple이 "어떤 트랜잭션도 더 이상 볼 수 없는" 상태일 때만 해당 공간을 회수합니다. PostgreSQL은 활성 트랜잭션들의 스냅샷 중 **가장 오래된 XID**(OldestXmin)를 계산합니다.

어떤 트랜잭션이 XID=100에서 시작했고 아직 살아있다면, OldestXmin=100입니다. `t_xmax=150`인 Dead Tuple은 "XID=100 스냅샷에서 보일 수 있음"이므로 VACUUM이 회수하지 못합니다.

확인 방법:
```sql
-- 가장 오래된 활성 트랜잭션 확인
SELECT pid, usename, xact_start, state,
       age(backend_xid) AS xid_age,
       left(query, 50) AS query
FROM pg_stat_activity
WHERE backend_xid IS NOT NULL
  AND state != 'idle'
ORDER BY xact_start;
```

10시간 동안 열린 트랜잭션이 있으면, 그 시간 동안 생성된 모든 Dead Tuple을 VACUUM이 회수하지 못합니다. 이로 인해 Table Bloat이 발생합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Storage Manager와 페이지 구조](./02-storage-manager-page.md)** | **[홈으로 🏠](../README.md)** | **[다음: WAL 완전 분해 ➡️](./04-wal-complete.md)**

</div>
