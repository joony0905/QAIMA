# Feature 1 실제 Chrome 조회·관심종목·분석 흐름

검증일: 2026-09-25. [browser-feature1-flow.cjs](browser-feature1-flow.cjs)의 Windows Chrome 검사 **10 PASS**, exit0입니다. 실제 React·Router·Axios·메모리 인증·브라우저 storage·차트 컴포넌트를 사용하지만 모든 API는 합성 응답입니다. 제품 소스/설정/의존성은 변경하지 않습니다.

## 실행·격리

Windows PowerShell, frontend 디렉터리:

```powershell
node tests/browser-feature1-flow.cjs
```

[격리 빌드](README.md)와 test 내부 playwright-core를 재사용합니다. StocksMockPage, TradingViewWidget, AnalysisResultPanel, 검색 컴포넌트, API/mapper/errorMessage/번역/App의18개 원본과 복사본을 실행 전에 바이트 비교합니다. 전체 번들 입력의 동일성 검증은 아닙니다.

테스트 소유 preview는 loopback4178·strictPort이며 Chrome은 headless·ko-KR·1440×1100입니다. 각 시나리오는 새 BrowserContext를 사용합니다. service worker와 다른 origin 요청은 차단하고 `/api/`는 명시한 fixture만 응답합니다. 미등록 API는500/최종실패로 처리합니다. 실제 금융·LLM·DB·Redis·크레딧·OAuth·메일 호출은 없습니다. 종료 시 자체 브라우저와 preview를 종료합니다.

## fixture와 실제 호출 순서

`/feature/1?q=005930`으로 진입합니다. 조회 fixture의종목ID는42·회사명은합성기업F1·가격은999입니다. 캔들2개의종가는100→120이므로독립기대값은현재가120·변화+20·등락률+20%입니다. 종목조회값을그대로표시한것과구분합니다.

실제 StocksMockPage→getStockByCode→특정연도/추세재무와snapshot→일봉조회→화면을 실행합니다. 현재 소스의 초기 특정 재무연도는2024로 고정돼 있으며 최신연도를 자동 선택하는 검증이 아닙니다. 재무와snapshot은 빈목록/null이며 재무계산·상세모달 전체 내용 검증은 범위 밖입니다.

분석은 날짜 입력→선택기간 일봉 재조회→Feature1 POST→실제analysisMapper→결과 컴포넌트→3년/Q 재무시계열 GET→잔액 GET 순서를 검사합니다. 합성설명·결과를 표시하는 검증이지 실제정량계산/LLM 품질검증이 아닙니다. 별도 mock경계 테스트와 실제TCP 검증은 백엔드 문서를 참조합니다.

## 10개 검사별 방법

| 검사 이름 | 입력·행동·기대값 |
|---|---|
| guest quote browsing remains public and analysis preserves selected stock login return | refresh401인비로그인조회·시세표시·watchlist요청0. 분석클릭→login, 복귀경로/feature/1?q=005930보존 |
| quote price and daily change use candles rather than stock lookup price | 현재120·+20·(+20.00%)·999표시없음·자동분석0. 초기일봉요청ONE_D·from/to에+09:00 |
| guest watchlist action shows login notice without a write | 별클릭→로그인안내·refresh외변경요청0 |
| watchlist add and remove preserve identifiers and star state | 별클릭→POST stockId42·응답itemId71·별채움/추가안내. 다시클릭→DELETE /71·별비움/삭제안내 |
| failed watchlist addition preserves unselected state | 추가500→공통서버오류·별비움유지·추가요청1 |
| six digit lookup failure keeps chart fallback but cannot save unknown stock ID | 6자리조회404→차트조회/120표시. 별클릭→종목정보안내·추가POST0. fallback에서회사명이정확한지는검사하지않음 |
| snapshot and financial failures do not prevent candle quote rendering | snapshot/재무500→일봉120표시·분석버튼활성·자동분석0 |
| analysis request uses selected dates and renders result before timeline balance refresh | 날짜2026-08-01~09-20입력→일봉KST자정재조회후분석1·Bearer·code/freq/날짜/J/설명true/ko전달. 합성요약표시·이후재무years3/periodQ와잔액재조회 |
| analysis failure maps server error and refreshes balance without an automatic retry | 분석500→원문대신한국어공통오류·분석1·잔액재조회. 현재동작상실행버튼0도기록하며복구UX의적절성PASS로해석하지않음 |
| no unexpected API or uncaught page errors | 미등록API0·pageerror0. 각시나리오dialog0도확인 |

분석 오류 후 잔액 GET이 발생해도 실제 환불 여부는 입증하지 않습니다. 관심종목 UI 상태변경은 실제 영속 저장이나 소유권 검사와 구분합니다. 합성 조회 결과만으로 시세 정확성·휴장일·지원 시장을 주장하지 않습니다.

## 실행 이력·증거

첫 실행 `.runtime/browser-feature1/run-jkXDRN/summary.json`은10개9 PASS·1 FAIL입니다. 테스트가등락률을괄호없는정확문자열로찾았으나실제화면은괄호를표시해timeout됐습니다. 소스와대조하여테스트선택자만수정했습니다. 제품의산술결함으로집계하지않습니다.

최종 `.runtime/browser-feature1/run-6pAiIW/summary.json`은10 PASS·미등록API0·pageerror0·유료0입니다. 자체preview listener의종료도확인했습니다.

시나리오별결과·합성요청method/path/query/body·Bearer유무·예외목록은각run의summary.json에보관하고성공화면은analysis-result.png입니다. 실제토큰/비밀번호/설정값은기록하지않습니다. 스크린샷은전체레이아웃·모바일·시각품질완료증거가아닙니다.

## 관측과 남은 작업

분석 시작 시 `setShowAnalyzeButton(false)`로 실행 버튼을 숨기며, 오류 경로에서 이 상태를 되돌리지 않습니다. AnalysisResultPanel도 showAnalyzeButton=false로 전달받습니다. 분석500 뒤 같은 화면의 실행 버튼이 없는 것을 확인했습니다. 다시 검색 등 다른 복구 경로의 존재와 별개로 사용자 복구 UX 검토가 필요한 관측이며 제품은 미수정입니다.

실제Spring/JWT·원천API·캐시·차감/환불·보고서저장, 종목검색자동완성/복수검색경합·입력날짜오류·중복분석경합, 실제재무/지표수치·상세모달, 캔들추가로딩/모든차트옵션, 설명수준4단계·영어/모바일·분석직후PDF는남았습니다. Swagger관련전체흐름완료로집계하지않습니다.
