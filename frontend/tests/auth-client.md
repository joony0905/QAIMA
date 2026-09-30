# 인증·잔액 클라이언트 동시성 검증

검증일: 2026-09-25. [auth-client.test.cjs](auth-client.test.cjs) 신규20개: **18 PASS·2 FAIL**. 기존 contracts5개와 합쳐 Windows Node 실행 **25개:23 PASS·2 FAIL**, SKIP/cancelled0, 1056.3453ms, exit1입니다. 제품 코드/의존성/빌드 설정은 수정하지 않았습니다.

## 실행·경계

Windows PowerShell, frontend 디렉터리:

```powershell
node --test tests/contracts.test.cjs tests/auth-client.test.cjs
```

TypeScript transpileModule로 소스를 메모리에서 CommonJS로 변환하고 각 검사마다 별도 VM context/모듈 캐시를 생성합니다. 실제 [apiClient.ts](../src/api/apiClient.ts), [tokenStore.ts](../src/api/tokenStore.ts), [billingStore.ts](../src/api/billingStore.ts), [userStore.ts](../src/api/userStore.ts), [auth.ts](../src/api/auth.ts), [errorMessage.ts](../src/utils/errorMessage.ts)를 실행합니다.

실제 Axios의 요청/응답 interceptor, config merge, 재요청, Promise 흐름을 사용합니다. 전송 adapter만 합성 상태/JSON/예외로 교체합니다. custom adapter는 실패 상태에서 실제 AxiosError를 만들어 기본 HTTP adapter의 reject 경계를 재현합니다. window.location·Event·local/sessionStorage는 메모리 대체물, i18n은 language 값, clientLog는 기록 함수로 대체합니다. React/DOM/라우터·실제 브라우저 저장소·cookie·실제 HTTP는 실행하지 않습니다. TypeScript 타입 검사는 transpile과 별개의 기존 tsc 결과입니다.

추가로 Node http.request/https.request/net.Socket.connect를 실패하도록 막고 테스트 종료 hook에서 실제 네트워크 시도0을 단언합니다. 합성 토큰·`.invalid` 이메일만 사용했습니다. 메일발송·실크레딧차감·유료호출0입니다.

동시성은 임의 긴 sleep 대신 deferred Promise로 refresh/잔액 응답을 보류하고 원하는 시점에 해제하여 재현합니다. 16개 요청이 동시에401을 받은 뒤 refresh가1개인지, logout 정리 뒤 응답이 도착하면 상태가 되살아나는지 분리합니다. 실제CPU/서버 부하 성능 측정은 아닙니다.

## 검사별 입력·기대값

아래 이름은 Node test의 정확한 이름입니다.

| 테스트 이름 | 수행·단언 | 결과 |
|---|---|---|
| token memory changes emit events without writing browser storage | 동일토큰 두번set·두번clear→변경event2개, 저장소쓰기0, 최종null | PASS |
| actual axios interceptor attaches bearer credentials and current language | token존재·ko-KR→en-US 전환, Bearer/withCredentials=true 및Accept-Language ko/en 확인 | PASS |
| sixteen simultaneous 401 responses share one refresh and retry each request once | 16개GET 최초401, refresh보류후해제. refresh1·업무요청32·전부성공·새token | PASS |
| refresh failure clears token and preserves pathname query before login redirect | GET401→refresh401. tokennull·복귀경로/portfolio?tab=risk 저장·/login 이동, 전송2 | PASS |
| second 401 stops retry loop and clears refreshed token | 최초401→refresh정상→재요청401. 총3요청·추가refresh없음·token삭제 | PASS |
| skipAuthRedirect 401 neither refreshes redirects nor clears token | skip플래그 true·401, 요청1·이동없음·기존token유지 | PASS |
| auth login 401 does not recurse into refresh | 실제auth.login→401, login1회·refresh0·/login 이동 | PASS |
| 403 redirects even with auth skip while 500 and transport errors do not | 세상태별skiptrue. 403만/forbidden, 500/합성전송예외는경로유지 | PASS(현재동작) |
| refresh response missing access token is rejected rather than replayed | refresh200의data에token없음, 명시예외·업무재전송없음·/login | PASS |
| refresh promise resets after a failure so a later request can refresh | 첫refresh실패, 나중새요청은두번째refresh성공·새token | PASS |
| bootstrap uses direct credentialed refresh and quietly handles missing or failed responses | token있는200/빈200/401 각각true/false/false. credentials사용, redirect없음 | PASS |
| two concurrent bootstrap calls currently issue two refresh requests observed | bootstrap2개직접동시호출→전송2. interceptor single-flight와별개 | PASS(관측) |
| logout clearing token must invalidate an in-flight authentication refresh | refresh보류→clearAccessToken→응답해제→원요청완료. 기대null이나synthetic-late가다시저장됨 | **FAIL** |
| eight balance refresh calls share one request and floor the authoritative response | 잔액갱신8개동시호출→GET1,17.9→17·저장17·listener1회 | PASS |
| balance storage normalization notifications and unsubscribe are deterministic | 음수storage→0,4.9→4,Infinity→0,unsubscribe이후알림없음,storage예외시메모리7유지 | PASS |
| logout clearing balance must invalidate in-flight previous-user balance response | 잔액응답보류→clearTokenBalance→0확인→늦은99응답. 기대0이나99복원 | **FAIL** |
| balance failure preserves cached value and allows a subsequent fresh request | cached5·GET500후5보존, 다음GET12→12·요청2 | PASS |
| temporary charge skips invalid amounts and stores server ledger balance | 0/-1/NaN/.2→요청0,3.9→amount3 POST·합성ledger23을반영 | PASS |
| user storage rejects malformed JSON and survives denied storage | 깨진JSON→null, storage차단시set/get메모리유지·clear정상 | PASS |
| error code mapping logs public message and keeps response rejection | 402/INSUFFICIENT_CREDIT→한국어정형안내,내부message아닌public메시지기록·Promise거부·이동없음 | PASS |

## 발견사항·한계

**FRONT-AUTH-LATE-001:** `requestRefresh`의 then은 새 access token을 무조건 저장합니다. 그 사이 `clearAccessToken`이 호출됐는지 확인하는 취소/세대 구분이 없어 늦은응답이 토큰을 복원하고 원요청도 새토큰으로 재전송합니다. 회귀1개FAIL입니다.

**FRONT-BALANCE-LATE-001:** `clearTokenBalance`가 메모리/storage를 정리해도 진행중 refreshPromise를 무효화하지 않습니다. 이전응답의 creditBalance가다시저장됩니다. 회귀1개FAIL입니다.

연결근거: 실제 [Sidebar.tsx](../src/layout/Sidebar.tsx)의 handleLogout finally는 clearAccessToken·clearUser·clearTokenBalance를 호출합니다. 이검사는 같은정리함수와실제Promise경계를호출한것이며Sidebar를mount하거나로그아웃서버를실행한것은아닙니다. 실제서버의refresh/logout응답순서·토큰폐기정책·브라우저전체흐름은추가검증해야합니다. 서버인증우회가확정됐다고주장하지않습니다.

bootstrap2회→refresh2회는현상만고정했습니다. ReactStrictMode/실사용UI에서반드시중복발생한다는판정은아닙니다. 403에skip플래그가적용되지않는것도현재코드의관측이며정책위반으로확정하지않습니다.

남은검증: 실제Chrome의refresh cookie/CORS/SameSite·스토리지차단·리다이렉트, 로그인/OAuth/로그아웃UI, 여러탭/사용자전환, 실제잔액API/원장, 토큰만료시각과동시refresh의서버세션정합성. 기존[브라우저fixture검사](README.md)와본VM검사를합쳐실제계정E2E완료로표기하지않습니다.
