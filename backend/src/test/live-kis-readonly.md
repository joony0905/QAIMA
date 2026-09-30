# KIS 실제 읽기 연동과 호출 제한 검증

검증일: 2026-09-25. Windows Java17·Gradle8.14. 제품 소스·설정은 변경하지 않았습니다.

## 결과와 범위

호출 제한 단위검사 4 PASS(0.683초), 실제 연동 1 FAIL(3.098초, Gradle18초)입니다. 토큰 POST와 종목 기본정보 GET은 각각 HTTP200을 반환했지만 후속 단언에서 `AssertionFailedError`가 발생했습니다. 실제 연동 전체를 PASS로 집계하지 않습니다.

최초 예외 처리가 원래 단언 메시지까지 제거하여 실패한 단언을 보존 자료만으로 특정할 수 없습니다. 제품 DTO/제공처 결함인지 테스트 기대값 문제인지도 미확정입니다. 재실행 없이 테스트에 고정 단계명 `last_check`를 추가했습니다. 이 보완은 이후 기본 suite에서 컴파일했지만 실제 경로 재검증은 아직 하지 않았습니다. 최초 증거를 새 형식으로 소급하지 않습니다.

[LiveKisReadOnlyTest](java/com/qaima/verification/LiveKisReadOnlyTest.java)는 [LiveInfrastructureTest](java/com/qaima/verification/LiveInfrastructureTest.java)의 YAML 읽기·환경변수 해석을 재사용합니다. application.yml/mysql/secret 설정값을 출력하지 않습니다. 실제 `KrStockClient.fetchSearchInfoRaw("005930", "J")` → 토큰 발급 → TLS 기본정보 조회 → 실제 JSON/업무응답코드 검사 → DTO까지 실행합니다. 프로젝트 RedisConfig의 ObjectMapper만 만들며 Redis 연결은 하지 않습니다. 토큰은 클라이언트 메모리에만 유지합니다.

Spring application context·scheduler·importer·repository는 시작하지 않습니다. DB·Redis 쓰기와 실제 사용자 크레딧 차감은 없습니다. Public API·JWT·브라우저·다른 KIS API·저장 일관성 전체 검증이 아니므로 Swagger 전체 상태도 미완료로 유지합니다. 실제 호출 동안 root logger를 OFF로 설정하고 종료 시 복구합니다. 인증정보·응답 원문·운영 주소는 결과에 남기지 않습니다.

## 메서드별 방법

| 메서드 | 입력·기대값 | 결과 |
|---|---|---|
| `exactTwoRequestsReserveSlotsAndRejectThirdBeforeTransport` | 임시 합성 예약1~3 이후 POST→GET을 mock 전송. 예약2·전송2·상태200/200, 세 번째 전송 거절 | PASS |
| `repeatedRunCannotReuseConsumedTokenSlot` | 동일 임시 디렉터리에 새 guard를 만들어 재실행. 기존 예약4 덮어쓰기 없이 거절, 총전송1 | PASS |
| `wrongDestinationMethodQueryOrSequenceNeverReachesTransport` | GET 선행·토큰 GET·다른 호스트·토큰 query 차단. 유효 POST 후 토큰 재요청도 차단 | PASS |
| `failedTransportConsumesReservationAndCannotAutomaticallyRetry` | 합성 전송 오류에 retry(3)를 적용해도 transport 진입1·예약1 유지 | PASS |
| `configuredKisTokenAndSingleStockMetadataDecodeWithoutPersistence` | 실제 DTO 존재, 상품번호 정확히005930, 상품명 비어있지 않음, 예약/전송 각각2, 상태200/200 기대 | FAIL. 전송2·응답200/200이나 후속단언실패, 최초 기록으로 위치 특정 불가 |

앞의 네 검사는 [KisReadOnlyBudgetTest](java/com/qaima/verification/KisReadOnlyBudgetTest.java)이며 실제 금융 호출은 없습니다. 상품번호 정확 일치는 테스트 기대값이며 실제 제공처 계약을 검증 완료했다는 뜻이 아닙니다.

## 호출 제한과 증거

[KisReadOnlyBudget](java/com/qaima/verification/KisReadOnlyBudget.java)는 설정된 HTTPS origin/port·method/path/query·순서를 제한합니다. `/oauth2/tokenP` POST1, `/uapi/domestic-stock/v1/quotations/search-stock-info` GET1만 허용하며 GET은 시장J·상품005930·상품유형300으로 고정합니다. Netty 자동 재시도·리다이렉트도 비활성화합니다.

기존 SMTP 예약1~3을 확인하고 송신 전에 `CREATE_NEW`로 예약4·5를 생성합니다. 실패해도 예약을 되돌리지 않으며 재실행은 기존 예약에서 차단됩니다. 파일 삭제·번호 변경으로 재시도하지 않았습니다. 보수적 세션 집계는 SMTP3+KIS2=5회이며 실제 청구금액 확인이 아닙니다. 사용자 원래 조건은 **테스트별 오류 수정 후 재시도 포함 최대5회**이고 현재 공통guard는 더 엄격합니다. 테스트별 예산으로 전환하는 작업은 아직 하지 않았습니다.

비공개 증거: `src/test/.runtime/live-kis-12031681897199586530/summary.json` 및 같은 디렉터리에 보존한 JUnit XML. 요약은 outcome=FAIL, reserved_calls=2, http_requests_sent=2, http_statuses=[200,200], exception_type=AssertionFailedError, database_writes=0입니다. 최초 기록에는 last_check가 없습니다. runtime에는 다른 민감자료도 있을 수 있으므로 공개하지 않습니다.

## 재현

[공통 실행 환경](README.md)의 Gradle/TEMP 설정을 적용한 Windows PowerShell에서 외부 호출 없는 검사:

```powershell
Remove-Item Env:QAIMA_LIVE_KIS_READONLY -ErrorAction SilentlyContinue
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test --tests '*KisReadOnlyBudgetTest'
```

실제 실행은 `QAIMA_LIVE_KIS_READONLY=1`과 `--tests '*LiveKisReadOnlyTest'`를 사용했습니다. 소비된 예약을 변경하거나 live 명령을 자동 반복하지 않습니다. 이후 기본 전체 suite는 opt-in 해제 상태에서1053개1002 PASS·34 FAIL·17 SKIP(57초)이며 live1개는 SKIP입니다. 별도 실제 FAIL 기록은 그대로 유지합니다.
