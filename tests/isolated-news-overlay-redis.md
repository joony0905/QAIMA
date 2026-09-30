# 뉴스·Feature3 지표 캐시의 격리 Redis QA — 2026-09-28

현재 `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03`와 기존 미커밋 변경을 포함한 소스를 검증합니다. 제품은 미수정이며 테스트 코드와 기록은 `tests` 내부에만 작성합니다. 전체 진행은 [최신 기록](QA_PROGRESS_2026-09-28.md), 이전6개 캐시 서비스의18개 검사는 [격리 Redis 기록](isolated-redis-cache.md)을 참조합니다.

## 어떻게 검증했는가

[새 Java 검사](java/com/qaima/qa/IsolatedNewsOverlayRedisTest.java)는 실제 `NewsSentimentService`, `Feature3OverlayService`, `RedisConfig` mapper/template, Lettuce를 새 Redis7.0.15에 연결합니다. 실제 GET/SET과 직렬화를 사용하고 별도 RESP socket으로 JSON·PTTL·ACL log·소유 정보를 확인합니다. 기존 `IsolatedRedisCacheTest`의 연결/소유 가드/정리 helper만 합성해 사용하며 이전18개 테스트를 상속·재실행하지 않습니다.

```bash
python3 -B tests/run_isolated_redis.py --suite news-overlay
python3 -B tests/run_isolated_redis.py --suite news-overlay --test overlappingRefreshMustTagEachScoreWithItsActualModelInputHash
```

[실행기](run_isolated_redis.py)에 `--suite core|news-overlay`를 추가했습니다. 기본값은 기존 core이고, 선택한 클래스의 새 JUnit XML만 수집합니다. 패키지 추출·Gradle8.14 offline 실행·빌드 출력 경로는 이전 Redis 문서와 같습니다. 매 실행마다 외부 `/tmp/qaima-qa-redis-*`의 새 디렉터리·loopback 포트·임의 인증을 사용합니다. 기존 Redis를 사용하지 않습니다. port/run_id/dir/소유 표식을 시작 전·사례 전후·최종 정리 전에 확인합니다.

대체 경계와 한계는 다음과 같습니다.

- Stock/News/NewsSecurityMap/SentimentResult Repository, Naver 검색·본문 extractor·감성 모델·observation 저장 경계를 mock합니다. 합성 회사·기사1개(id42), 모델점수0.25/0.75/0.9/0.1은 데이터 흐름 구별용이며 모델 정확도 주장이 아닙니다. 실제 SQL·모델·외부 API 호출은0입니다.
- Feature1의 지표 생성은 fixture입니다. Feature3의 실제 지표 캐시 작성/읽기·signal 조립·비용 preview/estimate를 실행합니다. `CreditService`, 차감/환불 원장, 최종 포트폴리오 분석은 이 검사에 포함하지 않습니다.
- 뉴스 GET2개는 실제 Controller/advice를 `WebTestClient.bindToController`에 연결한 in-memory HTTP 검사입니다. 실제 HTTP 포트·JWT/SecurityConfig 검사는 아닙니다.
- 뉴스 5min/15min/7day, 지표24h/7day는 실제 Redis PTTL을 확인합니다. 자연 만료까지 기다리지 않습니다. 계층 누락은 소유 서버의 테스트 키를 명시적으로 삭제한 fixture입니다.
- ACL은 실제 Redis `NOPERM`, timeout은 소유 Redis의 `CLIENT PAUSE 2600 ALL`을 사용합니다. 뉴스의 실제 `.block(2초)` 제한이 적용되고 이후 동일 template으로 연결 복구를 확인합니다. TCP 단절·Redis 재시작·failover를 검증한 것은 아닙니다. 테스트 모델 응답 제한은10초, 외부 block 제한12초입니다.

## 21개 사례와 독립 확인

| 메서드 | 입력·확인 방법 |
|---|---|
| `coldNewsAnalysisWritesAllFiveTtlsAndReusesExactInputHashAndScore` | 빈 캐시→검색/본문/모델 fixture→목록·refresh·detail·focus·sentiment5키 생성.5m/15m/7d PTTL, 모델 input SHA-256과 두 캐시/observation의 hash 일치, 점수 저장 인자, 정보성 FILTER_APPLIED 유지, 두 번째 조회의 제공자/Repository 호출0 |
| `refreshMarkerSkipsSourceAfterListEvictionAndPreservesFiveMinuteTtl` | 정상 분석 후 list만 삭제. refresh marker와 저장 기사 날짜가 같으면 검색 없이 DB fixture로 목록 재구성, 목록 TTL5m, 목록 조회는 감성 계산 안 함 |
| `forceRefreshRebuildsAllLayersAndReplacesStoredScore` | 완전 캐시 점수0.75→force.본문 문자열 변경·모델0.25, 검색/extractor/모델/저장 모두 호출, focus hash 갱신과 sentiment hash 일치,7d TTL |
| `publicListAndDetailDecodeActualCacheWithoutInvokingModel` | 실제 list/detail Controller GET.목록 감성null, 상세 본문·캐시점수0.75, 외부 검색/추출/모델 호출0 |
| `inspectionRejectsEveryMissingOrMalformedLayerWithoutProviderSideEffects` | list/detail/focus/sentiment를 하나씩 제거·깨진JSON 주입→miss, 원본복원→hit.검사 중 제공자·Repository·observation 호출0 |
| `incompatibleModelAndPromptVersionsTriggerReanalysisAndRepairCache` | 실제 sentiment JSON의 model/prompt 버전을 각각구버전으로 변경. inspection miss, 모델 재호출0.25, 다시 hit |
| `changedFocusShouldInvalidateInspectionAndRecomputeSentiment` | focus본문/hash만 변경, 이전점수0.75. 기대 miss·모델1회·새0.25. 기존 F2-NEWS-CACHE-001 회귀 |
| `nullCachedScoreShouldInvalidateInspectionAndTriggerModel` | 호환 버전이지만 sentimentScore만null. 기대 miss·모델1회·새0.25. 정상 쓰기가null을 저장한다는 주장은 아님 |
| `newsAclErrorsPreserveMetadataAndComputedSentimentWithWarnings` | GET/SET이 모두 실제 거절돼도 기사·계산점수 유지, READ/WRITE warning 각1개, 뉴스 키0 |
| `pausedOwnedRedisTriggersReadTimeoutFallbackAndConnectionRecovers` | 정상 캐시 후 새 소유서버를2600ms pause.첫 캐시 read의2초 제한→warning·저장기사 fallback.같은 template으로 표식과 list 조회 복구, 검색/모델0 |
| `modelFailureKeepsArticleWithoutCachingFalseSentiment` | 모델 fixture오류→기사 유지·점수null·기사별실패 warning, sentiment 키 없음·inspection miss·score save0 |
| `overlappingRefreshMustTagEachScoreWithItsActualModelInputHash` | A force의 모델 응답을 latch/Sinks로 보류.다른 본문의 B force를 완료해0.9/B hash 확인. A0.1을 해제한 뒤 최종 score/hash와 observation을 모델에 실제 전달한 A input hash와 비교 |
| `metricsWriteUsesNormalizedKeysAndActualFreshAndStaleTtls` | 공백/대문자 code→qa_a 키, fresh24h·stale7d PTTL과 동일JSON.실제 preview 두 overlay HIT/core1, 분석 경계 호출0 |
| `metricsCacheMissCostsPerOverlayAndPreviewDoesNotAnalyze` |2종목·fundamentals/technical 모두miss→core1+항목2=3.종목수당 과금 아님. preview와estimate 일치, 분석 호출0 |
| `allStockMetricHitsAreFreeAndOneMissingStockMakesOverlayChargeable` |2종목캐시hit→총1·가장 오래된 cachedAt의KST시각.한종목삭제→총2 |
| `forcePolicyAndPerOverlayOverrideUseActualCacheInspection` | 캐시hit여도2항목force→총3, technical만reuse override→총2.미리보기는 실제 분석 안 함 |
| `staleOnlyMetricCacheRemainsChargeableAndCorruptFreshBecomesMiss` | fresh삭제/stale잔존 및 깨진freshJSON→miss/총2.현재 policy의 stale write-only와 일치 |
| `metricsAclWriteFailureReturnsFalseAndPreviewBecomesMiss` | 실제 SET거절→cacheFeature1Metrics false/두키없음, GET거절→preview miss/총2 |
| `metricMissPopulatesCacheSignalHitReusesAndForceLoadsAgain` | loadOverlaySignals miss→Feature1fixture1회·캐시생성·signal1개, 다음hit호출 증가없음, force→2회·두키갱신 |
| `changedNewsFocusMustIncreasePreviewAndEstimateCost` | 실제 Overlay→실제News inspection 연결.정상캐시 총1 대조 후 focus 변경→MISS/추가1/총2 기대, preview와estimate 일치도 검사 |
| `missingNewsLayerRaisesCostAndForceRefreshChargesEvenValidCache` | 완전뉴스캐시force→총2, detail제거 후reuse→총2, 모델/제공자/score 저장0 |

모든 preview helper는 같은 종목/overlay/policy로 `estimateCredit`도 실행해 결과를 대조합니다. 단순히 작성된 테스트를 PASS로 간주하지 않으며 결과는 아래에 남깁니다.

## 실행 결과

최종 **21개17 PASS/4 FAIL**, skip0입니다. 증거는 `tests/.runtime/redis-isolated/runs/run-wqjld7fg/summary.json`, 같은 폴더의 `gradle.log`, `TEST-com.qaima.qa.IsolatedNewsOverlayRedisTest.xml`입니다. 실패는 `changedFocusShouldInvalidateInspectionAndRecomputeSentiment`, `nullCachedScoreShouldInvalidateInspectionAndTriggerModel`, `changedNewsFocusMustIncreasePreviewAndEstimateCost`, `overlappingRefreshMustTagEachScoreWithItsActualModelInputHash`이며 나머지17개는 PASS입니다. Gradle exit1은 보존한 회귀 실패 때문입니다.

- 최초 `run-zem5764b`는21개16 PASS/5 FAIL이었습니다. 기존 캐시 오판3메서드와 아래 새 동시성1메서드 외에 최초 생성 검사의 기대값1개가 잘못됐습니다. 제품은 정상 수집에서도 정보성 `NEWS_FILTER_APPLIED`를 붙이므로 빈 warnings 기대를 해당 코드1개로 바로잡았습니다. 이 테스트 기대값 오류를 제품 결함으로 합산하지 않습니다. 두 전체 실행의 동일21개를 중복 합산하지 않습니다.
- 신규 동시성은 두 실행에서 같은A/B hash·최종점수0.1/B hash·observation의B hash로 재현됐습니다. 실제 모델 호출이 아니라 fixture 모델 경계2회입니다.
- 최종 cold 경로에서 실제 PTTL은 list299986ms, refresh899984ms, detail604799988ms, focus604799989ms, sentiment604799991ms입니다. 지표 fresh86399999ms/stale604799996ms도 확인했습니다. 각각5m/15m/7d/24h 기대 범위 안입니다.
- 실제 ACL log는 GET10회/SET9회 거절입니다. 최초 연결 대조용 각1회가 포함됩니다. `CLIENT PAUSE` 사례는 `NEWS_CACHE_READ_FAILED`·기사 유지·동일 template 조회 복구가 PASS입니다.
- 두 실행 모두 fixture 정리 뒤 소유 표식1개만 남았고 표식 삭제 뒤 DBSIZE0, redisStopped true, 프로세스 종료0입니다.559개 입력의 실행 중 변경0, harnessError 없음입니다. 실행 당시의 실제 입력 해시는 각 summary에 있습니다.
- `tests` 밖 가시 코드·설정·문서924개 감사는 변경/소실/새파일0입니다. `tests/.runtime/redis-isolated/news-overlay-source-audit.json`에 보존합니다. 문서46개 대상 `.venv_wsl/bin/python -B -m unittest discover -s tests -v`의2개 검사는1.546초에 PASS했습니다. 로컬 링크와 설정 비밀값 대조이며 전체 보안 감사는 아닙니다. 로그는 `tests/.runtime/redis-isolated/news-overlay-documentation.log`입니다.

수정한 코드는 새 Java 파일과 기존 실행기의 suite 선택/해당 XML 수집, `test_documentation.py`의 문서 등록뿐입니다. 제품 코드와 기존 `backend/src/test`는 변경하지 않았습니다.

### F2-NEWS-CACHE-001 — 기존 결함의 실제 Redis 재현

- focus 본문/hash만 변경해도 실제 inspection은hit, 재분석은모델0회·이전0.75입니다. 점수null인 캐시도hit·모델0회·점수null입니다.
- 실제 Feature3OverlayService가 이 inspection을 사용해 변경된 focus를 `HIT`, 추가0/core포함총1로 판단합니다. 기대는MISS·추가1/총2이며 preview/estimate의 잘못된 결과가 서로 일치합니다.
- 이3개 메서드는 동일 기존 결함의 다른 경계입니다.3개의 새로운 독립 결함으로 세지 않습니다. focus/score 변형은 합성 캐시 주입이며 정상 쓰기가null을 만든다는 증거는 아닙니다.

### F2-NEWS-CACHE-RACE-001 — 겹친 강제 갱신이 점수의 입력 해시를 다른 요청 것으로 기록

재현은 임의 sleep 대신 `Sinks.One`와 구독 latch로 실행 순서를 고정했습니다.

1. 같은 기사42의 force A가 본문A로 생성한 모델 입력을 캡처하고 응답을 보류합니다.
2. force B는 다른 본문B로 완료합니다. 모델점수0.9와 B 입력의 SHA-256이 실제 sentiment 캐시에 저장됐는지 확인합니다.
3. A 응답0.1을 해제합니다. A 호출은0.1을 받지만 최종 Redis도0.1로 덮이고 `focusTextVersion`은B hash입니다.0.1의 observation 저장 요청도B hash입니다.

첫 실행의 A input hash는 `1f611c33de48c5db2477b96f92a2c2d4e2509a7b044bf6d6eba2001ab9df59dc`, B input hash와 최종 캐시/observation hash는 `450a5c651eb16aebbc09c605e8a287ac5fb7be55253ca48a2f3710348e5137c5`입니다. 원문과 점수는 모두 fixture입니다. summary의 `refreshRace`가 모델 호출2회·두 입력hash·최종 점수/hash·observation hash를 남깁니다.

원인은 [NewsSentimentService](../backend/src/main/java/com/qaima/service/feature2/NewsSentimentService.java)의 `analyzePendingSentiments`가 응답을 처리할 때 모델에 전달했던 input을 사용하지 않고 `getOrLoadFocusText`로 현재 공유 캐시를 다시 읽는 경계입니다. B가 그 사이 focus를 갱신했으면 A점수에 B입력의 출처가 붙습니다. 이 회귀는 가장 늦게 시작한 요청만 저장해야 한다는 별도 정책을 가정하지 않습니다. **저장하는 각 점수의 hash가 실제 해당 모델 입력과 일치해야 한다**는 조건을 검사합니다.

영향은 캐시 재사용과 관측 데이터의 입력/점수 연결이 잘못될 수 있다는 것입니다. 캐시 저장은 실제 Redis이고 observation은 실제 service가 만든 저장 요청 인자만 캡처했습니다. observation DB 영속성, 실제 모델 결과의 옳고 그름, 실사용 동시 갱신 발생률은 검증하지 않았습니다. 제품은 수정하지 않고 캐시·observation 두 단언의 FAIL을 유지합니다.

## 남은 범위

관측된 캐시 결과와 실제 차감/환불·분석 저장 orchestration의 결합, 실제 SQL observation 영속성, 실제 모델 품질·성능, 서로 다른 backend 인스턴스 사이 동시성, Redis TCP 단절/재접속·재시작/failover·장기 자연 만료는 별도입니다. 이번21개로 전체 QA 또는 Swagger123개를 완료한 것으로 보지 않습니다.
