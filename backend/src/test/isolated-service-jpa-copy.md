# 프로젝트 밖 복사본의 Controller·Service→실제 MySQL 검증

검증일: 2026-09-25 (Asia/Seoul). 앞선 [격리 JPA 검증](isolated-jpa-copy.md)의 Windows 임시 backend 복사본을 사용했습니다. 현재 원본의 `src/main`과 Gradle 정의 554개 파일을 SHA-256으로 다시 비교해 불일치 0개를 확인했습니다. 추가 QA 클래스는 **임시 복사본의 `src/test`에만** 작성했습니다. 원본 코드·테스트·설정·기존 DB는 수정하지 않았습니다.

복사본의 `@DataJpaTest`는 전용 `@SpringBootConfiguration`으로 entity/repository와 실제 `AnalysisReportService`, `PortfolioService`만 로드했습니다. Jackson `ObjectMapper`와 JPA `TransactionTemplate`은 테스트 설정에서 제공했습니다. `spring.jpa.hibernate.ddl-auto=validate`, `spring.flyway.enabled=false`, 기존 Flyway 이력52행의 임시 MySQL 스키마를 사용했습니다. 전체 `QaimaApplication`·scheduler는 시작하지 않았습니다. 합성 fixture 저장 직전과 정리 직전에 JDBC `@@port`·`@@datadir`·`DATABASE()`가 격리 대상과 일치하는지 검사했습니다.

서비스가 `Blocking.call`로 다른 스레드에서 저장소를 호출하므로 테스트 메서드의 기본 롤백 트랜잭션을 끄고, fixture는 실제 commit했습니다. 매 검사 뒤 합성 사용자 ID에 속한 리포트·포트폴리오 보유종목·포트폴리오·사용자를 격리 DB에서 정리했습니다. 최종 별도 연결 조회에서 users·portfolio·portfolio_holding·analysis_report 모두 **0행**, QA marker 1행, Flyway history 52행이었습니다. 임시 서버는 종료했습니다.

최종 실행은 임시 복사본에서 `gradlew.bat --offline --no-daemon --max-workers=2 test --tests com.qaima.verification.IsolatedServiceJpaCopyTest`입니다. WSL→Windows 환경 전달에는 `WSLENV`를 사용했고, DB URL·스키마·datadir는 환경변수로 주입했습니다. Gradle 종료 코드 0. XML `TEST-com.qaima.verification.IsolatedServiceJpaCopyTest.xml`은 **3 tests, 0 failures, 0 errors, 0 skipped**입니다.

| 검사 | 입력과 실제 실행 경로 | 관측 결과 |
|---|---|---|
| `reportServiceCreatesSnapshotsAndScopesRetention` | 사용자 A·B를 실제 MySQL에 생성. A의 9일 전/1일 전 리포트, B의 9일 전 리포트를 `AnalysisReportService.create`→JPA로 저장. A의 `listMine`·`getMine`과 B의 타인 리포트 상세 시도 | A 목록은 최신 1건, A의 오래된 건은 실제 삭제, B의 오래된 건은 A 조회 시 유지. 상세 응답의 사용자명·요청 JSON·결과 JSON·warning을 확인. 타인 상세는 거부. PASS |
| `portfolioServiceReplacesHoldingsAndKeepsOtherOwner` | 실제 `PortfolioService.replaceMyPortfolio`→`TransactionTemplate`→JPA/MySQL. 현금 100.123456, 공백 있는 QA_A·QA_B를 저장한 뒤 QA_C 하나로 교체하고 빈 목록으로 교체. B의 QA_X 포트폴리오를 대조군으로 저장 | trim·소수 보존·교체 후 이전 보유종목 제거·빈 목록 orphan 제거, B의 데이터 유지. PASS |
| `controllersPassPrincipalToRealServicesAndJpa` | 실제 `PortfolioController`와 `AnalysisReportController` **메서드를 직접 호출**, `Long` principal의 합성 Authentication을 A/B에 전달. 포트폴리오 저장·조회와 리포트 목록·상세, 미인증·타인 상세 조회 검사 | A/B 조회 분리·미인증 거부. 상세 GET 메서드 호출이 A의 9일 전 리포트를 실제 DB에서 삭제하는 부작용까지 확인. PASS |

세 번째 검사는 Controller 코드까지 연결하지만 HTTP 바인딩·실제 보안필터/JWT·브라우저를 거치지 않습니다. 이전의 HTTP fixture 검증과 이 실제 DB 검증을 하나의 완전한 E2E PASS로 합치지 않습니다. 리포트의 사용자별 삭제가 조회 중 발생하는 것은 현재 구현의 관측 결과이며 보존 정책의 적절성·동시성·대량 데이터 성능까지 검증한 것은 아닙니다. Feature1 저장 필드 뒤바뀜과 다른 기존 결함도 해결되지 않았습니다.
