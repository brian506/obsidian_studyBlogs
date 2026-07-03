
## 왜 필요한가
CPU는 한 번에 하나의 프로세스만 실행 가능 → 여러 프로세스를 **언제, 얼마나** 실행할지 결정하는 정책이 필요.

## 평가 기준
- **CPU 이용률(Utilization)** ↑
- **처리량(Throughput)** ↑ (단위 시간당 완료 프로세스 수)
- **응답 시간(Response Time)** ↓ (요청 → 첫 응답)
- **대기 시간(Waiting Time)** ↓ (Ready Queue에서 기다린 시간)
- **반환 시간(Turnaround Time)** ↓ (제출 → 완료)

## 선점 vs 비선점
- **선점(Preemptive)**: 실행 중인 프로세스를 강제로 뺏을 수 있음 (RR, SRT, MLFQ)
	- 컨텍스트 스위칭 오버헤드 큼
- **비선점(Non-preemptive)**: 끝나거나 I/O로 자발적으로 양보할 때까지 점유 (FCFS, SJF)
	- 컨텍스트 스위칭 오버헤드 적음
	- 잦은 우선순위 역전, 평균 응답 시간 증가

## 주요 알고리즘

### FCFS (First-Come, First-Served)
- 도착 순서대로 처리 (큐)
- 단점 : **Convoy Effect**
	- 긴 프로세스 뒤에 짧은 프로세스가 줄줄이 기다림 → 평균 대기 시간 ↑
- 배치 시스템에 적합, Interactive 시스템에 부적합

### Round Robin (RR)
- 도착 순서대로 처리 (ready 큐 기준)
- **자원 사용 제한 시간**(타임 퀀텀)이 있음
- 응답성 ↑ → 시분할 시스템에 적합
- 퀀텀이 너무 작으면 **Context Switching 오버헤드 ↑**, 너무 크면 FCFS와 동일

### SJF (Shortest Job First)
- 실행 시간이 짧은 것부터 처리 → 평균 대기 시간 **최적**
- 단점: **실행 시간 예측 불가능**, 긴 프로세스 **기아(Starvation)**

### SRT (Shortest Remaining Time)
- SJF의 선점 버전. 새 프로세스의 남은 시간이 현재보다 짧으면 교체

### Priority Scheduling
- 우선순위 높은 것부터 → 기아 가능 → **에이징(Aging)** 으로 보완

### MLFQ (Multi-Level Feedback Queue)
- 여러 우선순위 큐. **짧은 작업 = 높은 우선순위**에서 빠르게 / **긴 작업 = 낮은 우선순위**로 점진적 이동
- 실시간 OS에서 흔히 사용. CPU bound와 I/O bound 자동 분류

## 면접 빈출 질문
1. **선점형과 비선점형 차이?** → CPU 강제 회수 가능 여부, 응답성 vs 단순성 트레이드오프
2. **SJF의 단점?** → 실행 시간 예측 불가 + 긴 프로세스 기아
3. **Round Robin의 타임 퀀텀이 너무 작으면?** → Context Switching 오버헤드가 실행 시간보다 커짐
4. **기아(Starvation) 해결 방법?** → 에이징(Aging) — 대기 시간이 길어질수록 우선순위 ↑
5. **MLFQ가 좋은 이유?** → CPU bound·I/O bound를 동적으로 구분, 짧은 작업의 응답성과 긴 작업의 처리량 모두 챙김

## OS별 실제 스케줄러
- **Linux CFS (Completely Fair Scheduler)**: vruntime 기반, 가장 작은 vruntime을 가진 프로세스 선택 → 공정성
- **Windows**: 우선순위 + 라운드 로빈 혼합
