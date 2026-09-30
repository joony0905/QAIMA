# SEC·OpenDART 발행주식수와 종목 마스터의 실제 JPA 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03` 및 현재 미커밋 소스입니다. [Java 검사](java/com/qaima/qa/IsolatedSharesMasterJpaTest.java), [격리 실행기](run_isolated_backend.py)를 루트 tests에서 작성·확장했습니다. 제품 코드·설정·기존 DB는 수정하지 않았습니다.

## 이전 문서에서 확장한 경계

[기존 SEC/마스터 검사](../backend/src/test/sec-issued-master.md)와 [OpenDART 검사](../backend/src/test/opendart.md)는 Repository와 트랜잭션 관리자를 대체했습니다. 이번에는 실제 Stock/StockAlias/IssuedShares Repository·JPA callback·MySQL·PlatformTransactionManager와 다음 서비스를 연결합니다.

- UsStockMasterSyncService, SecIssuedSharesSyncService, IssuedSharesSyncService
- OpenDartCorpCodeSyncService, OpenDartDailySyncService
- SecIssuedSharesSyncScheduler, OpenDartSyncScheduler

SEC 두 client와 CompanyFacts/TickerExchange parser 및 WebClient JSON decode는 실제입니다. 테스트의 ExchangeFunction이 합성 JSON/상태 코드를 반환하므로 네트워크로 전송하지 않습니다. host가 sec-fixture.invalid이고 method/path가 기대한 GET인지 검사하며 모든 전송은 이 함수가 처리합니다. 기본 codec으로 작은 fixture를 읽는 검사이며 제품 WebClientConfig의 대용량 codec 제한을 다시 검증하지 않습니다.

OpenDartClient는 mock DTO/Mono 경계입니다. 실제 OpenDART JSON/XML/ZIP parser·HTTP·curl fallback은 이번에 실행하지 않습니다. SDK/provider 응답의 현실성·금융 데이터의 정확성 검사가 아니라 합성 입력에 대한 실제 저장/오류/재실행 검증입니다.

선택한 [Spring 격리 구성](isolated-http-jpa.md)을 재사용합니다. loopback 서버는 시작하지만 이번에는 HTTP/JWT 요청을 하지 않습니다. 기본 context에서는 자동 scheduling을 켜지 않고, cron 사례에서만 자식 context에 실제 두 scheduler를 등록해1초 주기로 실행합니다. 실제 일일 시간까지 기다리지 않으며 기본 식/zone은 별도로 검사합니다. 콜백 계측용 scheduler spy는 callRealMethod를 실행한 뒤 완료 latch만 내립니다. 각 테스트 이후 자식 context·작업 스레드를 닫습니다.

## 외부 DB와 쓰기 범위

새 MySQL8.0.46 `/tmp/qaima-qa-http-*` datadir·임의 loopback port·QA schema·소유 표식에 원본 V1~V51 SQL51개를 적용합니다. 업무47+표식1테이블, Hibernate validate, JDBC/Hibernate UTC 설정입니다. 실제 application YAML은 빌드 리소스에서 제외하고 읽어 접속하지 않습니다. Java17/Gradle8.14 기존 오프라인 캐시를 사용합니다.

fixture는 미국 종목 QAREFA/B/C, 국내009981/009985, 합성 CIK/법인코드/회사명과 기준일2026-06-30입니다. 각 테스트 전후 실제 port/datadir/schema/표식을 검증하고 issued_shares·stock_alias·QA stock/오류 trigger를 정리합니다. JPA를 테스트 전체 rollback transaction으로 감싸지 않고 서비스 commit/rollback 뒤 독립 JDBC SELECT로 검증합니다. 부모 runner도 mysql CLI로 업무9테이블+위2테이블·QA stock/trigger 잔여를 확인하고 자신이 띄운 DB를 종료합니다.

## 재현 명령

```bash
cd /mnt/c/qaima
python3 -B tests/run_isolated_backend.py --suite shares-master
```

단일 메서드는 `--test lateSecTimestampSaveMustPreserveConcurrentMasterRename`처럼 지정합니다. 이 명령 예시는 별도 실행 결과로 집계하지 않습니다. 과거 core/billing/batch/sec13f/Redis와 이전 mock 검사는 이번에 재실행한 결과가 아닙니다. 실행별 `tests/.runtime/backend-isolated/runs/run-*/`에 summary·새 JUnit XML·Gradle log·관측·입력 SHA-256을 보존합니다.

## 28개 방법과 기대값

| 메서드 | 입력·수행·독립 기대 또는 관측 | 결과 |
|---|---|---|
| masterCreatesBothMarketsAndReplaysWithRealAliasCallbacks | 실제 SEC 합성JSON→NASDAQ/NYSE2종목·alias4 생성. 재실행 unchanged2/추가alias0·ID 유지, 실제 callback의 ALPHALABS/BETALABS/QAREFA/QAREFB 키 및 USD/EQUITY/10자리CIK 대조 | PASS |
| masterRenameRepairsMetadataAndKeepsHistoricalAlias | 최초 저장 뒤 DB의 currency/asset/CIK를 바꾸고 원천 회사명도 변경. 실제 동기화가 USD/EQUITY/원천CIK/이름을 갱신하며 이전 alias 보존·새 alias 추가·ID 유지 | PASS |
| masterDuplicateRowsAndSameTickerOnDifferentMarketsKeepTwoListings | NASDAQ 동일행2개+NYSE 같은ticker1개→eligible3/created2/unchanged1·시장별SQL2/alias4 | PASS |
| masterAliasNormalizationCollisionCreatesOneAlias | ticker QAREFA와회사명 Qa-Ref A Inc.의 정규화 키가 같음→실제 JPA callback/DB의alias1, 재실행 추가0 | PASS |
| masterAliasSqlFailureRollsBackEarlierStockAndAliases | 둘째 종목 ticker alias INSERT를 trigger로 거절→첫 종목 포함 stock/alias0. trigger 제거 후 재실행2종목 생성 | PASS |
| masterLaterFailureRestoresExistingStockValuesAndAliases | 기존A를 새 이름으로 변경하고 B를 추가하는 도중 B alias SQL 실패→A 이름/동기화시각·기존alias 복원, B 없음 | PASS |
| masterOversizedCompanyNameRollsBackWholeTransaction | 정상A 뒤256자 회사명B→실제 SQL 문자열 길이 오류, 전체 stock/alias0. codec 상한 검사가 아님 | PASS |
| emptyMasterResponsePreservesExistingStocksAndAliases | 실제 빈 data JSON 재실행→total0, 이전stock1/alias2 유지 | PASS |
| secCodecParserPersistsReplaysAndUpdatesSameNaturalKey | 실제 CompanyFacts JSON→1000주 저장, 반복 unchanged1,1500주 변경 updated1·자연키ID 유지·source/동기화시각 확인 | PASS |
| secOverflowNumbersAndStringsMustNotPersistOneShare | 2^64+1을JSON 숫자/문자열로 각각 client/parser/service에 전달. 각 실제 DB 저장0 기대·저장값을 사전 기록 | FAIL |
| secNoDataMarksCheckedTimeButKeepsExistingIssuedRows | 이전 발행주식 저장 후404→noData=true·새 동기화시각 기록, 이전 issued row ID/데이터 보존 | PASS |
| secIssuedSqlFailureDoesNotMarkStockAndRetryRecovers | issued INSERT 오류→행0·stock동기화시각null, trigger 제거 후 생성1 | PASS |
| secStockTimestampSqlFailureRetainsIssuedCommitObserved | issued 저장 뒤 stock의 시각 UPDATE 오류→issued1 commit/stock시각null. 재시도 unchanged1/시각 저장 성공 | PASS |
| secBatchUsesSqlNullFirstOrderAndContinuesAfterProviderError | A최근/ B·C시각null 세종목에limit2→실제SQL B,C 순서. B503 실패 뒤 C404 noData 처리·C만 시각 저장 | PASS |
| secUnchangedFactPreservesAuxiliaryColumnsButChangedFactClearsThemObserved | 저장 후 treasury_shares=77을 넣고 동일fact 재실행→77 유지/unchanged1. 주식수 변경 시 관련 부가필드null 갱신 관측 | PASS |
| lateSecTimestampSaveMustPreserveConcurrentMasterRename | SEC 조회가 기존 stock을 읽은 뒤 합성 응답을 Sinks로 보류. 실제 master가 New Alpha Labs로 commit한 것을 JDBC로 확인하고 SEC 응답 해제. 나중 시각 저장이 새 회사명/SEC회사명을 보존하는지 검사·최종alias 함께 기록 | FAIL |
| secSchedulerSkipsOverlapAndLaterRunsActualDbServiceAgain | 실제 SEC 서비스 응답을 보류한 첫 scheduler 실행 중 두번째 호출은 추가요청0. 해제 후 다시 실행→총요청2/issued1행, 스레드 종료 | PASS |
| dartMappingPersistsCommonAndPreferredFromActualStockMetadata | 실제 국내 stock/시장/asset 메타데이터를 조회해 직접009981 COMMON·같은prefix009985 PREFERRED로 저장, 같은법인코드·직접/대체카운터1씩 | PASS |
| dartEmptyMappingClearsMetadataButPreservesIssuedHistoryObserved | 정상 매핑/발행주식 저장 후 빈 법인목록→DB의법인매핑null/OTHER, 과거issued2행 유지·다음batch대상0 | PASS |
| dartMappingSaveAllLaterSqlFailureRollsBackBothStocks | 두 매핑을 새 법인으로 바꾸다 뒤 종목 UPDATE 오류→첫 종목 포함 이전법인 유지. trigger 제거 재실행→둘 다 새 법인 | PASS |
| dartShareTypesReplayAndUpdateSameSqlNaturalKeys | 보통주/우선주/합계3DTO→SQL3, 반복 unchanged3, 보통주만1001로 변경 updated1·ID 유지. 실제 latest/type 조회의trim·기준일/값 대조 | PASS |
| dartSecondRowSqlFailureRetainsFirstCommitAndContinuesNextStockObserved | 첫종목 보통주 commit 뒤 합계 INSERT 실패, 다음종목2행성공→failed1/processed1/created2지만실제SQL3행 | PASS |
| dartDailyKeepsMappingCommitWhenAllShareProvidersFailObserved | 매핑2개 실제 commit 뒤 두 발행주식provider오류→매핑 유지·nested failed2·issued0. 실제 HTTP 상태를 검사한 것은 아님 | PASS |
| dartDailyMappingSqlFailurePreventsShareProviderCalls | 매핑 UPDATE 실제SQL 오류→일일publisher 오류, 발행주식provider0·SQL0·법인매핑null | PASS |
| dartReturnedCorporationIdentityIsStoredWithoutChangingStockMappingObserved | 원천 row의 다른법인코드QA000099가 issued에 저장되지만 stock의요청매핑QA000001은 유지되는 현재 정책 관측 | PASS |
| actualSpringCronCallbacksPersistSecAndDartIssuedShares | 실제 두 @Scheduled callback 각각완료를 latch로 확인. 테스트1초cron/등록2개→SEC1+OpenDART2 issued행. 자식 context 닫은 후SQL3행/원천구독 확인 | PASS |
| defaultDailyCronsUseSeoulAndIncludeWeekend | 실제 @Scheduled 기본 식을 Spring CronExpression.next로 계산. 금요일23:59KST 뒤 토요일SEC18:00/OpenDART03:10, 두zoneAsia/Seoul | PASS |
| offSwitchesPreventProviderCallsAndWrites | 두 enabled=false 상태에서 scheduler 직접호출→SEC전송/법인/발행주식provider0·issuedSQL0 | PASS |

순서 제어는 합성 전송 publisher와 latch/Sinks에서만 수행합니다. Repository 응답·SQL 결과·트랜잭션 관리자를 mock하지 않습니다. 실제 SQL 장애는 소유 DB의 BEFORE INSERT/UPDATE trigger입니다. 단일 scheduler 객체의 guard와 테스트cron만 검사하며 분산 중복 방지를 보장하지 않습니다.

## 실행 결과

`run-z49hgm1r`: **28개26 PASS/2 FAIL**, errors/skipped0. 첫 전체 실행이며 컴파일/context/fixture 보정 오류는 없었습니다. JUnit에 기록된 메서드 실행 구간은7.782초이고, DB 초기화·Gradle·Spring 준비를 포함한 전체 시간은 아닙니다. Gradle exit1은 아래 두 실패를 보존한 결과입니다.

### SEC-SHARES-STALE-STOCK-001 — 늦은 발행주식 동기화가 새 종목명을 덮어씀

실제 실행 순서:

1. 미국 마스터 동기화로 Old Alpha Labs를 저장합니다.
2. SEC 발행주식 동기화를 시작합니다. 실제 StockRepository 조회 뒤 CompanyFacts 응답의 Mono를 보류합니다.
3. 같은 종목에 실제 master 동기화를 실행해 New Alpha Labs를 commit하고 독립 JDBC에서 새 이름을 확인합니다.
4. SEC 합성 JSON1000주 응답을 해제합니다. 발행주식 저장과 stock의 동기화시각 저장이 완료됩니다.
5. JDBC로 읽은 company_name과 sec_company_name이 모두 Old Alpha Labs로 되돌아갑니다. alias에는 NEWALPHALABS/OLDALPHALABS/QAREFA가 함께 남습니다.

[SecIssuedSharesSyncService](../backend/src/main/java/com/qaima/service/issuedshares/SecIssuedSharesSyncService.java)는 원천 조회 전에 읽은 Stock 객체를 나중에 markStockIssuedSharesSynced에서 stockRepository.save로 저장합니다. 실제 JPA merge가 앞서 마스터에서 변경한 두 이름을 덮는 것을 확인했습니다. [UsStockMasterSyncService](../backend/src/main/java/com/qaima/service/stock/UsStockMasterSyncService.java)의 새 값 commit은 SEC 해제 전에 확인했으므로 단순히 먼저 실행된 요청을 새 요청이라고 가정한 결과가 아닙니다.

두 이름을 보존해야 한다는 단언2개가 같은 테스트에서 FAIL입니다. 이번 관측은 같은 JVM의 실제 두 서비스·별도 작업 순서이며 다른 모든 필드/운영 발생률/여러 인스턴스/공개 API 영향은 검사하지 않았습니다. 제품은 수정하지 않았습니다.

### 기존 SEC-SHARES-OVERFLOW-001 — parser 오류가 실제 issued_shares에 전파

합성 CompanyFacts의 val에18446744073709551617을 JSON 숫자와 문자열로 각각 전달했습니다. 실제 WebClient decode→parser→service→JPA가 두 입력 모두 issued_shares_total=1을 저장했습니다. 각 입력 전에 issued_shares를 비워 독립 검사했으며 저장 건수[1,1]/저장값[[1],[1]]을 관측했습니다.

기대는 범위 초과 값이 실제 저장되지 않는 것이며 두 단언은 FAIL입니다. [기존 parser 결함](../backend/src/test/sec-issued-master.md)의 실제 저장 영향으로 기록하고 신규 결함으로 다시 세지 않습니다. 실제 SEC가 이 입력을 제공했다는 의미는 아닙니다.

### SQL 원자성·부분 commit·예약 실행의 통과와 관측

- 미국 마스터의 REQUIRES_NEW 트랜잭션에서 뒤 alias INSERT가 실패하면 앞서 만든 stock/alias도0행이었습니다. 기존 종목 갱신 뒤 다른 alias가 실패한 경우에는 이름/동기화시각/기존alias가 복원됐습니다. 256자 회사명은 실제 MySQL1406/SQLState22001/company_name 길이 오류로 전체 rollback됐습니다.
- SEC issued INSERT 오류는 stock 동기화시각 갱신에 도달하지 않습니다. 반대로 issued commit 뒤 stock UPDATE만 실패하면 issued1행/stock시각null이 남고 다음 실행은 unchanged1로 시각을 저장했습니다. 전체 요청을 단일 transaction으로 처리한다는 증거가 아닙니다.
- OpenDART 매핑 saveAll의 뒤 종목 UPDATE 실패는 두 매핑 모두 rollback했습니다. 행별 issued save에서는 첫 종목 보통주 commit 뒤 합계 실패·다음 종목2행 성공으로 summary created2/실제SQL3이었습니다.
- 일일 OpenDART의 매핑은 발행주식 provider 오류 전에 이미 commit돼 남았습니다. 빈 법인목록에 의한 실제 매핑 해제는 과거 발행주식 행을 지우지 않았습니다. 원천 row의 다른 법인코드 우선 저장·동일 fact의 부가필드 유지도 정책 관측입니다.
- 실제 SEC/OpenDART 예약 callback 각1회가 완료됐고 SEC 요청1/OpenDART 발행주식 요청2/SQL3행을 확인했습니다. 기본 일일 cron/Seoul/주말은 별도 CronExpression 검사이고 실제 타이머는 테스트1초 cron입니다. SEC 중첩 guard와 후속 실행도 실제 DB 서비스로 통과했습니다.

## 증거·정리·수정 범위

- 실행 [summary](.runtime/backend-isolated/runs/run-z49hgm1r/summary.json), [JUnit XML](.runtime/backend-isolated/runs/run-z49hgm1r/TEST-com.qaima.qa.IsolatedSharesMasterJpaTest.xml), [Gradle log](.runtime/backend-isolated/runs/run-z49hgm1r/gradle.log).
- 원본/테스트 입력565개 실행 중 SHA 변경0, 업무9+reference2테이블 최종0행, QA stock/trigger0, MySQL exit0/mysqlStopped true입니다. 외부 datadir/log는 보존했고 서버는 종료했습니다.
- tests 밖 가시 코드·설정·문서924개 기준 변경/소실/새파일0: [감사 결과](.runtime/backend-isolated/shares-master-source-audit.json). 의존성/build/venv/agent/runtime은 이 비교에서 제외합니다.
- 새 Java/이 문서 및 기존 runner/README/진행 기록/문서 검사 목록만 수정했습니다. 이전 suite의 검사를 다시 실행하거나 합산하지 않았습니다.

문서52개 대상 로컬 링크/설정 문자열 검사2개 PASS: [검사 로그](.runtime/backend-isolated/shares-master-documentation.log). 루트에서 `.venv_wsl/bin/python -B -m unittest discover -s tests -v`로 실행했습니다. 설정값 자체를 출력하지 않으며 전체 보안 감사나 모든 비밀 유형 검사는 아닙니다.

## 남은 범위

실제 SEC/OpenDART 응답·인증·네트워크/제한·OpenDART curl, 관리자 HTTP/JWT 결합, 운영 규모 마스터·장기/여러프로세스 scheduler, 연결 단절/재시작, 수집한 발행주식수→공개재무/분석/브라우저의 전체 흐름은 남습니다. [전체 진행 기록](QA_PROGRESS_2026-09-28.md)을 이어갑니다.
