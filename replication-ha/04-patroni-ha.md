# Patroni 고가용성 — Leader Election과 Split-Brain 방지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Patroni는 etcd/Consul로 Leader Election을 어떻게 구현하는가?
- 자동 Failover(장애조치) 절차는 어떤 단계로 진행되는가?
- Split-Brain이란 무엇이고, Patroni는 이를 어떻게 방지하는가?
- Spring 애플리케이션에서 Failover 후 자동으로 연결이 전환되는 구조는?
- PgBouncer + Patroni 조합에서 VIP(Virtual IP)가 필요한 이유는?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

Streaming Replication으로 Standby를 구성해도 Primary가 다운되면 수동으로 Standby를 Promote해야 한다. 이 과정에 수 분이 걸리면 서비스가 중단된다. Patroni는 자동 Failover를 제공하며, etcd/Consul 같은 분산 합의 시스템으로 Split-Brain(두 노드가 동시에 Primary라고 생각하는 상황)을 방지한다. Spring 서비스와 연동 시 PgBouncer를 통해 Failover 후 자동으로 새 Primary로 트래픽이 전환된다.

---

## 😱 흔한 실수 (Before — 수동 Failover 운영)

```
수동 Failover의 문제:

  새벽 3시 Primary DB 서버 다운
  온콜 DBA 알림 수신 → 15분 후 응답
  상황 파악 → 5분
  Standby Promote: pg_ctl promote -D /data
  → 2분
  애플리케이션 DB 연결 변경 (설정 파일 수정 + 재배포)
  → 10분
  총 다운타임: 약 30분

Patroni 없는 수동 운영의 문제:
  ① 야간/주말 장애 시 대응 지연
  ② 사람이 개입하므로 실수 가능
  ③ Split-Brain 위험 (Primary 복구 후 두 Primary 동시 존재)
  ④ 애플리케이션이 새 Primary를 자동으로 모름

Split-Brain 시나리오 (방치 시):
  Primary가 네트워크 단절 (not actually down)
  Patroni 없이 Standby를 수동 Promote
  → 두 서버가 모두 Primary로 동작
  → 두 곳에 다른 데이터 쓰기 → 데이터 불일치
  → 복구 불가능한 상태
```

---

## ✨ 올바른 접근 (After — Patroni HA 구성)

```
Patroni 기본 아키텍처:

  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  etcd-1  │    │  etcd-2  │    │  etcd-3  │  ← 분산 합의 클러스터
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │               │
  ┌────▼──────────────────────────────▼─────┐
  │            etcd 클러스터                  │
  │  /service/pg-cluster/leader = "node1"   │ ← Leader 정보 저장
  └────────────┬──────────────┬─────────────┘
               │              │
  ┌────────────▼──┐  ┌────────▼──────────┐
  │  pg-node1     │  │  pg-node2         │
  │  Patroni +    │  │  Patroni +        │
  │  PostgreSQL   │  │  PostgreSQL       │
  │  (Primary)    │  │  (Standby)        │
  └───────────────┘  └───────────────────┘
         │
  ┌──────▼─────┐
  │  PgBouncer │ ← 항상 현재 Primary로 연결
  └──────┬─────┘
         │
  ┌──────▼─────┐
  │  Application│
  └────────────┘

Patroni 설정 (patroni.yml 예시):
  scope: pg-cluster
  namespace: /service/
  name: node1

  etcd:
    hosts: etcd-1:2379,etcd-2:2379,etcd-3:2379

  postgresql:
    listen: 0.0.0.0:5432
    connect_address: pg-node1:5432
    data_dir: /data/postgresql
    parameters:
      max_connections: 200
      shared_buffers: 4GB
      wal_level: replica
      hot_standby: on
      max_wal_senders: 10
      max_replication_slots: 10
```

---

## 🔬 내부 동작 원리

### 1. Leader Election with etcd

```
Patroni Leader Election 메커니즘:

  1. 초기 클러스터 시작:
     각 노드가 etcd에 /service/pg-cluster/leader 키에
     자신을 Leader로 등록 시도

  2. etcd 분산 잠금 (Compare-And-Swap):
     첫 번째 노드만 성공 → Leader (Primary)
     나머지 → Follower (Standby)

  3. Leader Lease (TTL):
     Leader는 etcd에 주기적으로 Heartbeat (기본 10초)
     TTL(기본 30초) 내 Heartbeat 없으면 → Lease 만료

  4. Leader 장애 감지:
     etcd: /leader 키 TTL 만료 → 키 삭제
     Follower: etcd 폴링으로 키 사라짐 감지
     → 새 Leader Election 시작

  etcd의 역할:
  ① 단일 진실 원천: "현재 Leader는 누구인가"
  ② 분산 잠금: 한 번에 하나의 Leader만 보장
  ③ Lease: Leader 활성 여부 자동 감지

  DCS(Distributed Configuration Store) 옵션:
  etcd:   권장 (간단, 안정적)
  Consul: etcd 대안
  ZooKeeper: 레거시
  Kubernetes API: 쿠버네티스 환경에서 etcd 대신
```

### 2. 자동 Failover 절차

```
Failover 상세 단계:

  T+0: Primary(node1) 다운

  T+0~30초: Patroni Lease 만료 대기
    node2의 Patroni: etcd의 /leader 키 폴링
    TTL(30초) 후 키 사라짐 감지

  T+30초: Candidate 선출 시작
    node2: etcd에 새 Leader 등록 시도
    (다른 후보가 있으면 race condition → 가장 빠른 노드 승리)

  Leader 후보 자격 확인:
    node2의 WAL이 충분히 최신인가?
    → pg_stat_replication.replay_lsn과 비교
    → 가장 최신 WAL을 가진 노드가 Leader 후보

  T+30~35초: Fencing (기존 Primary 격리)
    pg_rewind로 기존 Primary가 다시 Primary로 돌아오지 못하도록
    WAL 파일 삭제 또는 watchdog으로 서버 재시작 강제
    (Split-Brain 방지)

  T+35초: Promote 실행
    node2: pg_ctl promote → PostgreSQL Standby → Primary 전환
    etcd: /leader = "node2" 등록

  T+35~40초: PgBouncer 전환
    PgBouncer: patroni REST API 폴링으로 새 Primary 감지
    또는 HAProxy health check: Patroni API /primary (200 OK)
    → 새 Primary(node2:5432)로 연결 전환

  T+40초: 정상 운영 재개
    총 다운타임: 약 30~40초 (설정에 따라 10~20초 가능)

  원래 node1 복구 시:
    Patroni: pg_rewind로 node1을 node2의 Standby로 재구성
    → 자동으로 Standby로 참여
```

### 3. Split-Brain 방지 — Fencing

```
Split-Brain 시나리오:

  node1 (Primary): 네트워크 파티션 → etcd 연결 불가
  node2 (Standby): etcd에서 node1 Lease 만료 감지 → Leader 승격 시도

  위험:
  node1: "나는 아직 Primary야, etcd와 연결이 안 될 뿐"
  node2: "나는 새 Primary"
  → 두 노드 모두 Primary로 동작 → 데이터 불일치

Patroni Fencing 메커니즘:

  방법 1: Watchdog
    Patroni 에이전트가 etcd와 연결 끊기면:
    → 자신의 PostgreSQL을 즉시 중단 (pg_ctl stop)
    → Split-Brain 불가

    watchdog.py 설정:
    watchdog:
      mode: required  # etcd 연결 끊기면 PostgreSQL 강제 중단
      device: /dev/watchdog

  방법 2: pg_rewind + PGDATA 삭제
    Failover 후 신규 Primary가 구 Primary에 SSH → WAL 삭제
    → 구 Primary가 재시작해도 Standby로만 동작 가능

  방법 3: STONITH (Shoot The Other Node In The Head)
    클라우드 환경: 구 Primary 인스턴스를 API로 재시작
    → 인스턴스 재시작 → Patroni 재시작 → Standby로 합류

  etcd Quorum:
  etcd 자체도 분산 합의 (Raft) 사용
  3개 노드 etcd에서 2개가 살아있어야 쓰기 가능
  → etcd 클러스터 장애 시 Patroni는 Read-Only 모드로 전환 (안전)
```

### 4. PgBouncer + Patroni 연동

```
PgBouncer가 Primary를 자동 감지하는 방법:

  방법 1: Patroni REST API
    PgBouncer: 주기적으로 Patroni REST API 호출
    GET http://pg-node1:8008/primary → 200 OK (Primary)
    GET http://pg-node1:8008/primary → 503 (Not Primary)

    PgBouncer 설정:
    [databases]
    mydb = host=pg-node1 port=5432  # 정적 설정

    → Failover 후 PgBouncer가 503 감지 → 연결 실패 → 재시도 로직 필요

  방법 2: HAProxy + Patroni API
    HAProxy: 모든 PostgreSQL 노드를 백엔드로 설정
    Health Check:
      httpchk GET /primary HTTP/1.1 → 200이면 Primary
    → Primary만 활성 백엔드로 유지
    → Failover 자동 반영

    haproxy.cfg:
    frontend postgres
      bind *:5432
      default_backend postgres_primary

    backend postgres_primary
      option httpchk GET /primary HTTP/1.1
      server node1 pg-node1:5432 check port 8008
      server node2 pg-node2:5432 check port 8008

  방법 3: VIP (Virtual IP)
    Keepalived로 VIP를 현재 Primary에 할당
    Failover 시 VIP가 새 Primary로 이동
    → 애플리케이션은 항상 VIP로 연결 (DB 주소 불변)

Spring 연결 전환:
  HikariCP의 connectionTimeout + 재연결 설정
  JDBC URL에 여러 호스트:
  jdbc:postgresql://pg-node1:5432,pg-node2:5432/mydb?targetServerType=primary
  → PostgreSQL JDBC 드라이버가 Primary 자동 탐색
```

---

## 💻 실전 실험

### 실험 1: Patroni REST API 확인

```bash
# Patroni 상태 확인
curl http://pg-node1:8008/
# 응답: {"state": "running", "role": "master", ...}

# Primary 확인 (HAProxy health check용)
curl http://pg-node1:8008/primary
# Primary이면: HTTP 200
# Standby이면: HTTP 503

# Replica 확인
curl http://pg-node2:8008/replica
# Standby이면: HTTP 200

# 클러스터 전체 상태
curl http://pg-node1:8008/cluster
# {
#   "members": [
#     {"name": "node1", "role": "Leader", "state": "running", "lag_in_mb": 0},
#     {"name": "node2", "role": "Replica", "state": "streaming", "lag_in_mb": 0}
#   ]
# }
```

### 실험 2: 수동 Switchover (계획된 전환)

```bash
# patronictl로 안전한 Switchover (다운타임 최소화)
patronictl -c /etc/patroni/patroni.yml switchover pg-cluster \
  --master node1 --candidate node2 --force

# Switchover 모니터링
patronictl -c /etc/patroni/patroni.yml list
# + Cluster: pg-cluster --------+---------+-----------+
# | Member | Host        | Role    | State   | TL | Lag in MB |
# +--------+-------------+---------+---------+----+-----------+
# | node1  | pg-node1:5432 | Replica | running |  2 |         0 |
# | node2  | pg-node2:5432 | Leader  | running |  2 |           |
```

### 실험 3: Patroni 상태 SQL 조회

```sql
-- Primary에서 (Patroni가 관리하는 경우)
-- Patroni가 추가하는 설정 확인
SHOW hot_standby;
SHOW wal_level;

-- Failover 후 복제 상태 확인
SELECT application_name, state, sync_state, replay_lag
FROM pg_stat_replication;

-- 현재 타임라인 (Failover 시 증가)
SELECT timeline_id FROM pg_control_checkpoint();
-- Failover 전: 1, 후: 2

-- Standby에서 (새 Primary 연결 확인)
SELECT pg_is_in_recovery();  -- false: Primary로 Promote 완료
SELECT * FROM pg_stat_wal_receiver;  -- 연결된 Primary 정보
```

---

## 📊 MySQL과 비교

```
MySQL HA vs PostgreSQL + Patroni:

MySQL Group Replication / InnoDB Cluster:
  내장 HA: MySQL 자체가 Paxos 기반 Multi-Master 지원
  자동 Failover: MySQL Router로 연결 전환
  설정: MySQL Shell로 간단
  Split-Brain: Quorum으로 자동 방지
  쓰기 가능 노드 수: 여러 노드 동시 가능 (Multi-Primary)

PostgreSQL + Patroni:
  외부 도구 필요 (Patroni + etcd)
  단일 Primary 원칙 (Multi-Master 없음)
  자동 Failover: Patroni + HAProxy/PgBouncer
  Split-Brain 방지: etcd Lease + Fencing
  복잡도: MySQL 내장 HA보다 높음

MySQL HA 강점:
  내장 기능으로 비교적 쉬운 설정
  Multi-Primary로 쓰기 확장 가능

PostgreSQL + Patroni 강점:
  유연한 구성 (etcd/Consul/ZooKeeper/K8s)
  강력한 Fencing 메커니즘
  오픈소스 생태계 성숙
  Kubernetes 환경에서 operator (postgres-operator, CloudNativePG)
```

---

## ⚖️ 트레이드오프

```
Patroni 도입 트레이드오프:

장점:
  ① 자동 Failover (다운타임 30~40초)
  ② Split-Brain 안전하게 방지
  ③ 계획된 Switchover (다운타임 < 5초)
  ④ etcd/Consul로 클러스터 상태 중앙 관리
  ⑤ 구성 변경 자동 전파 (patronictl edit-config)

단점:
  ① 추가 구성 요소 (etcd 클러스터 관리 필요)
  ② etcd 자체도 3+ 노드 필요 (고가용성)
  ③ Patroni 버전 관리 및 업데이트
  ④ 네트워크 파티션 시 보수적 동작 (Read-Only 전환)

대안:
  Patroni + etcd: 범용 (베어메탈, VM)
  postgres-operator (Kubernetes): K8s 환경
  CloudNativePG: K8s 네이티브
  AWS RDS Multi-AZ: 클라우드 관리형
  Google Cloud SQL HA: 클라우드 관리형

Failover 허용 다운타임 기준:
  < 10초: 동기 복제 + Watchdog
  < 30초: 비동기 + 기본 Patroni 설정
  < 60초: 단순 Patroni 설정
  > 60초: 수동 운영 가능
```

---

## 📌 핵심 정리

```
Patroni HA 핵심:

구성 요소:
  Patroni: 각 PostgreSQL 노드의 에이전트
  etcd: 분산 합의 저장소 (Leader 정보)
  PgBouncer/HAProxy: 연결 전환

Leader Election:
  etcd의 Compare-And-Swap으로 단일 Leader 보장
  TTL Lease: Leader가 Heartbeat 중단 시 자동 만료

자동 Failover:
  Lease 만료 감지 → 새 Leader 선출 → Promote → PgBouncer 전환
  총 다운타임: 30~40초 (조정 가능)

Split-Brain 방지 (Fencing):
  Watchdog: etcd 연결 끊기면 PostgreSQL 즉시 중단
  pg_rewind: 구 Primary가 재시작해도 Standby로만
  STONITH: 구 Primary 강제 재시작

PgBouncer 연동:
  Patroni REST API /primary 폴링 (HAProxy)
  또는 VIP (Virtual IP) 방식
  Spring JDBC: targetServerType=primary로 자동 탐색
```

---

## 🤔 생각해볼 문제

**Q1.** etcd 클러스터 전체가 다운되면 Patroni 클러스터는 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

Patroni는 etcd를 단일 진실 원천(Source of Truth)으로 사용하므로, etcd 클러스터가 전체 다운되면:

1. **현재 Primary**: etcd에 Heartbeat를 보낼 수 없음. Patroni는 설정에 따라:
   - `ttl` 만료 후 스스로 읽기 전용으로 전환하거나
   - Watchdog 모드에서 PostgreSQL을 강제 중단

2. **Standby 노드**: etcd에서 Leader 정보를 읽을 수 없어 Promote 시도를 하지 않습니다.

결과: **클러스터가 Read-Only 또는 완전 중단** 상태가 됩니다.

이것은 의도된 동작입니다. etcd 없이 계속 Primary로 동작하면 Split-Brain이 발생할 수 있기 때문입니다. PostgreSQL의 ACID 보장을 위해 etcd 의존성을 받아들이는 설계입니다.

따라서 **etcd 클러스터도 반드시 3노드 이상의 고가용성 구성**이 필요합니다.

</details>

---

**Q2.** Patroni를 Kubernetes에서 사용할 때 etcd 대신 Kubernetes API를 DCS로 사용할 수 있는가?

<details>
<summary>해설 보기</summary>

네, 가능합니다. Patroni는 DCS(Distributed Configuration Store)로 Kubernetes API를 지원합니다. Kubernetes의 ConfigMap 또는 Endpoints/Service 오브젝트를 분산 잠금에 사용합니다.

설정:
```yaml
kubernetes:
  namespace: postgres
  labels:
    cluster-name: pg-cluster
```

장점:
- etcd를 별도로 관리할 필요 없음 (Kubernetes etcd 활용)
- Kubernetes operator와 자연스러운 통합

단점:
- Kubernetes 클러스터 자체가 의존성
- Kubernetes API 부하 증가

**CloudNativePG**나 **postgres-operator** 같은 Kubernetes 전용 PostgreSQL 오퍼레이터는 이 방식을 내장으로 사용합니다.

```bash
# CloudNativePG 설치 예시
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.21.0.yaml

# PostgreSQL 클러스터 생성
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-pg-cluster
spec:
  instances: 3
  storage:
    size: 10Gi
EOF
```

</details>

---

<div align="center">

**[⬅️ 이전: 논리 복제](./03-logical-replication.md)** | **[홈으로 🏠](../README.md)** | **[다음: Connection Pooling 심화 ➡️](./05-connection-pooling.md)**

</div>
