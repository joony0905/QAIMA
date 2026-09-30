# Feature2 Chrome 검증 — 2026-09-28

`dev`의 현재 작업 트리를 대상으로 [browser-feature2-flow.cjs](browser-feature2-flow.cjs)를 실행했습니다. **전체 18개: 15 PASS / 3 FAIL, exit 1**입니다. 실제 Chrome·React·Router·Axios·브라우저 storage·캔들 컴포넌트를 실행했고 Public API는 모두 합성 fixture입니다. 실패한 기대값을 현재 제품 동작에 맞춰 PASS로 바꾸지 않았습니다.

## 실행 방법과 격리

기존 [격리 빌드 준비 방법](README.md)을 사용합니다. 원본과 `tests/.runtime/build-project`의 `src`, `public`, 빌드 설정/lockfile **187개**를 비교했고 모두 일치했습니다. 디렉터리별 파일 목록도 비교하므로 복사본에 남은 삭제 파일은 검사를 실패시킵니다. 다음 명령으로 현재 복사본을 다시 빌드했습니다. Vite 7.1.12, 2165개 모듈, exit 0입니다. 큰 bundle 및 오래된 브라우저 메타데이터 경고가 남아 있습니다.

Windows PowerShell에서:

```powershell
Set-Location C:\qaima\frontend\tests\.runtime\build-project
node node_modules/vite/bin/vite.js build
if ($LASTEXITCODE -ne 0) { throw 'Isolated build failed' }
Set-Location C:\qaima\frontend
Remove-Item Env:QAIMA_F2_CASE -ErrorAction SilentlyContinue
node tests/browser-feature2-flow.cjs
```

WSL에서는 같은 명령을 `powershell.exe -NoProfile -Command '...'`로 실행했습니다. WSL에서 Windows 실행 파일의 출력을 shell 파일 리다이렉션한 두 시도는 `UtilBindVsockAnyPort: socket failed 1`로 실행 전 실패했고, 리다이렉션 없는 PowerShell 호출은 정상 실행됐습니다. 제품 빌드 실패로 집계하지 않습니다.

테스트는 빈 포트 확인 후 자체 Vite preview를 loopback 4179/strictPort로 시작합니다. 시나리오마다 새 BrowserContext, 기본 1440×1100, ko-KR, headless Chrome을 사용합니다. `Date`는 2026-09-28 12:00 KST로 고정하되 타이머는 계속 진행시킵니다. 모바일은 390×844입니다.

service worker와 다른 origin 요청은 차단합니다. `/api/`의 등록된 method/path만 fixture 응답을 주고 미등록 요청은 500 및 검사 FAIL입니다. Spring·분석 서버·DB·Redis·외부 금융/LLM·메일·OAuth 호출은 없습니다. Chrome이 요청한 외부 폰트/CDN도 차단됐습니다. 테스트 종료 시 소유 browser/preview만 종료하고 4179 listener 부재를 확인해 summary에 기록합니다.

각 실행 폴더에 요청 method/path/query/body, Bearer 존재 여부, 응답 상태, 응답 완료 순서, 지연/해제 여부, 관측 DOM, 결과, 실패 화면/본문, 입력 SHA-256과 dist 11개 파일 SHA-256을 남깁니다. 실제 인증 토큰이나 운영 설정을 복사하지 않습니다. 최초 전체 실행은 script hash 필드 추가 전이며, 이후 실행은 script SHA-256과 필터도 기록합니다.

## 전체 검사별 방법과 결과

기본 합성 종목 A는 `005930`, stockId 42, 조회 가격 999입니다. 캔들 종가가 100→120이므로 독립 기대값은 현재 120, 변화 +20, (+20.00%)입니다. B는 `000660`, stockId 43입니다. 실제 기업의 시세를 뜻하지 않습니다. 카드 중 금리 2.5%, 뉴스·유사종목은 합성 값이고 산업지수·수급·공매도·거시 시계열 일부는 null/빈 배열입니다. 따라서 해당 전체 차트의 수학·시각 품질 PASS가 아닙니다.

| 번호 | 입력·행동·검증 | 결과 |
|---|---|---|
| 1 | 공백 입력 후 Enter → 종목·Feature2 요청 없음 | PASS |
| 2 | guest 검색 → 공개 카드·뉴스·캔들 조회, 120/+20/+20% 표시, lookup 999 미표시, 자동 분석/관심종목 요청 0. 산업 ONE_D/window120, 관련주 limit30, 수급 limit60 확인 | PASS |
| 3 | guest 분석 클릭 → `/login`, 복귀 경로 `/feature/2`, 분석 POST 0 | PASS·현재 동작 기록 |
| 4 | guest 별 클릭 → 로그인 안내, watchlist 요청 0 | PASS |
| 5 | 로그인 별 클릭 → POST stockId42, itemId71 응답으로 별 채움 → DELETE `/items/71`, 별 비움 | PASS |
| 6 | 뉴스·거시 시계열·수급 500 → 캔들 유지, 뉴스/거시 오류 안내, 유사종목 표시와 분석 버튼 활성 유지 | PASS |
| 7 | guest 뉴스/산업 카드 401 → `/feature/2` 유지, 초기 refresh 1회, 캔들 유지 | PASS |
| 8 | 3개월·초급자 → window60, from 2026-06-28, 투자 수준 전달 | PASS |
| 9 | 6개월·중급자 → window120, from 2026-03-28, 투자 수준 전달 | PASS |
| 10 | 9개월·고급자 → window180, from 2025-12-28, 투자 수준 전달 | PASS |
| 11 | 1년·전문가 → window252, from 2025-09-28, 투자 수준 전달 | PASS |
| 12 | 영어·전문가 분석 → languageCode en, 수준 전문가, `INDUSTRY_MISSING`을 영어 안내로 표시하고 원시 코드 숨김 | PASS |
| 13 | 분석 500 → 공통 한국어 오류, 원천 상세문구 미표시, 분석 POST 1회, 잔액 재조회. 재실행 버튼 0 기록 | PASS·복구 UX는 관측 |
| 14 | 종합 분석 200 + 별도 기준금리 시계열 500 → 성공 분석을 유지해야 한다는 기대값 | **FAIL** |
| 15 | A 유사종목 응답 보류 → B 검색/유사종목 표시 → A 응답 해제 → B 목록을 유지해야 한다는 기대값 | **FAIL** |
| 16 | A 분석 응답 보류 → B 검색 → A 응답 해제 → 이전 분석이 B 화면에 붙지 않아야 한다는 기대값 | **FAIL** |
| 17 | 390×844 화면에서 검색·분석 완료, document width375 ≤ viewport390 | PASS·전체 모바일 품질은 아님 |
| 18 | 모든 시나리오의 미등록 API 0·uncaught pageerror 0 | PASS |

8~11은 body 전체를 독립 기대값과 비교합니다. 공통값은 stockCode005930, freq ONE_D, peerCount30, llmVendor `GPT-5 mini`, languageCode ko, to `2026-09-28T12:00:00+09:00`이며 from에도 12:00 KST를 기대합니다. 공매도 시계열은 선택 window, 기준금리 시계열은 limit365로 요청합니다. 합성 요약 렌더, 분석 후 뉴스 교체, 잔액 재조회도 확인합니다. 네 수준은 기간과 짝지어 검증했으며 4×4 모든 조합이나 실제 LLM 설명 품질/정량 불변성 검사가 아닙니다.

## 새 실패와 원인 근거

### FRONT-F2-AUX-001 — 보조 조회 실패가 성공 분석을 숨김

종합 분석 POST는 200, `/cards/base-rate-series`는 500입니다. 관측 결과 summaryVisible0, errorVisible1이며 화면에는 일반 서버 오류만 남습니다. [Feature2MockPage](../src/pages/Feature2MockPage.tsx)의 937~952행이 종합 분석과 보조 시계열 두 개를 `Promise.all`로 묶고, 959행의 결과 반영 전에 catch로 이동합니다.

[기능 정책](../../policy/Feature_analysis_user_flow_policy.md)의 종합 분석과 카드 조회 구분·부분 실패 시 나머지 결과 유지 방향과 대조한 회귀입니다. 검증된 사실은 브라우저에서 성공 payload를 사용할 수 없게 된다는 것입니다. 실제 차감/저장 여부나 실제 환불 실패를 검증한 것은 아닙니다.

### FRONT-F2-RELATED-RACE-001 — 이전 검색 응답으로 유사종목이 바뀜

A 관련주 GET을 응답 직전에 gate로 보류합니다. B 검색과 `B의 유사기업` 표시를 확인한 뒤 A 응답을 해제합니다. 두 렌더 프레임 이후 B heading1, A 관련행1, B 관련행0입니다. 고정 시간 sleep으로 우연한 순서를 유도하지 않고 응답 gate와 DOM/요청 완료를 사용합니다.

785~795행의 relatedStocksTask는 응답을 받은 후 현재 searchRequestId를 확인하지 않습니다. 인접 news/investorFlow 경로에는 요청 세대 비교가 있지만 이 경로에는 없습니다. 유사종목을 클릭하면 다음 검색으로 연결되므로 표시된 선택 종목과 관련 목록의 일관성이 깨집니다.

### FRONT-F2-ANALYSIS-RACE-001 — 이전 종목 분석과 새 종목 이름이 섞임

A 분석 POST만 보류하고 B를 검색합니다. B 관련주·캔들 응답이 완료된 뒤 A 분석 200을 해제합니다. **B 분석 POST는 0회**인데 A 결과 요약이 B 화면 DOM에 남았습니다. 스크린샷에서는 검색 입력 `000660`, 종목명 `합성기업B`, 종목코드 `005930`의 혼합도 확인했습니다.

요청 증거에는 B 관련주 응답 완료 순서33 → A 분석 완료38 → A 캔들 재조회39 → 잔액 GET40 → A 캔들 후속 조회가 남습니다(전체 실행 기준). 953~966행은 분석을 시작한 당시 A symbol을 다시 캔들에 적용하고 결과·뉴스를 저장하며 새 검색과의 세대/종목 일치를 확인하지 않습니다. `applyCandleResponse` 296~304행은 기존 이름을 유지하면서 symbol을 바꿉니다. 검색 초기화 674행만으로는 늦은 분석 응답을 막지 못합니다.

분석 응답과 사용자 선택의 일치성 결함이며 다른 사용자 데이터 유출 증거는 아닙니다. 제품 코드는 수정하지 않았습니다.

## 관측과 검증 경계

- `OBS-F2-RECOVERY-001`: 분석 500 후 재실행 버튼이 사라집니다. 935행의 false 상태를 catch에서 복원하지 않습니다. 테스트 13의 PASS는 현재 동작 관측이며 복구 UX 적합 판정이 아닙니다.
- `OBS-F2-RETURN-001`: guest 분석 시 저장한 복귀 경로는 `/feature/2`뿐이며 선택 종목 정보가 없습니다. 실제 로그인 후 종목 복원 흐름은 아직 검증하지 않았습니다.
- 잔액 GET이나 fixture watchlist 상태 변화는 실제 원장·환불·영속성·소유권 성공 증거가 아닙니다.
- 실제 Spring/FastAPI/MySQL, 원천 API·캐시·비용/환불·리포트 저장, 실제 OAuth는 이번 실행 범위 밖입니다.
- Feature2 분석 직후 PDF, 산업/peer/수급/공매도 전체 시계열 값, 모든 탭/모드·테마·모바일 접근성, 검색 자동완성 경합 등은 남았습니다. 기존 실패도 해소하지 않았습니다.

## 실행 이력·증거

- 최초 전체 `run-DxF57D`: 16개 중10 PASS/6 FAIL. 네 실패는 테스트가 프로필 `experience`를 영어 값으로 공급한 fixture 오류였습니다. 실제 [InvestmentLevel](../../backend/src/main/java/com/qaima/domain/InvestmentLevel.java)·[프론트 변환](../src/utils/investLevel.ts)은 한글 표시값을 사용하므로 fixture/기대값만 수정했습니다. 나머지 두 실패는 위 AUX/RELATED입니다.
- 보정 및 영어/분석 경합 추가 후 전체 `run-A8Ux87`: **18개 중15 PASS/3 FAIL**. `summary.json`, `failure-14.png/.txt`, `failure-15.png/.txt`, `failure-16.png/.txt`, `analysis-result.png`, `mobile-analysis.png`를 보존합니다. 미등록 API/pageerror0, previewStopped true입니다.
- 최종 실패 재확인 `run-4mREws`: 위3개 모두 다시 FAIL, 공통 미등록API/pageerror 검사는 PASS(필터 실행4개 중1 PASS/3 FAIL). `stale-analysis-report.png`를 별도로 렌더링해 육안 확인했습니다. 한 리포트 안에 **분석대상 합성기업B(005930) / 분석종목 합성기업A**가 함께 표시됩니다. summary에 script SHA-256, 필터, source187개/dist11개 해시, previewStopped true가 있습니다. 전체18개와 중복 집계하지 않습니다.
- 세 실패만 재확인할 때는 아래 필터를 사용할 수 있습니다. 출력된 테스트 수는 필터 실행 범위이며 전체18개와 합산하지 않습니다.

```powershell
$env:QAIMA_F2_CASE = 'successful core|latest search|previous stock'
node tests/browser-feature2-flow.cjs
Remove-Item Env:QAIMA_F2_CASE
```

증거 폴더는 `frontend/tests/.runtime/browser-feature2/` 아래에 있습니다. `.runtime`은 공개 문서가 아닌 로컬 재현 증거이며 기존 run을 덮어쓰지 않습니다.
