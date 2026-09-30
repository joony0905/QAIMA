# Spring CI/QA 테스트 기록

검증일: 2026-09-25. Windows Java 17.0.12, Gradle wrapper 8.14, Spring Boot 3.2.5. 기존 `src/test`는 없었으며 이번 검증에서 테스트 패키지를 추가했습니다. 서비스 코드·설정·운영 데이터 수정은 수행하지 않습니다.

## 실행

Windows PowerShell에서 backend 디렉터리 기준:

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY,Env:QAIMA_LIVE_FASTAPI_TCP,Env:QAIMA_LIVE_PEER_SOURCE,Env:QAIMA_LIVE_KIS_READONLY -ErrorAction SilentlyContinue
$env:GRADLE_USER_HOME = "$PWD\src\test\.runtime\gradle-home"
$env:TEMP = "$PWD\src\test\.runtime\tmp"
$env:TMP = $env:TEMP
New-Item -ItemType Directory -Force -Path $env:TEMP | Out-Null
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test
```

`verification.init.gradle`이 컴파일·리포트 산출물을 `src/test/.runtime/build`로 보냅니다. Gradle 다운로드·캐시·임시 파일도 test 안에 둡니다. `.runtime`은 테스트 패키지의 `.gitignore`로 제외합니다. 빌드 리소스에 secret 설정이 복사될 수 있으므로 runtime 디렉터리를 공개 배포하지 않습니다.

실제 인프라의 읽기 전용 검사만 별도로 실행하려면:

```powershell
$env:QAIMA_LIVE_READONLY = "1"
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests com.qaima.verification.LiveInfrastructureTest
Remove-Item Env:QAIMA_LIVE_READONLY
```

이 테스트는 YAML에서 설정을 읽어 MySQL `SELECT 1`과 Redis PING을 수행합니다. Spring application context, scheduler, importer를 시작하지 않습니다. 비밀값은 출력하지 않습니다. 기본 단위 테스트에서는 opt-in 환경변수 없을 때 건너뜁니다.

별도 Windows 임시 MySQL 8.0.27 인스턴스에서는 프로젝트 밖 datadir와 QA 스키마를 사용해 V1~V51 SQL을 모두 실행했습니다. 이 결과는 위 YAML 계정의 DB 생성 실패를 바꾸지 않으며 Flyway/JPA 전체 PASS도 아닙니다. [51개 실행·문자셋·격리·검증 경계](isolated-mysql-cli.md)를 참조하세요.

이어 **다른 새 QA 스키마**에서 Flyway baseline0→SQL51개→validate 성공→재실행0개를 실제 JDBC로 확인했습니다. 합성 사용자/포트폴리오/리포트의 중복·외래키·소유자 조건과 rollback도 검사했습니다. [Flyway/JDBC 근거와 남은 JPA 경계](isolated-flyway-jdbc.md)를 참조하세요.

같은 격리 스키마에 프로젝트 밖의 backend 복사본을 연결한 JPA slice 검증 2개도 통과했습니다. 원본 본체 파일 554개 일치, 실제 AnalysisReport/Portfolio Repository 조회·사용자별 오래된 리포트 삭제·보유종목 순서·트랜잭션 rollback을 확인했습니다. [방법·입력·실제/미검증 경계](isolated-jpa-copy.md)를 참조하세요.

이어 임시 복사본에서 실제 Report/Portfolio Service→Repository→MySQL과 Controller 직접 호출 **3 PASS**를 확인했습니다. 리포트 JSON snapshot·사용자별 7일 정리, 포트폴리오 교체/orphan 제거·타인 분리, 상세 조회의 DB 삭제 부작용을 검사했습니다. [3개 검사·commit/정리·HTTP/보안 경계](isolated-service-jpa-copy.md)를 참조하세요. 이 3개는 원본 Spring 기본1053개에 포함되지 않습니다.

## 검증 방법·기대값

KIS 실제 읽기 검증은 토큰/기본정보 요청2회 모두HTTP200이나 후속단언1 FAIL입니다. 최초마스킹으로 실패단언위치를특정하지못해단계기록을보완했고추가실제호출은하지않았습니다. 호출guard4개는PASS이며 [메서드별 방법·실제 흐름·예산·한계](live-kis-readonly.md)를 참조하세요.

WSL FastAPI→Windows 실제 Peer Controller/Service 역방향 TCP는 별도8개 중7 PASS·최신일누락1 FAIL이며 Java 임시서버 lifecycle1개 PASS입니다. 기본suite에선host1개가opt-out됩니다. [전체 실행·8개 방법·격리/한계](peer-reverse-tcp.md)를 참조하세요.

가격·산업지수 캐시 reader 추가39개는37 PASS·2 FAIL입니다. 실제 계산·codec·24개 겹치는 요청·산업지수 Public HTTP까지 연결했습니다. 가격 빈 목록 NPE와 부분 결과 캐시의 경고 소실을 재현했으며, 실제 Redis/DB 쓰기는 하지 않았습니다. [39개 상세 방법·결과](market-cache-readers.md)를 참조하세요.

| 클래스 | 입력·실행 흐름 | 기대값 | 대체한 경계·한계 |
|---|---|---|---|
| `ContractsTest` (5) | 사전 영문 공백/대소문자·한글 초성, 4개 투자 수준, Feature1 내부 DTO 직렬화, 동일일 candle2개·누락행 | 정규화·초성, 수준 roundtrip, snake_case·빈목록 기본값, 마지막 일봉·KST 자정·거래량0 | 실제 순수 코드. 금융 제공처·DB 없음 |
| `CreditServiceTest` (4) | 잔액5→Feature1 차감→환불, 잔액0 차감, Feature3 비용0, null 사용자 | 5→4→5, 잔액부족이면 저장0회, 최소비용1, UNAUTHORIZED | Repository와 TransactionTemplate mock. 트랜잭션 callback은 실행. 실제 잠금·원자성·중복환불 검증 아님 |
| `ReportOwnershipTest` (2) | 사용자7·리포트42 조회에 미존재/존재 응답 주입 | 사용자·기간 조건으로 조회, 미존재 RESOURCE_NOT_FOUND, snapshot JSON 값0.05 복원 | Repository mock. 조회 시 정리 delete 호출도 확인하며 실제 DB 삭제 없음 |
| `SwaggerInventoryTest` (1) | compiled Controller reflection ↔ 저장 OpenAPI | 양방향 operation 집합 동일, 총123 | 라우트 일치만 검사. 응답 schema·비즈니스 결과를 검사하지 않음 |
| `ApiSecurityMatrixTest` (369) | 123개 작업 × anonymous/USER/ADMIN. 실제 SecurityConfig→JwtAuthFilter→가상 handler | 관리자:401/403/200, 인증필수:401/200/200, 공개:200/200/200 및 오류 envelope | JWT provider·OAuth handler mock, 비즈니스 handler는 sentinel. 토큰 암호검증·실제 Controller/DB가 아님 |
| `LiveInfrastructureTest` (2, opt-in) | Windows YAML 설정→JDBC read-only transaction→SELECT1→rollback, Redis RESP PING | 값1, PONG | 실제 연결. 업무 테이블·캐시 hit/miss·트랜잭션 동시성은 별도 |
| `AuthPrimitivesTest` (4) | 합성 HMAC 키로 발행→검증→principal 복원, 다른키·잘못된문자·만료토큰, refresh cookie 생성·만료 | 사용자7·ROLE_USER, 잘못된 토큰 거부, HttpOnly·Secure·Path·SameSite·최소60초·만료0초 | 실제 JWT·cookie 코드, 사용자 DB·세션 회전 없음 |
| `Feature1FlowTest` (5) | 실제 Controller→하위 service mock→크레딧·캐시·리포트 orchestration | 성공 reportId42, 저장실패 경고·환불없음, 계산실패 환불, 미인증 사전차단, 내부 메시지 비공개 | 4 PASS, 내부 메시지 비공개1 FAIL. 직접 메서드 호출이며 HTTP status·DTO binding 검증 아님 |
| `DictionaryHttpFlowTest` (6) | 실제 WebTestClient HTTP→Controller→Service→Repository mock→DTO/ExceptionHandler | capm→CAPM·camelCase·description, 미존재404, q누락400, 초성목록 envelope, 잘못된 page400, 없는 단어 별칭404 | 실제 HTTP 처리층·Service. DB query·정상 검색/자동완성 후보 정렬·실데이터 내용 미검증 |
| `YahooPricePolicyTest` (6) | 실제 provider에 Yahoo client만 mock, 100일 중96/95/80/79개 가격, 제공처 오류, KOSPI/KOSDAQ/시장미상 | 결측4% 정상·5/20% 경고사용·21% 전체 raw fallback, 부분 수정종가 미반환, .KS/.KQ 선택·후보 순서 | 6 PASS. 실제 Yahoo 네트워크·raw DB 보존·전체 Spring 응답은 별도 |

초기 Security matrix에서 `/`와 `/api/debug/ticker-meta`를 가상 handler가 받지 않아 4개가 404로 실패했습니다. test의 handler 매핑을 `/**`로 고친 후 381개가 모두 통과했습니다. 제품 Controller나 보안 정책은 변경하지 않았습니다.

### Feature2/3 Controller 추가 검증

`Feature2FlowTest` 5 PASS: 실제 Controller 직접 호출로 성공 reportId42·warning 보존, 분석 오류 시 환불·빈지표 fallback, 저장 오류 시 결과 유지, 차감 거부 시 분석 구독0·환불0, 미인증 시 service 호출0을 확인했습니다. 분석·크레딧·리포트 service만 mock입니다. fallback은 현재 success envelope이며 HTTP binding·실제 계산·원장 저장은 별도입니다.

`Feature3FlowTest` 7개 중5 PASS·2 FAIL: 실제 Controller가 가격·벤치마크→내부DTO→FastAPI→overlay→snapshot을 조립하도록 경계 service/repository만 mock했습니다. 기본 옵션·통화·투자수준·무위험수익률 매핑, 정상 reportId42, 보고서 저장 실패 시 결과 유지, FastAPI 오류 환불, 비용계산 실패 차감0, 미인증 차단을 확인했습니다. 가격/overlay 로딩 실패 두 케이스는 차감 Mono 구독1회 이후 환불0회로 FAIL했습니다(`F3-CREDIT-001`). 실제 잔액 변경은 없습니다. 직접 메서드 검증이며 DB 잠금·HTTP DTO binding은 포함하지 않습니다.

추가분만 재실행하려면 아래를 기존 Gradle 명령 뒤에 붙입니다.

```powershell
--tests '*Feature2FlowTest' --tests '*Feature3FlowTest'
```

## 결과·증거 위치

`LoginSessionFlowTest` 6 PASS: 합성 키·실제 SecureRandom/HMAC/서비스와 상태를 보존하는 repository mock으로 64byte refresh token 발행→HMAC만 저장→최소TTL60초·UA512자 제한, 회전 후 이전 토큰 재사용 거부, 멱등 폐기와 폐기 후 회전 거부, 만료/만료일 누락 거부, 빈/미등록토큰 저장0, null/미저장 사용자 발행 거부를 검증했습니다. HMAC 기대값은 별도 JCA 계산으로 비교하고 token 원문은 출력하지 않습니다. 순차 회전 검증이며 실제 DB 경쟁 상태·동시 refresh 안전성은 미검증입니다.

`AuthHttpFlowTest` 6 PASS: 실제 WebTestClient HTTP→AuthController→AuthService→LoginSessionService→JWT/BCrypt/cookie→GlobalExceptionHandler를 연결했습니다. Repository와 가입 인증소비 경계(MailAuthService)만 mock하며, TransactionTemplate은 callback을 실행합니다. BCrypt는 테스트 속도를 위해 cost4, JWT는 합성 키입니다.

- signup: 대문자 email·하이픈 전화번호·7자리 생년월일·country 공백 fixture → 정규화·gender·초기크레딧5·인증소비·비밀번호 해시 저장. 응답 passwordHash 미포함.
- login→refresh→이전토큰 재사용→logout→폐기토큰 재사용: 서명 검증 가능한 access token, HttpOnly/Secure/Path cookie, refresh 교체, 재사용401, logout Max-Age0 및 revokedAt 기록.
- 잘못된 비밀번호·inactive·미인증 사용자 로그인: 현재 계약400, 세션 저장0.
- 빈 로그인 JSON: Bean Validation400, 저장소·메일 접근0.
- refresh cookie 없음:401, cookie 없는 logout:200·만료 cookie, session repository 접근0.
- find-id: 단일 일치 시 `fi***@example.invalid`, 다중 일치 시400.

이 HTTP 테스트는 보안필터 없는 Controller 바인딩 방식이며 실제 TCP·DB·브라우저 쿠키 정책 검증은 아닙니다. 실제 메일·OAuth·개인정보를 사용하지 않았습니다. 재실행 필터는 `--tests '*LoginSessionFlowTest' --tests '*AuthHttpFlowTest'`입니다.

추가 `WatchlistFlowTest` 9 PASS, `PortfolioFlowTest` 5 PASS. 두 클래스는 실제 Controller→Service→entity/DTO 변환을 통과하고 Repository만 mock합니다. 포트폴리오 TransactionTemplate은 mock callback을 실제 실행합니다.

| 범위 | 입력·수행·독립 기대값 | 남은 경계 |
|---|---|---|
| 관심목록 정상 | 사용자7·목록10·종목20·항목30 fixture. 기본/명시 목록 조회, 양쪽 추가, note 변경, 삭제 → 소유자·관계 유지 및 올바른 DTO | HTTP binding·실제 SQL 없음 |
| 관심목록 접근 차단 | 목록 소유자를8로 바꿔 사용자7의 조회·추가·메모수정·삭제 시도 → 예외, 항목 조회/저장/삭제 차단·기존 note 불변 | 실제 보안필터와 결합한 status 검증은 별도 |
| 관심목록 예외 | 중복종목 → 저장0회. 기본목록 없음 → 사용자7로 생성. 생성 unique 오류 → 재조회 결과 사용. 미인증 → 저장소 접근0 | unique 오류는 주입했으며 실제 경쟁·트랜잭션 복구 증거 아님 |
| 포트폴리오 | 미존재 조회 → 빈 DTO·생성0. cash100.25·quantity2.5·공백 종목 → 소수 보존·trim·position0·양방향 관계. 기존 항목을 빈 목록으로 교체 → append 아닌 전체 교체 | JPA orphan removal·commit·실제 decimal 정밀도 별도 |
| 포트폴리오 접근 | 사용자7로만 조회/생성. 사용자 미존재·미인증 → 저장0 | 다른 사용자간 실제 DB 격리 검증 별도 |

재실행 필터: `--tests '*WatchlistFlowTest' --tests '*PortfolioFlowTest'`. 추가14개 Windows 실행 PASS(2026-09-25).

- Windows 소스 compileJava·compileTestJava PASS.
- Peer 역방향 검증 뒤 기본전체 재실행에서 `QAIMA_LIVE_PEER_SOURCE`도 해제했습니다. 아래17 SKIP에는 기존15개 외 Peer임시서버1개가 추가되며, 이host는 별도 opt-in실행에서PASS했습니다.
- 최신 전체 회귀 실행(2026-09-25): **1053개 중1002 PASS·34 FAIL·17 SKIP**, 약57초. `QAIMA_LIVE_MAIL`, `QAIMA_PROVISION_ISOLATED`, `QAIMA_LIVE_READONLY`, `QAIMA_LIVE_FASTAPI_TCP`, `QAIMA_LIVE_PEER_SOURCE`, `QAIMA_LIVE_KIS_READONLY`를 모두 해제하고 실행했습니다. FAIL은 F1 내부문구 공개1건·F3 가격/overlay 실패 환불누락2건·F1 리포트 필드 매핑1건·캔들 decode 오류 흡수2건·F2 거시 정상값 소실1건·뉴스캐시 재사용판정4건·Peer 최신일제외1건·재무half수정1건·재무절대값누락2건·CSV인용숫자1건·수급미등록종목3건·스냅샷수정본성장률1건·OpenDART회사명조각소실2건·SEC주식수범위초과1건·13F필수열누락2건/날짜보정1건·F3제공처표기1건/중복거래일2건/미래지수가용1건·동기화권한1건·진단내부문구1건·빈가격목록1건·산업캐시경고소실1건이며, TCP11·Peerhost1·실제 메일1·인프라2·DB생성1·KIS1은 이 기본 실행에서 의도적으로 SKIP입니다(TCP11은 별도 실제 실행 PASS). Gradle XML 합계로 tests1053/failures34/errors0/skipped17를 대조했습니다. 실패는 알려진 결함을 숨기지 않는 회귀 테스트로 유지합니다.
- 초기 단위·계약·보안 테스트 381개 PASS (369+5+4+2+1), 실패0.
- 확장 전체 실행: 398개 중397 PASS·1 FAIL. 실제 인프라2·JWT/cookie4·F1흐름5·사전HTTP6 포함. 실패는 내부 예외 문구가 Public 응답에 노출되는 `F1-ERROR-001`이며 [발견사항](../../../docs/findings.md)에 재현 근거를 기록했습니다.
- XML: `.runtime/build/test-results/test/TEST-*.xml`.
- HTML: `.runtime/build/reports/tests/test/index.html`.
- 이후 필터 실행은 이 결과 파일을 덮어쓸 수 있으므로 실행 범위와 날짜를 함께 기록합니다.
- 실제 인프라 실행 결과는 [전체 검증 현황](../../../docs/verification.md)에서 관리합니다.

## Swagger 전체 흐름 확대

### Windows 클라이언트 ↔ WSL 실제 uvicorn

`LiveFastApiTcpTest` opt-in11개가 실제 Windows→WSL TCP에서 모두 PASS했습니다. F1 재무·지표/설명생략·422, F2 빈키 경고/null metrics422, F3 공분산·제약·시장별fallback/422, 실제 CPU 뉴스 모델·확률을 확인했습니다. Public Feature3 HTTP 처리층→실제 TCP 계산→보고서/환불 경계도 연결했으며 DB·크레딧/보고서는 mock입니다. [11개 상세 방법과 실행 이력](cross-os-fastapi.md)에 임시 토큰·외부송신/.env 차단·독립 기대값·환경변수 오류 수정과 실패 이력을 기록합니다. 약2분7초/최종 audit0·서버종료, 기본 suite에서는11개 opt-in을 SKIP합니다.

### 진단·OAuth 진입·종목 메타/동기화·실제 권한 결합

2026-09-25 Windows 추가25개: `DiagnosticHttpFlowTest`8개 중7 PASS·1 FAIL, `TickerMetadataSyncFlowTest`14 PASS, `SyncSecurityFlowTest`3개 중2 PASS·1 FAIL. 마지막 미검증9개 경로의 실제 handler/client/codec/service를 연결했습니다. `SYNC-AUTH-001`은 실제 SecurityConfig/JwtAuthFilter→동기화 Controller에서 USER가200·provider/service에 진입하는 문제이며 저장 경계는 mock입니다. `DEBUG-ERROR-001`은 진단2경로의 내부문구 공개입니다. [25개 상세 기록](diagnostic-ticker-flow.md)에 테스트 fixture 수정 이력·입력/기대값·산업분류 초기화/페이지 미추적 관측·남은 연동을 기록했습니다. 전체123개 경로에 부분 업무 검증이 생겼지만 실제 연동/전체 흐름 완료를 뜻하지 않습니다.

### Feature3 가격·벤치마크

2026-09-25 Windows 추가29개: `Feature3PriceHttpFlowTest` 15개 중12 PASS·3 FAIL, `Feature3BenchmarkHttpFlowTest` 14개 중13 PASS·1 FAIL. 실제 Controller/service/Yahoo JSON 파서·품질 판정/CandleLoadService/지수 bar 매핑까지 연결하고 전송·저장소·달력·종목 조회는 대체했습니다. 원본 보존/전체 fallback·보강 전후 조회·지수 원래값·입력/오류를 검사했습니다. 제공처 KIS 고정 표기1·중복 거래일2·미래 지수 가용판정1 실패를 유지하며, 최초 테스트의 응답 필드명 오류 수정도 [29개 상세 기록](feature3-market-data.md)에 공개합니다. 실제 API/DB·JWT 결합·FastAPI/브라우저 연결은 남아 있습니다.

### SEC 13F 파일·CUSIP·분기 집계

2026-09-25 Windows 추가40개: `Sec13fParserTest`9개(7 PASS·2 FAIL), `Sec13fAdminFlowTest`18개(17 PASS·1 FAIL), `Sec13fAggregateFlowTest`13 PASS. 관리자5경로→실제ZIP/TSV/service/계산/DTO, Repository·native SQL projection만mock입니다. 필수열누락SUCCESS2개·불가능한날짜보정1개실패를유지합니다. [40개상세방법](sec-13f.md)에파일생성/재실행·배치/매핑·정정영향·산술·mock한계를기록했습니다. 기관별최신/정정공시를선택하는native SQL은아직실제DB에서검증하지않았습니다.

### SEC 발행주식수·미국 종목 마스터

2026-09-25 Windows 추가50개: `SecCompanyFactsTest`13개(12 PASS·1 FAIL), `SecIssuedSharesFlowTest`17 PASS, `UsStockMasterFlowTest`16 PASS, `SecIssuedSharesSchedulerTest`4 PASS. 관리자3경로→실제service/client/parser/codec→메모리저장소/tx callback을검증했습니다. 정수범위초과가1주로변환되는 `SEC-SHARES-OVERFLOW-001` 회귀실패1개(숫자/문자열2sub-assertion)를유지합니다. [50개상세방법](sec-issued-master.md)에입력/기대값·mock경계·저장실패/집계관측을기록했습니다. SEC13F의5경로는별도40개로확대했으며실제SEC/SQL/JWT/cron검증은남아있습니다.

### OpenDART 회사코드·발행주식수·일일 동기화

2026-09-25 Windows 추가46개: `OpenDartClientTest`16개(14 PASS·2 FAIL), `OpenDartAdminFlowTest`25 PASS, `OpenDartSyncSchedulerTest`5 PASS. 관리자4경로→실제서비스/client/codec·XML→메모리저장소→DTO를검증했습니다. 회사명entity/CDATA조각을덮어쓰는 `OPENDART-XML-TEXT-001` 실패2개를유지합니다. 별도curl fallback은테스트JVM에서프로세스시작전에차단했습니다. [46개 상세방법](opendart.md)에입력·기대값·안전장치·집계/중첩실행관측·한계를기록했습니다. 실제OpenDART/SQL/cron까지완료한것은아닙니다.

### 시장 스냅샷 관리자·주식수 기준

2026-09-25 Windows 추가37개: `MarketSnapshotAdminFlowTest` 27개(26 PASS·1 FAIL), `ShareBasisResolverTest` 10 PASS. 실제 관리자2경로→backfill/service→주식수 resolver/재무계산기→tx callback→캐시 JSON→DTO를 연결했습니다. Repository/KIS/Redis/tx 관리자는 대체했습니다. 같은연도의재무수정본을전년처럼비교하는 `SNAPSHOT-GROWTH-VERSION-001` 실패를유지합니다. [스냅샷 상세 기록](market-snapshot.md)에37개 각각의방법·기대값·결과·한계를기록했습니다. 실제DB/제공처·공개시세통합/브라우저까지의완료는아닙니다.

### 종목·시장 수급 API·배치·공유 scheduler

2026-09-25 Windows 추가50개: `InvestorFlowAdminFlowTest` 30개(27 PASS·3 FAIL), `InvestorFlowBatchTest` 16 PASS, `KisBatchSchedulerTest` 4 PASS. 실제 관리자5경로→service→KIS client/codec·합성전송·저장객체·응답, 별도 배치 대상/날짜/재시도/집계·rate limiter·중첩실행 guard를 검증했습니다. 미등록종목3경로는 의도된오류대신200/빈본문 또는 빈rows 성공응답이되어 `INVESTOR-STOCK-MISSING-001` 실패회귀를 유지합니다. [수급 상세 테스트 기록](investor-flow.md)에50개 메서드 각각의 방법·기대값·실제/mock 경계·재현명령을 기록했습니다. 실제API/DB/cron 실행 완료는 아닙니다.

### FINRA·KRX 공매도 수집·스케줄러

2026-09-25 Windows에서 `ShortSellingAdminFlowTest` 32개와 `ShortSellingSchedulerTest` 8개, **40 PASS**. 관리자5경로의 실제 Controller→service→provider client/codec→SQL 인자 생성, 오류/부분커밋, 두 스케줄러의 시간대·중첩 실행 방지를 검사했습니다. 외부 HTTP 전송·Repository/JDBC/트랜잭션 관리자는 대체했고 실제 데이터 변경은 없습니다. [공매도 상세 테스트 기록](short-selling.md)에40개 각각의 입력·방법·기대값·관측·재현명령·한계를 기록했습니다. Swagger는 PARTIAL이며 실제연동 완료가 아닙니다.

### 사용자·크레딧 HTTP와 캐시 정책 추가 검증

2026-09-25 Windows 실행 결과 **28 PASS**: UserHttpFlowTest8·CreditHttpFlowTest6·OverlayCachePolicyTest9·MarketSnapshotCacheTest5. 최신 전체1053개 실행에도 포함되므로 두 숫자를 중복 합산하지 않습니다.

`UserHttpFlowTest`는 실제 UserController→UserService→DTO/예외처리를, `CreditHttpFlowTest`는 실제 Credit/CreditAdminController→CreditService→원장 DTO를 WebTestClient로 실행합니다. test WebFilter로 principal7을 주입하므로 실제 보안필터·관리자 권한 증거는 기존 ApiSecurityMatrixTest와 구분합니다. Repository만 mock이며, 크레딧 TransactionTemplate은 callback을 실행합니다. 실제 사용자 정보·잔액·원장은 변경하지 않습니다.

| 클래스·항목 | 입력·수행과 기대값 |
|---|---|
| User: 본인 조회 | principal7 조회, userId7·email 반환, passwordHash 비공개 |
| User: 프로필 수정 | name trim·phone 숫자정규화·glossaryHover 반영. legacy experience와 명시 investmentLevel이 함께 오면 명시 수준 우선 |
| User: 공백·중복 | 공백 이름/전화는 기존 값 유지, 다른 사용자 전화번호와 정규화 후 중복되면400·저장0 |
| User: 위험 점수 | 0/0.2/0.2001/0.4/0.6/0.8/1 → 5개 라벨 경계·정확한 BigDecimal 보존. -0.01/1.01/누락 →400·저장소 접근0. 필드명 defaultRiskGamma의 저장 범위는0~1이며 분석용 γ1~10과 구분 |
| User: 소셜 추가정보 | profile_required→active, 7자리/성별·전화 정규화. 반복완료·잘못된 성별자리·필수누락 거부, 실패 시 저장0 |
| Credit: 조회 | creditBalance5·null→0, 쓰기0. ledger limit0/1000 →PageRequest 크기1/100 |
| Credit: 임시충전 | amount3 →잔액5→8, typeCHARGE·TEMP_FRONTEND_CHARGE·principal7 원장. amount0/-1/누락은 validation400·저장소 접근0 |
| Credit: 관리자 | charge3→8·ADMIN_CHARGE, adjust-2→6·ADJUST, reason120자→100자. 초과차감402/INSUFFICIENT_CREDIT, 음수charge/0adjust400·저장0 |

초기 잔액 응답 테스트가 `balance`를 가정하여 실패했습니다. 실제 DTO 계약은 `creditBalance`임을 확인하고 테스트 JSONPath만 수정했습니다. 제품 응답을 바꾸거나 결함 테스트를 숨긴 것이 아닙니다.

`OverlayCachePolicyTest`는 실제 Feature3OverlayService와 JSON 직렬화·캐시 판정·예상비용 계산을 실행합니다. Redis value operation과 Feature1/2 원천 service만 mock합니다. 모든 preview fixture마다 같은 holdings/options를 분석 요청 DTO로 구성해 `estimateCredit`와 금액 일치를 비교합니다.

- core만 선택:1credit·캐시 조회0.
- 두 종목·fundamentals/technical 모두 miss:종목별이 아닌 overlay별1씩, 총3.
- 두 종목 모두 hit:총1, 가장 오래된 cache 시각을 KST로 표시.
- 한 종목만 miss:해당 overlay 비용1, 총2.
- hit 상태 FORCE_REFRESH:두 overlay 총3. technical만 REUSE_AVAILABLE override:총2.
- industry 정보 없음+FORCE_REFRESH:총1, 추가금액0·UNAVAILABLE·정책선택 불가.
- Redis GET 오류·깨진 JSON:miss로 처리해 예측 가능한 비용 응답.
- Feature1 metrics 저장:정규화 key·fresh24시간/stale7일 TTL, SET 실패는 false로 반환해 핵심 오류로 전파하지 않음.
- 실제 preview HTTP:빈 selectedOverlays는 총1, 필수필드 누락400. 분석·차감·제공처 service 호출0.

예상 비용과 preview의 동일성은 실제 실행 시점의 cache 변화나 동시성까지 증명하지 않습니다. news/correlation cache와 전체 overlay 실행·실차감 연결은 아직 남았습니다.

`MarketSnapshotCacheTest`는 실제 cache service·ObjectMapper와 모의 Redis/Repository로 다음을 검사합니다.

- miss→기준일 이하 최신 DB 조회→두 캐시 SET:TTL1시간/60초, 주식수1000·유통비율40%·자기주식10%→400/100, warning 중복 제거.
- 유효 latest hit→DB 조회0.
- 기준일보다 미래인 캐시→무시하고 기준일 이하 DB 조회.
- GET/SET 모두 오류→DB 결과 보존.
- 깨진 JSON+DB 없음→empty, SET0.

실제 Redis TTL 만료·서버 장애 복구·DB SQL/격리는 별도입니다. 추가 묶음 실행 필터:

```powershell
--tests '*UserHttpFlowTest' --tests '*CreditHttpFlowTest' --tests '*OverlayCachePolicyTest' --tests '*MarketSnapshotCacheTest'
```

[123개 작업 추적표](swagger-coverage.md)에 Controller·직접 주입 의존성·비즈니스 흐름·실제 연동 상태를 기록합니다. `tools/swagger_inventory.py`는 정적 검토 보조 목록을 만들며 reflection 검증을 대신하지 않습니다.

남은 필수 검증은 각 Controller의 파라미터·DTO 검증, Service→Repository/외부 API의 정상·오류 분기, 소유권·동시성, 분석 실패 환불·리포트 저장실패의 응답 보존, 캐시 hit/miss/장애, 외부 timeout·retry, 관리자 동기화·ETL 멱등성입니다. 381 PASS는 이 남은 항목의 완료 선언이 아닙니다.

실제 비용 호출은 사용자 허용 한도를 보수적으로 적용하여 세션 전체 재시도 포함5회 이내로 관리합니다. 최신 소비는 [호출 기록](../../../docs/verification.md)을 따릅니다.

## 종목·재무·차트 조회 흐름

2026-09-25 추가 **37개 중35 PASS·2 FAIL**: StockHttpFlowTest14 PASS, FinancialHttpFlowTest10 PASS, ChartCandleFlowTest13개 중11 PASS·2 FAIL. 실패는 `CANDLE-DECODE-001`의 차트 HTTP·Feature3 로더 두 경로이며 제품 코드는 미수정입니다.

기본 Windows Gradle 명령의 필터:

```powershell
--tests '*StockHttpFlowTest' --tests '*FinancialHttpFlowTest' --tests '*ChartCandleFlowTest'
```

### 종목 검색·매핑·시세 — StockHttpFlowTest

실제 StockController→StockService/StockMappingService→DTO/advice를 WebTestClient로 실행합니다. Repository·StockClient·MarketSnapshotService와 등록 관련 resolver는 mock입니다. 실제 스냅샷 계산·금융 API·SQL은 포함하지 않습니다.

| 검사 | 입력·독립 기대값·흐름 |
|---|---|
| 빈 검색 | q누락/공백→빈 배열·종목/별칭 저장소 접근0 |
| 통합 검색 | ticker1개→alias중복/다른종목/null→이름 후보27개 합성 fixture. ticker→alias→name 순서, stockId 중복 제거,20개 제한·symbol 구성 |
| 법인명 | `（주） 합성기업`→정규화된 동명 종목, 후보 query pageSize200, 정확한 이름 일치 시 alias 접근0 |
| 단일 매핑 오류 | 동명2개→400, 미존재→404, name/symbol 모두 누락→400 |
| 별칭·거래소 | alias의 KOSPI/NASDAQ·중복 엔티티에 exchange=xnas→NASDAQ만1개 |
| symbol 문법 | xnas:QA0002와 QA0002.XNAS→NASDAQ:QA0002, name보다 symbol 우선. bare/불완전 symbol→400 |
| 넓은 KRX 필터 | 동일 코드2개 응답: 단일 normalize는400·candidates는2개 반환. 미등록 symbol candidates는빈 배열, 단일은404 |
| 이름 후보 | 정확명/alias 없으면 substring 후보24개 중앞20개 |
| 시세 DTO | ID와 소문자 code.KS로 조회→가격123.45·등락률-1.25·currency·Public 제한필드, save0 |
| 시세 실패 | `5930`→`005930`, provider 오류→정적 종목정보·price null·save0 |
| 미등록 종목 | code조회404·ID조회400, provider/save0. 메서드명과 달리 현재 Public 경로는 신규 종목 생성 안 함 |
| snapshot | stock 해석→요청 asOfDate 전달→DTO 날짜값·PER12.5, Mono.empty면200/data null. 계산 service는 mock |
| binding 오류 | 잘못된 날짜/ID→400·하위 경계 접근0 |

초기 blank query fixture의 `%20`이 재인코딩되어 실제 공백으로 전달되지 않았습니다. URI 변수 인코딩으로 수정했습니다. standalone WebTestClient 기본 serializer에서 LocalDate는 `[2026,9,24]`로 반환됐으므로 snapshot 날짜는 typed DTO로 deserialize하여 값 일치를 검사합니다. 실제 Spring Boot 실행의 날짜 JSON 문자열 계약을 검증했다고 주장하지 않습니다.

### 재무 조회·지표 — FinancialHttpFlowTest

실제 FinancialController→FinancialReadService→FinancialMapper→DTO/advice를 사용하고 Stock/Financial Repository·RealtimePriceService·ShareBasisResolver만 mock입니다. 고정 합성 재무: 매출1000·영업이익200·순이익80·자산800·부채400·자본400, 유동자산300·유동부채100·재고50·이자20·영업현금120·CAPEX20+10, 가격10·발행주식100·평가주식80.

| 검사 | 기대값·방법 |
|---|---|
| 연간 핵심 지표 | 시총1000=10×100, EPS1=80/80, BPS5=400/80, PER10, PBR2, PSR1. 마진20%/8%, ROE20%, ROA10%, 부채비율100%, 유동300%, 당좌250%, 이자보상10, FCF90. HTTP JSON값 검증 |
| 조회 기간 | 기본 A/5년→현재연도-4~현재연도, asOf2026/years2→2025~2026. 연도경로2025/Q3→해당 연도·periodNo로만 query |
| Q/H/TTM | Q의 quarter, H의 half 매핑. Q/H EPS·ROE/PER 등 연간 지표는 null, TTM EPS/PER 계산 |
| 전년 성장 | 매출1000→1200은20%; EPS는80/100=.8→120/80=1.5이므로87.5%. 단순 순이익 성장50%와 구분하고 각 보고일 주식 수를 적용 |
| 비교 불가 | 전년 다른 분기는 비교하지 않음; 전년 매출/순이익0이면 성장률 null |
| 시세 장애/빈 값 | 재무·EPS 유지, marketCap/PER null. 가짜 가격을 보충하지 않음 |
| 분모/주식수 | 분모0·주식수없음→해당 지표 null, NaN/Infinity 미생성 |
| 검증 오류 | years0/-1, Q0/5·H3·A0·TTM1, 잘못된 enum/int/date/year→400·데이터 경계 접근0 |
| 없는 종목 |404, 시세/재무/주식수 서비스 미호출 |
| 보고일 없음 | 주식 수 조회에 요청 asOfDate 사용 |

기간별 SQL 필터·정렬은 mock의 전달 인자로 검사했으며 실제 JPQL 결과·보고시점 가용성·음수 분모 해석·전체 metric rounding 경계는 미완료입니다. 조회일이 과거여도 현재 시세를 사용하는 구현의 시점 의미는 별도 검토 대상입니다.

### 차트·캔들 저장 경로 — ChartCandleFlowTest

실제 ChartController→ChartService→CandleLoadService→CandleMapper를 사용하고 StockService·PriceOhlcvRepository·StockClient·TradingCalendarService는 mock입니다. 기준 거래일은2025-01-03, from은2025-01-01 KST로 고정해 실행 시각에 따른 freshness 차이를 제거했습니다. 현재 일봉16시 확정 경계·미국 거래일 정책을 검증한 것은 아닙니다.

| 검사 | 방법·기대값 |
|---|---|
| 최신 DB hit | 역순2행→외부 미호출·오름차순·KST 자정 epoch초·volume12.9→12. 국내일봉 DB 조회 from은9시간 확장 |
| 외부 merge | 기존1/2 close102, 외부1/2 close999·1/3 close103→화면999/103·신규1/3만 saveAll. 기존 객체102 불변, 저장 timestamp KST 자정. 응답 덮어쓰기와 DB 원본 보존을 구분 |
| global fallback | MARKETSTACK 응답→source와 CHART_FALLBACK_TO_GLOBAL warning |
| 예상 실패 | KIS HTTP/BIZ/MARKET_CLOSED→200·EMPTY·NO_DATA, save0 |
| 디코딩 실패 | KIS_DECODE_ERROR는 명시된 전파 분기/502 오류 계약을 기대하나200·EMPTY/NO_DATA로 바뀌어 **FAIL** |
| before DB hit | limit2면 from/range 경로 무시, DB PageRequest0/2·오름차순·provider0 |
| before DB miss | limit2→to-400일 외부조회→saveAll→DB 재조회2회째에서 slice·오름차순 반환 |
| before limit0/-1 |200·NO_DATA, 캔들 repo/provider 접근0. 종목 조회는 그 전에 수행됨 |
| Feature3 최소표본 | 최신1행/required2→외부 시도·HTTP 오류 시 기존 DB1행 보존. 최신2행/required2→외부0 |
| Feature3 디코딩 | 위 부족표본 조건에서 KIS_DECODE_ERROR가 throw되지 않고 DB fallback으로 흡수되어 **FAIL** |
| 입력/DB 오류 | 필수 query누락·enum/date오류→400·종목 조회0. DB 조회 예외는5xx·외부0 |

초기 차트 HTTP fixture에서 URI query의 `+09:00`을 그대로 넣어 `+`가 공백으로 해석되면서 binding400이 발생했습니다. URI template 변수로 올바르게 인코딩한 후 의도한 서비스 흐름을 검증했으며, 디코딩 결함2개만 실패로 남았습니다. 테스트 코드의 Iterable captor에는 unchecked 컴파일 경고가 있으나 실행 결과와 별도입니다. 실제 원본 DB 보존·저장 중복/경쟁·provider payload parsing·before limit상한·모든 frequency/시간대 검증은 남아 있습니다.

## 이메일·리포트 HTTP와 실제 메일 흐름 테스트

### 메일·비밀번호 재설정 HTTP 회귀 — 실제 발송 없음

`EmailHttpFlowTest` **14 PASS**(2026-09-25). 실제 EmailController→MailAuthService/AuthLoginLogService→DTO validation/GlobalExceptionHandler를 WebTestClient로 호출합니다. MIME 생성·SecureRandom·SHA-256·BCrypt(cost4)는 실제 코드이며 JavaMailSender와 모든 Repository는 mock입니다. 합성 주소만 사용하고 SMTP·DB·실제 계정에 접근하지 않습니다. Controller에 bind한 HTTP 처리층 테스트이며 TCP 서버·SecurityConfig·트랜잭션 proxy는 시작하지 않습니다.

| 테스트·입력 | 과정·기대값 |
|---|---|
| 가입 전 인증 | 대문자 합성 email로 request→MIME6자리 코드 추출→저장 hash를 독립 SHA-256과 대조→confirm→consume. 만료600초·소문자 email·미사용 이전토큰 삭제, confirm 전 consume 거부·후 deleteByEmail 확인 |
| 기존 미인증 계정 | request→confirm 후 emailVerified=true·emailVerifiedAt·token.user 연결 확인. 실제 JPA dirty checking은 미검증 |
| 이미 인증·60초 제한 | 인증된 계정의 request/confirm은200·발송/인증저장소 접근0. 최근 인증 요청 존재 시200·새 토큰 저장/발송0 |
| 잘못된 인증코드 | 생성 범위 밖 `000000`, usedAt 존재,1초 전 만료,expiresAt=null 각각400. 성공 상태로 전환하지 않고 실패 감사코드 기록 |
| consume·정리 | confirm된 토큰도 만료됐으면 consume 거부·삭제0. cleanup 직접 호출은 현재시각 cutoff로 삭제 요청. scheduler는 실행하지 않음 |
| 비밀번호 JSON 정상 흐름 | request→MIME링크의 URL-safe token 추출→Base64디코드32bytes·SHA-256·TTL1800초→confirm→BCrypt matches·usedAt→동일 토큰 재사용400·해시 불변 |
| reset 요청 은닉·제한 | 미가입 email 및 최근 발급 계정은 둘 다200·발송0. 전자는 reset 저장소 접근0, 후자는 저장0. 시간차 기반 enumeration까지 증명하지 않음 |
| reset 실패 | 미등록 token·7자리 비밀번호·만료·null 만료 각각400, 기존 passwordHash·usedAt 불변 |
| HTML form | token의 `<`, `>`, 쌍따옴표·작은따옴표·`&`를 entity escaping, text/html 및 confirm-form action 확인. token query 누락400 |
| form 제출 | 앞뒤 공백을 넣은 token/password 제출→trim된 비밀번호로 BCrypt matches·완료 HTML. token/password 각각 누락400. JSON 경로와 공백 처리 의미가 다름 |
| DTO validation |4개 JSON endpoint의 빈 body, 잘못된 email→400·서비스 Repository/SMTP/감사 접근0 |
| SMTP 실패 | mock send에 합성 내부 예외→500·일반화된 “메일 발송에 실패했습니다.”만 공개·실패 감사1건. 실제 DB rollback은 증명하지 않음 |
| 감사 실패 | audit repository 예외에도 요청/정상confirm200, 잘못된confirm400 유지 |
| dry-run·example.com | 둘 다 토큰 저장은 수행하고 JavaMailSender 접근0 |

정상 요청의 X-Forwarded-For 첫 값은 합성 문서용 IP,600자 User-Agent는512자로 제한됨을 확인했습니다. 이 검사는 프록시 헤더 신뢰 정책을 보안 승인하는 근거가 아닙니다. 실제 발송·수신 결과는 다음 opt-in 검사와 구분합니다. 토큰 충돌·동시 요청·DB 유일성·rollback·재설정 후 기존 로그인 세션 정책은 아직 남아 있습니다.

### 리포트 저장·조회 HTTP 회귀

`ReportHttpFlowTest` **12개 중11 PASS·1 FAIL**(2026-09-25). 실제 AnalysisReportService·ObjectMapper·Controller·DTO/advice를 사용하며 저장소는 mock, 인증은 test WebFilter의 principal7입니다. 실제 DB SQL·보존기간 삭제·소유권 저장소 조건의 실행을 대체하지 않습니다.

| 테스트 | 방법·기대값 |
|---|---|
| snapshot 저장→조회 | create→saveAndFlush 객체 캡처→원본 request와 사용자 이름 변경→GET detail. 저장시점 request/userName 유지·수익률0.05·warnings 복원·camelCase. 내부 user entity/raw JSON 필드 미노출 |
| 기본값 | 빈 feature/subject/title/userName→FEATURE/STOCK/분석 리포트/사용자, generatedAt 자동생성, null request/result→`{}`, null warnings→null |
| 저장 실패 | 사용자 없음→UNAUTHORIZED, getter가 예외를 내는 합성 payload→직렬화 오류. 둘 다 saveAndFlush0 |
| 목록·보존기간 | page=-9,size999→0/50. delete와 사용자별 조회에 동일한 현재시각-7일 cutoff 전달, summary에서 snapshot 미노출 |
| 필터 | ` FEATURE2 `→trim, page2/size0→2/1, feature별 조회에만 전달·빈 배열 응답 |
| 권한·입력 | 미인증/Long이 아닌 principal→401; 잘못된 ID/page→400·Repository 접근0. 없는/타인 리포트에 해당하는 scoped 조회 miss→404, unscoped findById0 |
| 손상 JSON | 잘못된 저장 JSON→빈 객체, 공백/null→null. HTTP 응답 원문을 ObjectMapper로 다시 파싱해 구분 |
| Feature1 저장 매핑 | 정상 request/metrics의 QA0001과 조회한 합성기업으로 create→저장 객체·GET detail 검증. **코드/회사명 뒤바뀜 FAIL: REPORT-MAPPING-001** |
| Feature2 저장 매핑 | metrics 없는 응답에서도 request stockCode+종목 lookup,120거래일 window·warnings 별도 JSON 보존 |
| Feature3 저장 매핑 |2종목·Expert·BALANCED·140일·ADJUSTED_CLOSE fixture→포트폴리오 요약·위험/가격/모델 metadata 보존 |
| 전체 정리 | scheduler 없이 deleteExpiredReports 직접 호출→7일 cutoff 전달·삭제건수3 반환, deleteAll 미호출 |

초기 손상 JSON 검사는 JsonPath `isEqualTo(Map.of())`의 비교에서 실패했습니다. HTTP 원문을 다시 파싱해 실제 빈 객체와 null을 각각 확인하도록 assertion을 변경한 뒤 통과했습니다. 제품의 JSON fallback을 변경하거나 기대값을 null로 완화한 것이 아닙니다.

추가26개 실행 방법은 기본 Gradle 명령에 다음 필터를 붙입니다. Feature1 필드 매핑 결함은 실패로 유지하며 skip하지 않습니다.

```powershell
--tests '*EmailHttpFlowTest' --tests '*ReportHttpFlowTest'
```

### 실제 SMTP opt-in 결과

`LiveMailFlowTest`는 `QAIMA_LIVE_MAIL=1`, 명시적 `QAIMA_TEST_EMAIL`, `QAIMA_MAIL_ATTEMPT=1..5`가 있어야 실행됩니다. 수신 주소·credential·인증번호는 공개 문서·콘솔에 출력하지 않습니다. `src/test/.runtime/external-call-N.reserved` 파일을 CREATE_NEW로 생성하므로 같은 번호로 재실행해 중복 발송할 수 없습니다.

YAML에서 읽은 SMTP credential을 실제 JavaMailSender에 적용하고, 실제 `MailAuthService.requestEmailVerificationCode`를 호출합니다. Repository만 메모리 mock이며 기존 DB의 가입·인증 상태를 변경하지 않습니다. 발송 성공 시 MIME 본문의 인증번호가 저장된 SHA-256과 일치하는지, confirm으로 usedAt이 기록되는지, signup consume이 토큰 삭제를 요청하는지 확인합니다. SMTP 성공은 실제 수신함 도착 증거가 아니므로 사용자 확인을 별도로 받아야 합니다.

첫 실제 발송은 service의 일반화된 메일 발송 실패 예외로 종료됐습니다. 2번 슬롯에서 오류 유형만 출력하여 TLS 인증서 신뢰 경로 오류를 확인했습니다. 테스트 발송기에서 누락한 YAML SMTP 속성을 모두 반영한3번 슬롯은 발송·해시·확인·소비가 PASS했습니다. 운영 설정·JVM truststore는 변경하지 않았고 사용자가 수신을 확인했습니다. 자동 반복·실패를 PASS 처리하는 fallback은 사용하지 않습니다.

## 사전 관리자·검색 전체 경로와 종목 순위 캐시

2026-09-25 추가 **26 PASS**: DictionaryWorkflowTest15·RankingCacheFlowTest11. 최신 전체1053개에도 포함됩니다. 기본 Windows Gradle 명령에 다음 필터로 재현합니다.

```powershell
--tests '*DictionaryWorkflowTest' --tests '*RankingCacheFlowTest'
```

### DictionaryWorkflowTest — 15개

실제 DictionaryController/DictionaryAdminController→DictionaryService→DTO/advice를 standalone WebTestClient로 호출합니다. Repository는 HashMap 상태를 공유하는 mock입니다. 관리자 권한 필터는 이 클래스에 없으며 이전 ApiSecurityMatrixTest의 별도 증거와 구분합니다. 실제 관리자 데이터는 변경하지 않습니다.

| 검사 | 입력·과정·기대값 |
|---|---|
| 용어 CRUD | ` capital   ratio ` PUT→CAPITAL RATIO 키·설명 trim·PUBLISHED, 재PUT은 동일 엔티티 수정·생략한 영문설명 null, GET 변경내용 확인→DELETE→GET404 |
| 메타데이터 | source 공백→null, org/URL/type/tag trim, draft→DRAFT, reviewedAt ISO 문자열→정확한 Instant |
| 잘못된 입력 | 필수필드 누락/공백·잘못된 reviewedAt→400·Repository 접근0 |
| 용어/별칭 충돌 | 기존 alias와 같은 정규화 용어 PUT→400·용어 저장0 |
| 별칭 CRUD·재지정 | 앞뒤 공백 제거·표시문자 보존, canonical 변경 시 동일 alias 엔티티 사용, alias로 GET하면 변경된 canonical 반환→정규화 alias DELETE |
| 별칭 제약 | canonical 자기참조·다른 canonical 키와 충돌→400, 없는 canonical→404, 필수필드 누락→400·별칭 저장0 |
| 삭제 오류 | 없는 용어/별칭404, alias query누락400, delete 요청0 |
| 조회 우선순위·영문명 | 비정상 중복 fixture에서 canonical 우선. planned_alias/영문 notes를 우선하고 그중 긴 영문명 선택, underscore·한글 혼합은 영문표시명 후보에서 제외. aliases 배열에는 원자료 유지 |
| 별칭 목록 | alias로 canonical 해석 후 해당 대표 용어의 alias/sourceType 응답 |
| 검색 LIKE 인자 | ` a%_\b `→대문자, `%`·`_`·`\` escaping된 prefix/contains를 repository에 전달, initial alpha→A, page-4/size999→0/200 |
| 초성/기본목록 | initial 가→ㄱ·size0→1, punctuation initial은필터없음·기본page0/size50 |
| 자동완성 우선순위 | canonical CAP/CAPA/CAPZ/SCAP와 alias cap/capa/CAPB/XCAP→대소문자 중복 제거 후 `[CAP,CAPA,CAPZ,CAPB,SCAP,XCAP]` 정확한 순서. 기본 후보 조회40개 |
| 자동완성 제한 | size999→반환30·후보pool120, size0→반환1, 공백query400 |
| 초성집계·빈목록 | 합성 ㄱ3/A2 projection 응답, 빈 일반검색은 alias batch lookup0 |
| 엔티티 callback | 실제 private lifecycle 메서드를 직접 호출하여 초성ㄱ·기본status·CAPITAL RATIO 정규화 확인. JPA 자동호출/유일성 실행 증거 아님 |

처음에는 테스트의 이종 Map 리스트 제네릭 추론으로 컴파일이 실패해 명시적 타입을 지정했습니다. 이어 자동완성 검사에서 JsonPath `isEqualTo(List.of(...))`의 타입 변환 비교가 실패했고, 실제 배열을 `.value`로 받아 독립 기대 리스트와 직접 비교한 후 통과했습니다. 정렬 기대값·제품 코드는 변경하지 않았습니다.

남은 범위: 실제 JPQL LIKE/collation/정렬·페이지 결과, alias FK 삭제 규칙, 동시 upsert/충돌, JPA callback·timestamp, SQL transaction, 실제 관리자 JWT와 Controller 결합, 사전 내용·출처 정확성. PUBLISHED/DRAFT의 공개 정책 전체를 승인한 검증이 아닙니다.

### RankingCacheFlowTest — 11개

실제 RankingStockController→TopRankingReader→ObjectMapper를 실행하며 KrStockClient·StockRepository·Redis value operations·TradingCalendarService는 mock입니다. 시간 경계는 LocalDateTime.now(Asia/Seoul)만 scoped static mock으로 고정합니다. cachePolicy는 fetch 호출 시 동기 계산되므로 시계 mock을 worker thread로 전파했다고 가정하지 않습니다. HTTP 케이스는 현재 시각에서도 cache 경로가 활성화되도록 비거래일 calendar 응답을 사용합니다.

| 검사 | 기대값·근거 |
|---|---|
| cache hit | `ranking-stocks:20260925:GAINERS:30`의 유효 JSON→원결과, provider Mono 구독0·stock DB0·SET0. 메서드 호출과 실제 구독을 분리해 카운트 |
| cache miss·지원종목 | EQUITY·ETF·미등록·null코드4개→등록 EQUITY1개만, stockId7·KOSPI 보강·가격123.45 보존. limit0→1, 장중TTL10초 |
| read 장애/손상 | Redis GET 예외·잘못된 JSON→각각 fresh fetch, 총 구독2 |
| write 장애/빈 hit | SET 예외에도 fresh 결과 유지, 이후 `[]` cache는 유효한 빈결과라 추가 provider 구독0 |
| 장전 bypass | 거래일08:25:00·08:29:59는 Redis 접근0·각 fresh 구독 |
| 장전 직전 |08:24:59는 이전 거래일 key·TTL1초 |
| 장중 양끝 |08:30:00·17:59:59는 당일 key·TTL10초 |
| 장후 | 금요일18:00:00→당일key, 다음거래일 월요일08:25까지TTL62시간25분 |
| 주말 | 토요일10:00→이전 금요일key, 월요일08:25까지TTL46시간25분 |
| provider/DB 실패 | 둘 다 실제 reader에서 예외전파, 성공결과로 캐싱하지 않음(SET0) |
| HTTP | GAINERS/LOSERS/NEAR_NEW_HIGH/NEAR_NEW_LOW/TOP_TURNOVER/VOLUME_SURGE6종 모두camelCase·limit999→30. topic누락/알수없는값/limit문자400·하위경계 접근0 |

캐시 hit에서도 구현은 fallback Mono를 만들기 위해 provider 메서드를 호출합니다. 이 테스트는 `Mono.defer` 내부의 구독 카운터로 실제 fetch 실행 여부를 검증하며, 단순 Mockito 호출 횟수0을 기대하지 않습니다. 실제 KrStockClient의 요청 조립 부수효과·금융 응답 파싱·Redis TTL 만료·동시 cache stampede·실제 휴장일 데이터는 별도입니다. 추가 테스트의 generic Redis mock에는 unchecked 컴파일 경고가 있으며 제품 경고와 구분합니다. 실제 금융/Redis/DB 호출·비용은0입니다.

## Feature2 카드9경로: HTTP·조립·부분 실패

`Feature2CardFlowTest` 18개 중17 PASS·1 FAIL(2026-09-25 Windows). 실제 `Feature2CardController`→`Feature2CardService`→DTO/envelope를 WebTestClient로 연결하고 GlobalExceptionHandler를 적용합니다. Repository, 종목/산업 resolver, BaseRate/ShortSellingFeatureService, IndustryIndexService, 각 금융 동기화 service, PriceSnapshotReader는 Mockito 대체입니다. Spring 애플리케이션·스케줄러·실제 TCP 서버·보안필터·DB·Redis·금융 API는 시작하지 않습니다. 기존123×3 보안 matrix와 이 업무 검증이 실제 필터+업무 결합 검증을 대신하지는 않습니다.

합성 종목 QA0001·산업8·날짜2026-09-24를 사용합니다. 엔티티를 직접 만들되 영속화하지 않으며 Repository는 정렬되지 않은 목록을 반환하여 서비스의 정렬·합계·선택을 실제 실행합니다. WebTestClient 기본 날짜 직렬화는 제품 Boot 설정 전체와 다를 수 있어 날짜 문자열 wire 계약은 주장하지 않습니다. 시계열 날짜 정렬 일부는 typed DTO에서 별도로 비교합니다.

| 메서드 | 입력·수행 및 기대값 | 결과 |
|---|---|---|
| `baseRateMapsMetricsAndWarningWithoutSnakeCaseLeak` | 금리2.75·KR·합성 stale warning → camelCase countryCode, meta.warnings 전달 | PASS |
| `baseRateEmptyAndReactiveFailureReturnNullCard` | 금리 source empty/error → HTTP200 data=null. 현재 fallback 동작을 기록하며 충분한 warning 제공을 보증하지 않음 | PASS |
| `baseRateSeriesSortsAscendingAndClampsRepositoryPage` | 역순3/2.75 → 오름차순2.75/3, limit9999→1095·0→1, 동기화 오류 시 빈목록·Repository 미접근 | PASS |
| `shortSellingMetricsAndSortedSeriesPreserveValuesAndCap252` | 비율12.5·이전일 금액비율4.25·거래량1000 → 필드 유지·날짜 정렬, limit9999→252·-1→1 | PASS |
| `missingStocksAndBlankCodesSkipDownstreamReads` | 종목 미존재/빈코드로5경로 호출 → 단일값null/목록[], 빈코드는 resolver도 미접근, 하위 조회0 | PASS |
| `industryDefaultsAndExplicitWindowReachIndexService` | 산업8·지수12, window-1→120·명시21 보존, ONE_D 전달, 지수 오류 시 data=null | PASS |
| `investorFlowSumsNullableFieldsSortsAndUsesKspMarket` | 두 날짜 외국인10/null·기관-4/2 → 금액합8, 수량100+20-40=80, 시장30+40=70, 날짜 정렬, KSP/0001·상한252 | PASS |
| `investorDirectionsIncludeBothSellOpposingAndZeroWithoutMarketForNasdaq` | 양수/양수·음수/음수·음수/양수·null/null의 방향4분기, limit0→1, NASDAQ 시장 조회0 | PASS |
| `emptyInvestorSeriesHaveNullSummariesAndKosdaqUses1001` | 종목/시장 빈목록 → summary=null, KOSDAQ→KSQ/1001 | PASS |
| `relatedStocksOrderBySimilarityExcludeAnchorAndInsufficientPrices` | anchor·near 가격100→100→110/거래량10, far2배가격/거래량100, null·가격부족 후보 → near우선·anchor제외·2개, 가격110/변동10/10%/거래량10, 조회60·limit0→1 | PASS |
| `relatedStocksRelaxFilterAndLimitToThirty` | 35후보 거래량이 anchor의100배로 모든 비율 필터 밖 → 완화 후 전체 후보 거리정렬 경로에서30개 반환 | PASS |
| `macroRatesFallbackUsesStoredRateWhenSyncFails` | KR sync오류→저장2.75, US정상4.25 → 두 값 유지, 저장값 fallback 호출 확인 | PASS |
| `macroRatesMapExchangeAndSortBondsByCountryThenMaturity` | USD/KRW1300.5·통화, 순서를 뒤집은 합성 KR채권과 US채권 → KR3Y/KR10Y/US2Y 정렬·필드 보존. 역순 fixture는 정렬 검사용이며 provider 유효성 검사가 아님 | PASS |
| `macroComponentDoubleFailureMustPreserveOtherRates` | KR정상2.75, US sync/저장조회 모두오류 → KR 보존 기대, 실제 전체 빈카드. [F2-MACRO-001](../../../docs/findings.md) | FAIL |
| `macroSeriesSortsAndClampsFiveToFiveHundredWithPerComponentFailureIsolation` | KR역순값·US시계열 오류 → KR유지·오름차순, limit0→5·9999→500 | PASS |
| `macroSeriesDropsOnlyFailedBondAndKeepsOtherInstrumentPoints` | KR3Y조회 오류·KR10Y역순2.5/3 → KR3Y만 제외, 다른7개 block유지·날짜와값 오름차순 | PASS |
| `repositoryFailuresBecomeEmptyCardsWithoutExposingInternalMessages` | 공매도/수급 저장소 합성오류 →200·빈data/errors, 내부 문구 응답 미포함. 장애 warning 적절성은 별도 | PASS |
| `badHttpParametersFailBeforeAnyBusinessDependency` | 잘못된freq/window/limit 및 필수종목누락8요청 →400, 업무 의존성 호출0 | PASS |

재현: 위 Windows Gradle 명령에 `--tests '*Feature2CardFlowTest'`를 붙입니다. 최초16개15 PASS·1 FAIL, 환율/채권 mapping·시계열 격리2개를 추가한 전체 회귀에서18개17 PASS·동일1 FAIL을 확인했습니다. 실패 기대값이나 제품 코드는 바꾸지 않았습니다. 카드9경로가 호출되었다고 뉴스·종합분석·하위 ETL·SQL·실제 provider 전체 흐름이 완료된 것은 아닙니다. 이번 실제 메일/유료 호출은0회입니다.

## 뉴스 목록·본문·감성·캐시와 overlay 비용 연결

`NewsFlowTest` 23개 중19 PASS·4 FAIL(2026-09-25 Windows). 실제 `Feature2NewsController`→`NewsSentimentService`를 WebTestClient로 연결합니다. 분석 전용 `loadNews`·원천 캐시 `inspectNewsCache`는 공개 service API를 호출합니다. 마지막 비용 검사는 실제 `Feature3OverlayService`→실제 뉴스 service를 연결하며 기존 overlay 테스트의 뉴스 service mock을 이 케이스에서는 사용하지 않습니다.

외부 경계는 모두 대체합니다: JPA Repository, NaverNewsClient, NewsArticleExtractorClient, Feature2NewsSentimentClient, 비동기 observation 저장 service. Redis는 `ConcurrentHashMap<String,String>`·GET/SET Mockito로 구현하고 실제 ObjectMapper/JavaTime codec을 통과시킵니다. TTL은 SET 인자만 캡처하며 시간을 흘려 실제 만료·경합을 검사하지 않습니다. 유료 API·기사 사이트·모델·실제 DB/Redis·CreditService·메일은 호출하지 않습니다. 새 Spring application context나 scheduler를 시작하지 않습니다.

합성 입력: 사용자 데이터가 아닌 종목QA0001·산업 미사용·기사42/43·2026-09-24T12:00+09:00, `fixture.invalid` URL, 모델명 `synthetic-model-v1`. 한국어 본문은 테스트에서 직접 작성한 두 문장입니다. 기존 점수0.75·재분석 점수0.25·DB점수0.4는 fixture 응답이며 모델 정확도/실제 추론값이 아닙니다. focus SHA-256 기대값은 별도 JCA 계산과 UTF-8로 비교합니다. mock 저장소의 save 호출은 실제 commit·트랜잭션·중복 경쟁 보장이 아닙니다.

| 메서드 | 테스트 수행·독립 기대값 | 결과 |
|---|---|---|
| `missingInputAndMissingDetailRejectWithoutExternalCalls` | stockCode누락/잘못된ID→400, 없는기사→404, 빈코드→빈목록, 미등록종목→빈목록+목록실패warning, 외부호출0 | PASS |
| `publicListUsesCacheSortsDedupesWarningsAndDoesNotComputeSentiment` | 기사42/43의 캐시역순→최신43우선, warning중복제거, 공개목록 sentimentScore=null·모델/본문/기사DB조회0 | PASS |
| `invalidMetadataIsSkippedButValidArticleSurvives` | 빈제목 기사43·정상42 →42만 반환·메타실패warning | PASS |
| `refreshMarkerWithCurrentStoredArticleSkipsSearchAndCachesListFiveMinutes` | refresh marker와 DB최신 발행시각 일치 →검색0·목록TTL5분·설정limit2 전달 | PASS |
| `sourceSearchFiltersDeduplicatesExpandsAndUpsertsMetadataAndMapping` | 무관기사·관련기사중복·다음페이지관련기사 →start1/3 두호출·관련2개저장·기존map제외신규map1개·refresh15분/list5분 | PASS |
| `sourceFailureReturnsStoredNewsAndPublicWarning` | 검색예외→저장기사42 유지·목록실패warning·메타save0 | PASS |
| `cachedDetailAndStoredScoreDoNotTriggerExtractionOrAnalysis` | 캐시본문+DB점수0.4 →상세반환·추출/모델0 | PASS |
| `detailExtractionRemovesNoiseAndReportsLowConfidenceWithSevenDayTtl` | 사진표기/무단전재/Copyright행+두본문문장 →노이즈제거·readerSummary·lowConfidencewarning·본문TTL7일 | PASS |
| `failedBodyExtractionKeepsArticleMetadataButNoInventedBody` | 추출예외→제목/URL유지·본문/요약null·기사별warning·본문cache저장0 | PASS |
| `redisReadWriteFailuresRemainWarningsAndDoNotHideStoredList` | Redis GET/SET 및provider오류 동시주입→기사42 유지·세warning존재·READ warning중복제거 | PASS |
| `analysisBuildsFocusHashesInputPersistsScoreAndQueuesObservation` | 목록만캐시→본문추출→TITLE/FOCUS/DETAIL→mock모델0.25→기사/모델키save·관측command·hash/버전일치, detail/focus/score TTL각7일 | PASS |
| `compatibleCacheIsReusableAndReturnsScoreWithoutModel` | 전체4계층·버전·hash일치→hit·발행시각cacheAsOf·0.75재사용·모델/추출/점수DB0·관측호출 | PASS |
| `oldModelVersionTriggersAnalysisRatherThanReusingDatabaseScore` | 모델버전만구버전→inspect miss·새점수0.25·모델1회 | PASS |
| `forceRefreshBypassesAllCachedLayersIncludingStoredSentiment` | 완전캐시가있어도force→검색·본문추출·모델·점수save 재호출·0.25 | PASS |
| `invalidUnknownAndMissingModelResultsDoNotAssignWrongArticleScore` | 모르는URL·null점수·invalidScoreUrl·빈결과 →잘못된기사점수부여0·warnings중복제거·save/관측0 | PASS |
| `modelFailurePreservesNewsAndReportsPerArticleWarning` | 모델Mono.error→기사목록유지·점수null·기사별실패warning·save0 | PASS |
| `cacheInspectionRequiresEveryLayerAndCompatibleFormat` | 목록/본문/focus/점수각각제거, 구입력형식, 손상JSON →miss·외부분석/DB접근0 | PASS |
| `inspectionMustRejectChangedFocusVersion` | 정상캐시 후focus/hash만변경 →miss기대, 실제hit | FAIL |
| `analysisMustNotReuseScoreForDifferentFocus` | 변경focus·기존0.75 →재분석0.25/모델1회기대, 실제0.75·모델0 | FAIL |
| `inspectionMustRejectCachedEntryWithoutScore` | 점수null인 불완전캐시 →miss·재분석기대, 실제hit·점수null·모델0 | FAIL |
| `overlayPreviewAndEstimateMustChargeForChangedNewsFocus` | 정상캐시추가0 대조군 후focus변경 →MISS/추가1/총2기대, 실제HIT/추가0/총1 | FAIL |
| `storedScoreWithMissingRedisScoreIsReusedAndCached` | focus존재·Redis점수없음·DB0.4 →모델0·0.4재사용·점수TTL7일. DB기록 자체의본문버전 추적은 이 검사 범위 밖 | PASS |
| `sentimentPersistenceFailureDoesNotRemoveComputedScore` | 모델0.25→save오류 →0.25유지·기사별warning·관측호출 | PASS |

4개 FAIL은 [F2-NEWS-CACHE-001](../../../docs/findings.md)의 서로 다른 재현 경로입니다. 캐시 적중 검사가 modelVersion/promptVersion만 보고 focusTextVersion 일치와 점수 존재를 확인하지 않습니다. null점수는 손상캐시 주입이며 제품의 정상 쓰기가 null을 생성한다는 증거가 아닙니다. 비용 미리보기와 estimate는 서로 같은 잘못된 HIT 판정을 따릅니다. 실제 과금 오류를 발생시킨 검사가 아닙니다.

재현: 기본 Windows Gradle 명령에 `--tests '*NewsFlowTest'` 추가. 최초22개19 PASS·3 FAIL 후 실제 overlay 연결1개를 추가하여23개19 PASS·4 FAIL. 실패 검사는 skip/expected-failure 처리하지 않습니다. 최종 전체1053개 실행에 포함됩니다. Mockito generic Redis/list captor에 대한 unchecked compile 경고가 있으나 compileTestJava는 통과했습니다.

남은 범위: 실제 Naver 검색/기사 HTML 추출, 외부 client JSON 검증, FastAPI 운영 모델 추론, 실제SQL upsert/transaction/동시성, Redis 만료·부분갱신, 전체Feature2 분석 HTTP와 사용자 화면의 결합. dataset export 전용 경로도 별도 검증해야 합니다.

## Peer 데이터 pack·FastAPI codec·결과 캐시

`PeerDataFlowTest`는 실제 Controller→`PeerClusterDataServiceImpl`, `PeerClientCacheFlowTest`는 실제 `PeerClusterServiceImpl`→`PeerClusterClient`→WebClient codec을 연결합니다. JPA Repository/거래일/Redis와 HTTP 전송만 대체합니다. HTTP client는 `MockClientHttpRequest/Response` 기반 메모리 connector이므로 `fixture.invalid` DNS나 외부 네트워크를 호출하지 않습니다. 서비스 자체나 JSON 변환 메서드를 mock하지 않았습니다. `RedisConfig.redisObjectMapper()`의 순수 bean 생성 메서드만 호출하여 ISO 날짜·offset 유지·미지필드 무시 설정을 사용하며 다른 빈/연결/애플리케이션은 만들지 않습니다.

### 데이터 pack: 입력·기대값

합성45개 연속 날짜(2026-08-01~09-14 KST0시), 종목close100~144·volume10, 산업지수1000~1044, window30을 사용합니다. 주말도 포함한 계산 fixture이며 실제 거래 데이터가 아닙니다. 달력 mock을 명시적으로 바꾸는 케이스 외에는 모든 날짜를 거래일로 답합니다. 가격 Repository mock은 실제 JPQL에 정의된 `>= from && < to` 조건을 적용합니다. 이는 쿼리 의미에 맞춘 테스트이지 실제 SQL/JPA 실행 증거는 아닙니다. 산업지수 range mock은 Repository의 오름차순 계약에 맞춰 반환합니다.

| `PeerDataFlowTest` 메서드 | 입력·수행·기대값 | 결과 |
|---|---|---|
| `snakeAndCamelRequestsProduceUnwrappedInternalPackWithUtcPricesAndTurnover` | 두이름규칙/숫자문자열/종목trim·freq소문자 → envelope없는members/metas/prices/liquidity/industry_index, UTC시각·100×10=1000 | PASS |
| `missingOrMalformedSemanticFieldsReturnWarningsWithoutRepositoryAccess` | 빈/잘못된입력→HTTP200빈pack+BAD_* warnings, 직접null request→REQ_NULL, 저장소접근0 | PASS |
| `snakeKeyTakesPrecedenceEvenWhenNullAndMalformedJsonIs400` | snake industry_id=null/camel8→snake우선 BAD_INDUSTRY_ID, 손상JSON→400 | PASS |
| `exactRangeKeepsAllPointsInsteadOfWindowTail` | from/to 지정·window30 →45개유지, 조회to는종료일다음날0시, recent조회0 | PASS |
| `exactRangeAdjustsHolidayStartAndFutureEndUsingKstCalendar` | 시작휴일→이틀뒤, 종료미래→최신거래일, UTC입력→KST날짜환산·반개구간 | PASS |
| `holidayEndMovesBackAndInvertedAdjustedRangeStopsBeforePrices` | 종료휴일→직전거래일, 역전범위→BAD_DATE_RANGE·가격조회0 | PASS |
| `malformedOrOneSidedDateCurrentlyFallsBackToWindowMode` | 잘못된from+유효to→현재구현상window모드30개, 달력0. 잘못된날짜를400으로 검증했다고 주장하지 않음 | PASS |
| `windowSortsAndTailsSeriesWhileWeeklyLookbackUsesNinetyWeeks` | 역순반환→오름차순tail30, ONE_W→90주 lookback. 주간경로 인자검사이며 주봉 집계검증 아님 | PASS |
| `universeCapRetainsAnchorEvenWhenItIsLastOfThreeHundredAndTwo` | 302종목 마지막anchor→anchor첫번째+총300, UNIVERSE_CAPPED:300 | PASS |
| `shortZeroAndInvalidSeriesAreExcludedWithoutDroppingMetadata` | 29포인트/전부0/close없음/ts없음/stock없음→유효anchor만members·metas4개유지·각warning | PASS |
| `missingVolumeBecomesZeroAndThirtyPointBoundaryIsInclusive` | 정확30포인트사용, volume=null→volume/turnover0 | PASS |
| `repositoryFailuresReturnDiagnosticPacksWithoutPrivateMessages` | stock오류→빈pack·코드/예외클래스, price오류→members/meta만유지, 합성private문구없음 | PASS |
| `missingIndustryIndexPreservesUsableStockPricesAndWarns` | 지수mapping없음→종목시계열유지·지수빈객체·warning | PASS |
| `noMembersEmptyCodesAndAbsentAnchorAreDistinguished` | 산업멤버없음/공백코드뿐/anchor미포함→각warning구분 | PASS |
| `windowMustIncludeLatestIndustryIndexDateInStockSeries` | 지수와종목의동일최신일 포함기대 →실제종목만하루전에서종료, F2-PEER-DATE-001 | FAIL |

### 외부 client·캐시: 입력·기대값

고정 snake_case JSON은 method/industry/freq/window, 후보수·선택수, 산업보정 정보, 시계열·band·coverage, peer 수급/상관/시차/점수, 경고를 포함합니다. 예시값은 상관0.8·보정상관0.7·lag-2·LEADER·peerScore0.75이며 실제 상관 계산을 검증하는 값은 아닙니다. 날짜는2026-09-24+09:00, 본문 및 종목명은 합성입니다.

| `PeerClientCacheFlowTest` 메서드 | 입력·수행·기대값 | 결과 |
|---|---|---|
| `snakeRequestAndFullInboundToPublicMappingPassThroughRealCodecs` | POST `/feature2/peer-cluster` JSON MIME·9개snake요청필드·from/to instant, 응답 DTO의대표값/enum·Public camelCase·시계열t/value | PASS |
| `freshResultMappingCachesAllFieldsAndWarningsForTwelveHours` | client응답DTO와service복사DTO를 warnings제외tree동등 비교·warning보존·cache v6/TTL12시간, 재조회HTTP0추가 | PASS |
| `legacyDtoCacheStillWorksWithoutNewEnvelopeWarnings` | 과거DTO단독캐시→결과재사용·warnings빈목록·HTTP0 | PASS |
| `malformedOrStructurallyIncompleteCacheFallsBackToClient` | 손상JSON/빈객체/필수값부족캐시3종→각client호출·정상결과 | PASS |
| `redisReadAndWriteFailureDoNotRemoveAnalysis` | Redis GET/SET error→정상 분석payload 유지 | PASS |
| `forceRefreshSkipsRedisGetAndCallsClientEvenWithCache` | 정상캐시있어도force→GET0·client1·새SET12시간 | PASS |
| `cacheInspectionHasNoProviderSideEffectAndReportsAsOf` | miss/hit/Redis오류→false/true/false·asOf instant·inspect만으로HTTP0 | PASS |
| `nullIndustryAndBlankAnchorStopBeforeCacheAndProvider` | null산업/공백anchor→빈결과·inspect miss·GET/HTTP0 | PASS |
| `everyAnalysisParameterSeparatesCacheKeysAndOffsetsUseInstants` | 산업/anchor/freq/window/peerCount/maxLag/displayLimit/기간별키9개, 동일instant다른offset은같은키. fixture응답과요청종목일치 검사는 아님 | PASS |
| `envelopePayloadAndMissingOptionalArraysDecode` | data envelope와raw payload지원, 선택배열누락→빈배열 | PASS |
| `configuredMapperPreservesOffsetsAndIgnoresAdditiveFields` | 제품ObjectMapper에서추가필드무시·+09:00offset유지 | PASS |
| `malformedRootsInvalidEnumsAndBadHttpBecomeServiceWarningsWithoutCaching` | 잘못된JSON/null/배열/data누락형태/invalidfreq/503→client오류·service INTERNAL_ERROR warning·cacheSET0 | PASS |

재실행: 기본 Windows Gradle 명령에 `--tests '*PeerDataFlowTest' --tests '*PeerClientCacheFlowTest'` 추가. 데이터15개14 PASS·1 FAIL, client/cache12개 PASS. 초기 client 검사9개는 모의 connector가 body write완료 전에읽어서 실패했습니다. `Mono.defer`로 완료후 읽게 수정하니10 PASS·1 FAIL이었고, 남은1개는 Java필드ts/pct를wire이름으로잘못 기대한 테스트 문제였습니다. 실제 `RelativePointDto.@JsonProperty`와 `frontend/src/types/feature2.ts`는t/value로 일치하므로 그 계약에 맞춰 수정했습니다. 마지막에는 실제 JSON bean을 적용하고날짜/추가필드검사를 보강했습니다. 제품 수정 없이 테스트 하네스/기대계약만 바로잡았으며 window경계 결함은FAIL로 유지합니다.

범위 한계: 실제 Spring↔WSL FastAPI TCP왕복·FastAPI상관/클러스터수학·DB쿼리·Redis실제만료/동시성·브라우저차트까지 연결한 E2E는 아닙니다. window/date경계 결함의 실제 SQL 재현, 운영 달력·원천 데이터 품질도 남았습니다. 이번 외부 유료호출·메일·기존DB변경0회입니다.

## 관리자 거시지표18경로: HTTP→수집→해석→저장 요청

`MacroAdminFlowTest`: **23 PASS**(2026-09-25 Windows Java17, 단독 Gradle22초·테스트XML7.069초, exit0). 실제 BaseRateAdminController6경로·BondYieldAdminController8경로·ExchangeRateAdminController4경로에 실제 sync service5개·BOK/FRED client5개·WebClient JSON codec을 연결했습니다. 실제 provider 전송만 `exchangeFunction`의 합성 ClientResponse로 대체하고 Repository3개는 Mockito+메모리 행 목록으로 대체합니다. 전송 카운터는 `Mono.defer` 내부여서 조립 횟수가 아닌 실제 구독/재시도를 셉니다.

standalone WebTestClient에는 실제 GlobalExceptionHandler와 프로젝트 `RedisConfig.redisObjectMapper()`의 순수 bean 생성 결과를 encoder/decoder로 적용했습니다. Redis 연결 bean이나 전체 Spring 애플리케이션은 만들지 않습니다. 관리자 보안필터는 이 클래스에 없고, 기존 ApiSecurityMatrixTest의 별도 접근제어 증거와 구분합니다. 원본 DB·금융 API·Redis·메일·scheduler 호출0, 실제 비용0회입니다.

fixture는 날짜2026-09-23/24·값2.50/2.75의 BOK 대문자필드JSON, 다른 지표와 한국 기준금리의 key-statistic 목록, 날짜 역순2026-09-01/08-01·4.25/4.50의 FRED observations입니다. 개별 검사에서 USD/KRW1300.50, 결측/null/`.` 및 연말연초·페이지 경계를 주입합니다. 합성 응답은 요청기간에 따라 자동필터링하지 않으므로 실제 원천의 기간 정확성을 증명하지 않습니다. save mock은 동일 객체를 유지하고 updatedAt만 현재 Instant로 갱신합니다. JPA callback·실제 unique constraint·SQL멱등성·rollback을 구현한 DB 대체가 아닙니다.

| 메서드 | 실행·기대값(모두PASS) |
|---|---|
| `koreanLatestFiltersStatisticMapsAllFieldsAndReusesNaturalKey` | 다른지표 제외·기준금리2.75/일자/코드/D/BOK_ECOS, KeyStatistic URI,2회sync→동일객체save2·메모리행1 |
| `koreanBackfillHasInclusiveDatesAndStoredValues` | 요청기간 BASIC_ISO_DATE·stat/item 경로전달,저장2·첫/마지막일·2.50보존 |
| `fredLatestIsDailyDffAndSelectsMaximumDate` | DFF·일간D·frequency미지정·현재KST기준120일조회,역순행에서최대일9/1선택 |
| `fredBackfillSkipsMissingAndDotValuesAndParsesComma` | null/누락/`.`행제외,공백/쉼표숫자1234.5·ISO rawTime·저장1 |
| `fxLatestUsesDefaultPairAndPreservesCommaValueAndFallbackMetadata` | 기본USD_KRW·USD/KRW·1300.5·원단위·누락메타의731Y001/0000001 fallback |
| `fxBackfillSplitsYearBoundaryAndReportsGlobalDates` | 12/30~1/2를12/31과1/1경계로2요청,저장2·전역최소/최대일 |
| `koreanBondsLatestAllAndSingleMapMaturity` | 코드생략→KR3Y/KR10Y·36/120개월,소문자단일코드→1결과·기존자연키재사용 |
| `koreanBondBackfillAllReportsCountsPerInstrumentAndSingleSplitsYears` | 전체4행·종목별2행,단일종목12/31~1/1을2연도로분할 |
| `fredBondsLatestAllAndSingleUseMonthlySeries` | US2Y/US5Y/US10Y·월간M·frequency=m·GS10·60개월,단일us5y선택 |
| `fredBondsBackfillAllAndSinglePreserveDateRange` | 전체6행·US10Y2행,단일US2Y2행·from그대로전달 |
| `latestWithoutSyncReadsAsOfAndNeverCallsProvider` | FRED금리/국내채권/미국채권/환율4GET의sync=false→저장값·source,과거asOf없음→null,전송0 |
| `latestReadEmptyReturnsNullForAllFourRoutes` | 빈저장소4GET→200/data=null·전송0 |
| `freshlyUpdatedRowsAvoidProviderSubscriptionEvenForHistoricalAsOf` | 당일updatedAt·과거asOf4GET+KR service→기존값·외부구독0 |
| `staleRowsRefreshThenReadAndDoNotInsertDuplicates` | updatedAt=epoch→4GET각1회수집후재조회·메모리행수동일 |
| `seriesRoutesClampLimitsAndPreserveDescendingRepositoryOrder` | FRED금리limit0→page1·행순서유지,국내/미국채9000→5000,환율-1→1,전송0 |
| `missingReversedOrMalformedDatesAndUnknownInstrumentsStopBeforeNetwork` | backfill5경로×필수누락/역전/형식오류+종목/통화오류3개→400·Repository/전송0 |
| `missingProviderCredentialsAndDirectArgumentsDoNotReachTransport` | 4client 빈키·stat/series빈값·null날짜→예외·전송0 |
| `bokPaginationCombinesThreePagesBeforeStorage` | total201·페이지당합성1행→1/100,101/200,201/300 순차3페이지·환율저장3 |
| `bokServerErrorsRetryThreeTimesButClientErrorsDoNot` | BOK503→첫시도+재시도3=4구독·retry-exhausted,429→1구독;12초상한·실제backoff사용 |
| `emptySourceKeepsExistingLatestWithoutSave` | source빈배열→FRED금리/국내채/미국채/환율모두저장된latest재사용·save0 |
| `bokNullRowsAndMissingValuesAreSkippedForFxAndBonds` | null/시간·값누락제외,유효1행각저장·채권단위%/통계코드fallback |
| `malformedNumericValueIsRejectedBeforeSave` | BOK환율/FRED금리 비숫자→400·save0 (현재advice 동작) |
| `secondYearFailureOccursAfterFirstYearSaveRequest` | 첫연도성공/둘째연도400원천오류→관리자500·전송2·이전save요청1관측 |

재현: 기본 Windows Gradle 명령에 `--tests '*MacroAdminFlowTest'`를 추가합니다. 첫 컴파일은 제공되지 않는 reactor-test 의존성의 StepVerifier 사용으로 실패했습니다. 빌드파일/의존성은 바꾸지 않고 테스트를 bounded block·retry-exhausted 검사로 변경했습니다. 다음 실행의5개 실패는 standalone codec의 LocalDate배열직렬화 때문에 발생했습니다. 프로젝트 mapper를 테스트에 명시적으로 적용한 뒤23개 전부 통과했습니다. 운영 HTTP bean wiring 전체 검증과는 구분하며 제품 코드를 고치거나 결함회귀를 숨긴 것이 아닙니다. 이후 전체1053개 실행에도 이23개가 통과했습니다.

한계: 실제 BOK/FRED 가용성·데이터품질·원천스키마/일간월간정책, 실제DB의unique/트랜잭션/부분실패원자성·동시upsert, 모든페이지/누락응답/수치·기간경계, 관리자JWT와업무handler결합, scheduler실행은남았습니다. 연도중간실패의 첫save관측은 실제DB rollback불가를 입증하지 않습니다. 날짜와현재KST비교검사는실행일에의존하며자정경계·DST전체를검증한것은아닙니다.

## 관리자 재무 CRUD·CSV 적재4경로

2026-09-25 Windows 추가28개: **FinancialAdminFlowTest13개 중10 PASS·3 FAIL, FinancialCsvFlowTest15개 중14 PASS·1 FAIL**. 실제 Controller/service/mapper 및 파일 읽기/parser/SQL인자생성을 실행합니다. Repository·JdbcTemplate·PlatformTransactionManager는 mock입니다. TransactionTemplate 자체는 실행되어 commit/rollback 호출을 관찰하지만 실제DB rollback·잠금·유일키를 검증하지 않습니다. CRUD HTTP에는 프로젝트JSON mapper를 적용하고 관리자권한은 기존 보안 matrix와 별도입니다.

CSV는 JUnit `@TempDir`에 UTF-8로 만든 합성파일만 읽습니다. Windows 실행 명령의TEMP/TMP는test/.runtime으로지정하며JUnit이자신의임시파일만정리합니다. 사용자 CSV를읽거나수정하지않습니다. CSV HTTP의path도이fixture만전달합니다. importer runner는직접객체를호출하며프로필기동·전체앱·scheduler를실행하지않습니다. 실제DB/외부API/메일/비용호출0입니다.

CRUD fixture는 QA0001/stockId7,2025년·공시일2026-03-31,매출1000·영업이익200·순이익80·자산800·부채400·자본400입니다. CSV 기본행은숫자종목코드5930→005930/KOSPI,연간2025/기간번호99→1,version1·매출1000.50·매출총이익300등이며동일종목코드DUP의NASDAQ/NYSE두상장을추가합니다.

| `FinancialAdminFlowTest` 메서드 | 방법·기대/관측 | 결과 |
|---|---|---|
| `createAnnualNormalizesPeriodMapsRatiosAndCommits` | A공백/소문자정규화·periodNo1·종목연결·ISO날짜·영업20/순익8/ROE20/부채100%,commit1. legacy시장가치2000은entity저장되지만override없는mapper응답은null | PASS |
| `quarterHalfAndTtmNormalizePeriodNumber` | Q3/H2/TTM→기간번호3/2/0·H/TTM분기null | PASS |
| `halfInferenceUsesReportMonthAndDefaultReportDateIsToday` | 3월→H1·7월→H2·reportDate생략→실행일 | PASS |
| `invalidPeriodCombinationsRollbackWithoutSave` | 잘못된type·Q분기누락/0/5·Qhalf혼합·H3/quarter혼합·Ahalf·TTMquarter·year/type누락11조합→400·rollback11·save0 | PASS |
| `missingStockOrStatementReturns404AndRollsBack` | 등록종목없음/update/delete대상없음→404·rollback3·write0 | PASS |
| `updateChangesNonNullFieldsPreservesOmittedAndAllowsZero` | revenue0/netIncome-10반영·생략영업이익200보존·분모0비율null·commit | PASS |
| `updatePeriodTransitionClearsQuarterAndRecomputesHalf` | Q4→H2→TTM,quarter삭제·periodNo0 | PASS |
| `deleteCommitsAndSecondDeleteReturns404` | 첫삭제commit·data=null·메모리제거,재삭제404/rollback | PASS |
| `saveFailureRollsBackAndHasSanitizedHttpResponse` | save RuntimeError→500·상세비공개·rollback1/commit0 | PASS |
| `malformedJsonAndPathStopBeforeService` | 잘못된JSON/날짜/숫자path→400·repo/tx0 | PASS |
| `halfOnlyUpdateMustChangePeriodNumber` | H1등록후half2만수정→200이지만H1,기대H2 | FAIL: FIN-ADMIN-HALF-001 |
| `createMustPreserveWritableAbsoluteFinancialFields` | grossProfit300/retainedEarnings250/cash90요청→저장객체null3개 | FAIL: FIN-ADMIN-FIELDS-001 |
| `updateMustPreserveWritableAbsoluteFinancialFields` | 같은3필드수정→저장객체null3개 | FAIL: 같은문제 |

| `FinancialCsvFlowTest` 메서드 | 방법·기대/관측 | 결과 |
|---|---|---|
| `utf8BomHeaderSqlOrderAndTypedNullsHaveIndependentExpectations` | BOM제거·44SQL컬럼/인자길이·종목/날짜/버전/숫자/통화/출처·INTEGER/DECIMAL typed null·동일생성/갱신시각·REQUIRES_NEW/commit | PASS |
| `everyDecimalCsvColumnBindsToItsOwnSqlColumn` | 32개decimal필드에1.125..32.125서로다른값을넣어각SQL컬럼의인자와일대일비교 | PASS |
| `listingNormalizationDefaultExchangeAndRowOverrideDisambiguateStocks` | DUP무거래소→ambiguous,기본XNAS→NASDAQ/id8,행XNYS가기본값보다우선→id9 | PASS |
| `uniqueCodeWithoutExchangeResolvesAndUnknownOrWrongListingSkips` | 고유소문자qaus→id10,알수없는코드/잘못된거래소→unknown2 | PASS |
| `rowFailuresAndBlankLinesDoNotPreventOtherValidRows` | 공백1·잘못된날짜/숫자/필수version3→각카운트,정상1행저장요청유지 | PASS |
| `periodRulesNormalizeAnnualTtmAndRejectQuarterHalfOutOfRange` | A/Q/H/TTM→1/3/2/0, Q5/H0실패2·정상4 | PASS |
| `thousandRowBoundaryUsesSeparateTransactionsAndCopiesParametersBeforeClear` | 1001행→1000/1두배치·commit2,upsert SQL·created_at미갱신 | PASS |
| `repeatedPeriodRowsArePassedToSqlUpsertNotDeduplicatedInMemory` | 같은기간2행을모두전달·version2/매출2000·revenue UPDATE절. DB멱등성검사아님 | PASS |
| `emptyAndHeaderOnlyFilesDoNotStartWriteTransaction` | 빈파일/헤더뿐→upsert0·JDBC/tx0 | PASS |
| `absentFileFailsBeforeStockLookup` | 파일없음→IllegalArgumentException·stock/JDBC/tx0 | PASS |
| `finalBatchDatabaseFailureRollsBackAndPropagates` | 마지막잔여배치실패→예외전파·rollback1·commit0 | PASS |
| `fullBatchFailureRetriesRetainedRowsAndOverlapsFailedCounter` | 첫1000실패후보존·다음행포함1001성공→upsert1001/failed1·rollback1/commit1. 집계가배타적이지않음 | PASS: 관측 |
| `actualImportHttpReturnsCountsAndMissingPathIsBadRequest` | 실제CSV HTTP→upsert1/defaultKOSPI/failed0,필수path없음400 | PASS |
| `runnerSkipsOptionsPassesExchangeAndDoesNotRunWithoutPath` | 경로없음실행0,Spring옵션제외·첫경로/exchange인자전달 | PASS |
| `quotedCsvNumericFieldMustBeParsedAsValidValue` | 합법적인quoted revenue `"1000.50"`→기대upsert1,실제0/failed1 | FAIL: FIN-CSV-QUOTE-001 |

재현 필터: `--tests '*FinancialAdminFlowTest' --tests '*FinancialCsvFlowTest'`. 초기27개 실행의6실패 중2개는 Mockito의when 재정의시기존Answer가실행된테스트문제였습니다. doThrow/doAnswer로재정의하여이를제거한뒤27개23 PASS·4 FAIL,32숫자필드매핑1개추가후전체회귀에서28개24 PASS·4 FAIL을확인했습니다. 제품은미수정이며실패4개를skip/expected처리하지않습니다.

한계: 실제SQL문법/마이그레이션·unique key·동시upsert·중복DB최종값·트랜잭션복구, 관리자권한과handler결합, 실제CSV인코딩/모든인용·개행형식,금액정밀도/범위·회계적일관성,경로접근정책·대용량성능은남았습니다. CSV상태upserted는영향행수합이아니라성공batch에전달한행수입니다. runner프로필선택이나설정전체를검증하지않았습니다.

## 격리 MySQL 생성 시도

사용자는 격리 DB 생성을 허용했으며 본 프로젝트에 영향이 없어야 한다고 명시했습니다. `IsolatedDatabaseSupport`는 기존 YAML에서 endpoint/credential을 메모리로만 읽고, `information_schema` 연결에서 고유한 `qaima_qa_YYYYMMDD_<12hex>` DB만 생성하도록 제한합니다. 기존 catalog와 이름이 같거나 QA 이름 규칙이 아니면 거부합니다. `IF NOT EXISTS`로 기존 DB를 인수하지 않습니다. 성공한 생성은 test runtime journal에 기록하고 전용 marker·JDBC catalog·`SELECT DATABASE()`가 일치해야 이후 연결을 반환합니다. 비밀번호는 journal에 저장하지 않습니다.

`IsolatedDatabaseProvisionTest`는 `QAIMA_PROVISION_ISOLATED=1`일 때만 실행됩니다. 새 DB에 ownership marker를 만든 뒤 Flyway baseline **0**에서 원본 V1부터 모든 migration 적용·validate·두 번째 migrate가0건인지 검사하도록 작성했습니다. 운영 Spring context·scheduler·Redis·메일·수집기를 시작하지 않습니다. Flyway clean은 비활성화합니다.

2026-09-25 실행은 **DB 생성 단계에서 FAIL**했습니다. SQLState42000/code1044: 현재 설정 계정의 생성 권한 부족. 새 DB/journal/marker는 생성되지 않았고 Flyway는 실행되지 않았습니다. 마이그레이션 성공으로 기록하지 않습니다. 기존 계정 권한을 변경하거나 다른 관리자 자격증명으로 우회하지 않았습니다. 별도 임시 MySQL 프로세스 사용 여부 확인이 필요합니다.

재현 명령(기존 Gradle 환경 설정 후):

```powershell
$env:QAIMA_PROVISION_ISOLATED = '1'
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*IsolatedDatabaseProvisionTest'
Remove-Item Env:QAIMA_PROVISION_ISOLATED
```

주의: 생성 권한이 생긴 환경에서 위 명령은 매번 새 QA DB를 만들기 때문에 무조건 반복하지 않습니다. 아직 자동 삭제는 구현하지 않았습니다.
