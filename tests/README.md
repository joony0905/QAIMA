# 공통 문서 검증

## 세션 인계와 최우선 QA 기준

최신 [Feature3 overlay·Redis·HTTP 취소/재시도](isolated-feature3-resilience.md)은 **23개19 PASS/4 FAIL**(`run-p4h4ipqi`)입니다. 선택 overlay 실패 경고 누락3사례와 Redis+가격 SQL 장애의 비용3 환불 누락1사례를 재현했습니다. 실제 FastAPI40회·응답38개·snapshot/상세33개·USE44/REFUND4와 취소6개·후속 복구16개를 대조했고 소유 서비스를 정리·종료했습니다. 취소 차감 잔존/동일 body 별도 차감은 명시 정책 공백 관측이며, 초기2회는 합산하지 않습니다.

직전 [Feature3 실제 원천 SQL·대체 조회·계산/과금](isolated-feature3-source-sql.md)은 **17개12 PASS/5 FAIL**(`run-sphw2mwo`)입니다. 가격/벤치마크 실제 SQL 오류의 환불 누락3사례, 금리 원천 모두 empty에서 빈 HTTP200/차감 잔존1사례, 오래된 가격의 freshness 오표시1사례를 재현했습니다. 실제 FastAPI28개·응답32개·snapshot/상세27개·USE32/REFUND0·장애 복구14개를 대조했고 소유 서비스를 정리·종료했습니다. 초기3회는 합산하지 않습니다.

직전 [Feature1 실제 Redis·HTTP 취소/재시도·SQL](isolated-feature1-resilience.md)은 **17개16 PASS/1 FAIL**(`run-rqzhknkk`)입니다. Redis 단절/재시작·계산 후 캐시 저장 실패의 결과 보존, 취소/timeout 전파·동일 입력 복구·취소된 이전 결과의 캐시/SQL 미덮어쓰기를 확인했습니다. Redis+가격 SQL 복합 장애의 정상 재무 소실은 기존 FAIL입니다. 실제 FastAPI32회·응답29개·snapshot/상세28개·USE35/REFUND1을 대조했습니다. 미완료 취소6개의 차감 잔존과 동일 body2개의 별도 차감은 정책 적합성 PASS와 구분합니다. 소유 서비스를 정리·종료했습니다.

직전 [Feature1 실제 서버·브라우저/저장/PDF](../frontend/tests/browser-feature1-pipeline.md)은 **17개5 PASS/12 FAIL**(`run-k_l2x_cx`)입니다. 계산 장애의 부분 결과/안내·동일 입력 복구와 보조 재무 조회 오류 중 분석 보존은 통과했습니다. 저장 정량·설명/핵심/저장 경고 누락과 기존 원천/snapshot/stale/날짜 실패를 실제 화면까지 재현했습니다. 실제 FastAPI17회·응답20개·snapshot/상세15개·USE20/REFUND3·동일 입력 복구3개를 대조했고 소유 서비스를 정리·종료했습니다. 별도 검토의 볼린저 위치91.2%→50.0% 및 PDF 스타일 실패는 JUnit에 중복 합산하지 않습니다.

직전 [Feature1 실제 SQL·snapshot·Redis·계산/저장](isolated-feature1-pipeline.md)은 **23개13 PASS/10 FAIL**(`run-y4nj39wc`)입니다. 원천 오류의 전체 전파, 기존 캔들 미보존, 빈 snapshot의 재무 fallback 차단, stale/저장 경고 누락, 날짜 경계와 기존 저장 매핑 실패를 재현했습니다. 실제 FastAPI28회·응답31개·snapshot/상세26개·USE31/REFUND3·후속 정상8요청을 대조했습니다. 소유 서비스는 정리·종료했고 초기3회는 합산하지 않습니다.

2026-09-28 사용자 지시: **전체 fallback 흐름을 철저히 검증하고, 여러 데이터를 조립하는 Feature2의 장애 대응을 특히 확인합니다.** 다른 세션은 [fallback 검증 기준·Feature2 점검표](FALLBACK_QA_HANDOFF.md)와 [최신 누적/남은 범위](QA_PROGRESS_2026-09-28.md)를 먼저 읽으세요. 단일/복수 장애, 정상 결과 보존, warning의 화면 전달, 차감/환불·저장, 복구까지 확인해야 합니다. 개별 mock 테스트나 HTTP200만으로 전체 흐름이 정상이라고 판단하지 않습니다. 수정 범위는 tests 내부, DB/Redis는 새 외부 격리 인스턴스입니다.

## 누적 기록과 검사 이력

직전 [Feature2 실제 SQL·Peer·로컬 모델의 브라우저/저장/PDF](../frontend/tests/browser-feature2-pipeline.md)은 **12개4 PASS/8 FAIL**(`run-kd_uymvv`)입니다. 실제 원천 경고의 화면 누락, 모델 점수 없음에서 보고서 기사 소실, 산업지수 장애의 정상 RAW Peer 차트 소실과 기존 저장 정량/거시/NULL 수급 실패를 확인했습니다. 실제 FastAPI42회·모델31결과/공개33점수·snapshot/상세14개/USE14개·PDF2개6페이지를 대조했습니다. 최종 소유 서비스는 정리·종료했으며 초기 보정/중단 실행은 합산하지 않습니다.

최신 누적은 **30묶음632개481 PASS/151 FAIL**입니다. [Feature2 실제 Peer·로컬 뉴스 모델·SQL 결합](isolated-feature2-pipeline.md)은 **14개8 PASS/6 FAIL**(`run-5br21593`)입니다. 신규 Peer 오류 캐시 복구 실패5사례와 기존 최신일 제외1사례를 재현했습니다. 실제 역방향SQL/독립 상관·시차,모델 누락/복구·RAW fallback·SQL 저장 장애의 부분 결과 보존을 확인했습니다. 실제FastAPI61개/역방향TCP18개·모델40결과/공개69점수·snapshot/상세26개/USE26개를 대조했고 소유 서비스는 정리·종료했습니다.

2026-09-28 QA 재개 증분은 [진행 기록](QA_PROGRESS_2026-09-28.md), [Feature2 Chrome 검증](../frontend/tests/browser-feature2-flow.md), [격리 MySQL HTTP/JWT/JPA 검증](isolated-http-jpa.md), [격리 Redis 캐시 검증](isolated-redis-cache.md), [뉴스/지표 Redis 검증](isolated-news-overlay-redis.md), [분석 과금·환불·리포트 검증](isolated-analysis-billing.md), [배치 실제 SQL 검증](../batch/tests/isolated-sql.md), [매핑·초기화·해외 SQL 검증](../batch/tests/isolated-sql-tools.md), [Spring 배치·예약 실행 검증](isolated-spring-batch.md), [SEC 13F 실제 SQL 검증](isolated-sec13f.md), [발행주식·마스터 실제 JPA 검증](isolated-shares-master.md), [회원가입·이메일·재설정 HTTP/JPA 검증](isolated-auth-email.md), [OAuth·소셜 프로필 HTTP/JPA 검증](isolated-oauth.md), [공매도 실제 SQL·카드·CSV 검증](isolated-short-selling.md)을 참조합니다. 최신 지시에 따라 이번 문서·코드 수정은 tests 폴더 내부로 제한했습니다. 이 README와 새 문서30개를 문서 검사에 추가해 대상은71개입니다. 아래40개 설명은 2026-09-25 검사 이력입니다.

격리 MySQL HTTP/JWT/JPA는12개11 PASS/1 FAIL입니다. 같은 refresh token의12개 동시 요청이 모두 성공하나 후속토큰1개만 저장되고 다른 성공 쿠키의 다음 refresh가401이 되는 문제를 재현했습니다. 상세 문서에 명령·입력·SQL 정리·검증 경계를 기록했으며 제품은 미수정입니다.

격리 Redis/Lettuce는18개15 PASS/3 FAIL입니다.10·20·60·120초 자연 만료·실제 ACL 오류 fallback·24개 요청의 조회 공유를 확인했습니다. 기존 빈 가격 목록/산업지수 경고 소실2건과 신규 실시간 null 가격 처리1건을 실패로 보존합니다. Redis 서버/JVM 시각 차이와 최초 측정 단언 보완도 상세 문서에 기록했습니다. 두 실행 모두 캐시를 비우고 Redis를 종료했습니다.

후속 뉴스/metrics Redis는21개17 PASS/4 FAIL입니다. 기존 뉴스 캐시 오판3메서드 외에 늦은 A 점수가 B 입력 hash와 함께 캐시·observation 저장 요청에 기록되는 새 동시성 결함을 재현했습니다. 뉴스2초 block timeout/복구와 Feature3 비용 preview/estimate도 확인했으며 실제 SQL 원장·모델은 호출하지 않았습니다.

분석 HTTP·MySQL/Redis 결합은21개15 PASS/6 FAIL입니다. 실제 차감/환불/리포트 SQL·캐시 정책별 비용·동시 요청을 검사했습니다. 기존 Feature3 준비 오류 환불 누락4개와 Feature1 리포트 매핑2개가 실패했고,62자 회사명으로 실제 stock_code 길이 초과 저장 실패까지 확인했습니다. 계산/가격/FastAPI 경계는 fixture이며 두 실행 모두 MySQL·Redis 데이터 정리와 정상 종료를 확인했습니다.

배치 실제 SQL은31개27 PASS/4 FAIL입니다. 기존 가격/산업지수 rollback 누락으로 SQL 오류 뒤 transaction/잠금이 남아 다른 연결의 UPDATE가1205로 실패하는 것을 확인했습니다. 여섯 경로의 재실행·실제 CSV 두 importer·장애 후 보정 재실행을 기록했으며 소유 MySQL은 정리·정상 종료했습니다.

후속 매핑·초기화·해외 SQL은22개14 PASS/8 FAIL입니다. 새 sector/industry 동시 생성 복구 실패와 기존 주봉·월봉의 실제 일봉 덮어쓰기를 확인했습니다. 수동 매핑 dry-run 및 main 오류 종료의 rollback 대조군은 통과했습니다. 두 실행의 실패와 데이터 정리·서버 종료를 기록했습니다.

Spring 배치·실제 JPA/예약 실행은22개19 PASS/3 FAIL입니다. 새 종목 수급 날짜 상한 오류로 수집 전 중단되며, 산업지수는 KST 당일 자료를 UTC 조회 날짜로 비교해 다시 수집합니다. JDBC Seoul/Hibernate 기본 조건에서도 재현했습니다. 실제 예약 콜백→SQL 저장·휴장일·재시도·실행 잠금 복구는 통과했고 두 소유 MySQL은 정리·종료했습니다.

SEC 13F 실제 ZIP/JPA/native SQL은24개21 PASS/3 FAIL입니다. 새 동시 CUSIP 매핑 경쟁으로 같은 식별자에 active 종목2개가 저장됐고, 기존 헤더/날짜 결함도 실제 저장소에서 재현했습니다. 최신 공시·정정 제외·기관 합산·보유비율·1000행 부분 commit 후 재실행은 통과했습니다. 테스트 컴파일/spy 보정 이력과 네 DB 정리·종료를 상세 기록에 남겼습니다.

발행주식·마스터 실제 JPA는28개26 PASS/2 FAIL입니다. SEC의 늦은 종목 저장이 앞서 갱신한 회사명을 덮는 새 동시성 결함과, 범위 초과 주식수가 실제1주로 저장되는 기존 결함을 확인했습니다. 마스터 전체 rollback·OpenDART 매핑 rollback·발행주식 부분 commit/복구·두 실제 cron 콜백의 SQL 저장은 통과했습니다.

회원가입·이메일 인증·비밀번호 재설정 HTTP/JPA는26개25 PASS/1 FAIL입니다. 새 AUTH-RESET-RACE-001은 같은 재설정 토큰의 두 동시 요청이 모두200/성공 감사2행을 남기지만 최종 암호는 한 요청 값만 저장됩니다. 실제 이메일 확인/가입의 잠금·SQL/메일 오류 rollback·해시/만료/재발급은 통과했습니다. 외부 메일은 발송하지 않았고 소유 MySQL은 정리·정상 종료했습니다.

OAuth·소셜 프로필 HTTP/JPA는25개23 PASS/2 FAIL입니다. 새 AUTH-SOCIAL-INACTIVE-001은 추가정보가 비어 있는 비활성 계정이 새 OAuth 세션을 받고 프로필 제출로 active가 되는 경로와, 기존 JWT로 재활성화되는 경로입니다. 세 합성 제공자의 실제 token/userinfo HTTP·state 검증·SQL rollback·계정 중복 제약은 통과했습니다. 초기 구성/쿠키 단언 보정 이력과 세 DB 정리·최종 제공자 종료를 기록했습니다.

공매도 실제 SQL/CSV/공개 카드/예약 실행은31개26 PASS/5 FAIL입니다. FINRA/KRX 비율 단위 불일치, 시계열 조회 오류 경고 누락, CSV의 다른 시장 종목 매칭·따옴표 숫자 누락·필수 헤더 누락 정상 반환을 확인했습니다. 두 제공자와 CSV의1000행 부분 commit 후 재실행, 정밀도·권한·실제 cron→SQL은 통과했습니다. 초기 테스트 구성2개 보정과 두 DB 정리/종료를 기록했습니다.

후속 [Feature2 차트·nullable·PDF](../frontend/tests/browser-feature2-charts.md)는20개13 PASS/7 FAIL입니다. 금리 시계열 날짜 정렬·공매도 null의 화면 소실·비율 표시·PDF 스타일 실패를 기록했고 별도 PDF 마지막 페이지 검사도 실패했습니다. 

후속 [Feature1 재무·차트·검색 경합·PDF](../frontend/tests/browser-feature1-charts.md)는25개18 PASS/7 FAIL입니다. 검색/분석 응답 경합·실패 후 이전 재무 유지·KST 표시·PDF 스타일을 기록했습니다. 

후속 [Feature3 포트폴리오 차트·입력 경합](../frontend/tests/browser-feature3-charts.md)는26개23 PASS/3 FAIL입니다. SCL/SML/프론티어의 독립 좌표·분산 기반 효용·4수준/영어/모바일을 확인했고 리포트 메타데이터2경로와 비용 미리보기 대상 불일치가 실패했습니다. 

후속 [Feature3 실제 HTTP·FastAPI·과금/리포트 SQL](isolated-analysis-pipeline.md)은18개18 PASS입니다. 독립 공분산/위험기여도·네 투자 수준·가격 부족 fallback·Redis 재무 overlay·14개 SQL snapshot·실제 계산 오류 환불을 확인했습니다. 초기 fixture/배열 순서 가정 오류와 보정 이력을 남겼고, 소유 MySQL·Redis·FastAPI 정리/종료를 확인했습니다. 가격 원천과 브라우저 경계는 별도입니다. 

후속 [Feature3 브라우저·실제 서버/SQL·저장 PDF](../frontend/tests/browser-server-pipeline.md)는 **8개7 PASS/1 FAIL**입니다. 실제 인증·계산·차감/환불·리포트 조회/PDF·소유권·로그아웃을 연결했고 저장 실패 안내의 화면 누락을 재현했습니다. 환경/테스트 보정 이력과 소유 서비스 정리를 기록했습니다. 

후속 [Redis TCP 단절·재시작·복구](isolated-redis-lifecycle.md)는 **22개22 PASS**입니다. 소유 TCP 프록시의 연결 종료/거절/바이트 차단과 실제7회 Redis 재시작·동일 client의 값/캐시 복구를 확인했습니다. 뉴스 fallback28.013초의 누적 대기는 Feature2 실제 조립에서 추가 확인합니다. 해당 단계까지19묶음은418개355 PASS/63 FAIL이었습니다.

후속 [Feature2 실제 조립·fallback·HTTP/SQL](isolated-feature2-fallback.md)은 **39개17 PASS/22 FAIL**입니다. 독립 데이터의 연쇄 누락·빈 완료 경고 누락·정량 결과0의 환불 누락을 재현했습니다. 실제 하위 서비스4사례와 경계 mock을 구분했고, 후속36개 정상 요청/71개 snapshot 대조·소유 DB 정리를 확인했습니다. 해당 단계까지20묶음은457개372 PASS/85 FAIL이었습니다.

후속 [Feature2 실제 브라우저·조립/서버/SQL](../frontend/tests/browser-feature2-server.md)은 **13개7 PASS/6 FAIL**입니다. 뉴스/Peer 부분 실패·명시 재분석 복구·잔액 부족 안내는 통과했습니다. 보조 시계열500으로 성공 분석이 숨겨지는 기존 결함을 실제 차감/저장과 연결했고, 저장/핵심 오류 안내·저장 정량/text 설명 누락을 기록했습니다. SQL snapshot12개/상세GET5개·정상 요약 PDF와 소유 MySQL/브라우저 정리를 확인했습니다. 해당 단계까지21묶음은470개379 PASS/91 FAIL이었습니다.

후속 [Feature2 실제 Redis·조립/HTTP 취소/SQL](isolated-feature2-resilience.md)은 **15개12 PASS/3 FAIL**입니다. 실제 Redis 단절/바이트 차단 중 조립 약40.6초, 복원 후 같은 client의 재접속 확인 최대27.340초를 관측했습니다. 취소된 옛 가격/산업 source의 새 캐시 덮어쓰기2경로와 복구 후 뉴스 오류 경고 잔류를 재현했습니다. snapshot/상세28개·USE 원장35개를 대조했고, 취소7개의 차감 잔존은 명시 환불 정책이 없어 별도 관측입니다. 소유 MySQL/Redis/프록시는 정리·정상 종료했습니다. 해당 단계까지22묶음은 **485개391 PASS/94 FAIL**입니다. 실패 사례 수는 고유 결함 수/전체 완료율과 다릅니다. 시장 원천 SQL/3기사/실제 설명의 후속 결과는 아래에 이어집니다.

후속 [Feature2 실제 시장 SQL·Redis·FastAPI 설명](isolated-feature2-source-sql.md)은 **24개13 PASS/11 FAIL**(`run-udwy3eb5`)입니다. 실제 JPA/SQL11개 원천 장애와 복수 장애·NULL/빈 행·뉴스3기사·실제 무키 설명을 연결했습니다. 정상 거시/수급 소실, 가격/환율 누락 경고 없음, 수급NULL→0, 정량0 환불 누락을 기록했습니다. 단절 중 전체84.885초/NEWS68.067초, 응답=SQLsnapshot=상세46개·실제 FastAPI44개를 대조했습니다. 소유 MySQL/Redis/프록시/FastAPI 정리·종료와 입력606개/제품 등924개 해시 보존을 확인했습니다. 해당 단계까지23묶음은 **509개404 PASS/105 FAIL**입니다.

후속 [Feature2 실제 Peer·로컬 뉴스 모델·SQL 결합](isolated-feature2-pipeline.md)은 **14개8 PASS/6 FAIL**(`run-5br21593`)입니다. 신규 Peer 오류 캐시 복구 실패5사례와 기존 최신일 제외1사례를 재현했습니다. 실제 역방향SQL/독립 상관·시차,모델 누락/복구·RAW fallback·SQL 저장 장애의 부분 결과 보존을 확인했습니다. 실제FastAPI61개/역방향TCP18개·모델40결과/공개69점수·snapshot/상세26개/USE26개를 대조했고 소유 서비스는 정리·종료했습니다. 입력610개/모델7파일·제품 등924개 해시 보존을 확인했습니다. 이 단계까지24묶음은 **523개412 PASS/111 FAIL**이었습니다. 후속 브라우저12개를 포함한 당시25묶음은 **535개416 PASS/119 FAIL**이었으며 전체 QA는 미완료입니다. 다음은 Feature3 나머지 overlay·실제 원천 브라우저/저장/PDF, 실제 외부 제공자·나머지 Swagger 사용자 흐름입니다.

검증일: 2026-09-25. 프로젝트 루트에서 실행합니다.

```bash
.venv_wsl/bin/python -B -m unittest discover -s tests -v
```

`test_documentation.py`는 이번에 작성·갱신한 공개 Markdown의 로컬 링크 대상 존재 여부를 검사합니다. 웹 링크의 실시간 가용성이나 Markdown rendering은 검사하지 않습니다.

검사 대상은 `test_documentation.py`의 DOCUMENTS에 명시된40개 문서입니다. Spring의 공매도·수급·스냅샷·OpenDART·SEC·Feature3·진단/종목동기화·실제TCP·캐시reader·Peer역방향TCP·실제KIS 기록과 Python 배치의 수급·OHLCV·수동매핑·해외조회·종목초기화·CSV적재 및 Front인증/잔액/Chrome인증화면·Feature3화면·Feature1화면 기록, 루트의 세션 인계 문서 QA_CONTEXT.md를 포함합니다. 각 상세 기록의 테스트 메서드40개·50개·37개·46개·50개·40개·29개·25개·11개·39개·30개·25개·24개·23개·25개·17개·8개·20개·11개·KIS5개·Feature3 Chrome15개·Feature1 Chrome10개와Peer Java host1개 문서 포함 여부도 별도 대조합니다.

두 번째 테스트는 YAML의 password·secret·API key·access key·client ID·username에 해당하는 문자열(8자 이상)을 메모리에서 읽고 새 공개 문서에 같은 값이 포함됐는지 검사합니다. 실패해도 값은 출력하지 않습니다. placeholder·짧은 값·다른 유형의 비밀정보는 검사 범위 밖이므로 전체 보안 감사나 모든 과거 문서의 안전성을 입증하지 않습니다.

서비스 기능·Swagger·수학 계산 테스트는 각 패키지의 tests/src/test 문서를 참조합니다. 과거 전체 현황은 docs/verification.md, 현재 진행과 남은 범위는 [최신 기록](QA_PROGRESS_2026-09-28.md)을 따릅니다.
