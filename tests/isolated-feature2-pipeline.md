# Feature2 실제 Peer·뉴스 모델·시장 SQL 폐회로 QA

## 진행 상태

최종 `run-5br21593`: **14개8 PASS/6 FAIL**, JUnit211.698초,errors/skipped0,Gradle5분44초입니다. [증거 감사](audit_feature2_pipeline.py)는 PASS입니다. 앞선 [시장 SQL·무키 설명](isolated-feature2-source-sql.md)의 Peer 계산 및 감성 모델 경계를 실제 Python 코드와 로컬 모델로 확장했습니다. 제품은 수정하지 않았으며 전체 QA는 미완료입니다.

## 실제 경로와 격리

`분석 HTTP/JWT → 크레딧 SQL → 시장 source JPA/Redis → 실제 PeerClusterClient HTTP → Python clustering → 실제 httpx 역방향 요청 → 같은 테스트 Spring의 PeerClusterDataController/PeerClusterDataServiceImpl/JPA → Python 상관/시차/선정 → Spring Peer 변환/Redis → 실제 NewsSentimentService/Feature2NewsSentimentClient → 실제 Python/CPU 로컬 모델 → score SQL/Redis → 실제 설명 API 무키 fallback → 응답/report SQL/상세`.

- 수정은 tests 내부입니다. 소유 외부 MySQL/Redis/TCP proxy와 loopback Spring/FastAPI를 시작하며 기존 DB/서버·제공자 키를 사용하지 않습니다.
- Python outbound는 해당 실행의 loopback Spring 포트 한 곳만 허용합니다. 생성한 nonce 경로는 Spring 필터에서 검증·제거한 뒤 실제 controller/security로 전달합니다. 다른 목적지 연결, dotenv 읽기, subprocess는 차단하고 횟수를 기록합니다. nonce/인증 헤더는 공개 문서·관측 본문에 남기지 않습니다.
- 로컬 모델 `kf_deberta_sentiment_v2`를 읽으며 외부 다운로드를 끕니다. 모델 파일의 크기·SHA256을 실행 전/후 대조합니다. 모델 파일/제품 소스는 수정하지 않습니다.
- source 정상 시 실제 service와 JPA를 호출합니다.503·잘못된 JSON 데이터·timeout은 tests의 source 필터에서 주입하고, SQL 장애는 소유 테이블에서 별도로 주입합니다.
- BOK/FRED/KIS·뉴스 검색/본문 제공자는 앞 단계의 합성 경계입니다. 유료 LLM 생성과 뉴스 observation 비동기 저장, 실제 브라우저는 이번 단계 밖입니다.

## 입력과 독립 기대값

기존 [Peer 수학 fixture](../analysis/tests/peer_fixture.py)의 seed20260925를 이용하되 실제 SQL DECIMAL(18,6)에 들어가는 값으로 먼저 양자화합니다.5종목/산업지수91개 날짜(90로그수익률),동일·2기간 후행·2기간 선행·반대 종목을 구성합니다. 기대 상관은 양자화된 가격의 로그 차이를 대상으로 독립 centered-product 식으로 계산하고 산업 조정은 stock−industry 로그수익률에 적용합니다. 과거 fixture의 연속 날짜이며 실제 거래일 달력이 아닙니다. Java calendar/service는 실제 코드와 격리 holiday repository를 사용합니다.

감성 모델은 합성 한국어3기사 본문을 사용합니다. 예측 방향이 실제 투자 성과를 설명하는지 평가하는 검사가 아닙니다. 확률의 범위/합,3확률의 최대값과label,정책 식 `(positive-negative)*(1-neutral)`과 점수,URL별 매칭,SQL DECIMAL(10,6) 오차를 대조합니다.

## 실행

```bash
python3 -B tests/run_isolated_backend.py --suite feature2-pipeline
python3 -B tests/audit_feature2_pipeline.py tests/.runtime/backend-isolated/runs/run-5br21593
```

[Java 테스트](java/com/qaima/qa/IsolatedFeature2PipelineTest.java), [FastAPI guard/관측기](feature2_pipeline_asgi.py), [fixture/독립 수학 기대값](build_feature2_pipeline_fixture.py), [실행기](run_isolated_backend.py), [소유 FastAPI](isolated_fastapi_fixture.py).

source callback의 실제 httpx timeout은8초,Spring news의 모델 timeout은 코드 기본30초를 유지합니다. HTTP170초/JUnit220초 및 cold 모델 요청 종료를 기다리는100초는 테스트 관측 한도입니다. 제품 SLA를 의미하지 않습니다. 정상 복구는 같은 client/service에서 먼저 관측하고 수동 캐시 삭제 여부를 별도로 기록합니다.

## 초기 구성 보정 이력

- `run-k7gd47z4`: Java의 Spring/Java HTTP wildcard import가 충돌해 compileTestJava에서 중단했습니다. JUnit 사례0이며 tests의 import를 명시형으로 수정했습니다. 소유 서비스는 정리·종료했습니다.
- `run-znz3horl`: 기준1사례 실행에서 응답/SQL JSON 전체의 exact equality가 FAIL했습니다. 서로 다른102개 값은 Peer 실수의 저장/표현 반올림 차이였으며 최대 절대차는1.11e-16이었습니다. 재귀 비교로 구조·문자열·배열 순서를 보존하고 수치만 `max(1,abs(expected))*1e-12` 오차를 허용하도록 보정했습니다. Peer 수학 oracle 자체는1e-10, 감성 SQL DECIMAL(10,6)은0.00000051을 사용합니다.
- 같은 초기 실행의 원천 pack은88일/87수익률이었습니다. 합성 시작일5월1일이 실제 holiday 데이터에서 휴장일이라5월4일로 보정된 결과입니다. 전체 합성 날짜를7일 옮겨 양쪽 경계가 거래일인5월8일~8월6일의91일 oracle로 고정했습니다. 내부 연속 날짜가 모두 실제 거래일임을 주장하지 않습니다.
- 초기 실제 모델 응답은99.186초였습니다. Spring은30초 모델 timeout 후 전체 HTTP37.076초에3기사/점수null/`NEWS_SENTIMENT_FAILED:<id>`와 다른 정량·무키 설명·리포트를 반환했습니다. 테스트는 남아 있는 실제 Python 요청이 끝난 후 정리했습니다. 최초 모델 로딩을 끄거나 timeout을 늘려 이 관측을 숨기지 않았습니다. warmup은 tests 설정에서 비활성화하므로 제품의 일반 기동 성능으로 확대하지 않습니다.
- 위 초기 실행은 누적에서 제외합니다. 제품 변경 없이 테스트 기대값/fixture만 보정했으며 최종 실행의 수치와 별도로 보존합니다.

## 14개 사례와 방법

모든 종합 분석 요청은 HTTP200만 확인하지 않고 응답→report SQL JSON→소유자 상세를 재귀 대조하고,경고 SQL·잔액−1·USE 원장1개·report1개를 확인합니다. 실수 저장 오차 한도는 위와 같습니다. 장애 복원은 같은 Java service/Redis template/Python 프로세스를 사용합니다.

| 메서드/입력 | 검증 방법과 관측 | 결과 |
|---|---|---|
| `aColdRealPeerAndModelThenWarmCachePreserveSqlResults` | 실제 SQL5종목/지수→Python91점/90수익률,동일/후행/선행3개와 상관/시차0,+2,−2 독립 대조. cold 모델 timeout 후 warm3점수 재계산·SQL,Peer 원천1회/Redis 재사용,다른 계정 상세404 | PASS |
| `bReverseTransportFallbackRecoversWithoutManualEviction` / unavailable | 역방향503→다른 카드/3점수/경고 보존. 원천 복구 후 재분석은Peer0,캐시 삭제 후3 | FAIL |
| 같은 메서드 / timeout | 실제 httpx8초 timeout/서버 취소1회→부분 결과. 복구 후Peer0,캐시 삭제 후3 | FAIL |
| 같은 메서드 / malformed | 역방향200의 잘못된 날짜→ValueError 경고/부분 결과. 복구 후Peer0,캐시 삭제 후3 | FAIL |
| `cReverseActualPriceSqlFailureRetainsOtherCardsAndRecovers` | 소유 price_ohlcv RENAME→실제1146/42S02/PRICE_BULK_FETCH_FAILED,나머지 카드/뉴스 보존. 테이블 복구 후Peer0,캐시 삭제 후3 | FAIL |
| `dMissingModelPreservesNewsAndRestorationRecomputesScores` | 실행 중 모델 globals를 보관/해제하고 경로를 소유 미존재 경로로 전환. 실제 Python 모델 누락→3기사/점수null/SQL점수0/Peer 유지. 모델 상태 복원→3점수 저장,누락 경고 제거 | PASS |
| `eCombinedPeerAndModelFailurePreservesIndependentMetrics` | Peer503+모델 누락 동시 주입→macro/수급/공매도/3기사/두 경고 보존. 복원 후 모델3점수는 회복하나Peer0,Peer 캐시 삭제 후3 | FAIL |
| `fIndustryIndexSqlFailureUsesActualRawPeerCorrelation` | 실제 산업지수 SQL 오류→조정 불가/RAW fallback 경고·3Peer 원래 가격 상관을 독립 oracle와 대조. SQL 복원+Peer 캐시 삭제 대조군에서 조정 상관 복구 | PASS |
| `gMissingCandidateRowsKeepRemainingPeersWithWarning` | 동일 종목 후보의 실제 가격 행 삭제→나머지 선행/후행2개 보존·후보 부족 경고,다른 카드/점수 유지 | PASS |
| `hMissingAnchorPricesRetainOtherCardsAndNews` | 기준 종목 가격 행 삭제→Peer0/ANCHOR_PRICE_SERIES_MISSING,나머지 카드/3점수/저장 유지 | PASS |
| `iWindowModeIncludesLatestAvailableStockDate` | from/to 없이 같은91일 SQL자료 요청,종목/지수 최신8월6일 대비 최종Peer8월5일 확인 | FAIL |
| `jInvalidPeerRequestStopsBeforeSqlCallback` | 실제 FastAPI에window1 요청→422/역방향 호출0/차감·report0 | PASS |
| `kActualModelMixedInvalidBatchPreservesValidUrlAndProbabilityMath` | 정상1+빈focus1+빈URL1→실제 정상1추론/INVALID_ITEM warning. URL·3확률 합·argmax label·점수 식 대조 | PASS |
| `lActualModelScoresSurviveSqlInsertFailure` | 실제 모델 추론 뒤 SQL INSERT trigger 오류→3점수/뉴스/경고/리포트 보존,score SQL0. trigger 제거 뒤 유효 캐시 재사용/경고 제거. SQL 자동 보충 여부는 별도 관측 | PASS |

## 실패와 영향

### F2-PEER-ERROR-CACHE-001 — 원천 복구 후 오류 Peer가 정상 캐시처럼 재사용됨

신규 결함1개를5사례에서 재현했습니다. Python은 원천503/timeout/형식 오류나 SQL 빈 pack에 대해 metadata가 채워진Peer0 응답을 반환합니다. 실제 `PeerClusterServiceImpl`은 이를12시간 캐시에 저장하며,사용 가능 여부에서Peer 결과/일시 오류를 구분하지 않습니다. 이후 같은 public 분석 요청은 캐시를 받아 원천을 재조회하지 않습니다.

- 역방향3오류 각각 `sourceRequests`: 장애1→복원 재시도1→캐시 삭제 후2. 복구 재시도에서도 빈 결과와 이전 오류 경고가 SQL report에 저장되고1credit이 차감됩니다. 다른 정량이 유효하므로 차감 자체를 별도 과금 결함으로 판정하지 않았습니다.
- 실패 캐시의 실제 남은 PTTL은43,198,843~43,198,905ms였습니다.12시간 자연 만료까지 기다린 결과는 아닙니다.
- 실제 SQL 복구와 Peer+모델 복수 장애 복구에서도 같은 현상이 발생합니다. 모델은 복원 후 점수를 재계산하지만Peer만0입니다.
- 캐시 삭제 후 세 Peer와 독립 수학 기대값이 회복됩니다. 따라서 원천 복원이 실패한 상황과 구분됩니다. 서비스의 forceRefresh overload가 존재하지만 이 public 분석 경로는 일반9인자 호출을 사용합니다.
- 판단 기준은 사용자의 장애 제거 후 복구 요구와 [fallback 인계 기준](FALLBACK_QA_HANDOFF.md)입니다. [캐시 정책](../policy/cache_policy.md)의12시간 TTL 자체를 위반했다고 주장하지 않습니다. 일시 실패/빈 결과를 정상12시간 캐시로 취급하는 문제입니다.

### F2-PEER-DATE-001 — 기존 최신일 제외가 실제 SQL/저장 리포트에서도 재현됨

window 모드는 실제SQL에서 산업지수 최신시각을 가격 조회의 배타적 상한으로 사용합니다. 동일하게8월6일 가격/지수가 존재하지만 종목 차트는8월5일에서 끝납니다. 명시from/to 경로의91점/90수익률은 정상 대조군입니다. 기존 결함을 새 고유 결함으로 중복 집계하지 않습니다.

## 시간·정밀도·복구 관측

- 최종 cold 모델 실제 응답93.097초,종합 HTTP36.941초. Spring30초 timeout 뒤 점수null/관련 경고를 반환했고 Python 추론은 뒤에 완료됐습니다. 종료를 기다린 뒤 같은 모델/service로 warm 분석은1.185초,실제 재추론0.787초에3점수가 SQL/응답에 반영됐습니다. cold 결과는 나중에 자동 수정되지 않았고 사용자의 후속 분석으로 복구한 것입니다.
- 역방향8초 timeout의 종합 HTTP는9.149초입니다.503은1.154초,잘못된 날짜는1.169초였습니다. 이 소규모 fixture/격리 환경의 관측이며 운영 SLA나 부하 성능 판정이 아닙니다.
- 모델 경로 누락 후 복원은 메모리에 이미 로딩된 모델을 복구하는 방식입니다. 해당 복원 사례를 모델 파일 재다운로드/재로딩 검증으로 부르지 않습니다.
- 실제 모델40개 결과에서 확률 범위/합·label 최대확률·감쇠 점수 식과 버전을 대조했습니다.69개 공개 기사 점수는 실제 모델 결과의URL·score와 일치했습니다. SQL은DECIMAL(10,6) 정밀도를 따릅니다. 이 검사는 학습 품질/투자 예측력 평가가 아닙니다.
- sentiment INSERT 오류 복원 후 SQL score행은 여전히0이고 계산된 유효 점수가 캐시에서 재사용됐습니다. 자동 backfill의 명시 정책이 없어 관측으로 기록합니다. 오류 중에도리포트에는 실제 계산 점수가 보존됩니다.

## 증거·정리·남은 범위

최종 증거는 `tests/.runtime/backend-isolated/runs/run-5br21593/`의 `summary.json`,JUnit XML,`fastapi-exchanges/`,`feature2-pipeline-evidence-audit.json`입니다.

- 실제 HTTP/SQL/상세26개,유일report26개,USE26개/REFUND0. 전체 핵심 실패 검증은 앞 단계이며 이번 사례는 부분 결과가 남는 조합입니다.
- 실제 FastAPI61개: Peer19개(정상/오류200의18개+validation422의1개),뉴스 모델16개,설명26개. 실제 Python→Spring 소유 TCP연결18개. 설명26개는Spring 입력의snake_case 변환 일치/LLM_API_KEY_MISSING을 확인했습니다.
- MySQL8.0.46/V1~V51/48테이블,Redis7.0.15를 사용했습니다. source12테이블·업무9테이블·참조fixture/숨김테이블/trigger/추가Peer종목 최종0. RedisDBSIZE0,프록시/제어/worker 종료/소켓0. MySQL·Redis exit0,Spring callback 포트 종료,FastAPI 의도한SIGTERM exit−15/포트 종료입니다.
- 입력610개와 로컬 모델7파일의 실행 전/후 hash가 같습니다. 별도 `tests/.runtime/backend-isolated/feature2-pipeline-source-audit.json`의 제품 등924개 추가변경/새파일/소실0을 확인했습니다. Python 외부 연결·dotenv·subprocess 시도0입니다.
- 전체 누적은 **24묶음523개412 PASS/111 FAIL**입니다. 실패 사례는 고유 결함 수가 아닙니다. 이번 최종14개만 합산했습니다.
- 다음 우선순위는 이번 실제 원천/계산 결과의 브라우저·저장 상세·PDF 전달,실제 외부 제공자 결합,Feature1 실제 서버 및 남은Swagger/사용자 흐름입니다. 다중 프로세스/취소·재시도 과금,전체 차트·언어·테마·접근성,배치 복구·규모,모델 품질/성능도 남아 있습니다.

문서67개 대상2검사 PASS(로컬 링크·설정 일부 비밀값의 공개 문서 노출 검사)는 `tests/.runtime/backend-isolated/feature2-pipeline-documentation.log`에 기록합니다. 초기2실행/최종1실행의 정리·14개 사례표·누적 집계 대조는 최종 run의 `feature2-pipeline-history-and-doc-audit.json`을 참조합니다.
