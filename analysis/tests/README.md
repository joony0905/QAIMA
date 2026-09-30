# 분석 서버 테스트 방법·결과

검증일: 2026-09-25. 루트 `reference.md`의 의도된 명세와 현재 작업 트리를 대조하는 사후 검증입니다. 기존 `feat1_test.json`은 fixture 파일이며 자동화 테스트 실행기가 아닙니다.

## 재현 명령

WSL, 프로젝트 루트에서:

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 PYTHONPATH=analysis \
  .venv_wsl/bin/python -B -m unittest discover -s analysis/tests -v
.venv_wsl/bin/python -B analysis/tests/uvicorn_smoke.py
```

Python bytecode 생성을 끄고, 첫 명령에서는 dotenv 로딩을 비활성화합니다. 실 서버 smoke는 스스로 비밀 환경변수 제거·warm-up 비활성화·테스트 프로세스 종료를 수행합니다. 임시 로그는 `/tmp`의 임시 파일에 두고 프로세스 종료 시 닫습니다. 유료 호출은 0회입니다.

실행 환경: Python 3.12.3, FastAPI 0.136.1, Pydantic 2.13.3, NumPy 2.4.4, SciPy 1.17.1, Pandas 3.0.2, scikit-learn 1.8.0, httpx 0.28.1, uvicorn 0.46.0. 이는 실행 환경의 설치 버전이며 requirements의 최소 버전과 구분합니다.

## 케이스와 독립 기대값

| 파일·범위 | 입력·실행 | 기대값·검사 | 외부 의존성 |
|---|---|---|---|
| `test_feature1.py`: EMA | 종가 1,2,3,4,5, period=3 | null,null,2,3,4. 초기 SMA와 후속 재귀 결과 | 없음 |
| Bollinger | 종가 1,2,3, period=3, k=2 | 중심 2, 상하단 `2 ± 2√(2/3)` | 없음 |
| Stochastic | 종가10·고가11·저가9 20개, 별도로 고저가 동일 | 마지막 K=D=50, 범위0은 null | 없음 |
| 빈 입력 | 각 계산 함수에 빈 배열 | 빈 결과 | 없음 |
| 재무 annual fallback | 종가20·주식수10·매출1000·순익100·자본500 | 시총200, PER2, PBR0.4, ROE20%, 영업이익률15%, 연간 fallback warning | LLM mock, 미호출 확인 |
| 지표 실패 | EMA 예외 주입 | 종가 metrics 보존, 지표 초기값·warning | EMA만 mock |
| 투자 수준 분리 | 동일 24개 가격·4개 투자 수준 | as_of를 제외한 metrics 동일 | 설명 생략 |
| `test_feature2.py`: 수익률·상관 | 100→110→99, 선형·역선형·상수 배열 | ln1.1·ln0.9, ±1, 표본부족·상수상관 NaN, 비양수 가격 거부 | 없음 |
| 감성 점수 | 각 확률 극값 및 (0.2,0.3,0.5) | -1,0,1 및 0.21 | 모델 추론 없음 |
| `test_feature3.py`: 최소분산 | 동일 독립 분산0.04 세 자산 | 각각 1/3 | 실제 SciPy |
| 효용 최적해 | 기대수익0.07·무위험0.02·γ5·분산0.04·현금상한0.5 | 각 위험자산0.25, 현금0.25. 일계조건으로 도출 | 실제 SciPy |
| 효용 비교·제약 | 서로 다른 세 수익·분산, γ=1/5/10 × 현금상한0/0.2/0.8 | 효용최적 ≥ 위험배분, 비중합1, 비음수, 종목·현금 상한 | 실제 SciPy, 9개 subcase |
| 거래일 교집합 | 한 자산1·2·3일, 다른 자산1·3일 | 1·3일만 사용, 보간 없음 | 없음 |
| CAPM·상한 | 표본59/60/119/120/252, 종목수1/2/3/4/5/10 | confidence 경계·적응형 상한, KOSPI/KOSDAQ·proxy 경고 | 없음 |
| 가격 어댑터 | camelCase 가격 envelope, 연결 실패 | 분석용 데이터 변환, UNAVAILABLE·warning | httpx Client mock |
| `test_llm_failures.py` | 잘못된 JSON, timeout 경고 | 최초+compact 재시도=2, metrics 보존, explain=null, warning 중복 제거 | 제공자 AsyncMock |

초기 테스트에서 감성 점수를 단순 확률 차이로, 3종목 상한을 고정40%로 가정한 부분은 코드 확인 후 수정했습니다. 현재 코드의 정의는 각각 중립확률 할인과 `2/3` 상한입니다. 제품 버그를 수정하거나 실패를 숨긴 것이 아니며, 정책 타당성 검증과 현재 구현의 회귀 검증을 구분합니다.

## 결과

- 별도 Peer 역방향 TCP: 실제 WSL uvicorn→Windows PeerClusterDataController/Service→합성Repository→실제Python계산의8개 중7 PASS·최신일누락1 FAIL. Windows host lifecycle1개PASS, 실제 source연결6회·금지연결0·유료0. 기본Python98개와별도이며 [방법·결과](../../backend/src/test/peer-reverse-tcp.md)에 기록했습니다.

- Windows Spring 클라이언트→WSL 실제 uvicorn의 별도 opt-in11개 **PASS**: F1 계산/DTO·F2 빈키 경고·F3 실제 수학/Public Controller 경유 보고서·환불 경계·실제 로컬 감성 모델까지 연결했습니다. `cross_os_tcp.py`와 `tcp_test_app.py`가 임시 포트/토큰·외부송신/.env 읽기 차단·XML/audit 보관·서버 종료를 담당합니다. 이11개는 Python unittest98개에 포함되지 않는 Java 검사입니다. [명령·11개 입력/기대값·실패 이력](../../backend/src/test/cross-os-fastapi.md).
- 최신 기본 unittest **98개:92 PASS·3개 메서드 FAIL·3 SKIP**(기존81+Feature2설명17, 2026-09-25 승인된WSL, 7.490초·exit1). unittest 원문은 `failures=5, skipped=3`: 실패 메서드3개 중2개는 제공자2종 subTest가 각각 실패해 failure 항목이5개입니다. subTest를 별도 테스트 수로 합산하지 않습니다. 모델3개는 기본suite에서 SKIP이며 별도 opt-in 실행은 모두 PASS했습니다. 이번 재실행은 설치 라이브러리의 올바른 `PYTHON_DOTENV_DISABLED=1`을 사용했습니다.
- 실제 WSL uvicorn smoke **10개 검사 PASS**: health, OpenAPI 경로 집합, 5개 POST의 빈 입력422, Feature1 설명 미호출·payload 응답, 제공 데이터 기반 Feature3 정상 계산·고급 후보/frontier 응답.

기존 `DOTENV_DISABLED`는 설치된 python-dotenv의 비활성화 변수가 아니었습니다. 테스트 패키지/명령을 `PYTHON_DOTENV_DISABLED`로 수정했습니다. TCP 중간 실행에서 .env 재로딩 후 발생한 연결 시도2회는 audit hook이 전송 전에 모두 차단했고 실제 유료 HTTP 요청은0회였습니다. 최종 TCP11개와 보강한 WSL smoke10개에서는 외부접속·dotenv읽기 시도가 모두0임을 확인했습니다. 이력은 [QA 발견사항](../../docs/findings.md)을 참조합니다.
- FastAPI 내부 경로6개 등록, Feature3 합성 시계열 전체 계산, 아래 Feature2 Peer의ASGI→실제Spring어댑터→합성pack→계산 응답을 확인했습니다. Feature2 전체 기능·실제Spring/모델/제공자 연동 PASS로 해석하지 않습니다.

## CAPM 독립 수학 검증

추가 CAPM 독립 검증(`test_capm.py`, 4개 PASS): 시장 로그수익률에 `자산수익률=2×시장수익률+0.0001`을 적용하여 가격을 생성했습니다. beta2·일 alpha0.0001·연 alpha0.0252·R²1·CAPM 수익0.10·120표본 가중치0.65·혼합수익0.093을 확인했습니다. 표본59/60/119의 제외/부분적용, 벤치마크 부재 시 과거수익 보존, confidence에 optimizer 변동성이 아닌 실현 변동성이 쓰이는 것도 검증했습니다. 실제 시장 데이터 회귀는 아닙니다. 전체 요청에서 optimizer 전달은 아래 별도 테스트로 확인했습니다.

추가로 현재 작업 트리의 Feature3 최적화 내부 함수를 합성 공분산 240개와 독립 경계해 열거 80개로 검증해 실패 0개를 확인했습니다. 이 별도 표준입력 실행은 위 unittest 98개에 포함되지 않으며 제품·테스트 코드를 바꾸지 않았습니다. [입력 범위·독립 기대값·재현 명령](optimizer-independent-check.md)을 참조하세요.

## Feature3 요청부터 응답까지

`portfolio_fixture.py`는 주말을 제외한141개 날짜(수익률140개), 사인/코사인으로 만든 서로 다른 두 시장 및 세 자산 로그수익률을 제공합니다. 실제 거래소 휴장일 달력이나 실제 종목이 아닙니다. 지수 누적으로 가격을 만들고 가격·benchmark 전체를 `input_data`에 넣습니다. KOSPI2종목/KOSDAQ1종목, 평가금액200/450/320·현금30, γ5·무위험수익률0.02·현금상한0.2·표본공분산·연율252를 사용합니다.

`test_portfolio_http.py` **9 PASS**: 실제 ASGI 애플리케이션→Pydantic→가격 변환→공통거래일→CAPM/혼합 기대수익률→SciPy→설명→JSON 직렬화를 실행합니다. 가격/benchmark fetch와 유료 설명 함수는 호출 시 실패하도록 차단하고 미호출도 확인합니다. ASGITransport는 TCP·서버 시작 이벤트를 실행하지 않으며 실제 uvicorn 검증과 구분합니다.

| 테스트 | 방법·독립 기대값 |
|---|---|
| 평가·공분산 | 원래 생성한 수익률로 `np.cov(ddof=1)×252`, `sqrt(wᵀΣw)`를 별도 계산하여 응답 변동성과 오차1e-6 이내 비교. 비중0.2/0.45/0.32/0.03·141가격/140수익 표본·기본 설명 확인 |
| 혼합수익 전달 | 실제 함수 실행을 유지하는 spy로 기본 target·max Sharpe·risk allocation·utility·frontier 입력 벡터를 관찰. 응답 자산별 blended 값과 오차6e-7 이내 일치, historical/CAPM 가중합 일치, 시장별 benchmark 매핑 확인 |
| 고급 제약 | MIN_VOL/MAX_SHARPE/RISK_ALLOCATION/UTILITY_OPTIMAL 비중합1·비음수·위험자산상한2/3·현금상한0.2. 효용최적≥위험배분. frontier 변동성 오름차순·기대수익 엄격증가 |
| overlay 분리 | fundamentals score0.8 추가 전후 current/advanced 완전 동일. QUALITY_TILT 생성, 단일 변화≤0.07·반회전율≤0.15 |
| benchmark 부분장애 | KOSDAQ benchmark만 unavailable·빈값 → KOSPI CAPM 유지, 해당 자산 CAPM 가중치0·historical 유지, 전체 분석 성공 |
| 요청 검증 | γ11·quantity0 →422, 분석 함수 미호출 |
| 비양수 가격 | QA_A의 한 가격0 → 해당 종목만 INVALID_PRICE_SERIES 제외, 남은 비중0.5625/0.4/0.0375 |
| 투자 수준 | 4단계별 같은 요청 → policy/summary/current/basic/advanced/overlay 동일 |
| 예외 공개 | 계산에 합성 내부 RuntimeError →500·언어별 정형 FEATURE3_ANALYZE_FAILED, 내부 문구는 HTTP 응답에 없음 |

별도 `uvicorn_smoke.py`는 같은 fixture를 실제 WSL uvicorn에 POST하고 정상 current·140표본·설명·고급4후보·frontier를 확인했습니다. 서버는 임시 loopback port를 사용하고 종료합니다. subprocess에서 provider fetch 함수를 mock하지는 않지만 외부 키를 제거하고 가격 fetch를 끄며 필요한 모든 가격/benchmark를 제공합니다. Spring 주소는 사용 불가 loopback으로 고정합니다.

## Feature3 LLM 실패 분리

`test_feature3_llm_failures.py` **4 PASS**(각 OpenAI/Gemini 두 subcase): 실제 포트폴리오 계산으로 만든 baseline을 복제하여 router에 제공하고, 실제 설명 prompt/provider/parser/fallback은 실행합니다. HTTP client만 mock하고 API key는 합성값 또는 빈값으로 대체합니다.

- timeout → LLM_EXPLAIN_TIMEOUT.
- HTTP429 → LLM_EXPLAIN_HTTP_429.
- 성공 HTTP의 잘못된 JSON → LLM_EXPLAIN_PARSE_FAILED.
- 키 없음 → LLM_API_KEY_MISSING, HTTP post0회.

모든 경우 current/advanced는 baseline과 동일, provider=DETERMINISTIC의 영문 설명과 경고1개를 확인했습니다. 키 누락 외에는 mock post1회이며 현재 구현은 이 경로에서 재시도하지 않습니다. 실제 유료 호출0회. 정상 provider JSON의 모든 필드·설명 품질은 아직 미검증입니다.

## Feature2 Peer: 실제 라우터·Spring 어댑터·수학·응답

`peer_fixture.py`와 `test_peer_cluster_http.py`를 추가했습니다. 신규 **17 PASS**(ASGI14·독립수학3), 당시 전체57 PASS. FastAPI app의 실제 `/feature2/peer-cluster` 동기 라우터→Pydantic→`compute_peer_cluster_v1`→실제 `SpringMarketDataProvider`→수익률·상관·산업보정·시차·후보/차트→JSON을 실행합니다. httpx의 Spring 요청 전송만 MockTransport로 대체하여 method/URL/body를 캡처합니다. `ASGITransport`는 실제 TCP 서버나 startup/lifespan을 실행하지 않습니다. 뉴스 모델 warm-up과 dotenv를 끄고 설명 함수는 호출하면 실패하게 막았으며 매 검사 종료 시 호출0을 확인합니다.

fixture는 seed20260925, 자기회귀계수0.85와 정규잡음으로 만든90개 수익률·91개 연속 UTC날짜입니다. 실제 주말/거래소 휴장일을 반영한 시장자료가 아닙니다. 산업 수익률은 `0.001*sin(0.31*t)+0.0002`, anchor는산업+잔차, SAME은산업+1.1×잔차, FOLLOW/LEAD는잔차를각각2칸뒤/앞으로이동, NEG는산업-잔차입니다. 가격은100에서로그수익률누적의지수로생성합니다. 종목별일정거래대금1000/1100/1200/1300/1400과거래량10/11/12/13/14를주입합니다.

기대 상관은 제품 `_corr`를 호출하지 않고 원래 생성한 수익률의 평균중심화·내적/제곱합으로 계산합니다. 산업보정 기대값은 각 수익률에서 동일 산업수익률을 차감하여 구합니다. 차트는 fixture가격을 직접첫가격으로나누고, 평균과분위수를별도계산하여 비교합니다. 이 검사는 알려진 입력의 구현 정확성을 검증하며 실제 시장 예측력·정책 타당성을 보증하지 않습니다.

| 메서드(`test_` 생략) | 수행·독립 기대값 | 결과 |
|---|---|---|
| `full_route_raw_and_industry_adjusted_correlations_match_independent_returns` | raw후보4→양의동행3선택, raw/보정Pearson 오차1e-10이내, sample90/coverage1, SIMPLE_SUBTRACTION·SELECTED·원천warning보존·시총필드없음 | PASS |
| `lag_direction_and_scores_match_constructed_leader_follower` | SAME lag0/COINCIDENT, FOLLOW+2/FOLLOWER, LEAD-2/LEADER, lag상관1. 출력성분의0.4/0.2/0.15/0.15/0.1가중합 및순위내림차순·score별칭일치 | PASS |
| `chart_rebase_centroid_quantiles_coverage_and_legacy_aliases` | 선택3종목가격에서 rebase/평균/20·80분위수 독립계산오차1e-12, 91일coverage1, centroid/peer_centroid·band/peer_band동일, t/value키 | PASS |
| `peer_missing_tail_preserves_anchor_dates_and_reduces_coverage` | SAME마지막10일제거→anchor/centroid91일유지·마지막coverage2/3, 마지막평균은다른2종목으로계산·채움없음 | PASS |
| `missing_industry_index_falls_back_to_raw_with_warning` | 지수없음→3peer유지·RAW_ONLY·adjustment_valid=false·INDUSTRY_INDEX_MISSING·fallbackwarning1개 | PASS |
| `short_industry_history_limits_chart_but_not_raw_peer_fallback` | 지수최근20일만→peer3유지·FALLBACK_RAW/INSUFFICIENT_COMMON_DATES, 차트20일·chartwarning | PASS |
| `bad_anchor_and_empty_members_have_payload_only_diagnostics` | 빈members/anchor가격없음→각warning·빈peers·raw후보수4 | PASS |
| `upstream_http_parse_and_transport_failures_return_empty_diagnostics` | source503/잘못된JSON/연결예외→HTTP200빈payload+SPRING_PACK_FETCH_FAILED·예외클래스, private문구미포함 | PASS |
| `pydantic_rejects_ranges_unknown_keys_and_camel_case_before_source` | 산업0/window29·366/peer2/lag0/display101/topK4/미지필드/freq오류/camel입력→422·source호출0 | PASS |
| `weekly_request_falls_back_to_daily_and_forwards_exact_dates` | ONE_W→ONE_D+warning, 실제어댑터POST1회·snake필드/from/to의+09:00유지 | PASS |
| `liquidity_thresholds_can_exclude_all_candidates` | 거래대금/거래량최소값100000 각각→evaluated0·빈peer·관련warning | PASS |
| `display_limit_does_not_reduce_selected_peer_count` | display_limit1→candidates1이지만선택peer3유지·첫순위일치 | PASS |
| `nonpositive_peer_price_is_excluded_without_failing_other_peers` | SAME중간가격0→SAME제외·LEAD/FOLLOW유지·부족warning | PASS |
| `unexpected_calculation_failure_has_sanitized_500` | 계산함수에합성RuntimeError→500·공통envelope코드PEER_CLUSTER_FAILED·private문구없음, source0 | PASS |
| `minimum_common_return_boundary_and_timezone_alignment` | 가격30개→수익률29개부족/가격31개→수익률30개사용, UTC와동일instant+09:00일치 | PASS |
| `symmetric_similarity_and_weighted_score_have_independent_expectations` | 동일유동성1·2배/절반각0.5·0값None, 고정성분가중합0.445·모두없음0 | PASS |
| `stability_and_lag_confidence_sample_penalty` | 동일수익률3분할상관1/안정성1·29개불충분, lag차이0.4·안정성0.8에서30표본0.16/60표본0.32 | PASS |

신규분만 실행하려면 위 unittest 명령에 `-p test_peer_cluster_http.py`를 추가합니다. 이 세션의 sandbox 실행에서는 첫 sync ASGI 요청이 멈췄습니다. 15초후 faulthandler는 AnyIO worker의queue대기와 주이벤트루프select대기를 보였고, 해당테스트프로세스를중단(exit130)했습니다. 승인된 sandbox밖 WSL에서 동일단일검사가2.029초에통과했고, 신규17개3.256초·전체57개7.051초에통과했습니다. 원인전체가확정된것은아니므로 제품계산실패로기록하지않습니다. 표기시간은해당실행소요이며성능SLA/벤치마크가아닙니다. 테스트환경이같은증상일때제품라우터를async로변경하지말고 승인된동일실행환경을사용합니다.

초기작성중500응답기대는라우터내HTTPException.detail이아니라`app.main`의정형코드로확인하도록맞췄습니다. 제품예외처리를바꾼것이아닙니다. 기존on_event deprecation경고는서비스수정금지범위라유지합니다.

Spring의 `PeerDataFlowTest`·`PeerClientCacheFlowTest`는별도Java경계를검증하고이테스트는실제Python어댑터/계산을검증합니다. 이17개 자체는TCP가아닙니다. 별도로[역방향TCP8개](../../backend/src/test/peer-reverse-tcp.md)를실행했으며7 PASS·1 FAIL입니다. 실제DB를공유한E2E완료로확대하지않습니다.

## 뉴스 감성: 서비스·로더·ASGI 오류 및 계약

`news_fixture.py`, `test_news_sentiment_flow.py`: **21 PASS**(서비스12·로더3·ASGI/기동6, 단독2.587초). 실제 `/feature2/news-sentiment`→Pydantic→`asyncio.to_thread`→서비스 배치/점수→응답 모델을 실행합니다. tokenizer/model/Torch는 해당 경계에서 합성 객체로 대체하며, softmax는 Python `math.exp`로 구현합니다. 확률0.2/0.3/0.5에 대응하는 로그값을 logits로 주어 점수0.21과 positive를 별도 기대값으로 검사합니다. 실제 모델 정확도 검사가 아닙니다. 입력 URL은 `.invalid` 합성 주소이며 기사를 내려받지 않습니다.

| 메서드(`test_` 생략) | 방법·입력·기대값 |
|---|---|
| `batches_preserve_order_probabilities_score_and_version` | 5기사·batch2→2/2/1, 순서/URL 유지, tokenizer padding·truncation·max64·pt 인자, 확률0.2/0.3/0.5·점수0.21·환경 버전, loader1회 |
| `invalid_items_are_filtered_and_warning_deduplicated` | 빈URL/공백focus/빈focus3개 제거, 나머지2개 순서 유지·INVALID_ITEM 경고1개 |
| `empty_or_all_invalid_does_not_load_model` | 빈 배열·모두 공백focus→빈 결과, loader0회 |
| `model_missing_and_load_error_have_sanitized_codes` | FileNotFoundError/RuntimeError→MODEL_NOT_FOUND/LOAD_FAILED:RuntimeError, 응답에 내부 문구 없음·tokenizer0회 |
| `later_batch_failure_preserves_completed_batch` | 두 번째 모델 batch 실패→첫2개 결과 유지·INFERENCE_FAILED, 이후 중단; 동일logit은 첫 label negative·점수0 |
| `tokenizer_and_malformed_logits_failure_return_warning` | tokenizer 예외·2열logit 주입→빈 결과+INFERENCE_FAILED; tokenizer 실패 때 모델0회 |
| `focus_normalization_and_existing_spring_markers` | TITLE/FOCUS/DETAIL 포함 문자열의 공백을 정규화한 뒤 전체 내용을 FOCUS와DETAIL에 각각 삽입하는 현재 포맷 확인 |
| `request_model_does_not_override_environment_model` | 요청 model 값을 바꿔도 환경 모델 버전·결과 동일; 현재 동작 기록이며 동적 모델 선택을 보증하지 않음 |
| `env_integer_defaults_and_minimum` | 정수 설정 오류/빈값→기본8, 0/음수→1, 공백 포함3→3 |
| `windows_absolute_and_relative_paths` | Windows 드라이브 두 표기→/mnt/c, WSL절대경로 유지, 상대경로→analysis 기준 |
| `score_clamps_and_warning_order` | 범위 밖 합성확률의 score±1 clamp, 빈 경고 제거·중복 제거·첫 순서 유지 |
| `warmup_executes_small_inference_and_swallows_load_failure` | 지정 warmup 문장·max32·모델1회, load 오류는 로그 후 전파하지 않음 |
| `load_is_cpu_eval_local_only_and_cached` | 가짜 torch/transformers 모듈로 loader 자체 실행, local_files_only=True·CPU·eval·thread2·두 호출에실제load1회 |
| `missing_path_does_not_start_pretrained_load` | 존재하지 않는 경로→FileNotFoundError, pretrained0회·전역캐시 없음 |
| `failed_load_does_not_poison_cache_and_next_attempt_can_retry` | 첫 load 실패 후 전역3개 None, 다음 호출 성공·load2회 |
| `actual_async_service_response_fields_and_score` | 실제 ASGI와 async service→HTTP200, payload results/warnings·결과8필드·점수0.21 |
| `schema_rejects_missing_extra_invalid_type_and_date_before_model` | 필수 누락/미지필드/null/숫자URL/잘못된날짜8조합→422, loader0회 |
| `empty_and_invalid_item_payloads_are_success_with_diagnostics` | items 생략·공백focus→HTTP200·빈결과 및 해당warning, loader0회 |
| `missing_model_is_warning_not_http_500` | 실제 async service의 모델누락→HTTP200+MODEL_NOT_FOUND, 내부경로 응답 제외 |
| `unhandled_service_error_has_sanitized_500` | async service에 합성예외→500·공통envelope NEWS_SENTIMENT_FAILED·내부문구 없음 |
| `warmup_schedules_once_and_startup_disable_flag` | 시작2회→task1회, startup disable1→미호출/0→호출; 실제백그라운드기동·동시성 검사는 아님 |

로더의 가짜 모듈·전역 상태·환경변수는 patch 종료 시 복원합니다. ASGITransport는 TCP/startup을 실행하지 않으며 마지막 검사의 startup 함수만 직접 호출합니다. 설명 함수는 호출 시 실패하도록 차단하고 미호출을 확인했습니다. 회귀 단독 실행:

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 PYTHONPATH=analysis \
  .venv_wsl/bin/python -B -m unittest discover -s analysis/tests -p test_news_sentiment_flow.py -v
```

### 별도 실제 로컬 모델 검증

`test_news_sentiment_local.py`는 `QAIMA_LOCAL_NEWS_MODEL=1`일 때만 실행합니다. 원본 `model.safetensors` 약710MiB와 tokenizer/config를 읽고, 학습용 pickle `training_args.bin`은 읽지 않습니다. 설치 버전은 Torch2.11.0·Transformers5.5.0·safetensors0.7.0·tokenizers0.22.2입니다. 모델 config의 architecture는 DebertaV2ForSequenceClassification, label은0 negative/1 neutral/2 positive입니다. 이 정보는 파일 존재와 메타데이터 확인이며 로딩 성공 증거와 구분합니다.

HF/Transformers offline·telemetry 비활성화·임시 HF_HOME을 설정하고 socket connect/connect_ex를 예외로 차단합니다. 새 캐시는 `/tmp`의 전용 임시 디렉터리이며 종료 시 해당 디렉터리만 정리합니다. 모델은 CPU·thread2·eval, 입력 max256·batch2로 고정합니다. 원본 모델·설정·DB·Redis를 변경하지 않습니다.

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 QAIMA_LOCAL_NEWS_MODEL=1 PYTHONPATH=analysis \
  .venv_wsl/bin/python -B -m unittest discover -s analysis/tests -p test_news_sentiment_local.py -v
```

검사3개: (1) 실제CPU/eval·label매핑·3개 합성한국어기사의 유한확률/합1·argmax label·독립 점수식·캐시재사용, (2) 긴 문장400회 반복의256토큰제한·단건/배치 일치허용오차1e-5, (3) 실제ASGI→스레드서비스→로컬모델→HTTP200 전체응답·설명미호출. 긍정/부정 문구에 특정 정답 label을 강제하지 않으며 평가셋 정확도·금융예측력 검증이 아닙니다. **별도 실제모델3개 PASS**(2026-09-25 승인된WSL, 로딩포함145.107초·exit0). 기본 suite의3 SKIP과 별도 실행 결과를 구분합니다. 네트워크 connect/connect_ex 호출0을 확인했고 임시 캐시는 정상 정리했습니다. 이 시간은 최초 라이브러리/모델 로딩도 포함하므로 API 응답시간 SLA가 아닙니다.

합성 문장 관측값은 실적증가/신규계약→positive/0.7984368184, 정기주주총회→neutral/0.0000160742, 손실확대/계약해지→negative/-0.9868663869였습니다. 정답라벨이 있는 독립 평가셋이 아니므로 정확도100% 등으로 해석하지 않습니다. Torch JIT와 FastAPI on_event의 기존 deprecation 경고는 남았지만 테스트 실패는 없었고 서비스·라이브러리를 수정하지 않았습니다. 모델 로딩 중 출력이 없어 프로세스를 확인했을 때 CPU사용12초·RSS약521MiB·p9_client_rpc 대기였고, 같은 프로세스를 재시작하지 않고 최종 exit0까지 관측했습니다.

## Feature2 설명: 실제 ASGI·제공자 client·prompt·parser

`test_feature2_explain_http.py` **17개:14 PASS·3개 메서드 FAIL**(단독2.656초·exit1). FastAPI `/feature2/analysis`→요청변환→실제 factory/OpenAIClient·GeminiClient→prompt→재시도→parser/용어치환→JSON을 실행합니다. HTTP 전송은 실제 httpx client에 MockTransport를 연결하여 대체하고 재시도 sleep만 AsyncMock으로 없앴습니다. 실제 키 대신 합성값을 주입했고 실제 네트워크·유료 호출0회입니다. ASGITransport이므로 TCP/startup은 실행하지 않습니다.

fixture는 QA0001, ONE_D/window90, 중급자, 기준금리2.75, anchor수익0.03·peer상관0.8, 뉴스점수0.21·3건입니다. 응답은6섹션과 overall을 갖는 합성 JSON입니다. 입력/응답의 숫자는 계산 정책의 예측력 검사가 아니라 변환·전달 보존 확인용입니다.

| 메서드(`test_` 생략) | 수행·기대/관측 | 결과 |
|---|---|---|
| `both_providers_success_mapping_prompt_and_localized_sections` | 2제공자×ko/en-US,6섹션·3bullets/2risks·용어치환·원본text보존, 요청불변·prompt수치/옵션·뉴스snake별칭, strict schema/MIME·토큰4000·전송1회 | PASS |
| `rate_limit_combines_three_provider_attempts_with_compact_retry` | 지속429→제공자당전송6회(4000×3+1800×3), sleep0.6/1.2 두세트, 경고1개·explain=null | PASS: 현재 동작 관측 |
| `timeout_two_attempts_null_explain_and_one_warning` | 두제공자 timeout→4000/1800 각1회, explain=null·TIMEOUT1개 | PASS |
| `invalid_json_is_not_compact_retried` | JSON아님/깨진JSON/sections배열×두제공자→PARSE_FAILED·null·전송1회 | PASS |
| `empty_content_returns_null_without_retry` | 공백응답→EMPTY·null·전송1회 | PASS |
| `missing_keys_prevents_any_provider_request` | 두키빈값→API_KEY_MISSING·HTTP0회 | PASS |
| `server_error_retry_policy_differs_by_provider` | 지속503→OpenAI3회/Gemini1회, HTTP_503경고·null | PASS: 오류본문은 아래 별도 실패 |
| `compact_recovery_keeps_initial_warning_and_uses_same_prompt` | OpenAI max_output_tokens 이후성공→설명복구·초기warning유지·4000→1800, prompt문자열동일 | PASS |
| `openai_refusal_content_filter_and_unknown_incomplete_do_not_retry` | refusal/content_filter/다른incomplete 각각 정형warning·null·전송1회 | PASS |
| `fenced_json_and_sparse_object_current_parser_behavior` | fence/주변텍스트 JSON 파싱, `{}`도 warning없이6개빈섹션·summary `-`로수용 | PASS: 현재 관대한 동작 기록 |
| `invalid_contract_rejected_before_provider` | 필수누락/미지키/subject누락/잘못된freq/metrics미지키5조합→422·전송0회 | PASS |
| `unexpected_factory_failure_returns_sanitized_500` | factory RuntimeError→500·FEATURE2_ANALYZE_FAILED·내부문구없음·전송0회 | PASS |
| `false_include_flag_must_not_call_paid_provider` | include_llm_explain=false인데 전송1회, 기대0 | FAIL: F2-LLM-FLAG-001 |
| `provider_error_body_must_not_escape_into_http_warnings` | 두제공자401본문 합성private문구가 응답warnings에 남음 | FAIL: F2-LLM-ERROR-001, subcase2 |
| `connection_exception_detail_must_not_escape_into_http_warnings` | 두제공자ConnectError의 합성private문구가 응답warnings에 남음 | FAIL: 같은문제, subcase2 |
| `investment_levels_change_instructions_not_data` | 4단계별지시문 변경·prompt 입력JSON완전동일 | PASS |
| `news_summary_legacy_aliases_are_normalized_without_mutation` | recent/daily×camel/snake4조합, 값0도보존·원본불변·extra유지 | PASS |

재현은 기본 unittest 명령에 `-p test_feature2_explain_http.py`를 추가합니다. 실패3개는 expectedFailure/skip으로 숨기지 않으며 제품 수정은 하지 않았습니다. 오류의 현재 확정 범위는 내부 FastAPI HTTP이며 Spring Public·브라우저까지의 실제전파는 별도입니다. 기본 suite도 동일3개 메서드에서 실패했습니다.

429의6회는 **합성 전송만** 실행한 결과입니다. 사용자 허용 최대5회를 넘을 수 있으므로 실제유료 검증은 전송직전 영속카운터·잔여한도 차단 등 테스트용 안전장치 없이 이 경로를 실행하면 안 됩니다. compact는 현재 prompt를 축약하지 않고 출력토큰한도만 줄입니다. `{}` 수용, raw `explain.text`의 내부용어 유지도 관측했으나 모든 빈응답 정책·UI 표시 문제를 확정하지 않습니다. LLM 설명의 사실성·금지표현·실제모델 출력품질·실제요금·Spring metrics 전체조립과의 연동은 남았습니다.

## 미검증

전체 frontier의 독립 최적성·THEORETICAL_UTILITY·모든 overlay 종류/조합·Ledoit-Wolf 전체요청 비교·다양한 시계열 품질 경계, 뉴스 모델 평가셋 정확도·실제Spring입력/캐시 전체연동·대규모/동시추론, peer 실데이터/대규모/동률/분위수필터경계·모든legacy provider조합, LLM 실제출력품질/요금/모든응답형식·Spring 설명전체연동, 동시 요청·성능, Spring↔FastAPI 정상 TCP/DB 통합 흐름이 남아 있습니다. 현재 fixture의 통과를 모든 시장·제공처·옵션 조합의 완료로 확대하지 않습니다.
