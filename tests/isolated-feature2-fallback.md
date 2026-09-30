# Feature2 실제 조립기·fallback·HTTP/과금/리포트 SQL 검증

## 목적과 현재 상태

2026-09-28 사용자의 재개 지시에 따라 [fallback 인계 기준](FALLBACK_QA_HANDOFF.md)을 이어갑니다. 기존 billing에서 mock이었던 Feature2AnalyzeService를 실제 객체로 연결했습니다. **최종 `run-bxtp_64j`: 39개17 PASS/22 FAIL, errors/skipped0, JUnit18.252초**입니다. 22개는 실패 사례 수이며 고유 결함 수가 아닙니다. 전체 QA는 아직 미완료입니다.

수정은 tests 내부에 한정합니다. 기존 업무 DB/Redis는 사용하지 않습니다. 새 `/tmp/qaima-qa-http-*` MySQL만 사용하며 포트·datadir·schema·소유 표식을 확인한 뒤 접근합니다. 제품 결함은 재현하고 기록하며 제품 코드는 수정하지 않습니다.

## 실제 경계와 대체 경계

- 실제: HTTP/JWT/보안/요청 검증 → Feature2AnalyzeController → Feature2RequestNormalizer/Feature2AnalyzeService/두 assembler/response factory → CreditService·ReportService·JPA·MySQL → 소유자 리포트 상세 HTTP.
- 합성: 기본 행렬에서는 종목/산업 resolver, 금리·카드·공매도·산업지수·Peer·뉴스 서비스, 추세 조회 Repository, 설명 API 경계. 모든 입력은 합성 데이터이며 외부 제공자·LLM·Redis·브라우저는 호출하지 않습니다.
- 추가4개는 실제 BaseRateFeatureService의 원천+대체 조회 오류, 실제 ShortSellingFeatureService의 Repository 오류, 실제 Feature2IndustryReader의 분류 없음, 실제 Feature2StockResolver의 Repository 오류를 조립기에 연결합니다. 이4개의 하위 Repository/동기화 경계는 여전히 mock이며 실제 DB 장애라고 주장하지 않습니다.
- `Mono.error`는 **하위 서비스 밖으로 빠져나온 오류**입니다. 하위 서비스가 보통 흡수하는 원천 오류까지 전부 같은 방식으로 전파된다고 해석하지 않습니다. 실제 원천 오류→하위 fallback→조립 연결은 후속 범위입니다.
- `Mono.empty`는 warning 없이 종료하는 경계 내구성 검사입니다. 산업 분류 없음은 실제 reader와 같은 `Optional.empty`+`INDUSTRY_MISSING` 형태를 사용합니다.
- 뉴스 timeout은 `Mono.never().timeout(120ms)`로 실제 Reactor 타이머를 사용합니다. 실제 원천/Redis timeout 누적이나 응답시간 SLA 검사가 아닙니다.
- 리포트 저장 실패만 소유 MySQL의 `BEFORE INSERT` trigger로 실제 SQL 오류를 주입합니다.

## 기대값과 검증 방법

정책 근거: [공개 정책 §4.3/§5.4](../policy/QAIMA_POLICY_PUBLIC.md), [분석 사용자 흐름](../policy/Feature_analysis_user_flow_policy.md), [API/저장 계약](../policy/API_DTO_CONTRACT_POLICY.md). 한 구성요소가 실패해도 독립 데이터는 유지하며, 누락은 warning으로 안내하고, 핵심 분석 실패는 환불해야 합니다. 설명/저장만 실패한 경우 정상 정량 결과와 과금은 유지합니다.

1. 정상 합성 기대값은 기준금리3.25%, 거시 KR3.5/US4.25/USDKRW1350, 금리 시계열3.75→3.5, 공매도7.5%, 수급330, 산업 상대지수0→12, 뉴스 감성(.25+.75)/2=.5입니다. 추세 요약은 역순 Repository 입력을 정렬해 금리 변화−.5, 공매도 평균6.25/변화2.5를 독립 대조합니다.
2. `Mono.defer`에서 실제 구독/값/오류/취소와 단조시계 경과를 기록합니다. Java 메서드 호출만으로 후속 단계 실행을 판정하지 않습니다.
3. 정상/실패 응답의 독립 필드는 public data, warning은 public meta에서 검사합니다. Feature2 DTO의 내부 warnings는 `@JsonIgnore`이며 공개 응답의 `meta.warnings`로 전달됩니다. HTTP200만으로 통과시키지 않습니다.
4. 실제 SQL 잔액·원장·리포트 수와 HTTP data=저장 snapshot=소유자 상세, warning 보존·다른 사용자404를 대조합니다.
5. 각 장애 뒤 같은 service/fixture에서 장애를 해제하고 다른 종목 `QA_F2_B`와 다른 기준금리4.25를 요청합니다. 정상 복구/추가1차감·경고 잔류 없음·이전 snapshot 불변을 확인합니다. 1차 단언 실패 전에 복구/SQL 증거부터 수집해 장애의 후속 결과를 남깁니다.
6. 실행기는 명령·JUnit XML·관측 JSON·소스 SHA-256·종료/정리 증거를 각 실행 폴더에 보존합니다. 정상 대조/초기 fixture 오류/재실행은 최종 사례 수에 중복 합산하지 않습니다.

## 실행

총39사례는 일반 메서드17개와 파라미터 메서드3개의22입력입니다. 아래는 최종 실행 결과입니다.

| 메서드 | 수 | 방법·독립 기대값 | 결과 |
|---|---:|---|---|
| baselineKeepsEveryIndependentValueAndSubscriptionOrder | 1 | 전체 값·실제 구독 순서·다른 종목 복구 대조 | PASS |
| singleServiceBoundaryErrorPreservesIndependentResults | 11 | BASE/MACRO/SERIES/SHORT/FLOW/INDEX/PEER/BASE_TREND/SHORT_TREND/NEWS/EXPLAIN 각각 오류, 나머지 독립 값 보존 | 6P/5F; BASE/MACRO/SERIES/SHORT/INDEX 실패 |
| emptyCompletionKeepsOtherValuesAndSignalsMissingComponent | 7 | MACRO/SERIES/FLOW/INDEX/PEER/NEWS/EXPLAIN empty, 정상 상대값·누락 warning | 0P/7F |
| industryMissingStillLoadsIndependentTrendsNewsAndExplain | 1 | 분류 없음+warning에도 산업과 무관한 추세/뉴스/설명 유지 | FAIL |
| industryBoundaryErrorStillLoadsIndependentTrendsNewsAndExplain | 1 | 분류 서비스 오류의 연쇄 중단 차단 | FAIL |
| stockResolutionErrorRefundsCoreFailureAndDoesNotSaveEmptySuccess | 1 | 종목 해결 오류·정량0에서−1/+1·성공 report0 기대 | FAIL |
| combinedErrorsPreserveEveryUnaffectedComponent | 4 | MACRO+PEER, NEWS+EXPLAIN, BASE+SHORT, FLOW+두 추세 오류 | 2P/2F; MACRO+PEER와 BASE+SHORT 실패 |
| allQuantitativeSourcesAndExplanationFailedMustRefund | 1 | 전체 정량 원천/설명 오류에서 환불·성공 리포트 없음 기대 | FAIL |
| realBaseRatePrimaryAndFallbackFailuresKeepIndependentMetrics | 1 | 실제 금리 서비스의 최초/대체 조회 모두 오류 후 독립 값 유지 | FAIL |
| realShortSellingRepositoryErrorBecomesWarningAndKeepsOtherMetrics | 1 | 실제 공매도 서비스의 조회 오류→빈 값+warning, 나머지 유지 | PASS |
| realIndustryReaderMissingClassificationKeepsIndependentNews | 1 | 실제 분류 reader의 산업 없는 stock→조립 후 독립 값 유지 | FAIL |
| realStockResolverRepositoryErrorRefundsUnavailableAnalysis | 1 | 실제 stock resolver가 조회 오류를 Optional.empty로 흡수한 뒤 과금/저장 대조 | FAIL |
| partialNewsRetainsArticlesAndComputesOnlyAvailableScores | 1 | 기사2개 유지/점수1개 null, 평균.25·scored1·설명 입력 기사2 | PASS |
| partialMacroAndDeterministicExplainCarryWarningsWithoutLosingMetrics | 1 | US금리 누락·설명 fallback warning, KR금리/환율/나머지 값 유지 | PASS |
| nullExplainObjectRetainsQuantitativeResultsAndWarns | 1 | explain object null의 안내와 정량 값 유지 | FAIL |
| actualReactorTimeoutInNewsContinuesToExplanationAndRecovers | 1 | 실제120ms Reactor timeout 후 뉴스 warning·설명 진행·복구 | PASS |
| reportInsertFailureKeepsResultsChargeAndPublicSaveWarning | 1 | 실제 SQL 저장 거절→정량 유지/−1/report0/REPORT_SAVE_FAILED | PASS |
| anonymousRequestCannotSubscribeOrCharge | 1 | JWT 없음401/구독0/원장·report0 | PASS |
| invalidRequestCannotSubscribeOrCharge | 1 | 공백 종목400/구독0/원장·report0 | PASS |
| insufficientCreditCannotSubscribeOrSave | 1 | 잔액0에서402/구독0/원장·report0 | PASS |

```bash
python3 -B tests/run_isolated_backend.py --suite feature2-fallback
```

정상 구성만 선택:

```bash
python3 -B tests/run_isolated_backend.py --suite feature2-fallback --test baselineKeepsEveryIndependentValueAndSubscriptionOrder
```

코드는 [테스트 클래스](java/com/qaima/qa/IsolatedFeature2FallbackTest.java), [격리 실행기](run_isolated_backend.py), [테스트 전용 Gradle 설정](backend-isolated.init.gradle)입니다. 결과는 `tests/.runtime/backend-isolated/runs/`에 남습니다.

## 실패와 영향

### F2-ASSEMBLY-ISOLATION-001 — 독립 구성요소의 연쇄 누락

같은 조립 불변식 아래 세 원인 경계를 기록합니다. 다음13개 실패를 고유 결함13건으로 세지 않습니다.

- **조립 중단5사례**: BASE/SHORT/INDEX의 escaped error, BASE+SHORT, 실제 금리 최초+대체 조회 오류. 실제 BaseRateFeatureService가 `ensureDailySynced` 실패 후 `findLatest`를 시도하지만 이 조회도 실패하면 조립기 최상위 handler로 이동합니다. 응답에는 종목만 남고 macro/공매도/수급/산업/뉴스/설명 구독이 진행되지 않습니다. 실제 공매도 서비스의 일반 Repository 오류는 자체 warning+empty로 처리되어 나머지 결과가 보존됐습니다. SHORT/INDEX의 합성 escaped error를 일반 원천 오류로 확대하지 않습니다.
- **산업 분류 의존성3사례**: 합성 Optional.empty/escaped error와 실제 Feature2IndustryReader의 분류 없음. 분류에 직접 의존하는 산업지수·Peer 외에 독립적인 두 추세·뉴스·설명도 실행되지 않습니다. 기존 금리·거시·공매도·수급은 보존됩니다. `industryContextOpt.isEmpty()`의 early return과 구독 기록이 일치합니다.
- **거시 zip 결합5사례**: 현재값/시계열 중 한쪽의 error 또는 empty, MACRO+PEER 오류. 독립적으로 정상인 상대쪽도 attach되지 않습니다. error는 MACRO_RATES_LOAD_FAILED가 있으나 empty는 warning도 없습니다. 이는 [기존 F2-MACRO-001](../docs/findings.md#f2-macro-001--한-거시-지표의-이중-실패가-다른-정상-지표까지-제거)의 카드 내부 원천 결합 문제와 관련되지만, 이번에는 카드 출력과 시계열을 묶는 **상위 AnalyzeService의 별도 zip 경계**를 검사했습니다. 실제 카드 서비스의 모든 원천 장애를 연결한 결과는 아닙니다.

BASE+SHORT/전체 정량 장애를 설정한 사례에서는 BASE 오류가 먼저 조립을 끝내므로 SHORT 등 후속 장애는 실제 구독되지 않았습니다. 여러 장애를 설정했다는 이유로 모든 fallback이 실행됐다고 표시하지 않습니다. NEWS+EXPLAIN, FLOW+두 추세는 각 오류 구독·warning·정상 나머지 보존을 확인했습니다.

### F2-EMPTY-WARNING-001 — 빈 완료/설명 누락 안내 없음

7종 warning 없는 Mono.empty와 explain=null 총8사례에서 누락된 필드가 있지만 public meta.warnings가 빈 배열입니다. macro 두 사례는 위 정상 상대값 소실에도 겹칩니다. 이는 서비스 경계의 내구성 실패이며, 실제 하위 서비스가 정상적으로 붙인 경고까지 항상 없어지는 현상이라는 주장은 아닙니다. 실제 공매도 조회 오류와 partial news/macro·설명 warning의 public/SQL 보존은 통과했습니다.

### F2-CORE-REFUND-001 — 정량 결과가 없어도 차감·리포트 저장

종목 resolver의 escaped error, 실제 resolver의 Repository 오류→Optional.empty, 전체 정량 원천 오류 설정 총3사례입니다. 모두 HTTP200/meta.status=success, 잔액5→4, 원장−1 한 행, report1행입니다. stock-error 두 경로는 모든 metrics가null이고, 전체 정량 오류는 종목 메타만 남습니다. SQL snapshot에도 이 응답이 저장되고 소유자 상세로 조회됩니다.

정책 §5.4는 핵심 분석 실패 시 환불을 요구합니다. 실제 조립기/stock resolver가 오류를 정상 Mono로 흡수하여 Controller의 refund onErrorResume에 도달하지 않는 흐름을 확인했습니다. 기존 controller 설명의 “컨트롤러 레벨 전체 실패 시 환불”과 이 상위 정책 사이의 구현 차이를 기록합니다. 설명/리포트 저장만 실패했을 때 차감 유지 사례는 별도로 통과했습니다. 운영 DB 장애가 아닌 합성 Repository 오류이며, 실제 원장/저장은 소유 MySQL입니다.

## 실행 증거와 정리

- 최종: `tests/.runtime/backend-isolated/runs/run-bxtp_64j/summary.json`, 같은 폴더의 `gradle.log`, `TEST-com.qaima.qa.IsolatedFeature2FallbackTest.xml`, `feature2-evidence-audit.json`.
- 분석 요청36개+각 장애 해제 후 재요청36개, 거절3개로 관측75개입니다. 재요청36개 모두 다른 종목/금리·정상 설명·warning0·추가1차감이 확인됐습니다. 이 보강 단언을 별도36사례로 합산하지 않습니다.
- 저장 성공 응답71개에서 HTTP data=SQL snapshot=소유자 상세 및 warning 보존·타 사용자404를 확인했습니다. 저장 실패1개는 정상 분석값/차감 유지·report0입니다. 이후 요청이 앞선 snapshot을 바꾸지 않았습니다.
- 입력575개 실행 중 변경0, 실행 후 해시 대조도 변경0입니다. 업무9테이블/소유 trigger 최종0, MySQL8.0.46·51마이그레이션/48테이블, exit0/stopped true·소유 포트 닫힘을 확인했습니다. Redis/FastAPI는 시작하지 않았습니다.
- tests 밖 코드·설정·문서924개 변경/소실/새 파일0은 `tests/.runtime/backend-isolated/feature2-source-audit.json`에 기록했습니다.
- 문서63개 대상2개 검사 PASS는 `tests/.runtime/backend-isolated/feature2-documentation.log`에 있습니다. 문서 표20행=일반17+파라미터3메서드, 실제39사례와 누적20묶음457개 합산을 별도 대조했습니다.

## 초기 보정 이력과 남은 범위

초기 정상 대조 `run-lsfc11gr`는1개 FAIL이었으며 제품 결함이 아닙니다. HTTP data/SQL snapshot/소유자 상세/다른 종목 재요청은 일치했으나 테스트가 내부 warnings도 data에 직렬화된다고 가정했습니다. 실제 DTO의 `@JsonIgnore` 계약에 맞게 public meta와 별도 SQL warnings를 비교하도록 보정했습니다. 설명용 DTO의 `recent_news` 이름도 보정했습니다. 초기 실행의 업무9테이블/trigger0·MySQL exit0/stopped true·소스 변경0을 확인했습니다. 최종 집계에서 제외합니다.

실제 하위 서비스의 원천/DB/Redis fallback 연결, timeout 누적·취소, 실제 FastAPI 설명, 브라우저 카드/경고/저장 상세/PDF 연결은 아직 이 검사로 입증하지 않습니다. 전체 QA 범위와 남은 항목은 [누적 진행기록](QA_PROGRESS_2026-09-28.md)을 따릅니다.
