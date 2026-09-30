# 프로젝트 밖 복사본의 실제 MySQL·Spring Data JPA 검증

검증일: 2026-09-25 (Asia/Seoul). 원본 backend의 `src/main`, Gradle 빌드 정의와 wrapper를 Windows 사용자 임시 디렉터리의 **별도 프로젝트 복사본**에 옮겼습니다. 현재 원본과 복사본의 본체 파일 554개를 SHA-256으로 다시 대조해 불일치 0개를 확인했습니다. QA 테스트 클래스는 복사본의 `src/test`에만 작성했습니다. 원본 프로젝트 코드·테스트·설정·DB는 수정하지 않았습니다.

## 실행 경계

복사본에 `@DataJpaTest`, `@AutoConfigureTestDatabase(replace=NONE)`와 전용 `@SpringBootConfiguration`을 두어 JPA entity·repository만 스캔했습니다. 전체 `QaimaApplication`이나 scheduler는 시작하지 않았습니다. `spring.jpa.hibernate.ddl-auto=validate`, `spring.flyway.enabled=false`를 사용해 [이미 Flyway 검증된 별도 QA 스키마](isolated-flyway-jdbc.md)의 매핑을 확인했습니다. 테스트 전 JDBC `@@port`, `@@datadir`, `DATABASE()`가 보존한 임시 서버·QA 스키마와 정확히 일치해야 fixture를 저장하도록 했습니다. 공급자 API와 기존 설정 DB를 대상으로 하지 않았습니다.

Windows Java 17·Gradle 8.14에서 복사본 `compileTestJava`가 성공했습니다. 초기 `--offline` 시도는 테스트 런타임 의존성 부족으로 실패했고, 복사본에서 누락 의존성을 내려받은 뒤 컴파일·실행했습니다. 첫 테스트 실행은 WSL 환경변수가 Windows Gradle 테스트 프로세스에 전달되지 않아 클래스 초기화에서 중단됐습니다. `WSLENV`로 격리 DB URL·스키마·datadir 변수 전달을 확인한 뒤 최종 실행을 했습니다. 이 두 환경 준비 실패를 제품 테스트 실패로 세지 않습니다.

최종 명령은 복사본에서 `gradlew.bat --offline --no-daemon --max-workers=2 test --tests com.qaima.verification.IsolatedJpaRepositoryCopyTest`였습니다. 격리 DB 값은 외부 환경변수로 주입했고 명령이나 문서에 계정 비밀값을 넣지 않았습니다. Gradle 종료 코드 0; XML `TEST-com.qaima.verification.IsolatedJpaRepositoryCopyTest.xml`은 **2 tests, 0 failures, 0 errors, 0 skipped**입니다.

| 테스트 메서드 | 실제 경로·고정 입력 | 기대·결과 |
|---|---|---|
| `reportRepositoryScopesOwnerCutoffAndDelete` | 실제 User/AnalysisReport entity를 MySQL에 저장. 소유자 A의 9/1·9/24 리포트와 B의 9/1 리포트, UTC cutoff 9/18. 실제 `AnalysisReportRepository`의 상세·목록·사용자별 이전 리포트 삭제 호출 | A의 최신 리포트는 A에게만 조회, A의 오래된 건은 cutoff 상세에서 제외. Feature3 최신 목록 1건, A의 오래된 건 삭제 1건. A 최신/B 오래된 건 유지. PASS |
| `portfolioRepositoryFetchesOwnedHoldingsInPositionOrder` | 실제 User/Portfolio/PortfolioHolding entity를 MySQL에 저장. 현금 100.123456, 보유종목 QA_B position1과 QA_A position0을 역순으로 추가. 실제 `PortfolioRepository.findByUserIdWithHoldings` 호출 | 두 보유종목이 QA_A→QA_B 순서로 fetch되고 현금 6자리 소수 보존. 다른 사용자 ID 조회는 빈 결과. PASS |

`@DataJpaTest` 트랜잭션 롤백 뒤 격리 DB를 별도 연결로 조회해 users·portfolio·portfolio_holding·analysis_report가 모두 0행, QA 소유 marker 1행, Flyway history 52행임을 확인했습니다. 임시 MySQL 서버도 종료했습니다. 실제 저장소 메서드와 Hibernate/JDBC를 통과했지만 Controller·Service·인증 필터, 리포트 조회 서비스의 7일 정리 부작용, 동시성·실제 사용자 데이터·Redis는 검증하지 않았습니다. JPA `validate` 성공은 모든 업무 규칙과 SQL 성능의 증명이 아닙니다.

이후 같은 격리 스키마와 임시 복사본에서 [Service 및 Controller 직접 호출 검증](isolated-service-jpa-copy.md)을 별도로 수행했습니다. 위 저장소 단독 검사와 중복 집계하지 않습니다.
