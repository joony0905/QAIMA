# Feature 분석 기능 사용자 체감 흐름 정책

## 문서 목적

이 문서는 `analysis/app/api/feature*` FastAPI 엔드포인트와 이를 사용하는 Spring/Frontend 세부 엔드포인트를 사용자 체감 기능 관점에서 정리한다.

데이터 구조의 camelCase/snake_case 경계, 내부 요청 블록 구조, 공통 explain 구조는 `policy/Feature_data_structure_policy.md`를 따른다. 이 문서는 구조 정책을 반복하기보다 사용자가 화면에서 보는 기능, 호출 흐름, fallback/warning 체감 지점을 설명한다.

## 전체 경계

- Frontend는 Spring public API만 호출한다.
- Spring은 인증, 크레딧 차감/환불, DB/외부 데이터 조회, FastAPI 내부 요청 조립, public `ApiResponse` envelope 생성을 담당한다.
- FastAPI는 payload-only 응답을 반환하며, 분석 계산과 LLM 설명 생성의 내부 엔진 역할을 한다.
- Feature1/2/3의 공통 내부 분석 엔드포인트는 다음 세 개다.
  - `POST /feature1/analysis`
  - `POST /feature2/analysis`
  - `POST /feature3/analysis`
- Feature2는 내부 보조 엔드포인트도 가진다.
  - `POST /feature2/peer-cluster`
  - `POST /feature2/news-sentiment`

## Feature1: 종목 기본 분석

### 사용자 체감 기능

사용자는 종목 상세/분석 화면에서 종목, 주기, 기간을 선택하고 분석을 실행한다. 결과 화면에서는 가격 흐름 요약, 재무 타임라인, 시장 스냅샷, 기술적 보조지표, 선택 시 LLM 설명을 본다.

### 공개 호출

- Frontend: `frontend/src/api/analysis.ts`
- Spring public API: `POST /api/v1/feature1/analyze`
- FastAPI internal API: `POST /feature1/analysis`

### Spring 흐름

`FeatOneController`는 로그인 사용자를 확인하고 Feature1 크레딧을 먼저 차감한다. 이후 `FeatOneService.getFeatOneData`가 분석 입력을 준비한다.

`FeatOneService`는 요청 파라미터를 검증한 뒤 종목을 조회하거나 생성하고, OHLCV 캔들, 재무제표, 시장 스냅샷을 준비한다. 캔들은 먼저 DB에서 조회하고, 최신 거래일 데이터가 비어 있거나 오래됐으면 `StockClient`를 통해 외부 데이터를 가져와 누락분만 저장한 뒤 병합한다. 재무 데이터는 최근 12개 항목을 조회하고, 시장 스냅샷은 최신 DTO를 조회한다.

Spring은 이 데이터를 `requestContext`, `subject`, `inputData`, `options` 구조로 바꿔 FastAPI `/feature1/analysis`에 전달한다. FastAPI 호출 실패 시 Spring은 차트/재무/스냅샷 기반 fallback 응답을 만들고 warning을 붙인다. public 응답은 `ApiResponse.successWithWarnings`로 감싸진다.

분석 성공 후에는 Feature3 overlay 재사용을 위해 Feature1 metrics를 `Feature3OverlayService.cacheFeature1Metrics`로 캐시한다. 이 캐시는 포트폴리오 분석에서 종목별 품질/가치/성장 신호를 overlay로 사용할 때 체감된다.

분석 성공 응답이 만들어진 뒤 Spring은 로그인 사용자 기준 `analysis_report`에 리포트 스냅샷 저장을 시도한다. 저장 성공 시 public envelope `meta.reportId`가 내려가며, 저장 실패 시 분석 결과는 유지되고 `REPORT_SAVE_FAILED` warning만 추가된다.

### FastAPI 흐름

FastAPI `analysis/app/api/feature1.py`는 내부 요청을 기존 `Feature1Request`로 변환한 뒤 `analyze_stock`을 실행한다.

사용자에게 보이는 주요 결과는 다음 계산에서 나온다.

- `ohlcv_summary`: 입력 OHLCV 개수, 시작/종료 시각, 마지막 종가
- `financial_series`: 연도별 매출, 영업이익, 순이익
- `market_snapshot`: PER/PBR/PSR/시가총액, ROE/ROA/마진, 부채비율/유동비율/이자보상배율, 매출/EPS 성장률, FCF
- `indicators`: EMA 20/60/120, Bollinger Band 20/2, Stochastic 14/3/3
- `indicator_summary`: LLM과 화면 요약에 쓰기 쉬운 텍스트형 지표 요약

시장 스냅샷은 Spring에서 이미 제공되면 그대로 사용한다. 제공값에 null이 있으면 `MARKET_SNAPSHOT_*_MISSING` 및 `MARKET_SNAPSHOT_PARTIAL` warning이 붙는다. 제공되지 않으면 FastAPI가 OHLCV와 재무제표로 TTM 또는 연간 fallback 기반 스냅샷을 계산한다.

`includeExplain=true`일 때만 Feature1 LLM 설명을 호출한다. false이면 `LLM_EXPLAIN_SKIPPED` warning이 붙고, 사용자는 수치/차트 중심 결과만 본다.

### 사용자 체감 warning

- 캔들이 없거나 외부 조회가 실패하면 차트/지표가 비거나 `CHART_DATA_UNAVAILABLE`, 분석 API fallback warning이 나타날 수 있다.
- 시장 스냅샷 일부 지표가 없으면 관련 카드 일부 값이 비고 `MARKET_SNAPSHOT_PARTIAL` 계열 warning이 붙는다.
- 보조지표 계산 실패 시 `INDICATOR_CALC_FAILED`가 붙고 지표 차트/요약이 비어 보일 수 있다.
- LLM 미사용 또는 실패 시 설명 영역만 비거나 fallback 설명으로 대체된다.

## Feature2: 외부 요인 분석

### 사용자 체감 기능

사용자는 종목의 외부 환경을 확인한다. 화면에는 기준금리/환율/채권금리, 업종지수, 공매도, 관련 종목/피어 클러스터, 투자자 수급, 뉴스 감성, 그리고 이 신호들을 종합한 설명이 표시된다.

Feature2는 “종합 분석”과 “카드별 데이터 조회”가 나뉜다. 카드별 조회는 화면 초기 로딩이나 일부 패널 갱신에 쓰이고, 종합 분석은 여러 데이터를 모아 LLM 설명까지 생성한다.

### 공개 호출

종합 분석:

- Frontend: `frontend/src/api/feature2.ts`
- Spring public API: `POST /api/v1/feature2/analyze`
- FastAPI internal API: `POST /feature2/analysis`

카드/세부 조회:

- `GET /api/v1/feature2/cards/base-rate`
- `GET /api/v1/feature2/cards/base-rate-series?limit=...`
- `GET /api/v1/feature2/cards/macro-rates`
- `GET /api/v1/feature2/cards/macro-rates-series?limit=...`
- `GET /api/v1/feature2/cards/industry-index?stockCode=...&freq=...&window=...`
- `GET /api/v1/feature2/cards/short-selling?stockCode=...`
- `GET /api/v1/feature2/cards/short-selling-series?stockCode=...&limit=...`
- `GET /api/v1/feature2/cards/related-stocks?stockCode=...&limit=...`
- `GET /api/v1/feature2/cards/investor-flow?stockCode=...&limit=...`
- `GET /api/v1/feature2/news?stockCode=...`
- `GET /api/v1/feature2/news/{newsId}`

FastAPI 보조 엔드포인트:

- `POST /feature2/peer-cluster`
- `POST /feature2/news-sentiment`

FastAPI가 peer cluster 계산에 필요한 원천 데이터를 다시 Spring에서 가져오는 내부 데이터팩 엔드포인트:

- `POST /api/v1/feature2/peercluster/data`

### Spring 종합 분석 흐름

`Feature2AnalyzeController`는 로그인 사용자를 확인하고 Feature2 크레딧을 차감한다. `Feature2AnalyzeService.analyze`는 요청을 정규화한 뒤 종목을 resolve한다. 종목이 없으면 빈 metrics와 `STOCK_NOT_FOUND` 성격의 warning으로 성공 응답을 만든다.

종목이 확인되면 Spring이 다음 데이터를 순서대로 붙인다.

- 종목/거래소/산업 기본 정보
- 최신 기준금리
- 한국/미국 기준금리, USD/KRW, 한국/미국 채권금리
- 최신 공매도 지표
- 종목/시장 투자자 수급
- 업종지수 시계열
- peer cluster
- 기준금리/공매도 trend summary
- 뉴스 목록과 뉴스 감성 summary

이후 `Feature2ExplainMetricsAssembler`가 화면에 필요한 전체 metrics에서 LLM 설명에 필요한 축약 metrics를 만들고, Spring은 FastAPI `/feature2/analysis`에 설명 생성을 요청한다. LLM 설명이 실패해도 종합 metrics는 유지되고 `LLM_EXPLAIN_FAILED` warning만 붙는다.

컨트롤러 레벨에서 전체 실패가 발생하면 크레딧을 환불하고 빈 metrics와 `FEAT2_INTERNAL_ERROR` warning이 담긴 fallback 응답을 준다.

Feature2 종합 분석 성공 후에도 Feature1과 동일하게 `analysis_report`에 요청 payload, 결과 snapshot, warnings를 MySQL JSON으로 저장한다. 저장 성공 시 `meta.reportId`, 저장 실패 시 `REPORT_SAVE_FAILED` warning을 사용한다.

### Feature2 카드 흐름

`Feature2CardController`의 카드 엔드포인트는 대부분 인증 리다이렉트를 건너뛰는 공개성 조회로 프론트에서 호출된다. 사용자는 종합 분석 전에도 기준금리, 매크로 금리, 업종지수, 공매도, 관련주, 투자자 수급 패널을 볼 수 있다.

`Feature2CardService`는 필요한 경우 동기화 서비스를 호출해 최신 데이터를 보장하려고 시도한다. 예를 들어 매크로 금리는 한국 기준금리, 미국 Fed funds, USD/KRW, 한국/미국 채권금리를 각각 ensure/findLatest 흐름으로 가져온다. 일부 소스가 실패해도 나머지 데이터를 반환하고 warning을 붙이는 방향이다.

### Peer cluster 흐름

Spring `PeerClusterServiceImpl`은 캐시를 먼저 확인하고 필요하면 FastAPI `/feature2/peer-cluster`를 호출한다. FastAPI의 `compute_peer_cluster_v1`은 `SpringMarketDataProvider`를 통해 Spring `/api/v1/feature2/peercluster/data`에서 산업 구성 종목, 기준 종목, 메타, 가격 시계열을 받아 계산한다.

사용자가 보는 결과는 다음 성격이다.

- 기준 종목 relative series
- 업종지수 relative series
- peer centroid와 peer band
- peer coverage
- 선정 peer와 표시 candidate
- 동시 상관, 조정 상관, 선행/후행 관계, 유동성/변동성 유사도

계산 실패나 후보 부족은 화면에서 peer 영역이 비거나 warning으로 체감된다. Spring 정책상 peer cluster 실패는 전체 Feature2 실패로 만들지 않고 `PEER_CLUSTER_MISSING` 계열 warning으로 흡수한다.

### 뉴스 감성 흐름

뉴스 목록은 `NewsSentimentService`가 종목명/별칭 기반으로 Naver 뉴스와 DB/Redis 캐시를 조합해 만든다. 뉴스 상세는 본문 추출 캐시가 있으면 재사용하고, 없거나 신뢰도가 낮으면 fallback detail과 warning을 반환할 수 있다.

감성 점수는 Spring `Feature2NewsSentimentClient`가 FastAPI `/feature2/news-sentiment`에 뉴스 URL, 제목, 발행사, 발행시각, focus text, 모델명을 보내 얻는다. FastAPI는 로컬 뉴스 감성 모델로 score, label, negative/neutral/positive probability, model version을 반환한다.

Feature2 뉴스 리스트와 종합 분석의 뉴스 감성 산출 대상은 최근 관련 뉴스 최대 30개다. 외부요인 패널 뉴스 리스트, 분석 리포트의 뉴스 리스트, 감성 점수 산출 뉴스 수는 이 30개 범위에서 사용자가 체감한다. 단, LLM 설명 입력의 `recent_news`는 토큰/품질 관리를 위해 최대 5개로 제한한다. 이 5개는 감성 점수가 있는 뉴스를 우선 선택하고, 5개가 부족할 때만 감성 점수가 없는 뉴스로 부족분을 채운다.

뉴스 감성 summary의 화면 표현은 “최근 평균 점수”다. 이는 감성 점수가 산출된 최근 뉴스 전체의 평균이며, 특정 하루의 일평균으로 해석하지 않는다. LLM 입력 payload에서도 일일 평균으로 오인될 수 있는 `dailyAvgScore`, `dailyNewsCount` 키는 `recent_avg_score`, `recent_news_count` 의미로 정규화해 전달한다.

뉴스 DB 중복 방지는 URL 기준이다. `news.url`은 unique이고, 적재 시 `findByUrl`로 기존 뉴스가 있으면 재사용한다. 종목과 뉴스의 관계는 `news_security_map(news_id, stock_id)` unique로 중복 매핑을 막는다. 따라서 같은 URL 기사는 여러 종목에서 공유될 수 있으나, URL이 다른 재송고/제휴 기사는 별도 뉴스로 적재될 수 있다.

사용자는 뉴스 리스트에서 기사별 감성 점수와 전체 뉴스 감성 summary를 본다. 감성 모델 응답이 깨졌거나 점수가 없으면 해당 뉴스 감성은 제외되거나 중립/누락처럼 보이고 warning이 붙는다.

### FastAPI Feature2 `/analysis`

FastAPI `analysis/app/api/feature2.py`의 `/analysis`는 Feature2 전체 데이터를 다시 계산하지 않는다. Spring이 준비한 metrics를 받아 `analyze_feature2_explain`에 넘기고, 구조화된 explain과 warnings만 반환한다.

따라서 Feature2의 사용자 체감 계산 책임은 Spring의 데이터 조립과 FastAPI의 peer/news/LLM 보조 역할로 나뉜다.

## Feature3: 포트폴리오 위험/최적화 분석

### 사용자 체감 기능

사용자는 보유 종목, 수량, 평균단가, 현금, 투자성향을 입력하고 포트폴리오 분석을 실행한다. 결과 화면에서는 현재 포트폴리오 위험, 투자성향 대비 적합성, 안정/균형/공격형 비교 포트폴리오, 고급 모드의 공분산/효율적 프론티어/CAPM/SML/SCL, overlay 기반 조정 제안성 비교를 본다.

### 공개 호출

분석:

- Frontend: `frontend/src/api/portfolio.ts`
- Spring public API: `POST /api/v1/feature3/analysis`
- FastAPI internal API: `POST /feature3/analysis`

보조 조회:

- `POST /api/v1/feature3/overlay-cache/preview`
- `GET /api/v1/feature3/market-data/price-series?stockCode=...&requestedPriceBasis=...&lookbackTradingDays=...&fetchCalendarDays=...`
- `GET /api/v1/feature3/market-data/benchmark-series?benchmarkCode=...&lookbackTradingDays=...&fetchCalendarDays=...`

포트폴리오 저장/조회:

- `GET /api/v1/portfolios/me/default`
  - auth: required
  - purpose: 로그인 사용자의 기본 포트폴리오를 조회해 Feature3 Portfolio Manager와 국내 투자자산비율 입력 상태를 복원한다.
  - response data: `portfolioId`, `cashAmount`, `holdings[]`
- `PUT /api/v1/portfolios/me/default`
  - auth: required
  - purpose: 현재 Feature3 Portfolio Manager 상태를 사용자 기본 포트폴리오로 저장한다.
  - write policy: 기존 기본 포트폴리오 holdings를 부분 수정하지 않고 요청 payload 기준으로 통째 교체한다.
  - request body: `cashAmount`, `holdings[].stockCode`, `holdings[].stockName`, `holdings[].quantity`, `holdings[].averagePrice`

### Spring 분석 흐름

`Feature3AnalyzeController`는 로그인 사용자를 확인하고, `Feature3OverlayService.estimateCredit`으로 선택 overlay와 캐시 상태를 반영한 비용을 계산한 뒤 Feature3 크레딧을 차감한다.

Spring은 FastAPI 호출 전 다음 정규화를 수행한다.

- holdings의 `stockCode`가 직접 식별자가 아니면 `StockMappingService`로 종목명을 코드로 매핑한다.
- 각 holding의 거래소 코드를 조회해 benchmark 선택에 사용한다.
- risk-free rate를 `Feature3RiskFreeRateService.resolve`로 결정한다.
- 각 종목 가격 시계열을 `Feature3PriceSeriesService.getPriceSeries`로 준비한다.
- 거래소별 benchmark 시계열을 `Feature3BenchmarkSeriesService.getBenchmarkSeries`로 준비한다.
- 선택 overlay가 있으면 `Feature3OverlayService.loadOverlaySignals`로 Feature1/Feature2 기반 신호를 준비한다.

Feature3의 뉴스 overlay는 별도 뉴스 파이프라인을 갖지 않고 Feature2 `NewsSentimentService`를 재사용한다. 따라서 Feature2 뉴스 정책과 동일하게 종목별 최근 관련 뉴스 최대 30개를 로딩하고, 감성 점수가 산출된 뉴스 수와 평균 점수를 overlay 신호로 요약한다. 화면의 뉴스 관측 카드는 “최근 뉴스 최대 30개” 기준의 Feature2 news sentiment 참고 정보로 표시된다.

이 데이터는 `requestContext`, `portfolio`, `inputData`, `options` 구조로 FastAPI `/feature3/analysis`에 전달된다. FastAPI 응답은 Spring DTO로 변환된 뒤 `Feature3OverlayService.enrich`를 거쳐 overlay 카드, freshness, 캐시 상태 등 사용자 표시 정보를 보강한다.

FastAPI 분석 실패 시 Spring은 차감한 Feature3 크레딧을 환불하고 에러를 전달한다.

Feature3 분석 성공 후에는 포트폴리오 리포트 스냅샷을 `analysis_report`에 저장한다. 개별 종목이 아닌 포트폴리오 분석이므로 `subjectType=PORTFOLIO`, `portfolioSummary`에 “대표 종목 외 N개 종목” 형태의 요약을 저장한다. 가격 기준, 위험성향, 분석기간도 리포트 메타에 포함한다.

### Spring 포트폴리오 저장 흐름

`PortfolioController`는 로그인 사용자를 확인하고 `/api/v1/portfolios/me/default`에서 사용자별 기본 포트폴리오 1개를 조회 또는 교체 저장한다.

- 저장 대상은 종목코드, 종목명, 수량, 평균단가, 보유현금으로 제한한다.
- 저장은 분석 실행과 분리된 public API에서 수행한다.
- `PUT` 요청은 기존 holdings를 유지하거나 병합하지 않고 현재 요청의 holdings로 전체 교체한다.
- 이 데이터는 Feature3 화면 입력 상태 복원과 국내 투자자산비율 계산에 사용한다.
- FastAPI 내부 분석 엔드포인트로는 전달되지 않으며, 분석 호출 시 프론트가 현재 화면 상태를 별도 request로 보낸다.

### FastAPI 분석 흐름

FastAPI `analysis/app/api/feature3.py`는 내부 요청을 `PortfolioAnalyzeRequest`로 변환하고 `analyze_portfolio`를 실행한다.

사용자에게 보이는 핵심 산출물은 다음이다.

- `policy`: 가격 기준, 투자성향 echo, 데이터 품질, risk-free policy
- `summary`: 위험등급, 투자성향 적합성, 연환산 변동성, 목표 변동성, 주요 위험 기여 종목
- `current_portfolio`: 현재 비중, 평가금액, 손익, 변동성, 위험기여도
- `basic_portfolios`: 안정/균형/공격형 등 비교 포트폴리오
- `advanced`: 공분산 진단, correlation matrix, frontier, CAPM/SCL/SML, expected return policy
- `overlays`: Feature1/2 신호 기반 insight, 종목별 overlay table, 조정 포트폴리오
- `freshness`: 가격/overlay 데이터의 최신성 메시지
- `explain`: LLM 또는 deterministic 설명
- `warnings`: 데이터 품질, 가격 fallback, 계산/설명 관련 경고

`includeLlmExplain=true`이면 FastAPI가 Feature3 LLM 설명을 생성한다. LLM 결과에 text가 없으면 deterministic explain으로 대체하고, LLM warning은 응답 warnings에 합쳐진다. false이면 deterministic explain만 제공된다.

### 시장 데이터 보조 조회

Feature3 market-data 엔드포인트는 포트폴리오 분석 전후 진단 또는 화면 보조용이다. 가격 시계열은 요청한 가격 기준, 실제 사용 가격 기준, source, cache status, 누락률, fallback 여부, warning을 함께 반환한다. 벤치마크 시계열도 사용 가능 여부와 누락률을 포함한다.

사용자는 직접 이 내부 품질 필드를 보지 않더라도, 고급 모드의 데이터 품질 카드나 warning, excluded holding, freshness 메시지로 체감한다.

### Overlay cache preview

`POST /api/v1/feature3/overlay-cache/preview`는 실제 분석 전에 선택 overlay가 추가 크레딧을 요구하는지 보여주는 기능이다. Feature1/Feature2 캐시가 fresh이면 추가 비용이 낮거나 없고, stale/miss이면 사용자 확인이 필요한 메시지가 내려간다.

뉴스 overlay 캐시 미리보기는 Feature2 뉴스 캐시를 확인한다. `REUSE_AVAILABLE`이면 기존 뉴스 리스트/detail/focus/sentiment 캐시를 재사용하고, `FORCE_REFRESH`이면 Feature2 뉴스 로딩을 강제 갱신한다. Redis가 연결된 운영 환경에서는 뉴스 리스트 캐시는 짧은 TTL로, detail/focus/sentiment 캐시는 더 긴 TTL로 동작하며, 캐시는 빠른 조회용이고 DB의 `news`, `news_security_map`, `sentiment_result`가 재사용의 기준 데이터다.

## 리포트 이력/재다운로드

Spring public endpoint:

- `GET /api/v1/reports/me`
  - 목적: 로그인 사용자의 분석 리포트 이력 목록 조회
  - 인증: 필요
  - query: `featureType` optional (`FEATURE1`, `FEATURE2`, `FEATURE3`), `page`, `size`
  - 응답: `ApiResponse<List<AnalysisReportSummaryDto>>`
- `GET /api/v1/reports/{reportId}`
  - 목적: 내 리포트 상세 snapshot 조회 및 프론트 재다운로드 렌더링
  - 인증: 필요
  - 권한: 본인 리포트만 조회 가능
  - 실패: 없거나 본인 소유가 아니면 `RESOURCE_NOT_FOUND`

프론트 내정보 화면은 이 목록을 “내 리포트” 섹션에 표시하고, 상세 보기와 PDF 재다운로드를 제공한다. 재다운로드는 DB에 PDF 바이너리를 저장하지 않고, 저장된 JSON snapshot을 프론트에서 다시 렌더링한 뒤 기존 PDF 캡처 유틸로 생성한다.

리포트 UI/PDF 정책:

- Feature1/Feature2 리포트 상단에는 분석종목, 분석일시, 분석모델, 사용자 이름, 투자 레벨, 분석기간, 데이터 기준일을 표시한다. 사용자 이름은 내정보의 이름을 사용하고 없으면 `사용자`로 대체한다.
- Feature3 리포트는 개별 종목 분석이 아니므로 분석대상은 `포트폴리오`, 상세는 “대표 종목 외 N개 종목” 형태로 표시한다. 추가로 위험성향, 가격 기준, 공분산 모형, 분석기간, 데이터 기준일을 리포트 메타에 포함한다.
- 확대 보기는 현재 스크롤 위치와 무관하게 사용자 viewport 중앙 기준으로 떠야 한다. 부모 요소의 `transform`/page transition 때문에 `position: fixed` 좌표가 틀어질 수 있으므로, modal/overlay는 stacking context를 명확히 분리하거나 확대 중 상위 transform을 제거한다.
- PDF 다운로드/재다운로드 시 화면용 스크롤 등장 애니메이션 때문에 섹션이 `opacity: 0` 상태로 캡처되면 안 된다. PDF 캡처 모드에서는 리포트 섹션을 강제로 visible 상태로 만들고, `data-pdf-exclude` 요소는 캡처에서 제외한다.
- 화면에서 별도 스크롤 컨테이너로 제한한 긴 섹션은 PDF 캡처 중 `overflow-visible` 형태로 전체 내용을 펼친다. Feature2의 유사종목/뉴스, Feature3의 긴 테이블/차트성 섹션이 여기에 해당한다.
- 화면에서 탭으로 표현되는 차트 묶음은 PDF에서는 선택된 탭 하나만 캡처하지 않고 전체를 세로로 나열한다. Feature2의 환율/기준금리/국내국채/미국국채/공매도, Feature3 최적화 관측의 SCL 개별종목 차트가 여기에 해당한다.
- CSS `conic-gradient` 기반 파이차트는 html2canvas 계열 PDF 캡처에서 누락될 수 있으므로, Feature3 PDF 캡처 모드에서는 SVG 파이차트로 대체 렌더링한다.
- PDF 파일 자체는 현재 QA/가배포 단계에서 장기 보관 대상이 아니다. 리포트 이력은 JSON snapshot을 DB에 저장하고, PDF는 다운로드 시점에 재생성한다. 단일 EC2 운영 단계에서는 필요 시 로컬/EBS 임시 저장을 우선하며, S3 전환은 장기 파일 보관이나 다중 서버 운영 필요가 생긴 뒤 검토한다.

내정보 UI 정책:

- 내정보에는 사용자 이름, 투자성향, 기본 포트폴리오 상태, 내 리포트 목록을 같은 사용자 소유 데이터로 취급해 표시한다.
- 내 리포트 목록은 Feature1/2/3 구분, 분석대상/포트폴리오 요약, 생성일시, 모델, warning 여부, 재다운로드 액션을 제공한다.
- 리포트 상세/재다운로드는 본인 리포트만 허용한다. API가 `RESOURCE_NOT_FOUND`를 반환하면 프론트는 권한 없음과 미존재를 구분해 노출하지 않고 “리포트를 찾을 수 없습니다” 수준으로 처리한다.

Overlay cache preview 기능은 분석 결과 자체를 만들지 않고, 사용자가 포트폴리오 분석 실행 전에 비용과 캐시 상태를 판단하게 한다.

## 사용자 체감 기능별 책임 요약

| 기능 | 사용자가 보는 것 | Spring 책임 | FastAPI 책임 |
| --- | --- | --- | --- |
| Feature1 종목 분석 | 가격/재무/스냅샷/지표/설명 | 인증, 크레딧, 종목/캔들/재무/스냅샷 조회, 외부 캔들 fallback, 응답 envelope, Feature3용 metrics 캐시 | 보조지표 계산, 스냅샷 보완 계산, Feature1 LLM 설명 |
| Feature2 종합 분석 | 매크로/수급/공매도/업종/피어/뉴스/종합 설명 | 인증, 크레딧, 모든 외부요인 metrics 조립, 카드 데이터 재사용, warning 흡수 | peer cluster 계산, 뉴스 감성 모델, Feature2 LLM 설명 |
| Feature2 카드 | 각 패널의 독립 데이터 | DB/동기화 서비스/캐시 기반 조회, 부분 실패 warning | 직접 관여 없음 |
| Feature3 포트폴리오 분석 | 위험도, 적합성, 비교 포트폴리오, 고급 효율성, overlay | 인증, 동적 크레딧, 종목 매핑, 가격/벤치마크/risk-free/overlay 입력 준비, enrich | 포트폴리오 수학, 공분산/CAPM/최적화, LLM 또는 deterministic 설명 |
| Feature3 market-data | 가격/벤치마크 품질 진단 | 가격/벤치마크 시계열 제공과 warning 정규화 | 직접 관여 없음 |

## 실패와 fallback 원칙

- Feature1은 FastAPI 분석 실패 시 Spring fallback 응답을 만들어 사용자가 최소한 차트/재무 기반 결과를 보게 한다.
- Feature2는 개별 데이터 소스 실패를 전체 실패로 만들지 않고 warning과 부분 metrics로 흡수한다. 전체 실패 시에도 빈 metrics fallback을 반환한다.
- Feature3는 포트폴리오 핵심 분석 실패 시 계산 결과를 만들기 어렵기 때문에 크레딧 환불 후 에러로 처리한다.
- LLM 설명 실패는 세 기능 모두 핵심 수치 계산 실패와 분리한다. 사용자는 설명이 비거나 deterministic/fallback 설명을 볼 수 있지만, 가능한 한 수치 결과는 유지된다.
- public API warning은 사용자가 “데이터가 일부 비어 있음”, “캐시/외부 소스 문제”, “설명 생성 실패”를 이해하는 데 쓰이는 UX 신호다.

## 관련 코드 위치

- FastAPI 라우터: `analysis/app/api/feature1.py`, `analysis/app/api/feature2.py`, `analysis/app/api/feature3.py`
- FastAPI 모델: `analysis/app/models/feature1.py`, `analysis/app/models/feature2.py`, `analysis/app/models/feature3.py`
- Spring FastAPI 클라이언트: `backend/src/main/java/com/qaima/external/FastApiAnalysisClient.java`
- Feature1 Spring: `backend/src/main/java/com/qaima/api/feat1/FeatOneController.java`, `backend/src/main/java/com/qaima/service/featone/FeatOneService.java`
- Feature2 Spring: `backend/src/main/java/com/qaima/api/feat2`, `backend/src/main/java/com/qaima/service/feature2`
- Feature3 Spring: `backend/src/main/java/com/qaima/api/feat3`, `backend/src/main/java/com/qaima/service/feature3`
- Frontend API: `frontend/src/api/analysis.ts`, `frontend/src/api/feature2.ts`, `frontend/src/api/portfolio.ts`
