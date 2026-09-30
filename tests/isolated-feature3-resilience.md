# Feature3 overlay·Redis·HTTP 취소/재시도 검증

검증일: 2026-09-29. 최종 `run-p4h4ipqi`: **23개19 PASS/4 FAIL**, errors/skipped0, JUnit182.159초/Gradle5분1초/exit1. 선택 overlay 실패 경고 누락3사례와 기존 준비 단계 환불 누락1사례를 기록합니다. 초기 compile/정상 선택 실행은 누적에서 제외합니다.

## 연결한 경계와 방법

[검사 클래스](java/com/qaima/qa/IsolatedFeature3ResilienceTest.java)는 [원천 SQL 검사](isolated-feature3-source-sql.md)의 실제 가격423행/벤치마크282행/금리 SQL과 실제 Feature3 source service를 재사용합니다. 선택 overlay는 fundamentals/technical이며 Feature3OverlayService의 비용 산정·Redis 읽기/쓰기·신호 조립, 실제 FastAPI 수학·공개 DTO·크레딧 원장·report SQL·소유자 상세 HTTP를 연결합니다.

Feature1 metrics leaf는 종목별 ROE15/영업이익률20/기술적 요약을 가진 합성 입력입니다. 실제 Feature1 전체 원천 실행이나 실제 외부 제공자 검증이 아닙니다. 그 외 가격/벤치마크/BOK의 외부 client 경계도 앞선 source SQL fixture와 같은 합성 입력입니다. 격리 계정의 초기 잔액은20이며 비용3/1·환불·취소 후 잔존을 추적합니다.

- Redis는 소유 서버 앞 TCP 프록시의 실제 단절/바이트 폐기/재시작을 사용합니다. 장애 제거 후 같은 Lettuce client의 재접속을 관측합니다. 복구 관측 한도45초/명령 timeout2초는 테스트 환경 값입니다.
- 원천·overlay·enrich의 subscription gate는 해당 단계 진입/취소를 관측합니다. API gate는 실제 HTTP 본문을 수신한 ASGI 경계에서 대기하고 실제 disconnect를 기록합니다. 실제 계산 후 반환 gate도 별도로 사용합니다.
- [ASGI 관측기](feature1_pipeline_asgi.py)는 소유 실행기가 선택한 Feature1 또는 Feature3 경로 하나만 열며 nonce, 외부 socket/dotenv/subprocess 차단을 유지합니다. 정상/hold 해제는 제품 FastAPI를 실행하고,503/깨진 JSON은 명시적 전송 경계 주입입니다. hold100초는 테스트 정리 상한이며 제품 SLA가 아닙니다.
- 모든 응답의 core 변동성/후보 제약과 overlay 신호, 실제 잔액/원장·report snapshot=소유자 상세·타 사용자404를 기록합니다. 원천 결과·전송 본문·캐시·취소/완료 이벤트·비용 추정도 별도로 남깁니다.
- 취소 환불 시점과 동일 body 멱등성은 공개 정책에서 정하지 않아 관측으로 분리합니다. API/파싱/후처리 오류의 명시 환불, 선택 overlay 누락 경고는 정책 적합성으로 판정합니다.

모든 작성/수정은 literal tests 폴더 안입니다. [실행기](run_isolated_backend.py)가 새 외부 `/tmp` MySQL/Redis·소유 FastAPI/프록시를 만들며 기존 사용자 DB/서비스를 사용하지 않습니다. 실행 중 runner/공유 helper/Java 입력을 수정하지 않습니다.

## 재현

```bash
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite feature3-resilience --test aBaselineColdWarmOverlayKeepsCoreAndPaidSql
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite feature3-resilience
```

## 초기 실행

`run-uffo2ne7`: 테스트 클래스의 DTO/FeatOneResult import 누락으로 컴파일 오류4개가 발생해 compileTestJava에서 종료했습니다. 실행 사례0, 제품 실패가 아닙니다. 소유 서비스 정리·입력 hash 보존을 확인했고 누적에서 제외합니다. import를 보정하고, 재사용 캐시 TTL은 생성 직후5초 이내라는 다른 suite의 전제를 가져오지 않도록0 초과/정책 상한 이하와 실제 잔여 TTL로 확인합니다.

보정 후 정상 선택 실행 `run-ei9ik5t4`는1 PASS(JUnit18.887초/Gradle2분26초)입니다. 실제 응답/계산/SQL상세2개, 원장USE2, 비용3→1, 가격846행/벤치마크564행과 캐시12값을 독립 감사로 대조했습니다. 소유 서비스 종료/615개 실행 입력 hash 보존을 확인했습니다. 전체 실행 `run-p4h4ipqi`를 아래 최종 결과로 확정했습니다.

## 남은 범위

선택하지 않은 industry/correlation/news overlay의 실제 원천 전체 결합·프로세스 강제 종료/다중 backend·멱등성 정책·브라우저 취소와 실제 원천의 화면/PDF 표시·실제 외부 제공자는 계속 남습니다. 전체 QA 미완료·goal active입니다. 최종 결과를 [누적 기록](QA_PROGRESS_2026-09-28.md)과 [상위 인계](FALLBACK_QA_HANDOFF.md)에 반영합니다.


## 최종 사례표

| JUnit 실행 이름 | 결과 | 어떻게 검증했는가 |
|---|---|---|
| aBaselineColdWarmOverlayKeepsCoreAndPaidSql() | PASS | 실제 core SQL/계산+합성 Feature1 metrics → 신호6/SQL상세, cold 비용3→warm1; 최초 leaf4회/후속 추가0 |
| [1] mode=unavailable | PASS | 실제 HTTP503 주입 → 예약3 환불/동일 reference/report0; 준비 캐시로 재요청 비용1/계산·저장 |
| [2] mode=malformed | PASS | 실제 HTTP200의 깨진 JSON → 역직렬화 실패/예약3 환불; 캐시 재사용 복구 |
| [3] mode=validation | PASS | 실제 Pydantic422(잘못된 covarianceModel) → 예약3 환불; 옵션 복원 후 캐시 재사용 복구 |
| [1] stage=PRICE | PASS | 실제 가격 source 구독 전 gate에서 HTTP 취소 → PRICE/HTTP cancel/report0; 같은 요청 cold 비용3으로 복구 |
| [2] stage=OVERLAY | PASS | overlay leaf 진입 gate에서 HTTP 취소 → OVERLAY/HTTP cancel/report0; 같은 요청 cold 비용3으로 복구 |
| [3] stage=API | PASS | 실제 FastAPI HTTP 본문 수신 후 hold에서 취소 → 실제 disconnect/report0; 이미 준비된 캐시로 비용1 복구 |
| [4] stage=ENRICH | PASS | 실제 계산 후 enrich gate에서 HTTP 취소 → API 계산 완료/보고서0; 캐시 재사용 비용1 복구 |
| cacheExpiresAfterEstimateRecordedAsCostSnapshot() | PASS | warm 비용1 확정 뒤 fresh키 삭제 → 실제 leaf 재계산/캐시 재생성/신호6/저장, 추가 차감 없음 관측 |
| cancelledOlderOverlayCannotOverwriteNewerMetrics() | PASS | ROE15 과거 leaf 지연, FORCE_REFRESH 새ROE35 계산/SQL/캐시 완료 후 과거 HTTP 취소·gate 해제 → 새값/기존 snapshot/다음 조회 유지 |
| clientTimeoutCancelsRealFastApiSocketAndAllowsRetry() | PASS | CORE_ONLY 실제 HttpClient3초 timeout/실제 FastAPI disconnect/report0 → 동일 입력 정상 재시도 |
| concurrentIdenticalBodiesUseIndependentReferencesAndReports() | PASS | 동일 body2개를 가격 gate에 함께 대기시켜 각각비용3 확정 → 별도 UUID/리포트2개; 중복 과금 정책 관측으로 분리 |
| enrichmentBoundaryErrorRefundsAfterRealCalculation() | PASS | 실제 계산/기본 enrich 뒤 합성 후처리 오류 주입 → 예약3 환불/report0; 경계 복구 후 캐시 재사용 비용1 |
| [1] scope=one | FAIL | 1종목의 두 overlay leaf 오류 → 정상 신호4/core/저장 보존·복구6은 PASS, 실패종목/overlay 경고 누락 FAIL |
| [2] scope=all | FAIL | 3종목의 두 overlay leaf 오류 → 신호0/core/저장 보존·복구6은 PASS, 선택 overlay 전체 실패 경고 누락 FAIL |
| heldActualApiWaitsThenReleaseProducesPaidReport() | PASS | 실제 API10초 hold에서 미완료/차감3/report0 관측, release 후 실제 계산/신호6/저장 성공; SLA 판정과 구분 |
| malformedMetricsCacheRefetchesSameSourceAndReplacesCache() | PASS | 세 fresh키 깨진 JSON → 같은 leaf 재조회/신호6/캐시 교체/계산·저장 |
| redisAndAllOverlaySourcesFailWithoutLosingCoreWarning() | FAIL | Redis blackhole+모든 overlay leaf 예외 → core/저장 보존·같은 client 복구는 PASS, 실패 안내 누락 FAIL |
| redisAndPriceSqlFailureRefundsReservedOverlayCost() | FAIL | Redis 단절+가격 SQL rename → 예약3 차감/HTTP500/API0/report0/환불0; 복구 후 같은 요청 비용3으로 성공 |
| redisFailureAfterActualCalculationKeepsReport() | PASS | 실제 계산 응답 반환 gate에서 Redis blackhole 주입 → 신호/core/report 보존, 같은 client 복구 뒤 캐시 재사용 |
| [1] mode=partition | PASS | cold Redis TCP 단절 → 실제 leaf 재계산·core/신호6/저장 보존·fresh키0; 같은 client 복구 후 캐시 재생성 |
| [2] mode=blackhole | PASS | cold Redis 바이트 폐기 → 실제 leaf 재계산·core/신호6/저장 보존; 같은 client 복구 |
| redisRestartRebuildsOverlayWithoutMutatingOldReport() | PASS | 실제 Redis 종료/새 runId/빈 cache → 같은 client 재접속·leaf 재계산·캐시 재생성·이전 SQL snapshot 불변 |

## 실패 원인과 영향

### F3-OVERLAY-WARNING-001 — 선택한 overlay의 원천 실패 경고가 신호 변환 중 소실

[Feature3OverlayService](../backend/src/main/java/com/qaima/service/feature3/Feature3OverlayService.java)는 leaf 오류를 `fallbackBundle`로 흡수하지만 fallback에는 카드만 있고 holding row가 없습니다. `toOverlaySignals`가 row를 기준으로 만들기 때문에 해당 종목/overlay가 실제 FastAPI 입력에서 사라집니다. `enrich`도 계산 응답의 신호/시나리오만 보존하므로 fallback 카드의 안내가 최종 공개 응답까지 전달되지 않습니다.

한 종목 실패 시 신호4개, 전 종목 및 Redis 동시 실패 시0개입니다. core 변동성·저장/소유권과 원천 복원 후 신호6개 재생성은 통과했습니다. 그러나 공개 warning에는 시장 혼합·목표 변동성/기대수익 관련 경고만 남고 선택 overlay 실패를 알리는 경고가 없습니다. snapshot/상세도 동일하며 세 사례 모두 비용3이 차감됩니다. core 성공 시 추가 overlay 비용 환불 여부는 정책에 명시되지 않아 별도 과금 결함으로 확대하지 않습니다.

### 기존 F3-CREDIT-001 — Redis+가격 SQL 장애에서도 예약3 환불 누락

Redis 장애로 캐시 미존재를 판단해 비용3을 확정한 다음 실제 가격 SQL이 실패하면 HTTP500/FastAPI0/overlay leaf0/report0, 잔액20→17/USE−3만 남습니다. 두 장애를 제거한 동일 요청은 실제 계산/저장에 성공하고 추가3차감으로 잔액14가 됩니다. 앞선 원천 SQL 단계의 비용1 실패를 동적 비용3/실제 Redis 결합으로 확장한 재현입니다.

## 취소·동시성·비용 관측

- 명시 취소4개(PRICE/OVERLAY/API/ENRICH), 실제 HTTP timeout1개, 과거 overlay 요청 취소1개에서 취소된 요청의 차감이 남았습니다. PRICE/OVERLAY 재시도는 캐시가 없어 비용3, API/ENRICH 재시도는 준비 캐시로 비용1입니다. 취소 전파/늦은 저장 방지 PASS를 취소 환불 정책 PASS로 해석하지 않습니다.
- ROE15 과거 요청을 취소한 뒤 새ROE35 캐시가 유지됐고 새 리포트/후속 리포트2개만 남았습니다. 취소 순간 보이는 기존 리포트1개는 새 요청의 결과입니다. 과거 요청이 늦게 저장된 결과가 아닙니다.
- 동일 body 동시2요청은 각각 예약3/서로 다른 reference/report를 사용했습니다. 공개 idempotency key·중복 요청 환불 규칙이 없어 관측으로 기록합니다.
- 비용 추정 당시 warm이었으나 가격 준비 gate에서 fresh키를 삭제한 경우, 실제 leaf를 다시 계산해도 예약1을 유지했습니다. 사후 추가 차감은 관측되지 않았습니다. 자연 TTL 만료의 시간을 기다린 시험과는 구분합니다.
- 실제 API10초 지연을 해제한 요청은10.419초에 성공했습니다. 관측한 구간에서 완료/환불이 없었음을 확인한 것이며 무한 대기나 운영 SLA 실패의 증거는 아닙니다.

## 시간·증거 대조·정리

[독립 감사기](audit_feature3_resilience.py)로 응답38개, snapshot=소유자 상세33개·타 사용자404, USE44/REFUND4·원장 상태68개를 대조했습니다. 환불4개는 실제 계산 연결/파싱/검증3개와 합성 후처리 오류1개이며 각각 USE와 동일 reference/반대 금액입니다. 실제 FastAPI40회 중 정상 수학35개, 공개 정량과 직접 대응33개, 계산 후 취소/후처리 실패2개, hold 후 실제 disconnect2개를 구분했습니다. 나머지는503/깨진JSON/422 각1개입니다.

실제 전송 가격16,920행/벤치마크11,280행을 SQL seed와 대조했고, 독립 공분산35회·입력→출력 overlay 신호190개·캐시214값을 검사했습니다. overlay 점수0.3/0.2는 합성 입력의 서비스 매핑 확인이며 모델 품질 검증으로 확대하지 않습니다. 같은 입력 후속 복구16개를 기록했습니다.

관측 시간: cold Redis 단절6.477초/blackhole6.468초, Redis+모든 overlay 오류4.344초, Redis+가격 SQL 실패2.242초, 계산 후 Redis 단절 중 결과 보존0.329초입니다. 같은 client 접속 확인63회 중 최대2.502초였습니다. 이 값은 이번 소유 환경의 측정이며 부하/운영 지연 분포를 대표하지 않습니다.

최종 업무9테이블/원천3테이블/QA 종목·추가 지수·숨긴 테이블·trigger0, Redis DB0을 확인했습니다. MySQL/Redis exit0, FastAPI SIGTERM−15/active0/외부 연결·dotenv·subprocess 시도0, 프록시/control/worker 종료와 active socket0입니다. 프록시는 accepted173/rejected44/forwarded302,782 bytes/discarded4,707 bytes를 기록했습니다.615개 실행 입력 hash가 보존됐습니다. 문서73개 검사2개·초기 포함3회 정리·tests 밖924파일 보존·누적30묶음/632개 결과는 실행 폴더의 `history-and-documentation-audit.json`에 기록합니다.

```bash
.venv_wsl/bin/python -B tests/audit_feature3_resilience.py tests/.runtime/backend-isolated/runs/run-p4h4ipqi
```

다음은 industry/correlation/news overlay 조합과 실제 원천을 사용하는 Feature3 브라우저/저장/PDF입니다. 이전 `F3-PRICE-FRESHNESS-001`, 금리 empty, 전체 Feature2 조립/원천/화면 FAIL도 계속 남습니다.
