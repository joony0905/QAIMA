# Feature2 실제 SQL·Peer·로컬 모델의 브라우저/저장/PDF QA

## 현재 상태

앞선 [실제 Peer/모델 서버 검증](../../tests/isolated-feature2-pipeline.md)을 Chromium/React까지 확장했습니다. 최종 `run-kd_uymvv`은 **12개 4 PASS / 8 FAIL**, JUnit 225.944초, errors/skipped 0입니다. Gradle 7분 19초/exit1은 제품 실패 단언의 결과이며 harness 오류는 없습니다. 최종 사례만 [누적 기록](../../tests/QA_PROGRESS_2026-09-28.md)에 합산했습니다. 전체 누적은 **25묶음 535개: 416 PASS / 119 FAIL**이며 전체 QA는 미완료입니다.

## 방법과 경계

실제 로그인/JWT→검색/카드/뉴스→분석 버튼→실제 시장 JPA/Redis/Peer 양방향 HTTP/로컬 CPU 모델→설명 무키 fallback→public 응답/SQL report→화면/저장 상세/PDF 경로를 확인합니다. Playwright는 public API를 가로채 응답을 만들거나 바꾸지 않습니다. 허용한API만 소유loopback Spring으로 전달하는 투명 프록시를 사용합니다.

- 제품 및 기존DB는 수정하지 않습니다. `tests/`, `frontend/tests/` 안에서 구성하고,외부 새 MySQL/Redis 및 소유 FastAPI/Spring을 사용합니다.
- 카드와 뉴스 service,시장/뉴스/리포트 JPA,Peer SQL callback/Python 계산,실제 뉴스 모델은 제품 코드입니다. 종목 검색/매핑·랭킹·외부 금융/뉴스 제공자·뉴스 observation 비동기 저장은 합성 경계입니다. 종목 헤더 값은 실제 stock resolver 결과에서 읽습니다.
- 캔들은 실제 ChartService/CandleLoadService/JPA/mapper/controller를 통과합니다. 외부 StockClient는 빈 DTO/DB source fixture이며 SQL에 넣은 합성 행 외 시장 자료는 받지 않습니다. 이를 실제 외부 제공자 응답 검증으로 해석하지 않습니다.
- 모델은 브라우저 시작 전에 실제 뉴스API로1기사 추론하여 로딩합니다. 앞 단계의 cold 모델93초/30초 timeout 관측은 유지하고 이번 화면 검사는 warm 상태에서 진행합니다. 모델 누락은 같은 프로세스의 메모리 모델을 보관하고 소유 미존재 경로를 주입한 뒤 복구합니다.
- 과거 검증된 frontend 소스/복사본187개와bundle hash를 매번 대조합니다. Chrome 전용profile/fontcache는 `/tmp/qaima-browser-*`에 만들고 종료 시 삭제합니다. Windows 설치 한국어 글꼴2개를 읽기 전용으로 사용합니다.
- UI 날짜는 실제 `buildAnalysisDateRange`가 생성합니다. 네트워크 입력을 고정하지 않고 첫/두번째 재분석 요청의 from/to와Peer 캐시 키를 기록하여 같은키 재시도와 구분합니다.
- 대응 경고·정상 수치/기사·Peer 상관/시차·분율→백분율·모델 점수/URL을 DOM과 비교합니다. public/SQL/소유자 상세는 구조/배열/문자열이 같아야 하며 실수 저장 오차는 `max(1,abs(expected))*1e-12`까지 허용합니다. 원장/잔액/다른 계정 상세404도 실제 확인합니다.

## 실행

```bash
python3 -B tests/run_isolated_backend.py --suite feature2-pipeline-browser
```

[Java host](../../tests/java/com/qaima/qa/IsolatedFeature2PipelineBrowserTest.java),[Chrome/투명 프록시](browser-feature2-pipeline.cjs),[공통 실제 Peer/모델 host](../../tests/java/com/qaima/qa/IsolatedFeature2PipelineTest.java).

브라우저 watchdog220초/Java child240초/JUnit280초는 테스트 관측 한도입니다. 제품 내부 news30초·Peer callback8초는 변경하지 않습니다. 전체 QA 완료나 운영 성능 보증을 뜻하지 않습니다.

## 사례별 검증

아래 결과는 최종 `run-kd_uymvv`입니다. 각 PASS는 해당 사례의 단언 범위이며, 공통으로 응답에 존재하는 설명 무키 경고의 화면 누락까지 정상이라는 뜻은 아닙니다.

| mode | 결과 | 주입·독립 기대값과 실제 관측 |
|---|---|---|
| normal_live | PASS | 실제 SQL 값·Peer 상관/시차·모달의 수익률 단위·모델 기사별 점수를 화면과 대조 |
| saved_quantitative_pdf | FAIL | SQL/상세 응답에는 정량 결과가 있으나 저장 화면·PDF에 없음. live PDF에는 정량 값이 남음 |
| missing_key_warning | FAIL | 실제 무키 설명의 `LLM_API_KEY_MISSING`이 public/SQL에 있지만 사용자 안내 없음 |
| peer_source_warning | FAIL | 실제 역방향 원천 HTTP 503의 `SPRING_PACK_FETCH_FAILED`가 public/SQL에 있으나 실패 안내 없이 0개 선정 표시 |
| model_missing | FAIL | 실제 모델 누락에서 기사 3개·null 점수·실패 안내는 유지. 뉴스 탭에는 기사가 있지만 보고서 기사 목록은 사라짐 |
| raw_index_fallback | FAIL | 산업지수 SQL 1146에서 실제 RAW Peer 계산·상관 표시·대체 안내는 유지. 정상 anchor/Peer 시계열의 차트·설명 모달은 없어짐 |
| peer_ui_recovery | PASS | 원천 503 제거 후 같은 화면의 재검색/재분석으로 Peer 2개 복구, 오류 경고 제거. 요청 시각과 캐시 키가 바뀐 경우 |
| model_ui_recovery | PASS | 모델 메모리/경로 복구 후 기사 점수 3개·보고서 목록 복구, 이전 모델 경고 제거 |
| combined_peer_model | FAIL | 원천 503+모델 누락에서 금리·수급·공매도 보존. Peer 실패 안내 누락과 보고서 기사 소실 함께 재현 |
| sentiment_sql_write | PASS | 실제 sentiment INSERT trigger 실패에도 계산한 점수 3개·화면 점수·저장 실패 관련 안내 유지 |
| macro_sql_failure | FAIL | 환율 SQL 1146 하나로 정상 미국금리 4.25% 등 거시 값까지 null/빈 목록. 화면에도 4.25% 없음 |
| null_flow | FAIL | 실제 SQL 수급 3행의 네 금액/수량 필드를 NULL로 설정했는데 합계 0·MIXED_OR_FLAT 및 화면 보합, 누락 안내 없음 |

### 수치 대조

합성 기준 종목은 `009991`, Peer는 `QAF2FOLLOW`와 `QAF2LEAD`입니다. SQL DECIMAL(18,6)에 저장한 종가를 먼저 양자화한 별도 fixture로 로그수익률 Pearson 상관·산업수익률 차감 후 상관·시차를 계산했습니다. 반환된 각 Peer의 corr/adjustedCorr는 독립 기대값과 `1e-10` 이내, bestLag는 정확히 일치해야 합니다. 화면은 해당 상관의 소수 둘째 자리·각 종목 코드·시차 일수와 대조합니다.

| Peer | RAW 상관 | 산업조정 상관 | 시차 |
|---|---:|---:|---:|
| QAF2FOLLOW | 0.759176121333587 | 0.7347358240726303 | +2일 |
| QAF2LEAD | 0.7681548410003053 | 0.7434424516898058 | −2일 |

시계열은 2026-05-08~08-06의 연속 합성 날짜 91개/수익률 90개입니다. 실제 거래일 분포를 재현한 자료는 아닙니다. 모달의 기준 종목 마지막 누적수익률은 분율 `0.03316125`에서 **3.32%**로 표시됩니다. UI는 실제 날짜로 6개월 범위/window120/peerCount30을 요청하며 이번 후보 4개 중 품질필터를 통과한 2개를 반환합니다. 후보 선정 알고리즘 전체의 독립 검증과 구분합니다.

뉴스 3개의 본문은 실제 로컬 DeBERTa 모델에 전달합니다. FastAPI 결과의 확률 범위·합계, argmax 라벨, `(positive-negative) × (1-neutral)` 점수식을 독립 대조합니다. 공개 기사별 URL/점수, 요약의 개수/평균과 DOM 점수를 확인합니다. 합성 문장을 시장 정답 라벨로 평가한 정확도 측정은 아닙니다.

### 복구 판정의 경계

브라우저가 소유 파일로 복구를 요청하면 Java가 원천 HTTP/실제 모델/숨긴 SQL 테이블을 복원하고 완료 파일을 씁니다. 이후 사용자의 명시 재검색·재분석 동작을 실행합니다. 중간에 Redis 키를 지우지 않습니다.

Peer 복구에서는 재분석의 from/to 초가 바뀌어 캐시 키가 새로 만들어집니다. 첫 오류 결과와 새 성공 결과가 서로 다른 키로 함께 남고 역방향 source 요청도 2회입니다. 따라서 이 PASS는 앞선 **F2-PEER-ERROR-CACHE-001의 완전히 같은 요청 재시도 실패**를 해결한 증거가 아닙니다. 자연 12시간 만료도 기다리지 않았습니다. 모델 복구 역시 저장해 둔 모델 객체를 복원한 warm 상태이며, 디스크 재로딩 완료로 해석하지 않습니다.

## 확인된 실패 경로

- **FRONT-F2-SOURCE-WARNING-001**: 실제 `LLM_API_KEY_MISSING` 및 `SPRING_PACK_FETCH_FAILED`/`HTTPStatusError`가 public meta와 SQL warnings에 보존되지만 [warningNotes](../src/utils/warningNotes.ts)의 등록되지 않은 코드 제거로 사용자 안내에 도달하지 않습니다. Peer 화면의 “선정된 유사 종목이 없습니다.”는 원천 장애와 정상 후보 없음의 차이를 전달하지 못합니다. 기존 저장/핵심 warning 누락과 같은 mapper 경계의 추가 코드입니다.
- **FRONT-F2-NEWS-NULL-SUMMARY-001**: 실제 모델 누락에서 newsList 3개와 기사 URL/본문은 유지됩니다. [AnalysisResultPanel](../src/components/AnalysisResultPanel.tsx)이 newsSentimentSummary가 있을 때만 기사 목록까지 렌더링하므로, null summary이면 보고서에서 뉴스가 사라집니다. 외부요인 패널의 뉴스 탭과 실패 안내는 남습니다. DB 기사 자체 소실로 분류하지 않습니다.
- **FRONT-F2-RAW-PEER-CHART-001**: 산업 SQL 장애 시 실제 RAW Peer는 계산되고 anchor/centroid 각 91점이 응답에 남습니다. 그러나 [Feature2MockPage](../src/pages/Feature2MockPage.tsx)는 industrySeries가 비어 있거나 오류이면 RelativeLineWidget 전체를 렌더링하지 않습니다. 유효한 기준 종목·Peer overlay와 설명 모달 접근도 함께 없어집니다. 보고서의 상관 목록/RAW 대체 안내는 유지됩니다.
- **FRONT-F2-SAVED-METRICS-001 (기존)**: [SettingPage의 SavedReportDocument](../src/pages/SettingPage.tsx)가 설명/summary/warnings만 사용하고 metrics를 렌더링하지 않습니다. 실제 무키 설명에서 정량 결과를 담은 SQL/상세 응답과 달리 저장 화면·PDF는 메타데이터와 일부 경고만 보여줍니다.
- **F2-CARD-MACRO-ISOLATION-001 / F2-FLOW-NULL-ZERO-001 (기존)**: 실제 시장 SQL 단계에서 확인한 거시 독립 값 소실 및 수급 누락의 0 변환을 브라우저까지 연결했습니다. 두 결함의 서버 근거는 [시장 SQL 검증](../../tests/isolated-feature2-source-sql.md)에 있습니다.

실패 8사례는 고유 결함 8건을 뜻하지 않습니다. 제품은 수정하지 않았으며 이전 실패도 해결 처리하지 않습니다.

## PDF 검사와 시각 확인

실제 버튼으로 live/saved PDF를 다운로드한 뒤 별도 PyMuPDF parser로 A4 크기·수리 여부·페이지 수·비어 있지 않은 raster를 검사합니다. 모든 페이지를 PNG로 렌더링하고 한글/수치/스타일/페이지 경계를 직접 확인합니다.

```bash
PYTHONPATH=frontend/tests/.runtime/pdf-tools PYTHONDONTWRITEBYTECODE=1 .venv_wsl/bin/python -B frontend/tests/inspect-report-pdf.py --artifacts frontend/tests/.runtime/browser-feature2-pipeline/run-kd_uymvv/saved_quantitative_pdf
```

최종 실행은 PDF 2개·6페이지(분석 직후 5, 저장 1)의 구조 검사 PASS입니다. live PDF에는 상관 .74/.73, 평균 −.32/기사 −.96, 금리 3.25/4.25%, 공매도 7.50/8.50%가 보입니다. 저장 PDF는 정량 결과가 없습니다. live PDF에는 카드/행 배치가 풀리고 페이지 3→4에서 제목/차트가 갈리는 기존 **FRONT-F2-PDF-STYLE-001**의 시각 증거가 남습니다. 이번에는 PDF renderer의 getComputedStyle 계측을 추가하지 않았으므로 새 CSS 내부 추적이라고 주장하지 않습니다. 파일 구조 PASS를 내용/스타일 전체 PASS로 해석하지 않습니다. 별도 PDF 검사는 12사례에 중복 합산하지 않습니다.

## 증거와 정리

최종 `run-kd_uymvv`에서 다음을 확인했습니다.

- 실제 public 분석 14개(복구 사례 2개가 각각 추가 1회), SQL snapshot/owner 상세 14개, 다른 계정 상세 404×14, USE −1×14/환불 0. 브라우저 직접 저장 상세 GET 3개. 최초 잔액 5에서 보통 4, 재분석은 3이며 자동 유료 분석은 없습니다.
- 실제 FastAPI 42회: Peer 14, 뉴스 모델 14(사전 1기사 warmup 포함), 무키 설명 14. 허용한 역방향 실제 TCP 14회입니다. 모델 결과 31개/공개 점수 33개를 URL·점수식으로 대조했습니다. Peer 복구의 재분석은 기존 뉴스 캐시를 사용하므로 공개 점수 수와 실제 모델 결과 수가 같을 필요는 없습니다.
- 사전 실제 모델 로딩/1기사 추론 91.834초, 이후 성공한 3기사 호출 0.864~0.990초입니다. 전체 운영 지연/품질 보장이 아니며 외부 뉴스 제공자 지연은 합성 경계입니다.
- host 입력 612개/모델 7개, frontend 원본·빌드 복사본 187개 및 bundle 11개의 SHA-256을 확인했습니다. tests 밖 가시 코드/설정/문서 924개는 변경/새 파일/소실 0입니다.
- MySQL/Redis exit0·행/키0, 소유 TCP proxy/control/worker 종료·socket0, Spring 포트 닫힘, FastAPI 의도한 SIGTERM −15·포트 닫힘, 브라우저 12개/proxy 종료·임시 profile 제거/비밀 manifest0을 확인했습니다. 원천/업무/reference 행·숨긴 테이블·trigger·숫자 합성종목009991 모두 0입니다.
- FastAPI 외부 요청/서브프로세스/.env 읽기 시도0입니다. 브라우저의 외부 글꼴 origin 요청은 차단하고 별도 기록하므로 브라우저 요청 시도까지 0이라고 주장하지 않습니다. 예상 밖 public API/pageerror는0입니다.

실행 증거는 `tests/.runtime/backend-isolated/runs/<run>`의 summary/XML/FastAPI 교환/host 로그와 `frontend/tests/.runtime/browser-feature2-pipeline/<run>/<mode>`의 public 교환·DOM/PNG/PDF입니다. [오프라인 증거 대조기](../../tests/audit_feature2_pipeline_browser.py)는 실제 Java의 SQL/owner/404 단언 기록, public 응답, 원장, explain 입력, 모델 점수, frontend 해시와 종료 상태를 교차 확인합니다. Java가 수행한 실시간 SQL/권한 검사를 오프라인에서 DB 재접속으로 반복하는 것은 아닙니다.

```bash
python3 -B tests/audit_feature2_pipeline_browser.py tests/.runtime/backend-isolated/runs/run-kd_uymvv
```

대조 결과는 해당 run의 `feature2-pipeline-browser-evidence-audit.json`에 **PASS**로 기록했습니다. PDF 구조·직접 시각 확인은 브라우저 saved_quantitative_pdf의 `pdf-inspection.json`/`pdf-visual-review.json`에 있습니다. source hash는 실행 당시 입력을 특정하며, 후속 QA에서 공유 테스트 도구를 수정하면 과거 run과 현재 도구의 hash가 달라질 수 있습니다. 과거 감사 PASS를 새 소스 검증으로 해석하지 않습니다.

실행 5개의 이력(최종 포함 4개 완료, 1개 중단)·문서 12모드와 실제 결과·누적 25묶음의 합계는 `history-and-documentation-audit.json`으로 대조해 PASS했습니다. `feature2-pipeline-browser-source-audit.json`은 tests 밖 924개 변경/새 파일/소실 0입니다. `.venv_wsl/bin/python -B -m unittest discover -s tests -v`의 문서 68개 대상 링크·설정 일부 비밀값 검사 2개도 PASS이며 로그는 `tests/.runtime/backend-isolated/feature2-pipeline-browser-documentation.log`입니다. 이 문서 검사는 전체 보안 감사가 아닙니다.

## 초기 실행 이력

`run-apkh5fh9`의 기준1사례는 검색 전에 사용한 영문 합성코드 `QAF2SQL`이 UI의 국내6자리코드 필터에 맞지 않아 분석0회/원장0/report0에서 중단했습니다. 실제 로그인과 검색200은 확인했습니다. 이 실패는 테스트 fixture 오류로 분리하며 누적하지 않습니다. 브라우저/profile/proxy와 MySQL/Redis/FastAPI/Spring 및 격리 행은 정리·종료했습니다.

보정은 browser 단계의 기준종목만 소유SQL에서 `009991`로 바꾸고 같은 실제 resolver/카드/Peer/모델 경로를 사용합니다. 원본 fixture의 가격·독립 상관 oracle는 동일합니다. 각 사례 정리 전에 원래 합성코드로 복원하여 공통 source cleanup을 실행하며,별도 잔존 종목 조회도 추가했습니다. 제품의 검색 필터를 우회하거나 변경하지 않았습니다.

`run-8s_tn2hr`의 기준1사례는 실제 분석200·차감1·SQL/상세/소유권을 확인한 뒤,서버 단계의Peer3개 기대값을 UI에 그대로 적용해 FAIL했습니다. UI의 실제 요청은6개월/window120/peerCount30이며 기존 서버 기준은명시기간/window90/peerCount3입니다. 원천 후보4개인 이번 데이터에서는 품질필터 완화 분기가 달라선행/후행2개만 반환됐고,각 상관/조정상관/시차는 원래 독립 oracle와 일치합니다. 브라우저 검증은 반환된 각 종목의 수치·시차·표시 대응을 확인하고,30개 요청의 선정 알고리즘 자체를 독립 검증했다고 주장하지 않습니다. 이 초기1사례도 누적하지 않습니다.

`run-0mlt4yr_`은 첫 전체 12개 4 PASS/8 FAIL입니다. 증거 대조는 PASS였으며 실제 macro 응답의 `usFedFundsRate`가 null이고 화면의 4.25%가 없는 점은 확인했습니다. 다만 해당 수치 단언에서 필드를 `usFedFunds`로 적어 정상 응답에서도 FAIL할 오타가 발견됐습니다. 올바른 `usFedFundsRate`로 고치고 최종 전체 실행을 다시 시작했습니다. 기존 실패 이력/실제 PDF·SQL/모델 증거를 보존하되 최종 집계와 중복 합산하지 않습니다. 제품 소스는 바꾸지 않았습니다.

`run-1vpjzrq7`은 실행 세션 exit143/SIGTERM으로 중단됐습니다. 원인은 확정하지 못했으며 제품 실패로 분류하지 않습니다. browser summary는 6모드(1 PASS/5 FAIL)까지만 있고 다음 Peer 복구 모드는 완료하지 못했습니다. 최종 runner summary/JUnit/SQL 정리 감사가 없어 12개 전체 결과나 정상 종료·잔존 행0을 주장하지 않습니다. 후속 조회 환경에서는 소유 Spring/MySQL/Redis 포트에 연결되지 않았고 남은 합성 인증 manifest를 삭제했습니다. 새 외부 DB로 다시 실행하며 중단된 소유 datadir는 보존합니다. 세부는 해당 run의 `interruption-audit.json`입니다. 이 실행도 누적 집계에서 제외합니다.

## 남은 범위

실제 외부 금융/뉴스/LLM 제공자 전체 결합, 최대 30기사·cold/동시 모델 성능과 품질, 나머지 UI 언어·테마·접근성·해외/FX/시차 조합은 남아 있습니다. Feature1 실제 서버/원천/설명/저장 흐름과 나머지 Swagger 사용자 흐름, 프로세스 중단·여러 인스턴스·취소 과금/환불/멱등성·배치 복구도 전체 goal의 범위입니다. 이 단계로 전체 QA 완료를 선언하지 않습니다.
