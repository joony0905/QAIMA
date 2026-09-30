# QAIMA ETL 운영 정책

## 범위

이 문서는 클라우드 가배포 이후 자동 적재 대상과 운영 규칙을 정의한다.

자동 적재 대상:

- `industry_index_ohlcv`
- `market_investor_flow`
- `stock_investor_flow`

자동 적재 대상 제외:

- `market_snapshot`: 원천 ETL 테이블이 아니라 파생지표 산출 결과로 관리한다.
- `price_ohlcv`: 클라이언트 조회 시 필요한 구간이 없으면 적재되는 lazy loading 방식으로 운영한다.

## 스케줄

| Job | Cron | Timezone | 목적 |
| --- | --- | --- | --- |
| `industry-index-ohlcv` | `0 10 18 * * MON-FRI` | Asia/Seoul | 산업지수 일봉 증분 적재 |
| `market-investor-flow` | `0 30 16 * * MON-FRI` | Asia/Seoul | KOSPI/KOSDAQ 시장 수급 증분 적재 |
| `stock-investor-flow` | `0 0 19 * * MON-FRI` | Asia/Seoul | 국내 종목별 수급 순환 적재 |

투자자 수급 데이터는 KIS 정책상 당일 데이터가 15:40 이후 조회 가능하다. 단, 데이터 산출 시간은 일정하지 않을 수 있으므로 15:40 직후의 empty/error 응답은 정상적인 산출 지연일 수 있다.

## KIS 조회 제한

KIS API는 초당 15건 이내로 호출해야 한다.

QAIMA 운영 기본값은 초당 10건이다.

```yaml
kis-batch:
  rate-limit:
    max-requests-per-second: 10
```

재시도 호출도 rate limit 대상에 포함한다.

## 날짜 산정

투자자 수급 자동 적재의 `effectiveTo`는 다음 규칙을 따른다.

- 한국 시간 기준 현재 시각이 15:40 이전이면 전 거래일
- 오늘이 휴장일이면 전 거래일
- 그 외에는 오늘

증분 조회 범위:

- 기존 데이터가 없으면 `effectiveTo - lookbackDays`부터 `effectiveTo`까지 조회한다.
- 기존 데이터가 있으면 `max(latestDate - 2일, effectiveTo - refreshTailDays)`부터 `effectiveTo`까지 조회한다.
- 최신 적재일이 `effectiveTo` 이상이면 해당 대상은 skip한다.

## 재시도 정책

일시적인 통신 장애로 일부 종목만 실패하는 상황을 줄이기 위해 대상 단위로 1회 재시도한다.

기본값:

```yaml
retry:
  max-attempts: 2
  delay-ms: 2000
```

재시도 대상:

- 네트워크 연결 실패
- timeout
- connection reset
- KIS HTTP 오류

재시도 제외:

- `KIS_MARKET_CLOSED`
- 인증/권한 오류
- 잘못된 요청 파라미터
- 종목 없음
- 응답 구조 파싱 오류

## 종목별 수급 순환 정책

`stock_investor_flow`는 전체 국내 종목을 한 번에 모두 처리하지 않는다. 기본값은 1회 300종목이다.

대상 우선순위:

1. 수급 데이터가 없는 종목
2. 최신 수급일이 오래된 종목
3. 종목코드 오름차순

이 정책은 KIS 호출 제한과 배치 시간을 안정적으로 유지하기 위한 것이다.

## 장애 처리

각 배치는 대상 단위 실패를 전체 실패로 전파하지 않는다. 실패 대상은 로그에 남기고 다음 대상으로 진행한다.

배치 종료 로그는 다음 항목을 포함한다.

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

예시:

```text
[StockInvestorFlowBatch] finished. target=300 success=297 skipped=0 empty=1 timeLimited=0 failed=2 retried=3 retryRecovered=1 savedRows=1485 failedTargets=[000000, 111111]
```

## 수동 보정

Python batch 스크립트는 초기 적재 및 수동 보정 용도로 유지한다.

- `batch/jobs/load_index_ohlcv.py`
- `batch/jobs/load_market_investor_flow.py`
- `batch/jobs/load_stock_investor_flow.py`

운영 자동화의 기준 구현은 Spring Scheduler이다.
