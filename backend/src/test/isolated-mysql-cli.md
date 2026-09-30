# 격리 MySQL에서 마이그레이션 SQL 실행 확인

검증일: 2026-09-25 (Asia/Seoul). Windows에 설치된 MySQL Server 8.0.27 실행 파일로 사용자 임시 디렉터리 아래에 **별도 datadir**를 초기화하고, `--no-defaults`와 임시 loopback 포트로 구동했습니다. 기존 프로젝트 설정의 MySQL 서버·스키마·계정은 사용하지 않았습니다. 매 SQL 변경 전 `@@port`와 `@@datadir`이 임시 서버와 일치하는지 확인했습니다. 프로젝트 파일·코드·기존 DB는 수정하지 않았습니다.

`backend/src/main/resources/db/migration`의 `V1`~`V51` SQL 51개를 번호순으로 새 무작위 `qaima_qa_20260925_...` 스키마에 MySQL CLI 표준입력으로 적용했습니다. 클라이언트에는 `--default-character-set=utf8mb4`를 명시했습니다. 정상 종료 51개, 스키마 테이블 47개, 외래키 제약 41개를 확인했고 `industry_index`의 한글 종목군 seed `00005 / 음식료·담배`가 정확히 1행 조회됐습니다. `dictionary`는 빈 상태였으며 실제 용어 데이터를 적재하거나 조회 품질을 검사한 결과가 아닙니다.

처음 실행은 Windows CLI 기본 문자셋으로 V1~V12까지 진행한 뒤 V13의 한글 문자열에서 MySQL 1366으로 멈췄습니다. `utf8mb4` 지정 후 나머지를 적용했고, 이 중간 상태를 최종 PASS의 근거로 쓰지 않았습니다. **완전히 새 QA 스키마를 생성해 V1부터 V51까지 동일한 `utf8mb4` 설정으로 다시 실행**한 결과가 위 51개 성공입니다. 처음의 부분 QA 스키마만 임시 서버에서 삭제했습니다.

재현 시 확인 순서는 다음과 같습니다.

1. 프로젝트 밖 임시 datadir에 `mysqld --no-defaults --initialize-insecure`를 실행합니다. 임시 서버를 별도 loopback 포트에서 시작합니다.
2. 임시 서버에 접속해 `SELECT @@port, @@datadir`을 실행하고, 두 값이 1단계에서 선택한 포트·datadir와 같은지 비교합니다. 다르면 **스키마 생성이나 SQL 실행을 중단**합니다.
3. 무작위 QA 스키마를 새로 만들고 `V[번호]__*.sql` 51개를 숫자 오름차순으로 `mysql --no-defaults --default-character-set=utf8mb4`의 표준입력에 각각 전달합니다. 각 명령의 종료 코드가 0인지 검사하고 첫 오류에서 중단합니다.
4. `information_schema.tables`와 `information_schema.table_constraints`, 한글 seed를 조회합니다. 임시 서버만 정상 종료합니다.

이 검사는 **SQL 파일의 MySQL 8.0.27 실행 가능성**에 한정됩니다. Flyway의 history·baseline·checksum·재실행 무변경, Java/JPA 매핑, 실제 트랜잭션·동시성·저장소·서비스/API 전체 흐름은 검사하지 않았습니다. 기존 `IsolatedDatabaseProvisionTest`는 프로젝트 YAML의 기존 MySQL 계정을 사용하므로 이 임시 서버에 연결하지 않았고, 프로젝트 안의 `.runtime` journal도 생성하지 않았습니다. 테스트 계정으로 `CREATE DATABASE`가 실패한 과거 결과는 이 검사와 별개로 유지합니다.

이후 같은 임시 인스턴스의 **새 QA 스키마**에서 Flyway baseline·51개 적용·validate·재실행과 실제 JDBC 제약 검사를 완료했습니다. [후속 검증](isolated-flyway-jdbc.md)을 참조하세요. CLI로 만든 성공 스키마와 최초 문자셋 오류의 부분 스키마는 정리했고, 최종 Flyway QA 스키마만 남겼습니다. 임시 서버는 종료했으며 datadir는 프로젝트 밖 Windows 임시 디렉터리에 보존했습니다.
