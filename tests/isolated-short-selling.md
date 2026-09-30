# 공매도 적재·CSV·공개 카드·예약 실행의 실제 SQL 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03`와 기존 미커밋 소스가 대상입니다. [테스트 클래스](java/com/qaima/qa/IsolatedShortSellingJpaTest.java), [실행기](run_isolated_backend.py), 문서는 tests 내부에서만 작성했습니다. 제품은 변경하지 않습니다.

## 방법과 실제 경계

[기존 공매도40개](../backend/src/test/short-selling.md)는 실제 SQL이 없는 client/service/인자·mock 트랜잭션 검사였습니다. 이번에는 실제 Java HttpClient → loopback Netty/WebFlux → SecurityConfig/JWT → 관리자 Controller5경로 → 실제 FINRA/KRX client/codec/parser → 실제 종목 JPA 조회·TransactionTemplate(REQUIRES_NEW)·JdbcTemplate batchUpdate → 새 MySQL8.0.46을 연결합니다. 이어 실제 Feature2StockResolver/ShortSellingRepository/ShortSellingFeatureService/Feature2CardService/공개 Controller의 카드·시계열2경로까지 조회합니다.

제공자 전송만 인메모리 ClientHttpConnector입니다. 실제 WebClientConfig factory의16MiB codec·KRX User-Agent 설정을 사용하고 form BodyInserter를 실행한 다음 method/path/body를 수집해 합성 FINRA text/KRX JSON을 반환합니다. 허용 host는 `short-fixture.invalid`뿐이며 네트워크 client로 전달하지 않습니다. FINRA/KRX 외부 호출·인증·실원천 자료는 사용하지 않습니다. 공개 카드의 가격 snapshot과 미등록 종목 fallback은 빈 Mono 대역이며, 다른 카드 기능의 서비스는 mock입니다. 공매도와 종목 Repository·SQL·DTO는 실제입니다.

선택한 Spring 구성의 HTTP JSON codec에는 제품 RedisConfig.redisObjectMapper()의 순수 factory를 호출한 ObjectMapper를 연결합니다. JavaTimeModule과 날짜 문자열 직렬화 설정을 재사용하며 Redis 서버에는 접속하지 않습니다. 전체 QaimaApplication을 시작한 결과는 아닙니다.

MySQL은 실행마다 `/tmp/qaima-qa-http-*`의 새 datadir·loopback 임시 포트·무작위 QA schema를 만듭니다. 제품 YAML DB에 접속하지 않고 @@port/datadir/DATABASE/소유표식을 Python·Java 양쪽에서 확인합니다. 원본51개 migration SQL 적용 및 Hibernate validate를 거칩니다. 테스트를 rollback annotation으로 감싸지 않고 응답 이후 독립 JDBC/JPA 조회로 실제 영속성을 확인합니다. 준비 절차는 [기본 HTTP/JPA 문서](isolated-http-jpa.md)를 따릅니다.

오류 주입은 전용 DB의 `qa_short_fault` INSERT trigger를 사용합니다. INSERT...ON DUPLICATE KEY UPDATE의 앞선 갱신까지 rollback되는지, 두 번째1000행 배치 실패가 첫 commit을 보존하는지 검사합니다. 공개 조회 오류는 소유 DB의 short_selling을 qa_short_hidden으로 잠시 rename하여 실제 테이블 없음 오류를 일으키고 finally에서 되돌립니다. CSV는 run 디렉터리의 short-fixtures 안에 직접 생성해 실제 importer로 읽습니다. runner가 최종 CSV 파일 크기/SHA-256을 저장하며 basic.csv는 갱신 실험 후의 최종 내용입니다.

동시 upsert는 두 provider 응답이 준비된 뒤 barrier를 해제하여 두 실제 서비스/트랜잭션이 같은 자연키를 쓰게 합니다. scheduler 중첩 검사는 실제 provider 응답을 Sinks로 보류한 첫 호출이 진입한 latch를 기다린 뒤 두 번째 호출과 다음 실행을 확인합니다. 별도 child context의 실제 Spring @Scheduled 타이머(테스트1초 식)로 두 scheduler 콜백→SQL 저장도 실행하고 close로 정리합니다.

## 재현 명령

프로젝트 루트에서:

```bash
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite short-selling
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite short-selling --test finraAndKrxPublicRatiosMustUseTheSamePercentUnit
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

선택 명령은 후속 재현용이며 전체31개와 중복 합산하지 않습니다. 코드/문서/결과는 tests 안에 남기고 제품 결함 단언을 실패 상태로 보존합니다.

## 사례별 검증

| 메서드 | 최종 결과 | 입력·수행·단언 |
|---|---|---|
| allFiveAdminRoutesEnforceActualJwtRolesBeforeProviders | PASS | FINRA sync/backfill·KRX sync/probe/backfill 각각 익명401/USER JWT403·provider0/SQL0, ADMIN JWT200·총11전송/SQL2행 |
| finraHttpPersistsDecimalsAndPublicLatestSurvivesDetachedJpa | PASS | 실제 관리자 sync1행→SQL short1/total3/ratio0.333333·금액null→익명 공개 카드에 FINRA/코드/소수/null 보존, 트랜잭션 밖 EntityGraph stock 접근 성공 |
| finraReplayUpdatesSameIdAndPreservesCreatedTimestamp | PASS | 최초 적재 후 created_at/updated_at을 고정 과거로 설정, 수정값 재수집→동일ID/1행·새 volume·created 보존/updated 갱신 |
| finraActualMarketLookupSkipsUnknownAndAmbiguousSymbols | PASS | 같은 ticker NASDAQ/NYSE2종목·미등록코드·정상코드의 실제 JPA lookup→fetched3/upsert1/unknown1/ambiguous1 |
| krxMapsAllThreeMarketsAndTenDistinctNumbersThroughSql | PASS | 같은 코드 KOSPI/KOSDAQ/KONEX3종목, 시장별 form 응답→거래소별SQL3·숫자10필드/반올림 결과·STK/KSQ/KNX 요청 순서 |
| krxSixDecimalVolumeAndDatabaseAmountRoundingAreRecorded | PASS | DECIMAL(24,6) 경계18자리정수+6소수 보존, DECIMAL(20,0) 금액 ±7.5→±8을 실제 SQL로 관측 |
| finraAndKrxPublicRatiosMustUseTheSamePercentUnit | FAIL | 양쪽 short1/total4, KRX TRDVOL_WT25를 적재→SQL과 같은 공개 DTO의 비율 비교. 프론트의 % 표시 계약에 따라 둘 다25 기대 |
| nullableSourceNumbersPersistAndPublicSeriesKeepsNulls | PASS | KRX 숫자10개 '-'→실제 SQL null·공개 최신/시계열 null 보존 |
| publicSeriesSelectsNewestLimitThenReturnsAscendingDates | PASS | 260일 저장→SQL의 최신 limit2·0→1·9999→252, 응답 날짜 오름차순. 최신 카드 및 JPA between/as-of 날짜 조건도 확인 |
| publicEmptyLatestHasMissingWarningAndEmptySeriesHasNoRows | PASS | 종목은 있으나 공매도행0→최신 카드null/SHORT_SELLING_MISSING·시계열빈배열 |
| publicSeriesSqlReadFailureMustExposeWarningLikeLatest | FAIL | 테이블 잠시 rename→실제 SELECT 오류. 최신/시계열 모두HTTP200 fallback, 최신 경고 대조·시계열에도 LOAD_FAILED 기대. finally 원복·SQL1행 확인 |
| finraMissingFilesAndEmptyResponseKeepExistingRows | PASS | 최초적재 후403/404/빈200→upsert0·기존행ID 보존 |
| krxProbeContinuesButSyncErrorPreventsAnySqlMutation | PASS | 기존행 준비, KSQ에서 LOGOUT→probe3시장/failed1·sync2시장만 진행 후500·기존ID 유지 |
| bothProvidersFirstBatchSqlFailureRollsBackAndRetryWorks | PASS | FINRA/KRX 각각 모든 INSERT trigger 오류→예외/SQL0, 제거 재시도→SQL1 |
| laterRowFailureRestoresEarlierUpdateInSameTransaction | PASS | 기존 첫종목 upsert갱신 후 둘째종목 INSERT trigger 오류→HTTP500/첫 값·ID 복원, 제거 재실행2행 |
| finraSecondThousandRowBatchFailureKeepsFirstCommitAndRetryRecovers | PASS | 고유종목1001개·마지막행 trigger 오류→HTTP500/SQL1000 commit. 제거 후1001 재전달→HTTP200/upserted1001/최종SQL1001 |
| krxSecondThousandRowBatchFailureKeepsFirstCommitAndRetryRecovers | PASS | 같은1001행 배치/SQL/재실행 경계를 실제 KRX JSON·form·client로 검사 |
| bothBackfillsKeepEarlierDayCommitWhenNextProviderRequestFails | PASS | 두provider 각각 첫날commit 뒤 둘째날503→셋째날미호출/SQL1 유지. 오류 제거3일 재실행→SQL3 |
| databaseOverflowRollsBackAndDoesNotReplacePreviousValue | PASS | 기존행 뒤19자리 정수 volume 재수집→실제 MySQL 범위 오류/HTTP500·이전값1/SQL1 유지 |
| concurrentUpsertsKeepOneNaturalKeyAndWholeRowValues | PASS | 두 서비스 호출에10/100과20/200 응답을 barrier 후 전달→양쪽upsert1, DB1행·완전한 한 쌍·ratio0.1 |
| invalidAdminInputsStopBeforeProviderAndDatabase | PASS | 날짜누락/불가능한날짜/역순/3000일초과400→provider0/SQL0 |
| csvImportPersistsDefaultsReplaysAndUpdatesSameNaturalKey | PASS | UTF-8 CSV→SQL 소수·기본 KRX/MDCSTAT301, 반복ID 유지, 파일 값을 바꿔 재적재→동일ID/수정값 |
| csvImportMustResolveDuplicateStockCodeUsingItsMarket | FAIL | 같은 코드 두 거래소의 실제 repository 순서를 기록하고 앞 종목 시장을 CSV로 지정→저장 stock_id가 해당 시장과 일치해야 함 |
| csvQuotedNumericCellMustImportAsValidCsv | FAIL | 표준 CSV의 따옴표 숫자 "1.25"를 실제 파일로 전달→SQL1행/1.25 기대 |
| csvMissingRequiredHeaderMustRejectInsteadOfSilentSuccess | FAIL | security_type 필수열 없는 실제 CSV→정상 반환 대신 오류 기대, 저장행수 기록 |
| csvSkipsMalformedAndUnknownRowsButCommitsValidRows | PASS | 정상행·미등록종목·NaN·빈행 혼합→정상행1/값3만 commit |
| csvLaterBatchSqlFailureKeepsFirstThousandAndReplayFinishes | PASS | 한 종목1001개 날짜 CSV·마지막날 INSERT 오류→첫1000 commit/예외. 제거 후 같은 파일 재실행→SQL1001 |
| schedulerSqlFailureReleasesGuardAndOffSwitchStopsBothProviders | PASS | 실제 scheduler→SQL 실패0행→trigger 제거 후 다시실행1행, off switch 이후 전송 추가0 |
| schedulerSkipsOverlapThenAllowsActualSqlRunAgain | PASS | 두scheduler 각각 첫provider응답 보류/latch→겹친 호출 추가0→해제 후 SQL1·다음실행 허용/자연키1 유지 |
| actualCronCallbacksRunBothServicesAndPersistRows | PASS | 실제 child Spring scheduler 타이머 두 콜백 완료 latch→FINRA/KRX 실제 서비스·SQL2행·요청 날짜 기록 |
| schedulerDefaultCronsDistinguishFinraWeekendAndKrxWeekdays | PASS | 제품 @Scheduled 기본식/Seoul zone 해석, 금요일밤 다음 FINRA 토요일09:00·KRX 월요일18:30 |

## 결과와 증거

최종 `run-47cv1av9`는 **31개26 PASS/5 FAIL**, errors/skipped0, JUnit39.468초입니다. Gradle exit1은 아래 제품 동작 단언5개에 따른 결과입니다. 메서드별 최종 상태는 위 표에 함께 기록합니다. 최종 XML·summary·gradle.log는 `tests/.runtime/backend-isolated/runs/run-47cv1av9/`에 있습니다.

### 실패5개

| 식별자 | 실제 관측과 기대 | 근거·범위 |
|---|---|---|
| SHORT-RATIO-UNIT-001 | short1/total4 조건에서 FINRA SQL·공개 카드0.25, KRX SQL·공개 카드25. 같은 퍼센트 필드에25를 기대한 단언 실패 | [FINRA 계산](../backend/src/main/java/com/qaima/service/shortselling/FinraShortSellingSyncService.java)은 나눗셈만 수행합니다. [KRX 적재](../backend/src/main/java/com/qaima/service/shortselling/KrxShortSellingSyncService.java)는 합성 TRDVOL_WT25를 그대로 저장하고, [차트](../frontend/src/components/ShortSellingTrendChart.tsx)와 [패널](../frontend/src/components/Feature2ExternalFactorPanel.tsx)은 추가100배 없이 %를 붙입니다. 실제 외부 제공자 단위 계약이나 브라우저 렌더링을 재검증한 결과는 아닙니다. |
| F2-SHORT-SERIES-WARNING-001 | 실제 SELECT 오류에서 최신 카드 warnings는 SHORT_SELLING_LOAD_FAILED, 시계열 warnings는 빈 배열. 둘 다HTTP200이며 시계열에는 빈 결과만 반환 | [최신 카드](../backend/src/main/java/com/qaima/service/feature2/ShortSellingFeatureService.java)와 [시계열](../backend/src/main/java/com/qaima/service/feature2/Feature2CardService.java)의 오류 처리 차이입니다. 테이블 복원 후 기존SQL1행을 확인했습니다. |
| SHORT-CSV-MARKET-001 | 동일 코드의 실제 Repository 순서13/14에서 앞 종목KOSPI/id13을 지정했으나 SQL stock_id14/NASDAQ, market KOSPI로 저장 | [importer](../backend/src/main/java/com/qaima/importer/ShortSellingImportService.java)의 전체 시장 종목 HashMap이 code만 key로 사용하여 뒤 종목으로 덮습니다. 특정 DB 정렬을 가정하지 않고 반환 순서를 관측한 뒤 CSV 시장을 선택했습니다. |
| SHORT-CSV-QUOTE-001 | 숫자 셀에 표준 CSV 따옴표를 사용한 "1.25" 한 행이 저장되지 않고 정상 반환, SQL0 | 같은 importer의 단순 comma split과 숫자 변환·행별 catch 경계입니다. 정상 CSV의 동일 숫자는 저장되는 대조군이 통과했습니다. |
| SHORT-CSV-HEADER-001 | 필수 security_type 열이 없는 파일이 오류 없이 반환, SQL0. 파일 형식 오류를 호출자에게 전달해야 한다는 단언 실패 | 같은 importer에 필수 헤더 사전 검증이 없고 각 행의 오류가 catch되어 건너뛰어집니다. 경고 로그가 없다는 뜻은 아닙니다. |

새 식별자는 이번 Java 공매도 importer/공개 카드 경계의 증거입니다. 과거 Financial importer 및 Python CSV의 유사 실패와 같은 테스트로 합산하지 않습니다. 비율·경고·필수열의 기대 계약과 원시 관측을 구분하고 제품 수정은 하지 않았습니다.

### 통과한 저장·복구 관측

- FINRA/KRX 각각1001행 중 마지막행 SQL 오류에서 첫1000행은 commit됐습니다. 오류를 제거하고 재전달하면 upserted1001/최종SQL1001입니다. CSV도 첫1000/재실행1001입니다. 기존 OBS-SHORT-SELLING-001의 부분 commit을 실제 SQL로 확장한 관측이며 전체 원자성이 통과했다는 의미는 아닙니다.
- 첫 배치 오류와 같은 배치의 나중 행 오류에서는 전체 배치 rollback 및 앞선 UPDATE 복원을 확인했습니다. backfill의 앞 날짜 commit은 뒤 날짜 제공자 오류 이후에도 유지되고 재실행으로3일을 채웠습니다.
- 두 동시 upsert는 SQL1행과 한 요청의 완전한 volume/total 쌍을 남겼습니다. 최종 관측은10/100이며 테스트는 두 요청 중 어느 쪽이 마지막인지 강제하지 않습니다.
- 실제 MySQL DECIMAL은 `123456789012345678.123456`을 보존했고 금액 ±7.5는 ±8로 저장했습니다. 최종 summary에는 숫자를 문자열로 기록해 JSON 소비 과정의 이중 정밀도 손실을 피했습니다. 최초 실행의 BigDecimal 단언과 JUnit 원시 stdout도 정확했으나 runner summary에서 큰 수가 float로 변환됐습니다.
- 실제 예약 콜백은 제공자 요청4회·SQL2행, 두 reportDate 모두2026-09-28로 기록됐습니다. 테스트 cron은1초이며 운영 기본식 검사는 별도 메서드입니다. 기본 FINRA는 주말 포함Seoul09시, KRX는 평일Seoul18:30 다음 실행을 확인했습니다.
- 시계열은 최신 limit을 SQL로 선택한 뒤 날짜 오름차순을 반환했습니다. 0→1,9999→252 제한과 최신/as-of/between 조회도 통과했습니다.

### 실행 이력과 테스트 보정

| 실행 | 결과 | 해석 |
|---|---|---|
| run-71evatuz | 31개24 PASS/7 FAIL, errors/skipped0,32.735초 | 위 제품 실패5개 외 테스트 구성2개를 발견했습니다. |
| run-47cv1av9 | 31개26 PASS/5 FAIL, errors/skipped0,39.468초 | 테스트 구성을 보정한 최종 결과. 제품 실패5개 모두 재현됐습니다. |

첫 cron 검사는 child context가 scheduler의 @Value를 다시 주입하면서 부모의 테스트 lookback1을 운영 기본7/5로 덮어 SQL12행을 만들었습니다. child property source에 양쪽 lookback1을 명시해 두 콜백의SQL2행을 검증했습니다. 첫 시계열 검사는 선택한 HTTP codec에 제품 ObjectMapper가 연결되지 않아 reportDate 문자열 단언에 실패했습니다. 원시 첫 응답은 별도로 보존하지 않았으므로 그 날짜 JSON 형태를 단정하지 않습니다. 최종 구성에 제품 mapper를 연결하고 날짜·limit·as-of 단언을 모두 재실행했습니다. 이2개는 제품 결함 수에 넣지 않습니다.

두 실행 모두 입력568개 SHA-256의 실행 중 변경0, 업무9테이블·short_selling·QASH 종목·오류 trigger·임시 rename 테이블 잔존0입니다. 각 run의6개 CSV 파일 크기/해시는 summary.shortFixtureFiles에 있으며 basic.csv는 갱신 실험 후의 내용입니다. MySQL exit0/mysqlStopped true를 모두 확인했습니다.

tests 밖 가시 코드·설정·문서924개의 시작 기준과 비교한 변경/소실/새파일0 결과는 `tests/.runtime/backend-isolated/short-selling-source-audit.json`에 있습니다. 소스/문서/JUnit 메서드 대조는 `short-selling-method-audit.json`, 문서55개 대상 검사 로그는 `short-selling-documentation.log`에 보존합니다. 재실행 횟수는31개 최종 케이스 수와 중복 합산하지 않습니다.

## 남은 범위

이번31개는 선택한 관리자5·공개2경로와 importer/scheduler의 실제 저장 경계입니다. 외부 FINRA/KRX의 현재 자료·차단/인증·달력·provider 비율 단위 계약 자체, 전체 생산 규모/성능, 여러 JVM의 분산 실행, 장기 cron·프로세스 중단/네트워크 복구, CSV 모든 형식/인코딩·대량 오류 요약 정책, 실제 브라우저 차트/nullable 필드 렌더링과 분석 모델 결합은 별도입니다. 금융 원천의 경제적 의미와 모델 정확도까지 검증한 것은 아닙니다.
