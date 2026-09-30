# Redis Cache Policy

## 적용 범위

이 문서는 현재 backend 애플리케이션에 적용된 전체 Redis 캐시 정책을 정리한다. 기준 코드는 `backend/src/main/java/com/qaima`이며, Redis 의존성은 `spring-boot-starter-data-redis-reactive`를 사용한다.

- 설정: `backend/src/main/resources/application.yml`
- Redis host/port: `localhost:6380`
- 공통 설정: `RedisConfig`
- 사용 자료구조: String value only (`opsForValue()`)
- 삭제/scan/hash/list/set/zset/stream/lock 사용처: 현재 코드 기준 없음
- 랭킹 캐시 상세 정책: `.agents/policy_ranking_redis_cache.md`

## 공통 원칙

- 모든 Redis key는 서비스별 prefix로 시작한다.
- 신규 Redis key는 `{domain}:{resource}:{qualifier...}` 형태의 colon-separated naming을 우선한다.
- key segment에는 DB id, 종목코드, 날짜, 주기, window, limit 등 조회 결과를 결정하는 입력을 모두 포함한다.
- 외부 API 또는 DB 조회 실패 시 Redis 실패가 사용자 요청 실패로 전파되지 않도록 fallback을 둔다.
- Redis read/write 실패는 warning log 또는 warning code로 남기고 원천 조회, DB 조회, stale cache, 빈 결과 중 서비스별 fallback으로 진행한다.
- TTL 없는 영구 key는 만들지 않는다.
- 캐시 payload schema를 바꾸는 경우 key prefix 또는 version segment를 올린다.

## Redis 직렬화 정책

### `ReactiveStringRedisTemplate`

JSON 문자열을 직접 저장한다.

- `TopRankingReader`
- `RealtimePriceService`
- `MarketSnapshotCacheService`
- `PeerClusterServiceImpl`
- `NewsSentimentService`
- `Feature3OverlayService`

### Typed `ReactiveRedisTemplate`

`RedisConfig`에서 key/hash key는 `StringRedisSerializer`, value/hash value는 `Jackson2JsonRedisSerializer`로 설정한다.

- `ReactiveRedisTemplate<String, IndustryIndexBlockDto>`
- `ReactiveRedisTemplate<String, PriceSnapshot>`

공통 `redisObjectMapper`는 Java time module을 등록하고 timestamp serialization, timezone adjustment, unknown property failure를 비활성화한다.

## Key/TTL 전체 목록

| 영역 | Service | Key format | TTL | Payload | 비고 |
|---|---|---|---:|---|---|
| 특징주 랭킹 | `TopRankingReader` | `ranking-stocks:{yyyyMMdd}:{topic}:{limit}` | 장중 10초, 장외/비거래일은 다음 KRX 거래일 08:25까지 | `List<FeaturedStockDto>` JSON | KRX 거래일 기준 날짜 사용 |
| 가격 스냅샷 | `PriceSnapshotReader` | `price:snapshot:{stockCode}:{freq}` | 120초 | `PriceSnapshot` typed JSON | DB 최신 2개 봉 기반 |
| 업종지수 | `IndustryIndexReaderImpl` | `industry:index:{indexCode}:{freq}:{window}` | 30분 | `IndustryIndexBlockDto` typed JSON | DB 부족 시 KIS fetch 후 저장 |
| 실시간 가격 fresh | `RealtimePriceService` | `price:{stockCode}` | 20초 | `PriceCacheEntry` JSON | KIS 현재가 |
| 실시간 가격 stale | `RealtimePriceService` | `price_stale:{stockCode}` | 20분 | `PriceCacheEntry` JSON | fresh/API 실패 시 fallback |
| 시장 스냅샷 일반 | `MarketSnapshotCacheService` | `snapshot:{stockCode}` | 1시간 | `SnapshotCacheEntry` JSON | `asOfDate` 조건 검증 |
| 시장 스냅샷 최신 | `MarketSnapshotCacheService` | `snapshot_latest:{stockCode}` | 60초 | `SnapshotCacheEntry` JSON | 최신 조회 shortcut |
| Peer cluster | `PeerClusterServiceImpl` | `feature2:peercluster:v6:{industryId}:{anchorStockCode}:{freq}:{window}:{from}:{to}:{peerCount}:{maxLag}:{displayLimit}` | 12시간 | `CachedPeerCluster` JSON | payload 변경 시 version segment 증가 |
| 뉴스 목록 | `NewsSentimentService` | `feat2:news:list:stock:{stockCode}` | 5분 | `CachedNewsListPayload` JSON | 종목별 표시 목록 |
| 뉴스 refresh marker | `NewsSentimentService` | `feat2:news:refresh:stock:{stockCode}` | 15분 | `CachedNewsRefreshPayload` JSON | Naver refresh 간격 제어 |
| 뉴스 본문 상세 | `NewsSentimentService` | `feat2:news:detail:news:{newsId}` | 7일 | `CachedNewsDetail` JSON | 기사 본문/문단/요약 |
| 뉴스 focus text | `NewsSentimentService` | `feat2:news:focus:news:{newsId}` | 7일 | `CachedFocusTextValue` JSON | 감성 분석 입력 텍스트 |
| 뉴스 sentiment | `NewsSentimentService` | `feat2:news:sentiment:news:{newsId}` | 7일 | `CachedSentimentValue` JSON | model/prompt/focus version 포함 |
| Feature1 metrics fresh | `Feature3OverlayService` | `feature1:metrics:fresh:{normalizedStockCode}` | 24시간 | `CachedFeature1MetricsPayload` JSON | Feature3 overlay credit preview/재사용 |
| Feature1 metrics stale | `Feature3OverlayService` | `feature1:metrics:stale:{normalizedStockCode}` | 7일 | `CachedFeature1MetricsPayload` JSON | 현재 코드는 write만 수행 |

## 서비스별 정책

### 특징주 랭킹

- Reader: `TopRankingReader`
- 외부 원천: KIS 국내주식 랭킹 API
- key: `ranking-stocks:{yyyyMMdd}:{topic}:{limit}`
- 날짜 기준: KRX 거래일
- 거래일 `08:25` 이전: 직전 거래일 key
- 거래일 `08:25` 이상: 당일 key
- 거래일 `08:25` 이상 `08:30` 이전: 캐시 비활성화
- 거래일 `08:30` 이상 `18:00` 이전: TTL 10초
- 거래일 장외/비거래일: 다음 거래일 `08:25`까지 TTL

상세 정책은 `.agents/policy_ranking_redis_cache.md`를 따른다. 해당 파일과 충돌이 생기면 랭킹 상세 문서를 먼저 갱신한 뒤 이 문서의 요약 표를 맞춘다.

### 가격 스냅샷

- Reader: `PriceSnapshotReader`
- key: `price:snapshot:{stockCode}:{freq}`
- TTL: 120초
- payload: `PriceSnapshot`
- cache miss 시 `price_ohlcv`에서 현재 시각 이전 최신 2개 row를 읽어 최신가, 직전가, 등락률을 계산한다.
- 동일 key miss가 동시에 들어오면 local `ConcurrentHashMap<String, Mono<PriceSnapshot>> inflight`로 중복 DB 조회를 줄인다.
- Redis GET/SET 실패 시 DB 결과를 반환하고 캐시 없이 계속 진행한다.

### 업종지수

- Reader: `IndustryIndexReaderImpl`
- key: `industry:index:{indexCode}:{freq}:{window}`
- TTL: 30분
- payload: `IndustryIndexBlockDto`
- cache miss 시 DB에서 최근 `window`개를 조회한다.
- DB row가 부족하면 KIS 업종지수 fetch를 수행하고 DB 저장을 시도한 뒤 block을 만든다.
- 동일 key miss는 local inflight map으로 중복 fetch를 줄인다.
- Redis 실패는 DB/fetch fallback으로 처리한다.

### 실시간 가격

- Service: `RealtimePriceService`
- fresh key: `price:{stockCode}`, TTL 20초
- stale key: `price_stale:{stockCode}`, TTL 20분
- payload: `PriceCacheEntry`
- fresh hit이면 즉시 가격을 반환한다.
- fresh miss이면 KIS 현재가를 조회하고 fresh/stale key를 함께 갱신한다.
- KIS 미지원 시장 또는 fetch/parse 실패 시 stale key를 읽고 `PRICE_STALE_USED` warning을 반환한다.
- stale도 없으면 `PRICE_FETCH_FAILED` warning과 null price를 반환한다.

주의: `price:{stockCode}`는 실시간 현재가이고, `price:snapshot:{stockCode}:{freq}`는 DB 기반 봉 스냅샷이다. prefix 충돌은 없지만 운영 조회 시 두 계열을 구분해야 한다.

### 시장 스냅샷

- Service: `MarketSnapshotCacheService`
- latest key: `snapshot_latest:{stockCode}`, TTL 60초
- snapshot key: `snapshot:{stockCode}`, TTL 1시간
- payload: `SnapshotCacheEntry`
- 조회 순서: latest key -> snapshot key -> DB
- `asOfDate`가 지정된 경우 cache payload의 `asOfDate`가 요청 기준일보다 미래이면 cache miss로 취급한다.
- DB hit 후 두 key를 모두 갱신한다.
- Redis SET 실패 시 DB 결과를 반환하고 캐시 없이 계속 진행한다.

### Peer cluster

- Service: `PeerClusterServiceImpl`
- key: `feature2:peercluster:v6:{industryId}:{anchorStockCode}:{freq}:{window}:{from}:{to}:{peerCount}:{maxLag}:{displayLimit}`
- TTL: 12시간
- payload: `CachedPeerCluster`
- `from`/`to`가 null이면 `-`, 아니면 `Instant.toString()`으로 key segment를 만든다.
- `forceRefresh=true`이면 cache read를 우회하고 FastAPI 결과로 갱신한다.
- cache payload가 현재 `CachedPeerCluster` 구조로 deserialize되지 않으면 legacy `PeerClusterDto` deserialize를 시도한다.
- payload 구조가 의미 있게 바뀌면 `v6` segment를 증가시켜 기존 캐시와 분리한다.

### Feature2 뉴스

- Service: `NewsSentimentService`
- 모든 key의 stockCode segment는 `normalizeCacheSegment`를 적용한다.
- normalization: trim, lowercase, whitespace를 `-`로 변경, 문자/숫자/hyphen 외 문자를 `-`로 변경, 연속 hyphen 축약, 선후행 hyphen 제거

뉴스 목록:

- key: `feat2:news:list:stock:{stockCode}`
- TTL: 5분
- payload: `CachedNewsListPayload`
- 화면 표시용 종목별 뉴스 목록 캐시다.

뉴스 refresh marker:

- key: `feat2:news:refresh:stock:{stockCode}`
- TTL: 15분
- payload: `CachedNewsRefreshPayload`
- Naver 뉴스 refresh 빈도를 줄이기 위한 종목별 marker다.

뉴스 상세:

- key: `feat2:news:detail:news:{newsId}`
- TTL: 7일
- payload: `CachedNewsDetail`
- 기사 본문, 문단, reader summary, extraction meta를 저장한다.

뉴스 focus text:

- key: `feat2:news:focus:news:{newsId}`
- TTL: 7일
- payload: `CachedFocusTextValue`
- 감성 분석 입력으로 사용할 focus text와 version을 저장한다.

뉴스 sentiment:

- key: `feat2:news:sentiment:news:{newsId}`
- TTL: 7일
- payload: `CachedSentimentValue`
- score, analyzedAt, modelVersion, promptVersion, focusTextVersion을 저장한다.

### Feature3 overlay의 Feature1 metrics

- Service: `Feature3OverlayService`
- fresh key: `feature1:metrics:fresh:{normalizedStockCode}`, TTL 24시간
- stale key: `feature1:metrics:stale:{normalizedStockCode}`, TTL 7일
- payload: `CachedFeature1MetricsPayload`
- stockCode normalization: null이면 `-`, trim/lowercase 후 `[a-z0-9_-]` 외 문자를 `_`로 변경
- preview/overlay 판단은 fresh key를 읽는다.
- write 시 fresh와 stale을 함께 저장한다.
- 현재 코드 기준 stale key는 write만 있고 read fallback은 없다. stale fallback을 도입할 경우 `feature1:metrics:stale:*`의 TTL 의미와 credit/cacheStatus 정책을 함께 갱신해야 한다.

## 무효화/갱신 정책

- 현재 구현에는 Redis delete 기반 명시적 무효화가 없다.
- 모든 key는 TTL 만료 또는 version segment 변경으로 교체된다.
- 강제 갱신이 필요한 흐름은 `forceRefresh` 또는 cache policy로 read 우회를 수행하고 같은 key에 새 값을 덮어쓴다.
- 운영자가 수동 삭제할 때는 prefix 단위로만 삭제하고, `KEYS`는 운영 Redis에서 사용하지 않는다. 필요 시 `SCAN` 기반으로 제한 삭제한다.

## 장애 처리 정책

- Redis 장애는 기본적으로 degraded cache miss로 처리한다.
- read 실패: 원천 API, DB, stale key, legacy deserialize, 빈 결과 등 서비스별 fallback으로 진행한다.
- write 실패: 원 요청 결과를 반환하고 캐시 저장만 포기한다.
- 사용자 응답에 warning을 노출하는 서비스는 기존 warning code를 유지한다.
- Redis timeout은 application 설정의 `spring.data.redis.timeout: 2s`를 따른다.

## 변경 시 확인 사항

- Redis 사용처 추가/변경 시 이 문서의 Key/TTL 전체 목록을 갱신한다.
- TTL 변경 시 원천 데이터 갱신 주기, 사용자 화면 freshness, 외부 API rate limit을 함께 검토한다.
- payload class 필드 변경 시 backward compatibility를 확인한다.
- backward compatibility가 어렵다면 key prefix 또는 version segment를 올린다.
- 랭킹 정책 변경 시 `.agents/policy_ranking_redis_cache.md`도 함께 갱신한다.
- Java 변경 후 `./gradlew --no-daemon compileJava`를 실행한다.

## 현재 미사용 Redis 패턴

현재 코드 검색 기준 다음 패턴은 사용하지 않는다.

- Redis lock 또는 `setIfAbsent`
- hash/list/set/zset/stream 자료구조
- explicit `expire`
- explicit `delete`/`unlink`
- Spring Cache abstraction (`@Cacheable`, `CacheManager`)
- Redis 기반 refresh token/session 저장

