# Feature2 차트·nullable 응답·분석 직후 PDF 검증

검증일: 2026-09-28. `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03`와 기존 미커밋 소스가 대상입니다. 제품 소스는 바꾸지 않았고 [브라우저 실행기](browser-feature2-charts.cjs), [합성 입력](feature2-chart-fixtures.cjs), 문서와 PDF 검사 변경은 tests 내부입니다.

## 방법·실제 경계

기존 [Feature2 Chrome18개](browser-feature2-flow.md)의 빈 시계열 경계를 확장합니다. 실제 Chrome/React/Router/Axios·SVG/DOM·html2canvas-pro/jsPDF를 실행하고, Public API는 Playwright route의 합성 응답입니다. Spring·FastAPI·DB·Redis·외부 제공자·실제 차감은 호출하지 않습니다. 공매도 null과0.25 입력은 [실제 SQL/HTTP 검증](../../tests/isolated-short-selling.md)에서 관측한 필드 형태를 합성 fixture로 재현한 것이며, 두 실행을 하나의 서버 연결 E2E로 해석하지 않습니다.

원본 src/public/빌드 설정과 tests/.runtime/build-project의187개 파일 집합·SHA-256을 비교합니다. 이번에 격리 복사본을 다시 Vite build하여2165개 모듈/exit0을 확인했습니다. 기존 큰 chunk/브라우저 메타데이터 경고는 남습니다. 각 run에 입력187개·dist11개·테스트/fixture 파일 해시, 사례별 fixture JSON·요청·완료순서·DOM 관측·오류·PNG·PDF를 남깁니다. 원래18개 실행기와 별도 파일·결과이며 재실행은 중복 합산하지 않습니다.

소유 preview는 빈 포트를 확인한 뒤 loopback4180/strictPort에서 시작합니다. 시나리오마다 새 BrowserContext, ko-KR,1440×1100이며 모바일만390×844입니다. Date는2026-09-28 12:00KST로 고정하고 실제 타이머는 유지합니다. 모든 외부 origin과 service worker를 차단하고 등록되지 않은 API는500 및 최종 검사 실패입니다. pageerror는 해당 시나리오의 실패로 기록하며 별도 중복 실패로 더하지 않습니다. 종료 시 browser/preview를 종료하고 포트가 닫혔는지 확인합니다.

시계열은2026-09-22/23/24 세 날짜입니다. 환율1300/1310/1320, 한국 금리3/3/2.5, 국내외 금리8종, 공매도 거래량 비율10/20/25와 거래대금 비율5/15/20, 외국인100/-50/25·기관-25/50/-10백만원을 사용합니다. 좌표 기대값은 이 수치와 렌더링 영역의 크기로 독립 계산합니다. 종목120 현재가는 기존100→120 캔들 fixture를 사용하며 실제 시세가 아닙니다.

## 재현 명령

Windows PowerShell:

```powershell
Set-Location C:\qaima\frontend\tests\.runtime\build-project
node node_modules/vite/bin/vite.js build
if ($LASTEXITCODE -ne 0) { throw 'Isolated build failed' }
Set-Location C:\qaima\frontend
Remove-Item Env:QAIMA_F2_CHART_CASE -ErrorAction SilentlyContinue
node tests/browser-feature2-charts.cjs
```

WSL에서는 위 명령을 `powershell.exe -NoProfile -Command '...'`로 실행했습니다. 특정 사례는 `QAIMA_F2_CHART_CASE`에 이름 정규식을 지정합니다. 필터 실행은 전체20개와 합산하지 않습니다.

프로젝트 루트에서 생성된 실제 run 경로를 지정합니다:

```bash
PYTHONPATH=frontend/tests/.runtime/pdf-tools PYTHONDONTWRITEBYTECODE=1 .venv_wsl/bin/python -B frontend/tests/inspect-report-pdf.py --artifacts frontend/tests/.runtime/browser-feature2-charts/run-gwm3LA
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

## 20개 사례

최종 `run-gwm3LA`는 **20개13 PASS/7 FAIL**, exit1입니다. 결과는 아래 표에 기록합니다.

| 번호 | 최종 결과 | 수행·독립 기대값 |
|---|---|---|
| 1 | PASS | 요약의 거시 sparkline7개, 환율1,320 표시, 분석POST0, SVG 수치 유한 |
| 2 | PASS | 금리·환율 탭의 차트3개/선8개, hover guide1개, 유한 좌표와 실제 PNG |
| 3 | FAIL | KR3Y의22/23/24일과 KR10Y의23/24일을 같은 차트에 표시. 같은23일의 x좌표 일치 기대 |
| 4 | PASS | 종목/시장 수급 차트2개, 첫 차트 막대6개. 높이210의 첫 외국인100은 y18/높이79, 기관-25는 y97/높이19.75. hover100/-25 |
| 5 | PASS | 외국인 첫 값null에서 수급 hover '-'·유한 좌표, 페이지 유지. null을0 높이로 그리는 현 정책은 별도 관측 |
| 6 | PASS | 공매도 높이260/최소5/최대25에서 거래량 점(28,174)/(380,70)/(732,18), 각 series3점·hover23일·25%표시. Feature2 추가 요청0 |
| 7 | PASS | 공매도 단일 시계열에서 양쪽1점과 유한 좌표 |
| 8 | PASS | 거래대금 중간null에서 거래량3점/금액2점, 유한 좌표 |
| 9 | PASS | 두 비율 모두null인3일은 명시적인 추이 데이터 없음 안내 |
| 10 | FAIL | FINRA형 거래대금·비율null 카드 수신 후 기본 요약 화면 유지·pageerror0 기대 |
| 11 | FAIL | 거래량 비율null 카드의 공매도 탭 클릭 후 페이지 유지·pageerror0 기대 |
| 12 | FAIL | FINRA에서 관측한1/4→0.25 필드를 표시할 때25.00% 기대 |
| 13 | FAIL | 공개 조회는 정상, 분석 응답만 FINRA 거래대금 비율null로 제공→분석 설명과 화면 유지 기대 |
| 14 | PASS | 정상 분석 결과의 환율/기준금리/국내국채/미국국채/공매도 모드 전환. 선1/2/2/3 및 공매도 좌표, 추가분석0 |
| 15 | PASS | 크게 보기→분석문2개/모달 닫기→1개, 분석POST1 유지 |
| 16 | PASS | 390×844에서 채워진 금리/수급/공매도/요약과 분석 결과의 document 가로넘침 없음 |
| 17 | FAIL | 단순 분석 직후 PDF 실제 다운로드·이름·구조·캡처 정리·추가 업무POST0·렌더러의 원본CSS 보존 |
| 18 | FAIL | 상세 분석 PDF의 같은 검사. 캡처 DOM의 거시5+종목수급1차트·마지막 결론도 관측 |
| 19 | PASS | 실제 PDF 도중 canvas.toDataURL에 합성 예외→캡처 class 제거·일반 차트 버튼 복귀·다운로드 버튼 사용 가능·추가분석0 |
| 20 | PASS | 모든 Public API 요청이 등록된 fixture와 일치 |

## 결함·관측 근거

### FRONT-F2-SHORT-NULL-001 — 실제 nullable 필드가 화면을 비움

Java DTO의 BigDecimal과 SQL은 null을 허용합니다. FINRA는 금액 자체가 없는 실제 경로이며, 앞선 격리 SQL 검사에서 공개 응답의 null을 확인했습니다. 프론트 [ShortSellingMetrics 타입](../src/types/feature2.ts)은 해당 필드를 number로만 선언합니다.

[외부 패널](../src/components/Feature2ExternalFactorPanel.tsx)의 기본 요약324행과 공매도617/620행, [분석 결과](../src/components/AnalysisResultPanel.tsx)의 비율 출력은 null 검사 없이 toFixed를 호출합니다. 세 사례에서 `Cannot read properties of null (reading 'toFixed')` pageerror가 발생했습니다. 거래대금null은 공매도 탭을 열기 전 검색 직후 기본 요약에서도 전체 화면을 비웁니다. 거래량null은 공매도 탭에서, 분석 응답의 금액null은 분석 결과 렌더에서 재현됩니다. React 페이지 전체의 본문 소실과 오류를 함께 기록했습니다.

### FRONT-F2-MACRO-DATE-001 — 날짜 대신 배열 순서로 정렬된 비교 차트

[MultiLineTrendChart](../src/components/Feature2TrendCharts.tsx)는 가장 긴 배열 길이에 맞춘 index를 x좌표로 쓰고, 날짜 label은 첫 series에서 가져옵니다. KR3Y의23일은 x380, KR10Y의 같은23일은 x34로 표시됐습니다. 다른 시점의 금리를 같은 날짜처럼 비교하게 되는 문제이며 제공자 오류를 전제로 하지 않습니다. 이번 입력은 날짜순이고 누락일만 다릅니다. 숫자 값과 서로 다른 길이의 원본 fixture를 함께 보존했습니다.

### SHORT-RATIO-UNIT-001의 브라우저 증거

FINRA1/4의 저장·공개값0.25를 합성 응답으로 전달했을 때 화면이0.25%를 표시했습니다. 앞선 SQL/카드 단위 불일치에 표시 영향의 증거를 더한 것이며 새로운 독립 결함으로 중복 계산하지 않습니다. 정상 퍼센트25인 대조군은25.00%입니다. 실제 외부 제공자 호출이나 경제적 원천 단위 검증은 아닙니다.

### FRONT-F2-PDF-STYLE-001 — 렌더 시 본체 CSS 소실

실제 PDF 렌더러의 getComputedStyle을 원래 함수에 위임하면서, 복제 문서의 캡처 root를 읽는 시점만 기록합니다. 원본 report는 display:flex·본체811규칙인데 실패 시 읽은 복제 root는 display:block·본체 규칙 없음입니다. 제품 파일과 렌더러 반환값을 바꾸지 않았지만 계측의 타이밍 영향은 남습니다. 기존 [Feature3 PDF의 OBS-F3-PDF-STYLE-001](browser-portfolio-flow.md)과 같은 공용 유틸 경계를 Feature2에서도 관측했습니다. 서로 다른 원인이라고 단정하지 않습니다.

기본/상세 두 PDF 파일은 실제로 다운로드되며 filename·PDF header/EOF·캡처 class 정리·추가 분석/업무 요청 없음은 확인됩니다. 그러나 파일 생성만으로 시각 품질을 PASS로 처리하지 않습니다. 상세 캡처 DOM에는 거시5개+종목수급1개 차트와 마지막 결론이 있으며, 외부 패널의 시장수급 차트는 원래 분석 리포트에 포함되지 않습니다.

별도 [PDF 검사기](inspect-report-pdf.py)는 수리 없는 파싱·A4 크기·페이지 수와, grayscale에서160미만 픽셀 비율0.1% 초과를 검사합니다. 이번에 빈 페이지 실패가 있어도 모든 페이지 PNG와 검사 JSON을 먼저 저장하도록 보강했으며 기존 임계값과 실패 exit는 유지했습니다. raster PDF의 한글·레이아웃은 생성 PNG를 별도로 확인합니다.

## 실행 이력·최종 증거

| 실행 | 전체 결과 | 구분 |
|---|---|---|
| run-EcBBlv | 20개10 PASS/10 FAIL | 테스트의 수급 높이220 가정1개와 공매도 높이220 selector/좌표3개가 실제210/260과 달랐습니다. 단순 PDF 스타일은 이 실행에서만 통과했습니다. |
| run-gGKykG | 20개12 PASS/8 FAIL | 위 높이4개를 보정했습니다. 공매도 hover 검사 중 별도 캔들의 과거자료 요청1회가 발생하여 전체 API 수 불변 단언이 실패했습니다. 기본/상세 PDF 모두 스타일 실패입니다. |
| run-gwm3LA | **20개13 PASS/7 FAIL** | 차트 조작의 요청 단언을 Feature2 경로로 한정하고 캔들 요청을 별도 관측했습니다. 제품 실패7개가 남았습니다. |

기본 요약에서 금액null이 즉시 오류를 내므로 최초/두 번째 실행의 해당 사례는 검색 가격/제목 대기 timeout으로 실패했습니다. 마지막 실행에서는 카드 수신과 pageerror를 직접 대기하도록 바꿔 명시적인 null 오류 단언으로 확인했습니다. 실패를 정상 동작으로 인정한 보정은 아닙니다.

상세 PDF의 차트 개수 단언도 실제 AnalysisResultPanel의 종목수급1개에 맞춰 거시5+수급1=6으로 정리했습니다. 이전 코드의7개 단언에는 스타일 실패 때문에 도달하지 않았고, 그 수를 제품 결함으로 집계하지 않습니다. 최종 completeRichDom true와 캡처 원문은 summary에 있습니다.

세 실행 모두 source187개 일치·dist11개 해시·미등록API0·pageerror3개(공매도 null 세 사례)·previewStopped true입니다. 모든 run에 fixture19개와 요청/응답 상태·개별 실패 PNG/TXT를 남깁니다. `scriptSha256`과 `fixtureSha256`은 실행별 테스트 버전을 특정합니다.

최종 PDF는 단순1페이지12,404,927bytes, 상세4페이지46,838,520bytes입니다. 두 파일 모두 수리 없이 파싱되고 페이지 수/A4 크기가 일치합니다. **빈 페이지 검사는 FAIL**이며 상세4페이지의 어두운 픽셀 비율은0.00009565(약0.009565%)로 임계0.1%보다 낮습니다. 이 결과는 브라우저20개와 별도 검사입니다. 첫 run에서 검사기가 해당 페이지에서 즉시 중단한 이력도 남겼고, 모든 페이지를 저장하도록 보강한 뒤 세 run을 검사해 모두 같은 마지막 페이지 실패를 확인했습니다.

생성 PNG에서 단순 리포트의 카드/헤더 레이아웃 소실, 상세 리포트의 본문 스타일 소실과 페이지2→3 사이 차트 제목 분할, 페이지3→4 사이 마지막 결론 글자 분할을 육안 확인했습니다. 상세 마지막 페이지에는 잘린 결론 일부와 워터마크만 남습니다(`OBS-F2-PDF-PAGINATION-001`). 원본 DOM에 결론이 존재한다는 것만으로 출력 페이지의 가독성이 보장되지 않습니다. 최초 상세1/4페이지와 최종 해당 PNG의 SHA-256도 같으며 `visual-page-comparison.json`에 기록했습니다. 나머지 최종3개 PNG도 직접 확인했습니다.

모든 산출물은 `frontend/tests/.runtime/browser-feature2-charts/` 아래입니다. 각 run의 `summary.json`, `fixture-*.json`, `feature2-*.pdf`, `pdf-inspection.json`, `pdf-blank-pages.json`, `feature2-*-page-*.png`가 근거입니다. tests 밖 코드/설정/문서924개의 변경·소실·새파일0은 `source-audit.json`에 있습니다. 문서 검사와 사례 표 대조 결과도 같은 디렉터리에 보존합니다. 제품 결함을 고치거나 전체 QA를 완료한 결과는 아닙니다.

## 남은 범위

이20개는 선택한 거시/수급/공매도 UI·PDF 경계입니다. Peer/산업 지수 전체 차트·키보드/접근성·모든 언어/테마, 전체 데이터 크기·모든 날짜/결측 조합, PDF 모바일/폰트/페이지 단락 분할의 전체 검증은 아닙니다. 실제 API/DB 스냅샷·분석 계산·원장/환불·외부 원천 결합도 남습니다. PDF 예외 이후 버튼 복귀 관측은 사용자에게 실패 안내가 충분하다는 판정이나 모든 예외 후 재다운로드 성공 증거는 아닙니다. 제품 결함은 수정하지 않았습니다.
