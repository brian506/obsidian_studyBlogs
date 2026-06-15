# 📊 SSE 성능 측정 — 목적·지표·목표

> 면접에서 SSE 관련 질문 받았을 때 정량으로 답할 수 있도록 측정값을 모으는 게 이 테스트의 본질.

---

## 🎯 큰 목적

> "SSE를 도입했다고 말할 때, 그 도입이 **어디까지 견디고 어떤 한계가 있는지**를 수치로 답할 수 있게 만든다."

면접에서 가장 약한 답은 *"동시 연결 잘 됩니다"* 같은 정성적 표현. 가장 강한 답은:

> *"t3.medium 1대에서 동시 SSE 연결 300개를 10분 soak 시 broadcast P95가 X ms로 안정 유지됐고, 1000개 부근에서 HikariCP 풀이 먼저 포화됐습니다. 캐시 적용 후엔 JVM heap이 다음 병목이었습니다."*

이런 답을 만들기 위한 측정.

---

## 🪜 단계별 목표

### 1차 목표 — MVP 보증
**동시 SSE 100 연결, 10분 soak 시 안정**
- 가정: 보호자 100명이 동시 접속 (MVP 100명 사용자가 모두 동시 접속 가정 — 최악 케이스)
- 통과 조건:
  - `sse_errors` 0 또는 무시 가능 수준
  - broadcast P95 < 500ms
  - heap 평탄 (10분간 증가폭 < 10%)
  - HikariCP 사용률 < 50%

### 2차 목표 — 3x 헤드룸
**동시 300 연결, 5분 soak 시 안정**
- 트래픽 3배 튀어도 견디는가
- P95 < 1000ms 정도까지 허용

### 3차 목표 — 한계 + 병목 stack 파악
**점진적 증가로 어디서 무엇이 먼저 터지는지 확인**
- 측정값이 아니라 **순서**가 중요
- 예: "DB 풀 → JVM heap → fd → CPU" 같은 순서로 답할 수 있어야 함
- 캐시 적용 전후의 병목 순서 비교가 더 강력

### 4차 목표 — 누수 검증
**Drop-to-zero 후 2분 유지 시 `sse_active_connections` = 0**
- VU 300 → 0 으로 떨어뜨린 뒤 emitter가 정말 0으로 복귀하는가
- 안 떨어지면 → onError/onCompletion 콜백 누락 = 메모리 누수

---

## 📏 측정 지표 — 분류별

### A. 처리량 (Throughput)

| 지표 | 메트릭 | 왜 보는가 | 목표 |
|---|---|---|---|
| 동시 SSE 연결 수 | `sse_active_connections` | **본질** — "몇 명까지 동시에 받아낼 수 있나" 직답 | 1차 100, 2차 300 |
| K6 VU 수 | k6 자체 | active와 일치해야 정상. 차이 나면 연결 실패 발생 중 | k6 stage에 따름 |
| RPS (SSE 엔드포인트만) | `rate(http_server_requests_seconds_count{uri=~"/api/v1/location/stream/.*"}[1m])` | 신규 연결 요청 빈도 | — |

### B. 지연 (Latency) — ★ 본질 지표

| 지표 | 메트릭 | 왜 보는가 | 목표 |
|---|---|---|---|
| Broadcast P50/P95/P99 | `histogram_quantile(0.95, ... sse_broadcast_duration_seconds_bucket ...)` | **GPS 한 건 → 모든 보호자에게 도달까지 시간**. 이게 클라가 실제로 느끼는 지연 | P95 < 500ms, P99 < 1s |
| 연결 자체 지연 | k6 `sse_connect_ms` | 인증+검증+첫 이벤트까지 걸리는 시간. 검증 캐시 효과 검증 | P95 < 1.5s |

### C. 메모리 / 누수 — ★ SSE의 가장 취약점

| 지표                        | 메트릭                                                    | 왜 보는가                                                            | 목표                 |
| ------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------- | ------------------ |
| JVM Heap                  | `jvm_memory_used_bytes{area="heap"}`                   | 연결당 메모리 점유. 선형 증가 → 정상, 지수 → 누수                                  | 100 conn → 50MB 이내 |
| GC Pause                  | `rate(jvm_gc_pause_seconds_sum[1m])`                   | 메모리 압박이 GC로 드러남                                                  | 평탄                 |
| Emitter 제거 경로             | `sum by(reason) (rate(sse_emitter_removed_total[1m]))` | completion(정상) vs timeout vs error 비율. error 비율 비정상 시 send 누적 실패 | error 비율 < 1%      |
| Send 실패율                  | `rate(sse_emitter_send_failures_total[1m])`            | broadcast 시 IOException. 누적되면 죽은 emitter 청소 누락                   | 0에 수렴              |
| **Drop-to-zero 시 active** | `sse_active_connections`                               | **누수 검증의 핵심**. K6가 0으로 떨어진 뒤에도 active가 양수면 누수                    | 정확히 0              |

### D. 자원 한계 — 어디서 먼저 터지는지

| 지표 | 메트릭 | 왜 보는가 | 임계 |
|---|---|---|---|
| HikariCP 사용률 | `hikaricp_connections_active / hikaricp_connections_max` | **현재 막힌 그곳**. 캐시 도입 전후 비교 핵심 | < 80% |
| HikariCP Pending | `hikaricp_connections_pending` | 풀에서 못 받고 대기 중인 스레드. 0 초과면 위험 | 0 |
| JVM Threads | `jvm_threads_live_threads` | Tomcat thread 한계. 연결당 thread 점유 시 폭증 | < max thread |
| File Descriptors | `process_files_open_files` | OS 한계. 연결당 fd 1개 점유 | < max files * 0.8 |
| EC2 CPU | `system_cpu_usage` | broadcast가 CPU 바운드인지 확인 | < 80% |
| EC2 Memory | `node_memory_MemAvailable_bytes` | OS 메모리 압박 | 500MB+ 여유 |

### E. 통신/기록 부수 효과

| 지표 | 메트릭 | 왜 보는가 |
|---|---|---|
| Redis Ops/sec | `redis_instantaneous_ops_per_sec` | 첫 연결 시 Redis 조회가 폭증하는지 |
| Redis Hit율 | `redis_keyspace_hits / (hits + misses)` | 최신 GPS 캐시 효율 |
| 5xx 비율 | 기존 panel-8 | 부하 중 에러 발생 |

---

## 🧭 측정 순서 (체크리스트)

- [ ] 1. `LocationValidator` 결과 캐시 적용 (Caffeine 5분 TTL)
- [ ] 2. `SseEmitterManager`에 Micrometer 메트릭 4개 추가
- [ ] 3. Spring Boot Actuator로 `/actuator/prometheus` 노출 확인 (`sse_`, `hikaricp_` 메트릭 보임)
- [ ] 4. Grafana 대시보드에 panel-28~34 추가
- [ ] 5. `ulimit -n` 확인 (≥ 65536), 부족하면 ECS task 정의에서 ulimits 설정
- [ ] 6. tokens.json 생성 (50~100쌍, 테스트 전용 계정)
- [ ] 7. K6 실행: `k6 run sse-load-test.js`
- [ ] 8. Grafana에서 K6 stage와 같은 시간축 그래프 캡쳐
  - 8-1. MVP soak 시점 (10분 평탄 구간)
  - 8-2. 3x 헤드룸 시점
  - 8-3. **Drop-to-zero 후 active=0 도달 시점** ← 면접용 한 장
- [ ] 9. 결과 노트 채우기

---

## 📊 결과 정리 빈 표 (측정 후 채울 곳)

| 시나리오 | active | broadcast P95 | heap 증가 | HikariCP 사용률 | 결과 |
|---|---|---|---|---|---|
| MVP (100, 10분 soak) | __ | __ ms | __ MB | __ % | __ |
| 3x (300, 5분 soak) | __ | __ ms | __ MB | __ % | __ |
| 한계 탐색 | 첫 병목 = __, 두 번째 = __ | | | | |
| 누수 검증 (drop → 0 → 2분) | active 도달값 = __ | | | | |

---

## 💬 면접 답안 템플릿 (측정 후 빈칸 채우기)

> Q. SSE 부하 테스트 해보셨나요?
>
> A. *"K6로 단계적 부하 시나리오를 작성해 측정했습니다. MVP 목표인 동시 100 연결에서 10분 soak 시 broadcast P95가 __ms 였고 heap은 평탄했습니다. 헤드룸 확인을 위해 300까지 올렸을 때 P95가 __ms로 증가했고, 이 시점에서 HikariCP 사용률이 __%까지 올랐습니다.
>
> 한계 탐색에선 ___이 먼저 병목이었는데, 그 원인이 LocationValidator의 검증 쿼리가 연결마다 DB를 친 거였습니다. Caffeine 캐시를 도입한 뒤엔 ___이 새로운 병목이었고요.
>
> 가장 신경 쓴 부분은 누수 검증인데, VU를 0으로 떨어뜨린 뒤 2분 유지하면서 `sse_active_connections` 가 정확히 0으로 복귀하는 걸 확인했습니다. onError/onCompletion 콜백이 빠짐없이 호출돼 emitter가 메모리에서 모두 해제됨을 그래프로 검증했습니다."*

이렇게 답할 수 있게 만드는 게 이 측정의 목적.
