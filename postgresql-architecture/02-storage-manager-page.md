# Storage Manager와 페이지 구조 — 8KB Heap Page 완전 분해

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PostgreSQL의 8KB 페이지에는 정확히 무엇이 어떤 순서로 저장되는가?
- `pd_lower`, `pd_upper`, `pd_special`은 각각 무엇을 가리키는가?
- 하나의 행(튜플)이 INSERT될 때 페이지 내부에서 어떤 변화가 일어나는가?
- Item Pointer(line pointer) 배열은 왜 존재하는가?
- MySQL InnoDB의 16KB 페이지와 PostgreSQL의 8KB 페이지는 어떻게 다른가?
- `pageinspect` 확장으로 실제 페이지 내부를 어떻게 들여다보는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL의 MVCC, VACUUM, HOT Update, TOAST는 모두 **페이지 구조** 위에서 작동한다. Dead Tuple이 페이지에 쌓이는 이유, HOT Update가 같은 페이지 내에서만 가능한 이유, VACUUM이 Free Space Map을 업데이트하는 이유를 이해하려면 먼저 8KB 페이지가 어떻게 생겼는지 알아야 한다. 페이지 구조를 모르면 VACUUM이 왜 테이블 크기를 줄이지 못하는지(OS에 반환 안 함) 납득하기 어렵다.

---

## 😱 흔한 실수 (Before — 페이지 구조를 모를 때)

```
상황: UPDATE를 1만 번 실행했더니 테이블이 10배로 커짐

원리를 모를 때의 판단:
  "데이터를 수정했을 뿐인데 왜 크기가 늘어나지?"
  "VACUUM을 실행하면 크기가 줄어들겠지"
  → VACUUM 실행 → 테이블 크기 그대로

혼란:
  "VACUUM이 Dead Tuple을 청소한다고 했는데 왜 크기가 안 줄지?"

근본 이유 (페이지 구조를 알면 명확):
  PostgreSQL UPDATE = 기존 튜플 xmax 설정 + 새 튜플 삽입
  → 페이지에 OLD 튜플(Dead)과 NEW 튜플이 공존
  → 페이지가 가득 차면 새 페이지 추가
  → 테이블 파일 크기 증가

VACUUM이 하는 일:
  Dead Tuple이 차지하던 공간을 "재사용 가능"으로 표시
  → 다음 INSERT/UPDATE 시 그 공간 활용
  → 하지만 이미 늘어난 파일 크기는 그대로 유지
  → OS에 공간 반환 = VACUUM FULL만 가능 (AccessExclusiveLock 필요)
```

---

## ✨ 올바른 접근 (After — 페이지 구조 이해 후)

```
페이지 내부 흐름을 이해한 운영:

  테이블 크기 모니터링:
    SELECT pg_size_pretty(pg_total_relation_size('orders'));
    → 비정상적 증가 = Dead Tuple 과도 축적 신호

  Dead Tuple 비율 확인:
    SELECT n_live_tup, n_dead_tup,
           round(n_dead_tup * 100.0 / (n_live_tup + n_dead_tup), 1) AS dead_ratio
    FROM pg_stat_user_tables
    WHERE relname = 'orders';

  VACUUM 타이밍:
    dead_ratio > 10~20% → Autovacuum이 처리해야 정상
    Autovacuum이 따라오지 못하면 수동 VACUUM 실행
    실제 파일 크기를 줄이고 싶다면 → pg_repack (운영 중 사용 가능)
    VACUUM FULL은 운영 중 AccessExclusiveLock → 절대 사용 주의

  fillfactor 활용:
    UPDATE가 빈번한 테이블: fillfactor=70 설정
    → 페이지의 30% 여유 공간 확보
    → HOT Update가 같은 페이지 내에서 가능해짐 (인덱스 갱신 불필요)
```

---

## 🔬 내부 동작 원리

### 1. PostgreSQL 페이지 레이아웃

```
PostgreSQL 8KB (8,192 bytes) 페이지 구조:

┌──────────────────────────────────────┐ ← offset 0
│         Page Header (24 bytes)        │
│  pd_lsn       (8 bytes) — WAL LSN    │
│  pd_checksum  (2 bytes) — 체크섬      │
│  pd_flags     (2 bytes) — 플래그      │
│  pd_lower     (2 bytes) ─────────────┼─── Item Pointer 배열 끝 위치
│  pd_upper     (2 bytes) ─────────────┼─── 튜플 데이터 시작 위치
│  pd_special   (2 bytes) ─────────────┼─── Special Space 시작 위치
│  pd_pagesize_version (2 bytes)       │
│  pd_prune_xid (4 bytes) — 가지치기 XID │
└──────────────────────────────────────┘ ← offset 24

┌──────────────────────────────────────┐
│    Item Pointer (Line Pointer) 배열   │
│  ItemId[1]: offset=7980, length=45  │ ← 튜플 1 위치 정보 (4 bytes)
│  ItemId[2]: offset=7930, length=50  │ ← 튜플 2 위치 정보 (4 bytes)
│  ItemId[3]: offset=7880, length=50  │ ← 튜플 3 위치 정보 (4 bytes)
│  ...                                 │
│              ↓ pd_lower              │
│         (여유 공간)                    │
│              ↑ pd_upper              │
│  ...                                 │
│  튜플 3 데이터 (50 bytes)              │
│  튜플 2 데이터 (50 bytes)              │
│  튜플 1 데이터 (45 bytes)              │
└──────────────────────────────────────┘
│  Special Space (인덱스 전용)           │ ← pd_special ~ 8192
│  (Heap Page에서는 비어있음)             │
└──────────────────────────────────────┘ ← offset 8192

핵심 포인터 의미:
  pd_lower: Page Header + Item Pointer 배열의 끝 위치
            → 새 Item Pointer가 추가될 위치
  pd_upper: 튜플 데이터 영역의 시작 위치
            → 새 튜플이 삽입될 위치 (위쪽으로 자람)
  pd_special: 인덱스 페이지에서 사용하는 메타 영역 시작
              Heap Page에서는 pd_special = 8192 (비어있음)

여유 공간:
  Free Space = pd_upper - pd_lower
  여유가 없으면 → VACUUM 후 재사용 또는 새 페이지 추가
```

### 2. 튜플(행) 내부 구조

```
하나의 튜플(행) 구조 — HeapTupleHeader:

┌──────────────────────────────────────┐
│  HeapTupleHeader (23 bytes)           │
│                                      │
│  t_xmin   (4 bytes) — 이 튜플을 생성한 트랜잭션 ID
│  t_xmax   (4 bytes) — 이 튜플을 삭제/갱신한 트랜잭션 ID
│  t_cid    (4 bytes) — Command ID (같은 트랜잭션 내 명령 순서)
│  t_ctid   (6 bytes) — 현재 버전 또는 새 버전의 물리적 위치
│                       (page_number, item_number)
│  t_infomask2 (2 bytes) — 컬럼 수 + 힌트 비트
│  t_infomask  (2 bytes) — 튜플 상태 플래그
│  t_hoff   (1 byte)  — 실제 데이터 시작 오프셋 (NULL 비트맵 포함)
└──────────────────────────────────────┘
│  NULL 비트맵 (컬럼이 NULL인 경우)        │
└──────────────────────────────────────┘
│  실제 컬럼 데이터                       │
│  col1_data | col2_data | col3_data   │
└──────────────────────────────────────┘

주요 필드 설명:

t_xmin:
  이 튜플을 삽입한 트랜잭션의 XID
  INSERT 시 현재 트랜잭션 XID가 기록됨
  가시성 판단: "내 스냅샷에서 t_xmin이 커밋된 상태인가?"

t_xmax:
  이 튜플을 삭제하거나 UPDATE로 대체한 트랜잭션의 XID
  INSERT 직후: t_xmax = 0 (아직 삭제 안 됨)
  DELETE/UPDATE 시: 현재 트랜잭션 XID가 기록됨
  → t_xmax가 설정된 튜플 = Dead Tuple (VACUUM 대상)

t_ctid:
  자기 자신 또는 새 버전의 위치
  INSERT 후: t_ctid = (현재 페이지, 현재 item 번호)
  UPDATE 후: t_ctid = (새 튜플의 페이지, 새 item 번호)
  → 버전 체인(version chain)을 따라가는 포인터

t_infomask 비트:
  HEAP_XMIN_COMMITTED  — xmin 트랜잭션이 커밋됨 (캐시 힌트)
  HEAP_XMIN_INVALID    — xmin 트랜잭션이 롤백됨
  HEAP_XMAX_COMMITTED  — xmax 트랜잭션이 커밋됨
  HEAP_XMAX_INVALID    — xmax 트랜잭션이 롤백됨
  HEAP_HAS_NULL        — NULL 컬럼 존재
  HEAP_UPDATED         — UPDATE로 생성된 버전
  HEAP_HOT_UPDATED     — HOT Update로 업데이트됨
```

### 3. INSERT/UPDATE/DELETE 시 페이지 변화

```
=== INSERT ===

INSERT INTO orders (id, status) VALUES (1, 'PENDING');

Before:
┌──────────────────────────────────┐
│ Header: pd_lower=24, pd_upper=8192 │
│ (비어있는 페이지)                  │
└──────────────────────────────────┘

After:
┌──────────────────────────────────┐
│ Header: pd_lower=28, pd_upper=8147 │
│ ItemId[1]: offset=8147, len=45  │ ← Item Pointer 추가
│                                  │
│            (여유 공간)             │
│                                  │
│  Tuple 1:                        │
│    t_xmin=501, t_xmax=0          │ ← 현재 트랜잭션 XID
│    t_ctid=(0,1)                  │ ← 자기 자신
│    status='PENDING'              │
└──────────────────────────────────┘

=== UPDATE ===

UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
(트랜잭션 XID=502)

Before: Tuple 1 (xmin=501, xmax=0, status='PENDING')

After:
┌──────────────────────────────────┐
│ Header: pd_lower=32, pd_upper=8097 │
│ ItemId[1]: offset=8147, len=45  │
│ ItemId[2]: offset=8097, len=50  │ ← 새 Item Pointer
│                                  │
│  NEW Tuple 2:                    │
│    t_xmin=502, t_xmax=0         │ ← 새 버전
│    t_ctid=(0,2)                  │
│    status='SHIPPED'              │
│                                  │
│  OLD Tuple 1 (Dead):             │
│    t_xmin=501, t_xmax=502       │ ← xmax 설정됨!
│    t_ctid=(0,2)                  │ ← 새 버전 가리킴
│    status='PENDING'              │
└──────────────────────────────────┘

핵심: 두 버전이 페이지에 공존!
  OLD Tuple: Dead (t_xmax=502가 커밋되면)
  NEW Tuple: Alive
  → Dead Tuple이 페이지 공간 차지

=== DELETE ===

DELETE FROM orders WHERE id = 1;
(트랜잭션 XID=503)

After:
  Tuple: t_xmin=501, t_xmax=503   ← xmax만 설정, 물리적 삭제 없음
  → VACUUM이 올 때까지 공간 점유
```

### 4. MySQL InnoDB 페이지와의 구조 비교

```
MySQL InnoDB 페이지 (16KB):

┌────────────────────────────────────┐
│ File Header (38 bytes)              │
│ Page Header (56 bytes)              │
├────────────────────────────────────┤
│  Infimum + Supremum (각 13 bytes)   │ ← 특수 레코드 (경계)
├────────────────────────────────────┤
│  User Records (실제 데이터)          │
│  ← Single-Linked List로 정렬        │
│     (Clustered Index 순서)          │
├────────────────────────────────────┤
│  Free Space                        │
├────────────────────────────────────┤
│  Page Directory (슬롯 배열)          │ ← Binary Search용 sparse 인덱스
├────────────────────────────────────┤
│  File Trailer (8 bytes)             │
└────────────────────────────────────┘

구조 비교:

항목              | PostgreSQL (Heap Page)    | MySQL InnoDB
─────────────────┼──────────────────────────┼──────────────────────
페이지 크기        | 8KB (컴파일 시 변경 가능)  | 16KB (innodb_page_size)
레코드 정렬        | 삽입 순서 (Heap)           | PK 순서 (Clustered Index)
UPDATE 방식       | 새 버전 삽입 (Heap 내)     | 현재 위치 수정 + Undo Log
이전 버전 저장     | 같은 Heap 페이지에        | 별도 Undo Log에
Dead Tuple        | Heap 페이지에 남음         | Undo Log에서 자동 제거
VACUUM 필요성      | 반드시 필요               | 불필요 (Purge Thread가 처리)
정렬된 읽기 성능   | Heap Scan (임의 순서)      | Clustered Index Scan (정렬)
Secondary Index   | 힙 튜플의 TID 참조         | PK 값 참조 (이중 조회)

핵심 차이:
  PostgreSQL: 모든 테이블은 Heap (Primary Key도 별도 B-Tree 인덱스)
              → Clustered Index 없음
              → PK로 조회 = Index Scan + Heap Fetch (이중 I/O)
              → 단, Index-Only Scan으로 Heap Fetch 생략 가능 (Visibility Map 활용)

  MySQL InnoDB: 모든 테이블은 Clustered Index
                → PK 순서로 물리적 저장
                → PK로 조회 = 단일 B-Tree Scan
                → Secondary Index = PK 값 포함 (Covering Index 활용)
```

### 5. Item Pointer가 존재하는 이유

```
Item Pointer(Line Pointer)의 역할:

문제: 튜플이 VACUUM으로 정리되거나 HOT Update로 같은 페이지 내에서 이동할 때
     이미 튜플을 가리키는 인덱스 엔트리가 있을 수 있음

해결: 인덱스는 (page_number, item_number)를 가리키고,
     item_number → 실제 offset은 Item Pointer 배열에서 찾음
     → 튜플이 이동해도 Item Pointer만 업데이트하면 됨

┌─────────────────┐         ┌─────────────────┐
│  B-Tree Index   │         │   Heap Page 0   │
│                 │         │                 │
│  key=1 → (0,1) │────────→│ ItemId[1] ──────┼──→ Tuple offset=7980
│  key=2 → (0,2) │────────→│ ItemId[2] ──────┼──→ Tuple offset=7930
└─────────────────┘         └─────────────────┘

VACUUM 후 Dead Tuple 제거 시:
  ItemId[1].lp_flags = LP_DEAD    ← 사용 불가 표시
  또는 ItemId[1] 재활용을 위해 lp_flags = LP_UNUSED 설정

HOT Update 시:
  OLD 튜플의 t_ctid = 새 튜플 위치
  ItemId는 OLD 튜플을 여전히 가리키나
  → PostgreSQL이 t_ctid 체인을 따라가 최신 버전 반환
  (인덱스 업데이트 없이 버전 체인만 업데이트)

Result: 인덱스 엔트리를 수정하지 않아도
        Item Pointer + t_ctid 체인으로 최신 버전을 찾을 수 있음
```

---

## 💻 실전 실험

### 실험 1: pageinspect로 페이지 내부 직접 관찰

```sql
-- pageinspect 확장 설치
CREATE EXTENSION pageinspect;

-- 실험용 테이블 생성
CREATE TABLE page_test (
    id SERIAL PRIMARY KEY,
    status TEXT,
    data TEXT
);

-- 데이터 삽입
INSERT INTO page_test (status, data) VALUES
    ('PENDING', 'first row'),
    ('ACTIVE', 'second row'),
    ('DONE', 'third row');

-- 페이지 헤더 확인
SELECT * FROM page_header(get_raw_page('page_test', 0));
-- 출력:
-- lsn       | checksum | flags | lower | upper | special | pagesize | version | prune_xid
-- 0/1234560 |     0    |   0   |  56   | 8056  |  8192   |   8192   |    4    |     0

-- 튜플 내용 확인 (t_xmin, t_xmax, t_ctid)
SELECT
    t_ctid,
    t_xmin,
    t_xmax,
    t_infomask::bit(16),
    t_data
FROM heap_page_items(get_raw_page('page_test', 0));

-- UPDATE 후 Dead Tuple 관찰
UPDATE page_test SET status = 'SHIPPED' WHERE id = 1;

SELECT
    t_ctid,
    t_xmin,
    t_xmax,
    CASE WHEN t_xmax = 0 THEN 'ALIVE' ELSE 'DEAD' END AS tuple_state
FROM heap_page_items(get_raw_page('page_test', 0));
-- id=1의 OLD 버전: t_xmax = 현재 트랜잭션 XID (Dead)
-- id=1의 NEW 버전: t_xmax = 0 (Alive)
```

### 실험 2: 페이지 여유 공간 확인 (FSM)

```sql
-- pg_freespacemap 확장 설치
CREATE EXTENSION pg_freespacemap;

-- 테이블 각 페이지의 여유 공간 확인
SELECT
    blkno AS page_number,
    avail AS free_bytes,
    round(avail * 100.0 / 8192, 1) AS free_pct
FROM pg_freespace('page_test')
ORDER BY blkno;

-- VACUUM 전후 비교
VACUUM page_test;

SELECT
    blkno AS page_number,
    avail AS free_bytes,
    round(avail * 100.0 / 8192, 1) AS free_pct
FROM pg_freespace('page_test')
ORDER BY blkno;
-- Dead Tuple이 정리된 페이지의 free_bytes 증가 확인
```

### 실험 3: 실제 테이블 페이지 수 및 크기

```sql
-- 테이블 파일 크기와 페이지 수
SELECT
    relname,
    relpages AS total_pages,
    pg_size_pretty(relpages * 8192::bigint) AS estimated_size,
    pg_size_pretty(pg_relation_size(oid)) AS actual_size
FROM pg_class
WHERE relname = 'page_test';

-- 대량 UPDATE 후 Table Bloat 관찰
-- 10,000행 INSERT
INSERT INTO page_test (status, data)
SELECT 'PENDING', md5(random()::text)
FROM generate_series(1, 10000);

-- UPDATE 전 크기
SELECT pg_size_pretty(pg_relation_size('page_test')) AS before_update;

-- 전체 행 UPDATE (Dead Tuple 대량 생성)
UPDATE page_test SET status = 'DONE';

-- UPDATE 후 크기 (약 2배)
SELECT pg_size_pretty(pg_relation_size('page_test')) AS after_update;

-- VACUUM 후 크기 (여전히 큼 - OS에 반환 안 함)
VACUUM page_test;
SELECT pg_size_pretty(pg_relation_size('page_test')) AS after_vacuum;
```

---

## 📊 MySQL과 비교

```
같은 INSERT 1만 건 후 UPDATE 1만 건 실행:

PostgreSQL:
  INSERT 1만 건:
    → Heap 페이지에 1만 개 튜플
    → 테이블 크기: ~5MB (튜플당 500바이트 가정)

  UPDATE 1만 건:
    → OLD 튜플(Dead) 1만 개 + NEW 튜플 1만 개
    → 테이블 크기: ~10MB (2배)

  VACUUM 후:
    → Dead Tuple 공간 "재사용 가능" 표시
    → 파일 크기: ~10MB (그대로, OS에 반환 안 함)
    → 다음 INSERT는 회수된 공간 재사용

MySQL InnoDB:
  INSERT 1만 건:
    → Clustered Index 페이지에 1만 개 레코드
    → 테이블 크기: ~5MB

  UPDATE 1만 건:
    → 현재 레코드를 직접 수정
    → OLD 버전은 Undo Log에 저장 (별도 파일)
    → 테이블 크기: ~5MB (증가 없음!)
    → Undo Log: 추가 공간 사용, Purge Thread가 정리

  결론:
    MySQL: 테이블 파일 크기는 안정적, Undo Log 별도 관리
    PostgreSQL: 테이블 파일이 커지나 VACUUM으로 재사용 가능 처리
                Table Bloat을 관리하지 않으면 지속 증가
```

---

## ⚖️ 트레이드오프

```
Heap 기반 저장 (PostgreSQL)의 장단점:

장점:
  ① 다중 버전 공존: 같은 데이터의 여러 버전을 Heap에 보존
                   → 읽기 트랜잭션이 잠금 없이 오래된 버전 읽기 가능
  ② Clustered 강제 없음: PK 순서와 물리적 순서 무관
                         → 대량 INSERT 시 핫스팟(hot page) 문제 없음
  ③ Update-in-place 없음: 쓰기 충돌이 적음

단점:
  ① Table Bloat: UPDATE/DELETE 시 Dead Tuple이 Heap에 쌓임
                 → VACUUM 없이는 테이블 크기 계속 증가
  ② PK 조회 이중 I/O: Index → Heap Fetch (MySQL Clustered Index 대비 불리)
  ③ VACUUM 운영 부담: Autovacuum 튜닝, 모니터링 필요

fillfactor 활용으로 단점 완화:
  UPDATE 빈번한 테이블: fillfactor=70~80
  → 각 페이지의 20~30%를 빈 채로 남겨둠
  → 같은 페이지에 새 버전 삽입 가능 (HOT Update 가능)
  → 인덱스 갱신 불필요 → 쓰기 성능 향상
  → 단, 테이블이 차지하는 페이지 수 증가 (읽기 약간 불리)
```

---

## 📌 핵심 정리

```
PostgreSQL 8KB 페이지 핵심:

구조:
  Page Header(24B) → Item Pointer 배열 → [여유공간] → 튜플 데이터

포인터:
  pd_lower: Item Pointer 배열 끝 (새 IP 추가 위치)
  pd_upper: 튜플 데이터 시작 (새 튜플 삽입 위치, 아래로 자람)
  pd_special: 인덱스 전용 메타 영역 (Heap에서는 비어있음)

튜플 헤더:
  t_xmin: 이 버전을 생성한 트랜잭션 XID
  t_xmax: 이 버전을 종료한 트랜잭션 XID (0이면 살아있음)
  t_ctid: 자신 또는 최신 버전의 물리적 위치

UPDATE의 실체:
  OLD 튜플: t_xmax 설정 → Dead Tuple
  NEW 튜플: 새로 삽입
  두 버전이 Heap 페이지에 공존 → VACUUM이 회수

MySQL과 차이:
  MySQL: UPDATE = 현재 위치 수정 + Undo Log 저장
  PostgreSQL: UPDATE = 새 버전 삽입 + OLD 버전 Dead 처리
```

---

## 🤔 생각해볼 문제

**Q1.** `fillfactor=70`으로 설정된 테이블에서 HOT Update가 가능한 이유는 무엇인가? `fillfactor=100`(기본값)에서는 왜 HOT Update가 불리한가?

<details>
<summary>해설 보기</summary>

HOT(Heap Only Tuple) Update는 새 버전이 **같은 페이지** 안에 삽입될 수 있을 때만 가능합니다. `fillfactor=100`이면 페이지를 꽉 채워 삽입하므로, UPDATE 시 새 버전을 넣을 공간이 없어 **다른 페이지**에 삽입됩니다. 이 경우 인덱스도 새 위치를 가리키도록 갱신해야 합니다.

`fillfactor=70`이면 페이지의 70%만 채우고 나머지 30%를 비워두므로, UPDATE 시 OLD 버전은 xmax가 설정되고 NEW 버전이 **같은 페이지의 빈 공간**에 삽입됩니다. 인덱스는 OLD 버전의 Item Pointer를 여전히 가리키고, PostgreSQL이 t_ctid 체인을 따라 NEW 버전을 찾으므로 인덱스 갱신이 불필요합니다.

```sql
-- fillfactor 설정
ALTER TABLE orders SET (fillfactor = 70);
-- 이후 VACUUM FULL 또는 CLUSTER로 재구성

-- HOT Update 비율 확인
SELECT n_tup_upd, n_tup_hot_upd,
       round(n_tup_hot_upd * 100.0 / nullif(n_tup_upd, 0), 1) AS hot_pct
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

</details>

---

**Q2.** `pd_ctid`에서 `t_ctid = (0, 2)`가 의미하는 것은 정확히 무엇인가? UPDATE 체인이 3번 이상 발생하면 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

`t_ctid = (page_number, item_number)`입니다. `(0, 2)`는 "0번 페이지의 2번 Item"을 의미합니다. 자기 자신을 가리키면 최신 버전입니다.

UPDATE 체인이 3번 발생하면:
```
Tuple 1 (t_ctid=(0,2)) → Tuple 2 (t_ctid=(0,3)) → Tuple 3 (t_ctid=(0,3) 자기자신)
```

PostgreSQL은 t_ctid를 따라가며 자기 자신을 가리키는 튜플(최신 버전)을 찾습니다. 단, HOT 체인이 다른 페이지를 넘어가면 HOT이 아닙니다. VACUUM이 오면 중간의 Dead Tuple들을 제거하고 Item Pointer를 정리합니다(LP_REDIRECT).

</details>

---

**Q3.** `pageinspect`의 `heap_page_items`에서 `t_infomask`의 `HEAP_XMIN_COMMITTED` 힌트 비트는 왜 중요한가?

<details>
<summary>해설 보기</summary>

PostgreSQL이 튜플을 읽을 때마다 `t_xmin`이 커밋됐는지 확인하려면 `pg_xact`(CLOG)를 조회해야 합니다. 이는 I/O 비용이 발생합니다. `HEAP_XMIN_COMMITTED` 힌트 비트는 "이 xmin이 이미 커밋된 것을 확인했다"는 캐시입니다.

한 번 커밋 여부를 확인하고 힌트 비트를 설정하면, 이후 읽기 시 CLOG 조회 없이 힌트 비트만 확인합니다. 이를 **"힌트 비트(hint bits)"**라 하며, 쓰기 시 WAL에 기록되지 않습니다(dirty page로만 표시). 따라서 WAL replay 시 힌트 비트는 재설정됩니다.

힌트 비트가 없으면 모든 튜플 읽기마다 CLOG I/O가 발생해 성능이 크게 저하됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: PostgreSQL vs MySQL 아키텍처](./01-architecture-vs-mysql.md)** | **[홈으로 🏠](../README.md)** | **[다음: PostgreSQL MVCC vs MySQL MVCC ➡️](./03-mvcc-vs-mysql-mvcc.md)**

</div>
