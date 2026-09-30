# Feature3 최적화 추가 QA: 독립 효용 계산과 경계해 열거

검증일: 2026-09-25 (Asia/Seoul). 현재 작업 트리의 `analysis/app/services/portfolio.py`를 읽어 실행했습니다. 제품 코드·테스트 코드·DB는 수정하지 않았습니다. 아래 Python은 표준입력에서만 실행했고 `PYTHONDONTWRITEBYTECODE=1`, `PYTHON_DOTENV_DISABLED=1`을 설정했습니다. 외부 HTTP, 저장소, 유료 제공처 호출은 없습니다.

## 기준과 결과

`reference.md`의 동일 제약하에서 `UTILITY_OPTIMAL`의 효용이 `RISK_ALLOCATION`보다 낮지 않아야 한다는 조건과, 직접 최대화한 효용의 전역 최적성을 검사했습니다. 효용은 제품의 결과 필드를 사용하지 않고 입력 기대수익률·무위험수익률·연율 공분산에서 `μᵀw - γ/2 × wᵀΣw`로 별도 계산했습니다. 현금의 공분산은 0입니다.

| 검사 | 입력 | 기대·관측 |
|---|---|---|
| 위험배분 대비 효용·제약 | NumPy seed `20260925`, 240개 시나리오, 자산 1~8개, 양의 준정부호 공분산, 기대수익률 -15~35%, γ 1/2/5/10, 현금상한 0/0.1/0.2/0.5/0.8/1 | 효용 차이 허용오차 `-1e-6`, 두 결과의 비중 합·비음수·종목별 상한·현금 상한 검사. 실패 0개, 최소 차이 `-1.5355772209346696e-13` |
| 독립 경계해 열거 | 같은 seed로 새 RNG, 80개 시나리오, 자산 1~4개, 기대수익률 -20~35%, γ 1~10 | 각 자산과 현금의 하한·상한·자유 상태를 모두 열거하고, 자유 변수는 합계 1의 라그랑주 연립방정식으로 계산. 유효 후보 중 최대 효용과 구현 반환값 비교. 허용오차 `1e-6`, 실패 0개, 최대 최적성 차이 `3.187172747942668e-13` |

첫 검사는 기존 고정 fixture 9조합보다 자산 수·상관·현금 한도를 넓혔습니다. 두 번째 검사는 구현의 SciPy SLSQP 결과를 다시 SLSQP로 검증하지 않고, 작은 차원의 박스 제약과 비중 합 제약에서 가능한 경계면을 전부 확인합니다. 재현 명령은 프로젝트 루트에서 아래와 같습니다.

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHON_DOTENV_DISABLED=1 .venv_wsl/bin/python - <<'PY'
import itertools
import sys
sys.path.insert(0, 'analysis')
import numpy as np
from app.services import portfolio as p

rng = np.random.default_rng(20260925)
bad = 0
minimum_gap = float('inf')
for case in range(240):
    n = 1 + case % 8
    a = rng.normal(size=(n, n))
    v = a @ a.T
    scale = rng.uniform(.1, .6, size=n) / np.sqrt(np.diag(v))
    cov = v * np.outer(scale, scale)
    mu = rng.uniform(-.15, .35, size=n)
    rf = rng.uniform(0, .06)
    gamma = rng.choice([1., 2., 5., 10.])
    cash_cap = rng.choice([0., .1, .2, .5, .8, 1.])
    risk = p._user_risk_allocation_weights(
        p._max_sharpe_weights(mu, cov, rf, n), mu, cov, rf, gamma, cash_cap)
    opt = p._utility_optimal_weights(mu, cov, rf, gamma, cash_cap, n)
    def utility(w):
        return float(mu @ w[:n] + rf * w[-1] - gamma / 2 * (w[:n] @ cov @ w[:n]))
    gap = utility(opt) - utility(risk)
    minimum_gap = min(minimum_gap, gap)
    bad += int(gap < -1e-6 or
               not p._weights_satisfy_constraints(opt, n, cash_cap) or
               not p._weights_satisfy_constraints(risk, n, cash_cap))
print('comparison', 240, bad, minimum_gap)

rng = np.random.default_rng(20260925)
bad = 0
maximum_gap = 0.
for case in range(80):
    n = 1 + case % 4
    a = rng.normal(size=(n, n))
    v = a @ a.T
    scale = rng.uniform(.08, .55, size=n) / np.sqrt(np.diag(v))
    cov = v * np.outer(scale, scale)
    mu = rng.uniform(-.2, .35, size=n)
    rf = float(rng.uniform(0, .06))
    gamma = float(rng.uniform(1, 10))
    cash_cap = float(rng.choice([0, .1, .2, .5, .8, 1]))
    caps = np.array([p._risky_max_weight(n)] * n + [cash_cap])
    m = np.r_[mu, rf]
    q = np.zeros((n + 1, n + 1))
    q[:n, :n] = cov
    def utility(w):
        return float(m @ w - gamma / 2 * w @ q @ w)
    best = -float('inf')
    for state in itertools.product((0, 1, 2), repeat=n + 1):
        fixed = np.array([i for i, s in enumerate(state) if s != 2], dtype=int)
        free = np.array([i for i, s in enumerate(state) if s == 2], dtype=int)
        w = np.zeros(n + 1)
        if len(fixed):
            w[fixed] = [caps[i] if state[i] == 1 else 0 for i in fixed]
        remaining = 1 - w.sum()
        if len(free):
            lhs = np.block([
                [gamma * q[np.ix_(free, free)], np.ones((len(free), 1))],
                [np.ones((1, len(free))), np.zeros((1, 1))],
            ])
            rhs = np.r_[m[free] - gamma * q[np.ix_(free, fixed)] @ w[fixed], remaining]
            solution = np.linalg.lstsq(lhs, rhs, rcond=None)[0]
            if np.linalg.norm(lhs @ solution - rhs) > 1e-7:
                continue
            w[free] = solution[:-1]
        elif abs(remaining) > 1e-7:
            continue
        if abs(w.sum() - 1) > 1e-7 or min(w) < -1e-7 or max(w - caps) > 1e-7:
            continue
        best = max(best, utility(w))
    actual = p._utility_optimal_weights(mu, cov, rf, gamma, cash_cap, n)
    gap = best - utility(actual)
    maximum_gap = max(maximum_gap, gap)
    bad += int(gap > 1e-6 or not p._weights_satisfy_constraints(actual, n, cash_cap))
print('enumeration', 80, bad, maximum_gap)
PY
```

이 결과는 위 합성 입력의 내부 최적화 함수에 한정됩니다. 실제 시장 자료·CAPM 추정 품질·FastAPI 전체 요청·Spring/브라우저 연결·종목 5개 이상의 독립 전역해 비교는 증명하지 않습니다. 입력 난수와 임계값은 의도된 정책의 충분성을 판단하는 자료도 아닙니다.
