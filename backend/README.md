# Spring Public API

작성·검증 기준: 2026-09-25. Java 17, Spring Boot 3.2.5, Gradle wrapper 8.14. Windows에서 실행합니다.

## 책임과 구조

- `api/`: HTTP 계약·인증 사용자 전달·응답 조립.
- `service/`: 조회·분석 조립·크레딧·프로필·리포트·동기화.
- `external/`: FastAPI 및 금융·뉴스 제공처 클라이언트, 내부 DTO.
- `repository/`, `domain/`: JPA 영속화. WebFlux에서 블로킹 작업은 `Blocking` 유틸을 통해 분리하는 경로가 있습니다.
- `security/`, `config/`: JWT·refresh cookie·OAuth·권한·CORS·Redis·HTTP 클라이언트.
- `src/main/resources/db/migration/`: Flyway 이력.

기본 분석 경로는 Feature 1 `/api/v1/feature1/analyze`, Feature 2 `/api/v1/feature2/analyze`, Feature 3 `/api/v1/feature3/analysis`입니다. 내부 FastAPI 분석 경로는 각각 `/feature1/analysis`, `/feature2/analysis`, `/feature3/analysis`입니다. 저장 Swagger 123개 작업과 현재 Controller 매핑의 일치 검사는 통과했습니다.

## 로컬 실행

Windows PowerShell에서 `backend`로 이동한 후 환경 설정을 확인하고 실행합니다.

```powershell
.\gradlew.bat bootRun
```

설정은 `application.yml`, 프로필별 YAML, `application-secret.yml`에서 확인합니다. MySQL·Redis 및 WSL FastAPI 접근 경로가 필요합니다. 비밀값을 명령행이나 문서에 복사하지 않습니다. `QaimaApplication`은 `@EnableScheduling`을 사용하므로 기존 DB를 대상으로 실행하기 전에 scheduler·importer 활성 조건을 확인해야 합니다. 전체 앱 기동을 단위 테스트의 선행 조건으로 사용하지 않습니다.

Swagger UI와 문서의 경로는 `SecurityConfig`와 Springdoc 설정을 확인합니다. 보관된 명세는 `../docs/api/openapi.json`입니다. Swagger에 표시된다는 사실만으로 현재 데이터·외부 연동이 검증된 것은 아닙니다.

## 테스트

실제 KIS 토큰 발급·종목 기본정보 조회는 HTTP200/200 이후 테스트 단언이 실패했습니다. 최초 로그의 마스킹 때문에 실패 단언 위치는 미확정이며 추가 실제 호출은 하지 않았습니다. 호출 제한4개 검사는 통과했고, [실제 연동 기록](src/test/live-kis-readonly.md)에 방법·예산·미검증 범위를 분리했습니다.

가격·산업지수 캐시 reader의 계산·직렬화·동시 요청·카드 HTTP 추가39개는37 PASS·2 FAIL입니다. 빈 가격 목록 내부 오류와 부분 결과 캐시의 경고 소실을 재현했습니다. 저장소·Redis·제공처 전송은mock이며 [상세 재현](src/test/market-cache-readers.md)에 경계와 방법을 기록했습니다.

Windows Spring 분석 client→WSL 실제 uvicorn의 opt-in11개도 모두 통과했습니다. 실제 계산·로컬 감성 모델·F3 Public handler의 보고서/환불 호출을 포함하며 가격/DB/실제 크레딧과 보고서 저장은 합성 경계입니다. [실제 TCP 검증 상세](src/test/cross-os-fastapi.md)에서 실행 방법·외부송신 차단·테스트 환경 오류 수정 이력을 확인할 수 있습니다.

진단·OAuth 진입·종목 메타/동기화9경로는25개 중23 PASS·2 FAIL입니다. 실제 보안필터와 Controller를 결합해 일반 USER의 관리자용 동기화 실행 허용을 재현했고, 진단 오류의 내부문구 공개도 기록했습니다. 서비스/보안 설정을 수정하지 않고 [상세 검사](src/test/diagnostic-ticker-flow.md)에 근거와 실제 연동 한계를 남겼습니다.

Feature3 가격·벤치마크2경로는29개 중25 PASS·4 FAIL입니다. 실제 Yahoo 파서/품질/캔들로더·지수보강 흐름에 합성 전송/저장소를 연결했습니다. 제공처 표기·중복 거래일·미래 지수 가용판정 문제와 검증 한계를 [상세 기록](src/test/feature3-market-data.md)에 남겼습니다. 실제 외부 API·SQL·브라우저 연동은 별도입니다.

[테스트 실행 및 케이스 설명](src/test/README.md)을 참조합니다. 테스트 빌드는 산출물을 `src/test/.runtime`에 둡니다. 기존 서비스 코드, 빌드 파일, 설정은 변경하지 않습니다. mock 테스트는 실제 DB 트랜잭션·잠금·원자성을 증명하지 않습니다.

리포트 조회에는 사용자별 7일 이전 리포트를 삭제하는 정리 로직이 있습니다. 실제 조회 검증 시 이 부수 효과도 고려해야 합니다. 공개 API와 관리자 적재 API를 같은 방식으로 일괄 실행하지 않습니다.

관리자 기준금리·환율·채권의18경로는 실제 Controller→sync service→BOK/FRED client·JSON에 합성 전송/Repository를 연결한23개 검사가 통과했습니다. 페이지·연도분할·재시도·결측처리·당일갱신재사용·저장요청을 확인했지만 실제 금융 API·SQL트랜잭션·관리자JWT 결합 완료를 뜻하지 않습니다.

OpenDART 관리자4경로·ZIP/XML·발행주식수·일일scheduler는 추가46개 중44 PASS·2 FAIL입니다. 회사명 텍스트가 XML entity/CDATA 경계에서 잘리는 결함을 재현했습니다. 합성 전송/Repository와 자식 프로세스 차단을 사용했으며 실제 제공처·DB·cron 검증은 남았습니다. [상세 테스트 방법](src/test/opendart.md)을 참조합니다.

SEC 발행주식수2경로·미국 종목 마스터1경로와 parser/scheduler는50개 중49 PASS·1 FAIL입니다. 범위초과 주식수의 잘못된 정수 변환을 재현했습니다. [상세 방법](src/test/sec-issued-master.md)에원천변환·자연키·별칭·실패처리와mock한계를기록했습니다. SEC13F 관리자5경로는별도40개 중37 PASS·3 FAIL로, 필수열누락성공처리·불가능한날짜보정을재현했습니다. [13F상세](src/test/sec-13f.md)에실제ZIP/서비스계산과대체한SQL경계를명시했습니다. 실제원천/DB검증은남아있습니다.
