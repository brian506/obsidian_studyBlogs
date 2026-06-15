# SSE GPS 실시간 전송: 브로드캐스트 fan-out에서 연결당 폴링으로

> 2,000 SSE 연결 + 80 GPS POST/초 부하 테스트에서 만난 병목과, 그 해결 과정에서 다시 공부해야 했던 "SSE는 사실 어떻게 동작하는가"에 대한 기록.

---

## 1. 전환 동기

### 기존 구조 — 이벤트 기반 브로드캐스트

```
WARD 앱 → POST /location/gps
            ↓
        LocationService.receiveGps()   [@Transactional]
            ↓ DB INSERT + Redis SET
            ↓ ApplicationEventPublisher.publish(GpsReceivedEvent)
            ↓
        GpsEventHandler  [@TransactionalEventListener(AFTER_COMMIT)]
            ↓
        SseEmitterManager.broadcast(wardKey, gps)
            ↓ Map<wardKey, List<SseEmitter>> 뒤져서 fan-out
```

GPS가 들어올 때마다 그 ward를 보는 모든 보호자에게 push.

### 발견된 두 가지 병목

**병목 1 — fan-out이 Tomcat 요청 스레드를 잡고 있음**

`@TransactionalEventListener(AFTER_COMMIT)`은 별도 스레드로 옮겨가지 않는다. **이벤트를 발행한 그 스레드(=GPS POST를 처리 중인 Tomcat 워커)에서 동기 실행.** 그래서 fan-out 루프가 끝날 때까지 GPS POST 응답이 반환되지 않았다.

**병목 2 — 본질적 책임 결합**

GPS 쓰기(WARD 동작)와 GPS 읽기(보호자 동작)는 주체·시점·빈도가 다른데, 브로드캐스트는 이 둘을 한 트랜잭션 흐름에 묶어버린다.

- 보호자가 화면을 안 보고 있어도 fan-out 비용은 그대로
- WARD 쓰기 경로가 보호자 수 N에 비례해 느려짐 (INSERT 1 + 전송 N)
- **쓰기 처리량이 읽는 사람 수에 종속됨**

전환의 결정적 질문은 이거였다.

> "broadcast()를 굳이 저 메서드 안에 넣어야 하나? 보호자가 위치를 볼 때 그쪽에서 당겨가게 하는 게 맞지 않나?"

GPS는 **모든 이벤트가 의미 있는 데이터가 아니다.** 최신 상태만 동기화되면 충분한 state-sync 성격의 데이터다. 본질적으로 push보다 pull에 어울리는 데이터였다.

---

## 2. 새 구조 — 연결당 폴링 (Push → Pull)

**쓰기 경로 (WARD)**

```
POST /location/gps
   ↓
LocationService.receiveGps()
   ↓ DB INSERT (이력)
   ↓ Redis SET gps:latest:{wardKey} (최신 1건, TTL 5분)
끝. fan-out 없음.
```

**읽기 경로 (보호자)**

```
GET /location/stream  (SSE)
   ↓
SseEmitterManager.connect()
   ↓ SseEmitter 생성 + 전용 폴링 스레드 1개 기동
   ↓ while (active) {
   │     Redis GET gps:latest:{wardKey}
   │     직전 전송값과 다르면 emitter.send()
   │     Thread.sleep(1000)
   │  }
보호자가 화면을 떠날 때까지 반복.
```

### 핵심 설계 결정: 초기 전송과 갱신 전송을 하나의 루프로 통합

처음엔 "연결되면 최신값 1회 전송 → 이후 갱신 시 전송" 두 갈래로 나눌까 했다. 결국 단일 루프로 통합:

```java
private void pollLoop(SseEmitter emitter, String wardKey, AtomicBoolean active) {
    Gps lastSent = null;
    while (active.get()) {
        Optional<Gps> latest = gpsLatestCacheReader.find(wardKey);
        if (latest.isPresent() && !latest.get().equals(lastSent)) {
            emitter.send(SseEmitter.event().name("location").data(latest.get()));
            lastSent = latest.get();   // 초기/갱신 구분 없이 동일 로직
        }
        Thread.sleep(1000);
    }
}
```

`lastSent`가 null로 시작하니 첫 루프에서 자연스럽게 초기값이 송신된다. 초기 전송이 특수 케이스가 아니라 "lastSent ≠ null 케이스"의 일반화로 사라진다.

### 트레이드오프

|항목|Broadcast (Before)|Polling (After)|
|---|---|---|
|쓰기 경로 비용|보호자 수에 비례|일정|
|읽기 경로 비용|이벤트 시에만|1초마다 Redis GET (상시)|
|전송 지연|즉각|최대 1초|
|Redis 부하|낮음|연결 수 × 1회/초|
|책임 분리|결합|완전 분리|

"아무도 안 봐도 매초 Redis를 때린다"는 비용은 감수했다. GPS 케이스에서는 Redis GET이 메모리 조회로 매우 싸고, **쓰기 처리량을 보호자 수에서 떼어내는 것이 더 중요했다.**

---

## 3. 깊이 들어가기 — SSE는 OS 레벨에서 실제로 어떻게 동작하나

이 전환을 하면서, 정말 풀어 이해해야 했던 게 "SseEmitter란 도대체 뭐고, 누가 무엇을 들고 있는가"였다. 단계별로 정리한다.

### 3-1. 연결 한 건이 만들어지는 전체 흐름

```
[클라이언트 브라우저]                       [Linux 커널]              [Tomcat]                    [Spring MVC]
       │                                        │                       │                            │
       │── TCP SYN ───────────────────────────→│                       │                            │
       │←─ SYN-ACK ────────────────────────────│                       │                            │
       │── ACK ───────────────────────────────→│ (3-way handshake)     │                            │
       │                                        │                       │                            │
       │── GET /location/stream                 │                       │                            │
       │   Accept: text/event-stream          ─→│ (socket buffer)       │                            │
       │                                        │  epoll: "read ready" ─→ NIO poller 깨움            │
       │                                        │                       │── 워커 스레드 T1 할당 ───→│
       │                                        │                       │                            │── 컨트롤러 실행
       │                                        │                       │                            │   return SseEmitter
       │                                        │                       │←─ request.startAsync() ───│
       │                                        │                       │   T1 즉시 풀로 반납       │
       │←─ HTTP/1.1 200 OK ───────────────────  │                       │                            │
       │   Content-Type: text/event-stream      │                       │                            │
       │   Transfer-Encoding: chunked           │                       │                            │
       │                                        │                       │                            │
       │   (소켓은 열린 채 유지. 쓸 스레드는 없음)                                                    │
       │                                        │                       │                            │
       │   (보호자가 화면 보는 동안)            │                       │                            │
       │←─ data: {"lat":..,"lng":..}\n\n ──────│←─ 폴링 스레드 V1이 ────────────emitter.send() ──────│
       │                                        │                       │                            │
```

이 그림이 머리에 박히면 나머지는 다 따라온다. 핵심은 **"연결을 들고 있는 주체"와 "데이터를 흘리는 주체"가 다르다**는 것.

- **연결을 들고 있는 주체**: OS 커널의 TCP 소켓 + Tomcat의 NIO 인프라
- **데이터를 흘리는 주체**: 누군가 `emitter.send()`를 호출해주는 스레드

### 3-2. NIO — 연결 N개를 스레드 N개 없이 유지하는 법

옛날 BIO(Blocking I/O) 모델은:

```
연결 1개 = 워커 스레드 1개가 socket.read()에 블로킹된 채 대기
연결 10,000개 = 10,000 스레드가 놀면서 점유
```

NIO는 OS의 이벤트 알림 메커니즘(Linux의 `epoll`, macOS의 `kqueue`)을 쓴다:

```
[Tomcat NIO 구조]
  Acceptor (1~2 스레드)      : 새 TCP 연결 accept만 함
  Poller   (1~2 스레드)      : 수천 개 소켓을 selector(epoll)로 감시.
                                "소켓 X에 읽을 데이터 도착" 같은 이벤트만 받음
  Worker pool (기본 200)     : Poller가 "이 요청 처리해!"라고 깨워주면 그제서야 일함
```

요점: **연결 자체는 OS와 Poller가 들고 있고, 워커 스레드는 "처리할 일이 있을 때만" 잠깐 빌려 쓴다.** 그래서 연결 2,000개를 워커 스레드 200개로 충분히 감당할 수 있다 — 단, **워커가 한 요청을 오래 붙들지 않을 때만.**

기존 브로드캐스트의 병목이 바로 여기였다. `@TransactionalEventListener(AFTER_COMMIT)`가 워커를 fan-out 끝날 때까지 붙들고 있으니, 이 NIO의 장점이 무력화됐다.

### 3-3. Servlet 비동기 — 왜 `return emitter` 한 줄에 워커가 반납되나

Servlet 3.0부터 `HttpServletRequest.startAsync()` API가 추가됐다. 의미는:

> "이 응답을 아직 다 못 만들었지만, 나중에 다른 스레드가 완성할 거니까 너(=Tomcat)는 이 워커 스레드를 풀어줘도 돼."

Spring MVC는 컨트롤러의 반환 타입이 `SseEmitter`인 걸 보면 **내부적으로 `startAsync()`를 호출**한다. 그 순간:

- Tomcat은 응답 객체를 `AsyncContext`로 감싸 보관 (소켓은 살아있음)
- 워커 스레드 T1은 풀로 반납 → 다른 요청 처리하러 감
- 클라이언트 입장에서는 응답 헤더가 이미 갔으니 "이 응답은 계속될 거구나" 인식 (chunked transfer)

`SseEmitter`는 이 `AsyncContext` 안에 들어있는 응답 출력 스트림에 대한 **쓰기 핸들**이다. `emitter.send(...)`는 결국 `response.getOutputStream().write(...)` → Tomcat NIO → 커널 소켓 버퍼 → 클라이언트.

> **SseEmitter는 스레드가 아니다. "쓰기 가능한 살아있는 응답"에 대한 자바 측 손잡이.**

### 3-4. 그러면 왜 별도 스레드가 필요한가

워커 T1이 반납됐으니, 이제 그 emitter에 `send()`를 호출해줄 사람이 없다. 두 가지 패턴이 있다:

**Push 패턴 (기존 브로드캐스트):**

- GPS POST를 처리하는 **다른** Tomcat 워커가 들어옴 → Map에서 emitter들 찾아 send → 끝
- 별도 백그라운드 스레드 불필요 (이벤트 드리븐)

**Pull 패턴 (지금 폴링):**

- 외부에서 트리거되는 이벤트가 없음 → **우리가 만든 백그라운드 스레드**가 주기적으로 Redis 읽고 send해야 함
- 이 스레드를 명시적으로 띄워야 함

지금 구조는:

```java
public SseEmitter connect(String wardKey) {
    SseEmitter emitter = new SseEmitter(5 * 60 * 1000L); // 5분 타임아웃
    AtomicBoolean active = new AtomicBoolean(true);

    emitter.onCompletion(() -> stop(active, removedCompletion));
    emitter.onTimeout(()    -> stop(active, removedTimeout));
    emitter.onError(e       -> stop(active, removedError));

    activeConnections.incrementAndGet();

    Thread.ofPlatform()                       // TODO: 가상 스레드로 교체
            .name("sse-poll-", 0)
            .daemon()
            .start(() -> pollLoop(emitter, wardKey, active));

    return emitter;   // ← 이 순간 요청 워커는 풀로 반납됨
}
```

스레드별 역할 정리:

| 스레드             | 누가 만들었나      | 무슨 일을 하나                                      | 언제 죽나                     |
| --------------- | ------------ | --------------------------------------------- | ------------------------- |
| Tomcat 요청 워커 T1 | Tomcat 풀     | `connect()` 메서드 실행, emitter 생성, V1 띄움, return | `return` 즉시 풀로 복귀         |
| 폴링 스레드 V1       | 우리가 명시적으로 생성 | 10초마다 Redis 읽고 `emitter.send()`               | `active=false` 되어 루프 탈출 시 |
| SSE 콜백 스레드      | Tomcat 컨테이너  | 연결 끊김/타임아웃 감지 시 `active=false` 세팅             | 콜백 끝나면 풀로 복귀              |

폴링 스레드 V1과 콜백 스레드는 **다른 스레드**다. 그래서 `active`가 두 스레드 간 통신 채널 역할을 해야 하고, 이게 `AtomicBoolean`이어야 하는 이유다 — 일반 `boolean`은 한 스레드의 쓰기가 다른 스레드에 즉시 보인다는 메모리 가시성 보장이 없다.

### 3-5. SseEmitter ↔ 스레드 관계 한 줄 정리

> **연결(소켓)은 OS와 Tomcat NIO가 들고 있고, SseEmitter는 그 연결에 글을 쓸 수 있는 "펜" 같은 객체다. 펜이 있어도 글을 쓰려면 손이 필요한데, 그 손이 폴링 스레드다. Push 방식에서는 이벤트가 손을 빌려주고, Pull 방식에서는 우리가 손을 따로 붙여줘야 한다.**

### 3-6. 플랫폼 스레드 vs 가상 스레드 — 그리고 왜 가상 스레드인가

지금 폴링 스레드는 플랫폼 스레드(`Thread.ofPlatform()`)다. 한계가 명확하다:

- 플랫폼 스레드 1개 = OS 스레드 1개 = **스택 메모리 ~1MB**
- 2,000 연결 → 2,000 OS 스레드 → **약 2GB 네이티브 메모리**
- 우리 서버 t3.medium(RAM 4GB)에서는 이게 메모리 압박 → GC STW 유발 → STW 중 GPS 트랜잭션이 DB 커넥션을 붙잡은 채 멈춤 → HikariCP 커넥션 풀 고갈

실제 로그에 이렇게 찍혔다:

```
HikariPool-1 - Connection is not available, request timed out
  after 30000ms (total=40, active=40, waiting=192)
```

이건 "DB 커넥션이 부족해서"가 아니라, **2,000개 플랫폼 스레드의 메모리 압박 → GC STW → 그동안 커넥션 반납이 지연된** 연쇄 효과였다.

가상 스레드(JDK 21+)는 다르다:

- 가상 스레드는 OS 스레드에 1:1로 묶이지 않음 — 캐리어(플랫폼 스레드) 위에 다대일로 얹힘
- `Thread.sleep()`이나 I/O 대기(Redis GET) 동안에는 캐리어를 **unmount**하고 양보
- 폴링 루프는 시간의 대부분을 `sleep(1000)`으로 보내므로, 가상 스레드의 이점이 극대화되는 워크로드

2,000개 가상 스레드를 띄워도 캐리어는 몇 개로 충분하고, 메모리 부담은 수백 MB 수준으로 떨어진다.

> 회고적으로: 처음부터 가상 스레드로 갔으면 깔끔했을까? 그렇긴 한데, **플랫폼 스레드로 한계를 먼저 측정하고 → 가상 스레드로 해결**하는 순서가 troubleshooting 서사로 훨씬 설득력 있다. "왜 가상 스레드여야 하는지를 데이터로 증명"하는 흐름.

### 3-7. HTTP/1.1 vs HTTP/2 — 같은 보호자가 여러 ward를 볼 때

지금 구조는 **보호자 한 명이 ward 1개를 보면 SSE 연결 1개**. 그런데 보호자가 ward 3명을 동시에 본다면? 보통은 **wardId마다 SSE 연결을 따로** 여는 게 코드가 단순하다 (각 연결이 자기 emitter + 자기 폴링 스레드를 가지므로 독립적).

문제는 브라우저의 동시 연결 한도:

**HTTP/1.1**

- 도메인당 동시 연결 ~6개가 일반적
- SSE는 **장기 점유** 연결이므로 1개당 슬롯 1개를 영구 차지
- 보호자가 ward 3명을 보면 SSE에 3슬롯 → 일반 API 호출용으로 3슬롯만 남음
- 보호자가 ward 6명 이상을 보려는 순간 막힘

**HTTP/2**

- TCP 연결 1개를 **다중 스트림(stream)으로 multiplexing**
- 한 TCP 연결 안에서 100+ 스트림 동시 가능
- SSE 연결 여러 개를 띄워도 같은 TCP 위에 얹히므로 "도메인당 6개" 한계가 사실상 사라짐

따라서 SSE를 본격적으로 쓰는 서비스는 **HTTP/2 (또는 그 위의 HTTPS) 활성화가 사실상 전제**다. 우리 인프라가 ALB(HTTP/2 지원) 뒤에 있으면 자동으로 혜택을 보고, 직접 Tomcat을 노출한다면 `server.http2.enabled=true`를 켜야 한다.

대안으로는 한 SSE 연결에 wardId를 다중화해서 흘리는 방법도 있지만:

- 메시지마다 `{wardId, lat, lng}` 식으로 라우팅 정보 포함
- 가상 스레드 1개가 N개 ward 모두 폴링 (여러 스레드가 같은 emitter를 만지면 동시성 문제)
- 클라이언트가 메시지 보고 분기

→ 코드는 더 복잡하지만 연결·메모리 1/N. 보호대상자가 많은 도메인(요양원 등)에서는 검토할 만하다.

---

## 4. 동작 흐름 (코드 레벨)

### 쓰기 — WARD가 GPS를 보낼 때

```java
@Transactional
public void receiveGps(String wardMemberKey, double lat, double lng) {
    Gps gps = new Gps(lat, lng, LocalDateTime.now());
    gpsHistoryWriter.save(wardMemberKey, lat, lng);   // DB INSERT (이력)
    gpsLatestCacheWriter.save(wardMemberKey, gps);    // Redis SET (최신값)
}
```

응답 즉시 반환, 워커 스레드 즉시 반납. 보호자가 몇 명이든 이 경로 비용은 동일.

### 읽기 — 보호자가 위치를 볼 때

```java
public SseEmitter connect(String wardKey) {
    SseEmitter emitter = new SseEmitter(5 * 60 * 1000L);
    AtomicBoolean active = new AtomicBoolean(true);

    emitter.onCompletion(() -> stop(active, removedCompletion));
    emitter.onTimeout(()    -> stop(active, removedTimeout));
    emitter.onError(e       -> stop(active, removedError));

    activeConnections.incrementAndGet();

    Thread.ofPlatform()
            .name("sse-poll-", 0)
            .daemon()
            .start(() -> pollLoop(emitter, wardKey, active));

    return emitter;
}

private void pollLoop(SseEmitter emitter, String wardKey, AtomicBoolean active) {
    Gps lastSent = null;
    try {
        while (active.get()) {
            Optional<Gps> latest = gpsLatestCacheReader.find(wardKey);
            if (latest.isPresent() && !latest.get().equals(lastSent)) {
                emitter.send(SseEmitter.event().name("location").data(latest.get()));
                lastSent = latest.get();
            }
            Thread.sleep(1000);
        }
    } catch (IOException | IllegalStateException e) {  // 연결 끊김/이미 닫힘
        sendFailures.increment();
        stop(active, removedError);
        completeQuietly(emitter, e);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}

private void stop(AtomicBoolean active, Counter reasonCounter) {
    if (active.compareAndSet(true, false)) {   // 최초 1회만 통과
        activeConnections.decrementAndGet();
        reasonCounter.increment();
    }
}
```

---

## 5. Q&A

### Q1. `AtomicBoolean active`는 왜 필요한가? 그냥 `boolean`이면 안 되나?

**스레드 간 메모리 가시성 + 한 번만 실행 보장.**

폴링 스레드와 SSE 콜백 스레드는 **다른 스레드**다. 일반 `boolean`은 한 쪽의 쓰기가 다른 쪽에 즉시 보인다는 보장이 없다(JMM 가시성 문제). `AtomicBoolean`은 가시성을 보장하고, `compareAndSet(true, false)`로 "종료 처리를 정확히 한 번만" 실행하게 해준다. 그래서 `stop()`에서 메트릭 카운터가 중복 감소하지 않는다.

### Q2. 스레드는 알아서 할당되는 거 아닌가? 왜 `Thread.ofPlatform()`으로 명시 생성하나?

`return emitter` 순간 요청 워커는 풀로 반납된다(서블릿 비동기 전환). 연결은 살아있지만 그 연결을 위해 일해줄 스레드는 남아있지 않다. 폴링 루프를 누군가는 돌려야 하는데, Tomcat 워커를 붙잡으면 그게 곧 옛 병목의 재현이다. 그래서 **연결마다 전담 스레드를 명시적으로 띄워야** 한다.

### Q3. Map을 안 쓰면, 보호자가 이탈했을 때 알아서 정리되나?

거의 그렇다. 정확히는:

||자동?|
|---|---|
|연결 끊김 감지|❌ 우리 코드 (`send()` 실패 catch + 콜백)|
|감지 후 루프 종료|❌ 우리 코드 (`active=false` → 다음 iteration 탈출)|
|스레드 종료 → emitter, lastSent 등 자원 회수|✅ 자동 (지역 변수 → GC)|

기존 Map 방식은 `Map.remove()`라는 **명시적 삭제** 코드가 필요했다. 새 구조는 그게 없다. 다만 **루프를 빠져나가게 만드는 코드는 여전히 필요**하다 — 그게 없으면 끊긴 연결의 스레드가 좀비로 남는다.

### Q4. 가상 스레드로 가면 뭐가 달라지나?

폴링 루프는 대부분의 시간을 `sleep(1000)`으로 보내는, 전형적인 IO/대기 위주 워크로드다. 가상 스레드는 sleep/IO 중에 캐리어 스레드를 양보하므로 2,000개를 띄워도 캐리어는 적은 수로 충분하다. 코드 변경은 거의 `Thread.ofPlatform()` → `Thread.ofVirtual()` 한 줄.

### Q5. 부하 테스트가 비현실적인 것 아닌가?

타당한 지적이다. k6는 80건/초를 거의 동시 발사해 **worst-case thundering herd**를 만든다. 현실에선 단말마다 네트워크 지터로 분산된다.

하지만 부하 테스트의 목적은 "현실 재현"이 아니라 **"한계 지점 탐색"**이다. 가혹한 동시성에서 병목을 먼저 드러내야 현실의 스파이크에 마진이 생긴다. 테스트는 유지하되, 결과 해석할 때 "이건 worst-case"라는 단서를 붙이는 게 정확하다.

---

## 6. 정리

||Before|After|
|---|---|---|
|패러다임|Push (이벤트 브로드캐스트)|Pull (연결당 폴링)|
|쓰기 경로|INSERT + 이벤트 발행 + fan-out|INSERT + Redis SET|
|읽기 경로|이벤트 수신 시 전송|10초 폴링, 변경 시 전송|
|연결 관리|Map 레지스트리 + 수동 remove|스레드 지역 상태, 자동 GC|
|쓰기·읽기 결합|강결합 (보호자 수에 종속)|완전 분리|
|다음 단계|—|플랫폼 → 가상 스레드 전환|

핵심은 **"실시간 push라는 매력적인 단어에 끌려 쓰기와 읽기를 한 트랜잭션에 묶었던 것"이 병목의 본질**이었고, 두 경로를 분리하니 쓰기 처리량이 읽는 사람 수에서 자유로워졌다는 점. 그 대가로 생긴 "상시 폴링 비용"은 가상 스레드로 흡수할 계획.

부수적으로 다시 짚고 가게 된 것들:

- **OS 소켓 / NIO / Servlet Async는 각자 다른 레이어**. 연결을 들고 있는 주체와 데이터를 흘리는 주체는 다르다.
- **SseEmitter는 스레드가 아니라 "쓰기 핸들"**. 누가 send() 호출해줄지 따로 정해줘야 한다.
- **HTTP/2 multiplexing**은 SSE를 본격적으로 쓰는 서비스에선 사실상 전제 조건이다.
- **상태 변수의 동시성**은 `AtomicBoolean` 한 줄이 보장. 일반 boolean으로는 못 잡는 가시성 문제가 실재한다.