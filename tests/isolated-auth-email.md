# 회원가입·이메일 인증·비밀번호 재설정의 실제 HTTP/JPA 검증

검증일: 2026-09-28. 대상은 `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03`와 기존 미커밋 소스입니다. [테스트 클래스](java/com/qaima/qa/IsolatedAuthEmailJpaTest.java), [실행기](run_isolated_backend.py), 새 문서만 tests 내부에서 작성합니다. 제품 코드·설정·기존 DB는 수정하지 않습니다.

## 검증 방법과 경계

Java HttpClient → 실제 loopback Netty/WebFlux → SecurityConfig/JWT 필터·DTO validation → 실제 AuthController/EmailController → 실제 AuthService/MailAuthService/AuthLoginLogService/LoginSessionService → 실제 Spring 트랜잭션/JPA/Hibernate/JDBC → 새 MySQL8.0.46을 사용합니다. 테스트 자체를 rollback 트랜잭션으로 감싸지 않고, 응답 후 별도 JDBC/Repository 조회로 commit 결과를 확인합니다.

기존 [메일 HTTP 기록](../backend/src/test/README.md)의 mock 저장소 검사를 실제 DB로 확장했습니다. [기본 격리 HTTP 구성](isolated-http-jpa.md)을 가져오되 실제 MailAuthService를 `@Primary`로 등록하고 실제 transaction proxy임을 확인합니다. **JavaMailSender만 메모리 대역**입니다. 실제 MimeMessageHelper/SecureRandom/SHA-256/BCrypt(cost4)를 실행하고 send에 전달된 MIME에서 인증번호·재설정 토큰을 메모리에서 추출합니다. 외부 SMTP나 메일 수신함에는 접근하지 않습니다. 합성 수신자는 `.test`/`.invalid`이고 제품의 example.com 전송 생략에 기대지 않습니다.

MySQL은 실행마다 `/tmp/qaima-qa-http-*` 새 datadir·임시 loopback 포트·무작위 QA schema를 만듭니다. 기존 설정을 읽어 접속하지 않으며 @@port/datadir/DATABASE/소유 표식을 Python·Java 양쪽에서 검사합니다. 원본 V1~V51 SQL을 순서대로 적용해 업무47+소유표식1테이블을 만들고 JPA validate를 통과해야 합니다. Flyway history 검사는 이 실행 범위 밖입니다.

실패 주입은 소유 DB의 `qa_auth_fault` trigger로 users INSERT/UPDATE 또는 auth_login_log INSERT를 거절합니다. 메일 실패는 메모리 send에서 합성 예외를 던집니다. HTTP 응답·독립 SQL·해시 일치 여부·감사 건수로 성공과 rollback을 구분합니다. 인증코드/토큰/비밀번호/MIME 본문은 로그·단언 메시지에 출력하지 않습니다.

동시성은 서로 다른 실제 HTTP 요청2개와 실제 저장소 proxy의 테스트용 MethodInterceptor를 사용합니다. 이메일 확인/가입은 실제 잠금 SELECT **직전** barrier, 재설정은 실제 SELECT가 사용 전 토큰을 읽은 **직후** barrier입니다. `call.proceed()`로 실제 SQL을 실행하며 반환값은 바꾸지 않습니다. 요청 종료 후 advice를 제거하고 executor 종료를 확인합니다. 제품에 barrier/lock을 추가하지 않습니다.

## 재현 명령

MySQL/Gradle 사전 준비는 [기본 실행 문서](isolated-http-jpa.md)를 따릅니다. 프로젝트 루트에서:

```bash
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite auth-email
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite auth-email --test concurrentPasswordResetMustConsumeTokenOnlyOnce
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

전체 클래스26개이며 선택 명령은 후속 재현용입니다. 이번에는 전체 실행1회만 했습니다. 선택 실행을 추가하더라도 동일 사례이므로 전체 수에 더하지 않습니다. 정상 동작을 기대하는 단언을 유지하므로 제품 결함이 재현되면 Gradle 종료코드는1입니다.

## 사례별 입력·단언

| 테스트 메서드 | 검증한 방법 |
|---|---|
| signupPersistsNormalizedVerifiedUserThenRealLoginAndJwt | 대문자 이메일 request→실제 MIME6자리→독립 SHA-256/600초 TTL→confirm→signup→JDBC/JPA. 소문자 email·전화번호 숫자·7자리 생년월일·국가 trim·인증시각·5크레딧·BCrypt·민감필드 미응답·실제 login/JWT/session 확인 |
| signupRequiresConfirmedUnexpiredCodeAndDoesNotConsumeOnRejection | confirm 전 가입400/사용자0/토큰1. confirm 후 SQL로 만료시켜 가입400/토큰 보존 |
| signupSqlFailureRollsBackConsumedCodeAndRetrySucceeds | 확인된 토큰을 consume한 뒤 users INSERT trigger 실패→500/사용자0/토큰·usedAt 복구/실패 감사. trigger 제거 후 같은 코드로 가입200·토큰 삭제 |
| signupInvalidBirthdateRollsBackCodeConsumption | 토큰 consume 뒤 생년월일 validation400→토큰 복구, 정상 입력 재시도 성공 |
| duplicatePhoneKeepsVerificationAndExistingUser | 기존 하이픈 전화번호와 신규 숫자 정규화 결과 충돌→400, 기존 사용자·가입 인증 보존 |
| existingUnverifiedUserConfirmationCommitsDirtyCheckedFields | 미인증 login400→request/confirm→실제 users 인증상태/시각·토큰 user FK/usedAt commit→login200 |
| verificationSqlFailureRollsBackUserAndTokenThenRetrySucceeds | confirm 중 users UPDATE 실패→500/인증false/usedAt null 복원, 제거 후 성공 |
| verificationRejectsWrongEmailWrongCodeExpiredAndRepeatedUse | 다른 email·범위 밖 코드·만료·재사용400과 실패 감사4건. 유효 코드 앞뒤 공백은 trim 후 성공 |
| verifiedAccountRequestAndConfirmRemainNoOpsObserved | 이미 인증된 계정의 request 및 잘못된 코드 confirm도200/새 토큰·메일0이라는 기존 동작 확인 |
| verificationCooldownAndResendReplaceOnlyUnusedRows | 60초 안 재요청은 메일/행 추가0. created_at을61초 전으로 이동한 재발급은 미사용 이전행 삭제·이전코드400. 확인된 행은 다음 발급 때 보존됨 |
| verificationMailFailureRestoresPreviousRowAndRetryRecovers | 재발급 send 예외→500/이전 토큰 ID 보존/실패 원문 미응답. send 복구 후 새 행으로 교체 |
| cleanupDeletesExpiredUsedAndUnusedButKeepsFutureRows | 만료 used/unused 두행·유효 한행 fixture→실제 cleanup 메서드 호출/SQL 삭제→유효1만 보존·반복 멱등. 예약 콜백 검사는 아님 |
| passwordResetPersistsHashAndUsedAtRejectsReplayAndChangesLogin | MIME 링크 토큰32bytes·SHA-256/1800초→confirm→passwordHash/usedAt commit. 재사용400·이전 암호 login400·새 암호 login200 |
| passwordResetUnknownEmailAndCooldownDoNotSendOrCreateExtraRows | 미가입 email200/토큰·메일0. 가입자 첫 요청 후 대문자 email 재요청200/기존ID/메일1 유지 |
| passwordResetResendDeletesOldTokenAndNewTokenWorks | created_at61초 전 이동→재발급→기존 토큰400/신규 토큰200/SQL1행 |
| passwordResetMailFailureRestoresPreviousToken | 재발급 send 실패→500/이전ID 복원·발송 성공1. 이전 토큰으로 실제 비밀번호 변경 성공 |
| passwordResetSqlFailureRollsBackPasswordAndUsedAtThenRetrySucceeds | users UPDATE trigger 실패→500/기존 BCrypt 유지/usedAt null·실패 감사. 제거 후 같은 토큰 성공 |
| passwordResetInvalidExpiredAndShortInputLeaveStateUntouched | 미등록 토큰·7자리 비밀번호·SQL 만료400→기존 암호/usedAt 유지 |
| jsonKeepsPasswordWhitespaceWhileFormTrimsItObserved | JSON의 공백 포함 암호는 그대로 BCrypt 일치, 다음 토큰의 form 제출은 token/password trim 후 저장·HTML200 |
| resetHtmlEscapesTokenAndMissingFormValuesCannotMutateDb | HTML token query의 따옴표/태그/앰퍼샌드 entity escaping. token/password 각각 빠진 form400·DB 토큰0 |
| resetDoesNotRevokeExistingSessionOrJwtObserved | 먼저 실제 login→reset→기존 access로 보호 API/기존 refresh cookie로 rotation. 응답·session행을 관측; 정책 승인 여부는 별도 |
| auditInsertFailureDoesNotUndoSuccessfulReset | reset 요청 이후 감사 INSERT trigger 오류→비밀번호 변경200/usedAt commit/성공 감사0 |
| invalidJsonDtosStopBeforeSqlAndMail | 이메일4개 JSON endpoint와 signup의 빈 입력 및 잘못된 email400→인증/재설정/감사행·메일0 |
| concurrentVerificationUsesActualPessimisticLockAndOnlyOneSucceeds | 가입 전 같은 코드로 실제2요청이 잠금 SELECT 직전에 도달→정확히200/400·usedAt commit |
| concurrentSignupConsumesLockedCodeOnlyOnce | 확인된 같은 코드로2가입 요청을 잠금 SELECT 직전까지 진행→상태200/400·사용자1/토큰0 기대 |
| concurrentPasswordResetMustConsumeTokenOnlyOnce | 같은 재설정 토큰을 실제2 SELECT가 미사용으로 읽은 직후 해제→서로 다른 새 암호로 요청. 하나만200 기대, 최종 암호/성공 감사/순차 재사용400 함께 기록 |

## 실행 결과와 증거

증거는 `tests/.runtime/backend-isolated/runs/run-m0rwa02v`의 `summary.json`, `TEST-com.qaima.qa.IsolatedAuthEmailJpaTest.xml`, `gradle.log`입니다. **26개:25 PASS/1 FAIL**, errors/skipped0, JUnit20.407초입니다. Gradle 전체는2분19초이며 DB 준비 시간을 포함하지 않습니다. 컴파일·테스트 구성 오류는 없었고 보정 재실행도 없습니다. 위 표에서 마지막 동시 재설정 사례만 FAIL이고 나머지25개는 PASS입니다.

### 신규 AUTH-RESET-RACE-001 — 같은 재설정 토큰의 동시 소비가 모두 성공

1. 실제 사용자와 이메일 링크 발급으로 pwd_reset1행을 commit합니다.
2. 서로 다른 새 암호를 제출한 두 HTTP 요청의 실제 `findByTokenHash` SELECT가 각각 usedAt=null을 읽습니다. 두 읽기가 끝난 뒤 barrier를 해제합니다.
3. 기대 상태는 `[200,400]`이지만 실제는 **`[200,200]`**이고 성공 감사도2행입니다. 최종 BCrypt는 첫 요청 암호에만 일치했습니다. 어느 요청 값이 마지막에 남는지는 실행 순서에 따라 달라질 수 있습니다.
4. advice 제거 후 순차 재사용은400입니다. 단순 재사용 방지 검사는 통과하면서 동시에 이미 읽은 두 요청은 모두 성공합니다. 두 개의 암호가 동시에 저장된다는 의미는 아닙니다.

원인 근거: [PwdResetRepository](../backend/src/main/java/com/qaima/repository/PwdResetRepository.java)의 조회에는 잠금이 없고 [PwdReset](../backend/src/main/java/com/qaima/domain/PwdReset.java)에도 version 필드가 없습니다. [MailAuthService](../backend/src/main/java/com/qaima/service/auth/MailAuthService.java)는 조회한 usedAt을 검사한 뒤 사용자 암호와 usedAt을 변경합니다. 사용 전 확인과 갱신 사이에 다른 요청이 같은 검사를 통과할 수 있다는 해석이 실제 두 SQL 읽기·두200·성공 감사2행과 일치합니다. 조건부 원자 갱신이나 토큰 잠금과 재확인을 고려할 수 있으나 **제품은 수정하지 않았습니다**. 동시성 gate는 가능한 순서를 재현하며 운영 발생 빈도나 여러 인스턴스 시험은 아닙니다.

대조군: EmailVerificationRepository의 실제 `PESSIMISTIC_WRITE` 조회를 거친 동일 코드 확인은200/400, 같은 확인 코드로 가입도200/400·사용자1/토큰0이었습니다. SMTP send 오류 때 이전 토큰의 삭제/새 토큰 저장이 함께 rollback했고, users INSERT/UPDATE 오류 때 토큰 소비·인증상태·암호 변경도 rollback했습니다. 실패 감사는 별도 저장됐습니다.

### 정책 관측과 한계

재설정 이후 기존 access token으로 보호 API200, 기존 refresh cookie로 rotation200, login_session1행 유지였습니다. 세션 폐기 요구사항의 판단 근거가 부족해 별도 결함으로 세지 않습니다. 이미 인증된 계정의 잘못된 인증코드 confirm200, JSON과 form의 암호 공백 처리 차이도 기존 정책 관측으로 기록했습니다. 감사 INSERT가 실패해도 재설정은200/commit이었으며 실패 감사의 내구성 보장 검사는 아닙니다.

### 정리·변경 범위

실행 입력566개 SHA-256은 실행 중 변경0입니다. Java가 각 사례 후 소유 표식을 확인해 인증2테이블부터 지우고 기본 업무9테이블을 정리했으며, Python이 별도 CLI로 업무9+인증2테이블 전부0·`qa_auth_fault` trigger0을 재확인했습니다. 자신이 시작한 MySQL은 exit0/stopped true입니다. `auth-email-source-audit.json`의 기존 tests 밖924개 비교도 변경/소실/새파일0입니다.

문서 검사는 새 문서를 포함한53개 대상으로 **2개 검사 PASS**이며 결과는 `tests/.runtime/backend-isolated/auth-email-documentation.log`에 보존합니다. 로컬 링크·설정 일부 비밀값 검사이며 전체 보안 감사를 뜻하지 않습니다.

## 남은 범위

전체 QA는 계속 진행 중입니다. 이번 범위는 인증 관련 선택된 실제 HTTP/DB 경계입니다. OAuth, 실제 SMTP·수신, frontend와 실제 API 결합, 장기 만료/자연60초 경계, RNG 충돌·전역 코드 유일성, 재발급 경쟁, cleanup 예약/서버 중단, 여러 backend 인스턴스, 탈퇴/비활성 계정 재설정 정책, 재설정 후 세션 폐기 요구사항, 성능·로그인 시도 제한은 별도입니다. SQL로 시간을 이동한 검사는 실제 장시간 경과 검사가 아닙니다. HTTP200 동일성은 시간차에 의한 가입 여부 추정까지 배제하지 않습니다.
