# Redis TCP 단절·재시작·같은 클라이언트 복구 QA

검증일: 2026-09-28. 현재 `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03` 및 기존 미커밋 소스를 대상으로 합니다. 테스트/문서는 `tests` 내부에만 작성하고, 매 실행 Redis 상태는 새 외부 `/tmp/qaima-qa-redis-*`에 둡니다. 전체 목표와 집계는 [QA 진행 기록](QA_PROGRESS_2026-09-28.md)을 따릅니다.

## 실제 실행과 대체 경계

[이전 Redis18개](isolated-redis-cache.md)와 [뉴스/metrics21개](isolated-news-overlay-redis.md)의 ACL·짧은 TTL·일시 정지에 실제 TCP 단절/재접속·서버 재시작을 추가합니다. [Java 검사](java/com/qaima/qa/IsolatedRedisLifecycleTest.java)는 제품 service/reader, RedisConfig mapper/typed serializer, 실제 Lettuce/Redis7.0.15를 실행합니다. Repository·KIS/산업/Peer/뉴스 검색·감성 모델·Feature1/observation 저장은 합성 fixture입니다. 실제 SQL·HTTP Controller·브라우저·외부 API·모델은 포함하지 않습니다.

두 개의 독립 Lettuce client resources와 연결 factory, 두 PriceSnapshotReader 인스턴스를 사용합니다. 같은 JVM 안에서 캐시 공유와 각 인스턴스의 inflight 맵을 검사하며 여러 backend 프로세스/호스트를 기동한 것으로 보지 않습니다. 기존 core/news 테스트 클래스의 fixture/helper만 조합하고 기존 사례를 상속·재실행하지 않습니다.

## 명령과 장애 제어

```bash
cd /mnt/c/qaima
python3 -B tests/run_isolated_redis.py --suite lifecycle
```

단일 메서드는 `--test 메서드명`으로 선택합니다. 기존 core/news-overlay 옵션은 유지합니다. Gradle8.14 offline·Java17·의존성 캐시·출력 경로는 이전 Redis 검사와 같습니다. 제품 application 설정을 접속 대상으로 읽지 않으며 YAML의 Redis timeout2초와 같은 command timeout을 테스트 factory에 명시합니다. 상위90초 block/150초 JUnit 제한은 테스트가 무기한 남지 않도록 한 종료 한도이며 운영 응답시간 SLA가 아닙니다. 뉴스 내부2초 block 제한은 제품 구현 그대로입니다.

[실행기](run_isolated_redis.py)가 Redis를 새로 생성하고 [소유 TCP 프록시](redis_fault_proxy.py)를 통해 Java를 연결합니다. 프록시는 Redis 프로토콜을 해석하거나 응답을 만들어내지 않고 바이트를 전달/폐기하거나 socket을 닫습니다. 별도 RESP 연결은 프록시를 우회해 실제 저장값·키·PTTL·명령 수·소유권을 확인합니다.

| 동작 | 실제 효과와 독립 확인 |
|---|---|
| cut | 기존 양방향 socket을 닫고 새 연결은 허용. 같은 factory에서 조회가 다시 성공하고 accept 수가 증가하는지 확인 |
| partition | 기존 socket을 닫고 재접속을 거절. 원본 Redis는 살아 있어 별도 RESP로 저장되지 않은 키와 소유 정보를 확인 |
| blackhole | 기존 socket은 유지하지만 전달 바이트를 폐기. 제품의 실제 timeout/대체 조회를 실행하고 Redis GET 누계 불변·폐기 바이트 수를 대조 |
| restore | 장애 socket을 정리하고 새 연결을 허용. 두 기존 클라이언트가 소유 표식을 읽을 때까지 제한 시간 내 확인 |
| restart | runner가 직접 생성한 Redis의 port·dir·run_id·process_id·표식 확인→소유 Popen에 SIGTERM→exit0/포트 닫힘→동일 port/dir의 새 프로세스 시작. 새 run_id·빈 DB 확인 후 표식/테스트 ACL 재설정 |

Redis는 snapshot/AOF를 껐으므로 재시작 시 캐시가 사라지는 설정입니다. 영속성 복구·Sentinel/Cluster·failover는 별도입니다. 재시작은 PID 검색이나 사용자가 지정한 주소를 받지 않고 현재 실행기가 보유한 Popen만 대상으로 합니다. loopback 제어 API는 실행별 임의 토큰을 요구하며 action allowlist만 허용합니다. 토큰/Redis 비밀번호/전송 본문은 증거 JSON에 쓰지 않습니다.

사례 전후 restore→기존 두 client의 marker 조회→현재 run_id/dir/marker 검증→fixture 키 정리를 수행합니다. 마지막에는 표식 삭제/DBSIZE0·Redis/프록시/제어 포트 종료와 입력 해시 불변을 확인합니다.

## 22개 사례의 검증 방법

15개 일반 메서드와 재시작 매개변수7개로 총22개입니다. 최종 `run-c0aa7rdm`에서 **22 PASS / 0 FAIL**, errors/skipped0, JUnit110.435초·Gradle exit0입니다. 초기 실패나 재실행 없이 이번 전체 실행의22개만 집계합니다.

| 메서드 | 입력·독립 확인 |
|---|---|
| `warmedClientsReconnectAfterActualSocketCutWithoutNewFactories` | 키 retained를 저장하고 기존 socket만 끊음. 두 기존 factory가 동일 값 재조회·새 accept 증가·Redis run_id 유지 |
| `blackholedPriceCommandsTimeOutAndRecoverOnSameClient` | warm 연결을 blackhole로 전환. 실제 GET/SET timeout 뒤 DB fixture110 반환, Redis GET 증가0·가격키 없음·폐기 바이트>0. 복구 후 같은 reader가 가격/120초 TTL 저장 |
| `partitionedPriceUsesRepositoryAndRecoversCache` | 실제 연결 거절 중 가격110/DB1회·키 없음, 복구 후 동일 reader의 정상 저장/120초 TTL |
| `partitionedMarketSnapshotPreservesSharesAndWarnings` | 두 GET/두 SET 연결 실패에도 유동400/자사100·FIRST/SECOND 경고 유지. 두 캐시키 없음, 복구 후 값/1시간 TTL |
| `partitionedIndustryPreservesRepositorySeries` | 연결 실패에도 DB fixture3개 시계열·끝20%·제공자0회 유지. 복구 후 typed 값/30분 TTL |
| `partitionedPeerPreservesProviderResultAndWarnings` | 캐시 단절에도 Peer fixture 대상/경고 유지. 실패 SET 키 없음, 복구 후 동일 service가12시간 TTL 저장 |
| `partitionedRealtimeRetainsFreshProviderPrice` | 캐시 단절에도 제공자가격12,345.67·stale=false 유지. fresh/stale 키 없음, 복구 후20초 fresh TTL |
| `unreachableStaleCacheAndFailedProviderReturnExplicitPriceFailure` | 먼저 fresh/stale 저장, 제공자 오류+캐시 단절→null 가격/PRICE_FETCH_FAILED. 복구 후 fresh만 제거하여 실제 stale 가격/PRICE_STALE_USED 확인 |
| `partitionedRankingUsesSourceAndCanCacheAfterRecovery` | KST10시 fixture에서 단절 중 provider1회·목록1개·키 없음. 복구 후10초 TTL |
| `partitionedNewsInspectionIsMissWithoutProviderSideEffects` | 유효5계층 캐시 저장 후 단절→inspection miss·모델/제공자/Repository/observation 호출0. 복구 후 기존 캐시 hit |
| `partitionedNewsAnalysisKeepsComputedScoreAndCacheWarnings` | 빈 캐시/연결 단절에서 기사+모델점수0.25 유지·READ/WRITE 경고·캐시키 없음. 복구 후 재분석/inspection hit |
| `partitionedMetricsWriteReturnsFalseThenRecoversBothLayers` | 단절 중 metrics 저장false·fresh/stale 없음. 복구 후 true·24시간/7일 TTL |
| `partitionedOverlayPreviewTreatsUnreachableCacheAsChargeableMiss` | 유효 metrics 저장 후 연결 단절→MISS/총2비용, 실제 Feature1 등 호출0. 복구 후 HIT/총1비용. 실제 차감 원장은 이 범위 밖 |
| `restartClearsCacheAndExistingServiceRepopulates` | price/market/industry/peer/realtime/news/metrics7개 각각 실제 쓰기→서버 재시작→새 run_id/표식 외 키0→동일 service/client 재조회·캐시 재생성. 값·source2회·PTTL/뉴스hit/metrics비용을 각 서비스별 확인 |
| `separateClientAndServiceReuseSharedPriceWithoutSecondRepositoryCall` | A source110/B source220으로 구별. A 저장 후 B가110 재사용·B DB0회, 실제 socket cut 뒤에도 동일 동작 |
| `twoIndependentReadersCoalesceOnlyWithinEachInstance` | 두 reader에 각각12개 겹친 요청, DB latch2개를 보류하고 Redis GET24회 확인 후 해제.24개110·DB각1회/총2회. 전역 단일조회 보장을 가정하지 않음 |

## 실행 기록

최종 증거는 [summary](.runtime/redis-isolated/runs/run-c0aa7rdm/summary.json), [JUnit XML](.runtime/redis-isolated/runs/run-c0aa7rdm/TEST-com.qaima.qa.IsolatedRedisLifecycleTest.xml), 같은 폴더의 gradle.log입니다. 22개 모두 PASS이고 실제7번의 재시작에서 이전 프로세스 exit0·새 run_id·표식 추가 전 DBSIZE0을 확인했습니다. 같은 client/service가 값을 복구하고 TTL을 재생성했습니다.

- 실제 연결 거절369회, blackhole 폐기174bytes, 제어 이벤트81개/제어 오류0입니다. blackhole 중 서비스 GET이 Redis 명령 누계에 추가되지 않은 상태에서도 가격110을 반환했습니다. 응답을 mock하거나 가짜 Redis 오류를 반환한 결과가 아닙니다.
- 두 독립 reader/client는 warm 캐시를 공유해 B의 다른 DB fixture220을 호출하지 않고 A의110을 재사용했습니다. 빈 캐시에 겹친24개 요청은 인스턴스별 DB1회/총2회였습니다. 프로세스 간 전역 단일 요청 또는 실제 두 backend 프로세스 검증으로 확대하지 않습니다.
- 최종 fixture 정리 뒤 owner만1개, owner 삭제 후 DBSIZE0입니다. Redis 정상종료0, 프록시/제어 포트 닫힘·작업 thread 종료·활성 socket0, 강제 kill/정리 오류0입니다. 실행 입력570개 변경0, 별도 tests 밖924개 변경/소실/새파일0을 확인했습니다.

### 장애 시 대기 시간 관측

fallback 결과가 반환돼도 지연이 합산될 수 있습니다. 실제 command timeout2초를 사용한 이번 단일 합성 요청의 관측은 다음과 같습니다. 환경의 단일 실행값이며 운영 성능 보장이나 SLA 합격 기준이 아닙니다.

| 경로 | 관측 시간 |
|---|---:|
| 뉴스 cache inspection | 2.000초 |
| overlay preview / metrics write | 2.121초 / 2.073초 |
| price 연결 거절 / blackhole | 4.232초 / 4.177초 |
| 산업 / Peer | 4.177초 / 4.178초 |
| 실시간 새 가격 / 원천도 실패 | 6.276초 / 4.175초 |
| 시장 snapshot | 8.378초 |
| 랭킹 | 4.215초 |
| 뉴스 전체 분석 | **28.013초** |

뉴스 전체 분석은 여러 Redis read/write의2초 대기가 누적되지만 합성 기사·점수0.25와 READ/WRITE warning을 반환했고 복구 후 cache hit도 확인됐습니다. **이 통과를 전체 Feature2 사용자 대기 시간의 정상 판정으로 삼지 않습니다.** 여러 요소를 조립하는 실제 Feature2 분석의 timeout 누적·호출 취소·부분 결과 보존은 [전체 fallback 인계 기준](FALLBACK_QA_HANDOFF.md)에 따라 다음 우선 검증으로 남깁니다.

문서62개 대상 로컬 링크/설정 비밀값 검사2개 PASS(1.590초)는 `tests/.runtime/redis-isolated/lifecycle-documentation.log`에 보존했습니다. 메서드15개+재시작7개/문서16행/JUnit22개·입력570개·정리 대조는 실행 폴더의 `lifecycle-evidence-audit.json`, tests 밖924개 비교는 `tests/.runtime/redis-isolated/lifecycle-source-audit.json`에 기록했습니다.

## 남는 범위

실제 SQL/원천/모델/차감·HTTP 사용자 흐름의 장애 결합, 서로 다른 backend 프로세스·호스트, Redis 영속성/Sentinel/Cluster/장기 장애, 무중단 배포·실제 망 지연/패킷 재정렬, 전체 성능 SLA와 기존 캐시 경고/뉴스 입력 hash 결함은 별도입니다. 이 단계로 전체 QA 완료를 주장하지 않습니다.
