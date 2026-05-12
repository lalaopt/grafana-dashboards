# vLLM 대시보드

> 두 개의 대시보드로 구성. Aggregation(전체 집계)과 Per-Model(모델별 그리드).

## 파일

| 파일 | UID | 용도 |
|------|-----|------|
| `vllm-overview.json` | `vllm-overview` | 클러스터 전반 SLO·시스템·KV cache·워크로드 집계 |
| `vllm-per-model.json` | `vllm-per-model` | 배포된 모든 모델을 가속기별 그리드로 나열 |

두 대시보드는 상단 dashboard link 로 연결되어 있고 variable(`accelerator`, `model`)이 carry-over 된다.

---

## 전제 조건 — ServiceMonitor `relabelings` 추가

vLLM `/metrics` 가 자체적으로 노출하는 라벨은 `model_name`(전체 파일 경로) 뿐이다. 본 대시보드는 K8s 라벨에서 추출한 `model`(짧은 ID), `accelerator` 라벨이 메트릭에 부착돼 있다고 가정한다. `benchmark-server.py:_k8s_create_service_monitor()` 의 endpoints 에 다음을 추가해야 한다.

```yaml
endpoints:
- port: http
  path: /metrics
  interval: 15s
  relabelings:
  - sourceLabels: [__meta_kubernetes_service_label_benchmark_ai_model]
    targetLabel: model
  - sourceLabels: [__meta_kubernetes_service_label_benchmark_ai_accelerator]
    targetLabel: accelerator
```

이미 떠 있는 서버는 ServiceMonitor 재생성 또는 `kubectl patch` 필요. 신규 배포부터 자동 적용.

검증:
```bash
curl -s 'http://<prometheus>/api/v1/series?match[]=vllm:num_requests_running' | jq
# → __name__, model, accelerator, instance, job, pod 등 라벨 확인
```

---

## Import 방법

Grafana UI → Dashboards → Import → JSON 업로드 → Datasource 선택(Prometheus). 두 파일을 각각 import.

---

## Variable 설계

### `vllm-overview.json`
| 변수 | 동작 |
|------|------|
| `DS_PROMETHEUS` | Datasource 선택 |
| `accelerator` (multi, All) | 모든 패널 PromQL 의 `{accelerator=~"$accelerator"}` 필터 |
| `model` (multi, All, depends on `$accelerator`) | `{model=~"$model"}` 필터 |

기본 `All` 선택 시 전체 집계. 특정 가속기/모델로 좁히고 싶을 때 토글.

### `vllm-per-model.json`
| 변수 | 동작 |
|------|------|
| `DS_PROMETHEUS` | Datasource 선택 |
| `model_a100` (multi, All, **hidden**) | A100 row 의 panel repeat 소스 |
| `model_h100` (multi, All, **hidden**) | H100 row 의 panel repeat 소스 |
| `model_v100` (multi, All, **hidden**) | V100 row 의 panel repeat 소스 |

가속기당 model 변수를 분리한 이유는 Grafana 의 row repeat × panel repeat 중첩 시 nested variable scope 가 동작하지 않기 때문. 가속기는 잘 늘어나지 않으므로 명시적 분리가 유지보수 부담 적음.

---

## 가속기 추가 시 (예: B200)

1. `benchmark-server.py` 의 `_k8s_create_service_monitor` 는 그대로 (relabelings 가 K8s 라벨을 동적으로 매핑).
2. `vllm-per-model.json` 에 변수 `model_b200` 추가:
   ```json
   { "name": "model_b200", "query": "label_values(vllm:num_requests_running{accelerator=\"b200\"}, model)", "hide": 2, "multi": true, "includeAll": true, ... }
   ```
3. 4개 카테고리 row 에 B200 sub-row 4개 추가 (기존 A100/H100/V100 row 의 panel `repeat`/필터를 `b200` 으로 변경한 복사본).

---

## Overview 대시보드 — 패널 선택 근거

| Row | 패널 | 시각화 | 이유 |
|-----|------|--------|------|
| ① SLO | TTFT (p50/p95/p99) | timeseries (3-line) | 사용자 체감 지연. avg 대신 분위수로 tail latency 가시화. |
| ① SLO | TPOT (p50/p95/p99) | timeseries (3-line) | 스트리밍 시 토큰 간 지연. tail 이 사용자 체감 가장 직접 영향. |
| ① SLO | E2E Latency (p50/p95/p99) | timeseries (3-line) | 전체 요청 처리 시간. 단순 비교 KPI. |
| ① SLO | Token Throughput | timeseries (2-line) | prompt vs gen 토큰 처리량을 동시에. 색 다르게. |
| ① SLO | Request Completion Rate | timeseries (stacked) | finished_reason(stop/length/abort) 비율 시간변화. abort 가 급증하면 즉시 인지. |
| ② System | Concurrency | timeseries (stacked area) | running + waiting 누적. waiting 이 크면 처리 한계. |
| ② System | KV Cache Usage | timeseries (avg/max) | avg 는 클러스터 압력, max 는 어느 모델이 한계 근접. |
| ② System | Queue Wait p95 | timeseries (single) | 큐 대기 자체가 SLO 침해 신호. |
| ② System | Prefill vs Decode p95 | timeseries (2-line) | 어느 단계가 병목인지 분리해서 본다. |
| ② System | Active Deployments | stat | 현재 운영 중인 모델×가속기 조합 수. 상황 파악용. |
| ③ KV cache | Prefix Cache Hit Rate | timeseries | 0~1 비율. PromQL 에서 rate(hits)/rate(queries) 로 계산. |
| ③ KV cache | KV Block Lifetime | heatmap | 분포가 핵심이라 heatmap. 짧은 lifetime 이 많으면 churn. |
| ③ KV cache | Idle Before Evict | heatmap | 같은 이유. 색 구분(Purples) 으로 lifetime 과 차별. |
| ③ KV cache | Reuse Gap | heatmap | 재사용 간격 분포로 캐시 효율 진단. |
| ④ Workload | Prompt Token Length | heatmap | 입력 길이 분포. histogram 메트릭이라 heatmap 자연스러움. |
| ④ Workload | Generation Token Length | heatmap | 출력 길이 분포. |
| ④ Workload | Finish Reason Ratio | donut | 종료 이유 비율을 한눈에. abort 가 보이면 위험 신호. |
| ④ Workload | Throughput Share by Accelerator | donut | 가속기 간 부하 분산 비율. |

---

## Per-Model 대시보드 — 패널 선택 근거

각 카테고리에 **1개 핵심 지표** 만 골라 그리드로 펼쳤다. 모델 수십 개를 동시에 비교할 때 패널 1개당 1개 지표가 인지 부담이 가장 낮다. 상세 분포·다중 분위수는 Overview 에서 본다.

| Row | 패널 | 시각화 | 임계값 | 이유 |
|-----|------|--------|--------|------|
| ① SLO | TTFT p95 | stat + area sparkline | 1s 주의, 3s 위험 | p95 한 숫자 + 추세. 색으로 위험 모델 즉시 식별. |
| ② System | KV Cache % | gauge | 70% 주의, 90% 위험 | 0~1 범위. gauge 가 한계 시각적으로 직관. |
| ③ KV cache | Prefix Hit Rate | stat + area sparkline | <20% 위험(red), ≥50% 좋음(green) | 높을수록 좋음이라 임계값 색 반전. |
| ④ Workload | Gen Throughput | stat + area sparkline | (단색 blue) | 절대값 비교용. 임계값 없음. |

패널 규격: `gridPos.w=4, h=4`, `maxPerRow=6`, `repeatDirection=horizontal`. 24-wide 그리드에 6 panel/줄. 30개 모델이면 5줄.

모든 row 는 기본 collapsed. 초기 로드 시 panel query 가 0개이므로 빠르고, 사용자가 보고 싶은 카테고리만 펼침.

---

## 비교 동선

- **가속기 비교** (예: 같은 모델 A100 vs H100):
  - Overview 에서 `model` 변수로 모델 1개 선택 → 모든 패널이 그 모델만 보여줌. PromQL legend 가 가속기 label 을 포함하지 않으므로 합산됨. 가속기별로 보려면 패널을 복제하고 `sum by (accelerator)` 추가.
  - 또는 Per-Model 에서 두 가속기 row 를 동시에 펼쳐 시각 비교.
- **모델 비교** (이종 모델 다수):
  - Per-Model 에서 카테고리 row 펼치면 그리드 자체가 비교 뷰. 색 임계값으로 outlier 즉시 식별.

---

## 시간 범위·refresh

기본 `now-1h ~ now`, 30s refresh. 운영 상황 모니터링에 적합. 장기 회고는 Time picker 로 변경.

---

## 알려진 한계

- TP/quantization/KV dtype 별 비교는 본 대시보드 범위 밖. (현재 라벨로 노출 안 됨 — `docs/servicemonitor-labels.md` 참고)
- KV residency heatmap 패널은 vLLM 서버에 `--kv-cache-metrics-sample` 옵션이 켜져 있어야 데이터가 들어온다. 옵션이 없으면 빈 패널.
- 모델 수가 100개 초과로 가면 Per-Model 의 panel repeat 가 무거워질 수 있다. 그 시점에 row 를 더 세분화하거나 별도 dashboard 로 분리.
