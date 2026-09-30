# 사후 검증 현황과 미해결 사항

작업·검증일: 2026-09-25 (Asia/Seoul). 졸업프로젝트 종료일: 2026-07-03. 대상은 `dev`의 `345aabb5075964e3ddd07ddd351915e330862e03` 및 기존 사용자 변경이 포함된 현재 작업 트리입니다. Git 이력만으로 현재 구현을 대체하지 않습니다.

## 현재 결과

| 검증 | 실행 환경 | 결과 | 해석 범위 |
|---|---|---|---|
| Spring compileJava·compileTestJava | Windows Java17·Gradle8.14 | PASS | 현재 소스 컴파일 |
| Peer WSL→Windows 역방향 TCP | 실제uvicorn·Netty/WebFlux Controller/Service·합성Repository | Python8개7 PASS·1 FAIL, Java host1 PASS | 실제raw/산업상관·lag·정확기간/fallback·오류/422, 기존최신일누락을최종차트까지재현. source연결6·금지연결0·유료0, 임시서버종료. [상세](../backend/src/test/peer-reverse-tcp.md) |
| 최신 Spring 전체 회귀 suite | Windows, 실제 연동 opt-in 해제 | 1053개:1002 PASS·34 FAIL·17 SKIP | F1 내부문구1·F3 환불누락2·F1 리포트 매핑1·캔들decode2·F2 거시부분결과소실1·뉴스캐시판정4·Peer최신일제외1·재무half1/필드누락2/CSV인용1·수급미등록종목3·스냅샷성장률1·OpenDART회사명조각2·SEC주식수범위초과1·13F필수열2/날짜1·F3제공처표기1/중복거래일2/미래지수1·동기화권한1·진단내부문구1·빈가격목록1·산업캐시경고소실1 실패. TCP11·Peerhost1·SMTP1·인프라2·DB생성1·KIS1은 이 기본 실행에서 의도적으로 미실행(TCP11은 별도 PASS). XML합계확인, 약57초 |
| Spring 단위·계약·보안 초기 suite | Windows | 381 PASS | 369개는 실제 보안필터+sentinel. 업무 API 전체 통과 아님 |
| MySQL SELECT1·Redis PING | Windows, YAML 실제 설정 | 2 PASS | read-only 인프라 가용성. 업무 데이터·저장 미검증 |
| 실제 KIS 토큰·종목 기본정보 | Windows, 실제 KrStockClient·TLS/JSON | 실제1 FAIL·호출guard4 PASS | 전송2회 모두200이나 후속단언실패. 최초마스킹으로실패단언특정불가·재실행없음. DB/Redis쓰기0, Public 전체PASS 아님. [상세](../backend/src/test/live-kis-readonly.md) |
| 가격·산업지수 캐시 reader 추가분 | Windows, 실제 서비스/codec·전송mock | 39개:37 PASS·2 FAIL | 가격21·산업18. 실제24개겹치는요청/분리key·산업카드HTTP2, 빈가격목록NPE와캐시warning소실 재현. 실제Redis/DB쓰기0 |
| JWT·쿠키·사전 HTTP·Feature1 흐름 포함 전체 | Windows | 398개 중 397 PASS, 1 FAIL | 내부 예외 메시지 공개 결함 재현. infra2 포함 |
| Yahoo 가격 정책 추가분 | Windows | 6 PASS | 외부client mock, 결측률·시장 심볼·fallback |
| Feature2/3 Controller 흐름 추가분 | Windows | 12개 중10 PASS·2 FAIL | 차감 후 가격/overlay 오류의 환불 누락 재현. 실제 잔액 변경 없음 |
| 관심목록·포트폴리오 Controller→Service | Windows | 14 PASS | 소유권·추가/수정/삭제·교체, Repository mock |
| 인증 세션·HTTP 흐름 추가분 | Windows | 12 PASS | 가입·로그인·refresh·logout·ID찾기, 실제 서비스·JWT·쿠키, Repository/인증소비 mock |
| 사용자·크레딧 HTTP·캐시 정책 추가분 | Windows | 28 PASS | User8·Credit6·overlay9·snapshot5. Repository/Redis mock, 최신 전체1053에도 포함 |
| 이메일·리포트 HTTP 추가분 | Windows | 26개:25 PASS·1 FAIL | Email14·Report12. SMTP/Repository mock, F1 코드/회사명 뒤바뀜 재현. 최신 전체1053에 포함 |
| 종목·재무·차트 추가분 | Windows | 37개:35 PASS·2 FAIL | Stock14·Financial10·Chart13. 실제 조회/mapper/loader, repo/provider/calendar mock. decode 오류 흡수2개 재현, 전체1053에 포함 |
| 사전 관리자·검색·종목순위 추가분 | Windows | 26 PASS | Dictionary15·Ranking11: 실제 service/HTTP, 저장소/provider/달력 mock·시계고정 TTL. 최신 전체1053에 포함 |
| Feature2 개별 카드 추가분 | Windows | 18개:17 PASS·1 FAIL | 실제 Controller/CardService9경로, 하위service/repo mock. 금리·시계열·수급·산업·관련종목 검증, 거시 이중실패 시 정상값 소실 재현. 전체1053에 포함 |
| 뉴스 조회·감성캐시·비용 연결 추가분 | Windows | 23개:19 PASS·4 FAIL | 실제 NewsController/Service·OverlayService, Redis/Repository/provider/모델 mock. focus변경/null점수 적중오판·재분석누락·비용판정 재현. 전체1053에 포함 |
| 기존 YAML 계정으로 격리 MySQL 생성 | Windows, YAML 계정 | FAIL: SQLState42000/code1044 | 이 계정의 DB 생성 권한 없음. 해당 서버에 새 DB 생성·마이그레이션 미실행, 기존 DB 변경 없음 |
| 별도 임시 MySQL SQL 마이그레이션 | Windows MySQL8.0.27, 프로젝트 밖 datadir·별도 loopback 포트 | 새 QA 스키마에 V1~V51 51개 성공·테이블47·외래키41·한글 seed1 | 설정된 기존 계정의 DB 생성 실패와 별개. 이 CLI 검사 자체는 Flyway history/JPA/업무 SQL을 검증하지 않음. [실행 경계](../backend/src/test/isolated-mysql-cli.md) |
| 별도 임시 MySQL Flyway·JDBC | Windows MySQL8.0.27·Java17·Flyway12.4.0·Connector/J9.7.0 | 새 QA 스키마 baseline1+SQL51·validate PASS·재실행0, JDBC 중복1062/FK1452/소유자1:0/rollback 후 0행 | 실제 DB 제약·트랜잭션이며 Spring JPA/Service/API·동시성 아님. [상세](../backend/src/test/isolated-flyway-jdbc.md) |
| 별도 임시 MySQL Spring Data JPA | 프로젝트 밖 backend 복사본·Java17·Gradle8.14·JPA slice | 2 PASS, 0 FAIL. 원본 본체554개 해시 일치; 리포트 소유자/기간·사용자별 삭제, 포트폴리오 보유종목 순서·현금 소수, rollback 후 업무4테이블 0행 | 실제 entity/repository/Hibernate/MySQL. Controller·Service·인증·동시성·실사용자 없음. [상세](../backend/src/test/isolated-jpa-copy.md) |
| 격리 MySQL Report/Portfolio Service·Controller 직접 호출 | 프로젝트 밖 backend 복사본·JPA slice·실제 MySQL | 3 PASS, 0 FAIL. 리포트 snapshot·7일 정리/소유권, 포트폴리오 교체/orphan·타인분리, Controller principal 전달·상세조회 삭제 부작용. 정리 후 업무4테이블0행 | 실제 Controller 메서드→Service→Repository→DB. HTTP 바인딩·보안필터/JWT·전체 앱/동시성은 아님. [상세](../backend/src/test/isolated-service-jpa-copy.md) |
| Peer pack·client·cache 추가분 | Windows | 27개:26 PASS·1 FAIL | Data15·ClientCache12: 실제 pack/client/codec/service, repo/calendar/Redis/HTTP전송 mock. window최신일 제외 재현. 전체1053에 포함 |
| 관리자 거시지표 수집18경로 | Windows | 23 PASS | 실제Controller/sync service/provider client/JSON, 전송·Repository만mock. BOK페이지/연도분할/재시도·FRED일간/월간·upsert요청·당일재사용·입력오류. 실제SQL/외부는미검증 |
| 관리자 재무CRUD·CSV4경로 | Windows | 28개:24 PASS·4 FAIL | 실제service/mapper/CSV/SQL인자·Repository/JDBC/tx mock. half단독무시1·절대값3필드누락2·quoted숫자거부1 재현 |
| 관리자 공매도5경로·scheduler | Windows | 40 PASS | 실제Controller/service/client/parser/codec·전송/저장소mock32개, 직접scheduler8개. form/SQL·오류·3000일/1000행·부분커밋·시간대/중첩실행. 실제API/DB/cron 미검증 |
| 관리자 수급5경로·배치/scheduler | Windows | 50개:47 PASS·3 FAIL | API30·배치/재시도/limiter16·scheduler4. 미등록종목3경로200/빈본문 또는 빈rows 성공응답 재현, 토큰/23숫자/21일앵커/fallback·15:40/대상/집계/guard. 실제전송·저장소는mock |
| 관리자 스냅샷2경로·주식수 기준 | Windows | 37개:36 PASS·1 FAIL | 실제backfill/주식수/계산기/캐시codec/DTO·저장소/tx/KIS/Redis mock. EPS/BPS/SPS분모·TTM/연간·불완전/skip/force·오류검증. 재무수정본을전년처럼비교하는성장률FAIL |
| OpenDART 관리자4경로·클라이언트/scheduler | Windows | 46개:44 PASS·2 FAIL | Client16·HTTP/service25·scheduler5. 실제ZIP/XML/JSON/매핑/10숫자/보고서날짜/자연키·일일순서/오류·중첩관측, 전송/저장소mock·curl시작차단. entity/CDATA회사명소실2 FAIL. 실제API/SQL/cron 미검증 |
| SEC 발행주식수2경로·미국마스터1경로/scheduler | Windows | 50개:49 PASS·1 FAIL | Facts13·주식수HTTP17·마스터16·scheduler4. 실제codec/parser/service·전송/저장소/tx mock. CIK/최신선택/자연키/별칭/limit/오류·guard. 2^64+1→1주 변환FAIL(숫자/문자열2단언). 13F5경로·실제SEC/SQL/JWT/cron 별도 |
| SEC13F 관리자5경로·파일/집계 | Windows | 40개:37 PASS·3 FAIL | Parser9·적재/매핑18·집계/조회13. 실제임시ZIP/TSV/service/비율/증감·저장소/native SQL projection mock. 필수열누락SUCCESS2·31-Feb→2/28보정1 FAIL. 기관별최신/정정선택SQL·실제DB/원천 미검증 |
| Feature3 가격·벤치마크2경로 | Windows | 29개:25 PASS·4 FAIL | 가격15·지수14. 실제service/Yahoo codec·품질/캔들로더/지수매핑·전송/저장소/달력 mock. 제공처표기1·중복거래일2·미래지수가용1 FAIL. 실제API/SQL/JWT결합·브라우저 미검증 |
| 진단·OAuth 진입·종목 메타/동기화9경로·권한 결합 | Windows | 25개:23 PASS·2 FAIL | 진단8·메타/동기화14·실제필터/handler3. USER 동기화허용1·진단2경로내부문구1 FAIL. 실제client/codec/service·전송/저장소 mock, OAuth는로컬302만. 원천/SQL/브라우저 미검증 |
| 분석 unittest | WSL Python3.12, 승인된sandbox밖 실행 | 98개:92 PASS·3개메서드 FAIL·3 SKIP | 실패항목5개(subcase포함): 설명false무시1·제공자HTTP본문2·연결예외문구2. 실제모델3개는기본suite opt-in미실행 |
| Feature3 최적화 독립 수학 추가 QA | WSL Python3.12, 표준입력 스크립트·합성 입력 | 240개 비교·제약 검사와 80개 경계해 열거, 실패 0 | 효용최적≥위험배분, 1~4종목 전역해 비교. 내부 함수 범위이며 실제시장·전체 HTTP/DB 연동 아님. [재현](../analysis/tests/optimizer-independent-check.md) |
| 뉴스 실제 로컬 모델 | WSL CPU, 외부연결차단 | 3 PASS | 별도 opt-in실행: 실제모델·확률합/점수·max256·단건/배치·ASGI. 로딩포함145.107초, 정확도/실제Spring TCP연동 아님 |
| uvicorn 실제 HTTP smoke | WSL 임시 프로세스 | 10개 검사 PASS | health·경로등록·422·F1 설명 생략·F3 제공데이터 전체 계산/고급후보 |
| Windows Java→WSL uvicorn 실제 TCP | Windows Java17 + WSL asyncio uvicorn | 별도 opt-in11 PASS | F1 재무/DTO·F2 빈키/422·F3 공분산/제약/시장별fallback·실제로컬모델. Public F3→TCP→보고서/환불 호출 연결, 저장소·가격·크레딧/보고서는mock. 약2분7초/audit0·서버종료·XML별도보존 |
| 프론트 단위·타입검사 | Windows Node25.1 | Node25개23 PASS·2 FAIL, 기존tsc2개 PASS | 계약5+auth/balance20. 실제Axios interceptor·VM store·전송mock, 늦은응답토큰/잔액복원FAIL. 브라우저/실계정아님 |
| 프론트 원본 Vite 빌드 | Windows | FAIL: Windows Rollup native 모듈 없음 | 기존 node_modules 변경 없음 |
| 프론트 격리 npm ci·build | Windows tests/.runtime | PASS | lockfile대로 새 설치한 복사본 |
| Chrome 브라우저 smoke | Windows, 격리 Vite preview | 6개 검사 PASS | 로그인·가입 필드, 비로그인 보호 경로2개, 사전 검색. API는 fixture이며 실제 연동 아님 |
| Chrome 로그인·로그아웃·OAuth화면 | Windows, 실제React·합성API | 11 PASS | 입력/validation/오류escape/잠금·submit비활성·복귀/스토어·logout200/500·callback추가정보/오류. 실제계정/서버cookie/OAuth제공자미실행, 늦은응답회귀해결아님. [상세](../frontend/tests/browser-auth-flow.md) |
| Chrome Feature1 조회·관심종목·분석 | Windows, 실제React·합성API | 10 PASS | 캔들120/+20/+20%·비로그인복귀·관심종목POST/DELETE/오류·선택기간/KST→분석→재무/잔액GET. 실패후실행버튼소실관측. 실제원천/DB/차감/환불/PDF아님. [상세](../frontend/tests/browser-feature1-flow.md) |
| 저장 리포트 PDF 브라우저 | Windows Chrome + WSL PDF 파서 | 9개 검사·PDF4개/6페이지 PASS | API fixture, 실제 PDF 생성·A4·비어있지 않음·한글 육안·상세404/만료 차단. 분석 직후 PDF/실제DB 별도 |
| 배치 unittest | WSL, guarded runner | 159개:144 PASS·15개메서드FAIL(실패항목33) | 기존15+수급30+OHLCV25+수동매핑24+해외23+초기화/조회25+CSV17. 실제11main/codec/decoder/SQL조립·전송/DB/파일mock. 비유한수/rollback/exit/encoding중복/해외주기저장/상장일/CSV필수열FAIL. 실제접속·dotenv읽기·유료호출0 |
| 감성 실험실 unittest | WSL | 4 PASS | 전처리·컬럼·평가지표만 |
| 공개 문서 검사 | WSL | 2 PASS | 로컬링크 존재·설정 secret 값 미포함 검사, 전체 보안 감사 아님 |
| Chrome Feature3 입력·저장·비용·분석/PDF | Windows, 실제React·합성API + WSL PDF파서 | 15개14 PASS·고급PDF1 FAIL | 기존입력/저장/비용/탭검사PASS. 렌더시점원본flex/811규칙→복제block/2규칙을단언으로검출. 외부CSS빈200조건에서도재현. 최신PDF2개5페이지·실제차감/DB/LLM아님. [상세](../frontend/tests/browser-portfolio-flow.md) |

명령·입력·기대값·mock 범위·출력 위치는 각 테스트 패키지 README에 상세히 기록했습니다. Windows Gradle XML은 UTC 시간을 사용하므로 한국 날짜와 표기가 다를 수 있습니다.

## 환경 확인과 대응

1. WSL sandbox 연결 검사는 PermissionError로 실패했고 승인된 동일 검사로 재실행했습니다.
2. WSL localhost에서 MySQL·Redis·Spring·FastAPI는 연결 거절이었습니다.
3. Windows에서 MySQL·Redis 포트가 열려 있음을 확인했습니다. WSL 게이트웨이 TCP 연결은 성공했으나 Python의 MySQL 인증/연결은 OperationalError였습니다. 원인 분류 전 업무 연동 FAIL로 단정하지 않습니다.
4. Windows JDBC에서 YAML 자격증명으로 read-only SELECT1, Redis PING이 실제 통과했습니다. Spring·FastAPI의 기존 실행 인스턴스는 확인되지 않았습니다.
5. WSL FastAPI는 외부 호출·뉴스 warm-up을 비활성화한 임시 uvicorn으로 HTTP 검증 후 종료했습니다.
6. Windows 프론트의 native 의존성 누락은 원본을 수정하지 않고 test 디렉터리의 격리 설치·빌드로 검증했습니다.
7. 신규Peer sync ASGI 검사가sandbox에서대기했습니다. 스택에AnyIO worker queue대기/이벤트루프select가관측되어테스트만중단했고, 승인된sandbox밖동일WSL에서단일·신규17·전체57개가통과했습니다. 근본원인확정이나제품계산실패로단정하지않습니다.

## 확인된 구현 차이·주의점

- **SYNC-AUTH-001 / DEBUG-ERROR-001:** 관리자용 종목 동기화를 일반 USER가 HTTP200으로 실행하고 provider/service에 진입합니다. 실제 보안필터+Controller/합성 신원·업무 경계로 재현했습니다. 진단2경로의 내부문구 공개도 별도 회귀로 남겼습니다. [25개 상세 검사](../backend/src/test/diagnostic-ticker-flow.md), 실제 데이터 변경 없음·제품 미수정.

- **F3-PRICE-SOURCE-001 / F3-PRICE-DATE-DUP-001 / F3-BENCHMARK-FUTURE-001:** 수정종가 fallback의 KIS 고정 표기1건·가격 경로 두 곳의 중복 거래일 집계2건·미래 지수 가용판정1건을 재현했습니다. [29개 상세 검사](../backend/src/test/feature3-market-data.md), 합성 전송/저장소 경계이며 제품은 미수정입니다.

- **SEC13F-HEADER-001 / SEC13F-DATE-001:** 필수TSV열없는파일이SUCCESS가되고불가능한31-Feb-2026이2/28로보정됩니다. parser/HTTP실패3개를유지했습니다. [40개상세검사](../backend/src/test/sec-13f.md), 제품미수정.

- **SEC-SHARES-OVERFLOW-001:** Long범위를넘는2^64+1이숫자/문자열모두1주로변환되어유효fact로채택됩니다. 1개회귀메서드의두단언실패를유지합니다. [50개상세검사](../backend/src/test/sec-issued-master.md), 제품미수정.

- **OPENDART-XML-TEXT-001:** XML entity/CDATA로회사명텍스트가나뉘면앞부분을잃습니다. 두입력의기대 `QA & Partners`가 `Partners`로반환되어2개FAIL을유지했습니다. [46개상세검사](../backend/src/test/opendart.md), 제품미수정.

- **SNAPSHOT-GROWTH-VERSION-001:** 연간재무의동일연도수정본이둘이상이면전년도대신구버전을비교합니다. 독립기대성장률25%/100%가실제11.111111%/14.285714%로반환되어회귀1개FAIL을유지했습니다. [근거](findings.md), 제품미수정.

- **INVESTOR-STOCK-MISSING-001:** 미등록종목의수급 sync/backfill/series가의도된오류분기로진입하지못하고200/빈본문 또는 빈rows 성공응답을반환합니다. 실제HTTP/service와Repository mock으로3개실패를재현했고 [근거](findings.md)에기록했습니다. 제품미수정입니다.

- **FIN-ADMIN-HALF-001 / FIN-ADMIN-FIELDS-001 / FIN-CSV-QUOTE-001:** 재무수정의half단독무시,등록/수정의매출총이익·이익잉여금·현금성자산누락,quoted CSV숫자거부를4실패로재현했습니다. 실제DB는변경하지않았고[근거](findings.md)를기록했습니다.

- **F2-LLM-FLAG-001 / F2-LLM-ERROR-001:** 실제FastAPI→provider client→mock전송에서 설명비활성화 플래그에도 전송1회, 두제공자의HTTP오류본문·연결예외문구가응답warnings에남는것을재현했습니다. 신규17개중14 PASS·3개메서드 FAIL이며 실제유료호출은없습니다. [재현 근거](findings.md). Spring Public/브라우저 전파까지 완료한 검증은 아닙니다.

- **F2-PEER-DATE-001: window 모드 최신 종목 가격 제외.** 산업지수 최신시각을 그대로 exclusive 조회to에 전달하여 동일시각 종목가격이 빠집니다. Repository fixture와 실제WSL→WindowsTCP에서FastAPI최종차트가원천7/30보다하루전7/29에끝남을재현했습니다. 실제SQL·실데이터영향규모는미검증입니다. [재현 근거](findings.md).

- **F2-NEWS-CACHE-001: 뉴스 캐시 호환성 판정 누락.** focus/hash를 변경하거나 점수가null이어도 재사용 가능하다고 판단합니다. 실제 뉴스 service는 이전0.75 또는null을 반환하고 모델호출0, 실제 overlay 서비스의 미리보기/estimate도 HIT·추가0으로 판정했습니다. 합성 경계의4개 실패이며 실제 모델·Redis·과금 오류를 관측한 것은 아닙니다. [재현 근거](findings.md).

- **F2-MACRO-001: 거시 카드의 정상 부분 결과 소실.** 미국 금리 동기화와 저장값 fallback이 모두 실패하면 정상 한국 금리2.75도 HTTP 응답에서 제거됩니다. 실제 Controller/CardService와 mock 소스로 재현했고 회귀1개 FAIL을 유지합니다. 거시 시계열의 소스별 실패 격리 대조군은 PASS입니다. [재현 근거](findings.md)에 기록했습니다.

- **CANDLE-DECODE-001: 캔들 디코딩 오류 흡수.** 명시적 decode 재전파 분기 뒤의 일반 fallback이 오류를 다시 잡습니다. 차트는200/NO_DATA, Feature3 로더는 기존 DB 데이터로 바뀌는 현상을 mock provider와 실제 loader로 재현했으며 두 실패 테스트를 유지합니다. [재현 근거](findings.md)를 참고하세요.

- **REPORT-MAPPING-001: Feature1 저장 리포트 필드 뒤바뀜.** 실제 createFeature1→저장 객체→상세 HTTP 응답에서 stockCode에 회사명, companyName에 종목코드가 들어가는 것을 합성 fixture로 재현했습니다. [재현 근거](findings.md)에 기록하고 실패 테스트를 유지합니다. 실제 DB·기존 리포트는 변경하지 않았습니다.

- **F3-CREDIT-001: Feature3 환불 누락.** 차감 후 가격 데이터 조회·overlay 로딩 오류가 refund 처리 범위 밖에 있습니다. 실제 Controller 테스트에서 차감 구독1회·refund 호출0회를 재현했고, FastAPI 오류 대조군은 환불했습니다. [재현 근거](findings.md)에 기록했으며 서비스는 미수정입니다.

- **F1-ERROR-001: Feature1 내부 예외 메시지 공개.** `FeatOneController.feature1ErrorResponse`가 `ex.getMessage()`를 잘라 Public 오류 문구에 붙입니다. `Feature1FlowTest.providerFailureDoesNotExposeInternalMessage`에서 합성 내부 문구를 포함한 service 예외를 주입했으며 해당 문구가 Public 응답에 남아 FAIL했습니다. 실제 secret을 사용하지 않았습니다. 서비스 수정은 허용 범위 밖이므로 테스트 실패와 근거를 유지합니다.

- 리포트 조회에는 7일 이전 사용자 리포트 삭제가 포함됩니다. 조회 요청도 데이터 변경을 일으킬 수 있으므로 기존 사용자 리포트로 연동 테스트를 수행하지 않습니다.
- Feature3 종목 상한은 고정40%가 아니라 종목 수에 따른 적응형 상한입니다. 세 종목이면 2/3입니다.
- Feature1 LLM 파싱·timeout 실패 fixture는 정량 결과를 유지하면서 explain=null을 반환합니다. 결정론적 설명 fallback을 모든 기능의 공통 완료 기능으로 적지 않습니다.
- FastAPI 성공 payload와 실패 envelope의 형식이 다릅니다.
- 운영 감성 입력과 실험실 입력 포맷이 다르므로 학습 결과의 운영 정확도를 추정하지 않습니다.

## 비용·외부 상태 변경 기록

사용자는 실제 연동·유료 LLM·외부 금융 API·메일·크레딧 검증을 허용했습니다. 비용 발생 호출은 오류 수정 후 재시도를 포함해 최대5회라는 제한을 받았으며, 우선 세션 전체5회 이하로 보수적으로 집계합니다. 라이브 wrapper나 provider 내부 재시도도 한도에 포함해야 하므로 호출 구조를 확인한 뒤 실행합니다.

| 구분 | 실제 수행 | 비용 호출 누계 |
|---|---|---:|
| MySQL SELECT1 | read-only transaction 후 rollback | 0 |
| Redis PING | 실제 응답 확인, cache write 없음 | 0 |
| 임시 uvicorn HTTP | 외부 provider 호출 없음 | 0 |
| Windows→WSL TCP/실제 로컬 감성 모델 | 최종11 PASS·외부연결/프로세스/.env읽기시도0. 중간 실행의 연결시도2회는 송신 전에 차단 | 0 |
| LLM 실패 재시도 | AsyncMock만 호출 | 0 |
| SMTP 인증메일1 | 실제 MailAuthService에서 발송 실패. 저장소는 mock | 1 / 5 보수적 소비 |
| SMTP 인증메일2 | TLS 신뢰 경로 오류 확인. 테스트 발송기가 YAML의 SMTP 속성 일부를 누락했음 | 2 / 5 보수적 소비 |
| SMTP 인증메일3 | YAML SMTP 속성을 모두 적용한 후 발송·해시·확인·소비 PASS. 사용자 수신 확인 완료 | 3 / 5 보수적 소비 |
| 실제 KIS 토큰·종목 기본정보 | 실제전송2회·HTTP200/200, 후속단언FAIL. 추가전송없음 | 5 / 5 보수적 소비 |
| 실제 LLM·크레딧 | 아직 미실행 | 0 |

OAuth 제공자 로그인은 사용자가 브라우저에서 직접 진행하기로 확인했습니다. 실제 메일 수신 주소도 사용자에게 확인받았으며 실행 환경에만 전달합니다. 사용자가 메일 수신을 확인했습니다. 환경 secret과 사용자 이메일·token은 공개 문서에 기록하지 않습니다.

SMTP 초기2회 실패는 테스트 발송기에서 실제 YAML 속성 반영이 불완전했던 조건의 결과입니다. 운영 Spring 메일 설정 자체가 실패한다고 단정하지 않습니다. 테스트 발송기에 동일 YAML 속성을 반영하여3회차가 통과했으며 운영 설정이나 JVM truststore는 변경하지 않았습니다. 이후 KIS2회를 수행해 SMTP3+KIS2=세션 보수적5회로 기록했습니다. 이는 청구금액 확인이 아닙니다. 사용자 원래 조건은 테스트별 재시도 포함5회이며 현재 공통guard는 더 엄격합니다. KIS 단언 실패 원인은 미확정이고 유료 LLM·실크레딧차감은 아직0회입니다.

Feature2 설명의 지속429 합성검사에서 provider 재시도3회×compact재시도2단계=전송6회를 확인했습니다. 실제 호출이 아니라 비용0회입니다. 실유료검증은 이 경로를 무제한 실행하지 않고 사용자한도와 잔여량을 전송직전 차단하는 테스트용 장치가 필요합니다.

사용자는 별도의 격리 테스트 DB 생성을 허용했으며 본 프로젝트에 영향이 없어야 한다고 명시했습니다. 기존 DB·Redis·설정 변경 없이 전용 DB만 대상으로 검증하며 자동 scheduler/importer는 실행하지 않습니다. 생성·마이그레이션·검증 결과는 실제 실행 이후 기록합니다.

## 남은 작업과 완료 조건

Redis QA 전용 무작위 키 생성·TTL 검사·해당 키만 정리하는 실제 연동은 사용자 확인을 요청한 상태입니다. 이번 캐시 reader39개는 서버 쓰기 없이 실행했으며 실제 TTL 만료 검증으로 집계하지 않습니다.

- 123개 Swagger 작업 각각의 Controller→Service→Repository·외부 연동 정상·오류·권한·fallback 흐름 테스트. [작업별 상태](../backend/src/test/swagger-coverage.md)를 모두 증거로 갱신해야 함.
- Feature1/2/3 DTO 전체 계약. Windows client→WSL 실제 TCP11개PASS, Peer역방향TCP8개7 PASS·1 FAIL이나 전체 Spring 기동·실제원천 데이터·캐시·비용미리보기·실차감/원장 환불 일치는 남음.
- Feature3 가격 품질경계·공분산·CAPM·blend·frontier·overlay 독립 기대값과 일관성.
- 사용자 전용 테스트 데이터에서 인증·refresh·소유권·snapshot·포트폴리오·관심목록 실제 저장, mail·OAuth 제공자 흐름.
- 모든 배치·Spring scheduler의 수집·중복 적재·재시도·시간대 검증.
- 브라우저 기능·오류 상태·PDF, 모델 추론·실험 재현, 적절한 성능·동시성 검사.
- 문서 링크·명령 재현성·비밀정보 제외·변경 파일 범위 최종 감사.

현재 전체 목표는 미완료이며, 사용자의 세션 이동 요청으로 Feature1 Chrome 검증과 문서 정리 후 일시정지합니다. 재개 컨텍스트는 루트의 [QA_CONTEXT.md](../QA_CONTEXT.md)에 기록했습니다. 테스트 숫자나 빌드 성공으로 이 남은 범위를 축소하여 완료 처리하지 않습니다.
