# 감성 실험실 테스트 기록

검증일: 2026-09-25. WSL Python의 순수 유틸 검증이며 기존 README의 Windows·Colab 학습 실행을 재현한 것은 아닙니다.

```bash
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=sentiment_lab \
  .venv_wsl/bin/python -B -m unittest discover -s sentiment_lab/tests -v
```

| 케이스 | 실행·fixture | 기대값 |
|---|---|---|
| 제목 fallback | title만 제공, 양끝 공백 포함 | TITLE·FOCUS·DETAIL 모두 정리한 title |
| 상세·focus 우선순위 | title+detail, title+focus+detail | focus 없음→detail, focus 있음→focus |
| 필수 컬럼 | text/label/extra, text만 | 정상, label 누락 ValueError |
| 평가값 | 정답 0,0,1,1,2,2; 예측 0,1,1,2,2,2 | 정확도4/6, macro-F1=(2/3+1/2+4/5)/3 |

결과: 4개 PASS. 혼동행렬로 직접 구한 F1과 비교했습니다. 모델 다운로드·학습·GPU·실제 뉴스 추론은 수행하지 않았고 데이터셋 정확도나 모델 성능을 새로 측정한 결과가 아닙니다.

미검증: split 재현성·누수·라벨 매핑 전체, oversampling의 train-only 적용, seed 고정, 실제 학습·저장·다시 로드, 운영 FastAPI 입력 포맷과의 호환성. 실험실의 TITLE/FOCUS/DETAIL 조합과 운영 분석의 FOCUS/DETAIL 조합은 코드상 다르므로 학습·운영 성능 일치로 간주하지 않습니다.
