# 시장 스냅샷 backfill·주식수 기준 CI/QA 기록

검증일: 2026-09-25. Windows Java17·Gradle8.14. 테스트 소스는 [MarketSnapshotAdminFlowTest](java/com/qaima/verification/MarketSnapshotAdminFlowTest.java)와 [ShareBasisResolverTest](java/com/qaima/verification/ShareBasisResolverTest.java)입니다.

최종 추가 **37개 중36 PASS·1 FAIL**: 관리자/계산연결27개 중26 PASS·1 FAIL, 주식수10 PASS. 전체 회귀는807개 중783 PASS·20 FAIL·4 SKIP, 36초/종료코드1입니다. XML 합계의 errors는0이며 기존19실패에 성장률1개가 추가됐습니다.

## 검증한 흐름과 대체 경계

관리자 단일·누락분 backfill HTTP부터 실제 `MarketSnapshotBackfillService → MarketSnapshotService → ShareBasisResolver + SnapshotCalculator → TransactionTemplate → MarketSnapshotCacheService → JSON → DTO`를 연결했습니다. 실제 Redis 대신 메모리 Map과 ReactiveValueOperations mock을 사용하며, Repository·KIS client·트랜잭션 관리자도 mock입니다. 금액·비율·주식수 계산과 serializer는 실제 코드입니다. `MarketMetricService`는 mock이고, batch 저장 과정에서 호출되지 않는 것을 확인합니다.

핵심 fixture는 종목 id7/QA0001/KOSPI, 보통주 발행100·자기주식20·유통60, 전체 발행200입니다. 실제 resolver의 outstanding은80, EPS/BPS 평가주식수는200입니다. 연간 재무는 매출1000·영업이익200·순이익80·자본400·자산800·부채400, 유동자산300·유동부채150·재고60·이자10·영업현금흐름90·CAPEX15+5입니다. 독립 기대값은 EPS0.4/BPS2/SPS12.5, ROE20/ROA10/영업이익률20/순이익률8/부채비율100, 유동비율200/당좌비율160/이자보상20/FCF70입니다. 퍼센트와 배수의 의미를 구분합니다.

스냅샷 날짜는 제품이 `LocalDate.now()`와 같은 날만 허용하므로 Windows JVM의 실행일을 사용합니다. 사용자의 기존 DB나 날짜를 변경하지 않습니다. 전체 Spring context·scheduler·외부 HTTP·실제 MySQL/Redis·메일/LLM·크레딧은 실행하지 않았습니다. mock 트랜잭션의 commit/rollback 호출은 실제 DB 원자성을 증명하지 않습니다.

## 재현 명령

Windows PowerShell, backend 디렉터리 기준:

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY -ErrorAction SilentlyContinue
$env:GRADLE_USER_HOME = "$PWD\src\test\.runtime\gradle-home"
$env:TEMP = "$PWD\src\test\.runtime\tmp"
$env:TMP = $env:TEMP
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*MarketSnapshotAdminFlowTest' --tests '*ShareBasisResolverTest'
```

최초 관리자 흐름27개 실행은26 PASS·1 FAIL, 19초/종료코드1입니다. 실패는 동일 연도 재무 수정본을 전년도처럼 비교하는 회귀 검사입니다. 제품은 수정하지 않았고 실패를 skip/expected 처리하지 않았습니다. ReactiveValueOperations의 generic mock에 unchecked compile 경고가 있었지만 컴파일은 통과했습니다. 주식수 단독10개를 추가한 최종 실행 결과는 아래에 기록합니다.

XML은 `.runtime/build/test-results/test/TEST-com.qaima.verification.{클래스명}.xml`, HTML은 `.runtime/build/reports/tests/test/index.html`입니다. 필터 없는 전체 회귀 합계는 [검증 현황](../../../docs/verification.md)에 반영하며, 이후 필터 실행이 XML을 덮어쓸 수 있습니다.

## 관리자·계산·저장·캐시 연결: 27개

| 메서드 | 입력·수행·기대/관측 | 결과 |
|---|---|---|
| `annualBackfillRunsActualShareMathStorageCacheAndPublicDto` | 실제 주식수/재무 계산과 위 독립 기대값; OPENDART_PRIMARY·연간fallback 경고; commit1/rollback0; snapshot 1h/latest 60s SET 인자·JSON 역복원/캐시조회; 실시간/KIS 호출0 | PASS |
| `forceUpdatesSameEntityAndClearsAllRealtimeDependentColumns` | 완성된 기존 객체에 force=true→동일 객체 갱신, marketCap/floatMarketCap/PER/PBR을null로 지움·실시간 호출0 | PASS |
| `completeSnapshotSkipsUnlessForcedButStillBuildsDto` | 완성된 오늘 자료→SKIPPED/ALREADY_FILLED·PSR2, 쓰기/tx/캐시0. DTO 생성용 재무조회는1회 | PASS |
| `eachMissingCoreFieldTriggersBackfill` | shares/EPS/BPS/SPS/source 각각null, source공백→6회 모두 UPDATED/save | PASS |
| `ownershipMetricsAreRequiredOnlyForOpenDartPrimary` | float/treasury ratio 누락: OPENDART_PRIMARY(대소문자무시)는 갱신, BATCH/KIS fallback은 skip | PASS |
| `exchangeAliasesAndTrimmedCodeResolveBeforeGlobalFallback` | XKRX/KRX/kospi→KOSPI, XKOS→KOSDAQ; 일치하면 전역검색0, NYSE 요청이 실패하면 코드 전역검색으로 KOSPI 종목을 선택하는 현재 동작 | PASS |
| `unknownStockAndInvalidDatesOrParametersRejectBeforeWrites` | 미등록/누락/공백종목, 어제/내일/잘못된날짜, force/limit 자료형오류→400·계산/저장/tx0 | PASS |
| `unsupportedExchangeReturnsSkippedWithoutSnapshotOrProviderAccess` | NASDAQ/NYSE/XKRX/공백/거래소null인 실제 Stock→NOT_ELIGIBLE·스냅샷/제공처/tx0. 입력 exchange alias 정규화와 엔티티 eligibility는 별개 | PASS |
| `missingBatchFiltersCompleteAndIneligibleStocksAndUsesAllListing` | ALL 목록의 완성자료·해외 제외, KOSDAQ/KONEX 미완성2개만 순서대로 갱신 | PASS |
| `missingBatchLimitOneMustNotWriteMoreThanOneSnapshot` | 후보4개·delay50ms·limit1→결과1·save1·메모리 스냅샷1 | PASS |
| `missingBatchNonpositiveLimitDefaultsToFifty` | 후보52개·limit0·delay0→결과50; 기본 KOSPI 목록조회. 결과 상한 검사이며 모든 취소 경쟁상태 검증은 아님 | PASS |
| `snapshotWriteFailureBecomesFailedResultRollsBackAndDoesNotCache` | save 예외→HTTP200 안의 FAILED·합성 예외문구, rollback1/commit0·캐시0 | PASS: 관측 |
| `missingBatchContinuesAfterPerStockWriteFailure` | 첫 종목 저장실패·다음 정상→FAILED/UPDATED 두 결과·rollback1/commit1 | PASS |
| `preflightRepositoryFailureAbortsBatchRatherThanBecomingPerStockResult` | 기존자료 확인 중 오류→전체HTTP500·상세 비공개·계산/tx0. 저장단계 오류와 다른 처리 | PASS |
| `bothRedisSetFailuresDoNotUndoSavedSnapshot` | 두SET 실패→UPDATED 유지·commit1/rollback0·SET2회 | PASS |
| `kisSharesFallbackUsesListedCountAndHasNoOwnershipMetrics` | 발행주식 자료 없음·합성 KIS 상장주식1,000→shares1000/EPS0.08/KIS_SHARES_FALLBACK, 소유비율null; 다음호출 skip | PASS |
| `absentOrMalformedKisSharesStillStoresIncompleteBatchSnapshot` | 주식수 자료없음·KIS 숫자변환실패→shares/EPSnull인데 UPDATED/BATCH_SNAPSHOT, 다음호출도 UPDATED/save2 | PASS: 관측 |
| `absentFinancialsDiscardEvenResolvedShareValuesAndRemainIncomplete` | 재무없음→이미 주식수를 구했어도 계산기 empty snapshot 때문에 shares/EPSnull·OPENDART_PRIMARY·반복갱신 | PASS: 관측 |
| `contiguousQuartersUseTtmIncomeButLatestBalanceAndCashFlowAnchor` | 연속4분기 각매출250/순익20→TTM EPS0.4/SPS12.5, 최신분기 재무상태·OCF90/CAPEX20, 연간경고없음 | PASS |
| `duplicatedQuarterVersionsSelectLatestBeforeTtmCalculation` | Q4의구버전9999와신버전250을혼합→최신version 선택 후 TTM0.4,경고없음 | PASS |
| `quarterGapFallsBackToAnnualAndNullTtmFieldDoesNotInventValue` | Q2 누락→연간fallback0.4/경고; 연속4분기 중순익null→EPSnull·SPS12.5 유지 | PASS |
| `zeroDenominatorsReturnNullRatiosWithoutInfinityAndCapexAllowsOneSide` | 주식수/매출/자본/자산/유동부채/이자0→비율null, Infinity/NaN 없음; CAPEX한쪽null은다른15 사용·FCF75 | PASS |
| `annualGrowthUsesRevenueAndDateSpecificValuationShares` | 현재 매출1000/전년800→25%; 순익80/40·평가주식200/100→EPS성장0%. 각reportDate 주식수 조회 | PASS |
| `annualGrowthMustCompareDistinctYearsInsteadOfTwoVersionsOfOneYear` | 2025v2(1000/80),2025v1(900/70),2024(800/40)→기대 성장25%/100%; 실제 같은2025 수정전값과 비교하여11.111111%/14.285714% | FAIL |
| `eightContiguousQuartersCompareTwoTtmWindowsForGrowth` | 연속8분기·현재4개250/20,이전4개200/10→매출25%·EPS100% | PASS |
| `negativePriorIncomeGrowthUsesSignedDenominatorAndZeroPriorRevenueIsNull` | 전년매출0→성장null,전년순익-40/현재80→signed 분모로-300%. 해석상의 주의점 기록 | PASS: 관측 |
| `nullStockOrSnapshotDirectServiceInputsReturnEmpty` | 직접service null Stock→empty, toDto(null)→null, 모든하위경계0 | PASS |

## 주식수 선택·fallback: 10개

| 메서드 | 입력·수행·기대값 | 결과 |
|---|---|---|
| `commonUsesNetOutstandingButTotalIssuedForValuation` | 보통주100/자사20/유통60·전체발행200→outstanding80/valuation200, KIS0 | PASS |
| `preferredUsesPreferredClassAndDoesNotFallBackToCommon` | 우선주40/자사5→35,전체200으로평가; 우선주행제거시보통주가있어도missing | PASS |
| `primaryFallbackPriorityIncludesSecCommonAndDomesticOrSecTotal` | 보통주→COMMON→합계→TOTAL 우선순위를각각단일자료로검사; 최초적중에서종료 | PASS |
| `secTotalOverridesValuationDenominatorAndMissingTotalFallsBackToPrimary` | COMMON100/자사10·TOTAL150→90/150, TOTAL제거시valuation100 | PASS |
| `optionalTreasuryAndExcessTreasuryHaveExplicitArithmetic` | 자사주null이면발행수그대로,자사주가발행수초과하면outstanding0;미지정유통수는추정하지않음 | PASS |
| `missingStockOrDateReturnsMissingBasisWithoutRepositoryAccess` | null Stock/날짜→primary null/missing basis·repo/KIS0 | PASS |
| `fallbackOptOutForeignMarketAndBlankCodeNeverCallKis` | allowKis=false,해외시장,거래소null,공백종목→KIS0 | PASS |
| `domesticAliasesPermitKisListedShareFallbackAndKeepNoOwnershipValues` | 국내6시장표기→KIS market J,1,234→주식수/평가수1234·float/treasury null·fallback출처 | PASS |
| `malformedEmptyAndFailedKisFallbackPreserveMissingBasis` | 상장수null/공백/비숫자,Mono.empty,전송예외→missing유지·예외외부전파없음 | PASS |
| `zeroIssuedValueStillPreventsKisAndAllQueriesReceiveRequestedAsOf` | 발행0도이미존재로간주하여fallback0; share-type 조회3회에요청asOf전달 | PASS |

위 표의 저장소는 반환값을 통제하는 mock입니다. 실제 JPQL의 과거일 필터·share class 데이터 품질·공시 시점·source별 의미·외부KIS의 상장수는 검증하지 않았습니다.

## 한계와 후속 검증

스냅샷의 UPDATED는 모든 필드가 완성됐다는 뜻이 아닙니다. 재무나 주식수가 없으면 불완전 스냅샷도 저장하고 다음 backfill에서 다시 처리합니다. source 태그도 주식수 입력에 따라 정하며 실제 모든 OpenDART 원천을 호출했다는 뜻은 아닙니다. 기존자료 조회오류는 전체실패, 저장오류는 종목별FAILED, Redis SET오류는UPDATED로 구분됩니다. 저장오류 문구는 관리자 결과 message에 그대로 들어가는 현재 동작입니다.

성장률 결함 `SNAPSHOT-GROWTH-VERSION-001`의 원인과 재현 값은 [발견사항](../../../docs/findings.md)에 기록했습니다. 해당 실패 검사는 제품 수정 권한 없이 제거하지 않습니다.

실제 MySQL의 migration/정밀도/트랜잭션·Redis TTL/동시성·제공처 연결, 보안필터와handler 결합, MarketMetricService의 시세통합→공개Stock/Feature1→브라우저 전체흐름, 과거공시/누락/중복·분기성장 전체경계, 대규모batch 취소/재실행은 남았습니다. 관리자2경로는PARTIAL, 실제연동은NOT_RUN입니다.
