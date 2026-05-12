# vLLM 메트릭 문서

> 출처: https://docs.vllm.ai/en/stable/design/metrics/
> 버전: stable (2026-05-12 기준)

---

## 1. 서버 레벨 메트릭 (Server-level Metrics)

### 요청 상태

| 메트릭명 | 타입 | 설명 | 레이블 |
|---------|------|------|--------|
| `vllm:num_requests_running` | Gauge | 현재 실행 중인 요청 수 | `model_name` |
| `vllm:num_requests_waiting` | Gauge | 대기 중인 요청 수 | `model_name` |
| `vllm:num_requests_swapped` | Gauge | **[폐기]** 스왑된 요청 수 (V1에서 스왑 기능 제거) | `model_name` |

### KV 캐시

| 메트릭명 | 타입 | 설명 | 레이블 |
|---------|------|------|--------|
| `vllm:kv_cache_usage_perc` | Gauge | 사용된 KV 캐시 블록 비율 (0~1) | `model_name` |
| `vllm:cpu_cache_usage_perc` | Gauge | **[폐기]** CPU 캐시 사용률 (V1에서 제거) | `model_name` |
| `vllm:cache_config_info` | Gauge | 캐시 설정 Info 메트릭 (값=1.0) | `block_size`, `cache_dtype`, `cpu_offload_gb` 등 |

### 접두사 캐시 (Prefix Cache)

| 메트릭명 | 타입 | 설명 | 레이블 |
|---------|------|------|--------|
| `vllm:prefix_cache_queries` | Counter | 접두사 캐시 쿼리 총 수 | `model_name` |
| `vllm:prefix_cache_hits` | Counter | 접두사 캐시 히트 총 수 | `model_name` |

> **히트율 계산 (PromQL)**: `rate(vllm:prefix_cache_hits[5m]) / rate(vllm:prefix_cache_queries[5m])`

### 토큰 처리량

| 메트릭명 | 타입 | 설명 | 레이블 |
|---------|------|------|--------|
| `vllm:prompt_tokens_total` | Counter | 처리된 프롬프트 토큰 총 수 | `model_name` |
| `vllm:generation_tokens_total` | Counter | 생성된 토큰 총 수 | `model_name` |

> **처리량 계산 (PromQL)**: `rate(vllm:prompt_tokens_total[1m])`, `rate(vllm:generation_tokens_total[1m])`

### 요청 완료

| 메트릭명 | 타입 | 설명 | 레이블 |
|---------|------|------|--------|
| `vllm:request_success_total` | Counter | 완료된 요청 수 (종료 이유별) | `finished_reason` (stop\|length\|abort), `model_name` |

---

## 2. 요청 레벨 메트릭 (Request-level Histograms)

### 크기

| 메트릭명 | 타입 | 단위 | 설명 | 레이블 |
|---------|------|------|------|--------|
| `vllm:request_prompt_tokens` | Histogram | 개수 | 입력 토큰 수 분포 | `model_name` |
| `vllm:request_generation_tokens` | Histogram | 개수 | 생성 토큰 수 분포 | `model_name` |
| `vllm:request_max_num_generation_tokens` | Histogram | 개수 | (병렬 샘플링) 최대 출력 길이 | `model_name` |

### 레이턴시

| 메트릭명 | 타입 | 단위 | 설명 | 레이블 |
|---------|------|------|------|--------|
| `vllm:time_to_first_token_seconds` | Histogram | 초 | TTFT — 첫 토큰까지의 시간 (arrival_time 기준) | `model_name` |
| `vllm:inter_token_latency_seconds` | Histogram | 초 | TPOT — 연속 토큰 간 지연시간 | `model_name` |
| `vllm:e2e_request_latency_seconds` | Histogram | 초 | 전체 요청 레이턴시 (arrival_time → 마지막 토큰) | `model_name` |
| `vllm:request_queue_time_seconds` | Histogram | 초 | 큐 대기 시간 (QUEUED → 최근 SCHEDULED) | `model_name` |

### 처리 단계별 시간

| 메트릭명 | 타입 | 단위 | 설명 | 레이블 |
|---------|------|------|------|--------|
| `vllm:request_prefill_time_seconds` | Histogram | 초 | 프리필 단계 소요 시간 | `model_name` |
| `vllm:request_decode_time_seconds` | Histogram | 초 | 디코드 단계 소요 시간 | `model_name` |

---

## 3. KV 캐시 잔존 메트릭

> `--kv-cache-metrics-sample` 옵션 필요. 성능 영향을 줄이기 위해 샘플링됨.

| 메트릭명 | 타입 | 단위 | 설명 | 레이블 |
|---------|------|------|------|--------|
| `vllm:kv_block_lifetime_seconds` | Histogram | 초 | KV 블록 생명주기 (할당 → 제거) | `model_name` |
| `vllm:kv_block_idle_before_evict_seconds` | Histogram | 초 | 마지막 접근 → 제거까지 유휴 시간 | `model_name` |
| `vllm:kv_block_reuse_gap_seconds` | Histogram | 초 | KV 블록 재사용 간격 (연속 접근 간 시간) | `model_name` |

---

## 4. LoRA 메트릭

| 메트릭명 | 타입 | 설명 | 레이블 |
|---------|------|------|--------|
| `vllm:lora_requests_info` | Gauge | LoRA 어댑터별 실행/대기 요청 수 | `running_lora_adapters`, `waiting_lora_adapters`, `max_lora` |

---

## 5. 스펙큘레이티브 디코딩 메트릭

| 메트릭명 | 타입 | 설명 |
|---------|------|------|
| `vllm:spec_decode_draft_acceptance_rate` | Gauge | 드래프트 토큰 수락률 (0~1) |
| `vllm:spec_decode_efficiency` | Gauge | 스펙 디코딩 효율성 |
| `vllm:spec_decode_num_accepted_tokens` | Counter | 수락된 드래프트 토큰 총 수 |
| `vllm:spec_decode_num_draft_tokens` | Counter | 생성된 드래프트 토큰 총 수 |
| `vllm:spec_decode_num_emitted_tokens` | Counter | 최종 방출된 토큰 총 수 |

---

## 6. OpenTelemetry 상세 추적 메트릭

> `--oltp-traces-endpoint` 및 `--collect-detailed-traces` 설정 필요

| 메트릭명 | 타입 | 단위 | 설명 | 레이블 |
|---------|------|------|------|--------|
| `vllm:model_forward_time_milliseconds` | Histogram | 밀리초 | 모델 포워드 패스 소요 시간 | `model_name` |
| `vllm:model_execute_time_milliseconds` | Histogram | 밀리초 | 모델 실행 함수 소요 시간 | `model_name` |

---

## 7. HTTP 메트릭 (prometheus_fastapi_instrumentator)

| 메트릭명 | 타입 | 단위 | 설명 | 레이블 |
|---------|------|------|------|--------|
| `http_requests_total` | Counter | 개수 | HTTP 요청 총 수 | `handler`, `method`, `status` |
| `http_request_size_bytes` | Histogram | 바이트 | HTTP 요청 크기 | `handler` |
| `http_response_size_bytes` | Histogram | 바이트 | HTTP 응답 크기 | `handler` |
| `http_request_duration_seconds` | Histogram | 초 | HTTP 요청 처리 시간 | `handler`, `method` |

---

## 메트릭 발행 방식

### Prometheus 엔드포인트
- **URL**: `/metrics`
- **포맷**: Prometheus 텍스트 형식
- 폴링 권장 간격: ~1초

### 로깅 (LoggingStatLogger)
- 5초마다 INFO 레벨 출력
- 포함 정보: 실행/대기 요청 수, GPU 캐시 사용률, 토큰 처리량, 접두사 캐시 히트율

### 멀티프로세스 모드 (`--api-server-count > 1`)
- vLLM 메트릭만 노출, Python/프로세스 메트릭 불가

---

## 레이턴시 계산 기준 (이벤트 타임스탬프)

```
QUEUED → SCHEDULED → [PREEMPTED]* → NEW_TOKENS → ... → NEW_TOKENS
```

| 인터벌 | 계산 구간 |
|--------|-----------|
| Queue time | QUEUED → 최근 SCHEDULED |
| Prefill time | 최근 SCHEDULED → 첫 NEW_TOKENS |
| Decode time | 첫 NEW_TOKENS → 마지막 NEW_TOKENS |
| TTFT | 프론트엔드 arrival_time → 첫 토큰 |
| E2E latency | 프론트엔드 arrival_time → 마지막 토큰 |

---

## 폐기된 메트릭

| 메트릭명 | 대체 방법 |
|---------|-----------|
| `vllm:num_requests_swapped` | — (V1에서 스왑 기능 제거) |
| `vllm:cpu_cache_usage_perc` | — (V1에서 CPU 스왑 제거) |
| `vllm:prefix_cache_hit_rate` | `rate(hits[5m]) / rate(queries[5m])` |
| `vllm:avg_prompt_throughput_toks_per_s` | `rate(vllm:prompt_tokens_total[5m])` |
| `vllm:time_in_queue_requests` | `vllm:request_queue_time_seconds` |
