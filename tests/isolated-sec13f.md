# SEC 13F ZIP 적재·실제 SQL 집계 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03` 및 현재 미커밋 소스입니다. [Java 검사](java/com/qaima/qa/IsolatedSec13fJpaTest.java)와 [격리 실행기](run_isolated_backend.py)를 루트 tests 안에 작성했습니다. 제품/설정/기존 데이터는 수정하지 않았습니다.

## 이전 증거에서 확장한 경계

[기존 13F 검사](../backend/src/test/sec-13f.md)는 Repository를 mock으로 대체했고 기관별 최신 공시·정정 제외·합산의 native SQL은 실행하지 않았습니다. 이번에는 실제 Sec13fDataSetParser → Sec13fInstitutionalHoldingImportService → Spring Data JPA/Hibernate → 새 MySQL로 실행합니다. ZIP/TSV는 합성 자료를 실제 디스크에 쓰고 parser가 직접 읽습니다. 집계 projection을 테스트에서 계산하거나 주입하지 않습니다.

외부 SEC 웹사이트·다운로드·제공자 client는 사용하지 않습니다. [선택한 Spring 구성](isolated-http-jpa.md)을 재사용하므로 서버는 loopback에 뜨지만 이번에는 HTTP/JWT 요청을 보내지 않습니다. 전체 application이나 scheduler의 실행 증거도 아닙니다.

MySQL8.0.46을 새 `/tmp/qaima-qa-http-*` datadir와 임의 loopback 포트/QA schema로 시작하고 V1~V51 SQL51개를 적용합니다. 47개 업무 테이블+소유 표식1개, Hibernate validate를 사용합니다. JDBC/Hibernate 시간대는 UTC이며 분기는 명시 LocalDate입니다. Java17/Gradle8.14는 기존 오프라인 캐시를 사용합니다. application YAML을 빌드 리소스에서 제외하고 runner는 실제 DB 설정을 읽지 않습니다.

소유 port/datadir/schema/표식을 확인한 뒤 fixture를 생성하고 정리합니다. JPA 검사를 테스트 전체 rollback transaction으로 감싸지 않고, 실제 repository commit/rollback 후 별도 JdbcTemplate SELECT로 상태를 확인합니다. 원본 SQL을 복제해 기대값을 만드는 대신 작은 입력의 합계/기관수/날짜/비율을 고정 기대값과 비교합니다. 합성 NASDAQ 종목 QA13FA/B, CUSIP QA1300001~3, 분기2026-03-31/06-30/09-30을 사용합니다.

동시 CUSIP 검사 한 곳은 실제 Repository proxy에 테스트 전용 MethodInterceptor를 잠시 추가해 **실제 충돌 조회가 완료된 다음에만** 두 호출을 CyclicBarrier에서 맞춥니다. proceed()로 실제 조회를 먼저 완료하고 각 조회 결과0행과 호출2회를 확인합니다. 반환 데이터와 INSERT는 실제 DB 값이며 mock 값으로 바꾸지 않습니다. 검사 후 advice를 제거합니다. 그 밖의 parser/service/Repository는 대체하지 않습니다. 이 실행 순서가 가능한 경쟁을 확인하며 자연 트래픽 발생률을 측정하지 않습니다.

## 재현 명령

```bash
cd /mnt/c/qaima
python3 -B tests/run_isolated_backend.py --suite sec13f
```

한 메서드만 실행하려면 `--test concurrentCusipMappingsMustPreserveSingleActiveStock`처럼 지정합니다. 이 예시는 별도 실행 증거로 집계하지 않습니다. core/billing/batch 및 과거13F mock suite는 이번 실행과 합산하지 않습니다.

실행별 `tests/.runtime/backend-isolated/runs/run-*/`에 summary·새 JUnit XML·Gradle log·입력 해시가 남습니다. 합성 ZIP은 같은 run의 sec-fixtures/메서드 디렉터리 안에 보존하며 같은 경로를 재작성할 때 이전 바이트를 .before-rewrite 파일로 남깁니다. 부모 실행기가 업무9테이블 외에 identifier/import audit/filing/holding/quarter/issued_shares6테이블, QA stock/trigger 정리를 독립 mysql CLI로 확인하고 자신이 시작한 MySQL을 종료합니다.

## 24개 방법과 기대값

| 메서드 | 입력·실행·기대 또는 관측 | 최종 결과 |
|---|---|---|
| nativeLatestFilingSumsEveryCusipAndIndependentManager | 기관1 구100 대신 최신120+30/CUSIP2개, 기관2의70. 다른 종목/분기 제외→기관2/원천행7/주식220/금액440, 대표CUSIP 선택 | PASS |
| nativeEqualDatesUseAccessionTieBreakAndKeepStocksSeparate | 동일 제출일의 accession tie-01/02 중 뒤 공시20 선택, 다른 종목70은 독립 projection | PASS |
| nativeEmptyRestatementSuppressesOnlyItsManagerPeriod | 기관1의 최신 RESTATEMENT는 보유행이 없어도 이전100 제외. 기관2의70만 집계, 원본 보유행은2개 유지 | PASS |
| nativeRestatementKeepsReplacementCusipsAndExcludesOmittedOnes | 구 공시의 두CUSIP100+200 뒤 정정 공시의 한CUSIP50. 최신 집계50, 원본3행 유지 | PASS |
| nativeNonRestatingAmendmentWithoutRowsRetainsOldHoldingObserved | NEW HOLDINGS 유형의 빈 정정 공시는 이전100을 제거하지 않는 현재 SQL 정책 관측 | PASS |
| nativeOlderOrDifferentPeriodRestatementsDoNotSuppressCurrent | 더 오래된 정정·다른 분기·다른 기관의 정정은 현재100을 제외하지 않음 | PASS |
| diskZipToNativeAggregatePersistsCountsRatiosAndAudit | 실제 ZIP 두행100+25/금액200+50→보유125/금액250/원천행2, 실제 발행주식1000으로 비율0.125. 공시/보유/집계/성공이력 저장과 조회 DTO 대조 | PASS |
| sameZipReplayKeepsNaturalKeyIdsAndUpdatesChangedValues | 같은 ZIP 재실행 unchanged1/집계변경0·네 자연키 ID 유지. 동일 경로 값을150으로 바꿔 재실행→holding/집계 갱신1·ID 유지 | PASS |
| malformedRequiredHeaderMustFailStoredAudit | INFOTABLE 필수열 대신 WRONG_COLUMN만 있는 실제 ZIP→FAILED 기대. 결과 DTO뿐 아니라 저장 이력 상태 확인 | FAIL |
| impossibleFilingDateMustNotCommitAnAdjustedDate | 31-Feb-2026 제출일의 실제 ZIP→FAILED/보유 저장0 기대. 실제 저장 제출일을 함께 기록 | FAIL |
| holdingSqlFailureRollsBackSaveBatchButKeepsFilingsThenRecovers | 보유2행 중 한 INSERT를 trigger로 거절. 해당 saveAll 보유0, 먼저 commit한 공시2·실패이력 카운터0 관측. trigger 제거 후 동일 파일 재시도 | PASS |
| aggregateSqlFailureKeepsImportedRowsAndRetryBuildsAggregate | 집계 INSERT 오류→FAILED지만 공시1/보유1 유지·집계0. trigger 제거 재시도→보유unchanged1/집계생성1·같은 audit 성공/오류 해제 | PASS |
| auditStartSqlFailurePreventsFilingAndHoldingWrites | 시작이력 INSERT 오류를 실제 MySQL에서 발생시켜 import publisher 오류와 audit/공시/보유0 확인 | PASS |
| directoryContinuesAfterMalformedNumericFile | 이름순 첫 파일 숫자 오류/둘째 정상/무시할 suffix의 세 ZIP→선택2·실패1·성공1·실제 감사2/보유1 | PASS |
| actualIssuedSharesPriorityIgnoresFutureAndFallsBackToNull | 미래 COMMON 제외, 이전 COMMON800이 최신TOTAL1000/OTHER600보다 우선. 차례로 제거해 TOTAL→OTHER→null을 실제 조회/분모/비율로 확인 | PASS |
| earlierQuarterUpdateRecalculatesFollowingDeltaFromRealSql | Q1/2/3=80/100/150 저장·재집계. Q2=120으로 변경→Q2증감40/Q3증감30·률0.25, 다시 실행 변경0 | PASS |
| restatementZipWithoutHoldingsDeletesQuarterAndRebasesNext | 위3분기 저장 후 Q2의 빈 정정 ZIP 적재. native SQL 결과가 없는 Q2집계 삭제, Q3은 Q1 대비70/0.875로 재계산, 원본 보유3 유지 | PASS |
| stockLimitRetainsAllPeriodsOfFirstStockAndLaterBuildsRemainder | 첫 종목2분기/둘째1분기에서 limit1→첫 종목2집계, 이후 limit0→나머지1만 추가 | PASS |
| negativeDeltaAndFractionalRatiosKeepEightSqlDecimalPlaces | 전기3/현재1/분모6→증감−2·률−0.66666667·비율0.16666667, 실제 SQL BigDecimal scale8 대조 | PASS |
| mappingNormalizesClampsReactivatesAndRejectsSequentialConflict | 공백/소문자 정규화·신뢰150→100/−5→0·비활성 매핑 복원·ID 유지. 다른 종목의 같은 active CUSIP 순차 요청은 거절/DB1행 | PASS |
| concurrentCusipMappingsMustPreserveSingleActiveStock | 두 종목 요청의 실제 active CUSIP 조회를 모두 완료한 뒤 gate 해제→동시 INSERT 결과와 distinct active stock 수 기록. 순차 충돌 차단과 동일하게1종목만 기대. advice 제거 후 두 종목의 순차 재요청 결과와 저장 종목코드도 기록 | FAIL |
| replacementZipLeavesOmittedHoldingRowsObserved | 같은 accession의 두CUSIP100+50을 저장한 뒤 한CUSIP120만 있는 ZIP으로 재적재. parser1행/실제보유2행/집계170의 upsert 잔여 행 정책 관측 | PASS |
| sameBasenameInTwoDirectoriesReusesAuditRowObserved | one/shared.zip과two/shared.zip의 동일 basename→audit ID1개 재사용/source_path는 나중 경로, holding 값 갱신 | PASS |
| thousandRowCommitSurvivesNextBatchFailureAndReplayCompletes | 실제 ZIP1001공시/보유. 1000행 이후 INSERT를 trigger로 거절→공시1001/보유1000 commit·FAILED 통계0. 제거 후 동일 ZIP 재시도→새1/불변1000·기관1001/주식1001 | PASS |

기관별 최신/정정 정책은 현재 SQL의 동작을 검증한 것이며 SEC 규정의 법률적 해석이나 원천 공시 품질을 검증하지 않습니다. NEW HOLDINGS, 파일 교체의 남은 행, basename 이력 공유, 부분 commit은 관측으로 분리합니다. 이력을 성공으로 바꾸는 재시도 이후에도 앞선 실패 증거를 결과에 보존합니다.

## 실행 결과

최초 `run-dsmyb2en`은 테스트의 ImportDirectoryResult accessor 이름2개를 잘못 사용해 compileTestJava에서 종료했습니다. JUnit 실행0개이며 제품 실패로 집계하지 않습니다. 실제 record의 successCount/failedCount에 맞춰 테스트 호출만 수정했습니다. 해당 실행도 업무9+SEC6테이블/QA stock/trigger0, 입력564개 실행 중 변경0, MySQL exit0/stopped true였습니다.

첫 전체 `run-lq1sxlw5`는24개21 PASS/3 FAIL, JUnit16.421초입니다. 헤더/날짜2개는 기존 제품 결함 재현이고, 동시 매핑1개는 Repository interface의 spy callRealMethod가 MockitoException을 내어 SQL 조회0/저장0인 테스트 오류였습니다. 이 결과를 매핑 경쟁의 재현 근거로 쓰지 않습니다. 실제 Repository proxy의 MethodInterceptor/proceed로 바꿔 반환 시점만 맞추도록 보정했습니다.

`run-4rxzv2s_`는 실제 조회 gate로 보정한24개21 PASS/3 FAIL, error/skip0, JUnit15.893초입니다. 세 실패는 이제 기존2결함과 신규 CUSIP 경쟁1건입니다. 동시 조회2개가 모두0행을 읽고 두 요청 모두 성공하여 active stock2개가 저장됐습니다.

최종 `run-3zv3x1nj`는 **24개21 PASS/3 FAIL**, error/skip0, JUnit18.531초입니다. 헤더 실패의 독립 DB 이력을 첫 단언 전에 읽도록 보강해 저장 상태SUCCESS를 확인했습니다. 매핑 경쟁 후 advice를 제거한 순차 재요청은 A/B 모두 IllegalArgumentException이었고 실제 저장 종목코드도 A/B2개였습니다. 세 실패의 기대값을 완화하지 않았습니다.

| 실행 | 결과 | 해석 |
|---|---|---|
| run-dsmyb2en | 컴파일 실패·JUnit0 | 테스트 accessor 이름 오류, 제품 결과에서 제외 |
| run-lq1sxlw5 | 24개21 PASS/3 FAIL | 기존 제품2결함+spy 테스트 오류1개 |
| run-4rxzv2s_ | 24개21 PASS/3 FAIL | 실제 SQL에서 기존2결함+새 매핑 경쟁1건 |
| run-3zv3x1nj | 24개21 PASS/3 FAIL | DB 이력·충돌 뒤 재요청 관측 보강, 동일 제품3결함 |

JUnit 시간은 DB 초기화·Gradle·Spring 준비를 제외한 메서드 실행 구간입니다. 실행들을 합산하거나 최초 spy 오류를 제품 결함으로 세지 않습니다.

## 확인한 결함과 관측

### SEC13F-CUSIP-RACE-001 — 동시 요청이 active CUSIP 충돌 검사를 통과

[서비스](../backend/src/main/java/com/qaima/service/sec/Sec13fInstitutionalHoldingImportService.java)의 rejectActiveCusipConflict는 다른 active 종목이 있으면 거절합니다. 순차 대조군에서 A에 매핑된 CUSIP을 B에 넣는 요청은 IllegalArgumentException이며1행을 유지했습니다.

동시 사례에서는 같은 새 CUSIP의 두 실제 충돌 조회가 모두0행인 것을 확인한 뒤 gate를 해제했습니다. QA13FA/QA13FB 요청이 모두 SUCCESS이며 실제 SQL의 distinct active stock은2개였습니다. 최종에는 gate advice를 제거한 후 A/B 각각의 순차 재요청도 두 요청 모두 IllegalArgumentException이었습니다. 두 행이 이미 존재해 각 요청이 상대 종목을 충돌로 봅니다. 이는 조회와 INSERT 사이 경쟁입니다. [V38](../backend/src/main/resources/db/migration/V38__create_sec_13f_institutional_holding_tables.sql)의 고유키는 stock_id/type/value 조합이므로 다른 종목에 같은 CUSIP을 넣는 두 INSERT를 거절하지 않습니다.

기관보유 원천을 실제로 잘못된 운영 종목에 연결했다고 주장하지 않습니다. 이번에는 합성 매핑의 active 독점 조건 위반과 후속 요청을 검사합니다. 실제 HTTP/여러 backend 프로세스/자연 발생률은 별도입니다. 최초 spy 오류 실행의 SQL0 관측은 이 결함 근거에서 제외합니다.

### 기존 SEC13F-HEADER-001 / SEC13F-DATE-001의 실제 저장 영향

- 필수 CUSIP 열이 없는 INFOTABLE ZIP도 SUCCESS를 반환했고, 독립 JDBC 조회의 sec_13f_import_file.status도SUCCESS였습니다. 실원천 schema가 바뀌었다고 주장하지 않으며 합성 손상 파일의 처리 검증입니다.
- 제출일31-Feb-2026은2026-02-28로 바뀌어 실제 sec_13f_filing에 저장됐습니다. 공시1/보유1/집계1까지 생성되고 SUCCESS입니다. 파서만의 예외 누락을 넘어 저장 영향까지 확인했습니다.
- 기존 결함을 신규2건으로 세지 않고 실패 회귀로 유지합니다. 제품 소스/설정은 수정하지 않았습니다.

### 부분 commit·재실행·원본 보존 관측

- 보유 saveAll 안의 SQL 오류는 그 배치 전체를 rollback했지만 앞서 공시2행은 commit돼 남았습니다. 집계 저장 오류는 공시1/보유1을 남겼습니다. 이 두 실패 이력의 생성/원천 카운터는0입니다. 실패 audit의0을 실제 저장0으로 해석하지 않습니다.
- 실제1001행 ZIP에서 첫1000보유 commit 후 다음 배치의 SQL 오류가 발생하면 공시1001/보유1000이 남았습니다. audit createdHoldings는0입니다. trigger 제거 후 동일 ZIP 재실행은 새1/불변1000으로 완료했고 native SQL 결과도 기관1001/주식1001이었습니다. 전체 import 원자성을 보장하는 결과가 아닙니다.
- 보유 오류 사례의−999는 테스트 trigger용 숫자입니다. trigger를 제거한 동일 파일 재실행은 음수도 저장하는 현재 parser/service 정책으로 성공합니다. 금융 값의 유효성을 승인하는 검사가 아닙니다.
- RESTATEMENT는 원본 holding을 지우지 않고 native SQL에서 제외합니다. 빈 Q2 정정 ZIP은 기존 Q2집계를 삭제하고 Q3증감을 살아 있는Q1 대비70/0.875로 갱신했습니다.
- 같은 accession의 ZIP을 한CUSIP만 남기도록 바꿔도 기존 다른CUSIP의 행은 유지됩니다. parsed1/DB2/합계170을 관측했으며 파일을 완전 교체하는 정책이라고 가정하지 않습니다.
- 경로가 다른 같은 basename은 이력1개를 공유하고 source_path를 갱신합니다. 원본 파일 식별 정책 관측입니다.
- 정상 ZIP의 중복 합산·재실행 ID 유지, 최신 공시의 모든CUSIP/기관 합산, 분기·종목 분리, COMMON/TOTAL/기타/미래/분모null, SQL 소수8자리와 다음 분기 재계산은 통과했습니다. 발행주식 fixture는 실제 issued_shares에 저장했으나 SEC/OpenDART 수집 자체는 이번 범위 밖입니다.

## 증거·정리·수정 범위

- 최초 컴파일 오류 [summary](.runtime/backend-isolated/runs/run-dsmyb2en/summary.json) / [Gradle log](.runtime/backend-isolated/runs/run-dsmyb2en/gradle.log). JUnit XML은 생성되지 않았습니다.
- 첫 전체 [summary](.runtime/backend-isolated/runs/run-lq1sxlw5/summary.json) / [JUnit XML](.runtime/backend-isolated/runs/run-lq1sxlw5/TEST-com.qaima.qa.IsolatedSec13fJpaTest.xml).
- 실제 경쟁 재현 [summary](.runtime/backend-isolated/runs/run-4rxzv2s_/summary.json) / [JUnit XML](.runtime/backend-isolated/runs/run-4rxzv2s_/TEST-com.qaima.qa.IsolatedSec13fJpaTest.xml).
- 최종 [summary](.runtime/backend-isolated/runs/run-3zv3x1nj/summary.json) / [JUnit XML](.runtime/backend-isolated/runs/run-3zv3x1nj/TEST-com.qaima.qa.IsolatedSec13fJpaTest.xml) / [Gradle log](.runtime/backend-isolated/runs/run-3zv3x1nj/gradle.log) / [합성 파일](.runtime/backend-isolated/runs/run-3zv3x1nj/sec-fixtures).
- 네 실행 모두 입력564개 실행 중 변경0, 업무9+SEC6테이블 최종0행, QA stock/trigger0, MySQL exit0/mysqlStopped true입니다. 실행 사이 보정된 test/runner는 각 sourceHashes로 구분합니다. 최초와 첫 전체는 후속17개 파일 manifest 기능 추가 전이고, 마지막 두 실행은 ZIP/수정 전 사본17개의 경로·바이트 수·SHA-256도 summary에 보존합니다.
- tests 밖 가시 코드·설정·문서924개 기준 변경/소실/새파일0: [감사 결과](.runtime/backend-isolated/sec13f-source-audit.json). 의존성/build/venv/agent/runtime은 제외한 범위입니다.
- 수정은 루트 tests의 새 Java·이 문서와 기존 runner/README/진행 기록/문서 검사 목록입니다. backend/src/test 문서는 읽기만 했고 core/billing/batch/Redis/과거13F mock 검사를 재실행하지 않았습니다.

문서51개 대상 로컬 링크/설정 문자열 검사2개 PASS: [검사 로그](.runtime/backend-isolated/sec13f-documentation.log). 실행 명령은 루트에서 `.venv_wsl/bin/python -B -m unittest discover -s tests -v`입니다. 설정 문자열 검사는 값 자체를 출력하지 않으며 전체 보안 감사나 모든 비밀 유형 검사가 아닙니다.

## 남은 범위

실제 SEC 원천/파일 크기·다른 schema 변화, 관리자 HTTP/JWT 결합, 여러 프로세스의 import/rebuild 동시성, DB 재시작/연결 단절, 전체 데이터 성능, 실제 발행주식 수집→13F 보유율→공개 분석/브라우저 흐름은 남습니다. [전체 진행 기록](QA_PROGRESS_2026-09-28.md)을 이어갑니다.
