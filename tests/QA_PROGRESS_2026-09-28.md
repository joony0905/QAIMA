# 2026-09-28 QA 재개 기록

## 최신 재개 지점

최신 [Feature3 overlay·Redis·HTTP 취소/재시도](isolated-feature3-resilience.md)은 **23개19 PASS/4 FAIL**(`run-p4h4ipqi`)입니다. 선택 overlay 실패 경고 누락3사례와 Redis+가격 SQL 장애의 비용3 환불 누락1사례를 재현했습니다. 실제 FastAPI40회·응답38개·snapshot/상세33개·USE44/REFUND4와 취소6개·후속 복구16개를 대조했고 소유 서비스를 정리·종료했습니다. 취소 차감 잔존/동일 body 별도 차감은 명시 정책 공백 관측이며, 초기2회는 합산하지 않습니다.

직전 [Feature3 실제 원천 SQL·대체 조회·계산/과금](isolated-feature3-source-sql.md)은 **17개12 PASS/5 FAIL**(`run-sphw2mwo`)입니다. 가격/벤치마크 실제 SQL 오류의 환불 누락3사례, 금리 원천 모두 empty에서 빈 HTTP200/차감 잔존1사례, 오래된 가격의 freshness 오표시1사례를 재현했습니다. 실제 FastAPI28개·응답32개·snapshot/상세27개·USE32/REFUND0·장애 복구14개를 대조했고 소유 서비스를 정리·종료했습니다. 초기3회는 합산하지 않습니다.

직전 [Feature1 실제 Redis·HTTP 취소/재시도·SQL](isolated-feature1-resilience.md)은 **17개16 PASS/1 FAIL**(`run-rqzhknkk`)입니다. Redis 단절/재시작·계산 후 캐시 저장 실패의 결과 보존, 취소/timeout 전파·동일 입력 복구·취소된 이전 결과의 캐시/SQL 미덮어쓰기를 확인했습니다. Redis+가격 SQL 복합 장애의 정상 재무 소실은 기존 FAIL입니다. 실제 FastAPI32회·응답29개·snapshot/상세28개·USE35/REFUND1을 대조했습니다. 미완료 취소6개의 차감 잔존과 동일 body2개의 별도 차감은 정책 적합성 PASS와 구분합니다. 소유 서비스를 정리·종료했습니다.

직전 [Feature1 실제 서버·브라우저/저장/PDF](../frontend/tests/browser-feature1-pipeline.md)은 **17개5 PASS/12 FAIL**(`run-k_l2x_cx`)입니다. 계산 장애의 부분 결과/안내·동일 입력 복구와 보조 재무 조회 오류 중 분석 보존은 통과했습니다. 저장 정량·설명/핵심/저장 경고 누락과 기존 원천/snapshot/stale/날짜 실패를 실제 화면까지 재현했습니다. 실제 FastAPI17회·응답20개·snapshot/상세15개·USE20/REFUND3·동일 입력 복구3개를 대조했고 소유 서비스를 정리·종료했습니다. 별도 검토의 볼린저 위치91.2%→50.0% 및 PDF 스타일 실패는 JUnit에 중복 합산하지 않습니다.

직전 [Feature1 실제 SQL·snapshot·Redis·계산/저장](isolated-feature1-pipeline.md)은 **23개13 PASS/10 FAIL**(`run-y4nj39wc`)입니다. 원천 오류의 전체 전파, 기존 캔들 미보존, 빈 snapshot의 재무 fallback 차단, stale/저장 경고 누락, 날짜 경계와 기존 저장 매핑 실패를 재현했습니다. 실제 FastAPI28회·응답31개·snapshot/상세26개·USE31/REFUND3·후속 정상8요청을 대조했습니다. 소유 서비스는 정리·종료했고 초기3회는 합산하지 않습니다.

직전 [Feature2 실제 SQL·Peer·로컬 모델의 브라우저/저장/PDF](../frontend/tests/browser-feature2-pipeline.md)은 **12개4 PASS/8 FAIL**(`run-kd_uymvv`)입니다. 실제 원천 경고의 화면 누락, 모델 점수 없음에서 보고서 기사 소실, 산업지수 장애의 정상 RAW Peer 차트 소실과 기존 저장 정량/거시/NULL 수급 실패를 확인했습니다. 실제 FastAPI42회·모델31결과/공개33점수·snapshot/상세14개/USE14개·PDF2개6페이지를 대조했습니다. 최종 소유 서비스는 정리·종료했으며 초기 보정/중단 실행은 합산하지 않습니다.

직전 [Feature2 실제 Peer·로컬 뉴스 모델·SQL 결합](isolated-feature2-pipeline.md)은 **14개8 PASS/6 FAIL**(`run-5br21593`)입니다. 신규 Peer 오류 캐시 복구 실패5사례와 기존 최신일 제외1사례를 재현했습니다. 실제 역방향SQL/독립 상관·시차,모델 누락/복구·RAW fallback·SQL 저장 장애의 부분 결과 보존을 확인했습니다. 실제FastAPI61개/역방향TCP18개·모델40결과/공개69점수·snapshot/상세26개/USE26개를 대조했고 소유 서비스는 정리·종료했습니다.

직전 [Feature2 실제 시장 SQL·Redis·FastAPI 설명](isolated-feature2-source-sql.md)은 `run-udwy3eb5`의 **24개13 PASS/11 FAIL**입니다. SQL 장애의 정상 거시/수급 소실, 수급NULL→0/경고 누락, 정량0의 차감/저장을 확인했습니다. 실제 설명44회·snapshot/상세46개·원장46개를 대조했고 소유 서비스는 모두 종료했습니다. 상세 단계는 아래에 누적합니다.

직전 [Feature2 실제 Redis·HTTP 취소/SQL](isolated-feature2-resilience.md)은 `run-8m54svrw`의 **15개12 PASS/3 FAIL**로 확정했습니다. 취소된 오래된 가격/산업 조회의 새 캐시 덮어쓰기2경로와 복구 후 뉴스 오류 경고 잔류를 재현했습니다. 실제 조립 fallback은 약40.6초, 같은 client의 Redis 재접속 확인은 최대27.340초였습니다. 초기 실행/관측 한도·기대값 보정은 상세 문서에 보존했고 최종15개만 누적에 합산합니다.

앞선 [Feature2 실제 브라우저/서버 fallback](../frontend/tests/browser-feature2-server.md)은 최종 `run-l3jr_n8_`의 **13개7 PASS/6 FAIL**로 확정했습니다. 실제 로그인→조립/차감/SQL→화면/저장 리포트/PDF를 연결했으며 보조 조회 실패 시 성공 분석 숨김, 저장/핵심 실패 안내 누락, 저장 정량/설명 표시 실패를 기록했습니다. 초기 실행·테스트 기대값 보정 이력은 아래 상세에 보존했고 최종13사례만 누적에 합산했습니다.

2026-09-28 인터넷 중단 후 사용자의 `resume` 지시로 QA를 재개했습니다. goal은 active이며 최신 누적은 **30묶음632개481 PASS/151 FAIL**입니다. 앞선 [실제 Feature2 조립 fallback·SQL 검증](isolated-feature2-fallback.md)은39개17 PASS/22 FAIL(`run-bxtp_64j`)입니다. 이번 소유 MySQL·Redis·TCP 프록시/제어 서버도 정리·종료했습니다. 다음은 Feature3 나머지 overlay·실제 원천 브라우저/저장/PDF, 실제 외부 제공자·나머지 Swagger 사용자 흐름입니다. 전체 목표는 계속 미완료입니다.

## 사용자 추가 지시: 전체 fallback 우선 검증

2026-09-28: 전체 fallback 흐름을 철저히 확인하고, 구성요소가 많은 Feature2의 단일/복수 장애에서 부분 결과·warning·화면·과금/환불·저장·복구가 이어지는지 특히 점검합니다. 다른 세션은 [상위 QA 인덱스](README.md)와 [fallback 인계/완료 기준·Feature2 점검표](FALLBACK_QA_HANDOFF.md)를 먼저 확인해야 합니다. 실제 Feature2 조립기를 대체한 기존 과금 테스트는 전체 조립 fallback의 완료 증거가 아닙니다. 이 요구는 전체 QA 완료 조건에 계속 포함합니다.

사용자 최신 제한은 **프로젝트 수정은 tests 폴더 내부만**, DB가 필요하면 외부 격리 DB 사용입니다. 과거 문서의 docs/루트 문서 수정 허용보다 이 제한을 우선했습니다. 이번 변경은 `frontend/tests/`, `batch/tests/`, 루트 `tests/` 안에 있습니다.

## 읽은 기준과 현재 상태

[reference.md](../reference.md), [QA_CONTEXT.md](../QA_CONTEXT.md), [기존 검증 현황](../docs/verification.md), [기존 추적표](../docs/traceability.md), 결함 기록과 프론트 테스트 문서, Feature 분석 흐름/API 계약 정책을 확인했습니다. 현재 브랜치는 `dev`이며 기존 사용자 미커밋 변경이 많습니다. 과거 일시정지 문서는 당시 이력이고 이번에는 QA를 재개했습니다. 전체 목표는 **미완료·계속 진행 필요**입니다.

이전 문서에서 미검증으로 남은 Feature2 브라우저 흐름을 추가했습니다. 새 방법·명령·사례·결함 근거는 [Feature2 Chrome 검증](../frontend/tests/browser-feature2-flow.md)에 기록합니다. 루트 문서와 docs/는 이번 수정 범위 밖이므로 최신 증분 기록은 이 파일을 참조합니다.

## 현재 증분 집계와 예상

앞선 [Feature1 실제 SQL·snapshot·Redis·계산/저장](isolated-feature1-pipeline.md)은 **23개13 PASS/10 FAIL**(`run-y4nj39wc`)입니다. 원천 오류의 전체 전파, 기존 캔들 미보존, 빈 snapshot의 재무 fallback 차단, stale/저장 경고 누락, 날짜 경계와 기존 저장 매핑 실패를 재현했습니다. 실제 FastAPI28회·응답31개·snapshot/상세26개·USE31/REFUND3·후속 정상8요청을 대조했습니다. 소유 서비스는 정리·종료했고 초기3회는 합산하지 않습니다.

직전 [Feature2 실제 SQL·Peer·로컬 모델의 브라우저/저장/PDF](../frontend/tests/browser-feature2-pipeline.md)은 **12개4 PASS/8 FAIL**(`run-kd_uymvv`)입니다. 실제 원천 경고의 화면 누락, 모델 점수 없음에서 보고서 기사 소실, 산업지수 장애의 정상 RAW Peer 차트 소실과 기존 저장 정량/거시/NULL 수급 실패를 확인했습니다. 실제 FastAPI42회·모델31결과/공개33점수·snapshot/상세14개/USE14개·PDF2개6페이지를 대조했습니다. 최종 소유 서비스는 정리·종료했으며 초기 보정/중단 실행은 합산하지 않습니다.

직전 완료: [Feature2 실제 Peer·로컬 뉴스 모델·SQL 결합](isolated-feature2-pipeline.md)은 **14개8 PASS/6 FAIL**(`run-5br21593`)입니다. 신규 Peer 오류 캐시 복구 실패5사례와 기존 최신일 제외1사례를 재현했습니다. 실제 역방향SQL/독립 상관·시차,모델 누락/복구·RAW fallback·SQL 저장 장애의 부분 결과 보존을 확인했습니다. 실제FastAPI61개/역방향TCP18개·모델40결과/공개69점수·snapshot/상세26개/USE26개를 대조했고 소유 서비스는 정리·종료했습니다.

직전 완료: [Feature2 실제 시장 SQL·Redis·FastAPI 설명](isolated-feature2-source-sql.md)은 **24개13 PASS/11 FAIL**입니다. 실제 SQL/같은 client 복구·3기사/설명 경계를 추가했고 제품 결함은 tests-only 범위에서 재현·기록했습니다.

직전 완료: [Feature2 실제 Redis·HTTP 취소/SQL](isolated-feature2-resilience.md)은 **15개12 PASS/3 FAIL**입니다. 단절/바이트 차단·실제 재시작·원천 동시 실패의 선택 조합은 실제 조립/원장/리포트로 연결했습니다. 가격220→옛110 및 산업58%→옛29% 캐시 역전과 정상 복구 후 뉴스 경고 잔류는 FAIL입니다. 취소된7요청의 차감은 남고 리포트는 없으며, 취소 환불의 명시 정책이 없어 적합성 판정과 구분해 기록했습니다.

앞선 완료: [Feature2 실제 브라우저/서버 fallback](../frontend/tests/browser-feature2-server.md)은 **13개7 PASS/6 FAIL**입니다. 뉴스/Peer 부분 실패·명시 재분석 복구·잔액 부족 차단은 통과했습니다. 성공 분석의 보조 조회 오류에 따른 화면 소실과 경고/설명/저장 정량 표시 실패를 실제 원장/리포트와 대조했습니다. 실제 시장 원천/Redis/FastAPI 설명은 이번 경계 밖입니다.

앞선 [실제 Feature2 조립 fallback·HTTP/SQL](isolated-feature2-fallback.md)은 **39개17 PASS/22 FAIL**입니다. 독립 결과의 연쇄 누락·빈 응답 warning 누락·정량 결과0의 환불 누락을 재현했습니다. 36개 후속 정상 요청의 복구와71개 저장 snapshot 대조는 통과했습니다. 개별 경계 mock과 실제 하위 service4사례를 구분했습니다. 후속15개로 실제 Redis를, 추가24개로 선택한 시장 원천 SQL/무키 설명 경계를 확장했습니다. 전체 제공자 결합은 남아 있습니다.

앞선 [Redis TCP 단절·재시작·복구](isolated-redis-lifecycle.md)는 **22개22 PASS/0 FAIL**입니다. 7종 캐시의 실제 재시작·동일 client 재접속/대체 조회와 두 독립 reader의 공유 캐시를 확인했습니다. 뉴스 전체 분석 fallback은28.013초가 걸렸습니다. 후속15개에서 전체 Feature2 조립 약40.6초를 확인했고 추가24개에서3기사/시장 SQL 조합은84.885초를 관측했습니다. 최대30기사/전 원천 조합은 남아 있습니다.

앞선 완료: [Feature3 브라우저→실제 서버/SQL 검증](../frontend/tests/browser-server-pipeline.md)의 8개 사례는 **7 PASS / 1 FAIL**입니다. 정상 분석·차감/환불·저장 리포트/PDF·소유권·로그아웃을 실제 브라우저와 격리 서버/DB로 연결했습니다. 저장 실패 안내 누락1개는 제품 결함으로 보존하고, 초기 환경/테스트 오류와 재실행은 중복 합산하지 않습니다.

Feature3 overlay/Redis/취소 최종 실행까지 이번 확장의 30개 검증 묶음은 **632개: 481 PASS / 151 FAIL**입니다. 각 묶음의 최종 사례만 합산하고 선택 재현·초기 구성 오류·재실행은 제외했습니다. 실패151개는 고유 결함151건이 아닙니다. 별도 PDF 페이지 검사·계산 응답의 수학 대조 단언·후속 복구 요청과 과거 전체 suite 수치도 합산하지 않습니다. API123개의 부분 증거가 전체 사용자 흐름 완료를 의미하지 않으므로 전체 QA 완료율은 산정하지 않았습니다.

| 묶음 | 사례 | PASS | FAIL |
|---|---:|---:|---:|
| Feature2 Chrome | 18 | 15 | 3 |
| HTTP/JWT/JPA | 12 | 11 | 1 |
| Redis core | 18 | 15 | 3 |
| Redis 뉴스/metrics | 21 | 17 | 4 |
| 분석 차감/환불 | 21 | 15 | 6 |
| Python 배치 SQL | 31 | 27 | 4 |
| 매핑/초기화/해외 SQL | 22 | 14 | 8 |
| Spring 배치/예약 | 22 | 19 | 3 |
| SEC13F | 24 | 21 | 3 |
| 발행주식/마스터 | 28 | 26 | 2 |
| 회원가입/이메일/재설정 | 26 | 25 | 1 |
| OAuth/소셜 프로필 | 25 | 23 | 2 |
| 공매도/CSV/공개 카드 | 31 | 26 | 5 |
| Feature2 차트/nullable/PDF | 20 | 13 | 7 |
| Feature1 재무/차트/경합/PDF | 25 | 18 | 7 |
| Feature3 차트/입력 경합 | 26 | 23 | 3 |
| Feature3 HTTP/실제 계산/SQL | 18 | 18 | 0 |
| Feature3 브라우저/실제 서버/저장 PDF | 8 | 7 | 1 |
| Redis TCP/재시작/복구 | 22 | 22 | 0 |
| Feature2 실제 조립/fallback/HTTP/SQL | 39 | 17 | 22 |
| Feature2 브라우저/실제 조립/서버/SQL | 13 | 7 | 6 |
| Feature2 실제 Redis/조립/HTTP 취소/SQL | 15 | 12 | 3 |
| Feature2 실제 시장 SQL/Redis/FastAPI 설명 | 24 | 13 | 11 |
| Feature2 실제 Peer/로컬 모델/SQL | 14 | 8 | 6 |
| Feature2 실제 SQL/Peer/로컬 모델/브라우저 | 12 | 4 | 8 |
| Feature1 실제 SQL/snapshot/Redis/계산·저장 | 23 | 13 | 10 |
| Feature1 실제 서버/브라우저/저장 PDF | 17 | 5 | 12 |
| Feature1 실제 Redis/HTTP 취소·재시도/SQL | 17 | 16 | 1 |
| Feature3 실제 원천 SQL/fallback/과금 | 17 | 12 | 5 |
| Feature3 overlay/Redis/HTTP 취소·재시도 | 23 | 19 | 4 |
| 합계 | 632 | 481 | 151 |

2026-09-28 최신 브리핑의 계획 추정은 남은 QA·종합 기록 약8~16작업시간(하루8시간 기준1~2근무일)입니다. 필요한 외부 연동 조건이 준비됐다는 가정이며 외부 승인/응답 대기 시간은 별도입니다. 확정 일정이나 남은 범위의 완전한 작업량 산정은 아니며, 새 실패·연동 조건에 따라 늘어날 수 있습니다. 제품 결함 수정 시간은 포함하지 않습니다. 현재 제한 안에서 격리 준비가 자동화되어 있어 tests 밖 수정이나 기존 업무 DB 접근만으로 큰 단축을 기대하지 않습니다.

## Feature2 브라우저 단계 결과

HEAD는 `345aabb5075964e3ddd07ddd351915e330862e03`입니다. 검증 대상은 이 커밋만이 아니라 기존 미커밋 변경을 포함한 현재 소스이며, 브라우저 summary의187개 입력 해시로 특정합니다.

| 대상 | 결과 | 경계 |
|---|---|---|
| 프론트 복사본 입력 비교 | src/public/설정187개 일치 | source와 복사본 파일 집합·SHA-256 |
| 격리 Vite build | PASS, 2165개 모듈 | 원본 코드·의존성 수정 없음, 큰 chunk/메타데이터 경고 유지 |
| Feature2 Chrome 전체 | **18개: 15 PASS / 3 FAIL** | 실제 React/Chrome, 모든 Public API fixture |
| 실패3개 별도 재현 | 3개 모두 재현, 공통 요청/pageerror 가드 PASS | 필터 실행4개1 PASS/3 FAIL, 전체 실행과 중복 집계 금지 |
| 문서 검사 | 43개 문서 대상2개 검사 PASS | 로컬 링크·설정 일부 비밀값 노출 검사, 전체 보안 감사 아님 |
| 수정 범위 비교 | 코드/설정/문서924개 변경·소실0 | tests/의존성/빌드/venv/agent/runtime 제외 가시 파일 SHA-256 |
| 외부 호출·운영 DB | 사용 없음 | 테스트 자체 preview 외 앱/DB 서버 기동 없음 |

새 실패는 `FRONT-F2-AUX-001`(보조 조회500으로 분석200 결과 숨김), `FRONT-F2-RELATED-RACE-001`(이전 관련주 응답이 새 목록 덮어씀), `FRONT-F2-ANALYSIS-RACE-001`(A 분석이 B 화면에 붙으며 이름/코드 혼합)입니다. 제품 수정 없이 실패를 보존합니다. 분석 실패 후 버튼 소실과 로그인 복귀 경로의 종목 누락은 관측으로 분리했습니다.

전체 실행 증거: `frontend/tests/.runtime/browser-feature2/run-A8Ux87/summary.json`. 과거 Spring/Python/배치/프론트 실패 수에 합산하거나 과거 테스트를 재실행한 것으로 보지 않습니다.

실패 재확인 증거는 `run-4mREws/summary.json`과 `stale-analysis-report.png`입니다. 실제 리포트 그림에서 B 이름/A 코드와 A 분석종목이 섞이는 것을 확인했습니다. 소유 preview는 두 실행 모두 종료됐고 summary의 previewStopped는 true입니다. 수정 범위 기준/비교 결과는 `tests/.runtime/qa-20260928/source-baseline.json`, `source-audit.json`입니다.

문서 검사는 루트에서 `.venv_wsl/bin/python -B -m unittest discover -s tests -v`로 실행했습니다. 브라우저 단계의 코드 변경 파일은 `frontend/tests/browser-feature2-flow.cjs`, `tests/test_documentation.py`이며 문서는 이 파일·`tests/README.md`·`frontend/tests/README.md`·`frontend/tests/browser-feature2-flow.md`입니다. 이 브라우저 단계에서는 DB를 사용하지 않았습니다.

## 격리 MySQL·HTTP/JWT/JPA 단계 결과

이전 DB 임시 상태 파일이 없어 `/tmp`에 MySQL8.0.46을 새로 준비했습니다. 시스템 설치 없이 공식 Ubuntu 패키지를 추출하고 새 datadir·loopback 포트·QA schema를 사용했습니다. 제품 application 설정을 읽어 접속하지 않았습니다. [명령·격리 가드·메서드별 방법·Swagger 증분](isolated-http-jpa.md)에 상세를 기록했습니다.

- V1~V51 SQL51개 적용, 업무47+소유표식1테이블. Hibernate validate 성공. 이번 실행은 Flyway history 재검증이 아닙니다.
- 실제 TCP→WebFlux→SecurityConfig/JWT→Controller/Service→JPA/MySQL **12개11 PASS/1 FAIL**(`run-f993pmv8`). 관리자 권한·포트폴리오/리포트/관심목록 소유권·정상 로그인/순차 refresh/logout·SQL 실패 rollback·12개 동시 차감/5개 성공을 확인했습니다.
- `AUTH-REFRESH-RACE-001`: 같은 refresh token으로12개 동시 요청 모두200이나 성공 쿠키 중 DB hash에 남은 것은1개. 제품 수정 없이 FAIL 보존.
- 단독 재현 `run-9yikqkqi`에서도12개200/유효후속토큰1개. 밀려난 성공 쿠키로 다음 refresh401, 남은 쿠키200까지 실제 HTTP 확인했습니다. 이1개 FAIL은 전체12개와 합산하지 않습니다.
- 업무9테이블 최종0행, MySQL 정상종료0, mysqlStopped true, 실행 중 sourceChangedDuringRun0. 코드/설정/문서924개 시작 기준과의 추가 비교도 변경0입니다.
- 최초11개 실행의 관리자1실패는 테스트 role 대소문자 fixture 오류였고 실제 로그인 토큰으로 수정했습니다. 최종 결과에 제품 결함으로 합산하지 않습니다.

새 코드는 `tests/run_isolated_backend.py`, `tests/backend-isolated.init.gradle`, `tests/java/com/qaima/qa/IsolatedHttpJpaTest.java`입니다. 실행 로그·XML·DB 정리/소스 해시는 `tests/.runtime/backend-isolated/runs/`에 보존합니다. 선택한14개 Swagger method/path에 실제 보안·DB 경계를 추가했으며 전체123개 완료 판정이 아닙니다. SMTP/OAuth 경계만 대체하고 사용자·원장·저장소는 실제 MySQL을 사용했습니다.

격리 DB 문서를 포함해 문서 검사 대상은44개이며2개 검사 PASS입니다. 최초 브라우저 단계의43개와 구분합니다. 3회 실행의 mysqld는 모두 정상 종료됐고 업무9테이블을 모두 비웠습니다. 외부 datadir만 재현 증거로 보존합니다. tests 밖 가시 코드·설정·문서924개 감사 결과는 `tests/.runtime/backend-isolated/source-audit.json`에 있습니다.

## 격리 Redis·실제 Lettuce 단계 결과

소유 Redis7.0.15를 매번 새 `/tmp` 디렉터리·loopback 포트·임의 인증으로 시작하고, port/run_id/dir/소유 표식을 검증했습니다. 실제 Lettuce와 제품 RedisConfig의 String/typed serializer를 사용했으며 Repository·외부 API만 fixture입니다. [설치·실행 명령,18개 방법,시각 측정,실패 근거](isolated-redis-cache.md)를 별도로 기록했습니다.

- 최종 `run-mnndkln9`: **18개15 PASS/3 FAIL**, skip0. market snapshot·시세·가격 snapshot·산업지수·Peer·랭킹의 실제 저장/재사용, 깨진 JSON, ACL GET/SET 장애 fallback을 확인했습니다.
- 10/20/60/120초 자연 만료와 이후 재수집·stale/long snapshot 재사용 PASS. 긴 TTL은 실제 PTTL을 확인했으며 만료까지 기다리지 않았습니다.24개 겹친 가격 요청은 실제 Redis GET24회 후 Repository1회를 공유했습니다.
- 기존 `PRICE-SNAPSHOT-EMPTY-001`, `INDUSTRY-CACHE-WARNING-001`을 실제 Redis에서도 재현했습니다. 새 `REALTIME-PRICE-EMPTY-001`은 quote의 가격null을 `getPrice()`의 Reactor map이 NPE로 바꾸는 문제입니다. 재무 조회 호출자는 소스상 이 오류를 흡수하므로 공개 API500으로 확대하지 않습니다.
- 최초 `run-aw6kb1a_`는14 PASS/4 FAIL이었습니다. 추가1실패는 JVM monotonic 경과와 서버 만료 시각의 차이를 구분하지 못한 시간 단언입니다. `TIME/PEXPIRETIME`을 추가한 최종 실행에서 서버 만료 시각 이후 삭제를 확인했고, host/서버 시계 차이도 숨기지 않고 기록했습니다. 최초 실행은 최종18개와 합산하지 않습니다.
- 두 실행 모두 fixture 정리·DBSIZE0·Redis 정상종료0·redisStopped true·입력558개 실행 중 변경0입니다. 별도924개 tests 밖 소스 감사도 변경/새파일0입니다. 기존 Redis·실제 DB·외부 API는 사용하지 않았습니다.

새 코드는 `tests/run_isolated_redis.py`, `tests/java/com/qaima/qa/IsolatedRedisCacheTest.java`입니다. 모든 산출물은 `tests/.runtime/redis-isolated`에 있습니다. 문서45개 검사2개 PASS입니다. 전체 Redis 정책이나 Swagger123개를 완료한 결과는 아닙니다.

## 격리 Redis·뉴스/Feature3 지표 캐시 단계 결과

이전18개에 포함하지 않은 뉴스5계층·Feature1 metrics fresh/stale와 실제 Feature3 비용 preview/estimate를 이어서 검사했습니다. [21개 사례·명령·fixture 경계·동시성 순서](isolated-news-overlay-redis.md)를 기록했습니다. 실제 Redis/Lettuce와 제품 service를 사용하고 DB/외부 제공자/모델/observation 저장 경계는 mock입니다.

- 최종 `run-wqjld7fg`: **21개17 PASS/4 FAIL**, skip0. 뉴스5m/15m/7d·지표24h/7d PTTL, 실제 키 재사용·force refresh·누락/깨진JSON·버전 불일치, 미리보기/예상 비용 일치, signal 로딩 후 캐시 재사용을 확인했습니다. 장기 자연 만료 검사는 아닙니다.
- 기존 `F2-NEWS-CACHE-001`을3메서드로 실제 Redis에서도 재현했습니다. focus 변경/점수null이hit로 남고 모델0회, 변경된 focus의 비용도HIT/추가0/총1입니다. 같은 기존 결함을3건의 신규 결함으로 세지 않습니다.
- 새 `F2-NEWS-CACHE-RACE-001`: A 모델응답 보류→다른 본문의B0.9 완료→A0.1 해제. 최종 점수0.1에B input hash가 붙고 observation 저장 요청도B hash입니다. 캐시와 observation 두 단언이FAIL이며 두 전체 실행에서 재현됐습니다. 실제 observation DB가 저장된 것이라고 주장하지 않습니다.
- 소유 Redis를2600ms `CLIENT PAUSE`해 실제 뉴스2초 block timeout→READ warning·저장기사 fallback·동일 template 복구도 PASS입니다. 네트워크 단절/서버 재시작 검사는 아닙니다. 실제 ACL GET10/SET9거절을 확인했습니다.
- 최초 `run-zem5764b`의16 PASS/5 FAIL 중1개는 정상 정보성 `NEWS_FILTER_APPLIED`를 기대하지 않은 테스트 오류였습니다. 수정 후 최종 결과에 제품 결함으로 합산하지 않습니다.
- 두 실행 모두 데이터 키 정리·DBSIZE0·Redis 종료0·redisStopped true·입력559개 실행 중 변경0입니다. tests 밖924개 감사도 변경/새파일0입니다. 코드/문서 수정은 tests 안에서만 했습니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedNewsOverlayRedisTest.java`입니다. 기존 `tests/run_isolated_redis.py`에 `--suite news-overlay`를 추가했고 기본 core 동작은 유지했습니다. 이 단계에서는 기존18개를 재실행하지 않았습니다. 당시 문서46개 대상2개 검사 PASS이며 실제 차감/환불 원장은 다음 단계에서 확장했습니다.

## 분석 HTTP·MySQL/Redis 과금·환불·리포트 단계 결과

실제 Feature1/2/3 분석 POST→JWT/validation→CreditService/AnalysisReportService→JPA/MySQL과 실제 Feature3OverlayService→Redis를 연결했습니다. 계산·가격/benchmark/금리·AnalysisApiClient 경계는 합성 fixture이며 실제 FastAPI/모델/외부 금융 API는 호출하지 않았습니다. [메서드별21개 검증 방법·명령·격리 구성·실패 근거](isolated-analysis-billing.md)에 자세히 기록합니다.

- 최종 `run-ekmi9ac6`: **21개15 PASS/6 FAIL**, skipped/error0. 잔액 부족/무인증/잘못된 body의 차감 차단, SQL 원장 실패 rollback, 정상 분석 리포트/snapshot/소유권, 리포트 SQL 실패 시 분석 성공·차감 유지, 캐시 정책별 실제1/3/2비용을 확인했습니다.
- 기존 `F3-CREDIT-001`을 실제 SQL에서 재현했습니다. PRICE/BENCHMARK/RISK_FREE/OVERLAY 경계 오류4개 모두 HTTP500인데 차감3만 남아 잔액2·원장[−3]·리포트0입니다. FastAPI 오류 대조군은 동일 referenceId로3을 환불합니다. 경계 오류 주입이므로 각 service 내부의 일반 fallback 동작과 구분합니다.
- 기존 `REPORT-MAPPING-001`은 실제 저장행과 리포트 상세 HTTP 모두 회사명/종목코드가 뒤바뀌었습니다. 추가62자 회사명 사례는 MySQL1406/stock_code 길이 초과로 실제 저장 실패, HTTP200+REPORT_SAVE_FAILED/reportId null이었습니다. 같은 종목의 Feature2 리포트는 정상 저장됐습니다. 기존 결함2건을6건의 새 결함으로 세지 않습니다.
- 잔액5에 동시 Feature2 분석6개 →5개200/1개402·잔액0·원장/리포트5개. 분석 보류 중 비용 예약→다른 요청402→첫 요청 실패 환불→새 요청 성공도 PASS입니다. 일반 멱등성/모든 장애에서 exactly-once 보장은 아닙니다.
- 최초 `run-jb3hm7hn`은20개14 PASS/6 FAIL입니다. 그중1개는 warning을 data 대신 meta에서 읽어야 하는 테스트 오류였고 보정 후 PASS입니다. 긴 이름 사례는 최종21개에만 추가됐습니다. 최초 결과와 합산하지 않습니다.
- 두 실행 모두 입력562개 실행 중 변경0, 업무9테이블0행·QA 종목/trigger 정리, Redis DBSIZE0, MySQL/Redis 정상종료0·stopped true입니다. 별도 tests 밖924개 감사는 변경/소실/새파일0입니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedAnalysisBillingTest.java`, `tests/isolated_redis_fixture.py`이며 `tests/run_isolated_backend.py`에 `--suite billing`을 추가했습니다. 기본 core12개와 과거 Redis18/21개를 이번에 재실행한 것은 아닙니다. 실행 로그·XML·관측/정리/source 해시는 `tests/.runtime/backend-isolated/runs/`에 보존했습니다. 문서47개 대상2개 검사 PASS이며 로그는 `tests/.runtime/backend-isolated/billing-documentation.log`입니다.

## 배치 실제 MySQL SQL·재실행·잠금 단계 결과

이전 배치의 mock SQL 검사를 새 외부 MySQL에 연결했습니다. [31개 방법·명령·실제/합성 경계·보정 이력](../batch/tests/isolated-sql.md)을 기록했습니다. 가격/산업지수/종목수급/시장수급/재무/공매도 여섯 저장 경로를 실제 mysql-connector-python9.7.0 CMySQLConnection으로 실행했으며, CSV 두 importer는 실제 디스크 BOM CSV부터 main/argparse/DB catalog/upsert까지 연결했습니다. KIS/외부 HTTP·토큰·Spring scheduler는 실행하지 않았습니다.

- 최종 `run-ltpuu307`: **31개27 PASS/4 FAIL**, errors/skipped0. 중복 재실행1행 유지·값 갱신·timestamp/정밀도, statement 실패 후 commit 값 보존, CSV 부분 commit과 수정본 재실행, freq별 PK/only_missing/latest 조회를 확인했습니다.
- 기존 `BATCH-OHLCV-ROLLBACK-001`의 실제 영향을 확인했습니다. price/index 실패 후 transaction이 남고 다른 연결 UPDATE가1205 lock wait timeout으로 실패했습니다. 원래 연결 명시 rollback 후555 저장 성공. stock/market/financial/shorts는 실제 rollback 후 다른 연결 쓰기도 정상입니다.4개 FAIL은 기존 결함1건의 transaction/잠금 회귀입니다.
- 오류 직후 connector.in_transaction 캐시값은false였으나 서버 상태를 SELECT로 갱신하면 price/index는true였습니다. 최초 캐시값 단언만으로 통과한2개를 정상 종료 근거로 사용하지 않고, 별도 연결의 쓰기와 함께 검사를 보강했습니다.
- 최초 `run-9u8fut73`:27개23 PASS/4 FAIL. 그중 공매도2개는 V35의 DECIMAL(24,6) 변경을 반영하지 않은 테스트 기대값 오류였고 보정 후 PASS입니다. 최종에는 rollback하는4개 대조군의 잠금 검사를 추가했습니다. 최초/최종 결과를 합산하지 않습니다.
- NaN/±Infinity는6경로×3입력 모두 실제 MySQL1054로 거절·저장0행입니다. 비유한 parser의 기존 결함이 고쳐졌다는 의미가 아닙니다. 실패 statement의 중간 값이 다음 성공 commit에 섞여 저장되는 현상도 이번에는 관측하지 않았습니다.
- 두 실행 모두 원본jobs/migrations/검사64개 실행 중 변경0, 업무6테이블·QA stock/index/trigger 최종0, MySQL 정상종료0/mysqlStopped true입니다. worker의 금지된 network/DB/HTTP/subprocess/.env/범위 밖 쓰기 시도0. 별도 tests 밖924개 감사도 변경/소실/새파일0입니다.

새 코드는 `batch/tests/run_isolated_sql.py`, `batch/tests/sql_cases.py`이며 모든 로그/CSV/결과는 `batch/tests/.runtime/isolated-sql/`에 있습니다. 기존 mock159개를 이번에 재실행한 것은 아닙니다. 이 증분을 포함한 문서48개 대상2개 검사 PASS이며 로그는 `batch/tests/.runtime/isolated-sql/documentation.log`입니다.

## 수동 매핑·종목 초기화·해외 차트 SQL 단계 결과

앞선 배치31개에 포함하지 않은 세 도구의 실제 SQL 경계를 [22개 방법·명령·실제 파일·동시성 순서](../batch/tests/isolated-sql-tools.md)로 확장했습니다. 외부 KIS는 합성 객체로 대체하고 원본 main/argparse·CSV/pandas·카탈로그/저장 SQL·JSON/CSV 출력은 실제입니다.

- 최초 `run-sy8c244c`와 재확인 `run-o5krlp_f` 모두 **22개14 PASS/8 FAIL**, errors/skipped0입니다. fixture 오류는 없었으며 두 실행을 합산하지 않습니다. 두번째 실행에는 동시 생성 오류 이후 같은 연결의 조회/rollback/재호출 관측을 추가했습니다.
- 신규 `BATCH-BOOTSTRAP-RACE-001`: sector/industry 모두 두 연결의 첫 SELECT를 없음으로 맞춘 뒤 INSERT하면 한 호출은 ID, 다른 호출은1062입니다. DB는1행이며 실패한 연결의 재조회는 rollback 전0행, rollback 후 같은 제품 메서드가 저장된 ID를 반환합니다. REPEATABLE-READ에서 실제 관측했고 다른 격리 수준/운영 발생률은 별도입니다.
- 기존 `BATCH-OVERSEAS-FREQ-001`: W/M main이 모두 return0이지만 같은 날짜의 일봉111을 freq4/222로 덮었고 freq5/6 행은 없었습니다. 실제 PK/upsert의 영향이며 원천 주기 규격 검사는 아닙니다.
- 기존 bootstrap/overseas rollback 누락은 실제 잠금과 다른 연결1205로 재현됐습니다. 해외 main 경로는 finally close 후 rollback·잠금 해제로 PASS입니다. Repository 직접 호출의 결과를 main 종료 뒤에도 잠금이 남는 것으로 확대하지 않습니다.
- 기존 mapping encoding은 고유401행→973행/line288 중복 오류, main close 후 미커밋 분류가NULL로 복원됐습니다. 상장일 후보 결함도 실제 stock.listed_at=NULL로 확인했습니다.8개 실패는 기존5결함의6회귀+새1결함의2회귀입니다.
- 두 실행 모두 입력65개 실행 중 변경0, 업무6테이블·QA stock/index/sector/industry/trigger0, MySQL exit0/stopped true입니다. 허용 DB 연결각93, 금지된 IO 시도0입니다. 별도 tests 밖924개 감사도 변경/소실/새파일0입니다.

새 코드는 `batch/tests/sql_tool_cases.py`이며 실행기에 `--suite tools`를 추가하고 공통 결과 수집을 재사용했습니다. 과거 core31개/mock159개를 이번에 재실행한 것은 아닙니다. 증거는 `batch/tests/.runtime/isolated-sql/` 두 run과 `tools-source-audit.json`입니다. 문서49개 대상2개 검사 PASS이며 로그는 `tools-documentation.log`입니다.

## Spring 배치·실제 JPA·달력·예약 실행 단계 결과

과거 Repository/배치 본체 mock의 경계를 실제 JPA/MySQL로 확장했습니다. [22개 방법·재현 명령·시간대 조건·실패 지점](isolated-spring-batch.md)을 기록했습니다. 실제 KIS 공유 scheduler·산업지수/시장/종목 수급 service·재시도·SQL 휴장일 달력을 사용하고 원천 Fetcher/Client만 합성 응답으로 대체했습니다.

- 최종 `run-kjg9h0dg`: **22개19 PASS/3 FAIL**, errors/skipped0. 산업지수 대상별 SQL rollback, 시장 수급 재시도/최신일 skip/DB fallback, 종목 service backfill 중복 제거, 실제 Spring 예약 콜백→SQL 저장, 단일 scheduler 중첩 차단/복구가 통과했습니다.
- 신규 `BATCH-STOCK-DATE-001`: LocalDate.MAX를 실제 최신 수급 조회 상한으로 전달해 MySQL DATE 오류가 발생했습니다. 종목별 try/catch 이전의 대상 선택에서 중단되어 두 회귀 모두 원천 구독0입니다. 우선순위/오류 종목 이후 계속 처리에는 도달하지 못했습니다. 같은 Repository의 상한9999-12-31은 정상이며 scheduler 직접2회는 예외 흡수/guard 복원·저장0으로 관측했습니다.
- 신규 `BATCH-INDEX-TIMEZONE-001`: KST 당일 자정 bar→실제 SQL 전날15시→JPA 전날15시Z. 제품이 toLocalDate로 비교해 다음 실행 skipped0/success1/fetch 총2였습니다. 최초 UTC/UTC와 최종 JDBC Seoul/Hibernate override 없음 조건 모두 재현됐고 최종 JVM 기본 시간대도 Asia/Seoul입니다. 확인한 영향은 중복 수집/upsert이며 원천 비용/운영 발생률은 미측정입니다.
- 시장 수급의 행별 save는 후속 SQL 오류에도 먼저 commit한 행이 남았고 summary savedRows1/실제SQL2였습니다. 마감 오류의 저장 자료 fallback은 새행0/savedRows2입니다. 정책 관측 PASS이며 savedRows를 신규 저장 건수로 해석하지 않습니다.
- 최초 `run-c45jq2eg`는20개17 PASS/3 FAIL이며 fixture/빌드 오류는 없었습니다. 후속에는 원인/오류 흡수 대조군2개·관측을 추가했습니다. 세 실패를 유지했으며 두 실행을 합산하지 않습니다.
- 두 실행 모두 입력563개 해시 변경0, 업무9+배치3테이블0행, QA stock/index/holiday/trigger0, MySQL exit0/stopped true입니다. tests 밖924개 감사도 변경/소실/새파일0이며 `tests/.runtime/backend-isolated/batch-source-audit.json`에 있습니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedBatchJpaTest.java`이고 실행기에 `--suite batch`/`--batch-timezone`을 추가했습니다. 기존 core/billing/Redis/배치 mock을 재실행한 결과가 아닙니다. 시간대 선택은 테스트 전용이며 application 설정은 변경하지 않았습니다. 증거는 `tests/.runtime/backend-isolated/runs/` 두 run에 있습니다. 문서50개 대상2개 검사 PASS이며 `batch-documentation.log`에 기록했습니다. 최초 부분 문자열 검사에 걸린 일반 문서 표현3곳을 보정했고 값은 출력하지 않았습니다.

## SEC 13F ZIP·native SQL·매핑 경쟁 단계 결과

[24개 방법·합성 ZIP·정정 정책·SQL 장애·재현 명령](isolated-sec13f.md)을 추가했습니다. 이전13F mock 검사에서 실행하지 않은 실제 ROW_NUMBER/NOT EXISTS/GROUP BY 집계와 Spring Data JPA를 새 MySQL에 연결했습니다. parser/service/Repository는 실제이며 동시성 검사에서만 실제 SELECT 완료 후 반환 시점을 맞추는 테스트 AOP advice를 사용합니다. 외부 SEC/HTTP/JWT는 사용하지 않았습니다.

- 최종 `run-3zv3x1nj`: **24개21 PASS/3 FAIL**, errors/skipped0. 기관별 최신 공시의 복수CUSIP 합산·날짜 동률·종목/분기 분리·빈 정정의 이전 보유 제외·발행주식 분모/소수8자리·다음 분기 재계산과 실제 디스크 ZIP 재실행이 통과했습니다.
- 신규 `SEC13F-CUSIP-RACE-001`: 두 실제 충돌 조회 모두0행에서 gate 해제→두 요청 모두SUCCESS·같은 CUSIP의 active 종목2개 저장. 순차 대조군은 다른 종목 매핑을 거절하지만 동시 INSERT는 통과했습니다. advice 제거 후 A/B 각각 재요청도 모두 IllegalArgumentException이었습니다. 자연 발생률/운영 파일 오매핑은 검사하지 않았습니다.
- 기존 `SEC13F-HEADER-001`은 필수열 누락 파일의 DTO·실제 SQL 이력이 모두SUCCESS였습니다. 기존 `SEC13F-DATE-001`도31-Feb-2026→실제 저장일2026-02-28·공시/보유/분기 집계 각1로 재현했습니다. 기존2건을 새 결함으로 세지 않습니다.
- 실제1001행 ZIP은 첫1000보유 commit 후 다음 배치 실패 시 공시1001/보유1000을 남겼고 audit createdHoldings는0이었습니다. trigger 제거 후 동일 파일 재실행은 새1/불변1000·최종1001로 복구했습니다. 공시/보유/집계 단계별 부분 commit도 확인했으며 전체 import 원자성으로 해석하지 않습니다.
- 최초 `run-dsmyb2en`은 테스트 accessor 컴파일 오류/JUnit0입니다. 첫 전체 `run-lq1sxlw5`의3실패 중1개도 실제 SQL 전에 spy가 낸 테스트 오류였습니다. 보정 후 `run-4rxzv2s_`부터 같은24개21 PASS/제품3FAIL이며 최종은 DB 이력/후속 요청 관측을 보강했습니다. 네 실행을 합산하지 않습니다.
- 네 실행 모두 입력564개 해시 변경0, 업무9+SEC6테이블·QA stock/trigger0, MySQL exit0/stopped true입니다. 마지막 두 run에는 ZIP/수정 전 사본17개 파일의 SHA/바이트 수도 남겼습니다. tests 밖924개 감사도 변경/소실/새파일0입니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedSec13fJpaTest.java`이며 runner에 `--suite sec13f`를 추가했습니다. 제품은 미수정이고 이전 suite의 재실행은 아닙니다. 실행 증거는 `tests/.runtime/backend-isolated/runs/` 네 run, 별도 감사는 `sec13f-source-audit.json`에 있습니다. 문서51개 대상2개 검사 PASS이며 로그는 `sec13f-documentation.log`입니다.

## 발행주식수·미국 마스터·OpenDART 실제 JPA 단계 결과

[28개 방법·명령·합성 전송·실제 SQL·동시성 순서](isolated-shares-master.md)를 기록했습니다. 실제 SEC client/parser와 미국 마스터/SEC·OpenDART 발행주식/법인매핑/일일 서비스, 실제 JPA callback·트랜잭션 관리자를 새 MySQL에 연결했습니다. SEC 전송은 합성 ExchangeFunction, OpenDartClient는 DTO/Mono mock이며 외부 요청은 하지 않았습니다.

- `run-z49hgm1r`: **28개26 PASS/2 FAIL**, errors/skipped0. 첫 전체 실행이며 fixture/컴파일 보정 오류는 없었습니다. 미국 마스터 전체 rollback·재실행/alias callback·시장별동일ticker, SEC 대상SQL정렬/오류계속처리·저장복구, OpenDART 매핑 rollback·행별저장·일일순서가 통과했습니다.
- 신규 `SEC-SHARES-STALE-STOCK-001`: SEC가 기존stock을 읽은 뒤 응답 보류→실제 master가 New Alpha Labs commit→JDBC 확인→SEC 해제. 나중 종목 저장이 company_name/sec_company_name을 Old Alpha Labs로 덮었습니다. 새 alias는 남아있습니다. 단일 객체를 mock으로 바꾼 결과가 아니라 실제 두서비스/JPA/SQL의 갱신 손실입니다.
- 기존 `SEC-SHARES-OVERFLOW-001`은 합성2^64+1의 JSON 숫자/문자열 모두 실제 issued_shares_total=1로 저장됐습니다. 두 단언을 가진1개 FAIL이며 신규 결함으로 세지 않습니다.
- 미국 마스터는 뒤 alias 실패/실제 회사명 길이 오류에도 전체 stock/alias를 rollback했습니다. SEC의 issued commit 뒤 stock시각 UPDATE 실패는 issued1을 남겼고, OpenDART 행별 save 실패는 summary created2/실제SQL3을 남겼습니다. 부분 commit은 관측으로 분리하고 원자성을 과장하지 않습니다.
- 실제 SEC/OpenDART cron callback 각1회 완료·요청1/2·issued3행, SEC 단일인스턴스 중첩 차단/후속실행, 기본Seoul 일일cron/주말을 확인했습니다. 테스트 타이머는1초 식이며 운영장기운용/분산중복은 남습니다.
- 입력565개 실행 중 변경0, 업무9+reference2테이블·QA stock/trigger0, MySQL exit0/stopped true입니다. tests 밖924개 감사도 변경/소실/새파일0입니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedSharesMasterJpaTest.java`이고 runner에 `--suite shares-master`를 추가했습니다. 제품은 미수정이며 이전 suite 재실행은 아닙니다. 증거는 `tests/.runtime/backend-isolated/runs/run-z49hgm1r`, 별도 감사는 `shares-master-source-audit.json`입니다. 문서52개 대상2개 검사 PASS이며 `shares-master-documentation.log`에 기록했습니다.

## 회원가입·이메일 인증·비밀번호 재설정 HTTP/JPA 단계 결과

기존 mock 메일 흐름의 실제 저장·rollback·동시성 공백을 [새 검증 문서](isolated-auth-email.md)로 확장했습니다. 현재 HEAD와 미커밋 제품은 그대로이며, 실제 HTTP/Security/JWT/Auth·MailAuth 서비스/JPA를 실행하고 JavaMailSender만 메모리 대역으로 처리합니다. 외부 SMTP 발송은 없습니다.

- 새 소유 MySQL에서 **26개25 PASS/1 FAIL**, errors/skipped0, JUnit20.407초입니다. 전체 실행1회이며 컴파일/구성 보정이나 추가 재실행은 없습니다.
- 신규 `AUTH-RESET-RACE-001`: 두 실제 SELECT가 같은 미사용 재설정 토큰을 읽은 뒤 두 HTTP 요청이 모두200·성공 감사2행을 기록했습니다. 최종 BCrypt는 첫 요청 암호에만 일치했고 순차 재사용은400입니다. 가능한 경합 순서를 gate로 재현했으며 운영 빈도나 여러 인스턴스 결과는 아닙니다.
- 동일 이메일 인증코드 확인과 가입의 실제 PESSIMISTIC_WRITE 조회는 각각200/400이었습니다. 가입1행/인증토큰0, 기존 미인증 사용자 dirty checking, 토큰 consume 후 users INSERT 실패의 rollback·재시도, 메일 실패로 삭제된 이전 토큰의 복구를 확인했습니다.
- JSON/form의 암호 공백 처리 차이와 reset 뒤 기존 JWT/refresh 유지200은 관측으로 분리했습니다. 세션 폐기 정책의 승인/결함 결론은 아닙니다. cleanup은 실제 메서드/SQL 호출이며 예약 콜백 검사는 아닙니다.
- 실행 입력566개 변경0, 업무9+인증2테이블/trigger0, MySQL exit0/stopped true입니다. tests 밖924개 감사도 변경/소실/새파일0입니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedAuthEmailJpaTest.java`, 실행기는 `--suite auth-email`을 추가했습니다. 증거는 `tests/.runtime/backend-isolated/runs/run-m0rwa02v`, 수정 범위는 `auth-email-source-audit.json`, 문서53개 대상2개 검사 PASS 기록은 `auth-email-documentation.log`입니다. 제품은 미수정이며 QA 전체는 미완료입니다.

## OAuth 콜백·소셜 계정·추가정보 HTTP/JPA 단계 결과

[25개 검증 방법·로컬 제공자·오류 주입·원인](isolated-oauth.md)을 추가했습니다. 실제 AuthOAuth2Controller/Spring Security authorization-code flow·state/WebSession·token HTTP·userinfo codec·OAuth 성공/실패 handler·소셜 서비스/JPA를 연결했습니다. 세 제공자 응답은 소유 loopback 합성이며 외부 Google/Kakao/Naver/SMTP에는 접속하지 않았습니다. OpenID Connect/실제 브라우저·운영 provider 설정 검사는 아닙니다.

- 최종 `run-faq332kk`: **25개23 PASS/2 FAIL**, errors/skipped0, JUnit18.370초입니다. 정상3provider 가입/재로그인·계정 identity·profile 완료·실제 JWT, state/세션 없음/거절/전송 오류, SQL rollback과 동시 신규 계정의 DB유일성 대조가 통과했습니다.
- 신규 `AUTH-SOCIAL-INACTIVE-001`: 추가정보4필드가 없는 inactive 계정이 새 OAuth 로그인 성공·refresh cookie·세션1을 받고 refresh200→추가정보 PATCH200→active로 저장됐습니다. 기존 JWT로 PATCH한 두 번째 경로도200/active입니다. 프로필이 완전한 inactive 계정은 ACCOUNT_INACTIVE/세션0으로 정상 차단됐습니다. 두 실패 메서드를 하나의 상태 우회 결함으로 기록합니다.
- social INSERT 실패는 user까지 rollback했고, session INSERT 실패는 앞서 commit한 user/social1 및 signup 감사를 남겼습니다. 재인증은 같은 사용자로 복구했습니다. 감사 INSERT 실패에도 가입/세션 발급은 성공했습니다. profile_required 계정의 보호 API 접근은 정책 관측으로 남깁니다.
- 초기 `run-kgqkk6c7`는 중복 OAuth 등록 저장소로 context 초기화 오류1개·실제25사례 미실행입니다. `run-tgt7dsqq`의25개12 PASS/13 FAIL 중12개는 cookie HttpOnly 대소문자 문자열 단언 오류,1개는 실제 비활성 상태 재활성화입니다. 테스트 bean 교체와 HttpCookie 파서로 보정한 뒤 위 최종 결과를 얻었습니다. 초기 실패를 제품 결함 수에 더하지 않습니다.
- 세 실행의 입력567개 변경0·업무9+social테이블/trigger0·MySQL exit0/stopped true입니다. 최종 소유 제공자 authorize38/token35/userinfo34·위반0·executor 종료/포트 닫힘을 확인했습니다. 첫 초기화 실패에는 provider manifest가 없으며 별도로 기록했습니다. tests 밖924개 감사 변경/소실/새파일0입니다.

코드는 `tests/java/com/qaima/qa/IsolatedOAuthJpaTest.java`, runner 옵션은 `--suite oauth`입니다. 세 run의 XML/summary/로그와 후반 두 run의 `oauth-provider.json`, `oauth-source-audit.json`, 문서54개 대상2개 검사 PASS 로그 `oauth-documentation.log`를 보존합니다. 소스/문서/JUnit 메서드25개도 대조했습니다. 제품은 미수정이며 전체 QA는 계속 진행 중입니다.

## 공매도 적재·CSV·공개 카드·예약 실행 단계 결과

관리자5경로의 실제 HTTP/JWT→FINRA/KRX client/codec/parser→JPA 종목 조회·JdbcTemplate/REQUIRES_NEW→새 MySQL, 공개2경로의 실제 공매도 조회/DTO까지 연결했습니다. 제공자 전송만 인메모리 connector이며 실제 외부 요청은 없습니다. [31개 입력·단언·명령·실행 이력](isolated-short-selling.md)을 기록했습니다.

- 최종 `run-47cv1av9`: **31개26 PASS/5 FAIL**, errors/skipped0, JUnit39.468초입니다. 관리자 권한·시장별 종목 조회·DECIMAL 정밀도·재실행·동시 upsert·배치 rollback·부분 commit 후 복구·실제 예약 콜백→SQL이 통과했습니다.
- 신규 `SHORT-RATIO-UNIT-001`: short1/total4에서 FINRA SQL/공개 카드0.25, KRX25로 같은 퍼센트 필드의 단위가 다릅니다. 프론트 소스의 추가 변환 없는 % 표시 계약과 대조했으며 실제 제공자 단위/브라우저 렌더링 검증은 별도입니다.
- 신규 `F2-SHORT-SERIES-WARNING-001`: 실제 SELECT 오류에서 최신 카드에는 LOAD_FAILED가 있지만 시계열은 HTTP200/빈 배열/경고 없음입니다.
- 신규 `SHORT-CSV-MARKET-001`, `SHORT-CSV-QUOTE-001`, `SHORT-CSV-HEADER-001`: 같은 코드의 다른 시장 종목에 저장, 따옴표 숫자 행 누락, 필수 헤더 누락에도 정상 반환을 실제 CSV/SQL로 확인했습니다.
- 최초 `run-71evatuz`의24 PASS/7 FAIL 중 추가2개는 child scheduler의 lookback 재주입과 선택한 HTTP 구성의 제품 ObjectMapper 연결 누락이었습니다. 테스트만 보정한 최종 실행에서 통과했고 제품 실패5개는 재현됐습니다. 초기 구성 오류는 결함 수에 더하지 않습니다.
- FINRA/KRX/CSV 각각1001행 중 마지막행 실패 시 첫1000 commit, 재실행 후1001행을 확인했습니다. 기존 부분 commit 관측을 실제 SQL로 확장한 결과입니다. 두 실행 모두 입력568개 변경0·업무9+공매도/QA종목/trigger/rename테이블0·MySQL exit0/stopped true입니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedShortSellingJpaTest.java`, runner 옵션은 `--suite short-selling`입니다. 두 run의 XML/summary/로그·CSV6파일 해시, `short-selling-source-audit.json`의 tests 밖924개 변경/소실/새파일0, 메서드 대조 및 문서55개 검사 증거를 보존합니다. 제품은 미수정이며 전체 QA는 계속 진행 중입니다.

## Feature2 차트·nullable·분석 직후 PDF 단계 결과

앞선18개 브라우저 검사의 빈 시계열을 확장해 실제 Chrome/React/SVG/html2canvas/jsPDF를 실행했습니다. 공개 응답은 모두 fixture이며 앞선 공매도 실제 SQL의 nullable 필드 형태를 합성 입력으로 재현했습니다. [20개 방법·좌표·실행 이력·PDF 시각 확인](../frontend/tests/browser-feature2-charts.md)을 기록했습니다.

- 최종 `run-gwm3LA`: **20개13 PASS/7 FAIL**. 요약7 sparkline·거시8선·수급 부호/hover·공매도 좌표·단일/부분null/전체null 시계열·분석5모드·모달·모바일·PDF 예외 복구가 통과했습니다.
- 신규 `FRONT-F2-SHORT-NULL-001`: 실제 DB/DTO가 허용하는 null 비율에서 기본 요약·공매도 탭·분석 리포트3경로가 toFixed 오류로 페이지 본문을 비웠습니다. FINRA 금액null은 탭 전환 전 검색 직후에도 발생합니다.
- 신규 `FRONT-F2-MACRO-DATE-001`: 날짜22/23/24의 KR3Y와23/24의 KR10Y에서 같은23일이 x380/x34에 그려졌습니다. 배열 index를 x로 사용하는 비교 차트의 불일치입니다. 기존 `SHORT-RATIO-UNIT-001`은 FINRA0.25가 화면0.25%로 표시되는 증거를 추가했습니다.
- `FRONT-F2-PDF-STYLE-001`: 기본/상세 PDF 모두 원본flex/본체811규칙을 유지하지 못하고 복제 root의 block/본체CSS없음으로 렌더했습니다. 기존 Feature3의 공용 PDF 유틸 관측과 함께 추적하며 독립 원인으로 단정하지 않습니다. 최초 기본PDF PASS 이후 두 실행에서 실패해 타이밍 변동도 남겼습니다.
- 실제 PDF2파일5페이지는 파싱/A4/페이지수는 맞지만, 상세 마지막 페이지의 어두운 픽셀0.009565%로 빈 페이지 검사가 FAIL입니다. PNG에서 제목·마지막 결론의 페이지 경계 분할과 마지막 페이지의 거의 빈 출력을 확인했습니다. 검사기는 실패해도 모든 PNG/JSON을 남기도록 보강했고 임계값·실패 exit는 유지했습니다.
- 최초20개10 PASS/10 FAIL의4개는 테스트 차트 높이 가정/selector 오류였습니다. 두 번째12 PASS/8 FAIL의1개는 별도 캔들 추가조회까지 금지한 단언이었습니다. 최종에서는 Feature2 요청 불변과 캔들 관측을 분리했고 제품 실패7개를 보존했습니다. 세 실행 모두 미등록API0/pageerror3/소유 preview종료, 입력187개 및 dist11개 해시를 남겼습니다.

파일은 `frontend/tests/browser-feature2-charts.cjs`, `feature2-chart-fixtures.cjs`, `inspect-report-pdf.py` 및 관련 tests 문서입니다. `.runtime/browser-feature2-charts`에 세 run의 fixture·요청·summary·PDF/PNG·사례표 대조와 tests 밖924개 변경/소실/새파일0 감사를 보존합니다. 문서 검사 대상은56개입니다. 제품과 실제 DB는 변경하지 않았으며 전체 QA는 계속 진행 중입니다.

## Feature1 재무·차트·검색 경합·분석 직후 PDF 단계 결과

[25개 방법·고정 입력·요청 보류 순서·독립 좌표·PDF 검사](../frontend/tests/browser-feature1-charts.md)를 추가했습니다. 실제 Chrome/React와 SVG·canvas·PDF를 실행하고 Public API만 합성 응답으로 대체했습니다. 직전 Feature2 빌드와 소스187개·산출물11개 해시가 동일해 해당 빌드를 재사용했습니다.

- 최종 `run-Y8f2W1`: **25개18 PASS/7 FAIL**. 재무 우선순위/fallback·기간별 모달·독립 가격/막대 좌표·EMA/Stoch·기간별 재무 요청·실패 격리·영어/모바일이 통과했습니다.
- `FRONT-F1-SEARCH-RACE-001` 두 사례: A lookup 또는 재무를 보류하고 B 완료 후 해제하면 A가 최신 이름/지표/캔들을 덮습니다. `FRONT-F1-STALE-FINANCIAL-001`: 순차 B 재무 실패 후에도 A 지표가 남습니다. `FRONT-F1-ANALYSIS-RACE-001`: B 분석0회인데 늦은 A 분석이 B 화면에 붙습니다.
- `FRONT-F1-PRICE-DATE-001`: 실제 KST00:00:00~23:59:59 요청은 정상이나 리포트에는 양쪽 오전9시로 표시됩니다. 요청과 표시의 차이로 한정합니다.
- `FRONT-F1-PDF-STYLE-001` 두 사례: Feature1 자체 다운로드 함수에서도 원본flex/811규칙이 복제문서block/본체CSS없음으로 바뀝니다. 실제 PDF2파일5페이지를 검사했습니다. 상세 마지막 페이지의 픽셀 비율0.0593824%로 별도 검사도 FAIL이나 실제 본문이 있으므로 빈 페이지로 단정하지 않습니다. 시각 검토에서 스타일 변화와 페이지 경계의 제목/본문 분리·일부 글자 절단을 확인했습니다.
- 최초 `run-xMxJ2Q`는24개16 PASS/8 FAIL이었습니다. 표시 단위/대소문자 관련 테스트 오류2개를 보정하고 날짜 비교1개를 추가했습니다. 제품 경합4개/PDF2개는 두 실행에서 재현됐습니다. 실행별 수치를 합산하지 않습니다.
- 두 실행 모두 미등록 API0·pageerror0·소유 preview 종료입니다. 최종 fixture24개·요청458개·PDF/PNG·실행기/fixture 해시와 사례표 대조를 보존했습니다. tests 밖924개 변경/소실/새파일0이며 제품/실제DB는 변경하지 않았습니다.

증거는 `frontend/tests/.runtime/browser-feature1-charts/`입니다. 문서57개 대상2개 검사 PASS(2.009초)를 `documentation.log`에 기록했으며 전체 QA는 계속 진행 중입니다.

## Feature3 차트·CAPM 표시·입력 경합 단계 결과

[26개 방법·독립 기대값·실행 이력](../frontend/tests/browser-feature3-charts.md)을 추가했습니다. 실제 Chrome/React/CSS/SVG에 고정 Public API 응답을 전달하고 SCL/SML 좌표·종목/벤치마크 전환·프론티어/CAL·효용·오버레이 및 입력 경합을 검사했습니다.

- 최종 `run-Gy8WPx`: **26개23 PASS/3 FAIL**. 위험 기여도/파이·혼합수익·4투자수준·nullable/단일점·SVG 좌표·hover·영어/모바일이 통과했습니다. 무차별곡선100점의 역변환 효용 오차는0.000030283입니다. 서버 optimizer를 실행한 증거는 아닙니다.
- 신규 `FRONT-F3-REPORT-META-001` 두 사례: A+B 분석 대기 중 A삭제→대상B만 표기, 1년 분석 뒤6개월 선택→기존 리포트 기간6개월. 분석 요청과 결과 본문은 이전 입력이고 메타데이터만 현재 입력을 따릅니다.
- 신규 `FRONT-F3-PREVIEW-STALE-001`: A+B 비용 미리보기 후 A삭제→확인→B만 분석. 미리보기 재요청 없이 다른 대상이 제출됩니다. 실제 차감 차이로 확대하지 않습니다.
- 최초26개16 PASS/10 FAIL에는 테스트 오류8개가 있었습니다. 다음26개22 PASS/4 FAIL의1개는 CSS 픽셀 hover 반올림입니다. 보정 후26개23 PASS/3 FAIL, 비용 fixture를 실제 항목당1credit으로 맞추고 차트 PNG를 추가한 최종도 같은 결과입니다. 재실행을 합산하지 않습니다.
- 네 실행 모두 원본/복사본187개·이전 빌드11개 일치, 미등록 API0·pageerror0·소유 preview 종료입니다. 최종 fixture25개/요청279개/관측25개와 PNG·사례/해시 대조를 보존했습니다. SCL/SML/프론티어 단독 그림을 직접 확인했습니다. 기존 PDF 실패는 이 단계에서 재실행하지 않았습니다.

파일은 `frontend/tests/browser-feature3-charts.cjs`, `feature3-chart-fixtures.cjs` 및 tests 문서입니다. 증거는 `frontend/tests/.runtime/browser-feature3-charts/`이며 tests 밖924개 변경/소실/새파일0입니다. 문서58개 대상2개 검사 PASS(1.769초)를 기록했습니다. 제품/기존 DB는 미수정이고 전체 QA는 계속 진행 중입니다.

## Feature3 실제 HTTP·FastAPI 계산·과금/리포트 SQL 단계 결과

[18개 입력·수학 기대값·오류 주입·실행 방법](isolated-analysis-pipeline.md)을 추가했습니다. 이전 billing의 계산 mock과 이전 TCP 검사의 원장/report mock 경계를 연결했습니다. 실제 JWT/HTTP→Spring CreditService/MySQL→FastApiAnalysisClient/WebClient→uvicorn/Pydantic/포트폴리오 계산·결정론적 설명→report SQL/소유자 상세 HTTP를 같은 요청에서 실행했습니다. 가격·벤치마크·금리 service는 합성 DTO이고 브라우저·실제 원천·유료 LLM은 포함하지 않습니다.

- 최종 `run-4b_55m01`: **18개18 PASS/0 FAIL**, errors/skipped0, JUnit25.344초·Gradle exit0입니다. 네 투자 수준·영어·현재가 생략·벤치마크 일부 없음·전부 빈 가격의 대체 결과·저장 포트폴리오 입력·기존 report 불변·실제 Redis 재무 overlay 경로를 확인했습니다.
- 실제 HTTP data=SQL snapshot=소유자 상세 결과를14개 report에서 비교하고, 타 사용자 상세404도 확인했습니다. 정상 분석은1크레딧, Python422/500은−1/+1 환불, report INSERT 오류는 계산200·차감 유지·저장 경고였습니다. 기존 준비 단계 환불 누락은 별도 FAIL로 유지합니다.
- NumPy 표본 공분산으로 계산한 현재 변동성0.06431484731932816과 실제0.064315가 허용오차 내에서 일치합니다. 추가 검사기로 실제 TCP의 네 투자 수준·현재/네 후보를 독립 위험기여도·정책 수익 일관성·수준 불변 기준으로420회 대조해 실패0, 최대 절대오차0.000000765821입니다. 독립 수학 대조는18사례의 보강 증거로 중복 합산하지 않습니다.
- 최초17개5 PASS/12 FAIL은 가격 fixture의 non-null 메타데이터 누락으로 정상 계산에 도달하지 못했습니다. 두 번째18개16 PASS/2 FAIL은 비동기 종목 배열 순서 가정 오류였습니다. 테스트만 보정했고 초기 실패를 제품 결함에 더하지 않습니다.
- 세 실행 모두 입력598개 변경0·업무9테이블/QA종목/trigger0·Redis DBSIZE0, MySQL/Redis 정상종료0입니다. FastAPI는 SIGTERM/exit−15·소유 포트 닫힘이고 강제 kill은 없었습니다. Python 외부 연결/하위프로세스/dotenv 접근 시도0, tests 밖924개 변경/소실/새파일0입니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedAnalysisPipelineTest.java`, `tests/build_pipeline_fixture.py`, `tests/isolated_fastapi_fixture.py`, `tests/pipeline_asgi.py`, `tests/inspect_pipeline_evidence.py`이고 runner는 `--suite analysis-pipeline`을 추가했습니다. 증거는 `tests/.runtime/backend-isolated/runs/run-4b_55m01`과 초기 두 run에 보존합니다. 메서드15개/문서15행/실행18사례를 대조했고, 문서59개 대상2개 검사 PASS 로그는 `tests/.runtime/backend-isolated/pipeline-documentation.log`입니다. 제품/기존 DB는 미수정이며 전체 QA는 계속 진행 중입니다.

## Feature3 브라우저·실제 서버/계산·SQL·저장 PDF 단계 결과

[8개 방법·실제/합성 경계·실행/보정 이력](../frontend/tests/browser-server-pipeline.md)을 추가했습니다. Linux Chromium153의 현재 React 빌드→실제 로그인/쿠키→Spring→FastAPI→크레딧/리포트 SQL→화면·저장 리포트 PDF를 연결했습니다. 가격/벤치마크/금리 service는 합성 DTO, LLM은 키 없는 결정론적 fallback이며 외부 금융 제공자 검증은 별도입니다.

- 최종 `run-0n529i50`: **8개7 PASS/1 FAIL**, errors/skipped0, JUnit81.420초입니다. 정상/Redis 재무 overlay·수량 저장/새로고침·계산500 환불·잔액0 차단·타 사용자 리포트404·refresh/로그아웃이 통과했습니다. 실제 FastAPI 호출5회 중 계산200은4회이며 정상 리포트3개가 저장됐습니다. 나머지1회는 계산500/환불입니다.
- `FRONT-F3-REPORT-SAVE-001`: 소유 report INSERT를 거절하면 실제 계산200·차감−1·report0이고 HTTP meta에 REPORT_SAVE_FAILED가 있으나 화면 안내가 없습니다. 프론트 fetchPortfolioAnalysis가 data만 반환하는 경로를 확인했고 세 실행에서 재현했습니다. 제품 수정은 하지 않았습니다.
- 브라우저6개월/125로그수익률에 독립 LedoitWolf 공분산으로 계산한 변동성과 실제 응답/화면6.7%를 대조했습니다. 실제 HTTP data·SQL snapshot·저장 상세가 일치하며 PDF 재다운로드로 추가 분석/차감은 없었습니다. PDF1개/A4 1페이지의 파싱·본문 검사 PASS, 실제 PNG에서 한글/메타데이터/본문 끝까지 확인했습니다. 이전 분석 직후 PDF의 스타일/긴 문서 실패는 유지합니다.
- Windows interop·Linux 임시 경로/한글 글꼴·선택자·세션 soft revoke·숫자 재직렬화·HTTP codec 날짜 설정의 테스트 보정 이력을 보존했습니다. 최종8개만 집계하며 중단된1실행은 정상종료 증거가 없다는 한계도 기록했습니다.
- 최종 입력600개 변경0·업무9테이블/QA종목/trigger0·Redis0, MySQL/Redis/FastAPI 종료·브라우저/프록시8개 정리를 확인했습니다. Python 외부 연결/하위프로세스/dotenv 접근 시도0, tests 밖924개 변경/소실/새파일0입니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedBrowserPipelineTest.java`, `frontend/tests/browser-server-pipeline.cjs`이며 runner에 `--suite browser-pipeline`을 추가했습니다. 증거는 `tests/.runtime/backend-isolated/runs/run-0n529i50`과 `frontend/tests/.runtime/browser-server-pipeline/run-0n529i50`입니다. 문서60개 대상2개 검사 PASS(2.013초)는 `tests/.runtime/backend-isolated/browser-pipeline-documentation.log`에 기록했습니다. JUnit/문서/브라우저8개 결과 대조·임시 manifest0은 `browser-evidence-audit.json`, tests 밖924개 비교는 `browser-pipeline-source-audit.json`에 보존했습니다. 전체 QA는 계속 진행 중입니다.

## Redis TCP 단절·재시작·동일 클라이언트 복구 단계 결과

[22개 방법·장애 제어·실제/합성 경계](isolated-redis-lifecycle.md)를 추가했습니다. 소유 byte proxy로 기존 TCP 종료·연결 거절·바이트 차단을 만들고 실제 Redis7.0.15/Lettuce/제품 service의 대체 조회를 실행했습니다. Repository·외부 provider·모델·observation은 합성 경계이며 실제 MySQL/HTTP/브라우저는 이 단계에 없습니다.

- 최종 `run-c0aa7rdm`: **22개22 PASS/0 FAIL**, errors/skipped0, JUnit110.435초·Gradle exit0입니다. 단절 중 가격/지분/산업/Peer/실시간/랭킹/뉴스/metrics·preview 결과가 정책대로 유지되고 같은 client/service가 복구됐습니다.
- 실제 Redis7회 재시작에서 소유 Popen exit0→새 run_id→빈 DB→소유 표식/ACL 복원→7종 캐시 재생성·값/TTL을 확인했습니다. snapshot/AOF는 껐으므로 영속성/failover 복구 검사가 아닙니다.
- 두 독립 client/resource/reader는 공유 warm 캐시를 재사용했고24개 겹친 cold 조회는 DB각1회/총2회입니다. 같은 JVM의 두 객체이며 여러 backend 프로세스를 시작한 것은 아닙니다.
- timeout2초가 서비스별로 누적돼 price4.232초·snapshot8.378초·뉴스 전체28.013초였습니다. fallback 값/경고는 정상이나 전체 Feature2 조립/브라우저 대기까지 검증한 것이 아니므로 [fallback 인계 문서](FALLBACK_QA_HANDOFF.md)의 우선 남은 범위로 기록했습니다.
- 입력570개 실행 중 변경0·fixture/owner 삭제 후 Redis0·Redis exit0, 프록시/제어 포트·thread 종료/활성socket0·제어오류0입니다. tests 밖924개 감사는 변경/소실/새파일0입니다. 첫 전체 실행에서 통과했고 중복 합산할 재실행은 없습니다.

새 코드는 `tests/redis_fault_proxy.py`, `tests/java/com/qaima/qa/IsolatedRedisLifecycleTest.java`이며 runner에 `--suite lifecycle`을 추가했습니다. 증거는 `tests/.runtime/redis-isolated/runs/run-c0aa7rdm`입니다. 문서62개 대상2개 검사 PASS(1.590초)를 `tests/.runtime/redis-isolated/lifecycle-documentation.log`에, 메서드/문서/JUnit22개 대조·입력/정리를 실행 폴더의 `lifecycle-evidence-audit.json`에 기록했습니다. 기존 캐시/뉴스 결함은 유지하며 전체 QA는 계속 진행 중입니다.

## 추적표 증분

| 요구사항 | 제품 구현 | 새 증거·남은 범위 |
|---|---|---|
| Feature2 공개 카드·조회·관심종목·분석 UI | Feature2MockPage, StockInputBox, Feature2ExternalFactorPanel, api/feature2/news/watchlist | Chrome18개. 실제 서버 저장·소유권·원천 데이터 별도 |
| KST 기간·투자 수준4개·언어 | buildAnalysisDateRange, DictContext, investLevel, api/feature2 | 4기간과4수준을 짝지은 요청 body PASS, 영어 warning PASS. 전 조합/실제 설명 품질 별도 |
| 부분 결과 유지·비동기 검색 일관성 | handleAnalyzeClick, relatedStocksTask, applyCandleResponse | 새 실패3건. 재현 gate·응답 순서·DOM·PNG 보존 |
| 모바일 | Feature2MockPage/AnalysisResultPanel | 390×844 한 fixture의 분석 완료/문서 가로넘침 없음. 전체 차트·PDF·접근성 별도 |
| 캐시 값·TTL·재사용·오류 fallback | RedisConfig, MarketSnapshotCacheService, RealtimePriceService, PriceSnapshotReader, IndustryIndexReaderImpl, PeerClusterServiceImpl, TopRankingReader | 소유 Redis/Lettuce18개15 PASS/3 FAIL.4개 짧은 TTL 자연 만료, 긴 TTL PTTL,24개 겹친 조회·Repository1회.망 장애·실제 원천 결합은 남음 |
| 뉴스 원천 캐시·metrics·추가비용 | NewsSentimentService, Feature3OverlayService | 추가21개17 PASS/4 FAIL.실제7키 저장/재사용·PTTL·ACL·뉴스block timeout, 기존 캐시오판 및 새 갱신경쟁.비용 preview/estimate까지이며 실제 차감/환불 별도 |
| 분석 차감·환불·리포트 저장 | FeatOneController, Feature2AnalyzeController, Feature3AnalyzeController, CreditService, AnalysisReportService, Feature3OverlayService | 추가21개15 PASS/6 FAIL.실제 HTTP/JWT/MySQL/Redis·SQL 실패·동시 요청.기존 환불/매핑 결함을 실제 저장소에서 확인.계산/원천/모델 경계는 fixture |
| 배치 SQL·중복 적재·실패 복구 | price/index/stock flow/market flow/financial/short selling Python jobs | 추가31개27 PASS/4 FAIL.실제 MySQL upsert·독립 SELECT·오류 trigger·다른 연결 잠금·디스크 CSV main.기존 OHLCV rollback 누락의 잠금 영향 확인.외부 제공자/scheduler/나머지job은 남음 |
| 수동 분류·초기화·해외 주기 저장 | apply_stock_industry_manual_mapping, bootstrap_stock_from_kis, kis_overseas_daily_chartprice | 추가22개14 PASS/8 FAIL.실제 dry-run/close rollback·canonical 동시 생성·파일 roundtrip·일봉 덮어쓰기.새 identity 경쟁과 기존5결함 확인.원천/전체backfill/scheduler 별도 |
| KIS 배치 SQL·달력·예약 실행 | KisMarketDataSyncScheduler, IndustryIndexOhlcvSyncService, InvestorFlowBatchSyncService, TradingCalendarServiceImpl | 추가22개19 PASS/3 FAIL.실제 cron→SQL·KST 경계·휴장일·rollback·재시도·guard.새 날짜 상한/일자 변환2결함.종목 대상 정렬/계속 처리와 다른 scheduler·실제 원천은 남음 |
| SEC 13F 적재·native SQL·분기 보유율 | Sec13fDataSetParser, Sec13fInstitutionalHoldingImportService, Sec13fHoldingRepository 및 실제 JPA | 추가24개21 PASS/3 FAIL.최신/정정/기관 합산·분모·부분commit·재실행 확인.새 CUSIP 경쟁과 기존2결함의 DB 영향.실원천/관리자HTTP/프로세스경쟁/운영성능은 남음 |
| 종목 마스터·SEC/OpenDART 발행주식·예약 | UsStockMasterSyncService, SecIssuedSharesSyncService, IssuedSharesSyncService, OpenDartCorpCodeSyncService 및 실제 scheduler | 추가28개26 PASS/2 FAIL.실제 트랜잭션/alias/행별commit/대상정렬·두cron SQL.새 오래된객체 덮어쓰기와 기존overflow 저장영향.실원천/관리자HTTP/공개분석 결합은 남음 |
| 회원가입·이메일 확인·재설정 | AuthController, EmailController, AuthService, MailAuthService와 실제 JPA | 추가26개25 PASS/1 FAIL. 실제 트랜잭션/SMTP 오류 rollback·잠금 대조·JWT 확인. 새 재설정 토큰 동시 소비 결함. 외부 SMTP/OAuth/정책/재발급 경쟁·여러 인스턴스는 남음 |
| OAuth·소셜 계정·추가정보 | AuthOAuth2Controller, SecurityConfig, OAuth2LoginSuccessHandler/FailureHandler, OAuth2SocialLoginService, UserService | 추가25개23 PASS/2 FAIL. 합성 로컬제공자/실제 state·token/userinfo·SQL·JWT. 새 비활성 상태 우회2경로/1결함. 외부 provider/OIDC/브라우저 쿠키/계정연결 정책은 남음 |
| 공매도 적재·CSV·공개 카드·예약 | Finra/KrxShortSellingSyncService, ShortSellingImportService, ShortSellingFeatureService, Feature2CardService와 실제 JPA | 추가31개26 PASS/5 FAIL. 관리자5/공개2 HTTP·SQL·CSV·cron. 비율 단위/조회 경고/CSV3결함. 실원천 단위/브라우저 nullable·여러 인스턴스·프로세스 중단은 남음 |
| Feature2 거시/수급/공매도 차트·nullable·PDF | Feature2ExternalFactorPanel, Feature2TrendCharts, InvestorFlowTrendChart, ShortSellingTrendChart, AnalysisResultPanel, reportPdf | 추가20개13 PASS/7 FAIL. 실제Chrome/SVG/PDF·독립좌표·페이지PNG. null3경로/날짜정렬/비율표시/PDF2실패. 별도 마지막페이지 검사FAIL. 실API/Peer/산업전체·언어/접근성/전체PDF입력은 남음 |
| Feature1 재무/차트/검색 경합/PDF | StocksMockPage, FinancialModal, TradingViewWidget, PriceFlowBars, FinancialTimelineChart, AnalysisResultPanel | 추가25개18 PASS/7 FAIL. 실제Chrome·API gate·독립좌표·PDF/PNG. 검색/분석 경합·이전 재무 유지·표시 시각·PDF 실패. 실API/전체날짜·언어/수준·marker/접근성은 남음 |
| Feature3 SCL/SML/프론티어·입력 경합 | PortfolioMockPage, portfolio API client | 추가26개23 PASS/3 FAIL. 실제Chrome·독립좌표·효용곡선·4수준/영어/모바일·PNG. 대상/기간 메타데이터2경로·비용 미리보기 대상 불일치. 실계산/원장/저장리포트는 후속18/8개 참조. 수학전체·해외입력/접근성은 남음 |
| Feature3 실제 계산·원장/리포트 SQL | Feature3AnalyzeController, FastApiAnalysisClient, analyze_portfolio, CreditService, AnalysisReportService, Feature3OverlayService | 추가18개18 PASS. 실제 TCP/계산/SQL·독립 공분산/위험기여도·캐시 overlay·환불·14개 snapshot. 브라우저 선택 흐름은 후속8개 참조. 실원천·전역 최적성·모델 정확도·중단 복구는 남음 |
| Feature3 실제 사용자 통합 흐름 | Auth/User/Portfolio/Feature3/Report Controller, FastAPI, SettingPage | 추가8개7 PASS/1 FAIL. 실제 브라우저·SQL·계산·저장 PDF·소유권·로그아웃. 저장 실패 안내 누락. 시장 데이터 원천·기타 기능/전 흐름은 남음 |
| Redis 실제 장애·복구 | RedisConfig/Lettuce, 가격/시장/산업/Peer/실시간/랭킹/뉴스/metrics | 추가22개22 PASS. TCP 종료/거절/blackhole·7회 재시작·같은 client 복구·두 reader 공유. 실제 SQL/사용자 흐름·여러 backend 프로세스·영속성/failover·누적 응답시간은 남음 |
| Feature2 실제 조립·fallback/SQL | Feature2AnalyzeService/normalizer/assembler, Feature2 Controller/CreditService/ReportService, 하위 실제 service4종 | 추가39개17 PASS/22 FAIL. 정상 상대값/후속 단계 누락·empty warning·정량0 환불 누락. 복구36개/저장 snapshot71개. 하위 경계 대부분 합성; Redis·원천/누적 대기 남음 |
| Feature2 브라우저·실제 조립/서버/SQL | Feature2MockPage/AnalysisResultPanel/SavedReportDocument, 실제 인증/조립/차감/리포트 저장·조회 | 추가13개7 PASS/6 FAIL. 실제 SQL snapshot12개/상세GET5개. 보조 오류의 결과 숨김·저장/핵심 경고 누락·저장 정량/text 설명 누락. 정상 요약 PDF 확인. 원천/Redis/FastAPI·전 카드/차트 결합 남음 |
| Feature2 실제 Redis·조립/HTTP 취소/SQL | StockResolver/PriceSnapshotReader, IndustryReader/IndustryIndexService/IndustryIndexReaderImpl, PeerClusterServiceImpl, NewsSentimentService, 실제 Controller/Credit/report | 추가15개12 PASS/3 FAIL. 가격/산업 취소 캐시 역전·뉴스 복구 경고 잔류. 조립40.6초/재접속 최대27.340초. snapshot/상세28개·원장35개, 취소7개 차감 잔존은 명시 정책 공백. 시장SQL/여러 기사/실제 설명·다중 프로세스 남음 |
| Feature2 실제 SQL/Peer/모델의 브라우저 fallback | Feature2MockPage/AnalysisResultPanel/SettingPage, 실제 카드/뉴스/Peer·로컬 모델/SQL | 추가12개4 PASS/8 FAIL. 실제 경고·기사/RAW 차트 소실과 저장 정량/거시/NULL 실패. 새 요청 키의 UI 복구 PASS이며 동일 요청 오류 캐시 실패는 유지. 실원천/전 UI 조합·모델 품질은 남음 |
| Feature1 실제 서버/브라우저/저장 PDF | StocksMockPage/AnalysisResultPanel/SettingPage, 실제 SQL/snapshot/Redis/FastAPI | 추가17개5 PASS/12 FAIL. 실제 화면/SQL·동일 입력 복구3개·PDF2개3페이지. 저장 정량/경고·원천 fallback 실패, 별도 볼린저 위치/스타일 실패. Redis 단절/취소는 아래17개 참조. 실제 제공자·전체 UI 조합은 남음 |
| Feature1 실제 Redis/HTTP 취소·재시도/SQL | FeatOneService/Controller·snapshot/시세/metrics Redis·실제 계산/원장/report | 추가17개16 PASS/1 FAIL. 실제 단절/재시작·복합 장애·HTTP 취소/timeout/복구. 기존 원천 독립성 FAIL. 취소6개 차감 잔존/동일 body 별도 차감은 정책 관측. 브라우저 취소·프로세스/멱등성·실원천 남음 |

## Feature2 실제 조립·fallback·HTTP/SQL 단계 결과

[39개 사례·경계·정책·실패/보정 이력](isolated-feature2-fallback.md)을 추가했습니다. 실제 normalizer/조립기/assembler/HTTP/JWT/CreditService/report/JPA/MySQL을 연결하고 하위 서비스/Repository/설명 경계에 합성 장애를 주입했습니다. 추가4개는 실제 금리·공매도·산업/종목 resolver의 fallback까지 연결했으며 실제 원천/Redis/브라우저는 별도입니다.

- 최종 `run-bxtp_64j`: **39개17 PASS/22 FAIL**, errors/skipped0, JUnit18.252초입니다. 정상·수급/Peer/뉴스/설명·두 추세 오류·뉴스 timeout·부분 뉴스/거시·SQL 저장 실패·권한/입력/잔액 차단은 선택 조합에서 통과했습니다.
- `F2-ASSEMBLY-ISOLATION-001`: escaped BASE/SHORT/INDEX 오류의 후속 중단, 실제 금리 최초+대체 조회 실패, 산업 분류 없음의 독립 추세/뉴스/설명 생략, macro 현재값/시계열 zip의 정상 상대값 소실입니다. 실제 공매도 Repository 오류는 하위 service에서 처리되어 정상 나머지를 보존했습니다. escaped 오류를 모든 원천 오류의 결과로 확대하지 않습니다.
- `F2-EMPTY-WARNING-001`: 7종 Mono.empty와 explain=null에서 누락 안내가 없습니다. 경계 내구성 검사이며 정상 하위 service가 붙인 warning까지 항상 사라지는 것은 아닙니다.
- `F2-CORE-REFUND-001`: 종목 조회 오류2경로/전체 정량 오류 설정에서 실제 잔액5→4·원장−1·report1입니다. 모든 metrics 또는 정량 필드가 비어도 성공 응답/저장으로 처리되어 refund에 도달하지 않습니다. 정책 §5.4와 실제 SQL을 대조했습니다. BASE+SHORT/전체 오류는 BASE에서 중단되어 후속 장애 구독0이라는 한계도 남겼습니다.
- 복구36개는 같은 조립기에서 다른 종목/기준금리4.25·warning0·추가1차감, 저장71개는 HTTP data=SQL snapshot=소유자 상세·타 사용자404·이전 snapshot 불변입니다. 이를39사례와 중복 합산하지 않습니다.
- 초기 정상 대조 `run-lsfc11gr`의1 FAIL은 DTO 내부 warning을 data에서 찾은 테스트 오류입니다. `@JsonIgnore`/public meta 계약에 맞게 보정했고 최종 정상 대조는 통과했습니다. 초기 실행은 집계에서 제외합니다.
- 두 실행 모두 업무9테이블/trigger0·MySQL exit0/stopped true·입력 해시 변경0입니다. 최종 입력575개와 tests 밖924개 변경/소실/새 파일0 감사,75개 관측/실행 명령/XML을 보존했습니다. 제품은 미수정입니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedFeature2FallbackTest.java`, runner 선택은 `--suite feature2-fallback`입니다. 문서63개 대상2개 검사 PASS 로그는 `tests/.runtime/backend-isolated/feature2-documentation.log`에 있습니다. 메서드20개/문서20행·파라미터 포함39사례와 누적20묶음457개 집계도 대조했습니다. 다음 세션은 실행 증거 `tests/.runtime/backend-isolated/runs/run-bxtp_64j`와 위 상세 문서를 먼저 확인하세요.

## Feature2 브라우저·실제 조립/HTTP/SQL 단계 결과

[13개 사례·명령·실제/합성 경계·시각 증거](../frontend/tests/browser-feature2-server.md)를 추가했습니다. 실제 Chromium153.0.8010.12/현재 React 빌드→실제 Spring 인증/Feature2 조립/크레딧 원장/리포트 MySQL→화면/저장 상세/PDF를 연결했습니다. API 응답 대체 없이 소유 프록시로 전달했으며 시장 원천/뉴스/설명/하위 leaf 서비스는 합성 경계입니다. 실제 Redis/FastAPI는 이번에 시작하지 않았습니다.

- 최종 `run-l3jr_n8_`: **13개7 PASS/6 FAIL**, errors/skipped0, JUnit118.381초입니다. 정상·뉴스 부분 점수·Peer 오류·뉴스+설명 오류·산업 없음 안내·명시 재분석 복구·실제402는 선택한 화면 기준에서 통과했습니다. 산업 없음의 독립 뉴스/추세 생략은 앞선 조립 검사 FAIL로 계속 유지합니다.
- 기존 `FRONT-F2-AUX-001`을 보조 금리/공매도 시계열500 두 사례에서 실제 서버로 재현했습니다. 종합 분석200/원장−1/report1인데 Promise.all 오류로 결과가 화면에 표시되지 않으며, 내정보의 같은 저장 상세는 조회됩니다.
- 신규 `FRONT-F2-REPORT-SAVE-001`: 실제 SQL trigger 저장 오류의 meta REPORT_SAVE_FAILED가 화면에 표시되지 않습니다. Feature2 API는 meta를 보존하지만 warningNotes에 해당 코드가 없습니다. 정상값/차감은 유지하고 report0입니다. 기존 Feature3 meta 소실과 원인은 구분합니다.
- 신규 `FRONT-F2-CORE-WARNING-001`: metrics 전부null/INTERNAL_ERROR 안내가 없습니다. 프론트는 FEAT2_INTERNAL_ERROR만 처리합니다. 실제−1/report1도 남아 기존 `F2-CORE-REFUND-001`의 브라우저 결합 증거를 보강했습니다.
- 신규 `FRONT-F2-SAVED-METRICS-001`: 설명 오류 시 live 정량/LLM 안내·SQL snapshot은 유지되지만 저장 상세는 정량을 렌더링하지 않아 경고만 남습니다. 신규 `FRONT-F2-TEXT-EXPLAIN-001`: overall 없는 text 설명이 빈 객체 분기에 가려 live 보고서에 표시되지 않습니다. 실제 provider의 해당 응답 빈도는 측정하지 않았습니다.
- 분석 HTTP200×13/402×1, SQL snapshot12개와 상세GET5개를 대조했습니다. 총13차감에는 core 오류의 잘못된 차감도 포함됩니다. 뉴스 복구의 두번째 명시 분석은 잔액3/report2/warning0·기사2입니다. 후속 요청/SQL 단언을13사례와 중복 집계하지 않습니다.
- 정상 저장 PDF10,408,847bytes/A4 1페이지/빈 페이지0을 파싱·PNG로 확인했습니다. 정상 PDF는 설명 요약을 보여주며 정량/차트 전체 재현 통과가 아닙니다. 최종 분석 PNG의 실제 스크롤 등장과 설명 실패 저장 화면도 직접 확인했습니다. 기존 장문 PDF/스타일 실패는 유지합니다.
- 초기 `run-02clehoh`의1 FAIL은 text-only 제품 실패를 발견한 대조로 보존했습니다. 첫 전체 `run-_7ntos21`은13개5 PASS/8 FAIL이며 화면 macro3.50%와 잔액 부족 문구를 잘못 기대한 테스트2개를 보정했습니다. Peer fraction fixture/실제 스크롤도 보강했으며 최종6개 제품 실패는 유지했습니다. 초기 두 실행은 누적에서 제외합니다.
- 세 실행 모두 소유 MySQL·브라우저 정리를 확인했습니다. 최종 업무9테이블/trigger0·MySQL exit0/stopped/포트 닫힘, 브라우저/프록시/임시 프로필13개 정리, 인증 임시 manifest0입니다. 예상 밖 API/pageerror0이며 정적 CDN/폰트 origin38회는 route에서 차단했습니다. 외부 요청 시도0으로 기록하지 않습니다.
- 입력577개/프론트187개/bundle11개와 tests 밖924개 변경0을 대조했습니다. host summary/XML/`feature2-browser-evidence-audit.json`, 각 모드 summary/DOM/PNG, `feature2-browser-source-audit.json`에 근거를 남겼습니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedFeature2BrowserTest.java`, `frontend/tests/browser-feature2-server.cjs`이고 runner 선택은 `--suite feature2-browser`입니다. 문서64개 대상 링크/설정 일부 비밀값 검사2개 PASS이며 로그는 `tests/.runtime/backend-isolated/feature2-browser-documentation.log`입니다. 문서13행과 실제 사례, 최신21묶음470개379 PASS/91 FAIL 집계도 대조했습니다. 이 단계 전20묶음457개372 PASS/85 FAIL에서 갱신한 수치입니다.

## Feature2 실제 Redis·조립/HTTP 취소/SQL 단계 결과

[15개 방법·명령·장애 시점·과금 해석·실행 이력](isolated-feature2-resilience.md)을 추가했습니다. 실제 HTTP/1.1→Spring 인증/Feature2 조립→가격·산업·Peer·뉴스 서비스/제품 serializer/Lettuce/Redis→Credit/report/MySQL을 연결했습니다. 시장 Repository·금리/공매도/거시/수급 및 외부 제공자/설명은 합성 경계입니다. 실제 MySQL은 사용자/원장/리포트 저장소이며 시장 원천 SQL 전체를 검증한 것은 아닙니다.

- 최종 `run-8m54svrw`: **15개12 PASS/3 FAIL**, errors/skipped0, JUnit274.921초입니다. 정상 cold/warm·Redis 단절/바이트 차단·Redis+가격/산업 source 실패·Redis+Peer/감성 실패·실제 재시작·선택 취소/timeout 전파와 복구는 통과했습니다.
- `PRICE-CANCEL-STALE-001`: 첫 가격 source110을 보류→HTTP 취소→후속220 응답/저장 완료→첫 source 해제→실제 캐시 및 다음 HTTP/SQL110입니다. `INDUSTRY-CANCEL-STALE-001`: 같은 순서로 새 시계열58%가 옛29%로 덮였습니다. 두 reader 각각의 `.cache()` source는 계속 실행되지만 subscriber 취소의 doFinally에서 inflight 엔트리가 제거되어 다음 source와 겹쳤습니다. latch로 순서를 재현했으며 자연 발생률은 미측정입니다.
- `NEWS-CACHE-RECOVERY-WARNING-001`: 뉴스 캐시 읽기 실패 후 source 검색을 보류하고 Redis를 복원했습니다. 목록에 저장한 NEWS_CACHE_READ_FAILED가 다음 정상 cache hit의 meta/SQL에도 남았고, 목록 키만 제거한 진단 요청에서는 warnings=[]였습니다. 모델 호출은 총1회, 점수/기사 값은 보존됐습니다. 과거 transport 오류의 재게시이며 브라우저 표시까지 새로 검증한 것은 아닙니다.
- Redis 단절 중 전체 분석40.654초, 바이트 차단40.569초입니다. 가격 약4.2초→산업 약4.2초→Peer 약4.2초→뉴스 약28.0초가 순차 누적됐습니다. Redis 복원 후 같은 분석 client의 marker 확인은 최대27.340초/13회였고 새 factory로 교체하지 않았습니다. 원천 즉시 응답/뉴스1건 조건이며 SLA나 실제 여러 기사 지연 상한은 아닙니다.
- 실제 응답=SQL snapshot=소유자 상세28개를 확인했습니다. USE 원장35개/환불0 중 report 없는7요청은 명시 취소4개·실제3초 HTTP timeout1개·두 경합의 취소된 첫 요청입니다. 네 취소 지점은 각각−1/report0, 다음 정상 분석 후−2/report1입니다. **취소 환불 기준은 정책에 명시되지 않아 관측으로 남겼으며, 취소 전파 PASS를 과금 정책 적합성으로 판정하지 않습니다.**
- 초기 컴파일2회는 테스트 import/Repository 이름 오류/JUnit0, 정상 대조1회는 public 산업 시계열 value 필드 단언 오류였습니다. 첫 전체14개9 PASS/5 FAIL은 가격/뉴스 제품 실패2개,15초 재접속 관측 한도 초과2개,기존 유효 캐시에 새 TTL을 기대한 테스트 오류1개입니다.45초 관측 한도/잔여 TTL/뉴스 source latch를 보강하고 같은 패턴의 산업 경합을 추가했습니다. 초기 실행을 최종15개와 합산하지 않습니다.
- 다섯 실행의 소유 MySQL/Redis 정리를 확인했습니다. 최종 업무9테이블/trigger0·Redis DBSIZE0·서버 exit0/stopped/포트 닫힘, TCP 프록시/제어 포트/worker 종료·active socket0입니다. 실제 재시작1회는 새 run_id/빈 Redis·SQL 이전 리포트 보존을 확인했습니다. 입력579개 해시 불변 및 tests 밖 가시 코드/설정/문서924개 변경·소실·새 파일0입니다.

새 코드는 `tests/java/com/qaima/qa/IsolatedFeature2ResilienceTest.java`, `tests/isolated_fault_redis_fixture.py`이고 runner 선택은 `--suite feature2-resilience`입니다. 최종 summary/XML/`feature2-resilience-evidence-audit.json`과 `feature2-resilience-source-audit.json`에 근거를 보존했습니다. 문서65개 대상 링크/설정 일부 비밀값 검사2개 PASS이며 로그는 `tests/.runtime/backend-isolated/feature2-resilience-documentation.log`입니다. 문서15행/실제 사례/누적22묶음 집계와 다섯 실행 정리도 evidence audit에서 대조했습니다. 이전21묶음470개379 PASS/91 FAIL에서 **22묶음485개391 PASS/94 FAIL**로 갱신합니다.

## Feature2 실제 시장 SQL·Redis·FastAPI 설명 단계 결과

최종 `run-udwy3eb5`: **24개13 PASS/11 FAIL**, JUnit144.830초. [방법·전체24사례·초기 보정 이력](isolated-feature2-source-sql.md)을 기록했습니다. 기존 source mock을 실제 MySQL JPA로 교체하고 실제 가격/금리·거시/공매도·수급/산업지수/뉴스 service와 Redis, 실제 Spring→FastAPI 설명·차감/report를 연결했습니다. 제공자·감성 모델·Peer 계산은 합성 경계, observation 비동기 저장과 브라우저는 별도입니다.

실제11종 테이블 조회 장애와2개 복수 장애에서SQL1146/42S02를15회 확인했습니다. 기준금리 fallback의 후속 조립 중단, 환율/채권 오류의 정상 거시 요약 소실, 한쪽 수급 오류의 양쪽 수급 소실이 실패합니다. 가격 SQL 오류·환율 빈 행·카드 장애의 안내 누락, 수급3행 모두NULL을0/pointCount3/MIXED_OR_FLAT로 public/SQL/실제 설명 입력에 전달하는2경로도 실패입니다. 가격+기준금리 오류로 정량0인데잔액5→4/원장[-1]/report1이 남는 기존 F2-CORE-REFUND-001을 실제 원천으로 재현했습니다.

공매도/산업/뉴스 저장소의 선택 오류, 감성 모델 일부·전체 실패와복구, 감성SQL INSERT 오류의 점수 보존,17회 거시 provider 오류→기존SQL fallback, 산업10행 fallback은 통과했습니다.3기사+Redis 단절 전체84.885초/뉴스68.067초/같은 client복원 확인최대8.975초를 관측했습니다. 기존1기사40.6초와는 실제 카드/SQL 구성도 달라 기사수만의 영향으로 해석하지 않습니다. 성능SLA/최대30기사/실원천 지연은 미검증입니다.

snapshot=응답=소유자 상세46개,USE46/REFUND0,실제FastAPI44개(2개는조립중단)를 대조했습니다.44개 wire metrics는 snake_case 변환 후 Spring 축약값과 일치하며,실제Python의LLM_API_KEY_MISSING이 public/SQL에 보존됩니다. 감성 INSERT 장애 복구 직후에는 캐시 점수가 재사용되어sentiment SQL0이 남는 관측도 분리했습니다.

4실행의 MySQL·Redis exit0/포트 종료,source/업무/참조/숨긴테이블/트리거 정리,proxy/control/worker종료,FastAPI SIGTERM(exit−15)/외부시도0을 확인했습니다. 최종 입력606개와tests밖924개 해시는 보존됩니다. `tests/audit_feature2_source_sql.py`·최종 run의 `feature2-source-sql-evidence-audit.json`·`tests/.runtime/backend-isolated/feature2-source-sql-source-audit.json`에 근거를 남깁니다. 초기 실행은 중복 합산하지 않습니다. 문서66개 검사2개 PASS와24사례/누적23묶음/4실행 정리 감사도 완료했습니다. 해당 단계까지23묶음은 **509개404 PASS/105 FAIL**입니다.

## 다음 검증과 전체 완료 조건

**최우선 다음 작업은 Feature3 원천 장애·HTTP 취소/환불·재시도입니다.** 실제 외부 제공자·나머지 Swagger 사용자 흐름도 전체 미완료 범위로 유지합니다. Redis 뉴스28초의 누적 대기는 실제 조립/HTTP15개에서 약40.6초로 연결했고, 선택한 취소/복구도 원장으로 확인했습니다. 이번39개 조립/SQL·13개 브라우저·15개 실제 Redis·24개 실제 시장 SQL/설명·14개 실제Peer/로컬 모델·12개 실제 원천 브라우저 검사는 선택한 경계의 증거이며 전 하위 원천·전체 장애 조합은 남아 있습니다. [사용자 지시·세션 인계](FALLBACK_QA_HANDOFF.md)의 공통 완료 기준을 적용합니다.

기존 [남은 범위](../QA_CONTEXT.md)의 범위를 유지합니다. 이번 브라우저·DB·Redis 검사로 QA 전체를 완료하지 않습니다.

1. 선택한 report/portfolio·watchlist·세션·크레딧 및 세 분석의 HTTP/DB 경계는 위 결과로 확장했습니다. refresh 동시성 결함과 로그아웃 경쟁은 남아 있습니다. signup/email/재설정은 위26개로 실제 HTTP/JPA 경계를 확장했고, 재설정 동시 소비 결함·메일 실원천/재발급 경쟁·세션 정책은 남아 있습니다. OAuth/소셜 프로필은 위25개로 합성 제공자와 실제 HTTP/JPA 경계를 확장했고, 비활성 상태 우회·실제 provider/OIDC/브라우저·계정연결 정책 및 나머지 업무 경계는 남아 있습니다. 분석 비용/환불21개에 포함하지 않은 원천/모델 결합·클라이언트 취소/프로세스 중단·환불 복구/멱등성도 별도입니다. 저장한 DB 상태 파일만으로 프로세스 생존을 가정하지 않습니다.
2. 격리 Redis core18개와 뉴스/metrics21개, 분석 billing의 일부 실제 차감 결합으로 선택한 캐시 경계를 확장했습니다. TCP 단절/재접속·7종 캐시 재시작의 선택 범위는 추가22개로 검증했습니다. 선택한 네 실제 캐시 경로의 조립 timeout 누적/차감 결합은 위15개로 확장했습니다. 선택한 실제 시장 SQL/3기사/무키 설명은24개로확장했고 실제Peer 역방향JPA/로컬 모델은14개로 확장했습니다. 남은 것은 최대30기사/실제 제공자·전체 source 조합, 여러 backend 인스턴스 동시성입니다. 기존 Redis 쓰기는 수행하지 않았습니다. Swagger123개 정상·오류·권한·fallback 업무 흐름의 남은 경계도 유지합니다.
3. Feature2의 선택한 거시/수급/공매도 차트·nullable·분석 직후 PDF는 위20개로 확장했습니다. PDF 스타일/페이지 분할·null 화면 소실·날짜 정렬 실패와 실제 API 연동·Peer/산업 지수 전체 차트·언어/테마/접근성은 남습니다. Feature1은 위25개로 재무/차트/PDF·검색 및 분석 경합을 확장했으며 실제API·날짜 유효성/경계·전체 marker·언어/수준/접근성은 남습니다. Feature3은 위26개로 SCL/SML·프론티어/CAL·효용곡선·4수준/영어/모바일·입력 경합을 확장했습니다. 실제 계산과 UI의 선택 흐름은 위8개로 연결했고 수학/오버레이 전 범위·해외시장/환율·입력 경계·접근성과 PDF 기존 실패는 남습니다. 인증/OAuth·모바일·동시성의 나머지 경계도 유지합니다.
4. 여섯 배치 저장 경로의 실제 SQL/재실행31개와 수동 매핑/종목 bootstrap/해외 저장의 선택한 SQL22개를 추가했습니다. bootstrap backfill 분류 일관성·나머지 입력/외부 원천, 연결 단절/프로세스 중단 복구는 남습니다. KIS Spring scheduler/시간대/합성 휴장일은 위22개로 확장했으나 종목 대상 선택 오류 이후 경로·cleanup scheduler·다중 인스턴스/장기 운용은 남습니다. SEC13F native SQL·ZIP 적재는 위24개로 확장했고 실제 원천/전체 규모 성능·여러 프로세스 import/rebuild 경쟁·관리자 HTTP 결합은 남습니다. 미국 마스터/SEC·OpenDART 발행주식과 두 scheduler의 실제 JPA28개도 추가했으며 외부 원천/관리자 HTTP·공개분석/브라우저 결합, 동시 갱신 손실의 다른 필드·인스턴스는 남습니다. Java 공매도 적재/CSV/공개 카드/두 scheduler는 위31개로 확장했으나 실원천 단위/장기 복구/여러 인스턴스/브라우저 nullable 렌더링은 남습니다. 모델 정확도와 성능, 전체 문서/수정 범위 감사도 별도입니다.

Feature3의 실제 Spring→FastAPI→과금/리포트 SQL 경계는 위18개로 확장했습니다. 브라우저에서 실제 서버·계산·저장 리포트/PDF까지의 선택 흐름은 추가8개로 검증했습니다. 다른 기능과 전체 사용자 흐름의 결합은 남습니다. 가격·벤치마크·금리 원천과 실제 모델/유료 설명, 준비 단계 환불 누락 및 중단/재시도/멱등성도 별도입니다.

새 테스트 코드가 더 필요하면 반드시 tests 폴더 안에 작성합니다. 기존 `backend/src/test` 문서는 읽었지만 이번에는 수정하지 않았습니다. 실제 외부 호출은 과거 예산 기록과 전송 guard를 확인해야 하며 이번에는 사용하지 않았습니다.


## Feature2 실제 Peer·로컬 뉴스 모델·SQL 결합 결과

[Feature2 실제 Peer·로컬 뉴스 모델·SQL 결합](isolated-feature2-pipeline.md)은 **14개8 PASS/6 FAIL**(`run-5br21593`)입니다. 신규 Peer 오류 캐시 복구 실패5사례와 기존 최신일 제외1사례를 재현했습니다. 실제 역방향SQL/독립 상관·시차,모델 누락/복구·RAW fallback·SQL 저장 장애의 부분 결과 보존을 확인했습니다. 실제FastAPI61개/역방향TCP18개·모델40결과/공개69점수·snapshot/상세26개/USE26개를 대조했고 소유 서비스는 정리·종료했습니다. 이 단계 당시 누적은 **24묶음523개412 PASS/111 FAIL**이었습니다.

- 실제 Python→Spring 역방향 pack은 실제 repositories/calendar를 사용했습니다. SQL DECIMAL(18,6) 가격에서 독립 상관·산업 조정·선행/후행 시차를 계산하고,실제 모델의 확률/label/점수 식·URL 매칭·SQL 정밀도를 대조했습니다. 외부 금융/뉴스 제공자와 observation 비동기 저장은 합성 경계로 남습니다.
- 신규 `F2-PEER-ERROR-CACHE-001`: 503/8초 timeout/잘못된 날짜/실제 price SQL 오류/Peer+모델 복수 장애에서,장애 제거 뒤 재분석은Peer0을12시간 캐시에서 받아 원천 재조회가 없었습니다. 테스트 캐시 삭제 후3Peer와 수학 기대값이 복구됩니다. 정책의 TTL 길이 자체가 아니라 일시 실패 결과의 정상 캐시 취급/복구 부재가 문제입니다.
- 기존 `F2-PEER-DATE-001`: SQL stock/index 최신8월6일 대비 window Peer 최종8월5일. 명시기간91점/90수익률은 PASS입니다. 모델 경로 누락 복원,지수 SQL 실패의 RAW 대체,감성 INSERT 실패 중 계산 점수/기사/리포트 유지는 PASS입니다.
- 최종 cold 모델93.097초,Spring30초 timeout 후 전체HTTP36.941초/뉴스3기사·점수null/경고. 실제 Python 요청 종료 후 재분석은1.185초/점수3개로 복구했습니다. warmup을 끈 격리 관측이며 운영 SLA가 아닙니다. INSERT 오류 제거 후 score SQL0/유효 캐시 재사용은 자동 backfill 정책이 없어 별도 관측입니다.
- 초기 import 충돌(JUnit0)과 exact JSON 부동소수점 비교/휴장일 fixture 보정을 상세 문서에 기록했습니다. 초기 모델99.186초/종합37.076초도 보존했습니다. 최종14개만 합산하며 제품은 수정하지 않았습니다.
- 최종 입력610개/모델7파일/제품 등924개 hash 보존,MySQL/Redis exit0·데이터0/프록시·worker·Spring callback 종료,FastAPI SIGTERM−15/포트 종료,외부연결/dotenv/subprocess 시도0을 확인했습니다. 증거감사와 문서67개 검사2개 PASS를 기록합니다. 다음은 Feature3 나머지 overlay·실제 원천 브라우저/저장/PDF, 실제 외부 제공자·나머지 Swagger 사용자 흐름입니다. 전체 QA는 미완료입니다.


## Feature2 실제 SQL·Peer·로컬 모델의 브라우저/저장/PDF 결과

[12개 방법·장애 주입·화면/SQL/PDF 대조·초기 실행 이력](../frontend/tests/browser-feature2-pipeline.md)을 추가했습니다. 최종 `run-kd_uymvv`은 **12개4 PASS/8 FAIL**, errors/skipped0, JUnit225.944초·Gradle7분19초입니다. 해당 단계 당시 누적은 **25묶음535개416 PASS/119 FAIL**이었으며 실패 사례 수와 고유 결함 수는 다릅니다.

- 실제 로그인/JWT→React의 검색/분석→실제 카드/뉴스/시장 JPA·Redis→Peer 역방향 SQL/독립 계산·로컬 CPU 모델→무키 설명→과금/리포트 SQL→화면·저장/PDF를 연결했습니다. public API 응답을 재작성하지 않았습니다. 검색 매핑/랭킹·외부 금융/뉴스 제공자·observation 비동기 저장은 합성 경계입니다.
- **FRONT-F2-SOURCE-WARNING-001**: 실제 LLM_API_KEY_MISSING과 Peer 원천503의 SPRING_PACK_FETCH_FAILED가 public/SQL에 남지만 warning mapper에서 사용자 안내가 빠집니다. Peer0을 정상 후보 부족과 구분해 안내하지 못합니다.
- **FRONT-F2-NEWS-NULL-SUMMARY-001**: 모델 누락 시 기사3개/null점수와 실패 경고는 남지만 summary를 조건으로 렌더하는 보고서에서 기사목록이 사라집니다. 외부요인 뉴스탭 기사와 저장 경고는 유지됩니다. **FRONT-F2-RAW-PEER-CHART-001**: 산업SQL 장애에도 실제 RAW Peer2개/anchor·centroid91점이 남지만 industrySeries 조건으로 차트·설명 모달까지 숨겨집니다.
- 기존 저장 정량 누락, 환율SQL 오류의 정상 거시 값 소실, 수급NULL→0/보합을 실제 화면까지 재현했습니다. 정상 Peer 상관/±2일 시차/분율→%·모델 점수 표시, 모델/Peer UI 복구, sentiment INSERT오류 중 계산 점수 보존은 해당 사례에서 PASS입니다.
- Peer UI복구에서는 사용자가 다시 분석하며 from/to 초가 바뀌어 새 Redis키를 사용했습니다. 오류Peer0/정상Peer2 키가 함께 남고 source2회입니다. 캐시 수동 삭제는 없지만, 앞선 동일 요청의 오류캐시12시간 잔류FAIL이 해결된 것은 아닙니다.
- 실제FastAPI42개/역방향TCP14개, 모델31결과/공개33점수, snapshot/소유자상세14개/다른계정404×14/USE14개/환불0을 대조했습니다. 브라우저 저장상세GET3개, API/pageerror 이상0, 외부글꼴origin34회 차단입니다. 모델 사전 로딩91.834초·warm3기사0.864~0.990초이며 운영 성능 보장이 아닙니다.
- PDF2개6페이지는 A4/파싱/수리불필요/비어있지않음 검사PASS입니다. 6페이지PNG를 직접 확인했으며 live 정량은 남고 저장PDF 정량은 없습니다. 기존 live PDF스타일/페이지경계 문제도 관측했습니다. PDF파일 구조PASS를 내용/스타일 전체PASS로 확대하거나 별도 사례로 합산하지 않습니다.
- 초기 영문종목 검색/Peer선정 기대값·거시필드명 오타 보정을 기록했습니다. 별도 보정실행 `run-1vpjzrq7`은 exit143/SIGTERM으로 중단돼 최종JUnit/SQL정리 미확인·집계 제외입니다. 남은 합성 인증manifest를 삭제하고 소유datadir를 보존했습니다. 정상 종료·SQL0의 증거는 최종run에서 확인했습니다.
- 최종입력612개/모델7파일·frontend소스/복사본187개/bundle11개, tests밖924개 hash를 확인했습니다. 최종MySQL/Redis exit0·업무9/원천12/reference/hidden/trigger/QA종목0·Redis0, proxy/control/worker/socket 종료, Spring/FastAPI 종료, 브라우저12개/profile/manifest 정리를 확인했습니다. 실제모델 프로세스 외부/dotenv/subprocess 시도0입니다.

최종 증거는 `tests/.runtime/backend-isolated/runs/run-kd_uymvv/feature2-pipeline-browser-evidence-audit.json`, 브라우저/PDF는 `frontend/tests/.runtime/browser-feature2-pipeline/run-kd_uymvv`에 있습니다. 문서68개 대상2개 검사와 실행이력/사례 대응 감사도 별도 기록합니다. 다음은 [인계 문서의 Feature1 실행 시작점](FALLBACK_QA_HANDOFF.md), 실제 외부 제공자/나머지 Swagger·프로세스/동시성/모델 품질 범위입니다. 전체 goal은 active·미완료입니다.

## Feature1 실제 SQL·snapshot·Redis·계산/저장 결과

앞선 [Feature1 실제 SQL·snapshot·Redis·계산/저장](isolated-feature1-pipeline.md)은 **23개13 PASS/10 FAIL**(`run-y4nj39wc`)입니다. 원천 오류의 전체 전파, 기존 캔들 미보존, 빈 snapshot의 재무 fallback 차단, stale/저장 경고 누락, 날짜 경계와 기존 저장 매핑 실패를 재현했습니다. 실제 FastAPI28회·응답31개·snapshot/상세26개·USE31/REFUND3·후속 정상8요청을 대조했습니다. 소유 서비스는 정리·종료했고 초기3회는 합산하지 않습니다.

- 정상160가격·EMA480개/초기null·마지막BB/stochastic·TTM 매출15000/순이익1500·PER120/PBR15/PSR12/ROE12.5를 제품 import 없는 기대값과 대조했습니다. 네 투자 수준·두 무키 설명·실제503/잘못된JSON·실제NULL 종가422·빈 재무·저장trigger·snapshot+계산 복수 장애의 선택 흐름은 통과했습니다.
- 가격/재무 SQL 오류2개와 캔들 provider 오류는 독립 자료/기존160가격까지 data=null로 만듭니다. 세 경우 환불은 동일reference의−1/+1로 정상이며 계산API0/report0입니다. `F1-ASSEMBLY-ISOLATION-001`, `F1-CANDLE-FALLBACK-001`로 기록했습니다.
- `F1-SNAPSHOT-FALLBACK-001`: 실제snapshot/발행주식 SQL 오류2개에서 재무6행을 Python에 보내지만null필드의snapshot 객체가 재계산을 막아 가능한ROE12.5도null입니다. 기술지표/부분 경고/저장은 유지됩니다. `F1-SNAPSHOT-WARNING-001`: 실제Redis stale 시세180은 유지되지만 단계의PRICE_STALE_USED가public/SQL에서 사라집니다.
- `F1-REPORT-WARNING-001`: 빈 캔들의CHART_DATA_UNAVAILABLE을meta에만 추가해저장 상세에서는 안내가 빠집니다. `F1-DATE-WIRE-001`: public의자정 아닌종료일이내부Python date에서422입니다. `REPORT-F1-WINDOW-001`: API가받은두25자시간대 날짜의기간53자가SQL VARCHAR(50)를 초과해정상 계산리포트 저장에실패합니다. 기존REPORT-MAPPING-001도실제SQL/상세에서 재현했습니다.
- 정상Python metrics23개와public, 실제wire 가격4320행분/재무162행분과seed를 대조했습니다. 공개31응답/SQLsnapshot=소유자상세26개/타사용자404×26/Redis metrics28개/USE31·REFUND3·복구8요청을 확인했습니다. 추가 수학/복구 단언은 별도 사례로 합산하지 않습니다.
- JUnit50.075초·Gradle2분59초이며 초기3회는제어HTTP·날짜 입력경계·HTTP codec보정 이력으로 상세에보존합니다. 최종609입력과tests밖924파일의해시가보존됩니다. 네실행 모두업무9/원천4/QA종목/숨긴테이블/trigger0·Redis0 및소유MySQL/Redis exit0/FastAPI SIGTERM−15 종료를 확인했습니다. 외부연결/dotenv/subprocess 시도0입니다.

최종증거는`tests/.runtime/backend-isolated/runs/run-y4nj39wc/feature1-pipeline-evidence-audit.json`이며 [23개 전체 방법·판정](isolated-feature1-pipeline.md)을참조합니다. 이 서버 단계 당시 누적은 **26묶음558개429 PASS/129 FAIL**, goal은active·전체미완료입니다. 다음은실제Feature1 브라우저/저장/PDF와전체미완료범위입니다.

이 단계의 증거 감사와23사례/26묶음 집계·네 실행 정리 감사가PASS입니다. 문서69개 대상 링크/설정 비밀값 검사2개도PASS이며 상세 방법과 로그 위치는위문서에기록했습니다.

## Feature1 실제 서버·브라우저/저장/PDF 결과

[17개 방법·장애 주입·판정·증거·초기 실행 이력](../frontend/tests/browser-feature1-pipeline.md)을 추가했습니다. 최종 `run-k_l2x_cx`는 **17개5 PASS/12 FAIL**, errors/skipped0, JUnit194.705초·Gradle5분8초입니다. 이 단계까지 누적은 **27묶음575개434 PASS/141 FAIL**입니다. 전체 QA는 미완료이며 사례 수를 완료율이나 고유 결함 수로 바꾸지 않습니다.

- 실제 Chromium/React→로그인·검색/캔들→Feature1 실제 가격/재무/발행주식 SQL·snapshot/TTM·Redis·FastAPI→차감/report SQL→화면/저장/PDF를 연결했습니다. API 응답을 재작성하지 않았습니다. 검색 매핑/랭킹·외부 시세/캔들 제공자는 합성 경계입니다.
- 계산503의 가격/재무·제한 안내 보존, 계산/snapshot/stale 장애 제거 뒤 동일 body 재분석3개의 정상 수치/잔액 복구, 보조 Q재무 GET500 중 성공 분석 보존을 확인했습니다. 수동 캐시 삭제는 없고 같은 브라우저/서비스입니다. snapshot/stale 최초 안내·정량 실패는 복구 성공과 별도로 FAIL을 유지합니다.
- `FRONT-F1-SAVED-METRICS-001`: SQL/상세에는 정량이 남아도 저장 화면/PDF에는 없습니다. `FRONT-F1-SOURCE-WARNING-001`: 실제 무키 설명 경고 안내 누락. `FRONT-F1-CORE-WARNING-001`: 실제 원천 오류의 환불은 정상이나 화면에 실패 안내가 없습니다. 분석 전 가격 요약/빈 투자지표 카드가 보여 정상 결과로 오인할 여지가 있습니다. `FRONT-F1-META-WARNING-001`: 빈 캔들/저장 실패의 meta 경고가 live 화면에 나타나지 않습니다.
- 기존 원천 독립성/저장 캔들 fallback/빈 snapshot의 재무 계산 차단/stale 출처 소실/저장 경고 소실/날짜 및 보고서 매핑 FAIL을 브라우저까지 재현했습니다. 두 날짜를 지운 실제 UI가 시간대 포함 날짜를 보내 Python422와 SQL기간53자→50자 초과를 유발했습니다.
- 별도 `FRONT-F1-BB-POSITION-001`: 최신종가179.5와 볼린저 상·하단으로 계산한 밴드 내 위치91.1877%가 화면50.0%로 표시됩니다. 컴포넌트가 종가 대신 중심선을 사용합니다. 공식 정의/독립식/API/DOM을 대조했고 정상 수치가 있는 보고서12개에서 재현했습니다. 추가 검토이며 JUnit 사례 수에는 더하지 않습니다.
- 실제 FastAPI17개(정상13·5033·4221), 공개응답20개/SQLsnapshot=소유자상세15개/타사용자404×15/브라우저 상세GET2개/USE20·REFUND3을 대조했습니다. Pythonmetrics13개·wire가격2560행분/재무102행분·최종Redis14사례도 확인했습니다. 예상 밖 API/pageerror0, 외부 글꼴origin42회 차단입니다.
- PDF2개3페이지의 파싱/A4/비어있지않음은 PASS입니다. 전 페이지 PNG를 직접 검토해 live 스타일 소실·이자보상배율의 페이지 경계 잘림과 저장 정량 누락을 기록했습니다. 파일 구조 PASS는 내용/시각 품질 PASS가 아닙니다.
- 초기 `run-8pyzrz5_` 1FAIL은 EMA60 소수 표시 반올림 기대값 오류입니다. 첫 전체 `run-a3h0jvtk` 2PASS/15FAIL 중3개는 실제 계산 제한 안내를 정규식이 놓친 오탐입니다. 테스트 기대값만 보정하고 최종17개를 실행했습니다. 초기 두 실행은 합산하지 않습니다.
- 입력611개 실행 중 변경0·frontend소스/복사본187개/bundle11개, 소유 MySQL 업무9/원천4테이블·QA종목·hidden/trigger0·Redis0과 두 서버exit0·FastAPI SIGTERM−15/active0·외부/dotenv/subprocess시도0을 확인했습니다. 브라우저17개/프록시/임시 프로필·합성 인증manifest를 정리했습니다.

서버 증거·강화된 증거 감사는 `tests/.runtime/backend-isolated/runs/run-k_l2x_cx`, 화면/PDF/볼린저 검토는 `frontend/tests/.runtime/browser-feature1-pipeline/run-k_l2x_cx`입니다. 세 실행 정리·tests밖924개 보존·문서70개 검사2개·17사례/27묶음 대응은 `history-and-documentation-audit.json`에 기록합니다. 다음은 [인계 문서의 다음 실행 시작점](FALLBACK_QA_HANDOFF.md)을 따릅니다. 제품은 미수정이며 goal은 active입니다.

## Feature1 실제 Redis 장애·HTTP 취소/재시도·SQL 결과

[17개 방법·장애 주입·정책 경계·전송/원장 대조](isolated-feature1-resilience.md)를 추가했습니다. 최종 `run-rqzhknkk`는 **17개16 PASS/1 FAIL**, errors/skipped0, JUnit127.216초·Gradle4분17초입니다. 이 단계까지 누적은 **28묶음592개450 PASS/142 FAIL**입니다. 정상 기준 `run-es3dgtup` 1PASS는 중복 합산하지 않습니다.

- 실제 F1 가격/재무/발행주식 SQL·snapshot/시세/비율·Redis/Lettuce·FastAPI 계산·JWT/차감/리포트/상세에 소유 TCP partition/blackhole/재시작을 연결했습니다. 외부 시세/캔들 경계는 합성이고 제품은 미수정입니다.
- Redis 단절/바이트 차단 중 실제 분석은12.541/12.642초 뒤 정상 결과/SQL을 유지했습니다. Redis+계산503은12.623초 뒤 가격/재무 대체 결과를 보존했고, 실제 계산 후 metrics SET 실패만으로 SQL 리포트는 소실되지 않았습니다. 같은 client 재접속 확인은52회 관측 중 최대6.698초/4시도였습니다. 운영 SLA가 아닙니다.
- 유일한 FAIL은 기존 `F1-ASSEMBLY-ISOLATION-001`: Redis+실제 가격 SQL 오류에서 정상 재무까지 public data=null입니다. 동일 reference의 USE−1/REFUND+1, report0·실제 계산0이며 복구 후 같은 body 정상 분석은 통과했습니다.
- 캔들/시세/API/metrics 저장의 네 지점 실제 HTTP 취소와 클라이언트3초 timeout의 전파·후속 복구를 확인했습니다. 네 취소/timeout/늦은 옛 계산 취소까지 총6요청은 차감−1이 남고 해당 리포트가 없었습니다. 취소 환불 정책이 명시되지 않아 과금 적합성 판정과 분리했습니다.
- 실제 옛179.5 계산을 보류→새 SQL가격279.5 분석/캐시/저장 완료→옛 HTTP 취소/해제 후에도 새 캐시/SQL이 유지됐고 다음 분석도279.5였습니다. 동시 동일 body2개는 별도 reference/차감/report2개로 관측됐으며 멱등성 정책 완료 판정은 아닙니다.
- 실제 API10초 보류 중 요청 미완료/차감/report0을 기록하고 해제 후 정상 계산/SQL을 확인했습니다. 제품 분석 WebClient/호출부에 명시적 요청 timeout이 없지만 이 관측만으로 무한 대기나 SLA 위반을 확정하지 않습니다.
- 공개29응답/SQLsnapshot=소유자상세28개/타사용자404×28, USE35·REFUND1/원장37상태를 대조했습니다. 실제 FastAPI32회 중 Python200은29회(그중27개 public과 일치·2개 계산 뒤 취소),5031회, 실제 disconnect/응답없음2회입니다. wire가격5120행분/재무192행분·캐시48관측도 대조했습니다.
- 입력613개 실행 중 변경0, 두 실행의 업무9/원천4테이블·QA종목·hidden/trigger0/Redis0, MySQL·Redis exit0/FastAPI SIGTERM−15·active/외부/dotenv/subprocess시도0, 프록시/제어/worker 종료·active socket0을 확인했습니다. 실제 Redis 재시작1회도 새 run_id·표식 전DB0·이전exit0입니다.

증거는 `tests/.runtime/backend-isolated/runs/run-rqzhknkk`의 summary/XML/`feature1-resilience-evidence-audit.json`과 전송 기록입니다. 문서71개 검사2개, 두 실행 정리·tests밖924개 보존·17사례/28묶음 대응은 `history-and-documentation-audit.json`에 남깁니다. 다음은 [Feature3 원천 장애·취소/환불·재시도와 전체 남은 범위](FALLBACK_QA_HANDOFF.md)입니다. goal은 active·전체 QA 미완료입니다.


## Feature3 실제 원천 SQL·대체 조회·계산/과금 결과

[17개 방법·원천 주입·초기 오류·실행별 결과](isolated-feature3-source-sql.md)를 누적했습니다. 최종 `run-sphw2mwo`는 **17개12 PASS/5 FAIL**, errors/skipped0, JUnit120.621초/Gradle4분9초입니다. 초기 setup 오류 `run-6glg3wdp`, 정상1PASS `run-xvzgj6z6`, 초기 전체17개11PASS/6FAIL `run-yfzoss8j`는 중복 합산하지 않습니다. 최초 전체의2FAIL은 테스트 날짜 복원 PK 충돌이어서 복원 순서를 수정했고, 원천 freshness 실패 단언을 보강했습니다.

- 실제 Price/Candle/YahooProvider·Benchmark·RiskFree/BondYield service와 가격423행/벤치마크282행/금리 SQL을 실제 Python 계산·과금·report에 연결했습니다. 외부 Yahoo/KIS/BOK client는 합성이며 실제 외부 제공자 성공을 의미하지 않습니다.
- 기존 F3-CREDIT-001을 실제 가격/벤치마크 SQL/두 SQL 동시 오류로3사례 재현했습니다. HTTP500/FastAPI0/report0/USE−1/환불0, 원천 복원 후 같은 입력은 성공하지만 잔액3입니다.
- 신규 F3-RISKFREE-EMPTY-001은 금리 SQL0+BOK 빈 목록에서 HTTP200/본문0/FastAPI0/report0/USE−1입니다. 예외의 DEFAULT_ZERO 대체와 구분합니다. 신규 F3-PRICE-FRESHNESS-001은9월21일까지의 가격 fallback인데 freshness9월28일·경고 없음입니다. 오래된 가격 보존과 후속 정상 복구 자체는 통과했습니다.
- 원천 일부/전체 누락의 부분 계산과 경고, Yahoo 오류/empty→RAW fallback, 벤치마크 stale/없음, 금리 SQL 오류/BOK 예외→0%, report INSERT 실패의 계산/차감 보존을 확인했습니다. 같은 입력 후속15개 중 장애 복구14개/정상 반복1개입니다.
- 증거 감사 PASS: 응답32개/실제FastAPI28개, snapshot=소유자 상세27개·다른 사용자404, USE32/REFUND0, 실제source168개, 가격10,857/지수7,473 전송 행, 독립 공분산25응답을 대조했습니다.610개 실행 입력 hash 보존·업무/원천/fixture/trigger0·Redis0·MySQL/Redis/FastAPI 종료를 확인했습니다.

최종 증거는 `tests/.runtime/backend-isolated/runs/run-sphw2mwo`의 summary/XML/`feature3-source-evidence-audit.json`입니다. 문서72개 검사2개·초기 포함4회 정리·누적29묶음/609개·tests 밖924개 보존은 `history-and-documentation-audit.json`에 기록합니다. 다음은 Feature3 overlay/Redis·HTTP 취소/timeout·환불/재시도·동시 요청이며 실제 외부 제공자·남은 Swagger/브라우저/배치/모델 품질 범위도 유지합니다. 전체 QA는 미완료·goal active입니다.


## 2026-09-29 Feature3 overlay·Redis·HTTP 취소/재시도 결과

[23개 주입 방법·실제/합성 경계·사례별 결과](isolated-feature3-resilience.md)를 누적했습니다. 최종 `run-p4h4ipqi`는 **23개19 PASS/4 FAIL**, errors/skipped0, JUnit182.159초/Gradle5분1초입니다. compile import 오류/실행0 `run-uffo2ne7`과 정상 선택1PASS `run-ei9ik5t4`는 합산하지 않습니다.

- 실제 가격/벤치마크/금리 SQL·core 수학·CreditService/ReportService와 실제 Feature3OverlayService/Redis를 연결했습니다. Feature1 leaf metrics는 합성이며 실제 전체 Feature1/외부 제공자 결합으로 확대하지 않습니다.
- 신규 F3-OVERLAY-WARNING-001: 한 종목/모든 종목/Redis+모든 종목의 선택 overlay 실패3사례에서 신호4/0/0·core/SQL은 보존되지만 fallback 카드의 안내가 신호 변환에서 소실됩니다. 후속 복구6신호는 통과했습니다. 기존 F3-CREDIT-001은 Redis+가격 SQL 실패에서 예약3/환불0으로 재현했습니다.
- 실제503/깨진JSON/422 및 계산 뒤 후처리 경계 오류의 예약3 환불과 warm 비용1 재시도, Redis 단절/blackhole/재시작·깨진 캐시·계산 후 단절의 core/신호/저장 보존을 확인했습니다. cold단절6.477초/blackhole6.468초, 동일 client 재접속 확인 최대2.502초입니다.
- HTTP 취소4지점·실제3초 timeout·과거 overlay 취소 뒤 새ROE35 cache/SQL 유지가 통과했습니다. 취소된6요청의 차감 잔존, 동일body2개의 별도 비용3/리포트, 비용 확정 뒤 cache 삭제 시 예약1 유지 등은 정책 공백/비용 시점 관측과 구분합니다.
- 증거 감사: 실제FastAPI40개/수학35개/공개 대응33개·응답38개/snapshot·상세33개/USE44·REFUND4/원장68상태, 가격16,920행/지수11,280행·overlay신호190개·캐시214값을 대조했습니다. 후속 복구16개, 전송 disconnect2개와 계산 후 취소/후처리 실패2개를 구분했습니다.
- 소유 업무/원천/fixture/trigger0·Redis0·FastAPI active0/외부연결·dotenv·subprocess0, MySQL/Redis/FastAPI·프록시/control/worker 종료 및615개 입력 hash 보존을 확인했습니다.

최종 증거는 `tests/.runtime/backend-isolated/runs/run-p4h4ipqi`의 summary/XML/`feature3-resilience-evidence-audit.json`입니다. 문서73개 검사2개·초기 포함3회 정리·누적30묶음/632개·tests 밖924개 보존은 `history-and-documentation-audit.json`에 기록합니다. 다음은 Feature3 industry/correlation/news overlay와 실제 원천 브라우저/저장/PDF이며 실제 외부 제공자·남은 사용자 흐름·프로세스/다중backend/모델 품질/접근성 범위도 유지합니다. 전체 QA 미완료·goal active입니다.
