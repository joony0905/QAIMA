# Feature2 실제 시장 원천 SQL·Redis·FastAPI 설명 QA

## 목적과 범위

[사용자 fallback 지시](FALLBACK_QA_HANDOFF.md)에 따라 실제 Feature2 조립기에 시장 원천 JPA/MySQL, Redis, FastAPI 설명 경로를 연결합니다. 이전 [Redis/취소 단계](isolated-feature2-resilience.md)의 source repository mock 경계를 확장하는 검사입니다.

수정은 `tests` 내부입니다. 기존 DB/Redis와 실제 외부 제공자의 계정·키를 사용하지 않습니다. runner가 새 `/tmp/qaima-qa-http-*` MySQL과 `/tmp/qaima-qa-redis-*` Redis, loopback FastAPI를 시작하고 소유 표식·포트·datadir을 확인합니다. FastAPI는 실제 애플리케이션이며 외부 socket connect, subprocess, `.env` 읽기를 audit hook으로 차단합니다.

실제 경로: HTTP/JWT → 실제 크레딧 SQL → 종목/가격·기준금리·거시 카드·공매도·수급·산업지수·뉴스 JPA/SQL → Feature2 조립/축약 → 실제 Spring FastApiAnalysisClient → 실제 Python `/feature2/analysis` → 공개 응답·report SQL·소유자 상세 HTTP.

제한: BOK/FRED/KIS, 뉴스 검색/본문, 뉴스 감성 모델은 명시적인 합성 제공자 경계입니다. Peer 계산과 뉴스 observation 비동기 저장도 별도입니다. 설명 서비스의 무키 fallback을 확인하며 유료 LLM의 품질·실제 외부 응답은 검증하지 않습니다. 이 단계는 브라우저 검사가 아닙니다.

## 실행 방법

```bash
python3 -B tests/run_isolated_backend.py --suite feature2-source-sql
```

구현: [Java](java/com/qaima/qa/IsolatedFeature2SourceSqlTest.java), [runner](run_isolated_backend.py), [ASGI 관찰기](pipeline_asgi.py). ASGI 관찰기는 요청/응답 본문을 그대로 전달하며 합성 입력과 응답만 해당 run의 `fastapi-exchanges/`에 남깁니다.

## 최종 결과

**24개13 PASS/11 FAIL**, 최종 `run-udwy3eb5`입니다. JUnit144.830초, Gradle4분34초이며 신규 격리 서버 준비 시간은 별도입니다. 실패는 제품 기대값 단언11개이고 JUnit errors/skipped는0입니다. 핵심 실패 관측 도우미 보정 뒤에도 동일한13/11을 확인했습니다.

최종 응답·SQL snapshot·소유자 상세46개, 고유 report46개, USE 원장46개(REFUND0)를 대조했습니다. 두 요청(기준금리 SQL 장애·가격+기준금리 동시 장애)은 설명 단계까지 진행하지 않아 실제 FastAPI 요청은44개입니다.44개 모두 Spring 축약 metrics와 wire의 snake_case 변환 후 값이 일치하며 실제 Python은200/`explain=null`/`LLM_API_KEY_MISSING`을 반환했습니다. Gemini/OpenAI 두 무키 경로를 포함합니다.

실제 SQL1146/42S02 장애15회, source 입력606개 실행 중/최종 해시 일치, tests 밖 코드·설정·문서924개 기준 해시 일치를 확인했습니다. `tests/audit_feature2_source_sql.py`의 PASS는 증거 정합성/격리 종료 감사이며 제품24개 모두 통과라는 뜻이 아닙니다. 근거는 최종 run의 `summary.json`, JUnit XML, `fastapi-exchanges/`, `fastapi-audit.json`, `feature2-source-sql-evidence-audit.json`입니다.

MySQL·Redis exit0/포트 종료, Redis DBSIZE0, source SQL12종·업무9종·QA 참조/숨긴 테이블/트리거0, proxy/control/worker 종료/activeSockets0을 확인했습니다. FastAPI는 소유 프로세스에 SIGTERM으로 종료했고 exit−15/포트 종료, 외부 접속·subprocess·dotenv 시도0입니다. 네 초기/최종 실행의 정리도 대조했습니다.

### Redis 단절과3기사의 누적 시간

최종 전체 HTTP84.885초: STOCK4.155초, 실제 수급 카드의 추가 가격 조회4.178초, INDEX4.193초, PEER4.200초, NEWS68.067초입니다. 실제 설명 HTTP는약9ms였습니다. 복원 후 같은 분석 Redis client의 연결 확인은최대8.975초, 캐시 수동 삭제 없이 후속 지표·뉴스/점수·설명 경고·SQL을 확인했습니다. 이는 대체 경로가 끝난다는 증거이며 사용자 응답 속도 기준을 충족했다는 판정은 아닙니다.

### 주요 실패 묶음

- `F2-ASSEMBLY-ISOLATION-001`: 기준금리 실제 SQL 오류가 정상 후속 단계까지 건너뜁니다.
- `F2-CARD-MACRO-ISOLATION-001`: 환율/채권 오류로 정상 거시 요약 소실. `F2-CARD-FLOW-ISOLATION-001`: 한쪽 수급 오류로 양쪽 수급 소실. 복수 장애 사례에서도 확인했습니다.
- `F2-SOURCE-WARNING-001`: 가격 SQL 오류·환율 빈 행과 위 카드 오류의 누락 안내가 없습니다. 정상 뉴스 필터 안내와 무키 LLM 경고는 해당 데이터 오류 안내로 인정하지 않습니다.
- `F2-FLOW-NULL-ZERO-001`: 수급3행의 순매수 값 전부NULL→합계0/`pointCount=3`/`MIXED_OR_FLAT`. 두 시장 범위 모두 공개 응답·저장·실제 설명 입력에 전달되며 경고가 없습니다.
- 기존 `F2-CORE-REFUND-001`: 종목 식별 외 모든 정량null인데 잔액5→4, 원장 `[-1]`, report1이 남습니다. 기대 `[-1,+1]`/잔액5/report0과 다릅니다.

실패11사례는 위 묶음의 고유 결함 수와 같지 않습니다.

## 입력·장애·판정 기준

- 종목 `QAF2SQL`, 별도 sector/industry/index 참조 행. 가격 30일은100→129, 산업지수 30일은200→258이며 기대 상대수익률은29%입니다.
- KR 기준금리3.25%, US4.25%, USD/KRW1350, KR채권2종3.5%, US채권3종4.5%. 일별3행 및 US 월별3행을 SQL에 넣고 `updated_at`을 실제 실행일로 설정합니다. 가격/기준일은 합성 고정값으로 실제 시장의 최신 수치가 아닙니다.
- 종목 수급3일 합계900백만원, 시장 수급3일 합계2100백만원. 공매도 거래량비율7.5%, 거래대금비율8.5%입니다.
- 실제 news/map SQL에 기사3건을 넣고 합성 감성 모델은 각0.25를 반환합니다. scores의 실제 저장·조회도 연결합니다. 기사 목록의 검색/필터 안내는 원천 SQL 오류 경고와 구분합니다.
- SQL 장애는 소유 테이블을 `qa_f2_hidden_*`으로 RENAME하여 실제 repository SQL이1146/42S02를 받게 합니다. `finally`와 afterEach에서 원래 이름을 복원합니다. 실제 사용자 DB에는 접근하지 않습니다.
- 각 완료 분석에서 HTTP200만 보지 않고 지표 수치, 경고, 크레딧−1/원장1행, report1행, 공개data=SQLsnapshot=소유자 상세를 확인합니다. 정량0의 차감 잔존은 별도 실패 단언으로 잡습니다.
- 장애 제거 뒤 같은 Redis client/서비스를 이용하며 SQL 오류 사례의 복구 전에 캐시를 지우지 않습니다. 재시도 후에도 이전 경고/빈 값이 남는지를 관찰합니다.
- Redis 단절은 소유 TCP proxy의 실제 연결 거절입니다. Redis command timeout2초와 News 내부 block2초를 유지합니다. HTTP170초/JUnit200초는 관측용 한도이며 제품 SLA가 아닙니다.

## 초기 실행 이력

`run-wah02168`: 정상 기준1개 PASS. 실제 FastAPI 두 요청이200/`explain=null`/`LLM_API_KEY_MISSING`을 반환했고 외부 접속·subprocess·dotenv 시도는 모두0입니다. 최초 HTTP7,510ms·두 번째173ms였으며 이는 초기 정상 입력에서의 단일 관측입니다. 소유 MySQL/Redis/FastAPI 및 source SQL 행 정리 확인. 이후 US채권 합성 날짜를 월별로 정리하고 Peer의 실제 Redis/service를 연결해 전체 사례를 확장했습니다. 선택 기준 실행은 누적에 중복 합산하지 않습니다.

`run-o1ylvybl`: 확장20개13 PASS/7 FAIL. 기준금리 SQL의 후속 단계 중단, 환율/채권 SQL의 정상 거시 요약 소실, 종목/시장 수급 SQL의 독립 수급 소실, 정량0의 차감 잔존을 재현했습니다. Redis 단절·3기사에서84,893ms를 관측했고 NEWS 단계는68,064.578ms였습니다. 기존 정상 뉴스 필터 안내가 일반 경고 단언을 만족시키는 테스트 허점을 발견해, 최종 단언은 `LLM_*` 및 `NEWS_FILTER_*`를 데이터 오류 경고로 계산하지 않습니다. 이 때문에 이 실행의 가격 SQL 경고 PASS는 최종 판정으로 사용하지 않습니다. 모델 복구 요청과 감성 SQL 쓰기 실패·환율 빈 행·수급 NULL 사례를 추가했습니다. 초기20개는 최신 누적과 별도로 합산하지 않습니다.

## 확인한 실패 경로

- **거시 요약의 장애 격리**: 환율 또는 채권 저장소 오류가 거시 카드의 최상위 오류 처리로 전파되어 정상 KR/US 금리·다른 거시 값까지 빈 요약으로 바뀝니다. 거시 시계열의 부분 보존과 구분합니다. 오류에 해당하는 공개 경고도 별도로 확인합니다.
- **수급의 장애 격리**: 종목 수급 쿼리와 시장 수급 쿼리가 같은 `buildInvestorFlow` 안에서 실행되어 한 쿼리 오류가 전체 수급 카드null로 처리됩니다. 다른 쿼리의 정상 합계를 보존하지 않습니다.
- **기준금리의 조립 중단**: 주 조회와 같은 저장소를 이용하는 fallback이 모두 실패하면서 후속 공매도·수급·산업지수·뉴스·설명까지 생략됩니다. 앞선 `F2-ASSEMBLY-ISOLATION-001`의 실제 SQL 재현입니다.
- **핵심 실패 과금**: 가격과 기준금리 SQL 동시 오류에서는 종목 식별 정보만 남고 정량값이 전부null인데 차감과 성공 report가 남습니다. 기존 `F2-CORE-REFUND-001`의 실제 원천 결합입니다.
- **수급 NULL의0 변환**: 종목/시장 수급 행이 존재하지만 순매수 값이 모두 SQL NULL인 경우를 따로 검사합니다. `pointCount`가 존재하는 것만으로 측정 합계0을 정당화하지 않습니다. 공개 응답·FastAPI 설명 입력·report에 같은 변환이 남는지 확인합니다.

원인 소스: [거시/수급 카드](../backend/src/main/java/com/qaima/service/feature2/Feature2CardService.java), [조립기](../backend/src/main/java/com/qaima/service/feature2/Feature2AnalyzeService.java), [가격 fallback](../backend/src/main/java/com/qaima/service/feature2/resolver/Feature2StockResolver.java). 제품 코드는 수정하지 않습니다.

`run-2teiuios`:24개13 PASS/11 FAIL, JUnit144.702초.46개 응답/SQL snapshot/상세와44개 FastAPI 요청을 대조했습니다. 감사 도구의 비교는 내부 wire의 snake_case 변환만 적용하고 값·null·배열은 그대로 비교합니다. 최종 검토에서 핵심 실패에도 공통 관측 도우미가 성공 저장을 요구하던 테스트 전제를 제거했습니다. 올바른 환불·report 미생성이 구현되면 해당 사례가 통과할 수 있도록 보정하고 차감/환불 원장 `[-1,+1]` 단언을 추가했습니다. 현재 제품에서 관측한 차감 잔존·report 저장 사실은 바뀌지 않습니다. 후속 확정 실행만 누적에 반영합니다.

## 검증 경계와 남은 작업

- 실제 SQL rows/repository/service, Redis socket/codec, FastAPI 무키 설명 경로를 연결한 선택 범위입니다. 실제 BOK/FRED/KIS·Naver/본문·유료 LLM·Peer 계산/뉴스 감성 모델 정확도와 observation 비동기 저장은 남아 있습니다.
- 기사3건의 누적 timeout 측정입니다. 정책상 최대30건, 실제 원천 지연/분량, 동시에 여러 요청인 경우를 검증한 것은 아닙니다. 이전1기사 검사와 이번 검사는 실제 카드/SQL 연결도 다르므로 기사수만으로 전체 지연을 비교하지 않습니다.
- 감성 SQL INSERT 오류 뒤에도 cache의 계산 점수는 유효합니다. SQL 복구 직후 캐시 hit에서 sentiment rows가0으로 남는 동작은 관측이며, 캐시 만료·프로세스 중단 뒤 재산출/저장 복구는 별도입니다.
- 산업지수10행+provider 실패는 기존 행 fallback을 확인했습니다. 원천 복구 후 이미 저장된 부분 캐시의 갱신·경고 수명은 별도입니다.
- 새 NULL→0/지표 소실/경고 누락이 실제 브라우저·저장 상세/PDF에서 어떻게 보이는지, 소스 전 범위/여러 backend/프로세스 중단/재시도는 이어서 검증합니다. 이 단계만으로 전체 Feature2와 전체 QA 완료를 선언하지 않습니다.

## 사례별 확정 결과

| JUnit 사례 | 결과 | 검증 내용 |
|---|---|---|
| `actualOpenAiMissingKeyAlsoPreservesRealSqlResults()` | PASS | 실제 OpenAI 무키 fallback; 지표/경고/SQL 보존 |
| `allNewsModelErrorsKeepThreeArticlesWithNullScores()` | PASS | 모델 전부 실패; 기사3개/점수null 유지 후 같은 서비스 복구 |
| `baselineActualMarketSqlRedisAndFastApiMissingKey()` | PASS | 실제 SQL·Redis cold/warm·Gemini 무키·소유권 |
| `combinedFxAndMarketFlowSqlErrorsPreserveOtherData()` | FAIL | 환율+시장 수급 오류; 정상 거시/종목 수급 소실·경고 누락 |
| `industryProviderFailureUsesTenExistingSqlRows()` | PASS | SQL10행+provider 실패;10행/7.5% 기존 자료 fallback |
| `missingFxRowsKeepOtherMacroDataAndWarn()` | FAIL | 환율 SQL 빈 행+provider 실패; 타 지표 유지/누락 경고 없음 |
| `missingOneNewsScoreKeepsAllThreeArticlesAndOtherScores()` | PASS | 점수1개 누락; 기사3개/점수2개 보존·복구 |
| `nullable source flow stock_investor_flow` | FAIL | 종목 수급 NULL→0·보합 오인/경고 누락·복구 |
| `nullable source flow market_investor_flow` | FAIL | 시장 수급 NULL→0·보합 오인/경고 누락·복구 |
| `providerErrorsUseExistingRealSqlRowsAndKeepExplanation()` | PASS | 17개 실제 provider 오류 구독→기존 SQL fallback; 갱신 후 추가 호출 중지 |
| `redisPartitionWithThreeRealSqlArticlesRecoversWithoutEviction()` | PASS | 실제 TCP 단절/3기사/84.885초·같은 client 복구 |
| `sentimentInsertSqlFailureKeepsComputedScoresAndRecovers()` | PASS | 실제 INSERT trigger 오류;3점수 유지/SQL0/복구 캐시 재사용 |
| `sourceSqlCoreFailureDoesNotRetainDebit()` | FAIL | 가격+금리 SQL 오류; 정량0에서 환불·저장 정책 실패 |
| `actual source SQL failure price_ohlcv` | FAIL | 나머지 지표/복구 유지, 가격 누락 경고 없음 |
| `actual source SQL failure base_rate` | FAIL | 금리 오류가 독립 후속 지표/뉴스/설명까지 중단 |
| `actual source SQL failure short_selling` | PASS | 공매도 경고+타 지표 유지·SQL 복원 후 회복 |
| `actual source SQL failure exchange_rate` | FAIL | 정상 금리/채권 요약 소실·경고 누락·복구 |
| `actual source SQL failure bond_yield` | FAIL | 정상 금리/환율 요약 소실·경고 누락·복구 |
| `actual source SQL failure stock_investor_flow` | FAIL | 정상 시장 수급까지 소실·경고 누락·복구 |
| `actual source SQL failure market_investor_flow` | FAIL | 정상 종목 수급까지 소실·경고 누락·복구 |
| `actual source SQL failure industry_index_ohlcv` | PASS | 산업지수 경고+타 지표 유지·복구 |
| `actual source SQL failure industry_index_map` | PASS | 매핑 조회 실패 경고+타 지표 유지·복구 |
| `actual source SQL failure news_security_map` | PASS | 뉴스 목록 경고+타 지표 유지·복구 |
| `actual source SQL failure sentiment_result` | PASS | 기사3개 유지/점수null·경고·복구 |

문서66개 대상 링크/설정 일부 비밀값 검사2개 PASS,24개 사례 표/누적23묶음 집계와4실행 정리를 확인했습니다. 로그는 `tests/.runtime/backend-isolated/feature2-source-sql-documentation.log`, 상세 감사는 최종 run의 `feature2-source-sql-history-and-doc-audit.json`입니다. [상위 누적](QA_PROGRESS_2026-09-28.md)과 [인계 기준](FALLBACK_QA_HANDOFF.md)에 함께 반영합니다.
