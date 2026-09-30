# 기능별 데이터 구조 정책

## 경계 규칙

- Spring, Java DTO, 프론트엔드 TypeScript 타입은 `camelCase`를 사용한다.
- FastAPI, Pydantic 모델, 분석 내부 JSON은 `snake_case`를 사용한다.
- Spring에서 FastAPI로 요청을 보낼 때는 경계에서 snake-case `ObjectMapper`로 직렬화한다.
- FastAPI는 payload-only 응답을 반환한다. public `ApiResponse` envelope는 Spring이 담당한다.
- 기존 public API route는 명시적인 breaking change 승인이 없는 한 호환성을 유지한다.

## 공통 내부 요청 구조

Spring에서 FastAPI로 전달하는 내부 요청은 다음 논리 블록을 기준으로 구성한다.

```json
{
  "request_context": {},
  "subject": {},
  "input_data": {},
  "options": {}
}
```

현재 Spring에서 사용하는 FastAPI 분석 엔드포인트는 다음 세 개만 유효하다.

- Feature1: `POST /feature1/analysis`
- Feature2: `POST /feature2/analysis`
- Feature3: `POST /feature3/analysis`

동일한 구조를 Spring DTO에서는 다음 필드명으로 표현한다.

```json
{
  "requestContext": {},
  "subject": {},
  "inputData": {},
  "options": {}
}
```

### request_context

요청 정책과 실행 옵션의 성격을 가진 값만 담는다. 원천 데이터는 넣지 않는다.

- `feature`
- `request_id`
- `as_of`
- `invest_level`
- `llm_vendor`
- `include_llm_explain`

### subject

분석 대상을 담는다.

- Feature1/2 종목 분석: `stock_code`, `company_name`, `exchange_code`, `currency`
- Feature3 포트폴리오 분석: 포트폴리오 성격상 `portfolio` 블록을 사용할 수 있다. 다만 가격, 벤치마크, overlay 같은 원천 데이터는 `input_data` 아래에 둔다.

### input_data

Spring이 준비한 원천 데이터와 LLM 설명에 필요한 요약 입력을 담는다.

- Feature1: `ohlcv`, `financials`, `market_context`, `market_snapshot`
- Feature2: `metrics`
- Feature3: `price_series`, `benchmark_series`, `overlay_signals`

### options

계산 방식과 표시 방식에 영향을 주는 분석 옵션을 담는다.

- Feature1: `freq`, `from`, `to`
- Feature2: `freq`, `window`
- Feature3: view mode, price basis, covariance model, return type, lookback, diagnostics, frontier, risk-free policy, LLM vendor

## 공통 Explain 구조

모든 기능의 보고서 설명은 같은 explain container를 사용한다.

```json
{
  "provider": "GEMINI|OPENAI|DETERMINISTIC|LLM",
  "model": "string|null",
  "text": "string|null",
  "sections": {},
  "overall": {
    "summary": "string",
    "bullets": [],
    "risks": [],
    "conclusion": "string|null"
  },
  "warnings": []
}
```

각 section은 반드시 다음 구조를 사용한다.

```json
{
  "title": "string",
  "summary": "string",
  "bullets": []
}
```

## 보고서 섹션

### Feature1

Feature1의 보조지표 계산 책임은 FastAPI에 유지한다.

- `price_flow`
- `market_snapshot`
- `indicators`
- `financial_timeline`

### Feature2

Feature2 explain은 JSON 문자열이 아니라 구조화된 객체로 전달한다.

- `macro_environment`
- `investor_flow`
- `short_selling`
- `peer_cluster`
- `news_sentiment`
- `cross_signal`

### Feature3

Spring은 FastAPI 호출 전에 가격, 벤치마크, overlay, risk-free 관련 원천 데이터를 준비한다. FastAPI는 포트폴리오 계산과 LLM 설명 생성을 담당한다.

- `core_risk`
- `volatility_analysis`
- `efficiency_analysis`
- `overlay_observations`
- `portfolio_comparison`
- `final_judgement`

## 기능별 책임

### Spring

- public API envelope 생성
- 인증, 크레딧 차감, 환불 처리
- 종목 매핑과 거래소 식별
- DB 및 외부 데이터 소스 조회
- Feature3 가격 시계열과 벤치마크 시계열 준비
- public API 경계에서 warning 정규화

### FastAPI

- payload-only 내부 응답 반환
- Feature1 기술적 보조지표 계산
- Feature3 포트폴리오 수학, 공분산, CAPM, 최적화 계산
- LLM 프롬프트 구성
- LLM JSON 출력 강제
- LLM 응답 파싱과 fallback warning 생성

## LLM 출력 정책

- 프롬프트는 JSON-only 출력을 요구해야 한다.
- OpenAI는 가능한 경우 JSON schema response format을 사용한다.
- Gemini는 가능한 경우 `responseMimeType: "application/json"`을 사용한다.
- parser는 잘못된 JSON을 구조화된 데이터처럼 통과시키지 않는다. 이 경우 `LLM_EXPLAIN_PARSE_FAILED` warning으로 처리한다.
- `invest_level`은 설명 난이도와 용어 밀도만 조절한다. 계산 결과, 위험 등급, 추천 정책을 바꾸면 안 된다.
- 보고서는 직접적인 매수, 매도, 리밸런싱 권유를 피해야 한다.
