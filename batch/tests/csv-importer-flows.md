# 재무·공매도 CSV 실제 파서·적재 흐름 검증

검증일: 2026-09-25. [test_csv_importer_flows.py](test_csv_importer_flows.py)의 추가 17개는 **14 PASS·3개 메서드 FAIL**(10개 실패 subcase)입니다. 기존 [CSV 변환·flush 10개](README.md)와 별개입니다. 제품 소스는 수정하지 않았습니다.

## 재현 명령과 경계

저장소 루트, WSL Python에서 실행합니다.

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 \
  .venv_wsl/bin/python -B batch/tests/run_guarded.py
```

추가 시점 전체 결과: 159개 **144 PASS·15개 메서드 FAIL**, 실패항목33, ERROR/SKIP0, 종료값1. 두 번 실행에서 동일 결과이며 unittest 측정0.639초/0.627초였습니다. socket/subprocess/dotenv 읽기/미대체 HTTP/미대체 DB 카운터 모두0, 유료 호출0입니다.

대상: [import_financials.py](../jobs/import_financials.py), [import_short_selling.py](../jobs/import_short_selling.py). 실제 `main → argparse → resolve_csv_paths → connect_db → stock catalog → read_csv_rows → TextIOWrapper/csv.DictReader → to_params → flush_batch`를 연결했습니다. MySQL connector/connection/cursor와 Path.open/exists만 합성 경계로 대체합니다. parser/to_params/flush/main은 대체하지 않습니다. Path.open은 합성 bytes를 가진 실제 TextIOWrapper를 돌려주며 BOM·CP949 디코더와 인용부호/여러 줄 CSV 파서가 실제 실행됩니다. SQL tuple은 제품이 batch.clear()하기 전에 복사하여 검증합니다. 실제 디스크 파일·DB·자식 CLI 프로세스는 사용하지 않았습니다.

## 메서드별 방법·결과

클래스는 `CsvImporterFlowTests`입니다. 공통 검사는 두 importer 모두에 subTest로 적용합니다.

| 메서드 | 수행·독립 기대값 | 결과 |
|---|---|---|
| `test_actual_decoder_bom_cp949_quotes_and_multiline` | BOM/CP949 bytes, 한글·쉼표·따옴표·필드 내부 개행을 actual reader로 원본 dict와 비교. CP949 fallback encoding 순서3회 | PASS |
| `test_late_decode_fallback_must_not_reyield_rows` | ASCII400행(40자 note)+뒤 CP949 한글1행. 고유401행 기대, 두 importer 모두1083행 반환 | **FAIL×2** |
| `test_actual_main_cli_paths_csv_sql_commit_and_close` | 실제 argv --csv/--batch-size2부터 main 실행. 합성stock id7·재무123.45/공매도 인용된1,234.5→Decimal1234.5·autocommitFalse·commit1·cursorclose2·connclose1 | PASS |
| `test_actual_csv_mixed_rows_skip_bad_unknown_blank_and_flush_tail` | 빈행/미등록/날짜오류/정상3행을 실제 CSV로 입력. batch2+잔여1, SQL stockId 순서7/8/7, commit2 | PASS |
| `test_multiple_files_flush_separately` | a/b 두 파일 각1행·batch2. 파일별 잔여flush로[[7],[8]], commit2 | PASS |
| `test_missing_required_header_must_not_finish_as_success` | wrong_column만 있는 파일. write0/close1이지만 실패가 호출자에게 전달되지 않고 None 정상 반환 | **FAIL×2** |
| `test_nonfinite_amounts_must_not_be_submitted` | 실제CSV의 revenue/short_volume_total에NaN/Infinity/-Infinity. 누락 또는 거부 기대와 달리 executemany의Decimal 매개변수에 남음 | **FAIL×6** |
| `test_stock_catalog_normalizes_and_closes_cursor` | nullid/nullcode 제외,5930→005930/abc→ABC와id, cursor.close1 | PASS |
| `test_failed_catalog_load_closes_connection_without_write` | 실제 catalog.fetchall 예외. 전파·connclose1·executemany0 | PASS |
| `test_transient_flush_error_retries_retained_batch_observed` | batch1, 첫SQL오류 후 다음행 성공. 제출[[7],[7,8]], rollback1/commit1. 실패 batch가 비워지지 않는 현재 동작 | PASS(관측) |
| `test_persistent_flush_failure_retries_then_propagates_at_tail` | batch1/3행/항상SQL오류. 제출크기1→2→3→마지막3, rollback4/commit0/close1, 잔여flush에서 예외 전파 | PASS(관측) |
| `test_later_flush_failure_keeps_prior_commit_and_closes` | 첫chunk성공, 두번째/잔여재시도실패. commit1·rollback2·close1·예외. 파일전체원자성 아님 | PASS |
| `test_overflow_fields_on_blank_row_abort_before_row_handler_observed` | stock_code 헤더1개, `,unexpected` 행의 여분필드가list. 빈행판정의.strip에서AttributeError, connclose1/commit0 | PASS(관측) |
| `test_paths_deduplicate_skip_missing_and_no_input_errors` | a.csv 중복제거/없는경로skip, 발견경로없으면SystemExit | PASS |
| `test_empty_path_item_currently_resolves_project_directory_observed` | 빈 --csv 항목이 Path('.')가 되어 프로젝트디렉터리를 선택하는 현재 동작. 디렉터리 실제읽기 없음 | PASS(관측) |
| `test_financial_interactive_ratio_yes_no_retry_and_eof` | per1000000·시총100/순익10. 잘못된답 재질문→y이면10.0000, n이면None, EOF면예외. 실제사용자입력 없음 | PASS |
| `test_financial_fractional_integer_truncation_observed` | version1.9/fiscal_year2025.9는 정수1/2025로 잘림, 실제tuple에반영. 허용정책 판정 아님 | PASS(관측) |

## 발견사항

**BATCH-CSV-ENCODING-001:** 두 reader의 yield-from 도중 디코딩 오류가 나면 이미 반환한 앞부분을 되돌리지 않고 다음 encoding으로 처음부터 읽습니다. 기대401/실제1083은 위 fixture에서 직접 측정한 값입니다. 수동 매핑의973행 fixture와 길이·구조가 다릅니다. 데이터베이스 중복행 생성/영속화를 입증한 것은 아닙니다.

**BATCH-CSV-SCHEMA-001:** 필수 열 없는 전체 입력이 행별 예외로만 기록되고 import_csvs는 정상 반환합니다. 호출자가 실패를 구분할 결과/예외를 받지 못합니다. 실제 CLI 종료 프로세스는 미실행이며 main이 import_csvs를 그대로 호출하는 코드와 반환 경계를 구분해 기록합니다.

**BATCH-CSV-NONFINITE-001:** 재무 금액·공매도 수량의 Decimal 생성에 유한성 검사가 없어서 NaN/±Infinity가 SQL 전송 경계에 도달합니다. 실제 MySQL 컬럼이 수용/거부하는지 확인한 결과는 아닙니다.

추가 관측: SQL 오류를 행 오류와 같은 catch에서 처리해 batch_size1인데도 유지된 batch가1→2→3으로 커지고, EOF 뒤 잔여flush에서야 실패가 전파됩니다. 이 동작의 대규모 시간·메모리 영향은 측정하지 않았습니다. 빈행판정의 여분열 list 처리, 빈경로의 디렉터리 선택, 소수 정수필드 자르기도 정상 정책이라고 승인한 것이 아니라 현재 동작을 고정한 검사입니다.

## 남은 범위

격리 MySQL에서 unique/FK/DECIMAL/날짜컬럼·두 번 적재 멱등성·동시성·commit 실패의 실제 영속성, 수백만 행과 실패 누적 시 메모리, 실제 파일권한/터미널/프로세스종료, Windows 배치 운영환경 여부, 원천파일 스키마·금액/비율단위 검증이 남습니다. Python11개 도구의 main이 합성 경계에서 실행됐다는 사실은 전체 연동 검증 완료를 뜻하지 않습니다.
