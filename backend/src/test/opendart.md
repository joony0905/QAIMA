# OpenDART 관리자·클라이언트·일일 동기화 검증

검증일: 2026-09-25. Windows Java17.0.12/Gradle8.14. 추가 **46개:44 PASS·2 FAIL**. `OpenDartClientTest`16개(14 PASS·2 FAIL), `OpenDartAdminFlowTest`25 PASS, `OpenDartSyncSchedulerTest`5 PASS. 실패2개는 같은 XML 회사명 파싱 결함의 entity/CDATA 입력별 회귀이며 skip하지 않습니다. 제품 코드·설정·기존 DB는 변경하지 않았습니다.

## 실행·증거

[공통 실행 준비](README.md)의 PowerShell 환경을 설정하고 backend에서 실행합니다. 실제 연동 opt-in 환경변수는 해제합니다.

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY -ErrorAction SilentlyContinue
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*OpenDartClientTest' --tests '*OpenDartAdminFlowTest' --tests '*OpenDartSyncSchedulerTest'
```

두 번째 필터 실행: 46개/실패2, BUILD FAILED, 약20초. XML은 `.runtime/build/test-results/test/TEST-com.qaima.verification.OpenDart*.xml`, HTML은 `.runtime/build/reports/tests/test/index.html`입니다. 이후 전체 실행이 같은 결과 위치를 갱신합니다. BUILD FAILED는 알려진 실패를 숨기지 않는 결과이며, 전체 기능 완료나 실제 제공처 장애를 뜻하지 않습니다.

이후 전체 회귀853개 중827 PASS·22 FAIL·4 SKIP, 약37초를 확인했습니다. 기존실패20개에위회사명2개가추가됐고다른기존검사의신규실패는없었습니다. XML tests/failures/errors/skipped 합계는853/22/0/4로독립대조했습니다.

초기40개 실행에서5개가 실패했습니다. 제품결함2개 외의3개는 테스트 자체를 보정했습니다: `%20` 이중 인코딩을 없애도록 URI 객체 사용, 유효하지 않은 날짜의 실제500 계약을 숫자/필수필드400과 구분, 0byte buffer가 빈 Mono가 아닌 파싱 예외가 되는 기대값 수정. 서비스는 수정하지 않았습니다. 이후 codec300KB·scheduler5개를 추가한46개 실행에서 제품결함2개만 남았습니다.

## 실제 코드와 대체 경계

`WebTestClient.bindToController` → 실제 `OpenDartAdminController`/`GlobalExceptionHandler` → 실제 corp/issued/daily 서비스 → 실제 `OpenDartClient`/WebClient codec → 합성 `ClientHttpConnector`입니다. Repository 두 개만 상태를 보존하는 mock으로 대체합니다. ZIP은 메모리에서 직접 만들고 파일로 풀지 않습니다. 발행주식수의 `(stockId, baseDate, shareType)` 키와 ID 부여를 메모리 맵으로 흉내 내어 생성/불변/수정과 종목 간 분리를 확인합니다. 이는 JPA SQL·유일성·commit·rollback·동시성을 증명하지 않습니다.

프로젝트의 `WebClientConfig.openDartWebClient` 팩토리를 사용하고 전송만 바꾸므로 실제 JSON decoder와16MiB 설정을 사용합니다. 300KB 응답 검사는 기본256KiB를 넘는 정상 응답을 대상으로 하며16MiB 정확한 경계 검사는 아닙니다. 합성 URI·API 키만 사용하고 실제 외부 호출·비용·잔액 차감은0회입니다.

클라이언트는 HTTP·decode·전송 오류 때 별도 `ProcessBuilder("curl", ...)`로 우회합니다. `OpenDartFixture`는 Mockito 초기화 뒤 Java17 테스트 JVM에 `SecurityManager.checkExec` 차단을 설치하고 클래스 종료 후 이전 관리자를 복원합니다. 기존 관리자가 있으면 권한 검사를 위임하며, 설치 실패 시 검증 실패로 종료합니다. 두 클래스의 `@Isolated`로 다른 JUnit 클래스와 병렬 실행되지 않도록 합니다. 세 오류 케이스는 curl 실행 시도1회를 기록한 뒤 **프로세스 시작 전에** SecurityException으로 막힌 것을 검사합니다. curl 실제 성공·stderr·종료코드·timeout 경로는 미검증입니다. 이 차단 API는 Java17에서 deprecated 경고가 있으며 향후 JDK에서 제거되면 안전장치 교체 전 테스트를 실행하지 않아야 합니다. 이 장치는 자식 프로세스 차단이지 범용 네트워크 격리 장치는 아닙니다. HTTP는 위 합성 connector가 처리합니다.

인증필터는 이 HTTP 흐름에 결합하지 않습니다. 별도369개 보안 matrix의 sentinel 검증과 실제 업무 handler 통합을 혼동하지 않습니다. Spring 전체 앱·스케줄러 엔진은 기동하지 않습니다.

## 클라이언트 16개

각 행은 하나의 테스트 메서드입니다. 별도 FAIL 표기 외에는 PASS입니다.

| 메서드 | 입력·실행 방법·독립 기대값 |
|---|---|
| `getQueryAndAnnualDefaultDecodeActualJson` | 단일 JSON row→실제 DTO. GET `/api/stockTotqySttus.json`, 합성key/corp/year 및 기본보고서11011을 URI에서 대조. 주식수799·보통주 복원, curl0 |
| `missingKeyAndRequiredParametersFailBeforeTransport` | 공백키의 corp/주식수, 공백회사코드·null보고서→IllegalState/IllegalArgument, 전송0·curl0 |
| `configuredCodecAcceptsJsonLargerThanDefault256KiB` | message30만자·정상row→실제 프로젝트 codec에서 row1개, fallback0 |
| `successNullListAndNoDataAreEmptyWithoutFallback` | 공백을 포함한000/list누락 및013/list존재→모두 빈목록, 전송2·curl0 |
| `providerBusinessErrorDoesNotInvokeCurl` | status020/010/공백→해당status를 가진 IllegalStateException, curl0. 비즈니스 오류는 fallback 뒤에서 처리됨 |
| `httpFailureAttemptsCurlButGuardPreventsStartingIt` | 합성503→curl시도1, SecurityException, 실제 프로세스 시작0 |
| `invalidJsonAttemptsCurlButGuardPreventsStartingIt` | 잘못된JSON→decode 오류의 curl시도1, 시작 전 차단 |
| `transportExceptionAttemptsCurlButGuardPreventsStartingIt` | corp전송예외→curl시도1, 시작 전 차단 |
| `zipSkipsDirectoriesAndNonXmlAndDecodesUnicodeAndBlankCode` | 디렉터리·txt·대문자XML ZIP→앞 항목 무시, Unicode/공백trim·BASIC_ISO_DATE·비상장stockCode=null, corp URL 검증 |
| `firstXmlEntryWinsAndLaterXmlIsNotMerged` | 첫XML은빈result·둘째XML은손상→빈목록. 현재 첫XML만 읽는 정책 관측 |
| `zipWithoutXmlAndMalformedXmlFailWithoutCurl` | XML없는ZIP·닫히지않은XML 각각 실패, 파싱오류는curl0 |
| `zeroBytePayloadFailsParsingWithoutCurl` | 0byte buffer→응답empty IllegalStateException, curl0. HTTP204/빈publisher 동작과 동일하다고 주장하지 않음 |
| `invalidModifyDateFailsWithoutCurl` | 20260230→날짜파싱실패, curl0 |
| `internalDtdEntityIsNotExpanded` | 내부DTD entity를회사명에서참조→파싱실패. 네트워크/파일 entity를 사용하지 않았으며 전체XXE공격군 검증은 아님 |
| `xmlEntityFragmentsMustPreserveCompleteCompanyName` | `QA &amp; Partners`→기대 `QA & Partners`, 실제 `Partners`: **FAIL** |
| `xmlCdataFragmentsMustPreserveCompleteCompanyName` | `QA<![CDATA[ & ]]>Partners`→기대 `QA & Partners`, 실제 `Partners`: **FAIL** |

## HTTP·서비스 25개

공통 stock fixture: 국내EQUITY/KOSPI 종목005930·005935, 전자는 XML stockCode5930→005930 정규화로 직접 매핑되고 후자는 동일5자리prefix 우선주 fallback입니다. 모두 새 메모리 객체이며 실제 종목 데이터는 읽거나 변경하지 않습니다.

| 메서드 | 입력·실행 방법·독립 기대값 |
|---|---|
| `corpSyncMapsCommonPreferredAndCountsNonListedEntries` | corp POST→XML2행(상장1/비상장1), matched2/direct1/fallback1/updated2. COMMON/PREFERRED·회사명·날짜·공통동기화시각, shares저장소0 |
| `repeatedMappingKeepsValuesButPreferredCountsUpdatedObserved` | 같은XML로2회실행→직접매핑unchanged1이나우선주updated1, 매번saveAll2종목. 최종값 동일; 집계 비멱등성 관측 |
| `emptyValidXmlClearsExistingMappingsAndShareClassesObserved` | 정상매핑뒤빈result 재실행→missing2/updated0, 두종목corp메타데이터null·OTHER·saveAll호출. 기존DB에서 수행하지 않음 |
| `ambiguousPrefixDoesNotAssignPreferredToWrongCorporation` | 동일prefix의직접종목2개/서로다른corp→우선주fallback0·missing1·OTHER |
| `fallbackRequiresSameMarketDomesticEquityAndAnExistingDirectStock` | 다른시장/외국/ETF의동일prefix는매핑제외; 직접기준종목제거후우선주도매핑해제 |
| `duplicateListedCodesCountTwiceAndLastEntryWinsObserved` | 5930/005930 두 XML행→listed2, 마지막corp/name이직접·fallback모두에반영 |
| `mappingRepositoryErrorStopsDailyBeforeSharesFetch` | saveAll예외→daily500, mapped-stock조회/주식수저장소0·corp전송1. 메모리 변경을 DB rollback 증거로 사용하지 않음 |
| `corruptArchiveStopsDailyBeforeMappingRepository` | ZIP내손상XML→daily500, 두Repository접근0 |
| `stockSyncMapsTenNumericColumnsMetadataAndSource` | 단일주식수POST→created1/checked1/noData=false. 10개Long값1001/902/103/14/25/36/28/799/48/751을독립목록과대조. 공시번호·법인구분·회사명trim·기준일2026-06-30·출처OPENDART_STOCK_TOTQY·stock객체일치 |
| `naturalKeyCreatesThenSkipsThenUpdatesSameEntity` | 동일row2회→created1/unchanged1·save1회; 발행수799→800→updated1·같은ID/객체·총save2회 |
| `commonPreferredAndTotalRowsUseDifferentNaturalKeys` | 보통주/우선주/합계3row→created3·키3개, 재실행unchanged3·추가save0 |
| `blankRowCorpFallsBackToStockAndNumericDashesBecomeNull` | row회사코드공백→stock회사코드, 숫자dash/쉼표공백/공백→null; 의미있는발행수799유지 |
| `rowCorporationOverridesRequestedCorporationObserved` | row의다른회사코드가저장객체에우선하며stock매핑은불변. provider응답identity대조가없다는관측 |
| `noDataAndNonMeaningfulRowsAdvanceToNextReport` | 첫013·둘째null/authorized-only·셋째유효row→checked3/save1. 실제요청순서를날짜별대상목록과대조; 날짜알고리즘은아래고정기대값검사로별도검증 |
| `allReportsEmptyReturnNoDataAndNoWrites` | 모든보고서013→noData=true·year/report=null·당일대상수만큼확인·shares저장소0 |
| `reportTargetDateThresholdsAndOrderUseFixedIndependentExpectations` | private 대상선택만 reflection으로3/30·3/31·5/14·5/15·8/14·8/15·11/14·11/15·12/31 고정검사. 기본전년Q3/H1/Q1·전전년annual4개에3/31전년annual,5/15금년Q1,8/15금년H1,11/15금년Q3를앞에추가해최대8개 |
| `missingBlankUnknownAndUnmappedStockHaveExplicitErrors` | query누락/공백/미등록400, 매핑없는종목500, provider/주식수저장소0 |
| `stockLookupTrimsButDoesNotZeroPadObserved` | 공백포함005930성공/조회trim; 5930은미등록400. corp XML정규화와단일종목조회규칙구분 |
| `malformedRequiredFieldsAndNumbersFailBeforeSaving` | se/접수번호공백·소수숫자·Long초과400, 잘못된날짜500; 모든경우save0 |
| `zeroAndNegativeShareCountsAreAcceptedObserved` | 발행수0·자사주-2→저장됨. 값정책관측이며금융유효성승인아님 |
| `batchIsolatesFailedStockAndCountsNoDataAsProcessed` | mapped3종목: provider020/013/정상→target3·processed2·noData1·failed1·created1, 셋째종목의행저장 |
| `secondRowSaveFailureLeavesFirstRepositoryCallButBatchCountsNoCreatesObserved` | 보통주save성공후합계save예외→batch failed1/created0, 첫mock행남음·save2시도. 실제DB부분commit확정은아님 |
| `dailyOrdersMappingBeforeMappedStockSelectionAndSharePersistence` | daily POST→corp저장→mapped선택→주식수조회·저장 순서. 매핑2·종목별행생성2 및nested DTO |
| `dailyReturnsSuccessfulEnvelopeWithPerStockFailuresInSummary` | corp성공/두종목provider020→HTTP200의nested failed2/processed0, 주식수저장소0 |
| `latestLookupGuardsNullsAndTrimsShareTypeWithoutProviderCalls` | null/미저장stock·공백type→빈Mono/저장소0; latest/type조회실제repository경계·trim·Optional.empty의빈완료, provider0 |

## 스케줄러 5개

daily service만 mock으로 대체하고 scheduler 메서드를 직접 호출합니다. 실행시계/cron엔진/여러프로세스 경쟁은 검사하지 않습니다.

| 메서드 | 입력·방법·기대값 |
|---|---|
| `offSwitchPreventsDailyServiceAccess` | enabled=false주입→daily접근0 |
| `successfulRunSubscribesAndWaitsForDailyResult` | Mono supplier counter→정상result를소비한구독1회 |
| `exceptionAndEmptyCompletionDoNotPreventSubsequentRun` | 오류→빈완료→정상3회호출모두예외비전파·daily3회 |
| `concurrentInvocationsBothEnterDailyServiceObserved` | 2스레드/latch로두실행이동시진입함을확인, bounded대기·finally해제/종료. 해당scheduler에중첩guard가없다는관측 |
| `cronUsesSeoulDaily0310Default` | annotation reflection→Asia/Seoul 및기본매일03:10; 실제스케줄기동아님 |

## 결함과 남은 검증

`OPENDART-XML-TEXT-001`: parser가각 CHARACTERS/CDATA 이벤트에서기존값을덮어써앞부분을잃습니다. [발견사항](../../../docs/findings.md)에두실패근거를기록했습니다. XML회사명→stock.dartCorpName으로전파될수있지만운영데이터의실제손상을확인한것은아닙니다.

빈정상XML의전체매핑해제, 우선주반복updated, 불일치row회사코드우선, 음수값허용, batch부분실패의성공envelope, scheduler동시진입은정책검토를위한관측으로분리합니다. 모든관측을확정결함으로간주하거나임의수정하지않습니다.

남은 범위: 실제OpenDART인증·응답규격/최신원천, curl실행경로의안전한실연동,16MiB상한·ZIP대용량/모든XML경계, 실제JPA/격리DB마이그레이션·유일성/rollback/경쟁, 관리자JWT와업무handler결합, 실제cron·분산중첩, 주식수→공개재무/시세→브라우저의종단간흐름. 따라서Swagger4개는PARTIAL/실제연동NOT_RUN으로표시합니다.
