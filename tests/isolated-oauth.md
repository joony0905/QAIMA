# OAuth 콜백·소셜 계정·추가정보의 실제 HTTP/JPA 검증

검증일: 2026-09-28. 대상은 `dev` HEAD `345aabb5075964e3ddd07ddd351915e330862e03` 및 기존 미커밋 소스입니다. [테스트 클래스](java/com/qaima/qa/IsolatedOAuthJpaTest.java)와 [실행기](run_isolated_backend.py), 문서는 tests 내부에서만 작성했습니다.

## 검증 방법

이번 구성은 HttpClient와 CookieManager → 실제 AuthOAuth2Controller → Spring Security OAuth authorization redirect/서버 WebSession/state → **테스트 소유 loopback 제공자 HTTP 서버**의 authorize/token/userinfo → 실제 OAuth2LoginSuccessHandler/FailureHandler → OAuth2SocialLoginService/LoginSessionService/AuthLoginLogService → 실제 JPA/트랜잭션 → 새 MySQL8.0.46입니다. 제공자의 승인 화면은 합성 redirect이고 토큰 JSON·사용자정보 JSON은 fixture입니다. 토큰 교환과 userinfo의 실제 HTTP client/codec, 앱 세션·refresh cookie·JWT 발급/보호 API는 제품 코드를 실행합니다.

Google/Kakao/Naver 등록의 endpoint만 소유 제공자 주소로 지정하며 실제 계정·외부 provider·SMTP에는 접근하지 않습니다. `scope=profile,email`의 OAuth2 authorization-code 흐름입니다. Google OpenID Connect discovery/ID token/JWK 검증, 실제 제공자의 인증 화면·동의·MFA·제한·운영 설정은 이 검사의 대상이 아닙니다. Java HttpClient는 실제 브라우저의 SameSite/Secure 정책을 검증하지 않습니다.

제공자는 loopback의 동적 포트에만 bind합니다. authorize는 client ID/response type/state 유무/앱 소유 포트의 정확한 callback을 검사하고 일회성 코드를 만듭니다. token은 POST·Basic client 인증·grant type·발급 코드/등록명을 확인하고 userinfo는 발급한 Bearer 토큰/등록명을 확인합니다. 기대 외 경로/요청은 violations로 누적합니다. 테스트 GET도 앱·제공자의 소유 loopback 포트만 허용하며 최종 `qa-ui.invalid` 성공/실패 redirect는 따라가지 않습니다. 원본 코드·state·토큰·cookie·암호 값은 로그에 남기지 않습니다.

DB 구성은 [기본 HTTP/JPA 검사](isolated-http-jpa.md)를 재사용합니다. 실행마다 `/tmp/qaima-qa-http-*`에 새 datadir/임시 포트/무작위 QA schema, 제품 YAML 연결 없이 소유 표식을 검증합니다. V1~V51 SQL 숫자 순서 적용과 JPA validate를 거칩니다. SQL fixture·오류 trigger는 소유 DB에만 쓰고 테스트를 rollback으로 감싸지 않습니다. 응답 후 독립 JDBC/Repository 조회로 실제 commit을 확인합니다.

`qa_oauth_fault` trigger로 social_account/login_session/auth_login_log INSERT 또는 social_account/users UPDATE를 거절합니다. 사용자·연결 계정 생성이 같은 transaction에서 rollback되는지와, 생성 이후 세션 발급 실패가 앞선 commit을 보존하는지를 나누어 확인합니다. 동시 신규 로그인은 실제 users.findByEmail SELECT 두 개가 없음으로 돌아온 **후** repository proxy의 MethodInterceptor/barrier로 해제합니다. 반환값을 대체하지 않으며 테스트 후 advice를 제거하고 executor를 종료합니다.

실제로 호출한 제품 경로는 `GET /api/v1/auth/oauth2/{provider}`, `GET /oauth2/authorization/{provider}`, `GET /login/oauth2/code/{provider}`, `POST /api/v1/auth/refresh`, `POST /api/v1/auth/login`, `PATCH /api/v1/users/me/social-profile`, `GET /api/v1/users/me`, `GET /api/v1/credits/balance`입니다. Security가 제공하는 OAuth 경로는 Swagger 업무 endpoint 전체 검증과 구분합니다.

## 재현 명령

MySQL·Gradle 준비는 기본 HTTP 문서를 따릅니다. 프로젝트 루트:

```bash
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite oauth
.venv_wsl/bin/python -B tests/run_isolated_backend.py --suite oauth --test inactiveIncompleteAccountMustNotGetSessionOrReactivateThroughOAuth
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

선택 실행 명령은 후속 재현용이며 전체25개와 중복 합산하지 않습니다. 제품 결함 단언은 실패로 유지하므로 실행 종료코드1이 정상 동작의 보증은 아닙니다.

## 사례별 입력과 검증

| 메서드 | 입력·검증 방법 |
|---|---|
| allThreeProvidersUseRealRedirectExchangeUserinfoAndDatabase | 세 등록별 entry→authorize/state/cookie→token POST→userinfo GET→성공 redirect/refresh cookie. SQL 사용자·연결·세션3개, 각 provider signup/login 감사1, 암호null·인증시각·profile_required·5크레딧·실제 refresh JWT 확인 |
| completeSocialProfileThenReloginKeepsIdentityAndSyncsProviderMetadata | 최초 OAuth→refresh JWT→추가정보 PATCH→active/전화·7자리생년월일·성별·국가. provider ID 유지·email/name 변경 후 재로그인→profileRequired=false·사용자/연결ID 유지·provider 메타만 변경·세션2. 반복완료·암호 없는 로컬login400 |
| existingEmailRequiresExplicitLinkAndDoesNotCreateSocialSession | 기존 로컬 email의 대문자 provider 응답→ACCOUNT_LINK_REQUIRED·사용자1/연결0/세션0·기존 BCrypt 유지·실패감사 |
| unverifiedGoogleProfileCannotCreateAccount | Google email_verified=false→PROVIDER_EMAIL_NOT_VERIFIED·업무행0·실패감사 |
| missingProviderEmailCannotCreateAccount | subject는 있지만 email 없는 사용자정보→PROVIDER_PROFILE_INVALID·업무행0 |
| kakaoConsentMissingPreventsAccountEvenWithVerifiedEmail | email/인증=true와 email_needs_agreement=true→PROVIDER_PROFILE_INVALID·업무행0 |
| naverNestedNicknameAndKakaoFallbackNameReachSql | Naver response.nickname·Kakao properties.nickname fallback, 숫자 provider ID와 문자열 "true"의 실제 JSON 변환→SQL 이름/ID 확인 |
| sameProviderSubjectAcrossProvidersRemainsDistinct | Google/Naver의 동일 subject 문자열·다른 email→사용자2/연결2, provider와 subject의 복합 identity 확인 |
| unsupportedEntryIs404AndMixedCaseEntryNormalizesProvider | 알 수 없는 provider404, GoOgLe entry는 google authorize path302·SQL/제공자호출0 |
| mismatchedStateStopsBeforeProviderTokenExchange | authorization session을 만든 뒤 callback state만 변경→실패 redirect·token 호출0/업무행0 |
| callbackWithoutAuthorizationSessionCannotExchangeCode | 정상 시작으로 받은 callback을 새 cookie jar로 호출→실패·token0/업무행0 |
| providerDenialUsesFailureRedirectAndClearsSession | 제공자 access_denied callback→실패 redirect·SESSION 만료쿠키·token0/업무행0 |
| tokenAndUserinfoTransportErrorsDoNotCreateRows | token HTTP400과 userinfo HTTP500→일반화된 실패 redirect·업무행0·실제 경계 호출건수 확인 |
| successfulCallbackClearsWebSessionAndReplayCannotIssueAnotherRefresh | 성공 후 SESSION Max-Age0, 같은 callback 재호출 실패·token추가0/세션1. JWT 없이 cookie jar만 가진 /users/me401 |
| socialInsertSqlFailureRollsBackNewUserAndRetrySucceeds | users saveAndFlush 다음 social INSERT trigger 오류→실패 redirect·사용자/연결/세션0. trigger 제거 후 같은 provider 새 인증 성공 |
| sessionInsertFailureLeavesCommittedAccountAndRetryReusesItObserved | login_session INSERT 오류→사용자/연결1 및 signup 감사 commit·세션0. 재인증은 같은 사용자ID로 세션1·signup 감사추가0 |
| auditSqlFailureDoesNotPreventSocialSignupOrSession | 감사 INSERT 오류→성공 redirect·사용자/연결/세션1·감사0 |
| providerMetadataUpdateSqlFailureKeepsOldValuesAndNoNewSession | active 계정의 provider email 변경 UPDATE 오류→실패 redirect·기존 메타/세션수 유지. 제거 후 재로그인 성공 |
| profileValidationAndDuplicatePhoneLeaveOriginalRowsUntouched | 잘못된 성별자리·빈 추가정보·다른 사용자의 정규화된 전화번호 충돌400→profile_required/기존 null 필드 유지 |
| socialProfileSqlFailureRollsBackAndRetryCompletes | 추가정보 users UPDATE trigger 실패500→상태/phone 유지. 제거 후 같은 JWT의 PATCH200·active |
| inactiveCompleteAccountCannotLogInThroughOAuth | 추가정보 완료 후 SQL로 inactive 전환·기존세션 제거→ACCOUNT_INACTIVE·세션0·상태 유지 |
| inactiveIncompleteAccountMustNotGetSessionOrReactivateThroughOAuth | 추가정보가 없는 계정을 inactive로 변경·기존세션 제거→새 OAuth 시도. 새 cookie/세션 거부 기대; 실제 발급되면 refresh→추가정보 PATCH까지 수행하여 최종 상태 기록 |
| existingAccessTokenMustNotReactivateInactiveIncompleteAccount | 실제 OAuth/refresh로 발급한 JWT를 보관하고 SQL로 inactive 전환→추가정보 PATCH 거부·inactive 유지 기대 |
| pendingProfileGetsJwtAndProtectedBalanceObserved | profile_required 계정의 refresh JWT로 credits/balance·users/me200 관측. 이 단계의 업무 접근 허용 정책을 승인한 검사는 아님 |
| concurrentNewSocialLoginKeepsSingleUserAndMappingObserved | 서로 다른 cookie jar2개의 신규 로그인, 실제 email SELECT 두 개가 없음인 시점에 gate→동일 email DB유일성으로 사용자/연결/세션1만 보존·성공/오류 redirect와 후속 재로그인 확인 |

## 실행 결과

### 테스트 구성 보정 이력

- `run-kgqkk6c7`: context 초기화 실패1개, 실제25개 메서드는 실행하지 못했습니다. 기본/추가 ReactiveClientRegistrationRepository 두 bean을 유지한 구성에서 SecurityWebFilterChain 생성이 `clientRegistrationRepository cannot be null`로 실패했습니다. `@Primary` 추가만으로 선택되지 않아 테스트의 bean override를 허용하고 같은 `registrations` 이름으로 기본 등록을 교체했습니다. 제품 설정은 변경하지 않았습니다. JUnit initializationError0.002초, 실제 OAuth 사례 결과로 세지 않습니다. provider manifest도 없는 초기화 실패 이력입니다.
- `run-tgt7dsqq`: 전체25개 중12 PASS/13 FAIL, JUnit15.293초. 실패12개는 공통 성공 helper가 쿠키 문자열의 `HttpOnly` 철자를 대소문자 구분해 검사한 테스트 오류로 그 시점에 중단됐습니다. HttpCookie 파서의 isHttpOnly/path 검증으로 보정합니다. 나머지1개는 기존 JWT로 비활성 계정을 active로 바꾸는 실제 제품 실패입니다. 실패를 숨기거나13개 제품 결함으로 세지 않습니다. 제공자 authorize30/token27/userinfo26, violations0·포트 종료 true, SQL 정리·MySQL 정상 종료를 확인했습니다.

### 최종 결과

`run-faq332kk`: **25개23 PASS/2 FAIL**, errors/skipped0, JUnit18.370초입니다. Gradle2분39초는 DB 준비 시간을 포함하지 않습니다. 위 표에서 `inactiveIncompleteAccountMustNotGetSessionOrReactivateThroughOAuth`, `existingAccessTokenMustNotReactivateInactiveIncompleteAccount` 두 개가 FAIL이고 나머지23개는 PASS입니다. 두 실패 메서드의 단언6개를 별도 테스트나6개 결함으로 합산하지 않습니다.

증거 디렉터리는 `tests/.runtime/backend-isolated/runs/run-faq332kk`이며 `summary.json`, `TEST-com.qaima.qa.IsolatedOAuthJpaTest.xml`, `gradle.log`, `oauth-provider.json`을 보존합니다. 이전 두 실행도 같은 runs 아래 남겨 최종 결과와 비교할 수 있습니다.

### 신규 AUTH-SOCIAL-INACTIVE-001 — 추가정보 누락 계정의 비활성 상태 우회

| 재현 경로 | 독립 SQL·HTTP 관측 | 기대 |
|---|---|---|
| 새 OAuth 로그인 | 먼저 Google 합성 제공자로 실제 계정 생성→추가정보4필드가 비어 있는 상태에서 users.status=inactive commit·기존 login_session 삭제→같은 provider subject로 새 authorization 흐름. 성공 redirect·새 refresh cookie·login_session1행·refresh200. 새 JWT로 social-profile PATCH200→최종 users.status=active | inactive 계정 로그인 실패·새 세션0·상태 유지 |
| 기존 JWT | 실제 OAuth→refresh JWT를 보관→users.status=inactive commit→같은 JWT로 social-profile PATCH200→최종active | 비활성 계정 추가정보 완료 거절·inactive 유지 |
| 대조군 | 실제 추가정보 완료 후 inactive 전환·세션 삭제→새 OAuth는 ACCOUNT_INACTIVE redirect·세션0·inactive 유지 | PASS |

원인 근거는 [OAuth2SocialLoginService](../backend/src/main/java/com/qaima/service/auth/OAuth2SocialLoginService.java)의 `loginExistingAccount`에서 profile_required 또는 추가정보 누락을 먼저 검사해 세션을 발급한 다음에 active 조건을 검사하는 순서입니다. [UserService](../backend/src/main/java/com/qaima/service/user/UserService.java)의 `canCompleteSocialProfile`도 추가정보가 비어 있으면 상태와 무관하게 허용하고 이후 status를 active로 설정합니다. `JwtAuthFilter`는 이 PATCH에서 JWT를 검증하지만 계정 상태를 별도로 조회하지 않습니다.

두 경로의 상태 검사를 하나의 신규 결함으로 묶어 기록합니다. 실제로 검증한 조건은 Google 형식의 검증된 합성 profile, 유효한 provider subject 또는 기존 JWT, 추가정보4필드 누락, `inactive` 상태입니다. 임의 사용자로 로그인하거나 자격증명 없이 계정을 바꿀 수 있음을 보인 것은 아닙니다. 다른 상태명·한 필드만 누락·Kakao/Naver의 같은 결함 경로·토큰 만료 뒤·운영 계정 비활성화 API까지 실행한 결과로 일반화하지 않습니다. 제품 수정 없이 기대 단언의 FAIL을 유지했습니다.

### 통과한 저장·실패 경계와 정책 관측

- Google/Kakao/Naver 세 합성 등록의 실제 token/userinfo HTTP와 신규 계정·refresh→JWT는 각각 성공했습니다. provider ID가 같고 email/name이 바뀐 재로그인은 동일 사용자와 연결을 유지하면서 provider 메타만 변경했습니다. 기존 로컬 이메일은 ACCOUNT_LINK_REQUIRED로 거절했습니다.
- state 불일치·cookie 없는 callback·제공자 거절은 token 호출0으로 차단됐고, token400/userinfo500도 DB 사용자·연결·세션0이었습니다. 성공 후 서버 WebSession 만료 cookie와 callback 재사용 거부를 확인했습니다.
- social INSERT 실패는 앞선 user INSERT까지 rollback했습니다. 반면 session INSERT 실패는 user/social 각1과 signup 감사를 남겼고 재인증은 같은 사용자로 세션1을 발급했습니다. 이 부분 commit은 관측이며 전체 로그인 원자성을 입증하지 않습니다. 감사 실패에도 사용자·연결·세션은 commit됐습니다.
- 신규 로그인2개의 실제 email SELECT가 모두 없음인 시점부터 진행해 DB 사용자/연결/세션1만 남았고 redirect는 success/error 각1이었습니다. 실패 요청까지 멱등 성공시키는 정책은 주장하지 않습니다. 다음 순차 재로그인은 성공했습니다.
- profile_required 계정의 JWT로 잔액·본인 프로필 조회200은 현재 동작 관측입니다. 제품의 추가정보 완료 전 업무 허용 정책 승인은 아닙니다.

### 종료·변경 범위

세 실행 모두 입력567개 SHA-256 변경0, 기본 업무9테이블·social_account0·`qa_oauth_fault` trigger0, MySQL exit0/stopped true입니다. 첫 실행의 provider 종료 manifest는 없으므로 제공자 종료를 직접 측정한 두 번째/최종 실행과 구분합니다. 최종 제공자 authorize38/token35/userinfo34, violations0이고 HTTP 서버 중지→executor 종료→같은 loopback 포트 연결 거부를 확인했습니다. 이 전송은 전부 합성 로컬 요청입니다.

`tests/.runtime/backend-isolated/oauth-source-audit.json`은 tests 밖 기준924개를 다시 읽어 변경/소실/새 가시 파일0을 확인한 기록입니다. 문서54개 대상 **2개 검사 PASS** 결과는 `tests/.runtime/backend-isolated/oauth-documentation.log`에 기록합니다. 링크와 설정 일부 비밀값 검사이며 전체 보안 감사는 아닙니다.

`oauth-method-audit.json`으로 소스25개 메서드/문서25개 사례/최종 JUnit25개 이름도 일치함을 확인했습니다. 초기 대조 스크립트의 Unicode 단어 패턴이 한글 표머리2개까지 포함했던 결과는 `oauth-method-audit-initial.json`에 남겼고, 실제 Java 메서드명에 맞는 ASCII 패턴으로 보정했습니다. 이 문서 집계 보정은 제품 테스트 실패에 포함하지 않습니다.

## 남은 범위

실제 제공자 로그인의 사용자 참여·운영 redirect URI/secret 설정·OIDC/ID token/JWK·브라우저 보안 쿠키 정책·provider별 인증 수명과 실제 profile payload·기존 계정의 명시적 연결/해제 제품 흐름·이메일 변경 정책·연결 동시성의 여러 backend 프로세스·강제 탈퇴/잠금 정책은 별도입니다. SQL로 계정을 inactive로 바꾼 것은 상태 경계 fixture이며 계정 비활성화 API 검사는 아닙니다. protected API의 전체 권한·Swagger123개 완료도 이번25개로 주장하지 않습니다.
