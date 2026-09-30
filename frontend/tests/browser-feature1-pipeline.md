# Feature1 실제 서버·SQL·계산·브라우저/저장/PDF QA

## 상태

2026-09-28 최종 `run-k_l2x_cx`: **17개 5 PASS / 12 FAIL**, errors/skipped 0입니다. JUnit 194.705초, Gradle 5분 8초/exit 1이며 12개 제품 실패가 종료 코드에 반영됐습니다. 초기 두 실행은 중복 합산하지 않습니다. 누적은 **27묶음575개434 PASS/141 FAIL**이며 전체 QA는 미완료입니다. 실패 사례 수는 고유 결함 수와 다릅니다.

## 방법

```bash
python3 -B tests/run_isolated_backend.py --suite feature1-pipeline-browser
```

- [서버 검증](../../tests/isolated-feature1-pipeline.md)의 실제Feature1 서비스·JPA·snapshot/TTM·Redis·FastAPI·JWT·차감/report SQL에 실제Stock/Chart/FinancialController와CandleLoadService/FinancialReadService를 연결합니다.
- 실제Chromium→React의 로그인/캔들·재무 조회/분석·저장/PDF를 실행합니다. frontend소스/복사본187개·bundle11개 해시를 기존 검증 빌드와 대조합니다. 공개 API 응답을 가로채 바꾸지 않는 loopback 프록시를 사용합니다.
- 검색/종목 조회의 시세·외부 캔들 제공자·랭킹은 합성 경계입니다. SQL에 저장하는160가격/6재무/발행주식1000주와시세180의 수치 기대값은 앞선fixture를 공유하며,브라우저용6자리종목009993을사용합니다.
- 초기 종목조회는 정상으로 끝낸 뒤 테스트 전용WebFilter가첫분석POST 직전에실제SQL테이블숨김·가격삭제·Redis fresh시세삭제/provider오류·계산503·report INSERT trigger를적용합니다. 보조재무 오류는분석성공후Q재무조회 직전에SQL을숨깁니다. 이필터는응답/분석데이터를재작성하지 않습니다.
- 정상/저장PDF/무키설명 안내·계산503/계산복구·snapshotSQL/snapshot복구·재무SQL·가격SQL·캔들provider·빈캔들·stale시세/stale복구·저장SQL·보조재무SQL·snapshot+계산 복수장애·날짜선택해제를검사합니다.
- 각시나리오의공개응답/DOM/PNG·실제원장/잔액·report snapshot/소유자상세/타사용자404·Redis·FastAPI호출범위·단계별시세출처를보존합니다. 같은브라우저와서버에서장애를해제하고종목을다시검색/분석해복구를확인합니다. 복구시캐시수동삭제는없습니다.
- 실제제공자/유료모델·모든시장/날짜·동시성/프로세스중단과전체QA는후속범위입니다. DB/Redis는새외부격리인스턴스이고수정은tests내부로제한합니다.

## 파일

[브라우저 실행](browser-feature1-pipeline.cjs) · [Spring 호스트](../../tests/java/com/qaima/qa/IsolatedFeature1PipelineBrowserTest.java) · [상위 인계](../../tests/FALLBACK_QA_HANDOFF.md) · [누적 기록](../../tests/QA_PROGRESS_2026-09-28.md)

## 17개 사례와 판정

JUnit 정상 메서드는 `aActualSqlCalculationLiveBrowser()`이며, 나머지 16개는 `bActualFallbackBrowser(String)`의 `actual Feature1 browser {mode}`입니다. 아래 mode는 브라우저 산출물 디렉터리와 동일합니다. PASS는 해당 단언 범위에 대한 판정이며 아래 별도 볼린저/PDF 검토까지 통과했다는 의미가 아닙니다.

| mode | 주입·입력과 확인 방법 | 실제 결과 | 판정 |
|---|---|---|---|
| normal_live | 날짜 2026-01-01~2026-09-26, 실제 SQL→계산→DOM 수치·재무 차트·차감/저장 대조 | EMA 480개/null·TTM 비율·재무 GET·유한 SVG 좌표 일치 | PASS |
| saved_quantitative_pdf | 정상 분석 후 live PDF→설정/보기→실제 상세 GET→저장 PDF | SQL에는 정량이 있으나 저장 화면/PDF에서 빠짐 | FAIL |
| missing_key_warning | 실제 설명 API에 키 없음, public/SQL 경고와 화면 대조 | LLM_API_KEY_MISSING 사용자 안내 없음 | FAIL |
| calculation_down | 실제 FastAPI 503 | schema 0.1 대체 결과·160가격·재무 비율/차트·계산 제한 안내·저장 유지 | PASS |
| calculation_ui_recovery | 계산 503 후 장애 해제, 같은 브라우저에서 동일 입력 재분석 | 0.1→1.0, 정상 EMA/ROE·두 번 차감/저장, 수동 캐시 삭제 없음 | PASS |
| snapshot_sql | 분석 직전 실제 market_snapshot 테이블 숨김 | 재무 6행을 보내도 ROE 12.5 대신 null, 기술지표/재무 시계열은 남음 | FAIL |
| snapshot_ui_recovery | 같은 snapshot 장애 후 테이블 복원·동일 입력 재분석 | 최초 ROE 누락은 FAIL, 후속 정상 ROE/EMA/차트 복구 확인 | FAIL |
| financial_sql | 분석 직전 실제 financial 테이블 숨김 | 독립 가격도 data=null, 환불 정상/저장 0, 화면 실패 안내 없음 | FAIL |
| price_sql | 분석 직전 실제 price_ohlcv 테이블 숨김 | 독립 재무도 data=null, 환불 정상/저장 0, 화면 실패 안내 없음 | FAIL |
| candle_provider | 외부 캔들 경계 오류, 실제 저장 캔들 160개 유지 | 기존 SQL 캔들로 대체하지 못하고 data=null, 환불/화면 안내 누락 | FAIL |
| empty_candles | 초기 차트 로딩 후 실제 가격 행 삭제 | 정량/재무는 부분 유지하나 CHART_DATA_UNAVAILABLE live 안내 없음·저장 경고도 소실 | FAIL |
| stale_price | fresh 시세 Redis 키 삭제 후 quote 오류 | stale 180/비율 유지, 단계 PRICE_STALE_USED가 public/화면에서 소실 | FAIL |
| stale_ui_recovery | stale 사용 후 provider 복원·동일 입력 재분석 | 최초 출처 안내 누락은 FAIL, 후속 정상 결과/단계 경고 제거 확인 | FAIL |
| report_insert | 실제 SQL INSERT trigger 오류 | 계산/차감 유지·reportId=null·REPORT_SAVE_FAILED 있으나 화면 안내 없음 | FAIL |
| aux_financial_sql | 분석 성공 후 Q 재무 보조 GET 직전 SQL 숨김 | 실제 보조 GET 500에도 분석 수치·리포트/차감 유지 | PASS |
| combined_snapshot_api | snapshot SQL 오류+계산 503 동시 주입 | 가격 요약·재무 시계열·계산 제한 안내·저장 유지; 전체 지표 복구를 뜻하지 않음 | PASS |
| cleared_date_range | 사용자가 두 날짜 입력을 지운 뒤 분석 | UI의 non-midnight 시간대 날짜→Python 422·기간 길이 53→SQL 50 초과/저장 실패 | FAIL |

정상 입력은 종가 `100+0.5×i` 160개, 최근 6개 분기 재무, 발행주식 1000주·합성 현재가 180입니다. EMA 20/60/120의 총 480개와 초기 null을 독립 fixture와 1e-9 이내로 비교합니다. 화면은 실제 검증된 API 값의 소수 1자리 반올림과 비교하고 독립 값 대비 오차 0.050000001 이내를 확인합니다. TTM 매출 15000/순이익 1500에서 PER 120/PBR 15/PSR 12/ROE 12.5를 대조합니다. Q 재무 GET은 5행이며 선택 기간에 표시되는 차트는 25.Q4/26.Q1/26.Q2의 3개입니다.

## 실패의 전달 경로

- **FRONT-F1-SAVED-METRICS-001**: public metrics=SQL result_snapshot=소유자 상세가 일치해도 [SavedReportDocument](../src/pages/SettingPage.tsx)는 설명/경고 위주로 렌더하고 metrics를 표시하지 않습니다. 무키 설명 fallback에서 live EMA/ROE는 남지만 저장 화면과 PDF에는 없습니다. Feature2에서 확인한 저장 정량 누락과 공통 컴포넌트의 같은 경계입니다.
- **FRONT-F1-SOURCE-WARNING-001**: 실제 무키 설명의 LLM_API_KEY_MISSING이 public/SQL에 남지만 [warning mapper](../src/utils/warningNotes.ts)에 대응 안내가 없습니다. 정상 일부 지표 부족 안내와 설명 실패 안내를 구분합니다.
- **FRONT-F1-CORE-WARNING-001**: 가격/재무 SQL·캔들 provider 오류의 HTTP 200/meta failure/data null, 실제 USE−1/REFUND+1을 확인했습니다. UI에는 분석 전 차트에서 만든 가격 요약과 빈 투자지표 카드가 보이며 실패 안내는 없습니다. 화면이 완전히 사라진다고 기술하지 않습니다. 원천 독립성 실패는 기존 `F1-ASSEMBLY-ISOLATION-001`, `F1-CANDLE-FALLBACK-001`입니다.
- **FRONT-F1-META-WARNING-001**: CHART_DATA_UNAVAILABLE 및 REPORT_SAVE_FAILED가 meta에 있어도 live 화면 안내로 이어지지 않습니다. 빈 캔들 경고는 data/SQL/상세에도 없어 기존 `F1-REPORT-WARNING-001`을 함께 재현합니다.
- 기존 `F1-SNAPSHOT-FALLBACK-001`, `F1-SNAPSHOT-WARNING-001`, `F1-DATE-WIRE-001`, `REPORT-F1-WINDOW-001`을 실제 브라우저 입력까지 연결했습니다. 저장 회사명/종목코드 역전 `REPORT-MAPPING-001`도 PDF에서 관측했습니다. tests-only 범위이므로 제품은 수정하지 않았습니다.

계산/snapshot/stale 복구 3개는 첫 번째와 두 번째 공개 요청 body 전체가 같습니다. 같은 브라우저·서비스에서 장애만 제거했으며 캐시를 수동으로 비우지 않았습니다. 최종 schema 1.0·160가격·ROE 12.5·잔액 3과 실제 계산 호출을 대조했습니다. snapshot/stale 사례의 후속 복구 성공이 최초 fallback FAIL을 없애지는 않습니다. 이 결과를 Feature2의 같은 키 Peer 오류 캐시 잔류 해결로 확대하지 않습니다.

## 증거 대조와 PDF

최종 서버 증거는 `tests/.runtime/backend-isolated/runs/run-k_l2x_cx`, 브라우저 증거는 `frontend/tests/.runtime/browser-feature1-pipeline/run-k_l2x_cx`에 있습니다. 각 mode의 `summary.json`, 분석/복구 DOM 텍스트·PNG, 실제 공개 요청/응답, SQL/원장 관측을 보존합니다.

- [증거 검사기](../../tests/audit_feature1_pipeline_browser.py): 공개 응답 20개, SQL snapshot=소유자 상세 15개, 타 사용자 404×15, 브라우저 상세 GET 2개, USE 20/REFUND 3, 실제 FastAPI 17개(정상 Python 13·503 3·422 1)를 대조했습니다. Python metrics 13개를 public과 비교하고 wire 가격 2560행분/재무 102행분을 fixture와 비교했습니다. 반복 입력을 포함하는 대조 횟수이며 고유 행 수가 아닙니다.
- 최종 Redis metrics 14사례와 마지막 성공 응답을 비교했습니다. 빈 캔들 report 10의 public/저장 경고 불일치는 결함 증거로 보존합니다. 예상 밖 API/pageerror 0, 외부 글꼴 origin 42회 차단입니다.
- PDF 2개/3페이지는 수리 불필요·A4·비어 있지 않음 검사 PASS입니다. live 45,606,315bytes/2페이지, saved 10,592,447bytes/1페이지입니다. 전 페이지 PNG를 직접 확인했고 `pdf-visual-review.json`에 파일 해시·페이지별 검토를 남겼습니다.
- live PDF에서는 숫자/재무 차트가 남지만 카드·그리드 스타일이 사라지고 이자보상배율 문구/값이 1~2페이지 경계에 걸쳐 잘립니다. 기존 `FRONT-F1-PDF-STYLE-001` 관측을 유지합니다. 저장 PDF는 정량이 빠진 화면 그대로입니다. 파일 구조 PASS를 내용/시각 품질 PASS로 해석하지 않으며 PDF 검토를 JUnit 집계에 추가하지 않습니다.

```bash
python3 -B tests/audit_feature1_pipeline_browser.py tests/.runtime/backend-isolated/runs/run-k_l2x_cx
python3 -B tests/audit_feature1_band_position.py frontend/tests/.runtime/browser-feature1-pipeline/run-k_l2x_cx
PYTHONPATH=frontend/tests/.runtime/pdf-tools PYTHONDONTWRITEBYTECODE=1 .venv_wsl/bin/python -B frontend/tests/inspect-report-pdf.py --artifacts frontend/tests/.runtime/browser-feature1-pipeline/run-k_l2x_cx/saved_quantitative_pdf
```

증거 검사 PASS는 실행 증거의 일관성에 대한 판정입니다. 볼린저 검사 exit 1은 아래 제품 표시 FAIL이며 JUnit 사례를 늘리지 않습니다. 현재 소스 해시를 비교하는 검사이므로 이후 입력 변경 뒤 과거 run을 그대로 재실행하면 hash 차이가 날 수 있습니다.

## 정리와 범위

최종 입력 611개 변경 0, frontend 소스/복사본 187개·bundle 11개 해시를 확인했습니다. 소유 MySQL 업무 9/원천 4테이블·QA종목·숨긴 테이블·trigger 0, Redis DBSIZE 0과 두 서버 exit 0입니다. FastAPI active/outbound/dotenv/subprocess 시도 0, SIGTERM−15로 종료했고 강제 kill은 없습니다. 브라우저 17개·프록시·임시 프로필·합성 인증 manifest도 정리했습니다.

초기 두 실행과 최종 실행의 정리, tests 밖 924개 보존, 문서 70개 대상 링크/설정 비밀값 검사 2개, 17사례/누적27묶음 대응은 최종 `history-and-documentation-audit.json`에 기록합니다. 이 문서에 없는 실제 금융 제공자/유료 LLM·전체 시장/날짜·취소/프로세스 중단·동시성 및 나머지 Swagger 사용자 흐름은 계속 남아 있습니다.

## 초기 실행 이력

`run-8pyzrz5_`: 정상 기준1개 FAIL은 실제EMA60=164.7499999999998 부근값의화면164.7을닫힌식164.75→164.8과exact비교한테스트기대값오류입니다. 실제480 EMA/null과독립기대값은1e-9내일치하고정상SQL/저장/원장/DOM까지연결됐습니다. 화면은검증된실제API값의toFixed(1)과비교하고독립값대비표시허용오차0.050000001을유지합니다. 초기실행은제품실패로합산하지않습니다. 이후 전체17개를 실행했으며 초기1개는 합산하지 않습니다.

## 별도 수치 표시 검토

`FRONT-F1-BB-POSITION-001`: 실제가격179.5·BB상단180.5162812973354/하단168.9837187026646에서가격의밴드내위치는91.18772355239581%인데화면은50.0%입니다. [IndicatorSnapshotCards](../src/components/IndicatorSnapshotCards.tsx)의percentB가최신종가대신중심선을사용합니다. [John Bollinger 공식설명](https://www.bollingerbands.com/_files/ugd/58be43_b120ddf0184540608baf19e2c0ae2019.pdf)의최근종가위치정의와 [TradingView 공식계산식](https://www.tradingview.com/support/solutions/43000501971-bollinger-bands-b-b/)을2026-09-28확인했습니다. 제품import없이`100×(종가−하단)/(상단−하단)`을계산하고API값/동일날짜/DOM텍스트/PNG를대조하는 [별도검사](../../tests/audit_feature1_band_position.py)를추가했습니다. 기존JUnit17사례집계와구분하며실행중테스트를변경하지않습니다.

첫전체 `run-a3h0jvtk`:17개2 PASS/15 FAIL입니다. 그중계산503/계산복구/복수장애3개는실제`일부 보조지표는 분석 구간 또는 데이터 상태에 따라 계산되지 않을 수 있어요.` 안내를테스트정규식이놓친오탐입니다. 안내문구를확인한뒤`계산되지`를수용하도록기대값을보정합니다. 실제13개Python수치/20응답/15저장/USE20·REFUND3·복구3요청의증거감사는PASS이고PDF2개3페이지파일검사도PASS입니다. 최종 `run-k_l2x_cx`는 5 PASS/12 FAIL로 확정했으며 초기 전체는 중복 합산하지 않습니다.

최종 별도 볼린저 검토는 정상 Python/schema1.0·160가격을 가진 live/복구 보고서 **12개 모두 FAIL**입니다. `bollinger-position-review.json`에 각 API 시각·밴드/종가·독립 기대값·DOM/PNG 해시를 보존했습니다. 이12개는 기존17사례의 추가 단언으로 누적 사례 수에 더하지 않습니다.
