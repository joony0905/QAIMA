# 격리 MySQL + 실제 HTTP/JWT/JPA 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03`와 현재 미커밋 소스를 대상으로 합니다. [실행기](run_isolated_backend.py), [Gradle 테스트 설정](backend-isolated.init.gradle), [테스트 클래스](java/com/qaima/qa/IsolatedHttpJpaTest.java)는 모두 루트 tests 안에 있습니다. 제품 소스·빌드 정의·운영 설정은 수정하지 않습니다.

## 검증 구성과 경계

기존 JPA 검증은 Controller 메서드 직접 호출까지였습니다. 이번 구성은 Java HttpClient → 실제 loopback TCP → Netty/WebFlux 라우팅·validation → 실제 SecurityConfig/JwtAuthFilter/JwtTokenProvider → 실제 Controller/Service → Spring Data JPA/Hibernate/JDBC → 외부 격리 MySQL입니다.

테스트 전용 SpringBootConfiguration에서 인증·포트폴리오·리포트·관심목록·크레딧 Controller와 필요한 실제 service만 명시적으로 등록합니다. 전체 QaimaApplication·component scan·scheduler·importer·금융 API client는 등록하지 않습니다. SMTP 경계 MailAuthService는 호출하면 테스트가 실패하는 mock이고 OAuth 성공/실패 handler는 mock입니다. 외부 OAuth·메일 발송·회원가입 인증을 수행한 검사가 아닙니다.

MySQL은 `/tmp/qaima-qa-http-*`의 새 datadir와 무작위 `qaima_qa_*` 스키마를 사용합니다. 기존 앱 YAML을 읽어 접속하지 않습니다. 서버는 `--no-defaults`, loopback 주소, 빈 임시 포트, mysqlx 비활성, local-infile 비활성으로 시작합니다. 스키마 생성 전 실제 `@@port`·`@@datadir`를 확인하고, 이후 SQL 전에는 `DATABASE()`와 QA 소유 표식도 확인합니다. Java DataSource 생성 시 같은 가드를 통과해야 JPA validate가 시작됩니다.

V1~V51 SQL은 숫자 순서로 원본을 utf8mb4 CLI stdin에 전달합니다. 이번 실행은 SQL 적용이며 Flyway history/checksum 검사는 과거 기록과 구분합니다. Hibernate ddl-auto=validate, open-in-view=false이고 테스트 메서드는 rollback annotation으로 감싸지 않습니다. HTTP 서비스의 실제 commit/rollback을 독립 JDBC 조회로 확인하기 위해 fixture를 실제 commit합니다.

각 테스트 전 합성 사용자 A/B/관리자 세 명을 저장하고, 테스트 후 소유 표식 확인 → 해당 전용 DB의 업무9테이블 삭제 → 0행 확인을 수행합니다. 관심목록 테스트의 QA 종목도 제거합니다. 실행기는 Gradle 종료 후 별도 CLI로 남은 행을 조회하고 자신이 시작한 mysqld에 SIGTERM을 전달해 정상 종료를 기다립니다. 기존 DB·서비스 PID는 제어하지 않습니다.

## 준비·재현 명령

WSL에서 `/tmp`에 Ubuntu 패키지를 **설치 없이 추출**합니다. apt download는 DB 서비스를 등록하거나 시작하지 않습니다. 2026-09-28 실행 버전은 MySQL 8.0.46-0ubuntu0.24.04.4입니다.

```bash
mkdir -p /tmp/qaima-qa-mysql-20260928/packages /tmp/qaima-qa-mysql-20260928/root
cd /tmp/qaima-qa-mysql-20260928/packages
apt-get download mysql-server-core-8.0 mysql-client-core-8.0 libaio1t64 libmecab2 libprotobuf-lite32t64 libevent-pthreads-2.1-7t64 libnuma1
cd /tmp/qaima-qa-mysql-20260928
for package in packages/*.deb; do dpkg-deb -x "$package" root; done
cd /mnt/c/qaima
python3 -B tests/run_isolated_backend.py
```

실행기는 기존 테스트 안의 Gradle8.14 배포본과 의존성 cache를 읽기 전용으로 재사용합니다. 새 Gradle cache·컴파일·리포트·임시 파일은 `tests/.runtime/backend-isolated`에 둡니다. 원본 `backend/src/test/.runtime`에는 쓰지 않습니다. `processResources`에서 실제 application*.yml/yaml/properties를 제외하고 테스트의 명시적 설정만 로드합니다. 네트워크 의존성 다운로드 없이 `--offline`으로 실행합니다.

초기 준비에서 읽기 전용 Gradle cache를 modules-2 하위 경로로 지정해 plugin resolution에 실패했습니다. 도구가 요구한 상위 caches 경로로 고친 후 제품 compileJava는 성공했습니다. SQL/MySQL·제품 테스트 실패로 세지 않습니다. 최초 Java 컴파일은 5분26초였으며 이후 컴파일 산출물을 재사용하고 test task만 항상 다시 실행합니다.

## 검사와 독립 기대값

| 메서드 | 입력·행동·기대값 |
|---|---|
| anonymousAndInvalidSignatureCannotReachProtectedHandlers | report/portfolio/watchlist/credit GET을 무인증과 잘못된 서명으로 요청 → 각각401. 포트폴리오 PUT401. DB 쓰기0 |
| loginRefreshAndLogoutPersistHashAndSessionState | 실제 BCrypt 비밀번호 로그인 → 서명 검증 가능한 JWT로 보호 API200. refresh 원문과 별도 JCA HMAC 기대값을 DB hash와 비교. refresh 회전 후 이전 토큰401, logout 만료cookie·revoked_at·이후401 |
| expiredSessionAndWrongPasswordAreRejectedWithoutRotation | 틀린 비밀번호400/세션0. 정상 로그인 후 격리DB expires_at을 과거로 바꾸고 refresh401/저장 hash 불변 |
| portfolioHttpPersistsReplacesAndSeparatesOwners | A/B 각각 저장 → A trim·6자리 현금 소수·전체교체/orphan 제거 → 빈 보유목록. B 종목 유지·독립DB행 대조 |
| invalidPortfolioAndSqlFailureDoNotPartiallyReplaceSavedHoldings | 기존 QA_KEEP/cash12.5 저장 → 음수 cash400 → DB column 한도 초과 종목코드500. 기존 cash/보유종목/행수 보존 |
| reportHttpScopesSnapshotsAndRetentionToJwtOwner | 실제 report service로 A 최근/9일 전·B9일 전 snapshot 저장 → A HTTP 목록은 최근1, A과거 삭제/B과거 유지. 상세 JSON0.125·warning, B의 A상세404 |
| watchlistHttpChecksOwnershipDuplicateAndActualDelete | 합성종목 추가 → 같은종목 중복400 → 소유자 note수정200 → 타인 수정/삭제400·note/행 유지 → 소유자 삭제200/행0 |
| creditAdminHttpPermissionAndLedgerAreAppliedAtomically | charge 무인증401/USER403/원장0 → ADMIN200 → 잔액5→8/원장+3. A/B HTTP 원장 분리 |
| creditUseAndRefundMatchStoredLedgerAndHttpBalance | 실제 CreditService 차감/환불5→4→5, DB USE/REFUND2행·합0과 HTTP 잔액 일치 |
| creditSqlInsertFailureRollsBackBalanceUpdate | column한도 초과 referenceId로 실제 원장 INSERT 실패 유도 → 잔액5/원장0으로 트랜잭션 rollback |
| concurrentCreditDebitsCannotSpendMoreThanAvailableBalance | 잔액5에12개 차감 작업을 latch로 동시에 출발 →5성공/7잔액부족, 잔액0/원장5행/sum−5/balance_after0~4 |
| concurrentRefreshConsumesOriginalTokenOnlyOnce | 실제 로그인으로 만든 같은 refresh token을12개 TCP 요청에서 동시에 사용 → 한 번만200, 나머지401이어야 함. 발급 쿠키들의 독립 HMAC과 최종 DB hash를 비교하고 원문 없이 개수만 기록 |

사용자는 DB fixture로 생성합니다. 로그인 검사는 실제 AuthService와 JWT/session/JPA를 통과하지만 다른 소유권 검사는 테스트용 서명키로 발행한 진짜 JWT를 전달합니다. BCrypt 비용은4, access/refresh TTL은3600초인 테스트 설정입니다. 운영 비밀번호 강도·운영 키·브라우저 HTTPS cookie·OAuth 설정의 적합성 검사가 아닙니다.

크레딧 차감/환불은 공개 호출 endpoint가 없어 실제 CreditService를 직접 호출하고 HTTP 읽기로 대조합니다. Feature1/2/3 전체 분석→실패→환불 orchestration을 이 검사만으로 완료했다고 보지 않습니다. SQL 오류 주입은 mock이 아닌 실제 컬럼 한도 위반입니다.

## 결과 기록

첫 실행 `tests/.runtime/backend-isolated/runs/run-8wjdugxs`: 11개10 PASS/1 FAIL, Gradle exit1, JUnit skipped/error0입니다. 실패는 테스트 helper가 role을 소문자로 JWT에 넣어 ADMIN 요청이403이 된 fixture 오류였습니다. 실제 AuthService의 로그인은 role을 대문자로 변환합니다. helper를 실제 계약에 맞추고 관리자 검사는 실제 로그인으로 발급받은 `ROLE_ADMIN` token을 사용하도록 보정했습니다. 제품의 관리자 권한 결함으로 집계하지 않습니다.

이 실행에서 V1~V51 SQL51개와 Hibernate validate, 실제 저장/롤백·소유권·로그인/회전/폐기·동시 차감은 통과했습니다. 별도 CLI에서 업무9테이블 모두0행, mysqld 종료코드0, mysqlStopped true, 실행 중 sourceChangedDuringRun0입니다. 요약의557개 해시는 제품 입력과 테스트 코드/설정을 포함합니다. XML은 `TEST-com.qaima.qa.IsolatedHttpJpaTest.xml`, 실행 로그는 `gradle.log`, DB/파일 감사는 `summary.json`입니다.

보정 후 전체 `run-f993pmv8`: **12개11 PASS/1 FAIL**, JUnit skipped/error0, Gradle exit1입니다. 관리자 정상200까지 통과했고 새 실패는 동시 refresh 검사입니다. 업무9테이블 모두0행, sourceChangedDuringRun0, mysqld 정상종료0/mysqlStopped true입니다. 테스트 개수는 HTTP 요청 수와 다릅니다.

Java17.0.20.1(Linux), Gradle8.14, Spring Boot3.2.5의 선택된 context를 사용했습니다. V1~V51 적용이나 Spring 전체 기본 suite를 이번12개에 중복 합산하지 않습니다.

## AUTH-REFRESH-RACE-001 — 동시 refresh 성공 응답과 저장된 세션 불일치

`run-f993pmv8`에서 같은 원래 refresh token을 가진12개 실제 TCP 요청을 latch로 출발시켰습니다. **HTTP200 12개, 401 0개, 최종 DB hash와 일치하는 성공 응답의 후속 token은1개**입니다. 세션 행은1개이고 각 응답의 후속 token을 별도 JCA HMAC으로 계산해 비교했습니다. 토큰 원문은 출력하지 않았습니다.

[LoginSessionService](../backend/src/main/java/com/qaima/service/auth/LoginSessionService.java) 60행은 hash로 조회하고 71~78행에서 새 hash를 만들고 저장합니다. 이 전체 구간을 하나의 잠금/원자적 조건부 갱신으로 묶지 않습니다. [LoginSessionRepository](../backend/src/main/java/com/qaima/repository/LoginSessionRepository.java)의 조회에도 row lock이 없습니다. 동시에 읽은 여러 요청이 같은 행의 hash를 서로 덮어쓰는 실행 결과와 일치합니다.

순차 회전 후 이전 토큰401은 통과했지만, 같은 토큰을 동시에 소비하는 경우 한 번만 성공해야 한다는 회귀는 실패합니다. 운영 발생률·다중 인스턴스·탈취/계정침해를 증명한 검사가 아닙니다.

추가 단독 재현 `run-9yikqkqi`: 1개 FAIL, skipped/error0, Gradle exit1입니다. 다시 **200 12개/401 0개/DB에 남은 성공 후속토큰1개**를 확인했습니다. 그중 DB hash에서 밀려난 성공 쿠키 하나로 실제 refresh를 요청하면 **401**, 남은 쿠키는 **200**이었습니다. JUnit XML system-out의 `QA_REFRESH_CONCURRENCY`, `QA_REFRESH_SUCCESSOR`에 개수를 남겼고 쿠키 원문은 남기지 않았습니다. 성공 응답을 받은 클라이언트가 이후 갱신에서 실패할 수 있는 세션 일관성 문제를 실제 SQL/TCP로 확인한 것입니다.

단독 재현은 전체12개와 중복 집계하지 않습니다. 단독 실행도 최종 업무9테이블0행, sourceChangedDuringRun0, mysqld 정상 종료0/mysqlStopped true입니다. 세 run의 데이터 디렉터리는 프로젝트 밖에 보존됐고 서버 프로세스는 종료됐습니다.

동시성 실패만 재현하려면 다음을 사용합니다. 새 DB를 생성하고 해당 메서드만 실행하므로 전체12개와 합산하지 않습니다.

```bash
python3 -B tests/run_isolated_backend.py --test concurrentRefreshConsumesOriginalTokenOnlyOnce
```

## Swagger 작업과 검증 경계

| method/path | 이번 연결 범위 |
|---|---|
| POST `/api/v1/auth/login`, `/refresh`, `/logout` | 실제 Controller·비밀번호/JWT·쿠키·세션/감사로그 SQL, 정상/오류·만료·재사용 |
| GET/PUT `/api/v1/portfolios/me/default` | JWT principal·저장/조회·다른 사용자 분리·validation·SQL 실패 rollback |
| GET `/api/v1/reports/me`, `/{reportId}` | JWT 소유권·필터 목록·snapshot·7일 정리·타인404. 생성은 실제 service 직접 호출 |
| GET `/api/v1/watchlist/me`, POST `/me/items`, PATCH/DELETE `/items/{itemId}` | JWT·기본목록 생성·중복·note·타인 차단·실제 삭제 |
| GET `/api/v1/credits/balance`, `/ledger`, POST `/api/v1/admin/credits/charge` | JWT·관리자 권한과 사용자별 원장/잔액, 실제 SQL |

총14개 method/path의 일부 정상·오류·권한·저장 경계를 확장했습니다. 이14개에 대해 모든 오류분기·동시성·전체 앱을 완료했다고 보지 않습니다. signup/find-id·명시 watchlist 경로·admin adjust/temp-charge 등 나머지 작업은 이전 상태를 유지합니다. 루트 docs/와 backend/src/test 문서는 이번 수정 범위 밖이므로 이 표를 증분 근거로 사용합니다.

## 남은 경계

refresh 동시 회전은 위12요청 검사에서 실패했습니다. 로그아웃 경쟁, 실제 signup/email/OAuth, 전체 앱의 다른 필터, Redis, 실제 분석 pipeline 비용/환불/저장, 모든 Swagger 업무경로, 다중 프로세스 동시성·성능은 별도입니다. 발견한 제품 실패는 수정하지 않고 재현 근거로 보존합니다.
