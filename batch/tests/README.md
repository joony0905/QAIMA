# 배치 테스트 기록

2026-09-28 증분: [격리 MySQL 실제 SQL 검증](isolated-sql.md)은 **31개27 PASS/4 FAIL**입니다. 여섯 적재 경로의 실제 upsert·재실행·SQL 오류 복구와 재무/공매도 디스크 CSV main을 검사했습니다. 기존 가격/산업지수 rollback 누락으로 transaction/잠금이 남아 다른 연결 UPDATE가1205로 실패하는 것을 재현했습니다. 방법·fixture 경계·최초 기대값 보정·로그·DB 정리는 상세 문서에 있습니다. 아래159개는 이전 mock 기반 결과이며 이번31개와 재실행으로 혼동하지 않습니다.

```bash
.venv_wsl/bin/python -B batch/tests/run_isolated_sql.py
```

후속 [수동 매핑·종목 초기화·해외 차트 SQL](isolated-sql-tools.md)은 **22개14 PASS/8 FAIL**입니다. 기존5개 결함의6개 회귀를 실제 파일/DB에서 확인했고, 새 `BATCH-BOOTSTRAP-RACE-001`은 sector/industry 동시 생성 중 한 호출이1062로 실패하는2개 회귀입니다. 해외 W/M은 기존 일봉111을222로 실제 덮었습니다. 수동 매핑의 dry-run과 중간 오류 시 main close rollback, 해외 main 오류 시 잠금 해제는 통과했습니다. 아래 명령은 새 DB를 준비하며 core31개를 다시 실행하지 않습니다.

```bash
.venv_wsl/bin/python -B batch/tests/run_isolated_sql.py --suite tools
```

검증일: 2026-09-25. WSL Python 3.12.3에서 고정 fixture로 실행했습니다. 배치의 실제 운영 OS·스케줄러와 같은 환경이라고 주장하지 않습니다.

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 \
  .venv_wsl/bin/python -B batch/tests/run_guarded.py
```

최신 전체159개 중144 PASS·15개 메서드 FAIL(실패항목33개), SKIP0입니다. [종목·시장 수급30개](investor-jobs.md), [가격·산업지수25개](ohlcv-jobs.md), [수동 산업 매핑24개](manual-mapping.md), [해외 조회23개](overseas-tools.md), [종목 초기화·대화형 조회25개](bootstrap-lookup.md), [CSV 파서·적재17개](csv-importer-flows.md)에 메서드별 방법과 결과를 기록했습니다. guarded 실행기는 실제 socket/HTTP/MySQL/subprocess/.env 읽기를 차단하며 이번 실행의 모든 차단시도 카운터는0입니다.

수급2개와가격·산업지수2개, 수동매핑, 해외조회2개, 종목초기화·대화형조회2개, 재무·공매도2개job은외부경계를대체한조건에서실제main도실행합니다. 외부 HTTP·실제 MySQL 적재·토큰 파일 생성은 실행하지 않습니다. 초기CSV검사는reader를mock했지만추가17개는실제decoder/DictReader부터적재루프까지연결했습니다. 수동매핑과종목초기화의 encoding·CSV/TSV parser도 합성 bytes와 실제 decoder로 검사합니다. 해외JSON/CSV writer는합성file handle문자열을parser로다시읽습니다.

| 케이스 | 입력 | 기대값 |
|---|---|---|
| 날짜 파싱 | null·빈값·00000000·20260230·잘못된 문자, 20260228 | 앞의 값은 None, 마지막은 2026-02-28 |
| 숫자 파싱 | 1,234.5·NaN·unknown | 1234.5·None·None |
| 중복 제거 | 1/2일 old, 1/1일, 1/2일 new | 1/1→1/2 new, 마지막 입력 우선 |
| 취득 범위 | 기준일1/20, latest 없음·1/18·1/20 | 30일 backfill·1/16부터 refresh·skip |
| 당일 일봉 | datetime만 mock하여 KST15:59/16:00 | 저장불가→저장가능 |

가격 유틸 결과: 5개 PASS. 현재 코드의 당일 확정시각 기본값 16:00을 검증했으며, 임의 환경변수로 변경한 시각에는 fixture를 조정해야 합니다.

## 재무·공매도 CSV importer 추가 검증

`test_csv_importers.py` 10개와 가격유틸5개의초기묶음은 **15 PASS**이며 최신159개에도통과했습니다. Decimal 계산과 tuple 생성은 실제 코드이며 MySQL connection/cursor를 MagicMock으로 대체합니다. 기존 문서의 `DOTENV_DISABLED`는 잘못된 변수였으므로 `PYTHON_DOTENV_DISABLED=1`과 .env읽기 audit guard로교체후재검증했습니다. 올바른변수없이제품module을import하려는테스트는명시적으로중단합니다.

| 검사 | 입력·수행 | 독립 기대값 |
|---|---|---|
| 재무 기간 | 5930, ttm, A/TTM, 잘못된 Q/H 분기 | 005930·TTM, A1·TTM0, Q0/5·H3·Q누락 거부 |
| 재무 비율 | 매출200·영업익30·순익10·자본50·시총100 | 영업이익률0.15·순이익률0.05·ROE0.2·PER10·PBR2, 분모0/누락은None |
| 비율 상한·반올림 | 999999.9999·1000000·0.12345, stdin 비대화형 | 상한 포함·상한 초과None 및 input 미호출·Decimal 기본 half-even0.1234 |
| 재무 mapping | 고정 날짜·버전·회계연도·매출123.45 | SQL placeholder 수=tuple 길이, 소수 보존, 공백 제거, 생성/갱신시각 동일, 미등록종목None |
| 공매도 mapping | 따옴표 포함5930·수량1,234.5·필수시장 | 005930→stockId7, Decimal1234.5, 기본KRX/MDCSTAT301, SQL tuple 길이 일치 |
| 잘못된 입력 | 2월30일·필수공백·비수치 | ValueError |
| flush 정상 | 두 importer 각각1개 row | executemany1·commit1·rollback0·cursor.close1 |
| flush 오류 | executemany에 합성 SQL 예외 | rollback1·commit0·cursor.close1·원래 예외 전파 |
| 빈 batch | [] | 0 반환·DB 관련 호출0 |
| CSV 전체 루프 | 빈행1·미등록종목1·날짜오류1·정상1, batch_size2 | 정상1개만 마지막 잔여 batch로 commit, connection.close1. 실제 변환·skip·flush 루프 실행 |

테스트 tuple과 SQL의 placeholder 일치는 실제 DB 컬럼·제약·upsert 동작 검증을 대체하지 않습니다. 성공/rollback 호출 확인 역시 실제 transaction 원자성 증거가 아닙니다.

2026-09-28에는 여섯 경로의 실제 SQL transaction·upsert 재실행·선택한 오류 복구를31개, 수동 매핑/초기화/해외 저장의 선택한 경계를22개로 확장했습니다. 남은 범위는 bootstrap backfill의 분류 일관성, 나머지 입력/외부연동, 거래소별 시간대·휴장일·미확정 거래일, 연결 단절/프로세스 장애 재실행, Spring scheduler와 중복 실행 방지입니다. 수동매핑의dry-run도 UPDATE 후rollback 방식이므로 실제운영DB에서읽기전용검사처럼실행해서는안됩니다. 이 문서는 Python 배치 전체 검증 완료를 의미하지 않습니다.
