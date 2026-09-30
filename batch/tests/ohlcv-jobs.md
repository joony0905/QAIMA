# Python 가격·산업지수 OHLCV 배치 검증

검증일: 2026-09-25. WSL Python3.12.3. `test_ohlcv_jobs.py`의신규25개는22 PASS·3개메서드FAIL(8실패subcase)입니다. 기존45개와합친전체70개는65 PASS·5개메서드FAIL(14실패subcase), errors0/skipped0,0.182초입니다. 제품코드·설정은변경하지않았습니다.

## 실행·격리·흐름

프로젝트루트에서실행합니다.

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 \
  .venv_wsl/bin/python -B batch/tests/run_guarded.py
```

실행기는실제socket/HTTP/MySQL/subprocess/.env읽기를차단합니다. 이번실행의network_attempts/subprocess_attempts/dotenv_read_attempts/unmocked_http_attempts/unmocked_db_attempts는모두0, paid_calls0입니다. HTTP Session·DB Connection/Cursor·토큰파일open은mock입니다. 실제requests.Response의JSON해석, 제품KisClient·재시도·기간분할·중복정리·Repository의SQL조립·main/대상루프는실행합니다. `time.sleep`은mock이며호출횟수와인자만검사합니다. 실제속도제한이나처리량벤치마크가아닙니다.

최대연결범위는두실제main→토큰응답처리→실제HTTP요청조립/JSON해석→실제구간분할→실제Repository/SQLtuple→합성commit/close입니다. CLI인자파서는Namespace로대체합니다. 전체앱·자동Spring scheduler·실제MySQL unique/upsert/transaction·실제KIS는검증하지않습니다.

기준일은합성2026-01-04, 시각은KST10:00입니다. 날짜는date/datetime의테스트subclass로고정하고종료시복구합니다. 실제거래일/휴장일을가정하지않습니다. 기본가격은open100/high120/low90/close110/volume1234이며, price는float/index는Decimal입니다. 기존가격유틸검사와함께확정시각기본설정16:00의경계도검사합니다.

## 신규25개메서드

| 메서드 | 입력·수행·기대값 | 결과 |
|---|---|---|
| `test_actual_json_mapping_drops_bad_date_and_missing_close_and_sorts` | 두client의실제JSON:1/3종가120·2월30일·close비수치·1/1종가100→1/1,1/3정렬·숫자/거래량1234·기간query·종목J/009999,지수U/0999·가격raw옵션0 | PASS |
| `test_invalid_index_code_is_rejected_before_token_or_http` | 지수code U한글자→ValueError·토큰/GET0 | PASS |
| `test_non_list_output_currently_becomes_empty_observed` | rt_cd0/output2가dict→빈목록으로처리 | PASS:관측 |
| `test_infinite_close_must_not_become_a_valid_candle` | 두client×±Infinity close→유효캔들제외기대, 실제float/Decimal무한대캔들유지 | FAIL:4subcase |
| `test_business_rate_limit_retries_twice_then_recovers` | 거래건수제한business응답2회→세번째성공·GET3·backoff2 | PASS |
| `test_http_rate_limit_retries_but_other_http_error_does_not` | HTTP429/거래건수문구→재시도성공GET2/sleep1; 일반403→즉시HTTPError/GET1/sleep0 | PASS |
| `test_continuous_business_rate_limit_stops_at_three_attempts` | 계속거래건수제한→GET3/backoff2후RuntimeError | PASS |
| `test_transport_timeout_is_not_retried_by_page_loop_observed` | 실제요청호출에Timeout→전파·GET1 | PASS:관측 |
| `test_price_token_rate_limit_stops_at_five_attempts` | 가격토큰403/EGW00133지속→POST5·sleep60초인자4·HTTPError·캐시쓰기0(실제대기/비용없음) | PASS |
| `test_token_ttl_invalid_falls_back_and_small_value_clamps` | 두client토큰TTL bad→3600,1→60초 | PASS |
| `test_full_range_partitions_without_gaps_and_dedupes_last_value` | 실제full범위1/1~1/4·page상한2일→1/1~2,1/3~4; 동일1/2일110→120최종값·범위밖1/5제거·100/120/130·sleep25ms×2인자 | PASS |
| `test_reversed_full_range_does_not_request_provider` | from>to→empty·GET0 | PASS |
| `test_target_selection_binds_key_freq_and_limit_and_closes` | 실제targetSQL→key/freq4/limit2 bound·NOT EXISTS·LIMIT·cursor.close | PASS |
| `test_price_sql_params_preserve_raw_values_and_return_driver_rowcount` | 실제가격Repository→id7/자정/freq4/OHLCV8필드·upsertSQL·commit1; 합성driver rowcount2반환 | PASS |
| `test_price_today_before_confirmation_is_filtered_before_cursor_creation` | 가격당일KST10:00→저장0·cursor도생성하지않음 | PASS |
| `test_price_confirmation_boundary_and_future_date_behavior_observed` | 15:59불가/16:00가능; 미래날짜는이helper가허용·다른freq도허용 | PASS:관측포함 |
| `test_index_splits_chunks_and_commits_each_with_decimal_values` | index3행/batch2→2+1분할·commit2·정확한Decimal/자정/id91/freq4 | PASS |
| `test_index_current_day_has_no_price_confirmation_guard_observed` | index당일KST10:00→tuple제출1·commit1(가격과다름) | PASS:관측 |
| `test_failed_executemany_must_rollback_before_connection_reuse` | 두Repository의executemany예외→commit0/close1, rollback1기대이나실제0 | FAIL:2subcase |
| `test_index_later_chunk_failure_keeps_first_commit_observed` | batch1·첫행성공/두번째SQLerror→첫commit1유지·rollback0·cursorclose | PASS:호출관측 |
| `test_skip_empty_failure_and_success_have_distinct_summary_counts` | 두대상루프4개:최신/empty/예외/성공→total4/success1/partial2/hardfail1·GET3 | PASS |
| `test_price_all_unconfirmed_rows_currently_count_as_success_observed` | 실제가격Repository당일미확정필터→DB0인데상위루프success1 | PASS:관측 |
| `test_all_failed_targets_must_produce_nonzero_cli_exit` | 두실제main/client/loop의유일대상Timeout→GET1/commit0/close1, 실패종료기대이나return0 | FAIL:2subcase |
| `test_both_actual_main_client_json_repository_paths_commit_and_close` | 두실제main/client/JSON/Repository연결·1/2가격110→토큰1/GET1/commit1/close1·SQL날짜/종가대조 | PASS |
| `test_top_level_target_query_error_returns_one_and_closes` | 대상조회자체SQLerror→두main return1·connclose1, 대상내실패return0과대조 | PASS |

## 발견사항

### BATCH-OHLCV-NONFINITE-001

`load_price_ohlcv.to_float`와`load_index_ohlcv.to_decimal`은일부NaN표현을거르지만±Infinity를거르지않습니다. 실제page JSON→parser에서무한대종가의CandleRow가유지됩니다. 이검사는캔들생성단계의증거이며실제DB저장성공이나오류를단정하지않습니다. 1메서드/4subcase실패를유지했습니다.

### BATCH-OHLCV-ROLLBACK-001

두Repository의upsert는finally에서cursor를닫지만executemany실패시connection.rollback을호출하지않습니다. 상위루프가예외를잡고동일connection을다음대상에재사용하는구조라실패한트랜잭션의정리경계가없습니다. 실제MySQL의부분실행/오염/commit결과는미검증입니다. 회귀는rollback호출누락만입증하며1메서드/2subcase실패입니다. index의분할commit관측에서는먼저성공한chunk는이미commit호출됐고두번째실패에서도rollback0이었습니다.

### BATCH-OHLCV-EXIT-001

두main은bootstrap이반환한BootstrapResult의hard_fail을검사하지않고return0합니다. 실제main/client/대상루프에서모든대상의Timeout을발생시키면commit0인데도성공종료값0이반환됐습니다. 대상조회자체의예외는main에서return1인대조군은통과했습니다. CLI/CI가종료코드만보면전체대상실패를놓칠수있습니다. 실제subprocess를실행한것은아니며main반환값을검사했고,파일하단의진입점이`sys.exit(main())`를쓰는코드와연결됩니다. 1메서드/2subcase실패입니다.

### 그외관측과미완료

산업지수Repository에는가격의당일확정시각검사가없습니다. 가격helper도미래날짜자체를거부하지않지만정상full조회는요청범위로최종필터링합니다. 미확정가격만받아실제저장요청이0이어도상위루프는success1로집계합니다. 잘못된output2형태는empty가되며timeout은page재시도대상이아닙니다. 이관측들은실제거래일·제공처규격·운영scheduler동작검증과구분합니다.

남은범위는실제SQL/upsert멱등성·동일connection장애복구·날짜timezone/휴장일, 실제KIS페이지상한/응답coverage·재시도속도, 토큰파일동시접근, CLI전체인자검증, 종목bootstrap·수동산업mapping·해외조회job입니다. 사용자승인전기존Redis쓰기/임시MySQL실행은하지않습니다.
