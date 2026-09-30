# Feature3 가격·벤치마크 HTTP 흐름 검증

검증일: 2026-09-25. Windows Java17.0.12/Gradle8.14. `Feature3PriceHttpFlowTest` 15개 중12 PASS·3 FAIL, `Feature3BenchmarkHttpFlowTest` 14개 중13 PASS·1 FAIL. 합계29개 중25 PASS·4 FAIL이며, 제품 코드·설정·기존 데이터는 수정하지 않았습니다.

## 실행 방법·증거

[공통 Windows 실행 준비](README.md)의 Gradle 캐시·TEMP 설정을 test/.runtime으로 적용한 뒤 backend에서 실행합니다.

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY -ErrorAction SilentlyContinue
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*Feature3PriceHttpFlowTest' --tests '*Feature3BenchmarkHttpFlowTest'
```

첫 실행은29개 중7 FAIL이었습니다. 벤치마크 테스트가 실제 DTO의 `benchmarkAvailable` 대신 존재하지 않는 `available`을 읽어 정상4개가 실패하고 미래일자 검사가 거짓 통과했습니다. 테스트만 실제 계약에 맞추고 `required("benchmarkAvailable")`로 필드 존재도 강제했습니다. 재실행은29개 중25 PASS·4 FAIL, 약20초, BUILD FAILED입니다. 이4개는 아래 제품 동작 불일치를 재현하며 skip/expected-failure로 숨기지 않습니다.

XML은 `.runtime/build/test-results/test/TEST-com.qaima.verification.Feature3PriceHttpFlowTest.xml` 및 `Feature3BenchmarkHttpFlowTest`의 동명파일입니다. HTML은 `.runtime/build/reports/tests/test/index.html`이며 이후 실행으로 갱신됩니다. 전체 회귀 최신 집계는 [전체 검증 현황](../../../docs/verification.md)을 따릅니다.

이후 전체 회귀는972개 중938 PASS·30 FAIL·4 SKIP, 약40초입니다. XML 합계972/30/0/4(tests/failures/errors/skipped)를 별도 파서로 대조했고 기존26개 실패에 이4개가 추가됐습니다. 두 클래스29개 메서드가 아래 표에 모두 기록됐는지도 대조했습니다.

## 실제 실행 경로·대체한 경계

- 가격: WebTestClient→실제 `Feature3MarketDataController`/예외처리→`Feature3PriceSeriesService`→`YahooFeature3PriceProvider`→`YahooFinancePriceClient`의 실제 요청 구성·JSON 파서·품질 계산 또는 `CandleLoadService`의 실제 조회/보강/저장 대상 판정→DTO/공개 경고.
- 벤치마크: WebTestClient→같은 Controller→실제 `Feature3BenchmarkSeriesService`→읽기·품질 판정·보강·`IndustryIndexBar.toEntity`·재조회·중복 거래일 제거·DTO/경고.
- `bindToController` 방식이므로 실제 TCP 서버나 SecurityConfig/JWT 필터를 결합한 검증은 아닙니다. 전체 Spring application context와 scheduler를 시작하지 않습니다.
- Yahoo WebClient의 전송은 합성 connector로 대체하고 URI와 응답 JSON을 기록합니다. 외부 호스트가 요청 URI에 있더라도 네트워크로 보내지 않습니다. 원시가격 `StockClient`와 지수 `IndustryIndexFetcher`, 거래달력, 종목 조회, Repository는 mock입니다. 실제 금융 API·메일·DB·Redis·유료 호출은0회입니다.
- 가격 저장소는 미리 정한 목록을 반환하고 saveAll 인자를 기록합니다. 로더의 응답 병합과 저장 대상 선택은 실제 코드지만 SQL/commit/재조회 영속성 증거는 아닙니다. 벤치마크 저장소는 합성 메모리 목록에 저장한 뒤 실제 서비스가 재조회하게 합니다. DB 정렬·페이지의 의도된 동작을 fixture로 적용했으며 실제 JPQL 실행은 아닙니다.
- 가격 테스트는 실행 시 UTC 전날을 최신 거래일로 주입하고 KST 정오의 합성 시각을 사용합니다. 주말/휴일도 달력 mock이 거래일로 인정하므로 실제 영업일 검증이 아닙니다. 벤치마크는 최신 거래일을2026-09-24로 고정합니다. 기본 lookback2/fetch5이며 두 클래스 모두 사용하지 않는 다른 서비스에 접근0을 검사합니다.

## 가격 경로 15개

경로: `GET /api/v1/feature3/market-data/price-series`. 기본 합성 종목은005930/KOSPI/ID7이고 Yahoo 수정종가50·51, 원시종가100·101을 구분해 가격 혼합 여부를 확인합니다.

| 메서드 | 입력·실행·독립 기대값/결과 |
|---|---|
| `adjustedSuccessUsesActualYahooCodecAndDoesNotTouchRawStorage` | 수정종가2일→YAHOO/YAHOO_ADJ_CLOSE/MISS/fallback=false·count2·50·KST 자정. 실제 client가 .KS 경로와 includeAdjustedClose=true를 생성하며 raw 저장소/제공처 접근0 |
| `poorAdjustedCoverageFallsBackAsWholeSeriesWithoutMixingOrOverwritingRaw` | 수정종가1일999/기대2일(결측50%)→전체 raw100·101로 전환, 품질/fallback/risk 경고3개가 meta에도 반영. 원본100 불변·저장0·raw 외부호출0 |
| `explicitCloseSkipsYahooAndReturnsRawDatabaseHit` | 요청 close→CLOSE 정규화·DB/HIT·fallback=false, 역순 DB 입력을 오름차순으로 반환, Yahoo/raw 외부 접근0 |
| `unknownPriceBasisUsesAdjustedPolicyObserved` | 요청 RAW_CLOSE도 ADJUSTED_CLOSE 정책으로 정규화하고 Yahoo1회. 현재 계약 관측이며 새로운 입력 enum을 정의하지 않음 |
| `rawInsufficientFetchPersistsOnlyMissingDateAndKeepsStoredOriginal` | DB 이전날100, 외부 이전날999/최신101→응답999·101/KIS, 저장은 없는 최신날1개만, 기존 객체100 유지. 응답 병합값과 원본 보존을 구분 |
| `providerHttpErrorKeepsPartialRawAndReportsMissingRate` | DB1일·raw HTTP 오류→부분 DB 반환·결측0.5·INSUFFICIENT_PRICE_HISTORY·저장0 |
| `emptyRawIsUnavailableWithPublicWarning` | 명시 CLOSE·DB/외부 모두 비어 있음→UNAVAILABLE·결측1·PRICE_SERIES_UNAVAILABLE |
| `adjustedHttpFailureAndMalformedJsonBothUseRawWithProviderWarning` | 합성 Yahoo503, 이어200/손상 JSON→각각 raw로 전환·YAHOO_ADJ_PROVIDER_UNAVAILABLE·요청2회·쓰기0 |
| `excessYahooPointsAreSortedAndTailLimited` | 뒤섞인3일100/101/102, 기대2일→마지막2일101·102만 오름차순 반환 |
| `yahooCodecDropsNullTextZeroAndNegativeAdjustedValues` | null/문자열51/0/-5/숫자50/51→실제 JSON 파서에서 유효 양수2개만 반환, raw 저장소/제공처 접근0 |
| `unsupportedUsPlainSymbolFallsBackWithoutYahooRequestObserved` | 합성 NASDAQ 종목 QA→YAHOO_SYMBOL_UNRESOLVED·Yahoo 전송0, raw 응답 시각은 뉴욕 offset. 미국 일반 티커 자동 변환 미지원이라는 현재 동작 관측 |
| `fallbackMustReportActualRawProviderInsteadOfHardCodedKis` | Yahoo 비어 있음·raw provider source=MARKETSTACK→source MARKETSTACK 기대, 실제 KIS: **FAIL** |
| `duplicateYahooDatesMustNotCountAsTwoTradingDays` | 동일 거래일의 정오/13시 수정종가2개, 기대2거래일→유효1일/결측50%로 전체 raw fallback 기대, 실제 fallback=false: **FAIL** |
| `duplicateRawDatesMustNotInflateAvailableTradingDays` | 동일 거래일 정오/13시 raw2개→availablePriceCount1 기대, 실제2: **FAIL** |
| `malformedQueryStopsBeforeServicesAndDatabaseErrorPropagates` | stockCode 누락/비숫자 lookback→400·stock service 접근0; 별도 raw DB 읽기 오류→500·Yahoo/raw 제공처 접근0 |

Yahoo 품질 임계값4/5/20/21%와 시장별 심볼 후보는 기존 `YahooPricePolicyTest` 6개에서 별도 검증합니다. 이번15개에는 실제 Yahoo JSON 파서부터 HTTP 반환까지 연결했지만 실제 인터넷 호출은 없습니다.

## 벤치마크 경로 14개

경로: `GET /api/v1/feature3/market-data/benchmark-series`. 기본 index00001/ID9, 원래 지수값2500·2600을 사용합니다. 가용 여부 필드의 실제 계약은 `benchmarkAvailable`입니다.

| 메서드 | 입력·실행·독립 기대값/결과 |
|---|---|
| `sufficientFreshDbPreservesOriginalIndexLevelsAndChronology` | 최신2일 역순 입력→DB/가용true/결측0·원래값2500·2600·KST 자정·오름차순. PageRequest2/id.ts DESC·보강0 확인 |
| `missingMasterReturnsUnavailableWithWarningAndNoPriceQueries` | UNKNOWN master 없음→UNAVAILABLE/가용false·BENCHMARK_INDEX_MISSING(data와meta)·OHLCV/제공처 접근0 |
| `insufficientDbBackfillPersistsMapsAndRereadsWithOriginalWarning` | DB 최신1일→외부2일 저장/indexID9/freq1d→재조회2회·KIS_BACKFILLED/가용true. 최초 이력부족 경고는 남음. 제공처 범위 latest-5~latest |
| `staleButSufficientDbTriggersBackfillAndThenUsesUpdatedRows` | 행수2이나 최신거래일 미도달→최신일1개 보강·마지막2일2500/2600·가용true/KIS_BACKFILLED·stale 경고 유지 |
| `emptyBackfillDistinguishesStaleInsufficientAndUnavailableResults` | 보강 빈 목록을 각각 오래된2일/최신1일/DB없음에 적용→DB_STALE/DB_INSUFFICIENT/UNAVAILABLE. 빈 원천 경고와 가용false 확인 |
| `providerAndSaveFailuresBecomeWarningsWhileDbRowsRemain` | 외부 오류, 다음 요청 저장 오류→각각 BENCHMARK_BACKFILL_FAILED·기존1행 유지/가용수1. 실제 transaction rollback 증거 아님 |
| `invalidFetchedBarsAreDroppedWithoutSavingAndWarnAsEmpty` | null bar/OHLC 누락/close0/-1→매핑 또는 양수 필터에서 제외·save0·BENCHMARK_BACKFILL_EMPTY/UNAVAILABLE |
| `sameTradingDateRowsAreDeduplicatedAndLatestTimestampWins` | 같은 거래일 두 시각2500/2600→마지막2600 하나·count1·missing0.5. 가격 경로의 중복 실패와 대조 |
| `unsupportedBenchmarkDoesNotFetchAndIgnoresFreshnessWhenSufficientObserved` | 소문자spx→SPX 정규화. 1000일 넘은2행이어도 DB/가용true·달력/제공처0;1행으로 줄이면 가용false/보강미지원 경고. 국내 외 시장 freshness 검사 생략이라는 관측 |
| `kosdaqCodeUsesSupportedFetcherAndKrxTradingCalendar` | 11001→실제 service가 KRX 달력과 KOSDAQ 지수코드/정확한 보강기간을 전달 |
| `defaultLookbackAndFetchDaysReachProviderAs252And370Window` | query 생략→조회 페이지252(전후2회)·보강 latest-370~latest |
| `nonPositiveWindowsClampToOneAndCalendarControlsExpectedCount` | lookback-1/fetch-5→기대1·보강 latest-1; 모든날 비거래일이어도 최소 expected1 |
| `initialDatabaseReadFailureAndBadQueryAreExplicitErrors` | 비숫자 lookback400·저장소/제공처0; 초기 OHLCV 읽기 오류500·보강0 |
| `futureBenchmarkRowsMustNotBeTreatedAsFreshCompletedHistory` | 달력 최신일 이후+1/+2일 두 행→완료된 과거 이력으로 가용true가 되면 안 됨. 실제 benchmarkAvailable=true: **FAIL** |

## 발견사항·한계

- `F3-PRICE-SOURCE-001`: 수정종가 fallback이면 실제 CandleSource와 무관하게 응답 source를 KIS로 고정합니다. 제공처 메타데이터 정책과 불일치합니다.
- `F3-PRICE-DATE-DUP-001`: Yahoo 품질 평가와 raw DTO에서 같은 거래일의 복수 시각을 각각 가격 하나로 셉니다. 서로 다른 두 거래일과 동일하지 않은데도 결측률을 낮추며 정규화된 응답 ts도 겹칠 수 있습니다. 두 경로의 회귀가 실패합니다.
- `F3-BENCHMARK-FUTURE-001`: 최신일보다 과거인지만 검사하고 미래일을 제외하지 않아, 미래 두 행이 최신 과거 이력으로 인정됩니다. 실제 원천/운영 DB에 이런 행이 존재한다는 주장이 아닙니다.

일반 미국 티커의 수정종가 미지원, unsupported benchmark의 freshness 생략, 보강 성공 후 초기 부족/stale 경고 유지, 응답의 보강값과 보존된 원본 차이는 별도 정책 관측입니다. [발견사항](../../../docs/findings.md)에 실패 증거와 영향 한계를 기록했습니다.

남은 검증: 실제 Yahoo/KIS/Marketstack 전송·원천 스키마와 제한, StockClient 내부 provider 선택/IndustryIndexFetcher 파서 전체, 실제 거래소 달력·휴장일·장 마감/당일 미확정 일봉, SQL 범위/정렬/페이지·저장 멱등성·동시성, 전체 Spring bean/security 설정, FastAPI/브라우저 가격어댑터까지의 종단간 계산. 따라서 두 Swagger 경로의 상태는 PARTIAL이고 실제 연동은 NOT_RUN입니다.
