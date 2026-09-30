# 격리 MySQL 배치 SQL·재실행·장애 복구 검증

검증일: 2026-09-28. 현재 `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03`와 미커밋 소스를 대상으로 합니다. [실행기](run_isolated_sql.py)와 [검사 코드](sql_cases.py)는 batch/tests 안에 있습니다. 제품 Python·migration·운영 설정은 수정하지 않았습니다.

## 이전 검사에서 확장한 부분

이전 [OHLCV25개](ohlcv-jobs.md), [수급30개](investor-jobs.md), [CSV17개](csv-importer-flows.md)는 SQL 조립·commit/rollback 호출을 mock으로 확인했습니다. 이번에는 원본 V1~V51 SQL로 만든 새 MySQL8.0.46에서 제품 Repository/변환/flush_batch를 실행합니다. 제품이 commit한 결과는 별도 autocommit 연결로 읽으며 테스트 전체를 rollback으로 감싸지 않습니다.

| 이름 | 제품 코드 | 실제 테이블·주요 대조 컬럼 |
|---|---|---|
| price | load_price_ohlcv.PriceOhlcvBootstrapRepository | price_ohlcv.close DECIMAL(18,6), stock_id/ts/freq PK |
| index | load_index_ohlcv.IndustryIndexOhlcvRepository | industry_index_ohlcv.close DECIMAL(18,6), index_id/ts/freq PK |
| stock | load_stock_investor_flow.to_params/upsert_rows | stock_investor_flow.close_price DECIMAL(20,4), stock/date/source 고유 키 |
| market | load_market_investor_flow.to_params/upsert_rows | market_investor_flow.foreign_net_buy_qty DECIMAL(24,0), market/industry/date/source 고유 키 |
| financial | import_financials.to_params/flush_batch/main | financial.revenue DECIMAL(20,2), fiscal period 고유 키·created/updated·ratio |
| shorts | import_short_selling.to_params/flush_batch/main | short_selling.short_volume_total **V35 이후 DECIMAL(24,6)**, stock/date 고유 키·ratio |

CSV 두 경로는 실제 `main → argparse → 경로 확인 → connect_db → stock catalog SQL → 디스크 UTF-8 BOM 파일 → csv.DictReader → to_params → flush_batch → 실제 MySQL`까지 연결합니다. fixture CSV도 tests/.runtime 아래에만 만듭니다. sys.argv만 테스트 인자로 대체합니다. 제품 CLI를 별도 프로세스로 띄워 종료코드를 검사한 것은 아닙니다. HTTP·KIS·토큰 발급/저장은 호출하지 않습니다. 가격/지수는 합성 CandleRow, 수급은 합성 provider dict에서 출발하므로 실제 제공자 응답 검증과 구분합니다.

모든 날짜는 합성2026년1월이고 가격 당일 미확정 필터를 통과하는 과거 날짜입니다. 시장 소수 수량의 저장 반올림은 현재 DDL/driver 동작 관측이며 원천 단위·업무 정책 적합성 판정은 아닙니다. Spring scheduler의 시간대·휴장일·자동 재시도·중복 실행 방지도 이번 범위 밖입니다.

## 격리·가드·실행 방법

MySQL은 [이전 준비 명령](../../tests/isolated-http-jpa.md)의 `/tmp/qaima-qa-mysql-20260928/root/usr` 추출본을 재사용합니다. 시스템 패키지를 설치하거나 기존 MySQL을 사용하지 않습니다. 매번 새 `/tmp/qaima-qa-batch-*` datadir·loopback 임의 포트·`qaima_qa_*` 스키마·소유 표식을 만듭니다. `--no-defaults`, mysqlx off, local-infile off, secure-file-priv NULL, binary log off로 시작합니다.

부모 실행기가 V1~V51 SQL51개를 숫자 순서로 적용합니다. Flyway history/checksum 검사와 구분합니다. 실제 `@@port`, `@@datadir`, `DATABASE()`, qa_test_owner를 반복 확인합니다. 업무47개+소유표식1개 테이블을 생성합니다.

자식 worker에는 새 DB 접속값만 전달합니다. 기존 QA/DB/MySQL/KIS/Spring/배치 환경과 key/secret/password/token/proxy 변수를 제거하고 `PYTHON_DOTENV_DISABLED=1`, bytecode 쓰기 금지를 설정합니다. worker는 다음을 강제합니다.

- mysql.connector.connect를 접속 허용 검사로 감쌉니다. 실제 연결은 원래 connector가 수행하며 Connection/Cursor/SQL은 mock하지 않습니다. 소유 host/port/schema/root/빈 임시 password 조합만 허용하고 연결 직후 SQL 소유 가드를 실행합니다. C extension의 Python socket audit 우회 가능성도 이 진입점 검사로 제한합니다.
- Python socket은 해당 loopback endpoint만 허용합니다. requests transport·자식 프로세스·`.env` 읽기와 해당 실행의 tests 산출물 폴더 밖 쓰기는 예외와 카운터로 차단합니다.
- fixture는 종목009998/합성 SQL 기업, QA_SQL_IDX/합성 SQL 지수입니다. SQL 오류는 소유 스키마의 QA BEFORE INSERT trigger가 값−999에 `SIGNAL SQLSTATE '45000'`을 발생시키는 방식입니다. 실제 statement 오류1644를 확인합니다.
- 각 테스트 후 남은 transaction rollback·연결 close → 소유 가드 → trigger/업무6테이블/합성 종목·지수 삭제 → 업무0행 단언을 실행합니다. 부모도 CLI로 최종0행을 확인하고 자신이 시작한 mysqld만 SIGTERM으로 종료합니다.

```bash
cd /mnt/c/qaima
.venv_wsl/bin/python -B batch/tests/run_isolated_sql.py
```

이 실행은 과거 guarded159개를 재실행하지 않습니다. `sql_cases.py`는 의도적으로 test_ 접두어가 없어 mock 전용 discover에 섞이지 않습니다. 부모는 worker의 로그·cases.json을 수집하고, 실패가 있으면 exit1을 유지합니다. 모든 산출물은 `batch/tests/.runtime/isolated-sql/run-*/`입니다. `summary.json`에는 DB 버전/sql_mode·명령·64개 입력 SHA-256·정리/종료·worker 결과가 들어갑니다.

## 31개 검사 방법

`<job>`은 위 표의 price/index/stock/market/financial/shorts입니다. 코드는 각 조합을 별도 unittest 메서드로 등록하므로 결과 이름은 `test_price_upsert_replay` 같은 형태입니다. 비유한 값3개는 해당 메서드 안의 subTest이고 별도 테스트 수로 합산하지 않습니다.

| 메서드 | 수 | 입력·독립 SQL 기대값 |
|---|---:|---|
| test_&lt;job&gt;_upsert_replay | 6 | 111.25 저장→동일 입력 재실행→222.5 갱신. 실제1행 유지, 최종 값 대조. market은 정수 scale이라111→223, shorts는 V35 scale6이라 소수 유지. timestamp가 있는4개는 created_at 유지/updated_at 갱신. price 반환값1/0/2와 다른 job의 입력건수 반환을 기록 |
| test_&lt;job&gt;_sql_failure_recovery | 6 | 기존111 commit. 같은 executemany에 기존행222 갱신+새날짜−999 거절 → 실제 오류1644, observer에는111만 유지. 오류 뒤 transaction 종료 기대. 같은 제품 연결로 다른 날짜333 적재 후111/333만 존재하는지 대조 |
| test_&lt;job&gt;_nonfinite_driver_observed | 6 | NaN/Infinity/−Infinity를 제품 변환 또는 CandleRow로 전달. 실제 connector/MySQL의 오류번호·저장값을 기록하고 비유한 값이 영속되지 않았는지 확인. parser가 올바르게 거른다는 회귀가 아닌 driver 결과 관측 |
| test_&lt;job&gt;_failed_batch_must_release_row_lock | 6 | 기존111→같은 statement의 기존행222+새날짜−999 실패. 다른 실제 연결에서 같은행444 갱신, lock wait timeout1초. 오류 없이 완료 기대. 이후 원래 연결에 명시적 rollback을 하고555 갱신 성공을 확인. rollback하는4개 job을 대조군으로 사용 |
| test_financial_real_csv_main_replay / test_shorts_real_csv_main_replay | 2 | 실제 BOM CSV: 정상1234.5/미등록종목/잘못된 날짜/정상 중복. batch2로 main 두 번 → 실제1행·금액/수량1234.5, 한글 source 보존, ratio0.1234/0.125679 |
| test_financial_persistent_sql_error_and_corrected_replay / test_shorts_persistent_sql_error_and_corrected_replay | 2 | CSV111/−999/333, batch1 → 앞111은 commit, poison 이후 마지막 flush에서 오류1644 전파. SQL에는111만 남음. trigger 제거 후 수정된111/222/333 CSV 재실행 →3행, 앞행 중복 없음 |
| test_price_target_query_frequency_and_latest_are_backed_by_sql | 1 | only_missing 실제 조회에 신규종목 포함. 주봉freq5만 저장해도 일봉 누락 대상 유지. 일봉freq4 저장 후 대상 제외. freq별 두 PK행·최신ts를 SQL과 비교 |
| test_index_first_chunk_remains_committed_after_later_chunk_failure | 1 | 테스트에서 chunk 크기를1로 설정.111 commit 후−999 거절로 세번째333 미실행. 명시 rollback 후 올바른3행 재실행 →3행. 실제 only_missing 제외·latest ts 대조 |
| test_financial_same_fiscal_period_updates_version_and_report_date | 1 | 같은 stock/2026/Q1을 version1→2, report_date1/1→4/1, revenue111→222로 재실행 →같은 fiscal key1행 갱신 |

모든 기대값은 제품의 반환 건수를 저장행 수로 간주하지 않고 별도 연결의 SELECT로 확인합니다. 하나의 executemany statement가 실패할 때 기존 갱신이 rollback되는 것과 여러 commit된 chunk 전체를 되돌리는 것은 다릅니다. 후자는 제품의 현재 commit 경계를 그대로 관측합니다.

## 최초 실행과 테스트 보정

최초 `run-9u8fut73`: **27개23 PASS/4 FAIL**, errors/skipped0,7.454초.31개 최종 구성에 포함된 rollback 대조군 잠금4개는 이 실행에 없었습니다. 실패 중2개는 공매도 수량을 V11의 정수 scale로 기대한 테스트 오류입니다. V35에서 DECIMAL(24,6)으로 변경된 DDL과 실제 저장값111.250000/1234.500000을 확인해 기대값을 고쳤습니다. 제품의 반올림 결함으로 세지 않습니다.

나머지2개는 가격/지수 SQL 실패 뒤 다른 연결의 쓰기가1205로 실패한 회귀입니다. rollback 전 driver의 `in_transaction` 속성은 false였지만 실제 잠금은 남았습니다. 최초 SQL 복구6개 중 가격/지수의 transaction 종료 단언은 이 캐시값만 읽어 통과했으므로 정상 종료 증거로 사용하지 않습니다.

설치된 connector의 `CMySQLConnection.in_transaction` 구현은 `_server_status` flag를 읽습니다. 보정한 검사는 실패 뒤 `SELECT 1`의 실제 서버 응답으로 상태를 갱신한 값도 기록합니다. 동일 SELECT가 정상 commit 직후에는 transaction을 열지 않는 대조 단언을 먼저 수행합니다. 다른 연결의 실제1205·명시적 rollback 후 쓰기 성공을 함께 사용해 상태 속성만으로 결론내리지 않습니다.

최초 실행의 실제 SQL에서는 NaN/±Infinity 18조합 모두1054로 거절되고0행이었습니다. 두 CSV poison 사례는 이전 commit1행을 보존했고 수정본 재실행 후3행을 확인했습니다. 실패 statement 중간 값222가 다음 성공 commit에 섞여 영속되는 현상은 관측하지 않았습니다.

## 최종 결과·정리 증거

보정 후 `run-ltpuu307`: **31개27 PASS/4 FAIL**, errors/skipped0, unittest7.558초, worker/부모 exit1입니다. 실패는 `test_price_sql_failure_recovery`, `test_index_sql_failure_recovery`, `test_price_failed_batch_must_release_row_lock`, `test_index_failed_batch_must_release_row_lock`입니다. 기존 BATCH-OHLCV-ROLLBACK-001 한 항목의 실제 transaction/잠금 증거4개이며 신규 결함4건으로 세지 않습니다. 처음27개와 중복 합산하지 않습니다.

환경은 WSL Python3.12.3, mysql-connector-python9.7.0, 실제 연결 클래스 CMySQLConnection, MySQL8.0.46-0ubuntu0.24.04.4입니다. 서버 sql_mode는 ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_IN_DATE, NO_ZERO_DATE, ERROR_FOR_DIVISION_BY_ZERO, NO_ENGINE_SUBSTITUTION입니다. 위 실행시간은 unittest 소요시간이고 DB 초기화/migration을 포함한 처리량 벤치마크는 아닙니다.

| 항목 | 최초 run-9u8fut73 | 최종 run-ltpuu307 |
|---|---|---|
| 테스트 결과 | 27개23 PASS/4 FAIL | 31개27 PASS/4 FAIL |
| 소유 DB 허용 연결 | 91 | 107 |
| 외부 network/DB/HTTP, subprocess, .env 읽기, 범위 밖 쓰기 차단시도 | 모두0 | 모두0 |
| 입력64개 실행 중 해시 변경 | 0 | 0 |
| 업무6테이블 최종 행 | 모두0 | 모두0 |
| 합성 stock/index/QA trigger 잔여 | 각각0 | 각각0 |
| MySQL 종료 | exit0/mysqlStopped true | exit0/mysqlStopped true |

실행별 tests.log·cases.json·summary.json을 보존했습니다. 초기화와 서버 로그/datadir는 프로젝트 밖 실행 전용 디렉터리에 남겼고 프로세스는 종료했습니다. 부모는 최종 독립 CLI로 업무6테이블/QA stock/QA index/QA trigger를 확인했습니다. 기존 앱 DB·Redis·외부 HTTP는 사용하지 않았습니다.

최종6개 비유한 값 관측(각3입력)도 **모두1054로 거절·저장0행**입니다. parser의 기존 비유한 값 수용 결함을 해결한 결과가 아닙니다. 금액/수량이 DB에 비유한 값으로 저장된 것으로 확대하지 않습니다. 두 실제 CSV main 재실행은1행 유지·한글 source·소수 정밀도/ratio까지 통과했습니다. CSV 실패 후 보정 재실행은 자동 재시도가 아닌 테스트가 수정본을 다시 실행한 것입니다.

별도 tests 밖924개 코드/설정/문서 비교는 변경·소실0/새 파일0입니다. `batch/tests/.runtime/isolated-sql/source-audit.json`에 기록했습니다. 의존성/빌드/venv/agent/runtime을 제외한 가시 파일 감사이며 전체 파일 시스템 감사가 아닙니다.

문서48개 대상 로컬 링크/설정 일부 비밀값 검사2개도 PASS입니다. 전체 보안 감사는 아니며 로그는 `batch/tests/.runtime/isolated-sql/documentation.log`입니다.

```bash
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

## 기존 BATCH-OHLCV-ROLLBACK-001의 잠금 영향

[기존 발견사항](../../docs/findings.md)의 rollback 호출 누락을 실제 SQL로 확장했습니다. 가격/지수 Repository는 `executemany → commit`을 try에, cursor.close만 finally에 두고 오류의 connection.rollback은 호출하지 않습니다. 실패한 statement의 데이터 수정이 되돌아가더라도 transaction 잠금은 남을 수 있는지 검사했습니다.

두 실행 모두 가격/지수에서 **다른 연결 UPDATE → MySQL1205**, 원래 연결에 명시적 rollback 후 같은 대상 UPDATE555 성공을 확인했습니다. 최종 실행에서는 서버 상태를 갱신하는 SELECT 후 `in_transaction=true`였고, rollback하는 stock/market/financial/shorts 대조군은 false·다른 연결 UPDATE 정상 완료였습니다.

| 최종 경로 | 오류 직후 캐시 속성 | 서버 응답 갱신 후 | 다른 연결 UPDATE | 명시 rollback 후 |
|---|---|---|---|---|
| price/index | false | true | 1205 lock wait timeout | 555 저장 성공 |
| stock/market/financial/shorts | false | false | 444 저장 성공 | 555 저장 성공 |

별도 복구 사례에서 같은 제품 연결의 다음 정상 적재는111/333만 commit했습니다. 오류 중간 갱신222가 영속되는 오염은 관측하지 않았습니다. 확인된 영향은 실패 후 정리되지 않은 transaction과 잠금입니다. 실제 동시 프로세스의 전체 배치 운용이나 운영 잠금 지속시간을 측정한 것은 아닙니다. 한 테스트 worker의 독립된 실제 연결 두 개와 서버 lock wait timeout1초를 사용했습니다. 제품은 수정하지 않고 실패를 유지합니다.

## 남은 범위

여섯 저장 경로의 실제 SQL 검증이며 배치 전체 완료가 아닙니다. KIS HTTP/응답 coverage·원천 단위·실제 거래일, Spring scheduler/시간대/휴장일, 여러 배치 프로세스·토큰 파일 경합, 연결 단절/프로세스 종료·재시도 자동화, 수동 산업 매핑·종목 bootstrap·해외 차트의 실제 DB 경계는 남습니다. 전체 QA 진행은 [최신 기록](../../tests/QA_PROGRESS_2026-09-28.md)을 따릅니다.
