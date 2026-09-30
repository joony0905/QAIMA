# 별도 MySQL에서 Flyway 이력과 JDBC 제약 검증

검증일: 2026-09-25 (Asia/Seoul). 기존 프로젝트 MySQL과 완전히 다른 Windows 임시 datadir의 MySQL Server 8.0.27을 `--no-defaults`로 구동했습니다. 프로젝트 설정값의 DB/계정은 사용하지 않았습니다. Java 17, 현재 Gradle 캐시의 Flyway core/mysql 12.4.0 및 MySQL Connector/J 9.7.0을 프로젝트 밖 임시 실행 공간에 복사해 사용했습니다. `backend/src/main/resources`는 **읽기 전용 classpath**로 참조했습니다. 프로젝트 코드·설정·DB·테스트 파일은 수정하지 않았습니다.

## Flyway 검사

새 무작위 `qaima_qa_20260925_...` 스키마를 만들고 소유 확인용 `qa_verification_marker` 테이블·1행을 넣었습니다. 매 실행 전 JDBC의 `@@port`, `@@datadir`, `DATABASE()`가 예상한 임시 서버·QA 스키마와 일치하지 않으면 중단하도록 했습니다. 아래 설정은 저장소의 `IsolatedDatabaseProvisionTest`와 같은 핵심 Flyway 조건입니다.

```java
Flyway flyway = Flyway.configure()
    .dataSource(isolatedJdbcUrl, "root", "")
    .defaultSchema(qaSchema).schemas(qaSchema)
    .locations("classpath:db/migration")
    .baselineVersion("0").baselineOnMigrate(false)
    .cleanDisabled(true).load();
flyway.baseline();
var first = flyway.migrate();
var valid = flyway.validateWithResult().validationSuccessful;
var second = flyway.migrate();
```

최종 새 QA 스키마에서 임시 Java 실행의 종료 코드 0, `MIGRATED=51 VALID=true SECOND=0 HISTORY_ROWS=52 MAX_RANK=52`를 확인했습니다. 이력 52행은 baseline 1행과 성공한 SQL 마이그레이션 51행입니다. 같은 datadir의 이전 QA 스키마에서도 별도로 `validate=true`, 두 번째 migrate 0, SQL 이력51행을 확인했으며, 이전 QA 스키마는 최종 검사 뒤 삭제했습니다. Windows 콘솔 로그는 CP949로 수집했습니다. 첫 임시 실행에서는 로그를 UTF-8로 디코딩하려다가 출력 수집만 실패했으므로, **새 스키마에서 전체 과정을 다시 실행해 종료 코드와 첫 적용 수를 직접 확보**했습니다.

## 실제 JDBC 트랜잭션 검사

최종 Flyway 스키마에서 합성 사용자2명·포트폴리오1개·Feature3 리포트1개를 단일 JDBC 트랜잭션에 넣었습니다. 같은 사용자에게 두 번째 포트폴리오를 넣으면 MySQL 1062, 존재하지 않는 사용자 ID의 포트폴리오를 넣으면 1452가 반환됐습니다. `report_id + user_id + generated_at` 조건의 SELECT는 소유자 1행, 다른 사용자 0행이었습니다. 트랜잭션 롤백 뒤 사용자·포트폴리오·리포트는 각각 0행, QA 소유 marker는 1행임을 다시 조회했습니다. 실행 출력은 `DUPLICATE=1062 FOREIGN_KEY=1452 OWNER=1 OTHER=0 REMAINING=0`이었습니다.

이 검사는 **실제 MySQL 스키마·제약·JDBC 트랜잭션**을 통과하지만, 소유자 SELECT는 임시 검증기가 직접 작성한 SQL입니다. Spring Data JPA Repository 메서드, Controller/Service, 인증 필터, 리포트 삭제 부작용, 동시성·크레딧 원장까지 통과했다고 주장하지 않습니다. 합성 입력만 사용했고 기존 사용자 데이터나 운영 DB는 읽거나 쓰지 않았습니다. 검증 후 임시 서버 정지를 확인했으며, 최종 QA 스키마가 든 datadir는 후속 격리 검증을 위해 프로젝트 밖에 보존했습니다.

이후 같은 스키마에서 [프로젝트 밖 복사본의 Spring Data JPA 검증](isolated-jpa-copy.md)을 별도로 수행했습니다. 위 JDBC 검사 자체의 경계와 구별합니다.
