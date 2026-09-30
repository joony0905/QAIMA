# 세션 인계: 전체 fallback 검증 기준과 Feature2 우선 범위

## 최신 실행 상태

최신 [Feature3 overlay·Redis·HTTP 취소/재시도](isolated-feature3-resilience.md)은 **23개19 PASS/4 FAIL**(`run-p4h4ipqi`)입니다. 선택 overlay 실패 경고 누락3사례와 Redis+가격 SQL 장애의 비용3 환불 누락1사례를 재현했습니다. 실제 FastAPI40회·응답38개·snapshot/상세33개·USE44/REFUND4와 취소6개·후속 복구16개를 대조했고 소유 서비스를 정리·종료했습니다. 취소 차감 잔존/동일 body 별도 차감은 명시 정책 공백 관측이며, 초기2회는 합산하지 않습니다.

직전 [Feature3 실제 원천 SQL·대체 조회·계산/과금](isolated-feature3-source-sql.md)은 **17개12 PASS/5 FAIL**(`run-sphw2mwo`)입니다. 가격/벤치마크 실제 SQL 오류의 환불 누락3사례, 금리 원천 모두 empty에서 빈 HTTP200/차감 잔존1사례, 오래된 가격의 freshness 오표시1사례를 재현했습니다. 실제 FastAPI28개·응답32개·snapshot/상세27개·USE32/REFUND0·장애 복구14개를 대조했고 소유 서비스를 정리·종료했습니다. 초기3회는 합산하지 않습니다.

직전 [Feature1 실제 Redis·HTTP 취소/재시도·SQL](isolated-feature1-resilience.md)은 **17개16 PASS/1 FAIL**(`run-rqzhknkk`)입니다. Redis 단절/재시작·계산 후 캐시 저장 실패의 결과 보존, 취소/timeout 전파·동일 입력 복구·취소된 이전 결과의 캐시/SQL 미덮어쓰기를 확인했습니다. Redis+가격 SQL 복합 장애의 정상 재무 소실은 기존 FAIL입니다. 실제 FastAPI32회·응답29개·snapshot/상세28개·USE35/REFUND1을 대조했습니다. 미완료 취소6개의 차감 잔존과 동일 body2개의 별도 차감은 정책 적합성 PASS와 구분합니다. 소유 서비스를 정리·종료했습니다.

직전 [Feature1 실제 서버·브라우저/저장/PDF](../frontend/tests/browser-feature1-pipeline.md)은 **17개5 PASS/12 FAIL**(`run-k_l2x_cx`)입니다. 계산 장애의 부분 결과/안내·동일 입력 복구와 보조 재무 조회 오류 중 분석 보존은 통과했습니다. 저장 정량·설명/핵심/저장 경고 누락과 기존 원천/snapshot/stale/날짜 실패를 실제 화면까지 재현했습니다. 실제 FastAPI17회·응답20개·snapshot/상세15개·USE20/REFUND3·동일 입력 복구3개를 대조했고 소유 서비스를 정리·종료했습니다. 별도 검토의 볼린저 위치91.2%→50.0% 및 PDF 스타일 실패는 JUnit에 중복 합산하지 않습니다.

직전 [Feature1 실제 SQL·snapshot·Redis·계산/저장](isolated-feature1-pipeline.md)은 **23개13 PASS/10 FAIL**(`run-y4nj39wc`)입니다. 원천 오류의 전체 전파, 기존 캔들 미보존, 빈 snapshot의 재무 fallback 차단, stale/저장 경고 누락, 날짜 경계와 기존 저장 매핑 실패를 재현했습니다. 실제 FastAPI28회·응답31개·snapshot/상세26개·USE31/REFUND3·후속 정상8요청을 대조했습니다. 소유 서비스는 정리·종료했고 초기3회는 합산하지 않습니다.

직전 [Feature2 실제 SQL·Peer·로컬 모델의 브라우저/저장/PDF](../frontend/tests/browser-feature2-pipeline.md)은 **12개4 PASS/8 FAIL**(`run-kd_uymvv`)입니다. 실제 원천 경고의 화면 누락, 모델 점수 없음에서 보고서 기사 소실, 산업지수 장애의 정상 RAW Peer 차트 소실과 기존 저장 정량/거시/NULL 수급 실패를 확인했습니다. 실제 FastAPI42회·모델31결과/공개33점수·snapshot/상세14개/USE14개·PDF2개6페이지를 대조했습니다. 최종 소유 서비스는 정리·종료했으며 초기 보정/중단 실행은 합산하지 않습니다.

직전 [Feature2 실제 Peer·로컬 뉴스 모델·SQL 결합](isolated-feature2-pipeline.md)은 **14개8 PASS/6 FAIL**(`run-5br21593`)입니다. 신규 Peer 오류 캐시 복구 실패5사례와 기존 최신일 제외1사례를 재현했습니다. 실제 역방향SQL/독립 상관·시차,모델 누락/복구·RAW fallback·SQL 저장 장애의 부분 결과 보존을 확인했습니다. 실제FastAPI61개/역방향TCP18개·모델40결과/공개69점수·snapshot/상세26개/USE26개를 대조했고 소유 서비스는 정리·종료했습니다.

직전 [Feature2 실제 시장 SQL·Redis·FastAPI 설명](isolated-feature2-source-sql.md)은 **24개13 PASS/11 FAIL**(`run-udwy3eb5`)입니다. 실제 원천SQL의 정상 거시/수급 소실·누락 경고 없음, 수급NULL→0, 정량0 차감/저장을 재현했습니다.3기사+Redis 단절은 전체84.885초/NEWS68.067초였고,같은 client복구·snapshot/상세46개·실제FastAPI44개를 대조했습니다. MySQL/Redis/프록시/FastAPI는 정리·종료했습니다. 초기 실행과 테스트 보정은 상세 문서에 기록하며 최종24개만 누적합니다.

직전 [Feature2 실제 Redis·조립/HTTP 취소/SQL](isolated-feature2-resilience.md)은 **15개12 PASS/3 FAIL**(`run-8m54svrw`)입니다. 취소된 이전 가격/산업 source가 새 캐시 값을 덮어 다음 HTTP/SQL도 과거 값이 되는2경로와, 복구 후 뉴스 오류 경고 잔류를 확인했습니다. Redis 장애 중 조립 약40.6초, 복원 후 같은 client 재접속 확인 최대27.340초를 기록했습니다. 소유 MySQL/Redis/프록시/제어 서버는 정리·종료했습니다.

2026-09-28 사용자 `resume` 이후 goal은 active입니다. 최신 누적은 **30묶음632개481 PASS/151 FAIL**이며 전체 QA는 미완료입니다. 다음은 Feature3 나머지 overlay·실제 원천 브라우저/저장/PDF, 실제 외부 제공자·나머지 Swagger 사용자 흐름입니다. 앞선 Feature2에서는 네 명시 취소·HTTP timeout1개·두 경합의 첫 요청까지7개에서 차감이 남고 report는 없었습니다. 취소 환불 기준이 정책에 명시되지 않아 관측으로 구분했고, 취소 전파 PASS를 과금 적합성으로 해석하지 않습니다.

앞선 [Feature2 실제 브라우저·조립/HTTP/SQL](../frontend/tests/browser-feature2-server.md)은13개7 PASS/6 FAIL(`run-l3jr_n8_`), [실제 조립/HTTP/SQL](isolated-feature2-fallback.md)은39개17 PASS/22 FAIL(`run-bxtp_64j`)입니다. 보조 조회 오류의 성공 결과 숨김, 저장/핵심 오류 안내 누락, 설명 실패 저장 상세의 정량 누락, text 설명 누락은 유지합니다. 정상 PDF는 요약 표시/파일 건전성을 확인했으며 정량/차트 전체 재현의 완료 증거가 아닙니다. 초기 실행은 중복 합산하지 않습니다.

## 사용자 지시와 재개 순서

2026-09-28 사용자 추가 지시: **전체적으로 fallback 흐름이 잘 작동하는지 철저히 확인하고, 구성요소가 많은 Feature2의 장애 시 fallback을 특히 점검한다. 다른 세션이 이어받을 수 있도록 상위 QA 문서에 기록한다.** 이 지시는 이번 Redis 검사에만 한정되지 않으며 전체 QA 완료 기준에 적용합니다.

세션 재개 시 [상위 QA 인덱스](README.md) → 이 문서 → [누적 진행/미완료 범위](QA_PROGRESS_2026-09-28.md) → 각 실행 증거 순서로 확인합니다. 기존 수정 제한은 **tests 폴더 내부만**, DB/Redis는 새 외부 격리 인스턴스입니다. 제품 결함은 재현·영향·근거를 기록하고 기존 사용자 변경/DB를 보존합니다.

정책 근거는 [공개 정책](../policy/QAIMA_POLICY_PUBLIC.md), [사용자 분석 흐름](../policy/Feature_analysis_user_flow_policy.md), [API/저장 계약](../policy/API_DTO_CONTRACT_POLICY.md), [캐시 정책](../policy/cache_policy.md), [설명 정책](../policy/LLM_explain_policy.md)입니다. 원천 일부 누락 시 가능한 부분 결과+warning, 설명만 실패하면 정량 결과 유지, 저장 실패 시 계산 성공 유지/저장 경고, 핵심 실패 시 정책에 따른 환불을 기준으로 삼습니다. 정책/구현/검증 기대값이 다르면 차이를 명시하고 구현 자체를 정답으로 삼지 않습니다.

## 공통 완료 기준

각 기능/구성요소별로 아래 증거가 있어야 해당 fallback을 검증 완료로 표시합니다. PASS 조합·재현된 FAIL·미검증 조합을 구분합니다.

1. **장애 위치와 형태**: 원천/API·DB·Redis·계산·설명·리포트 저장·브라우저 보조 요청 중 어디인지, timeout/연결 단절/5xx·4xx/빈 결과/null/형식 오류/부분 누락 중 무엇을 주입했는지 기록합니다.
2. **대체 경로**: fresh→원천/DB→허용된 stale→부분/빈 결과 등 서비스 정책에 맞는 경로와 실제 호출 순서/횟수를 확인합니다. 대체값의 출처·기준시각·종목/기간·단위·입력 hash가 맞아야 합니다.
3. **정상 결과 보존**: 한 구성요소가 실패해도 독립적인 나머지 값·차트·뉴스·수치가 유지되는지 확인합니다. 누락을0/정상/최신 데이터로 오인시키지 않아야 합니다.
4. **다중 장애**: 공유 원천 장애, 서로 다른 구성요소2개 이상 동시 실패, 대체 원천도 실패, 핵심 데이터 전부 없음의 조합을 확인합니다. 이미 수집한 결과 소실·후속 단계 건너뜀·연쇄 실패를 관찰합니다.
5. **응답에서 화면까지**: 내부 warning/error→Public data/meta→프론트 파싱→카드/리포트/저장 상세/PDF에서 안내와 부분 결과가 유지되는지 확인합니다. HTTP200이나 예외 없음만으로 통과시키지 않습니다.
6. **과금/저장/소유권**: 설명/저장 실패와 핵심 분석 실패를 구분하고 실제 잔액·원장·report snapshot을 대조합니다. 취소·재시도·중복/겹친 요청의 차감/환불은 별도 검증합니다.
7. **복구**: 장애 제거 후 같은 client/service의 정상 데이터/캐시 재생성과 다음 요청 상태를 확인합니다. 이전 종목/기간 응답이 새 결과를 덮는 경합도 포함합니다.
8. **검증 경계와 재현**: 명령·실행 ID·소스 해시·입력/기대값·실제 결과·경고·원장·DOM/PNG 등을 남깁니다. mock/실제 socket/실제 SQL/실제 provider를 명시하고 초기 테스트 오류와 제품 결함을 구분합니다.

## Feature2 우선 확인표

실제 조립기는 [Feature2AnalyzeService](../backend/src/main/java/com/qaima/service/feature2/Feature2AnalyzeService.java)입니다. 종목→기준금리→거시금리/시계열→공매도→수급→산업 분류→산업지수→Peer→추세 요약→뉴스/감성→설명→저장/UI 순서를 확인했습니다. 다음은 **완료 선언이 아니라 남은 조립 경계 점검표**입니다.

| 경계 | 중점 확인 | 현재 증거와 미검증 |
|---|---|---|
| 종목/산업 분류 | 종목 없음과 산업만 없음 구분, 독립 금리/공매도/수급/뉴스/설명 보존 | 실제 industry reader의 분류 없음→독립 추세/뉴스/설명 생략 FAIL. 남은 금리/공매도·산업 경고의 화면 표시는 PASS. stock resolver 오류→정량0인데 차감/저장 FAIL과 화면 INTERNAL_ERROR 안내 누락 확인. 실제 stock/price/industry SQL을 연결했고 가격 오류의 누락 경고 없음 FAIL 추가. 실제 미등록 종목/전체 외부 원천은 남음 |
| 기준금리/거시금리/시계열 | 현재값 정상+시계열 실패 및 역조합, 각 원천 부분 실패 | 실제 금리 서비스 최초+대체 실패→후속 조립 중단 FAIL. 상위 zip의 error/empty 양방향 정상 상대값 소실 FAIL. 선택한 실제 카드/JPA를 연결했고 환율/채권 SQL 오류의 정상 거시 요약 소실·환율 빈 행 안내 누락 FAIL 추가. 외부 제공자 전체 결합은 남음 |
| 공매도·수급 | 조회/시계열 실패 시 다른 지표 유지, null/단위/날짜 | 실제 공매도 service 조회 오류와 합성 수급/추세 오류의 나머지 값·SQL 보존 PASS. escaped SHORT 오류의 조립 중단 FAIL. 실제 한쪽 수급 SQL 장애의 양쪽 소실·수급3행NULL→0/보합·경고 누락 FAIL 추가. 기존 단위/null 화면 실패 유지 |
| 산업지수·Peer | 데이터 부족·provider/계산/캐시 오류의 전체 전파 차단 | 실제 service/Redis+합성 source 오류의 독립 정량/SQL 유지 PASS. 가격/산업 reader의 취소된 source가 새 캐시를 덮는2경로 FAIL 추가. escaped INDEX 후속 중단/empty 안내 FAIL 유지. 실제 산업 SQL 오류/10행 기존자료 fallback PASS. 실제 SQL→Python Peer 계산/RAW 대체 PASS. 원천 복구 후 오류Peer12시간 캐시 재사용5사례/최신일 제외FAIL. 실제 RAW 결과91점이 남아도 산업지수 없음에서 화면 차트/모달 소실FAIL 추가. 다른 요청 시각/키의 UI 복구PASS는 같은키 실패를 해결하지 않음. 외부 제공자 전체 결합은 남음 |
| 뉴스·본문·감성 | 기사 유지/점수 null, 부분 감성 실패, 출처 hash/버전 | 실제 뉴스 Redis fallback 약28초→전체 조립40.6초 확인. 부분 감성/명시 재분석의 화면 PASS. 실제 복구 후 이전 캐시 오류 경고 잔류 FAIL 추가, 입력 hash 경합 FAIL 유지. 3기사+실제 news/sentiment SQL/모델 일부·전체 실패·같은 서비스 복구 PASS, 실제 Redis 단절 NEWS68.067초 관측. 감성 INSERT 오류 이후cache 재사용으로 SQL0 잔존은 별도 관측. 실제 로컬 모델40결과/공개69점수,모델 누락/복구·INSERT 오류 보존 PASS. cold 모델93.097초/Spring30초 timeout 후 뉴스 보존. 후속 실제 브라우저에서 모델없음 안내·뉴스탭 기사유지는PASS지만 null summary의 보고서 기사목록 소실FAIL,복구/INSERT오류 점수 표시는PASS. 최대30기사/실제 제공자는 남음 |
| LLM 설명 | 오류/timeout/빈 explain에도 수치/뉴스 유지·경고 표시 | 합성 explain 오류 시 live 정량/안내·과금/SQL PASS. 저장 상세는 정량 누락 FAIL, text-only live 설명 누락 FAIL. empty/null explain 안내 누락 FAIL 유지. 실제 FastAPI44개 무키(Gemini/OpenAI) 경고·정량/SQL 보존 PASS. 실제 무키 경고의 브라우저 안내 누락/저장 정량 누락FAIL을 추가. 유료 모델/전체 응답·UI 조합은 남음 |
| Controller·차감·저장 | 부분 성공과 전체 핵심 실패 구분, 환불/저장 경고 | 실제 조립/SQL 정량0의 차감/저장 FAIL. 부분 성공 유지 PASS·저장 오류 화면 안내 FAIL. 앞선71개/브라우저12개 외 실제 Redis 단계 snapshot/상세28개, 시장SQL/설명 단계46개 및 실제Peer/모델 단계26개와 실제원천 브라우저 단계14개 추가. 취소7개 차감 잔존/저장 없음은 명시 환불 정책 공백 관측. 프로세스 중단/멱등성/나머지 경쟁 남음 |
| 브라우저 보조 조회 | 종합 분석200+관련주/카드500에도 분석 유지 | FRONT-F2-AUX-001을 실제 서버로2경로 재현. 종합 분석200/차감−1/report1인데 보조 시계열500으로 화면 결과 숨김, 저장 상세 조회는 성공. 나머지 보조 요청/취소/경합 남음 |
| 다중 장애/복구 | macro+Peer, 뉴스+LLM, Redis+원천, 전 optional source 실패, 후속 정상 요청 | macro+Peer/BASE+SHORT FAIL, 뉴스+LLM/수급+두 추세 PASS. 실제 Redis+가격/산업 source 오류 및 Redis+Peer/감성 오류의 선택 조합 PASS. 같은 client 재접속 확인 최대27.340초. 취소 경합/뉴스 경고 잔류 FAIL. 추가24개에서 환율+시장 수급 SQL 장애 FAIL, 실제 source/3기사+Redis 단절 복구 PASS. Peer503+실제 모델 누락에서 독립 결과 유지/모델 회복 PASS,Peer 오류 캐시 복구 FAIL. 전체 원천/프로세스 결합 남음 |

기존 [Feature2FlowTest](../backend/src/test/java/com/qaima/verification/Feature2FlowTest.java)와 [분석 billing 검사](java/com/qaima/qa/IsolatedAnalysisBillingTest.java)는 Controller/과금 경계에서 Feature2AnalyzeService를 mock합니다. **이 기존 결과만으로 실제 전체 조립 fallback을 검증했다고 표시하면 안 됩니다.** 추가39개는 실제 normalizer/assembler/Feature2AnalyzeService/SQL을 연결했지만 하위 경계 대부분은 mock입니다. 후속13개는 실제 브라우저까지, 추가15개는 실제 가격/산업/Peer/뉴스 service와 Redis/HTTP 취소까지 연결했습니다. 실제4종 하위 서비스의 증거·escaped error/일반 원천 오류·warning 없는 empty 내구성 검사를 구분하고, 추가24개로 실제 시장 원천 SQL/3기사/실제 무키 설명을 연결했습니다. 추가14개로 실제 Peer 역방향 JPA/계산과 로컬 뉴스 모델까지 연결했습니다. 후속12개로 선택한 실제 원천/계산 결과의 브라우저·저장/PDF를 연결했습니다. 실제 외부 제공자·전체 UI/원천 조합과 모델 품질/성능은 계속 남아 있습니다.

## 기존 증거와 해결되지 않은 실패

| 영역 | 근거 | 현재 한계/실패 |
|---|---|---|
| 캐시 오류/대체 조회 | [Redis core](isolated-redis-cache.md), [뉴스/metrics](isolated-news-overlay-redis.md), [TCP/재시작](isolated-redis-lifecycle.md), [실제 Feature2 조립/취소](isolated-feature2-resilience.md) | TCP/재시작22개 PASS(run-c0aa7rdm). 실제 Feature2 조립40.6초/취소·복구15개12 PASS/3 FAIL 추가. 산업 warning 소실·빈 가격/null accessor·뉴스 무효 캐시/입력 hash 경합과 새 취소 캐시 역전/복구 경고 잔류 FAIL 유지 |
| Feature1 | [실제 SQL/계산/저장](isolated-feature1-pipeline.md), [실제 브라우저/저장/PDF](../frontend/tests/browser-feature1-pipeline.md), [프론트](../frontend/tests/browser-feature1-charts.md), [billing](isolated-analysis-billing.md) | 서버23개13 PASS/10 FAIL에 브라우저17개5 PASS/12 FAIL 추가. 원천/snapshot/stale/날짜 실패·저장 정량/경고 누락. 계산503·동일 입력 복구3개·보조 재무500 중 분석 보존 확인. 별도 볼린저 위치91.2%→50.0%/PDF 스타일 실패. Redis/HTTP 취소17개16 PASS/1 FAIL 추가: 단절·재시작·계산 후 캐시 실패·취소/timeout/복구 확인, 가격SQL 복합 장애의 기존 재무 소실FAIL. 취소 차감은 정책 관측. 전체 원천·브라우저 취소/프로세스·멱등성은 남음 |
| Feature2 | [실제 조립/SQL](isolated-feature2-fallback.md), [실제 브라우저/서버](../frontend/tests/browser-feature2-server.md), [실제 Redis/취소](isolated-feature2-resilience.md), [Feature2 실제 시장 SQL·Redis·FastAPI 설명](isolated-feature2-source-sql.md), [실제Peer/로컬모델](isolated-feature2-pipeline.md), [브라우저 흐름](../frontend/tests/browser-feature2-flow.md), [차트/null](../frontend/tests/browser-feature2-charts.md), [공매도 SQL](isolated-short-selling.md) | 조립 연쇄 누락/empty 안내/정량0 환불 누락 및 화면 보조 오류/저장·설명 누락 FAIL. 새 캐시 취소 경합/복구 경고 FAIL. 실제 시장SQL/3기사/무키 설명24개에서 정상 거시/수급 소실·NULL→0/경고·환불 실패 추가. 실제 Peer/모델14개에서 오류 캐시 복구5사례/최신일 제외FAIL 추가. 실제 원천 브라우저12개4 PASS/8 FAIL로 확장: 실제 경고/기사/RAW 차트 누락 및 기존 저장 정량/거시/NULL 실패. 실제 제공자·전체 UI 조합/사용자 흐름은 남음 |
| Feature3 | [실제 계산/SQL](isolated-analysis-pipeline.md), [브라우저 통합](../frontend/tests/browser-server-pipeline.md), [billing](isolated-analysis-billing.md) | 입력 준비 단계 환불 누락·저장 실패 화면 안내 누락 FAIL. 원천 복합 장애·취소/재시도 미완료 |
| 배치 | [SQL](../batch/tests/isolated-sql.md), [Spring 예약](isolated-spring-batch.md), [ETL 정책](../policy/ETL_POLICY.md) | OHLCV rollback/잠금 등 FAIL. 프로세스 중단/연결 복구/여러 인스턴스 남음 |

고유 결함 수와 실패 사례 수는 다릅니다. 기존 실패는 tests-only 범위에서 수정 완료로 바꾸지 않습니다. 전체 fallback 정상 여부는 이 표와 실제 증거가 채워지기 전에는 확정하지 않습니다.

## 다음 세션에 남길 기록

- 이번 검증 단계/장애 조합, 단일·복수 장애 여부, 실제/대체 경계와 source/run ID.
- 기대 fallback과 실제 응답·warning·유지/누락된 결과·원장·복구 결과.
- 제품 FAIL, 테스트 구성 오류, 미검증/관측만인 항목의 구분.
- 누적 표에는 최종 사례만 추가하고 재실행/보강 단언은 중복 합산하지 않음.
- 다음 우선순위와 전체 미완료 범위. 이 사용자 지시를 다음 세션에서도 유지.

## 다음 실행 시작점과 전체 미완료 범위

[Feature3 overlay/Redis/HTTP 취소23사례](isolated-feature3-resilience.md)를 `run-p4h4ipqi`에서 확정했습니다.23개19 PASS/4 FAIL이며, 초기 compile0개/정상1PASS는 합산하지 않습니다. 선택 overlay 원천 실패의 경고 누락3사례와 Redis+가격 SQL 실패의 예약3 환불 누락1사례가 남습니다. 실제 원천 SQL17개는 `run-sphw2mwo`입니다.

1. 다음은 **Feature3 industry/correlation/news overlay 조합·실제 원천 브라우저/저장/PDF**입니다. 이번23개는 fundamentals/technical의 합성 Feature1 leaf와 실제 core SQL/수학/과금/저장을 연결했습니다. 나머지 overlay의 실제 Feature2 경계·다중 장애/경고·동적 비용과 화면 전달을 확장합니다. 사용자 취소6개의 차감 잔존은 정책 공백 관측이며, 다중 backend/프로세스 강제 종료/브라우저 취소·멱등성 정책은 계속 남습니다. 제품 코드는 바꾸지 않습니다.
2. Feature1은 이번 실제 Redis 단절/재시작·네 취소 지점/실제HTTP3초 timeout·같은 body 복구를 확인했습니다. 지연된 옛 계산을 취소한 뒤 새279.5 캐시/SQL은 유지됐습니다. 브라우저 취소/프로세스 강제 종료·다중 backend·멱등성 정책은 완료되지 않았습니다.
3. 취소6개의 차감 잔존/report 없음과 동일 요청2개의 별도 차감/report는 정책 공백 관측입니다. 취소 전파 PASS를 환불 적합성으로 확대하지 않습니다. Feature2의 취소7개 관측과 같은 기준을 유지합니다. Feature2의 동일 키 Peer 오류 캐시12시간 잔류/취소 캐시 역전/조립·화면 FAIL도 계속 남습니다.
4. 최신 증거는 `run-p4h4ipqi/summary.json`, XML, `feature3-resilience-evidence-audit.json`입니다. 응답38개/SQL상세33개/USE44·REFUND4/실제FastAPI40개·수학35개·후속복구16개를 대조했습니다. 문서73개 검사2개 및23사례/30묶음/초기 포함3회 정리·tests 밖924개 보존 감사는 `history-and-documentation-audit.json`에 기록합니다. 원천 SQL/금리 empty/가격 freshness 실패의 직전 증거는 `run-sphw2mwo`입니다.
5. runner 실행 중 runner/공유 helper/Java/CJS 입력을 바꾸지 않습니다. 이후 입력 변경으로 과거 hash가 달라질 수 있으므로 과거 auditor의 현재 hash 비교를 무조건 재실행하지 않습니다. 진행 중인 실행은 같은 handle로 확인하고 중복 기동하지 않습니다.

전체 외부 제공자·나머지 Swagger123개 사용자 흐름·프로세스/동시성/모델 품질·전체 기간/시장/접근성 범위는 누적 문서와 위 표에 그대로 남습니다. 전체 fallback, 특히 여러 구성요소가 연결된 Feature2의 장애/복구를 우선한다는 사용자 지시는 계속 유효합니다. 최신 누적 **30묶음632개481 PASS/151 FAIL**, 전체 QA 미완료·goal active입니다.
