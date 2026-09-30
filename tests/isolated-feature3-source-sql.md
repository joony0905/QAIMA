# Feature3 실제 원천 SQL·대체 조회·계산·과금 검증

최종 실행: **17개12 PASS/5 FAIL**, `run-sphw2mwo`, errors/skipped0, JUnit120.621초, Gradle4분9초/exit1입니다. 핵심 실패 환불 누락3개·금리 empty1개·가격 freshness1개를 제품 FAIL로 남깁니다. 앞선 준비 오류/선택 실행/초기 전체 실행은 누적에서 제외합니다.

## 방법과 경계

[검사 클래스](java/com/qaima/qa/IsolatedFeature3SourceSqlTest.java)는 [앞선 계산 통합 검사](isolated-analysis-pipeline.md)의 가격/벤치마크/금리 DTO 대체 경계를 실제 서비스와 JPA로 확장합니다. HTTP/JWT → 차감 SQL → 실제 Feature3PriceSeriesService/CandleLoadService/YahooFeature3PriceProvider, Feature3BenchmarkSeriesService, Feature3RiskFreeRateService/BondYieldSyncService → 실제 FastAPI 포트폴리오 계산 → 공개 DTO → report SQL → 소유자/다른 사용자 상세 HTTP를 연결합니다.

수정은 tests 내부에 한정합니다. 매 실행 [격리 실행기](run_isolated_backend.py)가 새 `/tmp` MySQL/Redis와 소유 FastAPI를 시작하며 기존 사용자 DB/서비스에 접근하지 않습니다. Yahoo/KIS/BOK의 외부 client는 합성 응답/오류입니다. 종목 조회 adapter는 실제 StockRepository에 위임합니다. 원천 service 대리 객체는 실제 서비스에 인자를 그대로 전달하고 결과/종료를 관측합니다. 실제 외부 제공자·브라우저·취소·Redis 장애·overlay 조합은 후속 범위입니다.

가격 3종목×141행, 벤치마크2종×141행, KR3Y 2%를 실제 SQL에 삽입합니다. 제품 거래일 서비스가 선택한 최신 거래일부터 역산한141일을 사용하며 날짜를 실행 증거에 남깁니다. 수치는 기존 독립 NumPy fixture를 SQL의 소수6자리로 반올림합니다. 실제 공분산/변동성은 기존 독립 기대값과 허용오차1e−6으로 대조합니다. SQL 값/실제 전송 JSON/공개 응답/원장/저장 상세를 후속 증거 감사에서 대조합니다.

장애는 소유 DB 테이블 rename/원천 행 제거/날짜7일 이동/리포트 INSERT trigger와 외부 client 경계 오류로 주입합니다. 재접속·재시작 없이 장애를 제거한 같은 서비스에서 동일 입력을 재실행합니다. 정리 전에 원천 subscription 종료를 확인하고 테이블/행/trigger를 복구·제거합니다.

## 재현

```bash
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite feature3-source-sql
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite feature3-source-sql --test baselineActualSourceSql
```

## 판정 기준

[공개 정책](../policy/QAIMA_POLICY_PUBLIC.md)의 부분 결과/Warning 및 핵심 분석 실패 환불 기준을 적용합니다. 설명/저장만 실패하면 계산 결과와 차감을 유지합니다. 무위험금리 SQL/제공자가 모두 비어도 과금 후 빈 HTTP 응답만 끝나는 상태를 정상 fallback으로 인정하지 않습니다. 가격이 전부 없을 때 이미 명시된 `NEAREST_FEASIBLE`/`CORE_RISK_PARTIAL_FALLBACK` 경로는 계산 품질을 성공으로 과장하지 않고 부분 결과 계약으로 판정합니다.

## 초기 실행

정상 경로 선택 실행 `run-6glg3wdp`는 migration이 이미 벤치마크 master를 생성한다는 점을 테스트가 고려하지 못해 setup에서 실패했습니다. 실제 분석은 실행되지 않았습니다. 기존 master를 재사용하고 새로 만든 행만 제거하도록 tests를 보정합니다. 소유 서비스/업무·원천 행은 정리됐으며 제품 FAIL/최종 누적에 포함하지 않습니다.

보정 후 정상 선택 실행 `run-xvzgj6z6`는1 PASS(JUnit18.819초)입니다. 외부 client 구독0, 실제 SQL→Python 계산/저장/소유권/차감과 서비스 정리·610개 입력 hash 보존을 확인했습니다. 최종 전체 실행에 중복 합산하지 않습니다.

첫 전체 실행 `run-yfzoss8j`는17개11 PASS/6 FAIL입니다.4개는 제품 실패(금리 empty1/준비 SQL 환불3),2개는 테스트 날짜 복원 UPDATE의 기본 순서에 따른 PK 충돌입니다. 가격/지수의 오래된 원천 fallback 자체는 실행됐지만2사례의 후속 복구가 실행되지 않았습니다. 날짜를 앞으로 옮기는 복원은 `ORDER BY ts DESC`로 보정합니다. 실제 오래된 가격의 freshness가 실행 시각으로 표시된 점을 `F3-PRICE-FRESHNESS-001` 단언에 추가합니다. 이 초기 전체 실행도 최종 누적에서 제외합니다. 최초 전체 증거 감사는 PASS이며, 검사 실패가 제품인지 테스트인지의 분류와 구분합니다.

## 재현한 실패와 원인

### F3-CREDIT-001 — 실제 원천 SQL 오류가 환불 구간 밖으로 전파

가격 SQL, 벤치마크 SQL, 두 SQL 동시 장애에서 각각 HTTP500/실제 FastAPI0/report0인데 잔액5→4, USE−1만 남았습니다. 테이블 복원 후 같은 입력은 실제 계산/저장에 성공하지만 추가 차감으로 잔액3이 됩니다. [앞선 과금 검증](isolated-analysis-billing.md)의 경계 mock 실패를 실제 service 내부 JPA 오류로 확장한 재현입니다. 새 고유 결함3건으로 세지 않습니다.

[Controller](../backend/src/main/java/com/qaima/api/feat3/Feature3AnalyzeController.java)의 `toFastApiRequest` 단계가 API 호출 이후의 환불 `onErrorResume` 밖에 있습니다. 원천 오류가 여기서 전파됩니다. 이번 기본 CORE_ONLY 비용은1이며, 앞선 선택 overlay의 비용3 검증과 구분합니다.

### F3-RISKFREE-EMPTY-001 — 금리 SQL과 BOK가 모두 비면 빈 HTTP200과 차감만 남음

소유 `bond_yield` 행을 비우고 BOK client를 빈 목록으로 돌려주면 실제 BondYieldSyncService의 재조회도 비어 완료됩니다. Feature3RiskFreeRateService는 error에는0%/DEFAULT_ZERO를 반환하지만 empty에는 대체값을 반환하지 않습니다. Controller의 zip이 빈 완료로 끝나 **HTTP200/본문0바이트, FastAPI0/report0, 잔액4/USE−1**이 남았습니다. 금리 행을 복원한 후 같은 요청은 계산/저장에 성공하고 잔액3이 됩니다. SQL 오류 또는 BOK 예외의 정상0% 대체 경로와 다른 실패입니다.

### F3-PRICE-FRESHNESS-001 — 과거 DB 가격 보존 후 최신성에 실행 시각 표시

가격423행을7일 과거로 옮긴 뒤 외부 캔들 provider를 실패시키면 실제 CandleLoadService는 기존141행씩을 보존합니다. 계산값/공분산 보존은 통과하지만, 원천 응답은 source=DB/cacheStatus=HIT/fallbackUsed=false/warnings=[]입니다. 실제 전송 가격의 마지막 거래일은2026-09-21인데 공개 `freshness.priceSeriesAsOf`는2026-09-28 실행 시각입니다. `oldestDataAt`/`newestDataAt`도 실행 시각이고 `hasMixedFreshness=false`입니다.

[Python portfolio 조립](../analysis/app/services/portfolio.py)은 가격 행의 마지막 날짜를 읽지 않고 freshness 네 시각을 `now`로 만듭니다. 최신성 표시는 [사용자 흐름 정책](../policy/Feature_analysis_user_flow_policy.md)의 가격/overlay 데이터 최신성 정보와 대조했습니다. 가격 값을 유지했다는 PASS로 오래된 데이터 안내까지 정상이라고 판단하지 않습니다.

## 증거 감사

[독립 감사기](audit_feature3_source_sql.py)는 source hash와 소유 서비스 종료, SQL seed→service 결과→실제 FastAPI JSON, 독립 공분산/변동성, Python→공개 정량, 공개 응답→snapshot→소유자 상세/타 사용자404, 실제 원장의 합계를 대조합니다. 종목 순서는 병렬 조회로 바뀔 수 있어 저장 요청을 종목 코드별로 비교합니다. 내부 임의 dictionary의 camelCase 필드와 현금 식별자도 보존해 비교합니다.

```bash
.venv_wsl/bin/python -B tests/audit_feature3_source_sql.py tests/.runtime/backend-isolated/runs/run-sphw2mwo
```

최종 전체 실행 `run-sphw2mwo`의 증거 감사는 **PASS**입니다. 이는 아래5개 제품 FAIL이 해결됐다는 뜻이 아니라 실행/기록의 일치 여부를 확인한 결과입니다.


## 최종 사례표

| JUnit 실행 이름 | 결과 | 어떻게 검증했는가 |
|---|---|---|
| allPricesBenchmarksAndRateUnavailablePartialFallback() | PASS | 가격/지수/금리 SQL0+세 provider 예외 → 제한적 결과/경고/차감/저장, 원천 복원 후 정상 계산 |
| baselineActualSourceSql() | PASS | 실제 가격423/벤치마크282/채권1행 → 독립 변동성/후보 제약/1차감/SQL/상세/소유권; 외부 구독0 |
| bondSqlErrorUsesZeroAndRecovers() | PASS | 채권 SQL 오류 → rate0/DEFAULT_ZERO 출처 유지, 같은 서비스에서 SQL 복원 후2%/DB 복귀 |
| [1] provider=error | PASS | 금리 SQL0+BOK 예외 →0% 대체 계산/저장, SQL 복원 후2% 정상 계산 |
| [2] provider=empty | FAIL | 금리 SQL0+BOK 빈 목록 → HTTP200/본문0/FastAPI0/report0/USE−1; 복구 후 추가1차감 |
| [1] scope=one | PASS | 가격1종목 SQL0 → 포함2종목/누락 경고/저장; 행 복원 후 포함3종목/정상 수학 |
| [2] scope=all | PASS | 가격3종목 SQL0 → NEAREST_FEASIBLE/포함0/경고/저장; 원천 복원 후 정상 수학 |
| missingBenchmarkMasterKeepsStockRisk() | PASS | KOSDAQ 지수 code 일시 변경으로 실제 master 조회 miss → 경고와 나머지 종목 위험 계산/저장 보존 |
| reportSqlFailurePreservesSourceMathAndCharge() | PASS | report INSERT trigger 오류 → 계산/차감 유지+REPORT_SAVE_FAILED/report0; trigger 제거 후 저장 복구 |
| [1] table=price_ohlcv | FAIL | 가격 SQL rename 오류 → HTTP500/FastAPI0/report0/USE−1/환불0; 복원 후 같은 입력 계산·저장 |
| [2] table=industry_index_ohlcv | FAIL | 벤치마크 SQL rename 오류 → HTTP500/FastAPI0/report0/USE−1/환불0; 복원 후 같은 입력 계산·저장 |
| [3] table=both | FAIL | 가격+벤치마크 SQL 동시 오류 → HTTP500/FastAPI0/report0/USE−1/환불0; 두 테이블 복원 후 정상 계산 |
| staleBenchmarkProviderFailureKeepsPortfolioMath() | PASS | 지수7일 과거+provider 예외 → 기존141행/DB_STALE/available=false와 warning, 정상 주식 변동성/저장; 날짜 복원 후 정상 |
| staleRawProviderErrorPreservesSqlRows() | FAIL | 가격7일 과거+provider 예외 → 기존141행/독립 수학·SQL 보존/복구는 PASS, 가격 freshness9월21일→28일 오표시 FAIL |
| [1] mode=error | PASS | 실제 Yahoo provider가 외부 예외3개 흡수 → RAW fallback/경고/계산/저장; 같은 요청으로 Yahoo 정상 복구 |
| [2] mode=empty | PASS | 실제 Yahoo provider가 빈 응답3개 흡수 → RAW fallback/경고/계산/저장; 같은 요청으로 Yahoo 정상 복구 |
| [3] mode=healthy | PASS | 실제 Yahoo provider가 합성 수정종가141행씩 전달 → fallback=false; 동일 요청 반복의 수학/저장 확인 |

## 대조 결과와 정리

- HTTP 응답32개, 실제 FastAPI28개, snapshot=소유자 상세27개·타 사용자404×27개, USE32/REFUND0을 대조했습니다.32개 차감 모두 정책 PASS라는 뜻은 아닙니다. 앞선 실패4개에는 계산/report 없이 차감이 남았습니다.
- 동일 입력 후속15개 중 장애 제거 후 복구14개, 원래 정상 Yahoo 응답의 반복1개입니다. 초기 오류가 있어도 같은 서비스/Redis를 재시작하거나 캐시를 강제 비우지 않고 복구했습니다.
- 실제 service 결과168개→wire를 대조했습니다. 반복 전송된 가격10,857행/벤치마크7,473행의 SQL 값·날짜를 비교했고, 정상 가격3종목이 있는25응답에서 SQL 소수6자리 기반 독립 공분산/변동성도 일치했습니다. Python→공개 current/basic/riskDrivers/advanced와 공개 riskFreePolicy의 rate/source를 대조했습니다.
- 최종 업무9테이블/원천3테이블/QA 종목·추가 지수·숨긴 테이블·report trigger 모두0, Redis DB0입니다. 마이그레이션이 만든 지수 master는 재사용·복원했습니다. MySQL/Redis exit0, FastAPI SIGTERM−15/포트 종료, outbound/dotenv/subprocess 시도0입니다.
- 실행 입력610개 hash가 실행 전후 일치했습니다. tests 밖924개 원본 보존 및 문서72개 링크/설정값 노출 검사2개 결과는 실행 폴더의 `source-audit.json`, `documentation.log`, `history-and-documentation-audit.json`에 기록합니다.

## 다음 검증과 한계

실제 Yahoo/KIS/BOK 접속·인증·rate limit·기업행사 수정주가 정확성을 검증한 결과가 아닙니다. Yahoo 정상 합성 시계열은 원천 수치 비교를 위해 SQL 종가와 같은 값이며 실제 수정계수 검증으로 확대하지 않습니다. 실제 금리/가격/벤치마크 조립과 core 결과·차감·저장까지 연결했습니다.

다음은 실제 원천을 유지한 Feature3 overlay 조합·Redis 장애·HTTP 취소/timeout·재시도/환불·동시 요청입니다. 실제 원천 결과의 브라우저/저장/PDF 표시, 유료 외부 제공자 및 나머지 전체 사용자 흐름도 남아 있습니다. 전체 QA는 미완료이며 [상위 인계](FALLBACK_QA_HANDOFF.md)와 [누적 기록](QA_PROGRESS_2026-09-28.md)을 계속 사용합니다.
