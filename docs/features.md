# 기능별 실행 흐름과 확인된 정책

작성·검증일: 2026-09-25. 구현의 경로를 설명하는 문서이며 각 항목의 검증 상태는 [추적표](traceability.md)를 따릅니다.

## Feature 1: 종목 심층 분석

`StocksMockPage` → `api/analysis.ts` → `POST /api/v1/feature1/analyze` → `FeatOneController` → `FeatOneService` → 가격·재무·스냅샷 조립 → `FastApiAnalysisClient` → `/feature1/analysis` → `analyze_stock` → Public DTO → overlay 원천 캐시·리포트 저장 → 화면 표시입니다.

별도 조회 경로의 StockMappingService는 ticker→등록 alias→회사명 순서로 중복 제거·20개 제한 검색을 수행하며 정규화 단일 조회는 모호한 결과를400으로 처리합니다. StockService의 현재 코드 조회는 미등록 시404이며 자동 생성하지 않습니다. 실제 Controller/Service와 Repository/provider mock을 연결한14개 검사가 통과했습니다.

재무 조회는 FinancialReadService→FinancialMapper에서 시가총액에는 발행주식 수, EPS/BPS에는 평가주식 수를 적용합니다. 합성 수치의 독립 기대값·동일 전년 기간 성장률·주식 수 변경·시세 없음 fallback 등10개 검사를 통과했습니다. 과거 기준일 조회도 RealtimePriceService를 호출하므로 역사적 시점 가치평가의 정확성을 검증했다고 해석하지 않습니다.

스냅샷 관리자backfill은실행일만허용하며국내시장종목의재무·주식수로계산한값을저장하고현재가의존marketCap/PER/PBR등은비웁니다. 실제ShareBasisResolver는보통주/우선주·전체발행수와자사주를구분하고EPS/BPS는평가주식수,SPS는outstanding을분모로사용합니다. 실제계산/캐시JSON연결37개중36 PASS·1 FAIL이며동일연도수정본을전년처럼비교하는성장률결함을재현했습니다. UPDATED라도재무/주식수누락으로불완전할수있습니다. [상세검증](../backend/src/test/market-snapshot.md)은합성저장소경계이며실제SQL·공개시세통합/브라우저는별도입니다.

관리자재무CRUD와CSV적재는실제서비스·mapper·UTF-8파일/parser·SQL인자생성을실행하되Repository/JDBC/tx관리자를mock한28개중24개통과·4개실패입니다. half단독수정과절대값3필드복사가누락되고,quoted숫자가거부됩니다. CSV는거래소/종목정규화와1000행배치·REQUIRES_NEW를사용하며실패후재시도집계가중첩될수있습니다. 실제DB의중복최종값·rollback·원자성완료를뜻하지않습니다.

차트는 ChartService→CandleLoadService→CandleMapper로 range/before 경로를 나눕니다. 신규 일봉만 저장하고 응답에는 외부 결과를 merge하는 동작, 빈 데이터/제공처 fallback을 mock으로 검증했습니다. KIS_DECODE_ERROR는 전파 의도와 달리 일반 onErrorResume에서 성공 fallback으로 흡수되는 결함을 두 경로에서 재현했습니다(`CANDLE-DECODE-001`).

FastAPI는 OHLCV 요약, EMA20/60/120, Bollinger20·2, Stochastic14·3·3을 계산합니다. market snapshot이 없으면 재무와 시장 문맥으로 파생하며, 연속된 최근4분기를 확보하지 못하면 연간 값의 fallback 경고가 발생할 수 있습니다. 지표 계산 예외는 지표 초기값과 경고로 처리하고 가격 요약을 유지하는 fixture를 검증했습니다.

설명은 `include_explain` 조건에 따라 호출합니다. 잘못된 JSON이나 timeout 시 compact 재시도하는 경로가 있습니다. provider mock 기준 실패 시 metrics는 유지되며 explain은 null이 됩니다. 모든 기능이 항상 결정론적 설명으로 대체된다고 문서화하면 현재 Feature1 동작과 다릅니다.

## Feature 2: 외부 요인

수급 관리자5경로는 KIS 토큰발급/캐시→종목·시장응답→저장객체→Public DTO로 연결됩니다. 종목backfill은21일간격기준일로조회한뒤범위필터·날짜정렬·중복제거하고, 운영시간제한은기존DB결과fallback입니다. savedCount에는새로저장하지않은fallback행도포함됩니다. 시장수집의from은로컬필터이며원천두날짜query는모두to입니다. 배치는KST15:40/거래일경계·국내EQUITY/자료없음/오래된순서·refresh-tail을사용합니다. 합성전송/저장소검사50개중47 PASS·미등록종목의200/빈본문 또는 빈rows 성공응답3 FAIL이며 실제SQL/제공처규격·화면전체연동은남았습니다. [수급 검증 기록](../backend/src/test/investor-flow.md)을참고하세요.

공매도 관리자 수집은 FINRA CNMS 일별 텍스트 또는 KRX 3시장 form POST→parser/client→등록종목 매핑→1000행 단위 SQL upsert 요청입니다. FINRA는403/404를 빈자료로 처리하고 ratio를6자리 분수로 계산하며, KRX는 제공된 비율값을 그대로 전달합니다. backfill은 휴장일 필터 없이 양끝을 포함한 최대3000일을 순차조회합니다. 두 스케줄러는 각각New York/Seoul 기준일을 사용하고 동일객체의 중첩실행을 막습니다. 실제서비스/codec·전송/저장소mock40개 검사는 통과했지만 배치·날짜별 부분커밋이 가능하고 실제SQL/제공처·적재후브라우저 전체연동은 남았습니다. [상세 방법과 관측](../backend/src/test/short-selling.md)을 참고하세요.

`Feature2MockPage`는 카드·시계열 조회와 종합 분석을 분리합니다. `/api/v1/feature2/cards/*`는 금리·매크로·산업·공매도·수급·관련 종목을 제공하고, `/api/v1/feature2/analyze`는 `Feature2AnalyzeService`에서 조립한 metrics를 설명하는 흐름입니다. FastAPI `/feature2/analysis`가 모든 원천 데이터를 수집하는 구조는 아닙니다.

카드 HTTP→`Feature2CardService`의 합성 검증을 추가했습니다. 기준금리 시계열은 limit1~1095, 공매도·수급은1~252, 거시 시계열은5~500, 관련 종목은1~30으로 제한합니다. 수급은 날짜 오름차순·null을0으로 합산하며 KOSPI/KOSDAQ 시장값을 별도 조회합니다. 관련 종목 카드는 같은 산업의 일봉 최대60개로 거래대금·로그수익률 변동성 유사도를 비교하는 별도 경량 경로이며, FastAPI Peer Cluster 결과와 같지 않습니다. 거시 카드에서 한 소스의 동기화와 fallback 조회가 모두 실패하면 정상 소스도 제거되는 `F2-MACRO-001`을 재현했습니다. Repository·resolver·동기화/하위 분석 service는 mock으로 실제 적재·캐시·외부 연동까지 검증한 것은 아닙니다.

Peer Cluster는 Spring 캐시 확인→FastAPI `/feature2/peer-cluster`→Spring `/api/v1/feature2/peercluster/data` 데이터 pack→수익률·상관·시차·유동성·변동성 유사도→Spring DTO·캐시 흐름입니다. 현재 조정 방법 상수는 `SIMPLE_SUBTRACTION`입니다. 이 결과를 정식 요인 회귀·미래 방향 예측으로 설명하지 않습니다.

Peer 데이터 endpoint는 내부 소비자용 예외 계약으로 envelope 없이 snake_case pack을 반환하며 요청은 snake_case/camelCase 모두 해석합니다. 합성 HTTP→실제 pack service 검증에서 300종목 상한(anchor 보존), 최소30포인트, 종목별 window tail, UTC 시각·거래대금, 거래일 보정과 조회 장애 fallback을 확인했습니다. 일반 window 모드에는 지수 최신 시각과 같은 종목 가격을 exclusive 조회가 제외하는 `F2-PEER-DATE-001`이 있습니다. 구체적 기간 모드는 window tail을 하지 않고 범위 내 데이터를 유지합니다. Repository/달력은 mock으로 실제 거래일·SQL 완료 검증이 아닙니다.

`PeerClientCacheFlowTest`는 실제 WebClient codec과 client/service를 연결하고 전송만 메모리 connector로 대체합니다. snake_case 요청·응답변환, Public t/value 시계열 계약, cache v6의 인자별 키·12시간 TTL 인자, legacy DTO·force refresh·Redis장애·잘못된응답 처리12개가 통과했습니다. 실제 FastAPI 호출·상관 계산·Redis 만료까지 확인한 것은 아닙니다.

Python의 `test_peer_cluster_http.py`는별도로실제ASGI→SpringMarketDataProvider→합성pack→Peer계산/응답17개를검증했습니다. raw/산업차감상관은원래수익률에서독립계산하고, 선행-2/후행+2·chart평균/20·80분위수·부분결측coverage2/3을확인했습니다. Python은현재ONE_W요청을warning과함께ONE_D로변환하며, Spring어댑터에일봉으로요청합니다. 실제Spring서버와의TCP연동이나통계적예측력검증을의미하지않습니다.

뉴스는 목록·기사·focus·감성 캐시와 영속화를 거쳐 `/feature2/news-sentiment`로 전달됩니다. 운영 모델 입력은 `[FOCUS]`와 `[DETAIL]`에 정리한 focus text 전체를 각각 사용합니다. Spring에서 이미 조립한 TITLE/FOCUS/DETAIL도 일반 문자열로 정규화·중복 삽입하는 현재 동작을 확인했습니다. 점수는 `(positive - negative) × (1 - neutral)`입니다. 합성모델을 사용한 실제ASGI/서비스·로더21개로 배치순서·점수·부분실패보존·입력/로딩오류·warmup을 검증했습니다. 실제 로컬 CPU모델3개도 오프라인으로 실행해 확률/점수·입력길이·단건/배치·ASGI응답을 확인했지만, 모델 정확도나 전체 시장 심리를 대표한다는 근거는 아닙니다.

공개 뉴스 목록은 기존 메타데이터만 반환하며 새 감성 분석을 호출하지 않습니다. 분석용 `loadNews`는 실제로 `[TITLE]`/`[FOCUS]`/`[DETAIL]` 입력과 SHA-256을 만들고 감성 결과 저장·관측 기록을 요청합니다. `NewsFlowTest`로 이 orchestration과 필터·캐시·실패 처리를 합성 검증했으며 provider/모델·Repository·Redis는 대체했습니다. 목록5분, refresh marker15분, 본문/focus/감성7일의 TTL 인자를 확인했으나 실제 만료는 미검증입니다. 현재 재사용 판정은 focus hash 변경·null 점수를 놓쳐 이전 결과를 사용하거나 재분석을 생략하는 `F2-NEWS-CACHE-001`이 있습니다. 모델 정확도·전체 뉴스 ingestion 완료를 의미하지 않습니다.

Feature2 설명은 실제ASGI→요청변환→두제공자client/prompt→parser를 연결하고 전송만mock한17개중14개가통과했습니다. timeout/429/파싱실패는경고와explain=null이며 Feature3의결정론적설명과다릅니다. include_llm_explain=false무시·제공자오류상세응답포함의3개메서드가실패했습니다. 지속429는compact포함최대6회전송이라 실제유료검증에별도한도차단이필요합니다. 유료호출·설명품질·Spring전체연동을완료한것은아닙니다.

## Feature 3: 포트폴리오

`PortfolioMockPage`→기본 포트폴리오 저장 또는 분석 입력→overlay 비용 미리보기→`POST /api/v1/feature3/analysis`→`Feature3AnalyzeController`→가격·벤치마크·무위험수익률·overlay 조립→FastAPI `analyze_portfolio`→설명·보강→리포트 snapshot→차트·PDF 흐름입니다.

Windows Java→WSL 실제 uvicorn 검증에서는 합성141가격/140수익률의 공분산·현재비중·최적화 제약·효용우위·일부 벤치마크 부재를 확인했습니다. 실제 Public Controller 처리층에서 요청을 조립해 이 TCP 계산을 수행한 뒤 보고서 서비스에 결과를 전달하고, FastAPI422 시 차감3과 동일한 환불 호출을 수행하는 것도 PASS했습니다. 가격/저장소·차감/보고서는 mock이므로 실제 원장·snapshot 저장·브라우저 PDF를 결합한 완료 증거는 아닙니다. [11개 상세 검사](../backend/src/test/cross-os-fastapi.md).

의도된 가격 정책은 원본 OHLCV 보존, 종목 단위 수정종가 우선·raw fallback, 날짜별 제공처 이어붙이기 금지, 보간 없는 공통 거래일 계산입니다. 공통 날짜 교집합·로그수익률 유틸, Spring Yahoo provider의 결측 경계·fallback은 고정 fixture/client mock으로 검증했습니다. 실제 DB 원본 비변경과 전체 provider 연동까지 끝낸 상태는 아닙니다.

KOSPI/KOSDAQ별 benchmark 선택과 미확인 시장 proxy 경고를 확인했습니다. CAPM sample factor는 60 미만0, 60에서0.5, 119에서0.64, 120에서0.65, 252에서1입니다. beta2·alpha·혼합수익의 독립 fixture, 다중시장 정상 ASGI 요청, 한 시장 benchmark 부재의 부분 fallback, 주요 optimizer에 혼합수익 벡터가 전달되는 경로를 검증했습니다. 실제 시장 데이터의 예측 정확도 증거는 아닙니다.

| 결과 | 의도·구현 역할 | 현재 검증 |
|---|---|---|
| MIN_VOL | 위험자산 100% 최소분산 | 동일 독립3자산 해1/3 검증 |
| MAX_SHARPE | 위험자산 100% 최대 샤프 | 효용 비교 fixture의 입력으로 실행, 독립 최적해 검증은 남음 |
| RISK_ALLOCATION | 최대 샤프 조합+현금, γ·현금상한 | 9개 조건에서 효용최적과 비교 |
| UTILITY_OPTIMAL | 제약 내 `E[R] - γ·분산/2` 최대화 | 해석해0.25씩·제약·효용 우위 검증 |
| THEORETICAL_UTILITY | 제약 밖 CAL 접점 별도 관측 | 전체 결과·표시 미검증 |
| Frontier·Overlay | 일관된 기대수익·공분산 기반 비교 | 정상 전체요청의 frontier 비지배 정렬·fundamentals overlay core 보존·변화상한 검증. 독립 최적성·전체 overlay 조합 미완료 |

위험 점수는 0~1, 실제 위험회피계수 γ는 기본 `10-9×점수`로 구분합니다. 투자 수준 4단계는 설명 난이도입니다. 현재 종목 상한은 자산수1→100%, 2→75%, 3→2/3, 4→50%, 5이상→40%입니다. 이 적응형 정책은 코드에서 확인한 구현 사실이며 추정값이 아닙니다.

Overlay는 재무·기술·뉴스·상관·산업 보조 관측입니다. 의도된 정책상 산업은 무료, 나머지는 원천 캐시 적중 시 무료·미적중/강제갱신 시 과금합니다. 미리보기·실차감·실행의 일치 여부는 전체 검증이 남아 있습니다.

## Feature 4: 용어 사전

`DictionaryMockPage`, `DictTerm`, `DictionaryText`→`api/dictionary.ts`→`DictionaryController`→`DictionaryService`→`DictionaryRepository`·`DictionaryAliasRepository`→정규화·별칭·영문 표시명·DTO입니다. Public API는 검색·자동완성·초성 집계·단어 상세·별칭 조회 5개이며, 관리자 등록·삭제 경로가 별도로 있습니다.

영문 대소문자·공백 정규화, 한글 초성·영문 첫 글자, 검색 LIKE 특수문자 인자, 자동완성 exact/prefix/contains 순서·중복/상한, 별칭→canonical 조회·영문표시명 선택, 관리자 용어/별칭 CRUD·충돌거부를 실제 HTTP 처리층/서비스와 Repository mock으로 검증했습니다. 실제 JPQL·FK/유일성·권한과 업무 handler 결합·사전 내용/출처 정확성은 별도 검증 대상입니다.

## 메인 종목 순위

`RankingStockController`→`TopRankingReader`→Redis 또는 KrStockClient→등록된 EQUITY 필터→종목ID/거래소 보강입니다. limit은1~30입니다. 거래일08:25~08:30(KST)은 캐시를 우회하고,08:30~18:00은10초 TTL, 그 외에는 다음 거래일08:25까지 캐싱합니다. 장전08:25 이전·비거래일에는 이전 거래일 key를 사용합니다. 시계고정·달력/provider/Redis mock으로 경계와 장애 fallback11개 검사를 통과했으며 실제 휴장일 정확성·TTL 만료/동시성은 미검증입니다.

## 공통 기능

JWT와 refresh cookie, 일반 가입 이메일 선인증, OAuth 추가정보, 관심종목·포트폴리오 소유권, 공식 DB 위험 성향과 이번 분석 localStorage 조건, 크레딧 원장, 리포트 소유권·보존기간, PDF가 포함됩니다.

저장 리포트는 `SettingPage`에서 목록→상세 JSON snapshot→`SavedReportDocument`/`ReportHeader`→html2canvas/jsPDF로 재생성합니다. Windows Chrome의 합성 API fixture로 Feature1/2/3 PDF와 상세 재다운로드,7일 보존기간 초과·404 차단을 검증했습니다. 생성된4개 PDF/6페이지는 별도 파싱·렌더링과 한글 육안 확인을 통과했습니다. 실제 DB 스냅샷 연결과 분석 직후 각 화면의 차트/PDF는 별도 검증 대상입니다.

Spring의 실제 저장 로직을 Repository mock과 연결한 회귀 검사에서는 Feature1의 stockCode/companyName 전달 순서가 뒤바뀌는 결함을 재현했습니다(`REPORT-MAPPING-001`). 프론트 PDF fixture 검증이 실제 리포트 저장의 정확성을 보장하지 않는 사례이며 제품 코드는 미수정입니다. 이메일 인증·재설정6개 HTTP 경로는 실제 서비스·해시·MIME/BCrypt를 사용하고 SMTP/저장소만 대체하여 정상·만료·재사용·발송/감사 실패를 검증했습니다.

크레딧 Service mock 검증에서 잔액부족 저장 차단과 차감·환불 산술을 확인했습니다. 리포트 조회는 본인·보존기간 조건을 사용하며 미존재와 타인 자료를 같은 조회결과 부재로 처리합니다. mock은 DB 실제 쿼리·잠금·경쟁 상황을 대신 증명하지 않습니다.

OAuth 실제 제공자 브라우저 로그인은 사용자가 참여하기로 확인했습니다. 실제 인증메일 발송·수신을 확인했고, 가입·로그인·refresh·logout·ID찾기는 실제 HTTP/서비스와 repository mock으로 검증했습니다. 테스트 전용 DB에서 실제 가입·재설정·영속 세션 흐름은 아직 남아 있습니다. 유료 호출은 retry까지 포함하여 보수적으로 세션 전체5회 한도로 기록합니다.
