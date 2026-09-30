# Feature1 재무·차트·검색 경합·분석 직후 PDF 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03`와 기존 미커밋 소스가 대상입니다. [브라우저 실행기](browser-feature1-charts.cjs), [고정 fixture](feature1-chart-fixtures.cjs), 문서만 tests 안에 추가했습니다. 제품은 수정하지 않았습니다.

## 실제 경계와 방법

기존 [Feature1 Chrome10개](browser-feature1-flow.md)는 재무/스냅샷이 빈 응답이었습니다. 이번에는 실제 StocksMockPage→Axios→mapper→투자지표/재무 모달/TradingViewWidget/AnalysisResultPanel과 SVG·PDF를 실행하고, 수치가 있는 Public API fixture를 전달합니다. Chrome/React/브라우저 storage·lightweight-charts·html2canvas-pro/jsPDF는 실제이며 Spring/FastAPI/MySQL/Redis/외부 금융·LLM·OAuth·실제 크레딧 차감은 실행하지 않습니다.

시나리오마다 새 BrowserContext, 기본1440×1100·ko-KR, 모바일390×844, 영어 사례는 앱 언어en입니다. Date는2026-09-28 12:00KST로 고정하며 타이머는 계속 실행합니다. 빈 포트를 확인한 뒤 소유 loopback4181/strictPort preview를 기동합니다. service worker 및 다른 origin 요청은 차단하고 모든 `/api/`는 명시된 method/path fixture만 응답합니다. 미등록 API는500/마지막 검사FAIL입니다. 각 사례의 pageerror와 dialog도 검사합니다. 소유 browser/preview만 종료하고4181 listener 부재를 확인합니다.

원본과 격리 복사본의 src/public/설정187개 집합·SHA-256을 비교합니다. 직전 [Feature2 차트 단계](browser-feature2-charts.md)에서 새로 빌드한2165개 모듈의 dist를 재사용하며, 그 최종 run과 입력187개/산출물11개 해시가 같은지도 확인했습니다. 새 source를 오래된 bundle에 대해 검사한 것이 아닙니다. 테스트·fixture 해시와 실제 응답 fixture JSON, 요청 body/query·완료 순서·DOM 관측·PNG/PDF는 매번 새 run에 기록합니다.

입력 A는005930/stockId42/합성기업A, B는000660/stockId43/합성기업B입니다. 조회 price999/888과 실제 캔들 마지막120/240을 구분합니다. A의 두 캔들 종가100→120, 최고130·최저80·거래량10/20이므로 수익률20%·범위50·평균15주입니다. B는 가격과 주요 재무 금액을2배로 정합니다. A의 annual PER9.99와 snapshot11.11, B snapshot22.22는 잘못된 출처·종목의 섞임을 식별하기 위한 값입니다.

분기 재무를 Q3/Q1/Q2 순서로 전달합니다. 매출200/100/150, 영업이익-50/20/30, 순이익-25/10/15이며 Q2의 명시적 영업이익률은18%, Q1/Q3은 null이라20%/-25% 계산 fallback을 거칩니다. 실재무의 품질이나 계산 서버의 정확도를 검증한 것이 아닙니다. 범위 내 정렬·음수/누락 표시·독립 좌표 기대값을 검사합니다.

응답 경합은 임의 시간 sleep 대신 Playwright route gate로 A의 lookup/특정연도 재무/분석을 보류합니다. B의 heading·240 시세·22.22배 지표 확인 후 A를 해제하고, 후속 캔들 또는 잔액 요청 완료·렌더 프레임을 대기하여 최종 DOM을 비교합니다.

## 재현 명령

Windows PowerShell에서:

```powershell
Set-Location C:\qaima\frontend
Remove-Item Env:QAIMA_F1_CHART_CASE -ErrorAction SilentlyContinue
node tests/browser-feature1-charts.cjs
```

WSL에서는 `powershell.exe -NoProfile -Command '...'`를 통해 같은 명령을 실행했습니다. 선택 재현은 `QAIMA_F1_CHART_CASE`에 시나리오 이름 정규식을 지정합니다. 선택/재실행을 전체25개와 합산하지 않습니다.

프로젝트 루트에서 실제 run의 PDF와 문서를 검사합니다:

```bash
PYTHONPATH=frontend/tests/.runtime/pdf-tools PYTHONDONTWRITEBYTECODE=1 .venv_wsl/bin/python -B frontend/tests/inspect-report-pdf.py --artifacts frontend/tests/.runtime/browser-feature1-charts/run-Y8f2W1
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

## 25개 사례

| 번호 | 입력·행동·독립 기대값 | 최종 결과 |
|---|---|---|
| 1 | 초기2024년 재무 조회 관측, snapshot PER11.11배가 annual9.99배보다 우선, 분석POST0 | PASS |
| 2 | snapshot PERnull이면 annual9.99배로 fallback | PASS |
| 3 | 재무 모달 연간/분기/반기 전환, 컬럼2/3/2·전달 순서 및 -50 음수 셀, 닫기 | PASS |
| 4 | 분기 GET500→그 탭 데이터 없음·연간/반기 값 유지 | PASS |
| 5 | 분석 캔들100→120에서20%,최저80/최고130,평균15주·변동폭50%. 요청 KST00:00~23:59:59 | PASS |
| 6 | 같은 요청의 실제 KST 시작/끝 시각과 가격 흐름 리포트의 표시 시각 일치 기대 | FAIL |
| 7 | EMA20 마지막null 이전 유한110,EMA60/120=100/90·상승정렬, Stoch85/70·차이15·과열 표시 | PASS |
| 8 | Stoch K20 경계에서 침체 표시 | PASS |
| 9 | Stoch K50에서 중립 표시 | PASS |
| 10 | 분석 후 보조지표 토글→실제canvas 개수 감소·다시 켜기, 마커 aria-pressed 전환, 추가분석0 | PASS |
| 11 | 분기 시계열 Q1/Q2/Q3 정렬·막대9/이익률점3. 매출 막대(y,height)=(124,100)/(74,150)/(24,200), 음수 영업이익 y224/height50. hover Q3·유한 좌표 | PASS |
| 12 | Q3 순이익null→해당 막대만 제외한8개·나머지 유한 좌표 | PASS |
| 13 | 2026-01-01~09-24 분석 후 재무 periodQ/years3 요청 | PASS |
| 14 | 2023-01-01~2026-09-24 분석 후 periodH/years4 요청 | PASS |
| 15 | 2018-01-01~2026-09-24 분석 후 periodA/years5 요청 | PASS |
| 16 | 분석200 이후 분기 재무500→핵심 설명 유지·재무 차트 없음 | PASS |
| 17 | A lookup gate→B 검색 완료→A 해제 후 B heading 유지 기대 | FAIL |
| 18 | A 재무 gate→B 시세/지표 완료→A 해제 후 B22.22배와 종목 일관성 유지 기대 | FAIL |
| 19 | A 정상조회 후 B의 재무500→B 화면에 A11.11배가 남지 않아야 함 | FAIL |
| 20 | A 분석 gate→B 검색→A 해제. B 분석0회이며 A 분석이 B 화면에 붙지 않아야 함 | FAIL |
| 21 | 영어/전문가 분석의 languageCode en·investLevel 전문가, 영어 재무 제목·유한 차트 | PASS |
| 22 | 모바일 분석 완료·확대 모달에서 document 가로넘침 없음·PNG | PASS |
| 23 | 희소 지표 입력의 실제 PDF 다운로드·이름/구조·추가분석0·복제 리포트의 본체CSS 보존 | FAIL |
| 24 | 채워진 지표/재무 입력의 같은 PDF 검사·재무차트1개/마지막 결론 캡처 관측 | FAIL |
| 25 | 전체 Public API의 미등록 요청0 | PASS |

## 결함·관측과 근거

### FRONT-F1-SEARCH-RACE-001 — 이전 검색 결과가 최신 종목을 덮음

lookup과 재무 gate 두 경로입니다. [StocksMockPage](../src/pages/StocksMockPage.tsx)의 handleSearch는 await 뒤 현재 검색 세대를 확인하지 않습니다. A lookup을 늦게 해제하면 검색창000660 상태에서 heading이 합성기업A로 바뀝니다. A 재무만 보류했다가 해제하면 B22.22배가 A11.11배로 바뀌고, A의 loadCandles가 다시 실행되어 종목코드·가격도 영향을 받습니다. 현재 사용자가 선택한 종목과 데이터의 일관성 문제이며 다른 사용자 데이터 접근 증거가 아닙니다.

### FRONT-F1-STALE-FINANCIAL-001 — 새 종목 조회 실패 후 이전 지표 유지

같은 페이지의 검색 초기화에 sections 초기화가 없고 재무 Promise.all 실패에서도 이전 sections를 지우지 않습니다. A 정상조회 뒤 B 재무500에서 B 이름/240 시세와 A11.11배가 함께 남습니다. 경합이 없는 순차 검색에서도 재현됩니다. B snapshot은 정상 응답이지만 함께 묶인 재무 실패로 새 지표를 반영하지 못하는 경계입니다.

### FRONT-F1-ANALYSIS-RACE-001 — 이전 분석이 새 종목 화면에 붙음

A 분석 응답을 늦게 해제하면 B 검색 뒤에도 setAnalysisResult·가격요약·재무시계열·보조지표를 A 실행이 저장합니다. B 분석POST는0회인데 A 합성 요약이 B 화면의 리포트에 남습니다. 검색 초기화만으로 이미 진행 중인 분석의 후속 저장을 차단하지 못합니다. 실제 차감/보고서 DB의 혼합까지 확인한 것은 아닙니다.

### FRONT-F1-PRICE-DATE-001 — 가격 요약 시각이 실제 요청 범위와 다름

실제 캔들 GET은2026-01-01T00:00:00+09:00부터2026-09-24T23:59:59+09:00입니다. 가격 요약에는 날짜만 있는 from/to를 넘기고 [KST 표시 함수](../src/utils/kst.ts)가 new Date(dateOnly)로 해석해 KST로 출력하므로 양쪽 모두 오전9:00:00으로 표시됩니다. 테스트는 같은 브라우저 Intl로 offset이 명시된 실제 요청 경계의 문자열을 만들어 대조합니다. 캔들 요청 범위가 틀렸다는 주장은 아닙니다.

### FRONT-F1-PDF-STYLE-001 — 분석 PDF 렌더 시 본체 CSS 소실

Feature1은 공용 reportPdf.ts가 아닌 StocksMockPage의 자체 handleDownloadClick을 실행합니다. 동일한 html2canvas-pro/jsPDF 경계에서, 원본 report의 display:flex·본체811규칙이 복제문서를 읽는 시점에는 block·본체CSS없음으로 바뀌었습니다. [Feature2](browser-feature2-charts.md)/[Feature3](browser-portfolio-flow.md)의 유사 현상과 연결하되 별개의 원인이라고 단정하지 않습니다.

getComputedStyle을 원래 함수에 위임하고 복제 문서의 qaima-report-enter를 읽는 시점의 style/rules/text/재무차트 수만 수집했습니다. 반환값이나 제품 파일은 변경하지 않았지만 계측의 타이밍 영향은 남습니다. 희소 입력도 컴포넌트의 빈 지표 카드가 렌더되므로1페이지를 기대하지 않습니다. 파일이 다운로드돼도 스타일 단언을 실패로 유지합니다.

별도 [PDF 검사기](inspect-report-pdf.py)는 수리 없는 파싱·페이지 수·A4 크기 및 비어있지 않은 페이지를 확인하고 모두 PNG로 렌더합니다. 어두운 픽셀 비율 기준0.1%를 유지합니다. 이 검사와 시각 검토는 브라우저 사례 수와 별도로 기록합니다.

## 실행 이력·증거

최종 `run-Y8f2W1`은 **25개 18 PASS / 7 FAIL**, 종료코드1입니다. 실패7개는 위 결함5종의 사례이며 고유 결함7건으로 세지 않습니다. 보조지표를 끄면 실제 canvas는21개에서7개로 줄었고 재활성화·마커 토글도 통과했습니다.

- 최초 `run-xMxJ2Q`: 24개16 PASS/8 FAIL. 두 실패는 테스트 기대값 오류였습니다. 평균 거래량의 단위(15주)·양수 수익률의 표시(20.00%)와 영어 제목의 대소문자(Financial timeline)를 실제 계약에 맞췄습니다. 제품 경합4개/PDF2개 실패는 유지했고 실제 요청과 표시 시각을 비교하는 사례1개를 추가했습니다. 최초 실행과 최종 실행은 합산하지 않습니다.
- 두 실행 모두 미등록 API0·pageerror0·previewStopped true입니다. 최종 응답 fixture24개, 요청458개와 완료 순서, 관측24개, 실패 PNG/TXT7쌍을 보존했습니다. 별도 공통 가드가25번째 사례입니다.
- 입력187개와 bundle11개는 두 실행 모두 직전 Feature2 최종 빌드와 동일합니다. 최종 summary의 실행기/fixture SHA-256도 현재 파일과 대조했습니다. tests 밖 가시 코드·설정·문서924개는 변경/소실/새 파일0입니다.
- 두 실행의 PDF는 각각 희소 입력2페이지42,198,699bytes, 상세 입력3페이지50,202,673bytes입니다. 다운로드 파일명·PDF 구조·추가분석0은 확인했지만 두 스타일 단언은 FAIL입니다. 복제 문서에는 두 경우 모두 마지막 QA-F1-END가 있고 상세 재무 SVG1개도 있습니다.
- 별도 PDF 검사도 두 실행 모두 종료코드1입니다. 파싱/A4/페이지 수는 통과했으나 상세3페이지의 어두운 픽셀 비율0.0593824%가 기준0.1% 미만입니다. 실제 PNG에는 핵심 포인트·리스크·결론 본문이 있으므로 빈 페이지 결함으로 단정하지 않습니다. 낮은 본문 밀도에 대한 검토 신호로 기록하고 기준과 FAIL을 유지합니다.
- 최종5페이지를 시각 확인했습니다. 상세 마지막 페이지는 먼저 확인한 최초 PNG와 해시가 동일함을 별도로 대조했습니다. 본체 스타일 소실로 인한 레이아웃 변화와 상세2→3페이지의 제목/본문 분리·글자 일부 절단을 관측했습니다. 마지막 결론은 보이므로 전체 내용 누락으로 해석하지 않습니다.

증거는 `.runtime/browser-feature1-charts/` 아래 두 run의 summary·fixture·요청·PDF/PNG, `source-audit.json`, `visual-page-comparison.json`, `case-document-audit.json`입니다. 문서57개 대상2개 검사 PASS(2.009초)이며 실행 로그는 `documentation.log`입니다.

## 검증 한계와 다음 범위

이25개는 합성 Public API를 쓰는 UI 경계입니다. 실제 서버/계산/영속성·원천 시세·전체 재무 계약·차감/환불 성공 증거는 아닙니다. 초기2024년 선택은 관측이며 최신연도 자동선택 PASS가 아닙니다. 모달의 응답 순서를 유지하는 동작도 모든 정렬 정책 충족을 뜻하지 않습니다. 보조지표 토글은 실제 canvas/pane과 버튼 상태를 검사했으며 캔들 전체 픽셀·모든 매수/매도 marker 알고리즘 정확도 검증은 아닙니다. 영어/모바일 한 fixture, 연간/반기/분기 대표 범위이며 모든 수준·언어·날짜 경계·접근성·PDF 입력의 조합은 남습니다. 기존 결함은 수정하지 않았습니다.
