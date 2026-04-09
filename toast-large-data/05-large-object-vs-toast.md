# Large Object vs TOAST — 파일 저장 전략 선택 기준

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Large Object(`lo_*`)는 TOAST와 어떻게 다른가? 언제 Large Object가 필요한가?
- `pg_largeobject` 테이블은 내부적으로 어떻게 구성되어 있는가?
- 파일을 데이터베이스에 저장하는 것과 파일 시스템에 저장하는 것의 트레이드오프는?
- Spring에서 Large Object에 접근하는 방법은?
- BYTEA 컬럼 vs Large Object vs 파일 시스템 중 어느 것을 선택해야 하는가?

---

## 🔍 왜 이 개념이 PostgreSQL에서 중요한가

이미지, PDF, 동영상 같은 대용량 바이너리 파일을 데이터베이스에 저장할 때 세 가지 선택지가 있다: TOAST 기반의 BYTEA 컬럼, PostgreSQL Large Object, 그리고 파일 시스템. 각각의 한계와 적합한 사용 사례를 이해하지 못하면 잘못된 선택으로 성능, 복잡성, 운영 문제를 겪는다. 특히 1GB 이상의 파일을 BYTEA로 저장하면 TOAST 청크 수가 폭발적으로 늘어나고, 부분 읽기가 불가능해서 전체 파일을 매번 메모리에 올려야 하는 심각한 문제가 생긴다.

---

## 😱 흔한 실수 (Before — 잘못된 파일 저장)

```
실수 1: 대용량 파일을 BYTEA 컬럼에 저장

  CREATE TABLE files (
      id SERIAL PRIMARY KEY,
      filename TEXT,
      content BYTEA  -- 100MB PDF 저장
  );

  INSERT INTO files (filename, content)
  VALUES ('document.pdf', pg_read_binary_file('/tmp/document.pdf'));

  문제:
  100MB BYTEA → TOAST 테이블에 약 5만 개 청크
  SELECT content FROM files WHERE id = 1:
  → 5만 개 청크 읽기 + 재조합 + 메모리 100MB 로드 필요
  → 메모리 폭발, 느린 응답
  → 스트리밍 불가 (전체 로드 후 전송)

실수 2: 트랜잭션 없이 Large Object 생성

  -- Large Object는 트랜잭션 내에서만 안전
  OID loid = conn.getLargeObjectAPI().createLO();
  LargeObject lo = conn.getLargeObjectAPI().open(loid, LargeObjectManager.WRITE);
  lo.write(bytes);
  lo.close();
  conn.commit();
  -- 하지만 INSERT ... VALUES (loid) 없이 commit 시:
  -- → OID가 pg_largeobject에 있지만 참조 테이블에 없음
  -- → 고아(orphan) Large Object 발생 → vacuumlo 필요

실수 3: Large Object를 BYTEA처럼 SQL로 접근

  SELECT lo_get(oid_col) FROM files;  -- 전체 로드
  vs
  SELECT lo_get(oid_col, 0, 1024) FROM files;  -- 첫 1KB만
  → Large Object의 핵심 장점인 스트리밍/부분 읽기를 활용 안 함
```

---

## ✨ 올바른 접근 (After — 파일 크기별 저장 전략)

```
파일 크기 및 사용 패턴별 전략:

  1KB ~ 1MB (작은 바이너리, 항상 전체 로드):
    → BYTEA 컬럼 (TOAST 자동 처리)
    → 단순, SQL 친화적
    예: 아이콘, 썸네일, 서명 이미지

  1MB ~ 500MB (중형 파일, 부분 읽기 필요):
    → Large Object (스트리밍 가능, 트랜잭션 지원)
    예: PDF, 문서, 중간 크기 이미지

  500MB 이상 (대용량 파일):
    → 파일 시스템 + S3/GCS 등 오브젝트 스토리지
    → DB에는 파일 경로/URL만 저장
    예: 동영상, 대용량 데이터셋

  트랜잭션 일관성 필요?
    → BYTEA 또는 Large Object (ACID 보장)
    파일 시스템: 트랜잭션 없음 (DB-파일 불일치 가능)

  결론:
  대부분의 경우: 파일 시스템 + DB에 경로 저장
  소형 바이너리: BYTEA
  중형 + 부분 읽기 + 트랜잭션: Large Object
```

---

## 🔬 내부 동작 원리

### 1. Large Object vs BYTEA 비교

```
BYTEA (TOAST 방식):

  저장: pg_toast.pg_toast_XXXXX 테이블에 최대 ~2KB 청크로 분할
  접근: 전체 청크를 순서대로 읽어 재조합 → 메모리에 완전한 값
  부분 읽기: 불가 (전체 decompress 후 substring)
  최대 크기: 1GB (실제로는 훨씬 작게 유지 권장)
  트랜잭션: MVCC 완전 지원
  SQL 접근: SELECT content FROM files WHERE id = 1

Large Object:

  저장: pg_largeobject 시스템 테이블에 2KB 페이지로 분할
        pg_largeobject_metadata: OID, 소유자 정보
  접근: lo_open() → lseek() → read() 스트리밍 방식
  부분 읽기: 가능! lo_read(loid, offset, nbytes)
  최대 크기: 4TB
  트랜잭션: 지원 (단, 명시적 트랜잭션 필요)
  SQL 접근: lo_get(loid), lo_get(loid, offset, nbytes)

pg_largeobject 구조:
  CREATE TABLE pg_largeobject (
      loid OID NOT NULL,   -- Large Object 고유 ID
      pageno INT4,         -- 페이지 번호 (0부터)
      data BYTEA           -- 최대 LOBLKSIZE(2KB) 바이트
  );

  1MB Large Object:
  → pageno 0~511까지 512 행 (각 2KB)
  → pg_largeobject에 512행

  접근 방법:
  SELECT data FROM pg_largeobject
  WHERE loid = 12345 AND pageno BETWEEN 0 AND 9;
  → 첫 20KB만 읽기 (나머지 읽지 않음)
```

### 2. Large Object SQL 함수

```
Large Object 함수 (PostgreSQL 9.4+):

생성:
  lo_create(loid OID) → OID  -- 특정 OID로 생성
  lo_creat(mode INT) → OID   -- 자동 OID 생성

쓰기:
  lo_open(loid OID, mode INT) → fd
  lo_write(fd INT, buf BYTEA) → INT  -- 쓴 바이트 수
  lo_close(fd INT) → INT

읽기:
  lo_open(loid OID, INV_READ) → fd
  lo_read(fd INT, len INT) → BYTEA
  -- 또는 단순 버전:
  lo_get(loid OID) → BYTEA                    -- 전체
  lo_get(loid OID, offset INT, len INT) → BYTEA -- 부분

탐색:
  lo_lseek(fd INT, offset INT, whence INT) → INT
  lo_tell(fd INT) → INT   -- 현재 위치
  lo_truncate(fd INT, len INT) → INT

정보:
  lo_get(loid OID) → BYTEA  -- 전체 읽기
  pg_lo_size(loid OID) → BIGINT  -- 크기 확인 (PostgreSQL 16+)

삭제:
  lo_unlink(loid OID) → INT

모드 상수 (lo_open의 두 번째 인자):
  INV_READ = 262144
  INV_WRITE = 131072
  INV_READ | INV_WRITE = 393216  -- 읽기+쓰기

실용적 패턴:
  -- 파일 저장 (단순):
  SELECT lo_from_bytea(0, pg_read_binary_file('/tmp/file.pdf')) AS loid;

  -- 파일 읽기 (전체):
  SELECT lo_get(loid) FROM files WHERE id = 1;

  -- 파일 읽기 (부분 - 스트리밍 유사):
  SELECT lo_get(loid, 0, 65536) FROM files WHERE id = 1;  -- 첫 64KB
```

### 3. Large Object 트랜잭션 관리

```
Large Object와 트랜잭션:

올바른 사용 패턴:
  BEGIN;
    -- Large Object 생성
    SELECT lo_create(0) INTO loid;

    -- 데이터 쓰기
    SELECT lo_open(loid, 393216) INTO fd;
    SELECT lo_write(fd, E'\\x...'::BYTEA);
    SELECT lo_close(fd);

    -- 참조 테이블에 OID 저장
    INSERT INTO files (name, lo_oid) VALUES ('doc.pdf', loid);
  COMMIT;

  → 둘 다 성공하거나 둘 다 롤백 (원자적)

고아(Orphan) Large Object 문제:
  Large Object를 생성했지만 참조 테이블 INSERT가 실패하면:
  → pg_largeobject에는 데이터 있음
  → 참조 테이블에는 없음
  → "고아" Large Object

  정리 방법:
  -- 참조 없는 Large Object 찾기
  SELECT loid FROM pg_largeobject_metadata
  WHERE loid NOT IN (SELECT lo_oid FROM files);

  -- vacuumlo 유틸리티 사용
  vacuumlo mydb
  → 참조 없는 Large Object 자동 삭제

VACUUM과 Large Object:
  VACUUM은 pg_largeobject를 직접 관리하지 않음
  → vacuumlo 별도 실행 필요 (또는 정기 스케줄)
  → 트랜잭션 롤백으로 생성된 LO는 자동 삭제됨
```

### 4. 파일 시스템 vs DB 저장 상세 비교

```
파일 시스템 저장 (S3/GCS 포함):

구현:
  DB: files(id, name, size, s3_key TEXT, uploaded_at TIMESTAMPTZ)
  파일 자체: S3/GCS에 저장
  → Java: S3Client.putObject(), getObject()

장점:
  ① 무제한 크기 (S3: 단일 파일 최대 5TB)
  ② CDN 연동 용이
  ③ 스트리밍 기본 지원 (Range 요청)
  ④ DB 부하 분산 (파일 I/O가 DB를 거치지 않음)
  ⑤ 비용: 파일 스토리지가 DB보다 저렴

단점:
  ① 트랜잭션 없음 (DB INSERT 성공 + S3 업로드 실패 = 불일치)
  ② 두 시스템 관리 (S3 접근 키, IAM 등)
  ③ 파일 삭제 처리 복잡 (참조 정리 필요)

DB 저장 (BYTEA/LO):

장점:
  ① ACID 보장: 파일과 메타데이터 원자적 처리
  ② 단일 백업: pg_dump 하나로 파일도 포함
  ③ 접근 제어: PostgreSQL 권한 시스템 활용

단점:
  ① DB 크기 급증 → VACUUM 부하 증가
  ② 파일 제공 시 DB를 거쳐야 함 (CDN 불가)
  ③ 대용량 파일 메모리 부담 (BYTEA)
  ④ 스케일아웃 어려움

선택 가이드:
  작은 파일 (< 1MB) + 트랜잭션 일관성 중요 → BYTEA
  중형 파일 + 스트리밍 + 트랜잭션 → Large Object
  대용량 파일 또는 파일 수 많음 → S3/파일 시스템
  나중에 확장 예정 → 처음부터 S3 고려
```

### 5. Spring에서 Large Object 접근

```java
// Spring JDBC + Large Object

// 1. JDBC 직접 사용 (LargeObjectManager)
@Transactional  // 반드시 트랜잭션 필요!
public Long saveLargeObject(byte[] data) throws Exception {
    return jdbcTemplate.execute((Connection conn) -> {
        // PostgreSQL 연결로 캐스팅 필요
        PGConnection pgConn = conn.unwrap(PGConnection.class);
        LargeObjectManager lom = pgConn.getLargeObjectAPI();

        // LO 생성
        long loid = lom.createLO(LargeObjectManager.WRITE);
        LargeObject lo = lom.open(loid, LargeObjectManager.WRITE);
        lo.write(data);
        lo.close();
        return loid;
    });
}

@Transactional(readOnly = true)
public byte[] readLargeObject(long loid) throws Exception {
    return jdbcTemplate.execute((Connection conn) -> {
        PGConnection pgConn = conn.unwrap(PGConnection.class);
        LargeObjectManager lom = pgConn.getLargeObjectAPI();

        LargeObject lo = lom.open(loid, LargeObjectManager.READ);
        byte[] data = lo.read(lo.size());
        lo.close();
        return data;
    });
}

// 2. 부분 읽기 (스트리밍)
@Transactional(readOnly = true)
public void streamLargeObject(long loid, OutputStream out) throws Exception {
    jdbcTemplate.execute((Connection conn) -> {
        PGConnection pgConn = conn.unwrap(PGConnection.class);
        LargeObjectManager lom = pgConn.getLargeObjectAPI();
        LargeObject lo = lom.open(loid, LargeObjectManager.READ);

        byte[] buffer = new byte[65536];  // 64KB 버퍼
        int bytesRead;
        while ((bytesRead = lo.read(buffer)) > 0) {
            out.write(buffer, 0, bytesRead);
        }
        lo.close();
        return null;
    });
}

// 3. SQL 함수 활용 (단순한 경우)
// lo_get(loid) 전체, lo_get(loid, offset, len) 부분
byte[] firstChunk = jdbcTemplate.queryForObject(
    "SELECT lo_get(lo_oid, 0, 65536) FROM files WHERE id = ?",
    byte[].class, fileId
);
```

---

## 💻 실전 실험

### 실험 1: BYTEA vs Large Object 크기 비교

```sql
-- BYTEA 테이블
CREATE TABLE bytea_files (
    id SERIAL PRIMARY KEY,
    filename TEXT,
    content BYTEA
);

-- LO 참조 테이블
CREATE TABLE lo_files (
    id SERIAL PRIMARY KEY,
    filename TEXT,
    lo_oid OID REFERENCES pg_largeobject_metadata
);

-- 1MB 데이터 삽입 (BYTEA)
INSERT INTO bytea_files (filename, content)
VALUES ('test.bin', repeat('x', 1048576)::BYTEA);

-- 1MB Large Object 생성
DO $$
DECLARE
    loid OID;
    fd INT;
BEGIN
    loid := lo_create(0);
    fd := lo_open(loid, 393216);  -- INV_READ | INV_WRITE
    PERFORM lo_write(fd, repeat('x', 1048576)::BYTEA);
    PERFORM lo_close(fd);
    INSERT INTO lo_files (filename, lo_oid) VALUES ('test.bin', loid);
END $$;

-- TOAST 테이블 크기 확인 (BYTEA)
SELECT
    c.relname,
    t.relname AS toast_table,
    pg_size_pretty(pg_relation_size(t.oid)) AS toast_size
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
WHERE c.relname = 'bytea_files';

-- pg_largeobject 크기 확인
SELECT pg_size_pretty(pg_total_relation_size('pg_largeobject'));
```

### 실험 2: Large Object 부분 읽기

```sql
-- Large Object OID 확인
SELECT lo_oid FROM lo_files WHERE id = 1;

-- 전체 읽기
SELECT length(lo_get(lo_oid)) AS total_size FROM lo_files WHERE id = 1;

-- 부분 읽기 (처음 1KB)
SELECT length(lo_get(lo_oid, 0, 1024)) AS first_1kb FROM lo_files WHERE id = 1;

-- EXPLAIN: BYTEA vs LO 부분 읽기 비용 비교
-- BYTEA (전체 decompress 필요)
EXPLAIN (ANALYZE, BUFFERS)
SELECT substring(content FROM 1 FOR 1024) FROM bytea_files WHERE id = 1;

-- LO (해당 페이지만 읽기)
EXPLAIN (ANALYZE, BUFFERS)
SELECT lo_get(lo_oid, 0, 1024) FROM lo_files WHERE id = 1;
```

### 실험 3: 고아 LO 찾기 및 정리

```sql
-- 고아 Large Object 확인 (참조 없는 LO)
SELECT l.loid
FROM pg_largeobject_metadata l
LEFT JOIN lo_files f ON f.lo_oid = l.loid
WHERE f.lo_oid IS NULL;

-- 정리 (고아 LO 삭제)
DO $$
DECLARE
    orphan_loid OID;
BEGIN
    FOR orphan_loid IN
        SELECT l.loid
        FROM pg_largeobject_metadata l
        LEFT JOIN lo_files f ON f.lo_oid = l.loid
        WHERE f.lo_oid IS NULL
    LOOP
        PERFORM lo_unlink(orphan_loid);
    END LOOP;
END $$;

-- 또는 vacuumlo 유틸리티 (외부에서 실행)
-- vacuumlo -v mydb
```

---

## 📊 MySQL과 비교

```
MySQL BLOB vs PostgreSQL 파일 저장:

MySQL:
  TINYBLOB:   최대 255 bytes
  BLOB:       최대 65,535 bytes (64KB)
  MEDIUMBLOB: 최대 16,777,215 bytes (16MB)
  LONGBLOB:   최대 4,294,967,295 bytes (4GB)

  저장: InnoDB 페이지 내 또는 외부 오버플로우 페이지
  접근: 전체 값 로드 (부분 읽기 제한적)
  스트리밍: ResultSet.getBinaryStream() (JDBC)

  MySQL에서 4GB LONGBLOB:
  SELECT content FROM files WHERE id = 1 → 4GB를 메모리에 로드!
  → 실무에서 BLOB에 대용량 파일 저장 비권장

PostgreSQL:
  BYTEA: 최대 1GB (실제로 더 작게 유지 권장)
  Large Object: 최대 4TB (스트리밍 가능)

  스트리밍:
  MySQL: BLOB 스트리밍 제한적
  PostgreSQL LO: lo_read()로 청크 단위 스트리밍 가능

결론:
  대용량 파일 DB 저장:
  MySQL: LONGBLOB 비권장 → 파일 시스템/S3 권장
  PostgreSQL: LO로 스트리밍 가능하지만 여전히 S3가 나은 경우 많음
```

---

## ⚖️ 트레이드오프

```
저장 전략별 트레이드오프:

BYTEA:
  장점: SQL 단순, ACID, 자동 TOAST, 작은 파일에 최적
  단점: 부분 읽기 비효율, 메모리 부담, 1GB 실질 한계

Large Object:
  장점: 스트리밍/부분 읽기, 4TB 크기, ACID
  단점: 복잡한 API, 고아 LO 관리, vacuumlo 별도 필요

파일 시스템/S3:
  장점: 무제한 크기, CDN 연동, DB 부하 없음
  단점: 트랜잭션 없음, 두 시스템 관리, 불일치 가능

실용적 결론:
  대부분의 현대 서비스: S3/GCS + DB에 URL 저장
  소형 바이너리(프로필 사진 썸네일 등): BYTEA
  레거시 또는 DB 일원화 필요: Large Object 검토
```

---

## 📌 핵심 정리

```
Large Object vs TOAST vs 파일 시스템:

BYTEA (TOAST):
  저장: pg_toast 테이블, 2KB 청크
  최대: ~1GB (실제로 수 MB 권장)
  부분 읽기: 전체 decompress 필요
  트랜잭션: 완전 지원

Large Object:
  저장: pg_largeobject 테이블, 2KB 페이지
  최대: 4TB
  부분 읽기: lo_get(loid, offset, len) 가능
  트랜잭션: 지원 (명시적 트랜잭션 필요)
  관리: vacuumlo로 고아 LO 정리 필요

파일 시스템/S3:
  최대: 사실상 무제한
  부분 읽기: Range 요청
  트랜잭션: 없음
  운영: 별도 시스템

선택 기준:
  < 1MB + 트랜잭션: BYTEA
  1MB ~ 500MB + 스트리밍 + 트랜잭션: Large Object
  > 500MB 또는 CDN 필요 또는 파일 많음: S3/파일 시스템

Spring LO 접근:
  PGConnection.getLargeObjectAPI() → LargeObjectManager
  반드시 @Transactional 안에서 사용
```

---

## 🤔 생각해볼 문제

**Q1.** Large Object를 트랜잭션 없이 사용하면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

트랜잭션 없이 Large Object를 열면 **연결 종료 시 자동으로 닫힙니다**. 더 심각한 문제는 고아 LO 발생입니다.

시나리오:
1. `lo_create(0)` → OID 12345 생성 (pg_largeobject에 데이터 저장)
2. `lo_write(fd, data)` → 데이터 쓰기
3. `INSERT INTO files (lo_oid) VALUES (12345)` → 실패 (예외 발생)
4. 트랜잭션이 없으면 LO는 이미 커밋된 상태
5. OID 12345는 pg_largeobject에 존재하지만 files 테이블에 없음 → **고아 LO**

트랜잭션이 있으면:
- LO 생성과 INSERT가 같은 트랜잭션
- INSERT 실패 → ROLLBACK → LO도 롤백 → 고아 없음

Spring에서 반드시 `@Transactional`을 사용해야 하는 이유입니다.

</details>

---

**Q2.** `vacuumlo`를 실행하지 않으면 어떤 결과가 발생하는가?

<details>
<summary>해설 보기</summary>

`vacuumlo`를 실행하지 않으면 **고아 Large Object가 영구적으로 `pg_largeobject`에 남아있습니다**. 이로 인해:

1. **디스크 공간 낭비**: 사용되지 않는 LO 데이터가 pg_largeobject를 계속 차지
2. **VACUUM 부하 증가**: pg_largeobject도 VACUUM이 처리해야 하는 테이블이므로 크기가 커질수록 부담
3. **백업 크기 증가**: pg_dump가 고아 LO도 포함해서 백업 크기 증가

`vacuumlo`는 PostgreSQL 기본 VACUUM과 별도로 실행해야 합니다:
```bash
vacuumlo mydb          # 드라이 런 없이 삭제
vacuumlo -v mydb       # 자세한 출력
vacuumlo -n mydb       # 드라이 런 (삭제 없이 목록만)
```

예방:
1. LO 생성과 참조 INSERT를 항상 같은 트랜잭션에 묶기
2. 주기적으로 `vacuumlo` 실행 (cron job 등)
3. PostgreSQL 14+: `lo_manage` 트리거로 참조 삭제 시 LO 자동 삭제

```sql
-- lo_manage 트리거 (PostgreSQL contrib)
CREATE TRIGGER t_files BEFORE UPDATE OR DELETE ON files
FOR EACH ROW EXECUTE FUNCTION lo_manage(lo_oid);
-- files.lo_oid가 변경되거나 행이 삭제될 때 old lo_oid의 LO도 삭제
```

</details>

---

<div align="center">

**[⬅️ 이전: 전문 검색](./04-full-text-search.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 파티셔닝과 대용량 테이블 관리 ➡️](../partitioning/01-partition-overview.md)**

</div>
