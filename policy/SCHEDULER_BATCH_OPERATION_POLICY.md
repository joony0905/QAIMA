# QAIMA 운영 배치/스케줄러 정책

## 목적

이 문서는 Spring Scheduler, admin-triggered sync, Python batch script의 운영 기준을 정의한다.

기존 `policy/ETL_POLICY.md`는 KIS 기반 자동 적재 대상인 업종지수 OHLCV, 시장 수급, 종목 수급을 중심으로 한다. 이 문서는 전체 운영 스케줄러와 수동 batch의 공통 운영 원칙을 다룬다.

## 적용 범위

Spring Scheduler:

- FINRA short selling sync
- OpenDART daily sync
- SEC issued shares sync
- KIS industry index OHLCV sync
- KIS market investor flow sync
- KIS stock investor flow sync
- email verification cleanup

Admin/manual sync:

- `/api/v1/admin/**` 하위 backfill/sync API
- SEC 13F import
- market snapshot backfill
- base-rate, bond-yield, exchange-rate sync

Python batch:

- `batch/jobs/load_price_ohlcv.py`
- `batch/jobs/load_index_ohlcv.py`
- `batch/jobs/load_market_investor_flow.py`
- `batch/jobs/load_stock_investor_flow.py`
- `batch/jobs/bootstrap_stock_from_kis.py`
- 기타 초기 적재 또는 수동 보정 스크립트

## 스케줄러 등록 원칙

신규 Scheduler는 다음 조건을 만족해야 한다.

- `enabled` property를 반드시 가진다.
- cron property를 외부 설정으로 분리한다.
- timezone을 명시한다. 국내 시장 기준 작업은 `Asia/Seoul`을 사용한다.
- batch 대상 수가 많은 작업은 `limit` 또는 대상 selection policy를 가진다.
- 외부 API 호출 작업은 timeout, retry, rate limit 또는 sleep 정책을 문서화한다.
- 시작/종료 로그에 job name과 summary를 남긴다.

권장 property 구조:

```yaml
domain:
  sync:
    enabled: true
    daily-cron: "0 0 0 * * *"
    limit: 300
```

KIS 계열처럼 하위 job이 여러 개면 현재 `kis-batch.*` 구조를 따른다.

## 현재 자동 스케줄

| Job | Property | Default cron | Timezone | 비고 |
| --- | --- | --- | --- | --- |
| FINRA short selling | `finra.sync.*` | `0 0 9 * * *` | Asia/Seoul | `lookback-days=7` |
| OpenDART daily sync | `opendart.sync.*` | `0 10 3 * * *` | Asia/Seoul | 국내 공시/재무 관련 |
| SEC issued shares | `sec.issued-shares.sync.*` | `0 0 18 * * *` | Asia/Seoul | 기본 `limit=300` |
| KIS industry index OHLCV | `kis-batch.industry-index-ohlcv.*` | `0 10 18 * * MON-FRI` | Asia/Seoul | 기본 `lookback-days=1095` |
| KIS market investor flow | `kis-batch.market-investor-flow.*` | `0 30 16 * * MON-FRI` | Asia/Seoul | KIS 산출 지연 고려 |
| KIS stock investor flow | `kis-batch.stock-investor-flow.*` | `0 0 19 * * MON-FRI` | Asia/Seoul | 기본 `limit=300` |
| Email verification cleanup | `auth.email-verification.cleanup-interval-ms` | `3600000ms` | JVM 기준 | 만료 인증 레코드 정리 |

스케줄 변경 시 `backend/src/main/resources/application.yml`과 이 문서를 함께 갱신한다.

## 환경별 enabled 정책

local/dev:

- 개발 중 의도하지 않은 외부 API 호출을 피하기 위해 장기/대량 sync는 기본 비활성화를 권장한다.
- 수동 검증이 필요한 job만 명시적으로 enabled 한다.
- Python batch는 `.env`와 token cache가 준비된 로컬에서 수동 실행한다.

staging:

- prod와 같은 cron을 쓰기 전에 limit을 낮춰 검증한다.
- 외부 API quota가 prod와 공유되는 경우 staging scheduler는 기본 비활성화한다.
- E2E 검증용 최소 dataset과 backfill 범위를 별도로 둔다.

prod:

- 자동 스케줄러는 운영자가 승인한 목록만 enabled 한다.
- 대량 backfill은 정규 스케줄과 겹치지 않는 시간에 수동 실행한다.
- 외부 API quota, DB write load, Redis load를 함께 본다.

## 중복 실행 방지

현재 코드 기준 분산 lock 정책은 없다. 따라서 운영 배포는 다음 중 하나를 만족해야 한다.

- Scheduler가 켜진 인스턴스를 1개로 제한한다.
- 동일 job의 cron이 겹치지 않도록 배포/재시작 시간을 관리한다.
- 다중 인스턴스 운영이 필요하면 Redis 또는 DB 기반 job lock 정책을 먼저 추가한다.

lock 도입 전에는 같은 scheduler를 여러 인스턴스에서 동시에 켜지 않는다.

## 대상 선정과 재실행 정책

batch job은 재실행 가능해야 한다.

- DB upsert 또는 unique key로 중복 row 생성을 막는다.
- 최신 데이터가 이미 있으면 skip한다.
- `lookback-days`, `refresh-tail-days`, `limit`으로 재처리 범위를 제한한다.
- partial failure는 실패 대상만 다음 실행에서 재시도할 수 있어야 한다.

대상 우선순위는 다음을 권장한다.

1. 데이터가 없는 대상
2. 최신 적재일이 오래된 대상
3. 운영상 중요도가 높은 대상
4. stable key 오름차순

## 외부 API 호출 정책

외부 API 호출 batch는 provider 정책을 우선한다.

KIS:

- 초당 15건 제한을 넘지 않는다.
- QAIMA 기본 운영 한도는 초당 10건이다.
- token 발급 rate limit과 data API rate limit을 분리해서 본다.
- retry 호출도 rate limit에 포함한다.

SEC:

- SEC EDGAR user-agent를 운영 연락 가능한 값으로 설정한다.
- 대량 sync는 limit을 두고 점진 실행한다.

OpenDART, FRED, BOK, FINRA:

- API key 누락은 설정 오류로 보고 job을 명확히 실패시킨다.
- provider 장애는 재시도 가능한 실패와 불가능한 실패를 구분한다.
- raw response 전체를 info log에 남기지 않는다.

## Retry 정책

재시도 대상:

- timeout
- connection reset
- 일시적 5xx
- provider rate limit 중 backoff 가능한 응답

재시도 제외:

- 인증/권한 오류
- 필수 설정 누락
- 잘못된 요청 파라미터
- symbol/code mapping 없음
- 영구적인 provider business error
- response schema decode 실패

재시도는 bounded 횟수와 bounded delay를 가져야 한다. 무기한 retry는 금지한다.

## 로그와 관측성

모든 batch 종료 로그는 가능한 한 같은 summary vocabulary를 사용한다.

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

장애 분석에 필요한 식별자는 남기되, secret과 token은 남기지 않는다.

권장 로그 레벨:

- 정상 시작/종료: `info`
- 대상 단위 실패 후 계속 진행: `warn`
- 전체 job 실패: `error`
- provider raw body/detail trace: `debug`

## Admin Sync 정책

관리성 sync/backfill endpoint는 원칙적으로 `/api/v1/admin/**` 아래에 둔다.

- admin 권한이 필요한 작업은 public/authenticated route로 노출하지 않는다.
- 대량 write endpoint는 `limit`, `from`, `to`, `dryRun` 또는 이에 준하는 안전장치를 검토한다.
- response에는 처리 summary를 반환한다.
- 장시간 작업은 timeout과 client retry 중복 실행을 고려한다.

`/api/v1/sync/**`처럼 admin prefix 밖에 있는 sync endpoint는 운영 노출 전에 권한 정책을 재검토한다.

## Python Batch 정책

Python batch는 다음 목적으로 유지한다.

- 초기 적재
- 운영 전 수동 backfill
- 장애 후 부분 보정
- 외부 API/DB 상태 진단

Python batch는 정규 운영 자동화의 단일 기준이 아니다. Spring Scheduler와 같은 데이터를 다루는 경우 다음을 지킨다.

- 같은 시간대에 Spring Scheduler와 동시에 실행하지 않는다.
- 대상 범위를 명시한다.
- `.env`와 token cache 파일은 git에 포함하지 않는다.
- 실행 후 saved/skipped/failed summary를 기록한다.
- 운영 DB에 쓰기 전 staging 또는 limit 실행으로 검증한다.

## 변경 시 확인 사항

- 신규 scheduler 추가 시 `enabled`, cron, timezone, limit, retry, log summary가 있는지 확인한다.
- cron 변경 시 시장 데이터 산출 시간과 provider quota를 함께 검토한다.
- 대량 job enabled 변경 시 다중 인스턴스 중복 실행 가능성을 확인한다.
- batch가 쓰는 table의 unique key/upsert 정책을 확인한다.
- admin sync endpoint를 추가하면 SecurityConfig 권한 경계를 확인한다.
- 운영 설정 변경 후 최소 한 번은 제한된 범위로 dry run 또는 limit run을 수행한다.
