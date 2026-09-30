# WSL FastAPI → Windows Spring Peer 역방향 TCP 검증

검증일: 2026-09-25. 제품 코드/설정은 변경하지 않았습니다. 별도 테스트 서버만 실행·종료했고 기존 Spring/DB/Redis/외부 금융 API는 사용하지 않았습니다.

## 실행과 결과

WSL, 저장소 루트:

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 PYTHONPATH=analysis:analysis/tests \
  .venv_wsl/bin/python -B analysis/tests/peer_cross_os_tcp.py
```

이 명령은 Windows PowerShell/Gradle로 테스트 전용 Spring 서버를 띄우고, WSL의 실제 uvicorn에서 Peer 요청을 실행한 뒤 두 프로세스를 종료합니다. Windows 방화벽·WSL NAT·포트 바인딩 권한이 필요합니다. 운영 QaimaApplication·scheduler·JPA/Redis 자동설정은 기동하지 않습니다. 테스트 코드에서 필요한 작은 WebFlux context만 구성합니다.

- Python HTTP 검사: **8개 중7 PASS·1 FAIL**, ERROR/SKIP0, 2.257초. 종료코드1은 최신일 제외 회귀 때문입니다.
- Windows 서버 lifecycle JUnit: **1 PASS**, Gradle 성공, 전체 Gradle 소요37초. 이1개를 기능검사8개와 혼동하지 않습니다.
- FastAPI 실제 허용 source TCP 연결6회. Spring 업무 endpoint 수신7회(역방향6+pack 직접 대조1), 의도적 nonce없는 접근거부1회.
- 금지 목적지 연결·subprocess·dotenv읽기 시도0. 유료 호출0. 모델 warm-up/다운로드와 가격 원천 fetch 비활성화.
- 두 테스트 프로세스 종료 확인. 기본 suite에서는 `QAIMA_LIVE_PEER_SOURCE`가 없으므로 Java host1개를 SKIP합니다.

실행 산출물은 비공개 `.runtime/peer-tcp/run-y0pn36aq/`에 있습니다: Python/JUnit summary, `fastapi-audit.json`, `spring-audit.json`, opt-in XML 복사본, fixture와 두 프로세스 로그. fixture에는 임시 nonce가 있으므로 공개 배포하지 않습니다. 후속 기본 Gradle 실행이 build XML을 덮어써도 run별 복사본은 남습니다. 실 주소·임시 nonce·설정값은 이 문서에 기재하지 않습니다.

## 실제 흐름과 대체한 경계

`Python HTTP probe → 실제 WSL uvicorn/Pydantic → compute_peer_cluster_v1 → 실제 SpringMarketDataProvider/httpx → 실제 Windows Reactor Netty/WebFlux → PeerClusterDataController → PeerClusterDataServiceImpl → 합성 Repository → JSON pack → 실제 Python 상관·시차·후보/차트 → HTTP JSON`.

코드:

- [Windows host 테스트](java/com/qaima/verification/LivePeerSourceTcpTest.java): 실제 Controller/Service, 작은 EnableWebFlux context, 임시포트, nonce path gate, 종료 handshake.
- [WSL 실행기·8개 검사](../../../analysis/tests/peer_cross_os_tcp.py): 양쪽 프로세스 관리·actual HTTP·독립 기대값·결과 보관.
- [FastAPI 테스트 wrapper](../../../analysis/tests/peer_tcp_app.py): 실제 app.main을 감싸고 nonce header와 허용 경로 제한. outbound socket은 이번 Windows IP/port 정확한 한 쌍만 허용.
- [공유 합성 fixture](../../../analysis/tests/peer_fixture.py): seed20260925, 91개 연속 날짜/90로그수익률, 5종목/산업지수. 실거래일 달력이 아닙니다.

Spring Repository/TradingCalendar만 mock이며 소스 JSON을 바로 반환하는 가짜 HTTP server가 아닙니다. fixture prices를 실제 Stock/PriceOhlcv/IndustryIndexOhlcv 엔티티로 바꿔 service에 전달합니다. Repository mock은 현재 SQL의 `[from,to)` 범위를 적용하고 지수 recent는 역순+limit을 적용합니다. 실제 SQL/JPA/잠금/영속성은 미검증입니다. 거래일은 모두 true, 최신일은 fixture 마지막 날로 고정합니다.

Java 거래량은 종목id1~5에11~15를 부여하고 실제 service가 close×volume으로 turnover를 계산합니다. 기존 ASGI fixture의 일정 turnover를 그대로 반환하지 않습니다. 이 차이를 반영하고도 선택집합·상관·시차 기대값이 유지됨을 확인했습니다.

## 8개 검사와 기대값

클래스 `LivePeerProbes`:

| 메서드 | 입력·수행·독립 기대값 | 결과 |
|---|---|---|
| `test_both_servers_reject_missing_nonce` | FastAPI header nonce 제거, Windows nonce path 없는 POST 각각403 | PASS |
| `test_real_spring_pack_unwrapped_json_dates_values_and_liquidity` | 직접 Windows endpoint로 정확기간5/1~7/30 요청. data envelope 없음·5종목 각91행·가격fixture 오차1e-14·첫UTC날짜·anchor volume11/turnover1100 | PASS |
| `test_exact_range_tcp_correlations_and_lag_match_independent_math` | 실제 uvicorn에 정확기간 요청→역방향TCP. SAME/FOLLOW/LEAD 선택, 원래 수익률 centered Pearson과 raw/industry차감 상관오차1e-10, lag0/+2/-2, adjusted90표본·차트91행 | PASS |
| `test_missing_index_real_spring_warning_reaches_fastapi_fallback` | 합성industry9는 mapping없음. Spring INDUSTRY_INDEX_SERIES_MISSING이 Python까지 유지, RAW fallback/adjustment false/peer3 | PASS |
| `test_repository_error_and_empty_universe_diagnostics_are_sanitized` | industry10의실제service가repo예외처리, industry11은빈종목. 각각 STOCK_REPO_FAILED/INDUSTRY_MEMBERS_EMPTY·빈peers, 합성 내부예외 상세는 최종JSON에없음 | PASS |
| `test_invalid_request_is_422_without_reverse_source_call` | industry0→실제Pydantic422. 요청전후Windows수신counter같음 | PASS |
| `test_weekly_request_falls_back_to_daily_through_tcp` | ONE_W 요청→ONE_D fallbackwarning·정확기간91차트행 | PASS |
| `test_window_mode_must_include_latest_available_source_date` | 날짜미지정window90. 종목·지수원천 마지막2026-07-30 기대, 최종anchor_series는2026-07-29 | **FAIL** |

Java `serveRealPeerPackUntilWslProbesComplete`는 handshake 완료·업무HTTP수신>0·실제service의fixture repository호출을 확인하고 server.disposeNow/context.close합니다. Python 계산검사 실패가 있어도 host lifecycle 자체는 정상일 수 있으므로 두 결과를 분리합니다.

## 결함과 남은 범위

기존 **F2-PEER-DATE-001**을 실제 TCP/최종 FastAPI 차트 응답까지 확장 재현했습니다. 새로운 별도 결함으로 중복 집계하지 않습니다. 정확기간 모드는91행을 보존하는 대조군입니다. 상관계수의 window 변화량이나 실제 사용자 데이터에서의 영향 규모는 측정하지 않았습니다.

아직 미검증: 실제MySQL/JPA·휴장일·운영 데이터, 전체Spring security/JWT, Public Peer client/Redis/크레딧/브라우저까지의 폐회로, 동시요청·대규모성능·timeout/네트워크단절, 실제배포의방화벽/인증정책. nonce gate는 테스트 서버 보호용이며 제품 인증 검증을 대체하지 않습니다. Swagger `/api/v1/feature2/peercluster/data`는 여전히 PARTIAL입니다.
