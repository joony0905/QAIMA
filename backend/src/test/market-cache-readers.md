# 가격·산업지수 캐시 reader 검증

검증일: 2026-09-25. Windows Java17/Gradle. 제품 코드·설정은 수정하지 않았습니다. 실제 DB·Redis·금융 API·메일·유료 LLM 호출은0회입니다. Redis QA 전용 키 생성/정리 허용 여부는 사용자에게 질문했으며, 이 문서의 테스트는 해당 쓰기를 수행하지 않습니다.

## 범위와 실행

실제 `PriceSnapshotReader`, `IndustryIndexReaderImpl`, `IndustryIndexService`, `Feature2StockResolver`, `RedisConfig`의 값 직렬화 codec을 실행합니다. Repository·Redis GET/SET·IndustryIndexFetcher는 mock입니다. 가격 동시성 검사는 실제 Reactor/boundedElastic과 latch를 쓰고, Redis codec 검사는 실제 직렬화 바이트를 ConcurrentHashMap에 저장합니다. Redis 서버 TTL 만료·네트워크·SQL·분산 동시성을 입증하지 않습니다.

산업지수 HTTP2개는 기존 `Feature2CardFlowTest`의 fixture 생성/정리만 재사용하고, 해당 fixture의 mock IndustryIndexService를 실제 Service→Reader로 교체합니다. 실제 Controller·CardService·ExceptionHandler·DTO가 실행되며 Stock/Industry resolver와 Repository·제공처·Redis는 대체합니다. WebTestClient의 in-process HTTP 처리이고 실제 서버·JWT 필터·브라우저는 아닙니다.

Windows PowerShell, backend 기준:

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY,Env:QAIMA_LIVE_FASTAPI_TCP -ErrorAction SilentlyContinue
$env:GRADLE_USER_HOME = "$PWD\src\test\.runtime\gradle-home"
$env:TEMP = "$PWD\src\test\.runtime\tmp"
$env:TMP = $env:TEMP
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests com.qaima.verification.PriceSnapshotReaderFlowTest --tests com.qaima.verification.IndustryIndexReaderFlowTest
```

최초 가격20개는19 PASS·1 FAIL(17초), 산업18개를 합친38개는36 PASS·2 FAIL(18초)이었습니다. 이후 실제 StockResolver까지 연결한 관측1개를 추가하여 최종39개 중37 PASS·2 FAIL입니다. 최신 전체 회귀는1047개 중998 PASS·34 FAIL·15 SKIP,44초이며 [검증 요약](../../../docs/verification.md)에 기록합니다. 실패 회귀는 skip/expected failure로 처리하지 않습니다.

## PriceSnapshotReaderFlowTest — 21개

기본 fixture는 합성 종목 QA_PRICE, ONE_D, 최신110/이전100입니다. 독립 기대 등락률은(110−100)/100×100=10%입니다. 가격 계산은 BigDecimal 비교로 검증합니다. GET/SET이 모의이므로 TTL 검사는 실제 경과 시간이 아니라 SET에 전달한 Duration 검사입니다.

| 메서드 | 실행·독립 기대값 | 결과 |
|---|---|---|
| `cachedSnapshotSkipsRepositoryAndSet` | 캐시777→동일 객체, Repository/SET0 | PASS |
| `missUsesLatestTwoRowsAndWrites120SecondTtl` | 110/100→10%, page0/size2, 조회시각 전후 범위, TTL120초 | PASS |
| `oneRowHasNoPreviousPriceOrPercentage` | 110 한 행→이전가/등락률null | PASS |
| `zeroPreviousPriceDoesNotDivideByZero` | 이전가0→등락률null, 나눗셈 예외 없음 | PASS |
| `percentageUsesSixDecimalRatioRoundingBeforeTimes100` | 2/3→비율 소수6자리 반올림 후100배=-33.333300 | PASS |
| `halfUpBoundaryRoundsAwayFromZeroInBothDirections` | ±1/2000000→±0.000100%, inflight 정리 후 반대 방향 재검사 | PASS |
| `veryLargeDecimalsDoNotNarrowToDoubleOrLong` | 큰 정수부와 .01을 정확히 유지, 미세 등락은 정책 반올림0 | PASS |
| `nullLatestClosePreservesPreviousAndNoPercentage` | 최신null/이전100→이전100 보존·등락null | PASS |
| `nullPreviousClosePreservesLatestAndNoPercentage` | 최신110/이전null→최신110 보존·등락null | PASS |
| `emptyRepositoryShouldReturnEmptyWithoutWritingFalseSnapshot` | 빈 목록→정상 empty 기대, 실제 Reactor map이 반환null을 거부해 NPE | FAIL |
| `nullRepositoryResultCompletesEmptyWithoutCaching` | Repository 자체null→fromCallable empty·SET0 | PASS |
| `actualStockResolverAbsorbsEmptyPriceErrorAndKeepsMetadataObserved` | 실제 resolver+reader, 종목은 존재/가격목록empty→종목 메타 유지·price null·warning없음·외부 fallback 구독0 | PASS: 현재동작 관측 |
| `redisReadErrorFallsBackToRepository` | GET 예외→DB110 반환 | PASS |
| `redisWriteErrorDoesNotDiscardCalculatedSnapshot` | SET 예외→10% 유지 | PASS |
| `falseRedisWriteResultDoesNotDiscardCalculatedSnapshot` | SET false→10% 유지 | PASS |
| `repositoryFailurePropagatesAndNextRequestRetries` | 첫 조회error→전파, inflight 정리→다음 조회 성공·총DB2/SET1 | PASS |
| `everyFrequencyHasAnIndependentCacheKeyAndRepositoryArgument` | Freq7종 각각 별도 key/Repository 인자 | PASS |
| `realRedisValueCodecRoundtripMakesSecondCallHit` | 실제 price RedisSerializationContext codec→bytes→새 객체·값 보존·DB1/SET1 | PASS |
| `twentyFourOverlappingMissesShareOneRepositoryFetch` | Repository latch로 차단 중24개 GET 구독 확인→해제→24결과·DB1/SET1·inflight비움 | PASS |
| `separateKeysDoNotBlockEachOthersRepositoryFetch` | 두 key의 Repository가 모두 진입한 후 해제→110/120·DB2 | PASS |
| `completedInflightIsRemovedSoUncachedNextRequestReloads` | GET항상miss, 첫110→정리→다음120·DB2 | PASS |

동시성 검사는 결과를 만들기 위해 고정 sleep으로 타이밍을 추측하지 않습니다. latch 대기5/8초와 future8초 제한이 있고 finally에서 latch를 해제하며 미완료 future를 취소합니다. `awaitIdle`은 실제 inflight map을 읽기 전용 reflection으로 확인하며 최대3초/5ms 간격입니다. map을 변경하거나 구현 동작을 mock하지 않습니다. 24개는 겹치는 구독 수이지24개 OS thread나 처리량 벤치마크가 아닙니다.

## IndustryIndexReaderFlowTest — 18개

기본 fixture는 합성 산업지수91/U0999/합성산업/KRW, KST 2026-01-01~03 가격100/110/120을 역순·비정렬로 입력합니다. 독립 재기준화 값은0/10/20%입니다. 입력값이 합성이며 실제 지수 데이터의 품질·최신성을 주장하지 않습니다.

| 메서드 | 실행·독립 기대값 | 결과 |
|---|---|---|
| `databaseRowsSortAscendingRebaseAndUseThirtyMinuteTtl` | 오름차순0/10/20·label/통화/id·page3/id.ts DESC·TTL30분·fetch0 | PASS |
| `invalidIdentityFrequencyAndWindowNeverAccessStorage` | index/id/code/freq null·window0/음수→empty·저장소/제공처/Redis0 | PASS |
| `realValueCodecRetainsKoreanLabelsAndOffsetTimeOnCacheHit` | 실제 industry codec bytes왕복→한글/+09:00/값 유지·새 객체·DB1 | PASS |
| `fetchedRowsAreSortedLimitedForResponseAndAllSaved` | 비정렬4행50/100/120/150→마지막3행0/20/50, 저장요청4행·id/freq/volume0 확인 | PASS |
| `fetchDateWindowsUseKstAndFrequencySpecificLookbacks` | 일/주/월/분 window3→KST to 기준180일/104주/120개월/365일 lookback | PASS |
| `emptyProviderKeepsPartialDatabaseSeriesAndWarning` | DB2행+제공처빈목록→0/10·FETCH_FAILED·save0 | PASS |
| `providerErrorKeepsPartialDatabaseSeriesAndWarning` | 제공처error+DB1행→0·FETCH_FAILED | PASS |
| `noDatabaseOrProviderRowsReturnEmptyWithWarningAndNoCache` | 양쪽빈목록→empty·FETCH_FAILED·SET0 | PASS |
| `saveFailureStillReturnsFreshSeriesWithWarning` | fresh3행/save error→0/10/20 유지·SAVE_FAILED | PASS |
| `oneInvalidFetchedBarCurrentlyDiscardsEntireFetchedBatchObserved` | 정상1행+OHLC누락1행→fresh 전체 폐기·기존1행fallback·FETCH_FAILED·save0 | PASS: 현재동작 관측 |
| `dirtyDatabaseRowsAreFilteredBeforeRebasing` | null행/id없음/close없음 제거→0/20. 원본 행수로충분판정되어fetch0 | PASS: 현재동작 관측 포함 |
| `zeroBaseReturnsEmptyWithWarningInsteadOfInfinity` | 첫가격0→empty·OHLCV_EMPTY·SET0 | PASS |
| `redisReadAndWriteErrorsDoNotDiscardDatabaseSeries` | GET/SET 동시error→0/10/20 유지 | PASS |
| `repositoryFailureIsConvertedToMissingWarningByActualIndexService` | 실제 service→reader의DBerror→empty·INDEX_MISSING | PASS |
| `missingIndustryMapDoesNotAttemptPriceRead` | 실제 service의산업map없음→empty·INDEX_MISSING·후속IO0 | PASS |
| `cachedPartialSeriesShouldRetainItsSourceFailureWarning` | DB1행+fetch실패→warning있는부분결과; 다음codec cache hit에도동일warning 기대, 실제빈warnings | FAIL |
| `publicIndustryCardConnectsActualIndexServiceReaderAndCamelCaseDto` | 실제 GET cards/industry-index window3→indexId91·0/10/20·t·camelCase·warnings없음 | PASS |
| `publicIndustryCardPreservesPartialResultAndWarning` | 같은HTTP+DB1행+제공처실패→200·series1행·FETCH_FAILED | PASS |

## 발견사항과 남은 검증

- **PRICE-SNAPSHOT-EMPTY-001:** `PriceSnapshotReader.toSnapshot(emptyList)`가null인데 Reactor `map`에서null을 허용하지 않아 뒤의 flatMap null 방어에 도달하지 못합니다. null Repository 응답은 오히려 정상 empty입니다. 실제 StockResolver가 이 오류를 흡수하는지 별도 관측을 추가했습니다. 이 reader 단독 오류를 Public API 전체500으로 확대하지 않습니다.
- **INDUSTRY-CACHE-WARNING-001:** fallback의 `Feature2MetaDto` 경고가 `IndustryIndexBlockDto` 캐시에 포함되지 않습니다. 동일한 불완전 데이터가 다음 요청에서 경고없이 반환됩니다. 실제 값 codec을 통과했지만 Redis 전송 자체는mock입니다. reference.md의 부분 결과/제한 경고 정책에 대한 회귀입니다.
- 정상·비정상 fetched 행 혼합 시 유효한 fresh 행도 함께 버리는 동작과, DB 행수가 충분하면 filtering 뒤 부족해도 다시 fetch하지 않는 동작을 관측했습니다. 정책 전반의 수용 여부는 미확정이고 테스트 PASS를 데이터 품질 승인으로 해석하지 않습니다.
- 실제 Redis 서버의 key 격리·TTL·오류복구, 실제DB/query/트랜잭션, KIS client의응답파싱·실제제공처, 공동 inflight의서로다른meta 전달, JWT·브라우저 결합은 남았습니다. Swagger 작업은 PARTIAL입니다.
