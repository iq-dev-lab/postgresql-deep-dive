# 논리 복제(Logical Replication) — CDC와 이기종 복제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Logical Replication은 Physical Replication과 어떻게 다른가?
- Publication/Subscription 모델은 어떻게 동작하는가?
- Logical Replication으로 이기종 PostgreSQL 버전 간 복제가 가능한 이유는?
- CDC(Change Data Capture)에 PostgreSQL Logical Replication이 어떻게 활용되는가?
- Logical Replication의 한계(DDL 복제 불가 등)와 대응 방법은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

Physical Replication은 WAL을 그대로 전달하므로 동일 버전의 PostgreSQL 인스턴스만 Standby로 사용할 수 있다. 반면 Logical Replication은 "어떤 행이 INSERT/UPDATE/DELETE됐는가"라는 논리적 변경을 전달하므로 다른 버전, 다른 OS, 심지어 다른 DBMS로도 복제할 수 있다. 또한 테이블 단위로 선택적 복제가 가능해 마이크로서비스 데이터 동기화, 무중단 메이저 버전 업그레이드, Kafka/Elasticsearch 데이터 파이프라인(CDC)에 활용된다.

---

## 😱 흔한 실수 (Before — Logical Replication 오해)

```
실수 1: DDL이 자동으로 복제된다고 가정

  Publisher에서:
  ALTER TABLE orders ADD COLUMN note TEXT;

  Subscriber: 컬럼이 추가되지 않음!
  → Logical Replication은 DML만 복제 (INSERT/UPDATE/DELETE)
  → DDL은 수동으로 양쪽에 적용해야 함

  ALTER TABLE orders ADD COLUMN note TEXT; -- Publisher
  -- Subscriber에서도 별도 실행 필요
  ALTER TABLE orders ADD COLUMN note TEXT; -- Subscriber

실수 2: Logical Replication Slot 미관리

  Physical Replication Slot과 동일한 문제:
  Subscriber가 오프라인이면 WAL(logical decoding에 필요한)이 보존
  → pg_wal/ 무한 증가 → 디스크 꽉 참

  SELECT slot_name, active, catalog_xmin, restart_lsn
  FROM pg_replication_slots
  WHERE slot_type = 'logical';

  inactive Slot의 retained_wal 모니터링 필수

실수 3: 초기 동기화 중 타임아웃

  CREATE SUBSCRIPTION sub_orders ...;
  → 대용량 테이블(100GB)의 초기 스냅샷 복사 시작
  → 몇 시간 소요 → 타임아웃 발생
  → wal_sender_timeout, wal_receiver_timeout 증가 필요
```

---

## ✨ 올바른 접근 (After — Logical Replication 올바른 활용)

```
Logical Replication 설정:

  Publisher (postgresql.conf):
  wal_level = logical  # logical 필수 (replica보다 상위)
  max_replication_slots = 10
  max_wal_senders = 10

  Publisher에서 Publication 생성:
  -- 특정 테이블만 복제
  CREATE PUBLICATION pub_orders FOR TABLE orders, order_items;
  -- 전체 테이블 복제
  CREATE PUBLICATION pub_all FOR ALL TABLES;
  -- 조건부 복제 (PostgreSQL 15+)
  CREATE PUBLICATION pub_active_users FOR TABLE users WHERE (active = true);

  Subscriber에서 Subscription 생성:
  -- 테이블이 미리 존재해야 함 (DDL 수동 실행 필요)
  CREATE TABLE orders (...);  -- Publisher와 동일 구조

  CREATE SUBSCRIPTION sub_orders
  CONNECTION 'host=publisher-host dbname=mydb user=replication password=...'
  PUBLICATION pub_orders;

  복제 상태 확인:
  -- Publisher:
  SELECT * FROM pg_stat_replication WHERE application_name = 'sub_orders';

  -- Subscriber:
  SELECT * FROM pg_stat_subscription;
  SELECT * FROM pg_subscription_rel;  -- 각 테이블 복제 상태
```

---

## 🔬 내부 동작 원리

### 1. Logical Decoding 메커니즘

```
WAL 레코드 → 논리적 변경 변환:

Physical WAL 레코드:
  rmgr=Heap, tx=501, lsn=0/1234, desc="UPDATE off 3: (old xmax=500, bitmask=...) new xmax=501"
  → 페이지/오프셋 수준의 물리적 변경

Logical Decoding 결과:
  table orders: UPDATE old=(id=1, status='PENDING') new=(id=1, status='SHIPPED')
  → 행 수준의 논리적 변경

변환 과정:
  ① WAL 레코드에서 Row ID 추출
  ② 해당 시점의 Row 버전 재구성 (pg_catalog 참조)
  ③ 테이블명, 컬럼명, 값으로 변환
  ④ pgoutput 플러그인: PostgreSQL 표준 프로토콜로 인코딩
     wal2json: JSON 형식
     decoderbufs: Protobuf 형식

Output Plugin:
  pgoutput (내장): Logical Replication 표준
  wal2json: JSON → Kafka, Debezium 연동
  decoderbufs: Protobuf → Debezium

wal_level = logical 이 필요한 이유:
  wal_level = replica: 물리적 변경만 기록 (컬럼 값 없음)
  wal_level = logical: 컬럼 값 포함 (row-level 변경 정보)
```

### 2. Publication / Subscription 모델

```
Publication (발행):

  Publisher 인스턴스에서 정의
  어떤 테이블의 어떤 변경을 공개할지 선언

  CREATE PUBLICATION pub_orders
  FOR TABLE orders
  WITH (publish = 'insert, update, delete');
  -- publish 옵션: insert, update, delete, truncate (기본: 모두)

  행 필터 (PostgreSQL 15+):
  CREATE PUBLICATION pub_active FOR TABLE users
  WHERE (status = 'ACTIVE');
  → status='ACTIVE'인 행의 변경만 복제

  컬럼 목록 (PostgreSQL 15+):
  CREATE PUBLICATION pub_partial FOR TABLE users
  (id, name, email);  -- 민감 정보(password) 제외

Subscription (구독):

  Subscriber 인스턴스에서 정의
  Publisher의 Publication을 구독

  CREATE SUBSCRIPTION sub_orders
  CONNECTION 'host=pub-host dbname=mydb user=repl_user'
  PUBLICATION pub_orders
  WITH (copy_data = true,  -- 초기 스냅샷 복사 여부
        create_slot = true,  -- 자동 Slot 생성
        slot_name = 'sub_orders_slot');

  초기 동기화 (copy_data = true):
  ① Publisher: 현재 시점의 스냅샷 생성
  ② 테이블 데이터를 COPY 형식으로 전송
  ③ 초기 COPY 완료 후 그 이후 변경사항부터 WAL 스트리밍

복제 Identity:
  DELETE/UPDATE WAL에 구 버전(OLD) 포함 필요
  기본: PRIMARY KEY 컬럼만 포함
  REPLICA IDENTITY FULL: 모든 컬럼 포함 (더 안전, WAL 커짐)
  → PK 없는 테이블: REPLICA IDENTITY FULL 또는 인덱스 설정 필요

  ALTER TABLE orders REPLICA IDENTITY FULL;
  또는
  ALTER TABLE orders REPLICA IDENTITY USING INDEX orders_pkey;
```

### 3. 이기종 버전 업그레이드 패턴

```
무중단 메이저 버전 업그레이드 (PG 14 → PG 16):

  기존: PostgreSQL 14 (Primary)
  목표: PostgreSQL 16으로 업그레이드

  방법:
  ① PG 16 신규 인스턴스 준비
  ② PG 14 → PG 16 Logical Replication 설정 (이기종 가능!)
     PG 14: Publication 생성
     PG 16: Subscription 생성 → 초기 동기화 → 변경 수신

  ③ PG 16이 PG 14를 따라잡을 때까지 대기
  ④ 짧은 유지보수 창: PG 14 쓰기 중단
  ⑤ PG 16 Promote (Subscription 제거 후 독립 운영)
  ⑥ 애플리케이션을 PG 16으로 전환
  → 다운타임: 수 초 ~ 수 분

  제약:
  PG 14→16: 가능 (higher → lower 안 됨)
  DDL 변경은 양쪽에 수동 적용 필요
  시퀀스는 수동 동기화 필요 (Logical Replication에서 복제 안 됨)
```

### 4. CDC (Change Data Capture) — Debezium 연동

```
CDC 파이프라인:

  PostgreSQL → Debezium → Kafka → Consumer (ES, Redis 등)

  1. PostgreSQL 설정:
     wal_level = logical
     max_replication_slots = 10
     CREATE USER debezium_user REPLICATION LOGIN PASSWORD '...';

  2. Debezium PostgreSQL Connector 설정:
     {
       "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
       "database.hostname": "pg-host",
       "database.port": "5432",
       "database.user": "debezium_user",
       "database.dbname": "mydb",
       "plugin.name": "pgoutput",
       "slot.name": "debezium_slot",
       "table.include.list": "public.orders,public.users",
       "topic.prefix": "mydb"
     }

  3. Kafka Topic으로 변경사항 전달:
     mydb.public.orders → INSERT/UPDATE/DELETE 이벤트

  Debezium이 생성하는 Kafka 메시지 형식:
  {
    "op": "u",                    // u=update, c=create, d=delete, r=read
    "before": {"id": 1, "status": "PENDING"},
    "after": {"id": 1, "status": "SHIPPED"},
    "source": {"table": "orders", "lsn": 12345678}
  }

  CDC 활용 사례:
  ① Elasticsearch 실시간 인덱싱
  ② Redis 캐시 무효화
  ③ 마이크로서비스 간 이벤트 동기화
  ④ 감사 로그(Audit Log) 구축
  ⑤ 데이터 웨어하우스 실시간 적재
```

---

## 💻 실전 실험

### 실험 1: 기본 Logical Replication 설정

```sql
-- Publisher 인스턴스에서:
-- 1. wal_level = logical 확인 (재시작 필요)
SHOW wal_level;  -- 'logical'이어야 함

-- 2. 복제용 사용자 생성
CREATE USER replication_user REPLICATION LOGIN PASSWORD 'strong_password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;

-- 3. Publication 생성
CREATE PUBLICATION pub_orders FOR TABLE orders, users;

-- 4. Publication 목록 확인
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;

-- Subscriber 인스턴스에서:
-- 1. 테이블 미리 생성 (DDL 수동 실행)
CREATE TABLE orders (id BIGSERIAL PRIMARY KEY, user_id INT, amount NUMERIC, status TEXT);
CREATE TABLE users (id BIGSERIAL PRIMARY KEY, name TEXT, email TEXT);

-- 2. Subscription 생성
CREATE SUBSCRIPTION sub_orders
CONNECTION 'host=publisher-host port=5432 dbname=mydb user=replication_user password=strong_password'
PUBLICATION pub_orders;

-- 3. 복제 상태 확인
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_subscription_rel;  -- ready면 정상 복제 중
```

### 실험 2: 행 필터와 컬럼 목록 (PostgreSQL 15+)

```sql
-- Publisher:
-- 행 필터 (active 사용자만)
CREATE PUBLICATION pub_active_users
FOR TABLE users WHERE (active = true);

-- 컬럼 필터 (민감 정보 제외)
CREATE PUBLICATION pub_public_users
FOR TABLE users (id, name, email, created_at);
-- password, social_number 등 제외

-- Subscriber:
CREATE SUBSCRIPTION sub_active_users
CONNECTION '...'
PUBLICATION pub_active_users;

-- 비활성 사용자 변경은 복제 안 됨 확인
-- Publisher: UPDATE users SET active = false WHERE id = 1;
-- Subscriber: SELECT * FROM users WHERE id = 1;  -- 변경됨 (기존 행 업데이트)
-- Publisher: INSERT INTO users (..., active=false) VALUES (...);
-- Subscriber: 이 INSERT는 복제 안 됨 (WHERE 조건 불만족)
```

### 실험 3: Logical Replication Slot 모니터링

```sql
-- Logical Slot 상태 확인
SELECT
    slot_name,
    slot_type,
    active,
    catalog_xmin,
    age(catalog_xmin) AS catalog_xmin_age,
    restart_lsn,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal,
    confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_type = 'logical'
ORDER BY restart_lsn;

-- Subscription 상태
SELECT
    subname,
    subenabled,
    subpublications,
    subslotname
FROM pg_subscription;

-- 각 테이블 복제 상태
SELECT
    s.subname,
    c.relname AS table_name,
    r.srsubstate AS state  -- 'i'=initialize, 'd'=data copy, 'f'=finished, 'r'=ready
FROM pg_subscription_rel r
JOIN pg_subscription s ON s.oid = r.srsubid
JOIN pg_class c ON c.oid = r.srrelid;
```

---

## 📊 MySQL과 비교

```
MySQL Binlog Replication vs PostgreSQL Logical Replication:

MySQL:
  Binary Log 기반 논리 복제 (ROW/STATEMENT/MIXED mode)
  GTID로 복제 위치 추적
  Master → Slave → Slave (연쇄 복제 가능)
  Debezium: MySQL Binlog 커넥터로 CDC

PostgreSQL:
  WAL Logical Decoding
  Publication/Subscription 명시적 모델
  pgoutput (내장) 또는 wal2json (확장)
  Debezium: PostgreSQL 커넥터로 CDC

이기종 복제 비교:
  MySQL: Binlog STATEMENT 모드로 다른 DBMS에 일부 가능 (제한적)
  PostgreSQL: pgoutput/wal2json으로 다른 시스템에 논리 스트리밍
              PG 버전 간 복제: Logical Replication으로 가능

선택적 복제:
  MySQL: 복제 필터(replicate-do-table, replicate-wild-do-table)
  PostgreSQL: Publication FOR TABLE 명시적

CDC 성숙도:
  MySQL + Debezium: 오래되고 안정적
  PostgreSQL + Debezium: 충분히 성숙, pgoutput 권장
```

---

## ⚖️ 트레이드오프

```
Logical Replication 장단점:

장점:
  ① 이기종 버전/OS 복제 가능
  ② 테이블/행/컬럼 단위 선택적 복제
  ③ 메이저 버전 업그레이드를 짧은 다운타임으로
  ④ CDC 파이프라인 구축 (Debezium + Kafka)
  ⑤ 양방향 복제 가능 (Multi-Master, 주의 필요)

단점:
  ① DDL 복제 불가 (수동 관리 필요)
  ② Sequence 복제 불가 (ID 충돌 가능)
  ③ Large Object 복제 불가
  ④ Slot 관리 필요 (WAL 보존 위험)
  ⑤ REPLICA IDENTITY 설정 없는 테이블 복제 제한

Physical Replication이 유리한 경우:
  ① 전체 DB 정확한 복제 (High Availability)
  ② Failover가 목적 (Physical이 더 단순)
  ③ DDL 포함 모든 변경 자동 복제

Logical Replication이 유리한 경우:
  ① 선택적 테이블 복제
  ② 이기종 버전 업그레이드
  ③ CDC/이벤트 스트리밍
  ④ 다른 시스템으로 데이터 동기화
```

---

## 📌 핵심 정리

```
Logical Replication 핵심:

구조:
  Publication (Publisher): 어떤 테이블의 변경을 공개
  Subscription (Subscriber): 어떤 Publication을 구독
  Logical Slot: WAL에서 논리 변경 추출 보존

Physical vs Logical:
  Physical: WAL 그대로 전달, 동일 버전/구조 필요
  Logical: 행 수준 변경 추출, 이기종 가능

wal_level = logical 필수:
  replica보다 상위 → WAL에 컬럼 값 포함

제약:
  DDL 복제 안 됨 (수동 양쪽 적용)
  Sequence 복제 안 됨
  Large Object 복제 안 됨
  REPLICA IDENTITY 설정 필요

CDC 활용:
  pgoutput → Debezium → Kafka → ES/Redis
  변경 이벤트를 실시간으로 다른 시스템에 전달

버전 업그레이드:
  구버전 → Logical Replication → 신버전
  복제 따라잡기 → 짧은 다운타임 전환
```

---

## 🤔 생각해볼 문제

**Q1.** Logical Replication에서 Primary Key가 없는 테이블을 복제하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

Primary Key가 없으면 UPDATE/DELETE WAL에서 어떤 행이 변경됐는지 식별할 수 없어 복제가 실패합니다. 해결 방법:

1. **Primary Key 추가**: 가장 권장하는 방법
```sql
ALTER TABLE table_name ADD PRIMARY KEY (id);
```

2. **UNIQUE NOT NULL 인덱스를 Replica Identity로 설정**:
```sql
CREATE UNIQUE INDEX ON table_name(unique_col);
ALTER TABLE table_name REPLICA IDENTITY USING INDEX table_name_unique_col_idx;
```

3. **REPLICA IDENTITY FULL**: 모든 컬럼을 OLD 값으로 포함
```sql
ALTER TABLE table_name REPLICA IDENTITY FULL;
```
→ WAL 크기 증가, 업데이트/삭제 성능 약간 저하

INSERT만 복제하는 경우에는 Replica Identity 없이 가능하지만, UPDATE/DELETE가 있으면 위 방법 중 하나가 필요합니다.

</details>

---

**Q2.** 같은 테이블에 Publisher와 Subscriber가 동시에 쓰기를 한다면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

이것이 **Multi-Master 시나리오**로 매우 복잡한 충돌 문제를 일으킵니다.

발생 가능한 문제:
1. **PK 충돌**: A 노드에서 id=100 INSERT, B 노드에서 id=100 INSERT → Subscriber에서 중복 키 오류
2. **Update 충돌**: A에서 row 업데이트 → B로 복제 → B도 같은 row 업데이트 → A로 복제 → 루프
3. **DELETE 충돌**: A에서 삭제 → B로 복제 → B에서 같은 row 삭제 시도 → "row not found"

PostgreSQL의 기본 Logical Replication은 충돌 해결 메커니즘이 없으므로 오류가 발생하면 복제가 중단됩니다.

Multi-Master가 필요하다면:
- BDR (Bi-Directional Replication) - EDB 상용 제품
- Citus - 분산 PostgreSQL
- 충돌 없는 데이터 분리 (A노드는 짝수 ID, B노드는 홀수 ID)

단순히 읽기 분산이 목적이라면 Physical Replication의 Hot Standby를 사용하는 것이 더 안전합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Hot Standby](./02-hot-standby.md)** | **[홈으로 🏠](../README.md)** | **[다음: Patroni 고가용성 ➡️](./04-patroni-ha.md)**

</div>
