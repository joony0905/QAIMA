# 종목·시장 수급 API와 배치 CI/QA 기록

검증일: 2026-09-25. Windows Java17·Gradle8.14에서 추가 **50개 중47 PASS·3 FAIL**. [InvestorFlowAdminFlowTest](java/com/qaima/verification/InvestorFlowAdminFlowTest.java) 30개(27 PASS·3 FAIL), [InvestorFlowBatchTest](java/com/qaima/verification/InvestorFlowBatchTest.java) 16 PASS, [KisBatchSchedulerTest](java/com/qaima/verification/KisBatchSchedulerTest.java) 4 PASS입니다. 실패3개는 미등록 종목의 HTTP 오류 응답 누락을 재현하며 skip/expected-failure로 처리하지 않았습니다.

## 검증 경계와 안전장치

관리자5경로는 `WebTestClient → 실제 Controller → 실제 Stock/MarketInvestorFlowService → 실제 KisInvestorFlowClient → 실제 WebClient JSON codec/요청 구성 → 인메모리 connector → 실제 entity/응답 DTO`로 실행했습니다. 전송 connector·Repository·rate limiter만 mock입니다. Repository는 합성 행을 메모리에 보관하고 자연키로 같은 객체를 찾아 갱신합니다. 따라서 실제 JPA/SQL·유일성·트랜잭션 커밋·롤백을 증명하는 검사가 아닙니다.

인증 토큰도 실제 client의 발급/캐싱 코드를 거치되 요청과 응답은 합성 값입니다. KIS YAML secret을 읽거나 실제 인증·금융 API로 전송하지 않습니다. `KisBatchRateLimiter.acquireMono()`의 mock은 구독 시 카운트를 증가시켜 토큰/데이터 요청 앞에서 실제 구독됐는지 확인합니다. 별도 배치 검사에서는 실제 limiter와 retry helper를 실행하되 기다림 분기에는 전용 스레드 interrupt를 사용합니다.

배치는 실제 대상 필터·날짜범위·재시도·결과 집계를 실행하고, 달력/Repository/종목별·시장별 서비스는 mock입니다. 달력의 previousTradingDay를 고정하여 현재 시각에 의존하는 배치 끝 날짜를 2026-09-24로 통제합니다. 15:40 경계는 실제 private 날짜 함수를 ReflectionTestUtils로 직접 호출하여 별도 검사합니다. 이는 Spring bean wiring이나 실제 휴장일 데이터 검증은 아닙니다.

스케줄러는 실제 객체를 직접 호출합니다. 타이머나 Spring application context를 시작하지 않으며 작업 본체는 mock입니다. 산업지수 job도 공유 scheduler의 독립 guard 검사에 포함되지만 산업지수 수집 구현의 통과로 계산하지 않습니다. 실제 외부 호출·DB·Redis·메일·크레딧 변경과 비용 발생은 모두0입니다.

## 실행 명령과 기록

Windows PowerShell에서 backend 디렉터리 기준:

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY -ErrorAction SilentlyContinue
$env:GRADLE_USER_HOME = "$PWD\src\test\.runtime\gradle-home"
$env:TEMP = "$PWD\src\test\.runtime\tmp"
$env:TMP = $env:TEMP
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*InvestorFlowAdminFlowTest' --tests '*InvestorFlowBatchTest' --tests '*KisBatchSchedulerTest'
```

처음 API30개 실행은25 PASS·5 FAIL이었습니다. 이 중2개는 테스트의 오류 경로 JSONPath를 공통 envelope에 맞추지 않은 문제와 공백 query를 이중 URL 인코딩한 문제였습니다. 각각 `$.errors[0].code`와 명시적인 URI 입력으로 테스트만 수정했습니다. 수정 후 API30·배치16·scheduler4의50개를 실행해47 PASS·제품 결함3 FAIL을 확인했습니다. 토큰 공유 검사는 Sinks gate로 토큰 응답을 보류하여 두 publisher의 요청이 실제로 겹치도록 보강했습니다.

전체 회귀는 위 명령에서 `--tests` 필터를 제거해 실행합니다. 최신 합계와 기존 결함 목록은 [전체 검증 현황](../../../docs/verification.md)을 참조하세요. XML은 `.runtime/build/test-results/test/TEST-com.qaima.verification.{클래스명}.xml`, HTML은 `.runtime/build/reports/tests/test/index.html`에 있습니다. 필터 실행은 이전 XML을 덮어쓸 수 있습니다.

## API·client·service: 30개

기준일은 2026-09-24, 종목 id7/코드000001입니다. 원천 숫자23개에 서로 다른 양수/음수 소수 값을 넣고 snake_case→entity→Public camelCase의 매핑을 개별 비교합니다. 숫자 정밀도 검사는 Java/JSON 값 기준이며 DB scale 보존까지 검증하지 않았습니다.

| 메서드 | 입력·수행·기대/관측 | 결과 |
|---|---|---|
| `stockSyncMapsEveryRawNumberThroughEntityAndPublicDto` | 1.XKRX→000001; 원천 숫자23개를 저장객체와 응답의 독립 기대값에 대조; J/KIS/TR ID·기간 min/max·요청코드 보존·entity 관계/ID 비노출 | PASS |
| `tokenJsonHeadersStockQueryAndPermitSubscriptionsAreExact` | 토큰 POST JSON의3필드·content-type, Bearer/app headers/TR ID/custtype, 종목 GET query5필드·XKOS 제거·NX override·permit2구독 | PASS |
| `concurrentStockAndMarketPublishersShareOneTokenRequest` | 토큰 응답을 Sinks.One으로 보류→종목/시장 두 publisher 동시대기·토큰전송1, 해제후 각자료1; 후속 요청도 토큰 재사용·총permit4 | PASS |
| `repeatedStockDateReusesEntityAndReplacesValues` | 같은 종목/날짜/KIS 두번 수집→객체1개 유지, 숫자1,234→1234·save2 | PASS |
| `stockSourceDatesAreFilteredButLatestPreservesProviderOrder` | 최신·과거·미래·2월30일 응답→유효2행만 저장, min/max 정확·rows는 제공처 순서 유지 | PASS |
| `stockMalformedNumbersBecomeNullWithoutRejectingRow` | 23숫자에 공백/dash/NaN→모두null, 행 자체는 저장·성공 | PASS: 관측 |
| `stockBackfillUsesTwentyOneDayAnchorsInclusiveFilteringSortingAndDeduplication` | 43일차 범위→to/to-21/to-42/from4앵커; 밖의행 제외, 중복2행은 save8회 후 결과2개·날짜오름차순 | PASS |
| `equalBackfillDatesMakeOneRequestAndServiceDefaultsUseSixtyDays` | from=to→1회; 서비스 null기간→Seoul오늘~60일전4앵커. HTTP는 기간필수라 별도직접호출 | PASS |
| `stockLatestWithoutDateUsesSeoulToday` | asOfDate 생략→Seoul 현재날짜 query. 전후 날짜를 기록하여 자정 경계 허용 | PASS |
| `stockSeriesIsReadOnlyAndClampsLimitFromOneTo365` | limit0/1/30/999→PageRequest1/1/30/365·내림차순 DTO, 제공처/limiter/save0 | PASS |
| `timeLimitLatestUsesOnlyLastExistingRowAtOrBeforeAsOfDate` | OPSQ2001→기존 과거행 중 최신1개, 미래행 제외·save0인데 savedCount1 | PASS: fallback 관측 |
| `timeLimitBackfillFallsBackToExistingRangeWithoutWrites` | TIME LIMIT 문구→각앵커마다 기존기간조회, 결과중복제거·save0 | PASS |
| `missingOutputAndTimeLimitWithoutStoredDataReturnEmptySuccess` | rt_cd0의 output누락/null 또는 운영시간제한+저장자료없음→두서비스200/빈rows/saved0/기간null | PASS |
| `marketSyncNormalizesAliasesFiltersDatesSortsAndMapsSixFields` | kosdaq→KSQ/1001·6숫자·KIS/시장TR ID·범위밖제외·오름차순; query의 날짜1/2는 모두to | PASS |
| `marketCustomIndustryAndRepeatedNaturalKeyUpdateExistingEntity` | KOSPI/KSP+industry0002 두번→같은객체 갱신·-1,234→-1234·save2 | PASS |
| `marketInvalidDatesAreSkippedAndAllMalformedNumbersBecomeNull` | 잘못된날짜 제외, 숫자6개 invalid문자열은null로 저장 | PASS: 관측 |
| `marketSeriesUsesDefaultKspIndustryAndClampsWithoutExternalCall` | 기본KSP/0001·limit1~365·내림차순, KOSDAQ별칭→KSQ/1001, 원천/save0 | PASS |
| `marketTimeLimitUsesStoredInclusiveRangeAndDoesNotSave` | 운영시간제한→양끝포함 저장범위2행·savedCount2/save0 | PASS |
| `marketServiceDefaultsBothDatesToTodayAndBlankMarketToKsp` | 서비스의 공백시장/null날짜 직접호출→KSP/0001/Seoul오늘 | PASS |
| `unknownMarketIsCurrentlyForwardedWithKospiDefaultIndustry` | unknown→UNKNOWN/0001로 원천 전송; 지원시장 사전차단 없음 | PASS: 관측 |
| `malformedInputAndReversedRangesStopBeforeTokenOrStorage` | 필수항목/날짜오류·공백종목·역순·잘못된limit→400, 전송/저장소/limiter0 | PASS |
| `unknownStockSyncMustReturnExplicitBadRequest` | 미등록종목 sync→코드상 의도된400/error envelope 기대; 실제200/빈body. lookup1·외부/flow저장소0 | FAIL |
| `unknownStockBackfillMustReturnExplicitBadRequest` | 같은 조건 backfill→200/성공envelope·savedCount0·rows[]·errors[]; 의도된오류아님 | FAIL |
| `unknownStockSeriesMustReturnExplicitBadRequest` | 같은 조건 series→동일200/빈body | FAIL |
| `providerHttpDecodeAndBusinessErrorsHaveDistinctCodesAndNoWrites` | 두client 각각429/503→HTTP_ERROR, 비JSON/빈body→DECODE_ERROR, rt누락/거부→BIZ_ERROR; 자동재시도0 | PASS |
| `businessErrorsAre502AndNotMistakenForTimeLimitFallback` | 정상HTTP의 biz거부→두관리자502/KIS_BIZ_ERROR, 저장소fallback0 | PASS |
| `tokenHttpFailurePreventsDataRequestAndWrites` | 토큰401→관리자5xx, 인증1회/permit1·자료요청0/저장0 | PASS |
| `nullClientArgumentsStopBeforeTokenAndRateLimiter` | 종목/날짜/시장null·역순→IllegalArgumentException, 토큰/limiter0 | PASS |
| `laterSaveFailureLeavesEarlierSaveInvocationForBothServices` | 두서비스 두번째save에서예외→500·상세비공개, 첫행mock저장 유지·save2. 실제DB 부분커밋 증거는 아님 | PASS |
| `backfillSecondAnchorErrorStopsSubsequentRequestsAfterEarlierSave` | 첫앵커 저장 후둘째503→502·요청2에서중단·첫save1 유지 | PASS |

실패 원인과 영향은 [INVESTOR-STOCK-MISSING-001](../../../docs/findings.md#investor-stock-missing-001--미등록-종목-수급-api의-빈-성공-응답)에 기록했습니다. 빈목록으로 임의 수용하도록 기대값을 바꾸지 않았습니다.

## 배치·재시도·rate limiter: 16개

아래는 모두 PASS입니다. 배치의 target 합계는 success+skipped+empty+timeLimited+failed와 대조하고, retried/retryRecovered는 별도 중첩 집계로 검사합니다. 일반 테스트의 retry delay와 종목간 sleep은0으로 설정합니다.

| 메서드 | 입력·수행·기대/관측 |
|---|---|
| `effectiveDateUsesSeoul1540BoundaryAndCalendarHoliday` | UTC로 표현한 동일시각을 KST로 환산: 15:39:59→이전거래일,15:40/23:59:59→오늘; 휴일18시→이전거래일 |
| `marketNoHistoryUsesInclusiveThirtyDayLookbackAndBothIndustryCodes` | 이력없음→KSP/0001,KSQ/1001 둘다 to-30~to 요청·target2/success2/saved2; 달력일 양끝포함이면31일인 범위 |
| `marketUpToDateSkipsAndStaleHistoryUsesTailCap` | 오늘자료있는시장skip,40일오래된시장은최근10일만 요청. 오래된 공백 전체복구라고 해석하지 않음 |
| `stockTargetsFilterDomesticEquityThenPrioritizeMissingAndOldestHistory` | 국내3시장 EQUITY/null asset 포함; 상폐/ID없음/거래소없음/null/미국/ETF 제외; 자료없음 우선→오래된날짜→코드순,limit3; 신규30일·기존날짜-2일 범위 |
| `stockLimitZeroIsUnlimitedAndCurrentOrFutureHistorySkips` | limit0은무제한, 오늘/미래최신행은모두skip·서비스0 |
| `lookbackAndRefreshTailAreClampedToAtLeastOneDay` | lookback0/refreshTail-2→최소1일 전부터 요청 |
| `marketHttpRetryRecoveryAndEmptyPublisherHaveSeparateCounters` | KSP HTTP오류후2행성공·KSQ Mono.empty→success1/empty1/retried1/recovered1/saved2 |
| `marketRetryExhaustionAndTimeLimitedTargetDoNotStopOtherTargets` | KSP HTTP지속오류2회·KSQ time-limit1회→failed1/timeLimited1/retried1/failedTargets[KSP] |
| `stockBatchSummarizesSuccessEmptyTimeLimitedFailedAndRecoveredRetry` | 5종목:정상1행/빈목록/time-limit/decode오류/HTTP오류후2행복구→success2/empty1/timeLimited1/failed1/retry1/recovered1/saved3 |
| `retryAfterHttpThenTimeLimitIsCurrentlyCountedAsFailedWrappedError` | 첫HTTP오류→재시도에서time-limit→RetryFailedException으로감싸져 timeLimited0/failed1/retried1. cause를 풀어 분류하지 않는 현재 동작 |
| `retrySupportRecoversWrappedNetworkErrorsAndReportsAttemptCount` | RuntimeException의cause가SocketTimeoutException→2번째성공·attempts2/recovered=true |
| `retrySupportDoesNotRetryBusinessDecodeOrTimeLimit` | BIZ/DECODE/MARKET_CLOSED는각1회로종료·원래ErrorCode 유지 |
| `retrySupportNullOrZeroAttemptsStillCallsOnceAndExhaustionPreservesCause` | retry설정null/max0→최소1회, max3의지속HTTP오류→attempts3·원래cause보존 |
| `interruptedRetryDelayPreservesInterruptAndDoesNotRunNextAttempt` | 전용스레드를interrupt후재시도대기100ms분기에진입→IllegalStateException/interrupt보존/다음시도0. finally에서스레드종료 |
| `rateLimiterExpiredWindowResetsAndMonoIsLazy` | window를3초전/사용량15로설정→Mono생성만으로변경0,구독후window초기화·사용량1 |
| `rateLimiterClampsToHardFifteenAndMinimumOneWithoutSleeping` | 설정0/100·사용량1/15에서각대기분기진입을interrupt예외로검출;1~15clamp확인. 느린CI에서도window미만료를유지하도록미래시각fixture사용,실제처리량/실시간대기정확성검사아님 |

## 공유 scheduler: 4개

| 메서드 | 입력·수행·기대값 | 결과 |
|---|---|---|
| `offSwitchesPreventAllThreeJobs` | 산업지수/시장수급/종목수급 enabled=false→모든작업0 | PASS |
| `eachJobReleasesGuardAfterExceptionAndNullSummary` | 각작업 첫예외·둘째null summary·이후2정상→4번모두진입,예외외부전파0·guard복귀 | PASS |
| `eachGuardRejectsSameJobOverlapButAllowsOtherJobsAndLaterRun` | 각작업을전용executor와latch로보류→같은작업중첩차단·다른2작업은실행가능·해제후재실행가능 | PASS |
| `allCronAnnotationsUseSeoulWeekdayDefaults` | 실제 @Scheduled3개 reflection: Seoul/평일·산업18:10·시장16:30·종목19:00 기본cron 확인. scheduler엔진발화검사아님 | PASS |

스케줄러 동시성 검사에는 진입latch3초·작업해제/완료5초·executor종료5초 상한이 있습니다. finally에서 해제·취소·종료를 확인하며 고정 sleep으로 진입 여부를 추정하지 않습니다. 단일 객체 내부 guard 검사로, 다중 JVM 분산 중복 실행 방지는 검증하지 않았습니다.

## 전체 실행 결과와 해석

신규50개를 포함한 전체 회귀는 **770개 중747 PASS·19 FAIL·4 SKIP**입니다. 첫 전체 실행은41초/종료코드1이었고, 오류본문 검사 보강 후에도 같은 합계를 재확인했습니다(38초/종료코드1). 기존16실패에 미등록종목3실패가 추가됐고, XML의 errors는0입니다. 테스트 실패를 제품 수정으로 해소하지 않았습니다.

결과 확인 중 backfill은sync/series와달리 collectList가빈스트림을빈목록으로바꾸어성공envelope를반환함을구분했습니다. 이에오류본문검사를단순null여부가아닌errors내용까지보강하고문서를정정했습니다. sync/series는200/빈본문,backfill은200/빈rows성공응답이며세경로모두의도된400오류가아닙니다.

다음은 확정 결함과 구분한 관측입니다.

- 수집 응답 savedCount는 실제save횟수나DB영향행수가 아닙니다. time-limit fallback의 기존행도집계하고, backfill의중복저장은최종결과에서제거합니다.
- 시장기간 요청은 두원천날짜query에 모두to를전달하고 from은반환행의로컬필터에사용합니다. 현원천API의응답기간/페이지를실제확인하지않아기간전체수집보장으로기록하지않습니다.
- 잘못된 숫자문자열은 null로바뀌어행저장이진행되며 output누락/null은정상빈목록으로수용됩니다. 스키마변경탐지완료가아닙니다.
- 최신배치는 오래된누락전체가아니라최근refresh-tail범위만수집할수있습니다. 직접time-limit와HTTP재시도후time-limit의결과집계가다릅니다.
- 두번째저장/앵커실패이전에발생한save호출은이미완료된상태입니다. 실제DBrollback·원자성은별도검증이필요합니다.

아직 남은 범위: 실제KIS인증/만료/재발급·제공처규격과보존기간/페이지·실휴장일, 실제JPA/SQL·중복/동시성·정밀도/부분커밋, 관리자인증필터와업무handler결합, 적재→Feature2카드/종합분석→브라우저, Python수급배치전체, 실제cron·실시간rate limit성능·분산중복방지입니다. Swagger5경로는PARTIAL이며실제연동은NOT_RUN입니다.
