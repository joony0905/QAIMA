# Python 종목·시장 수급 배치 검증

검증일: 2026-09-25, WSL Python3.12.3. 대상은 `jobs/load_stock_investor_flow.py`, `jobs/load_market_investor_flow.py`입니다. [ETL 정책](../../policy/ETL_POLICY.md)의 운영 자동화 기준은 Spring Scheduler이고 Python은 초기 적재·수동 보정용입니다. Spring의 재시도/대상순환/거래일 규칙을 이 수동 스크립트가 모두 구현했다고 가정하지 않습니다.

## 실행·결과·격리

프로젝트 루트에서:

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 \
  .venv_wsl/bin/python -B batch/tests/run_guarded.py
```

신규 `test_investor_jobs.py`는30개 중28 PASS·2개 메서드 FAIL입니다. 각 실패 메서드가3개의 비유한 숫자를 검사하므로 unittest의failure 항목은6개입니다. 기존15개와 합친45개 중43 PASS·2개 메서드 FAIL·SKIP0입니다. 실패를 expectedFailure/skip으로 숨기지 않아 exit1이 정상적인 현재 회귀 결과입니다.

수급검사추가시점의 guarded 실행은45개/0.082초, errors0/skipped0/failed_methods2/failures6이었습니다. network_attempts·subprocess_attempts·dotenv_read_attempts·unmocked_http_attempts·unmocked_db_attempts는모두0, paid_calls0입니다. 이 시간은합성fixture의unittest 실행시간이며실제배치처리량벤치마크가아닙니다.

`run_guarded.py`는 dotenv 비활성화 변수를 강제하고 자격증명/프록시 관련 환경변수를 제거한 뒤, import 이전에 socket.connect·subprocess·.env 읽기를 차단합니다. 추가로 requests의 기본 request와 mysql.connector.connect를 차단하여 HTTP transport 또는 MySQL native extension으로 mock을 우회하는 실수를 막습니다. 실제 호출 대신 테스트가 설치한 합성 Session/Connection만 사용합니다. 최종 카운터가 하나라도0이 아니면 suite 결과와 별개로 실패합니다.

이전 배치 문서의 `DOTENV_DISABLED`는 설치된 python-dotenv가 인식하는 변수가 아니었습니다. 이번에 테스트 문서를 `PYTHON_DOTENV_DISABLED`로 고치고 기존 테스트2파일도 올바른 플래그 없이 제품 module을 import하기 전에 거부하도록 했습니다. 제품 dotenv 동작은 수정하지 않았습니다. 이전 실제 환경이 격리됐다는 증거로 잘못된 변수명을 사용하지 않으며, 새 guarded 재실행을 현재 증거로 사용합니다.

토큰 파일은 `mock_open`으로 메모리에서만 읽기/쓰기를 수행합니다. 사용하는 key/secret/token은 모두 합성 문자열이며 YAML 비밀값은 읽지 않습니다. 외부 네트워크·실제DB·토큰파일 저장·유료 호출·기존 데이터 변경은0입니다. native MySQL driver의 실제 transaction/unique key/upsert/Decimal 전송 결과는 검증하지 않았습니다.

## 입력과 흐름

합성 종목은 stockId7/code009999/합성기업/KOSPI, 조회기간은2026-01-01~02-12입니다. 종목23개 숫자 필드에1.125~23.125를 서로 다르게 넣어SQL tuple 위치를 대조합니다. 시장6개 숫자는-1234.5/2.125/3/4/5/6입니다. 값 단위는 코드의 million 명칭을 그대로 대조하며 실제 KIS 단위 규격의 독립 검증은 아닙니다.

가장 넓은 연결 검사는 두 실제 main→실제 KisClient→실제 requests.Response.json→실제 날짜/Decimal mapping→실제 repository/SQL 조립→합성 cursor/commit까지 실행합니다. 인자 parser의입력은Namespace로 고정하고 자격증명 조회/HTTP Session/DB connector/file IO를 대체합니다. SQL 문자열을DB에서 실행하지 않습니다.

## 신규30개 메서드별 방법

| 메서드 | 입력·실행·독립 기대값 | 결과 |
|---|---|---|
| `test_stock_23_numeric_fields_map_to_distinct_sql_positions` | 종목23개고유소수→tuple4~26의위치/stock/date/시장J/source/TR/timestamp·placeholder31개 대조 | PASS |
| `test_market_six_numeric_fields_map_without_scaling` | 시장6개소수/음수/쉼표→단위추가변환없는Decimal·tuple/source/TR/timestamp·placeholder13개 | PASS |
| `test_dates_filter_invalid_and_outside_but_include_both_boundaries` | null/빈/2월30일/범위밖거부, 시작·끝일포함(두job) | PASS |
| `test_finite_decimal_missing_and_normalization` | 음수쉼표소수유지·null/빈/문자None·6자리/.XKRX/.XKOS·KOSDAQ→KSQ·시장기본코드 | PASS |
| `test_stock_nonfinite_numbers_must_not_reach_mysql_decimal_params` | 종목NaN/+Infinity/-Infinity→결측None 기대, 실제비유한Decimal이SQLtuple에남음 | FAIL:3subcase |
| `test_market_nonfinite_numbers_must_not_reach_mysql_decimal_params` | 시장동일3입력→동일비유한Decimal통과 | FAIL:3subcase |
| `test_token_issue_headers_and_memory_reuse` | 양쪽client 실제issue/header→POST1·memory재사용·Bearer1회·TR/timeout7/합성auth body·mock파일쓰기 | PASS |
| `test_valid_cached_bearer_prefix_is_not_duplicated` | mock캐시issued900/expires2000/clock1000→Bearer중복없음·POST0 | PASS |
| `test_expired_and_five_second_margin_tokens_refresh` | expires999→재발급; issued900/TTL105/clock1000의5초경계→재발급 | PASS |
| `test_missing_token_response_raises_and_does_not_cache` | 토큰응답키없음→RuntimeError·memory토큰없음·파일쓰기0 | PASS |
| `test_stock_request_normalizes_code_and_actual_response_selection` | 9999.XKRX→009999·시장J·기준일20260212·기타query·output2선택 | PASS |
| `test_market_request_uses_to_for_both_dates_observed` | from≠to에도HTTP두날짜가to·시장KSQ·industry1001 | PASS:현재동작 관측 |
| `test_provider_business_error_is_not_an_empty_success` | 양쪽rt_cd1→RuntimeError, 빈성공결과로숨기지않음 | PASS |
| `test_http_error_and_malformed_json_propagate_without_hidden_retry` | 양쪽HTTP예외/JSON예외→전파·GET각1회 | PASS |
| `test_upsert_commit_rollback_cursor_close_for_both_jobs` | 양쪽정상executemany→commit1/close1; SQLerror→rollback1/close1; empty→DB0 | PASS |
| `test_target_query_parameters_and_no_domestic_exchange_filter_observed` | 실제종목SQL의delisted/equity/missing/limit·bound인자검사, 합성NASDAQ행도StockRef로반환 | PASS:현재동작 관측 |
| `test_latest_date_normalizes_datetime_and_always_closes_cursor` | row없음/날짜null/datetime→None/None/date, cursor항상close | PASS |
| `test_repository_filters_range_but_submits_duplicate_dates_observed` | 동일일값1/2+범위밖1→범위밖제거·동일일두tuple그대로executemany·반환건수2 | PASS:현재동작 관측 |
| `test_anchor_dates_are_inclusive_and_unique` |42일기간→2/12·1/22·1/1, 단일일1개, 나누어떨어지지않는시작일도끝에포함 | PASS |
| `test_fetch_range_backfill_refresh_and_skip_boundaries` | 최신없음전체; 최신2/10→2/8; 오래된최신→tail2/2; 최신to→SKIP | PASS |
| `test_success_calls_every_anchor_and_sleeps_after_requests_and_target` | 실제종목루프3anchor·save3→success1, sleep250ms인자4회(실제대기없음) | PASS |
| `test_up_to_date_is_partial_not_success_and_skips_io` | 최신to→partial1/already up-to-date·GET/save/sleep0 | PASS |
| `test_zero_saved_rows_reports_partial_even_if_provider_rows_exist` | raw3/save0→partial1·rawRows3요약 | PASS |
| `test_failure_does_not_retry_but_next_stock_continues` | 첫종목timeout1회·다음종목3anchor→hardfail1/partial1, 같은대상재시도없음 | PASS:현재동작 관측 |
| `test_later_failure_keeps_prior_commit_and_rolls_back_failed_chunk` | 실제Repository/upsert연결 첫chunkcommit·두번째SQLerrorrollback→hardfail1·세번째미실행 | PASS:호출경계 |
| `test_both_actual_mains_clients_json_and_sql_pipeline_without_external_io` | 두실제main/client/json/repo·하루fixture12.50→종목commit1/시장commit2·토큰각1·rollback0·connclose1 | PASS |
| `test_stock_main_wires_arguments_and_closes_connection` | 실제main→onlyMissing/refreshTail7·실제Repository 전달·connclose1(상위루프mock) | PASS |
| `test_invalid_range_is_rejected_before_any_client_or_db` | 두main의from>to→SystemExit·client/DB0 | PASS |
| `test_market_main_connects_both_markets_maps_filters_and_commits` | 실제시장main·fetchmock→기본KSP/0001,KSQ/1001·범위밖제거·Decimal123.5·commit2·close1 | PASS |
| `test_market_failure_aborts_remaining_markets_but_closes_connection_observed` | 첫시장timeout→전파·두번째시장미실행·commit0·close1 | PASS:현재동작 관측 |

## 발견사항·한계

**BATCH-INVESTOR-NONFINITE-001:** 두 스크립트의 `parse_decimal`은 Decimal 변환 예외만 처리하고 `is_finite`를 확인하지 않습니다. NaN/±Infinity는 Decimal 생성에 성공하므로수량SQL인자로전달됩니다. 보통의비수치문자는None인대조군은통과했습니다. 실제DB가어떤예외를내는지 또는 데이터가저장되는지는실행하지않았으므로단정하지않습니다. 제품미수정·실패회귀2메서드/6subcase유지.

관측상 Python종목배치는 timeout을재시도하지않고다음종목으로진행하며, Python시장배치는첫시장오류로전체main이종료됩니다. 시장client는요청from을HTTP query에쓰지않고to를두날짜에사용합니다. 종목선택SQL에는국내거래소제한조건이없고, 중복행을실제upsert가어떻게병합했는지검증하지않았습니다. 이는Spring자동스케줄러검증결과와별도이며 실제제공처규격/기간coverage/운영영향확인전 추측으로정상·결함을확정하지않습니다.

남은 범위: 실제SQL 멱등성·부분커밋복구·driver 숫자변환, 실제KIS 응답/접근권한/기간coverage, 동시프로세스토큰캐시경합·실제요청속도, CLI parser의모든옵션, index/price전체배치·종목bootstrap·해외조회·수동산업매핑입니다. 두job의synthetic흐름검증을배치전체완료로기록하지않습니다.
