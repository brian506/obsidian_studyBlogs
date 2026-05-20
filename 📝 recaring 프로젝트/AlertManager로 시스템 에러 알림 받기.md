
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
#### 1. Prometheus -> Alertmanager
- Prometheus는 `/actuator/prometheus`를 주기적으로 긁어오면서, 수집된 데이터를 rule.yml과 계속 대조한다.
- rule.yml 조건에 충족되면, prometheus.yml에 등록된 Alertmanager의 주수로 HTTP POST 요청을 보낸다.
#### 2.Alertmanager -> Trigger Server (Webhook 호출) 
- AlertManager는 프로메테우스로부터 알림을 받으면, 해당 알림을 처리할 Webhook URL로 전송한다. (ex. Spring App, Slack 등)
##### 1. AlertWebhookController (Alertmanager의 요청 JSON 바디)
```java
public record AlertWebhookRequest(  
        String version,  
        String groupKey,  
        String status,  
        @NotNull List<AlertItemRequest> alerts  
) {}

public record AlertItemRequest(  
        String fingerprint,  
        String startsAt,  
        String endsAt,  
        String status,  
        Map<String, String> labels,  
        Map<String, String> annotations  
) {}
```
**AlertWebhookRequest** : Alertmanager -> webhook url
- version : Alertmanager 웹푹 페이로드 포맷의 버전
- groupkey : `alertmanegr.yml` 의 group_by에 따라 그룹화된 알림들의 고유 식별 키(같은 원인이나 그룹으로 묶인 알림들을 식별할 때 사용)
- status : firing - 현재 발생 중인 알림, resolved - 문제 해결된 알림
- alerts : 발생한 개별 알림들의 목록
	- Alertmanager는 알림이 발생할때마다 매번 웹훅을 쏘지 않고, 설정된 시간(group_wait)동안 모인 알림들을 리스트 형태로 한 번에 묶어서 보내기 때문에 배열 형태를 갖는다.
**AlertItemRequest** : 에러/경고 내용
- fingerprint : 해당 알림의 고유 해시. 알림의 중복 여부를 판별하거나, 이전에 발생(firing)했던 특정 알림이 나중에 해결처리 되었는지 생명 주기를 추적할 때 사용
- startsAt : 알림이 처음 발생한 시간
- endAt : 알림이 해결된 시간
- status : 개별 알림의 상태(firing or resolved)
- labels : 메트릭을 수집할 때 설정된 알림의 식별자
	- _예시:_ `alertname="HighCPUUsage"`, `instance="10.0.1.55:8080"`, `severity="critical"`, `job="spring-boot-app"`
	- 어느 서버의 어떤 서비스에서 알림이 발생했는지 식별
- annotations : 알림에 대한 사람이 읽을 수 있는 부가 설명

##### 2.AlertService
1. webhook secret 값 검증
2. request에서 받은 AlertItem 토대로 객체를 만들어서 `orchestrate()`비동기 실행(가상 스레드)
	- HTTP 응답은 미리 반환

##### 3.AlertInvestigationOrchestrator
1. 중복 실행 방지 - InvestigationStateReader
	- Redis(investigation:{fingerprint}) 형식으로 '지금 이 알림 조사 중'이라는 중복 방지를 위해서 캐싱한다.
	- Redis에 이미 처리 중인 알림 사항을 조회해서, 있으면 중복되므로 스킵한다.
	- Alertmanager는 문제가 해결될때까지 똑같은 알림을 계속 반복해서 쏘아서 무한 재발송을 하지 않도록 방어 로직이 필요하다
2. 런북 캐시 확인 - RunbookManager
	- alertName으로 이전에 같은 오류가 발생한 기록이 있는 지 조회한다. (alertName은 프로메테우스에서 설정한 에러 이름)
3. cachehit or cachemiss
	- 초기 슬랙 알림 전송
	- stateWriter 에서 Redis에다가 `RUNNING` 상태 저장
- **cachehit** : 해당 알림은 이미 해결했던 알림이고, DB에 해결책이 있을 때
	-  executeRunbook()- 가상 스레드로 비동기 수행
		1. SSM 실행
			- 성공 시 : 조사 알림, Runbook(성공 횟수) DB 저장, 슬랙 성공 내용 전송
			- 실패 시 : 조사 알림 실패로 저장, 슬랙 실패 내용 전송
		2. Redis에서 해당 알림 수행 데이터 삭제
- **cachemiss** : DB에 없는 알림이고, 이때 AI로 분석해서 수행
	- investigate() - 가상 스레드로 비동기 수행
		1. investigateAgent.investigate() 수행
		```java
		String analysis = chatClient.prompt()  
        .user(buildPrompt(alert))  
        .tools(alertTools)  
        .call()  
        .content();
		```
- 이 구조로 AI에게 작업을 전달한다.
- 여기서 buildPrompt() 안에 있는 프롬프트 안에는 AI가 읽고 자율적으로 판단할 수 있도록 tools에 관한 지시사항이 들어있다.
- 이 프롬프트의 tools 지시사항은 alertTools에서 미리 제공하는데, 이 alertTools 에는 Spring AI가 @Tool 어노테이션이 붙은 메서드의 스펙을 AI에게 전달한다. 이후 AI는 프롬프트 지시사항을 보고 스스로 판단해서 필요한 tool을 순서대로 호출한다.
#### Trigger Server -> Claude Agent SDK 호출
	- Alertmanager가 Trigger Server를 호출하여 해당 경로의 service 클래스가 Claude Agent를 호출한다. 그리고 받은 메시지를 분석하여 다음 일을 처리할 수 있도록 한다.