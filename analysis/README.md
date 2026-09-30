# FastAPI 분석 서버

작성·검증일: 2026-09-25. WSL에서 uvicorn으로 실행합니다. 의존성 선언은 `requirements.txt`에 있으며 최소 버전 범위이므로 설치 버전이 고정되어 있지 않습니다.

## 실행

프로젝트 루트의 WSL 가상환경을 사용하는 예입니다.

```bash
cd analysis
../.venv_wsl/bin/python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

`SPRING_BASE_URL`은 WSL에서 Windows Spring에 접근할 수 있는 주소여야 합니다. LLM 제공처 설정은 선택한 provider에 맞게 구성합니다. 정상 기동 시 뉴스 감성 모델 warm-up을 시작하며, 검증용 프로세스에서는 `QAIMA_DISABLE_NEWS_WARMUP=1`로 비활성화할 수 있습니다. 모델 파일 경로와 메모리 조건은 별도 확인 대상입니다.

## API와 책임

| 경로 | 처리 |
|---|---|
| `GET /health` | 프로세스 응답 확인. DB·LLM 가용성을 증명하지 않음 |
| `POST /feature1/analysis` | 계약 변환 → 가격·재무 요약 → 지표 → 선택적 설명 |
| `POST /feature2/peer-cluster` | Spring 데이터 pack → 수익률·상관·시차·유사도 |
| `POST /feature2/news-sentiment` | focus text → 로컬 모델 확률 → 감성 점수 |
| `POST /feature2/analysis` | Spring에서 조립한 metrics의 설명 생성 |
| `POST /feature3/analysis` | 제공 시계열·벤치마크 → 수익·위험·최적화 → 설명 |

성공 분석은 payload 형태입니다. `main.py`의 HTTP·일반 예외 응답은 별도의 `meta/data/errors` 형태이므로 모든 응답이 payload-only라고 가정하면 안 됩니다.

## 계산 정책

Feature 1은 SMA로 초기화한 EMA, 모집단 분산을 사용하는 Bollinger band, Stochastic을 계산합니다. Feature 3은 공통 거래일의 로그수익률을 사용하며 기본 연율화 계수는 252입니다. 가격·벤치마크·설명 실패는 각 경로별로 확인해야 합니다.

현재 `_risky_max_weight(n)`은 1종목이면 1, 그 외 `max(0.4, min(0.75, 2/n))`입니다. 40%가 모든 종목 수에 적용되는 고정 상한은 아닙니다. 위험회피계수는 명시 입력이 없으면 `10 - 9 * risk_tolerance_score`입니다. 뉴스 감성 점수는 `(positive_prob - negative_prob) * (1 - neutral_prob)`입니다. 계산 정책의 타당성·모델 정확도는 단위 테스트 통과와 구분합니다.

## 검증 상태

별도 Windows Spring client→WSL uvicorn **실제 TCP11개 PASS**를 추가했습니다. F1 계산/DTO·F2 빈키 경고와422·F3 독립 공분산/제약/부분fallback·실제 로컬 감성 모델을 확인했고, F3 Public Controller→실제 TCP→보고서/환불 경계도 연결했습니다. 외부연결/.env 읽기 차단과 임시서버 종료를 검증했으며 DB·가격 제공처·실제 과금/보고서 저장은 제외했습니다. [재현 명령과 상세 범위](../backend/src/test/cross-os-fastapi.md).

2026-09-25 WSL 최신 기본 unittest는98개 중92 PASS·3개 메서드 FAIL·3 SKIP입니다(실패subcase는5개). 올바른 `PYTHON_DOTENV_DISABLED=1`로 재실행한 결과이며 Feature2 설명비활성화 플래그 무시·제공자 오류문구 응답포함을 재현했고 서비스는 미수정입니다. Peer는 실제어댑터/계산·전송mock, 뉴스감성은 서비스/로더/ASGI21개·모델경계mock을 검증했습니다. 실제 로컬 모델3개도 별도 opt-in 실행에서 PASS했습니다(오프라인CPU·확률/점수·길이제한·단건/배치·ASGI응답). 위 TCP11개가 전체 Spring·DB·브라우저의 종단간 흐름이나 모델 정확도까지 입증한 것은 아닙니다. 같은 송신 차단을 적용해 재실행한 WSL uvicorn smoke10개도 PASS입니다.

[테스트 상세](tests/README.md), [전체 기능 흐름](../docs/features.md), [검증 현황](../docs/verification.md)을 참조합니다.
