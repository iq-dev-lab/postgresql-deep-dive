# postgresql.conf 핵심 설정 — 메모리부터 연결까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `shared_buffers`는 왜 RAM의 25%가 권장이고, 그 이상 설정하면 어떻게 되는가?
- `effective_cache_size`는 실제 메모리를 할당하지 않는데 왜 설정해야 하는가?
- `work_mem`을 크게 설정하면 어떤 위험이 있는가?
- `max_connections`와 PgBouncer의 관계에서 최적값을 어떻게 계산하는가?
- `checkpoint_completion_target`과 `max_wal_size`를 왜 조정해야 하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL의 기본 설정은 매우 보수적이다. `shared_buffers = 128MB`는 현대 서버의 RAM에 비해 터무니없이 작다. 설치 후 적절한 설정 조정 없이 운영하면 성능이 10배 이하로 제한된다. 반대로 `work_mem`을 너무 크게 설정하면 동시 쿼리가 많을 때 OOM이 발생한다. 각 설정의 역할과 상호작용을 이해해야 서버 특성에 맞는 최적 설정을 도출할 수 있다.

---

## 😱 흔한 실수 (Before — 기본값 방치)

```
실수 1: 64GB RAM 서버에 shared_buffers = 128MB (기본값 방치)

  문제:
  데이터 페이지를 128MB만 캐시 → 대부분 OS Page Cache 의존
  이중 캐시 구조에서 shared_buffers 낭비
  쿼리마다 디스크 I/O 높음 → 성능 저하

  64GB RAM 서버 권장:
  shared_buffers = 16GB  (RAM의 25%)

실수 2: work_mem = 1GB + max_connections = 500

  work_mem은 연결당 정렬/해시 조인에 할당
  500 연결 × 1GB = 최대 500GB 메모리 요구
  → 실제 서버 RAM이 64GB이면 OOM 발생

  올바른 계산:
  work_mem = (available_memory × 0.5) / (active_connections × avg_sorts_per_query)
  일반적으로 4~64MB가 적절

실수 3: checkpoint_completion_target = 0.5 (기본값)

  Checkpoint I/O가 checkpoint 간격의 50% 시간에 집중됨
  → 짧은 시간에 많은 Dirty Page를 디스크에 씀
  → I/O 스파이크 → 쿼리 지연

  권장: checkpoint_completion_target = 0.9
  → 90% 시간 동안 분산하여 쓰기 → I/O 평탄화
```

---

## ✨ 올바른 접근 (After — 서버 특성 기반 설정)

```
서버 사양별 권장 설정:

서버: 64GB RAM, 8 core, SSD
  shared_buffers = 16GB              # RAM 25%
  effective_cache_size = 48GB        # RAM 75% (플래너 힌트)
  work_mem = 64MB                    # 동시 쿼리 고려
  maintenance_work_mem = 2GB         # VACUUM, 인덱스 생성
  wal_buffers = 64MB                 # 기본 -1(shared_buffers 1/32)보다 명시
  max_connections = 200              # PgBouncer 사용 전제
  checkpoint_completion_target = 0.9 # I/O 분산
  max_wal_size = 4GB                 # Checkpoint 간격 조절

  # Autovacuum
  autovacuum_vacuum_cost_limit = 800  # 기본 200보다 공격적
  autovacuum_max_workers = 5          # 기본 3보다 많이

  # 로깅
  log_min_duration_statement = 1000   # 1초 이상 쿼리 로깅
  log_checkpoints = on
  log_connections = off               # 고빈도 연결 환경에서는 off

pgtune (자동 설정 도구):
  https://pgtune.leopard.in.ua/
  → DB 유형, RAM, CPU, 디스크 입력하면 권장값 자동 계산
```

---

## 🔬 내부 동작 원리

### 1. 메모리 설정 그룹

```
shared_buffers:

  PostgreSQL의 공유 버퍼 풀 크기
  모든 백엔드 프로세스가 공유하는 페이지 캐시

  작동 방식:
  데이터 페이지 읽기 → shared_buffers 확인
  캐시 히트: 메모리에서 직접 반환 (빠름)
  캐시 미스: 디스크 → OS Page Cache → shared_buffers → 프로세스

  권장값: RAM의 25%
  이유:
  ① OS Page Cache도 PostgreSQL 파일을 캐시 (이중 캐시)
  ② 50% 이상 설정 시 OS 메모리 부족 → 페이지 스왑 발생 위험
  ③ 실험적으로 25%가 최적 균형점

  변경 방법:
  postgresql.conf 수정 후 pg_ctl restart (재시작 필요)

effective_cache_size:

  실제 메모리 할당 없음! 플래너 힌트만
  OS Page Cache + shared_buffers의 추정 합산값

  플래너 활용:
  Index Scan vs Seq Scan 비용 비교 시
  "이 크기의 캐시가 있다면 Index 페이지가 캐시에 있을 확률" 계산

  권장값: RAM의 75% (shared_buffers 포함)
  effective_cache_size = 48GB (64GB RAM에서)
  → 플래너가 더 적극적으로 Index Scan 선택

work_mem:

  정렬(ORDER BY), 해시 조인, 집계 시 할당되는 메모리
  각 정렬 노드당 work_mem 할당 (1 쿼리에 여러 노드 가능)

  위험:
  100 동시 연결 × 쿼리당 평균 3 정렬 노드 × work_mem
  = 300 × work_mem 최대 메모리 필요

  설정 방법:
  전역: work_mem = 32MB (보수적)
  세션 수준 (복잡 쿼리 실행 전):
    SET work_mem = '1GB';  -- 이 쿼리에만 큰 메모리
    RESET work_mem;

maintenance_work_mem:

  VACUUM, ANALYZE, CREATE INDEX, ALTER TABLE 등 유지보수 작업 메모리
  각 Autovacuum Worker당 maintenance_work_mem 할당

  권장값: RAM의 5~10% (최대 2GB)
  크게 설정할수록: VACUUM 빠름, 인덱스 생성 빠름
  단, autovacuum_max_workers × maintenance_work_mem < available_memory
```

### 2. WAL 및 체크포인트 설정

```
wal_buffers:

  WAL 레코드를 디스크에 쓰기 전 임시 버퍼
  기본값: -1 (shared_buffers의 1/32, 최대 16MB)
  WAL 집중 쓰기 환경에서: 64MB로 명시 설정

max_wal_size:

  Checkpoint 사이 WAL 최대 크기
  이 크기 초과 시 → 강제 Checkpoint 발생

  기본값: 1GB
  권장: 2~4GB (더 긴 Checkpoint 간격)
  크게 설정하면:
  ① Checkpoint 덜 빈번 → I/O 부하 감소
  ② 크래시 복구 시간 증가 (더 많은 WAL 재실행)

  SSD 환경에서 복구 빠르므로 4~8GB까지 늘릴 수 있음

checkpoint_completion_target:

  Checkpoint를 checkpoint 간격의 몇 % 시간에 걸쳐 처리할지
  기본값: 0.5 (50%)

  0.5: Checkpoint 간격의 절반 시간에 집중 → I/O 스파이크
  0.9: 90% 시간 동안 분산 → I/O 평탄화

  권장: 0.9

checkpoint_timeout:

  자동 Checkpoint 간격 (기본 5분)
  크게 설정하면 Checkpoint 덜 발생하나 복구 시간 증가
  권장: 5~15분

  Checkpoint 발생 조건:
  ① max_wal_size 초과
  ② checkpoint_timeout 경과
  ③ 수동 CHECKPOINT 명령어
```

### 3. 연결 및 인증 설정

```
max_connections:

  최대 동시 접속 수
  각 연결 = 프로세스 fork → 메모리 소비

  PgBouncer 없을 때:
  max_connections = CPU 코어 수 × 4 (경험치)
  8 코어 → 32~64 (읽기 집중)
  초과 연결 → 컨텍스트 전환 부하 증가

  PgBouncer 있을 때:
  max_connections = 100~300 (PgBouncer가 수천 클라이언트를 수십 연결로 줄임)

  메모리 계산:
  superuser_reserved_connections = 3 (슈퍼유저 전용)
  일반 사용자 = max_connections - 3
  각 연결 최소 5MB → max_connections=200 → 1GB 프로세스 메모리

listen_addresses:

  PostgreSQL이 수신할 IP 주소
  기본값: localhost (외부 연결 불가)
  '*': 모든 인터페이스 (방화벽으로 보호 필요)
  '0.0.0.0': IPv4 모든 주소

log_min_duration_statement:

  이 시간 이상 걸린 쿼리를 자동 로깅 (ms 단위)
  기본: -1 (비활성)
  권장: 1000 (1초 이상 쿼리 로깅)

  로그 형식:
  duration: 1234.567 ms  statement: SELECT ...

  pg_stat_statements와 함께 사용:
  느린 쿼리 로그 → 원인 분석 → 인덱스 추가 또는 쿼리 최적화
```

### 4. 워크로드별 권장 설정

```
OLTP (웹 서비스, 짧은 트랜잭션):

  shared_buffers = RAM × 0.25
  effective_cache_size = RAM × 0.75
  work_mem = 4~32MB (동시 연결 많음)
  max_connections = 200~500 (PgBouncer 사용)
  checkpoint_completion_target = 0.9
  max_wal_size = 2GB

OLAP (분석, 복잡한 집계):

  shared_buffers = RAM × 0.25
  effective_cache_size = RAM × 0.75
  work_mem = 256MB~1GB (동시 연결 적음)
  max_connections = 20~50 (분석 쿼리)
  maintenance_work_mem = 4GB (대형 인덱스 생성)
  max_parallel_workers_per_gather = 4 (병렬 쿼리)

Mixed (OLTP + OLAP):

  work_mem = 64MB (기본)
  분석 쿼리 실행 전 세션 수준:
  SET work_mem = '1GB';
  → 해당 쿼리만 큰 메모리 사용
  RESET work_mem;
```

---

## 💻 실전 실험

### 실험 1: 현재 설정 확인 및 효과 측정

```sql
-- 주요 설정 한 번에 확인
SELECT name, setting, unit, context, short_desc
FROM pg_settings
WHERE name IN (
    'shared_buffers', 'effective_cache_size', 'work_mem',
    'maintenance_work_mem', 'max_connections', 'wal_buffers',
    'checkpoint_completion_target', 'max_wal_size',
    'autovacuum_vacuum_cost_limit', 'log_min_duration_statement'
)
ORDER BY name;

-- 설정 적용 방법 확인 (context)
-- 'postmaster': 재시작 필요
-- 'sighup': pg_reload_conf() 또는 pg_ctl reload 가능
-- 'superuser': SET으로 현재 세션에서 변경 가능
-- 'user': 일반 사용자도 SET 가능

-- shared_buffers 히트율 확인
SELECT
    sum(heap_blks_hit) AS heap_hit,
    sum(heap_blks_read) AS heap_read,
    round(sum(heap_blks_hit) * 100.0 /
          nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2)
    AS hit_ratio_pct
FROM pg_statio_user_tables;
-- hit_ratio_pct < 90%: shared_buffers 증가 검토
```

### 실험 2: work_mem이 정렬 성능에 미치는 영향

```sql
-- 큰 데이터셋 정렬 시 work_mem 효과
CREATE TABLE sort_test AS
SELECT random() AS val, md5(random()::text) AS txt
FROM generate_series(1, 5000000);

-- 작은 work_mem (디스크 정렬 발생 가능)
SET work_mem = '4MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM sort_test ORDER BY val LIMIT 100;
-- Sort Method: external merge (디스크 정렬)

-- 큰 work_mem (메모리 정렬)
SET work_mem = '512MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM sort_test ORDER BY val LIMIT 100;
-- Sort Method: quicksort (메모리 정렬 → 빠름)

RESET work_mem;
```

### 실험 3: Checkpoint 동작 관찰

```sql
-- 현재 Checkpoint 정보
SELECT
    checkpoint_lsn,
    checkpoint_time,
    now() - checkpoint_time AS time_since_checkpoint,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), checkpoint_lsn)) AS wal_since_checkpoint
FROM pg_control_checkpoint();

-- Checkpoint 통계 (버퍼 기록 현황)
SELECT
    checkpoints_timed,
    checkpoints_req,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend,
    round(buffers_checkpoint::numeric / nullif(checkpoints_timed + checkpoints_req, 0), 1) AS avg_buffers_per_checkpoint
FROM pg_stat_bgwriter;

-- checkpoints_req > checkpoints_timed: max_wal_size 증가 검토 (강제 Checkpoint 多)
```

---

## 📊 MySQL과 비교

```
MySQL vs PostgreSQL 메모리 설정:

MySQL:
  innodb_buffer_pool_size: InnoDB 버퍼 풀 (shared_buffers 대응)
    권장: 서버 RAM의 70~80% (OS Cache와 이중화 없음, InnoDB가 전담)
  innodb_log_buffer_size: Redo Log 버퍼 (wal_buffers 대응)
  sort_buffer_size: 정렬 버퍼 (work_mem과 유사, 세션당)
  join_buffer_size: 조인 버퍼

  MySQL InnoDB: O_DIRECT로 OS Page Cache 우회 가능
  → OS Cache와 이중화 없음 → buffer_pool_size를 크게 설정 가능

PostgreSQL:
  shared_buffers: 25% 권장 (OS Page Cache와 이중화)
  O_DIRECT 미지원 → 이중 캐시 감수

차이:
  MySQL: buffer_pool_size = RAM 70~80%
  PostgreSQL: shared_buffers = RAM 25% (OS Cache 25% + 나머지 OS/기타)

effective_cache_size 대응:
  MySQL: optimizer_switch, innodb_stats_persistent 등으로 통계 제어
  PostgreSQL: effective_cache_size로 플래너에 캐시 크기 힌트
```

---

## ⚖️ 트레이드오프

```
설정 조정의 트레이드오프:

shared_buffers 크게:
  장점: 더 많은 페이지 캐시 → 캐시 히트율 증가
  단점: OS 메모리 부족 → 스왑 발생 위험, 25% 초과 시 역효과

work_mem 크게:
  장점: 외부 정렬 없음 → 복잡 쿼리 빠름
  단점: OOM 위험 (연결 수 × 정렬 노드 수 × work_mem)

max_connections 크게:
  장점: 더 많은 동시 연결 허용
  단점: 프로세스 오버헤드 증가, 컨텍스트 전환 부하

max_wal_size 크게:
  장점: Checkpoint 덜 빈번 → I/O 평탄화
  단점: 크래시 복구 시간 증가

checkpoint_completion_target = 0.9:
  장점: I/O 스파이크 없음 → 쿼리 지연 감소
  단점: 더 빠른 Dirty Page 기록 불가 (그러나 거의 항상 0.9 권장)
```

---

## 📌 핵심 정리

```
postgresql.conf 핵심 설정:

메모리:
  shared_buffers = RAM × 0.25  (페이지 캐시)
  effective_cache_size = RAM × 0.75  (플래너 힌트, 실제 할당 없음)
  work_mem = 4~64MB  (정렬/해시, 연결 수 고려)
  maintenance_work_mem = RAM × 0.05~0.1  (VACUUM/인덱스)

WAL/체크포인트:
  wal_buffers = 64MB
  max_wal_size = 2~4GB
  checkpoint_completion_target = 0.9

연결:
  max_connections = 100~300  (PgBouncer 사용)
  superuser_reserved_connections = 3 (기본)

로깅:
  log_min_duration_statement = 1000  (1초 이상 로깅)
  log_checkpoints = on

튜닝 도구:
  pgtune: https://pgtune.leopard.in.ua/ (자동 설정 계산)
  pg_config.sh: pgBadger 통계 기반 설정
```

---

## 🤔 생각해볼 문제

**Q1.** `shared_buffers = 16GB`로 설정했는데, `pg_statio_user_tables`에서 캐시 히트율이 여전히 70%라면 무엇을 확인해야 하는가?

<details>
<summary>해설 보기</summary>

캐시 히트율이 낮은 원인:

1. **working set이 shared_buffers보다 큰 경우**: 실제 쿼리가 접근하는 데이터가 16GB를 초과하면 히트율이 낮음. 데이터 총 크기 확인 필요.

2. **OS Page Cache 히트 포함 안 됨**: `pg_statio`는 shared_buffers 히트만 측정. OS Page Cache에서 서빙되는 것은 shared_buffers "미스"로 집계. 실제 디스크 I/O는 `iostat`으로 확인.

3. **Cold Start**: PostgreSQL 재시작 후 shared_buffers가 워밍업되지 않음. 시간이 지남에 따라 히트율 증가.

4. **Sequential Scan으로 캐시 오염**: 대형 테이블 SeqScan이 shared_buffers를 자주 교체. `pg_stat_user_tables`에서 seq_scan이 높은 테이블 확인 후 인덱스 추가.

확인 순서:
```sql
-- 가장 많이 읽는 테이블/인덱스 캐시 히트율
SELECT relname, heap_blks_hit, heap_blks_read,
       round(heap_blks_hit * 100.0 / nullif(heap_blks_hit + heap_blks_read, 0), 1) AS hit_pct
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC LIMIT 10;
```

</details>

---

**Q2.** `work_mem`을 세션 수준에서 높게 설정하는 것이 전역 설정보다 안전한 이유는?

<details>
<summary>해설 보기</summary>

전역 `work_mem = 1GB` 설정 시:
- 모든 쿼리, 모든 정렬 노드가 각 1GB 사용 가능
- 100 동시 연결 × 쿼리당 3 정렬 노드 = 300GB 최대 필요 → OOM

세션 수준 설정:
```sql
SET work_mem = '1GB';
SELECT * FROM large_table ORDER BY complex_expr;
RESET work_mem;
```
- 이 세션에서만 1GB 사용
- 다른 세션은 기본값(예: 32MB) 유지
- 해당 분석 쿼리가 빠르게 완료되고 나면 메모리 반환

실용적 패턴:
```sql
-- 분석 쿼리 실행 전
SET work_mem = '512MB';
-- 복잡한 집계/정렬 쿼리 실행
SELECT ...;
-- 반드시 원복
RESET work_mem;
```

Spring에서는 `@Transactional` 내에서 `jdbcTemplate.execute("SET work_mem = '512MB'")`로 설정 후 쿼리를 실행하고 트랜잭션 종료 시 자동으로 원복됩니다 (Transaction Mode PgBouncer에서는 세션 상태가 초기화됨).

</details>

---

<div align="center">

**[⬅️ 이전 챕터: Connection Pooling 심화](../replication-ha/05-connection-pooling.md)** | **[홈으로 🏠](../README.md)** | **[다음: 쿼리 성능 진단 ➡️](./02-query-diagnostics.md)**

</div>
