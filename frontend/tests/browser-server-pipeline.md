# Feature3 브라우저·실제 Spring/FastAPI·과금/저장 리포트 QA

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03` 및 기존 미커밋 소스를 대상으로 합니다. 제품·기존 DB는 수정하지 않고 테스트 코드/문서/산출물은 tests 안에, MySQL/Redis datadir는 외부 `/tmp`에 둡니다.

## 연결한 실제 경계

[앞선 서버 통합18개](../../tests/isolated-analysis-pipeline.md)에 Linux Chromium/React/Axios와 로그인 화면·쿠키 갱신·포트폴리오 입력·저장 리포트/PDF를 연결합니다. [JUnit 호스트](../../tests/java/com/qaima/qa/IsolatedBrowserPipelineTest.java)가 새 사용자/포트폴리오를 준비하고, [브라우저 실행기](browser-server-pipeline.cjs)를 자신이 시작한 Spring 서버에 연결합니다. 실행 명령은 다음과 같습니다.

```bash
cd /mnt/c/qaima
python3 -B tests/run_isolated_backend.py --suite browser-pipeline
```

Linux Node20/Chromium153과 WSL Java17/Gradle8.14·Python venv·추출 MySQL8.0.46/Redis7.0.15를 사용합니다. 원본 프론트187개와 격리 복사본, 빌드11개를 앞선 Feature3 차트 최종 실행의 해시와 대조합니다. 일치한 빌드를 테스트 소유 HTTP 서버가 그대로 제공합니다. API는 실제 Spring에 전달하며 Playwright로 응답을 만들거나 요청/응답 본문을 수정하지 않습니다. 이 검사는 운영 웹서버/TLS 배포 구성을 검증하는 것은 아닙니다.

| 영역 | 실행/대체 방식 |
|---|---|
| 브라우저 | 실제 Chromium·현재 React 빌드·Axios·메모리 토큰·HttpOnly cookie·화면 입력·차트·저장 리포트/PDF |
| 인증/업무 데이터 | 실제 Auth/User/Portfolio/Report/Credit/Watchlist/Dictionary Controller·service·SQL. 초기 계정과 포트폴리오만 합성 |
| 분석 | 실제 Feature3 Controller/WebClient→uvicorn/Pydantic/수학 계산·설명 fallback→Public DTO·SQL 저장 |
| 가격/벤치마크 | service 반환 DTO를 합성 시계열로 대체. 요청 lookback에 맞춰 마지막126가격/125로그수익률로 자름. 실제 원천/가격 SQL은 별도 |
| 금리 | 계산 금리2%의 합성 service, 화면 거시카드는 실제 Controller와 빈 합성 CardService 응답 |
| overlay | 실제 Redis metrics 저장·비용 preview·추정·신호 조립·Python 조정. 외부 Feature1 재분석0 |
| LLM | 브라우저의 includeLlmExplain=true를 그대로 전달. 키를 제거하고 제품의 LLM_API_KEY_MISSING→결정론적 설명 fallback 실행. Python outbound/subprocess/dotenv 가드로 외부 전송0을 확인 |

선택한 Spring 구성만 시작하며 전체 앱/scheduler/운영 설정을 로드하지 않습니다. 테스트 origin용 CORS loopback 패턴을 설정합니다. cookie 기본 HttpOnly/path/SameSite를 실제 브라우저에서 확인하지만 운영 HTTPS secure/domain 정책 전체를 검증하지 않습니다.

## 입력과 기대값

새 사용자 A/B는 각각5크레딧·위험성향0.5를 갖습니다. A 기본 포트폴리오는 QAA/QAB/QAC 수량2/3/4, 평균단가90, 현금30이며 시장은 KOSPI/KOSPI/KOSDAQ입니다. 기본 포트폴리오 준비는 실제 PUT HTTP를 사용합니다. 브라우저에서는 현재가를 보내지 않아 마지막 제공 종가로 평가합니다.

브라우저의 6개월 선택은126거래일·LEDOIT_WOLF·연율252이며 옵션을 바꾸지 않습니다. [기대값 생성기](../../tests/build_pipeline_fixture.py)는 원래 합성 로그수익률의 마지막125개에 sklearn LedoitWolf를 직접 적용하고 현재 비중과 `sqrt(wᵀΣw)`를 계산합니다. 제품 계산 함수는 호출하지 않으며 실원천 수익률/모델의 경제적 타당성을 검증하는 것은 아닙니다. HTTP 응답 변동성은 독립 기대값±10⁻⁶, 화면 표시는 그 값의 소수1자리 퍼센트와 대조합니다.

모든 실제 성공 분석의 HTTP data를 저장 SQL snapshot과 비교합니다. 저장 리포트 화면 사례에서는 실제 GET 상세의 resultSnapshot도 동일한 값인지 확인합니다. 각 브라우저 자식이 종료된 뒤 Java의 별도 JDBC로 잔액·원장·리포트·로그인 세션 commit 상태를 확인하고 마지막에 UI 검사 결과를 판정합니다. 따라서 UI 실패가 서버의 금전/저장 검사를 생략하지 않게 합니다.

브라우저 증거 JSON을 다시 저장할 때 숫자0.0은0으로 표현될 수 있으므로 SQL 대조는 JSON 구조·문자열·배열 순서·null을 유지하면서 숫자만 BigDecimal 값으로 비교합니다. 허용오차나 필드 제외로 snapshot 불일치를 숨기지 않습니다. 선택한 Spring 구성에 제품 `RedisConfig.redisObjectMapper()` bean과 Spring Boot `CodecsAutoConfiguration`을 함께 제공해 HTTP codec에 제품의 ISO 날짜 직렬화를 연결합니다.

## 8개 매개변수 사례

JUnit `browserAgainstOwnedServers`의 8개 invocation입니다. 최종 `run-0n529i50`는 **7 PASS / 1 FAIL**, errors/skipped0, JUnit81.420초입니다. 제품의 저장 실패 안내 누락을 보존하므로 Gradle exit1입니다.

| mode | 최종 결과 | 직접 수행하는 검증 |
|---|---|---|
| `core_saved_report_pdf` | PASS | 로그인 화면→저장 보유3개→6개월 분석→화면/독립 변동성→최적화 탭→내정보/리포트 보기→원본 snapshot 대조→PDF 재다운로드. 분석1회·차감1·report1·PDF로 재분석 없음 |
| `save_reload` | PASS | QAA 수량7 입력→저장 확인→실제 PUT→새로고침/실제 cookie refresh→화면7·SQL7. 분석/원장/report0 |
| `cached_overlay` | PASS | 합성 metrics3개를 실제 Redis에 저장→종목 체력 checkbox→실제 비용 preview1credit→확정→실제 신호3개·조정 포트폴리오·차감1 |
| `calculation_failure` | PASS | QAA source만 잘못된 enum으로 주입→실제 Python500→Public500·화면 오류/재시도 버튼·잔액5→실제 원장−1/+1·report0 |
| `report_save_failure` | FAIL | 소유 report INSERT trigger로 SQL 실패→계산200·REPORT_SAVE_FAILED·원장−1/report0. 저장 실패가 실제 화면에도 보이는지 검사 |
| `insufficient` | PASS | 소유 A 잔액0→분석 클릭→Public402·화면 오류/버튼 복구·잔액0. Python 계산·원장/report0 |
| `owner_isolation` | PASS | A의 실제 계산 report1개를 준비→B로 실제 로그인→내 리포트 비어 있음→B의 실제 브라우저 인증으로 A 상세404/data없음 |
| `session_refresh_logout` | PASS | 실제 로그인→새로고침/refresh200→내정보 로그아웃/logout200→cookie삭제·SQL활성세션0/revoked행1→Feature3 비로그인 안내 |

## 증거와 종료

`tests/.runtime/backend-isolated/runs/run-*`에 JUnit XML·Spring/FastAPI 로그·SQL 정리·계산 요청/응답을 보존합니다. 각 브라우저의 `frontend/tests/.runtime/browser-server-pipeline/<run>/<mode>`에 민감한 인증 본문/헤더를 제외한 Public 요청/응답·PNG·PDF·summary를 저장합니다. 로그인용 임시 manifest는 자식 실행 후 삭제합니다. 비밀번호·access/refresh token·쿠키 값은 관측 JSON에 기록하지 않습니다.

Linux browser/proxy와 소유 임시 프로필은 각 자식 finally에서 종료/정리하고, MySQL/Redis/FastAPI는 상위 실행기 finally에서 정리합니다. 기존 서비스의 PID를 찾아 종료하지 않습니다. 외부 origin과 service worker는 브라우저에서 차단합니다. 허용하지 않은 API는 기록/실패하며 pageerror도 실패합니다. PDF는 실제 다운로드를 검증하고 후속 파서/시각 확인 결과를 별도로 기록합니다.

## 실행 이력

최초 `run-z7znr3n_`는8개 모두 브라우저 미기동으로 실패했습니다. 현재 실행 환경의 Windows interop가 `UtilBindVsockAnyPort: socket failed`로 차단돼 Node 자식이 증거를 생성하지 못했습니다. 브라우저 제품 동작은 실행되지 않았으므로 제품 결함으로 집계하지 않습니다. Java가 소유자 격리를 준비하며 수행한 계산1회는200이고, SQL/Redis 정리·세 서버 종료는 상위 summary에서 확인했습니다.

Linux Chromium153.0.8010.12의 사전 기동에서는 기본 crashpad 설정 경로 문제가 있었으며 소유 /tmp 경로에 XDG_CONFIG_HOME/XDG_CACHE_HOME을 지정해 페이지 기동/종료를 확인했습니다. 그 보정 전에 시작한 `run-w3ol0gku`는 JUnit 전에 중단했고 summary가 없습니다. 소유 MySQL의 TCP 및 Unix socket 연결은 모두111/거부이고, 정상 shutdown/SQL 정리 증거는 없으므로 정상 종료로 집계하지 않습니다. [중단 기록](../../tests/.runtime/backend-isolated/runs/run-w3ol0gku/interrupted-cleanup.json)에 그 한계를 남깁니다. 이후 `run-rojonopp`의8개도 mounted 경로의 브라우저 임시 프로필을 사용할 때 SIGTRAP으로 기동하지 못했습니다. API 호출0이며 제품 결함에 추가하지 않습니다. 이 실행의 SQL/Redis 정리·세 서버 종료는 summary로 확인했습니다. 브라우저 프로필/TMPDIR와 XDG 경로를 짧은 소유 `/tmp/qaima-browser-*`로 옮겨 사전 DOM 검사에 통과했고 후속 통합 실행에 적용했습니다. 해당 임시 브라우저 경로만 finally에서 삭제하며 제품 소스와 기존 프로필은 사용하지 않습니다.

`run-0fp0mla_`에서는 실제 브라우저 로그인·보유 조회·저장 후 새로고침·다른 사용자 리포트 차단·로그아웃까지 실행됐습니다. JUnit8개2 PASS/6 FAIL 중5개는 분석 버튼의 실제 문구와 다른 선택자를 사용해 분석 요청 자체가 없었고, 나머지1개는 로그아웃 시 세션행 삭제를 기대한 테스트 오류였습니다. 제품은 revoked_at을 남기는 방식이므로 활성세션0/revoked행1로 보정했습니다. Node 화면 검사는 저장/소유권/로그아웃3개 PASS였습니다. SQL·Redis 정리/세 서버 종료·입력600개 변경0을 확인했습니다. 한글 글꼴 부재로 PNG가 사각형으로 보였으므로 기존 설치된 Malgun Gothic regular/bold를 읽기 전용 symlink와 테스트 FONTCONFIG_FILE로 연결했습니다. 최종 summary에 두 글꼴 파일의 해시를 기록하며 제품 CSS는 바꾸지 않습니다.

`run-8ae82m4t`는 JUnit8개4 PASS/4 FAIL입니다. 숫자 값은 동일하지만 JavaScript의 JSON 재직렬화로0.0이0이 되어 Jackson node 타입 비교2개가 실패했고, 계산 오류1개는 제품의 한국어 오류 문구와 다른 기대값을 사용했습니다. 둘은 테스트 오류로 보정했습니다. 저장 리포트 화면에는 테스트 구성에서 빠진 제품 ObjectMapper 때문에 날짜가 epoch초로 내려와 목록이 숨겨졌습니다. 실제 `RedisConfig.redisObjectMapper()`를 테스트 bean으로 사용해 ISO 날짜 직렬화를 제품과 맞췄습니다. 저장 실패 경고는 HTTP meta에 있으나 화면에는 없는 것을 관측했으며 보정 후 재확인합니다. SQL/Redis 정리·세 서버 종료·입력600개 변경0을 확인했습니다. 이 실행은 최종 누적 사례 수에 더하지 않습니다.

후속 `run-lzghpqy0`는 8개6 PASS/2 FAIL입니다. 숫자 snapshot 비교·계산 오류 화면·환불·캐시 overlay는 통과했고, 저장 실패 안내 누락은 재현됐습니다. 리포트 화면의 나머지1개는 ObjectMapper bean만으로 HTTP codec에 적용되지 않은 테스트 구성 문제입니다. 선택 구성에서 빠진 `CodecsAutoConfiguration`을 추가해 실제 Boot의 codec 연결을 포함했습니다. 8개 모두 SQL 과금/리포트/세션 대조까지 실행됐고 최종 SQL/Redis0·세 서버 종료·입력600개 변경0을 확인했습니다. 최종 `run-0n529i50`에서는 해당 리포트 조회/PDF 사례도 통과해 7 PASS/1 FAIL입니다.

## 저장 실패 안내 관측

`FRONT-F3-REPORT-SAVE-001`: 소유 DB의 리포트 INSERT만 trigger로 거절하면 실제 계산은200이고 HTTP `meta.warnings`에는 `REPORT_SAVE_FAILED`가 있습니다. 실제 잔액은5→4, 원장은−1, 저장 리포트는0행이지만 브라우저에는 분석 결과만 표시되고 저장 실패 안내가 없습니다. 재현 기준은 실패가 사용자에게 보이는지이며, 계산 성공 시 차감 유지 자체는 결함으로 판정하지 않습니다.

관련 구현은 [서버 attachReportId](../../backend/src/main/java/com/qaima/api/feat3/Feature3AnalyzeController.java)와 [fetchPortfolioAnalysis](../src/api/portfolio.ts)입니다. 서버는 저장 예외를 meta 경고로 내보내지만 프론트 함수는 `res.data.data`만 반환합니다. `run-8ae82m4t/report_save_failure`의 실제 HTTP·SQL 관측과 한글 PNG에서 이를 확인했습니다. 보정한 `run-lzghpqy0`와 최종 `run-0n529i50`에서도 같은 문제가 재현됐습니다. 제품 코드는 변경하지 않았습니다.

## 최종 결과와 PDF 확인

최종 [서버 summary](../../tests/.runtime/backend-isolated/runs/run-0n529i50/summary.json)·[JUnit XML](../../tests/.runtime/backend-isolated/runs/run-0n529i50/TEST-com.qaima.qa.IsolatedBrowserPipelineTest.xml)과 [브라우저 산출물](.runtime/browser-server-pipeline/run-0n529i50)을 보존했습니다. 브라우저 8개의 Public 요청은 총127개이며 미등록 API/pageerror0, browserClosed/proxyClosed/runtimeCleaned는 모두true입니다. 외부 CDN/Google Fonts 요청은 차단했고 기존 설치 한글 글꼴로 표시했습니다.

- 실제 FastAPI 계산5회는200×4/500×1입니다. 정상 core·cached overlay·소유권 준비의 리포트3개가 저장됐고 HTTP와 SQL snapshot을 대조했습니다. 브라우저 core에서는 GET 상세 snapshot도 일치합니다. 저장 실패 사례는 계산 성공/차감1/report0, 계산500은−1/+1 환불, 잔액0은 계산 호출0입니다.
- 실제 저장 리포트 PDF 재다운로드는 추가 분석/과금 없이 실행됐습니다. [브라우저 summary](.runtime/browser-server-pipeline/run-0n529i50/core_saved_report_pdf/summary.json)의 PDF는20,452,848bytes·A4 1페이지이고 SHA-256은`64f19802ea7d6657f31cbe0a09ce34566a91669a0c2beb1ba2a178f11020ff72`입니다. 파서로 수리 없음·페이지수·크기·본문 픽셀 비율1.904%를 확인했습니다. [화면 PNG](.runtime/browser-server-pipeline/run-0n529i50/core_saved_report_pdf/saved-report.png)와 [PDF 페이지 PNG](.runtime/browser-server-pipeline/run-0n529i50/core_saved_report_pdf/saved-report-page-1.png)를 직접 열어 한글·6.7%/14.0%·섹션/워터마크와 마지막 포트폴리오 비교 본문까지 확인했습니다. 이 1페이지 결과로 기존 분석 직후 PDF의 스타일/긴 문서 분할 실패가 해결됐다고 판단하지 않습니다. 파일 크기는 관측으로 남기며 성능 기준 통과를 주장하지 않습니다.
- 종료 시 업무9테이블·QA종목·trigger0, Redis DBSIZE0입니다. MySQL/Redis는 exit0·정상종료, FastAPI는 SIGTERM/exit−15·소유 포트 닫힘입니다. Python outbound/subprocess/dotenv 접근 시도0, 임시 로그인 manifest 잔여0, 입력600개 실행 중 변경0을 확인했습니다. 중단된 과거 `run-w3ol0gku`의 종료 증거 한계는 위 이력에 별도로 유지합니다.

PDF 보강 검사는 다음 명령으로 실행했고 1파일/1페이지 PASS입니다. 이 검사는 JUnit 8개 사례 수에 추가하지 않습니다.

```bash
PYTHONPATH=frontend/tests/.runtime/pdf-tools PYTHONDONTWRITEBYTECODE=1 \
  .venv_wsl/bin/python -B frontend/tests/inspect-report-pdf.py \
  --artifacts frontend/tests/.runtime/browser-server-pipeline/run-0n529i50/core_saved_report_pdf
```

문서60개 대상 링크/설정 비밀값 검사2개는 PASS(2.013초)이며 로그는 `tests/.runtime/backend-isolated/browser-pipeline-documentation.log`입니다. `browser-evidence-audit.json`에 JUnit/문서/브라우저8개·스크립트 해시·원본187개/빌드11개·서비스 정리·임시 manifest0 대조 결과를 기록했습니다. 별도 tests 밖924개 소스 비교도 변경/소실/새파일0입니다.

## 남는 범위

가격/금리 실원천·운영 provider/LLM·실제 SMTP/OAuth·TLS/배포 구성, 해외시장/환율·전 언어/테마/투자 수준 조합, 장기 장애/중단·동시 브라우저/다중 서버, 전 API 정상/오류/권한 흐름은 별도입니다. 앞선 tests-only 제약과 전체 QA 범위를 유지합니다.
