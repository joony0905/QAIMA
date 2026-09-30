# QAIMA API/DTO 계약 버전 정책

## 목적

이 문서는 Spring public API, Spring-FastAPI 내부 API, FastAPI Pydantic 모델, React TypeScript 타입 사이의 계약 drift를 막기 위한 기준을 정의한다.

현재 QAIMA는 다음 네 계층이 같은 데이터를 서로 다른 형태로 표현한다.

- Spring public DTO: `backend/src/main/java/com/qaima/dto/**`
- Spring external/inbound DTO: `backend/src/main/java/com/qaima/external/dto/**`
- FastAPI Pydantic model: `analysis/app/models/**`
- React type/mapper: `frontend/src/types/**`, `frontend/src/mappers/**`, `frontend/src/api/**`

기존 데이터 구조 원칙은 `policy/Feature_data_structure_policy.md`를 따른다. 이 문서는 변경 관리, 버전, 검증 기준을 추가로 정한다.

## 계약 경계

### Public API

브라우저 또는 외부 client가 호출하는 Spring `/api/v1/**` 응답은 public contract다.

- envelope는 `ApiResponse` 형태를 유지한다.
- 필드명은 `camelCase`를 사용한다.
- 성공/부분 성공/실패 표현은 `policy/error_warning_log_frontend_policy_2026-05-23.md`를 따른다.
- public route, HTTP method, status semantics, 주요 response field 제거는 breaking change로 본다.

### Spring to FastAPI

Spring이 FastAPI로 보내는 분석 요청과 FastAPI 응답은 internal contract다.

- 필드명은 `snake_case`를 사용한다.
- public `ApiResponse` envelope를 사용하지 않는다.
- Spring은 경계에서 명시적으로 snake_case 직렬화/역직렬화를 수행한다.
- FastAPI 응답을 Spring public DTO로 그대로 흘려보내지 않는다. Spring이 public contract로 변환한다.

### React Contract

React 타입은 public API contract를 기준으로 한다.

- 서버 응답 shape를 보정하는 로직은 `frontend/src/mappers/**` 또는 domain API wrapper에 둔다.
- UI component 안에서 서버 raw response 필드를 깊게 직접 해석하지 않는다.
- optional field는 UI fallback 정책을 함께 가져야 한다.

## 버전 정책

### Additive 변경

다음은 기본적으로 호환 변경이다.

- nullable 또는 optional field 추가
- `meta.warnings`에 신규 warning code 추가
- `meta.reportId`처럼 성공 응답의 재조회/이력 식별자를 optional field로 추가
- explain/report section의 optional block 추가
- diagnostics, policy echo, debug성 비필수 필드 추가

주의: 프론트에 노출되는 신규 warning code는 먼저 정책 문서에 등록한다.

### Breaking 변경

다음은 breaking change로 취급한다.

- public field 삭제 또는 의미 변경
- 필수 field를 nullable로 바꾸거나 nullable field를 필수로 바꾸는 변경
- enum/string literal 값 삭제 또는 의미 변경
- route, HTTP method, query/body parameter 이름 변경
- `camelCase`/`snake_case` 경계 변경
- success payload를 error payload로 바꾸거나 반대로 바꾸는 상태 의미 변경

breaking change는 다음 중 하나를 적용해야 한다.

- 새 route 또는 version segment 도입
- 일정 기간 dual-read/dual-write 호환
- React mapper에서 구버전 payload 변환 지원
- 명시적인 migration 계획과 제거 시점 기록

## 계약 문서화 기준

기능별 public API는 최소한 다음을 문서 또는 fixture로 남긴다.

- endpoint, method, auth requirement
- request query/body 예시
- success response 예시
- partial/degraded response 예시
- failure response 예시
- warning code 목록
- nullable/optional field의 UI fallback 기준

Feature1/2/3 분석 API는 추가로 다음 케이스를 포함한다.

- LLM 설명 포함/미포함
- 외부 provider 일부 실패
- 데이터 부족 또는 fallback 사용
- 계산은 성공했지만 explain만 실패한 경우
- synthetic 또는 unavailable quality 상태

## Golden JSON Fixture 정책

계약 안정성이 필요한 endpoint는 golden JSON fixture를 둔다.

권장 위치:

- Spring public API fixture: `backend/src/test/resources/contracts/{domain}/`
- FastAPI internal fixture: `analysis/tests/contracts/{domain}/`
- Frontend mapper fixture: `frontend/src/**/__fixtures__/` 또는 `frontend/src/test/contracts/`

fixture 파일명은 다음 형식을 권장한다.

```text
{endpoint-or-usecase}.{case}.request.json
{endpoint-or-usecase}.{case}.response.json
```

예시:

```text
feature3.analysis.success.request.json
feature3.analysis.synthetic.response.json
feature2.peercluster.partial.response.json
feature1.analysis.llm_failed.response.json
```

## 검증 정책

계약 변경 PR은 변경 범위에 맞는 검증을 수행한다.

필수 검증:

- Java compile: `./gradlew --no-daemon compileJava`
- FastAPI model/import 확인: `python3 -m compileall -q analysis/app`
- Frontend type/build 확인: `npm run build`

계약 변경 시 추가 검증:

- Spring DTO serialization/deserialization 테스트
- FastAPI Pydantic parse 테스트
- React mapper/type guard 테스트
- golden JSON fixture 갱신 여부 확인

테스트가 아직 없는 영역은 최소한 fixture와 수동 호출 예시를 문서에 남긴 뒤, 후속 작업으로 자동화한다.

## Naming 정책

- Spring/React public field: `camelCase`
- FastAPI/internal analysis field: `snake_case`
- Java enum external value: 명시적 문자열 contract로 취급한다.
- warning/error code: `UPPER_SNAKE_CASE`

## 분석 리포트 이력 계약

분석 생성형 public endpoint는 분석 성공 후 리포트 스냅샷 저장을 시도할 수 있다.

- Feature1: `POST /api/v1/feature1/analyze`
- Feature2: `POST /api/v1/feature2/analyze`
- Feature3: `POST /api/v1/feature3/analysis`

저장 성공 시 envelope의 `meta.reportId`에 저장된 `analysis_report.report_id`를 내려보낸다. 기존 `data` payload 구조는 변경하지 않는다.

저장 실패는 분석 결과 자체를 실패시키지 않는다. 분석 응답은 성공으로 유지하고 `meta.warnings`에 `REPORT_SAVE_FAILED`를 추가한다.

리포트 조회 endpoint:

- `GET /api/v1/reports/me`
  - auth: required
  - query: `featureType` optional (`FEATURE1`, `FEATURE2`, `FEATURE3`), `page` default `0`, `size` default `20`
  - response: `ApiResponse<List<AnalysisReportSummaryDto>>`
- `GET /api/v1/reports/{reportId}`
  - auth: required
  - ownership: authenticated user owns report
  - response: `ApiResponse<AnalysisReportDetailDto>`
  - not found or non-owned report: `RESOURCE_NOT_FOUND`

`analysis_report`는 MySQL `JSON` 컬럼을 사용해 `requestPayload`, `resultSnapshot`, `warnings` 스냅샷을 보관한다. 사용자 이름은 재현성을 위해 생성 당시 `users.name` 값을 `userName`으로 저장하고, 이름이 없으면 `사용자`를 저장한다.

리포트 메타 표시 계약:

- Feature1/Feature2: `subjectLabel`은 “종목명(종목코드)” 형태를 기본으로 한다.
- Feature3: `subjectLabel=포트폴리오`, `subjectType=PORTFOLIO`를 사용하고, `subjectDetail` 또는 `portfolioSummary`에는 “대표 종목 외 N개 종목” 형태의 요약을 둔다.
- 공통 표시 메타는 `generatedAt`, `analysisModel`, `userName`, `investLevel`, `analysisWindow`, `dataAsOf`를 포함할 수 있다.
- Feature3는 포트폴리오 특성상 `riskProfile`, `priceBasis`, `covarianceModel` 같은 추가 메타를 포함할 수 있다.
- public DTO는 기존 `data` payload를 깨지 않기 위해 위 필드들을 optional로 유지한다.

PDF/재다운로드 계약:

- API는 PDF 바이너리를 필수 저장하지 않는다. `GET /api/v1/reports/{reportId}`의 JSON snapshot이 PDF 재다운로드의 원천이다.
- 프론트는 snapshot을 리포트 컴포넌트로 재렌더링하고 PDF 캡처 유틸을 통해 파일을 생성한다.
- PDF 모드에서 화면 전용 탭/스크롤/애니메이션은 캡처 친화적인 펼침 상태로 바꿀 수 있으며, 이는 API DTO 변경으로 간주하지 않는다.
- Redis key 또는 cache policy field는 해당 cache 문서를 따른다.

내부 구현체 이름이 바뀌어도 public field 이름은 유지한다.

## 인증/회원 프로필 계약

회원가입, 로그인, 소셜 추가정보 입력의 상세 정책은 `policy/auth_signup_login_policy_2026-05-23.md`를 따른다. 이 섹션은 public API/DTO 계약 관점의 최소 기준만 정의한다.

### 사용자 프로필 DTO

`UserResponseDto`와 프론트 `MyProfile`은 public contract다. 현재 인증/회원 화면이 의존하는 필드는 다음과 같다.

- `userId`
- `email`
- `name`
- `phone`
- `birthdate`
- `gender`
- `country`
- `experience`
- `status`
- `glossaryHover`

계약 기준:

- `birthdate`는 DB에는 7자리까지 저장할 수 있으나, 화면 노출은 앞 6자리 생년월일만 사용한다.
- `gender`는 `male|female|null` 값을 사용한다. 프론트 표시 문자열은 `남성|여성|-`로 변환한다.
- `country`와 `status`는 additive field지만, 소셜 추가정보 플로우에서는 기능 필드로 취급한다.
- `status=profile_required`는 소셜 가입 후 추가정보 입력이 필요한 상태다. 프론트는 일반 서비스 화면으로 보내지 않고 추가정보 입력 화면으로 유도한다.

### 인증 public endpoint

인증/회원 프로필 관련 public endpoint는 다음 계약을 유지한다.

- `POST /api/v1/auth/signup`
  - 일반 회원가입.
  - request는 `email`, `password`, `verificationCode`, `name`, `birthdate`, `phone`, `country`를 포함한다.
  - `birthdate`는 7자리 숫자 문자열이다.
- `POST /api/v1/auth/login`
  - 일반 로그인.
  - 성공 시 access token payload와 refresh token cookie를 발급한다.
- `POST /api/v1/auth/refresh`
  - refresh token cookie 기반 access token 재발급.
- `GET /api/v1/users/me`
  - 로그인 사용자 프로필 조회.
- `PATCH /api/v1/users/me`
  - 일반 프로필 수정. 현재 생년월일/성별/국적 변경 계약은 포함하지 않는다.
- `PATCH /api/v1/users/me/social-profile`
  - 소셜 가입 추가정보 완료.
  - request는 `name`, `phone`, `birthdate`, `country`를 포함한다.
  - 성공 시 `status=active`로 전환된 `UserResponseDto`를 반환한다.

### OAuth redirect contract

OAuth 성공 리다이렉트는 프론트가 해석하는 query contract를 가진다.

```text
/login/oauth2/success?status=success&provider={provider}&profileRequired={true|false}
```

- `profileRequired=true`: 프론트는 access token bootstrap 후 `/signup/social-complete`로 이동한다.
- `profileRequired=false`: 프론트는 기존 redirect target 또는 기본 서비스 화면으로 이동한다.

OAuth 실패 리다이렉트는 기존 error query contract를 유지한다.

```text
/login?status=error&provider={provider}&code={errorCode}
```

## Deprecation 정책

필드를 제거하기 전 다음 단계를 거친다.

1. 신규 field를 additive 방식으로 추가한다.
2. Spring/React가 신규 field를 우선 읽고, 기존 field를 fallback으로 읽는다.
3. 정책 문서에 deprecated field와 제거 예정 조건을 기록한다.
4. 최소 한 번의 release 또는 검증 주기를 지난 뒤 제거한다.

legacy 호환 필드는 코드 주석만으로 남기지 않는다. 문서에 유지 이유와 제거 조건을 기록한다.

## 변경 시 확인 사항

- public API 변경이면 React type, mapper, page 사용처를 함께 검색한다.
- FastAPI model 변경이면 Spring `external/dto`와 `FastApiAnalysisClient` mapping을 함께 확인한다.
- warning/error code 변경이면 프론트 `getErrorMessage`, warning display, 정책 문서를 함께 확인한다.
- Feature3처럼 깊은 response는 optional field 추가 후 UI에서 undefined-safe rendering을 확인한다.
- 계약 변경 후 관련 문서의 예시 payload가 현재 코드와 맞는지 확인한다.
