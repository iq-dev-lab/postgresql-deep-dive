# Hot Standby — 복제 중인 Standby에서 읽기 쿼리 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Hot Standby에서 읽기 쿼리가 WAL 적용과 충돌하는 메커니즘은?
- `hot_standby_feedback = on`이 Primary의 VACUUM에 어떤 영향을 주는가?
- Standby에서 쿼리가 취소(Cancel)되는 정확한 조건은?
- `max_standby_streaming_delay`와 `max_standby_archive_delay`의 역할은?
- 읽기 트래픽 분산 시 Standby의 복제 지연을 허용 가능한 수준으로 유지하는 방법은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

Hot Standby는 복제 중인 Standby에서 SELECT 쿼리를 처리할 수 있는 기능이다. 읽기 부하를 Primary와 Standby로 분산할 수 있어 확장성이 크게 향상된다. 하지만 Standby가 WAL을 적용하는 도중에도 읽기가 실행되므로 충돌이 발생할 수 있다. Primary의 VACUUM이 Standby가 읽고 있는 Dead Tuple을 제거하려 하면 쿼리가 취소될 수 있다. `hot_standby_feedback`이 이를 해결하지만 Primary VACUUM을 늦추는 부작용이 있다.

---

## 😱 흔한 실수 (Before — Hot Standby 운영 오해)

```
실수 1: Standby 쿼리 갑작스러운 취소 원인 모름

  애플리케이션 로그:
  ERROR: canceling statement due to conflict with recovery
  DETAIL: User query might have needed to see row versions that must be removed.

  원인:
  Primary: Dead Tuple을 VACUUM으로 제거
  Standby: 그 Dead Tuple을 포함한 페이지를 쿼리가 읽는 중
  Standby WAL 적용: "이 페이지의 Dead Tuple을 제거해야 함"
  충돌 → max_standby_streaming_delay(기본 30초) 후 쿼리 취소

  해결 방향:
  hot_standby_feedback = on → Primary VACUUM이 Standby 스냅샷 고려
  또는 max_standby_streaming_delay 증가 (대기 시간 늘림)

실수 2: hot_standby_feedback 없이 장기 Standby 쿼리

  Standby에서 10분짜리 보고서 쿼리 실행
  → Primary에서 해당 데이터 VACUUM 시도
  → 9분 후 쿼리 취소!
  → hot_standby_feedback = on 설정으로 Primary에 Standby 스냅샷 정보 전달

실수 3: Standby 복제 지연 과도

  읽기 쿼리를 Standby로 보냈는데:
  "최신 데이터가 없다!" 사용자 불만
  → Standby 복제 지연 = 5분

  해결:
  pg_stat_replication.replay_lag 모니터링
  지연 기준 초과 시 Standby 비활성화 (애플리케이션 레벨)
  또는 중요 읽기는 Primary로, 통계/보고서만 Standby로
```

---

## ✨ 올바른 접근 (After — Hot Standby 올바른 운영)

```
Hot Standby 설정 전략:

  Primary (postgresql.conf):
  wal_level = replica  # Hot Standby 활성화
  hot_standby = on     # Standby에서 읽기 허용 (Primary에도 설정)

  Standby (postgresql.conf):
  hot_standby = on
  hot_standby_feedback = on        # Primary VACUUM에 Standby 스냅샷 피드백
  max_standby_streaming_delay = 60s # 충돌 시 최대 대기 시간 (기본 30s)

  읽기 트래픽 분산 전략:
  ① PgBouncer: Primary 포트 5432, Standby 포트 5433
  ② 애플리케이션: 읽기 중요도에 따라 DB URL 선택
     - 최신 데이터 필수: Primary
     - 통계/보고서: Standby (약간의 지연 허용)
  ③ Spring: @Transactional(readOnly=true) + Standby DataSource 라우팅

  복제 지연 기반 Standby 비활성화:
  SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) > 10*1024*1024
  AS standby_too_behind  -- 10MB 이상 지연 시 Standby 비활성화
```

---

## 🔬 내부 동작 원리

### 1. Hot Standby 읽기-쓰기 공존 방법

```
Standby의 두 가지 병렬 작업:

  ┌─────────────────────────────────────────────────────────┐
  │  Standby 프로세스                                        │
  │                                                         │
  │  Startup 프로세스:        읽기 쿼리 Backend:             │
  │  WAL 레코드 수신            SELECT * FROM orders         │
  │  → Heap/Index 페이지 수정   WHERE user_id = 42           │
  │  (WAL Redo)                                             │
  │         ↕ 충돌 가능                                      │
  │  Primary에서 VACUUM         이 Dead Tuple 읽는 중        │
  │  → Dead Tuple 제거 WAL                                  │
  └─────────────────────────────────────────────────────────┘

Hot Standby MVCC 방식:
  Standby도 스냅샷 기반으로 어떤 버전을 보는지 결정
  WAL Redo가 Standby 읽기 쿼리의 스냅샷 이전 데이터를 제거하려 하면 → 충돌

Standby의 스냅샷:
  Standby 읽기 트랜잭션의 snapshot.xmin 이후 Dead Tuple을 WAL Redo가 제거하려 하면
  → 아직 읽고 있는 버전 제거 시도 → 충돌!
```

### 2. 충돌(Conflict) 유형과 해결

```
Hot Standby 충돌 유형:

① VACUUM 충돌 (가장 흔함):
   Primary: VACUUM이 Dead Tuple 제거 → WAL 생성
   Standby WAL Redo: 해당 튜플 페이지 변경 적용 시도
   Standby 읽기 쿼리: 그 튜플을 아직 읽는 중
   → 충돌!

② 접근 독점 잠금 충돌:
   Primary: DROP TABLE, LOCK TABLE 등 AccessExclusiveLock
   Standby WAL Redo: 동일한 잠금 획득 시도
   Standby 읽기 쿼리: 해당 테이블에 AccessShareLock 보유
   → 충돌!

③ 테이블스페이스 삭제 충돌:
   Primary: DROP TABLESPACE
   Standby: 그 테이블스페이스에서 읽기 중
   → 충돌!

충돌 해결 방식:

  max_standby_streaming_delay = 30s (기본):
  → WAL Redo가 충돌 발생 시 최대 30초 대기
  → 30초 내 읽기 쿼리 완료 → Redo 진행
  → 30초 경과 → 읽기 쿼리 강제 취소

  쿼리 취소 오류:
  ERROR: canceling statement due to conflict with recovery
  DETAIL: User query might have needed to see row versions that must be removed.

  max_standby_streaming_delay 조정:
  짧은 쿼리만 Standby로 → 30s 유지
  장기 보고서 → 300s 이상으로 증가
  -1: 무한 대기 (Redo 차단 → 복제 지연 누적 위험)
```

### 3. hot_standby_feedback 동작

```
hot_standby_feedback = off (기본):

  Standby 읽기 쿼리의 xmin을 Primary에 알리지 않음
  → Primary VACUUM은 자신의 OldestXmin 기준으로 Dead Tuple 제거
  → Standby에서 여전히 필요한 Dead Tuple 제거됨 → 충돌

hot_standby_feedback = on:

  Standby: 현재 활성 트랜잭션의 xmin을 Primary에 주기적으로 전송
  Primary: VACUUM 시 Standby의 xmin도 고려 (OldestXmin = min(local, standby))
  → Standby가 읽는 Dead Tuple을 Primary VACUUM이 보존
  → VACUUM 충돌 대폭 감소

  부작용:
  Primary OldestXmin이 낮아짐
  → Dead Tuple이 더 오래 유지됨
  → Primary Table Bloat 증가 가능
  → Standby 읽기 쿼리가 길면 Primary VACUUM 효율 저하

  균형:
  장기 쿼리를 Standby에 실행해야 한다면 → hot_standby_feedback = on 필수
  Standby 쿼리가 짧다면 → off + max_standby_streaming_delay 조정으로 충분
```

### 4. Standby에서 허용/불허용 작업

```
Hot Standby에서 가능한 작업:
  SELECT (읽기 전용)
  EXPLAIN (계획만)
  BEGIN READ ONLY (읽기 전용 트랜잭션)
  COPY TO (내보내기)
  pg_stat_* 뷰 조회
  SHOW 설정 조회

Hot Standby에서 불가능한 작업:
  INSERT, UPDATE, DELETE
  CREATE, ALTER, DROP
  Sequences (nextval 등)
  LOCK TABLE
  LISTEN/NOTIFY
  CURSOR with FOR UPDATE

쓰기 시도 시:
  ERROR: cannot execute INSERT in a read-only transaction

Standby 적용 지연 설정:
  recovery_min_apply_delay = '5min'
  → WAL을 수신해도 5분 후에 적용
  → 의도적 지연 복제 (Primary 실수 감지 후 롤백 창)
  → 적용 전이므로 5분 내 실수는 Standby에서 복구 가능

  (단, 이 기간 동안 Standby 데이터가 5분 뒤처짐)
```

---

## 💻 실전 실험

### 실험 1: Hot Standby 충돌 재현

```sql
-- Standby에서 실행
-- 터미널 1: 장기 트랜잭션 (스냅샷 고정)
BEGIN;
SELECT count(*) FROM large_table;  -- 스냅샷 획득

-- Primary에서:
-- VACUUM large_table;  (Dead Tuple 제거 → WAL 생성)

-- Standby 터미널 1: max_standby_streaming_delay 후 취소될 수 있음
-- ERROR: canceling statement due to conflict with recovery

-- 충돌 로그 확인 (Standby)
SELECT now() - query_start AS duration, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

### 실험 2: 복제 지연 모니터링

```sql
-- Primary에서 복제 상태 확인
SELECT
    application_name,
    state,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)) AS send_lag,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replay_lag,
    write_lag,
    flush_lag,
    replay_lag AS time_lag
FROM pg_stat_replication;

-- Standby에서 복제 지연 확인
SELECT
    pg_last_xact_replay_timestamp() AS last_replayed_txn,
    now() - pg_last_xact_replay_timestamp() AS replication_lag,
    CASE
        WHEN now() - pg_last_xact_replay_timestamp() > INTERVAL '5 minutes'
        THEN '⚠️ 지연 과도'
        WHEN now() - pg_last_xact_replay_timestamp() > INTERVAL '1 minute'
        THEN '🟡 주의'
        ELSE '🟢 정상'
    END AS status
FROM pg_stat_recovery_prefetch;  -- PG 14+
-- 또는 단순하게:
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

### 실험 3: hot_standby_feedback 효과 확인

```sql
-- Standby에서 설정 확인
SHOW hot_standby_feedback;

-- Primary에서 Standby feedback 확인
SELECT
    application_name,
    backend_xmin  -- Standby가 보고한 xmin
FROM pg_stat_replication;

-- hot_standby_feedback = on 시 Primary의 OldestXmin
SELECT min(backend_xmin) AS oldest_xmin
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL;
-- Standby의 xmin도 포함되어 낮아짐
```

---

## 📊 MySQL과 비교

```
MySQL Read Replica vs PostgreSQL Hot Standby:

MySQL Read Replica:
  Binary Log 기반 논리 복제
  Replica에서 SELECT 가능
  복제 지연: Seconds_Behind_Master
  충돌 없음 (Undo Log 방식으로 이전 버전 유지)
  쓰기: 설정으로 허용 가능 (read_only = OFF)

PostgreSQL Hot Standby:
  WAL 기반 물리 복제
  복제 중인 상태에서 SELECT 가능
  VACUUM 충돌: hot_standby_feedback으로 관리
  완전 읽기 전용 (Promote 전에는 쓰기 불가)

충돌 없는 이유 (MySQL):
  MySQL은 Undo Log 방식으로 이전 버전을 별도 보관
  → Replica에서 읽기 쿼리와 WAL 적용이 충돌할 필요 없음
  → VACUUM 개념 없음 (Purge Thread 자동)

PostgreSQL 충돌이 있는 이유:
  Dead Tuple이 Heap 페이지에 있고
  WAL Redo가 그 Dead Tuple을 제거하려 할 때
  읽기 쿼리가 그 페이지에 접근 중이면 충돌
  → MVCC의 차이에서 비롯됨
```

---

## ⚖️ 트레이드오프

```
Hot Standby 운영 트레이드오프:

hot_standby_feedback = on:
  장점: VACUUM 충돌 감소, 장기 Standby 쿼리 안전
  단점: Primary Table Bloat 가능, VACUUM 효율 저하

hot_standby_feedback = off:
  장점: Primary VACUUM 자유롭게 실행
  단점: 장기 Standby 쿼리 → 충돌 취소 가능

max_standby_streaming_delay 높이기:
  장점: 쿼리 취소 줄어듦
  단점: Redo 차단 시간 증가 → 복제 지연 누적 가능

읽기 분산 최적화:
  짧은 쿼리 (< 100ms): Standby OK, feedback off
  장기 보고서 (> 1분): feedback on + delay 증가
  최신 데이터 필수: Primary만 사용
  통계/분석: Standby (지연 허용)
```

---

## 📌 핵심 정리

```
Hot Standby 핵심:

개념:
  Streaming Replication 중인 Standby에서 SELECT 처리
  WAL Redo + 읽기 쿼리 병렬 실행

충돌 유형:
  VACUUM 충돌 (Dead Tuple 제거 vs 읽기)
  AccessExclusiveLock 충돌 (DROP/LOCK)

충돌 해결:
  max_standby_streaming_delay: 대기 시간 (기본 30s)
  hot_standby_feedback: Standby xmin을 Primary에 전달 → VACUUM 충돌 감소

설정:
  primary_standby.conf: hot_standby = on
  Standby: hot_standby_feedback = on (장기 쿼리 시)

읽기 분산:
  최신 필수 → Primary
  통계/보고서 → Standby
  복제 지연 모니터링 → pg_stat_replication.replay_lag
```

---

## 🤔 생각해볼 문제

**Q1.** Standby에서 `SELECT pg_cancel_backend(pid)`를 실행해 Primary의 쿼리를 취소할 수 있는가?

<details>
<summary>해설 보기</summary>

아니요. `pg_cancel_backend()`는 로컬 PostgreSQL 인스턴스의 백엔드만 취소할 수 있습니다. Standby에서 실행하면 Standby의 백엔드 프로세스만 영향을 받습니다.

Primary의 쿼리를 취소하려면 Primary 인스턴스에서 직접 실행해야 합니다. 반대도 마찬가지입니다.

Standby의 읽기 쿼리 취소:
```sql
-- Standby에서 직접 실행
SELECT pg_cancel_backend(pid) FROM pg_stat_activity WHERE query LIKE '%slow query%';
```

충돌로 인한 자동 취소는 PostgreSQL이 내부적으로 처리합니다.

</details>

---

**Q2.** `recovery_min_apply_delay = '1 hour'` 설정이 있는 Standby에서 Failover가 발생하면 어떤 일이 생기는가?

<details>
<summary>해설 보기</summary>

`recovery_min_apply_delay`는 WAL 수신 후 1시간 지연 후 적용합니다. Primary 장애 시 Standby를 Promote하면:

1. **적용되지 않은 1시간 치 WAL이 있습니다.** Promote 시 이 WAL을 즉시 적용할지 건너뛸지 선택해야 합니다.
2. **Promote 프로세스**: 기본적으로 Promote 시 버퍼에 있는 미적용 WAL을 모두 적용하고 새 Primary가 됩니다. 따라서 1시간 치 데이터가 모두 적용됩니다.
3. **Failover 지연**: 1시간 치 WAL 적용이 완료될 때까지 Promote에 시간이 걸릴 수 있습니다.

`recovery_min_apply_delay`는 Failover 시간도 늦추는 단점이 있습니다. 실수 감지 창(예: DBA가 DROP TABLE을 실수했을 때 1시간 내 Standby에서 복구)을 위한 기능이므로, **고가용성이 최우선이라면 사용하지 않는 것**이 권장됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Streaming Replication](./01-streaming-replication.md)** | **[홈으로 🏠](../README.md)** | **[다음: 논리 복제 ➡️](./03-logical-replication.md)**

</div>
