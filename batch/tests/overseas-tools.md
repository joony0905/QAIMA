# 해외 산업 코드·차트 도구 검증

검증일: 2026-09-25, WSL Python3.12.3. 대상은 `kis_overseas_industry_codes.py`, `kis_overseas_daily_chartprice.py`입니다. 신규 `test_overseas_tools.py` 23개 중20 PASS·3개 메서드 FAIL(실패항목5개)입니다. 전체 배치는117개 중108 PASS·9개 메서드 FAIL(실패항목20개), errors0/skipped0,0.312초입니다. 제품 코드는 수정하지 않았습니다.

## 실행과 경계

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 \
  .venv_wsl/bin/python -B batch/tests/run_guarded.py
```

실제 KisClient·requests.Response.json·날짜/Decimal 변환·페이지 분할·JSON/CSV writer·Repository SQL 생성·main·argparse를 실행합니다. HTTP Session/DB connector와 cursor/file open만 mock입니다. 출력 JSON·CSV는 mock 파일 handle이 받은 문자열을 실제 json.loads/csv.DictReader로 다시 읽어 검증합니다. 실제 파일 encoding을 디스크에서 roundtrip한 것은 아니며 요청 encoding 인자도 별도 확인합니다.

자격증명과 응답은 모두 합성입니다. 기본 네트워크·HTTP·MySQL·subprocess·.env 읽기 차단 장치는 유지했습니다. 최종 network_attempts/subprocess_attempts/dotenv_read_attempts/unmocked_http_attempts/unmocked_db_attempts는 모두0, paid_calls0입니다. 주봉·월봉 저장 결함도 합성 SQL 매개변수에서만 재현했으며 실제 DB에 기록하지 않았습니다.

첫 실행의 JSON/CSV 검사1개 ERROR는 테스트 helper가 `mock_open()`을 다시 호출해 마지막 open 인자 기록을 덮어쓴 문제였습니다. helper를 이미 생성된 return_value handle 참조로 고친 뒤 통과했습니다. 제품 writer 변경은 없고 이 오류를 제품 결함으로 집계하지 않습니다. 최종 제품 실패는 아래3개 메서드입니다.

## 신규23개 검사

| 메서드 | 수행·독립 기대값 | 결과 |
|---|---|---|
| `test_industry_codes_actual_json_trim_skip_and_query` | 실제 JSON→code/name trim·양쪽빈행제외·NAS/QA01/합성산업·AUTH/EXCD query | PASS |
| `test_name_only_and_code_only_industry_rows_are_retained_observed` | 이름만/코드만 있는 행은 각각 유지 | PASS: 관측 |
| `test_chart_actual_json_numeric_mapping_sort_and_query` | invalid date/close 제외·날짜정렬·OHLCV Decimal/쉼표거래량·raw보존·5개query 인자 | PASS |
| `test_overseas_infinite_close_must_not_become_valid_history` | ±Infinity 종가→유효행 제외 기대, 실제 Decimal 무한대 행 반환 | FAIL:2subcase |
| `test_business_http_and_json_failures_are_not_success` | 두client×HTTP500/business오류/잘못된JSON→예외·GET1 | PASS |
| `test_malformed_output_container_currently_returns_empty_observed` | 두client output2=dict→empty | PASS: 관측 |
| `test_token_rate_limit_retries_five_times_without_actual_sleep` | 두client 403/EGW00133 지속→POST5/backoff4후HTTPError, 실제sleep없음 | PASS |
| `test_token_issue_reuse_ttl_clamp_and_expiry_margin` | TTL1→60·첫발급후재사용·clock1000→1055 경계재발급 | PASS |
| `test_daily_pagination_filters_deduplicates_and_returns_last_raw_page` |1/1~4 page2일:연속2구간·중복일마지막값·범위밖제거·100/120/130·raw는마지막page·중간sleep25ms1 | PASS |
| `test_nondaily_path_keeps_out_of_range_and_duplicates_observed` | W/M/Y는단일fetch결과직접반환, 중복/범위밖행도유지·sleep0 | PASS: 관측 |
| `test_rows_to_dicts_preserves_decimal_precision_and_nulls` | 큰소수12345678901234567890.123456 문자열·null·yyyyMMdd 유지 | PASS |
| `test_industry_json_and_csv_roundtrip_korean_and_quoted_commas` | 한글·쉼표·따옴표 이름을 JSON/CSV writer→parser로복원·CSV utf-8-sig인자 | PASS |
| `test_chart_json_and_csv_preserve_raw_response_and_decimal_strings` | raw_response 한글·rows Decimal 문자열을 JSON/CSV로복원 | PASS |
| `test_optional_index_lookup_and_chunked_upsert_bind_exact_values` | code→id91 bound query·2행/batch1 commit2·자정/freq4/Decimal8필드 | PASS |
| `test_failed_overseas_upsert_must_rollback` | executemany예외→close1/commit0, rollback1 기대이나실제0 | FAIL |
| `test_main_chart_without_save_never_connects_to_db` | 실제main/client/JSON, save옵션없음→토큰1/GET1·DB connector0 | PASS |
| `test_main_daily_save_connects_actual_client_parser_sql_and_closes` | 실제main/client/parser/Repository→code trim·freq4·commit1/close1 | PASS |
| `test_main_nondaily_save_must_not_label_weekly_or_monthly_rows_as_daily` | 실제main의W/M query 확인→SQLfreq 각각5/6 기대, 실제둘다4 | FAIL:2subcase |
| `test_main_missing_index_master_returns_error_and_closes_without_write` | 저장옵션+없는master→main1·UPDATE/commit0·close1 | PASS |
| `test_main_invalid_iso_date_fails_before_provider_fetch` |2026-02-30→main1·토큰/GET0 | PASS |
| `test_industry_main_normalizes_exchanges_and_writes_outputs` | NAS/NYS trim·uppercase·빈exchange건너뜀·stdout합성한글·두writer에2행전달(writer단독검사는위에서별도) | PASS |
| `test_industry_main_error_stops_remaining_exchange_and_returns_one` | 첫exchange Timeout→다음미실행·GET1·main1 | PASS |
| `test_cli_defaults_and_explicit_choices` | 실제argparse의기본NAS선택용None·chart D/save없음, W/save옵션허용 | PASS |

## 발견사항과 의미

### BATCH-OVERSEAS-FREQ-001 — 주봉·월봉 저장 주기가 일봉으로 고정

실제main에period W 또는M과save-index-code를주면client query는해당주기로전달하지만Repository호출의freq는항상DEFAULT_FREQ_ONE_D입니다. 현재기본값4이며, Spring `domain/Freq.java`와 `IndustryIndexOhlcvId`의EnumType.ORDINAL 기준ONE_D=4/ONE_W=5/ONE_M=6입니다. SQLtuple에서도W→4/M→4가재현됐습니다. 서로다른주기가일봉키에저장될위험이며, 실제기존일봉이덮였다는주장은아닙니다. 회귀1메서드/2subcase실패를유지했습니다.

### BATCH-OVERSEAS-ROLLBACK-001 — SQL 오류 rollback 누락

Repository가executemany예외후cursor.close만호출하고connection.rollback을호출하지않습니다. 실제SQL영향은미검증입니다. 이도구의main은finally에서connection을닫으므로같은connection으로다음종목을계속처리하는국내OHLCV배치와영향범위를구별합니다. 명시적실패트랜잭션정리회귀1개FAIL입니다.

### BATCH-OVERSEAS-NONFINITE-001 — 비유한 종가 수용

to_decimal이±Infinity를통과시켜실제JSON에서무한대종가의CandleRow가생성됩니다. 유효과거가격으로제외해야한다는회귀1메서드/2subcase가실패했습니다. 실제SQL전송·driver처리는검증하지않았습니다.

## 남은 범위

KIS실제규격·기간coverage·시장종류별시간대/지원주기, 실제네트워크/속도제한/토큰파일경합, 물리파일encoding/권한, 실제DB unique/upsert/rollback·기간주기충돌, CLI전체경계는남았습니다. Y주기의DB표현도현재Spring Freq enum과별도판단이필요하며임의로정의하지않았습니다. 각관측PASS는현재동작기록이지지원범위·원천정확성승인이아닙니다.
