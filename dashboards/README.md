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
| `groupby` (custom, single) | 시리즈 분리 키 — `(total)` / `accelerator` / `model + accelerator`. **model 단독은 같은 모델이 여러 가속기에 떠 있을 때 라인이 합쳐져 모호하므로 제거**. |
| `percentile` (custom, single) | latency 패널 5개가 표시할 분위수 — `p50` / `p95`(default) / `p99`. 한 번에 한 분위수만 보여 group-by 와 결합 시 라인 폭발 방지. |

기본 `All` 선택 시 전체 집계. 특정 가속기/모델로 좁히고 싶을 때 토글.

**groupby 동작:** Overview 의 timeseries/heatmap/piechart 13개 패널 중 12개가 영향을 받는다. 예외는 "Throughput Share by Accelerator" donut(자체 accelerator 분리), "Active Deployments" stat(그룹 의미 약함).

- Histogram (timeseries): `sum by (le${groupby:raw}) (rate(..._bucket[5m]))` — `le` 는 분위수 계산에 필수, group-by 라벨이 거기에 추가됨. TTFT/TPOT/E2E/Queue Wait/Prefill+Decode 5개.
- Counter/Instant (timeseries): `sum by (__name__${groupby:raw}) (...)` — `__name__` 는 메트릭당 동일하므로 합산 결과 단일 시리즈 보존용 dummy. group-by 라벨이 거기에 추가되면 그 라벨별로 시리즈 분리. Throughput/Completion/Concurrency/KV Usage/Prefix Hit Rate 5개.
- Heatmap (workload 분포): `sum by (le${groupby:raw}) (...)` — (total) 이 아닐 때 multi-series heatmap 이 되어 Grafana 가 시리즈별 row 분리 시각화. 시각이 복잡해질 수 있어 `(total)` 권장.
- Piechart (Finish Reason): `sum by (finished_reason${groupby:raw}) (...)` — group-by 시 slice 수 (finished_reason × 그룹) 곱 증가.

variable value 인코딩: `(total)` → `""`, `accelerator` → `", accelerator"`, `model + accelerator` → `", model, accelerator"`. PromQL 표현식에 그대로 삽입되므로 trailing/empty grouping 문법 오류 없음.

**`model + accelerator` 선택 시 왜 둘 다 분리하나:** 같은 모델 ID 가 여러 가속기에 떠 있을 때 (예: `llama32-1b` 가 a100/h100/v100 셋에) `sum by (model)` 만 하면 가속기 라벨이 제거되어 데이터가 한 줄로 합쳐진다. `(model, accelerator)` 둘 다 보존해야 인스턴스가 분리되어 비교 가능.

**percentile 동작:** latency 패널 5개(TTFT/TPOT/E2E/Queue Wait/Prefill+Decode)는 한 번에 한 분위수만 표시. `histogram_quantile(${percentile:raw}, ...)`. 분위수 비교가 필요하면 variable 토글. 한 번에 모든 분위수 + 모든 그룹을 보면 라인 수가 폭발(예: 3 percentile × 20 model+accelerator = 60 라인)하기 때문에 의도적으로 단일 분위수만 노출.

### `vllm-per-model.json`
| 변수 | 동작 |
|------|------|
| `DS_PROMETHEUS` | Datasource 선택 |
| `model_a100` (multi, All, **hidden**) | A100 row 의 panel repeat 소스 |
| `model_h100` (multi, All, **hidden**) | H100 row 의 panel repeat 소스 |
| `model_v100` (multi, All, **hidden**) | V100 row 의 panel repeat 소스 |
| `percentile` (custom, single) | SLO Latency · Prefill vs Decode 패널이 표시할 분위수 |

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
| ① SLO | TTFT (선택 분위수) | timeseries (group-by + percentile 토글) | 사용자 체감 첫 토큰 지연. avg 대신 분위수로 tail latency 가시화. 분위수 1개 × 그룹 라인 — 라인 폭발 방지. |
| ① SLO | TPOT (선택 분위수) | timeseries (group-by + percentile 토글) | 스트리밍 시 토큰 간 지연. tail 이 사용자 체감 가장 직접 영향. |
| ① SLO | E2E Latency (선택 분위수) | timeseries (group-by + percentile 토글) | 전체 요청 처리 시간. 단순 비교 KPI. |
| ① SLO | Token Throughput | timeseries (2-line, group-by 토글) | prompt vs gen 토큰 처리량을 동시에. group-by 적용 시 가속기/모델별로 prompt·gen 각각 분리. |
| ① SLO | Request Completion Rate | timeseries (stacked, group-by 토글) | finished_reason(stop/length/abort) 비율 시간변화. group-by 시 finished_reason × 그룹 라인. |
| ② System | Concurrency | timeseries (stacked area, group-by 토글) | running + waiting 누적. group-by 시 가속기별 running/waiting 분리. |
| ② System | KV Cache Usage + Waiting | timeseries (avg/max + dashed waiting, group-by 토글) | avg/max 만 보면 한계 도달 여부만. waiting 큐 길이를 이중 축으로 같이 그려 "사용률 높고 waiting 도 늘면 실질 압력" 컨텍스트. group-by 시 가속기/모델별 avg·max·waiting. |
| ② System | Queue Wait (선택 분위수) | timeseries (group-by + percentile 토글) | 큐 대기 자체가 SLO 침해 신호. |
| ② System | Active Deployments | stat | 현재 운영 중인 모델×가속기 조합 수. 상황 파악용. |
| ③ Prefix Cache | Prefix Cache Hit Rate + Queries/sec | timeseries (full width, dual axis, group-by 토글) | hit rate 단독은 분모 0(트래픽 없음/prefix caching 비활성) 시 NaN 으로 빈 시리즈 → 원인 불분명. queries/sec 보조선을 함께 그려서 0/0 상황을 시각적으로 즉시 구분. group-by 시 그룹별 hit rate 와 queries/sec 분리. |
| ④ Workload | Prompt Token Length | heatmap (group-by 토글) | 입력 길이 분포. histogram 메트릭이라 heatmap 자연스러움. group-by 시 multi-series row 분리. |
| ④ Workload | Generation Token Length | heatmap (group-by 토글) | 출력 길이 분포. |
| ④ Workload | Finish Reason Ratio | donut (group-by 토글) | 종료 이유 비율을 한눈에. abort 가 보이면 위험 신호. group-by 시 finished_reason × 그룹 곱한 slice. |
| ④ Workload | Throughput Share by Accelerator | donut | 가속기 간 부하 분산 비율. 자체 accelerator 분리라 group-by 미적용. |

---

## Per-Model 대시보드 — 패널 선택 근거

이전 stat + sparkline 위주 구조는 정보 밀도가 낮고 X-Y 축 없이 숫자만 노출되어 시계열 컨텍스트 부족. **Overview 와 같은 timeseries 패널로 재설계**, 카테고리 5개 × 가속기 3개 = 15 row.

| Row | 패널 (multi-series timeseries) | 이유 |
|-----|--------------------------------|------|
| ① SLO Latency | TTFT · TPOT · E2E (선택 분위수 1개) | 3 가지 latency 가 한 패널, 분위수는 `percentile` variable. 라인 3개라 깔끔. |
| ② Throughput | prompt + generation (tok/s) | 입력 vs 출력 처리량을 동시에. |
| ③ System (Concurrency + KV) | running + waiting (stacked) + kv% (right axis) | 동시성 + 캐시 압력을 한 panel 안. KV% 는 우측 축, 점선 오렌지로 구분. |
| ④ Prefix Cache | hit rate (left) + queries/sec (right) | 분모 0 즉시 인지를 위해 queries/sec 함께. 점선 블루. |
| ⑤ Prefill vs Decode | prefill + decode (선택 분위수) | Aggregation 에서 옮긴 패널. 모델별로 단계 시간 차이를 본다. (Aggregation 에서 group-by 시 prefill+decode×그룹 라인이 폭발해 부적합) |

**패널 규격**: `gridPos.w=8, h=6`, `maxPerRow=3`, `repeatDirection=horizontal`. 24-wide 그리드에 3 panel/줄. 30개 모델이면 10줄.

**가속기별 sub-row**: 카테고리 row 5개 × A100/H100/V100 = 15 row, 모두 기본 collapsed. 초기 로드 시 panel query 가 0개이므로 빠르다. 가속기 추가 시 sub-row + `model_<accel>` variable 추가.

**percentile 동작**: SLO Latency 와 Prefill vs Decode 두 카테고리만 percentile variable 적용. Throughput·System·Prefix Cache 는 분위수 무관.

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
- KV residency 메트릭 (`kv_block_lifetime`, `kv_block_idle_before_evict`, `kv_block_reuse_gap`) 은 vLLM 측 미구현 (`--kv-cache-metrics-sample` 플래그 미동작) 으로 판단되어 패널 제외. 추후 vLLM 에서 정식 노출되면 heatmap 으로 다시 추가 가능.
- 모델 수가 100개 초과로 가면 Per-Model 의 panel repeat 가 무거워질 수 있다. 그 시점에 row 를 더 세분화하거나 별도 dashboard 로 분리.
