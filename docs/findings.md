# 검증 발견사항

작성일: 2026-09-25. 제품 소스는 수정하지 않고 재현 테스트와 근거를 기록합니다.

## FRONT-AUTH-LATE-001 / FRONT-BALANCE-LATE-001 — 로그아웃 정리 후 늦은 응답이 상태 복원

실제 Axios interceptor/tokenStore/billingStore를 Windows Node VM에서 실행하고 전송응답만 보류했습니다. refresh진행중 clearAccessToken 후 응답을해제하면 토큰이복원되고 원요청이재전송됩니다. 잔액GET진행중 clearTokenBalance 후 응답을해제하면0으로정리한잔액이이전응답99로복원됩니다. 각각회귀1개FAIL입니다. Sidebar.handleLogout finally가같은정리함수를사용함을확인했으나실제UI·서버세션/쿠키·사용자전환은미검증이며인증우회확정으로표기하지않습니다. 실제API/크레딧변경0·제품미수정. [20개상세검증](../frontend/tests/auth-client.md).

## BATCH-CSV-ENCODING-001 / BATCH-CSV-SCHEMA-001 / BATCH-CSV-NONFINITE-001

재무·공매도 importer 모두에서 실제 TextIOWrapper/DictReader의 encoding 재시도 시 앞서 yield한 행이 중복됩니다(합성401행→1083행). 필수열없는 파일은 SQL쓰기0이나 호출자에게 오류 없이 정상 반환합니다. 실제 CSV의 NaN/±Infinity 금액·수량도 executemany Decimal매개변수에 전달됩니다. 3메서드/10subcase FAIL, 제품 미수정·실제DB접속0입니다. SQL 오류가 지속되면 batch_size1에서도 유지된 batch 크기가1→2→3으로 커지고 잔여flush에서 오류가 전파되는 별도 관측도 기록했습니다. 실제 영속성·대규모 성능은 미검증입니다. [17개 상세검증](../batch/tests/csv-importer-flows.md).

## BATCH-BOOTSTRAP-EXIT-001 / BATCH-BOOTSTRAP-DATE-001 / BATCH-BOOTSTRAP-ROLLBACK-001

종목 초기화의 실제 main에서 유일 종목 KIS GET이 Timeout이어도 return0입니다(종목 INSERT0, 거래소 seed commit요청1). 상장일 후보는 앞값00000000 때문에 뒤의20200102가 누락되어 None이 됩니다. stock INSERT 오류 시 cursor.close1/commit0이나 rollback0이며 상위 루프는 같은 연결로 다음 종목을 계속합니다. 세 회귀가 FAIL이고, 실제 DB 변경·외부 호출 없이 합성 경계에서 재현했습니다. 실제 transaction 영향과 원천 응답에서의 날짜 필드 조합 빈도는 미검증입니다. [25개 상세검증](../batch/tests/bootstrap-lookup.md).

## BATCH-OVERSEAS-FREQ-001 — 주봉·월봉을 일봉 주기 코드로 저장 요청

해외차트main의period W/M query는정상전달되지만저장은DEFAULT_FREQ_ONE_D=4로고정됩니다. Spring Freq의ORDINAL정의ONE_W=5/ONE_M=6과불일치하며실제SQLtuple을캡처한1메서드/2subcase가FAIL입니다. 다른주기의데이터가일봉으로분류될위험이며실제DB쓰기·기존행변경은없습니다. [23개상세검증](../batch/tests/overseas-tools.md).

## BATCH-OVERSEAS-ROLLBACK-001 / BATCH-OVERSEAS-NONFINITE-001

해외차트Repository의executemany오류에서rollback0회(1회귀FAIL), client의±Infinity종가가유효CandleRow로유지됨(1회귀/2subcaseFAIL)을재현했습니다. 실제DB전송은없습니다. main이연결을닫는점은국내배치의연결재사용과구분하며실제transaction영향은미검증입니다. [재현과범위](../batch/tests/overseas-tools.md).

## BATCH-MAPPING-ENCODING-001 — 파일 뒤쪽 인코딩 실패 시 앞부분 행 중복 반환

수동 산업 매핑의read_mapping_rows는encoding별yield from도중실패하면이미반환한행을남긴채다음encoding으로처음부터재시도합니다. 실제TextIOWrapper/CSV parser에고유ASCII400행+CP949한글1행을주면401행대신973행이반환됐습니다. 행중복방지회귀1개FAIL입니다. 실제main연결검사에서도정상원본이duplicate mapping row오류로중단되고일부합성UPDATE후commit0/rollback0/close1임을관측했습니다. 실제DB는사용하지않았으며제품미수정입니다. dry-run도UPDATE후rollback구조라읽기전용으로간주할수없습니다. [24개상세검증](../batch/tests/manual-mapping.md).

## BATCH-OHLCV-EXIT-001 — 전체 대상 실패에도 가격·산업지수 CLI 성공 반환

두실제main→client→대상루프에합성Timeout을주면유일대상이실패하고commit0인데도return0입니다. main은BootstrapResult.hard_fail을보지않고성공값을반환합니다. 대상조회자체예외는return1인대조군은통과했습니다. CI/CLI종료코드감지가전체대상실패를놓칠수있습니다. 실제process실행이아닌main반환값검증이며전송/DB는mock입니다. 1메서드/2subcase FAIL, 제품미수정. [상세25개검사](../batch/tests/ohlcv-jobs.md).

## BATCH-OHLCV-ROLLBACK-001 — SQL 실행 오류 뒤 rollback 없이 연결 재사용 가능

가격·산업지수Repository의executemany에합성예외를주면cursor.close는호출하지만connection.rollback은0회입니다. 상위대상루프는예외를잡고같은connection으로다음대상을진행합니다. 회귀1메서드/2subcase가실패했습니다. index분할적재는첫chunkcommit후다음chunk실패에서도rollback0인관측을추가했습니다. 실제MySQL부분실행·잠금·후속commit영향은아직미검증이며호출누락만입증합니다. [상세근거](../batch/tests/ohlcv-jobs.md).

## BATCH-OHLCV-NONFINITE-001 — 무한대 종가를 정상 캔들로 수용

가격/산업지수두client의실제JSON→숫자parser에Infinity/-Infinity를입력하면float inf/Decimal Infinity가있는CandleRow가생성됩니다. 유효종가로제외해야한다는1메서드/4subcase가실패했습니다. 일부NaN문자열거부가전체유한성검사를대신하지못합니다. 실제SQL로전송하지않았으며DB저장여부는단정하지않습니다. [재현방법](../batch/tests/ohlcv-jobs.md).

## BATCH-INVESTOR-NONFINITE-001 — Python 수급 배치가 비유한 숫자를 SQL 매개변수로 전달

종목·시장 수급 배치의 `parse_decimal`은 Decimal변환의InvalidOperation만처리하며유한성검사를하지않습니다. 신규2개회귀에서NaN/Infinity/-Infinity를수량에입력하면None으로거르지않고각Decimal값이to_params의SQLtuple에남습니다(2메서드·6실패subcase). 정상유한소수/비수치문자대조군및main→client→JSON→Repository·commit호출은합성경계에서통과했습니다. 실제DB전송·저장결과는미실행이며제품미수정입니다. [30개상세검증](../batch/tests/investor-jobs.md).

## PRICE-SNAPSHOT-EMPTY-001 — 빈 가격 목록이 내부 mapper 오류로 처리됨

`PriceSnapshotReaderFlowTest.emptyRepositoryShouldReturnEmptyWithoutWritingFalseSnapshot`에서 실제 reader에 Repository 빈 목록을 전달하면 `toSnapshot`이null을 반환하고 Reactor map이 NPE를 발생시킵니다. 뒤 flatMap의null 방어에는 도달하지 않습니다. 정상 empty·SET0을 기대하는 회귀1개 FAIL입니다. Repository 자체null 응답 대조군은 정상 empty입니다. 실제 Feature2StockResolver를 연결한 추가 관측에서는 오류가 흡수되어 종목 메타·price null·warnings없음으로 반환됐으므로 Public API 전체500이라고 주장하지 않습니다. 제품 미수정, 실제DB/Redis 미사용. [상세 방법](../backend/src/test/market-cache-readers.md).

## INDUSTRY-CACHE-WARNING-001 — 부분 산업지수 결과를 재사용하면 수집 실패 경고 소실

`IndustryIndexReaderFlowTest.cachedPartialSeriesShouldRetainItsSourceFailureWarning`은3행 요청에DB1행·제공처빈결과를 주어 최초 응답의 `INDUSTRY_INDEX_FETCH_FAILED`를 확인합니다. 실제 RedisConfig 값 codec으로 캐시한 DTO를 다음 요청이 재사용하면 series1행은 유지하지만 새 meta의warnings는 비어 있습니다. 동일 제한 결과에 경고가 유지되어야 한다는 회귀1개 FAIL입니다. 원인은 캐시가 IndustryIndexBlockDto만 담고 meta 경고를 보존하지 않는 경계입니다. 최초 Public HTTP의200·부분series·warning 전달은 별도 PASS이며, 서버 Redis 전송은mock입니다. 제품 미수정. [상세 방법](../backend/src/test/market-cache-readers.md).

## TEST-ENV-GUARD-001 — 테스트 dotenv 차단 변수 오류, 외부 연결은 차단됨

설치된 python-dotenv는 `PYTHON_DOTENV_DISABLED`를 확인하지만 기존 Python 테스트 실행기는 `DOTENV_DISABLED`를 사용했습니다. 신규 Windows→WSL 실제 TCP의 F2 정상 입력에서 app.main의 .env 로딩 후 제공자 연결 시도2회가 발생했고, 테스트의 socket.connect audit hook이 두 시도 모두 연결 전에 차단했습니다. 원격 제공자에 전송된 유료 HTTP 요청은0회이며 실제 자격증명 값은 출력/공개 문서에 기록하지 않았습니다.

이는 제품의 의도된 dotenv 로딩 결함이 아니라 **QA 실행기 오류**입니다. 테스트/문서만 올바른 변수로 고치고 .env 파일 읽기 차단과 앱 import 후 제공자 키 부재 검사도 추가했습니다. 기존 WSL smoke도 같은 차단 wrapper로 재실행해10개 PASS/외부접속·dotenv읽기 시도0, Python 기본98개도 기존 결과 그대로 재확인했습니다. 차단이 발생한11개 실행의5 FAIL 중4개는 누적 audit 확인 실패이며 새로운 계산 결함5개로 집계하지 않습니다. [실행 이력과 범위](../backend/src/test/cross-os-fastapi.md).

## OBS-F2-EMPTY-METRICS-001 — null metrics 조립과 정상 빈 목록의 차이

`Feature2ExplainMetricsAssembler.from(null)` 또는 빈 explain DTO 직접 build는 macroTrendSummaries/recentNews를null로 남깁니다. 실제 Java→WSL TCP에서 FastAPI의 두 List 필드가 이를422로 거부했습니다. 비null `Feature2MetricsDto.empty()`를 같은 실제 assembler에 전달하면 두 목록이[]가 됩니다. 정상 입력 테스트를 실제 assembler 경로로 연결하고 null 입력은 별도 관측으로 남겼습니다. 일반 서비스 경로 전체가422를 발생시킨다는 주장은 아닙니다.

## SYNC-AUTH-001 — 관리자용 종목 동기화를 일반 USER가 실행 가능

- 우선 확인 필요: 관리자 권한 경계 불일치입니다. `StockSyncController`의 Tag 설명은 Admin-only/ADMIN role이나, 경로는 `/api/v1/sync/tickers`입니다. `SecurityConfig`는 `/api/v1/admin/**`에만 ADMIN을 요구하므로 이 경로는 anyExchange().authenticated()로 처리됩니다. Controller에도 별도 역할 차단이 없습니다.
- 재현: `SyncSecurityFlowTest.ordinaryUserMustNotExecuteDocumentedAdminTickerSync`. 실제 SecurityConfig/JwtAuthFilter와 동기화 Controller를 작은 application context에 연결하고, 합성 JWT provider가 ROLE_USER를 부여하게 합니다. 제공처와 동기화 service만 mock입니다.
- 기대/실제:403 및 두 경계 호출0 / HTTP200·provider.fetchTickers와service.syncMarketStackTickers 호출. status·provider·service를 독립 단언하는1개 회귀 FAIL입니다. 익명401/ADMIN200, 관리자 ping의 USER403 대조군은 PASS입니다.
- 영향/한계: 일반 인증 사용자가 원천 수집과 종목 수정 업무를 시작할 수 있는 경로입니다. 실제 DB/외부 호출은 하지 않았습니다. 별도 실제 서비스 fixture에서는 동기화가 기존 산업/섹터를null로 만드는 것도 관측했으나 실제 사용자 데이터 변경은 없습니다.
- 기존369개 security matrix는 현재 경로 규칙/sentinel을 검증하므로 이 설정도 PASS였습니다. 그 결과가 관리자 업무 전체 권한 적정성을 입증하지 않음을 명시합니다. 제품 보안 설정은 수정하지 않았습니다. [상세 재현](../backend/src/test/diagnostic-ticker-flow.md).

## DEBUG-ERROR-001 — 진단 API 두 경로에서 내부 예외 상세 공개

- 구현: `HelloController.testExternal/testAnalysis`는 safeMessage(ex.getMessage())를 Public errors.message에 붙입니다. safeMessage는 공백 처리와200자 길이 제한뿐이며 내용 비공개 처리가 아닙니다.
- 재현: `DiagnosticHttpFlowTest.debugErrorsMustNotRevealSyntheticInternalDetails`. 외부/분석 client에 합성 내부 식별문구 오류를 주입한 뒤 두 HTTP 응답에서 해당문구 부재를 기대합니다.
- 실제: `/api/v1/test/external`과 `/api/v1/test/feature1`의 HTTP200 오류 envelope에 모두 문구가 남아1개 메서드의2단언이 FAIL입니다. 실제 secret/실제 제공처 오류를 사용하지 않았습니다.
- 영향/한계: 진단 경로가 내부 장애 정보를 인증 사용자에게 노출할 수 있습니다. 익명 접근 가능성이나 실제 비밀값 유출을 입증한 것이 아닙니다. 제품 미수정.

## OBS-TICKER-SYNC-001 — 정규화·분류·건수·페이지·실패 범위

실제 StockSyncService의 반복 fixture 실행은 기존 stock의 industry/sector를null로 바꿉니다. 메타 조회는 종목코드 trim/대문자/접미사제거와 거래소 별칭 정규화를 하지만 동기화 저장은 원시 symbol/acronym을 보존하고 acronym이null일 때만 MIC를 사용합니다. 빈 acronym도 저장 대상으로 전달됩니다.

exchange없는 행을 건너뛰어도 성공 응답의 건수는 입력 개수입니다. 원천 pagination이count1/total3을 알려도 실제 GlobalStockClient/Controller는1회 요청의1개만 처리합니다. 저장 실패 전 exchange mock save는 완료되지만 뒤 종목은 처리하지 않습니다. 실제 부분 commit/rollback은 미검증이며 `@Transactional`의 비동기 실행 범위는 격리 DB에서 별도로 확인해야 합니다.

public meta는 미존재404이나 debug meta는200 오류 envelope입니다. symbol 누락400과 달리 빈 symbol은500으로 매핑됩니다. 이 항목들은 정책 관측으로 분리했으며 임의로 정상 정책이라고 선언하거나 제품을 수정하지 않았습니다.

## F3-PRICE-SOURCE-001 — 수정종가 fallback의 원시가격 제공처를 KIS로 고정

- 구현: `Feature3PriceSeriesService.toRawResponse`는 `adjustedFallback ? "KIS" : rawFeature3Source(result.getSource())`로 source를 생성합니다. risk 경고에도 KIS를 고정 표기합니다.
- 재현: `Feature3PriceHttpFlowTest.fallbackMustReportActualRawProviderInsteadOfHardCodedKis`. 실제 Yahoo JSON 파서에는 빈 응답을, 원시가격 제공처 경계에는 두 일봉과 CandleSource.MARKETSTACK을 주입합니다.
- 기대/실제: 사용한 제공처 MARKETSTACK / 응답 KIS. HTTP→실제 가격 service/로더를 통한 회귀1개 FAIL입니다. 실제 금융 API나 DB는 호출하지 않았습니다.
- 영향/한계: reference.md의 가격 기준·제공처 기록 정책과 달리 fallback 데이터의 출처를 오인하게 합니다. 원천 값 자체를 잘못 계산했다는 증거는 아닙니다. 제품 미수정, [상세 방법](../backend/src/test/feature3-market-data.md).

## F3-PRICE-DATE-DUP-001 — 같은 거래일의 두 관측을 두 거래일로 계산

- 구현: `YahooFeature3PriceProvider.evaluate`와 `Feature3PriceSeriesService.toRawResponse`는 시각순 정렬과 마지막 N개 선택 뒤 행수로 가용가격/결측률을 계산하며 거래일 중복을 제거하지 않습니다.
- 재현: `Feature3PriceHttpFlowTest.duplicateYahooDatesMustNotCountAsTwoTradingDays`와 `duplicateRawDatesMustNotInflateAvailableTradingDays`. 같은 거래일의 KST 정오와13시 두 값을 넣고2거래일을 요청합니다.
- 기대/실제: Yahoo는 고유1일/기대2일의 결측50%로 전체 raw fallback해야 하나 fallback=false. raw 응답은 가용1일이어야 하나 availablePriceCount=2. 두 회귀 FAIL입니다. 지수 경로의 동일일 중복 제거 대조군은 PASS입니다.
- 영향/한계: 관측수가 실제 거래일 coverage보다 커져 품질 판정을 통과하고 정규화된 시각이 중복될 수 있습니다. 실제 원천/DB에서 중복이 발생한다는 증거는 아니며 최종 공분산/사용자 결과에 미친 영향은 미검증입니다. 제품 미수정.

## F3-BENCHMARK-FUTURE-001 — 기준 거래일 이후 지수를 유효 과거 이력으로 인정

- 구현: `Feature3BenchmarkSeriesService.sanitizeRows`는 양수/시각/중복을 검사하지만 기준일 상한을 적용하지 않습니다. `isStale`은 최신 DB 날짜가 기준일보다 과거인지로만 판정합니다.
- 재현: `Feature3BenchmarkHttpFlowTest.futureBenchmarkRowsMustNotBeTreatedAsFreshCompletedHistory`. 달력 latestTradingDay=2026-09-24, 저장소 두 행은09-25/09-26, lookback2입니다.
- 기대/실제: 아직 도달하지 않은 거래일로 완료된 과거 이력을 충족하면 안 되지만 benchmarkAvailable=true가 반환됩니다. 필드 존재를 강제한 HTTP 회귀1개 FAIL입니다.
- 영향/한계: 합성 손상/미래 입력에 대한 방어 검증이며 실제 DB에 미래 지수가 있다는 주장이 아닙니다. CAPM 전체 연결·실제 SQL은 별도입니다. 제품 미수정.

## OBS-F3-MARKET-DATA-001 — 지원 범위와 응답/저장 차이

일반 미국 티커 QA/NASDAQ은 Yahoo 심볼을 만들지 않고 raw로 전환합니다. 지수 SPX는 충분한 행수만 있으면1000일 이전 자료도 가용true이며 달력을 호출하지 않습니다. 이는 국내 지수만 freshness를 확인하는 현재 정책 관측입니다. 보강 후 충분해져도 최초 이력부족/stale 경고를 유지합니다.

raw 보강 시 기존 날짜의 외부999가 응답에 반영되지만 기존 원본100은 그대로이고 새 날짜만 저장 대상으로 선택됩니다. 원본 보존과 응답의 최신값 병합은 별개입니다. 실제 DB commit·재조회·장 마감 판단은 아직 검증하지 않았습니다.

## SEC13F-HEADER-001 — 필수 TSV 열이 없는 파일을 적재 성공으로 처리

- 구현: `Sec13fDataSetParser.TsvHeader.read`는header행이없는경우만거부하고필수열집합을검사하지않습니다. `value`는열누락시빈문자열을반환하므로CUSIP가없는INFOTABLE행을모두unmapped로제외합니다.
- 재현: `Sec13fParserTest.missingRequiredColumnsMustNotSilentlyBecomeEmptyData`, `Sec13fAdminFlowTest.malformedRequiredHeaderMustProduceFailedImportNotSuccess`. 정상SUBMISSION/COVERPAGE와WRONG_COLUMN만있는INFOTABLE을합성ZIP으로생성합니다.
- 기대: 스키마검증오류,관리자적재FAILED. 실제: parser예외없음·관리자HTTP200/SUCCESS. 두회귀FAIL을유지합니다.
- 영향/한계: 원천스키마손상/변경을정상적인수집대상없음과구분하지못할수있습니다. 실제SEC원천이나운영DB에서의발생을확인한것은아닙니다.
- 상태: 제품미수정. [40개상세검사](../backend/src/test/sec-13f.md).

## SEC13F-DATE-001 — 불가능한 제출일을 월말로 자동 보정

- 구현: `Sec13fDataSetParser`의 `SEC_DATE_FORMAT`은 `dd-MMM-yyyy`와기본SMART resolver를사용합니다.
- 재현: `Sec13fParserTest.impossibleCalendarDateMustNotBeSilentlyAdjusted`. 합성SUBMISSION의제출일31-Feb-2026을실제ZIP/TSV파서로읽습니다.
- 기대: 잘못된달력날짜를거부. 실제: 예외없이2026-02-28로보정되며회귀1개FAIL입니다. 전체943개실행의XML실패메시지에서실제날짜를확인했습니다.
- 영향/한계: 입력손상을숨기고제출일에따른최신공시선택/단위판정에다른날짜를전달할수있습니다. 실제원천/DB영향은미검증입니다. 제품은수정하지않았습니다.

## OBS-SEC13F-001 — 적재 이력·집계·실패 범위

manager가없는보유행은parser통계에포함되지만service가제외해도SUCCESS입니다. 보유save오류전에공시mock save가완료돼도실패이력통계는0으로남으며오류문구는1000자로제한됩니다. 시작이력실패는500,파일처리중오류는대개HTTP200/FAILED로집계되고디렉터리적재는뒤파일을계속처리합니다. 실제DB부분commit여부는검증하지않았습니다.

매핑충돌은관리자upsert에서거부하지만기존충돌목록의적재선택은confidence최대/동률조회순입니다. 변경분기뿐아니라다음기존분기도증감을재계산하며SQLprojection이없는기존집계는삭제합니다. 최신공시/정정공시제외/기관별합산의native SQL은mock으로대체했으므로이관측을SQL검증완료로해석하지않습니다.

## SEC-SHARES-OVERFLOW-001 — 범위 초과 발행주식수가 작은 양수로 변환

- 구현: `SecCompanyFactsParser.parseLong`은 정수JSON에서 `node.longValue()`, 그외에서 `BigDecimal.longValue()`를 사용합니다. 두변환모두범위초과에대한정확성검증이없고, 후속 `parseFact`는변환결과가양수인지검사합니다.
- 재현: `SecCompanyFactsTest.outOfLongRangeMustNotBecomeValidPositiveShares`. 2^64+1인18446744073709551617을JSON숫자/문자열두형식으로유효기준일·CIK·공시번호와함께제공합니다. Long최댓값대조군도포함합니다.
- 기대: 저장가능범위를초과한행은유효발행주식수로사용되지않음. 실제: 두경로모두sharesOutstanding=1의fact를반환하므로1개메서드안의두단언이실패합니다.
- 영향/한계: 손상되거나예상범위를벗어난원천값이그럴듯한양수로바뀌어후속저장/평가분모에사용될수있습니다. 실제SEC응답/운영DB에서해당값이존재함을검증한것은아닙니다.
- 상태: 제품미수정,실패회귀유지. [50개검사의상세방법](../backend/src/test/sec-issued-master.md).

## OBS-SEC-ISSUED-MASTER-001 — 원천 선택·동기화 표시·별칭·집계 경계

CompanyFacts는소수123.9를123으로절삭하고미래기준일도허용합니다. 최신fact의공시번호가없으면저장400으로실패하며공시번호가있는구fact로돌아가지않습니다. 원천404/유효fact없음도stock동기화시각을갱신합니다. 저장된fact의핵심값이같으면기존자사주/유통주식수는보존하지만핵심값이바뀌면null로지웁니다. shares save뒤stock marker save실패경로도확인했으나실제DB부분commit은미검증입니다.

미국마스터는시장+티커로구분하며동일원천재실행에도unchanged stock을다시save합니다. CIK가없어도유효시장/티커/이름이면종목을생성하고, parser에서빠진잘못된행은service의total/skippedInvalid집계에남지않습니다. 빈원천은기존종목을삭제하지않습니다. 회사명과티커의정규화키가같으면별칭한개만저장합니다. 모두정책검토용관측이며임의제품수정은하지않았습니다.

## OPENDART-XML-TEXT-001 — XML 텍스트 조각 처리 시 회사명 앞부분 소실

- 구현: `OpenDartClient.parseCorpCodeXml`이 CHARACTERS/CDATA 이벤트마다 trim한 문자열을 `corpName`에 대입합니다. 같은 요소의 앞선 이벤트 문자열을 누적하지 않습니다.
- 재현: `OpenDartClientTest.xmlEntityFragmentsMustPreserveCompleteCompanyName`, `xmlCdataFragmentsMustPreserveCompleteCompanyName`. 메모리 ZIP에 각각 `QA &amp; Partners`, `QA<![CDATA[ & ]]>Partners` 회사명을 넣고 실제 WebClient byte decoder→ZIP→StAX parser를 통과시킵니다.
- 기대: 두 입력 모두 `QA & Partners`. 실제: 모두 `Partners`여서2개 FAIL. 일반Unicode·공백trim·날짜파싱 대조군은 통과했습니다.
- 영향: XML회사명은 매핑 service에서stock.dartCorpName으로 복사되므로 수집된 회사명의 앞부분을 잃을 수 있습니다. 실제 제공처 응답·운영DB 손상을 관측한 것은 아닙니다.
- 상태: 제품미수정, 실패회귀유지. [상세46개 검사와 실행방법](../backend/src/test/opendart.md). 외부전송은합성이고curl프로세스도시작전에차단했습니다.

## OBS-OPENDART-001 — 전체매핑·응답검증·일일집계·중첩실행

유효한 빈XML은 모든기존stock의DART매핑을해제하고OTHER로바꿉니다. 우선주는 같은입력으로재실행해도메타데이터를먼저비웠다가다시채워updated로집계됩니다. row의회사코드가요청회사코드와달라도우선저장하고, 주식수0/음수도허용합니다. 종목별오류는batch/daily의HTTP200 summary에failed로표시합니다. 둘째row저장오류전에첫mock저장호출은남지만created집계는0이며, 실제DB부분commit은검증하지않았습니다.

OpenDART scheduler 직접2스레드호출은둘다daily로진입합니다. 이는중첩guard부재관측이며실제cron중복발생을증명하지않습니다. 원천규격/운영정책확인전위관측을모두확정결함으로단정하지않고제품도수정하지않았습니다. 실제연동은격리저장소에서만수행해야합니다.

## SNAPSHOT-GROWTH-VERSION-001 — 연간 재무 수정본을 전년도처럼 비교

- 구현: `MarketSnapshotService.calculateGrowthMetrics`는 연간재무를 fiscalYear/periodNo/reportDate 역순으로 정렬하고 앞의2개를 current/previous로 선택합니다. 같은 회계기간의 version 중복 제거가 없습니다. 반면 실제 스냅샷 핵심지표의 `SnapshotCalculator`는 기간별 최신version을선택합니다.
- 재현: `MarketSnapshotAdminFlowTest.annualGrowthMustCompareDistinctYearsInsteadOfTwoVersionsOfOneYear`. 실제 관리자HTTP→주식수resolver/계산/저장callback/캐시JSON→DTO에2025v2(매출1000/순익80),2025v1(900/70),2024v1(800/40)을주입합니다. 평가주식수는각200으로동일합니다.
- 기대: 최신2025와2024비교로매출성장25%·EPS성장100%. 실제: 두2025수정본비교로11.111111%·14.285714%, 하나의회귀메서드안에서2산술단언이실패합니다.
- 대조군: 중복버전없는연간비교·과거평가주식수변경·연속8분기TTM비교는통과하고, 핵심EPS의동일분기중복버전선택도통과합니다. 모든재무계산이잘못됐다고확대하지않습니다.
- 영향/한계: 수정공시가있는입력에서스냅샷DTO 성장률이수정전대비변화로바뀔수있습니다. 실제운영DB·브라우저에나타난사례를확인한것은아닙니다.
- 상태: 제품미수정,실패회귀유지. [37개검사의방법](../backend/src/test/market-snapshot.md)에기록했습니다.

## OBS-SNAPSHOT-001 — UPDATED·출처·누락값·오류 격리의 의미

재무나주식수가없어핵심필드가null인스냅샷도관리자결과는UPDATED이며,다음backfill에서다시미완성으로판정합니다. 재무가없으면이미확보한주식수도계산기emptySnapshot에서제거됩니다. source는주식수입력조건에따라정하므로OPENDART_PRIMARY가모든필드의완성을보증하지않습니다.

저장시현재가의존값은null로비우고캐시두키를갱신합니다. Redis SET오류는UPDATED유지,저장오류는HTTP200안의FAILED/예외문구,사전조회오류는전체HTTP500입니다. 전년순익이음수이면성장률은signed분모를써서-40→80이-300%로표현됩니다. 시장이일치하지않는단일backfill요청도코드전역검색fallback으로다른시장종목을선택할수있습니다. 이는관측정책이며모두를확정결함으로단정하지않았습니다. 실제DB/Redis·제공처검증은남았습니다.

## INVESTOR-STOCK-MISSING-001 — 미등록 종목 수급 API의 빈 성공 응답

- 구현 의도: `StockInvestorFlowService.resolveStock`에 stock이null이면 `IllegalArgumentException("stock not found...")`을반환하는분기가있고, 공통예외처리기는이를400/error envelope로변환합니다.
- 원인: 직전 `Blocking.call(() -> repository.findByStockCodeWithExchange(...).orElse(null))`은 `Mono.fromCallable`을사용합니다. null값은Mono의empty완료가되므로후속flatMap의null분기가실행되지않습니다. sync/series의Controller응답포장map도실행되지않습니다. backfill은collectList가empty를빈목록으로바꾸므로성공envelope가생깁니다.
- 재현: `InvestorFlowAdminFlowTest.unknownStockSyncMustReturnExplicitBadRequest`, `unknownStockBackfillMustReturnExplicitBadRequest`, `unknownStockSeriesMustReturnExplicitBadRequest`. 미등록코드UNKNOWN의Repository결과를Optional.empty로주입해각실제HTTP경로를실행합니다.
- 기대: 의도된400과오류본문. 실제: sync/series는200/빈본문,backfill은200/성공envelope·savedCount0·rows[]·errors[]로3개모두실패. 정규화lookup1·원천전송/flow저장소/rate limiter0도별도확인했습니다.
- 영향: 관리자는존재하지않는종목요청을명시적으로구분할수없고sync/series에는공통응답envelope도없습니다. backfill은정상적인자료없는종목의200/빈rows와구분되지않습니다. 실제운영DB/사용자장애를관측한것은아닙니다.
- 상태: 제품미수정,3개실패회귀유지. [상세방법](../backend/src/test/investor-flow.md). Windows전체770개중19실패에포함됩니다.

## OBS-INVESTOR-FLOW-001 — 저장 집계·범위·재시도 결과의 해석

수급의savedCount는반환rows길이입니다. time-limit fallback은save0이어도기존행수를반환하고, 종목backfill의중복기간은save8회후결과2행으로집계할수있습니다. 잘못된숫자는null로저장하며output누락/null도빈성공입니다. 시장원천요청의날짜1/2는모두to이고from은로컬필터에만사용됩니다. 원천API의실제보존기간/페이지규격을확인하지않아전체기간수집성공으로해석하지않습니다.

배치에서40일오래된자료가있어도refreshTail10이면최근10일범위만조회합니다. HTTP오류후재시도에서time-limit가나면RetryFailedException이cause를감싸므로직접time-limit와달리failed로집계됩니다. 두번째save/앵커오류전에첫save호출이완료되지만실제트랜잭션원자성검증은아닙니다. 이는정책검토용관측이며제품을수정하거나모든항목을확정결함으로분류하지않았습니다.

## F1-ERROR-001 — 분석 실패 응답의 내부 예외 메시지 노출

- 요구사항: 공개 응답에 내부 provider 정보·raw exception·secret을 노출하지 않는 오류 정책.
- 구현: `backend/src/main/java/com/qaima/api/feat1/FeatOneController.java`의 `feature1ErrorResponse`, `safeMessage`.
- 재현: `Feature1FlowTest.providerFailureDoesNotExposeInternalMessage`. 분석 service에 합성 문구 `qaima-fixture-private-detail`을 가진 IllegalStateException을 주입하고 Controller의 Public 오류 응답을 확인.
- 기대: 정형 오류 코드·안내문만 제공, 내부 예외 문구 미포함.
- 실제: 내부 문구 포함. 2026-09-25 Windows 전체398개 실행에서 이 케이스만 FAIL.
- 영향: 실제 외부 클라이언트 예외에 내부 요청·접속 세부가 들어 있으면 Public 응답으로 전달될 수 있음. 실제 비밀정보 유출을 관측했다는 뜻은 아님.
- 상태: 미수정. 사용자 허용 범위는 테스트·문서·임시 수행이며 서비스 변경은 포함하지 않음.
- 재실행: backend 테스트 문서의 Gradle 명령에 `--tests '*Feature1FlowTest'` 추가. 실패 테스트를 skip/expected failure로 바꾸지 않음.

## F3-CREDIT-001 — 입력 데이터·overlay 로딩 실패 시 환불 누락

- 요구사항: reference.md의 Feature3 핵심 분석 실패 시 크레딧 환불 후 오류 반환 정책.
- 구현: `Feature3AnalyzeController.analyze`의 refund `onErrorResume`이 FastAPI 호출 이후 내부 체인에만 적용됩니다. 차감 이후의 `toFastApiRequest`와 `loadOverlaySignals` 오류는 이 범위 밖입니다.
- 재현: `Feature3FlowTest.priceFailureAfterChargeMustRefund`, `overlayLoadFailureAfterChargeMustRefund`. 사용자7·비용3 fixture에서 차감 Mono의 실제 구독을 카운트한 후 가격 조회 또는 overlay 로딩에 오류를 주입합니다.
- 기대: 차감1회 이후 분석 실패 시 동일 비용3·referenceId로 환불 요청, 원래 오류 반환.
- 실제: 두 경우 모두 차감 구독1회·오류 반환·FastAPI 미호출을 확인했으나 refund 호출0회여서 FAIL. 대조군인 FastAPI 오류는 refund 호출 후 오류 반환으로 PASS.
- 실행: 2026-09-25 Windows에서 Feature2/3 추가12개 중10 PASS·2 FAIL. 실제 DB·금융 API·사용자 잔액은 사용하지 않았습니다.
- 영향: 해당 준비 단계가 오류로 끝나면 결과 없이 차감만 남을 가능성. 실제 사용자의 손실 발생을 관측했다는 뜻은 아닙니다.
- 상태: 미수정. 실패를 skip 처리하지 않았으며 `--tests '*Feature3FlowTest'`로 재현 가능합니다. 실제 트랜잭션·원장 중복방지는 별도 검증 필요.

## REPORT-MAPPING-001 — Feature1 저장 리포트의 종목코드·회사명 뒤바뀜

- 구현: `AnalysisReportService.createFeature1`에서 `AnalysisReportCreateCommand`의 stockCode 위치에 `resolveCompanyName(...)`, companyName 위치에 stockCode를 전달합니다. command record와 `create`의 저장 필드 순서는 stockCode→companyName입니다.
- 재현: `ReportHttpFlowTest.featureOneKeepsStockCodeAndResolvedCompanyNameInCorrectColumns`. 합성 종목코드 QA0001·회사명 합성기업, Feature1 request/정상 metrics를 실제 createFeature1에 전달하고 Repository saveAndFlush의 객체와 GET `/api/v1/reports/42` 응답을 확인합니다. Repository만 mock이며 제품 로직은 실제입니다.
- 기대: stockCode=QA0001, companyName=합성기업. 실제: stockCode=합성기업, companyName=QA0001. 회귀 테스트는 FAIL로 유지합니다.
- 영향: 저장 리포트 목록·상세·PDF의 종목 식별 정보가 바뀔 수 있습니다. stockCode 컬럼 길이32보다 긴 회사명은 저장에도 영향을 줄 가능성이 있으나 실제 DB 오류·기존 데이터 손상은 검증하지 않았습니다.
- 대조군: Feature2의 동일 코드/회사명 매핑은 통과했습니다. 분석 계산값 자체의 오류를 의미하지 않습니다.
- 상태: 미수정. 제품 코드·기존 리포트 데이터를 변경하지 않았습니다. 위 테스트 필터로 재현할 수 있습니다.

## CANDLE-DECODE-001 — 캔들 디코딩 오류가 성공 fallback으로 흡수됨

- 구현: `CandleLoadService.load/loadForFeature3`의 첫 `onErrorResume(ErrorException.class, ...)`은 KIS_DECODE_ERROR를 `Mono.error`로 다시 전달합니다. 하지만 바로 뒤의 일반 `onErrorResume`이 같은 오류도 잡아 성공 결과로 바꿉니다.
- 기대 근거: loader의 “decode는 숨기지 말고 터뜨림” 주석과 두 메서드의 명시적 decode 재전파 분기, ErrorCode.KIS_DECODE_ERROR의502 계약. 운영 정책 확인·변경은 별도이며 테스트가 새로운 fallback 정책을 적용한 것은 아닙니다.
- 재현: `ChartCandleFlowTest.decodeFailureMustRemainErrorInsteadOfSuccessfulNoData`와 `featureThreeDecodeFailureMustNotBecomeStaleSuccess`. 합성 provider에 해당 ErrorException을 주입하고 실제 로더를 실행합니다.
- 실제: 차트 HTTP는200·EMPTY·NO_DATA, Feature3 로더는 부족한 기존 DB 데이터·source DB를 반환합니다. 두 오류전파 회귀 테스트 FAIL. HTTP/BIZ/MARKET_CLOSED의 의도된 fallback 대조군은 PASS.
- 영향: 외부 응답 파싱 결함이 데이터 없음/재사용으로 보일 수 있습니다. 실제 금융 제공처 장애나 잘못된 분석 결과를 관측했다는 뜻은 아닙니다.
- 상태: 제품 미수정, 실제 DB/금융 API 미호출. `--tests '*ChartCandleFlowTest'`로 재현합니다.

## F2-MACRO-001 — 한 거시 지표의 이중 실패가 다른 정상 지표까지 제거

- 요구사항 근거: reference.md의 Feature2 일부 데이터 실패는 부분 결과로 흡수한다는 정책. 실제 장애 발생 여부와 별개로 카드의 부분 결과 보존을 검사합니다.
- 구현: `Feature2CardService.loadMacroRates`는 각 동기화 오류에 `findLatest`로 fallback하지만, 이 조회도 실패하면 `Mono.zip` 바깥의 `onErrorResume`에서 모든 금리·환율 필드가 null인 빈 카드로 대체합니다.
- 재현: `Feature2CardFlowTest.macroComponentDoubleFailureMustPreserveOtherRates`. 한국 금리 정상2.75, 미국 동기화와 저장값 조회는 각각 합성 오류, 다른 소스는 정상 빈값으로 주입. 실제 Controller→CardService HTTP 응답을 검사합니다.
- 기대: HTTP200 부분 응답에 정상 한국 금리2.75 보존. 실제: HTTP200이지만 krBaseRate/usFedFundsRate/usdKrw=null, bondYields=[]이므로 회귀 FAIL. 외부 금융 API·실제 DB를 호출하지 않았습니다.
- 대조군: 동기화만 실패하고 저장값 조회가 성공하면 금리 보존 PASS. 별도 `/macro-rates-series`는 실패한 소스·채권만 제외하고 다른 시계열을 보존합니다.
- 영향: 하나의 소스와 fallback 장애가 전체 매크로 카드 데이터 없음으로 나타날 수 있습니다. 실제 사용자 장애·분석 전체 실패를 관측한 것은 아닙니다.
- 상태: 제품 미수정. 실패를 expected/skip으로 바꾸지 않습니다. 재현 필터 `--tests '*Feature2CardFlowTest'`.

## F2-NEWS-CACHE-001 — 뉴스 focus 변경·점수 누락을 캐시 적중으로 오판

- 요구사항: reference.md의 뉴스 목록뿐 아니라 본문·focus·감성 및 버전 호환 확인, 재사용 가능한 원천 결과와 비용 미리보기·실행의 일관성.
- 구현: `NewsSentimentService.isReusableSentiment`는 modelVersion/promptVersion만 비교합니다. 감성의 focusTextVersion과 현재 focus hash를 비교하지 않고 sentimentScore 존재도 검사하지 않습니다. `inspectNewsCacheBlocking`와 `getOrAnalyzeSentiment`가 이 판정을 사용합니다.
- 재현1: `NewsFlowTest.inspectionMustRejectChangedFocusVersion`, `analysisMustNotReuseScoreForDifferentFocus`. 합성 본문·focus·점수0.75의 완전 캐시를 만든 후 focus와 SHA-256 hash만 변경합니다. 기대는 miss/재분석 결과0.25, 실제는 hit/이전0.75·모델 호출 없음입니다. 모델0.25는 fixture 값이며 실제 모델 정확도 검증이 아닙니다.
- 재현2: `inspectionMustRejectCachedEntryWithoutScore`. 나머지 계층/버전은 유효하고 점수만 null인 캐시를 주입하면 hit=true, 실행에서도 점수null·재분석 없음입니다. 이는 손상/불완전 캐시를 주입한 검사이며 정상 쓰기 경로가 null 점수를 저장한다고 주장하지 않습니다.
- 비용 연결: `overlayPreviewAndEstimateMustChargeForChangedNewsFocus`는 실제 Feature3OverlayService→실제 NewsSentimentService를 연결합니다. 정상 캐시의 뉴스 추가비용0 대조군 후 focus를 변경하여 MISS·추가1·총2를 기대했으나 실제 HIT·추가0·총1로 FAIL했습니다. [Spring 테스트 기록](../backend/src/test/README.md)에 실행 근거를 기록했습니다. 실제 CreditService·원장·차감은 호출하지 않습니다.
- 대조군: 계층 누락·잘못된 JSON·구버전 모델/입력 형식은 miss, 정상 캐시는 hit, FORCE_REFRESH는 source/body/model 재실행입니다.
- 상태: 테스트·문서만 추가, 제품 미수정. 재현 필터 `--tests '*NewsFlowTest'`. 실제 Redis TTL·동시 갱신·실사용 데이터에서의 발생 여부는 별도 검증이 필요합니다.

## F2-PEER-DATE-001 — window pack의 최신 종목 가격이 경계에서 제외됨

- 구현 의도 근거: `PeerClusterDataServiceImpl.safeBuild`의 window 모드는 산업지수 DB 최신 시각을 기준으로 peer 차트 기간을 통일한다는 주석·분기를 가집니다.
- 구현 차이: `resolveIndustryIndexLatestTs` 값을 그대로 `findRangeBulk(..., from, to)`의 to로 전달합니다. `PriceOhlcvRepository.findRangeBulk`의 JPQL은 `p.id.ts < :to`이므로 산업지수 최신 시각과 같은 종목 일봉이 제외됩니다. 산업지수는 `findRecent(window)`에서 그 일봉을 포함합니다.
- 재현: `PeerDataFlowTest.windowMustIncludeLatestIndustryIndexDateInStockSeries`. 종목과 지수에 동일 시각의45개 합성 일봉을 두고 window30을 요청합니다. Repository mock은 현재 JPQL의 `[from,to)` 필터를 그대로 적용합니다. 기대 종목 마지막 시각은 지수의2026-09-13T15:00:00Z, 실제는 하루 전2026-09-12T15:00:00Z입니다.
- 검증 한계: 실제 SQL을 실행한 것이 아니라 현재 query 조건에 맞춘 fixture입니다. DB timezone·JPA 변환·실사용 데이터 발생 여부는 별도 검증이 필요합니다. 기간 명시 모드에서는 종료일 다음날0시를 exclusive to로 사용하여 동일45개가 유지되는 대조군이 통과했습니다.
- 영향: window 모드에서 종목/산업 비교의 마지막 날짜·공통 표본이 달라질 수 있습니다. 실제 WSL uvicorn→Windows Controller/Service→Python계산 TCP에서도 원천최신7/30에대해최종anchor_series가7/29에서끝나는것을재현했습니다. [역방향TCP8개중1회귀FAIL](../backend/src/test/peer-reverse-tcp.md). 정확기간은91행을보존합니다. 실제SQL·실사용데이터에서상관계수변화량/사용자영향규모는미검증입니다.
- 상태: 제품 미수정·회귀 FAIL 유지. `--tests '*PeerDataFlowTest'`로 재현합니다.

## FIN-ADMIN-HALF-001 — half 단독 수정이 반영되지 않음

- 구현: `FinancialAdminService.update`의 periodRelatedChanged는 reportDate/year/quarter/periodType만 확인하고 half는 포함하지 않습니다. 따라서 half 단독 요청에서는 `applyPeriodFields`가 실행되지 않습니다.
- 재현: `FinancialAdminFlowTest.halfOnlyUpdateMustChangePeriodNumber`. 실제HTTP→service→mapper, Repository/트랜잭션관리자mock. H/half1 등록 후 PUT `{half:2}`를 보내면 HTTP200이지만 half1 그대로입니다. 기대 half2로 FAIL 유지합니다.
- 대조군: periodType=H와half2를 함께 보내는 기간변경 요청은 반영됩니다. 기존DB 변경·SQL 실행 없이 저장요청객체와응답을 확인했습니다.
- 상태: 제품 미수정. `--tests '*FinancialAdminFlowTest'`로 재현합니다.

## FIN-ADMIN-FIELDS-001 — 재무 등록·수정 요청의 일부 절대값 필드 누락

- 계약/구현 근거: 입력과출력에공유하는 `FinancialDto`는 grossProfit/retainedEarnings/cashAndEquivalents를 절대값 재무지표로 선언하며 명시적인 read-only 표시는 없습니다. mapper는 entity의해당값을읽지만 create/update는이3필드를entity에복사하지않습니다.
- 재현: `createMustPreserveWritableAbsoluteFinancialFields`, `updateMustPreserveWritableAbsoluteFinancialFields`. grossProfit300·retainedEarnings250·cashAndEquivalents90을HTTP로요청하면200이어도Repository save객체의3값이모두null입니다. 두회귀 FAIL, 나머지 revenue/operatingIncome/netIncome/자산·부채·자본의대조군은통과했습니다.
- 영향: 해당필드를등록/수정할수있다는입력계약과실제처리가맞지않습니다. 의도적으로읽기전용인정책이있다면별도명시/확인이필요합니다. 실제DB손실이나기존데이터손상은관측하지않았습니다.
- 상태: 제품 미수정, 같은 클래스 필터로 재현합니다.

## FIN-CSV-QUOTE-001 — 따옴표로 감싼 CSV 숫자 거부

- 구현: `FinancialImportService`는 각행을 `line.split(",", -1)`로 나누고 따옴표를 해석하지 않은 채 BigDecimal로 변환합니다.
- 재현: `FinancialCsvFlowTest.quotedCsvNumericFieldMustBeParsedAsValidValue`. 정상합성행의revenue만 `"1000.50"`으로감싸실제UTF-8파일에서읽으면기대upserted1·failed0과달리upserted0·failed1입니다. 일반숫자1000.50은통과합니다.
- 검증 경계: 단일 quoted numeric cell의재현입니다. 쉼표/개행/이스케이프를포함한모든CSV형식이나실제원본파일에대한실패율을검사한것은아닙니다. SQL은호출되지않았습니다.
- 상태: 제품 미수정, `--tests '*FinancialCsvFlowTest'`로재현합니다.

## OBS-FIN-CSV-001 — 배치 실패 후 재시도와 결과 집계 중첩

1001개합성행에서첫1000행의mock JDBC를실패시키면행루프가failed1을더하고배치를보존합니다. 다음행을추가한1001행재시도가성공하면결과는upserted1001·failed1입니다. 즉두숫자는상호배타적행개수가아닙니다. rollback1·commit1호출을확인했으며실제SQL의롤백/원자성은미검증입니다. 마지막잔여배치실패는루프밖에서전파됩니다. importer의upserted는JDBC영향행수합이아니라전달한배치행수이고, 중복기간행은DB ON DUPLICATE KEY UPDATE에맡깁니다.

## F2-LLM-FLAG-001 — 설명 비활성화 요청에도 제공자 호출

- 계약 근거: `Feature2RequestContext.include_llm_explain`은 bool 필드로 false를 받지만 `Feature2AnalysisRequest.to_explain_request`는 이를 전달하지 않습니다. `/feature2/analysis`는 변환 결과를 무조건 `analyze_feature2_explain`에 전달합니다.
- 재현: `test_feature2_explain_http.Feature2ExplainHttpTests.test_false_include_flag_must_not_call_paid_provider`. 실제 ASGI/요청변환/factory/client/prompt를 실행하고 httpx 전송만 MockTransport로 대체합니다. 정상 합성응답, include_llm_explain=false에서 기대 전송0회·실제1회로 FAIL했습니다.
- 영향: 내부 API의 설명비활성화 옵션이 비용호출 방지로 작동하지 않습니다. 실제 비용이 발생한 것은 아니며 Spring Public 옵션/브라우저 흐름까지 연결하여 검증한 것도 아닙니다. 기존 Feature1/3의 생략 동작과 구분합니다.
- 상태: 제품 미수정, 실패 회귀 유지. 분석 테스트 명령의 `-p test_feature2_explain_http.py`로 재현합니다.

## F2-LLM-ERROR-001 — 제공자 오류 상세가 FastAPI 응답 경고에 포함

- 구현: 두 provider client가 HTTP오류본문 또는 연결예외 `str(exc)`를 warning에 붙이고 `analyze_feature2_explain`이 그대로 반환합니다. 예상 밖 factory 예외는 공통500에서 정형코드로 처리되는 대조군과 다릅니다.
- 재현: `test_provider_error_body_must_not_escape_into_http_warnings`, `test_connection_exception_detail_must_not_escape_into_http_warnings`. OpenAI/Gemini 각각 합성401본문·ConnectError에 식별문구를 주입하면 HTTP200·explain=null의 warnings에 그대로 남습니다. 테스트 메서드2개·제공자별 실패subcase4개입니다. 실제 자격증명은 사용하지 않았습니다.
- 기대: 진단 상세를 HTTP에 넘기지 않고 정형warning만 전달. 실제: `LLM_EXPLAIN_HTTP_401:<합성상세>`, `LLM_EXPLAIN_EXCEPTION:ConnectError:<합성상세>`. 이는 실제secret 유출 관측이 아니라 상세문구 전파 경로 재현입니다.
- 현재 검증 경계: 내부FastAPI HTTP까지 확정. Spring의 `loadExplain`/`Feature2MetaDto.addWarning(String)`에는 전달 경로가 보이지만 Public 응답·프론트까지의 실행 검증은 남았습니다.
- 상태: 제품 미수정, 실패회귀 유지. 재현 필터는 위 파일과 같습니다.

## OBS-F2-LLM-001 — 재시도 상한·compact·관대한 파싱

지속429를 합성 전송하면 OpenAI/Gemini 모두 provider3회 뒤 compact에서3회를 더 호출하여 요청당6회가 됩니다. 사용자 검증 허용한도5회를 초과할 수 있으므로 실제유료검사는 별도 전송직전 카운터·잔여량 차단 없이 실행하지 않습니다. 현재는 mock이라 실제비용0입니다. timeout은2회, 지속503은OpenAI3회/Gemini1회로 확인했습니다. compact의prompt는동일하고 출력토큰한도만4000→1800으로바뀝니다. 잘못된JSON은1회후PARSE_FAILED·null이며 `{}`는빈6섹션으로수용하고 raw `explain.text`는 용어치환전 값을 유지합니다. 이 관측들을 모든 설명품질/사용자표시 정책 위반으로 확대하지 않습니다.

## ENV-FRONT-001 — 원본 Windows native 빌드 의존성 누락

- 재현: Windows Node로 기존 frontend의 Vite build 실행.
- 실제: `@rollup/rollup-win32-x64-msvc`를 찾지 못하여 빌드 중단.
- 검증: tests/.runtime 복사본에서 기존 lockfile로 npm ci 후 build 성공.
- 상태: 원본 node_modules·lockfile 수정 없음. 코드 컴파일 오류와 설치 환경 오류를 분리하여 기록.

## OBS-REPORT-001 — 조회의 보존기간 정리 부수 효과

`AnalysisReportService.listMine/getMine`은 조회 전 사용자별7일 이전 리포트 삭제를 호출합니다. `ReportOwnershipTest`로 mock delete 호출을 확인했습니다. 코드 존재가 정책 위반이라는 판단은 하지 않으며, 실제 연동 테스트 대상 계정을 고를 때 기존 자료 삭제 가능성을 고려해야 합니다.

## OBS-LLM-001 — Feature1 설명 실패의 실제 반환

잘못된 JSON·timeout mock에서 지표는 유지되고 explain=null 및 warning이 반환됩니다. Feature3의 결정론적 설명 경로와 같다고 문서화하지 않습니다. reference의 “대체할 수 있다”를 모든 기능의 “항상 대체한다”로 확대하지 않습니다.

## OBS-F1-RECOVERY-001 — 분석 실패 후 실행 버튼 소실

실제Chrome의Feature1에서합성분석500을반환하면공통오류와잔액재조회는확인되지만같은화면의분석실행버튼은사라집니다. `StocksMockPage.handleAnalyzeClick`이처리시작후showAnalyzeButton을false로만들고오류시복원하지않으며, 결과패널도showAnalyzeButton=false를전달받습니다. 재검색등다른복구방법과별개로직접재시도UX검토가필요한관측입니다. 요구사항상의특정버튼정책위반이나실제환불성공으로단정하지않습니다.

[Chrome10개검사](../frontend/tests/browser-feature1-flow.md)는이현재동작을명시적으로기록하며제품은수정하지않았습니다. 재무초기연도가2024로고정된것도소스에서확인했지만최신연도자동조회PASS로집계하지않습니다.

## OBS-F3-PDF-STYLE-001 — 격리 Chrome PDF에서 스타일 손실 관측

Feature3 분석직후기본/고급PDF다운로드자동검사2개는성공했지만렌더링이미지에서는카드레이아웃·스타일손실을확인했습니다. 최초기본fixture는스타일이있는1페이지였고다음실행의동일기본입력은스타일없는2페이지가됐습니다. 고급차트fixture는4페이지로프론티어/파이/한글수치는남아있지만본문중간분할과배치손실이있습니다. 자동15 PASS·PDF구조6페이지PASS를시각품질PASS로확대하지않습니다.

진단재실행에서원본문체CSS811규칙·report display=flex·CSS요청완료를확인했습니다. 외부폰트CSS는테스트의origin차단으로실패하므로환경영향을배제할수없고, 제품버그/스타일로딩/캡처타이밍중원인은미확정입니다. 제품미수정이며 [방법·PDF·진단증거](../frontend/tests/browser-portfolio-flow.md)에기록했습니다. 실제백엔드·DB·유료호출은없습니다.

후속 계측으로 범위를 좁혔습니다. 외부 stylesheet를 네트워크 없이 빈 CSS200으로 응답해도 고급 PDF 문제가 남았습니다. 실제 렌더러의 `getComputedStyle` 호출을 원래 함수에 위임하며 기록한 결과, 정상 기본은복제문서flex/811+2규칙, 실패고급은block/2규칙을읽었습니다. 요청의완료여부와달리복제문서의본체CSS가준비되기전에렌더링계산이진행되는경로입니다.20ms간격중간표본은정상캡처전에도block을관측할수있어그것만으로판정하지않습니다.

렌더러의실제읽기시점CSS/배치보존단언을추가한전체회귀는15개14 PASS·고급PDF1 FAIL입니다(`run-PjKufe`). 과거15 PASS는생성만검사한이력입니다. 설치된html2canvas-pro2.0.2의iframe준비/스타일읽기경계가검토대상이며제품·의존성은수정하지않았습니다. 계측의타이밍영향과환경별발생률은미검증입니다.

## TEST-KIS-DIAGNOSTIC-001 — 실제 조회 단언 실패의 위치 기록 부족

실제 `KrStockClient` 토큰·기본정보 TLS 요청2회는 HTTP200/200이나 후속단언에서 `AssertionFailedError`가 발생했습니다. 최초 테스트가 오류의 타입만 보존하여 어떤 단언인지 특정할 수 없습니다. 제품 결함이나 정상 연동으로 확정하지 않습니다. 테스트에 비밀값 없는 고정 단계명을 추가했으며 컴파일·기본 회귀는 실행했지만 실제 요청은 반복하지 않았습니다. [5개 검사·증거·비용 기록](../backend/src/test/live-kis-readonly.md)에 세부 한계를 기록했습니다.

## OBS-STOCK-001 — 조회 메서드명·주석과 현재 호출 경로 차이

`StockService.getOrCreateStockByCode/getStockWithRealtimeByCode`에는 생성 관련 이름·주석과 private 생성 보조 메서드가 남아 있지만, 현재 Public 경로는 Repository에 없으면404이며 생성 보조 메서드를 호출하지 않습니다. `StockHttpFlowTest`에서 미등록 코드의 provider/save0을 확인했습니다. 반면 ID 미존재는400입니다. 신규 등록 지원·일관된404 정책을 완료 기능으로 문서화하지 않습니다.

## OBS-SHORT-SELLING-001 — 수집 상태·진단·부분커밋의 의미

`ShortSellingAdminFlowTest`에서 FINRA403/404는 HTTP200/fetched0, 정상200의 malformed 원천행은 관리자400, 요청일과 다른 원천행 날짜도 SQL에 그대로 전달됨을 확인했습니다. KRX probe는 시장별 전송예외 상세문구를 관리자200 응답에 포함하며, sync는 한 시장이 실패하면 저장소 접근 전 중단합니다. 두 제공처의 빈200 응답은 빈목록으로 처리됩니다.

두 sync service는1000행마다 REQUIRES_NEW를 사용합니다. 두번째배치/둘째날 실패 주입 시 첫배치/첫날 commit 호출은 이미 완료된 상태로 HTTP500이 됩니다. `upsertedCount`는 JDBC영향행수 합이 아니라 성공배치에 전달한 행수이며, KRX backfill의 unknownStockSamples는 일별 문자열 중복제거일 뿐 전체기간 종목별누계가 아닙니다. 실제DB 트랜잭션을 실행한 증거로 확대하지 않습니다.

이는 오류분류·진단노출·재실행/집계 정책 검토가 필요한 관측이며 확정 결함으로 단정하지 않았습니다. 제품은 미수정입니다. [40개 검사의 방법·경계](../backend/src/test/short-selling.md)에 자세히 기록했습니다.
