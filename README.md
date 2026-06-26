# KPX 전력수요 단기부하예측 MLOps

KPX 전력수요 데이터를 기반으로, 향후 6시간의 수요를 예측하고 그 결과를 저장·평가·시각화하는 MLOps 파이프라인을 설계한 프로젝트입니다.

이 프로젝트에서 중점적으로 본 것은 단순히 예측 모델 하나를 학습시키는 일이 아니었습니다. 모델이 실제 운영 흐름 안에서 쓰이려면 데이터가 안전하게 관리되어야 하고, 어떤 모델이 예측에 사용되었는지 기록되어야 하며, 시간이 지난 뒤에는 실제값과 비교해 성능도 확인할 수 있어야 합니다. 그래서 데이터 정제, 모델 학습, artifact 관리, 추론, 지연 평가, 대시보드까지 이어지는 흐름을 Google Cloud 기반으로 직접 구성했습니다.

## 1. 프로젝트를 시작한 이유

전력수요는 시간대, 요일, 계절성에 따라 비교적 뚜렷한 패턴을 보입니다. 처음에는 현재 시점에서 15분 뒤 한 시점만 예측하는 방식으로 접근했습니다. 하지만 실제 운영 관점에서는 한 시점의 값만 아는 것보다, 앞으로 몇 시간 동안 수요가 어떻게 움직일지를 함께 보는 편이 더 유용하다고 판단했습니다.

그래서 예측 문제를 15분 뒤 단일 예측이 아니라, 향후 6시간을 한 번에 예측하는 24-step direct forecasting 문제로 다시 정의했습니다.

| 항목 | 내용 |
|---|---|
| 입력 | 최근 24시간의 15분 단위 수요 시퀀스 |
| 출력 | 향후 6시간, 15분 간격 24개 시점의 수요 |
| 예측 방식 | 24-step direct forecasting |
| 주요 목표 | 학습, 추론, 평가, 시각화를 연결한 운영형 파이프라인 구현 |

한 시점씩 반복해서 예측하면 앞에서 생긴 오차가 뒤쪽 예측으로 이어질 수 있습니다. 이 프로젝트에서는 그런 누적 오차를 줄이기 위해, 한 번의 추론에서 24개 시점을 동시에 예측하는 direct multi-step 방식을 사용했습니다.

## 2. 현재 상태

이 저장소는 완전히 운영 배포가 끝난 서비스라기보다, 전력수요 예측 모델을 운영 흐름에 올리기 위한 MLOps 구조를 설계하고 검증한 프로젝트에 가깝습니다. 구현한 부분과 앞으로 더 안정화해야 할 부분을 구분해서 정리했습니다.

완료한 작업은 다음과 같습니다.

- raw 데이터를 보호하기 위한 review 데이터셋 분리
- raw → silver → 15분 view 데이터 계층 구성
- 최근 96개 입력, 향후 24개 출력의 direct forecasting 문제 정의
- lag, rolling, ramp, calendar feature 설계
- Transformer 학습 파이프라인 실행 및 artifact 저장
- `model_registry_control` 기반 후보 모델 관리
- current 모델 단일성 gate 설계
- Cloud Run Job 기반 추론 구조 설계
- prediction, run log, evaluation schema 설계
- Looker Studio용 view 계층 설계
- Last-value baseline, LightGBM, LSTM, GRU, TCN, Transformer, N-HiTS 비교 실험

아직 남아 있는 부분도 있습니다. 고도화 모델의 current 전환 검증, Cloud Run inference 반복 안정화, 15분 자동화 장기 검증, 대시보드 개선, 모델 성능 튜닝은 후속 작업으로 남겨 두었습니다. 이 부분을 따로 구분해 둔 이유는, 프로젝트의 상태를 과장하지 않고 정확히 보여주는 것이 더 중요하다고 봤기 때문입니다.

## 3. 전체 구조

전체 데이터 흐름은 아래와 같습니다.

    KPX raw 5분 데이터
      → BigQuery raw / silver / 15분 view
      → sequence feature table
      → Vertex AI Pipelines 학습
      → GCS model artifacts
      → BigQuery model_registry_control
      → Cloud Run Job inference
      → predictions / prediction_run_log
      → delayed evaluation
      → Looker Studio dashboard

역할을 나누면 크게 네 계층입니다.

| 계층 | 역할 |
|---|---|
| 데이터 계층 | raw 데이터를 읽고 silver 및 15분 view로 정제 |
| 학습·모델 제어 계층 | Vertex AI Pipelines로 모델 학습, GCS에 artifact 저장, current 모델 관리 |
| 추론 계층 | Cloud Run Job으로 current 모델을 불러와 24-step 예측 생성 |
| 평가·시각화 계층 | 실제값 도착 후 WMAPE, MAPE, MAE를 계산하고 Looker Studio에서 확인 |

## 4. 기술 스택

| 구분 | 사용 기술 |
|---|---|
| Cloud | Google Cloud Platform |
| Data Warehouse | BigQuery |
| Training | Vertex AI Pipelines |
| Artifact Storage | Cloud Storage |
| Inference | Cloud Run Jobs |
| Scheduling | Cloud Scheduler, BigQuery Scheduled Query |
| Dashboard | Looker Studio |
| Models | Last-value baseline, LightGBM, LSTM, GRU, TCN, Transformer, N-HiTS |
| Metrics | MAE, RMSE, MAPE, WMAPE |

## 5. 데이터 파이프라인

운영 raw 데이터를 직접 수정하지 않기 위해 원천 데이터와 실험용 데이터 계층을 분리했습니다. 이 프로젝트에서 가장 먼저 잡은 원칙은 “raw는 건드리지 않는다”였습니다.

- `kpx_power.raw_kpx_5m`: KPX 5분 단위 원천 데이터
- `kpx_power_review.silver_demand_5m`: 같은 시각의 중복 row 중 최신 수집값만 유지
- `kpx_power_review.silver_demand_15m_v`: 15분 grid로 정렬한 예측 입력 view

raw 데이터는 읽기 전용으로 두고, 정제·학습·예측·평가 결과는 모두 `kpx_power_review` 계층에서 관리했습니다. 이렇게 나누면 실험을 반복하더라도 원본 데이터가 오염되지 않고, 문제가 생겼을 때 어느 단계에서 발생했는지도 비교적 쉽게 추적할 수 있습니다.

## 6. 예측 문제 정의

최종 예측 문제는 다음과 같이 정리했습니다.

| 항목 | 내용 |
|---|---|
| 입력 길이 | 최근 96개 15분 시점 |
| 입력 시간 범위 | 24시간 |
| 예측 길이 | 향후 24개 15분 시점 |
| 예측 시간 범위 | 6시간 |
| 예측 방식 | direct multi-step forecasting |
| Target | demand_mw |

이 구조에서는 한 번의 추론 결과가 24개의 horizon을 가집니다. 예를 들어 `horizon_min`은 15, 30, 45, ..., 360분까지 이어집니다. 덕분에 현재 시점에서 앞으로 6시간 동안의 수요 흐름을 하나의 예측 run으로 볼 수 있습니다.

## 7. Feature Engineering

초기에는 시간대, 요일, 주말 여부 같은 달력 feature를 중심으로 시작했습니다. 이후 모델 성능을 더 안정적으로 보기 위해 과거 수요 기반 feature를 추가했습니다.

사용한 feature는 다음과 같습니다.

- lag feature: 직전, 1시간 전, 24시간 전 수요
- rolling feature: 최근 구간 평균 및 표준편차
- ramp feature: 직전 대비 변화량
- calendar feature: hour, dayofweek, month, 15분 slot, weekend 여부
- cyclic encoding: hour, dayofweek, quarter slot의 sin/cos 변환

여기서 특히 신경 쓴 부분은 feature의 순서였습니다. 모델은 입력 배열의 몇 번째 위치에 어떤 값이 들어오는지를 기준으로 학습합니다. 학습할 때와 추론할 때 feature 순서가 달라지면, 값 자체가 맞더라도 모델 입장에서는 전혀 다른 입력이 됩니다.

이를 막기 위해 학습 시 사용한 feature 순서를 `feature_columns.json`에 저장했습니다. 추론 단계에서도 이 파일을 기준으로 같은 순서의 입력을 만들도록 고정했습니다.

또한 예측 시점 이후에야 알 수 있는 실제값이나, 그 실제값에서 파생된 값은 feature로 사용하지 않았습니다. 검증 성능이 실제보다 좋아 보이는 정보 누수를 막기 위한 기준입니다.

## 8. 모델 실험

처음부터 복잡한 모델을 운영 파이프라인에 바로 올리지는 않았습니다. 성능이 나쁘거나 실행이 실패했을 때, 그 원인이 모델인지 파이프라인인지 구분하기 어려워지기 때문입니다. 그래서 가장 단순한 기준선부터 시작해 점차 복잡한 모델로 확장했습니다.

| 모델 | 목적 |
|---|---|
| Last-value baseline | 수집 → 정제 → 예측 → 평가 루프 검증 |
| LightGBM | 트리 기반 예측 모델 비교 |
| LSTM / GRU / TCN | 시계열 딥러닝 baseline 비교 |
| Transformer | self-attention 기반 24-step 예측 후보 |
| N-HiTS | 장기 구간 예측 성능 개선 후보 |

오프라인 비교 실험에서는 Transformer가 평균 MAPE 2.01%로 가장 낮은 오차를 보여 운영 파이프라인의 우선 후보로 선정했습니다. 추가 실험에서는 N-HiTS 튜닝 모델도 Rolling MAPE 1.80% 수준의 좋은 결과를 보여 후속 개선 후보로 검토했습니다.

다만 오프라인 실험 성능이 좋다고 해서 바로 운영 current로 전환하지는 않았습니다. 실제 추론에 사용하려면 artifact 생성, feature contract 일치, inference image 검증, current 모델 제어까지 함께 통과해야 하기 때문입니다.

## 9. 모델 학습과 Artifact 관리

모델 학습은 Vertex AI Pipelines에서 수행했습니다. 학습이 끝난 모델은 GCS에 저장하고, BigQuery의 `model_registry_control` 테이블에 후보 모델로 등록했습니다.

모델 artifact에는 단순히 모델 가중치만 저장하지 않았습니다. 추론에 필요한 설정과 검증 정보를 함께 저장했습니다.

| Artifact | 역할 |
|---|---|
| `model.pt` | 모델 가중치 및 구조 정보 |
| `feature_columns.json` | 학습과 추론에서 사용할 feature 순서 |
| `model_config.json` | input window, horizon, 모델 구조 정보 |
| `preprocessing_config.json` | 정렬, 결측 처리, leakage 방지 규칙 |
| `metrics.json` | valid/test WMAPE 등 학습 성능 지표 |
| `artifact_manifest.json` | 필수 파일 목록과 검증 기준 |

추론을 시작하기 전에는 `artifact_manifest.json`을 기준으로 필수 파일이 모두 있는지 확인하도록 설계했습니다. 필수 파일이 누락되면 예측을 저장하지 않고 실패 로그를 남깁니다.

## 10. Model Registry Control

학습된 모델을 곧바로 운영에 사용하지 않고, 먼저 `CANDIDATE` 상태로 등록했습니다. 검증을 통과한 모델만 `ACTIVE/current` 상태로 전환하도록 했습니다.

가장 중요한 조건은 current 모델이 정확히 하나만 존재해야 한다는 점입니다.

    current_count = 1

이를 위해 다음 기준을 적용했습니다.

- current 모델이 0개이면 추론하지 않음
- current 모델이 2개 이상이어도 추론하지 않음
- current 전환은 transaction으로 처리
- `v_current_model_control`은 current가 정확히 1개일 때만 row 반환

이 구조를 통해 의도하지 않은 모델이 추론에 사용되거나, current 모델이 없어 추론이 실패하는 상황을 방지했습니다.

## 11. 추론 파이프라인

추론은 Vertex AI Pipeline을 15분마다 실행하는 방식이 아니라, Cloud Run Job으로 가볍게 실행하는 방식으로 구성했습니다.

    Cloud Run Job
      → current model load
      → recent sequence input load
      → 24-step prediction
      → BigQuery predictions MERGE
      → prediction_run_log 기록

예측 결과는 `predictions` 테이블에 저장했습니다. 같은 예측이 중복으로 쌓이지 않도록 다음 조합을 사실상의 key로 사용했습니다.

    prediction_time + horizon_min + model_version

BigQuery는 일반적인 primary key를 강제하지 않기 때문에, 저장 로직은 INSERT가 아니라 MERGE 방식으로 구현했습니다. 이렇게 하면 같은 추론을 여러 번 실행해도 중복 row가 쌓이지 않습니다.

또 한 가지 주의한 점은 inference image입니다. feature를 8개에서 22개 수준으로 확장하면, 기존 Cloud Run image가 새 feature를 만들지 못할 수 있습니다. 그래서 새 모델을 current로 올리기 전에 Cloud Run image를 먼저 갱신해야 합니다. 이 순서를 지키지 않으면 모델 artifact와 추론 입력이 맞지 않아 inference가 실패할 수 있습니다.

## 12. 자동화 설계

자동화는 두 가지가 함께 동작해야 합니다.

1. raw 데이터를 review silver로 갱신하는 BigQuery Scheduled Query
2. current 모델로 24-step 예측을 생성하는 Cloud Scheduler → Cloud Run Job

검증 단계에서는 Scheduler를 계속 켜 두지 않고 기본적으로 `PAUSED` 상태로 관리했습니다. 자동화를 무작정 켜 두면 비용이 늘거나 같은 작업이 중복 실행될 수 있기 때문입니다.

검증이 필요할 때는 아래 순서로 실행했습니다.

    resume
      → run now
      → pause

자동화는 단순히 스케줄을 켜는 것으로 끝나지 않습니다. raw → silver 갱신이 멈춰 있으면 같은 `prediction_time`이 반복될 수 있고, Cloud Run Job만 정상이어도 실제 예측 누적은 기대한 대로 보이지 않을 수 있습니다. 그래서 inference 자동화와 함께 silver freshness도 확인하도록 했습니다.

## 13. 평가 구조

예측값은 현재 시점에 만들 수 있지만, 그 예측에 대응하는 실제값은 시간이 지나야 도착합니다. 그래서 예측과 평가를 같은 실행 사이클에 묶지 않고 분리했습니다.

평가는 지연 평가 방식으로 설계했습니다.

- 예측 직후에는 평가하지 않음
- 마지막 target_time까지 실제값이 도착한 뒤 평가
- 24개 actual이 모두 붙은 run만 COMPLETE로 판단
- COMPLETE run에 대해서만 WMAPE, MAPE, MAE 계산

평가 상태는 다음과 같이 구분했습니다.

| 상태 | 의미 |
|---|---|
| COMPLETE | 24개 예측과 24개 실제값이 모두 존재 |
| PENDING | 아직 실제값이 모두 도착하지 않음 |
| ACTUAL_MISSING | 시간이 지났지만 일부 실제값이 누락됨 |
| PREDICTION_INCOMPLETE | 예측 자체가 24개 미만 |

이렇게 분리함으로써 실제값 지연 때문에 전체 파이프라인이 멈추는 상황을 방지했습니다. 또 24개 실제값이 모두 붙지 않은 상태에서 계산된 성능을 최종 성능처럼 보여주지 않도록 했습니다.

## 14. Looker Studio Dashboard

Looker Studio에서는 예측 결과와 평가 결과를 확인할 수 있도록 view 계층을 구성했습니다.

### Page 1. 최신 24-step 예측 곡선

가장 최근 실행된 예측 run을 기준으로 앞으로 6시간의 수요 흐름을 확인하는 화면입니다.

- X축: `horizon_min`
- Y축: `yhat`
- 목적: 최근 예측 곡선 확인

### Page 2. 완료된 run의 성능 추이

실제값이 모두 도착해 평가가 끝난 run만 대상으로 WMAPE, MAPE, MAE를 확인하는 화면입니다.

- COMPLETE run만 표시
- PENDING run은 성능 계산에서 제외
- WMAPE/MAPE는 BigQuery에서 계산된 값을 사용
- Looker에서는 SUM이 아니라 AVG로 집계

예측 직후 성능 화면이 비어 있는 것은 오류가 아닙니다. 6시간 뒤 실제값이 모두 도착해야 성능 평가가 가능하기 때문입니다.

## 15. 주요 결과

초기 Transformer 학습 파이프라인에서 확인한 주요 지표는 다음과 같습니다.

| 지표 | 값 |
|---|---:|
| train_sample_count | 6,889 |
| valid_sample_count | 1,517 |
| test_sample_count | 1,308 |
| train_wmape | 0.028759 |
| valid_wmape | 0.031660 |
| test_wmape | 0.039556 |
| input_window / horizon | 96 / 24 |
| max_epochs | 5 |

위 수치는 초기 baseline 학습 설정에서 얻은 결과입니다. 이 단계에서는 성능을 끝까지 끌어올리기보다, 데이터 정제 → 학습 → artifact 저장 → 모델 등록 → 추론 → 평가 → 시각화로 이어지는 전체 MLOps 골격이 제대로 동작하는지 확인하는 데 초점을 두었습니다.

## 16. 구현 과정에서 해결한 문제

| 문제 | 해결 방법 |
|---|---|
| raw 데이터 오염 위험 | raw는 읽기 전용, 모든 쓰기는 review dataset에서 수행 |
| current 모델이 0개 또는 2개 이상이 되는 문제 | `current_count=1` gate와 strict view 적용 |
| artifact 파일 누락 | `artifact_manifest.json` 기반 필수 파일 검증 |
| 학습/추론 feature 순서 불일치 | `feature_columns.json`으로 순서 고정 |
| feature 개수와 inference image 불일치 | 새 모델 전환 전 Cloud Run image 먼저 갱신 |
| prediction 중복 저장 | `prediction_time + horizon_min + model_version` 기준 MERGE |
| 실행 로그 누락 | `prediction_run_log`도 MERGE 방식으로 기록 |
| Looker 지표 왜곡 | run별 계산값은 SUM이 아니라 AVG로 집계 |
| 실제값 지연 도착 | predictor와 evaluator 분리, PENDING/COMPLETE 상태 구분 |
| 자동화 중복·비용 위험 | 기본 PAUSED, resume → run → pause 순서로 검증 |

## 17. 앞으로 개선할 부분

- Transformer와 N-HiTS를 동일 조건에서 재비교
- lag / rolling / ramp feature 확장 효과 분석
- input window와 horizon별 오차 분석
- max_epochs, d_model, dropout, patience 등 학습 설정 튜닝
- Cloud Run inference 반복 실행 안정화
- 15분 자동화 장기 검증
- drift detection 및 promotion policy 고도화
- Looker Studio dashboard UX 개선
- horizon별 오차와 모델별 성능 비교 화면 추가

## 18. Repository Structure

    .
    ├── README.md
    ├── docs/
    │   ├── architecture.md
    │   ├── data-pipeline.md
    │   ├── model-experiments.md
    │   ├── mlops-pipeline.md
    │   └── troubleshooting.md
    ├── sql/
    │   ├── 01_create_review_dataset.sql
    │   ├── 02_raw_to_silver_merge.sql
    │   ├── 03_create_15m_view.sql
    │   ├── 04_training_features.sql
    │   └── 05_evaluation_views.sql
    ├── src/
    │   ├── training/
    │   ├── inference/
    │   └── evaluation/
    ├── configs/
    │   ├── feature_columns.example.json
    │   ├── model_config.example.json
    │   └── env.example.yaml
    └── assets/
        ├── architecture.png
        ├── dashboard_forecast.png
        └── dashboard_evaluation.png

## 19. 공개 저장소 업로드 시 주의사항

공개 저장소에는 다음 정보를 포함하지 않습니다.

- API Key
- Secret 값
- 실제 서비스 계정 이메일
- 개인 이메일 및 전화번호
- GCP 프로젝트 ID
- GCS bucket 전체 경로
- 정적 IP
- 원본 데이터 전체
- 개인 계정이 포함된 권한 부여 명령

공개용 파일에는 실제 환경값 대신 `.example` 파일만 포함합니다. 원본 데이터와 민감한 설정 파일은 `.gitignore`에 포함해 저장소에 올라가지 않도록 관리합니다.

## 20. 배운 점

이번 프로젝트를 하면서 예측 모델을 만드는 일과, 그 모델을 실제 운영 가능한 흐름에 올리는 일이 다르다는 것을 많이 느꼈습니다.

모델 성능만 보면 실험은 끝난 것처럼 보일 수 있습니다. 하지만 실제 시스템에서는 데이터가 최신인지, 모델 artifact가 완전한지, feature 순서가 맞는지, current 모델이 하나만 존재하는지, 같은 예측이 중복 저장되지 않는지까지 확인해야 했습니다. 예측값에 대응하는 실제값이 늦게 도착하는 상황도 고려해야 했습니다.

BigQuery, Vertex AI Pipelines, Cloud Run, Cloud Scheduler, Looker Studio를 연결하면서 MLOps는 모델 코드만의 문제가 아니라는 점을 배웠습니다. 데이터 관리, 권한, 배포, 평가, 시각화가 함께 맞아야 비로소 운영 가능한 시스템에 가까워진다는 것을 직접 확인한 프로젝트였습니다.
