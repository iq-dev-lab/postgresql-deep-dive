# Spring + PostgreSQL 최적화 — HikariCP, R2DBC, JPA 특화 기능

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HikariCP의 `maximumPoolSize`, `connectionTimeout`, `idleTimeout`은 어떻게 설정해야 하는가?
- R2DBC PostgreSQL 드라이버와 JDBC의 근본적 차이는 무엇인가?
- Spring Data JPA에서 PostgreSQL JSONB 컬럼을 어떻게 쿼리하는가?
- Flyway와 Liquibase 중 어느 것을 선택해야 하는가?
- HikariCP + PgBouncer 조합에서 주의해야 할 설정은?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

Spring Boot와 PostgreSQL을 함께 쓸 때 기본 설정만으로는 최적 성능을 낼 수 없다. HikariCP의 `maximumPoolSize`를 너무 크게 설정하면 PostgreSQL 연결이 폭발하고, `connectionTimeout`이 짧으면 DB 순간 부하에 예외가 발생한다. JPA와 PostgreSQL의 JSONB, 배열 타입을 함께 쓰려면 별도 설정이 필요하다. 마이그레이션 도구 선택도 팀의 워크플로우에 큰 영향을 준다. 이 문서에서 실무에서 바로 적용 가능한 Spring + PostgreSQL 최적화 패턴을 정리한다.

---

## 😱 흔한 실수 (Before — Spring + PostgreSQL 기본 설정 그대로 사용)

```
실수 1: maximumPoolSize 기본값(10) 사용

  10개 스레드가 DB 작업을 기다리는데 11번째 스레드:
  → HikariCP 연결 대기 → connectionTimeout(30s 기본) 초과
  → SQLTimeoutException 발생

  반대로 maximumPoolSize = 100으로 설정:
  Spring 서버 10대 × 100 = 1000개 PostgreSQL 연결
  → max_connections 초과 또는 프로세스 메모리 10GB 낭비

실수 2: N+1 문제 방치 (JPA)

  // 사용자 100명 조회 → 각 사용자마다 orders 별도 조회
  List<User> users = userRepo.findAll();  // 1번 쿼리
  users.forEach(u -> u.getOrders().size()); // 100번 추가 쿼리

  해결: @EntityGraph 또는 fetch join
  @Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id IN :ids")

실수 3: Flyway와 JPA 자동 DDL 동시 사용

  spring.jpa.hibernate.ddl-auto=update + Flyway 동시 사용
  → Flyway가 먼저 마이그레이션 실행
  → Hibernate가 추가로 DDL 자동 실행 → 스키마 불일치

  올바른 설정:
  spring.jpa.hibernate.ddl-auto=validate  # 스키마 검증만
  # Flyway 또는 Liquibase가 DDL 전담
```

---

## ✨ 올바른 접근 (After — Spring + PostgreSQL 최적 설정)

```yaml
# application.yml 권장 설정

spring:
  datasource:
    url: jdbc:postgresql://pgbouncer:5432/mydb?prepareThreshold=0
    username: app_user
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      # 풀 크기: CPU 코어 수 기반
      maximum-pool-size: 10          # (서버당) PgBouncer 사용 시 5~10
      minimum-idle: 2               # 최소 유지 연결
      # 타임아웃
      connection-timeout: 5000      # 5초: 연결 못 얻으면 예외
      idle-timeout: 300000          # 5분: idle 연결 반환
      max-lifetime: 1800000         # 30분: 연결 교체 주기
      keepalive-time: 60000         # 60초: 연결 유지 heartbeat
      # PostgreSQL 최적화
      connection-test-query: SELECT 1  # 연결 검증 (선택)
      # PgBouncer Transaction Mode 호환
      # URL에 prepareThreshold=0 포함

  jpa:
    hibernate:
      ddl-auto: validate            # Flyway/Liquibase 사용 시
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        default_batch_fetch_size: 100  # N+1 최적화
        format_sql: false
    open-in-view: false             # OSIV 비활성화 (권장)

# Flyway 설정
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
```

---

## 🔬 내부 동작 원리

### 1. HikariCP 풀 크기 최적화

```
HikariCP 기본 동작:

  연결 풀 = 미리 생성해 놓은 DB 연결 집합
  요청 시 풀에서 연결 대여 → 쿼리 실행 → 반환
  maximumPoolSize: 최대 보유 가능 연결 수

  연결 요청 흐름:
  스레드 → 풀에서 연결 대여 시도
  연결 있음: 즉시 반환
  연결 없음(모두 사용 중): connectionTimeout 동안 대기
  connectionTimeout 초과: SQLTimeoutException

최적 풀 크기 계산:

  "The Optimal Postgres Connection Pool Size" (Percona):
  pool_size = (core_count × 2) + effective_spindle_count
  예) 4 core CPU, SSD(spindle=1): pool_size = 4×2 + 1 = 9

  실용적 계산:
  ① 동시 활성 DB 작업 스레드 수 파악
  ② 평균 트랜잭션 시간 × 초당 요청 수 = 동시 연결 수
     예) 평균 50ms × 100 req/s = 5개 동시 연결 필요

  PgBouncer 사용 시:
  HikariCP pool_size = 5~10 (PgBouncer가 통합 관리)
  PgBouncer pool_size = 20~50 (PostgreSQL 실제 연결)

연결 유지 설정:

  keepaliveTime = 60000 (60초):
  → idle 연결이 DB 방화벽에 의해 끊기기 전에 keepalive 전송
  → "connection closed" 오류 방지

  maxLifetime = 1800000 (30분):
  → 오래된 연결 주기적 교체 → 연결 상태 초기화
  → DB 재시작 등으로 인한 stale 연결 방지

  idleTimeout = 300000 (5분):
  → minimumIdle 이상의 idle 연결 반환 → 리소스 절약

validationTimeout = 5000 (5초):
  → 연결 사용 전 검증 쿼리 타임아웃 (connectionTestQuery)
  → 너무 짧으면 일시적 DB 부하에 연결 오류

HikariCP 상태 모니터링 (Spring Actuator):
  metrics:
    enable:
      hikaricp: true
  → /actuator/metrics/hikaricp.connections.active
  → /actuator/metrics/hikaricp.connections.pending  # > 0이면 풀 부족
```

### 2. R2DBC — 반응형 PostgreSQL

```
JDBC vs R2DBC 근본 차이:

  JDBC (동기/블로킹):
  스레드 → DB 쿼리 → 스레드 대기 → 결과 수신 → 스레드 재개
  → 쿼리 실행 중 스레드가 블로킹됨
  → 동시 요청 수 = 스레드 수 (HikariCP 풀 크기)

  R2DBC (비동기/논블로킹):
  스레드 → DB 쿼리 시작 → 스레드 해제(다른 작업)
           → DB 응답 도착 → 콜백으로 처리
  → 적은 스레드로 많은 동시 요청 처리 가능

R2DBC PostgreSQL 설정:

  의존성:
  implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
  implementation 'org.postgresql:r2dbc-postgresql'

  application.yml:
  spring:
    r2dbc:
      url: r2dbc:postgresql://localhost:5432/mydb
      username: app_user
      password: ${DB_PASSWORD}
      pool:
        initial-size: 5
        max-size: 20
        max-idle-time: 30m
        validation-query: SELECT 1

  R2DBC Repository:
  @Repository
  public interface OrderRepository extends ReactiveCrudRepository<Order, Long> {
      Flux<Order> findByUserId(Long userId);
      Mono<Long> countByStatus(String status);
  }

  R2DBC + Transaction:
  @Service
  public class OrderService {
      @Transactional  // R2DBC용 트랜잭션 (반응형)
      public Mono<Order> createOrder(OrderDto dto) {
          return orderRepo.save(new Order(dto))
              .flatMap(order -> inventoryRepo.decreaseStock(dto.productId()))
              .flatMap(inv -> Mono.just(inv.getOrder()));
      }
  }

JDBC vs R2DBC 선택:

  JDBC 적합:
  ✓ 기존 Spring MVC 프로젝트 (스레드 모델)
  ✓ JPA 필요 (R2DBC와 JPA 조합 불가)
  ✓ 복잡한 ORM 매핑 필요

  R2DBC 적합:
  ✓ Spring WebFlux + 완전 반응형 스택
  ✓ 대규모 동시 접속 (낮은 스레드로 처리)
  ✓ 스트리밍 데이터 처리

  R2DBC 제약:
  ✗ JPA/Hibernate 사용 불가
  ✗ 일부 PostgreSQL 기능 제한 (Large Object 등)
  ✗ 복잡한 조인/집계 쿼리 작성 어려움
```

### 3. Spring Data JPA + PostgreSQL 특화 기능

```
JSONB 컬럼 매핑:

  방법 1: String으로 저장/조회
  @Entity
  public class Event {
      @Column(columnDefinition = "jsonb")
      private String payload;  // JSON 문자열로 관리
  }

  방법 2: Hibernate Types 라이브러리 (권장)
  의존성:
  implementation 'io.hypersistence:hypersistence-utils-hibernate-63:3.7.0'

  @Entity
  @TypeDef(name = "jsonb", typeClass = JsonBinaryType.class)
  public class Event {
      @Column(columnDefinition = "jsonb")
      @Type(JsonBinaryType.class)
      private Map<String, Object> payload;  // Map으로 사용
  }

  JSONB 쿼리 (네이티브 쿼리 필요):
  @Query(value = "SELECT * FROM events WHERE payload @> :filter::jsonb",
         nativeQuery = true)
  List<Event> findByPayload(@Param("filter") String jsonFilter);

  // 호출
  List<Event> events = eventRepo.findByPayload("{\"type\": \"click\"}");

  JSONB 업데이트:
  @Modifying
  @Query(value = "UPDATE events SET payload = payload || :patch::jsonb WHERE id = :id",
         nativeQuery = true)
  void patchPayload(@Param("id") Long id, @Param("patch") String patch);

배열 컬럼 매핑:

  // Hibernate Types로 배열 지원
  @Entity
  public class Article {
      @Column(columnDefinition = "text[]")
      @Type(StringArrayType.class)
      private String[] tags;
  }

  // 배열 검색 (네이티브 쿼리)
  @Query(value = "SELECT * FROM articles WHERE tags @> ARRAY[:tag]::text[]",
         nativeQuery = true)
  List<Article> findByTag(@Param("tag") String tag);

N+1 문제 해결:

  // 문제 있는 코드
  @ManyToOne(fetch = FetchType.LAZY)
  private User user;
  // user.getOrders() 호출 시마다 SELECT 발생

  // 해결 1: @EntityGraph
  @EntityGraph(attributePaths = {"orders", "orders.items"})
  Optional<User> findWithOrdersById(Long id);

  // 해결 2: JPQL JOIN FETCH
  @Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
  Optional<User> findWithOrders(@Param("id") Long id);

  // 해결 3: default_batch_fetch_size (지연 로딩 일괄)
  spring.jpa.properties.hibernate.default_batch_fetch_size=100
  // 관련 엔티티를 IN절로 묶어 한 번에 조회
```

### 4. Flyway vs Liquibase 마이그레이션

```
Flyway:

  특징:
  ① SQL 파일 기반 (직관적)
  ② 버전 관리: V1__init.sql, V2__add_column.sql
  ③ 체크섬 기반 무결성 검사

  파일 구조:
  src/main/resources/db/migration/
  ├── V1__create_tables.sql
  ├── V2__add_index.sql
  ├── V3__seed_data.sql
  └── R__refresh_view.sql  (반복 실행 가능)

  Spring Boot 자동 설정:
  spring.flyway.enabled=true
  spring.flyway.locations=classpath:db/migration
  spring.flyway.baseline-on-migrate=true  # 기존 DB에 처음 적용 시

  V3__add_column.sql 예시:
  ALTER TABLE orders ADD COLUMN note TEXT;
  CREATE INDEX CONCURRENTLY idx_orders_note ON orders(note);

Liquibase:

  특징:
  ① XML/YAML/JSON/SQL 형식 지원
  ② changeset 단위 관리 (더 세밀한 제어)
  ③ preconditions, rollback 지원

  파일 구조:
  src/main/resources/db/changelog/
  ├── db.changelog-master.yaml
  ├── 2024/
  │   ├── 20240101-create-tables.yaml
  │   └── 20240115-add-index.yaml

  changelog 예시:
  databaseChangeLog:
    - changeSet:
        id: 20240115-add-note-column
        author: developer
        changes:
          - addColumn:
              tableName: orders
              columns:
                - column:
                    name: note
                    type: text
        rollback:
          - dropColumn:
              tableName: orders
              columnName: note

선택 기준:

  Flyway:
  ✓ SQL만 사용, 단순함
  ✓ 팀이 SQL에 익숙
  ✓ 롤백 로직 필요 없음

  Liquibase:
  ✓ 다양한 형식 지원 필요
  ✓ 롤백 시나리오 관리
  ✓ preconditions (조건부 마이그레이션)
  ✓ 데이터베이스 독립적 마이그레이션

  실무 추천:
  단순한 서비스: Flyway (간단, 직관적)
  복잡한 엔터프라이즈: Liquibase (기능 풍부)

공통 주의사항:
  ① CREATE INDEX는 CONCURRENTLY 사용 (운영 중 락 없이)
     단, Flyway/Liquibase는 트랜잭션 내에서 실행
     CONCURRENTLY는 트랜잭션 내 실행 불가!
     → 해결: spring.flyway.mixed=true + 별도 연결

     -- Flyway migration (별도 JDBC 연결 필요)
     CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

  ② DDL 자동 커밋 (PostgreSQL은 DDL도 트랜잭션 가능)
     Flyway: 기본적으로 트랜잭션 내 실행
     CONCURRENTLY 사용 시 주의

  ③ 롤백 불가 DDL 주의
     DROP TABLE, DROP COLUMN 등은 롤백 어려움
```

---

## 💻 실전 실험

### 실험 1: HikariCP 설정 및 모니터링

```java
// HikariCP 상세 설정 (Java Config)
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.hikari")
    public HikariDataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb?prepareThreshold=0");
        ds.setUsername("app_user");
        ds.setPassword(password);

        // 풀 크기
        ds.setMaximumPoolSize(10);
        ds.setMinimumIdle(2);

        // 타임아웃
        ds.setConnectionTimeout(5_000);   // 5초
        ds.setIdleTimeout(300_000);       // 5분
        ds.setMaxLifetime(1_800_000);     // 30분
        ds.setKeepaliveTime(60_000);      // 1분

        // 연결 초기화 (세션 설정)
        ds.setConnectionInitSql(
            "SET application_name = 'myapp'; " +
            "SET statement_timeout = '30s';"
        );

        ds.setPoolName("MyApp-HikariPool");
        return ds;
    }
}
```

```sql
-- PostgreSQL에서 HikariCP 연결 확인
SELECT application_name, count(*) AS connections, state
FROM pg_stat_activity
WHERE application_name = 'myapp'
GROUP BY application_name, state;

-- HikariCP가 설정한 application_name으로 추적 가능
```

### 실험 2: JSONB 쿼리 + Spring Data

```java
// JSONB Repository
@Repository
public interface EventRepository extends JpaRepository<Event, Long> {

    // 네이티브 JSONB 쿼리
    @Query(value = """
        SELECT * FROM events
        WHERE payload @> :filter::jsonb
        ORDER BY created_at DESC
        LIMIT :limit
        """, nativeQuery = true)
    List<Event> findByPayloadContaining(
        @Param("filter") String jsonFilter,
        @Param("limit") int limit
    );

    // JSONB 특정 필드 값 조회
    @Query(value = """
        SELECT * FROM events
        WHERE payload->>'event_type' = :eventType
        AND created_at >= :since
        """, nativeQuery = true)
    List<Event> findByEventType(
        @Param("eventType") String eventType,
        @Param("since") LocalDateTime since
    );

    // JSONB 배열 원소 검색
    @Query(value = """
        SELECT * FROM events
        WHERE payload->'tags' ? :tag
        """, nativeQuery = true)
    List<Event> findByTag(@Param("tag") String tag);
}

// 사용 예시
List<Event> events = eventRepo.findByPayloadContaining(
    """{"event_type": "purchase", "status": "completed"}""",
    100
);
```

### 실험 3: Flyway 마이그레이션 설정

```sql
-- db/migration/V1__create_orders_table.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status TEXT NOT NULL DEFAULT 'PENDING',
    amount NUMERIC(12, 2) NOT NULL,
    payload JSONB,
    tags TEXT[],
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- V2__add_indexes.sql
-- 주의: CONCURRENTLY는 트랜잭션 외부에서만 사용 가능
-- Flyway에서는 @transactional=false로 설정 필요
CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders(user_id);
CREATE INDEX IF NOT EXISTS idx_orders_status ON orders(status) WHERE status != 'DONE';
CREATE INDEX IF NOT EXISTS idx_orders_payload ON orders USING GIN(payload);
```

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    validate-on-migrate: true
    out-of-order: false
    # CONCURRENTLY 사용 시 트랜잭션 비활성화 필요한 경우
    # mixed: true  # 트랜잭션 내/외 마이그레이션 혼용
```

```java
// Flyway 콜백으로 마이그레이션 후 처리
@Component
public class FlywayPostMigrationCallback implements Callback {

    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.AFTER_MIGRATE;
    }

    @Override
    public void handle(Event event, Context context) {
        // 마이그레이션 완료 후 처리
        log.info("Database migration completed");
    }
}
```

---

## 📊 MySQL과 비교

```
Spring + MySQL vs Spring + PostgreSQL:

HikariCP:
  MySQL: 동일하게 사용
  PostgreSQL: prepareThreshold=0 (PgBouncer Transaction Mode 필요 시)

JPA JSONB:
  MySQL: JSON 타입 + JSON_CONTAINS 네이티브 쿼리 필요
  PostgreSQL: JSONB + @> 연산자 + GIN 인덱스 + Hibernate Types

배열:
  MySQL: 배열 타입 없음 → JSON 배열 또는 별도 테이블
  PostgreSQL: 네이티브 배열 타입 + GIN 인덱스 + Spring 지원

마이그레이션:
  MySQL: Flyway/Liquibase 둘 다 지원
  PostgreSQL: 동일하게 지원, CONCURRENTLY 주의

R2DBC:
  MySQL: r2dbc-mysql 드라이버
  PostgreSQL: r2dbc-postgresql 드라이버 (성숙도 높음)

Dialect:
  MySQL: MySQLDialect
  PostgreSQL: PostgreSQLDialect (JSONB, 배열 타입 지원)
```

---

## ⚖️ 트레이드오프

```
HikariCP 풀 크기 트레이드오프:

  너무 작음 (maximumPoolSize = 2):
  → 동시 요청 대기 → connectionTimeout 예외
  → 트래픽 피크 시 서비스 장애

  너무 큼 (maximumPoolSize = 100):
  → PostgreSQL 연결 과다 → 메모리 낭비
  → PgBouncer 없을 때 연결 폭발

  적정값 결정 방법:
  1. HikariCP metrics 모니터링 (pending connections > 0: 증가 필요)
  2. 부하 테스트 중 active/pending connections 관찰
  3. PgBouncer와 조합 시 5~10으로 시작

OSIV (Open Session In View):

  spring.jpa.open-in-view=true (기본):
  → HTTP 요청 전체에서 영속성 컨텍스트 유지
  → View 렌더링 중에도 DB 연결 점유
  → 연결 낭비 + 의도치 않은 쿼리 발생

  spring.jpa.open-in-view=false (권장):
  → 서비스 레이어에서만 영속성 컨텍스트
  → LazyInitializationException 명시적 처리 필요
  → 연결 효율 향상

Flyway vs Liquibase 실용적 선택:
  Flyway: SQL 파일, 단순, 팀이 SQL에 익숙 → 추천
  Liquibase: 롤백 지원 필요, 다양한 형식 → 추천
  동시 사용: 절대 금지 (스키마 충돌)
```

---

## 📌 핵심 정리

```
Spring + PostgreSQL 최적화 핵심:

HikariCP:
  maximumPoolSize: CPU×2 + 1 (PgBouncer 있으면 5~10)
  connectionTimeout: 3000~5000ms
  idleTimeout: 300000ms (5분)
  maxLifetime: 1800000ms (30분)
  keepaliveTime: 60000ms (1분)
  PgBouncer Transaction Mode: prepareThreshold=0 URL에 추가

R2DBC:
  WebFlux + 완전 반응형 스택에서 사용
  JPA 사용 불가, 낮은 스레드로 높은 동시성
  r2dbc:postgresql://... URL 형식

JPA + PostgreSQL 특화:
  JSONB: Hibernate Types + 네이티브 쿼리
  배열: StringArrayType + @> 네이티브 쿼리
  N+1 방지: @EntityGraph, JOIN FETCH, default_batch_fetch_size

마이그레이션:
  ddl-auto=validate (Flyway/Liquibase 사용 시)
  Flyway: SQL 파일, 단순
  Liquibase: 롤백 지원, 형식 다양
  CONCURRENTLY 인덱스: 트랜잭션 외부에서 실행

OSIV:
  open-in-view=false 권장 (연결 효율)
  LazyInitializationException → 명시적 fetch 처리
```

---

## 🤔 생각해볼 문제

**Q1.** HikariCP `maximumPoolSize = 10`인데 `minimumIdle = 2`로 설정하면 idle 연결이 언제 2개로 줄어드는가?

<details>
<summary>해설 보기</summary>

`idleTimeout` 이후에 idle 연결이 `minimumIdle` 수준으로 줄어듭니다.

동작 과정:
1. 트래픽 증가 → HikariCP가 연결을 10개까지 생성
2. 트래픽 감소 → 연결들이 idle 상태로 전환
3. `idleTimeout`(기본 10분, 권장 5분) 경과 후 → idle 연결이 `minimumIdle`(2개)까지 반환
4. 결과: PostgreSQL 연결 2개만 유지 → 리소스 절약

중요한 점:
- `minimumIdle >= 0`: 모든 idle 연결 반환 가능 (cold start 리스크)
- `minimumIdle = maximumPoolSize`: 항상 최대 풀 유지 (예열 상태, 일관된 응답)
- 트래픽 패턴이 예측 불가하면 `minimumIdle = maximumPoolSize / 2` 정도 권장

```yaml
hikari:
  maximum-pool-size: 10
  minimum-idle: 3          # idle 시 최소 3개 유지
  idle-timeout: 300000     # 5분 idle 후 반환 시작
```

</details>

---

**Q2.** Spring Boot에서 Flyway 마이그레이션으로 `CREATE INDEX CONCURRENTLY`를 실행하면 어떤 문제가 발생하고 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

`CREATE INDEX CONCURRENTLY`는 트랜잭션 내부에서 실행할 수 없습니다. Flyway는 기본적으로 각 마이그레이션을 트랜잭션으로 감싸므로 오류가 발생합니다.

```
ERROR: CREATE INDEX CONCURRENTLY cannot run inside a transaction block
```

해결 방법:

**방법 1: 해당 마이그레이션에서 트랜잭션 비활성화 (Flyway)**
```java
@Bean
public FlywayMigrationStrategy flywayMigrationStrategy() {
    return flyway -> {
        // CONCURRENTLY 인덱스는 별도 Connection으로 실행
    };
}
```

**방법 2: `@FlywayMigration` 어노테이션 (Spring Flyway 6+)**
```sql
-- V2__add_concurrent_index.sql
-- flyway.nonTransactional: true (파일 내 주석)
```
일부 Flyway 버전에서는 파일명에 `__non_transactional` 접미사를 지원.

**방법 3: Flyway Java Migration 사용**
```java
@Component
public class V3__ConcurrentIndex extends BaseJavaMigration {

    @Override
    public void migrate(Context context) throws Exception {
        try (Statement stmt = context.getConnection().createStatement()) {
            // 트랜잭션 없이 실행
            context.getConnection().setAutoCommit(true);
            stmt.execute("CREATE INDEX CONCURRENTLY IF NOT EXISTS " +
                        "idx_orders_user_id ON orders(user_id)");
        }
    }

    @Override
    public boolean canExecuteInTransaction() {
        return false;  // 트랜잭션 없이 실행 선언
    }
}
```

**방법 4: `IF NOT EXISTS` + 일반 인덱스로 우선 적용**
운영 초기: `CREATE INDEX` (잠금 있음, 빠름)
운영 후: `CREATE INDEX CONCURRENTLY` 수동 실행

</details>

---

<div align="center">

**[⬅️ 이전: 운영 중 발생하는 문제 패턴](./04-operational-issues.md)** | **[홈으로 🏠](../README.md)**

</div>
