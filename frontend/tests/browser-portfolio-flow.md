# Feature 3 실제 Chrome 입력·저장·비용·분석 화면

검증일: 2026-09-25. [browser-portfolio-flow.cjs](browser-portfolio-flow.cjs) 최신 자동검사는 **15개 중14 PASS·1 FAIL**, exit1입니다. 고급 PDF 렌더러가 본체 CSS 없이 스타일을 읽는 회귀를 추가하여 실패로 검출했습니다. Windows Chrome의 실제 React·Axios·Router·브라우저 저장소를 실행하지만 HTTP는 모두 합성응답입니다. 실제서버·DB·Redis·LLM·크레딧원장은호출하지않습니다.

## 실행과 경계

Windows PowerShell, frontend 디렉터리:

```powershell
node tests/browser-portfolio-flow.cjs
```

[격리 빌드](README.md)의 `tests/.runtime/build-project`와 test내부playwright-core를재사용합니다. PortfolioMockPage·API·riskProfile/errorMessage·StockSearchCell·번역/App·PDF유틸/헤더 등14개원본과복사본을바이트비교합니다. 모든번들입력의무결성증명은아닙니다. 제품코드·설정·원본의존성은변경하지않았습니다.

자체 Vite preview는loopback4177·strictPort이며새BrowserContext마다ko-KR·1440×1100·service worker차단을적용합니다. 다른origin은abort하고모든`/api/`는명시된합성응답으로가로챕니다. 미등록API는500과최종실패로처리합니다. 자체Chrome/context·preview만종료합니다.

실제흐름: PortfolioMockPage→portfolio.ts/apiClient→합성응답→React state/기본·고급결과입니다. 저장은실제확인창→PUT직렬화→합성응답반영이며SQL영속화가아닙니다. 위험변경은실제localStorage와reload로검사하고공식프로필변경요청이없음을확인합니다.

기본fixture는합성기업005930·2주·평균단가10000·현금5000·위험점수0.6입니다. 응답변동성0.15는고정합성값으로브라우저표시15.0%만검증합니다. 6개월요청126/190일은검사하지만응답의고정252일품질수치를실제재계산결과로해석하지않습니다. 투자수준별설명·최적화산술·유료LLM은범위밖입니다.

## 검사별 방법·기대값 — 최신 고급 PDF 1개 FAIL, 나머지 PASS

| 검사 이름 | 입력·행동·확인 |
|---|---|
| guest browsing avoids protected portfolio APIs and preserves login return | refresh401·잠금안내. 분석클릭→login·복귀경로/feature/3보존. portfolio/macro/risk/analysis요청0 |
| empty portfolio is rejected before preview or analysis | 빈보유목록·위험0.6→최소1종목안내·preview/analysis0 |
| missing risk disables analysis until local input is supplied | 기본위험null→버튼비활성, 로컬0.7입력/blur→활성·분석0 |
| risk and manual cash changes persist locally without profile writes | 위험0.8→자동현금0.2, 수동현금0.35→위험0.4에도0.35. reload후0.40/0.35·manual=true. refresh외변경요청0 |
| save cancellation sends no PUT and confirmed save filters blank row | 확인아니오→PUT0. 빈행추가후예→유효1행/현금5000만PUT·성공문구·빈행제거 |
| failed save retains the form and re-enables confirmation | PUT500→한국어공통서버오류·확인창/입력보존·예버튼활성·PUT1 |
| core request preserves options and renders basic and advanced result tabs | 6개월→보유/현금/위험·maxCash0.4·126/190일·수정종가·LedoitWolf·연율252·ko·설명true·overlay빈목록·Bearer·analysis1/preview0. 고급탭→분석진단생성/기본영역제거, 기본복귀→역전·15.0%표시·잔액재조회 |
| analysis failure renders error restores action and refreshes balance | analysis500→공통오류·버튼복원·요청1·잔액GET증가. 실제환불검증아님 |
| overlay preview force refresh updates displayed cost and submitted policies | 기술HIT0·뉴스MISS1·산업AVAILABLE0·core1→2credit. 기술force선택→3credit. 정책checkbox2개만표시. 확인→기술FORCE_REFRESH/뉴스·산업REUSE_AVAILABLE·preview1/analysis1 |
| cancelled overlay preview does not analyze | 옵션3개→preview→취소→영역제거·preview1/analysis0 |
| preview failure never falls through to paid analysis | preview500→공통오류·버튼복원·preview1/analysis0. 실제유료전송없음 |
| pending analysis blocks duplicate submission until response | DOM button.click2회를동일JS작업에서실행. 응답보류중분석1·버튼비활성, 해제후결과탭·분석1 |
| analysis result basic PDF downloads without another analysis or write | 기본분석완료→실제PDF버튼→download파일저장·정확한파일명/헤더/EOF/페이지존재·capture class정리·추가분석/쓰기0·버튼재활성 |
| analysis result advanced PDF downloads without another analysis or write | 합성프론티어3점·후보4개·진단→고급탭/SVG→PDF파일/정리/추가요청검사. 최신추가단언인렌더러CSS보존에서FAIL: 원본flex/811규칙→복제block/2규칙 |
| no unexpected API or uncaught browser errors | 전체미등록API0·pageerror0. 각scenario의dialog0도확인 |

산업강제갱신선택없음·기술HIT에만정책선택2개를검사했습니다. 실제캐시판정·TTL·차감은합성경계밖이며잔액GET발생은실제환불을증명하지않습니다.

## 실행 이력·증거

첫실행 `.runtime/browser-portfolio/run-Nug11x/`는13개8 PASS·5 FAIL입니다. 오류2개는500에서원문을표시한다고잘못기대했고옵션3개는checkbox접근성이름이짧은라벨과정확히같다고가정한selector문제입니다. 실제errorMessage.ts의공통한국어문구와label내부안내버튼기반선택자로테스트만수정했습니다.

두번째 `.runtime/browser-portfolio/run-OKCnXJ/summary.json`은13 PASS입니다. 탭별실제영역생성/제거와15.0%표시를보강한최종 `.runtime/browser-portfolio/run-5KN40R/summary.json`도13 PASS입니다. `core-result.png`는보조화면증거입니다. 초기fullPage캡처에는스크롤진입애니메이션으로숨겨진영역이있어빈영역을데이터소실로판정하지않습니다. 전체시각품질PASS는아닙니다.

artifact는결과·합성요청method/path/body·Bearer유무·오류목록을보존하며실제인증값은없습니다. 실제유료호출0·DB/Redis쓰기0입니다.

## 남은 범위

실제Spring/JWT/영속저장·동시편집·차감/환불·Redis/가격·LLM전체연동, 실계정OAuth, 종목검색후선택·해외시장·모바일/영어·다른기간/수준, 음수/극단값·저장소권한오류, 모든최적화차트상호작용·PDF스타일안정성은남았습니다. 기존Spring환불누락과Node늦은응답FAIL은해결되지않았습니다. Swagger전체완료로표기하지않습니다.

## 분석 직후 PDF — 생성 PASS와 시각 검증의 구분

`handlePortfolioDownloadClick` → `waitForPdfCaptureReady` → `downloadElementAsPdf` → html2canvas/jsPDF를 실제 실행합니다. PDF 모드에서는 제품의 SVG 파이차트를 사용합니다. 고급 fixture의 프론티어3점·최소분산/최대샤프/위험배분/효용최대4개 후보는 화면용 합성 수치이며 최적화 정답/현실 투자 포트폴리오가 아닙니다.

PDF 추가 첫 실행 `.runtime/browser-portfolio/run-eqnvpg/`는15개PASS·기본1페이지20,293,045bytes/고급1페이지11,542,235bytes입니다. 두 PDF의한글·15.0%·기본80/20파이·워터마크를확인했습니다. 이때고급입력은차트없는진단만포함했습니다.

차트입력보강후 `.runtime/browser-portfolio/run-C328ig/`도자동15개PASS이나기본2페이지29,364,647bytes/고급4페이지72,581,308bytes가생성됐습니다. 독립파서에서는총6페이지모두A4·수리없이열림·어두운픽셀0.1%초과검사가통과했습니다. 전체6페이지를육안확인하니한글/수치/파이/프론티어는있지만카드배치·스타일이사라졌고문장중간에서페이지가나뉩니다. 같은기본fixture의이전1페이지와다른결과여서안정적품질PASS로판정하지않습니다. 제품문제/격리브라우저환경/캡처시점중원인은미확정입니다.

다운로드성공·PDF파싱PASS만으로스타일보존을입증하지못한다는한계를명시합니다. 파일크기도수십MB이며성능기준통과를주장하지않습니다. 후속진단으로테스트에스타일시트요청성공/실패·캡처전스타일규칙수/너비·화면사진을추가했습니다. 제품코드는미수정입니다.

진단추가실행 `.runtime/browser-portfolio/run-UpWV8e/summary.json`도자동15 PASS·기본2/고급4페이지입니다. 캡처전원본문서는CSS규칙811개·report display=flex·너비약1089/1104px였고본체CSS요청은두번모두완료됐습니다. 외부폰트CSS는테스트의외부origin차단으로실패했습니다. 이기록만으로외부폰트차단이전체스타일손실의원인이라고확정할수없습니다. `pdfDiagnostics`, `before-basic-pdf.png`, `before-advanced-pdf.png`와최종PDF/페이지PNG를보존했습니다. 관측사항은 [OBS-F3-PDF-STYLE-001](../../docs/findings.md)로추적합니다.

### 렌더러의 실제 스타일 읽기 계측

추가 진단은 별도 프로필입니다. `QAIMA_PDF_DIAGNOSTIC=1`이면 기존 화면12개를 실행하지 않고 PDF2개와 최종 요청/오류 검사만 실행합니다. 전체15개 회귀와 별개이며 합산하지 않습니다. `QAIMA_PDF_EMPTY_EXTERNAL_CSS=1`은 외부 stylesheet 요청에 네트워크 대신 빈 CSS의200응답을 주는 통제조건입니다. 나머지 외부요청은 여전히 차단합니다.

처음에는20ms 간격으로 복제 iframe의CSS상태를 읽었습니다. `.runtime/browser-portfolio/run-RV5i5K/`에서 기본복제문서는block/규칙2, 고급은flex/811+2였고 PDF는2/2페이지였습니다. 외부CSS를빈200으로바꾼 `run-h3Kdxl/`에서는모든stylesheet요청이완료돼도고급스타일손실/4페이지가남았습니다. 단, 같은 실행의기본PDF는1페이지정상인데간격표본은초기block을기록했습니다. 따라서간격표본만으로최종캡처상태를판정하지않습니다.

이후 테스트가원래`window.getComputedStyle`을그대로호출해반환하면서, 복제문서의보고서root를읽는호출만기록하도록계측했습니다. PDF가끝나면함수를복구하고타이머를정리합니다. 이계측은제품파일수정이아니지만실행타이밍에영향을줄가능성은있으므로무계측관측과함께해석합니다.

빈CSS200조건의 `run-SKwuhp/summary.json`에서실제렌더러읽기는기본flex/811+2규칙(1페이지), 고급block/2규칙(4페이지)이었습니다. 요청성공과관계없이렌더러가복제문서의본체CSS가준비되지않은상태를읽는경로가확인됐습니다. 단순외부폰트오류만으로설명되지않습니다. 격리설치의html2canvas-pro2.0.2소스에서iframe준비대기는body존재/readyState를사용하고스타일은이후`getComputedStyle`로읽는것도확인했습니다. 전체브라우저/제공환경에서동일원인이라고일반화하지않습니다.

현재PDF검사에는실제렌더러호출기록의존재·원본문체CSS규칙수보존·보고서display보존단언을추가했습니다. 간격표본을단언기준으로쓰지않습니다. 단순다운로드PASS로스타일손실을숨기지않으며발견된제품/라이브러리경로는수정하지않았습니다.

보강후전체실행 `run-PjKufe/summary.json`은15개14 PASS·고급PDF1 FAIL, exit1입니다. 기본렌더러는flex/811+2규칙·1페이지, 고급은block/2규칙·4페이지입니다. 따라서과거15 PASS는스타일단언추가전기록으로만해석합니다. 페이지파싱/A4/비어있지않음검사는이단언과별도입니다. 캡처타이밍에따라실패대상이달라질수있으며한번의정상기본PDF로기본모드전체안정성을보장하지않습니다.

진단재현(Windows frontend):

```powershell
$env:QAIMA_PDF_DIAGNOSTIC = '1'
$env:QAIMA_PDF_EMPTY_EXTERNAL_CSS = '1'
node tests/browser-portfolio-flow.cjs
Remove-Item Env:QAIMA_PDF_DIAGNOSTIC,Env:QAIMA_PDF_EMPTY_EXTERNAL_CSS
```

전체15개를실행할때는위두변수를해제합니다. 실패시에도다운로드파일과pdfDiagnostics는summary에남기며최종종료코드는1입니다. PDF읽기/A4/비어있지않음검사는스타일단언실패와독립입니다.

WSL 독립파서 재현:

```bash
PYTHONPATH=frontend/tests/.runtime/pdf-tools PYTHONDONTWRITEBYTECODE=1 .venv_wsl/bin/python -B frontend/tests/inspect-report-pdf.py --artifacts frontend/tests/.runtime/browser-portfolio/run-C328ig
```

[inspect-report-pdf.py](inspect-report-pdf.py)에선택인자를추가했으며기존기본경로의저장리포트4개/6페이지도재검사PASS입니다. 디렉터리는test의.runtime하위만허용하고manifest의PDF상위경로탈출도거절합니다. `--artifacts /tmp`의exit2로허용범위밖디렉터리거절을확인했습니다. 이한번의거절검사는모든악성manifest경계검증을대체하지않습니다. 출력은pdf-inspection.json/페이지PNG이며래스터PDF이므로한글/시각품질은직접렌더링확인이필요합니다.
