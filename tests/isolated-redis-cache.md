# 격리 Redis 캐시 QA — 2026-09-28

이 검증은 현재 `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03`와 기존 미커밋 변경을 포함한 소스를 대상으로 합니다. 제품 코드를 수정하지 않았습니다. 실행별 `summary.json`의 입력 SHA-256이 검증 대상을 특정합니다. 전체 QA는 [진행 기록](QA_PROGRESS_2026-09-28.md)을 따릅니다.

## 목적과 검증 경계

기존 [캐시 reader 검사](../backend/src/test/market-cache-readers.md)는 Redis 전송을 대체했습니다. 이번에는 실제 Redis 7.0.15 TCP → Lettuce → 제품 `RedisConfig`의 String/typed template → 제품 service/reader를 사용합니다. 반환값뿐 아니라 별도 RESP socket으로 저장 JSON·서버 PTTL·명령 수를 확인합니다.

- 실제 실행: `MarketSnapshotCacheService`, `RealtimePriceService`, `PriceSnapshotReader`, `IndustryIndexReaderImpl`, `PeerClusterServiceImpl`, `TopRankingReader`, 제품 ObjectMapper/serializer.
- 대체 경계: Repository, KIS/Peer/산업지수 제공자, 거래일 calendar. 실제 MySQL·외부 API·FastAPI·LLM은 이 검사에서 호출하지 않습니다. HTTP Controller와 전체 Spring application context도 기동하지 않습니다.
- 랭킹의 제품 시각만 Mockito static으로 `2026-09-25` KST에 고정합니다. Redis 서버 시간·TTL은 조작하지 않습니다. 만료 검사는 `EXPIRE/PEXPIRE`로 줄이지 않고 실제 10·20·60·120초를 기다립니다.
- ACL 오류는 격리 서버가 반환한 실제 `NOPERM`입니다. 네트워크 단절·패킷 지연·재접속·운영 설정의 2초 timeout 검증은 아닙니다. 이 연결의 command timeout은3초, 단일 block 제한은8초입니다.
- 긴 TTL은 실제 PTTL과 저장/재사용을 확인합니다. 20분·30분·1시간·12시간·장외 약62시간의 자연 만료까지 기다렸다는 의미는 아닙니다.

## 재현 명령과 격리

새 코드: [실행기](run_isolated_redis.py), [Java 사례](java/com/qaima/qa/IsolatedRedisCacheTest.java). 이전 DB 단계의 [Gradle init](backend-isolated.init.gradle)을 재사용해 테스트 source set을 루트 `tests/java`로 바꾸고 모든 Gradle 출력은 `tests/.runtime/backend-isolated`로 보냅니다. 기존 `backend/src/test`와 제품 build 파일은 읽기만 합니다.

시스템 설치 없이 Ubuntu noble 패키지를 외부 `/tmp`에 추출했습니다. 이 날짜에 내려받은 Redis package version은 `5:7.0.15-1ubuntu0.24.04.4`입니다. `redis-tools`에 실제 서버 바이너리가 포함됩니다. 재현 시 패키지 버전은 저장된 파일명과 SHA-256으로 대조합니다.

```bash
mkdir -p /tmp/qaima-qa-redis-20260928/packages /tmp/qaima-qa-redis-20260928/root
cd /tmp/qaima-qa-redis-20260928/packages
apt-get download redis-server redis-tools libjemalloc2 liblzf1
python3 -B - <<'PY'
from pathlib import Path
import subprocess
base = Path('/tmp/qaima-qa-redis-20260928')
for package in sorted((base / 'packages').glob('*.deb')):
    subprocess.run(['dpkg-deb', '-x', str(package), str(base / 'root')], check=True)
PY
cd /mnt/c/qaima
python3 -B tests/run_isolated_redis.py
```

단일 사례는 `python3 -B tests/run_isolated_redis.py --test 메서드명`으로 실행합니다. 실행기는 매번 다음 순서를 수행합니다.

1. `/tmp/qaima-qa-redis-*` 임의 새 디렉터리·빈 loopback 포트·임시 임의 password/소유 표식을 생성합니다. 기존 Redis 주소와 application YAML은 접속 정보로 읽지 않습니다. `bind 127.0.0.1`, 인증 필수, snapshot/AOF 저장 해제, 64MB/noeviction을 지정합니다.
2. 소유 프로세스를 시작하고 `INFO server`의 port/run_id, `CONFIG GET dir`, `qa:owner` 값으로 서버를 확인합니다. Java도 연결 전과 매 사례 전후 동일 확인을 합니다. 테스트 데이터 삭제는 이 가드가 성공한 새 인스턴스에서만 수행합니다.
3. `qa_denied` 전용 ACL 사용자를 만들어 GET/SET만 거부합니다. 실제 template의 두 명령이 `NOPERM`으로 실패하는 대조 검사 후 제품 fallback을 실행합니다. admin RESP 연결로 실패한 write가 키를 남기지 않았는지 확인합니다.
4. 기존 읽기 전용 Gradle8.14 의존성 캐시와 `--offline --no-daemon --max-workers=2`를 사용하고 `com.qaima.qa.IsolatedRedisCacheTest`만 선택합니다. `application*.yml/yaml/properties`는 test resource에서 제외합니다.
5. 사례 전후 fixture 키를 지우고 최종 소유 표식1개만 남았는지 확인합니다. 표식 삭제 뒤 DBSIZE0을 확인하고 소유 Redis만 SIGTERM 종료합니다. 종료 코드·포트 닫힘·입력 해시 불변·새 JUnit XML·ACL log를 summary에 남깁니다.

`KEYS`/삭제 명령은 위 소유 가드를 통과한 폐기 가능한 새 Redis에만 사용합니다. 운영 캐시 관리 절차를 시험한 것이 아닙니다. 로그는 `tests/.runtime/redis-isolated/runs/run-*/`, Redis 설정과 서버 로그는 외부 `/tmp`에 보존합니다. 임시 비밀번호는 공개 문서나 summary에 기록하지 않습니다.

## 사례별 방법

아래는18개 Java 메서드의 검사 방법입니다. 실제 결과와 실행 이력은 다음 절에 기록합니다. TTL 단언은 예상값 이하·예상값−5000ms 초과이며, 정확한 관측값은 summary의 `observations`에 보존합니다.

| 메서드 | 입력·독립 확인 |
|---|---|
| `marketSnapshotRoundtripPreservesWarningsAndRejectsFutureForHistoricalRead` | 발행1000·유동40%·자사10% →400/100, 중복 warning 정리. String JSON 두 키 일치·1h/60s PTTL·두 번째 DB 호출 없음. 미래일 캐시를 넣고 과거 요청이 DB 날짜로 돌아가는지 확인 |
| `marketSnapshotCorruptJsonAndActualAclErrorsFallBackToRepository` | 실제 두 키의 깨진 JSON → DB fixture. ACL GET/SET 실패에도 DB 결과 유지, 저장된 키 없음 |
| `typedPriceRoundtripRetainsExactDecimalsAndSeparatesFrequencyAndStock` | 110/100 →10%, typed 왕복 후 DB1회,120s PTTL. 종목/일봉·주봉 키 구분, 정수부39자리·소수부2자리 금액의 JSON/BigDecimal 보존 |
| `typedPriceMalformedJsonAndActualAclErrorsFallBackToRepository` | 실제 키의 잘못된 JSON 역직렬화 오류 및 ACL GET/SET 오류 → 동일 계산 결과, 거절된 SET의 키 없음 |
| `twentyFourOverlappingRealRedisMissesShareOneRepositoryFetch` | Repository 첫 호출 latch를 보류.24개 reactive 구독과 서버 GET 수 증가를 확인한 뒤 해제.24개 값·Repository1회·실제 키/TTL 확인. 부하 성능 시험은 아님 |
| `emptyPriceRepositoryShouldCompleteEmptyWithoutMapperError` | 실제 캐시 miss·Repository 빈 list → empty 기대. 기존 `PRICE-SNAPSHOT-EMPTY-001` 회귀 |
| `industryTypedRoundtripPreservesKoreanOffsetSeriesAndThirtyMinuteTtl` | 날짜 역순100/110/120 → 오름차순0/10/20, 한글명·+09:00 offset·typed JSON 왕복·DB1회·제공자0회·30min PTTL |
| `industryCorruptJsonAndActualAclErrorsPreserveRepositorySeries` | 깨진 typed JSON과 실제 GET/SET 거절에도3개 시계열 반환, 거절된 write의 키 없음 |
| `industryCachedPartialSeriesShouldRetainSourceWarning` | 요청3개/DB1행/제공자빈결과 → 최초 경고 확인. 실제 저장 후 두 번째1행도 경고 유지 기대. 기존 `INDUSTRY-CACHE-WARNING-001` 회귀 |
| `peerEnvelopePreservesWarningsTwelveHourTtlInspectionForceAndLegacy` | result+warning envelope 왕복·12h PTTL. inspection의 asOf/호출 없음, force로 설명 교체, 구형 direct DTO 재사용 확인 |
| `peerEveryRequestParameterSeparatesKeysAndEquivalentInstantsShare` | 산업·종목·주기·window·peerCount·maxLag·displayLimit·기간 변형9키. 같은 instant의 UTC/+09:00 요청은 한 키 공유, provider 총9회 |
| `peerCorruptStructuresAndActualAclErrorsFallBackWithoutLosingResult` | 깨진 JSON/빈객체/불완전 envelope의3회 fallback. ACL 실패에서 결과 유지·키 없음·inspection miss가 provider를 추가 호출하지 않음 |
| `realtimeCorruptFreshFallsBackAndCorruptStaleReturnsExplicitFailure` | 깨진 fresh → 쉼표 가격12,345.67 파싱. fresh 삭제와 제공자오류 → stale 값/경고. stale도 깨지면 null 가격/PRICE_FETCH_FAILED |
| `realtimeActualAclReadAndWriteErrorsPreserveFreshProviderPrice` | 실제 GET/두 SET 거절에도 새 제공자가격 유지, fresh/stale 키 없음 |
| `realtimeMissingQuotePriceAccessorShouldCompleteEmptyWithoutMapperError` | fresh/stale 없는 상태·제공자오류에서 quote의 null/경고 확인. `getPrice`도 empty 완료 기대 |
| `rankingMalformedJsonAclErrorsAndPreOpenBypassUseActualTransport` | 깨진JSON fallback·10s PTTL.08:25/08:29:59 조회에서 Redis GET/SET 서버 누계 불변·provider 구독 증가. ACL 실패에도 목록 유지 |
| `rankingAfterCloseStoresTtlUntilNextTradingDay0825` | 금요일18:00, 다음거래일 월요일 fixture →62h25m PTTL, 캐시 hit가 provider 재구독 안 함 |
| `naturalTenTwentySixtyAndOneHundredTwentySecondExpiryUsesFallbacks` | 4종 키를 먼저 저장하고 실제10/20/60/120초 순서로 GET nil/PTTL−2 확인. 랭킹 재수집, 시세 stale 경고, latest→1h snapshot 재사용/DB1회,120s 가격 재조회/새값120/TTL 재설정. 각 경과시간 기록 |

## 실행 결과

최종 전체 실행 **18개15 PASS/3 FAIL**, skip0입니다. 증거는 `tests/.runtime/redis-isolated/runs/run-mnndkln9/summary.json`, 같은 폴더의 `gradle.log`, `TEST-com.qaima.qa.IsolatedRedisCacheTest.xml`입니다. 위 표의 빈 가격 목록·산업 경고 유지·실시간 null 가격 메서드3개가 FAIL이고 나머지는 모두 PASS입니다. Gradle exit1은 이 회귀 실패 때문이며 실행기 오류는 없습니다.

- 558개 입력 해시 실행 중 변경0. Java17.0.20.1/Gradle8.14/Redis7.0.15를 사용했고, 제품 `compileJava`는 기존 산출물과 입력이 일치해 UP-TO-DATE였습니다. 테스트는 새로 compile/실행했습니다.
- 실제 ACL log에 `qa_denied`의 GET9회/SET9회 거절이 남습니다. 연결 대조 검사 각1회와6개 제품 서비스의 fallback 경로를 포함합니다.
- 두 실행 모두 최종 fixture 키0·소유 표식 삭제 후 DBSIZE0, `redisStopped=true`, 종료 코드0, `sourceChangedDuringRun=[]`입니다. 서버는 종료됐고 외부 설정/로그만 남았습니다.
- `tests` 밖 가시 코드·설정·문서924개의 기준 해시 비교는 변경/소실/새파일0입니다. `tests/.runtime/redis-isolated/source-audit.json`에 보존합니다.
- 문서45개를 대상으로 로컬 링크·설정 비밀값 검사2개 PASS입니다. 명령은 루트의 `.venv_wsl/bin/python -B -m unittest discover -s tests -v`이며 전체 보안 감사는 아닙니다.

최초 실행 `tests/.runtime/redis-isolated/runs/run-aw6kb1a_/summary.json`은18개14 PASS/4 FAIL입니다. 아래 제품 회귀3개 외에 자연 만료 검사1개가 JVM 경과시간 하한 단언에서 실패했습니다. 최종 실행과 중복 합산하지 않습니다.

### 만료 시각 측정 보완

최초 실행의 Redis PTTL은10/20/60/120초로 설정됐으나, 10·20·60초 키가 사라진 시점의 JVM nanoTime 경과는9481/19533/58936ms였습니다.120초 키에서 `Expired too early` 단언이 실패했습니다. 최초 코드는 이 단언 뒤에 경과값을 기록했으므로120초의 정확한 JVM 값은 남지 않았습니다. 이 결과만으로 제품 TTL 결함이나 WSL 시계 보정의 원인을 확정하지 않습니다.

Redis는 만료 정보를 서버 절대 Unix 시각으로 보관하며 시스템 시각 변화의 영향을 받습니다. [Redis 공식 EXPIRE 설명](https://redis.io/docs/latest/commands/expire/). 이에 테스트를 보완해 `PEXPIRETIME`의 절대 만료 시각과 `TIME`의 관측 시각을 직접 기록하고, 키가 사라진 시각이 서버의 만료 시각 이상인지 확인합니다. JVM monotonic 경과도 진단값으로 함께 남깁니다. 제품 TTL이나 서버 시계·키 만료 설정을 바꾸지 않았습니다.

최종 실행의 관측은 아래와 같습니다. 서버 경과는 `TIME − PEXPIRETIME + 지정 TTL`로 계산하며, JVM 경과는 최초 조회가 끝난 직후의 nanoTime 기준입니다.200ms polling/왕복 지연이 포함됩니다. 서버와 JVM 측정 차이는 실제로 남았으며 운영 시계 안정성이나 그 원인까지 검증하지 않았습니다.

| 키 | 지정 TTL | 서버 기준 경과 | JVM 경과 | 결과 |
|---|---:|---:|---:|---|
| ranking-stocks… |10s|10121ms|10120ms|GET nil/PTTL−2, provider 재구독 |
| price:QA_REALTIME |20s|20175ms|19595ms|GET nil/PTTL−2, stale 값과 경고 |
| snapshot_latest:QA_MARKET |60s|60065ms|58893ms|GET nil/PTTL−2,1h 키 재사용/DB1회 |
| price:snapshot:QA_PRICE:ONE_D |120s|120040ms|117681ms|GET nil/PTTL−2, DB2회·새값120·TTL 재설정 |

20min stale의 PTTL1199998ms,30min 산업지수1800000ms,1h snapshot3599997ms,12h Peer43199999ms, 장외62h25m 랭킹224699999ms도 서버에서 읽었습니다. 이 장기 키의 자연 만료는 검증하지 않았습니다.

### 제품 회귀

1. **PRICE-SNAPSHOT-EMPTY-001 — 기존 결함 재현.** 실제 Redis miss 후 Repository 빈 목록이 `toSnapshot`에서null이 되고 Reactor `map`이 `NullPointerException`을 반환합니다. empty 완료가 기대값입니다. [PriceSnapshotReader](../backend/src/main/java/com/qaima/service/marketdata/reader/PriceSnapshotReader.java)의 뒤쪽 null 방어에 도달하지 못합니다. 실제 DB 없이 Repository 반환값을 고정했으며 공개 API 전체500으로 확대하지 않습니다.
2. **INDUSTRY-CACHE-WARNING-001 — 기존 결함 재현.** 최초1행 결과의 `INDUSTRY_INDEX_FETCH_FAILED`가 실제 Redis 저장/조회 뒤 빈 warnings로 바뀝니다. 첫 번째·캐시 재사용 warnings를 summary에 남겼고 Repository는1회입니다. [IndustryIndexReaderImpl](../backend/src/main/java/com/qaima/service/marketdata/reader/IndustryIndexReaderImpl.java)이 캐시에 block만 넣고 meta 경고를 보존하지 않는 경계입니다.
3. **REALTIME-PRICE-EMPTY-001 — 이번에 추가한 회귀.** 실제 fresh/stale miss와 합성 KIS 실패에서 `getPriceQuote`는 정상적으로 `(price=null, PRICE_FETCH_FAILED)`를 반환합니다. 그러나 [RealtimePriceService](../backend/src/main/java/com/qaima/service/marketmetric/RealtimePriceService.java)의 `getPrice`가 먼저 `map(PriceQuoteResult::price)`를 적용해 `NullPointerException`을 만들며 `flatMap(Mono::justOrEmpty)`까지 도달하지 못합니다. 정상적인 가격 없음이 내부 오류로 바뀌는 service 경계입니다. 현재 호출자인 [FinancialReadService](../backend/src/main/java/com/qaima/service/financial/FinancialReadService.java)는 소스 확인상 이 오류를 empty로 흡수합니다. 이 신규 검사에서 실제 재무 HTTP를 실행하거나 사용자 응답500을 확인한 것은 아닙니다.

세 회귀의 기대 단언을 유지하고 실패를 보존합니다. 제품 수정은 하지 않습니다.

## 남은 범위

뉴스 목록·refresh marker·본문·focus·sentiment, Feature3에서 사용하는 Feature1 metrics fresh/stale 캐시는 이18개에 포함하지 않습니다. 분석 orchestration의 비용 미리보기/실제 차감·환불과 캐시의 관계도 별도입니다. Redis 네트워크 단절·timeout·재시작·재접속, 실제 DB/제공자와의 결합, 다중 backend 인스턴스 간 동시성은 남아 있습니다. Spring Boot 실제 배포 설정이나 전체 Redis 정책 완료 판정이 아닙니다.

후속 [뉴스/지표 Redis21개 검사](isolated-news-overlay-redis.md)에서 해당7개 키와 실제 비용 preview/estimate, 뉴스 block timeout을 추가했습니다. 위18개 실행 결과와는 별도이며 실제 차감/환불 원장 및 TCP 단절 검증은 계속 남아 있습니다.
