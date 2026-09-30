# 실제 Chrome 로그인·로그아웃·OAuth 화면 흐름

검증일: 2026-09-25. [browser-auth-flow.cjs](browser-auth-flow.cjs)의 **11개 검사 PASS**, exit0. Windows Chrome headless·1440×1000·ko-KR, 격리 Vite preview에서 실제 React 화면과 API client를 실행했습니다. 제품 소스/설정/의존성은 변경하지 않았습니다.

## 실행과 안전 경계

Windows PowerShell, frontend 디렉터리:

```powershell
node tests/browser-auth-flow.cjs
```

기존 `tests/.runtime/build-project`의 빌드와 `tests/.runtime/browser-tools`의 playwright-core를 사용합니다. 실행 전 로그인/OAuth/스토어/Sidebar/App 관련10개 원본파일과 빌드 복사본의 바이트가 같은지 확인합니다. 이는 해당 입력들의 일치 검사이며 모든 번들 입력·빌드 재현성 증명은 아닙니다. 소스가 다르면 기존 [격리 빌드 명령](README.md)으로 다시 빌드해야 합니다.

preview는별도 loopback4176/strictPort입니다. 기존서버를종료하지않으며자신의preview가종료되어있으면검사를중단합니다. 검사끝에자체Chrome/context·preview를종료합니다. 각scenario마다새BrowserContext를사용하고service worker를차단합니다.

모든 `/api/` 요청은 Playwright route에서 명시된 합성 JSON/HTTP상태로 응답합니다. 다른origin은 abort하고 미등록API는500과최종실패로처리합니다. 실제로그인·OAuth제공자·서버쿠키·메일·실크레딧차감·유료호출은없습니다. 화면에합성계정과토큰이보여도실계정연동PASS가아닙니다.

주요실행경로:

- LoginPage form → auth.login → 실제Axios/apiClient → 합성응답 → tokenStore/userStore → Router → Dictionary/Sidebar.
- Sidebar account menu → auth.logout → 합성200/500 → finally의세스토어정리 → 비로그인Sidebar.
- OAuth2SuccessPage callback → 실제bootstrapAccessToken → 추가정보화면/오류LoginPage → URL정리.

실제브라우저DOM·form validation·React state·Router·local/sessionStorage·Axios를사용합니다. 네트워크서버의인증/쿠키/JWT는대체했습니다.

## 검사별 방법·기대값

| 검사 이름 | 입력·행동·검증 | 결과 |
|---|---|---|
| empty form shows local error without login request | 빈폼submit→한국어필수입력안내·login요청0 | PASS |
| native invalid email prevents network submit | not-an-email·합성password→브라우저validity.typeMismatch=true·login요청0 | PASS |
| server error is rendered as text and button recovers | 400의message에img/onerror문자열. 메시지는그대로텍스트, form img0·주입플래그false·버튼재사용가능·login1 | PASS |
| 423 uses localized locked-account fallback | 서버message없는423→정확한한국어잠금안내 | PASS |
| successful login disables submit then restores saved route and memory auth | refresh401후폼입력,login응답보류중submit비활성. 성공해제후저장된/feature/4?qa=return복귀·키삭제·사용자정보storage·token문자열은두storage에없음·dictionary에Bearer·login1 | PASS |
| logout 200 cleans user and balance on public page | 합성인증으로사전진입→내정보메뉴→로그아웃. login버튼표시·현재/feature/4유지·user/balance저장키제거·logout1 | PASS |
| logout 500 cleans user and balance on public page | 같은UI에logout500. finally정리로비로그인표시·키제거·현재공개경로유지 | PASS |
| OAuth profileRequired reaches protected additional-info form | callback?profileRequired=true·refresh성공·profile.status=profile_required. 보호된추가정보화면·fixture이름미리입력·users/me Bearer·App+callback refresh총2 | PASS |
| OAuth error renders mapped message and removes only error parameters | callback error/Google/provider email unverified→한국어정형오류·/login으로복귀·오류query없음. 이fixture엔다른query가없어서임의query보존전체는미검증 | PASS |
| OAuth missing refresh token shows error then returns to login | callback의refresh401→실패안내→제품1.8초timer후/login·입력폼표시 | PASS |
| no page errors unexpected API or business mutations | 전scenario의pageerror0·미등록API0·dialog0, GET외에는합성refresh/login/logout만발생 | PASS |

로그인검사는폼의실제입력email/password가요청으로전달됐는지메모리에서확인합니다. artifact에는password/토큰값을저장하지않고method/path/Bearer유무/language만기록합니다. HTML문자열escape검사는해당로그인오류표시경계만검사한것이며사이트전체XSS보증이아닙니다.

## 실행 이력·증거

첫실행 `.runtime/browser-auth/run-Thx7rF/`는11개중8 PASS·3 FAIL이었습니다. 테스트가계정버튼을`내 정보`로찾았지만실제번역문구는`내정보`여서성공로그인/로그아웃2개의selector가timeout됐습니다. 제품오류로집계하지않고테스트selector만수정했습니다.

재실행 `.runtime/browser-auth/run-9bckl4/summary.json`은11 PASS이며scenario별결과와합성API요청기록을보관합니다. `logged-in-dictionary.png`에서로그인후사전·내정보·합성5토큰표시를확인했습니다. 캡처는페이지전환중옅은상태이며외부이미지차단으로이미지누락도있으므로완성화면의시각품질PASS증거로삼지않습니다.

## 남은 검증

이검사는로그아웃중늦은refresh/잔액응답을의도적으로지연하지않았습니다. 따라서[FRONT-AUTH-LATE-001/FRONT-BALANCE-LATE-001](auth-client.md)의VM회귀2개FAIL은여전히유효하며해결된것이아닙니다.

실제서버세션/JWT/HttpOnlycookie·OAuth제공자동의/계정연결은미실행이며사용자의직접로그인이필요합니다. 추가정보저장submit·보호페이지로그아웃후/main·다른언어·모바일·여러탭·스토리지권한거부·장기timeout·실서버동시refresh도남았습니다. Swagger전체업무흐름완료로표기하지않습니다.
