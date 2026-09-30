# Feature1 실제 Redis 장애·HTTP 취소/재시도·SQL QA

## 현재 상태

2026-09-28 최종 `run-rqzhknkk`: **17개16 PASS/1 FAIL**, errors/skipped0, JUnit127.216초·Gradle4분17초입니다. 유일한 FAIL은 기존 `F1-ASSEMBLY-ISOLATION-001`의 Redis+가격 SQL 복합 장애 재현입니다. 실행기 exit1은 이 제품 실패의 결과입니다. 정상 기준1개는 중복 합산하지 않습니다. 이 단계까지 누적은 **28묶음592개450 PASS/142 FAIL**이며 전체 QA는 미완료입니다.

## 명령과 경계

```bash
python3 -B tests/run_isolated_backend.py --suite feature1-resilience
python3 -B tests/run_isolated_backend.py --suite feature1-resilience --test aBaselineColdWarmActualSqlCalculationAndCache
```

[검사 코드](java/com/qaima/qa/IsolatedFeature1ResilienceTest.java)는 기존 [실제 Feature1 SQL·계산 구성](isolated-feature1-pipeline.md)을 재사용합니다. 실제 HTTP/JWT→Controller/CreditService→FeatOneService·가격/재무/발행주식 JPA·snapshot/시세 비율→Redis/Lettuce→FastAPI 계산→metrics 캐시/report SQL·상세 소유권을 연결합니다. 외부 시세/캔들 제공자는 합성이며 브라우저/실제 금융 제공자/유료 LLM은 이 단계에 없습니다.

- 새 외부 격리 MySQL·Redis만 사용합니다. [소유 TCP 프록시](redis_fault_proxy.py)의 partition은 연결 종료/재접속 거절, blackhole은 바이트 폐기, restart는 소유 Redis 프로세스의 정상 종료와 같은 포트 재기동입니다. 프록시는 Redis 응답을 재작성하지 않습니다.
- 실제 분석 client의 Lettuce command timeout은 제품 설정과 같은 2초입니다. 같은 client의 재접속을 45초 관측 한도로 측정하며 factory를 교체하지 않습니다. HTTP110초/JUnit180초는 테스트 종료 장치입니다.
- [FastAPI 전송 helper](feature1_pipeline_asgi.py)는 nonce로 보호한 loopback에만 노출합니다. `hold`는 실제 요청 본문을 받은 뒤 제품 계산 진입을 보류하고, 실제 socket disconnect를 관측하거나 `release` 뒤 원래 본문으로 실제 제품 계산을 실행합니다. 원천/계산 반환 경계의 Reactor gate와 stage subscribe/value/CANCEL을 함께 기록합니다. 제품 timeout을 새로 삽입하지 않습니다.
- 정상 fixture는 SQL가격160행/재무6행/주식수1000/시세180입니다. 기존 독립 EMA480개/null·TTM 기대값과 실제 FastAPI/public/SQL/상세/Redis 값을 대조합니다. 취소된 오래된 계산과 새 계산의 경합은 SQL 가격을100만큼 보정하고 lastClose179.5/279.5를 구분합니다.
- 정상/부분 결과·warning·USE/REFUND·잔액·report와 후속 복구를 분리합니다. metrics stale 키는 [현재 캐시 정책](../policy/cache_policy.md)에 write-only로 명시되어 있으므로 없는 read fallback을 임의 요구하지 않습니다. 시세 stale 대체는 별도 경로입니다.
- [공개 정책](../policy/QAIMA_POLICY_PUBLIC.md)의 핵심 실패 환불과 취소를 구분합니다. 취소 환불 시점과 공개 idempotency key는 명시되지 않았으므로 취소 잔존 차감/동일 요청 두 차감은 관측으로 기록합니다. 취소 전파/재시도 PASS를 과금 정책 적합성 PASS로 확대하지 않습니다.

각 실행의 summary/XML/응답·원장·캐시/원천 단계·TCP/FastAPI 증거·입력 해시와 소유 서비스 정리를 `tests/.runtime/backend-isolated/runs`에 보존합니다. 제품/기존 DB는 수정하지 않으며 새 코드와 문서는 tests 내부에만 작성합니다. 전체 기준과 남은 범위는 [상위 인계](FALLBACK_QA_HANDOFF.md), [누적 진행 기록](QA_PROGRESS_2026-09-28.md)을 따릅니다.

## 최종17사례

| JUnit 실행 이름 | 판정 | 입력·검증 방법 |
|---|---|---|
| aBaselineColdWarmActualSqlCalculationAndCache() | PASS | cold→warm 실제 분석2회. 독립 EMA480개/null·TTM 비율·Redis fresh/stale 값/TTL·원장/SQL을 대조. quote1회 재사용, 계산2회. |
| cancelledOlderCalculationCannotOverwriteNewerMetricsOrSql() | PASS | 옛179.5 실제 계산 반환을 gate로 보류→SQL가격+100→새279.5 분석/SQL/캐시 완료→옛 HTTP 취소/해제→새 snapshot/캐시 불변·세번째279.5 확인. |
| actual F1 HTTP cancellation at CANDLE | PASS | 외부 캔들 Mono를 gate로 보류→차감 후 실제 HTTP future 취소→CANDLE CANCEL·report0/원장→동일 body 재분석. |
| actual F1 HTTP cancellation at QUOTE | PASS | 시세 원천 Mono 보류→차감 후 HTTP 취소→QUOTE CANCEL·report0/원장→동일 body 재분석. |
| actual F1 HTTP cancellation at API | PASS | 실제 Python HTTP 본문 수신 후 hold→HTTP 취소→Spring API CANCEL/Python socket disconnect·report0→재분석. |
| actual F1 HTTP cancellation at METRICS | PASS | 실제 계산 완료 직후 Redis blackhole→metrics SET 구독 중 HTTP 취소→METRICS CANCEL·report0→Redis 복원/재분석. |
| heldActualApiHasNoEarlyCompletionAndReleasePreservesResult() | PASS | 실제 API hold10초 동안 미완료·잔액/report 관측→release 뒤 원래 요청으로 실제 계산/SQL 완료. timeout SLA 적합성 판정은 아님. |
| metricsWriteFailureAfterCalculationKeepsPaidSqlReport() | PASS | 실제 계산 반환 전 gate→Redis blackhole→계산 반환→cache SET timeout에도 계산/차감/SQL 유지→같은 client 복원/재분석. |
| realHttpClientTimeoutCancelsActualPythonSocketAndRetryWorks() | PASS | 실제 HttpClient3초 timeout→보류된 FastAPI 구독 CANCEL/실제 socket disconnect→report/원장 관측→동일 body 재시도. |
| redisAndCalculationFailureKeepFallbackAndRecover() | PASS | Redis blackhole+실제 FastAPI503→schema0.1/160가격·재무/계산 경고·SQL 유지→둘 다 복구/정상 재분석. |
| redisAndPriceSqlFailureMustKeepIndependentFinancials() | FAIL | Redis partition+실제 price_ohlcv 테이블 숨김→독립 재무 보존을 기대하나 data=null. 환불 정상. 원천/Redis 복원 후 정상 분석. |
| redisAndQuoteFailurePreserveIndependentFinancials() | PASS | 정상 캐시 생성→Redis partition+quote 오류→접근 불가한 stale 대신 시세/valuation null·부분 경고, 기술지표/ROE12.5 유지→복원. |
| redisRestartRepopulatesMetricsAndPreservesPriorSql() | PASS | 소유 Redis 실제 종료/동일 포트 재시작·run_id 변경/빈 캐시→같은 client 재분석·캐시 재생성·이전 SQL 불변. |
| actual F1 Redis partition | PASS | cold 연결 종료/재접속 거절→실제 계산/SQL·cache 없음→같은 client 복구·동일 body 재분석. |
| actual F1 Redis blackhole | PASS | cold 실제 TCP 바이트 폐기→Redis timeout 누적→계산/SQL 유지·cache 없음→동일 client 복구. |
| restoreDuringQuoteRepopulatesSameRequestAndNextRequest() | PASS | Redis partition의 조회 실패 후 quote gate 진입→같은 client Redis 복원→quote 해제→동일 요청/다음 요청의 수치·캐시/SQL 확인. |
| twoIdenticalConcurrentRequestsRecordSeparateChargesAndReports() | PASS | 동일 body2개를 실제 API hold에서 겹침→차감2/리포트0 관측→함께 해제→계산/원장/reference/report2개 확인. 멱등성 정책 통과로 해석하지 않음. |

## 판정과 관측

- **F1-ASSEMBLY-ISOLATION-001 유지**: Redis 연결 거절과 실제 price_ohlcv SQL 오류가 겹치면 정상 financial6행도 public에서 사라져 data=null/meta failure가 됩니다. FastAPI 호출0·report0, 같은 reference의 USE−1/REFUND+1로 잔액5를 유지합니다. 장애 해제 뒤 같은 body의 실제 계산/SQL은 정상입니다. 환불 PASS가 독립 재무 fallback FAIL을 해소하지 않습니다.
- 네 명시 취소·실제 HTTP timeout1개·늦은 이전 계산 취소1개, 총6요청은 차감−1이 남고 해당 리포트가 생성되지 않았습니다. 취소 전파/후속 정상 분석은 확인했으나 정책이 명시하지 않은 취소 환불의 적합성으로 판정하지 않습니다. 네 명시 취소는 각 잔액5→4/report0, 복구 분석 후3/report1입니다.
- 동시 동일 body2개는 각각 다른 Controller reference와 USE를 만들고 별도 report2개가 저장됐습니다. 공개 idempotency key가 없다는 현재 계약에서 관측한 결과이며, 사용자 재시도의 이중 차감 방지 완료 증거가 아닙니다. 응답을 기다린 순서와 report ID 순서가 다름도 보존합니다(첫 future report28, 둘째 report27).
- 취소된 옛 계산은 실제 Python이 만든179.5 결과를 제품에 반환하기 직전에 보류했습니다. 새279.5 결과의 완료 뒤 옛 요청을 취소/해제해도 새 Redis metrics와 이미 저장한 SQL snapshot은 유지됐고 세번째 요청도279.5였습니다. 여러 프로세스/취소되지 않은 늦은 요청/모든 원천 경쟁을 통과한 것은 아닙니다.
- 실제 API hold10초 동안 분석은 미완료·차감−1/report0이었고 release 후 정상 계산/SQL을 보존했습니다. 제품 `WebClientConfig.analysisWebClient`/`FastApiAnalysisClient`에 명시된 요청 timeout이 없음을 함께 확인했습니다. 10초 관측을 무한 대기의 증명이나 명시되지 않은 SLA 위반으로 확대하지 않습니다.
- metrics 캐시 write 실패만으로 분석 결과/리포트 SQL은 소실되지 않았습니다. cold Redis 장애에서는 raw owner 연결로도 metrics fresh/stale 없음이 확인됐고, 정상 복구 후 같은 분석 client가 두 키를 다시 만들었습니다. 시세+Redis 오류에서는 실제 단계 PRICE_FETCH_FAILED·public 부분 지표 경고와 valuation null을 확인했습니다.

## 시간·전송·저장 대조

| 최종 관측 | 실제 소요/결과 |
|---|---|
| cold Redis partition | 12.541초, 정상 실제 계산·SQL 보존 |
| cold Redis blackhole | 12.642초, 정상 실제 계산·SQL 보존 |
| Redis blackhole+계산503 | 12.623초, 가격/재무 fallback·SQL 보존 |
| warm Redis 접근 불가+quote 오류 | 10.487초, 기술지표/ROE 보존·valuation null |
| 실제 계산 후 metrics SET 실패 | 2.187초, 정상 수치/리포트 SQL 보존 |
| Redis 조회 실패 중 quote 보류→복원 | 같은 요청9.101초, 후속 정상0.072초 |
| 같은 client Redis 재접속 확인 | 52회 관측 중 최대6.698초/4시도; 테스트 관측 한도45초 |
| 실제 API hold→release | 보류10.000초, 전체10.224초 |

시간은 이 격리 환경/합성 원천 조건의 관측입니다. 단일 요청의 Redis2초 제한이 여러 조회/저장 단계에서 누적됩니다. 운영 지연 분포나 최대 응답시간/SLA를 측정한 것이 아닙니다.

[별도 증거 검사기](audit_feature1_resilience.py)는 공개 응답29개/SQL snapshot=소유자 상세28개/타사용자404×28, USE35/REFUND1·원장 상태37개, 실제 FastAPI32개를 대조합니다. 실제 Python200은29개이고 그중27개는 public metrics와 일치합니다. 나머지2개는 실제 계산 후 취소된 결과(늦은 옛 계산/metrics 저장 중 취소)이며 public 성공으로 세지 않습니다. 503은1개, 실제 socket disconnect로 HTTP 응답이 없는 hold2개입니다.

wire의 가격5120행분/재무192행분을 독립fixture/SQL가격+100 조건과 비교했고 캐시 metrics48개 관측을 실제 응답/Python 결과와 대조했습니다. 반복 대조 횟수이며 고유 입력 행 수가 아닙니다. 실제 Redis 재시작1회(이전 exit0·새 run_id·소유 표식 전 DB0), TCP 거절84회/폐기162338bytes를 확인했습니다. 원장35개는 완료된29요청과 미완료 취소6개이며 모두 정상 과금으로 판정하지 않습니다.

```bash
python3 -B tests/audit_feature1_resilience.py tests/.runtime/backend-isolated/runs/run-rqzhknkk
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

최종 run의 `feature1-resilience-evidence-audit.json`/로그, `summary.json`, XML, `fastapi-exchanges`에 근거가 있습니다. 증거 감사 PASS는 실행 기록의 일관성에 대한 판정이며 제품17사례 전부 PASS라는 의미가 아닙니다.

## 초기 실행·정리·남은 범위

정상 기준 `run-es3dgtup`은1 PASS(cold6.818초/warm0.188초·quote원천1회·계산/SQL2회)입니다. 이후 취소 경합의 새 SQL snapshot 불변 단언을 추가하고 전체17개를 실행했습니다. 초기 테스트 구성 오류/재실행 오탐은 없으며 정상 기준을 최종17개와 중복 합산하지 않습니다.

최종 입력613개 실행 중 변경0입니다. 두 실행 모두 업무9/원천4테이블·QA종목·숨긴테이블·trigger0, Redis DBSIZE0/MySQL·Redis exit0, FastAPI active/outbound/dotenv/subprocess시도0·SIGTERM−15 종료, 프록시/제어/worker 종료·active socket0을 확인합니다. tests 밖924개 보존, 문서71개 대상 검사2개, 두 실행 정리/최종17사례·누적28묶음 대응은 최종 run의 `history-and-documentation-audit.json`에 기록합니다. 현재 파일 hash를 비교하는 감사는 이후 테스트 입력 변경 뒤 과거 run에 무조건 재실행하지 않습니다.

Feature1 전체 제공자/시장·브라우저 취소·프로세스 중단/여러 backend·취소 환불/멱등성 정책, Feature2 미해결 캐시/조립/화면 실패와 전체 외부 제공자 조합, Feature3 원천 장애·취소/재시도·전체 Swagger 사용자 흐름은 여전히 남아 있습니다. 제품 결함을 수정하거나 전체 목표를 완료로 표시하지 않았습니다.
