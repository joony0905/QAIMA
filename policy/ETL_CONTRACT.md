# QAIMA ETL 데이터 계약

## 공통 계약

자동 적재는 idempotent 해야 한다. 동일 기간을 여러 번 실행해도 중복 행이 생성되지 않아야 하며, 최신 값은 upsert로 반영한다.

모든 자동 적재는 Asia/Seoul 기준으로 동작한다.

KIS 호출은 전역 rate limit을 공유한다.

- 운영 기본값: 10 requests/sec
- 외부 정책 상한: 15 requests/sec

## `industry_index_ohlcv`

Source:

- KIS `inquire-daily-indexchartprice`
- Java client: `IndustryIndexFetcher`

Target:

- `industry_index_ohlcv`

Key:

- `index_id`
- `ts`
- `freq`

Frequency:

- `Freq.ONE_D`

Update contract:

- `industry_index` master 전체를 대상으로 한다.
- 기존 최신 `ts`가 없으면 1095일 lookback을 수행한다.
- 기존 최신 `ts`가 있으면 최근 tail 구간만 refresh한다.
- KIS 응답의 OHLC 필수값이 없으면 해당 row는 저장하지 않는다.

Failure contract:

- 특정 index 실패는 전체 배치를 중단하지 않는다.
- empty 응답은 `empty`로 집계한다.

## `market_investor_flow`

Source:

- KIS `inquire-investor-daily-by-market`
- Java service: `MarketInvestorFlowService`

Target:

- `market_investor_flow`

Key:

- `market_code`
- `industry_code`
- `trade_date`
- `source`

Market targets:

- `KSP` with default industry code `0001`
- `KSQ` with default industry code `1001`

Update contract:

- 기존 최신 `trade_date`가 없으면 30일 lookback을 수행한다.
- 기존 최신 `trade_date`가 있으면 최근 tail 구간만 refresh한다.
- 당일 데이터는 15:40 이후 조회 가능하나 산출 지연이 있을 수 있다.
- 15:40 이전 또는 휴장일에는 전 거래일을 `effectiveTo`로 사용한다.

Failure contract:

- market 단위로 1회 재시도한다.
- `KIS_MARKET_CLOSED`는 재시도하지 않고 `timeLimited`로 집계한다.
- 최종 실패한 market은 `failedTargets`에 기록한다.

## `stock_investor_flow`

Source:

- KIS `investor-trade-by-stock-daily`
- Java service: `StockInvestorFlowService`

Target:

- `stock_investor_flow`

Key:

- `stock_id`
- `trade_date`
- `source`

Target stock contract:

- `delisted_at IS NULL`
- `asset_type`이 비어 있거나 `EQUITY`
- exchange code가 `KOSPI`, `KOSDAQ`, `KONEX`

Ordering contract:

- 수급 데이터가 없는 종목 우선
- 최신 수급일이 오래된 종목 우선
- 종목코드 오름차순

Batch size:

- 기본 300종목
- `kis-batch.stock-investor-flow.limit`으로 조정

Update contract:

- 기존 최신 `trade_date`가 없으면 30일 lookback을 수행한다.
- 기존 최신 `trade_date`가 있으면 최근 tail 구간만 refresh한다.
- KIS API 특성상 기간 조회 내부에서 여러 base date 호출이 발생할 수 있으며, 각 호출은 rate limit 대상이다.
- 당일 데이터는 15:40 이후 조회 가능하나 산출 지연이 있을 수 있다.
- 15:40 이전 또는 휴장일에는 전 거래일을 `effectiveTo`로 사용한다.

Failure contract:

- 종목 단위로 1회 재시도한다.
- `KIS_MARKET_CLOSED`는 재시도하지 않고 `timeLimited`로 집계한다.
- 최종 실패한 종목은 `failedTargets`에 기록한다.
- 일부 종목 실패는 전체 배치를 실패시키지 않는다.

## Observability

각 배치는 종료 시 `BatchSummary`를 로그로 남긴다.

Required fields:

- `target`
- `success`
- `skipped`
- `empty`
- `timeLimited`
- `failed`
- `retried`
- `retryRecovered`
- `savedRows`
- `failedTargets`

이 로그를 기준으로 클라우드 운영 알림을 구성한다.
