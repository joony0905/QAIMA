# Spring 배치·달력·예약 실행의 격리 MySQL 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03` 및 미커밋 소스입니다. [테스트](java/com/qaima/qa/IsolatedBatchJpaTest.java)와 [실행기](run_isolated_backend.py)는 루트 tests 안에 있습니다. 제품 코드·기존 DB·application 설정을 수정하지 않았습니다.

## 실행 경계

실제 KisMarketDataSyncScheduler, IndustryIndexOhlcvSyncService, InvestorFlowBatchSyncService, StockInvestorFlowService, MarketInvestorFlowService, KisBatchRetrySupport, TradingCalendarServiceImpl을 실제 Spring Data JPA/Hibernate/MySQL에 연결합니다. IndustryIndexFetcher와 KisInvestorFlowClient만 합성 Mono/DTO 경계로 대체하며 외부 HTTP·KIS 토큰·Redis는 사용하지 않습니다.

기본 context는 [기존 격리 HTTP/JPA 구성](isolated-http-jpa.md)을 재사용하지만, 이번 클래스는 HTTP/JWT 요청을 보내는 검사가 아닙니다. 필요한 batch bean만 추가하고 자동 scheduling은 켜지 않습니다. 예약 실행 사례에서만 별도 AnnotationConfigApplicationContext의 @EnableScheduling/ThreadPoolTaskScheduler를 켜고 실제 제품 scheduler bean을 등록합니다. 테스트 종료 시 해당 context와 스레드를 닫습니다.

MySQL8.0.46의 새 `/tmp/qaima-qa-http-*` datadir·임의 loopback 포트·QA schema·소유 표식을 사용하고 V1~V51 SQL51개를 적용합니다. 업무47+표식1테이블 및 Hibernate validate 구성입니다. 기본 `utc` 모드는 serverTimezone=UTC, Hibernate jdbc.time_zone=UTC입니다. 후속 `mysql-profile` 모드는 제품 application-mysql.yml의 시간대 선택을 따라 JDBC serverTimezone=Asia/Seoul로 설정하고 Hibernate jdbc.time_zone은 지정하지 않습니다. 실제 application YAML을 읽어 실행하거나 운영 환경 전체를 복제하지 않습니다. 두 조건 모두 서비스 거래일·cron 기준은 Asia/Seoul입니다. JVM 기본 시간대/실제 조회 offset/SQL datetime은 관측에 별도로 남깁니다.

fixture는 정렬상 앞에 오는 합성 지수 `!QA_BATCH_A/B`, 종목009991~009995, 합성 market_holiday입니다. 원천 지수코드 규격을 검사하지 않습니다. 기존 migration의 holiday 데이터는 해당 새 DB에서 비우고 필요한 합성 휴장일만 삽입합니다. 연도별 메모리 달력 캐시는 테스트 사이에 비워 서로의 fixture가 섞이지 않게 합니다. 실제 한국 휴장일의 정확성 검사가 아닙니다.

JPA 테스트 전체를 rollback transaction으로 감싸지 않고 실제 서비스 commit/rollback 후 별도 JDBC로 값을 읽습니다. 각 테스트 전후 소유 가드를 통과한 뒤 OHLCV/수급/holiday fixture·합성 종목/지수·QA 오류 trigger를 정리합니다. 부모 실행기가 업무행/합성 메타데이터/trigger 잔여를 독립 CLI로 확인하고 자신이 시작한 MySQL만 종료합니다.

## 재현 명령

```bash
cd /mnt/c/qaima
python3 -B tests/run_isolated_backend.py --suite batch
python3 -B tests/run_isolated_backend.py --suite batch --batch-timezone mysql-profile
```

원하는 메서드만 실행할 수 있습니다. 아래는 재현용 예시이며 실제 별도 실행 결과로 집계하지 않습니다.

```bash
python3 -B tests/run_isolated_backend.py --suite batch --test actualSpringCronCallbacksReachRealServiceAndCommitSql
```

기본 core와 billing suite는 유지하며 이번 실행과 합산하거나 재실행으로 간주하지 않습니다. 로그·선택한 클래스의 새 JUnit XML·관측·소스 해시·DB 정리는 `tests/.runtime/backend-isolated/runs/run-*/`에 남습니다. 실제 application YAML은 빌드 리소스에서 제외하며 원본 제품 소스는 읽기만 합니다.

## 최종 22개 방법과 기대값

표의 비활성화2개 사례는 공개 문서에서 한국어 이름을 사용합니다. 전체 메서드명은 연결된 tests Java와 JUnit XML에 있습니다.

| 메서드 | 입력·실행·독립 기대값 | 최종 결과 |
|---|---|---|
| industryPersistsValidBarsAndReplaysWithoutDuplicatePrimaryKeys | KST 기준 오늘−1일의 UTC 자정 OHLCV 두 지수 저장→재실행. summary 성공2·saved2, SQL PK2행·종가111 유지 | PASS |
| industryInvalidBarIsSkippedWhileValidBarCommits | 필수 필드 빠진 bar와 정상123.456789를 함께 반환. 지수별 정상1행만 저장·summary2/2 | PASS |
| industryCurrentKstTradingDateMustBeSkippedAfterDatabaseRoundTrip | 당일 KST 자정 bar를 실제 저장/조회한 후 같은 batch 재실행. 당일 자료이므로 skipped1·추가 fetch0 기대. 입력/조회 OffsetDateTime·summary·fetch 구독 수 기록 | FAIL |
| industrySqlFailureRollsBackOneTargetAndContinuesNext | 첫 지수의 정상222/거절−999를 같은 saveAll에 전달, 실제 BEFORE INSERT trigger로1644. 해당 지수0행/다음 지수1행, failed1/success1·실패코드 대조 | PASS |
| marketBatchCommitsBothMarketsAndSkipsAlreadyCurrentRows | 실제 KSP/KSQ 대상 각각1행 commit. DB의 최신 날짜로 다음 실행 skipped2·fetch 총2·SQL2행 | PASS |
| marketTransientProviderErrorRetriesThenCommitsOnce | KSP의 첫 구독에 KIS_HTTP_ERROR, 다음 구독은 정상. 실제 RetrySupport max2/delay0 →success2/retried1/retryRecovered1·SQL2행 | PASS |
| marketRowFailureRetainsEarlierCommitAndContinuesOtherMarketObserved | KSP 첫 날짜111 commit 후 다음−999 SQL 실패, KSQ는 정상. summary failed1/success1/saved1이지만 SQL에는 먼저 commit된 행 포함2행. 요청 전체 원자성 보장은 아님 | PASS |
| marketClosedProviderUsesActualStoredRangeObserved | 전 거래일 자료를 실제 DB에 미리 저장하고 client에 KIS_MARKET_CLOSED 주입. 실제 service의 범위 fallback→success2/timeLimited0/saved2, 새 SQL행0 | PASS |
| marketDuplicateProviderDateUpdatesOneNaturalKeyObserved | 같은 날짜111→222 두 row 응답. 실제 반환목록2이나 SQL 고유 키1행/최종222를 확인 | PASS |
| stockBatchUsesDatabasePriorityAndExcludesForeignEtfAndDelisted | 한 종목에7일 전 수급 저장, 다른 종목은 없음. NASDAQ/ETF/상장폐지 fixture도 추가. 실제 SQL 대상 선택→누락 종목 먼저·오래된 종목 다음, 대상2/success2 기대 | FAIL |
| stockSqlFailureContinuesNextTargetWithoutBadRows | 첫 종목의 실제 INSERT를 trigger로 거절. failed1/success1·실패종목0행·다음종목 저장 기대 | FAIL |
| stockOverlappingBackfillAnchorsPersistAndReturnOneNaturalKey | 30일 backfill의 겹친 anchor 응답에 같은 날짜789. 실제 JPA upsert/중복 제거로 반환1·SQL1행/789 | PASS |
| stockRepositoryAcceptsSqlRangeButRejectsLocalDateMaxObserved | 실제 수급1행 저장 후 동일 Repository의 상한9999-12-31 조회는 성공, LocalDate.MAX는 실제 MySQL DATE 오류, 이후 정상 날짜 조회는 다시 성공. 제품 결함의 원인·오류 후 정상 조회를 확인하는 관측 대조군 | PASS |
| stockSchedulerCatchesTargetQueryFailureAndResetsGuardObserved | 원본 scheduler 종목 작업을2회 직접 호출. 대상 조회 예외를 scheduler가 흡수하고 매번 실행 flag=false 복원·원천 구독0·저장0을 확인. cron 발화 사례와 별개 | PASS |
| realHolidaySqlAndWeekendFindPreviousAndNextTradingDates | 합성2026-01-05 휴장일을 SQL에 저장. 주말+휴장일을 건너뛴 이전1/2·다음1/6을 실제 CalendarService로 검사 | PASS |
| effectiveInvestorDateHonors1540KstHolidayAndEquivalentUtcInstant | 실제 private 날짜 결정 메서드에 명시 ZonedDateTime 전달. 월요일15:39→금요일,15:40→당일, 같은 UTC instant→동일 결과, 합성 휴장일이면금요일. 시스템 시계를 변경하지 않음 | PASS |
| defaultCronExpressionsUseSeoulAndSkipWeekend | 제품 @Scheduled의 기본 cron/zone을 읽고 실제 Spring CronExpression.next 실행. 금요일23:59 KST 이후 월요일18:10/16:30/19:00 기대. cron은 거래소 휴장일 정보를 포함하지 않음 | PASS |
| 세 enabled=false 기능 비활성화 | 세 enabled=false에서 실제 scheduler 메서드 호출→원천 구독0/업무SQL0행 | PASS |
| sameSchedulerInstanceSkipsOverlapAndRunsAgainAfterCompletion | 첫 실제 산업지수 작업의 provider Mono를 Sinks로 보류. 같은 인스턴스 두번째 호출은 구독 추가 없음. 해제 후 다시 실행 가능·동일 SQL PK1행 | PASS |
| schedulerSqlFailureDoesNotPreventNextSuccessfulRun | 실제 index INSERT 전부 거절→0행. trigger 제거 후 같은 scheduler 다시 호출→정상2행. 실행 flag가 복원되는지 확인 | PASS |
| actualSpringCronCallbacksReachRealServiceAndCommitSql | 해당 테스트 context만 cron을1초 주기로 override. 실제 ScheduledAnnotationBeanPostProcessor 등록3개, 산업지수 callback2회 완료를 latch로 기다린 후 SQL2행/원천 구독≥4. market/stock은 enabled=false. 기본 일일 시간까지 기다린 검사가 아님 | PASS |
| 세 cron을 '-'로 지정해 예약 비활성화 | 테스트 context의 세 cron을 '-'로 지정→실제 등록 task0·fetch0·SQL0행 | PASS |

원천 구독은 Mono.defer/doOnSubscribe 안에서 셉니다. service 메서드 호출 수와 실제 Mono 구독을 혼동하지 않습니다. SQL 장애는 mock Repository 예외가 아니라 소유 DB trigger입니다. callback을 기다리는 상한은 테스트 대기 제한이고 운영 처리량/지연 보장이 아닙니다. 단일 scheduler 객체의 guard만 검사하며 여러 인스턴스의 분산 중복 방지를 증명하지 않습니다.

## 최초 실행과 보강

최초 `run-c45jq2eg`는 UTC/UTC 조건의20개17 PASS/3 FAIL입니다. 산업지수 당일 판단1개와 종목 배치2개가 실패했습니다. fixture/컴파일/context 오류는 없었습니다. 실행 입력563개 해시 변경0, 업무9+배치3테이블 최종0행, QA stock/index/holiday/trigger0, MySQL exit0/mysqlStopped true를 확인했습니다.

산업지수 입력 `2026-09-28T00:00+09:00`은 실제 JPA 조회에서 `2026-09-27T15:00Z`였고, 두번째 실행은 skipped0/success1/fetch 총2였습니다. 종목 수급 두 메서드는 대상 선택의 최신 날짜 SQL에서 `Incorrect DATE value: '169104628-12-09'`로 중단됐습니다. 제품 코드가 Repository에 전달한 상한은 LocalDate.MAX입니다. 따라서 두 메서드의 뒤쪽 우선순위/SQL trigger 이후 다음 종목 처리는 아직 검증하지 못했습니다.

최초 증거를 보존하고 유효 날짜 상한/스케줄러의 오류 흡수2개 대조군과 실패 직후의 원천 구독 수·저장 수·시간대 관측을 추가했습니다. 제품은 수정하지 않았고 실패 기대값을 통과로 변경하지 않았습니다. 후속 실행은 위22개 전체를 별도 새 DB와 mysql-profile 시간대 조건으로 재확인했습니다. 두 실행을 합산하지 않습니다.

## 최종 결과와 새 결함

| 실행 | 조건 | 결과 | JUnit 메서드 실행 시간 |
|---|---|---|---|
| run-c45jq2eg | JDBC UTC / Hibernate UTC, 최초20개 | 17 PASS / 3 FAIL / error·skip0 | 8.664초 |
| run-kjg9h0dg | JDBC Seoul / Hibernate override 없음, 최종22개 | **19 PASS / 3 FAIL / error·skip0** | 9.298초 |

시간은 JUnit suite에 기록된 값이며 MySQL 초기화·Gradle·Spring context 준비를 포함한 전체 소요시간이 아닙니다. 실패3개는 새 결함2건의 회귀입니다. 새 관측2개가 통과한 것은 결함 해결을 뜻하지 않습니다.

### BATCH-STOCK-DATE-001 — 최신 수급 날짜 조회가 MySQL 허용 범위를 초과

[InvestorFlowBatchSyncService](../backend/src/main/java/com/qaima/service/batch/InvestorFlowBatchSyncService.java)의 latestStockInvestorFlowDate가 LocalDate.MAX (`+999999999-12-31`)를 실제 Repository 조회 상한으로 전달합니다. 이번 JDBC/Hibernate 조합에서는 MySQL이 `169104628-12-09`라는 DATE 값을 거절했습니다. stockTargets 구성은 종목별 try/catch보다 앞에 있어 BatchSummary를 반환하기 전에 전체 실행이 중단됩니다.

- 최종 두 회귀 모두 원천 구독0입니다. 우선순위 사례는 미리 저장한1행만 남았고, 오류 trigger 사례는0행입니다. 오류 trigger에 도달하기 전에 실패했으므로 종목별 실패 복구가 검증됐다고 표시하지 않습니다.
- 같은 실제 Repository에서 상한9999-12-31은 저장된2026-09-21행을 정상 반환하고, LocalDate.MAX만 실패했습니다. 이후 정상 범위의 조회도 성공했습니다. 동일한 물리 connection 재사용 여부를 추적한 검사는 아닙니다. Repository를 mock으로 교체하거나 제품의 잘못된 인자를 테스트에서 가로채지 않았습니다.
- 실제 scheduler를 직접2회 실행하면 예외가 밖으로 전파되지는 않으나 provider0/SQL0입니다. 매번 stockInvestorFlowRunning=false로 복구되는 대조군은 PASS입니다. 이2회는 실제 cron 콜백 횟수와 별개입니다.
- 과거 [배치 mock 검사](../backend/src/test/investor-flow.md)는 Repository 조회를 대체하므로 실제 MySQL 날짜 범위 오류를 검출하지 못했습니다. 조회 상한/타입 문제이며 원천 KIS 실패나 입력 fixture 오류가 아닙니다.

### BATCH-INDEX-TIMEZONE-001 — 당일 KST bar를 DB 조회 후 전날로 판단

[IndustryIndexFetcher](../backend/src/main/java/com/qaima/external/IndustryIndexFetcher.java)는 원천 일자를 Asia/Seoul 자정의 OffsetDateTime으로 만듭니다. 같은 형식의 합성 bar를 실제 저장했으며, [IndustryIndexOhlcvSyncService](../backend/src/main/java/com/qaima/service/batch/IndustryIndexOhlcvSyncService.java)의 날짜 판단은 조회한 OffsetDateTime에 곧바로 toLocalDate를 적용합니다.

최종 조건은 JDBC Asia/Seoul, Hibernate jdbc.time_zone override 없음, 실제 JVM 기본 시간대 Asia/Seoul입니다. 그럼에도 입력2026-09-28T00:00+09:00 → SQL datetime `2026-09-27 15:00:00` → JPA 조회 `2026-09-27T15:00Z`였습니다. 두번째 실행에서 skipped1/fetch 총1을 기대했지만 skipped0/success1/fetch 총2여서 FAIL입니다. UTC override를 사용하는 최초 조건에서도 같은 날짜 판단을 재현했습니다.

확인한 영향은 이미 저장한 당일 자료의 추가 수집·upsert입니다. 원천 호출 비용/운영 발생률이나 데이터 소실을 측정한 결과는 아닙니다. 전체 application 설정·다른 Hibernate timezone storage 정책·다른 DB/배포 환경은 검증 범위 밖입니다.

### 정상 동작·관측과 집계 해석

- 실제 Spring 예약 콜백2회 완료, 원천 구독4회, 최종 SQL2행을 두 실행 모두 확인했습니다. 기본 일일 cron은 reflection/CronExpression으로 다음 발화 시각을 확인하고 실제 타이머는 테스트 전용1초 cron을 사용했습니다.
- 산업지수 saveAll의 SQL 실패는 해당 지수2행을 모두 rollback하고 다음 지수는 commit했습니다. 반면 시장 수급의 행별 save는 첫 행 commit 후 다음 행 실패 시 이미 저장한 행이 남았습니다. 후자는 관측 PASS이며 전체 요청 rollback을 보장하지 않습니다.
- 시장 batch 실패1/성공1/savedRows1에도 DB에는 먼저 commit된 행을 포함2행이 있었습니다. 원천 마감 오류의 DB fallback은 새 행0인데 성공2/savedRows2, 같은 날짜 두 응답도 반환2/SQL1행이었습니다. savedRows를 신규 삽입 건수로 해석하지 않습니다.
- 재시도 후 저장, 최신 날짜 skip, 실제 휴장일 SQL·주말·15:40 KST 경계, enabled=false, cron '-'의 등록0, 같은 scheduler 중첩 차단/후속 실행 복구는 통과했습니다.

## 증거·정리·수정 범위

- 최초 [summary](.runtime/backend-isolated/runs/run-c45jq2eg/summary.json), [JUnit XML](.runtime/backend-isolated/runs/run-c45jq2eg/TEST-com.qaima.qa.IsolatedBatchJpaTest.xml), [Gradle log](.runtime/backend-isolated/runs/run-c45jq2eg/gradle.log).
- 최종 [summary](.runtime/backend-isolated/runs/run-kjg9h0dg/summary.json), [JUnit XML](.runtime/backend-isolated/runs/run-kjg9h0dg/TEST-com.qaima.qa.IsolatedBatchJpaTest.xml), [Gradle log](.runtime/backend-isolated/runs/run-kjg9h0dg/gradle.log).
- 두 실행 모두 원본/테스트 입력563개 실행 중 해시 변경0, 업무9+배치3테이블 최종0행, QA stock/index/holiday/trigger 잔여0, MySQL exit0/mysqlStopped true입니다. 두 실행 사이에는 보강한 테스트/runner 해시가 다르며 실행별 sourceHashes로 구분합니다. 외부 datadir와 로그는 증거로 보존했고 서버는 종료했습니다.
- tests 밖 가시 코드·설정·문서924개 기준 비교의 변경/소실/새파일0: [감사 결과](.runtime/backend-isolated/batch-source-audit.json). 의존성/build/venv/agent/runtime 경로는 이 감사에서 제외합니다.
- 이번 수정은 루트 tests의 Java·runner·이 문서·README·진행 기록·문서 검사 목록뿐입니다. 기존 core12개/billing21개/Redis suite/과거 batch mock은 이번에 재실행하지 않았습니다.

문서50개 대상 로컬 링크/설정 문자열 검사2개 PASS: [최종 로그](.runtime/backend-isolated/batch-documentation.log). 최초에는 링크 검사는 통과했으나 메서드명/상태의 일반 표현3곳이 설정 문자열과 겹쳐 부분 문자열 검사가 실패했습니다. 해당 문서 표현을 한국어/설정값 표현으로 바꿨고 검사 기준은 완화하지 않았습니다. [최초 로그](.runtime/backend-isolated/batch-documentation-initial.log)는 실제 값을 출력하지 않은 상태로 보존합니다. 제품22개 결과와 문서 검사 결과를 합산하지 않습니다.

## 남은 범위

이 suite는 선택한 KIS 산업지수·수급 배치와 예약 실행 경계이며 모든 scheduler를 완료한 결과가 아닙니다. 실제 KIS 응답·rate limit/장애·원천 단위, 기본 일일 cron 장기 운용·재시작·다중 인스턴스, 다른 short-selling/OpenDART/SEC/보고서 cleanup scheduler는 별도입니다. 종목 수급의 대상 정렬·첫 종목 실패 후 다음 종목 처리도 대상 조회 결함이 먼저 발생해 미검증입니다. [전체 진행 기록](QA_PROGRESS_2026-09-28.md)을 이어갑니다.
