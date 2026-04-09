# PostgreSQL vs MySQL 아키텍처 비교 — 연결당 프로세스 fork가 만드는 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PostgreSQL은 왜 연결당 스레드가 아닌 **프로세스**를 fork하는가?
- `shared_buffers`와 OS Page Cache가 이중으로 존재하면 메모리가 낭비되지 않는가?
- PostgreSQL에서 `max_connections`를 높이면 왜 성능이 오히려 떨어지는가?
- PgBouncer 없이 Spring 애플리케이션을 운영하면 어떤 문제가 발생하는가?
- MySQL의 스레드 기반 아키텍처와 PostgreSQL의 프로세스 기반 아키텍처 중 어느 것이 더 나은가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL을 처음 운영하면 반드시 마주치는 문제가 있다. HikariCP를 기본 설정으로 사용하는 Spring Boot 서비스를 여러 인스턴스 배포하면, `max_connections`를 초과하며 연결이 거부된다. `max_connections`를 높이면 이번엔 메모리가 부족해진다. 그 이유는 PostgreSQL이 **연결당 OS 프로세스**를 하나씩 fork하기 때문이다. 프로세스당 최소 5~10MB의 메모리가 소요되고, 1000개 연결이면 5~10GB가 프로세스 오버헤드에만 쓰인다. 이 구조를 이해하지 못하면 Connection Pooling이 왜 선택이 아닌 필수인지 납득하지 못한다.

---

## 😱 흔한 실수 (Before — MySQL 방식 그대로 적용)

```
상황: MySQL에서 PostgreSQL로 마이그레이션
      Spring Boot 4개 인스턴스, HikariCP 기본값 (pool size 10)

MySQL에서의 경험:
  max_connections=500 → 여유롭게 운영 가능
  스레드 기반이므로 연결 생성 비용이 낮음

PostgreSQL에서 그대로 적용:
  max_connections=500 설정
  4개 서비스 × 10 pool = 40 연결 → "충분해 보임"

트래픽 증가로 인스턴스를 20개로 수평 확장:
  20 × 10 = 200 연결
  → 아직 max_connections 이내이므로 문제없어 보임

문제 발생:
  PostgreSQL 서버 메모리 사용량 급등
  200개 연결 × 프로세스당 8MB = 1.6GB (프로세스 오버헤드만)
  shared_buffers에 할당해둔 메모리가 실질적으로 줄어듦
  work_mem 16MB × 200 연결 = 최대 3.2GB (정렬/해시 조인 시)
  → 서버 메모리 압박, OOM 발생

또 다른 실수:
  모든 연결이 항상 active한 게 아닌데도 상시 연결 유지
  1개의 HTTP 요청 = 1개의 DB 연결 (HikariCP pool에서 빌려씀)
  → idle 연결도 PostgreSQL에서는 프로세스를 유지
  → 자원 낭비
```

---

## ✨ 올바른 접근 (After — PostgreSQL 아키텍처 이해 후)

```
올바른 설계:

  PostgreSQL max_connections:
    일반적으로 100~200 (PgBouncer 앞에 두는 경우)
    절대로 수천 개로 설정하지 않음

  PgBouncer Transaction Mode:
    애플리케이션 ↔ PgBouncer: 수천 연결 허용 (소켓만, 프로세스 없음)
    PgBouncer ↔ PostgreSQL: 실제 프로세스는 20~50개만 유지
    → "쿼리를 실행하는 순간에만 PostgreSQL 프로세스 사용"

  Spring Boot 설정:
    # JDBC URL을 PgBouncer 포트로 변경
    spring.datasource.url=jdbc:postgresql://pgbouncer:5432/mydb
    # HikariCP pool size는 PgBouncer 앞단이므로 느슨하게
    spring.datasource.hikari.maximum-pool-size=20

  PostgreSQL 직접 연결 시 (PgBouncer 없이):
    서비스 인스턴스 수 × pool size < max_connections × 0.8 유지
    max_connections = (RAM - shared_buffers) / (work_mem × 최대_동시_쿼리) 계산 필요
```

---

## 🔬 내부 동작 원리

### 1. PostgreSQL 프로세스 구조

```
PostgreSQL 프로세스 계층:

postmaster (PID 1234) — 슈퍼바이저 프로세스
  │
  ├── checkpointer    — WAL Checkpoint 수행
  ├── background writer — Dirty Page를 Disk로 주기적 flush
  ├── walwriter       — WAL 버퍼를 WAL 파일로 flush
  ├── autovacuum launcher — Autovacuum Worker 생성 관리
  ├── stats collector — pg_stat_* 통계 수집
  ├── logical replication launcher
  │
  ├── backend (PID 2001) ← 클라이언트 1 연결 시 fork()
  ├── backend (PID 2002) ← 클라이언트 2 연결 시 fork()
  ├── backend (PID 2003) ← 클라이언트 3 연결 시 fork()
  └── ...

클라이언트 연결 흐름:
  1. 클라이언트 → postmaster:5432 TCP 연결
  2. postmaster: fork() 호출 → 자식 프로세스 생성 (backend)
  3. backend 프로세스가 클라이언트 소켓 인계받아 전담 처리
  4. 연결 종료 시 backend 프로세스 exit()

프로세스당 메모리 구성:
  ┌─────────────────────────────────────┐
  │ backend 프로세스 (PID 2001)           │
  │                                     │
  │ 고정 오버헤드: ~5MB                    │
  │   - 프로세스 스택                      │
  │   - 로컬 캐시 (catalog cache 등)      │
  │   - PostgreSQL 내부 구조체             │
  │                                     │
  │ work_mem: 설정값 (기본 4MB)            │
  │   - 정렬/해시 조인 수행 시 할당          │
  │   - 한 쿼리에서 여러 Sort 노드가        │
  │     있으면 work_mem × 노드 수 소비      │
  │                                     │
  │ temp_buffers: 기본 8MB               │
  │   - 임시 테이블 저장                   │
  └─────────────────────────────────────┘
```

### 2. MySQL 스레드 기반 vs PostgreSQL 프로세스 기반

```
MySQL (InnoDB) 스레드 모델:
  
  mysqld 단일 프로세스
    │
    ├── Thread Pool (or One-Thread-Per-Connection)
    │     ├── worker thread 1 ← 클라이언트 1
    │     ├── worker thread 2 ← 클라이언트 2
    │     └── worker thread N ← 클라이언트 N
    │
    └── 모든 스레드가 같은 프로세스 주소 공간 공유
          → 데이터 공유: 포인터 직접 전달
          → 동기화: Mutex/RW Lock 필요
          → 스레드 생성 비용: ~수 KB (스택만)
          → 1,000 스레드: ~1GB (스택 1MB 가정)

PostgreSQL 프로세스 모델:

  postmaster 프로세스
    │
    ├── backend 1 (fork)  ← 클라이언트 1
    ├── backend 2 (fork)  ← 클라이언트 2
    └── backend N (fork)  ← 클라이언트 N
         → 각각 독립된 주소 공간
         → 데이터 공유: Shared Memory(shared_buffers)를 통해서만
         → 동기화: LWLock (Lightweight Lock)
         → 프로세스 생성 비용: ~5~10MB (완전한 프로세스)
         → 1,000 프로세스: ~5~10GB

비교 표:
항목                | MySQL (Thread)  | PostgreSQL (Process)
───────────────────┼────────────────┼──────────────────────
연결당 메모리        | ~1MB           | ~5~10MB
1,000 연결 오버헤드 | ~1GB           | ~5~10GB
프로세스간 격리      | 낮음 (동일 주소) | 높음 (독립 주소)
한 연결 크래시 영향  | 전체 위험        | 해당 프로세스만
멀티코어 활용        | 쉬움            | Shared Memory로 조율
Connection Pooling | 권장            | 필수

PostgreSQL이 프로세스 모델을 선택한 이유:
  ① 격리성: 하나의 backend가 메모리 손상을 일으켜도
              다른 backend와 postmaster는 영향받지 않음
  ② 안정성 우선: PostgreSQL 역사적으로
                 "데이터 안전성 > 연결 효율성" 설계 철학
  ③ Copy-On-Write: fork() 후 메모리 페이지를 실제로 복사하지 않고
                   수정 시에만 복사 → 실제 메모리 사용은 낮음
```

### 3. Shared Memory (shared_buffers) 구조

```
PostgreSQL 메모리 아키텍처:

  ┌────────────────────────────────────────────────────────┐
  │                  OS 메모리                               │
  │                                                        │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │          PostgreSQL Shared Memory                 │  │
  │  │                                                  │  │
  │  │  shared_buffers (기본 128MB, 권장 RAM의 25%)        │  │
  │  │  ┌────────────────────────────────────────────┐  │  │
  │  │  │  Buffer Pool — 8KB 페이지 캐시              │  │  │
  │  │  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐      │  │  │
  │  │  │  │Page 1│ │Page 2│ │Page 3│ │Page N│ ...  │  │  │
  │  │  │  └──────┘ └──────┘ └──────┘ └──────┘      │  │  │
  │  │  └────────────────────────────────────────────┘  │  │
  │  │                                                  │  │
  │  │  WAL Buffer — WAL 레코드 임시 저장                  │  │
  │  │  Lock Table — 행/테이블 잠금 관리                   │  │
  │  │  Proc Array — 활성 트랜잭션 정보 (가시성 판단용)       │  │
  │  │  CLOG — 트랜잭션 커밋 상태 (Commit Log)             │  │
  │  └──────────────────────────────────────────────────┘  │
  │                                                        │
  │  ┌─────────────────┐  ┌─────────────────┐              │
  │  │  backend 1      │  │  backend 2      │              │
  │  │  (local memory) │  │  (local memory) │              │
  │  │  work_mem       │  │  work_mem       │              │
  │  │  temp_buffers   │  │  temp_buffers   │              │
  │  └─────────────────┘  └─────────────────┘              │
  │                                                        │
  │  ┌────────────────────────────────────────────────┐    │
  │  │  OS Page Cache (커널이 관리)                      │    │
  │  │  PostgreSQL이 읽은 파일 데이터를 OS가 캐싱          │    │
  │  └────────────────────────────────────────────────┘    │
  └────────────────────────────────────────────────────────┘

이중 캐시 문제:
  shared_buffers에 있는 페이지 → 디스크에서 읽을 때 OS Page Cache에도 저장
  → 같은 데이터가 두 곳에 캐싱됨
  → PostgreSQL은 이를 인지하고 effective_cache_size로 OS 캐시 존재를 플래너에 힌트로 제공

effective_cache_size:
  실제로 메모리를 할당하지 않음! (힌트값일 뿐)
  플래너에게 "OS Page Cache + shared_buffers 합쳐서 이 정도 있으니
  Index Scan이 메모리에서 처리될 가능성 계산에 참고해라"
  일반적으로 서버 전체 RAM의 75% 설정

MySQL과 비교:
  MySQL: Buffer Pool 하나에 집중 (OS 캐시와 이중화 최소화)
         → O_DIRECT 옵션으로 OS 캐시 우회 가능
  PostgreSQL: O_DIRECT 미지원 → 이중 캐시 감수
              대신 effective_cache_size로 플래너 보정
```

### 4. 연결 수가 성능에 미치는 영향

```
max_connections를 높일수록 손해인 이유:

① 메모리 압박
   각 backend가 local memory 확보
   work_mem이 4MB일 때 연결 1,000개
   → work_mem 최대 소비: 4MB × 1,000 = 4GB
   (실제로 모든 연결이 동시에 정렬하는 경우)

② Lock 경합 증가
   Shared Memory의 Lock Table 크기 고정
   연결이 많을수록 같은 Lock을 두고 경쟁

③ Context Switch 증가
   backend 프로세스 수 = OS 스케줄링 부담
   200개 프로세스가 동시에 실행 요청
   → CPU가 프로세스 전환하며 캐시 miss 발생

④ VACUUM 간섭
   old snapshot 문제: 연결 수가 많으면
   oldest 트랜잭션이 오래 살아남을 가능성 ↑
   → VACUUM이 Dead Tuple을 회수 못함 (Ch2에서 상세 설명)

실측 데이터 (참고):
  연결 100개: throughput 100%
  연결 200개: throughput ~95%
  연결 500개: throughput ~80%
  연결 1,000개: throughput ~60% (context switch 오버헤드)
  → 연결 수가 늘어도 처리량은 선형으로 증가하지 않음
```

---

## 💻 실전 실험

### 실험 1: 현재 연결 상태 확인

```sql
-- 연결 현황 전체 보기
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - state_change AS duration_in_state,
    left(query, 50) AS current_query
FROM pg_stat_activity
ORDER BY state, duration_in_state DESC;

-- 상태별 연결 수 요약
SELECT
    state,
    count(*) AS count,
    max(now() - state_change) AS max_duration
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state
ORDER BY count DESC;

-- idle 연결이 10분 이상 유지되는 경우 (연결 풀 문제 신호)
SELECT pid, usename, application_name, client_addr,
       now() - state_change AS idle_duration
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > interval '10 minutes'
ORDER BY idle_duration DESC;
```

### 실험 2: 메모리 사용량 확인

```bash
# PostgreSQL 프로세스별 메모리 확인
ps aux | grep postgres | awk '{print $6, $11}' | sort -rn | head -20
# RSS (실제 사용 메모리, KB 단위)

# shared_buffers 현재 설정 확인
psql -c "SHOW shared_buffers;"
psql -c "SHOW work_mem;"
psql -c "SHOW maintenance_work_mem;"

# 현재 shared_buffers 실제 사용량
SELECT
    count(*) AS total_buffers,
    count(*) FILTER (WHERE NOT isdirty) AS clean_buffers,
    count(*) FILTER (WHERE isdirty) AS dirty_buffers,
    pg_size_pretty(count(*) * 8192) AS total_size_used
FROM pg_buffercache;
-- pg_buffercache 확장 필요: CREATE EXTENSION pg_buffercache;
```

### 실험 3: 연결 생성 비용 측정

```bash
# 연결 100개 순차 생성 시간 측정
time for i in $(seq 1 100); do
    psql -c "SELECT 1;" > /dev/null
done
# 직접 연결 시: 연결당 수십 ms (fork 비용)

# PgBouncer 경유 시
time for i in $(seq 1 100); do
    psql -h pgbouncer-host -p 5432 -c "SELECT 1;" > /dev/null
done
# PgBouncer: 연결당 ~1ms (소켓 재사용)
```

---

## 📊 MySQL과 비교

```
연결 처리 방식 비교 실험:

1,000개 동시 연결, 각각 단순 SELECT 쿼리 실행

MySQL (기본 Thread 모델):
  - 메모리: ~1GB (thread stack 포함)
  - 연결 생성: ~1ms
  - CPU: 스레드 관리 오버헤드 낮음

PostgreSQL (프로세스 모델, PgBouncer 없음):
  - 메모리: ~5~10GB
  - 연결 생성: ~5~10ms (fork() 비용)
  - CPU: 프로세스 context switch 오버헤드 있음

PostgreSQL (프로세스 모델, PgBouncer Transaction Mode):
  - 실제 PostgreSQL 연결: 20~50개
  - 메모리: shared_buffers + 50 × 8MB = 최소화
  - 연결 생성(PgBouncer 기준): <1ms
  - CPU: 최소

결론:
  MySQL: Connection Pooling 권장 (성능 향상)
  PostgreSQL: Connection Pooling 필수 (없으면 확장 불가)
```

---

## ⚖️ 트레이드오프

```
프로세스 기반 모델의 장단점:

장점:
  ① 강한 격리: backend 하나가 Segfault → 해당 연결만 종료
               MySQL Thread 모델: 하나의 잘못된 플러그인이 전체 mysqld 크래시 가능
  ② 메모리 보호: 프로세스 간 주소 공간 독립 → 버그가 다른 연결 데이터 오염 불가
  ③ Copy-On-Write: fork() 후 공통 메모리 페이지 공유
                   (실제 메모리 소비는 수치보다 낮음)

단점:
  ① 연결 생성 비용: fork()는 clone()보다 무거움 (5~10ms)
  ② 프로세스당 오버헤드: 최소 5MB, 1,000 연결 = 5GB
  ③ IPC 복잡성: 프로세스 간 통신은 Shared Memory를 통해서만
                → 구현 복잡도 증가

현실적 결론:
  PgBouncer를 앞에 두면 프로세스 모델의 단점이 대부분 상쇄됨
  PostgreSQL 직접 연결 제한 → 실제 동시 작업 수만큼만 유지
  대부분의 운영 환경에서 PostgreSQL: max_connections=100~300이 최적
```

---

## 📌 핵심 정리

```
PostgreSQL 아키텍처 핵심:

연결 처리:
  클라이언트 연결 → postmaster가 backend 프로세스 fork()
  → 프로세스당 5~10MB 오버헤드
  → max_connections = 수천으로 설정하면 안 됨

메모리 구조:
  shared_buffers (모든 backend 공유): DB 페이지 캐시
  local memory (backend별 독립): work_mem, temp_buffers
  OS Page Cache: 파일 읽을 때 OS가 자동으로 캐싱 → 이중 캐시

PgBouncer가 필수인 이유:
  Transaction Mode: 쿼리 실행하는 순간에만 PostgreSQL 연결 사용
  → 수천 개 애플리케이션 연결 → 수십 개 PostgreSQL 프로세스

설정 가이드:
  shared_buffers = RAM × 0.25
  effective_cache_size = RAM × 0.75  (플래너 힌트, 실제 할당 아님)
  work_mem = 4~16MB                  (연결 수 × work_mem 총합 고려)
  max_connections = 100~300          (PgBouncer 사용 전제)
```

---

## 🤔 생각해볼 문제

**Q1.** `max_connections=100`으로 설정하고 PgBouncer Transaction Mode로 10,000개 애플리케이션 연결을 받는다면, `work_mem=64MB`로 설정할 때 실제로 메모리가 얼마나 필요한가?

<details>
<summary>해설 보기</summary>

PostgreSQL 프로세스는 100개이므로, `work_mem`의 최대 소비는 `64MB × 100 = 6.4GB`입니다. 단, `work_mem`은 **정렬이나 해시 조인이 실행될 때만** 할당되며, 하나의 쿼리에서 여러 Sort/Hash 노드가 있으면 `work_mem × 노드 수`가 됩니다.

실제로 모든 100개 연결이 동시에 정렬을 수행하는 경우는 드물기 때문에, 평균 동시 정렬 쿼리 수를 추정해서 설정해야 합니다. 대부분의 운영 환경에서는 `work_mem × max_connections × 평균_동시_정렬_비율`로 계산합니다.

PgBouncer가 10,000개 연결을 받아도 PostgreSQL 프로세스는 100개이므로, `work_mem` 최대 소비는 PostgreSQL 프로세스 수 기준으로 계산합니다.

</details>

---

**Q2.** PostgreSQL에서 `SHOW max_connections`가 200인데, `pg_stat_activity`에서 연결이 150개일 때 새 연결이 거부되었다. 왜인가?

<details>
<summary>해설 보기</summary>

PostgreSQL은 `max_connections`에서 `superuser_reserved_connections`(기본 3)을 슈퍼유저 전용으로 예약합니다. 일반 사용자는 `max_connections - superuser_reserved_connections = 197`개까지만 연결할 수 있습니다.

또한, Autovacuum Worker, WAL Sender 등 내부 프로세스도 연결을 사용합니다. `pg_stat_activity`에서 `backend_type`이 `autovacuum worker`, `walsender` 등인 연결도 카운트됩니다.

```sql
-- 실제 연결 수 확인 (타입별)
SELECT backend_type, count(*)
FROM pg_stat_activity
GROUP BY backend_type
ORDER BY count DESC;
```

</details>

---

**Q3.** PgBouncer Transaction Mode에서 `Prepared Statement`가 동작하지 않는다. 왜이고 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

PgBouncer Transaction Mode는 트랜잭션이 끝나면 PostgreSQL 연결을 풀에 반환합니다. `PREPARE statement` 후 다음 쿼리에서 `EXECUTE statement`를 실행할 때, 이미 다른 PostgreSQL 연결에 배정될 수 있습니다. Prepared Statement는 해당 연결(세션)에만 존재하므로 "prepared statement does not exist" 오류가 발생합니다.

해결 방법:
1. **Protocol-level Prepared Statement 비활성화**: JDBC에서 `prepareThreshold=0` 설정
2. **Session Mode 사용**: Prepared Statement가 반드시 필요하면 PgBouncer Session Mode
3. **Statement-level Prepared Statement**: `SET plan_cache_mode = force_generic_plan`
4. **PgBouncer 1.21+ 이후**: Protocol-level prepared statements를 일부 지원

Spring Boot + HikariCP 기본 설정에서는 Prepared Statement를 내부적으로 사용하므로, PgBouncer Transaction Mode 사용 시 반드시 `prepareThreshold=0`을 JDBC URL에 추가해야 합니다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://pgbouncer:5432/mydb?prepareThreshold=0
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Storage Manager와 페이지 구조 ➡️](./02-storage-manager-page.md)**

</div>
