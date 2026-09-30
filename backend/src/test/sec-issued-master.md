# SEC 발행주식수·미국 종목 마스터·scheduler 검증

검증일: 2026-09-25. Windows Java17.0.12/Gradle8.14. 추가50개 중49 PASS·1 FAIL: `SecCompanyFactsTest`13개(12 PASS·1 FAIL), `SecIssuedSharesFlowTest`17 PASS, `UsStockMasterFlowTest`16 PASS, `SecIssuedSharesSchedulerTest`4 PASS. 정수범위 초과를 잘못된 작은 양수로 변환하는 회귀는 실패 상태로 유지합니다. 제품 코드·설정·기존DB는 수정하지 않았습니다.

이 문서는 SEC 전체 완료 보고가 아닙니다. SEC 발행주식수2경로와 별도 미국 종목 마스터1경로를 다룹니다. 동일 SecAdminController의 **13F 파일/디렉터리 적재·CUSIP·분기집계·조회5경로는 이 테스트에서 실행하지 않았으며 [별도40개검증](sec-13f.md)으로추가했습니다.** 실제SQL검증은남아있습니다.

## 실행·증거

[공통 실행 준비](README.md)의 Windows PowerShell/Gradle 캐시·TEMP를 test/.runtime으로 설정한 후 backend에서 실행합니다.

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY -ErrorAction SilentlyContinue
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*SecCompanyFactsTest' --tests '*SecIssuedSharesFlowTest' --tests '*UsStockMasterFlowTest' --tests '*SecIssuedSharesSchedulerTest'
```

보정 후 필터 실행은50개/실패1, 약21초, BUILD FAILED입니다. 최초50개 실행의4실패 중3개는 기존 Answer를 가진 mock을 `when(...).then...`으로 재설정할 때 null 매처로 기존응답이 실행된 테스트 작성 문제였습니다. `doReturn`/`doThrow`로 바꿔 테스트만 수정했고 제품 결함1개가 남았습니다.

이후전체회귀는903개 중876 PASS·23 FAIL·4 SKIP, 약41초입니다. XML tests/failures/errors/skipped합계903/23/0/4를별도대조했습니다. 기존실패22개에SEC범위초과1개가추가됐으며다른기존검사의새로운실패는없었습니다. 이실행에서범위초과의실제반환fact에sharesOutstanding=1이숫자/문자열두형식모두기록됨을확인했습니다.

증거는 `.runtime/build/test-results/test/TEST-com.qaima.verification.SecCompanyFactsTest.xml`, `SecIssuedSharesFlowTest`, `UsStockMasterFlowTest`, `SecIssuedSharesSchedulerTest`의 동명 XML이며 HTML은 `.runtime/build/reports/tests/test/index.html`입니다. 필터·전체실행 모두 같은 위치를 갱신하므로 범위를 함께 기록합니다. 실패1개에는 숫자형·문자열형의 두 sub-assertion이 포함됩니다.

## 범위·안전장치·fixture

실제 `SecAdminController`/`UsStockMasterAdminController` → 실제 서비스 → 실제 SEC client/parser/WebClient codec → 합성 `ClientHttpConnector` → 실제 DTO/예외처리를 연결했습니다. 기본JSON codec 설정은 `WebClientConfig.secWebClient` 팩토리를 사용하고 전송만 대체했습니다. 합성 User-Agent와 `.invalid` URI만 사용하며 실제SEC/DB·유료호출·메일·크레딧 변경은0회입니다. 이 두SEC client에는 별도curl fallback이 없습니다.

HTTP는 `WebTestClient.bindToController`로실행합니다. 보안필터/실제TCP는포함하지않습니다. 발행주식수의13F의존성은mock이며전케이스에서접근0을검사해미검증영역과구분합니다. Repository는메모리목록/자연키맵을보존하는mock이며실제SQL정렬·collation·유일성·동시성을증명하지않습니다.

발행주식수fixture는stock QA/CIK1, 기준일2026-06-30, 공시일2026-08-01, 공시번호합성값·1000주입니다. 원천root의cik99보다요청의cik1이우선합니다. `(stockId, baseDate, COMMON)`을자연키로ID부여/재실행을확인합니다.

미국마스터는NASDAQ/NYSE exchange, QA Company/QA와QB Inc./QB입니다. `StockAlias`의실제 JPA callback `normalizeAlias`를mock save에서명시적으로호출해ticker/회사명의같은정규화키가중복생성되지않는지검사합니다. 이는실제Hibernate lifecycle wiring의증거는아닙니다. 실제 `TransactionTemplate` callback을실행하고mock관리자의REQUIRES_NEW/commit/rollback호출을확인합니다. 메모리맵자체는트랜잭션을구현하지않으므로롤백후맵상태를DB결과처럼해석하지않습니다.

## CompanyFacts parser·client 13개

아래 FAIL 이외에는 PASS입니다.

| 메서드 | 입력·방법·독립 기대값 |
|---|---|
| `parserNormalizesCikMetadataAndCommaFormattedShares` | CIK-42·문자열1,234·metadata공백→10자리0000000042·1234L·회사명/접수번호/form/fp trim·기준일/공시일·fy2026 |
| `blankCikUsesRootButExplicitCikTakesPrecedence` | 공백CIK→root99, 명시1→1, 숫자없는명시값→행제외. 요청ID우선정책관측 |
| `missingRootsAndUnitsHaveNoFacts` | null root/JSONnull/빈객체/배열/잘못된units→빈목록 |
| `onlySharesUnitsAreAcceptedCaseInsensitively` | SHARES도수집, USD unit은제외 |
| `invalidDatesNonNumbersZeroNegativeAndMissingValuesAreSkipped` | null/0/-1/bad/true·불가능한날짜·빈row/nullrow→모두제외 |
| `optionalBadDatesAndFiscalYearDoNotDiscardOtherwiseValidFact` | 유효기준일/7주에잘못된filed/fy·빈접수번호→fact유지, 해당metadata=null. 이후서비스의접수번호필수조건과구분 |
| `latestSelectsEndThenFiledThenLexicalAccession` | 4개fact: 최신end우선, 같은end는filed우선, 같으면accn문자열순. end가오래된행은filed가미래여도선택안함. 최종4주, 전체parser는4행유지 |
| `fractionalValueTruncatesAndFutureDateIsAcceptedObserved` | 123.9→123주, 2099기준일허용. 현재변환/날짜정책관측이며적합성승인아님 |
| `outOfLongRangeMustNotBecomeValidPositiveShares` | Long최댓값의숫자/문자열대조군정확히보존. 2^64+1의숫자/문자열은제외기대, 실제1주fact두개가각각생성: **FAIL** |
| `clientUsesDataHostNormalizedCikAndConfiguredHeaders` | 실제client호출→CIK0000000042.json 경로·data host·합성User-Agent·Accept JSON, fact CIK복원 |
| `client404IsNoDataWhile403429And503PropagateWithoutRetry` | 404→Optional.empty; 403/429/503→동일status의WebClientResponseException, 전송총4회로재시도없음확인 |
| `clientInvalidCikFailsBeforeHttpAndInvalidJsonPropagates` | 공백/숫자없는CIK→IllegalArgument·전송0; 잘못된JSON→오류·전송1 |
| `clientCodecAcceptsCompanyFactsLargerThan256KiB` | 회사명30만자응답→실제프로젝트codec에서복원. 기본256KiB초과확인이며32MiB정확한상한검증아님 |

## 발행주식수 HTTP·서비스 17개

| 메서드 | 입력·방법·기대값/관측 |
|---|---|
| `singleHttpParsesLatestAndPersistsCommonSharesBeforeSyncMarker` | 오래된fact와최신fact→단건POST created1·1000주·COMMON·SEC·정규화corp/회사명/접수번호/기준일/source. shares save→stock시각save 순서, 원천에없는authorized/treasury/float=null |
| `repeatedFactIsUnchangedButAlwaysRefreshesStockMarker` | 동일입력2회→첫created/둘째unchanged, 같은entity·shares save1·stock save2 |
| `revisedFactUpdatesSameNaturalKeyAndClearsOtherSourceFields` | 저장행에부가필드주입후1000→1100주수정→updated1·같은키/객체, authorized/treasury/float/감소기타null |
| `unchangedFactDoesNotClearSupplementalFieldsObserved` | 동일fact의기존자사주12/유통988은changed검사대상이아니므로unchanged·값유지·추가save0 |
| `differentBaseDateCreatesAnotherNaturalKey` | 기준일6/30→9/30→created1추가·두키존재 |
| `noFactsAnd404MarkStockSyncedWithoutWritingShares` | facts없음·404→noData=true·기준일/주식수null·stock동기화시각저장2·shares접근0 |
| `missingBlankUnknownStockAndMissingCikFailBeforeProvider` | query누락/공백/없는종목400, stock CIK없음500. 전송·shares·stock save0 |
| `lookupTrimsButLeavesCaseForRepositoryObserved` | 공백QA는QA조회, qa는그대로조회. mock은대소문자를구분하나실제DBcollation/검색정책검증은아님 |
| `missingAccessionFailsValidationBeforeAnySave` | 최신fact접수번호공백→400·두save0·동기화시각불변 |
| `latestWithoutAccessionDoesNotFallBackToOlderUsableFactObserved` | 구fact는접수번호있고최신fact는없음→400, 구fact로대체안함·save0 |
| `sharesSaveFailurePreventsStockMarker` | shares save예외→500·stock save0·동기화시각null |
| `markerFailureHappensAfterSharesSaveCallObserved` | stock marker save예외→500이나shares save는앞서1회완료. 실제부분commit여부는미검증 |
| `emptyBatchAvoidsTargetSelectionAndProvider` | mapped count0→target0·대상page조회0·provider/shares0 |
| `batchDefault300PositiveClampAndNonPositiveAllUseRequestedPageSize` | available400/빈대상fixture. 기본limit→page300,500/0/-3→page400, Long.MAX count/limit0→pageInteger.MAX_VALUE. 실제400개/수십억개처리/성능검증아님 |
| `batchHonorsLimitAndSeparatesAvailableFromTargetCount` | mapped2/limit1→available2/target1·전송1 |
| `batchContinuesAfterProviderErrorAndCounts404AsProcessedNoData` | 3종목503/404/정상→target3/processed2/failed1/noData1/created1·전송3. 실패종목marker없음, 404종목marker있음 |
| `preflightRepositoryErrorAbortsBatchAndBadLimitNeverAccessesRepository` | limit=oops→400·repo0; count예외→500·provider/shares0 |

## 미국 종목 마스터 16개

| 메서드 | 입력·방법·기대값/관측 |
|---|---|
| `httpCreatesTwoMarketsCikMetadataAndDeduplicatedNormalizedAliases` | 실제POST→created2/eligible2/alias2. 시장별entity·uppercase ticker·회사명trim·CIK10·USD/EQUITY·같은sync시각. ticker/회사명정규화중복방지, REQUIRES_NEW/commit, 원천host/path검사 |
| `repeatedSyncKeepsIdsAndAliasesButSavesUnchangedStocks` | 2회동기화→unchanged2/추가alias0·동일ID·stock save총4·alias save총2 |
| `existingListingUpdatesNameCikCurrencyAndAssetType` | 기존name/SECname/CIK/currency/asset을변경후수집→updated1/unchanged1·원천name/CIK와USD/EQUITY복구 |
| `renamedCompanyAddsAliasWithoutDeletingOldAlias` | QA Company→New Name Inc.→새정규화alias NEWNAME1개추가, 기존QA/QB보존 |
| `sameTickerOnDifferentExchangesCreatesSeparateListings` | Nasdaq/NYSE의동일QA→stock2/별도ID/alias4. 시장+티커키사용 |
| `duplicateRowsUseExistingInRunMapAndDoNotDuplicateAliases` | 완전히같은원천행2개→eligible2/created1/unchanged1·entity1/alias2 |
| `exchangeAliasesAreAcceptedAndUnsupportedMarketsSkipped` | X NAS/XNYS/OTC→NASDAQ/NYSE2개생성·skippedExchange1·total3 |
| `parserDroppedRowsAreNotIncludedInServiceSkippedInvalidCountObserved` | 정상1·빈이름/nullticker/nullrow/객체/짧은행→parser뒤total1·skippedInvalid0. 원천전체행수와service집계구분 |
| `missingCikIsStillImportedAsEligibleStockObserved` | CIK=null이나시장/티커/회사명정상→created1·SEC매핑null |
| `emptyPayloadDoesNotDeleteExistingStocksOrAliases` | 정상2개적재후빈data→total0, 기존stock/alias2개씩보존·삭제없음·빈tx도commit |
| `missingExchangeRollsBackBeforeWritingAnyStock` | NYSE참조데이터미존재→500·rollback1·commit/stock save/alias save0 |
| `aliasFailureRollsBackWholeCallbackWithoutClaimingDatabaseRollback` | 첫stock save뒤alias예외→500·tx rollback1·commit0. mock맵에stock1개남지만DB결과로주장하지않음 |
| `providerErrorsAndMalformedPayloadStopBeforeTransaction` | 403/429/503→관리자응답500, 잘못된JSON→400. 전송4·tx/저장소0 |
| `parserSupportsReorderedCaseInsensitiveFieldsAndNormalizesTickerCik` | 대소문자/공백/순서바꾼fields→정상매핑, brk.b→BRK.B·CIK-42→10자리·회사명trim |
| `parserRequiresAllFourFieldNamesAndArrays` | 필드누락/잘못된루트/잘못된JSON→IllegalArgument, null/공백문자열→빈목록 |
| `clientCodecAcceptsTickerPayloadLargerThan256KiB` | 회사명30만자→client/parser반환정확, 저장서비스미호출. DB문자열길이검증아님 |

## 스케줄러 4개

service만mock이고scheduler메서드를직접실행합니다. 실제Spring cron을켜지않습니다.

| 메서드 | 입력·방법·기대값 |
|---|---|
| `offSwitchPreventsSync` | enabled=false→service접근0 |
| `exceptionAndEmptyResultReleaseGuardAndPreserveConfiguredLimit` | limit17주입·오류/빈완료/정상3회→예외비전파·service호출3·guard복구 |
| `sameJobOverlapIsSkippedAndLaterRunSucceeds` | latch로첫작업진행중두번째호출→service추가접근0; 해제후재호출정상. bounded대기/finally해제·executor종료 |
| `cronUsesSeoulDaily1800Default` | annotation reflection→Asia/Seoul·매일18시기본식. cron동작/분산잠금검증아님 |

## 발견사항·남은 범위

`SEC-SHARES-OVERFLOW-001`: BigInteger/BigDecimal의비정확한long변환으로2^64+1이1주로바뀌어양수검증을통과합니다. 숫자/문자열두경로의실패회귀를유지하며[발견사항](../../../docs/findings.md)에근거를기록했습니다. 실제SEC가그범위의응답을보냈다거나운영데이터에오류가존재한다는주장은아닙니다.

소수주식수절삭·미래기준일허용·최신fact접수번호누락시구fact미사용·기존부가주식수필드보존·404에도동기화시각기록·원천누락CIK허용·parser제외행미집계는정책검토용관측으로분리했습니다. 승인없이제품을바꾸지않습니다.

남은검증은13F의native SQL/실제원천, 실제SEC응답·식별헤더/호출제한·네트워크,32MiBcodec상한, 실제JPA/격리DB·정렬/콜레이션/원자성/중복/동시성, 관리자JWT와업무handler결합, scheduler실행/분산경쟁, 수집주식수→공개재무/분석→브라우저연결입니다. 세경로의Swagger상태는PARTIAL, 실제연동은NOT_RUN으로유지합니다.
