# Windows Spring 클라이언트 ↔ WSL uvicorn 실제 TCP 검증

검증일: 2026-09-25. `LiveFastApiTcpTest`는11개 opt-in 검사입니다. Windows Java에서 실제 Spring WebClient/분석 client를 사용해 WSL uvicorn의 실제 FastAPI 라우터·Pydantic·계산·로컬 감성 모델을 호출합니다. DB·Redis·금융 제공처·메일·유료 LLM은 이 검증 범위에 포함하지 않습니다.

최종 실제 실행은 **11 PASS·0 FAIL·0 SKIP**, Gradle BUILD SUCCESSFUL/약2분7초, JUnit 클래스109.499초입니다. audit은 outbound_attempts=0/subprocess_attempts=0/dotenv_read_attempts=0이며 임시 uvicorn 종료까지 확인했습니다. 실행기의 run 디렉터리에11개 실행 XML·junit-summary.json·audit.json을 보존했습니다. 기본 suite에서는 명시적 opt-in 없이 이11개를 건너뛰므로 기본 SKIP을 실제 연동 실패로 해석하지 않습니다.

이후 live opt-in을 모두 끈 기본 전체 회귀는1008개 중961 PASS·32 FAIL·15 SKIP, 약40초입니다. 신규11개 TCP를 건너뛰고 기존 실패32개는 동일했습니다. XML 합계1008/32/0/15를 별도 파서로 대조했으며, 실제 TCP11 PASS의 보관 XML은 기본 회귀 결과와 분리했습니다.

## 실행 방법과 안전장치

프로젝트 루트의 WSL에서 실행합니다. Windows Java/PowerShell과 기존 `.venv_wsl`이 필요합니다.

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 PYTHONPATH=analysis:analysis/tests \
  .venv_wsl/bin/python -B analysis/tests/cross_os_tcp.py
```

실행기는 다음을 수행합니다.

1. `backend/src/test/.runtime/cross-os-tcp/run-*`에 합성 포트폴리오 입력과 독립 공분산 기대값을 만듭니다. `portfolio_fixture.py`의140개 수익률과 NumPy 표본공분산(ddof1)×252를 사용하며, 제품 포트폴리오 계산 함수를 기대값으로 호출하지 않습니다. Java Controller가 직접 종목코드로 인식할 수 있도록 합성 코드만 QAA/QAB/QAC로 바꿉니다.
2. WSL 기본 라우트 인터페이스의 IPv4 주소와 임시 포트를 사용합니다. 주소나 임시 토큰은 공개 문서에 기록하지 않습니다. uvicorn listener에는64자리 난수 토큰 헤더가 없으면403을 반환하는 테스트 전용 ASGI wrapper를 씌웁니다. 제품 라우터의 입력·계산·응답은 바꾸지 않습니다.
3. 자식 서버에서 dotenv·자동 뉴스 warm-up·Feature3 가격 수집을 끄고, 상속된 API key/secret/password/token/proxy 환경변수를 제거합니다. Hugging Face/Transformers는 offline 및 local_files_only 경로를 사용하고 CPU2 thread/배치2/최대256 token으로 기존 로컬 모델을 읽습니다. 모델/패키지를 다운로드하지 않습니다.
4. 테스트 서버의 Python audit hook은 모든 socket.connect와 subprocess 실행을 차단·집계합니다. uvicorn은 asyncio loop를 명시합니다. 요청 경로/status와 차단 시도 수만 audit으로 남기며 임시 토큰·HTTP 본문은 audit에 쓰지 않습니다. 외부 연결 시도가 있으면 이를0회라고 기록하지 않고 검사 실패로 처리합니다.
5. 별도 Windows PowerShell 자식에서 Gradle/TEMP를 test/.runtime에 지정하고 `QAIMA_LIVE_FASTAPI_TCP=1`로 해당 클래스만 실행합니다. 기존 실제 메일/DB생성/인프라 opt-in은 끕니다. Java는 사설/loopback HTTP 주소·고번호 포트·토큰 형식·fixture 경로를 확인한 뒤 실행합니다.
6. Java 종료 후 audit.json을 저장하고 성공/실패에 관계없이 임시 uvicorn을 종료합니다. `uvicorn.log`, fixture.json, audit.json과 Gradle XML/HTML은 test/.runtime에만 남습니다. 로그에는 합성 입력과 테스트 접속정보가 있을 수 있으므로 공개 산출물로 배포하지 않습니다.

최초 preflight는 전체 인터페이스에 복수 IPv4가 있어 중단됐고 서버를 시작하지 않았습니다. 기본 라우트 eth0의 단일 주소를 확인해 실행기 선택 범위를 보완했습니다. 운영 네트워크 설정이나 방화벽은 변경하지 않았습니다.

첫 실제 TCP 실행은10개 중9 PASS·1 FAIL, 약2분19초였습니다. F2 테스트가 빈 metrics DTO를 직접 build하여 macro_trend_summaries/recent_news=null을 보냈기 때문에 실제 Pydantic이422를 반환했습니다. 정상 검사는 제품의 `Feature2ExplainMetricsAssembler.from(Feature2MetricsDto.empty())`와 같은 비null 입력 경로로 변경하여 빈리스트를 생성하는 실제 처리를 통과시켰습니다. `from(null)`이null 목록을 생성해422가 되는 동작은 별도 관측 테스트로 추가했습니다. 일반 assembler 경로도 실패한다고 주장하지 않습니다. 최초 실행도 외부 연결/자식 프로세스 시도0이었고 uvicorn은 종료됐습니다.

두 번째 실제 TCP 실행은11개 중6 PASS·5 FAIL, 약2분9초였습니다. 설치된 python-dotenv의 비활성화 변수는 `PYTHON_DOTENV_DISABLED`인데 기존 테스트 실행기는 `DOTENV_DISABLED`를 사용해 app.main의 .env 재로딩을 막지 못했습니다. 정상 F2 입력이 제공자 분기까지 도달하자 socket.connect 시도2회를 테스트 audit hook이 연결 전에 차단했습니다. 전송된 유료 HTTP 요청은0회입니다. F2의 예상 경고와 달랐고, 뒤4개 검사도 누적 audit≠0 때문에 실패했습니다. 이를 제품 계산의 추가 실패5개로 집계하지 않습니다. 이 실행의 XML/audit은 별도 runtime 디렉터리에 보존했습니다.

테스트 패키지의 비활성화 변수와 문서 명령을 올바르게 고쳤고, TCP wrapper에 .env 열기 차단·앱 import 후 제공자 자격증명 부재 확인을 추가했습니다. 기존 `uvicorn_smoke.py`도 같은 wrapper를 사용하도록 보강했습니다. 제품 .env·서버 코드·설정은 수정하지 않았습니다. Python 기본98개 재실행은 기존92 PASS·3실패메서드·3 SKIP(실패항목5) 그대로이며7.490초, 보강한 WSL smoke10개도 PASS/외부접속·dotenv읽기 시도0입니다.

## 실제 실행과 남은 경계

Java의 `FastApiAnalysisClient`와 `Feature2NewsSentimentClient`는 제품 코드를 그대로 사용합니다. WebClient에는 테스트 서버 주소/임시 인증 헤더·프로젝트 ObjectMapper·16MiB 응답 한도를 명시합니다. 따라서 제품의 전체 WebClient bean 구성/타임아웃/서비스 부팅 설정까지 검증한 것은 아닙니다. 실제 전송은 Reactor Netty→Windows/WSL 네트워크→uvicorn입니다.

Feature3 두 검사는 추가로 실제 Public Controller의 HTTP binding/예외처리→내부 DTO 조립→실제 TCP 계산→Public DTO 변환→보고서/환불 호출까지 연결합니다. Public 측 WebTestClient는 in-process이고 테스트용 principal을 주입하므로 사용자 브라우저·실제 Spring listener·JWT 검증은 포함하지 않습니다. 종목/가격/벤치마크/무위험금리·overlay·CreditService·AnalysisReportService는 합성 경계입니다. 크레딧3 차감/환불은 호출과 구독 횟수만 확인하며 실제 잔액·원장·DB를 바꾸지 않습니다.

Feature1은 합성 OHLCV/재무를 실제 분석 client에 직접 전달합니다. Feature2 설명은 API key가 없는 상태의 실제 경고 전달만 검사하고 유료 제공자 호출은 하지 않습니다. 뉴스 감성은 합성 한국어 기사2개를 실제 기존 CPU 모델로 추론하지만 투자 의미의 정확도/성능을 입증하지 않습니다. Peer cluster의 정상 원천 조회→역방향 Spring 호출과 전체6경로의 모든 성공/실패 분기는 별도입니다.

## 검사11개와 기대값

| 메서드 | 입력·실행·독립 기대값 |
|---|---|
| `serverRejectsMissingNonceAndReportsActualFastApiRoutes` | 토큰없는 실제 TCP health→403. 정상 토큰의 OpenAPI→6경로·Feature3 analysis 등록 |
| `featureOneActualTcpPreservesValuationIndicatorsAndSkippedExplanation` | 종가20의30일/고가21/저가19, 발행10주, 연간매출1000/순익100/영업익150/자본500/자산800/부채300→30개·lastClose20·시총200·PER2·PBR0.4·indicator DTO·explain=null·LLM_EXPLAIN_SKIPPED |
| `featureOneInvalidSubjectReturnsReal422AsAnalysisException` | stockCode=null→실제 Pydantic422→Java ErrorException/ANALYSIS_API_FAILED·status422 표시 |
| `portfolioActualTcpMatchesIndependentCovarianceAndPublicDtoConstraints` | 3종목·두시장·141가격/140수익률→포함3·공통140·현재비중0.2/0.45/0.32/현금0.03, 독립 공분산 변동성과1e-6 내 일치. Public DTO의4최적화후보·합계1/비음수/현금≤0.2/종목≤2/3·효용우위·frontier 확인 |
| `unavailableOneMarketBenchmarkKeepsOtherCapmAndHistoricalFallbackOverTcp` | KOSDAQ benchmark만 가용false/빈자료→QAA CAPM비중>0 유지, QAC CAPM비중0·혼합기대수익=역사기대수익, 전체분석 SUCCESS |
| `portfolioInvalidQuantityIsRejectedByRealPydantic` | 합성 첫보유 quantity0→실제422/Java ANALYSIS_API_FAILED |
| `featureTwoMissingCredentialsReturnsWarningWithoutPaidRequest` | 실제 F2 설명 요청·key 없음→explain=null·LLM_API_KEY_MISSING, 외부 연결 시도0 |
| `featureTwoNullMetricsAssemblerProducesReal422Observed` | 실제 assembler.from(null)→두 목록이null인 내부 JSON→Pydantic422→Java ANALYSIS_API_FAILED. 일반 비null metrics의 빈리스트 대조군과 구분 |
| `actualLocalNewsModelCrossesTcpAndPreservesProbabilities` | 합성 긍정/부정 기사2개→실제 로컬 모델·결과2·경고/invalid0·확률0~1/합1·argmax label·score=(positive-negative)×(1-neutral)·모델/입력버전 보존. 모델 최초로딩 때문에 제한240초 |
| `publicControllerToActualTcpAnalysisReturnsReportAndExactCharge` | Public Feature3 POST→합성 종목/가격 조립→실제 WSL 계산→Public commonReturnSampleSize140·합성 reportId44·차감Mono 구독1·환불0·성공결과 report service 전달 |
| `actualFastApiValidationFailureRefundsControllerChargeAndDoesNotSaveReport` | Public에 covarianceModel=INVALID_FIXTURE→실제 FastAPI422→Public500/ANALYSIS_API_FAILED·차감3 구독1/같은금액 refund1·보고서호출0 |

각 검사 후 서버 audit의 outbound_attempts/subprocess_attempts/dotenv_read_attempts가0인지 확인합니다. 실행기에서 실제 실행된 JUnit XML을 run 디렉터리로 보관하므로 이후 기본 Gradle suite가 opt-in을 SKIP한 XML로 덮어써도 실행 증거가 남습니다. 감성 분류 방향 자체를 기대값으로 고정하지 않고 확률·점수·라벨 정합성을 확인합니다. 성공 후에도 실제 DB·Redis·유료 제공처·OAuth·브라우저까지 연결한 전체 목표는 별도로 남습니다.
