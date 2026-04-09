# Streaming Replication 완전 분해 — WAL 스트리밍과 동기/비동기 복제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Primary가 WAL을 Standby에 전달하는 정확한 단계는?
- 동기(Synchronous) 복제와 비동기(Asynchronous) 복제의 차이와 선택 기준은?
- Replication Slot은 언제 필요하고, 관리하지 않으면 어떤 위험이 있는가?
- `pg_stat_replication`으로 복제 지연을 어떻게 측정하는가?
- `wal_level`, `max_wal_senders`, `hot_standby` 설정의 역할은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

Streaming Replication은 PostgreSQL 고가용성의 기반이다. Primary 장애 시 Standby를 Promote해 서비스를 유지하고, Hot Standby로 읽기 트래픽을 분산한다. 동기/비동기 복제 선택이 데이터 손실 허용 범위를 결정하고, Replication Slot 미관리는 Primary 디스크를 가득 채울 수 있다. WAL 스트리밍의 내부 동작을 이해해야 복제 지연 원인을 진단하고 올바른 설정을 결정할 수 있다.

---

## 😱 흔한 실수 (Before — Streaming Replication 운영 실수)

```
실수 1: Replication Slot 무한 누적

  Slot 생성 후 Standby 연결 끊김 (Standby 서버 장애)
  → Slot이 inactive 상태로 남음
  → Primary: "Standby가 아직 읽지 않은 WAL" 삭제 안 함
  → pg_wal/ 디렉토리 무한 증가
  → Primary 디스크 꽉 참 → PostgreSQL 중단

  모니터링:
  SELECT slot_name, active,
         pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
  FROM pg_replication_slots;
  → retained_wal이 수 GB 이상: 위험 신호

  해결: inactive Slot 즉시 삭제
  SELECT pg_drop_replication_slot('standby1_slot');

실수 2: 동기 복제 설정 후 Standby 없어서 Primary 정지

  synchronous_standby_names = 'standby1'
  → Standby1이 다운됨
  → Primary: 모든 COMMIT이 Standby 응답 대기
  → Primary 사실상 정지 (쓰기 불가)

  올바른 설정:
  synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
  → standby1 또는 standby2 중 하나만 응답하면 됨

실수 3: max_wal_senders 부족

  max_wal_senders = 5 (기본값)
  Standby 3개 + pg_basebackup 2개 + 모니터링 1개 = 6개 필요
  → 복제 연결 거부
  → max_wal_senders를 Standby 수 + 여유분으로 설정
```

---

## ✨ 올바른 접근 (After — Streaming Replication 올바른 설계)

```
권장 Primary 설정 (postgresql.conf):

  wal_level = replica          # Streaming Replication 허용 (기본)
  max_wal_senders = 10         # WAL Sender 프로세스 최대 수
  max_replication_slots = 10   # Replication Slot 최대 수
  hot_standby = on             # Standby에서 읽기 허용 (기본)
  wal_keep_size = 1GB          # Slot 없이 WAL 보존 크기

pg_hba.conf 설정:
  host replication replicator 192.168.1.0/24 scram-sha-256

권장 Standby 설정 (postgresql.conf):
  hot_standby = on
  hot_standby_feedback = on  # Standby 쿼리가 Primary VACUUM 방해 시 피드백

복제 모니터링 자동화:
  -- Prometheus 지표로 수집 권장
  SELECT
      application_name,
      pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
      pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
      write_lag, flush_lag, replay_lag
  FROM pg_stat_replication;

  -- 임계값: replay_lag_bytes > 100MB → 경고
```

---

## 🔬 내부 동작 원리

### 1. Streaming Replication 전체 흐름

```
연결 수립 과정:

Standby → Primary:5432 TCP 연결
  → replication 프로토콜로 인증
  → "START_REPLICATION LSN=0/1A2B3C4D TIMELINE=1"

Primary 측:
  postmaster가 WAL Sender 프로세스 fork()
  WAL Sender: pg_wal/에서 요청된 LSN 이후 WAL 읽기 + 전송

WAL 전송 단계:

  Primary:
  ① 트랜잭션 실행 → WAL 레코드를 WAL Buffer에 기록
  ② COMMIT → WAL Buffer → WAL 파일(pg_wal/)에 fsync
  ③ WAL Sender: WAL 파일에서 읽어 TCP로 Standby에 전송

  Standby:
  ④ WAL Receiver: TCP에서 WAL 수신 → 로컬 pg_wal/ 저장
  ⑤ Startup 프로세스: WAL Receiver에서 WAL을 읽어 Heap/Index에 적용(Redo)
  ⑥ pg_stat_replication 업데이트 (sent_lsn, write_lsn, flush_lsn, replay_lsn)

4가지 LSN 포인트:
  sent_lsn:   Primary → Standby TCP에 전송 완료된 LSN
  write_lsn:  Standby OS 버퍼에 기록된 LSN
  flush_lsn:  Standby 디스크(pg_wal/)에 fsync된 LSN
  replay_lsn: Standby가 Heap에 실제 적용 완료한 LSN

복제 지연 = pg_current_wal_lsn() - replay_lsn
(바이트 단위, pg_wal_lsn_diff() 함수로 계산)
```

### 2. 동기 vs 비동기 복제

```
비동기 복제 (기본값):

  synchronous_standby_names = ''  (기본: 비어있음)

  COMMIT 흐름:
  ① WAL을 로컬 디스크에 fsync
  ② OK 반환 (클라이언트)
  ③ 이후 Standby에 전송 (비동기)

  특성:
  - Primary 성능: 최대 (Standby 대기 없음)
  - 데이터 손실: Primary 크래시 시 최근 N ms 트랜잭션 유실 가능
  - 적합: 성능 우선, 약간의 데이터 손실 허용 가능한 경우

동기 복제:

  synchronous_standby_names = 'standby1'

  COMMIT 흐름:
  ① WAL을 로컬 디스크에 fsync
  ② WAL Sender → Standby로 WAL 전송
  ③ Standby의 응답 레벨에 따라 대기:
     synchronous_commit = on (기본): flush_lsn 도달까지 대기
     synchronous_commit = remote_write: write_lsn 도달까지 대기
     synchronous_commit = remote_apply: replay_lsn 도달까지 대기
  ④ 응답 수신 후 OK 반환

  특성:
  - Primary 성능: 네트워크 RTT만큼 COMMIT 지연
  - 데이터 손실: 없음 (Standby에 기록됨)
  - 적합: 금융 거래, 데이터 손실 절대 불가

synchronous_standby_names 고급 설정:

  FIRST 1 (standby1, standby2):
  → standby1, standby2 중 먼저 응답하는 1개 대기
  → standby1 다운 시 standby2로 자동 전환

  ANY 2 (s1, s2, s3):
  → s1, s2, s3 중 임의 2개 응답 시 OK
  → 더 높은 데이터 안전성

  퀀럼(Quorum) 복제:
  QUORUM (standby1, standby2, standby3):
  → 과반수 응답 시 OK (3개 중 2개)
```

### 3. Replication Slot 동작

```
Replication Slot 없을 때:

  wal_keep_size = 1GB (이전: wal_keep_segments)
  → 최근 1GB 분량의 WAL만 보존
  → Standby가 느려서 1GB 이상 뒤처지면 WAL 삭제됨
  → Standby 오류: "requested WAL segment already removed"
  → Standby를 처음부터 재동기화 필요 (pg_basebackup)

Replication Slot 있을 때:

  -- Slot 생성
  SELECT pg_create_physical_replication_slot('standby1_slot');

  Standby에서 Slot 사용:
  primary_slot_name = 'standby1_slot'  (recovery.conf 또는 postgresql.conf)

  Primary: restart_lsn(Slot이 필요로 하는 최소 LSN) 이전 WAL 삭제 안 함
  → Standby가 며칠 오프라인이어도 WAL 보존

  위험:
  Standby 영구 제거 후 Slot 삭제 안 하면:
  → Primary WAL 무한 보존 → 디스크 꽉 참 → DB 중단

  모니터링 필수:
  SELECT slot_name, active,
         age(restart_lsn) AS slot_age,
         pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
  FROM pg_replication_slots
  ORDER BY restart_lsn;

  임계값 설정 (예시):
  retained_wal > 10GB → 경고
  retained_wal > 50GB → 즉시 Slot 삭제 또는 Standby 복구
```

### 4. pg_basebackup으로 Standby 구성

```
Standby 초기 동기화 과정:

  1. Primary에서 pg_basebackup 실행:
     pg_basebackup -h primary-host -U replicator \
       -D /var/lib/postgresql/standby-data \
       -P -R --wal-method=stream \
       --slot=standby1_slot

     -R: standby.signal + postgresql.auto.conf 자동 생성
     --wal-method=stream: basebackup 중 WAL도 스트리밍

  2. 생성된 파일 확인:
     postgresql.auto.conf에 추가됨:
     primary_conninfo = 'host=primary-host user=replicator ...'
     primary_slot_name = 'standby1_slot'

  3. standby.signal 파일 존재 → Standby 모드로 시작

  4. PostgreSQL 시작:
     Startup 프로세스: WAL을 pg_wal/에서 읽어 적용
     WAL Receiver: Primary에 연결 → 새 WAL 수신

  Standby 상태 확인:
     SELECT pg_is_in_recovery();  -- true: 복제 중
     SELECT pg_last_wal_replay_lsn(), pg_last_xact_replay_timestamp();
```

---

## 💻 실전 실험

### 실험 1: 복제 지연 측정

```sql
-- Primary에서 실행
SELECT
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag,
    sync_state  -- async/sync/quorum
FROM pg_stat_replication
ORDER BY replay_lag DESC;

-- Standby에서 실행
SELECT
    pg_is_in_recovery() AS is_standby,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn,
    pg_wal_lsn_diff(
        pg_last_wal_receive_lsn(),
        pg_last_wal_replay_lsn()
    ) AS apply_lag_bytes,
    pg_last_xact_replay_timestamp() AS last_applied_txn_time,
    now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

### 실험 2: Replication Slot 관리

```sql
-- Primary: Slot 생성
SELECT pg_create_physical_replication_slot('test_slot');

-- Slot 목록 및 보존 WAL 크기
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal_size
FROM pg_replication_slots;

-- Slot 삭제 (Standby 제거 시)
SELECT pg_drop_replication_slot('test_slot');

-- WAL 파일 수 및 크기 확인
SELECT count(*), pg_size_pretty(sum(size)) AS total_wal_size
FROM pg_ls_waldir();
```

### 실험 3: 동기 복제 설정 (Primary에서)

```sql
-- 현재 동기 설정 확인
SHOW synchronous_standby_names;
SHOW synchronous_commit;

-- 동기 복제 설정 (postgresql.conf 변경 또는)
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (standby1)';
SELECT pg_reload_conf();

-- 동기 복제 확인 (sync_state 컬럼)
SELECT application_name, sync_state, replay_lag
FROM pg_stat_replication;
-- sync_state: 'async' → 비동기, 'sync' → 동기, 'potential' → 대기 중

-- 세션 수준 동기 설정 (중요 트랜잭션만)
SET synchronous_commit = 'remote_apply';
-- 이후 COMMIT → Standby 적용 완료까지 대기
RESET synchronous_commit;
```

---

## 📊 MySQL과 비교

```
MySQL Replication vs PostgreSQL Streaming Replication:

항목                | MySQL                    | PostgreSQL
───────────────────┼──────────────────────────┼───────────────────────────
복제 로그           | Binary Log (논리/물리)    | WAL (물리)
복제 방식           | Pull (Slave가 요청)       | Push (Primary가 전송)
이기종 버전 복제     | 지원 (일부 제한)           | 물리 복제 미지원 (논리 복제 가능)
Replication Slot   | 없음                     | 있음 (WAL 보존 관리)
동기 복제           | Semi-Synchronous         | 완전 동기 (여러 모드)
읽기 분산           | Read Replica             | Hot Standby
GTID                | 지원                     | 없음 (타임라인으로 관리)
지연 복제           | MASTER_DELAY             | recovery_min_apply_delay

MySQL Semi-Synchronous vs PostgreSQL 동기:
  MySQL: Primary가 Binary Log를 Slave 디스크에 write 확인 후 OK
  PostgreSQL: flush_lsn/replay_lsn 등 세밀한 단계 선택 가능

실시간 복제 지연 감지:
  MySQL: SHOW SLAVE STATUS\G → Seconds_Behind_Master
  PostgreSQL: pg_stat_replication → replay_lag
```

---

## ⚖️ 트레이드오프

```
동기 vs 비동기 복제:

비동기:
  장점: Primary 성능 최대, 네트워크 지연 영향 없음
  단점: Primary 크래시 시 최근 N ms 데이터 손실 가능
  적합: 대부분의 웹 서비스 (약간의 손실 허용)

동기 (synchronous_commit=on):
  장점: 데이터 손실 없음 (flush 보장)
  단점: RTT만큼 COMMIT 지연 (수 ms ~ 수십 ms)
  적합: 금융, 결제, 중요 데이터

동기 (synchronous_commit=remote_apply):
  장점: Standby에서 즉시 읽기 가능 (Failover 직후 완전 일관성)
  단점: COMMIT 지연 가장 큼 (Standby 적용 시간 포함)
  적합: 읽기 일관성 필수 환경

Replication Slot 사용 결정:
  사용: Standby 재동기화 없이 WAL 보존 필요
  미사용: Slot 관리 부담 회피 + wal_keep_size로 대체
  Slot 모니터링 필수 (미관리 시 디스크 위험)
```

---

## 📌 핵심 정리

```
Streaming Replication 핵심:

흐름:
  Primary WAL 생성 → WAL Sender → TCP → WAL Receiver → pg_wal → Startup Redo

4 LSN:
  sent → write → flush → replay (뒤로 갈수록 Standby에 더 깊이 적용)
  복제 지연 = current_wal_lsn - replay_lsn

동기 복제:
  synchronous_standby_names 설정
  synchronous_commit으로 확인 레벨 선택 (on/remote_write/remote_apply)

Replication Slot:
  WAL 보존 보장, Standby 재동기화 방지
  inactive Slot = 디스크 폭탄 → 모니터링 필수

필수 설정:
  wal_level = replica
  max_wal_senders = Standby 수 + 여유
  max_replication_slots = Slot 수 + 여유
  hot_standby = on (Standby 읽기 허용)
```

---

## 🤔 생각해볼 문제

**Q1.** `synchronous_commit = remote_write`와 `remote_apply`의 정확한 차이는? 어느 쪽이 데이터 안전성이 더 높은가?

<details>
<summary>해설 보기</summary>

**`remote_write`**: Standby의 OS 버퍼에 WAL이 기록된 것을 확인합니다. Standby OS 크래시(정전 등) 시 OS 버퍼가 날아가면 데이터 손실 가능. 하지만 Standby 프로세스 크래시는 OS 버퍼가 안전하므로 보호됩니다.

**`remote_apply`**: Standby에서 WAL이 실제로 Heap에 적용(Redo)된 것을 확인합니다. Standby Promote 후 즉시 최신 데이터로 읽기 가능. 가장 강한 보장.

안전성 순서: `remote_apply > flush(on) > remote_write`

`remote_apply`는 Standby의 적용 속도에 COMMIT 지연이 의존하므로, Standby가 느리거나 부하가 높으면 Primary COMMIT이 크게 느려질 수 있습니다.

</details>

---

**Q2.** Replication Slot을 삭제하지 않고 Standby를 폐기하면 Primary에 어떤 일이 발생하는가? 어떻게 감지하는가?

<details>
<summary>해설 보기</summary>

Slot이 `inactive` 상태로 남으면 `restart_lsn`이 고정된 채로 Primary는 그 이후 WAL을 모두 보존합니다. Standby가 없어서 `restart_lsn`이 전진하지 않으므로 `pg_wal/` 디렉토리가 무한히 증가합니다.

디스크가 가득 차면 PostgreSQL이 WAL을 쓸 수 없어 **강제 종료**됩니다.

감지 방법:
```sql
-- inactive Slot과 보존 WAL 크기 확인
SELECT slot_name, active,
       pg_size_pretty(
           pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       ) AS retained_wal
FROM pg_replication_slots
WHERE NOT active;
-- retained_wal이 수 GB 이상이면 즉시 조치 필요
```

해결:
```sql
SELECT pg_drop_replication_slot('old_standby_slot');
```

Prometheus 등에서 `pg_replication_slot_wal_status` 지표를 모니터링하거나, `retained_wal > 10GB` 조건으로 Alert를 설정하는 것이 필수입니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: 집계 함수 심화](../advanced-sql/06-aggregation-functions.md)** | **[홈으로 🏠](../README.md)** | **[다음: Hot Standby ➡️](./02-hot-standby.md)**

</div>
