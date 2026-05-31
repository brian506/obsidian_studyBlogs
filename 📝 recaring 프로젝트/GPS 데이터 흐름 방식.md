## SSE를 활용한 실시간 이동 경로 전송

### SSE 개념
SSE는 클라이언트와 서버 간의 지속적인 통신이 필요할 때 사용된다. 

클라이언트가 한 번 GET 요청(HTTP)을 보내고, 서버는 그 응답을 끊지 않은 채(Content-Type : text/event-stream) 계속 데이터를 흘려보내는 단방향 스트림이다.

#### SseEmitter
> 클라이언트와 연결된 전용 파이프라인 이라고 생각하면 좋다
> -> '현재 연결된 클라이언트 한 명'

클라이언트는 서버에게 받는 응답은 `SseEmitter` 객체로 단방향으로 연결한다.
- 이때 헤더에 `Content-Type : text/event-stream` 을 달아서 끊기지 않는 연결임을 명시
서버와의 통신을 통해 해당 객체의 인스턴스(emitter)의 메서드로 통신한다.

**서버 입장에서는 연결된 응답 객체(SseEmitter)를 메모리에 두고 계속해서 실시간으로 데이터를 쏴준다.**

`Map<String, CopyOnWriteArrayList<SseEmitter>>`

ex) 만약 보호 대상자(Key)와 연결된 보호자(Value)들에게 좌표값을 전송하기 위해서?
Key : 보호 대상자
Value(CopyOnWriteArrayList) : 한 명의 위치를 동시에 구독 중인 여러 보호자들
#### CopyOnWriteArrayList 인 이유?
- broadcast 스레드가 리스트를 순회하는 동안 다른 스레드가 새 보호자를 추가하거나, emitter를 제거해도 동시성 이슈가 나지 않도록 하기 위해

#### 죽은 SseEmitter 정리
> SSE 연결은 와이파이가 끊기거나 브라우저를 닫는 등 여러 이유로 언제든 끊길 수 있음
- 데이터를 쏘다가(send()) 전송 실패 예외가 발생하면, 해당 SseEmitter 객체를 끊어진 것으로 간주해야하므로 예외가 발생하거나 통신이 끊기는 로직에서 remove()로직이 반드시 필요
	- 메모리에 데이터를 보내지 못하는 SseEmitter가 매번 축적되기 때문
- SseEmitter는 메모리상에 존재할 때마다 주소값이 달라지므로 같은 클라이언트라도 항상 다른 객체로 존재

## SSE 동작 과정 및 데이터 흐름

### 1. 클라이언트와의 첫 연결
```java
public SseEmitter streamLocation(String caregiverKey, String wardKey) {  
    locationValidator.validateCaregiverAccess(caregiverKey, wardKey);  
  
    SseEmitter emitter = sseEmitterManager.connect(wardKey);  
  
    gpsLatestCacheReader.find(wardKey)  
            .ifPresent(gps -> sseEmitterManager.sendInitialEvent(emitter, gps));  
  
    return emitter;  
}
```
1. 보호자 - 보호대상자 관계인지 검증
2. `connect()` 
	```java
	public SseEmitter connect(String wardKey) {  
    SseEmitter emitter = new SseEmitter(SSE_TIMEOUT);  
    emitters.computeIfAbsent(wardKey, k -> new CopyOnWriteArrayList<>()).add(emitter);  
  
    emitter.onCompletion(() -> remove(wardKey, emitter));  
    emitter.onTimeout(() -> remove(wardKey, emitter));  
    emitter.onError(e -> remove(wardKey, emitter));  
  
    return emitter;  
}
	```
	1. 새로운 SseEmitter 생성
	2. 타임아웃 30분
	3. 특정 상황이 발생했을 떄 remove() : 메모리 누수 방지 목적
		- `onCompletion`: 클라이언트가 연결을 종료하거나, 사용자가 브라우저 탭을 닫을때 
		- `onTimeOut` : 연결 타임아웃 시간이 지났을 떄
		- `onError` : 네트워크 문제가 발생했을 떄
3.Redis에서 가장 최신 GPS를 한 번 조회후 sendInitialEvent()로 전송
- 보호자가 처음 화면 켰을 때 빈 화면을 보지 않고 마지막으로 알려진 위치를 보여주기 위해
	- **만약 레디스 장애가 일어난다면?**
		- 레디스에서 읽어오지 못하는 현상이 발생하게 되면 첫 연결 시 레디스가 아닌 DB에서 조회
4.emitter 반환 
- Spring MVC가 응답을 끊지 않고 유지


### 2. 보호자가 GPS 송신
1. GPS DB 입력
2. 이벤트 핸들러 비동기 호출
	- Redis 에 해당 GPS 값 저장
		- 최초 연결 시에 최신 GPS값 보여주기 위해
	- broadCast() : SSE 푸시
		- 이후부터는 클라이언트에게 이 메서드로 SSE 통신함

