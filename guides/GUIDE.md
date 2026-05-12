# ServiceMonitor 라벨 주입 — 사내 처리 가이드

> 작성일: 2026-05-12
> 대상: `~/github/vllm-serving-optimizer/benchmark-server.py`

---

## 1. 지금 안 되는 것

Grafana 대시보드 2개 (`vllm-overview`, `vllm-per-model`) 의 모든 PromQL 이 `model`, `accelerator` 라벨을 참조한다. 현재 이 두 라벨이 Prometheus 메트릭에 **없다**.

### 현재 메트릭에 있는 라벨
- `model_name` — vLLM 이 자체 노출. 값은 `--served-model-name` 인자 = **전체 모델 파일 경로** (예: `/models/openai/gpt-oss-120b`). 짧은 ID 아님.
- `job` — Prometheus 가 ServiceMonitor 기반 자동 부여. 값 = `{model_id}-{accel_id}` 형식 (예: `baseline-llama70b-h100`).
- `pod`, `service`, `namespace`, `instance`, `endpoint` — Prometheus 자동.

### 없는 라벨 (대시보드가 필요한 것)
- `model` — 짧은 모델 ID (예: `baseline-llama70b`)
- `accelerator` — 가속기 키 (예: `h100`)

### 그 결과
- Overview/Per-Model 의 모든 패널 PromQL → 빈 결과 (`vllm:...{model=~"$model",accelerator=~"$accelerator"}` 가 매칭되지 않음)
- variable 쿼리 `label_values(vllm:num_requests_running, accelerator)` → 빈 드롭다운
- 대시보드 import 해도 **화면이 전부 비어 보임**

---

## 2. 필요한 수정 — `benchmark-server.py`

### 파일 / 함수
`~/github/vllm-serving-optimizer/benchmark-server.py`, `_k8s_create_service_monitor()` (296~322줄 부근).

### Before (현재)
```python
spec = {
    "spec": {
        "selector": {"matchLabels": {
            "benchmark.ai/managed-by": "benchmark-client",
            "benchmark.ai/accelerator": accel,
            "benchmark.ai/model": model,
        }},
        "endpoints": [{"port": "http", "path": "/metrics", "interval": "15s"}],
    },
}
```

### After (수정)
```python
spec = {
    "spec": {
        "selector": {"matchLabels": {
            "benchmark.ai/managed-by": "benchmark-client",
            "benchmark.ai/accelerator": accel,
            "benchmark.ai/model": model,
        }},
        "endpoints": [{
            "port": "http",
            "path": "/metrics",
            "interval": "15s",
            "relabelings": [
                {
                    "sourceLabels": ["__meta_kubernetes_service_label_benchmark_ai_model"],
                    "targetLabel": "model",
                },
                {
                    "sourceLabels": ["__meta_kubernetes_service_label_benchmark_ai_accelerator"],
                    "targetLabel": "accelerator",
                },
            ],
        }],
    },
}
```

### 주의
- `relabelings` 는 ServiceMonitor `spec.endpoints[].relabelings` (`metricRelabelings` 아님). 둘 차이:
  - `relabelings` — **스크레이프 시점** 에 target label 부착. 모든 시계열에 들어감. ← 우리가 원하는 것.
  - `metricRelabelings` — 스크레이프된 메트릭 값을 후처리할 때.
- K8s 라벨 키의 `.`, `/`, `-` 는 Prometheus discovery 메타라벨에서 `_` 로 변환됨:
  - `benchmark.ai/model` → `__meta_kubernetes_service_label_benchmark_ai_model`
  - `benchmark.ai/accelerator` → `__meta_kubernetes_service_label_benchmark_ai_accelerator`
- `sourceLabels` 는 배열, `targetLabel` 은 단일 문자열.

---

## 3. 기존 떠 있는 서버 처리

코드 수정만 하면 **신규 배포부터** 적용된다. 이미 떠 있는 ServiceMonitor 는 그대로라 라벨 안 붙음. 둘 중 하나:

### 옵션 A — 재배포 (간단)
콘솔에서 서버 회수 → 다시 배포. 다운타임 있음.

### 옵션 B — 기존 ServiceMonitor 일괄 patch (다운타임 없음)
```bash
kubectl get servicemonitor -A -l benchmark.ai/managed-by=benchmark-client -o name | while read sm; do
  ns=$(kubectl get $sm -A -o jsonpath='{.metadata.namespace}' 2>/dev/null || echo default)
  kubectl patch -n "$ns" "$sm" --type=json -p='[
    {"op":"add","path":"/spec/endpoints/0/relabelings","value":[
      {"sourceLabels":["__meta_kubernetes_service_label_benchmark_ai_model"],"targetLabel":"model"},
      {"sourceLabels":["__meta_kubernetes_service_label_benchmark_ai_accelerator"],"targetLabel":"accelerator"}
    ]}
  ]'
done
```

> Prometheus Operator 가 변경을 감지해서 자동으로 reload 한다 (10~30초 내).

---

## 4. 검증

### Prometheus API 로 확인
```bash
curl -s 'http://<prometheus-host>:9090/api/v1/series?match[]=vllm:num_requests_running' | jq '.data[0]'
```

기대 결과 — 다음 키들이 보이면 성공:
```json
{
  "__name__": "vllm:num_requests_running",
  "model": "baseline-llama70b",
  "accelerator": "h100",
  "model_name": "/models/.../...",
  "job": "baseline-llama70b-h100",
  "pod": "...",
  ...
}
```

### Grafana Explore 로 확인
```promql
vllm:num_requests_running{model="baseline-llama70b",accelerator="h100"}
```
값이 나오면 OK. 빈 결과면 relabeling 이 안 붙은 것.

### 대시보드 import 후
1. Grafana → Import → `dashboards/vllm-overview.json`, `vllm-per-model.json`
2. Datasource = Prometheus 선택
3. Overview 상단 `Accelerator` 드롭다운 클릭 → `a100`, `h100`, `v100` 가 보이면 성공
4. Per-Model 의 row 펼치면 모델별 panel 들이 나타나면 성공

---

## 5. (옵션) 추가 라벨 노출이 필요해진다면

향후 quant/kv/tp 별 비교가 필요해지면, 콘솔에서 배포 시 해당 정보를 **K8s 라벨로도 부착**한 뒤 (현재는 어노테이션만), `relabelings` 에 같은 패턴으로 추가:

```python
{
    "sourceLabels": ["__meta_kubernetes_service_label_benchmark_ai_quantization"],
    "targetLabel": "quantization",
},
```

단 Pod/Service 이름 충돌 (`{model_id}-{accel_id}` 고정) 문제부터 해결해야 동일 모델+가속기를 옵션 달리 동시 배포 가능. 자세한 한계는 `docs/servicemonitor-labels.md` 참조.

---

## 6. 한 줄 요약

`benchmark-server.py` 296~322줄의 `endpoints` 에 `relabelings` 두 줄 추가 → 신규 배포 자동 적용 + 기존 서버는 옵션 B 의 `kubectl patch` 한 번 실행 → 대시보드가 살아난다.
