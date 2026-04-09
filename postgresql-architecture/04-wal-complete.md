# WAL(Write-Ahead Log) 완전 분해 — PostgreSQL 내구성과 복제의 기반

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- WAL은 왜 "Write-Ahead"인가? 데이터를 디스크에 쓰기 전에 WAL을 먼저 쓰는 이유는?
- WAL 레코드 하나에는 정확히 무엇이 담겨 있는가?
- LSN(Log Sequence Number)은 어떤 구조이고 어디에 사용되는가?
- Checkpoint는 언제 발생하고, 크래시 복구 시 WAL의 어디서부터 재실행하는가?
- PostgreSQL Streaming Replication은 WAL을 어떻게 활용하고, MySQL Binary Log와 어떻게 다른가?
- `pg_waldump`로 WAL 레코드를 직접 분석하는 방법은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

WAL은 PostgreSQL의 내구성(Durability), 크래시 복구, Streaming Replication, Logical Replication, Point-in-Time Recovery(PITR)의 **모든 기반**이다. `synchronous_commit` 설정 하나가 "트랜잭션 커밋 = WAL이 디스크에 기록됨"의 보장 수준을 결정하고, `wal_level` 설정이 복제 방식을 결정한다. WAL을 이해하지 못하면 복제 지연, 크래시 후 데이터 손실 범위, checkpoint 과부하를 진단하기 어렵다.

---

## 😱 흔한 실수 (Before — WAL을 블랙박스로 둘 때)

```
흔한 실수 1: synchronous_commit 오해

  "트랜잭션이 COMMIT 됐으니 데이터는 무조건 안전하다"

  실제:
    synchronous_commit = off 설정 시
    COMMIT 반환 시점 ≠ WAL이 디스크에 기록된 시점
    → 서버 크래시 시 최근 수십ms 트랜잭션 유실 가능

  언제 써야 하는가:
    결제, 주문 등 데이터 손실 절대 불가 → synchronous_commit = on (기본값)
    로그, 통계 등 약간의 손실 허용 → synchronous_commit = off (성능 2~3배 향상)

흔한 실수 2: WAL 파일 수동 삭제

  "WAL 파일이 /var/lib/postgresql/pg_wal 에 쌓여 디스크가 꽉 참"
  → pg_wal 디렉토리의 파일을 수동 삭제

  결과:
    복제 중인 Standby가 참조하는 WAL 삭제 → Standby 복제 중단
    Point-in-Time Recovery 불가
    최악의 경우 Primary도 영향

  올바른 방법:
    pg_archivecleanup 사용 (아카이브된 WAL 정리)
    Replication Slot 모니터링 (WAL 파일 보존 원인 확인)
    max_wal_size, checkpoint_completion_target 튜닝

흔한 실수 3: wal_level 낮게 설정 후 복제 시도

  wal_level = minimal 설정 → "WAL 최소화로 성능 향상"
  → Streaming Replication 불가 (replica 이상 필요)
  → 나중에 복제 추가 시 WAL 레벨 변경 + 재시작 필요
```

---

## ✨ 올바른 접근 (After — WAL 이해 후)

```
WAL 이해 기반 운영:

  wal_level 결정:
    Streaming Replication → wal_level = replica (기본값)
    Logical Replication / CDC → wal_level = logical
    단독 운영 (복제 불필요) → wal_level = minimal (성능 약간 향상)

  synchronous_commit 전략:
    기본값 on: 중요 트랜잭션 (결제, 주문)
    off: 덜 중요한 쓰기 (로그, 방문 기록)
    remote_write: Standby의 OS 버퍼까지 전달 보장
    remote_apply: Standby에 실제 적용 완료 보장

  WAL 크기 모니터링:
    SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0'));
    SELECT count(*), pg_size_pretty(sum(size)) FROM pg_ls_waldir();

  Checkpoint 튜닝:
    checkpoint_completion_target = 0.9  -- I/O를 체크포인트 간격의 90%에 분산
    max_wal_size = 1GB                   -- Checkpoint 간격 조절
```

---

## 🔬 내부 동작 원리

### 1. WAL의 기본 원리 — Write-Ahead의 의미

```
WAL 없는 세계에서의 크래시:

  BEGIN;
  UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
  COMMIT;
  → shared_buffers의 Dirty Page (수정된 8KB 페이지)를 디스크에 씀
  → "OK" 반환

  서버 크래시 시:
    shared_buffers = 메모리 → 날아감
    Dirty Page가 디스크에 쓰이는 중 크래시 → 반쪽만 쓰인 페이지 (Torn Page)
    데이터 손상!

Write-Ahead Log의 원칙:
  "데이터 페이지를 디스크에 쓰기 전에, 반드시 WAL에 먼저 기록한다"

  BEGIN;
  UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
  ① shared_buffers의 페이지 수정 (Dirty)
  ② WAL 레코드 생성 → WAL Buffer에 추가

  COMMIT;
  ③ WAL Buffer → WAL 파일 (pg_wal/)에 fsync
  ④ "OK" 반환 (아직 데이터 페이지는 디스크에 없어도 됨!)

  크래시 복구:
  ⑤ WAL 파일 읽기 → 마지막 Checkpoint 이후 레코드 재실행
  ⑥ Dirty Page를 다시 생성 → 정상 상태 복구

핵심:
  WAL이 디스크에 있으면 데이터 페이지가 없어도 복구 가능
  WAL 쓰기 = 순차 I/O (빠름)
  데이터 페이지 쓰기 = 임의 I/O (느림) → 나중에 백그라운드에서 처리
  → 성능 + 내구성 동시 달성
```

### 2. WAL 레코드 구조

```
WAL 파일 위치: $PGDATA/pg_wal/

WAL 파일 이름 형식:
  000000010000000000000001
  ├── 00000001: Timeline ID (복제/복구 시 분기 추적)
  ├── 00000000: LSN 상위 32비트 (세그먼트 번호)
  └── 00000001: LSN 하위 32비트 (세그먼트 내 오프셋)

WAL 세그먼트 크기: 기본 16MB (컴파일 시 설정)

LSN (Log Sequence Number):
  64비트 정수: WAL 파일 내 바이트 오프셋
  표현: 0/1A2B3C4D (상위 32비트/하위 32비트, 16진수)

  예시:
    0/01000000 = WAL 파일의 16MB 지점
    0/02000000 = WAL 파일의 32MB 지점

WAL 레코드 내부 구조:

  ┌─────────────────────────────────────────┐
  │  XLogRecord Header (24 bytes)           │
  │  xl_tot_len: 레코드 전체 길이             │
  │  xl_xid:    트랜잭션 ID                  │
  │  xl_prev:   이전 WAL 레코드의 LSN        │
  │  xl_info:   레코드 타입 (UPDATE/INSERT/...) │
  │  xl_rmid:   Resource Manager ID        │
  │  xl_crc:    CRC32 체크섬               │
  └─────────────────────────────────────────┘
  │  XLogRecordBlockHeader (블록 참조)       │
  │  rnode: 테이블스페이스/DB/파일 식별자      │
  │  blkno: 수정된 8KB 페이지 번호           │
  │  bimg:  Full Page Image (FPI) 포함 여부  │
  └─────────────────────────────────────────┘
  │  데이터 영역                              │
  │  변경된 바이트들 (또는 Full Page Image)    │
  └─────────────────────────────────────────┘

Resource Manager (rmid):
  각 PostgreSQL 모듈이 WAL에 기록하는 주체
  XLOG:    트랜잭션 제어 (COMMIT, ABORT, CHECKPOINT)
  Heap:    테이블 행 변경 (INSERT, UPDATE, DELETE)
  Btree:   B-Tree 인덱스 변경
  Hash:    Hash 인덱스 변경
  Gin:     GIN 인덱스 변경
  Brin:    BRIN 인덱스 변경
  Sequence: 시퀀스 값 변경

Full Page Image (FPI):
  Checkpoint 이후 페이지가 처음 변경될 때:
  → 페이지 전체를 WAL에 포함 (Torn Page 방지)
  → 이후 같은 페이지의 변경: delta만 기록
  → wal_compression = on: FPI를 압축해 WAL 크기 감소
```

### 3. Checkpoint 동작

```
Checkpoint의 역할:
  "이 LSN 이전의 모든 Dirty Page는 디스크에 안전하게 기록됨"
  → 크래시 복구 시 이 Checkpoint LSN 이후 WAL만 재실행하면 됨

Checkpoint 발생 조건:
  ① 시간: checkpoint_timeout (기본 5분)마다
  ② 크기: WAL 크기가 max_wal_size (기본 1GB)에 도달 시
  ③ 수동: CHECKPOINT 명령어

Checkpoint 수행 과정:
  1. Checkpoint 시작 WAL 레코드 기록 (REDO point 기록)
  2. shared_buffers의 Dirty Page들을 디스크에 점진적으로 씀
     (checkpoint_completion_target 비율에 맞춰 I/O 분산)
  3. Checkpoint 완료 WAL 레코드 기록
  4. pg_control 파일에 Checkpoint LSN 업데이트

크래시 복구 흐름:

  크래시 발생
     ↓
  PostgreSQL 재시작
     ↓
  pg_control 파일에서 마지막 Checkpoint LSN 읽기
     ↓
  해당 LSN 이후 WAL 레코드 순서대로 재실행 (Redo Phase)
     ↓
  정상 상태 복구

Checkpoint 튜닝:

  checkpoint_completion_target = 0.9 (기본 0.5)
    → Checkpoint 기간의 90% 동안 분산해서 Dirty Page 기록
    → 0.5이면 50% 시간에 몰아서 기록 → I/O 스파이크 발생

  max_wal_size = 1~4GB (기본 1GB)
    → 값이 클수록 Checkpoint 간격 길어짐 → 복구 시간 증가
    → 값이 작을수록 Checkpoint 빈번 → I/O 부하 증가

  Checkpoint 경고:
  LOG:  checkpoint request for checkpoint of type "time" taking 45.2 s
  → Checkpoint가 checkpoint_warning (기본 30초)보다 오래 걸리면 경고
  → max_wal_size 증가 또는 I/O 장비 업그레이드 필요
```

### 4. Streaming Replication — WAL 기반 복제

```
Streaming Replication 구조:

Primary                          Standby
─────────────────────            ─────────────────────
트랜잭션 실행                      WAL 수신 및 적용
      │                                │
      ▼                                │
WAL Buffer에 레코드 추가               │
      │                                │
      ▼                                │
WAL 파일 (pg_wal/) 기록               │
      │                                │
      ▼                                │
WAL Sender 프로세스               WAL Receiver 프로세스
      │                                │
      └──── TCP 스트리밍 ──────────────┘
                                       │
                              WAL Receiver가 수신
                                       │
                              pg_wal/에 저장
                                       │
                              Startup 프로세스가 적용
                                       │
                              Standby 데이터에 반영

복제 흐름:
  1. Primary에서 트랜잭션 커밋 → WAL 레코드 생성
  2. WAL Sender: pg_wal에서 Standby로 WAL 스트리밍
  3. Standby WAL Receiver: 수신한 WAL을 로컬 pg_wal에 저장
  4. Standby Startup 프로세스: WAL 레코드를 순서대로 재실행
  5. Standby 데이터가 Primary와 동기화

동기(Synchronous) vs 비동기(Asynchronous) 복제:

  비동기 (기본값):
    Primary: COMMIT → WAL 로컬 기록 → OK 반환 (Standby 확인 안 함)
    → Primary 크래시 시 Standby에 복제 안 된 트랜잭션 유실 가능

  동기:
    synchronous_standby_names = 'standby1'
    Primary: COMMIT → WAL 로컬 기록 → Standby 확인 대기 → OK 반환
    → 데이터 유실 없음 (단, 지연 발생)

Replication Slot:
  Standby가 WAL을 다 받아가기 전에 Primary가 WAL 삭제하는 것 방지
  SELECT * FROM pg_replication_slots;
  → slot이 inactive인데 남아있으면 WAL 파일이 계속 쌓임 주의!
  → 미사용 Slot 삭제: SELECT pg_drop_replication_slot('slot_name');

복제 지연 모니터링:
  SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes
  FROM pg_stat_replication;
```

### 5. MySQL Binary Log와의 비교

```
PostgreSQL WAL vs MySQL Binary Log:

항목              | PostgreSQL WAL          | MySQL Binary Log
─────────────────┼─────────────────────────┼─────────────────────────
기록 수준         | 물리적 (페이지 변경)         | 논리적 (SQL 또는 행 변경)
기본 목적         | 크래시 복구 (복제는 부가)    | 복제 (크래시 복구는 Redo Log)
크기             | 크다 (Full Page Image 포함)  | 작다 (논리 변경만)
복제 방식         | 물리 복제 (WAL 스트리밍)     | 논리 복제 (SQL/Row 재실행)
이기종 복제        | 불가 (같은 버전 필요)        | 가능 (MySQL 버전 간)
포맷             | 바이너리 (PostgreSQL 전용)   | ROW/STATEMENT/MIXED

MySQL WAL의 이중 로그 문제:
  - InnoDB Redo Log: 물리적 변경 → 크래시 복구
  - Binary Log: 논리적 변경 → 복제, PITR
  → 동일 트랜잭션이 두 로그에 모두 기록 (2PC로 원자성 보장)
  → 쓰기 증폭 발생

PostgreSQL WAL 단일 로그 장점:
  - WAL 하나로 크래시 복구 + 복제 + PITR 모두 처리
  - 쓰기 증폭 없음

Logical Decoding (논리 복제):
  PostgreSQL: wal_level = logical → WAL에서 SQL 수준 변경 추출
  → Publication/Subscription 또는 pgoutput 플러그인 사용
  → CDC (Change Data Capture): Debezium + PostgreSQL Logical Replication
```

---

## 💻 실전 실험

### 실험 1: pg_waldump로 WAL 레코드 분석

```bash
# 현재 WAL 파일 위치 확인
psql -c "SELECT pg_walfile_name(pg_current_wal_lsn());"
# 출력: 000000010000000000000001

# WAL 파일 목록
ls -la $PGDATA/pg_wal/

# WAL 레코드 분석 (실험용 트랜잭션 실행 후)
psql -c "CREATE TABLE wal_test (id INT, val TEXT);
         INSERT INTO wal_test VALUES (1, 'hello');"

# pg_waldump로 WAL 레코드 확인
pg_waldump -n 20 $PGDATA/pg_wal/000000010000000000000001
# 출력 예시:
# rmgr: Heap        len (rec/tot):     59/    59, tx:        501, lsn: 0/01234560
#   desc: INSERT off 1, blkref #0: rel 1663/16384/16385 blk 0
# rmgr: Transaction len (rec/tot):     46/    46, tx:        501, lsn: 0/01234598
#   desc: COMMIT 2024-01-01 12:00:00.123456 UTC

# 특정 LSN 범위만 보기
pg_waldump --start=0/01234000 --end=0/01235000 $PGDATA/pg_wal/000000010000000000000001
```

### 실험 2: WAL 생성량 측정

```sql
-- 현재 LSN 확인
SELECT pg_current_wal_lsn() AS current_lsn;

-- 대량 작업 실행
INSERT INTO wal_test SELECT i, md5(i::text) FROM generate_series(1, 100000) i;

-- WAL 생성량 측정
SELECT
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), '0/01234560')
    ) AS wal_generated;

-- Full Page Image 비율 확인
SELECT * FROM pg_stat_bgwriter;
-- buffers_checkpoint: Checkpoint가 기록한 페이지 수
-- buffers_clean: Background Writer가 기록한 페이지 수
-- buffers_backend: Backend가 직접 기록한 페이지 수 (많으면 문제)
```

### 실험 3: Checkpoint 동작 관찰

```sql
-- 현재 Checkpoint 정보
SELECT
    checkpoint_lsn,
    prior_lsn,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), checkpoint_lsn)) AS wal_since_checkpoint,
    checkpoint_time,
    now() - checkpoint_time AS time_since_checkpoint
FROM pg_control_checkpoint();

-- Checkpoint 강제 실행
CHECKPOINT;

-- Checkpoint 후 정보 다시 확인
SELECT checkpoint_lsn, checkpoint_time FROM pg_control_checkpoint();

-- WAL 파일 수 및 크기
SELECT count(*) AS wal_files,
       pg_size_pretty(sum(size)) AS total_wal_size
FROM pg_ls_waldir();
```

### 실험 4: Streaming Replication 지연 측정

```sql
-- Primary에서 실행
SELECT
    application_name,
    state,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag_size,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

-- Standby에서 실행 (복제 수신 상태)
SELECT
    pg_is_in_recovery() AS is_standby,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn,
    pg_wal_lsn_diff(
        pg_last_wal_receive_lsn(),
        pg_last_wal_replay_lsn()
    ) AS apply_lag_bytes,
    pg_last_xact_replay_timestamp() AS last_replay_time,
    now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

---

## 📊 MySQL과 비교

```
동일 트랜잭션의 WAL/Log 기록 비교:

BEGIN;
UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
COMMIT;

PostgreSQL WAL:
  레코드 1: Heap UPDATE (OLD 버전 + NEW 버전, 또는 페이지 변경 delta)
  레코드 2: COMMIT (트랜잭션 XID, 타임스탬프)
  → 2개 레코드, 단일 WAL

MySQL:
  InnoDB Redo Log:
    변경 물리 기록 (Buffer Pool의 페이지 변경)

  Binary Log (binlog_format=ROW):
    Table_map event (테이블 정보)
    Update_rows event (BEFORE + AFTER 이미지)
    Xid_event (트랜잭션 커밋)

  2PC (Two-Phase Commit):
    1. Redo Log에 PREPARE 기록
    2. Binary Log에 기록
    3. Redo Log에 COMMIT 기록
    → 원자성 보장을 위한 2단계 커밋 오버헤드

크기 비교 (100만 행 INSERT):
  PostgreSQL WAL: ~200MB (Full Page Image 포함 시 더 클 수 있음)
  MySQL Redo Log: ~100MB
  MySQL Binary Log (ROW): ~150MB
  MySQL 합계: ~250MB (이중 로그 부담)

PostgreSQL WAL 압축:
  wal_compression = on 설정 시 FPI를 압축
  → WAL 크기 20~40% 감소
  → CPU 부하 약간 증가 (압축 비용)
```

---

## ⚖️ 트레이드오프

```
WAL 레벨별 트레이드오프:

wal_level = minimal:
  장점: WAL 크기 최소 (일부 DDL 후 WAL 기록 생략)
  단점: Streaming Replication 불가, WAL Archiving 불가
  적합: 단독 개발/테스트 환경

wal_level = replica (기본값):
  장점: Streaming Replication 가능, PITR 가능
  단점: minimal보다 WAL 크다
  적합: 대부분의 운영 환경

wal_level = logical:
  장점: Logical Replication, CDC 가능
  단점: WAL 가장 크다 (논리 변경 정보 추가)
  적합: Debezium, Kafka Connect, 이기종 복제 필요 시

synchronous_commit 트레이드오프:

  on (기본값):
    COMMIT = WAL이 로컬 디스크에 기록됨 보장
    성능: 기준
    데이터 손실: 없음

  off:
    COMMIT = WAL이 WAL Buffer에 있음 (디스크 미보장)
    성능: 2~3배 향상 (fsync 대기 없음)
    데이터 손실: 크래시 시 최근 wal_writer_delay(200ms) 트랜잭션 손실 가능

  remote_write:
    Standby의 OS 버퍼까지 전달 보장 (Standby fsync 미보장)
    Primary 크래시 후 Standby 승격 시 안전

  remote_apply:
    Standby에 실제 적용 완료 보장
    Standby에서 즉시 읽기 가능 → 읽기 일관성 보장
    성능: 복제 지연만큼 COMMIT 대기
```

---

## 📌 핵심 정리

```
PostgreSQL WAL 핵심:

Write-Ahead 원칙:
  데이터 페이지 디스크 기록 전에 WAL을 먼저 fsync
  WAL이 있으면 크래시 후 복구 가능 (Redo Phase)

WAL 레코드 구성:
  Resource Manager (rmid) + 변경 내용 + LSN 포인터
  Full Page Image: Checkpoint 후 첫 변경 시 페이지 전체 포함

LSN:
  64비트 WAL 파일 내 절대 오프셋
  복제 지연 = Primary LSN - Standby replay LSN

Checkpoint:
  모든 Dirty Page가 디스크에 기록된 시점 표시
  크래시 복구 시 마지막 Checkpoint 이후 WAL만 재실행

Streaming Replication:
  WAL Sender → WAL Receiver → Startup 프로세스 적용
  비동기(기본) vs 동기(synchronous_standby_names)

MySQL Binary Log와 차이:
  PostgreSQL WAL: 단일 로그로 복구 + 복제
  MySQL: Redo Log(복구) + Binary Log(복제) 이중 로그
```

---

## 🤔 생각해볼 문제

**Q1.** `synchronous_commit = off`인 상태에서 서버가 크래시되면 정확히 어느 트랜잭션까지 손실될 수 있는가?

<details>
<summary>해설 보기</summary>

`synchronous_commit = off`일 때 `COMMIT`은 WAL Buffer에 기록된 직후 반환됩니다. WAL Buffer는 `wal_writer_delay`(기본 200ms) 주기로 WAL 파일에 flush됩니다. 따라서 크래시 시 최근 **최대 200ms** 내의 트랜잭션이 손실될 수 있습니다.

단, 이는 "커밋됐다고 응답을 받았음에도 손실"을 의미합니다. 즉, **Durability(내구성)**를 완화하는 설정입니다. 결제, 주문 등 절대 손실이 없어야 하는 트랜잭션에는 사용하면 안 됩니다.

로그, 클릭 통계 등 "약간의 손실은 허용 가능하지만 처리량이 중요한" 경우에 테이블 단위로 적용 가능합니다:
```sql
ALTER TABLE click_logs SET (synchronous_commit = off);
```

</details>

---

**Q2.** Full Page Image(FPI)는 WAL 크기를 크게 만드는 원인인데, 왜 반드시 필요한가? `wal_init_zero`와는 어떤 관계인가?

<details>
<summary>해설 보기</summary>

FPI는 **Torn Page 문제**를 방지하기 위해 필요합니다. 8KB 페이지를 디스크에 쓰는 도중 크래시가 발생하면, 페이지의 일부만 새 버전이고 나머지는 구 버전인 "찢어진 페이지(Torn Page)"가 생길 수 있습니다.

Checkpoint 이후 처음으로 변경되는 페이지는 그 전체를 WAL에 포함합니다. 이후 같은 페이지의 추가 변경은 delta(변경 부분)만 기록합니다. 크래시 복구 시 FPI로 페이지를 안전한 상태로 복원한 뒤 이후 delta를 적용합니다.

`wal_init_zero`는 WAL 파일을 미리 0으로 초기화하는 옵션입니다. FPI와는 직접 관련 없으며, 파일시스템이 새 WAL 세그먼트 생성 시 임의의 바이트를 읽는 것을 방지합니다. SSD 환경에서는 `wal_init_zero = off`로 WAL 세그먼트 생성 속도를 높일 수 있습니다.

FPI 크기를 줄이려면: `wal_compression = on` 설정으로 FPI를 LZ4/lz4hc로 압축할 수 있습니다.

</details>

---

**Q3.** Replication Slot이 없을 때와 있을 때, Standby가 느려질 경우 Primary WAL 파일에 어떤 차이가 생기는가?

<details>
<summary>해설 보기</summary>

**Replication Slot 없을 때:**
Primary는 Standby가 WAL을 다 받았는지 추적하지 않습니다. WAL 파일은 Checkpoint와 `max_wal_size`에 따라 주기적으로 삭제됩니다. Standby가 느리면 아직 받지 못한 WAL이 Primary에서 삭제될 수 있습니다. 이 경우 Standby는 "requested WAL segment ... has already been removed"  오류와 함께 복제 중단됩니다. 해결하려면 Standby를 처음부터 다시 동기화해야 합니다.

**Replication Slot 있을 때:**
Primary는 Slot의 `restart_lsn`(Standby가 필요로 하는 최소 LSN)을 추적합니다. Standby가 아직 처리하지 못한 WAL은 삭제하지 않습니다. 이로 인해 Standby가 오래 느리면 **Primary의 pg_wal 디렉토리가 무한히 커질 수 있습니다(디스크 풀 위험!)**.

모니터링:
```sql
SELECT slot_name, pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
) AS retained_wal
FROM pg_replication_slots;
```
retained_wal이 수 GB 이상이면 해당 Slot의 Standby를 점검해야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: PostgreSQL MVCC vs MySQL MVCC](./03-mvcc-vs-mysql-mvcc.md)** | **[홈으로 🏠](../README.md)** | **[다음: 트랜잭션 ID(XID)와 가시성 규칙 ➡️](./05-xid-visibility.md)**

</div>
