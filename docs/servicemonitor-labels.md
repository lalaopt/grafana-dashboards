# ServiceMonitor 라벨 구분 현황 및 한계

> 작성일: 2026-05-12

---

## 1. 모델 배포 시 부착되는 라벨·어노테이션

콘솔에서 모델을 배포하면 Pod + Service에 다음이 붙는다.

### 라벨 (Labels)

| 라벨 키 | 예시 값 | 용도 |
|--------|---------|------|
| `app` | `baseline-llama70b` | K8s 기본 셀렉터 |
| `benchmark.ai/model` | `baseline-llama70b` | 모델 카탈로그 ID |
| `benchmark.ai/accelerator` | `h100`, `a100`, `v100` | 가속기 식별 |
| `benchmark.ai/accel-type` | `H100`, `A100` | 가속기 타입 대문자 |
| `benchmark.ai/managed-by` | `benchmark-client` | 콘솔 회수 권한 |
| `benchmark.ai/expose` | `"true"` | Service 드롭다운 노출 여부 (Service에만) |

### 어노테이션 (Annotations)

| 어노테이션 키 | 포맷 | 예시 |
|-------------|------|------|
| `benchmark.ai/vllm-args` | `tp=N,maxlen=N,gpumem=0.xx,quant=X,kv=X,spec=X` | `tp=4,maxlen=4096,gpumem=0.90,quant=fp8,kv=,spec=` |
| `benchmark.ai/display-name` | `"모델명 (가속기명)"` | `"Llama 3.1 70B (H100)"` (Service에만) |

### vllm-args 필드 상세

| 키 | 의미 | 빈값 의미 |
|----|------|----------|
| `tp` | tensor-parallel size | — |
| `maxlen` | max-model-len | — |
| `gpumem` | gpu-memory-utilization | — |
| `quant` | weight quantization (fp8/awq/gptq/int8) | BF16 기본 |
| `kv` | KV cache dtype (turboquant_k8v4) | vLLM 기본 |
| `spec` | speculative decoding draft_id/N | 미사용 |

---

## 2. ServiceMonitor 셀렉터

`benchmark-server.py:_k8s_create_service_monitor()`에서 생성하는 ServiceMonitor의 `selector.matchLabels`:

```yaml
benchmark.ai/managed-by: "benchmark-client"
benchmark.ai/accelerator: "<accel_id>"
benchmark.ai/model:       "<model_id>"
```

모델 + 가속기 조합이 ServiceMonitor 단위이며, Prometheus는 해당 Service의 `:8000/metrics`를 15초마다 수집한다.

---

## 3. 구분 가능 여부 평가

| 구분 기준 | 방법 | 충분한가 |
|----------|------|---------|
| 가속기별 | 라벨 `benchmark.ai/accelerator` | ✅ 완전 분리 |
| 모델별 | 라벨 `benchmark.ai/model` | ✅ 완전 분리 |
| TP 수(GPU 수)별 | 어노테이션 `vllm-args.tp` | ⚠️ Prometheus 레이블 없음 |
| 양자화 옵션별 | 어노테이션 `vllm-args.quant` | ⚠️ Prometheus 레이블 없음 |
| KV dtype별 | 어노테이션 `vllm-args.kv` | ⚠️ Prometheus 레이블 없음 |
| Speculative Decoding별 | 어노테이션 `vllm-args.spec` | ⚠️ Prometheus 레이블 없음 |

### 핵심 한계

1. **어노테이션은 K8s 셀렉터에 사용 불가** — ServiceMonitor `matchLabels`는 라벨만 참조한다.
2. **Prometheus metric 레이블** — vLLM `/metrics`에서 노출되는 `vllm:*` 메트릭의 레이블은 `model_name`(= `--served-model-name`, 모델 파일 경로)만 붙는다. quant/kv/spec 정보는 없다.
3. **동일 모델+가속기 중복 배포 불가** — Pod/Service 이름이 `{model_id}-{accel_id}` 고정이므로, 같은 조합을 다른 옵션으로 동시에 띄울 수 없다(이름 충돌).

---

## 4. 옵션별 Prometheus 구분이 필요한 경우 임시 방법

1. **PromQL에서 job으로 필터** — ServiceMonitor 이름이 `{model_id}-{accel_id}`이므로, `job=~"baseline-llama70b-h100"` 식으로 모델+가속기 조합까지 필터 가능.

2. **재배포 후 시간 구간으로 비교** — 동일 조합을 quant/kv 바꿔 재배포하면 이전 Pod이 삭제되고 새 Pod으로 교체되므로, 시계열 구간(시간 범위)으로 구분 가능.

3. **근본 해결(선택)** — `benchmark.ai/quantization`, `benchmark.ai/kv-dtype` 등을 라벨로도 추가하면 ServiceMonitor selector에서 활용 가능. 단, Pod/Service 이름 충돌을 막으려면 이름 체계도 변경 필요 (`{model_id}-{accel_id}-{quant}` 등).

---

## 5. 현재 권장 사용 방식

| 목적 | 방법 | 가능 여부 |
|------|------|----------|
| 모델별 / 가속기별 비교 | Grafana에서 `benchmark.ai/model`, `benchmark.ai/accelerator` 라벨로 패널 분리 | ✅ 지금도 가능 |
| 옵션 효과 비교(quant, kv 등) | 옵션별로 순차 배포 후 시간 구간을 달리해 Prometheus 쿼리 | ⚠️ 수동 관리 필요 |
| 콘솔 UI 확인 | 서버 카드 hover 시 어노테이션에서 파싱된 quant/kv/spec 표시 | ✅ UI에서는 구분 가능 |
