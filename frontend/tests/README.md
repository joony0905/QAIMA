# 프론트 테스트 실행·결과

최신 [Feature1 실제 서버·브라우저/저장/PDF](browser-feature1-pipeline.md)은 **17개5 PASS/12 FAIL**(`run-k_l2x_cx`)입니다. 계산 장애의 부분 결과/안내·동일 입력 복구와 보조 재무 조회 오류 중 분석 보존은 통과했습니다. 저장 정량·설명/핵심/저장 경고 누락과 기존 원천/snapshot/stale/날짜 실패를 실제 화면까지 재현했습니다. 실제 FastAPI17회·응답20개·snapshot/상세15개·USE20/REFUND3·동일 입력 복구3개를 대조했고 소유 서비스를 정리·종료했습니다. 별도 검토의 볼린저 위치91.2%→50.0% 및 PDF 스타일 실패는 JUnit에 중복 합산하지 않습니다.

최신 [Feature3 overlay/Redis/취소 검증](../../tests/isolated-feature3-resilience.md)은23개19 PASS/4 FAIL입니다. 실제 원천과 선택 overlay를 연결한 브라우저/저장/PDF 및 브라우저 취소는 후속 범위입니다.

후속 [Feature3 실제 원천 SQL 검증](../../tests/isolated-feature3-source-sql.md)은17개12 PASS/5 FAIL이며, 이번 원천 결과의 추가 브라우저 검증은 남아 있습니다.

전체 누적은 **30묶음632개481 PASS/151 FAIL**이며 전체 QA는 미완료입니다. 앞선 [Feature1 서버23개](../../tests/isolated-feature1-pipeline.md)와 최종 브라우저17개를 구분합니다. [후속 Feature1 Redis/HTTP 취소 검증](../../tests/isolated-feature1-resilience.md)은17개16 PASS/1 FAIL로 확정했습니다. 브라우저 취소/전체 UI 조합 및 나머지 사용자 흐름은 계속 남습니다.

2026-09-28 직전: [Feature2 실제 SQL·Peer·로컬 모델의 브라우저/저장/PDF](browser-feature2-pipeline.md) **12개4 PASS/8 FAIL**(`run-kd_uymvv`). 실제 경고의 화면 누락·모델 없음에서 보고서 기사 소실·정상 RAW Peer 차트 소실과 기존 저장 정량/거시/NULL 수급 실패를 기록했습니다. 실제 분석/SQL/소유권14개·모델31결과/공개33점수·PDF2개6페이지를 대조했습니다. 초기 구성 보정/중단 실행은 제외하고 최종 격리 서비스를 정리·종료했습니다. 실제 외부 제공자와 전체 UI 조합은 남습니다.

2026-09-28 후속: [Feature3 브라우저·실제 Spring/FastAPI·SQL·저장 PDF](browser-server-pipeline.md) **8개7 PASS/1 FAIL**. 실제 로그인/refresh·분석·차감/환불·저장 리포트 조회/PDF·소유권·로그아웃을 연결했습니다. 리포트 저장 실패의 화면 안내 누락을 재현했습니다. PDF1개/A4 1페이지의 파싱·본문/한글 확인은 통과했습니다. 가격/금리 service는 합성 입력이며 실제 금융 제공자는 별도입니다.

2026-09-28 후속: [Feature3 포트폴리오 차트·CAPM 표시·입력 경합](browser-feature3-charts.md) **26개23 PASS/3 FAIL**. 실제 Chrome에서 SCL/SML·벤치마크 전환·프론티어/CAL·분산 기반 효용·4수준/영어/모바일·오버레이 부호를 대조했습니다. 분석 대상/기간의 메타데이터 변경2경로와 비용 미리보기/분석 대상 불일치1경로가 실패입니다. 모든 API는 fixture이며 실제 계산/원장 결합은 별도입니다.

2026-09-28 후속: [Feature1 재무·차트·검색 경합·PDF](browser-feature1-charts.md) **25개18 PASS/7 FAIL**. 재무 출처/fallback·기간별 모달·독립 차트 좌표·보조지표·영어/모바일을 실제 Chrome으로 검사했습니다. 이전 검색/분석의 최신 종목 덮어쓰기, 실패 후 이전 재무 유지, KST 표시 시각, 두 PDF 스타일 단언이 실패했습니다. 별도 PDF2파일5페이지의 상세 마지막 페이지는 본문이 있으나 낮은 픽셀 비율로 검사 FAIL입니다. 모든 API는 fixture이며 실제 서버/DB 결합은 별도입니다.

2026-09-28 후속: [Feature2 차트·nullable·분석 직후 PDF](browser-feature2-charts.md) **20개13 PASS/7 FAIL**. 실제 Chrome에서 거시/수급/공매도 값·hover·모드·모바일과 PDF를 확인했습니다. 공매도 null 화면 소실3경로, 다른 날짜 배열의 좌표 불일치, 기존 비율 문제의 표시 영향, 두 PDF 스타일 소실이 실패입니다. 별도 PDF2파일5페이지 검사에서는 상세 마지막 페이지가 거의 비어 있어 FAIL입니다. 모든 API는 fixture이며 앞선18개와 별도 범위입니다.

2026-09-28 추가: [Feature2 Chrome 검증](browser-feature2-flow.md) **18개15 PASS/3 FAIL**. 공개조회·관심종목·기간/수준·영어·모바일을 검사했고, 보조조회 실패의 성공결과 소실·이전 검색 목록 덮어쓰기·이전 분석의 새 종목 화면 반영을 재현했습니다. 모든 API는 fixture입니다. 아래 2026-09-25 결과는 과거 기록으로 유지합니다.

검증일: 2026-09-25. Windows Node 25.1.0, npm 11.6.2. 기존 package.json에는 테스트 script가 없으며 이를 수정하지 않았습니다.

## 명령

Windows PowerShell, frontend 디렉터리에서:

```powershell
node --test tests/contracts.test.cjs
node --test tests/contracts.test.cjs tests/auth-client.test.cjs
node node_modules/typescript/bin/tsc -p tsconfig.app.json --noEmit --incremental false
node node_modules/typescript/bin/tsc -p tsconfig.node.json --noEmit --incremental false
powershell -NoProfile -ExecutionPolicy Bypass -File tests/verify-windows.ps1
```

순수 TS 모듈을 TypeScript transpileModule로 메모리에서 CommonJS로 변환하여 Node의 실제 테스트 runner로 실행합니다. 타입 정확성은 별도 tsc 두 명령으로 검사합니다. transpileModule 자체는 타입 검사기가 아닙니다.

## 케이스

| 테스트 | 입력·검증 과정 | 독립 기대값 |
|---|---|---|
| candle mapper | 초·밀리초 timestamp를 뒤섞은 2개 candle 변환 | 시간 오름차순·초단위, 거래량10/20, 원본 배열 불변 |
| risk/cash | NaN, -1, 2, score0.8, 라벨 경계0.2/1 | 점수0/0/1, 현금 기본0.2·자동상한0.2, 안정형/공격투자형 |
| localStorage | Map 기반 브라우저 storage 대체, default 동기화 후 현금 수동저장 | 점수0.6·현금0.4→0.25, manual false→true |
| 투자 수준 | 4단계와 unknown | 각각 roundtrip·unknown은 초급자 |
| API 경로·인코딩 | Feature1/3 경로, 사전 `A/B & C`, page0 | 실제 Public 상대 경로·URL 인코딩·page0 보존 |

위순수계약5개PASS. 추가[인증·잔액클라이언트20개](auth-client.md)는18 PASS·2 FAIL이며합쳐25개23 PASS·2 FAIL입니다. 실제Axios interceptor·동시refresh·스토어를VM에서검사했고늦은응답의로그아웃상태복원2개를재현했습니다. 화면mount/router/실제브라우저storage는이Node검사에포함되지않습니다.

## 빌드 결과

원본 디렉터리에서 TypeScript app/node 프로젝트 검사 모두 PASS. Vite 직접 빌드는 `@rollup/rollup-win32-x64-msvc` 누락으로 FAIL했습니다.

`verify-windows.ps1`은 소스·public·빌드 설정·package-lock을 `tests/.runtime/build-project`로 복사하고 Windows에서 npm ci와 npm run build를 실행합니다. 환경 파일·기존 node_modules는 복사하지 않습니다. 격리 복사본 빌드는 PASS했고 Vite 7.1.12, 2165개 모듈 변환을 확인했습니다. 큰 bundle chunk와 오래된 브라우저 메타데이터 경고가 남았습니다. 원본 node_modules를 수리한 결과는 아닙니다.

산출물은 `.runtime/build-project/dist`, 설치 캐시는 `.runtime/npm-cache`에 있습니다. `.runtime`은 Git 추적에서 제외합니다. 재실행 시 복사본은 재사용되므로 소스 파일 삭제까지 반영해야 하는 검증은 새 격리 디렉터리에서 수행해야 합니다.

## 남은 검증

Feature1의실제Chrome조회·관심종목·분석요청/결과/오류는 [10개검사기록](browser-feature1-flow.md)에방법과mock경계를기록했습니다. 실행명령은 `node tests/browser-feature1-flow.cjs`입니다. 실제DB/유료호출/환불검증은아닙니다.

Feature3의 [자동15개검사](browser-portfolio-flow.md)는렌더시점CSS단언보강후14 PASS·고급PDF1 FAIL입니다. 입력·저장·비용·결과화면은통과했지만PDF렌더러가복제문서의본체CSS없는상태를읽는경로가확인됐습니다. 최신PDF2파일5페이지의구조검사와시각품질은구분합니다. 모든API는합성이며실제원장/DB/유료LLM통과가아닙니다. `QAIMA_PDF_DIAGNOSTIC`과`QAIMA_PDF_EMPTY_EXTERNAL_CSS`를해제한뒤 `node tests/browser-portfolio-flow.cjs`로전체재현합니다.

추가로실제Chrome의[인증화면11개](browser-auth-flow.md)가통과했습니다. 폼제출/오류·복귀경로·200/500로그아웃·OAuth추가정보/오류복귀를합성API로검증했으며실제계정/OAuth연동은아닙니다. 늦은응답의상태복원회귀2개는여전히FAIL입니다.

모든 Swagger 관련 화면→실제 API→응답 rendering, 로그인 refresh 경쟁, OAuth 추가정보, 소유권 UI, 차트 null·비정상 데이터, 모드/언어별 분석 표시가 남아 있습니다. 저장 리포트의 fixture 기반 재조회·PDF는 아래 범위이며분석직후Feature3 PDF는위추가검사에서스타일문제를관측했습니다. Feature1/2 분석 직후 PDF는 위 후속 차트 검사에서 확인했고 스타일 실패를 기록했습니다. 실제 DB 스냅샷 연결은 남았습니다.

## Windows Chrome 브라우저 smoke

격리 빌드 후 Windows Chrome이 설치된 환경에서 실행합니다. 브라우저 도구 의존성도 test 내부에만 설치합니다.

```powershell
npm install --prefix tests/.runtime/browser-tools --no-save --package-lock=false playwright-core
node tests/browser-smoke.cjs
```

`browser-smoke.cjs`는 격리 빌드의 Vite preview를 127.0.0.1:4173에 시작하고 headless Chrome으로 방문합니다. 종료 시 자체 브라우저·preview 프로세스를 종료합니다. 다른 origin 요청은 차단하고 `/api/` 요청은 모두 fixture로 응답하므로 실제 로그인·외부 발송·크레딧 차감은 없습니다. 실제 백엔드 연동 E2E와 구분합니다.

2026-09-25 **6개 검사 PASS**:

1. refresh 401 이후 로그인 페이지 email/password 필드 표시.
2. 가입 화면 비밀번호와 확인 필드2개 표시.
3. 비로그인 `/signup/social-complete` → `/login`, sessionStorage 복귀 경로 보존.
4. 비로그인 `/invest-level-survey` → `/login`, sessionStorage 복귀 경로 보존.
5. 사전 페이지 초기 CAPM fixture 표시 → 검색어 입력 → 검색 버튼 클릭 → `q=CAPM` HTTP 응답 대기 → 설명 표시 확인.
6. 이 이동 과정에서 uncaught pageerror 없음.

초기 테스트는 Enter로 검색 요청이 발생한다고 가정하여 FAIL했습니다. 실제 입력에는 Enter handler가 없고 버튼의 onClick이 검색을 수행함을 소스에서 확인했습니다. 버튼 클릭 및 해당 응답 대기로 테스트를 수정한 후 PASS했습니다. Enter 지원 여부를 요구사항 위반으로 단정하지 않으며 제품 코드는 수정하지 않았습니다.

증거: `.runtime/browser-artifacts/summary.json`에 검사 목록·모의 API 요청 method/path/query, `dictionary-fixture.png`에 사전 화면을 저장합니다. pageerror 부재는 HTTP 오류·레이아웃·이미지 로딩·접근성 전체 통과를 뜻하지 않습니다.

## 저장 리포트 PDF: 실제 Chrome 생성 + 별도 PDF 파서

2026-09-25 `browser-report-pdf.cjs` **9개 검사 PASS**, `inspect-report-pdf.py` **4개 PDF·총6페이지 PASS**. 실제 `SettingPage`→`api/reports.ts`→`SavedReportDocument`/`ReportHeader`→`waitForPdfCaptureReady`→`downloadElementAsPdf`→html2canvas/jsPDF를 실행합니다. API·인증 응답은 합성 fixture이며 서버·DB·OAuth 연동 성공을 의미하지 않습니다.

Windows frontend 디렉터리:

```powershell
node tests/browser-report-pdf.cjs
```

WSL 프로젝트 루트에서 별도 렌더링 검사:

```bash
.venv_wsl/bin/python -m pip install --no-cache-dir --target frontend/tests/.runtime/pdf-tools pymupdf==1.28.2
PYTHONPATH=frontend/tests/.runtime/pdf-tools PYTHONDONTWRITEBYTECODE=1 .venv_wsl/bin/python -B frontend/tests/inspect-report-pdf.py
```

격리 Vite preview는 127.0.0.1:4174/strictPort, Windows Chrome은 headless·ko-KR·1440×1000입니다. 다른 origin은 차단하고 `/api/`는 명시된 fixture만 응답합니다. 미등록 API는 500 및 최종 assertion 실패로 처리하며 service worker도 차단합니다. 종료 시 자신이 시작한 브라우저·preview만 종료합니다. 실제 메일·유료 호출·크레딧 차감은0회입니다.

| 검사 | 입력·행동 | 확인한 기대값 |
|---|---|---|
| 보존기간 목록 | 현재 시각의 Feature1/2/3와8일 전 리포트1개 | 최근3개만 표시·PDF 버튼3개, 만료기업 미표시 |
| 목록 PDF3개 | 각 PDF 버튼 클릭→해당 reportId 상세 GET→download 이벤트→저장 | `qaima_report_ID.pdf`, download 실패없음, `%PDF-`/`%%EOF`, 페이지 개수, capture class 정리 |
| 긴 문서 | Feature2에22개 설명 구간 | 실제3페이지 생성, 나머지 Feature1/3은 각각1페이지 |
| 상세 재다운로드 | 보기→한글 설명 확인→PDF 재다운로드 | 상세 스냅샷 재사용, 추가 상세 GET 없음,1페이지 PDF |
| 상세404 | 목록은 정상, 상세는 RESOURCE_NOT_FOUND/404 | 정확한 한국어 안내, 다운로드0·버튼 다시 사용 가능·capture class 없음 |
| 다운로드 직전 만료 | 목록은 정상, 상세 generatedAt은8일 전 | 보존기간 안내, 목록에서 제거·빈 목록 표시, 다운로드0 |
| 부수효과 | 전체 요청·pageerror·dialog 수집 | pageerror0, 예상 오류 알림2개 외 없음, GET와 합성 refresh 외 업무 변경 요청0 |

첫 실행에서는 앱의 사전 초기 조회가 fixture에 누락돼 미등록 API assertion이 실패했습니다. 실제 초기 로딩 요청을 확인해 빈 사전 fixture를 명시한 뒤 통과했습니다. 정상 PDF만 만들어졌다고 실패 실행을 PASS로 기록하지 않았습니다.

PDF 검사는 브라우저가 저장한 파일을 PyMuPDF1.28.2로 열어 수리 없이 파싱되는지, 페이지 수가 브라우저 결과와 같은지, 각 페이지가 A4(약595.28×841.89pt)인지 검사합니다. grayscale 렌더에서160 미만의 어두운 픽셀 비율이0.1%를 넘는지 검사해 빈 페이지/옅은 워터마크만 있는 파일을 걸러냅니다. 모든 페이지를 PNG로 렌더링하여 한글 제목·본문·숫자·워터마크를 육안 확인했고, 긴 문서의 구간1~22가 마지막 페이지까지 표시됨을 확인했습니다. 이는 모든 한글 글리프·임의 길이·문단 경계 분할을 자동 증명하는 검사는 아닙니다.

증거는 `.runtime/report-pdf-artifacts/`의 `summary.json`, `pdf-inspection.json`, PDF4개, 페이지 PNG6개, 상세 화면 `report-detail.png`입니다. PDF는 래스터 기반이므로 검색 가능한 텍스트·접근성 지원을 검증한 것이 아닙니다. 이번 fixture의1페이지 파일은13,940,448bytes,3페이지는43,425,748bytes로 크며 최적화·다운로드 성능 기준은 별도입니다. 이 저장리포트 검사는 분석 직후 화면 버튼을 포함하지 않습니다. Feature3 버튼/차트 PDF의 추가검사는 위 상세문서에서 스타일 손실을 관측했으며, Feature1/2 버튼은 위 후속 검사로 확장했고 Feature2 canvas 예외 복구도 확인했습니다. 다른 언어/테마·이미지 실패·실제 저장소와 소유권 결합은 남았습니다.
