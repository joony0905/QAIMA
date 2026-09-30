# 수동 매핑·종목 초기화·해외 차트의 격리 SQL 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03` 및 현재 미커밋 소스입니다. [검사 코드](sql_tool_cases.py), [실행기](run_isolated_sql.py), 공유 [DB 가드/결과 수집 코드](sql_cases.py)는 batch/tests 안에 있습니다. 제품과 기존 DB는 수정하지 않았습니다.

## 실제 실행과 대체 경계

[앞선 여섯 적재 경로의 실제 SQL31개](isolated-sql.md)에 이어 기존 [수동 매핑](manual-mapping.md), [종목 초기화](bootstrap-lookup.md), [해외 도구](overseas-tools.md)의 mock DB 경계를 새 외부 MySQL로 확장했습니다.

- 수동 매핑: 실제 디스크 CSV/TSV·decoder·DictReader·main/argparse·카탈로그 SELECT·분류 UPDATE·commit/rollback/close를 실행합니다.
- 종목 초기화: 실제 pandas CSV·main/argparse·분류 변환·거래소 seed·sector/industry/stock SQL·중복 갱신을 실행합니다. KIS client 생성/응답만 합성 객체로 대체해 외부 요청·토큰 파일을 사용하지 않습니다. 별도 build_kis_stock_info→stock 저장 사례도 있습니다.
- 해외 차트: 실제 main/argparse·JSON/CSV 파일 출력·industry_index 조회·OHLCV upsert·close를 실행합니다. client의 fetch_chart_full만 합성 CandleRow/raw 응답 객체로 대체합니다. 실제 원천 수집·기간 분할·거래소 규격을 검사한 결과는 아닙니다.
- DB Connection/Cursor/SQL은 실제 CMySQLConnection입니다. 동시 생성2개 사례만 cursor를 전달하는 wrapper가 첫 빈 SELECT 직후 barrier로 멈춥니다. 결과를 조작하지 않고 두 실제 연결이 INSERT 전에 모두 “없음”을 읽도록 순서를 고정합니다.

테스트 worker가 호출한 main 반환값을 검사하며 제품 CLI를 별도 프로세스로 띄운 종료코드 검사는 아닙니다. 실제 OS별 거래일·스케줄러·운영 동시 프로세스 부하·종목 분류의 업무 타당성은 별도입니다.

## 실행·격리·정리

```bash
cd /mnt/c/qaima
.venv_wsl/bin/python -B batch/tests/run_isolated_sql.py --suite tools
```

`--suite` 기본값 core는 앞선31개이며 이번 tools22개와 합산하거나 재실행으로 간주하지 않습니다. 새 tools 선택 기능과 공통 result 수집 함수만 tests 코드에 추가했습니다. 매번 새 `/tmp/qaima-qa-batch-*` MySQL datadir·loopback 임의 포트·QA schema/소유 표식을 만들고 원본 V1~V51 SQL51개를 적용합니다. 업무47+표식1테이블입니다. Flyway history 검사는 아닙니다.

MySQL8.0.46-0ubuntu0.24.04.4, Python3.12.3, mysql-connector-python9.7.0/CMySQLConnection 환경입니다. 기존 application/.env/접속 환경을 읽어 사용하지 않습니다. 소유 host/port/schema·@@datadir·marker 확인, 외부 HTTP/DB/socket·worker subprocess·.env 읽기·해당 tests 산출물 밖 쓰기 차단은 [공통 구성](isolated-sql.md)을 그대로 사용합니다.

fixture는 KOSPI/KOSDAQ의 동일 코드009998, bootstrap용009997, QA_SQL 접두어 sector/industry, QA_SQL_IDX입니다. DB를 테스트 전체 rollback으로 감싸지 않고, 별도 autocommit 연결에서 commit 여부를 확인합니다. 매 테스트 후 모든 테스트 연결을 rollback/close하고 업무6테이블·합성 종목/지수/분류·오류 trigger를 제거합니다. 부모 실행기도 독립 CLI로 최종0행을 확인하고 자신이 시작한 mysqld만 종료합니다. 거래소 seed 등 migration 카탈로그는 새 임시 DB 내부에만 있습니다.

## 22개 방법

아래 메서드명에는 실제 코드의 `test_` 접두어를 생략했습니다.

| 메서드 | 방법·독립 기대값 |
|---|---|
| mapping_update_is_uncommitted_until_explicit_commit_or_rollback | 실제 helper UPDATE1행. writer에서는 새 sector/industry, observer에서는NULL. rollback 후 두 거래소 모두NULL |
| mapping_real_tsv_main_dry_run_leaves_database_unchanged | 실제 TSV에 동일종목/서로 다른 거래소2행. 기본 dry-run main0, 종료 후 두 SQL 분류쌍NULL |
| mapping_apply_repeat_preserve_existing_and_explicit_remap | 명시 apply 두 번→거래소별 올바른 분류. 다른 산업으로 기본 apply하면 기존값 보존. allow-remap이면 KOSPI만 새 sector/industry, KOSDAQ 유지 |
| mapping_late_duplicate_error_closes_and_rolls_back_first_update | 009998/9998 정규화 중복. 첫 UPDATE 뒤 오류→main finally close. observer에는NULL, 다른 연결 UPDATE가 timeout 없이 가능 |
| mapping_real_sql_error_rolls_back_all_updates_on_main_close | 첫 KOSPI UPDATE 후 두번째 KOSDAQ UPDATE를 실제 trigger1644로 거절. 연결 close 후 두 분류쌍 모두NULL |
| mapping_cross_exchange_target_rejects_without_writing | KOSPI 종목에 KOSDAQ industry ID 지정→ValueError/쓰기 없음 |
| mapping_ambiguous_code_rejects_but_explicit_id_selects_one | 같은 거래소의 서로 다른 섹터에 같은 산업코드 생성. code 선택은 ambiguous 오류, 명시 industry ID는 정확한 쌍 저장 |
| mapping_unique_cp949_file_must_apply_without_decoder_duplicates | 고유401행/뒤쪽 CP949 한글 실제 파일. 중복 없이 main 성공·첫 종목 분류 저장 기대. decoder 중복으로 main 실패하는 기존 회귀 |
| bootstrap_canonical_sector_industry_replay_keeps_ids_and_single_rows | 실제 동일 identity 재호출→같은 ID·각1행. 기존 이름을 유지하는 현재 동작도 독립 SELECT로 확인 |
| bootstrap_stock_replay_keeps_id_and_updates_bound_metadata | 같은 거래소/코드 재실행→동일 LAST_INSERT_ID, 따옴표 회사명·분류쌍·ISIN·상장일 갱신. 다른 거래소의 같은 코드 포함 총2행 |
| bootstrap_invalid_first_listing_date_must_preserve_valid_alias_in_sql | 첫 날짜00000000/뒤 날짜20200102 합성 응답→실제 mapper/stock upsert. SQL listed_at=2020-01-02 기대 |
| bootstrap_failed_stock_upsert_must_release_existing_row_lock | 기존 stock upsert의 UPDATE를 trigger1644로 실패. 서버 상태·다른 연결 UPDATE(timeout1초) 확인. 원래 연결 rollback 후 다른 UPDATE 성공 대조 |
| bootstrap_concurrent_sector_creation_returns_one_canonical_id | 두 실제 연결의 첫 identity SELECT를 모두 빈 결과까지 진행 후 INSERT. DB1행·두 호출 모두 같은 ID 기대 |
| bootstrap_concurrent_industry_creation_returns_one_canonical_id | 같은 sector까지 포함한 industry identity에 위 동시 생성 수행. DB1행·두 호출 동일 ID 기대 |
| bootstrap_real_csv_main_replay_saves_one_stock_and_classification | 실제 BOM CSV009997·합성 KIS 응답→main 두 번. 실제 stock1행·유효 상장일·sector/industry 코드 연결 대조. skip-backfill 명시 |
| overseas_daily_main_replay_sql_and_real_output_files | 같은 일봉123.456789로 main 두 번→freq4/SQL1행. 실제 JSON/UTF-8 BOM CSV 파일을 다시 읽어 소수 문자열 확인 |
| overseas_weekly_main_must_not_overwrite_existing_daily | 기존 같은 날짜 일봉111 저장→period W의222. 기대는freq4=111 보존·freq5=222 별도행 |
| overseas_monthly_main_must_not_overwrite_existing_daily | 같은 구성에period M, 기대freq4=111·freq6=222 |
| overseas_failed_repository_must_release_row_lock | 같은 executemany의 기존111→222와 새날짜−999를 trigger로 거절. Repo 반환 뒤 다른 연결 UPDATE가 정상이어야 함. 명시 rollback 후 성공 대조 |
| overseas_main_sql_failure_close_preserves_daily_and_releases_lock | 동일 SQL 오류를 실제 main으로 실행→return1/finally close. 기존111 유지, 다른 연결444 UPDATE 성공 |
| overseas_nonfinite_mysql_outcomes_observed | NaN/±Infinity CandleRow를 실제 Repository로 전송→각1054/저장0행. parser 정상화 판정은 아님 |
| overseas_missing_index_master_returns_one_and_does_not_write | 실제 카탈로그에 없는 저장코드→main1·OHLCV0행 |

## 최초 결과

`run-sy8c244c`: **22개14 PASS/8 FAIL**, errors/skipped0, unittest8.972초, exit1. fixture 오류는 없었으며 실패는 기존 항목6개 회귀와 새 동시 생성2개 회귀입니다. 입력65개 실행 중 변경0, 업무6테이블·QA stock/index/sector/industry/trigger 최종0, MySQL exit0/mysqlStopped true입니다. worker 허용 DB 연결93, 금지된 network/DB/HTTP/subprocess/.env/범위 밖 쓰기 시도는 모두0입니다.

이번8개 실패를8건의 신규 결함으로 세지 않습니다. 기존 BATCH-MAPPING-ENCODING-001, BATCH-BOOTSTRAP-DATE-001, BATCH-BOOTSTRAP-ROLLBACK-001, BATCH-OVERSEAS-ROLLBACK-001은 각1개, BATCH-OVERSEAS-FREQ-001은 W/M2개입니다. 신규 BATCH-BOOTSTRAP-RACE-001은 sector/industry2개입니다. 기대 실패/skip으로 바꾸지 않았습니다.

## 최종 재현·정리

동시 생성 오류 뒤 조회/rollback/재호출 관측을 보강한 `run-o5krlp_f`도 **22개14 PASS/8 FAIL**, errors/skipped0, unittest7.853초, exit1입니다. 실패 메서드는 위 encoding1·date1·bootstrap lock1·sector/industry race2·overseas lock1·W/M2이고 나머지14개는 PASS입니다. 첫 실행과 중복 집계하지 않습니다. 이 시간은 테스트 worker 소요시간이며 DB 준비와 migration을 포함한 성능 측정값이 아닙니다.

| 실행 | 입력65개 변경 | 업무6테이블·QA stock/index/sector/industry/trigger | DB 허용 연결 | 금지된 IO 시도 | 종료 |
|---|---:|---|---:|---|---|
| run-sy8c244c | 0 | 모두0 | 93 | 모두0 | MySQL exit0/stopped true |
| run-o5krlp_f | 0 | 모두0 | 93 | 모두0 | MySQL exit0/stopped true |

tests.log·cases.json·summary.json과 실제 CSV/TSV/JSON 출력은 `batch/tests/.runtime/isolated-sql/`의 해당 run에 보존했습니다. 프로젝트 밖 임시 datadir/log는 남기고 서버 프로세스는 종료했습니다. tests 밖924개 코드/설정/문서 감사는 변경/소실/새 파일0이며 `tools-source-audit.json`에 있습니다. 의존성/빌드/venv/agent/runtime 제외 가시 파일 비교입니다.

문서49개 대상 로컬 링크/설정 일부 비밀값 검사2개도 PASS입니다. 전체 보안 감사는 아니며 로그는 `batch/tests/.runtime/isolated-sql/tools-documentation.log`입니다.

```bash
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

## 발견사항과 실제 영향

### BATCH-BOOTSTRAP-RACE-001 — 동시 분류 생성의 중복 키 복구 실패

StockMasterBootstrapRepository.upsert_sector/upsert_industry는 먼저 일반 SELECT로 ID를 찾고 없으면 INSERT합니다. IntegrityError를 받으면 같은 identity를 다시 SELECT해 복구하려고 합니다. 이전 mock 검사는 두번째 SELECT가 ID를 반환하도록 설정돼 있었습니다.

이번에는 두 실제 연결에서 첫 SELECT가 모두 없음인 상태를 barrier로 고정했습니다. 실제 REPEATABLE-READ에서 sector/industry 모두 **한 호출은 ID 성공, 다른 호출은1062**, SQL에는1행만 저장됐습니다. 데이터가 중복 생성됐다는 결함이 아니라, 이미 생성된 identity를 반환하도록 작성된 복구 분기가 동시 호출 하나를 실패시키는 문제입니다.

재실행의 실패한 동일 연결에서 canonical row 조회는 **rollback 전0행**, rollback 후 제품 메서드 재호출은 **이미 저장된 같은 ID**였습니다. industry는 성공/저장/재호출 ID7, sector는10이었습니다. 제품 catch의 재조회가 첫 조회와 같은 transaction 안에 있으며, 이번 REPEATABLE-READ에서 새 identity를 보지 못하는 관측과 일치합니다. 다른 격리 수준은 검사하지 않았습니다.

이 명시 rollback/재호출은 테스트의 진단이며 제품이 자동 복구했다는 의미가 아닙니다. winner의 순서나 자동증가 ID 자체는 고정 기대값이 아니고, 두 최초 호출이 저장된 단일 ID를 반환해야 한다는 것이 회귀 기준입니다. 여러 OS 프로세스나 운영 발생률은 측정하지 않았습니다.

### BATCH-OVERSEAS-FREQ-001 — 주봉·월봉이 기존 일봉을 실제로 덮음

각 사례에서 `(index_id, 2026-01-01, freq4)`에 종가111을 실제 commit한 뒤, period W/M과 종가222로 main을 실행했습니다. 둘 다 return0이지만 SQL은 **freq4/222 한 행**이고 주봉5·월봉6 행은 없습니다. 이전 SQL tuple 관측에서 가능성으로 남긴 일봉 덮어쓰기를 실제 고유 키/upsert로 확인한 증분입니다. Spring Freq의 ONE_D=4/ONE_W=5/ONE_M=6을 기대값으로 사용합니다. Y 주기 표현은 임의로 정의하지 않았습니다.

### 기존 rollback 두 항목 — Repository와 main 종료를 구분

bootstrap stock upsert와 overseas Repository는 오류 뒤 `SELECT 1`로 서버 상태를 갱신하면 transaction=true, 별도 연결 UPDATE는1205입니다. 원래 연결 rollback 후 다른 쓰기는 성공합니다. 오류1644는 실제 소유 DB trigger로 주입했습니다.

해외 도구는 실제 main 경로로 같은 오류를 내면 finally의 connection.close가 실행돼 기존 일봉111이 유지되고 다른 연결 UPDATE444도 성공했습니다. Repository를 직접 호출한 잠금 유지 결과를 main 종료 뒤에도 잠금이 남는다는 주장으로 확대하지 않습니다. 수동 매핑 main의 중간 중복/SQL 오류도 close 후 모든 미커밋 변경이 되돌아가는 대조군입니다.

### 기존 encoding/date 항목 — 실제 파일·저장값

고유401행 CP949 mapping 파일은 decoder 재시도로973행이 됐습니다. 실제 main은 line288의009998 중복으로 실패했고 첫 종목의 미커밋 UPDATE도 close 후 rollback돼 두 거래소의 분류가NULL로 남았습니다. 원본 입력의 중복이나 일부 변경의 영구 저장으로 기록하지 않습니다.

상장일은 첫 후보00000000을 선택해 파싱한 결과None이 실제 stock.listed_at에NULL로 저장됐습니다. 뒤 후보20200102를 사용해야 한다는 기존 회귀는 실패입니다. 실제 제공자가 이 필드 조합을 보내는 빈도는 별도입니다.

## 남은 경계

세 도구의 선택한 DB 경계를 확장한 결과입니다. bootstrap backfill의 거래소/동명·기존 sector 불일치, 전체 입력/파일 권한·큰 CSV, 다른 격리 수준·여러 backend/배치 프로세스, 네트워크 단절/프로세스 종료, 실제 KIS 응답/분류 정확도/시장 주기·시간대, token 파일과 Spring scheduler는 남습니다. [전체 진행 기록](../../tests/QA_PROGRESS_2026-09-28.md)을 이어갑니다.
