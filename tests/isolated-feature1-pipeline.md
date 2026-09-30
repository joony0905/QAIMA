# Feature1 실제 SQL·snapshot·Redis·계산·과금/저장 QA

## 상태

최종 `run-y4nj39wc`: **23개13 PASS/10 FAIL**, errors/skipped0, JUnit50.075초·Gradle2분59초입니다. 최초 정상 기준3회는 중복 합산하지 않습니다. 전체 누적은 **26묶음558개429 PASS/129 FAIL**이며 전체 QA는 미완료입니다. 실패 사례10개는 아래8개 결함 묶음에 대응하며 고유 결함10개를 뜻하지 않습니다.

[테스트](java/com/qaima/qa/IsolatedFeature1PipelineTest.java), [독립 입력/기대값](build_feature1_pipeline_fixture.py), [HTTP 장애/기록 서버](feature1_pipeline_asgi.py), [증거 감사](audit_feature1_pipeline.py)를 사용했습니다. 제품 파일은 수정하지 않았습니다.

## 검증 방법

```bash
python3 -B tests/run_isolated_backend.py --suite feature1-pipeline
```

- 새 외부 격리 MySQL/Redis와 실제 Spring 공개 HTTP/JWT/차감/report SQL, 실제 가격·재무·발행주식 JPA, snapshot/TTM/실시간 비율 계산, 실제 FastAPI 기술지표를 연결합니다.
- 160개 합성 일별 가격은 close=100+i/2입니다. 제품 계산을 import하지 않는 fixture에서 EMA20/60/120의 닫힌 식, BB20의 모집단 분산, 마지막 stochastic K/D를 구하고 실제 응답과 대조합니다. 합성 날짜는 거래일 정확도 검증이 아닙니다.
- 분기 재무6행과 발행주식1000주/시세180으로 TTM 매출15000·순이익1500, PER120/PBR15/PSR12·ROE12.5 등의 독립 기대값을 대조합니다.
- SQL 원천4종은 소유 테이블 이름 변경으로 실제 조회 오류를 만들고 복원합니다. 계산 서버503/잘못된JSON·실제Python422, 원천 제공자 오류, Redis stale 시세, SQL 저장 trigger, 복수 장애를 주입합니다. 동일 service의 후속 정상 요청으로 복구를 확인합니다.
- public 응답=SQL snapshot=소유자 상세, 타 사용자404, 잔액·원장·리포트 메타데이터, 실제 FastAPI 입출력, Redis metrics를 기록합니다. 실패 전에 증거를 출력합니다.
- 외부 시세/캔들 제공자는 합성 경계이며 기존 사용자DB·외부 유료 모델을 사용하지 않습니다. 실제 브라우저 결합·실원천·전체 기간/시장/동시성은 후속 범위입니다.

## 23개 사례와 판정

공통으로 실제 공개POST를 보낸 뒤 응답·SQLsnapshot·소유자 상세·타 사용자404·잔액/원장·Redis를 기록합니다. 복구 사례는 같은Spring service/FastAPI 프로세스에서 장애를 제거하고 재요청합니다. 각 사례의 전체 메서드명·증거가 summary/XML에 남습니다.

| JUnit 사례 | 주입/입력 | 결과 | 직접 확인한 내용 |
|---|---|---|---|
| `aBaselineActualSqlSnapshotIndicatorsCacheAndReport()` | 정상/중급자 | PASS | 160가격·480EMA 초기null/닫힌 식·마지막BB/stochastic·TTM 비율·SQL/상세/캐시·차감1 |
| `actualPipelineReportMetadataMustRetainStockIdentity()` | 종목/회사명 저장 | FAIL | REPORT-MAPPING-001: 두 컬럼이 실제 SQL/상세에서 역전 |
| `actual FastAPI fault unavailable` | 계산HTTP503→복구 | PASS | schema0.1/가격·재무/snapshot 보존·경고/저장·동일 service schema1.0 복구 |
| `actual FastAPI fault malformed` | 계산HTTP200 잘못된JSON→복구 | PASS | 실제 JSON 파싱 오류 후 부분 결과/경고/저장·복구 |
| `combinedSnapshotAndCalculationFailureKeepsStoredPriceAndFinancialData()` | snapshot SQL+계산503 | PASS | 두 오류에도160가격·재무시계열/저장 유지·장애 제거 후 정상 수치/저장 복구 |
| `explicitTimezoneDateRangeMustSaveWithoutColumnOverflow()` | 25자 시간대 날짜2개 | FAIL | REPORT-F1-WINDOW-001: 실제 계산성공이나 기간53자→SQL1406/저장 실패 |
| `insufficientBalanceAndAnonymousCallsDoNotSubscribeToSources()` | 잔액0·JWT없음 | PASS | 실제402/401·원천 구독/계산/원장/report0 |
| `investment level 초급자` | 초급자 | PASS | 동일 실제 가격·기술지표/재무 기대값과 저장/차감 |
| `investment level 고급자` | 고급자 | PASS | 동일 실제 가격·기술지표/재무 기대값과 저장/차감 |
| `investment level 전문가` | 전문가 | PASS | 동일 실제 가격·기술지표/재무 기대값과 저장/차감 |
| `missing LLM key gemini` | Gemini 키 없음 | PASS | 실제 무키 설명 실패 경고·정량/저장 보존·외부요청0 |
| `missing LLM key GPT-5 mini` | OpenAI 키 없음 | PASS | 실제 무키 설명 실패 경고·정량/저장 보존·외부요청0 |
| `noCandlesKeepsFinancialSnapshotAndChartWarning()` | 가격0·재무 정상 | FAIL | F1-REPORT-WARNING-001: 재무유지/public CHART_DATA_UNAVAILABLE 있지만 저장 경고에서 소실 |
| `noFinancialsKeepsActualIndicatorsAndPartialSnapshotWarning()` | 재무0·가격 정상 | PASS | 160가격/실제 기술지표·빈 재무시계열·부분 snapshot 경고/저장 |
| `nonMidnightEndDateMustReachActualIndicatorCalculation()` | UTC 종료일23:59:59 | FAIL | F1-DATE-WIRE-001: public 입력→내부Python date 검증422, 기술지표 없는schema0.1 |
| `source SQL unavailable price_ohlcv` | 가격SQL1146→복구 | FAIL | F1-ASSEMBLY-ISOLATION-001: 독립 재무도public data=null·계산0/환불, 복구 정상 |
| `source SQL unavailable financial` | 재무SQL1146→복구 | FAIL | F1-ASSEMBLY-ISOLATION-001: 독립 가격도public data=null·계산0/환불, 복구 정상 |
| `quoteStaleFallbackWarningMustReachPublicReport()` | fresh시세 삭제+시세provider 오류 | FAIL | F1-SNAPSHOT-WARNING-001: 실제Redis stale 시세180/비율은 유지하나 출처 경고가public/SQL에서 소실 |
| `realPythonValidationFailureUsesSpringPartialFallback()` | SQL 첫종가NULL | PASS | 실제Python422 위치input_data/ohlcv/0/c·Spring 가격/재무 fallback·저장/차감 유지 |
| `reportInsertFailureKeepsActualNumericsAndCachedMetrics()` | 실제report INSERT trigger | PASS | SQL45000에도 실제 수치/Redis/차감 유지·report0/REPORT_SAVE_FAILED |
| `snapshot SQL unavailable market_snapshot` | snapshotSQL1146→복구 | FAIL | F1-SNAPSHOT-FALLBACK-001: 재무6행 있어도 빈 snapshot 객체가Python 재무fallback을 막음; 기술지표/저장/복구는 유지 |
| `snapshot SQL unavailable issued_shares` | 주식수SQL1146→복구 | FAIL | F1-SNAPSHOT-FALLBACK-001: 주식수가 필요 없는ROE까지null; 재무6행·기술지표/저장/복구는 유지 |
| `unavailableCandleProviderMustKeepAlreadyStoredCandles()` | 캔들provider 오류→복구 | FAIL | F1-CANDLE-FALLBACK-001: SQL기존160가격이 있어도data=null/계산0/환불; provider복구 뒤 정상 |

## 실패 원인과 범위

- **F1-ASSEMBLY-ISOLATION-001(2사례)**: [FeatOneService](../backend/src/main/java/com/qaima/service/featone/FeatOneService.java)의 가격·재무·snapshot `Mono.zip` 앞에서 가격/재무SQL 오류를 부분 결과로 변환하지 않아 전체data가null입니다. 두 경우 실제 차감−1/환불+1·잔액5·report0·계산API0입니다. 독립 결과 보존 실패이며 환불 자체는 정상입니다.
- **F1-CANDLE-FALLBACK-001**: SQL에160개 가격을 읽은 뒤 최신일 보충 provider가 오류를 내면 기존 값 반환까지 도달하지 못합니다. public data=null/환불이며 같은 provider 복구 뒤160가격·지표·snapshot이 정상입니다. 정상 경계의 외부캔들 응답은 빈 목록이고 기존SQL을 사용합니다.
- **F1-SNAPSHOT-FALLBACK-001(2사례)**: snapshot 또는 issued_shares SQL 오류를 optional 빈 값으로 흡수해도 `toFeatOneMarketSnapshot`이null필드를 가진 객체를 만듭니다. 실제 wire에재무6행+빈 snapshot 객체가 함께 전달됩니다. [Python](../analysis/app/api/feature1.py)은 snapshot이 제공됐다고 판단해 곧바로 반환하므로 재무 기반ROE12.5 등을 재계산하지 않습니다. 실제 가격/지표는 살아 있고 관련missing/partial 경고와 저장도 유지됩니다. 없는 주식수에서PER를 억지로 기대한 테스트가 아닙니다.
- **F1-SNAPSHOT-WARNING-001**: 실제 단계의MarketMetricResult에`PRICE_STALE_USED`가 있으나 [MarketSnapshotService](../backend/src/main/java/com/qaima/service/stock/MarketSnapshotService.java)의 DTO 변환 후public/SQL 경고에서 사라집니다. Redis fresh시세만 지우고 provider 오류를 주입했으며 stale 시세180·PER120/PBR15/PSR12는 유지됐습니다. 자연TTL 만료나 이 사례의후속 provider복구는 이번 범위에 포함하지 않습니다. 정상 snapshot의`SNAPSHOT_FALLBACK_USED` 출처도public에는 없습니다.
- **F1-REPORT-WARNING-001**: [FeatOneController](../backend/src/main/java/com/qaima/api/feat1/FeatOneController.java)가 빈 캔들 경고를meta에만 추가하고 [저장service](../backend/src/main/java/com/qaima/service/report/AnalysisReportService.java)는data.warnings를 저장합니다. 재무수치는 그대로이나 저장 상세에는`CHART_DATA_UNAVAILABLE`이 없습니다.
- **F1-DATE-WIRE-001**: 유효한public 종료일`2026-09-26T23:59:59Z`가 내부Python `date`로 그대로 전달되어 실제422의위치가`body/options/to`입니다. 정상UTC자정은 실제계산1.0, 해당날짜는Spring부분0.1로 저장됩니다. 브라우저는 선택한 날짜 또는차트범위를 보내므로 화면 영향은 후속 실제결합에서 확인합니다.
- **REPORT-F1-WINDOW-001**: 두25자 시간대 표기를`from ~ to`로 합치면53자입니다. 실제 [migration](../backend/src/main/resources/db/migration/V44__create_analysis_report_table.sql)의VARCHAR(50)에삽입하다SQL1406이 발생합니다. 계산/캐시는 정상·차감−1/report0/REPORT_SAVE_FAILED입니다. 저장 장애 흡수는 동작하지만API가 수락한 날짜의리포트 저장은 실패입니다.
- **기존 REPORT-MAPPING-001**: 실제pipeline에서도stock_code=기능일합성기업,company_name=QAF1SQL로 뒤바뀝니다. [앞선 과금/저장 검사](isolated-analysis-billing.md)의동일 결함으로 분류합니다.

## 실행 증거·수학 대조·정리

최종 산출물은`tests/.runtime/backend-isolated/runs/run-y4nj39wc`입니다. [오프라인 감사](audit_feature1_pipeline.py)는summary/XML·fixture/wire 해시를 확인하고 다음을 직접 대조했습니다. 감사PASS는 아래 증거 일치이며 제품23사례 전체PASS가 아닙니다.

- 공개응답31개, 실제snapshot=HTTPdata=소유자상세26개, 타 사용자404×26, 실제Redis metrics28개. meta경고와저장경고의불일치1개는 위실패로 유지합니다.
- 실제FastAPI28회:200×24(잘못된JSON1회 포함),422×2,503×2. 실제Python정상metrics23개를 camel/snake 변환 후public과 비교했습니다. Java DTO가추가하는`indicators.spec:null`만 명시적으로 보정합니다.
- SQL에seed한 합성가격4320행분/재무162행분을 실제wire와 대조했습니다. 이는28개 호출의반복 합계이며 고유 원천행은160/6입니다. 기본사례의480 EMA값/초기null·마지막BB/stochastic·TTM 비율은제품 import 없는 수학 식으로 대조했습니다. 비선형 가격·모든marker/초기stochastic 정의·실제거래일의검증은 아닙니다.
- USE31행/REFUND3행. 세전체실패는각동일reference의−1/+1을 확인했습니다. 부분분석28회중저장26개·저장실패2개이며차감은각1회 유지됩니다. 잔액0/비인증402·401에서는원천 구독/계산/원장0입니다.
- 동일service 후속정상8요청에서schema1.0/160가격·독립 지표/snapshot 기대값·저장 복구를 확인했습니다. 복구요청8개를23사례 외에추가 합산하지 않습니다.
- 최종609개입력 해시불변. tests밖가시코드/설정/문서924개도변경·추가·소실0입니다. 업무9테이블/원천4테이블/QA종목/숨긴테이블/trigger0, Redis DBSIZE0입니다. 소유MySQL/Redis는exit0·정상종료, FastAPI는SIGTERM−15·포트종료/active0·외부연결/dotenv/subprocess시도0입니다.
- 초기3회도소유데이터 정리/서버종료를summary로 확인합니다. 제품 설정을읽되기존 사용자DB/Redis에접속하지 않았습니다. 모델API키는주입하지 않았고로컬감성모델도이단계에서로드하지 않았습니다.

## 초기 실행 이력

- `run-3bfz0wtu`: 정상 기준1개 FAIL. 제품 호출0. Java HTTP 제어 요청의 h2c 업그레이드에서 본문이 비어 ASGI 제어가500을 반환했습니다. 테스트 제어/감사 요청을HTTP/1.1로 고정했습니다. 제품 결함으로 집계하지 않습니다. 소유 MySQL/Redis 종료 및 FastAPI 종료 기록은 해당 summary에 보존됩니다.

- `run-2g079oq_`: 기준1개 FAIL. 실제FastAPI1회422와 SQL1406/analysis_window 초과를 확인했습니다. 종료일의23:59:59를 내부Python date로 전달한 문제와25자+25자+구분자3자의 기간 문자열을VARCHAR(50)에 저장한 문제입니다. 제품 결함 두 경로를 별도 사례로 추가하고 정상 계산 기준은 자정UTC/43자 기간으로 분리합니다. Redis 조회 키의 대소문자 정규화도 테스트에서 보정했습니다. 실제 숫자 snapshot은 독립 TTM 기대값과 일치했습니다. 초기 실행은 중복 합산하지 않습니다.

- `run-10fps7yv`: 실제Python schema1.0/160가격/480EMA·마지막BB/stochastic·TTM 기대값을 산출물에서 독립 대조했습니다. JUnit1개는HTTP 기본 codec이 날짜를숫자로,제품 저장mapper가ISO로 직렬화해 첫snapshot 비교에서 FAIL했습니다. 제품과 같은RedisConfig mapper와CodecsAutoConfiguration을 테스트Spring에 등록합니다. 실제 report1개/차감1개는 남고 기존 종목/회사명 역전도 관찰됐습니다. 초기 실행을 누적하지 않으며 후속 전체 실행의 기준 사례로 다시 대조합니다.

## 관련 문서

[상위 인덱스](README.md) · [누적 기록](QA_PROGRESS_2026-09-28.md) · [fallback 인계 기준](FALLBACK_QA_HANDOFF.md)

## 증거와 문서 검사 재현

```bash
python3 -B tests/audit_feature1_pipeline.py tests/.runtime/backend-isolated/runs/run-y4nj39wc
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

최종 증거 감사PASS, 문서69개 대상 링크/설정 비밀값 검사2개 PASS입니다. `feature1-pipeline-evidence-audit.json`과 `history-and-documentation-audit.json`에23사례 대응·26묶음 합계·네 실행 정리를 기록했습니다. 문서 로그는 `tests/.runtime/backend-isolated/feature1-pipeline-documentation.log`, tests 밖924개 보존 감사는 `tests/.runtime/backend-isolated/feature1-pipeline-source-audit.json`입니다. 앞으로 테스트 코드를 변경하면 과거 실행의 입력 해시와 달라지므로 과거 auditor의 현재 파일 비교는 그 차이를 고려해야 합니다.

## 다음 검증

Feature1 실제 서버 결과를React 검색/분석·카드/차트·저장 상세/PDF에연결합니다. 특히위경고/빈snapshot/날짜 경계의화면 보존,보조재무조회 실패의부분결과,재분석 복구를확인합니다. 실제 외부제공자·해외시장/환율·전체기간·Redis 단절/timeout·취소/재시도·다중프로세스·나머지Swagger123개 사용자 흐름과전체 QA범위는계속남습니다.
