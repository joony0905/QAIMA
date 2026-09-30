# Feature2 브라우저·실제 조립/HTTP/SQL fallback 검증

## 목적과 현재 상태

2026-09-28 [Feature2 실제 조립 검사](../../tests/isolated-feature2-fallback.md)의 다음 단계입니다. 실제 Chromium에서 현재 React 빌드를 사용해 로그인·검색·카드·종합 분석·저장 리포트/PDF를 실제 Spring HTTP/JWT/Feature2 조립기/크레딧 원장/MySQL에 연결했습니다. 최종 `run-l3jr_n8_`은 **13개7 PASS/6 FAIL**, errors/skipped0, JUnit118.381초입니다. 실행기 exit1은 제품 실패 단언의 결과이며 소유 MySQL은 exit0으로 종료했습니다.

수정은 tests 내부에 한정하고 기존 제품 파일·기존 DB/Redis를 보존합니다. MySQL은 실행기가 새 `/tmp/qaima-qa-http-*` 디렉터리에서 시작한 인스턴스이며 포트·schema·datadir·표식을 검사합니다. Redis/FastAPI는 이번 단계에서 시작하지 않습니다.

## 실제/대체 경계

- 실제: Linux Chromium → 현재 React/Vite 빌드 → 소유 프록시 → 제품 로그인/stock/chart/card/news/Feature2/report/user Controller → 실제 Feature2AnalyzeService/normalizer/assembler/CreditService/ReportService/JPA/MySQL.
- 프록시는 허용 경로를 검사하고 요청/응답을 그대로 전달합니다. Playwright에서 API 응답을 만들어 주거나 재작성하지 않습니다. 외부 origin과 service worker는 차단합니다.
- 합성: stock/chart/ranking/카드/뉴스/Peer/금리/설명 API와 추세 Repository 경계. 시장 원천/LLM 정확도 검사는 아닙니다. 이전 정상 조립 fixture를 재사용하되 정상 공매도 항목에는 두 비율/거래량/금액을 채워, 기존 nullable 결함을 별도 사례와 혼동하지 않도록 합니다.
- 리포트 저장 오류는 소유 MySQL INSERT trigger, 보조 시계열 오류는 실제 카드 Controller가 구독하는 service Mono.error입니다. 원천 provider/실제 Redis 장애는 후속 범위입니다.
- 로그인은 실제 폼/비밀번호/JWT/cookie를 사용합니다. 합성 계정의 암호를 전달하는 임시 manifest는 자식 종료 후 삭제하며 인증 본문·토큰·쿠키 값은 증거에 기록하지 않습니다.

## 검증 방법과 기대값

정책: [공통 분석/부분 성공/환불](../../policy/QAIMA_POLICY_PUBLIC.md), [설명 실패 시 정량 유지·리포트 흐름](../../policy/Feature_analysis_user_flow_policy.md), [저장 경고 계약](../../policy/API_DTO_CONTRACT_POLICY.md), [전체 fallback 인계 기준](../../tests/FALLBACK_QA_HANDOFF.md).

1. frontend source/copy/bundle의 SHA-256을 기존 검증 빌드와 대조한 후 실행합니다. 화면 selector/CSS/제품 소스를 바꾸지 않습니다.
2. 실제 로그인 후 합성 종목009991 검색→카드 요청 완료→분석 버튼을 눌러 POST 응답/잔액 재조회/화면 상태를 기록합니다.
3. HTTP의 독립 기대값 baseRate3.25/공매도7.5와 화면의 macro.krBaseRate3.50%/공매도7.50%·설명·부분 warning을 대조합니다. baseRate와 macro.krBaseRate는 서로 다른 합성 sentinel입니다. 뉴스 부분 점수는 기사2개 중1개null/평균.25를 대조합니다. 실제 scrollIntoView를 순서대로 수행해 보고서 각 section의 opacity≥.99를 확인한 뒤 전체 PNG/DOM을 저장하며 제품 CSS/class는 강제로 바꾸지 않습니다.
4. 실패 UI 단언은 가능한 한 모아서, 실제 성공 리포트 상세/저장 증거를 먼저 확보합니다. 브라우저가 실패해도 Java가 별도 JDBC로 실제 잔액·원장·report 수를 읽습니다.
5. 공개 HTTP data와 SQL snapshot을 숫자 값 기준으로 비교하고 warning 보존을 대조합니다. 저장 상세에서는 실제 GET resultSnapshot=분석 응답 및 사람이 읽을 수 있는 경고를 검사합니다.
6. 뉴스 첫 요청만 실패시킨 뒤 같은 페이지에서 명시적으로 다시 검색/분석하여 warning/기사 복구·총2차감을 확인합니다. 자동 재분석/재시도나 PDF 다운로드로 추가 차감이 없어야 합니다.
7. 정상 저장 PDF는 실제 다운로드 파일·PNG·페이지 수·해시를 남기고 별도 파싱/시각 검사합니다. 이 결과를 분석 직후 PDF/모든 입력의 완료 증거로 확대하지 않습니다.

## 실행

```bash
python3 -B tests/run_isolated_backend.py --suite feature2-browser
```

정상 구성 대조만 실행:

```bash
python3 -B tests/run_isolated_backend.py --suite feature2-browser --test baselineActualFeature2Browser
```

[Java host](../../tests/java/com/qaima/qa/IsolatedFeature2BrowserTest.java), [브라우저 script](browser-feature2-server.cjs), [실행기](../../tests/run_isolated_backend.py). Java 명령/XML/SQL 정리는 `tests/.runtime/backend-isolated/runs/<run>`에, public 요청/응답·DOM text/PNG/PDF·브라우저 정리는 `frontend/tests/.runtime/browser-feature2-server/<run>/<mode>`에 보존합니다.

## 최종 사례

| mode | 결과 | 검사/관측 |
|---|---|---|
| normal_saved_report_pdf | PASS | 실제 정상 분석/−1/report1→내정보 상세/동일 snapshot→요약 PDF 재다운로드 |
| news_partial | PASS | 기사2개/점수1개null·평균.25·실시간/저장 warning |
| peer_failure | PASS | Peer 오류/누락 안내와 독립 정량·뉴스·설명 유지 |
| explain_failure_saved | FAIL | live 정량/안내 유지, 저장 상세에는 경고만 표시되고 정량 누락 |
| combined_news_explain_failure | PASS | 뉴스+설명 오류의 두 경고/정량 유지 |
| industry_missing_partial | PASS | 분류 없음 warning/남아 있는 금리·공매도 표시; 기존 서버 후속 생략은 별도 FAIL 유지 |
| core_failure | FAIL | 정량0/INTERNAL_ERROR 안내 누락·환불 없이−1/report1 |
| report_save_failure | FAIL | 실제 SQL 저장 실패의 안내 누락, 정량 결과/차감 유지/report0 |
| aux_base_series_failure | FAIL | 보조 금리 시계열500으로 성공 분석 숨김·저장 조회는 성공 |
| aux_short_series_failure | FAIL | 보조 공매도 시계열500으로 성공 분석 숨김·저장 조회는 성공 |
| recovery_after_news_failure | PASS | 첫 뉴스 오류 후 명시 재분석에서 warning 제거/기사 복구·총2차감 |
| insufficient | PASS | 실제402와 안내/잔액0·원장/report0 |
| text_only_explain | FAIL | overall 없는 text 설명의 live 보고서 누락 |

## 증거 대조·정리

- host: `tests/.runtime/backend-isolated/runs/run-l3jr_n8_`의 summary/XML/13개 브라우저 로그. 브라우저: `frontend/tests/.runtime/browser-feature2-server/run-l3jr_n8_/<mode>`의 summary/public 응답/DOM/PNG/PDF. 대조 결과는 host의 `feature2-browser-evidence-audit.json`입니다.
- Chromium153.0.8010.12, MySQL8.0.46, migrations51개/48테이블. host 입력577개·프론트 원본/빌드 복사본187개·bundle11개 해시를 대조했습니다. tests 밖 가시 코드/설정/문서924개 변경·소실·새 파일0 감사는 `tests/.runtime/backend-isolated/feature2-browser-source-audit.json`입니다.
- 분석 POST는200×13/402×1, 실제 저장 snapshot12개 대조, 저장 상세 GET5개입니다. 원장 총13차감에는 잘못된 core_failure 차감1도 포함되므로 정상 과금13건이라는 뜻은 아닙니다. recovery의 추가 요청/SQL 단언을13사례와 중복 집계하지 않습니다.
- 브라우저13개 모두 예상 밖 API/pageerror0, 브라우저·프록시 종료/임시 프로필 삭제를 확인했습니다. 정적 폰트/CDN의 외부 origin 요청 시도는 route에서 차단해 별도 집계했으며 외부 요청 시도0이라고 주장하지 않습니다. 인증 본문/응답은 증거에서 제외했습니다. 임시 manifest0, 업무9테이블/trigger0, MySQL stopped true/exit0/포트 닫힘을 확인했습니다.
- 정상 PDF는10,408,847bytes/A4 1페이지/수리 불필요/빈 페이지0, darkInkRatio .00482입니다. SHA-256은 `78cf74577b11bd5d117139d4848f2731c55c9b5d8d5e3e27653ba85dcf3d23a2`입니다. `inspect-report-pdf.py`로 파싱·렌더링하고 최종 분석 PNG/정상 PDF 페이지 PNG/설명 실패 저장 상세 PNG를 직접 확인했습니다.
- 정상 저장 PDF에는 메타데이터·설명 요약이 보입니다. **정량 수치·차트 전체를 재현하는 PDF로 통과 판정하지 않았습니다.** 기존 장문 PDF 스타일/페이지 분할 실패도 그대로 유지합니다.

PDF 대조 명령:

```bash
PYTHONPATH=frontend/tests/.runtime/pdf-tools PYTHONDONTWRITEBYTECODE=1 .venv_wsl/bin/python -B frontend/tests/inspect-report-pdf.py --artifacts frontend/tests/.runtime/browser-feature2-server/run-l3jr_n8_/normal_saved_report_pdf
```

## 초기 실행 이력

`run-02clehoh`는 정상 대조1개 FAIL입니다. 실제 로그인·분석200/−1·SQL report1·저장 상세·PDF 다운로드는 실행됐으나, 원래 fixture의 text-only 설명은 live 보고서에서 누락됐습니다. `parseFeature2Explain`이 overall=null을 빈 객체로 바꾸고 AnalysisResultPanel이 이 객체를 우선하여 text fallback에 도달하지 않는 코드와 일치합니다. 저장 상세에는 동일 text가 표시됩니다. 이 실패를 삭제하지 않고 별도 text_only_explain 사례로 추가했으며, 일반 정상 대조는 overall.summary가 있는 구조화된 설명을 사용합니다. 초기 MySQL/브라우저/프록시/프로필 정리·업무9테이블/trigger0·입력 변경0을 확인했습니다. 전체 실행과 중복 합산하지 않습니다.

첫 전체 `run-_7ntos21`은13개5 PASS/8 FAIL입니다. 그중 산업 분류 없음 사례가 화면에 없는 별도 baseRate3.25를 요구한 것과, 잔액 부족의 실제 문구 “분석 토큰이 부족합니다.”를 기다리지 못한 것은 테스트 기대값 오류2개입니다. 실제 표시하는 macro 금리3.50%와 문구에 맞게 보정했습니다. 나머지6개 제품 실패는 유지합니다. 하단 section의 스크롤 등장 동작을 실제 스크롤/opacity 대조로 보강하고, Peer 합성 누적 수익률도 fraction .12(12%)·anchor .20(20%)로 맞췄습니다(산업지수는12 percentage points). Peer 차트 전체 정확도 통과를 주장하지 않습니다. 첫 전체 실행도 소유 서버/브라우저 정리·업무9테이블/trigger0·입력 변경0을 확인했습니다.

## 확인된 실패 경로

- **FRONT-F2-AUX-001 (기존 결함, 실제 서버/SQL 보강)**: 종합 분석200·원장−1·report1인데, 함께 요청한 base-rate-series 또는 short-selling-series가500이면 handleAnalyzeClick의 Promise.all이 reject되어 분석 결과가 표시되지 않습니다. 내정보의 저장 리포트는 조회됩니다. 서버의 정상 계산/저장이 브라우저 분석 실패 화면과 함께 나타나는 경우입니다.
- **FRONT-F2-REPORT-SAVE-001**: 실제 report INSERT가 실패하면 HTTP meta에 REPORT_SAVE_FAILED와 정상 분석값이 있고 원장−1/report0입니다. 분석 화면은 정상값을 보여주지만 저장 실패 안내는 없습니다. Feature2는 meta를 보존하며 warningNotes의 등록되지 않은 코드 필터에서 안내가 빠집니다. 기존 Feature3의 API client meta 소실과 원인 경계가 다릅니다.
- **FRONT-F2-CORE-WARNING-001 / F2-CORE-REFUND-001**: 실제 조립기가 반환한 metrics 전부null/INTERNAL_ERROR 응답에서 화면의 오류 안내가 없고, 실제 차감−1/report1이 남습니다. warningNotes는 FEAT2_INTERNAL_ERROR는 알고 있지만 INTERNAL_ERROR는 처리하지 않습니다. 환불/빈 결과 저장 문제는 앞선39개 조립 검사에서 확인한 기존 실패입니다. 화면의 거시 카드 값은 앞선 카드 조회에서 남아 있는 값이므로 정상 종합 분석으로 해석하지 않습니다.
- **FRONT-F2-SAVED-METRICS-001**: 설명 API 오류에서 live 보고서의 정량 값·LLM_EXPLAIN_FAILED 안내와 SQL snapshot은 보존되지만, 저장 상세는 메타데이터/경고만 보여줍니다. SavedReportDocument가 explain/summary/warnings만 렌더링하고 metrics를 사용하지 않는 경로와 일치합니다. 정책의 설명 실패 시 정량 결과 유지와 사용자 재열람 흐름을 기준으로 검사했습니다. DB의 정량 값 소실을 의미하지 않습니다.
- **FRONT-F2-TEXT-EXPLAIN-001**: explain.text가 있고 overall이null인 합성 설명을 실제 서비스/SQL로 전달하면 live 보고서에서 설명 본문이 빠집니다. parseFeature2Explain의 overall={}와 AnalysisResultPanel의 truthy 분기가 text fallback을 가립니다. 초기 저장 상세에는 같은 설명이 보였습니다. 실제 LLM이 해당 형태를 얼마나 반환하는지는 측정하지 않았습니다.

보조 조회 실패2개에서 설명 누락 단언도 함께 발생하지만, 해당 두 사례의 원인은 결과 전체가 표시되지 않는 FRONT-F2-AUX-001입니다. 독립 text fallback 결함은 text_only_explain으로 구분합니다. 실패6사례를 고유 결함6건으로 세지 않습니다.

## 남은 범위

실제 하위 제공자·Redis 장애의 누적 대기/클라이언트 취소, 실제 FastAPI 설명, 전체 카드/Peer 차트·단위/언어/접근성·기능1 통합 흐름은 별도입니다. 이 단계가 기존 제품 실패를 수정하거나 전체 QA를 완료하지는 않습니다. [전체 진행기록](../../tests/QA_PROGRESS_2026-09-28.md)의 범위를 유지합니다.
