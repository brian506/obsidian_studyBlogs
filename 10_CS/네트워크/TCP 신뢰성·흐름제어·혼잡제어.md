# TCP 신뢰성·흐름 제어·혼잡 제어

> 3-way / 4-way handshake는 [[TCP - IP 4계층]]. 이 노트는 **데이터 전송 중**에 작동하는 메커니즘.

## 큰 그림
TCP는 단순히 "연결 + ACK" 가 아니라 4가지 메커니즘으로 **신뢰성**을 만들어낸다:
1. **순서 번호** + **확인 응답(ACK)** → 신뢰성·순서
2. **재전송** → 손실 복구
3. **흐름 제어** → 수신자 압도 방지
4. **혼잡 제어** → 네트워크 압도 방지

## 1. 순서 번호와 ACK

### Sequence Number
- 전송하는 **모든 바이트에 번호** 부여 (초기값 ISN은 랜덤)
- 수신자는 번호로 재정렬

### ACK Number
- "이 번호까지는 잘 받았어. 다음엔 N번부터 보내줘"
- **누적 ACK**: 마지막으로 연속 수신한 다음 번호

### SACK (Selective ACK)
- "1~100, 200~300 받음" 처럼 **구멍을 알려줌**
- 송신자는 빠진 부분만 재전송 (효율적)

## 2. 재전송 (Retransmission)

### RTO (Retransmission Timeout)
- ACK 안 오면 **타임아웃 후 재전송**
- RTO는 **RTT(왕복 시간) 측정** 기반으로 동적 조정

### Fast Retransmit
- 같은 ACK가 **3번 연속** 오면 손실로 판단 → 타임아웃 기다리지 않고 즉시 재전송
- 예: 1, 2, **(3 손실)**, 4, 5 받으면 수신자가 ACK 3을 3번 → 송신자 즉시 3 재전송

## 3. 흐름 제어 (Flow Control) — 수신자 보호

### Sliding Window
- 수신자가 ACK와 함께 **수신 가능한 윈도우 크기(rwnd)** 알림
- 송신자는 "ACK 안 받은 데이터"가 rwnd를 넘지 않게 조절

```
송신자: [전송 완료/ACK 받음 | 전송 완료/ACK 대기 | 아직 전송 X ]
                              ←   윈도우   →
```

### Zero Window
- 수신자가 처리 못해 rwnd=0 → 송신자 일시 정지
- 수신자가 다시 받을 수 있게 되면 Window Update

## 4. 혼잡 제어 (Congestion Control) — 네트워크 보호

흐름 제어는 종단(수신자)을 보호. **혼잡 제어는 네트워크 전체를 보호.**

### 핵심 변수
- **cwnd (congestion window)**: 송신자가 자체적으로 정하는 윈도우
- 실제 송신량 = `min(rwnd, cwnd)`

### 4단계 알고리즘 (TCP Reno 기준)

#### 1) Slow Start
- cwnd = 1부터 시작 → **ACK 받을 때마다 2배** (지수 증가)
- 한계점 `ssthresh` 까지

#### 2) Congestion Avoidance
- `ssthresh` 도달 후 **선형 증가** (1 RTT마다 +1)
- 안전하게 한계 탐색

#### 3) Fast Retransmit
- 중복 ACK 3개 → 손실 감지 → 즉시 재전송

#### 4) Fast Recovery
- 손실 후 cwnd를 1로 줄이지 않고 **절반으로** → 더 빨리 회복

### 알고리즘 변종
| 알고리즘 | 특징 |
| --- | --- |
| **Reno** | 위 4단계 표준 |
| **Cubic** | **Linux 기본**. 3차 함수로 빠른 회복 |
| **BBR** (Google) | 손실이 아닌 **대역폭·RTT 측정** 기반 → 무손실 환경(데이터센터·CDN)에서 우수 |

## HOL (Head-of-Line) Blocking
TCP는 **순서 보장** → 한 패킷 손실 시 그 이후 패킷이 도착해도 애플리케이션에 전달되지 않음.
→ HTTP/2 멀티플렉싱의 한계 원인 → **HTTP/3 (QUIC)** 이 UDP로 해결

## TCP가 효율적이지 못한 상황
- **고지연 환경** (위성 통신): handshake·ACK 왕복이 너무 비쌈
- **무선·이동 환경**: 패킷 손실을 혼잡으로 오해해 cwnd 급감
- **다중 스트림 (HTTP/2)**: HOL Blocking
→ 이런 한계를 해결하려고 **QUIC**가 등장

## 면접 빈출 질문
1. **TCP가 신뢰성을 보장하는 4가지 메커니즘?** → 순서번호 + ACK, 재전송, 흐름 제어, 혼잡 제어
2. **흐름 제어와 혼잡 제어 차이?** → 수신자 보호(end-to-end) vs 네트워크 보호(중간 경로)
3. **Slow Start와 Congestion Avoidance 차이?** → 지수 증가 vs 선형 증가. ssthresh가 분기점
4. **Fast Retransmit 트리거 조건?** → 동일 ACK 3번 연속
5. **TCP의 HOL Blocking이란?** → 손실 패킷 때문에 뒤이은 정상 패킷들도 애플리케이션에 전달되지 않음
6. **무선 환경에서 TCP가 비효율적인 이유?** → 손실을 혼잡으로 오해 → cwnd 급감 → 처리량 ↓
7. **TCP Cubic과 BBR 차이?** → Cubic은 손실 기반, BBR은 대역폭·RTT 측정 기반

## 프로젝트 연결
- 부하 테스트(K6) 결과 해석 시 네트워크 지표(평균 응답, 99 percentile) → TCP RTT·재전송이 원인일 수 있음
- ALB/CloudFront 사용 시 HTTP/2 활성화로 멀티플렉싱 효과 + 향후 HTTP/3로 HOL 해소
