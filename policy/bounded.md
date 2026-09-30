# WebFlux, Blocking I/O, boundedElastic 정책

작성 기준: 2026-05-19  
적용 범위: `backend`, `analysis`, `batch`

---

## 1. 문서 목적

이 문서는 QAIMA 프로젝트에서 동기 처리와 비동기 처리의 경계를 정의한다.

현재 백엔드는 Spring WebFlux를 사용하지만, 데이터베이스 접근은 JPA/Hibernate 기반이다. 따라서 HTTP 요청/응답 흐름은 `Mono`/`Flux`로 구성하되, JDBC, JPA, 파일 I/O, 동기 파서, 일부 블로킹 외부 호출은 Reactor 이벤트 루프에서 직접 실행하면 안 된다.

핵심 원칙은 다음과 같다.

| 구분 | 현재 정책 |
| --- | --- |
| WebClient 외부 API 호출 | 논블로킹 체인으로 유지 |
| JPA Repository 호출 | `boundedElastic`으로 분리 |
| 파일, 파서, 대용량 import | `boundedElastic` 또는 별도 배치 프로세스에서 실행 |
| 스케줄러 내부 배치 | 동기 배치로 취급 가능, `.block()` 허용 |
| 사용자 요청 처리 중 `.block()` | 원칙적으로 금지 |
| 트랜잭션이 필요한 여러 DB 작업 | 하나의 블로킹 작업 단위로 묶거나 `TransactionTemplate` 사용 |

---

## 2. 현재 프로젝트 구조

### 2.1 Backend

`backend`는 Spring Boot 3.2.5, Java 17 기반이다.

주요 의존성:

- `spring-boot-starter-webflux`
- `spring-boot-starter-data-jpa`
- `spring-boot-starter-data-redis-reactive`
- `spring-boot-starter-security`
- `spring-boot-starter-oauth2-client`
- `spring-boot-starter-mail`
- `flyway-mysql`
- `mysql-connector-j`

애플리케이션 진입점은 `QaimaApplication`이며 `@EnableScheduling`, `@EnableAsync`가 활성화되어 있다.

### 2.2 Analysis

`analysis`는 FastAPI 기반 분석 서버다.

역할:

- Feature1 분석 응답 생성
- Feature2 explain 생성
- Feature3 포트폴리오 분석
- LLM client 호출
- 일부 가격/뉴스/포트폴리오 계산

Spring 백엔드는 `AnalysisApiClient`를 통해 FastAPI를 호출한다. 이 호출은 WebClient 기반 논블로킹 호출로 취급한다.

### 2.3 Batch

`batch/jobs`는 Python 기반 독립 실행 배치 스크립트다.

주요 작업:

- `bootstrap_stock_from_kis.py`
- `load_price_ohlcv.py`
- `load_index_ohlcv.py`
- `load_market_investor_flow.py`
- `load_stock_investor_flow.py`
- `import_financials.py`
- `import_short_selling.py`
- `apply_stock_industry_manual_mapping.py`

이 스크립트들은 WebFlux 요청 처리와 분리된 동기 배치 작업이다. Python 배치 내부에서는 DB 커넥션, HTTP 요청, 파일 처리 모두 동기 방식으로 실행되어도 된다.

---

## 3. 계층별 책임

| 계층 | 패키지 | 책임 | 동기/비동기 기준 |
| --- | --- | --- | --- |
| API Controller | `com.qaima.api` | HTTP 요청 수신, DTO 반환 | 대부분 `Mono<ApiResponse<...>>` 반환 |
| Service | `com.qaima.service` | 비즈니스 흐름 조합 | Reactor 체인 구성, 블로킹 작업 분리 |
| Repository | `com.qaima.repository` | JPA DB 접근 | 항상 블로킹 |
| Domain | `com.qaima.domain` | JPA Entity | 외부 응답으로 직접 노출 금지 |
| DTO | `com.qaima.dto` | API 요청/응답 계약 | 순수 변환은 일반 `map` 가능 |
| External | `com.qaima.external` | 외부 API client | WebClient는 논블로킹, 일부 예외는 명시 |
| Config | `com.qaima.config` | WebClient, Redis, Security 설정 | Bean 구성 |
| Common | `com.qaima.common` | 공통 응답, 예외, blocking helper | `Blocking.call`, `Blocking.run` 제공 |

---

## 4. 동기/비동기 경계

### 4.1 논블로킹으로 유지해야 하는 작업

다음 작업은 Reactor 이벤트 루프에서 논블로킹 체인으로 유지한다.

- WebClient 기반 외부 API 호출
- `ReactiveRedisTemplate`을 그대로 반환 체인에서 사용하는 캐시 조회/저장
- `Mono.zip`, `Mono.when`, `Flux.fromIterable` 기반 조합
- DTO 조립, 간단한 계산, 필터링, 정렬
- FastAPI 분석 서버 호출

대표 예:

- `AnalysisApiClient`
- `StockApiClient`
- `KrStockClient`
- `GlobalStockClient`
- `IndustryIndexFetcher`
- `PeerClusterClient`
- `BaseRateSyncService`
- `FredBaseRateSyncService`
- `ExchangeRateSyncService`
- `BondYieldSyncService`

### 4.2 boundedElastic으로 분리해야 하는 작업

다음 작업은 블로킹으로 취급하고 `boundedElastic`으로 분리한다.

- JPA Repository `find`, `save`, `saveAll`, `delete`
- `TransactionTemplate`으로 감싼 DB 작업
- JDBC/MySQL 기반 import
- 파일 읽기/쓰기
- ZIP/XML/JSON 대용량 파싱
- 동기 parser 호출
- Reactive API를 내부에서 `.block()`으로 호출하는 레거시/혼합 코드

현재 프로젝트에는 공통 헬퍼가 있다.

```java
public final class Blocking {
    public static <T> Mono<T> call(Callable<T> c) {
        return Mono.fromCallable(c)
                .subscribeOn(Schedulers.boundedElastic());
    }

    public static Mono<Void> run(Runnable r) {
        return Mono.fromRunnable(r)
                .subscribeOn(Schedulers.boundedElastic())
                .then();
    }
}
```

새 코드는 가능하면 `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`를 반복하지 말고 `Blocking.call(...)` 또는 `Blocking.run(...)`을 우선 사용한다.

---

## 5. boundedElastic 사용 규칙

### 5.1 Repository 호출

JPA Repository는 JDBC를 사용하므로 모두 블로킹이다.

허용 패턴:

```java
return Blocking.call(() -> repository.findById(id));
```

또는 기존 코드와 같이:

```java
return Mono.fromCallable(() -> repository.findById(id))
        .subscribeOn(Schedulers.boundedElastic());
```

금지 패턴:

```java
return Mono.just(repository.findById(id));
```

### 5.2 저장과 조회를 함께 수행하는 작업

조회 후 저장까지 하나의 DB 작업으로 묶어야 하면, 블로킹 lambda 안에서 처리한다.

```java
return Blocking.call(() -> {
    Stock stock = stockRepository.findById(id).orElseThrow();
    stock.setCompanyName(name);
    return stockRepository.save(stock);
});
```

트랜잭션이 필요한 경우 `TransactionTemplate`을 함께 사용한다.

```java
return Blocking.call(() -> transactionTemplate.execute(status -> {
    // 여러 JPA 작업
    return result;
}));
```

현재 이 패턴은 `CreditService`, `FinancialService`, `FinancialAdminService`, `MarketSnapshotService` 등에서 사용한다.

### 5.3 WebClient 호출

WebClient 호출에는 `boundedElastic`을 붙이지 않는다.

허용 패턴:

```java
return webClient.get()
        .uri(uri)
        .retrieve()
        .bodyToMono(ResponseDto.class);
```

금지 패턴:

```java
return webClient.get()
        .retrieve()
        .bodyToMono(ResponseDto.class)
        .subscribeOn(Schedulers.boundedElastic());
```

### 5.4 DTO 변환과 순수 계산

순수 변환은 `map`에서 처리한다.

```java
return service.load()
        .map(this::toResponseDto);
```

다만 계산량이 크거나 외부 라이브러리 파서가 동기적으로 오래 도는 경우에는 블로킹 작업으로 분리한다.

---

## 6. @Transactional 정책

### 6.1 Mono 반환 메서드의 `@Transactional` 한계

`@Transactional`은 프록시 메서드가 실행되는 동안 트랜잭션을 연다. 그러나 `Mono`는 실제 작업을 구독 시점까지 지연한다.

따라서 아래 형태는 트랜잭션 경계가 실제 Repository 실행을 감싸지 못할 수 있다.

```java
@Transactional
public Mono<Result> load() {
    return Mono.fromCallable(() -> repository.save(entity))
            .subscribeOn(Schedulers.boundedElastic());
}
```

메서드는 `Mono`를 반환하고 끝나며, 실제 `save`는 나중에 다른 스레드에서 실행된다.

### 6.2 현재 권장 방식

여러 DB 작업을 하나의 트랜잭션으로 묶어야 하면 다음 중 하나를 사용한다.

1. 동기 메서드 안에서 DB 작업을 끝내고 그 메서드를 `Blocking.call`로 감싼다.
2. `TransactionTemplate`을 `Blocking.call` 안에서 실행한다.
3. 단순 단건 조회/저장은 트랜잭션 요구가 약하면 Repository 기본 트랜잭션에 맡긴다.

현재 코드에서 `TransactionTemplate` 기반 원자 작업을 사용하는 영역:

- 크레딧 차감/충전: `CreditService`
- 재무 데이터 생성/수정: `FinancialService`, `FinancialAdminService`
- 시장 스냅샷 저장: `MarketSnapshotService`

### 6.3 `@Async`와 `@Transactional`

`@Async` 메서드는 별도 스레드에서 실행된다. 이 경우 메서드 자체가 동기적으로 DB 작업을 수행하므로 `@Transactional`이 의미를 가진다.

현재 예:

- `NewsSentimentObservationAsyncService.saveObservation(...)`

---

## 7. .block() 사용 정책

### 7.1 사용자 요청 처리 경로

사용자 HTTP 요청을 처리하는 Controller/Service 체인에서는 `.block()`을 사용하지 않는다.

이유:

- WebFlux 이벤트 루프를 막을 수 있다.
- 요청 처리 스레드가 고갈될 수 있다.
- timeout, backpressure, cancellation 전파가 깨질 수 있다.

### 7.2 스케줄러와 서버 내부 배치

`@Scheduled` 메서드는 Spring scheduler thread에서 실행되는 서버 내부 배치다. 이 경로에서는 명시적으로 동기 배치로 취급하고 `.block()` 사용을 허용한다.

현재 `.block()`을 사용하는 스케줄러/배치 예:

- `KisMarketDataSyncScheduler`
- `IndustryIndexOhlcvSyncService`
- `InvestorFlowBatchSyncService`
- `OpenDartSyncScheduler`
- `SecIssuedSharesSyncScheduler`
- `FinraShortSellingSyncScheduler`

스케줄러는 다음 보완 규칙을 따른다.

- 중복 실행 방지 플래그를 둔다.
- 실패는 job 전체를 죽이지 않고 로그와 summary로 남긴다.
- 외부 API rate limit을 고려해 sleep, retry, limit을 둔다.
- cron zone은 KST 기준으로 명시한다.

현재 `KisMarketDataSyncScheduler`는 `AtomicBoolean`으로 이전 실행 중복을 방지한다.

### 7.3 예외적인 동기 외부 호출

일부 external client는 내부에서 `.block()`을 사용한다.

- `NaverNewsClient`
- `NewsArticleExtractorClient`

이 client를 호출하는 상위 서비스는 블로킹 경로로 격리해야 한다. 현재 `NewsSentimentService`는 뉴스 로딩 진입점을 `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`로 감싼다.

---

## 8. Redis 사용 정책

현재 Redis는 `spring-boot-starter-data-redis-reactive`와 `ReactiveRedisTemplate`을 사용한다.

원칙:

- Reactive chain에서 Redis를 직접 사용할 때는 `.block()` 금지
- JPA/동기 파서와 함께 하나의 블로킹 함수 안에서 처리하는 레거시 경로는 `boundedElastic`으로 감싼다

현재 혼합 경로:

- `NewsSentimentService`는 내부 블로킹 로직에서 Redis reactive call을 timeout을 둔 `.block(...)`으로 사용한다.
- 이 서비스의 public entrypoint는 `boundedElastic`으로 격리되어 있으므로 이벤트 루프 직접 차단은 피한다.

신규 코드에서는 Redis만 사용하는 경우 `.block()` 대신 `ReactiveRedisTemplate`의 `Mono`를 그대로 반환한다.

---

## 9. 현재 기능별 흐름

### 9.1 Feature1

주요 클래스:

- `FeatOneController`
- `FeatOneService`
- `StockService`
- `MarketSnapshotService`
- `AnalysisApiClient`

현재 흐름:

1. Controller가 `Mono<ApiResponse<...>>`를 반환한다.
2. `StockService.getOrCreateStockByCode(...)`가 종목을 조회한다.
3. JPA 조회는 `boundedElastic`에서 실행한다.
4. 캔들은 DB에서 먼저 조회한다.
5. 최신 거래일 데이터가 부족하면 `StockClient.fetchCandles(...)`로 외부 API를 호출한다.
6. 외부 캔들은 누락분만 `saveAll`한다.
7. 재무 데이터와 시장 스냅샷을 조회한다.
8. `AnalysisApiClient`로 FastAPI 분석 서버를 호출한다.
9. FastAPI 실패 시 fallback response를 구성한다.

현재 반영된 변경점:

- 과거 문서의 “DB -> 없으면 KIS -> 실패하면 Marketstack” 표현은 단순화된 설명이다.
- 현재 외부 provider 라우팅은 `StockClient`/`StockApiClient`가 담당한다.
- `StockService.getOrCreateStockByCode(...)`는 현재 미등록 종목을 자동 생성하지 않고, DB에 없으면 `ResourceNotFoundException` 경로로 간다.
- 캔들 외부 조회 여부는 “DB가 비어 있음”뿐 아니라 최신 거래일 누락 여부도 본다.

### 9.2 Feature2

주요 클래스:

- `Feature2AnalyzeController`
- `Feature2AnalyzeService`
- `Feature2CardService`
- `Feature2StockResolver`
- `Feature2IndustryReader`
- `IndustryIndexService`
- `PeerClusterServiceImpl`
- `NewsSentimentService`
- `AnalysisApiClient`

현재 흐름:

1. 요청을 `Feature2RequestNormalizer`로 정규화한다.
2. 종목을 resolve한다.
3. 기준금리, 매크로 금리, 공매도, 투자자 수급을 조합한다.
4. 업종을 resolve한다.
5. 업종지수, peer cluster, trend summary, 뉴스 sentiment를 붙인다.
6. 필요한 경우 FastAPI explain을 호출한다.
7. 실패 가능한 보조 데이터는 warning으로 내려보내고 가능한 응답을 유지한다.

동기/비동기 경계:

- Repository 조회는 `Blocking.call` 또는 `boundedElastic` 사용
- macro card 조합은 `Mono.zip`
- trend summary의 DB 조회는 `boundedElastic`
- FastAPI explain 호출은 WebClient 논블로킹
- 뉴스 로딩은 내부적으로 동기 처리와 Redis block이 섞여 있어 entrypoint에서 `boundedElastic` 격리

### 9.3 Feature3

주요 클래스:

- `Feature3AnalyzeController`
- `Feature3PriceSeriesService`
- `Feature3BenchmarkSeriesService`
- `Feature3OverlayService`
- `YahooFeature3PriceProvider`
- `AnalysisApiClient`

현재 흐름:

1. 보유 종목 요청을 검증한다.
2. 종목별 가격 series를 로딩한다.
3. benchmark series를 로딩한다.
4. overlay signal과 risk-free rate를 조합한다.
5. FastAPI 포트폴리오 분석을 호출한다.

동기/비동기 경계:

- 여러 종목/벤치마크 로딩은 `Flux.fromIterable`과 `Mono.zip`으로 조합한다.
- JPA 조회 및 저장은 `boundedElastic`에서 수행한다.
- Yahoo/외부 가격 provider와 FastAPI 호출은 논블로킹 체인으로 유지한다.

### 9.4 인증, 사용자, 포트폴리오, 관심목록

주요 클래스:

- `AuthService`
- `LoginSessionService`
- `OAuth2SocialLoginService`
- `UserService`
- `PortfolioService`
- `WatchlistService`
- `CreditService`

현재 정책:

- Controller는 `Mono<ApiResponse<...>>`를 반환한다.
- 사용자/세션/관심목록/포트폴리오 Repository 호출은 `Blocking.call`로 감싼다.
- 크레딧 차감/충전처럼 원자성이 필요한 작업은 `TransactionTemplate`을 `Blocking.call` 안에서 실행한다.
- 이메일 인증 cleanup은 `@Scheduled` + `@Transactional` 동기 작업이다.

### 9.5 시장 데이터와 캐시

주요 클래스:

- `CandleLoadService`
- `IndustryIndexReaderImpl`
- `PriceSnapshotReader`
- `TopRankingReader`
- `MarketMetricService`
- `MarketSnapshotCacheService`

현재 정책:

- DB 우선 조회 후 부족하면 외부 API fetch
- 조회/저장은 `boundedElastic`
- Reactive Redis cache는 가능하면 `Mono` 체인 유지
- DB fallback 또는 snapshot 계산이 들어가면 `Blocking.call`로 격리

### 9.6 서버 내부 스케줄러

현재 활성 스케줄러:

| 클래스 | 기본 cron/주기 | 역할 |
| --- | --- | --- |
| `KisMarketDataSyncScheduler` | 평일 KST 18:10 / 16:30 / 19:00 | KIS 업종지수 OHLCV, 시장/종목 투자자 수급 |
| `OpenDartSyncScheduler` | 매일 KST 03:10 | DART 일일 동기화 |
| `SecIssuedSharesSyncScheduler` | 매일 KST 18:00 | SEC issued shares 동기화 |
| `FinraShortSellingSyncScheduler` | 매일 KST 09:00 | FINRA short selling 동기화 |
| `MailAuthService` cleanup | 기본 1시간 간격 | 이메일 인증 만료 cleanup |

스케줄러 내부 `.block()`은 허용한다. 단, 사용자 요청 경로로 해당 코드를 그대로 가져오면 안 된다.

---

## 10. Batch 정책

Python `batch/jobs`는 Spring WebFlux와 별개인 운영 배치다.

정책:

- 동기 DB 커넥션 사용 가능
- 동기 HTTP 요청 사용 가능
- 파일 I/O 사용 가능
- commit/rollback을 명시한다
- 대량 작업은 batch size, sleep, retry, limit을 둔다
- EC2/로컬 DB 차이를 고려해 auto-increment PK보다 안정적인 business key를 우선한다

업종 수동 매핑 같은 작업에서는 `industry_id`만 믿지 말고 다음 조합을 우선 사용한다.

```text
exchange_code + target_sector_code + target_industry_code
```

---

## 11. 신규 코드 체크리스트

### 11.1 Controller

- `Mono<ApiResponse<T>>` 또는 `Mono<T>`를 반환한다.
- Controller에서 Repository를 직접 호출하지 않는다.
- Controller에서 `.block()`을 사용하지 않는다.
- 검증과 파라미터 정규화만 수행하고 비즈니스 판단은 Service로 넘긴다.

### 11.2 Service

- JPA 호출은 `Blocking.call` 또는 `boundedElastic`으로 격리한다.
- WebClient 호출에는 `boundedElastic`을 붙이지 않는다.
- 여러 독립 데이터는 `Mono.zip` 또는 `Mono.when`으로 조합한다.
- 실패 가능한 보조 데이터는 필요한 경우 warning으로 남기고 부분 응답으로 낮춰 처리한다.
- 원자성이 필요한 DB 변경은 `TransactionTemplate` 또는 동기 트랜잭션 메서드로 묶는다.

### 11.3 Repository

- JPA Repository는 블로킹으로 간주한다.
- Repository 결과 Entity를 API 응답으로 직접 반환하지 않는다.
- Lazy loading이 필요한 필드는 query method, fetch join, DTO 변환 시점으로 통제한다.

### 11.4 External Client

- WebClient 기반 client는 `Mono`/`Flux`를 그대로 반환한다.
- `.block()`이 필요한 예외 client는 클래스/메서드 주석 또는 상위 서비스 정책으로 명시한다.
- 외부 API timeout, retry, fallback은 호출 계층에서 의도를 드러낸다.

### 11.5 Scheduler

- `@Scheduled` 메서드는 중복 실행 방지 장치를 둔다.
- 내부 `.block()` 사용은 허용하지만 스케줄러 경로로 한정한다.
- 실패 target은 summary와 로그에 남긴다.
- 운영 속성으로 enabled, cron, limit, sleep, retry를 조절할 수 있게 한다.

---

## 12. 현재 예외와 개선 후보

| 항목 | 현재 상태 | 개선 방향 |
| --- | --- | --- |
| `FeatOneService`의 클래스 레벨 `@Transactional(readOnly = true)` | `Mono` 반환 구조에서는 실제 DB 실행을 안정적으로 감싸지 못함 | 제거하거나 동기 트랜잭션 경계로 재구성 |
| `StockService.loadOrCreateStockMono(...)` | 생성 경로 코드가 남아 있으나 현재 public 경로에서는 미사용에 가까움 | 정책에 맞춰 제거 또는 명시적 admin bootstrap 경로로 분리 |
| External client 내부 `.block()` | `NaverNewsClient`, `NewsArticleExtractorClient`에 존재 | 상위 서비스에서 계속 boundedElastic 격리, 장기적으로 WebClient chain으로 전환 |
| `NewsSentimentService` Redis `.block(timeout)` | public entrypoint가 boundedElastic이라 이벤트 루프 차단은 피함 | Redis 접근을 reactive chain으로 분리하면 구조 단순화 가능 |
| `@Async` 사용 범위 | 현재 뉴스 sentiment observation 저장에 사용 | executor 설정, 실패 로깅, queue 정책 명시 검토 |
| `@EnableAsync` | 활성화되어 있으나 async executor 커스텀 설정은 확인되지 않음 | 운영 부하에 맞는 executor 설정 추가 검토 |

---

## 13. 최종 요약

| 작업 종류 | 실행 위치 | 정책 |
| --- | --- | --- |
| HTTP 요청/응답 | WebFlux | `Mono`/`Flux` 유지 |
| JPA 조회/저장 | boundedElastic | `Blocking.call` 우선 |
| 여러 JPA 작업의 원자 처리 | boundedElastic + transaction | `TransactionTemplate` 권장 |
| WebClient 외부 API | Netty event loop | `boundedElastic` 금지 |
| FastAPI 분석 호출 | WebClient | 논블로킹 유지 |
| DTO 변환 | Reactor chain | 일반 `map` |
| 파일/대용량 파싱 | boundedElastic 또는 batch | 이벤트 루프 금지 |
| Spring scheduler batch | scheduler thread | `.block()` 허용 |
| Python batch | 별도 프로세스 | 동기 처리 허용 |
| 사용자 요청 중 `.block()` | 금지 | 스케줄러/격리 경로 예외만 허용 |
