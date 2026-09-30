# Feature2 실제 Redis·HTTP 취소·과금/리포트 SQL QA

## 상태와 범위

2026-09-28 기능2 조립39개와 실제 브라우저13개의 후속입니다. 최종 `run-8m54svrw`는 **15개12 PASS/3 FAIL**, errors/skipped0, JUnit274.921초입니다. 실행기 exit1은 제품 실패 단언의 결과이며 소유 MySQL/Redis는 모두 exit0으로 종료했습니다. 전체 기준은 [fallback 인계](FALLBACK_QA_HANDOFF.md), 누적은 [진행기록](QA_PROGRESS_2026-09-28.md)을 따릅니다.

수정은 tests 안에 한정합니다. 실행기는 새 `/tmp/qaima-qa-http-*` MySQL과 `/tmp/qaima-qa-redis-*` Redis를 소유하며 port/datadir/schema/run_id/PID/표식을 확인합니다. 기존 DB/Redis와 외부 시장 제공자·유료 LLM은 사용하지 않습니다.

## 실제 경계와 합성 경계

- 실제: HTTP/1.1 client→Spring Security/JWT/Feature2 Controller→실제 조립기→StockResolver/PriceSnapshotReader·IndustryReader/IndustryIndexService/IndustryIndexReaderImpl·PeerClusterServiceImpl·NewsSentimentService→Lettuce/제품 serializer/실제 Redis. 실제 CreditService/report/JPA/MySQL·저장 상세도 연결합니다.
- 합성: 시장 SQL Repository·금리/공매도/거시/수급 leaf·Peer/뉴스 검색/본문/감성/설명 제공자. 가격·산업 시계열 source Repository는 mock이며 실제 MySQL은 사용자/원장/리포트 저장소입니다. FastAPI·브라우저·모든 기능2 하위 원천을 실행한 것은 아닙니다.
- 실제 뉴스 서비스의 입력 정제·캐시 계층·감성 대체 흐름을 사용합니다. 기사1개/감성.25, 가격110, 산업30개100→129/29%, 기준금리3.25, 공매도7.5가 독립 sentinel입니다.
- 기존 transport/fixture helper만 조합합니다. 과거 suite의 테스트 메서드를 상속하거나 다시 집계하지 않습니다.

## 명령과 장애 주입

```bash
python3 -B tests/run_isolated_backend.py --suite feature2-resilience
```

정상 대조만 실행:

```bash
python3 -B tests/run_isolated_backend.py --suite feature2-resilience --test baselineColdAndWarmHttpReuseAllFourRealCachePaths
```

[Java 검사](java/com/qaima/qa/IsolatedFeature2ResilienceTest.java), [소유 Redis 실행기](isolated_fault_redis_fixture.py), [TCP 장애 프록시](redis_fault_proxy.py), [공통 실행기](run_isolated_backend.py).

TCP partition은 기존 연결을 끊고 재접속을 거절합니다. blackhole은 실제 연결의 바이트를 버려 제품 timeout을 실행합니다. restart는 실행기가 소유한 Popen만 정상 종료하고 같은 포트/datadir에 새 run_id의 서버를 시작합니다. 프록시는 RESP/JSON 응답을 만들지 않습니다. 별도 RESP 연결로 키·값·TTL·표식·run_id를 대조합니다.

Lettuce command timeout은 제품 설정과 같은2초입니다. 뉴스 내부 Redis `.block` 제한도 제품 그대로입니다. 정상/장애 HTTP의95초 한도와 JUnit150초는 검사 종료 장치이며 제품 SLA가 아닙니다. 단계별 subscribe/value/terminal 시각과 HTTP 전체 시간을 함께 기록해 순차 누적 대기를 관측합니다. 정책에 응답 SLA가 없으므로 느리다는 이유만으로 임의 FAIL을 만들지 않습니다. 실제 분석 client의 재접속은45초 관측 한도 내에서 marker 조회의 시도 수/소요 시간을 기록합니다. 새 factory로 교체하거나 재접속 설정을 바꾸지 않습니다.

## 최종15사례

| 메서드/파라미터 | 결과 | 검증 방법 |
|---|---|---|
| baselineColdAndWarmHttpReuseAllFourRealCachePaths | PASS | cold→warm 실제 HTTP2회/네 source 호출 불증가/TTL·뉴스 inspection hit/원장−1−1/report2 |
| cacheTransportFailurePreservesComputedResultsAndRecovers: partition | PASS | 네 캐시 경로 연결 거절→정상 정량·뉴스/cache 경고/키 없음→복구 요청·SQL |
| cacheTransportFailurePreservesComputedResultsAndRecovers: blackhole | PASS | 실제 바이트 폐기·timeout→부분 fallback·전체 시간→같은 client 복구 |
| warmCacheUnavailableAndBothPeerAndSentimentProvidersFail | PASS | 먼저 캐시 저장→Redis 단절+Peer/감성 오류→접근 못 하는 옛 점수 대신 null/안내·나머지 정량 유지→복구 |
| redisRestartRepopulatesCachesWithoutLosingSqlReports | PASS | 실제 Redis 정상 종료/재시작·빈 캐시·동일 service 재수집/SQL 이전 snapshot 불변 |
| redisRestoredDuringNewsAllowsSameRequestAndNextRequestToComplete | FAIL | 뉴스의 첫 캐시 읽기 실패 후 source 검색을 latch로 보류→Redis 복원 후 해제·같은 HTTP 완료→정상 다음 요청/저장 캐시 경고→목록 키만 제거한 진단 요청 대조 |
| redisAndBothPriceIndustryRepositoriesFailKeepIndependentMetrics | PASS | Redis+가격/산업 Repository 동시 오류→가격/산업 null·금리/뉴스/설명 유지·SQL→복구 |
| redisAndPeerAndSentimentFailuresKeepRemainingQuantitativeData | PASS | cold Redis blackhole+Peer/감성 오류→독립 가격/산업/공매도 보존·경고·SQL→복구 |
| clientDisconnectCancelsDownstreamAndRecordsCreditRecovery: STOCK | PASS | 실제 Redis GET 대기에서 차감 확인→HTTP future cancel→서버 CANCEL·저장/원장 관측·다음 분석 |
| clientDisconnectCancelsDownstreamAndRecordsCreditRecovery: PEER | PASS | 실제 Peer service의 제공자 Mono 보류→HTTP cancel·제공자 cancel 전파/원장·다음 분석 |
| clientDisconnectCancelsDownstreamAndRecordsCreditRecovery: NEWS | PASS | 실제 뉴스 감성 block 대기→HTTP cancel·block 내부 구독 cancel·원장/후속 분석 |
| clientDisconnectCancelsDownstreamAndRecordsCreditRecovery: EXPLAIN | PASS | 정량 조립 후 설명만 보류→HTTP cancel·설명 구독 종료·원장/후속 분석 |
| actualHttpClientTimeoutCancelsPendingPeerAndAllowsRetry | PASS | 정상 warmup 완료 후 Peer 캐시만 삭제/원천 보류→실제 HttpClient3초 timeout→서버 Peer CANCEL/원장·재시도 |
| cancelledOlderPriceLoadCannotOverwriteLaterSuccessfulPrice | FAIL | 첫 source110을 latch로 보류→HTTP cancel→새 source220 분석/저장 완료→옛 source 해제→실제 캐시와 세번째 HTTP 가격 비교 |
| cancelledOlderIndustryLoadCannotOverwriteLaterSuccessfulSeries | FAIL | 첫 산업 source100→129/29%를 보류→HTTP cancel→새 source100→158/58% 분석/저장→옛 source 해제→실제 캐시/다음 HTTP 비교 |

## 판정 기준과 해석 제한

정책 [공개 정책 §5.4](../policy/QAIMA_POLICY_PUBLIC.md)는 핵심 분석 실패 환불과 설명/저장만 실패할 때 정량 성공 기준을 명시합니다. **클라이언트 취소의 환불 시점·책임은 명시하지 않습니다.** 취소 사례는 실제 TCP 취소 전파·완료되지 않은 요청의 리포트 미생성·후속 요청 복구를 단언하고, 기존 차감 잔존/환불은 정책 공백 관측으로 기록합니다. 이 사례의 PASS를 취소 과금 정책 적합성으로 해석하지 않습니다.

취소된 오래된 source가 나중의 성공 값을 덮는 경합은 실제 데이터 정합성을 기준으로 검사합니다. 강제 latch로 순서를 재현한 것이며 자연 발생률을 측정하지 않습니다. 저장 SQL snapshot과 현재 Redis 값/다음 HTTP 값을 각각 대조합니다.

전체 응답 HTTP200만으로 PASS하지 않습니다. 공개 response=data SQL snapshot=소유자 상세, warnings_json=meta warnings, 원장/잔액/report 수와 독립 기대값을 확인합니다. 요청 취소 시의 토큰·쿠키·Redis/제어 암호는 증거에 기록하지 않습니다.

## 확인된 실패 경로

- **PRICE-CANCEL-STALE-001**: 첫 전체 실행에서 취소한 오래된 가격 조회가 계속 진행되어 나중에 완료한220 캐시를110으로 덮었고 다음 실제 HTTP/저장 snapshot도110이었습니다. [PriceSnapshotReader](../backend/src/main/java/com/qaima/service/marketdata/reader/PriceSnapshotReader.java)의 `.cache()` 뒤 subscriber별 `doFinally`가 취소 시 inflight 엔트리를 제거하지만 공유 source는 계속 실행되는 구조와 일치합니다. 동일 구조의 [IndustryIndexReaderImpl](../backend/src/main/java/com/qaima/service/marketdata/reader/IndustryIndexReaderImpl.java)도 아래 사례에서 재현했습니다.
- **INDUSTRY-CANCEL-STALE-001**: 새 산업 시계열의 마지막 수익률58%를 저장/응답한 뒤 취소된 이전 source29%가 실제 Redis를 덮었고, 다음 HTTP/SQL도29%였습니다. 두 source 조회는 모두 동일 key이며 총2회입니다. 가격과 산업 각각의 실제 reader에서 같은 취소/캐시 경합을 재현했습니다. 자연 발생률·여러 backend 프로세스 경합은 측정하지 않았습니다.
- **NEWS-CACHE-RECOVERY-WARNING-001**: Redis가 복구되고 기사/감성이 정상 캐시에서 재사용돼도 이전 NEWS_CACHE_READ_FAILED가 새 meta/리포트에 남았습니다. [NewsSentimentService](../backend/src/main/java/com/qaima/service/feature2/NewsSentimentService.java)가 목록 payload에 경고를 저장하고 다음 cache hit의 warnings로 합치는 경로입니다. 점수 오류가 아니라 과거 transport 오류와 현재 요청 상태를 구분하지 못하는 문제이며 브라우저 표시까지 검사한 것은 아닙니다. source latch로 읽기 실패→복구→목록 저장 순서를 고정했습니다. 목록 payload/다음 정상 응답/SQL에 같은 경고가 남았고, 소유 Redis의 목록 키만 제거한 진단 요청은 warnings=[]였습니다. 감성 모델 호출은 총1회로 유지됐습니다.

## 증거와 정리

실행별 `tests/.runtime/backend-isolated/runs/<run>`에 명령·JUnit XML·summary/단계별 시각·원장·소스 해시·서버/프록시 정리를 보존합니다. HTTP future·latch를 종료하고 소유 캐시 키/업무 테이블/trigger를 정리한 후 서버를 종료합니다. 초기 구성 실패와 최종 제품 실패를 구분하여 누적합니다.

최종 증거는 `tests/.runtime/backend-isolated/runs/run-8m54svrw`의 summary/XML/`feature2-resilience-evidence-audit.json`입니다. 실제 응답=SQL snapshot=소유자 상세28개, 고유 report28개·USE 원장35개/환불0을 대조했습니다. 차이7개는 네 명시 취소·HTTP timeout1개·두 경합의 취소된 첫 요청입니다. 각 취소의 원장과 후속 요청을 구분했으며35차감을 모두 정상 과금으로 판정하지 않습니다.

네 명시 취소는 각각 서버/leaf의 CANCEL, 잔액5→4/원장−1/report0, 후속 정상 분석의 잔액3/report1을 확인했습니다. 실제3초 HTTP timeout도 보류된 Peer 구독을 취소했고 차감은 남았습니다. `cancel(true)` 반환true·future exceptional completion·서버 CANCEL을 함께 사용했으며 JDK future의 `isCancelled` 필드만으로 전송 취소를 판정하지 않았습니다. 취소 과금은 위 정책 공백 관측입니다.

| 최종 관측 | 시간/결과 |
|---|---|
| Redis 연결 단절 중 전체 분석 | 40.654초, 정량/뉴스/설명·원장−1/report1 유지 |
| Redis 바이트 차단 중 전체 분석 | 40.569초, 정량/뉴스/설명·원장−1/report1 유지 |
| Redis+가격/산업 Repository 오류 | 36.366초, 가격/산업 null·독립 정량/뉴스/설명 유지 |
| cold Redis+Peer/감성 오류 | 28.675초, 독립 가격/산업/공매도 유지·기사1건 유지/점수null·경고 |
| warm 캐시 접근 불가+Peer/감성 오류 | 28.692초, 접근 못 하는 과거 정상 점수 대신 null·경고 |
| 같은 분석 client 재접속 확인 | 관측 최대27.340초/13회 marker 시도; 다른 느린 사례23.137초/11회 |

연결 단절 사례의 단계별 관측은 가격4.200초·산업4.191초·Peer4.200초·뉴스28.012초입니다. 각 단계의 첫 subscribe→value 시각 차이를 대조했으며 상세는 evidence audit에 있습니다.

시장 source는 즉시 반환하는 합성 자료이고 뉴스1건 조건입니다. 실제 지연 분포/여러 기사/외부 제공자 지연의 상한이나 SLA 측정이 아닙니다. 재접속 확인 시간은 Redis 복원 후 marker 조회를 시작한 시점부터의 관측입니다. 제품 command timeout2초와 뉴스의 반복된 Redis block이 전체 요청에 누적되므로, 개별 제한을 전체 응답시간으로 해석하지 않습니다.

입력579개 실행 중/최종 해시 불변, tests 밖 가시 코드/설정/문서924개 변경·소실·새 파일0을 확인했습니다. 범위 감사는 `tests/.runtime/backend-isolated/feature2-resilience-source-audit.json`입니다. 업무9테이블/trigger0·Redis DBSIZE0·MySQL/Redis 정상 종료0·포트 닫힘, 프록시/제어 포트/worker 종료와 active socket0을 확인했습니다. 실제 Redis 재시작1회는 이전 exit0·새 run_id·표식 전 DB0이며 기존 SQL 리포트는 보존됐습니다. 외부 제공자·기존 DB/Redis 사용은 없습니다.

문서65개 대상 링크/설정 일부 비밀값 검사2개 PASS를 확인했습니다. 실제15사례와 위 표, 누적22묶음485개391 PASS/94 FAIL도 대조했습니다. 문서 검사 명령/로그:

```bash
.venv_wsl/bin/python -B -m unittest discover -s tests -p test_documentation.py -v
```

`tests/.runtime/backend-isolated/feature2-resilience-documentation.log`

## 초기 실행과 보정 이력

초기 `run-i2ijpcmh`는 테스트 import 경로, `run-iukrhoru`는 테스트에서 참조한 Repository 메서드명 오류로 compileTestJava에서 종료했고 둘 다 JUnit0입니다. 제품 실패로 집계하지 않습니다. 정상 대조 `run-ivdgkakf`의1 FAIL도 공개 산업지수 시계열을 내부 `pct`명으로 단언한 테스트 오류입니다. 실제 응답의 `value`는 기대29%, 네 실제 source 각1회·차감−1/report1·SQL=상세까지 확인하고 필드 경로를 보정했습니다. 세 초기 실행도 업무9테이블/trigger0·Redis DBSIZE0·소유 서버 정상 종료/입력 변경0을 확인했습니다.

첫 전체 `run-02r1go39`는14개9 PASS/5 FAIL, JUnit275.101초입니다.5개 중2개는 실제 분석 client가 기존15초 복구 관측 한도 내 marker를 읽지 못했고,1개는 유효한 기존 가격 캐시의 남은85,505ms를 새 저장 TTL처럼 기대한 테스트 오류입니다. 실제 분석 client만 사용하도록 helper 구성을 정리하고 재접속 시간을45초 한도까지 기록하도록 보강했습니다. 제품의 timeout/backoff는 변경하지 않았습니다. 재사용 캐시는 TTL>0 및 원래 상한 이하인지 검사합니다.

나머지2개는 제품 관측입니다. 취소된 옛 가격110이 새로 완료한220을 실제 Redis에 덮어써 세번째 HTTP/SQL이110이었고, Redis 복구 후 다음 정상 뉴스 요청에 이전 NEWS_CACHE_READ_FAILED가 남았습니다. 가격/산업 reader의 같은 `.cache()`·`doFinally` 구조를 확인해 산업 경합1개를 추가했습니다. 뉴스는 캐시 읽기 실패와 복원 시점을 source latch로 고정하고, 저장 목록의 경고와 목록 키만 제거한 후의 응답까지 대조하도록 보강했습니다. 첫 전체의 연결 단절/응답 차단 분석은 각각40.602/40.589초에 결과/SQL을 보존했습니다. 네 취소 지점과 실제 HTTP timeout, Redis 재시작은 통과했습니다. 기존 실행을 최종15개와 중복 합산하지 않습니다.

## 남은 범위

실제 원천 SQL/여러 뉴스 기사/FastAPI/브라우저 전체 결합, 다중 backend 프로세스·강제 프로세스 종료/환불 복구·중복 요청 식별자 정책·실제 지연 분포·모든 기능1/3 fallback은 전체 목표에 남습니다. 취소 환불 정책의 확정·제품 결함 수정은 이 tests-only 검증에서 수행하지 않았습니다.
