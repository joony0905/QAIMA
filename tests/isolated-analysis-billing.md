# 분석 HTTP + 격리 MySQL/Redis 과금·환불·리포트 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03` 및 현재 미커밋 소스를 대상으로 합니다. [테스트 클래스](java/com/qaima/qa/IsolatedAnalysisBillingTest.java), [실행기](run_isolated_backend.py), [Redis 동반 실행기](isolated_redis_fixture.py)는 루트 tests 안에 있습니다. 제품 소스·운영 설정·기존 데이터는 수정하지 않았습니다.

## 실제 실행 경계

Java HttpClient → 실제 loopback TCP/Netty → WebFlux validation·SecurityConfig/JwtAuthFilter → Feature1/2/3 Controller → 실제 CreditService/AnalysisReportService → JPA/Hibernate/JDBC → 새 외부 MySQL을 연결했습니다. Feature3 비용 계산·metrics 캐시·overlay 결합은 실제 Feature3OverlayService와 별도 새 Redis/Lettuce를 사용합니다. HTTP 응답 후 독립 JDBC 쿼리로 commit된 잔액·원장·리포트를 대조합니다. 테스트 전체를 rollback transaction으로 감싸지 않습니다.

[기존 HTTP/JPA 구성](isolated-http-jpa.md)의 명시적 Spring context를 확장했습니다. 전체 QaimaApplication이나 scheduler를 기동한 결과는 아닙니다. 사용자 A/B/관리자와 종목 `009999`는 합성 fixture이고, 테스트 키로 제품 JwtTokenProvider가 서명한 JWT를 실제 HTTP Authorization 헤더에 넣습니다. 이번 클래스에서는 로그인 절차를 다시 검사하지 않습니다.

| 경계 | 실제 실행 / 대체 내용 |
|---|---|
| 과금·환불 | 실제 CreditService, TransactionTemplate, 사용자 row lock, UserRepository/UserCreditLedgerRepository, MySQL commit/rollback |
| 리포트 | 실제 AnalysisReportService/Repository, stock 회사명 조회, JSON snapshot SQL 저장, 소유자 상세 HTTP200/타인404 |
| Feature3 캐시·비용 | 실제 Feature3OverlayService, fresh/stale metrics Redis 키, cold/reuse/force/개별 override 비용. 한 오류 사례만 loadOverlaySignals 경계에 Mono.error 주입 |
| Feature1/2 분석 | FeatOneService/Feature2AnalyzeService mock. 합성 metrics와 `SYNTHETIC_PARTIAL` warning 또는 오류 반환. 정상 분석은 Mono.defer 구독 시점에 별도 SQL 잔액을 읽음 |
| Feature3 입력·계산 | 가격/benchmark/무위험금리 service 및 AnalysisApiClient mock. 정상 DTO는 합성 최소 데이터. FastAPI 서버·실제 시계열·모델·수학 계산은 실행하지 않음 |
| 나머지 overlay | 이번에는 fundamentals/technical만 선택. 뉴스/산업/peer/resolver 경계는 mock이며 실제 뉴스 결합은 이전 Redis 검사와 구분 |
| 외부 인증·메일 | 기존 context의 OAuth handler/MailAuthService mock. 실제 외부 인증이나 메일 전송 없음 |

기본 잔액은5입니다. Feature1/2 비용1, Feature3 core 비용1, 선택한 fundamentals/technical 두 개가 cold이면 총3을 독립 기대값으로 사용합니다. 캐시 정책 사례만 시작 잔액10입니다. 차감 전에 분석 Mono가 구독되지 않는지 확인하며, 단순 Mockito 메서드 호출 수와 구분합니다. Controller가 Mono를 조립하면서 service 메서드를 미리 호출할 수 있기 때문입니다.

## 격리·준비·명령

[MySQL 준비 명령](isolated-http-jpa.md)과 [Redis 준비 명령](isolated-redis-cache.md)의 패키지 추출본을 재사용합니다. 버전은 MySQL8.0.46, Redis7.0.15, Java17.0.20.1, Gradle8.14, Spring Boot3.2.5입니다. 패키지는 시스템에 설치하지 않았고, DB 상태는 매 실행 새 `/tmp/qaima-qa-http-*` 및 `/tmp/qaima-qa-redis-*`에 만듭니다.

MySQL은 loopback 임의 포트·새 datadir·새 `qaima_qa_*` schema에 V1~V51 SQL51개를 적용합니다. 업무47개+소유표식1개 테이블, Hibernate validate를 확인합니다. SQL 적용은 Flyway history 검사와 구분합니다. `@@port`, `@@datadir`, `DATABASE()`, 소유 표식을 확인한 뒤 fixture 변경·정리를 합니다.

Redis는 loopback 임의 포트·임의 인증·persistence off로 시작합니다. 실제 INFO port/run_id, CONFIG dir, `qa:owner`를 확인합니다. 기존 Redis 주소를 사용하지 않습니다. 실행기가 QA/SPRING/MYSQL/REDIS 접속 환경을 제거하고 자신이 만든 서버 환경만 Gradle에 전달합니다. application 설정 파일은 [Gradle 설정](backend-isolated.init.gradle)의 processResources에서 제외합니다.

```bash
cd /mnt/c/qaima
python3 -B tests/run_isolated_backend.py --suite billing
```

한 메서드만 재현할 때도 새 MySQL·Redis를 만들고 종료합니다. 아래 명령은 재현 방법이며 별도 실행 결과로 집계하지 않습니다.

```bash
python3 -B tests/run_isolated_backend.py --suite billing --test feature3InputFailureMustRefund
python3 -B tests/run_isolated_backend.py --suite billing --test feature1ValidLongCompanyNameMustNotPreventReportPersistence
```

`--suite` 생략 시 기존 core 클래스입니다. 이번 billing 실행은 core12개나 Redis18/21개 재실행이 아닙니다. 실행별 Gradle 명령·로그·선택한 클래스의 새 JUnit XML·source SHA-256·SQL 정리·Redis 정리·종료 상태는 `tests/.runtime/backend-isolated/runs/run-*/`에 남깁니다. 알려진 제품 실패가 있으면 Gradle exit1을 그대로 반환하고 skip/expected failure로 바꾸지 않습니다.

## 메서드별 방법과 기대값

아래19개 메서드 중 입력 실패가3개 parameter invocation이므로 최종 구성은21개입니다. 테스트 수는 HTTP 요청 수와 다릅니다.

| 메서드 | 실제 입력·검사 |
|---|---|
| anonymousRequestsCannotChargeOrAnalyzeAnyFeature | 세 분석 POST 무인증401, 잔액5·원장0·리포트0·분석 구독0 |
| invalidBodiesAreRejectedBeforeCharge | Feature1 잘못된 freq enum, Feature2 빈 stockCode, Feature3 빈 holdings → 각각400, 차감/구독 없음 |
| insufficientBalanceDoesNotSubscribeOrCreateFalseRefund | SQL 잔액0에서 세 POST402/INSUFFICIENT_CREDIT, 분석 구독·원장·리포트0. 잘못된 환불도 없음 |
| failedUseLedgerInsertRollsBackBalanceAndPreventsAnalysis | 실제 MySQL BEFORE INSERT trigger로 음수 원장 거절 → 세 POST500, 잔액5·원장/리포트/분석 구독0 |
| feature1SuccessCommitsDebitCacheAndOwnedReportSnapshot | 분석200 → 잔액4·USE−1·구독 시 SQL잔액4, 리포트 JSON=HTTP data, 소유자200/타인404, 실제 metrics fresh 키와24h PTTL |
| feature1ReportMustKeepStockCodeAndCompanyNameInTheirOwnColumns | Feature1 분석 후 실제 report SQL과 상세 HTTP의 stockCode=009999/companyName=합성 QA 분석기업 기대. 기존 REPORT-MAPPING-001 회귀 |
| feature1ValidLongCompanyNameMustNotPreventReportPersistence | stock에 합법적인62자 회사명 저장. 같은 종목의 Feature2는 올바른 컬럼에 리포트 저장. Feature1도 리포트/ID가 있고 REPORT_SAVE_FAILED가 없어야 함. 실제 차감2건과 저장행 독립 SQL 대조 |
| feature1AnalysisErrorRefundsSameReferenceAndReturnsErrorEnvelope | 분석 Mono.error → HTTP200/FEATURE1_ANALYZE_FAILED, 원장 USE−1/REFUND+1·동일 referenceId·잔액5·리포트0 |
| feature2SuccessCommitsDebitAndCorrectOwnedSnapshot | 분석200 → 잔액4·USE−1·구독 시 SQL잔액4, snapshot/종목코드/회사명/소유권 대조 |
| feature2AnalysisErrorRefundsAndReturnsDocumentedFallbackWithoutReport | 분석 Mono.error → HTTP200 success fallback·meta.warnings의 FEAT2_INTERNAL_ERROR, 동일 referenceId 환불1·잔액5·리포트0 |
| reportSqlFailurePreservesAllThreeSuccessfulAnalysesAndCharges | report INSERT를 실제 SQL trigger로 거절 → 세 분석200 success/data 보존·REPORT_SAVE_FAILED·reportId 없음, 잔액5→4→3→2·USE3개·REFUND0·리포트0 |
| feature3ColdOverlayCostChargesThreeAndSavesPortfolioReport | cold Redis에 fundamentals/technical 선택 → 실제 예상 비용3·잔액2·USE−3, 분석 구독 시 SQL잔액2, PORTFOLIO 리포트/소유권/생성된 metrics 캐시 |
| feature3CacheReuseForceAndOverrideChargeOneThreeAndTwo | 실제 metrics 캐시 사전 저장. 두 overlay reuse 비용1, force 비용3, force+technical reuse override 비용2. 잔액10→9→6→4·원장[−1,−3,−2]·리포트3. 최초 reuse에는 Feature1 재분석 구독0 |
| feature3FastApiFailureRefundsExactStoredCostAndReference | cold 비용3 차감 후 AnalysisApiClient Mono.error → HTTP500, 동일 referenceId의−3/+3·잔액5·리포트0 |
| feature3InputFailureMustRefund[PRICE/BENCHMARK/RISK_FREE] | 입력 service별 Mono.error → HTTP500/FastAPI 구독0/리포트0. 실제 차감3 이후 동일 금액·referenceId 환불 기대. 기존 F3-CREDIT-001을3경계로 검사 |
| feature3OverlayBoundaryFailureMustRefundActualEstimatedCost | 실제 예상 비용3은 유지하고 loadOverlaySignals 경계만 오류 → HTTP500/FastAPI 구독0/리포트0. 비용3 환불 기대 |
| refundLedgerSqlFailureDoesNotCommitAnUnrecordedBalanceIncrease | Feature2 실패 환불의 양수 원장 INSERT를 실제 trigger로 거절 → HTTP500, 잔액4·USE−1만 남음. 환불 transaction의 잔액 증가도 함께 rollback했는지 확인 |
| sixConcurrentFeature2RequestsCannotSpendMoreThanFiveCredits | latch로6개 실제 TCP 분석 출발 →5개200/1개402·분석 구독5·잔액0·USE−1 5행·서로 다른 referenceId5개·리포트5 |
| pendingAnalysisReservesCreditThenRefundAllowsRetryWithoutDuplicateRefund | 첫 Feature3 API Mono를 Sinks로 보류. 차감3/잔액2 확인 → 동시에 온 비용3 요청402 → 첫 API 오류 해제500/환불3/잔액5 → 새 요청200/잔액2. 원장[−3,+3,−3]·referenceId2개·리포트1 |

SQL 장애는 예외 mock 대신 소유 schema에 `SIGNAL SQLSTATE '45000'` trigger를 생성합니다. 각 메서드 전후 해당 QA trigger를 제거합니다. 정상 SQL과 실패 SQL을 같은 실제 CreditService/리포트 경로에서 비교합니다. 환불 INSERT 실패 검사의 PASS는 원자성 확인이며, 환불 완료나 자동 복구 보장을 뜻하지 않습니다.

referenceId 단언은 이번 요청의 USE/REFUND가 같은 ID인지 확인합니다. 임의 중복 요청의 멱등성, 재시도 키 처리, 모든 장애에서 정확히 한 번 환불되는 일반 보장은 검사하지 않았습니다. 입력/overlay service 내부 fallback을 우회해 경계 오류를 주입한 사례이므로 실제 원천 장애가 반드시 같은 HTTP500을 만든다고 확대하지 않습니다.

## 실행 결과

최초 `run-jb3hm7hn`: **20개14 PASS/6 FAIL**, skipped/error0, Gradle exit1. 실패 중1개는 테스트가 Feature2 warning을 `data.warnings`에서 읽은 오류입니다. DTO의 `@JsonIgnore`와 Controller의 envelope 승격 계약에 따라 `meta.warnings`로 수정했습니다. 제품 결함으로 집계하지 않습니다. 당시 이 테스트는 warning 단언에서 중단됐으므로 그 실행만으로 이후 환불 단언을 통과했다고 주장하지 않습니다.

나머지5개는 리포트 매핑1개와 Feature3 환불4개입니다. 첫 XML의 임시 리포트 식별자 `REPORT-F1-SUBJECT-001`은 기존 문서와 대조 후 `REPORT-MAPPING-001`로 통일했습니다. 새 결함 ID가 아닙니다. 긴 회사명 사례는 첫20개에 포함하지 않았습니다.

최초 실행은 업무9테이블0행, sourceChangedDuringRun0, MySQL 정상종료0/mysqlStopped true, Redis 소유 표식 외 키0→표식 제거 후 DBSIZE0, 정상종료0/redisStopped true입니다.

보정 및 긴 회사명 사례 추가 후 `run-ekmi9ac6`: **21개15 PASS/6 FAIL**, skipped/error0, Gradle exit1입니다. 위 표에서 리포트 매핑2개·입력 실패3개·overlay 경계 실패1개가 FAIL이며 나머지는 PASS입니다. 실패6개는 기존 결함2건의 회귀이며 신규 결함6건을 뜻하지 않습니다. 첫20개와 중복 합산하지 않습니다. 최초 Feature2 테스트 오류는 보정 후 warning/환불/리포트0 단언까지 통과했습니다.

| 실행 | JUnit | 입력 해시 | 최종 업무9테이블 | Redis | 프로세스 종료 |
|---|---|---|---|---|---|
| run-jb3hm7hn | 20개14 PASS/6 FAIL | 562개·실행 중 변경0 | 전부0행 | DBSIZE0 | MySQL/Redis exit0, stopped true |
| run-ekmi9ac6 | 21개15 PASS/6 FAIL | 562개·실행 중 변경0 | 전부0행 | DBSIZE0 | MySQL/Redis exit0, stopped true |

업무9테이블은 users/login_session/auth_login_log/watchlist/watchlist_item/portfolio/portfolio_holding/analysis_report/user_credit_ledger입니다. 각 테스트의 QA 종목 삭제·0행 단언과 QA trigger 제거도 실행됐습니다. 소유 marker 확인 후 자신이 만든 프로세스만 종료했습니다. 외부 임시 디렉터리는 증거로 보존하지만 DB 프로세스는 남기지 않았습니다.

최종 관측에서 Feature1 fresh metrics PTTL은86,399,926ms(설정24h)였습니다. 24시간 자연 만료를 기다린 검사는 아닙니다. 동시 분석은6개 중5개200/1개402, 잔액0·리포트5를 두 실행에서 확인했습니다. refund INSERT 거절은 두 실행 모두 HTTP500/잔액4/원장[−1]로 잔액과 원장의 transaction rollback이 일치했습니다.

수정 범위 별도 감사는 `tests/.runtime/backend-isolated/billing-source-audit.json`입니다. 2026-09-28 시작 기준의 tests 밖 가시 코드/설정/문서924개는 변경·소실0, 새 파일0이었습니다. 의존성/빌드/venv/agent/runtime을 제외한 감사이며 파일 시스템 전체 비교는 아닙니다. 문서47개 대상 로컬 링크/설정 일부 비밀값 검사2개도 PASS입니다. 전체 보안 감사는 아닙니다. 결과 로그는 `tests/.runtime/backend-isolated/billing-documentation.log`입니다.

```bash
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

## 기존 발견 항목의 실제 저장소 검증

### F3-CREDIT-001 — 준비 단계 오류 후 차감만 남음

[기존 결함 기록](../docs/findings.md)의 Controller/credit mock 검사를 실제 HTTP/JPA/MySQL로 확장했습니다. 두 실행에서 PRICE/BENCHMARK/RISK_FREE/OVERLAY 모두 비용3이 commit되고 HTTP500을 반환했으나 잔액2·원장[−3]·리포트0이었습니다. 기대값은 잔액5·원장[−3,+3]·동일 referenceId입니다. 이후 FastAPI 오류 대조군은 동일 비용3을 환불해 통과했습니다.

[Feature3AnalyzeController](../backend/src/main/java/com/qaima/api/feat3/Feature3AnalyzeController.java)의 `onErrorResume` 환불은 API 호출 이후 내부 체인에 있습니다. `toFastApiRequest`와 `loadOverlaySignals`는 그 밖에 있어 주입한 준비 오류가 환불 없이 전파됩니다. 운영 사용자의 손실·발생률을 관측한 결과가 아니며, 특히 risk-free/overlay 내부에서 흡수되는 일반 원천 오류와 이번 경계 오류를 구분합니다.

### REPORT-MAPPING-001 — 실제 SQL과 상세 HTTP 모두 필드 뒤바뀜

[AnalysisReportService.createFeature1](../backend/src/main/java/com/qaima/service/report/AnalysisReportService.java)이 command의 stockCode 자리에 회사명을, companyName 자리에 종목코드를 넘깁니다. 두 실행에서 실제 SQL과 `GET /api/v1/reports/{id}` 모두 stockCode=`합성 QA 분석기업`, companyName=`009999`였고 네 독립 단언이 실패했습니다. Feature2의 정상 매핑은 통과했습니다.

최종 실행에서는62자 회사명 `QA company with a valid name longer than thirty two characters`를 실제 stock.company_name에 저장했습니다. 같은 종목의 Feature2는 stock_code=009999/company_name=긴 이름으로 report 저장에 성공했습니다. Feature1은 **MySQL1406/SQLState22001, Data too long for column 'stock_code'**로 저장 실패했고 HTTP200 success에 REPORT_SAVE_FAILED, reportId=null, Feature1 report SQL0행이 남았습니다. 두 분석 차감은 모두 commit돼 잔액5→3·원장[−1,−1]이었습니다.

V44의 stock_code 한도32/company_name 한도255와 실제 SQL 오류가 일치합니다. 예전 문서에서 가능성으로 남긴 저장 실패 영향을 격리 DB에서 확인한 증분입니다. report INSERT 거절 trigger는 이 사례에 사용하지 않았습니다. 분석 응답 성공을 유지하며 리포트 저장 실패만 warning으로 처리한 부분은 기존 정책대로이고, 올바른 회사명 때문에 저장이 실패하는 원인이 회귀 대상입니다. 제품 코드는 수정하지 않고 실패를 보존합니다.

## Swagger 증분과 남은 범위

이번에는 `POST /api/v1/feature1/analyze`, `POST /api/v1/feature2/analyze`, `POST /api/v1/feature3/analysis` 세 경로에 실제 인증/validation/비용/실패/SQL 저장 검사를 추가했습니다. 생성 리포트의 `GET /api/v1/reports/{reportId}` 소유권도 대조합니다. 전체 Swagger123개 완료나 세 분석의 계산 정확도 판정은 아닙니다.

실제 가격/benchmark/금리·뉴스/peer/산업·FastAPI/LLM 연결, 결과 수학, 여러 backend 프로세스, 네트워크 단절/클라이언트 취소/서버 재시작 도중 보상, 환불 복구·일반 멱등성은 남아 있습니다. [진행 기록](QA_PROGRESS_2026-09-28.md)에서 전체 QA 잔여 범위를 이어갑니다.
