# QAIMA 로컬 실행 안내

| 위치 | 준비 항목 | 기준                                            |
|---|---|-----------------------------------------------|
| Windows | Git | 저장소 복제                                        |
| Windows | JDK 17 | `JAVA_HOME`과 `java -version` 확인               |
| Windows | Node.js·npm | lockfile의 Vite 요구 조건: Node `^20.19.0` 또는 `>=22.12.0` |
| Windows 또는 접근 가능한 로컬 서버 | MySQL 8 | 개발용 빈 DB와 그 DB에 대한 사용자 권한                     |
| Windows에서 접근 가능한 로컬 서버 | Redis | 아래 예시는 Windows의 `127.0.0.1:6379`에서 접근         |
| WSL | Python 3.12, pip, venv | fast api 서버 모델이 리눅스 친화적이라 wsl 사용              |

MySQL·Redis는 각자 설치한 로컬 서비스 또는 컨테이너를 사용할 수 있습니다. 저장소에는 전체 서비스를 한 번에 올리는 Compose 구성이 없습니다. Spring이 실행되는 **Windows 기준 접속 주소**를 설정합니다.

Windows PowerShell:

```powershell
git clone https://github.com/joony0905/KNU_SW_QAIMA_GraduationProject.git C:\qaima
cd C:\qaima
java -version
node --version
npm --version
```

WSL:

```bash
cd /mnt/c/qaima
python3 --version
python3 -m venv .venv_wsl
.venv_wsl/bin/python -m pip install -r analysis/requirements.txt
```

`venv`나 `pip`가 없다면 사용 중인 WSL 배포판에서 해당 패키지를 먼저 설치합니다. Windows용 가상환경과 WSL용 가상환경을 공유하지 않습니다. 프론트의 `npm ci`는 아래에서 Windows PowerShell로 실행합니다.

## 2. 로컬 DB와 Redis 준비

MySQL에 관리자 계정으로 접속해 **새 개발용 DB**를 만듭니다. 다음은 로컬 초기화 예시이며 `<LOCAL_DB_PASSWORD>`는 직접 정한 비밀번호로 바꿉니다.

```sql
CREATE DATABASE qaima_local CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'qaima_local'@'localhost' IDENTIFIED BY '<LOCAL_DB_PASSWORD>';
GRANT ALL PRIVILEGES ON qaima_local.* TO 'qaima_local'@'localhost';
```

DB 이름과 계정명은 예시입니다. 컨테이너나 다른 호스트에서 접속한다면 MySQL 사용자 host 조건도 실제 접속 방식에 맞게 준비합니다. 아래 설정의 DB명·계정·비밀번호를 동일하게 맞춥니다.

Redis를 시작한 뒤 Windows에서 포트를 확인합니다.

```powershell
Test-NetConnection 127.0.0.1 -Port 3306
Test-NetConnection 127.0.0.1 -Port 6379
```

TCP 성공은 계정·스키마 검증을 의미하지 않습니다. 실제 DB 연결과 Flyway 결과는 Spring 시작 로그에서 확인합니다.

## 3. Spring 로컬 설정

`C:\qaima-local` 폴더를 만들고, 그 안에 `application-local.yml`을 작성합니다. 개인별 설정을 저장소 밖에서 관리하기 위한 위치입니다. Spring은 `mysql,local` 프로필과 이 디렉터리를 함께 지정해 실행합니다.

아래는 시작점으로 사용할 설정 예시입니다. DB 비밀번호와 JWT 키는 직접 채우고, 주소는 본인 환경에 맞춥니다. JWT 키는 코드에서 UTF-8 바이트로 읽으므로 **최소 32바이트 이상의 충분히 긴 무작위 문자열**을 사용합니다.

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/qaima_local?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Seoul
    username: qaima_local
    password: '<LOCAL_DB_PASSWORD>'
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    baseline-on-migrate: false
  data:
    redis:
      host: 127.0.0.1
      port: 6379
  mail:
    host: localhost
    port: 1025
    username: ''
    password: ''
    properties:
      mail:
        debug: false
        smtp:
          auth: false
          starttls:
            enable: false
            required: false
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: '${GOOGLE_CLIENT_ID:local-google}'
            client-secret: '${GOOGLE_CLIENT_SECRET:local-google}'
          kakao:
            client-id: '${KAKAO_CLIENT_ID:local-kakao}'
            client-secret: '${KAKAO_CLIENT_SECRET:local-kakao}'
          naver:
            client-id: '${NAVER_OAUTH_CLIENT_ID:local-naver}'
            client-secret: '${NAVER_OAUTH_CLIENT_SECRET:local-naver}'
analysis:
  base-url: http://localhost:8000
jwt:
  secret-key: '<LOCAL_RANDOM_JWT_SECRET_AT_LEAST_32_BYTES>'
  access-token-validity-seconds: 3600
qaima:
  app:
    base-url: http://localhost:5173
  mail:
    from: noreply@example.com
    dry-run: true
  cors:
    allowed-origin-patterns: http://localhost:5173
auth:
  cookie:
    secure: false
  oauth2:
    success-redirect-url: http://localhost:5173/login/oauth2/success
    failure-redirect-url: http://localhost:5173/login
kis:
  base-url: https://openapi.koreainvestment.com:9443
  app-key: '${KIS_APP_KEY:}'
  app-secret: '${KIS_APP_SECRET:}'
marketstack:
  base-url: https://api.marketstack.com/v2
  access-key: '${MARKETSTACK_ACCESS_KEY:}'
news:
  naver:
    base-url: https://openapi.naver.com/v1/search/news.json
    client-id: '${NAVER_NEWS_CLIENT_ID:}'
    client-secret: '${NAVER_NEWS_CLIENT_SECRET:}'
bok:
  api-key: '${BOK_API_KEY:}'
fred:
  api-key: '${FRED_API_KEY:}'
opendart:
  api-key: '${OPENDART_API_KEY:}'
  sync:
    enabled: false
finra:
  sync:
    enabled: false
krx:
  short-selling:
    sync:
      enabled: false
sec:
  issued-shares:
    sync:
      enabled: false
kis-batch:
  industry-index-ohlcv:
    enabled: false
  market-investor-flow:
    enabled: false
  stock-investor-flow:
    enabled: false
analysis-report:
  cleanup:
    enabled: false
batch:
  news-export:
    enabled: false
```

빈 외부 API 키와 OAuth의 `local-*` 값은 **서비스 사용 자격증명이 아닙니다**. 초기 기동과 DB·API 연결을 확인한 뒤 사용할 제공처의 키를 설정합니다. OAuth 기본값은 실제 로그인을 수행할 수 없으며, 메일 `dry-run: true`에서는 인증 메일이 발송되지 않습니다.

현재 기본 설정에는 자동 수집 스케줄러가 켜져 있습니다. 예시는 데이터 준비 순서를 직접 확인할 수 있도록 각 스케줄러와 뉴스 export를 끕니다. 리포트 정리 스케줄러를 꺼도 조회 서비스 자체의 오래된 리포트 정리 로직까지 비활성화되는 것은 아닙니다.

`application.yml`은 `application-secret.yml`을 선택적으로 읽습니다. 기존 개인 설정 파일이 있는 작업 폴더라면 그 내용도 적용될 수 있으므로 위 로컬 프로필과 충돌하는 설정을 확인합니다. Spring은 Python처럼 `.env`를 자동으로 읽지 않습니다. `${...}`에 넣을 값은 **Spring을 실행할 PowerShell의 환경변수** 또는 로컬 YAML에서 제공합니다.

### 기능에 따라 추가할 설정

| 기능 | 설정할 값·조건 |
|---|---|
| KIS 가격·종목·수급 수집 | `KIS_APP_KEY`, `KIS_APP_SECRET`; 사용하는 API를 지원하는 계정·키 |
| Marketstack 조회 | `MARKETSTACK_ACCESS_KEY` |
| 뉴스 검색 | `NAVER_NEWS_CLIENT_ID`, `NAVER_NEWS_CLIENT_SECRET` |
| 국내외 거시 지표 | `BOK_API_KEY`, `FRED_API_KEY` |
| OpenDART | `OPENDART_API_KEY` |
| SEC 수집 | `sec.edgar.user-agent`에 제공처가 요구하는 식별·연락 정보 설정 |
| 실제 이메일 인증·재설정 | `spring.mail.*`, `qaima.mail.from`, `qaima.mail.dry-run: false`와 SMTP 제공처에 맞는 인증·TLS |
| Google·Kakao·Naver 로그인 | 위 OAuth 환경변수와 제공처별 redirect URI 등록 |

이 문서의 로컬 프로필은 **Naver 뉴스와 OAuth의 키 이름을 분리해 명시**했습니다. 기본 `application.yml`의 Naver 변수명과 혼용하지 않습니다. OAuth 콜백은 기본 설정상 `http://localhost:8080/login/oauth2/code/{registrationId}`이며 `registrationId`는 `google`, `kakao`, `naver`입니다.

현재 Vite는 `/api`만 프록시하고, Spring의 OAuth 진입 API는 `/oauth2/authorization/{provider}`로 상대경로 리다이렉트합니다. 따라서 프론트의 소셜 버튼만으로 개발 프록시를 통한 전체 로그인이 연결된다고 가정하면 안 됩니다. 제공처 설정 후 브라우저에서 `http://localhost:8080/api/v1/auth/oauth2/google`처럼 **Spring 주소로 직접 진입**해 로그인·콜백을 확인할 수 있습니다. 프론트 버튼 경유 동작에는 `/oauth2` 경로의 전달 구성도 필요하며, 현재 실행 안내만으로 그 경로가 추가되지는 않습니다.

## 4. 분석 서버 설정과 모델 준비

WSL의 `analysis/.env`에 분석 서버 설정을 둡니다. 아래 `SPRING_BASE_URL`은 **WSL에서 Windows Spring에 접근할 주소**입니다.

```dotenv
SPRING_BASE_URL=http://localhost:8080
LLM_VENDOR=gemini
NEWS_SENTIMENT_MODEL_PATH=app/models/kf_deberta_sentiment_v2
```

설명을 생성할 때는 선택한 제공처에 따라 `GEMINI_API_KEY` 또는 `OPENAI_API_KEY`를 추가합니다. 모델을 지정하려면 각각 `GEMINI_MODEL`, `OPENAI_MODEL`을 사용합니다. 키가 없는 경우 설명 경로에 오류·대체 설명·경고가 생길 수 있으며, 모든 기능이 동일하게 정상 동작하는 무키 모드는 아닙니다.

뉴스 모델의 기본 위치는 `analysis/app/models/kf_deberta_sentiment_v2`입니다. 같은 학습 결과에서 나온 다음 파일들을 준비합니다.

```text
kf_deberta_sentiment_v2/
├── config.json
├── model.safetensors
├── tokenizer.json
└── tokenizer_config.json
```

현재 로컬 작업 폴더에는 약 744MB의 `model.safetensors`가 있지만 Git 추적 대상은 아닙니다. 새로 clone했다고 가중치까지 내려받았다고 가정하면 안 됩니다. 별도로 보관한 학습 산출물을 복사하거나 [sentiment_lab의 학습 절차](../sentiment_lab/README.md)를 따라 생성해야 합니다. 저장소에는 이 가중치의 공개 다운로드 주소가 없습니다.

로더는 `local_files_only=True`로 읽고 CPU에서 추론합니다. 임의의 기반 모델 가중치만 넣는 것으로 감성 분류 모델이 준비되지는 않습니다. 라벨 순서도 코드의 `negative / neutral / positive`와 맞아야 합니다. `NEWS_SENTIMENT_MODEL_PATH`에 WSL에서 읽을 수 있는 절대 경로를 지정해 다른 위치의 모델을 사용할 수도 있습니다.

모델 없이 서버의 연결부터 확인할 때는 `QAIMA_DISABLE_NEWS_WARMUP=1`을 사용할 수 있습니다. 이 값은 시작 시 warm-up만 생략하며 뉴스 감성 기능을 대체하지 않습니다. 모델 준비 후 변수를 해제하고 서버를 다시 시작합니다.

## 5. Windows와 WSL 사이의 주소 확인

두 방향의 주소가 모두 맞아야 합니다.

| 방향 | 설정 | 확인할 요청 |
|---|---|---|
| Windows Spring → WSL FastAPI | 로컬 YAML의 `analysis.base-url` | FastAPI `GET /health` |
| WSL FastAPI → Windows Spring | `analysis/.env`의 `SPRING_BASE_URL` | Spring `GET /api/v1/test/ping` |
| Windows 브라우저 → Spring | Vite `/api` 프록시 | 프록시 경유 `GET /api/v1/test/ping` |

WSL 네트워크 설정에 따라 양쪽 `localhost`가 통할 수도 있고 별도 주소가 필요할 수도 있습니다. `localhost` 요청이 실패하면 WSL에서 아래 명령으로 주소 후보를 확인합니다.

```bash
hostname -I
ip route show default
```

Windows→WSL에는 접근 가능한 WSL 주소, WSL→Windows에는 Windows 호스트 주소를 사용합니다. 기본 게이트웨이는 NAT 구성에서의 후보이며, 모든 WSL 구성에서 Windows 주소가 되는 것은 아닙니다. 실제 요청이 성공하는지 확인하고 방화벽은 필요한 로컬 연결 범위에 맞춥니다.

브라우저 API 클라이언트는 `/api/v1`, Vite 개발 프록시는 `http://localhost:8080`을 사용합니다. 현재 코드에는 API 주소를 바꾸는 `VITE_API_URL` 설정이 없습니다. 프론트 `.env`에 FastAPI 주소를 넣어 연결하는 구조가 아닙니다.

## 6. 서버 실행과 스키마 생성

### 6.1. FastAPI — WSL

```bash
cd /mnt/c/qaima/analysis
../.venv_wsl/bin/python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### 6.2. Spring — Windows PowerShell

```powershell
cd C:\qaima\backend
$env:SPRING_CONFIG_ADDITIONAL_LOCATION = "file:C:/qaima-local/"
.\gradlew.bat bootRun --args="--spring.profiles.active=mysql,local"
```

첫 실행에는 Gradle과 의존성을 내려받습니다. `local` 프로필의 MySQL 접속과 `spring.flyway.enabled: true`가 적용되면 Spring 기동 중 Flyway가 마이그레이션을 수행합니다. 현재 스키마 이력은 V1~V51이며, JPA는 그 결과를 `validate`합니다. 별도 Flyway Gradle 플러그인은 선언되어 있지 않으므로 `gradlew flywayMigrate`를 선행 명령으로 사용하지 않습니다.

Spring 로그에서 DB 접속·Flyway·JPA 검증과 서버 시작을 확인합니다. 기존 테이블과 마이그레이션 이력이 맞지 않으면, 이 문서에서는 새 빈 개발 DB로 시작합니다. 스키마 생성은 전체 시장 데이터·사용자·모델 초기화와 별개입니다.

### 6.3. 프론트 — 새 Windows PowerShell

```powershell
cd C:\qaima\frontend
npm ci
npm run dev -- --port 5173 --strictPort
```

`http://localhost:5173`에 접속합니다. `--strictPort`는 5173이 이미 사용 중일 때 자동으로 다른 포트를 골라 OAuth·CORS 설정과 어긋나는 것을 막습니다.

### 6.4. 연결 확인

새 Windows PowerShell:

```powershell
Invoke-RestMethod http://localhost:8000/health
Invoke-RestMethod http://localhost:8080/api/v1/test/ping
Invoke-RestMethod http://localhost:5173/api/v1/test/ping
```

WSL의 별도 터미널:

```bash
curl --fail http://localhost:8080/api/v1/test/ping
```

앞서 별도 호스트 주소를 설정했다면 요청 주소도 바꿉니다. FastAPI는 `status: ok`, Spring은 `data.ok: true`가 기대값입니다. 이 요청은 서버·프록시 연결 확인이며, 금융 데이터·모델·메일·LLM의 가용성까지 검사하지 않습니다.

API 명세는 `http://localhost:8080/swagger-ui/index.html`, FastAPI 명세는 `http://localhost:8000/docs`에서 볼 수 있습니다. 각 서버는 실행한 터미널의 `Ctrl+C`로 종료합니다.

## 7. 초기 데이터 준비

Flyway는 거래소·일부 지수·달력·매핑을 포함하지만, 종목별 가격·재무·뉴스와 용어 사전 전체를 채우지는 않습니다. 현재는 데이터 종류별로 적재 경로가 나뉘어 있습니다. 아래 작업은 새 개발 DB에 필요한 데이터를 넣는 절차입니다.

### 7.1. 종목과 가격

`batch/jobs/bootstrap_stock_from_kis.py`는 CSV의 종목 코드를 읽고 KIS 기본정보로 종목을 생성합니다. `load_price_ohlcv.py`는 해당 종목의 일봉을 가져옵니다.

배치를 실행하는 WSL 터미널에는 다음 값을 설정해야 합니다. Spring YAML의 DB 설정을 자동으로 읽지는 않습니다.

| 환경변수 | 값의 의미 |
|---|---|
| `DB_HOST`, `DB_PORT` | WSL에서 접근 가능한 개발 MySQL 주소·포트 |
| `DB_NAME`, `DB_USER`, `DB_PASSWORD` | 앞서 만든 개발 DB와 사용자 |
| `KIS_APP_KEY`, `KIS_APP_SECRET` | 수집에 사용할 본인의 KIS 자격증명 |
| `KIS_BASE_URL` | 키에 맞는 제공처 주소; 기본값은 실전 API 주소 |

값은 개인 환경변수 또는 배치가 읽는 로컬 `.env`로 제공합니다. `analysis/.env`에만 넣은 값이 프로젝트 루트의 배치 실행에도 적용된다고 가정하지 않습니다. WSL에서 MySQL에 접속할 경우 DB 사용자 host 권한도 그 연결을 허용해야 합니다.

프로젝트 루트에 `data/local_stock_seed.csv`를 만들고 확인할 종목 코드를 넣습니다. 아래는 입력 형식 예시입니다.

```csv
stock_code
005930
```

WSL, 프로젝트 루트에서:

```bash
.venv_wsl/bin/python batch/jobs/bootstrap_stock_from_kis.py --csv data/local_stock_seed.csv --skip-backfill
.venv_wsl/bin/python batch/jobs/load_price_ohlcv.py --stock-code 005930 --lookback-days 365
```

두 명령은 **실제 제공처 호출과 개발 DB 저장을 수행**합니다. 거래일과 제공처 응답에 따라 확보되는 데이터 길이는 달라집니다. `--lookback-days`는 조회 범위이며 그만큼의 행을 보장하지 않습니다. 먼저 한 종목의 이름·거래소·가격 기간을 확인한 뒤 필요한 범위를 추가합니다.

### 7.2. 용어 사전

`data/dictionary_v2.csv`가 있는지 확인합니다. 이 파일도 새 clone에 없을 수 있으므로 별도 보관한 CSV를 준비해야 합니다. 실행기는 `import-dictionary-v2` 프로필을 사용하며 기본은 dry-run입니다.

기존 Spring 서버를 종료한 다음, 앞서 로컬 설정 경로를 지정한 **Windows PowerShell의 backend 디렉터리**에서 실행합니다.

```powershell
.\gradlew.bat bootRun --args="--spring.profiles.active=mysql,local,import-dictionary-v2 ../data/dictionary_v2.csv"
```

검사 결과를 확인한 뒤 실제 개발 DB에 적용하려면:

```powershell
.\gradlew.bat bootRun --args="--spring.profiles.active=mysql,local,import-dictionary-v2 ../data/dictionary_v2.csv --apply"
```

이 실행기는 작업 완료 후 Spring context를 종료합니다. 이후 일반 `mysql,local` 프로필로 서버를 다시 시작하고 사전 화면에서 검색을 확인합니다. CSV 형식과 검증 방식은 [사전 import 구현](../backend/src/main/java/com/qaima/importer/DictionaryV2ImportService.java)을 기준으로 합니다.

### 7.3. 기능별로 더 필요한 데이터

| 기능 | 필요한 자료·적재 경로 |
|---|---|
| Feature 1 | 종목·OHLCV, 재무 CSV와 시장 스냅샷. 재무 적재는 `batch/jobs/import_financials.py` 또는 Spring `import-financial-csv` 경로 |
| Feature 2 | 종목의 산업 분류, 산업 지수·거시 지표·수급·공매도·뉴스. Python 배치와 Spring 관리자 동기화 API가 각각 담당 |
| Feature 3 | 종목별 충분한 공통 거래일의 가격, 시장 벤치마크, 무위험수익률 원천. 선택한 오버레이에는 Feature 1·2 원천도 필요 |
| Feature 4 | 용어 사전 CSV와 별칭 데이터 |

관리자 API는 관리자 권한이 필요합니다. 일반 회원가입으로 관리자 계정이 생성되지는 않습니다. 현재 저장소에는 관리자 계정과 전체 금융 데이터를 한 번에 준비하는 공개 bootstrap 절차가 없습니다. 이메일 회원가입 시에는 코드에서 **초기 크레딧 5**를 부여합니다. 이후 분석 화면의 예상 비용과 잔액, 원천 데이터 준비 상태를 확인합니다.

일반 회원가입을 확인하려면 SMTP를 설정하고 `qaima.mail.dry-run`을 해제한 뒤 이메일 인증을 진행합니다. OAuth는 제공처 등록을 마친 뒤 사용합니다. 모든 초기화 작업을 대신하는 운영 DB 복사나 일괄 SQL은 이 문서에 포함하지 않습니다.

배치 종류와 제약은 [batch 안내](../batch/README.md), 관리자 API 목록은 [OpenAPI](api/openapi.json), 데이터 구조는 [ERD](erd/README.md)를 참고합니다. 배치의 전체 대상 실패·rollback 처리에도 [미해결 QA 사례](../batch/tests/isolated-sql.md)가 있으므로 종료코드만 보지 말고 저장 결과도 확인합니다.

## 8. 자주 막히는 부분

| 증상 | 확인할 부분 |
|---|---|
| Spring에서 설정 placeholder 오류 | 로컬 YAML의 위치, `mysql,local` 프로필, 필수 `kis.*`·`marketstack.*` 키 존재 여부 |
| JWT 초기화 오류 | `jwt.secret-key`의 누락·길이, 예시 문자열을 실제 무작위 값으로 바꿨는지 |
| MySQL 연결·권한 오류 | Windows 기준 DB 주소, DB 사용자 host, 스키마 생성 권한 |
| Flyway·JPA 검증 오류 | 새 DB 여부, V1~V51 적용 상태, MySQL 사용 여부. H2를 이 안내의 대체 DB로 사용하지 않음 |
| FastAPI는 열리지만 Peer 실패 | WSL→Spring 역방향 주소인 `SPRING_BASE_URL`과 실제 요청 결과 |
| 뉴스 모델을 찾지 못함 | 모델 가중치 포함 여부, WSL 경로, 함께 배포한 tokenizer·config |
| 회원가입 이메일이 오지 않음 | `qaima.mail.dry-run`, SMTP 계정·TLS·발신자 설정 |
| OAuth 로그인 실패 | 실제 제공처 키, 등록한 콜백 URI, Spring의 성공·실패 프론트 주소 |
| 화면은 열리지만 분석이 비어 있음 | 종목·가격·벤치마크 등 원천 데이터와 크레딧, 응답 경고 |
| Windows Rollup native 모듈 누락 | Windows에서 Node 조건을 맞춰 `npm ci` 수행; WSL에서 설치한 `node_modules`를 공유하지 않음 |

## 9. 테스트

제품의 실패 회귀가 남아 있어 전체 테스트가 모두 통과하는 상태는 아닙니다. 과거 통과 수치는 [최신 누적 기록](../tests/QA_PROGRESS_2026-09-28.md)의 범위와 함께 확인합니다.

**문서 링크·설정값 노출 검사 — WSL, 프로젝트 루트**

```bash
.venv_wsl/bin/python -m pip install PyYAML
.venv_wsl/bin/python -B -m unittest discover -s tests -p test_documentation.py -v
```

기존 문서 검사는 로컬 Spring 설정에서 비교할 비밀값 후보를 읽습니다. 후보가 없는 새 clone에서는 해당 검사가 실패할 수 있습니다. 이를 통과시키려고 실제 비밀값을 저장소에 추가하지 않습니다.

**FastAPI 기본 회귀 — WSL, 프로젝트 루트**

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 PYTHONPATH=analysis \
  .venv_wsl/bin/python -B -m unittest discover -s analysis/tests -v
```

**프론트 계약 검사·타입 검사·빌드 — Windows PowerShell, frontend**

```powershell
node --test tests/contracts.test.cjs tests/auth-client.test.cjs
npx tsc -p tsconfig.app.json --noEmit --incremental false
npx tsc -p tsconfig.node.json --noEmit --incremental false
npm run build
```

**Spring 회귀 — Windows PowerShell, backend**

```powershell
.\gradlew.bat --no-daemon --project-cache-dir src/test/.runtime/project-cache -I src/test/verification.init.gradle test
```

Spring 기본 검사 전에는 [테스트 문서](../backend/src/test/README.md)의 opt-in 환경변수 해제·캐시 경로 준비를 적용합니다. 실제 인프라 검사는 각각 별도의 실행 조건이 있습니다. 브라우저 검사도 [프론트 테스트 문서](../frontend/tests/README.md)의 격리 빌드·Playwright·Chrome 준비가 필요합니다.

이번 안내에서 `health`와 기본 서버 실행을 확인하는 것, 외부 제공처를 대체한 테스트, 실제 금융 API·모델·메일을 포함한 전체 사용자 흐름은 서로 다른 검증 범위입니다.
