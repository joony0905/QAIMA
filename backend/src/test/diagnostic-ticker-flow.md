# 상태 확인·OAuth 진입·종목 메타/동기화 검증

검증일: 2026-09-25. Windows Java17.0.12/Gradle8.14. `DiagnosticHttpFlowTest`8개, `TickerMetadataSyncFlowTest`14개, `SyncSecurityFlowTest`3개로9개 Swagger 경로의 실제 handler를 실행합니다. 제품 코드·설정·기존 데이터는 수정하지 않았습니다. 실제 로그인·제공처 호출·DB 저장은 하지 않았습니다.

최종 실행: 진단7 PASS·1 FAIL, 종목 메타/동기화14 PASS, 권한2 PASS·1 FAIL로 추가25개 중23 PASS·2 FAIL입니다. 전체 Windows 회귀는997개 중961 PASS·32 FAIL·4 SKIP, 약43초/BUILD FAILED이며 기존30개 실패 외에 아래2개만 추가됐습니다. XML 합계997/32/0/4(tests/failures/errors/skipped)와25개 메서드의 문서 포함 여부를 별도 대조했습니다.

## 실행과 검증 경계

[공통 Windows 실행 준비](README.md)의 Gradle 캐시·TEMP를 test/.runtime으로 지정하고 backend에서 실행합니다.

```powershell
Remove-Item Env:QAIMA_LIVE_MAIL,Env:QAIMA_PROVISION_ISOLATED,Env:QAIMA_LIVE_READONLY -ErrorAction SilentlyContinue
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*DiagnosticHttpFlowTest' --tests '*TickerMetadataSyncFlowTest' --tests '*SyncSecurityFlowTest'
```

XML은 `.runtime/build/test-results/test/TEST-com.qaima.verification.DiagnosticHttpFlowTest.xml`과 나머지 클래스의 동명파일, HTML은 `.runtime/build/reports/tests/test/index.html`입니다. 재실행이 같은 위치를 갱신하며 최신 전체 집계는 [검증 현황](../../../docs/verification.md)에 기록합니다.

최초25개 실행은6 FAIL이었습니다. 진단 테스트4개는 fixture의 요청 본문을 write 완료 전에 읽은 오류였고 `Mono.defer`로 순서를 수정했습니다. 다음 실행25개/3 FAIL 중1개는 실제 내부 요청이 평면이 아닌 subject/request_context/input_data/options 중첩 구조인데 잘못 기대한 테스트 오류였습니다. 실제 `FeatOneInternalAnalysisRequestDto`와 전송 JSON에 맞춰 수정하고 필수 필드 존재를 강제했습니다. 나머지2개는 아래 내부문구 공개와 동기화 권한 회귀입니다. 테스트 자체의 수정을 제품 결함 해결로 계산하지 않습니다.

실제 경로는 다음과 같습니다.

- 진단: 실제 HelloController/AdminController/AuthOAuth2Controller→TestExternalClient 또는 FastApiAnalysisClient→실제 요청/JSON codec→응답/예외처리. 전송 connector만 합성 응답으로 대체합니다. 내부문구 공개 검사1개만 하위 두 client를 오류 mock으로 교체합니다.
- 메타: 실제 MetaController/StockDebugController→StockApiClient→GlobalStockClient의 요청/JSON/매핑 또는 KIS 경계 mock→DTO/오류처리. Yahoo는 모든 검사에서 접근0입니다. KIS 자체 HTTP/parser는 이 클래스 범위가 아닙니다.
- 동기화: 실제 StockSyncController→StockApiClient→GlobalStockClient→합성 JSON→실제 StockSyncService→메모리 Repository. Exchange는 code, Stock은 exchange code+원천 symbol로 키를 구성해 반복 실행과 저장 요청을 관측합니다. 실제 DB의 유일성/트랜잭션/rollback은 미검증입니다.
- 권한: 제한된 application context에 실제 SecurityConfig·JwtAuthFilter·StockSyncController·AdminController를 등록합니다. 합성 JWT 문자열을 검증/역할로 변환하는 JwtTokenProvider와 OAuth handler/registration은 fixture이며 서명 검증은 이 검사 범위가 아닙니다. 종목 제공처/동기화 서비스는 mock입니다. 따라서 정책 허용 여부와 실제 business handler 진입을 확인하되 외부 호출이나 쓰기는 없습니다.

앞의 두 클래스는 `bindToController` 방식이라 필터 없는 HTTP 처리층 검사입니다. 권한 클래스만 필터와 handler를 결합합니다. context에는 기존 보안 fixture의 sentinel도 있지만 검사 경로의 구체적 Controller mapping이 처리하며 응답/서비스 호출로 이를 확인합니다. 실제 TCP 서버·전체 application boot·scheduler는 실행하지 않습니다. 모든 키·토큰·회사명·주가·오류 문구는 합성값이고 실제 비용 호출은0회입니다.

## 진단·OAuth 8개

| 메서드 | 입력·실행·기대값/관측 |
|---|---|
| `homeHealthAndAdminPayloadsDoNotCallExternalServices` | `/`의 안내문, `/api/v1/test/ping`의 data.ok=true/version=v1·성공 envelope, `/api/v1/admin/ping`의 admin ok 확인. provider 전송0. 필터 없는 결과로 익명 관리자 접근 허용을 주장하지 않음 |
| `supportedOAuthProvidersRedirectLocallyAndNormalizeCase` | GOOGLE/Kakao/naver→302·각 `/oauth2/authorization/{provider}` Location·빈본문, enum 앞뒤 공백도 정규화. 외부 OAuth 로그인/토큰교환은 수행하지 않음 |
| `unsupportedOAuthProviderIs404WithoutRedirectOrExternalCall` | unknown→404/RESOURCE_NOT_FOUND·Location 없음, enum null/공백 미지원·전송0 |
| `externalDiagnosticUsesActualClientButReturnsJsonAsString` | 합성 JSON id1/title fixture→실제 client 요청 `/posts/1`, Public response 필드는 JSON 객체가 아닌 원문 문자열 |
| `externalHttpFailureUsesErrorEnvelopeInsideHttp200Observed` | 합성 외부503→HTTP200 안 EXTERNAL_TEST_FAILED·data=null·meta 실패, 요청1회. 현재 오류 status 정책 관측 |
| `featureOneDiagnosticUsesActualSnakeCodecAndDisablesExplanation` | `/api/v1/test/feature1`→실제 FastApiAnalysisClient POST `/feature1/analysis`; subject.stock_code=AAPL, request_context.include_llm_explain=false, input_data.ohlcv/financials 빈배열, options.freq=ONE_D. 합성 snake 응답→Public metrics.stockCode·경고 복원 |
| `featureOneRemoteStatusAndMalformedPayloadsBecomeDebugErrorEnvelope` | 내부422,200/손상 JSON,data=null,빈metrics 네 가지→각 HTTP200/FASTAPI_ERROR·전송총4회. 실제 FastAPI 계산은 호출하지 않음 |
| `debugErrorsMustNotRevealSyntheticInternalDetails` | 두 하위 client에 합성 내부 상세문구 예외 주입→Public 오류에서 비공개여야 하지만 두 경로 모두 포함. 회귀1메서드/2단언 **FAIL** |

## 종목 메타·동기화 14개

메타 경로는 `/api/v1/meta/tickers`, `/api/debug/ticker-meta`, 동기화는 `POST /api/v1/sync/tickers`입니다. 기본 원천은 QA/Fixture Co/NASDAQ, 가격123.5·등락-2.5이며 합성 HTTP URI와 저장소 호출을 대조합니다.

| 메서드 | 입력·실행·기대값/관측 |
|---|---|
| `overseasMetadataUsesActualGlobalCodecCanonicalSymbolAndMicMapping` | 공백 소문자 qa.us→QA, 실제 요청 `/tickers/QA`/합성 key. acronym=null·MIC XNAS→NASDAQ, source MARKETSTACK·가격/등락 보존·currency=null. KIS/DB 접근0 |
| `koreanMetadataCachesPriceSubscriptionAndPrefersSearchAbbreviation` | 005930.XKRX→005930, KIS price와 search 결과 조립. abbreviation 우선·sector/industry·kospi200=true, price Mono 구독1회·Global 전송0. KOSDAQ이라도 현재 search 요청 구분은 J임을 확인 |
| `searchInfoSoftFailureRetainsPriceMetadataWithoutGlobalTransmission` | search-info KIS_HTTP_ERROR→가격10/Quote Name 유지·industryCode=null·Global 전송0 |
| `koreanPriceFailureFallsBackToGlobalWithKoreanSuffix` | KIS price 오류→Global `/tickers/005930.XKRX` fallback·source MARKETSTACK·search-info 호출0 |
| `emptyMetadataIs404PublicBut200ErrorEnvelopeOnDebugRoute` | 원천 빈 data→공개 meta404/META_NOT_FOUND, debug200/META_NOT_FOUND. 두 Controller의 오류 status 차이와 DB 접근0 |
| `missingSymbolIs400ButBlankSymbolMapsTo500Observed` | symbol 누락은 두 경로 모두400. 공개 meta의 빈 symbol은 provider 전송 없이500/INTERNAL_ERROR. 원천 validation error를 내부오류로 매핑하는 관측 |
| `globalHttpAndJsonErrorsProducePublic500WithoutDatabaseWrites` | Global503,200/손상 JSON→공개 meta500, 저장소 접근0 |
| `tickerSyncCreatesExchangeAndStockAndReusesKeysOnRepeat` | 원천1개→exchange/stock 생성·이름/국가 매핑, 같은 원천 재실행→같은 stock 객체와 키·stock save2회/exchange save1회. 실제SQL 멱등성 증거는 아님 |
| `tickerWithoutExchangeIsSkippedButReportedInputCountIncludesItObserved` | 거래소없는 원천1개→저장소 접근0이나 성공 메시지 건수1. 성공/처리 count가 원천 입력수라는 관측 |
| `repeatedTickerSyncClearsExistingIndustryAndSectorObserved` | 최초 적재 후 합성 stock에 산업/섹터 설정→동일원천 재실행 시 두 필드null. 실제분류DB는 바꾸지 않음 |
| `syncUsesMicOnlyWhenAcronymIsNullAndPreservesRawSymbolObserved` | acronym=null이면 XNAS 사용, 빈문자열이면 MIC가 있어도 빈코드 사용. raw symbol 공백·소문자·접미사도 그대로 저장 요청. meta 경로와 정규화 정책이 다름 |
| `providerAndSaveFailuresReturnSyncErrorAndStopFollowingRows` | Global503→200/SYNC_ERROR·저장소0. 별도 원천2행에서 첫 stock save 실패→SYNC_ERROR·exchange 저장호출은 앞서 완료, 두번째 QB 조회0. 실제 부분 commit 여부는 미검증 |
| `paginationAdvertisedRemainingRowsAreNotFetchedObserved` | 원천 limit1/count1/total3→전송1회·저장1개, 다음 페이지 조회 없음. 전체 시장 동기화 완료라는 증거가 아님 |
| `emptyNullAndMalformedTickerPayloadsAvoidDatabaseWrites` | 원천 `{}`/data=null/data=[]/손상 JSON→각 SYNC_ERROR·저장소 접근0 |

이14개는 현재 동작과 호출 경계를 검사해 통과했습니다. 이름에 Observed가 붙은 항목을 요구사항 충족/올바른 정책으로 단정하지 않습니다. 응답에 보이는 동기화 한글 문구의 인코딩 손상도 코드에서 확인되나 문구 회귀로 별도 측정하지 않았습니다.

## 실제 필터·업무 handler 결합 3개

| 메서드 | 입력·실행·기대값/결과 |
|---|---|
| `anonymousSyncIsRejectedBeforeProviderAndWrites` | 인증없는 동기화 POST→401·제공처/서비스 접근0 |
| `adminSyncReachesBusinessHandlerAndAdminPingEnforcesRoles` | admin ping은 익명401/USER403/ADMIN200·admin ok. ADMIN 동기화→200/success·provider/service 각1회 |
| `ordinaryUserMustNotExecuteDocumentedAdminTickerSync` | USER 동기화→403·provider0·service0 기대. 실제200·두 mock 모두 호출됨: **FAIL**. 응답 status와 두 경계 접근을 독립 단언 |

기존 `ApiSecurityMatrixTest`는 경로별 현재 설정과 sentinel을 기준으로 검사해서 `/api/v1/sync/**`에 일반 인증만 필요한 현재 동작도 PASS였습니다. 이번에는 Controller의 관리자 전용 명시를 요구사항으로 삼고 실제 handler를 연결했으므로 권한 불일치를 발견할 수 있었습니다. 기존369개 PASS를 관리자 업무 전체 보호의 증거로 해석하면 안 됩니다.

## 발견사항·미검증

`SYNC-AUTH-001`: SecurityConfig의 관리자 matcher가 `/api/v1/admin/**`뿐이고 동기화는 `/api/v1/sync/tickers`라 일반 인증으로 통과합니다. `DEBUG-ERROR-001`: 진단 경로의 safeMessage는 예외 메시지 길이만200자로 자르고 내용을 공개합니다. 모두 [발견사항](../../../docs/findings.md)에 근거를 기록하고 실패 회귀를 유지합니다. 서비스/보안 설정은 허용 범위 밖이므로 수정하지 않았습니다.

남은 검증: 실제 JWT/DB 계정으로 전체 요청, 브라우저 OAuth 승인→callback·추가정보·세션, 실제 KIS/Marketstack/진단 제공처/WSL FastAPI TCP, stock sync의 SQL/트랜잭션·중첩 실행·페이지 정책, 전체 시간대/제공처 오류 조합. 아홉 경로는 PARTIAL이고 실제 연동은 NOT_RUN입니다. Swagger 전체123개의 PARTIAL 표기는 전체 흐름 완료가 아니라 명시된 부분에 실행 증거가 생겼다는 뜻입니다.
