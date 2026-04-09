# Connection Pooling 심화 — PgBouncer 모드와 HikariCP 조합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PgBouncer의 Session/Transaction/Statement Mode는 어떻게 다른가?
- Transaction Mode에서 Prepared Statement가 깨지는 이유와 해결법은?
- HikariCP와 PgBouncer를 함께 쓸 때 최적 설정은?
- PgBouncer가 수천 개 연결을 수십 개 PostgreSQL 연결로 줄이는 원리는?
- Statement Mode는 왜 실무에서 거의 사용하지 않는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

PostgreSQL은 연결당 프로세스를 fork하는 아키텍처라 수천 개 동시 연결은 메모리와 CPU를 낭비한다. PgBouncer는 애플리케이션과 PostgreSQL 사이에서 연결을 풀링하여, 수천 개 애플리케이션 연결을 수십 개 PostgreSQL 프로세스로 처리한다. 하지만 Transaction Mode에서는 Prepared Statement, SET, 트랜잭션 상태 같은 세션 상태를 유지할 수 없어 호환성 문제가 생긴다. HikariCP와 PgBouncer를 함께 쓸 때 이 설정을 이해하지 못하면 운영 장애가 발생한다.

---

## 😱 흔한 실수 (Before — PgBouncer 오해)

```
실수 1: Transaction Mode에서 Prepared Statement 사용

  PgBouncer Transaction Mode 설정
  Spring Boot + HikariCP 기본 설정 (Prepared Statement 자동 사용)

  오류:
  org.postgresql.util.PSQLException: ERROR: prepared statement "S_1" already exists
  또는
  ERROR: prepared statement "S_1" does not exist

  원인:
  Transaction Mode에서 트랜잭션 종료 후 연결이 풀로 반환됨
  다음 트랜잭션은 다른 PostgreSQL 연결을 받을 수 있음
  이전 연결에 생성된 Prepared Statement는 새 연결에 없음
  → "does not exist" 오류

실수 2: Session Mode인데 PgBouncer 이점 없음

  PgBouncer Session Mode: 연결이 클라이언트 전체 세션 동안 하나의 PostgreSQL 연결에 고정
  → 연결 풀링 효과 없음 (1:1 매핑)
  → idle 연결도 PostgreSQL 프로세스 점유

  Session Mode가 의미 있는 경우:
  애플리케이션이 Prepared Statement, SET, 트랜잭션 상태 사용 多
  → 기능 호환성 유지하면서 연결 수 제한

실수 3: PgBouncer max_client_conn 부족

  max_client_conn = 100 (기본)
  애플리케이션 10개 × HikariCP 풀 20 = 200 연결 필요
  → "sorry, too many clients already" 오류

  설정:
  max_client_conn = 10000  (PgBouncer가 허용하는 클라이언트 연결)
  pool_size = 25           (PostgreSQL로 유지하는 연결 수)
```

---

## ✨ 올바른 접근 (After — PgBouncer Transaction Mode + HikariCP)

```
권장 아키텍처:

  애플리케이션(10개 인스턴스)
  HikariCP pool_size = 5 per instance
  → 총 50개 클라이언트 연결 → PgBouncer (5432 포트)

  PgBouncer → PostgreSQL (수십 개 연결만 유지)
  pool_mode = transaction
  pool_size = 20

  결과:
  PostgreSQL 프로세스: 20개
  vs PgBouncer 없이: 50개

PgBouncer pgbouncer.ini 설정:
  [pgbouncer]
  pool_mode = transaction        # Transaction Mode
  max_client_conn = 10000        # 클라이언트 최대 연결
  default_pool_size = 25         # DB당 PostgreSQL 연결 수
  server_idle_timeout = 600      # 유휴 서버 연결 유지 시간
  client_idle_timeout = 0        # 클라이언트 연결 타임아웃 없음
  listen_port = 5432

Spring + Transaction Mode 설정:
  application.yml:
  spring:
    datasource:
      url: jdbc:postgresql://pgbouncer:5432/mydb?prepareThreshold=0
      hikari:
        maximum-pool-size: 5     # PgBouncer 앞이므로 작게
        connection-timeout: 5000  # 5초 내 연결 못 얻으면 예외
        idle-timeout: 600000

  prepareThreshold=0:
  → PostgreSQL JDBC의 서버 사이드 Prepared Statement 비활성화
  → Transaction Mode와 완전 호환
```

---

## 🔬 내부 동작 원리

### 1. PgBouncer 3가지 풀링 모드

```
Session Mode:

  클라이언트 연결 → PostgreSQL 연결이 1:1로 고정
  클라이언트 연결 해제 시 → PostgreSQL 연결이 풀로 반환

  장점:
  ① 세션 상태 완전 유지 (SET, Prepared Statement, Temp Table)
  ② 모든 PostgreSQL 기능 완전 호환

  단점:
  ① 연결 풀링 효과 제한적
     idle 클라이언트도 PostgreSQL 연결 점유
  ② 실제 쿼리 실행 없이 연결만 유지 = 자원 낭비

  적합: 기존 애플리케이션 마이그레이션, 세션 상태 의존성 높음

Transaction Mode:

  트랜잭션 시작 → PostgreSQL 연결 할당
  트랜잭션 종료(COMMIT/ROLLBACK) → 연결 즉시 반환

  장점:
  ① 높은 연결 풀링 효과
     초당 100개 트랜잭션, 평균 10ms = 평균 동시 연결 1개
     → PostgreSQL 연결 최소화
  ② 수천 클라이언트를 수십 PostgreSQL 연결로 처리

  단점:
  ① 세션 상태 비유지: SET 영향이 트랜잭션 후 사라짐
  ② Server-side Prepared Statement 문제
  ③ LISTEN/NOTIFY 불가
  ④ WITH HOLD CURSOR 불가

  적합: 대부분의 현대 웹 애플리케이션 (세션 상태 의존성 낮음)

Statement Mode:

  각 SQL 문장 실행 후 즉시 연결 반환
  → 자동 커밋 (auto-commit) 환경에서만 의미 있음
  → 멀티 문장 트랜잭션 불가

  단점:
  ① 트랜잭션 지원 없음 (실무에서 거의 사용 불가)
  ② BEGIN/COMMIT 사이에 연결이 바뀔 수 있음 → 심각한 문제

  적합: 읽기 전용 단순 쿼리, auto-commit 환경
```

### 2. Transaction Mode와 Prepared Statement 충돌 상세

```
PostgreSQL JDBC Prepared Statement 동작:

  prepareThreshold = 5 (기본):
  → 같은 쿼리가 5번 실행되면 서버 사이드 Prepared Statement 생성
  → PREPARE stmt_name AS SELECT * FROM orders WHERE id = $1;

  Transaction Mode에서 문제:

  세션 A (연결 #1 사용):
    → 쿼리 5번 실행 → PREPARE 생성 → 연결 #1에 Prepared Statement 존재
    → 트랜잭션 종료 → 연결 #1 풀로 반환

  세션 B (연결 #2 받음):
    → 이전 PREPARE 없음 → 새로 생성 OK

  세션 A (연결 #2 받음):
    → 연결 #2에 PREPARE 없음 → EXECUTE 시도 → 오류!

해결 방법:

  방법 1: prepareThreshold=0 (권장)
    서버 사이드 Prepared Statement 완전 비활성화
    클라이언트 사이드만 사용 (JDBC 드라이버 내부에서 파라미터 바인딩)
    → Transaction Mode 완전 호환
    spring.datasource.url=jdbc:postgresql://host/db?prepareThreshold=0

  방법 2: PgBouncer Session Mode
    세션 상태가 유지되므로 Prepared Statement 정상 작동
    → 연결 풀링 효과 감소 (절충)

  방법 3: PgBouncer 1.21+의 Protocol-level Prepared Statement 지원
    PgBouncer 자체가 Prepared Statement를 트래킹
    → prepareThreshold=0 없이 Transaction Mode 사용 가능
    (아직 실험적, 프로덕션 주의)
```

### 3. PgBouncer 내부 동작

```
연결 풀 구조:

  PgBouncer 프로세스 (단일 스레드 + epoll)
  클라이언트 소켓 10,000개를 단일 이벤트 루프로 처리

  클라이언트 풀:
  client → PgBouncer 소켓: 클라이언트 수만큼 존재 (가벼운 소켓 연결)

  서버 풀 (PostgreSQL 연결):
  {active: 트랜잭션 중인 연결, idle: 대기 중인 연결}
  pool_size = 25: 최대 25개 PostgreSQL 프로세스

  라우팅 (Transaction Mode):
  클라이언트가 BEGIN → idle 서버 연결 할당 (없으면 새로 생성)
  COMMIT/ROLLBACK → 서버 연결을 idle 풀로 반환

연결 생명주기:

  PostgreSQL 연결 생성: (클라이언트가 처음 요청할 때)
  연결 재사용: (idle 연결에서 가져옴 → fork 없이 빠름)
  연결 해제: (server_idle_timeout 초과 또는 max_db_connections 초과)

admin 명령어:
  psql -h localhost -p 6432 -U pgbouncer pgbouncer
  SHOW POOLS;      -- 풀 상태 (cl_active, cl_waiting, sv_active, sv_idle)
  SHOW STATS;      -- 쿼리 통계
  SHOW CLIENTS;    -- 연결된 클라이언트 목록
  RELOAD;          -- 설정 재로드 (연결 유지)
  PAUSE db;        -- DB 일시 정지 (배포 시)
  RESUME db;       -- 재개

주요 지표:
  cl_active:    쿼리 실행 중인 클라이언트
  cl_waiting:   서버 연결 대기 중인 클라이언트 → pool_size 증가 필요
  sv_active:    트랜잭션 중인 PostgreSQL 연결
  sv_idle:      대기 중인 PostgreSQL 연결
  sv_used:      최근 사용됐지만 반환된 연결
```

### 4. HikariCP + PgBouncer 최적 조합

```
이중 풀링 전략:

  [앱] → [HikariCP] → [PgBouncer] → [PostgreSQL]

  HikariCP 역할:
  ① 애플리케이션 연결 재사용 (PgBouncer와의 연결 재사용)
  ② 연결 건강 체크 (connectionTestQuery)
  ③ 최대 연결 수 제한

  PgBouncer 역할:
  ① 여러 앱 인스턴스의 연결을 단일 PostgreSQL 풀로 통합
  ② PostgreSQL 프로세스 수 최소화

  권장 설정:

  HikariCP (각 앱 인스턴스):
    maximumPoolSize = 5~10   # PgBouncer가 있으므로 작게
    minimumIdle = 2          # 최소 유지 연결
    connectionTimeout = 5000 # 5초 (PgBouncer 응답 대기)
    idleTimeout = 300000     # 5분 idle → 반환
    maxLifetime = 1800000    # 30분마다 연결 교체
    keepaliveTime = 60000    # 60초 keepalive

  PgBouncer (모든 앱 인스턴스 합산):
    default_pool_size = 20   # PostgreSQL 실제 연결 수
    max_client_conn = 5000   # 클라이언트 최대 연결
    pool_mode = transaction

  계산 예:
  앱 20개 인스턴스 × HikariCP 5 = 최대 100개 클라이언트 연결
  PgBouncer pool_size = 20 → PostgreSQL 프로세스 20개만

  PgBouncer 없이:
  앱 20개 × HikariCP 20 = 400개 PostgreSQL 프로세스
  → 메모리: 400 × 10MB = 4GB 프로세스 오버헤드

  PgBouncer 있이:
  PostgreSQL 프로세스 20개 × 10MB = 200MB
  → 20배 절약!
```

---

## 💻 실전 실험

### 실험 1: PgBouncer 연결 상태 모니터링

```bash
# PgBouncer 관리 콘솔 접속 (pgbouncer 데이터베이스로 접속)
psql -h localhost -p 6432 -U pgbouncer pgbouncer

-- 풀 상태 확인
SHOW POOLS;
-- database | user | cl_active | cl_waiting | sv_active | sv_idle | sv_used | maxwait
-- mydb     | app  |     45    |      0     |     18    |   7     |   2     |   0

-- 클라이언트 목록
SHOW CLIENTS;

-- 서버(PostgreSQL) 연결 목록
SHOW SERVERS;

-- 통계
SHOW STATS;
-- total_query_count, total_wait_time, avg_query_time 등

-- 설정 확인
SHOW CONFIG;
```

### 실험 2: prepareThreshold 설정 효과 비교

```bash
# prepareThreshold=0 없이 Transaction Mode (오류 발생 가능)
psql "host=pgbouncer port=5432 dbname=mydb"
# 같은 쿼리 5번 실행 후 Prepared Statement 생성됨
# 연결이 바뀌면 "does not exist" 오류

# prepareThreshold=0 적용 후 (정상)
psql "host=pgbouncer port=5432 dbname=mydb?prepareThreshold=0"
# Prepared Statement 사용 안 함 → Transaction Mode 완전 호환
```

```sql
-- PostgreSQL에서 현재 Prepared Statement 목록
SELECT name, statement, prepare_time
FROM pg_prepared_statements;
-- prepareThreshold=0이면 항상 비어있음
```

### 실험 3: PgBouncer 설정 파일 작성

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
# 연결할 데이터베이스 정의
mydb = host=pg-primary port=5432 dbname=mydb

# Read-Only 풀 (Standby)
mydb_readonly = host=pg-standby port=5432 dbname=mydb

[pgbouncer]
# 모드 설정
pool_mode = transaction

# 연결 수 설정
max_client_conn = 10000
default_pool_size = 25
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 5

# 타임아웃
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 0
query_wait_timeout = 120

# 인증
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# 로깅
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1

# 관리
listen_addr = *
listen_port = 5432
admin_users = pgbouncer
stats_users = stats_user

# TLS (선택)
# server_tls_sslmode = require
```

---

## 📊 MySQL과 비교

```
MySQL Connection Pooling vs PostgreSQL + PgBouncer:

MySQL:
  MySQL 자체: Thread Pool (Enterprise 기능)
  ProxySQL: MySQL용 Connection Pooler
    → Statement 라우팅, Read/Write 분리, 캐싱
  MySQL Connector/J: 클라이언트 사이드 풀링 (HikariCP 같은)

PostgreSQL:
  PgBouncer: 가장 범용
  pgpool-II: 로드 밸런싱, 병렬 쿼리, 연결 풀링 (복잡)
  Odyssey: Yandex 개발, 고성능

연결 아키텍처 차이:
  MySQL (Thread): 연결당 Thread → Thread Pool로 최적화 가능
  PostgreSQL (Process): 연결당 Process → PgBouncer 필수적

ProxySQL vs PgBouncer:
  ProxySQL: Read/Write 분리, 쿼리 캐싱, 복잡한 라우팅
  PgBouncer: 단순 연결 풀링, 가볍고 빠름
  → PostgreSQL에서 PgBouncer + HAProxy 조합이 표준
```

---

## ⚖️ 트레이드오프

```
PgBouncer 모드 선택:

Session Mode:
  적합: 세션 상태 의존성 높은 레거시 앱
  단점: 풀링 효과 낮음

Transaction Mode (권장):
  적합: 현대 Spring/Node.js 앱 (stateless)
  단점: Prepared Statement, SET 유지 안 됨
  해결: prepareThreshold=0

Statement Mode:
  적합: 거의 없음 (auto-commit 단순 조회)
  단점: 트랜잭션 불가

HikariCP pool_size 결정:
  PgBouncer 없을 때: CPU 코어 수 × 2 ~ 4
  PgBouncer 있을 때: 5~10 (PgBouncer가 PostgreSQL 연결 통합)

pool_size 너무 크면:
  PgBouncer 효과 감소 (모든 HikariCP 연결 = PgBouncer 서버 연결)
  연결 자원 낭비

pool_size 너무 작으면:
  HikariCP 연결 대기 → 응답 지연
  connectionTimeout 초과 → 예외
```

---

## 📌 핵심 정리

```
Connection Pooling 핵심:

PgBouncer 모드:
  Session: 세션 전체 연결 고정 (풀링 효과 낮음, 완전 호환)
  Transaction: 트랜잭션 단위 전환 (풀링 효과 높음, 세션 상태 없음)
  Statement: 문장 단위 (실무 거의 미사용)

Transaction Mode 호환성:
  prepareThreshold=0: Prepared Statement 비활성화
  SET 세션 변수: 트랜잭션 내에서만 유효
  LISTEN/NOTIFY, 임시 테이블: 미지원

HikariCP + PgBouncer:
  HikariCP pool_size 작게 (5~10): PgBouncer가 통합 관리
  PgBouncer pool_size: 실제 PostgreSQL 연결 수 (20~50)
  prepareThreshold=0 JDBC URL에 추가

모니터링:
  SHOW POOLS: cl_waiting > 0이면 pool_size 증가 필요
  sv_idle: 사용 안 하는 연결 → server_idle_timeout 조정
  avg_query_time: PgBouncer 오버헤드 확인

다운타임 없는 재로드:
  RELOAD; (PgBouncer 관리 콘솔에서)
  → 기존 연결 유지하면서 설정 변경
```

---

## 🤔 생각해볼 문제

**Q1.** Spring `@Transactional` 메서드 안에서 `SET TIME ZONE 'Asia/Seoul'`을 실행하면 Transaction Mode에서 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Transaction Mode에서 트랜잭션 종료 후 연결이 풀에 반환될 때 PgBouncer는 세션 상태를 초기화합니다. `SET TIME ZONE`은 세션 설정이므로 트랜잭션 종료 후 다른 클라이언트가 이 연결을 받으면 타임존이 초기화된 상태가 됩니다.

PgBouncer Transaction Mode의 세션 상태 초기화:
- `server_reset_query = DISCARD ALL` (기본): 각 연결 반환 시 `DISCARD ALL` 실행
  → SET 변수, Prepared Statement, 임시 테이블 모두 초기화

해결 방법:
1. **트랜잭션 내에서 SET**: `@Transactional` 안의 SET은 해당 트랜잭션에서만 유효 → 일관성 유지
2. **Connection 수준 설정**: JDBC URL에 `options='-c timezone=Asia/Seoul'` 추가
3. **PgBouncer `server_reset_query_always = 0`**: 오염된 연결만 초기화 (성능 향상)

</details>

---

**Q2.** `cl_waiting > 0`이 지속적으로 발생한다. 어떤 설정을 조정해야 하는가?

<details>
<summary>해설 보기</summary>

`cl_waiting`은 PgBouncer가 클라이언트에게 PostgreSQL 연결을 할당하지 못해 대기 중인 상태입니다. 원인과 해결:

1. **`default_pool_size` 부족** (가장 흔함):
   - PostgreSQL 연결이 모두 사용 중 → pool_size 증가
   - 단, PostgreSQL `max_connections`와 메모리 고려

2. **PostgreSQL 쿼리가 느림**:
   - 연결이 오래 점유됨 → 쿼리 최적화 필요
   - `SHOW STATS`의 `avg_query_time` 확인

3. **`max_client_conn` 부족**:
   - 클라이언트 연결 자체가 거부됨 → 증가

4. **`reserve_pool_size` 활용**:
   ```ini
   reserve_pool_size = 5
   reserve_pool_timeout = 3
   ```
   → 3초 대기 후 reserve pool에서 추가 연결 할당

진단:
```
SHOW POOLS;
-- maxwait 컬럼: 현재 가장 오래 대기 중인 클라이언트의 대기 시간(초)
-- maxwait > 0이면 pool_size 증가 또는 쿼리 최적화 필요
```

</details>

---

<div align="center">

**[⬅️ 이전: Patroni 고가용성](./04-patroni-ha.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: postgresql.conf 핵심 설정 ➡️](../performance-tuning/01-postgresql-conf.md)**

</div>
