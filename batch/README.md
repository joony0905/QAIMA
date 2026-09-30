# 수집·적재 배치

작성·검증일: 2026-09-25. `jobs/`의 Python 스크립트와 Spring의 scheduler·sync service는 별도 실행 경로입니다. Python 배치 전체가 FastAPI와 함께 자동 시작되는 구조는 아닙니다.

| 작업 파일 | 담당 |
|---|---|
| `bootstrap_stock_from_kis.py` | KIS 종목 기초 정보 |
| `load_price_ohlcv.py` | 국내 가격 OHLCV backfill·refresh |
| `load_index_ohlcv.py` | 산업 지수 시계열 |
| `load_stock_investor_flow.py`, `load_market_investor_flow.py` | 종목·시장 투자자 수급 |
| `import_financials.py`, `import_short_selling.py` | 재무·공매도 import |
| `apply_stock_industry_manual_mapping.py` | 수동 산업 분류 매핑 |
| `kis_overseas_industry_codes.py`, `kis_overseas_daily_chartprice.py` | 해외 산업 코드·차트 조회 및 저장 도구 |
| `kis_raw_lookup.py` | KIS 응답 조회 도구 |

각 스크립트의 `parse_args`, 환경변수 이름과 `main`을 확인하고 실행합니다. 스크립트에 따라 MySQL 연결·외부 호출·파일 저장이 발생합니다. 인증정보는 로컬 설정으로 제공하고 문서에 기재하지 않습니다. 테스트는 기본적으로 순수 함수·합성 경계를 사용하며, 수급2개job의main은 HTTP·DB·파일을대체하고실제접속을차단한조건에서검사했습니다.

검증된 `load_price_ohlcv.py` 유틸은 날짜·숫자 파싱, 같은 거래일의 마지막 행 유지·정렬, backfill/refresh/skip 범위, KST 당일 16:00 전 일봉 제외입니다. 이 결과는 SQL upsert 멱등성이나 제공처의 실제 거래일 정확성까지 입증하지 않습니다.

현재 배치 suite는159개 중144 PASS·15개 메서드 FAIL입니다. 비유한숫자·rollback누락·전체대상실패시종료코드0·인코딩재시도행중복·CSV필수열누락정상반환·해외주봉/월봉의일봉주기저장요청·대체상장일누락을재현했습니다. 열한도구의실제main과처리흐름을합성경계로연결했으며실제DB저장·금융API호출은없습니다. [수급](tests/investor-jobs.md), [OHLCV](tests/ohlcv-jobs.md), [수동매핑](tests/manual-mapping.md), [해외조회](tests/overseas-tools.md), [종목초기화·대화형조회](tests/bootstrap-lookup.md), [재무·공매도 CSV](tests/csv-importer-flows.md)의상세검증을참조합니다.

수동매핑의기본dry-run은UPDATE후rollback이며읽기전용이아닙니다. 검증은mock DB에서만수행했습니다.

실행 명령·fixture·기대값은 [tests/README.md](tests/README.md), 운영 원칙은 [ETL 정책](../policy/ETL_POLICY.md)을 참조합니다. 기존 데이터에 전체 배치를 재실행하는 검증은 별도 테스트 데이터 범위와 스케줄러 중복 실행 여부를 확인한 뒤 수행합니다.
