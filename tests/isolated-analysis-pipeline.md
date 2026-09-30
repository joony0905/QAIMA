# Feature3 실제 HTTP·FastAPI 계산·MySQL 리포트 통합 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03`와 기존 미커밋 소스를 대상으로 합니다. [검사 클래스](java/com/qaima/qa/IsolatedAnalysisPipelineTest.java), [실행기](run_isolated_backend.py), [FastAPI 동반 실행기](isolated_fastapi_fixture.py), [ASGI 관측기](pipeline_asgi.py), [독립 입력/기대값 생성기](build_pipeline_fixture.py)는 모두 tests 안에 있습니다.

## 이전 검사에서 확장한 경계

[분석 과금/리포트 검사](isolated-analysis-billing.md)는 실제 HTTP·MySQL/Redis를 사용했지만 계산 응답은 mock이었습니다. [이전 실제 TCP 계산 검사](../backend/src/test/cross-os-fastapi.md)는 FastAPI를 실행했지만 원장·리포트 저장소가 mock이었습니다. 이번에는 아래 경계를 같은 분석 요청으로 연결합니다.

`Java HttpClient → 실제 TCP/WebFlux 보안필터/JWT·validation → Feature3AnalyzeController → CreditService의 MySQL 차감 → 실제 FastApiAnalysisClient/WebClient → uvicorn/FastAPI/Pydantic·analyze_portfolio·결정론적 설명 → Public DTO·Feature3OverlayService → AnalysisReportService/JPA/MySQL snapshot → 소유자 상세 HTTP`

| 경계 | 실제 실행 또는 대체 |
|---|---|
| 인증/저장/과금 | 실제 제품 JWT 검증, CreditService의 row lock/transaction·원장, report/portfolio service·JPA/MySQL. 사용자 A/B/관리자는 새 합성 계정 |
| 종목/시장 | 실제 stock/Exchange SQL 삽입과 StockRepository의 시장 조회. QAA/QAB는 KOSPI, QAC는 KOSDAQ |
| 가격·벤치마크·무위험금리 | 세 service 경계를 고정 시계열 DTO로 대체. 원천 수집/가격 SQL/실제 시장 데이터는 실행하지 않음 |
| 계산 | 제품 FastApiAnalysisClient의 snake_case 변환·TCP·응답 변환, 실제 Python 계산/최적화/설명. 기대값 생성에 제품 계산 함수를 사용하지 않음 |
| overlay/Redis | 실제 Feature3OverlayService와 새 Redis. 별도 사례에서 세 종목의 합성 재무 metrics를 실제 캐시에 저장하고 신호를 실제 Python으로 전달 |
| LLM/외부 서비스 | includeLlmExplain=false, 제공자 자격정보 제거, Python outbound/subprocess/dotenv 읽기 차단. 결정론적 설명만 실행 |
| 브라우저 | 이번 단계에는 포함하지 않음. 앞선 Chrome fixture 검사와 실제 계산/SQL 결과를 연결하는 후속 범위 유지 |

전체 QaimaApplication을 띄우지 않고 [HTTP/JPA 명시적 구성](isolated-http-jpa.md)에 Feature3 Controller/실제 분석 client만 추가했습니다. 운영 application 설정·scheduler·기존 DB/Redis는 사용하지 않습니다. 테스트를 rollback transaction으로 감싸지 않고 HTTP 응답 뒤 별도 JDBC로 commit 상태를 확인합니다.

## 격리와 재현

```bash
cd /mnt/c/qaima
python3 -B tests/run_isolated_backend.py --suite analysis-pipeline
```

선택 메서드도 새 서버/DB를 생성합니다:

```bash
python3 -B tests/run_isolated_backend.py --suite analysis-pipeline --test successfulCalculationMatchesIndependentCovarianceAndOwnedSqlSnapshot
```

MySQL8.0.46과 Redis7.0.15의 추출본, Java17/Gradle8.14, 기존 `.venv_wsl`을 사용합니다. 매번 새 외부 `/tmp/qaima-qa-http-*` datadir·무작위 loopback 포트/schema를 만들고 SQL51개를 적용합니다. 업무47개+소유표식1개 테이블을 만들며 Flyway history 검사와는 구분합니다. DB port/datadir/schema/소유표식을 Python과 Java에서 확인합니다. Redis도 새 외부 경로·포트·임의 인증·소유표식으로 보호합니다.

FastAPI는 `127.0.0.1`의 소유 listener FD로 시작합니다. 임의 nonce가 없으면403인 [기존 테스트 ASGI guard](../analysis/tests/tcp_test_app.py)를 재사용하고, [새 관측기](pipeline_asgi.py)는 합성 분석 JSON 요청/응답을 읽는 동작만 추가합니다. 요청 본문이나 응답을 변경하지 않습니다. 상속된 앱 연결/키·토큰·proxy 설정을 제거하고 dotenv/가격 fetch/뉴스 warmup과 HF 네트워크를 끕니다. Python audit hook의 차단 시도도0인지 검사합니다.

실행기는 소유 FastAPI·Redis·MySQL을 finally에서 종료하고 listener 부재와 종료코드를 기록합니다. 종료 과정에서 한 서버의 검증이 실패해도 나머지 서버 종료를 수행합니다. 기존 프로세스를 찾아 종료하거나 DB 설정을 변경하지 않습니다.

## 합성 입력과 독립 기대값

[기존 수학 fixture](../analysis/tests/portfolio_fixture.py)의 평일141개 가격/로그수익률140개를 사용합니다. 서로 다른 두 합성 시장 수익률과 세 종목의 사인/코사인 수익률을 가격으로 적산합니다. NumPy의 표본 공분산(ddof1)×252를 독립 기대 공분산으로 저장합니다.

수량2/3/4와 명시된 현재가100/150/80, 현금30의 총평가액은1000입니다. 종목 비중은0.20/0.45/0.32, 현금0.03이며 현재 변동성은 `sqrt(wᵀΣw)`로 계산합니다. 현재가 생략 사례는 각 제공 시계열의 마지막 가격으로 비중과 변동성을 별도 산출합니다. 원래 입력의 평균단가90을 현재가로 대신 기대하지 않습니다.

위험회피계수5, 최대현금0.2, 표본 공분산, 연율252를 지정합니다. MIN_VOL/MAX_SHARPE/RISK_ALLOCATION/UTILITY_OPTIMAL 후보의 비중 합1·비음수·현금≤0.2·종목≤2/3, 효용 `수익률−2.5×변동성²`에서 UTILITY_OPTIMAL≥RISK_ALLOCATION을 검사합니다. 이 유한 입력의 제약/효용 비교이며 모든 최적화 문제의 전역 최적성 증거는 아닙니다.

각 성공 요청마다 HTTP data=실제 SQL result_snapshot_json=소유자 상세 resultSnapshot을 구조적으로 비교합니다. 저장 request options도 제출값과 대조하고 타 사용자 상세404를 확인합니다. 리포트 저장 장애는 소유 schema의 실제 BEFORE INSERT trigger로 주입합니다.

완료된 실행의 실제 TCP 응답을 [수학 증거 검사기](inspect_pipeline_evidence.py)로 추가 검사합니다. 제품 모듈·DB를 사용하지 않고 fixture/캡처 파일의 보존 해시를 확인합니다. 가격의 로그 차분으로 공분산을 다시 만들고, 네 투자 수준의 CURRENT 및 네 제약 후보에 대해 `sqrt(wᵀΣw)`, 종목 변동성, 한계 위험기여도 `Σw/σ`, 위험기여도 `wᵢ(Σw)ᵢ/σ`, 기여 비율을 비교합니다. 응답은 소수6자리이므로 반올림한 비중을 재사용할 때의 전파 오차를 포함해 절대 허용오차2×10⁻⁶를 사용합니다. 기대수익률은 반환된 종목별 blendedExpectedReturn의 가중합+현금수익과 비교하므로 과거/CAPM 추정 자체의 독립 정확성 검사가 아닌 계산 내부 일관성 검사입니다. 투자 수준 간 수익·변동성·샤프·비중·위험 등급/상태도 대응 비교합니다. 이는 JUnit18사례의 보강 증거이며 별도 사례로 중복 집계하지 않습니다.

## 18개 사례의 검증 방법

메서드15개 중 투자수준 메서드가4개 invocation입니다. 아래 성공 경로는 공통 helper에서 실제 SQL 저장·소유자 상세 조회·다른 사용자404를 함께 검사합니다. 가격 fixture의 cache_status/fallback_used는 실제 Java service의 non-null 메타데이터를 명시합니다. Java record의 null 직렬화가 Python의 생략 필드 기본값과 같다고 가정하지 않습니다.

최종 `run-4b_55m01`에서 아래 **18개 모두 PASS**, errors/skipped0, JUnit25.344초·Gradle exit0입니다.

| JUnit 메서드 | 입력과 직접 확인하는 결과 |
|---|---|
| `successfulCalculationMatchesIndependentCovarianceAndOwnedSqlSnapshot` | 현재가·현금 명시. 독립 비중/공분산 변동성·후보 제약/효용, 분석1회·차감1회·설명 존재. 가격 구독 시 별도 JDBC 잔액4로 선행 차감 commit 확인 |
| `allInvestmentLevelsKeepActualNumericalCalculation` | 초급/중급/고급/전문가4회. 같은 비중·변동성·최적화 제약과 효용, 각 SQL snapshot/원장 확인 |
| `allMissingPricesReturnDocumentedFallbackAndPersistCharge` | 세 가격의 available0/missing1/data[]와 유효한 EMPTY source. 실제 대체 결과 NEAREST_FEASIBLE·공통표본0·포함0/제외3·CORE_RISK_PARTIAL_FALLBACK, 저장과 차감 확인 |
| `englishDeterministicExplanationPreservesActualMathAndSavedLanguage` | languageCode=en. 독립 수치 유지, 설명에 영어가 있고 한글이 없는지, 저장 옵션·차감 확인 |
| `missingCurrentPricesUseActualLastSeriesPrices` | currentPrice 전부 생략. 제공한 시계열의 마지막 종가로 별도 산출한 비중/변동성 비교 |
| `missingOneBenchmarkPreservesOtherCapmAndHistoricalFallback` | KOSDAQ benchmark unavailable/빈 값. KOSPI CAPM 비중>0, QAC CAPM 비중0·혼합 기대수익=과거 기대수익, 전체 분석/저장 성공 |
| `savedDefaultPortfolioCanFeedActualAnalysisAndPersistReport` | 기본 포트폴리오 PUT→GET→읽은 보유3개로 분석. 실제 holding3행, 현재가 없는 별도 수치 기대값과 저장 결과 확인 |
| `laterAnalysisDoesNotMutatePreviouslySavedCalculatedSnapshot` | 현금30 분석 후1030으로 재분석. 변동성 변경, 기존 report 상세 snapshot 불변, report2개/차감2회 |
| `cachedFundamentalsReachRealOptimizerWithoutChangingCoreMathOrExtraCharge` | core 분석 후 새 Redis에 재무 metrics3개 저장. REUSE_AVAILABLE/fundamentals로 실제 overlay 신호3개·조정 포트폴리오 생성, core 수치 불변·분석당1크레딧·Feature1 호출0 |
| `publicValidationStopsBeforeChargeAndFastApi` | quantity0은 Public400, 계산 호출·원장·report0 |
| `realFastApiValidation422RefundsCommittedCostAndCreatesNoReport` | Java에서 허용하는 QA_INVALID 공분산명. 실제 Python422의 단일 오류 위치 options/covariance_model, Public500, 동일 reference 차감−1/환불+1·report0 |
| `unsupportedSourceResponseFailureRefundsCommittedCostAndCreatesNoReport` | 정상 가격 한 종목의 source만 QA_UNSUPPORTED_SOURCE. Python 입력은 허용하지만 출력 품질 enum 검증에서 실제500. Public500·동일 reference 차감/환불·report0. 외부 제공자 metadata 오류 주입이며 정상 빈 가격 경로와 구분 |
| `reportSqlFailureKeepsRealCalculationAndChargeWithWarning` | 새 schema의 실제 report INSERT trigger로 SIGNAL. 실제 계산 수치/HTTP200 유지, reportId0·REPORT_SAVE_FAILED, 차감−1 유지·report0 |
| `insufficientBalanceDoesNotReachRealCalculation` | 실제 잔액0. HTTP402·계산/원장/report0 |
| `anonymousRequestCannotReachRealCalculationOrWriteMoney` | JWT 없음. HTTP401·계산/원장0·기존 잔액5 유지 |

## 실행 이력

최초 `run-o55bf97b`는17개5 PASS/12 FAIL이었으나 정상 계산 경로는 fixture의 cache_status/fallback_used null로 실제422를 받아 진행되지 않았습니다. 의도한 공분산422에도 같은 오류가 섞였고, 빈 가격500 사례는 source=FIXTURE라는 잘못된 enum 때문에 실패했습니다. 따라서 이 실행의 PASS를 의도한 계산/가격 부족 처리의 성공 증거로 쓰거나12개를 제품 결함에 추가하지 않습니다. 테스트 데이터 기본값과 실패 주입의 의미를 보정하고, 정상 빈 가격의 대체 결과1개를 추가해18개를 재실행합니다. 최초 산출물은 그대로 보존합니다.

두 번째 `run-5cto3ma3`는18개16 PASS/2 FAIL, errors/skipped0입니다. 두 실패는 비동기 종목 조회 뒤 배열 순서가 같다고 가정한 테스트 오류였습니다. overlay 적용 전후 core의 모든 값은 같았지만 weights/riskContributions 배열 순서가 달랐고, 잘못된 source를 넣은 QAA가 전송 배열의 첫 요소가 아니었습니다. 코드별 중복을 금지하고 종목 코드로 대응해 모든 필드를 비교하도록 테스트만 보정했습니다. HTTP 응답과 같은 요청의 SQL/detail snapshot 비교는 계속 배열 순서를 포함해 엄격하게 수행합니다. 이 두 실패도 제품 결함에 더하지 않습니다.

최종 증거는 [summary](.runtime/backend-isolated/runs/run-4b_55m01/summary.json), [JUnit XML](.runtime/backend-isolated/runs/run-4b_55m01/TEST-com.qaima.qa.IsolatedAnalysisPipelineTest.xml), [독립 수학 대조](.runtime/backend-isolated/runs/run-4b_55m01/independent-math-audit.json), [실제 TCP 요청/응답17개](.runtime/backend-isolated/runs/run-4b_55m01/fastapi-exchanges), [FastAPI 감사](.runtime/backend-isolated/runs/run-4b_55m01/fastapi-audit.json)에 있습니다. 외부 제공자 데이터나 사용자 비밀정보가 아닌 이 테스트의 합성 본문만 보존합니다.

```bash
.venv_wsl/bin/python -B tests/inspect_pipeline_evidence.py tests/.runtime/backend-isolated/runs/run-4b_55m01
```

- 실제 Python 응답은200×15,422×1,500×1입니다. 세 요청은 Java 인증/잔액/입력 검증에서 중단했고, 두 사례는 각각 분석2회를 수행했습니다. 정상 계산15회 중 report 저장 장애1회를 제외한14개 snapshot을 각각 HTTP·SQL·소유자 조회로 비교했습니다.
- 독립 현재 변동성0.06431484731932816→응답0.064315, 현재가 없는 경우0.07248642201956891→0.072486입니다. 정상 공통 가격141개/수익률140개와 포함종목3개를 확인했습니다. 전부 빈 가격에서는 공통표본0·포함0/제외3·대체 변동성0.1652·경고를 저장했고1크레딧을 차감했습니다.
- core와 캐시 재무 overlay 분석 각각1크레딧, 실제 Redis 신호3개·조정 결과 생성, 기존 core 값 불변이 확인됐습니다. 계산422/500은 동일 reference로−1/+1, report SQL 장애는−1 유지·저장 경고였습니다. 이를 기존 입력 준비 단계의 환불 누락까지 해결된 것으로 보지 않습니다.
- 네 투자 수준×현재/네 후보의 독립 위험 계산·정책 기대수익 일관성·수준 불변 대조420개는 모두 PASS이며 최대 절대오차0.000000765821입니다. 각각의 실제/기대값·허용오차는 별도 JSON에 기록합니다. 전역 최적성·실제 금융 모델 정확도 검증으로 확대하지 않습니다.
- 세 실행 모두 입력598개 실행 중 변경0·업무9테이블/QA종목/trigger0, Redis 최종DBSIZE0, MySQL/Redis exit0 및 종료 확인입니다. FastAPI는 소유 PID에 SIGTERM을 보내 exit−15/포트 닫힘을 확인했으며 강제 kill은 없었습니다. Python outbound/subprocess/dotenv 접근 시도도0입니다. 운영 DB·기존 Redis는 사용하지 않았습니다.
- tests 밖 코드/설정/문서924개의 변경·소실·새 파일0을 [감사 기록](.runtime/backend-isolated/pipeline-source-audit.json)에 남깁니다. 선택한 확장자와 가시 파일에 대한 감사이며 모델 바이너리/의존성 등 제외 범위를 유지합니다.
- [메서드/문서 대조](.runtime/backend-isolated/runs/run-4b_55m01/case-documentation-audit.json)는15메서드/문서15행/실행18사례 일치입니다. 루트 `unittest discover -s tests -v`의59개 문서 대상 링크·설정 일부 비밀값 검사2개도 PASS이며 [실행 로그](.runtime/backend-isolated/pipeline-documentation.log)를 보존합니다.

## 검증 한계

가격·벤치마크·금리 service 경계는 합성이며 실제 원천·가격 저장소·외부 유료 설명을 포함하지 않습니다. 로그인 화면/브라우저 쿠키·차트/PDF·전 범위 API·프로세스 중단/네트워크 복구·여러 backend 인스턴스는 별도입니다. 일반적인 과금 멱등성이나 모든 실패의 환불을 보증하지 않으며 기존 입력 준비 오류의 환불 누락은 [기존 FAIL](isolated-analysis-billing.md)로 유지합니다. 제품 소스는 수정하지 않습니다.
