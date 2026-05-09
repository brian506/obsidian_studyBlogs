
프로메테우스가 메트릭을 scrape하는데, 여기서 메트릭 수집으로 얻는 데이터에 대해서 에러 위험 대비를 위한 알림이나, 컨테이너 활동 여부에 따른 알림을 수집해서 즉시 대응할 수 있도록 할 것이다. 

## 수집 데이터

### 컨테이너 Up/Down
- ECS의 태스크 상태가 바뀔때마다 EventBridge로 이벤트 자동 발행
- 각 컨테이너의 상태 모니터링은 **cAdvisor**로 수집

cAdvisor의 볼륨 추가
![](../images/스크린샷%202026-05-05%2016.07.15.png)
### EC2 CPU & Memory & Disk
node-exporter로 수집
`proc`
- CPU 사용량, 메모리 상태(`/proc/meminfo`), 현재 실행 중인 프로세스 정보들이 파일 형태로 존재
`sys`
- 네트워크 인터페이스(트래픽 인/아웃), 블록 디바이스, 하드웨어 장치들의 상태 정보
`rootfs`
- EC2 인스턴스의 실제 전체 디스크 용량과 사용량을 확인하기 위해 루트(`/`) 경로를 마운트

- CPU > 85%
- Memory > 80%
- Disk > 75%

### JVM Heap 사용률
- Spring Boot Actuator 로 수집
### HTTP 5xx 비율
- Spring Boot Actuator 로 수집
### DB 커넥션 풀
- Spring Boot Actuator 로 수집
### HTTP P95 응답 시간
- Spring Boot Actuator 로 수집
### Redis
- Memory > 80%
- connected_clients
- evicted_keys (메모리 부족 신호)

## 지표별 알림 기준

### 응급 - LLM이 코드 분석 후 오류 수정
- 크래시 루프 (컨테이너가 반복해서 재시작 시도)
- 5xx 비율
- DB 커넥션 풀 고갈
- Disk 임박

### 경고 - slack으로 알림만 
- CPU
- Memory
- P95 응답시간
- Redis 메모리
- Error 로드
## 알림 플로우
![](../images/스크린샷%202026-05-07%2014.51.05.png)


### 동작 순서
1. Prometheus -> Alertmanager
	- Prometheus는 `/actuator/prometheus`를 주기적으로 긁어오면서, 수집된 데이터를 rule.yml과 계속 대조한다.
	- rule.yml 조건에 충족되면, prometheus.yml에 등록된 Alertmanager의 주수로 HTTP POST 요청을 보낸다.
2. Alertmanager -> Trigger Server (Webhook 호출)
	- AlertManager는 프로메테우스로부터 알림을 받으면, 해당 알림을 처리할 Webhook URL로 전송한다. (ex. Spring App, Slack 등)
3. Trigger Server -> Claude Agent SDK 호출
	- Alertmanager가 Trigger Server를 호출하여 해당 경로의 service 클래스가 Claude Agent를 호출한다. 그리고 받은 메시지를 분석하여 다음 일을 처리할 수 있도록 한다.