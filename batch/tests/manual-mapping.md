# 수동 산업 분류 매핑 배치 검증

검증일: 2026-09-25. WSL Python 3.12.3. 대상은 `jobs/apply_stock_industry_manual_mapping.py`이며 제품 코드는 변경하지 않았습니다. 신규 `test_manual_mapping.py`는 24개 검사입니다.

신규24개는23 PASS·1 FAIL입니다. 최신 전체 배치는94개 중88 PASS·6개 메서드 FAIL(실패항목15개), errors0/skipped0, 실행시간0.234초입니다. network_attempts/subprocess_attempts/dotenv_read_attempts/unmocked_http_attempts/unmocked_db_attempts는 모두0, paid_calls0입니다. 이 시간은 합성 테스트 실행시간이며 실제 배치 성능 측정값은 아닙니다.

## 실행 방법과 경계

프로젝트 루트에서 실행합니다.

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 \
  .venv_wsl/bin/python -B batch/tests/run_guarded.py
```

실제 CSV/TSV parser, UTF-8/CP949 decoding, 정규화, 산업 조회/선택, SQL 생성, apply_rows, main, CLI 옵션 parser를 실행합니다. MySQL connection/cursor와 파일 열기만 합성 경계로 대체하고 logger를 캡처합니다. 파일 바이트는 BytesIO→TextIOWrapper를 통해 실제 encoding decoder로 읽습니다. parser 자체나 UnicodeDecodeError를 mock하지 않습니다. 실제 파일·DB·환경 설정·기존 분류를 변경하지 않습니다.

주의: 제품의 **dry-run은 읽기 전용이 아닙니다.** UPDATE를 실행한 뒤 마지막에 rollback합니다. 따라서 실제 DB에서 실행하면 transaction/lock에 영향을 줄 수 있습니다. 이번 검사는 UPDATE·commit·rollback 호출을 모두 mock에서만 관찰했습니다. SQL의 조건, commit/close 호출 결과가 실제 원자성·유일성·DB 격리를 입증하지는 않습니다.

합성 카탈로그는 KOSPI=1/KOSDAQ=2, 산업10→거래소1/섹터100, 산업20→거래소2/섹터200입니다. 중복 산업 코드와 서로 다른 거래소에 같은 종목 코드도 포함합니다. 실제 회사 분류의 타당성은 검증하지 않습니다.

## 24개 메서드별 테스트

| 메서드 | 방법과 기대값 | 결과 |
|---|---|---|
| `test_normalization_and_optional_whitespace` | 따옴표/공백 제거, 9999→009999, qa.a→QA.A, 빈 선택값→None | PASS |
| `test_all_id_and_code_aliases_resolve` | target/일반/manual의 id·code 별칭 6개, 고유 산업10 선택; 입력없음→None | PASS |
| `test_explicit_id_wins_over_conflicting_code_observed` | id10과 모호한 code를 동시에 주면 id10 우선 | PASS: 현재 동작 |
| `test_unknown_malformed_cross_exchange_and_ambiguous_targets_reject` | 비숫자/없는 id/다른 거래소 id/없는 code/중복 code→ValueError | PASS |
| `test_whitespace_primary_alias_currently_shadows_valid_secondary_observed` | target id가 공백이고 secondary id10이 있어도 None | PASS: 현재 동작 |
| `test_utf8_bom_tsv_and_quoted_csv_use_actual_parser` | BOM TSV의 한글·header와 따옴표 CSV의 내부 쉼표 유지 | PASS |
| `test_early_cp949_decode_failure_falls_back_without_losing_korean` | 앞부분 CP949 한글→utf-8-sig/utf-8 실패 후 cp949 성공, 한글 보존 | PASS |
| `test_late_encoding_fallback_must_not_duplicate_previously_yielded_rows` | ASCII 400행 뒤 CP949 한글1행: 실제 decoding 재시도 후 401행 기대, 실제973행 | FAIL |
| `test_empty_file_currently_raises_index_error_before_db_observed` | 빈 파일에서 header 분리 중 IndexError | PASS: 현재 동작 |
| `test_catalog_loaders_preserve_duplicate_code_candidates_and_close` | 실제 catalog loader가 같은 code의 id11/12 후보 보존, cursor close | PASS |
| `test_update_sql_is_scoped_by_stock_exchange_and_preserves_existing_by_default` | UPDATE 인자 sector100/industry10/stock009999/exchange1, industry_id IS NULL 조건·commit0 | PASS |
| `test_allow_remap_removes_only_missing_guard_not_identity_scope` | allow-remap은 NULL 조건만 제거, stock/exchange 조건·bound 인자는 유지 | PASS |
| `test_dry_run_executes_updates_then_rolls_back_without_commit` | 유효1행 UPDATE1→rollback1/commit0, 요약 updated1 | PASS |
| `test_apply_commits_one_transaction_and_sets_industry_sector_pair` | 두 거래소의 유효2행→각 sector/industry 쌍, 최종 commit1/rollback0 | PASS |
| `test_blank_mapping_is_skipped_but_other_rows_apply` | 분류 미입력1행 건너뛰고 정상1행 UPDATE, skipped1 요약 | PASS |
| `test_duplicate_normalized_stock_exchange_is_rejected_before_second_update` | 9999/009999 같은 거래소→첫 UPDATE 후 두 번째 중복 거부, commit0 | PASS |
| `test_same_stock_code_on_two_exchanges_is_not_duplicate` | 같은 종목 코드의 KOSPI/KOSDAQ 행은 각각 UPDATE, dry-run rollback1 | PASS |
| `test_missing_required_or_unknown_exchange_rejected_before_update` | 종목/거래소 빈값·미등록 거래소→ValueError·UPDATE0·commit0 | PASS |
| `test_unknown_stock_zero_affected_rows_is_only_a_summary_observed` | 합성 affected0→예외 없이 commit, updated0 요약 | PASS: 현재 동작 |
| `test_main_after_late_validation_error_closes_without_explicit_rollback_observed` | 실제 main에서 정상행 다음 중복행→예외 전파·UPDATE1·commit0/rollback0·connection close1 | PASS: 호출 관측 |
| `test_actual_main_parser_sql_and_dry_run_flow` | 실제 main→TSV bytes/decoder/parser→카탈로그→industry code resolve→SQL 인자→rollback/close | PASS |
| `test_encoding_retry_duplicates_abort_actual_main_observed` | 같은 정상401행 CP949 fixture를 실제 main으로 처리하면 parser가 만든 중복 때문에 UPDATE 일부 후 중단·commit0/rollback0·close1 | PASS: 결함 영향 관측 |
| `test_cli_defaults_require_explicit_apply_and_remap` | 실제 argparse 기본 apply/remap false; 명시적 옵션을 주면 true·mapping 경로 유지 | PASS |
| `test_missing_file_is_rejected_before_connect` | 없는 경로→SystemExit·DB connector 호출0 | PASS |

## BATCH-MAPPING-ENCODING-001 — 인코딩 재시도에서 이미 반환한 행 중복

`read_mapping_rows`는 각 encoding으로 `yield from csv.DictReader`를 실행합니다. 파일 뒤쪽에서 UnicodeDecodeError가 발생하면 앞부분의 이미 반환된 행을 취소하지 않은 채 다음 encoding으로 파일 처음부터 다시 읽습니다.

fixture는 header, 각각 40개의 ASCII note 문자를 가진 고유 종목400행, 마지막 CP949 한글 행으로 구성합니다. 첫 decoding buffer보다 뒤에서 실패하므로 utf-8-sig와 utf-8의 일부 행이 먼저 반환됩니다. 최종 cp949 성공분401행에 앞선 실패 시도들의 일부 행이 추가되어973행이 됩니다. 중복 없이401행이어야 한다는 회귀는 실패로 유지합니다.

실제 main 연결 검사에서도 원본 입력에 중복이 없는데 `duplicate mapping row` 오류로 중단되는 것을 확인했습니다. 일부 UPDATE는 합성 connection에 호출됐고 commit은0, 명시적 rollback도0, connection close는1입니다. 실제 DB close의 rollback 동작이나 잠금 영향은 실행하지 않았으므로 단정하지 않습니다. 제품 코드 수정 또는 실제 수동 분류 적용은 하지 않았습니다.

## 기타 관측과 남은 범위

- 공백으로 채운 우선 별칭이 유효한 후순위 별칭을 가리고, 미등록 종목/기존 분류 때문에 UPDATE가0건이어도 오류 대신 요약으로 끝납니다. 이를 운영 정책 승인으로 해석하지 않습니다.
- 정상 dry-run의 rollback은 확인했지만, 중간 예외에서는 명시적 rollback 없이 connection close만 실행됩니다. 위 OHLCV의 연결 재사용 문제와 달리 이 main은 finally에서 연결을 닫으므로 같은 영향이라고 단정하지 않습니다.
- 실제 파일 경로 권한/OS encoding 차이, 실제 DB UPDATE·rollback·경합, 데이터 내용의 분류 정확성, 사용자 실수 시 복구는 남았습니다. 전체 배치·프로젝트 QA 완료가 아닙니다.
