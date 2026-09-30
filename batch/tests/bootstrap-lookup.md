# 종목 초기화·대화형 KIS 조회 검증

검증일: 2026-09-25. 제품 소스는 수정하지 않았습니다. 테스트는 [test_bootstrap_lookup.py](test_bootstrap_lookup.py), 대상은 [bootstrap_stock_from_kis.py](../jobs/bootstrap_stock_from_kis.py)와 [kis_raw_lookup.py](../jobs/kis_raw_lookup.py)입니다.

## 실행·결과·격리

저장소 루트의 WSL Python에서 실행합니다.

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 \
  .venv_wsl/bin/python -B batch/tests/run_guarded.py
```

추가 25개: **22 PASS·3 FAIL**, SKIP/ERROR 0. 추가 시점의 전체 결과는 142개: **130 PASS·12개 메서드 FAIL**, 실패항목 23개, ERROR/SKIP 0, unittest 측정 1.264초입니다. subTest 실패항목과 실패 메서드 수를 구분합니다. 실패를 expectedFailure로 숨기지 않았고 실행기 종료값은 1입니다.

guarded runner의 socket 연결·subprocess·dotenv 읽기·미대체 HTTP·미대체 MySQL 시도는 모두 0, 유료 호출 0입니다. 임시 MySQL 실행이나 실제 Redis 키 생성은 하지 않았습니다. 실제 token 파일도 생성하지 않았습니다.

실행 경로는 `main → pandas CSV decoding/파싱 → seed 정규화 → KIS client/Response.json → 분류 mapping → Repository SQL/commit 요청 → connection.close`입니다. `main`, 변환기, repository 구현은 실제 코드입니다. 다만 파일 경계는 BytesIO/mock_open, Session과 MySQL connection/cursor는 MagicMock, time.sleep은 기록만 하는 mock입니다. CSV 테스트는 `pd.read_csv` 진입점을 합성 BytesIO로 연결하고 원래 pandas parser를 호출합니다. 독립 기대값은 합성 종목코드·날짜·SQL tuple·호출 횟수로 직접 단언합니다.

대화형 조회가 import한 `bootstrap_stock_from_kis` 모듈을 그대로 사용해 client 경로가 다른 module mock으로 빠지지 않게 했습니다. 실제 터미널 자식 프로세스는 실행하지 않았습니다. bootstrap의 `sys.exit(main())`, lookup의 `SystemExit(main())`와 별개로 여기서는 main 반환값/예외를 검사합니다.

## 메서드별 수행 방법

모든 메서드는 `BootstrapLookupTests`에 있습니다. PASS 중 `observed` 또는 현재 동작이라고 표시한 항목은 정책 적합성까지 보증하지 않습니다.

| 테스트 메서드 | 입력·수행 및 독립 단언 | 결과 |
|---|---|---|
| `test_normalize_stock_codes_and_permissive_digit_extraction_observed` | None/nan/abc→None, 5930.0/A005930→005930. 7자리와 12x34→001234도 허용하는 현재 동작 확인 | PASS |
| `test_real_csv_utf8_bom_and_cp949_preserve_leading_zero` | UTF-8 BOM/CP949 bytes의 005930·인용된 한글 쉼표 필드를 실제 pandas로 읽음. CP949는 3개 encoding 순서, BOM은 첫 시도 성공 | PASS |
| `test_seed_dedup_sort_blank_and_cross_market_identity` | 5930/J와005930/J 중복 제거, 빈 코드 제외, 같은005930/Q는 별도 유지, 000660 먼저 정렬 | PASS |
| `test_seed_missing_column_and_empty_paths_rejected` | 필수 stock_code 없는 CSV 및 입력경로 빈 목록 각각 ValueError | PASS |
| `test_path_resolution_and_market_inference` | 쉼표/반복 CSV 인수 분리, KOSPI/J·KOSDAQ/Q·KONEX/K 파일명 추론, 기본경로도 없으면 ValueError | PASS |
| `test_mapping_aliases_date_and_explicit_market_priority` | 앞 이름 공백 후 prdt_name, isin_cd/list_dt/mcls_cd/scls_cd alias 실제 변환. 윤년 날짜 보존, 명시 CSV Q가 응답 KOSPI보다 우선. 회사명 없음은 오류 | PASS |
| `test_invalid_primary_listing_date_must_fall_through_to_valid_alias` | scts_mket_lstg_dt=00000000, kosdaq_mket_lstg_dt=20200102. 기대 2020-01-02, 실제 None | **FAIL** |
| `test_client_real_json_output_query_and_memory_token_reuse` | 실제 Response.json으로 output 변환, 연속2조회에서 token POST1. 두 번째 Q/000660/PRDT_TYPE_CD300과 합성 bearer 확인 | PASS |
| `test_client_http_business_empty_and_malformed_fail_closed` | HTTP500·rt_cd1·output 빈dict/list·비JSON은 예외이며 정상 output으로 통과하지 않음 | PASS |
| `test_token_retry_budget_backoff_and_transport_timeout_observed` | EGW00133/403은 POST5·sleep60초 인수4, HTTP500·token누락은 POST5·sleep5초 인수4. Session.post 자체 Timeout은1회 후 전파. 실제 대기/전송0 | PASS |
| `test_bootstrap_counts_full_partial_failure_and_continues` | 완전분류/분류누락/Timeout 3종목 실제 client→loop. total3/full1/partial1/fail1, stock저장요청2, 실패/부분코드와 KRW/null분류 확인 | PASS |
| `test_bootstrap_min_interval_and_configured_sleep_arguments` | 시간1000 고정·2종목·sleep_ms25: sleep(.025), sleep(.12), sleep(.025) 인수. 실제 wall-clock 호출 제한 적합성 아님 | PASS |
| `test_repository_seed_and_stock_bound_sql_commit_close` | 거래소 seed tuple, 따옴표 회사명 SQL바인딩, stock9필드와 LAST_INSERT_ID 유지 확인. commit2/close2 | PASS |
| `test_repository_classification_identity_and_integrity_collision` | sector/industry 각각 최초SELECT 없음→INSERT IntegrityError→정체성 동일 재조회 id41. industry는 sector까지 조건, 두cursor 닫힘·commit0 | PASS |
| `test_repository_failure_must_rollback_before_connection_reuse` | stock INSERT에 합성 SQL 예외. cursor닫힘/commit0, 기대 rollback1이나 실제0 | **FAIL** |
| `test_prefix_map_normalized_names_and_conflicting_sources` | 보통주·2우B 정규화된 같은 prefix는 단일(11,22,회사명), 다른 회사명 source 두 개는 conflict | PASS |
| `test_backfill_match_mismatch_absent_conflict_and_already_resolved` | 6대상: 정상우선주1만 UPDATE 인수(5935,22,11), 명칭불일치1·source/종목없음2·충돌1·이미해결1은 갱신 안 함 | PASS |
| `test_actual_main_csv_json_mapping_sql_and_close` | 실제 bootstrap main부터 합성 CSV005930→token/JSON→sector/industry/stock SQL tuple 확인. seed 포함 commit4, close1, return0 | PASS |
| `test_actual_main_all_provider_failures_must_exit_nonzero` | 유일 종목 GET Timeout. stock INSERT0·거래소 seed commit1·close1 확인 후 기대 비0, 실제0 | **FAIL** |
| `test_actual_main_invalid_csv_returns_one_after_exchange_seed_observed` | 필수열 없는 파일. return1/close1/GET0이나 입력 검증 전에 seed commit1인 현재 순서 확인 | PASS |
| `test_actual_main_partial_row_runs_backfill` | 분류없는 종목 main, skip_backfill=False. 실제 부분저장 후 backfill 함수에 해당 코드 전달, commit2. 이 메서드의 backfill 함수는 mock이며 알고리즘은 별도 직접 검사 | PASS |
| `test_lookup_interactive_invalid_stock_and_market_reprompt` | input bad→5930.0/invalid→q, 재질문4번·005930/J·오류/정규화 안내 확인 | PASS |
| `test_lookup_main_actual_client_outputs_inner_json_not_envelope` | 실제 lookup main과 client. 5930/Q→005930/J query, 출력 한글 JSON은 전체 envelope가 아니라 내부 output과 동일 | PASS |
| `test_lookup_failure_and_eof_propagate_no_false_success` | 실제 lookup main GET Timeout 전파, 별도 input EOFError 전파·조회0 | PASS |
| `test_cli_defaults_and_explicit_flags` | 실제 argparse로 반복csv/skip-backfill/sleep0, 기본stock_code열, lookup의 두 인수 기본None 확인 | PASS |

## 발견사항과 해석 한계

- **BATCH-BOOTSTRAP-EXIT-001:** 모든 종목 조회가 실패해도 main은 BootstrapResult.hard_fail을 검사하지 않고 0을 반환합니다. 거래소 seed의 성공이 종목 적재 성공을 뜻하지 않으며 CI가 실패를 놓칠 수 있습니다. 실제 프로세스가 아닌 main 반환값 재현입니다.
- **BATCH-BOOTSTRAP-DATE-001:** 날짜 후보를 문자열의 비어 있지 않음으로 먼저 선택한 뒤 한 번만 파싱합니다. 앞 sentinel 00000000 때문에 뒤 유효한 상장일을 사용하지 못합니다. 합성 응답으로 재현했으며 실 제공처가 이 필드 조합을 보내는 빈도는 측정하지 않았습니다.
- **BATCH-BOOTSTRAP-ROLLBACK-001:** stock upsert 오류 후 rollback 호출이 없습니다. 상위 bootstrap 루프는 예외를 집계하고 같은 repository/connection으로 다음 종목을 계속 처리합니다. 호출 누락은 입증하지만 실제 MySQL statement 부분 효과·lock·후속 commit 영향은 미검증입니다.
- 현재 bootstrap은 CSV Q/K를 query에 그대로 보내며 대화형 lookup은 J로 정규화합니다. 서로 다름을 관측했으나 실제 제공처의 허용 코드와 대조하지 않았으므로 오류로 확정하지 않습니다.
- 분류별 commit이 분리되어 있고 CSV 입력 검증보다 exchange seeding이 먼저입니다. 테스트는 이 순서를 기록한 것이며 초기화 전체의 원자성을 보장하지 않습니다.
- permissive 종목 정규화·raw 조회가 내부 output만 출력하는 동작도 현상 기록이지 변경 승인/정책 판정이 아닙니다.

## 남은 검증

실제 KIS 인증/응답과 시장별 지원, 격리 MySQL의 DDL/unique/FK/SQL문법/재실행 멱등성·동시 경쟁·부분 실패 복구, backfill의 기존 sector와 source industry 관계 및 거래소별 동명/동일prefix 정책, 실제 파일 token cache와 권한·만료, 실제 subprocess 종료코드, 사용자 EOF/반복 입력의 실제 터미널을 추가 검증해야 합니다. 이 결과는 batch 전체 또는 Swagger 전체 흐름 완료를 뜻하지 않습니다.
