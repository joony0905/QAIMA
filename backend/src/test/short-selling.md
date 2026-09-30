# FINRA·KRX 공매도 수집 CI/QA 기록

검증일: 2026-09-25. Windows Java17·Gradle8.14. [ShortSellingAdminFlowTest](java/com/qaima/verification/ShortSellingAdminFlowTest.java) 32개와 [ShortSellingSchedulerTest](java/com/qaima/verification/ShortSellingSchedulerTest.java) 8개, **40 PASS**. 전체 Spring 회귀 720개에 포함된 수치이며 별도 합산하지 않습니다.

## 실제 실행한 경계

`WebTestClient → 실제 관리자 Controller 2개 → 실제 SyncService 2개 → 실제 WebClient 설정/Client/FINRA parser → 인메모리 HTTP connector → 실제 SQL 인자 생성/TransactionTemplate → mock JDBC·트랜잭션 관리자`를 실행했습니다. KRX 요청은 BodyInserter가 form 본문을 실제 직렬화한 뒤 검사합니다. 서비스나 제공처 client를 통째로 mock한 HTTP 정상 흐름이 아닙니다. 예외는 마지막 3000일 집계 검사 2개로, 이때만 syncDaily를 spy로 대체하고 실제 backfill의 순서·상한·집계를 검증합니다.

`WebClientConfig.finraWebClient/krxWebClient`의 순수 factory를 호출하고 connector만 교체했습니다. 16MiB 응답 버퍼·KRX User-Agent도 제품 설정입니다. Public 날짜 JSON은 프로젝트 RedisConfig의 ObjectMapper factory로 직렬화하지만 Redis에 연결하지 않습니다. StockRepository는 거래소별 합성 종목을 반환하고 JDBC는 SQL과 인자를 복사해 보관합니다. 서비스가 batch.clear()를 호출하기 전에 복사하여 기록 소실을 막았습니다. mock 트랜잭션 관리자는 호출마다 새로운 SimpleTransactionStatus를 반환합니다.

Spring 전체 application context·스케줄링 타이머·실제 DB·Redis·외부 API·메일·LLM은 시작하지 않았습니다. 네트워크 connector가 인메모리 구현으로 교체되어 외부 전송은 0회입니다. 보안필터는 이 테스트에 포함되지 않으며 별도 ApiSecurityMatrixTest의 sentinel 검증과 구분합니다.

## 재현 명령과 증거

Windows PowerShell에서 backend 디렉터리 기준입니다. 실제 연동 opt-in을 먼저 해제합니다.

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY -ErrorAction SilentlyContinue
$env:GRADLE_USER_HOME = "$PWD\src\test\.runtime\gradle-home"
$env:TEMP = "$PWD\src\test\.runtime\tmp"
$env:TMP = $env:TEMP
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*ShortSellingAdminFlowTest' --tests '*ShortSellingSchedulerTest'
```

처음 API 28개 실행은 27 PASS·1 FAIL이었습니다. 테스트가 기본 WebClient의 256KiB 버퍼를 사용해 KRX 1001행 fixture를 읽지 못했습니다. 실제 제품 factory의 16MiB 설정을 적용한 뒤 API28·scheduler8 모두 PASS였습니다. 이후 경계/진단4개를 추가하고 전체 회귀를 실행하여 API32·scheduler8의40 PASS를 확인했습니다. 제품 소스나 설정은 변경하지 않았습니다.

최종 전체 명령은 위 명령에서 `--tests` 필터를 제거한 것입니다. 36초 후 종료코드1: **720개, 700 PASS·16 FAIL·4 SKIP**. 실패16개는 기존 결함 회귀이며 공매도 신규 실패는 없습니다. XML은 `.runtime/build/test-results/test/TEST-com.qaima.verification.ShortSellingAdminFlowTest.xml`과 `TEST-com.qaima.verification.ShortSellingSchedulerTest.xml`, HTML은 `.runtime/build/reports/tests/test/index.html`입니다. 이후 필터 실행은 XML을 덮어쓸 수 있습니다.

## API·원천 해석·저장 요청: 32개

기준일은 2026-09-24입니다. 미국 종목 QA_US=id1, DUP는 NASDAQ=id2/NYSE=id3의 모호한 코드입니다. 국내 같은 코드000001은 KOSPI=id4/KOSDAQ=id5/KONEX=id6으로 구분합니다. KRX 숫자10개는 서로 다른 1.125~10.125를 넣어 잘못된 컬럼 연결을 탐지합니다. 아래 모든 메서드가 PASS이며, ‘관측’ 표기는 바람직한 정책이나 실제 DB 성공을 승인했다는 의미가 아닙니다.

| 테스트 메서드 | 수행·기대값·확인한 범위 |
|---|---|
| `finraHttpMapsSymbolRatioTypedNullsAndSqlTransaction` | POST sync→날짜/집계 JSON; CNMSshvol20260924.txt GET; symbol trim/대문자→id1; 18 SQL 인자, short1/exempt0/total3/ratio0.333333; 적용거래량·금액6필드 DECIMAL null; FINRA/CNMS·생성/갱신시각 동일·REQUIRES_NEW/commit1·created_at UPDATE 제외 |
| `finraParserSupportsBomCrLfFooterBlankSymbolsAndNullNumbers` | BOM/CRLF/빈행/숫자footer 제거, 빈 symbol 제외, trim·대문자, 공백 숫자null; null/공백/헤더만 입력은 빈목록 |
| `finraRejectsWrongHeaderMalformedRowsNumbersAndDates` | 잘못된 헤더·열수·NaN·실재하지 않는 날짜는 parser 예외 |
| `finraAmbiguousAndUnknownSymbolsAreSkippedWithoutGuessingExchange` | 정상/DUP/UNKNOWN3행→저장1·ambiguous1·unknown1 |
| `finraDuplicateSameIdAndInvalidStockMetadataDoNotMakeSymbolAmbiguous` | 같은ID 중복은 모호하지 않음; ID없음/공백코드 무시, 정상행 저장요청1 |
| `finraMissingFile403And404AndEmptySuccessReturnZeroWithoutWrite` | 403/404/빈200 각각 fetched0·HTTP200, JDBC/tx0. 관측:403을 권한오류와 구별하지 않음 |
| `finraOtherHttpErrorsAreNotRetriedOrWritten` | 401/429/503→HTTP500·외부상세 비포함; 각1회 전송, repo/JDBC/tx0 |
| `finraMalformedProviderFileCurrentlyBecomesHttp400` | 정상 HTTP200의 잘못된 원천행→관리자400·저장0. 관측:원천오류가 입력오류로 분류됨 |
| `finraRatioRoundsHalfUpAndInvalidDenominatorIsTypedNull` | 2/3→0.666667; 분모0/음수·분자null은 typed null |
| `finraUsesRowDateEvenWhenDifferentFromRequestedDate` | 요청24일/행23일→응답date24일, SQL report_date23일. 원천 날짜 대조는 하지 않는 현재 동작 |
| `finraBackfillIncludesWeekendAndAggregatesOnlyNonEmptyDays` | 금~일3일 순차GET; 첫날1행/주말404→requested3/data1/fetched1/upsert1 |
| `krxHttpSerializesEveryMarketFormAndMapsAllSqlNumbers` | STK→KSQ→KNX 순차POST; 경로·form7필드·content-type/accept/origin/referer/X-Requested-With/User-Agent 검사; 3행 거래소별ID·18인자·숫자10개·출처·commit1 |
| `krxParserHandlesCommaDashNullWhitespaceAndBlankCode` | 1,234.50→1234.50, dash/null/공백→null, 코드trim/대문자·회사명 보존, 빈코드 제외 |
| `krxEveryNumericColumnConvertsDashToTypedSqlNull` | 숫자10필드 모두 dash→각 DECIMAL typed null |
| `krxErrorTaxonomyDistinguishesStatusLogoutHtmlSchemaJsonColumnsAndNumbers` | 403/429, LOGOUT, HTML/script, 비JSON, OutBlock 누락/null/객체, 필수열누락, 잘못된숫자의7종 errorCode. 전송당 재시도0 |
| `krxEveryRequiredColumnIsCheckedAndEveryNumericColumnRejectsInvalidText` | 필수13열을 하나씩 제거→COLUMN_MISSING/필드명; 숫자10열을 하나씩 변조→DECIMAL_PARSE_ERROR/필드명, 23개 합성응답 |
| `krxTransportFailureHasExplicitCodeAndDoesNotRetry` | 합성 전송예외→KRX_REQUEST_FAILED·상세문구 포함, 전송1회 |
| `krxEmptySuccessBodyAndEmptyArrayHaveNoWrite` | 빈body/공백body/빈array 각각3시장 조회·fetched0, JDBC/tx0 |
| `krxProbeContinuesAfterOneMarketErrorAndNeverTouchesStorage` | KOSDAQ LOGOUT·다른2시장 정상→HTTP200/fetched2/failed1; 시장별성공·errorCode, 저장소 접근0 |
| `krxSyncStopsAtFailedMarketBeforeAnyWriteOrLookup` | 같은 실패조건에서 sync→HTTP500·2시장까지만 전송, lookup/JDBC/tx0 |
| `krxProbeCurrentlyIncludesTransportExceptionDetailInAdminPayload` | 3시장 전송예외→HTTP200/failed3/KRX_REQUEST_FAILED·합성 상세문구 반환, 저장소0. 관리자 진단 노출 관측 |
| `krxUnknownSamplesAreCountSortedLexicalTieBrokenAndCappedAtTwenty` | 미등록25종27행→unknown27; 최빈X24=3이 먼저, 동률은 key순, 샘플20개 상한; 쓰기0 |
| `krxBackfillVisitsCalendarDaysAndAggregatesAllMarkets` | 금~일×3시장9전송, 첫날만자료→requested3/data1/fetched3/upsert3·commit1 |
| `invalidHttpDatesRangesAndMissingParametersStopBeforeProviders` | sync/probe 필수날짜 누락·2월30일; backfill 필수to누락·역순·3001일→모두400, 전송/저장소0 |
| `nullArgumentsAreRejectedBeforeProviderTransport` | client 날짜/시장·sync/probe/backfill null→IllegalArgumentException, 전송/저장소0 |
| `finraBackfillAcceptsExactlyThreeThousandDaysAndAggregatesDailyCounters` | syncDaily만 spy대체, 실제backfill3000일 허용·양끝/순서·requested/data3000/fetched9000/upsert·unknown·ambiguous 각3000 |
| `krxBackfillAcceptsExactlyThreeThousandDaysAndDeduplicatesSampleStrings` | syncDaily만 spy대체,3000일 허용·양끝/순서·fetched9000/upsert3000/unknown6000; 동일sample은1문자열 유지(누계6000으로 재작성되지 않음) |
| `bothProvidersSplitThousandAndOneRowsIntoIndependentTransactions` | FINRA/KRX 각각1001행→1000+1 두배치·commit2/rollback0 |
| `jdbcUpdateCountsDoNotDetermineReportedUpsertCount` | JDBC반환 영향행수0 주입, FINRA응답 upsert1: 성공 요청행수라는 현재 의미 |
| `writeFailureRollsBackAndDoesNotAutomaticallyRetryEitherProvider` | 두provider JDBC예외→각HTTP500·상세 비공개·rollback1·commit0·자동재시도0 |
| `laterBatchFailureDoesNotUndoEarlierCommittedBatch` | 두provider 각각1001행의 두번째배치 실패→HTTP500, 첫1000 commit1·두번째 rollback1 호출. 전체요청 원자성 없음 관측 |
| `laterBackfillProviderFailureRetainsEarlierDayCommit` | 두provider 첫날성공 후 둘째날503→HTTP500·셋째날 미실행, 첫날commit1/rollback0. 실제DB 보존을 실행한 것은 아님 |

## 스케줄러: 8개

실제 scheduler 객체를 직접 호출하고 서비스 publisher만 mock합니다. @Value 필드는 ReflectionTestUtils로 주입합니다. 실제 cron 등록/실행은 하지 않습니다.

| 테스트 메서드 | 수행·기대값 |
|---|---|
| `finraOffSwitchPreventsServiceInvocation` | enabled=false→서비스0 |
| `krxOffSwitchPreventsServiceInvocation` | 동일 KRX 대조 |
| `finraUsesNewYorkDayAndClampsLookbackToOne` | lookback -3/0/1/5/7→최소1; New York 오늘을 끝으로 inclusive범위. 실행 전후 날짜를 기록해 자정 경계를 허용 |
| `krxUsesSeoulDayAndClampsLookbackToOne` | 같은 경계, Seoul 오늘 적용 |
| `finraResetsRunningAfterPublisherAndSynchronousFailure` | 첫호출 Mono.error, 둘째 메서드 자체 throw, 이후2번정상empty; 예외 외부전파0·4번모두진입·running=false 복귀 |
| `krxResetsRunningAfterPublisherAndSynchronousFailure` | 같은 오류 복구 대조 |
| `finraSkipsOverlappingRunAndAllowsRunAfterCompletion` | 단일 executor에서 첫호출을 Sinks.Empty로 보류, latch로 실제진입확인 후 중첩호출→서비스1회 유지; 완료 후재호출→2회 |
| `krxSkipsOverlappingRunAndAllowsRunAfterCompletion` | 같은 동시실행 방지 대조 |

동시성 검사는 latch3초·future5초·종료5초 제한을 사용하고 finally에서 gate해제·cancel·executor종료를 확인합니다. 고정 sleep을 이용한 추측은 하지 않습니다. FINRA/KRX끼리의 상호배제나 여러 JVM 간 분산잠금을 검증하는 것은 아닙니다.

## 해석과 남은 범위

403을 빈자료로 간주하는 FINRA 정책, 요청일/원천행 날짜 불일치 허용, KRX probe의 전송예외 상세문구, 빈200 허용, FINRA parser 오류의400분류, 배치/날짜별 부분커밋, JDBC영향행수와 다른 upsert집계는 관측사항입니다. 확정된 정책 위반으로 단정하거나 제품을 수정하지 않았습니다.

실제 금융 제공처의 현재 응답/휴장일·인증·차단, 실제 SQL문법/마이그레이션/정밀도/unique key/동시upsert/rollback, 관리자 필터와 handler 결합, 적재→ShortSellingFeatureService/카드/브라우저 전체 연결, Java CSV importer, cron·장기 실행·재시작/분산 중복방지 검증은 남았습니다. 따라서 Swagger 5개 경로의 상태는 PARTIAL이며 실제연동은 NOT_RUN입니다.
