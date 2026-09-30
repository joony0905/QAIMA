# React 프론트엔드

작성·검증일: 2026-09-25. Windows에서 실행합니다. React 19, TypeScript 5.9, Vite 7 기반이며 실제 버전은 package-lock.json을 기준으로 합니다.

## 실행

Windows PowerShell의 frontend 디렉터리에서:

```powershell
npm ci
npm run dev
```

Vite의 /api 프록시는 Windows의 로컬 Spring 서버로 전달합니다. 브라우저 API 클라이언트의 baseURL은 /api/v1입니다. FastAPI나 외부 금융 API를 직접 호출하는 기본 구조가 아닙니다.

## 화면과 데이터 흐름

| 화면 | 구현 파일 | API 모듈 |
|---|---|---|
| 종목 분석 | src/pages/StocksMockPage.tsx | api/stock.ts, charts.ts, financial.ts, analysis.ts |
| 외부 요인 | src/pages/Feature2MockPage.tsx | api/feature2.ts, news.ts |
| 포트폴리오 | src/pages/PortfolioMockPage.tsx | api/portfolio.ts |
| 용어 사전 | src/pages/DictionaryMockPage.tsx | api/dictionary.ts |
| 로그인·가입·OAuth | LoginPage, SignupPage, OAuth2SuccessPage, SocialProfileCompletePage | api/auth.ts |
| 내정보·설문 | EditProfilePage, SurveyPage, InvestLevelSurveyPage | api/user.ts |
| 리포트·PDF | 분석 화면·ReportHeader·PDF context | api/reports.ts, utils/reportPdf.ts |

파일명에 Mock이 포함되어 있어도 실제 API 호출 코드를 포함합니다. 파일명만으로 더미 구현이라고 판단하지 않습니다.

apiClient.ts는 메모리 access token을 첨부하고, 401 발생 시 refresh cookie로 한 번 갱신하여 재시도합니다. 동시 refresh 요청은 하나의 Promise로 공유합니다. 이 동작의 전체 브라우저 회귀 검증은 아직 남아 있습니다.

Feature 3 화면의 위험 점수·현금 상한은 localStorage의 전용 키에 저장합니다. 이는 공식 사용자 프로필을 갱신하는 API와 별도입니다. 투자 설명 수준은 초급자·중급자·고급자·전문가 4단계입니다.

## 검증

Feature1의종목조회·캔들시세·관심종목추가/삭제·분석기간/요청/결과·오류복구흐름은 [Chrome10개검사](tests/browser-feature1-flow.md)에기록했습니다. 모든API는합성응답이며실제금융/DB/크레딧연동과구분합니다.

Feature3 입력·저장·비용·결과/PDF의 [실제Chrome15개검사](tests/browser-portfolio-flow.md)는14 PASS·고급PDF1 FAIL입니다. PDF렌더러가복제문서의본체CSS없는상태를읽는스타일손실을검출했습니다. 서버응답은모두합성이며실제저장/유료호출/환불보장은아닙니다.

[test 패키지 문서](tests/README.md)에 실행 명령, 입력·기대값, 실제 결과와 한계를 기록합니다.

2026-09-25 Windows Node 검사는25개23 PASS·2 FAIL입니다. 기존계약5개에[인증·잔액동시성20개](tests/auth-client.md)를추가했고로그아웃정리후늦은응답이토큰/잔액을다시저장하는경로를재현했습니다. 실제HTTP전송·React화면은대체한경계입니다. 원본 프로젝트의 TypeScript 검사는 통과했습니다. 기존 node_modules에서는 Windows Rollup 모듈 누락으로 Vite 빌드가 실패했습니다. tests/.runtime/build-project의 격리 복사본에서 lockfile대로 npm ci 후 npm run build는 통과했습니다. 원본 의존성 디렉터리는 수정하지 않았습니다.

빌드 통과는 화면 렌더링·실제 API 연결·차트 상호작용·PDF 생성의 통과를 의미하지 않습니다. 큰 bundle chunk 경고와 브라우저 데이터 갱신 경고가 남아 있습니다.

별도 Windows Chrome 검증에서 로그인·가입·보호경로·사전 검색6개와 저장 리포트 PDF 흐름9개 검사가 통과했습니다. 합성 API 응답으로 생성한 PDF4개/6페이지를 별도 파서로 렌더링해 A4·빈 페이지 여부·한글 표시를 확인했습니다. 실제 백엔드 연동, 분석 직후 화면의 차트/PDF까지 검증한 결과는 아닙니다.

추가[인증화면11개](tests/browser-auth-flow.md)도실제Chrome에서통과했습니다. 폼요청·로그아웃·OAuthcallback화면은실행하되API는합성응답이며실제로그인/외부OAuth제공자연동검증은남았습니다.
