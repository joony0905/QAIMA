# QAIMA 회원가입 및 로그인 정책

작성일: 2026-05-23
적용 범위: React 프론트, Spring 인증 API, OAuth2 소셜 로그인, users/social_account 스키마

## 1. 목적

이 문서는 QAIMA의 일반 회원가입, 일반 로그인, 소셜 로그인, 소셜 추가정보 입력 흐름의 현재 정책과 이번 세션 변경사항을 정리한다.

인증 기능 변경 시 이 문서를 기준으로 다음 계약을 유지한다.

- 일반 회원가입은 이메일 인증과 필수 개인정보 입력을 완료해야 한다.
- 소셜 로그인은 provider가 제공하지 않는 QAIMA 필수 정보를 추가 입력받아야 한다.
- DB에는 생년월일 7자리, 성별, 국적을 저장한다.
- 모바일 환경에서는 기본적으로 PC 버전 안내 화면을 먼저 노출한다.

## 2. 사용자 정보 스키마 정책

### users 필드

현재 인증/회원정보 흐름에서 사용하는 주요 `users` 필드는 다음과 같다.

- `email`: 로그인 아이디. 일반/소셜 모두 필수.
- `password_hash`: 일반 가입은 필수, 소셜 계정은 `NULL` 허용.
- `name`: 사용자 이름.
- `phone`: 전화번호. 숫자만 저장한다.
- `birthdate`: 주민등록번호 앞 6자리와 뒤 1자리를 합친 7자리 숫자 문자열.
- `gender`: `birthdate` 마지막 자리에서 계산한 성별 값.
  - `1`, `3`: `male`
  - `2`, `4`: `female`
- `country`: 국적.
- `status`:
  - `active`: 정상 이용 가능.
  - `profile_required`: 소셜 가입 후 추가정보 입력이 필요한 상태.
- `email_verified`: 이메일 인증 완료 여부.

### 마이그레이션

관련 migration은 다음을 포함한다.

- `V45__extend_user_birthdate_and_add_gender.sql`
  - `users.birthdate`를 `VARCHAR(7)`로 확장.
  - `users.gender VARCHAR(10)` 추가.
- `V46__add_user_country.sql`
  - `users.country VARCHAR(50)` 추가.

운영 또는 로컬 DB는 서버 재시작 또는 Flyway 실행으로 위 migration이 적용되어야 한다. `birthdate`가 여전히 `VARCHAR(6)`이면 7자리 회원가입 저장 시 서버 오류가 발생한다.

## 3. 일반 회원가입 정책

### 프론트 입력

회원가입 화면은 다음 필드를 입력받는다.

- 이메일
- 이메일 인증번호
- 비밀번호
- 비밀번호 확인
- 이름
- 전화번호
- 생년월일 6자리
- 주민등록번호 뒤 1자리
- 국적
- 개인정보 수집 및 이용 동의

생년월일과 뒤 1자리 입력은 숫자만 허용한다.

- 생년월일 칸: 6자리 숫자만 허용.
- 뒤 1자리 칸: 1자리 숫자만 허용.
- 뒤 1자리는 `1`부터 `4`까지만 유효하다.
- 제출 payload의 `birthdate`는 `앞 6자리 + 뒤 1자리`로 전송한다.

### 개인정보 동의

회원가입 버튼 클릭 시 개인정보 수집 및 이용 동의가 없으면 가입 요청을 보내지 않는다.

사용자 메시지:

```text
개인정보 수집 및 이용에 동의해야 회원가입이 가능합니다.
```

상세보기에는 최소 다음 내용을 노출한다.

- 수집 및 이용 목적: 회원가입, 본인 확인, 서비스 제공, 이용 통계 분석, 맞춤형 서비스 개선.
- 수집 항목: 이메일, 비밀번호, 이름, 전화번호, 생년월일, 성별, 국적, 접속 IP, 브라우저 정보, 서비스 이용 기록.
- 보유 및 이용 기간: 회원 탈퇴 시까지, 단 법령 보관 필요 시 해당 기간.
- 동의 거부 시 회원가입 제한.

### 백엔드 처리

일반 회원가입 API는 `POST /api/v1/auth/signup`을 사용한다.

백엔드는 다음을 수행한다.

- 이메일 중복 확인.
- 전화번호 중복 확인.
- 이메일 인증번호 소비.
- `birthdate` 7자리 정규화.
- `birthdate` 마지막 자리로 `gender` 계산.
- `country` 저장.
- `status=active` 설정.
- `email_verified=true` 설정.

일반 회원가입은 가입 성공 후 로그인 페이지로 이동한다.

## 4. 일반 로그인 정책

일반 로그인 API는 `POST /api/v1/auth/login`을 사용한다.

처리 기준:

- 이메일은 trim 후 lowercase 처리한다.
- `password_hash`가 없거나 비밀번호가 일치하지 않으면 로그인 실패.
- `status`가 `active`가 아니면 로그인 실패.
- `email_verified=false`면 로그인 실패.
- 성공 시 access token 응답과 refresh token cookie를 발급한다.

프론트는 로그인 성공 시 access token을 메모리 저장소에만 저장하고, refresh token은 httpOnly cookie에 의존한다.

## 5. 소셜 로그인 정책

### 지원 provider

백엔드는 Google, Kakao, Naver provider를 지원한다.

프론트 렌더링 정책:

- 현재 로그인 화면에는 Naver와 Google만 노출한다.
- Kakao는 사업자등록 이슈로 프론트 버튼을 렌더링하지 않는다.
- 백엔드 Kakao 지원 코드는 유지할 수 있다.

### OAuth client-id 설정

Spring OAuth2가 실제로 읽는 경로는 `spring.security.oauth2.client.registration.{provider}`다.

현재 `application-secret.yml`의 provider 값은 `security.oauth2.client.registration.{provider}` 경로에 있으므로, `application.yml`은 다음 fallback 구조로 읽는다.

```yaml
google:
  client-id: ${security.oauth2.client.registration.google.client-id:${GOOGLE_CLIENT_ID:google-client-id}}
  client-secret: ${security.oauth2.client.registration.google.client-secret:${GOOGLE_CLIENT_SECRET:google-client-secret}}

naver:
  client-id: ${security.oauth2.client.registration.naver.client-id:${NAVER_CLIENT_ID:naver-client-id}}
  client-secret: ${security.oauth2.client.registration.naver.client-secret:${NAVER_CLIENT_SECRET:naver-client-secret}}
```

로그에서 `client_id=google-client-id` 또는 `client_id=naver-client-id`가 보이면 설정이 적용되지 않은 상태다.

### Google redirect URI

Google Cloud Console의 Authorized redirect URI에는 실제 요청 URI를 정확히 등록해야 한다.

로컬 기준:

```text
http://localhost:8080/login/oauth2/code/google
```

`http/https`, host, port, path, trailing slash가 모두 정확히 일치해야 한다.

## 6. 소셜 첫 가입 추가정보 입력 정책

소셜 provider는 이름과 이메일 정도만 제공하므로 QAIMA 필수 정보인 전화번호, 생년월일, 성별, 국적은 별도 입력받는다.

### 첫 가입 흐름

1. 사용자가 Google/Naver 소셜 로그인 시작.
2. provider 인증 성공.
3. 백엔드가 provider profile에서 provider user id, email, name, email verified 여부를 추출.
4. 동일 provider user id의 `social_account`가 없고, 동일 email 일반 계정도 없으면 신규 소셜 계정을 생성.
5. 신규 `users` row는 다음 상태로 저장.
   - `password_hash=NULL`
   - `name=provider name`
   - `email=provider email`
   - `status=profile_required`
   - `email_verified=true`
6. refresh token cookie를 발급한다.
7. OAuth 성공 리다이렉트에 `profileRequired=true`를 붙인다.
8. 프론트는 access token bootstrap 후 `/signup/social-complete`로 이동한다.
9. 사용자가 전화번호, 생년월일 6자리, 뒤 1자리, 국적을 입력한다.
10. `PATCH /api/v1/users/me/social-profile` 호출.
11. 백엔드는 입력값 저장 후 `status=active`로 전환한다.
12. 프론트는 `/feature/1`로 이동한다.

### 기존 불완전 소셜 계정 처리

과거 흐름에서 이미 `active`로 생성되었지만 전화번호, 생년월일, 성별, 국적이 비어 있는 소셜 계정이 있을 수 있다.

정책:

- 소셜 로그인 시 계정이 `active`라도 필수 profile이 비어 있으면 `profileRequired=true`로 리다이렉트한다.
- `PATCH /api/v1/users/me/social-profile`은 `profile_required` 또는 필수 profile 누락 계정에 대해 허용한다.
- 저장 완료 후 `status=active`로 정규화한다.

### 추가정보 API

Endpoint:

```text
PATCH /api/v1/users/me/social-profile
```

Auth:

- access token 필요.
- 소셜 로그인 직후 발급된 refresh token으로 access token bootstrap 후 호출한다.

Request:

```json
{
  "name": "홍길동",
  "phone": "01012345678",
  "birthdate": "9901011",
  "country": "대한민국"
}
```

Validation:

- `name`, `phone`, `birthdate`, `country` 필수.
- `phone`은 숫자만 저장.
- `birthdate`는 숫자 7자리.
- `birthdate[6]`은 `1`부터 `4`까지만 허용.
- phone 중복은 현재 user를 제외하고 검사한다.

## 7. 아이디 찾기 정책

아이디 찾기는 사용자가 생년월일 앞 6자리만 입력하는 기존 UX를 유지한다.

DB에는 `birthdate`가 7자리로 저장되므로 조회는 다음 기준을 사용한다.

- 입력값은 6자리 생년월일로 정규화.
- DB 조회는 정확히 6자리 legacy row 또는 7자리 row prefix match를 허용한다.
- 동일 이름/생년월일 계정이 여러 개면 전화번호 입력을 요구한다.

## 8. 내정보 표시 정책

내정보 화면은 다음 값을 표시한다.

- 이름
- 아이디 이메일
- 전화번호
- 생년월일
- 성별

표시 기준:

- 생년월일은 DB에 7자리로 저장되어도 화면에는 앞 6자리만 `YY.MM.DD`로 표시한다.
- `gender=male`은 `남성`, `gender=female`은 `여성`으로 표시한다.
- `gender`가 비어 있으면 `birthdate` 7번째 자리로 fallback 계산한다.

개인정보 수정 화면에서도 생년월일과 성별은 읽기 전용으로 표시한다.

## 9. 모바일 접속 정책

현재 QAIMA는 PC 화면만 제공한다.

프론트는 `max-width: 767px` 환경에서 전체 앱 라우트 대신 안내 화면을 먼저 표시한다.

제목:

```text
현재는 PC 환경만 제공합니다
```

버튼:

```text
PC버전으로 보기
```

사용자가 버튼을 누르면 `sessionStorage`에 강제 PC 보기 플래그를 저장하고, 현재 세션 동안 앱을 그대로 렌더링한다.

## 10. 구현 파일 기준

주요 구현 파일:

- 일반 회원가입 화면: `frontend/src/pages/SignupPage.tsx`
- 소셜 추가정보 화면: `frontend/src/pages/SocialProfileCompletePage.tsx`
- OAuth 성공 처리 화면: `frontend/src/pages/OAuth2SuccessPage.tsx`
- 로그인 화면: `frontend/src/pages/LoginPage.tsx`
- 프론트 user API: `frontend/src/api/user.ts`
- 프론트 auth API: `frontend/src/api/auth.ts`
- 라우팅 및 모바일 안내: `frontend/src/App.tsx`
- 일반 인증 서비스: `backend/src/main/java/com/qaima/service/auth/AuthService.java`
- 소셜 인증 서비스: `backend/src/main/java/com/qaima/service/auth/OAuth2SocialLoginService.java`
- OAuth 성공 핸들러: `backend/src/main/java/com/qaima/security/OAuth2LoginSuccessHandler.java`
- 사용자 서비스: `backend/src/main/java/com/qaima/service/user/UserService.java`
- 사용자 API: `backend/src/main/java/com/qaima/api/UserController/UserController.java`
- 사용자 엔티티: `backend/src/main/java/com/qaima/domain/User.java`
- 소셜 계정 엔티티: `backend/src/main/java/com/qaima/domain/SocialAccount.java`

## 11. 검증 기준

인증/회원가입 관련 변경 후 최소 검증:

```bash
cd backend
./gradlew --no-daemon compileJava

cd frontend
npm run build
```

수동 확인:

- 일반 회원가입: 이메일 인증, 개인정보 동의, 7자리 생년월일 저장, gender/country 저장.
- 일반 로그인: access token 발급 및 refresh cookie 동작.
- Google/Naver 첫 소셜 로그인: 추가정보 입력 페이지 이동.
- 기존 불완전 소셜 계정 로그인: 추가정보 입력 페이지 이동.
- 추가정보 저장 후 재로그인: 바로 서비스 화면 이동.
- 모바일 폭 접속: PC 환경 안내 화면 표시 및 PC버전으로 보기 동작.
