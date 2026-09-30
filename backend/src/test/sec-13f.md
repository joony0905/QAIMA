# SEC 13F 파일·매핑·분기 집계 검증

검증일: 2026-09-25, Windows Java17.0.12/Gradle8.14. 추가40개 중37 PASS·3 FAIL: `Sec13fParserTest`9개(7 PASS·2 FAIL), `Sec13fAdminFlowTest`18개(17 PASS·1 FAIL), `Sec13fAggregateFlowTest`13 PASS. 필수열 누락을 성공으로 처리하는 문제2개와 불가능한 날짜를 보정하는 문제1개를 실패 회귀로 유지합니다. 제품 코드·설정·기존 데이터는 수정하지 않았습니다.

## 실행 방법·증거

[공통 Windows 실행 준비](README.md)의 Gradle 캐시·TEMP 설정을 test/.runtime으로 적용하고 backend에서 실행합니다.

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY -ErrorAction SilentlyContinue
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*Sec13fParserTest' --tests '*Sec13fAdminFlowTest' --tests '*Sec13fAggregateFlowTest'
```

최초 필터 실행은40개/실패3, 약22초, BUILD FAILED입니다. 테스트 코드의컴파일·기본fixture는통과했고기대했던입력검증회귀3개가실패했습니다. 날짜회귀의실패메시지에실제파싱날짜를추가했지만기대값이나제품코드는바꾸지않았습니다.

이후전체회귀는943개 중913 PASS·26 FAIL·4 SKIP, 약44초입니다. XML합계943/26/0/4(tests/failures/errors/skipped)를독립대조했고기존23실패외에는위3개만추가됐습니다. 날짜회귀에서31-Feb-2026의실제결과가2026-02-28임을확인했습니다. 40개모든테스트메서드의문서포함여부도대조했습니다.

XML은 `.runtime/build/test-results/test/TEST-com.qaima.verification.Sec13fParserTest.xml` 및 `Sec13fAdminFlowTest`/`Sec13fAggregateFlowTest`의동명파일, HTML은 `.runtime/build/reports/tests/test/index.html`입니다. 이후전체실행이같은위치를갱신합니다. 실패를skip/expected-failure로숨기지않습니다.

## 실제 처리와 대체한 경계

실제 관리자5경로의Controller/예외처리→`Sec13fInstitutionalHoldingImportService`→실제ZIP/TSV parser 또는분기계산→DTO를 `WebTestClient.bindToController`로실행합니다. 파일은 `@TempDir` 아래에서Java ZIP writer로생성하고테스트종료시JUnit이정리합니다. TEMP가test/.runtime으로지정되어있어프로젝트원천파일을덮어쓰지않습니다. 외부API·DB·메일·유료호출은0회입니다.

Repository 7개는메모리상태/명시적응답을가진mock으로대체합니다. 이력키는sourceFile, 공시키는accession, 보유키는accession/stockId/CUSIP, 집계키는stockId/period입니다. saveAll전달목록이제품에서clear되므로테스트는호출시점에객체/배치크기를복사해기록합니다. 다른SEC주식수동기화service는전케이스접근0을확인합니다. 실제TCP·JWT와업무handler결합·JPA·DB유일성/잠금/원자성은검증하지않습니다.

특히기관별최신공시·정정공시제외·기관수/행수/주식수합산은 `Sec13fHoldingRepository.aggregateLatestByStockIdsAndReportPeriods`의native SQL책임입니다. **테스트는이SQL을Java로흉내내지않고고정된projection을주입합니다.** 따라서SQL의ROW_NUMBER/NOT EXISTS/GROUP BY가MySQL에서올바르게실행된다는증거가아닙니다. 확인한것은SQL요청대상/100개분할과그뒤의실제분모선택·보유비율·전기증감·수정/삭제·DTO입니다. SQL정정공시정책의실증은격리DB검증으로남깁니다.

기본ZIP은 SUBMISSION/COVERPAGE/INFOTABLE TSV3개입니다. 공시a·manager-1·기준일2026-06-30·제출일2026-08-01, CUSIP QA0000001→stock QA/ID1,100주·원시금액200을사용합니다. 메모리객체는모두합성값입니다. 날짜표기는dd-MMM-yyyy이고헤더는TSV대문자입니다.

## 파서 9개

| 메서드 | 입력·수행·독립 기대값 |
|---|---|
| `zipTsvCombinesSameAccessionCusipAndTracksEveryFilter` | 같은a/CUSIP100+25주/200+50금액, 미매핑·PUT·PRN 각1→보유1행/원천합산2행/125주/250. submission1·cover1·info5·matched2·각skip1·parsed1 및날짜복원 |
| `nestedCaseInsensitiveFilenamesBomAndTrailingEmptyColumnsAreRead` | ZIP내하위폴더·대소문자다른파일명·BOM·공백CUSIP·쉼표숫자1234.5/2000→정확한BigDecimal·manager명 |
| `distinctAccessionOrCusipRemainSeparateGroups` | accession다름/CUSIP다름3조합→독립3행 |
| `duplicateSubmissionLastWinsButPhysicalRowCountIncludesBoth` | 같은accession의manager수정행추가→물리행수2·맵1·마지막manager채택 |
| `nullMappingsExcludeAllHoldingsAndBlankNumbersBecomeZeroObserved` | null매핑이면모두unmapped; 정상매핑/공백숫자는shares/value=0. 현재정책관측 |
| `malformedNumbersFailAndMissingZipMembersOrHeadersAreRejected` | 비숫자→NumberFormatException, 필수TSV파일누락·완전히빈header·nullpath→IllegalArgument |
| `missingRequiredColumnsMustNotSilentlyBecomeEmptyData` | INFOTABLE에WRONG_COLUMN만존재→필수열오류기대, 실제예외없음: **FAIL** |
| `impossibleCalendarDateMustNotBeSilentlyAdjusted` | 제출일31-Feb-2026→DateTimeParseException기대, 실제날짜보정/반환: **FAIL** |
| `negativeSharesAndFirstIssuerMetadataAreRetainedObserved` | -5+2주→-3, 뒤행회사명이달라도첫회사명보존. 정책관측 |

## 관리자 적재·매핑 18개

| 메서드 | 입력·수행·기대값/관측 |
|---|---|
| `fileHttpRunsActualZipParserAndPersistsFilingHoldingAndAudit` | 실제file POST→100+25주합산, matched2/parsed1/createdFiling1/createdHolding1/aggregate0, STARTED→SUCCESS/시각기록, 공시·종목관계·원시/정규화금액·출처대조,집계저장소0 |
| `repeatedFileReusesAuditAndNaturalKeysThenUpdatesChangedHolding` | 같은ZIP2회→두번째unchangedHolding1/createdFiling0; 파일내100→150주변경후재실행→updatedHolding1/같은객체·같은파일이력,holding save배치2회 |
| `numericScaleOnlyChangeDoesNotCauseHoldingUpdate` | 100/200→100.00/200.0→BigDecimal비교로unchanged·추가save0 |
| `marketValueUnitChangesExactlyAt2023January3FilingDate` | 구현의제출일경계02-Jan-2023→THOUSANDS_USD/200000,03-Jan-2023→USD/200. SEC실규정검증이아닌현재코드경계검사 |
| `coverPeriodFallbackWorksButMissingManagerSkipsParsedHoldingObserved` | submission기준일공백→cover Q2사용; manager공백이면parsedHolding1이나created0/SUCCESS. parser집계와service채택의차이 |
| `noCusipMappingsReturnsFailedAuditInsideHttp200` | 매핑전체없음→HTTP200 안FAILED·오류문구·STARTED→FAILED,공시/보유/집계접근0 |
| `missingFileOrBlankPathFailsBeforeAuditCreation` | 없는파일·공백path·query누락→400·모든적재Repository접근0 |
| `malformedNumericFileRecordsFailedThenValidRetryResetsAudit` | 숫자bad→FAILED,같은파일을정상합성값으로재작성후재시도→SUCCESS·error=null·created1·이력키1 |
| `holdingSaveFailureLeavesEarlierFilingCallAndZeroAuditCountsObserved` | 보유save예외→FAILED,오류1200자→1000자제한; 공시mock save1회완료이나이력created/info통계0. 실제DB부분commit주장아님 |
| `auditStartFailurePropagatesBeforeParserOrMappingLoad` | 시작이력save예외→500·매핑/공시/보유/집계0 |
| `directoryUsesFilenameOrderSuffixFilterLimitAndContinuesAfterFailure` | a_form13f(손상숫자),b_form13f(정상),ignored.zip. 기본sourceDir/limit1→a만실패; 명시dir/limit-1→이름순2개·success1/failed1·뒤파일계속 |
| `missingDirectoryIs400AndEmptyDirectoryIsSuccessfulZeroFiles` | 없는dir→400; 빈기본dir→fileCount0/이력0 |
| `highestConfidenceCusipMappingWinsAndEqualConfidenceKeepsFirstObserved` | 같은CUSIP의서로다른stock/신뢰80·90→90선택; 동률90·90→조회순첫매핑선택. 관리자충돌차단과기존충돌데이터적재정책구분 |
| `thousandsOfRowsUse500LookupAnd1000SaveBatches` | 실제ZIP의공시/보유1001개→created1001,공시/보유lookup각500/500/1·save각1000/1. DB성능/commit시험아님 |
| `mappingUpsertNormalizesClampsReactivatesAndReportsCreateOrUpdate` | 공백stock/CUSIP소문자→trim/대문자·created. confidence150→100,-5→0,누락→100. 같은비활성매핑재요청→active=true/updated,issuer누락→null |
| `activeCusipConflictAndMissingInputsPreventMappingSave` | 다른stock의active CUSIP충돌·미등록stock·공백stock/CUSIP→400·identifier save0 |
| `malformedRequiredHeaderMustProduceFailedImportNotSuccess` | 필수INFOTABLE열없는ZIP의실제관리자POST→FAILED기대, 실제SUCCESS: **FAIL** |
| `restatementWithoutMappedRowsStillLoadsAffectedStocksAndDeletesObsoleteAggregate` | 매핑보유행없는RESTATEMENT→공시1개저장·해당manager/period영향stock조회·집계projection없음→기존Q2집계삭제1/holding0. SQL에서정정이기존보유를제외하는지는mock경계밖 |

## 분기 집계·조회 13개

주입projection의기관수2·행수3·주식수/금액을입력으로보고독립산술과비교합니다. 단위는보유비율·증감률모두0~1형식의fraction이며%문자열이아닙니다.

| 메서드 | 입력·수행·독립 기대값 |
|---|---|
| `rebuildComputesShareChangeFractionalRateAndHoldingRatio` | 이전80/현재100/발행800/금액2000→증감20/증감률0.25/보유비율0.125·기관2/행3/CUSIP/source·stock1/period1/changed1 |
| `initialPeriodAndMissingOrZeroDenominatorsRemainNull` | 이전/발행없음→증감/증감률/분모/보유비율null; 이전0/발행0→증감100이나두비율null |
| `commonBeforeOrOnPeriodOutranksNewerTotalAndFutureCommonIsIgnored` | 미래COMMON9999제외·이전COMMON800이당일TOTAL1000보다우선;COMMON제거후TOTAL·TOTAL제거후OTHER500 fallback |
| `fractionalRatiosUseEightPlacesHalfUpAndNegativeChangeIsPreserved` | 이전3/현재1/분모6→증감-2·증감률-0.66666667·보유비율0.16666667,8자리HALF_UP |
| `repeatedProjectionDoesNotSaveUnchangedAggregate` | 같은projection2회→changed1후0·집계saveAll1회 |
| `changedEarlierPeriodAlsoRecalculatesNextExistingPeriodsDelta` | Q1=80/Q2기존100→120/Q3=150→Q2와Q3재계산·changed2, Q3증감30/률0.25·두기간SQL요청확인 |
| `absentProjectionDeletesAggregateAndNextUsesEarlierSurvivingPeriod` | Q2projection없음/Q3=150→Q2삭제, Q3는살아있는Q1=80과비교해70/0.875·changed2 |
| `stockLimitRetainsAllPeriodsForFirstStockAndDeduplicatesPeriods` | 두종목/기간중복·limit1→첫stock의모든2기간·중복제외,limit-1→두stock/3기간 |
| `noPeriodsAvoidsAllAggregateAndStockLookups` | distinct기간없음→changed0·stock/quarter/issued/aggregateSQL경계0 |
| `aggregateProjectionRequestsSplitAt100StocksPerPeriod` | 같은기간101종목→집계SQL경계100/1분할·changed101. 실제SQL성능검증아님 |
| `fileAggregateOptionConnectsImportToProjectionMathAndPublicQuery` | file POST aggregate=true→집계1,이어GET→stock/기준일/100주/분모1000의0.1·source JSON. 원천합산projection은명시mock |
| `publicQueryAppliesDefaultAndClampedLimitsAndRejectsMissingStock` | GET기본limit12/0→1/999→120의PageRequest검사;미등록stock/query누락400 |
| `aggregateRepositoryFailureIs500ButFileImportRecordsFailedSummary` | 같은집계조회예외가rebuild에서는500, file import에서는HTTP200/FAILED. 보유mock save는앞서완료됨 |

## 발견사항과 남은 검증

`SEC13F-HEADER-001`: header행존재만검사하고필수컬럼이없으면빈문자열로읽어행을제외합니다. 결과적으로잘못된스키마파일이빈정상데이터처럼SUCCESS가됩니다. parser/HTTP회귀2개FAIL입니다.

`SEC13F-DATE-001`: `DateTimeFormatterBuilder`의기본SMART해석으로불가능한날짜를월말로바꿉니다. 실제잘못된날짜를거부해야하는회귀1개FAIL을유지합니다. [발견사항](../../../docs/findings.md)에근거를기록합니다.

공백숫자0·음수허용·중복행첫issuer메타데이터·동일신뢰CUSIP조회순선택·manager누락행의SUCCESS·부분저장호출후0집계·HTTP200내FAILED는확정결함과구분한관측입니다. 허용범위를넓혀제품을임의수정하지않습니다.

남은범위: 실제원천ZIP/SEC규격, MySQL native SQL의기관별최신공시/정정공시/복수CUSIP합산·유일성·원자성·동시성·migration, 1000행집계저장/삭제와500종목조회경계전체, BOM외의TSV인용/모든날짜·필수열조합, 실제관리자JWT와파일접근정책, 운영규모파일메모리/성능, 공개Feature2카드→브라우저까지의종단간흐름. 따라서5개Swagger경로는PARTIAL이며실제DB연동은NOT_RUN입니다.
