> 보호대상자의 위치를 보호자에게 실시간으로 보여주는 GPS 서비스에서, SSE(Server-Sent Events) 동시연결 부하를 견디기 위해 거쳐 온 트러블슈팅 기록.


## 요약

```
[단계 1] "1 연결 = 1 OS 스레드"
   → 네이티브 스택 ×N → 메모리 폭발 → OOMKill (~500)
   [해결: 가상 스레드 + Push→Pull 구조 분리]
        ↓
[단계 2a] OSIV가 EntityManager를 응답 lifecycle에 묶음
   → DB 커넥션 ×N 5분 점유 → 풀 사이즈에서 천장 (50)
   [해결: open-in-view=false]
        ↓
[단계 2b] 플랫폼 스레드 한도 = 암묵적 back-pressure
   → 가상 스레드가 제거 → 무한 수락 → FD 60K 고갈
   [해결: 명시적 timeout/제한]
        ↓
[단계 2c] 가상 스레드 continuation이 힙으로 압력 이전
   → Serial GC 단일 스레드 → Full STW 910ms
   → 트랜잭션 멈춤 → 커넥션 반납 지연 → 풀 고갈 재발
   [해결: G1GC]
        ↓
[단계 3] STW 6ms, pending 0, 2K 완주
        ↓
[단계 4] 물리 메모리 95% → 인프라 영역
```

추가로 같은 시기에 같이 잡힌 것들:
- complete() 누락 → TCP RST (`completeQuietly`)
- Spring Security ASYNC 재디스패치 (`DispatcherType.ASYNC/ERROR permitAll`)
- 캐시 미스 폭발 (Redis 캐시 필요)
- 스케줄러 인덱스 누락 (DDL 필요)
- GPS history saveAsync로 응답 경로에서 분리

**네이티브 메모리 → 힙 → DB 커넥션 → FD → 다시 힙(GC) → 물리 메모리.**

## SSE는 OS 레벨에서 실제로 어떻게 동작하나

### 연결 한 건의 전체 흐름

```
[클라이언트]                       [Linux 커널]              [Tomcat]                    [Spring MVC]
   │── TCP SYN ───────────────────→ │
   │←─ SYN-ACK ──────────────────── │
   │── ACK ──────────────────────── │  (3-way handshake)
   │
   │── GET /location/stream
   │   Accept: text/event-stream ─→ │ (socket buffer)
   │                                │ epoll: "read ready" ─→ NIO poller 깨움
   │                                │                        │── 워커 T1 할당 ────→
   │                                │                        │                       │── 컨트롤러 실행
   │                                │                        │                       │   return SseEmitter
   │                                │                        │←─ request.startAsync()
   │                                │                        │   T1 즉시 풀로 반납
   │←─ HTTP/1.1 200 OK ───────────  │
   │   Content-Type: text/event-stream
   │   (소켓은 열린 채. 쓸 스레드는 없음)
   │
   │←─ data: {"lat":..} \n\n ────── │←─ 폴링 스레드 V1이 emitter.send() ──
```

**연결을 들고 있는 주체(OS 소켓 + Tomcat NIO)** 와 **데이터를 흘리는 주체(emitter.send() 호출)** 가 다르다.

### NIO — 연결 N개를 스레드 N개 없이 유지

옛 BIO는 1 연결 = 1 스레드가 socket.read() 블로킹. NIO는 OS의 epoll/kqueue로 수천 소켓을 selector 하나가 감시.

```
[Tomcat NIO]
  Acceptor (1~2)   : 새 TCP 연결 accept
  Poller   (1~2)   : selector로 수천 소켓 감시
  Worker (기본 200) : Poller가 깨워줄 때만 처리
```

연결은 OS+Poller가 들고, 워커는 "처리할 일 있을 때만" 잠깐 빌려 씀. 그래서 2,000 연결을 200 워커로 충분히 감당 — **단, 워커가 한 요청을 오래 안 붙들 때만.**

옛 broadcast의 본질적 병목이 정확히 여기였다. AFTER_COMMIT이 워커를 fan-out 끝까지 점유 → NIO의 장점 무력화.


###  HTTP/1.1 vs HTTP/2

- HTTP/1.1: 도메인당 동시 연결 ~6개. SSE는 장기 점유라 슬롯 영구 차지. 보호자가 ward 6명 이상 보면 막힘.
- HTTP/2: 단일 TCP에 100+ 스트림 multiplexing. SSE 여러 개 띄워도 한 TCP 위.

SSE를 본격적으로 쓰는 서비스에선 HTTP/2 활성화가 사실상 전제.

## 서비스 개요

- WARD(보호 대상자) 단말이 30초마다 GPS POST
- 보호자가 앱 화면에서 SSE로 위치를 실시간 수신
- 운영 기준 ~ 2000(보호 대상자 400명 X 보호자 5명) 동시 SSE 연결 

##  초기 구조 - 이벤트 기반 Broadcast

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

GPS가 들어올 때마다 그 ward를 보는 모든 보호자에게 push

### AFTER_COMMIT의 한계

`@TransactionalEventListener(AFTER_COMMIT)`는 별도 스레드로 옮겨가지 않는다. **이벤트를 발행한 그 스레드(GPS POST 처리중인 Tomcat 워커)에서 동기로 실행**된다. 그래서 broadcast()이 끝날 때까지 GPS POST 응답이 반환되지 않았다.

### 이 방식의 한계

GPS 쓰기(WARD 동작)와 GPS 읽기(보호자 동작)는 주체·시점·빈도가 다른데 broadcast 구조는 이 둘을 한 트랜잭션 흐름에 묶었다.

- 보호자가 화면을 안 보고 있어도 fan-out 비용 그대로
- WARD 쓰기 경로가 보호자 수 N에 비례해 느려짐
- **쓰기 처리량이 읽는 사람 수에 종속됨**

GPS POST와 broadcast()가 하나의 스레드에 묶여있어 응답이 올때까지 스레드를 점유하게 되므로 스레드 고갈 문제가 발생한다.
사용자가 실시간 이동 경로 분석 화면에 진입할때만 스레드를 할당하여 동작하도록 한다.

## 병목 1. ~500연결 OOMKill

**구성**: Serial GC + 플랫폼 스레드 + open-in-view=true

**개념**: "1 연결 = 1 OS 스레드" 모델. 플랫폼 스레드 1개가 살아있는 동안 네이티브 스택 ~1MB가 잡힌다. 

**구조**
```
[쓰기 경로 — WARD]
POST /location/gps
   ↓ LocationService.receiveGps()
   ↓ DB INSERT + Redis SET gps:latest:{wardKey}
끝. fan-out 없음.

[읽기 경로 — 보호자]
GET /location/stream  (SSE)
   ↓ SseEmitterManager.connect()
   ↓ SseEmitter 생성 + 전용 폴링 스레드 1개 기동
   ↓ while (active) {
   │     Redis GET gps:latest:{wardKey}
   │     직전 전송값과 다르면 emitter.send()
   │     Thread.sleep(1000)
   │  }
```

보호자에게 SSE 연결된 통신은 별도의 플랫폼 스레드를 생성해서 각 연결마다 플랫폼 스레드를 할당한다.
**플랫폼 스레드는 네이티브 스택에 쌓인다.**
연결이 누적되면서 컨테이너의 1GB 한도가 초과되어 ECS OOMKill로 인해 connection refused

### 연결 개통 스레드와 데이터 전송 스레드 별개 존재

#### 연결 개통 스레드
**역할** : 클라이언트의 `GET /location/stream` 요청을 최초로 받아서 컨트롤러까지 전달하는 역할
**동작** : 컨트롤러에서 SseEmitter 객체를 생성하고 반환한다. startAsync()로 **톰캣에게 스레드는 반납하고 소켓은 닫지 않도록 한다.**
- 연결 통로(SseEmitter)만 만들어주고 톰캣 스레드 풀로 반납

#### 데이터 전송 스레드
**역할** : 데이터를 미리 만들어둔 연결 통로(SseEmitter)를 통해 데이터를 쏘는 역할
- HTTP 요청 처리 스레드가 아니라, 별도로 특정 행위에 대한 처리를 하는 스레드
- 앞서서 연결 개통 스레드를 반납했으므로 별도의 스레드를 생성해서 데이터를 쏴줘야함

**Pull 패턴** : 백그라운드 스레드가 주기적으로 Redis 읽고 send

| 스레드             | 누가 만들었나     | 무슨 일                                    | 언제 죽나                 |
| --------------- | ----------- | --------------------------------------- | --------------------- |
| Tomcat 요청 워커 T1 | Tomcat 풀    | connect() 실행, emitter 생성, V1 띄움, return | return 즉시             |
| 폴링 스레드 V1       | 우리가 명시      | 1초마다 Redis 읽고 emitter.send()            | active=false 되어 루프 탈출 |
| SSE 콜백 스레드      | Tomcat 컨테이너 | 연결 끊김 감지 시 active=false 세팅              | 콜백 끝나면 풀로 복귀          |

### 가상 스레드 도입시 스레드 할당 구조

```
보호자 A   보호자 B   보호자 C        (구독)
   │         │         │
   ▼         ▼         ▼
 Em_A      Em_B      Em_C            (응답 쓰기 핸들, 1:1)
   │         │         │
   ▼         ▼         ▼
  V_A       V_B       V_C            (전담 가상스레드, 1:1)
   ↓ sleep(1s) 동안 캐리어 unmount
   ▼
  ┌──────────────┐  ┌──────────────┐
  │ Carrier T1   │  │ Carrier T2   │  (플랫폼 스레드, N:M)
  └──────────────┘  └──────────────┘
```

위 그림이 추상 레이어, 아래가 물리 레이어. 가상스레드 2,000개를 띄워도 캐리어는 보통 CPU 코어 수(2~4개)로 충분 — sleep/IO 중 unmount되니까.

  플랫폼 스레드 2000개:

    native memory: 2000 × ~1MB = ~2GB

  가상 스레드 2000개:

    carrier thread (플랫폼): CPU 코어 수만큼 = t3.medium 기준 2~4개 × 1MB = ~4MB

    가상 스레드 continuation: 2000 × 수 KB = ~수 MB (힙에 저장)

    합계: ~10MB 수준
    
## 병목 1 해결과 새로운 병목들

플랫폼 스레드의 무한 생성으로 인한 메모리 초과로 가상 스레드로 전환했다. 
가상 스레드의 전환으로 처리량이 많아지고 다른 점유에 관한 문제가 발생했다.

### Tomcat이 여전히 플랫폼 스레드

**증상**: 가상 스레드 적용했는데 로그에 `[io-8080-exec-N]` (플랫폼 스레드)
**원인**: AsyncConfig executor 빈만 바꿨고 Tomcat HTTP 핸들러는 별도 설정 필요
**해결**: `spring.threads.virtual.enabled: true`

### OSIV → DB 커넥션이 5분 점유 

**개념**: Spring Boot 기본값 `open-in-view=true`는 EntityManager를 요청 lifecycle 전체에 묶음. 일반 HTTP는 ms라 무해, SSE는 5분.

**인과 사슬**:
```
SSE 요청 시작
  → OpenEntityManagerInViewInterceptor.preHandle()
  → EntityManager 생성
  → validateCaregiverAccess() → HikariCP 커넥션 1개 획득
  → @Transactional 종료 — 보통은 여기서 반납되지만
  → EntityManager 살아있어 커넥션 안 놓음
  → SseEmitter 반환, async 전환
  → 5분간 EntityManager+커넥션 점유
  → afterCompletion()이 SSE 종료 때 호출 → 그제서야 반납

결과: SSE 연결 N개 = HikariCP 커넥션 N개
      활성 SSE가 풀 사이즈에서 정확히 천장 (50 풀 → 50 천장)
      GPS POST waiting=913
```

**해결**: `spring.jpa.open-in-view: false`. Reader 계층이 트랜잭션 안에서 VO로 변환 후 반환하는 구조라 LazyInitException 위험 없음.

### Back-pressure 상실 → FD 고갈

**개념**: 플랫폼 스레드 200 한도는 사실 "동시 요청 200건"이라는 **암묵적 back-pressure**였음. 가상 스레드 무제한이 되자 사라짐.

**인과 사슬**:
```
한도 제거
  → GPS POST 무한 수락
  → 각 요청이 HikariCP 30초 대기
  → 대기 중에도 소켓·FD 점유
  → JVM 스레드 60,000개, HikariCP pending 1,100+
  → FD 고갈
  → 새 연결 실패: dial: i/o timeout
```

**교훈**: 가상 스레드는 자원을 절약하지만 **암묵적 rate limit까지 제거**한다. 명시적 back-pressure가 없으면 다른 자원이 바로 천장.

**해결**: connection-timeout 단축 + 명시적 동시성 제한.

### Serial GC → Full STW 910ms (진짜 근본 병목)

위 점유들을 풀고 2K까지 도달했지만 GC가 천장.

**개념**: 가상 스레드는 스택을 안 쓰는 대신 **continuation을 힙에 저장**. 점유처가 네이티브 메모리 → 힙으로 이동. Serial GC는 단일 스레드 Full STW.

**인과 사슬**:
```
가상 스레드 2,000 continuation 객체 (수 KB × 2K)
  + GPS 직렬화 byte[] copyOfRange 매 송신 신규 생성
  + emitter 내부 상태
  ↓
Young Gen 빠르게 채워짐 → Allocation Failure
  ↓
Serial GC 단일 스레드 → Full STW 910ms
  ↓
STW 중 모든 스레드 정지 → GPS 트랜잭션 멈춤
  ↓
DB 커넥션 쥔 채 pause → HikariCP 반납 지연
  ↓
새 GPS 요청 커넥션 대기 → pending 150 스파이크
  ↓
HikariPool timeout → k6 connection error
```

**병목의 본질**: 가상 스레드 도입은 점유의 **위치만 이동**시킨 것 — 네이티브 → 힙. 힙 압력은 GC 알고리즘에 맡겨졌고, Serial GC는 처리 능력이 부족.

## 마지막 해결 G1GC 전환

**개념**: G1GC는 region 기반 + concurrent. Young Gen은 STW지만 ~6ms 수준, Old Gen은 대부분 concurrent.

**인과 사슬(긍정)**:
```
-XX:+UseG1GC 적용
  → Full STW 910ms → minor pause 6ms
  → DB 커넥션 잡고 멈추는 시간 사라짐
  → HikariCP 반납 적체 미발생 → pending 0
  → GPS POST 평균 96.4ms 안정, 5xx 0건, 2K 완주
```

| 항목               | Serial+플랫폼     | Serial+가상  | G1GC+가상      |
| ---------------- | -------------- | ---------- | ------------ |
| 천장               | ~500 (OOMKill) | ~1,500 불안정 | **2,000 완주** |
| 최대 STW           | 387ms          | 910ms      | **6ms**      |
| JFR GC Pressure  | 59             | 88         | 37           |
| HikariCP pending | 150 스파이크       | 150 스파이크   | **0**        |
| 5xx              | 다수             | 없음         | 없음           |
| GPS POST 평균      | —              | —          | 96.4ms       |

**병목의 본질**: 알고리즘이 처리 가능한 동시성 압력의 천장을 키운 것. 압력 자체를 줄인 게 아니라 **견디는 능력**을 키운 해결.

## 마지막 한계 : 물리 메모리

```
ECS 메모리 사용률 95% (1GB 물리 한계)
힙 768MB 사용 중
  ↓
연결을 더 늘리려면:
  - 인스턴스 업그레이드 (t3.medium → large)
  - ECS 태스크 메모리 증설
  - 또는 수평 확장
```

여기서부터는 코드가 아니라 **트레이드오프 결정의 영역** (비용 vs 동시성 vs SLO).

## SSE 연결이 살아있는 동안 발생할 수 있는 문제들

### A. 5분 내내 점유되는 자원

| # | 자원 | 메커니즘 | 결과 |
|---|---|---|---|
| 1 | DB 커넥션 | OSIV가 EntityManager를 응답 끝까지 살림 | 풀 사이즈에서 천장 |
| 2 | OS 스레드(스택) | 폴링 플랫폼 스레드 1MB×N | 메모리 폭발 → OOMKill |
| 3 | FD | TCP 소켓 5분 + back-pressure 부재 | FD 고갈 |
| 4 | Map 엔트리 (옛 구조) | broadcast 레지스트리 누수 위험 | 메모리 누수 |

### B. 별도 스레드가 필요해 생기는 문제

| # | 무엇 | 왜 |
|---|---|---|
| 5 | 폴링 스레드 명시 생성 | 톰캣 워커는 emitter 반환 즉시 풀로 복귀 |
| 6 | (옛 구조) fan-out이 GPS POST 워커 점유 | AFTER_COMMIT 동기 실행 |

### C. 연결 종료 시점 문제

| # | 무엇 | 왜 |
|---|---|---|
| 7 | TCP RST | complete() 미호출 |
| 8 | Access Denied 폭증 | ASYNC 재디스패치 + AuthorizationFilter 기본 검사 |
| 9 | 캐시 미스 폭발 | 종료 동기화로 인한 재연결 burst |

### D. 본질적 lifecycle 충돌

| #   | 무엇                                 | 왜                       |
| --- | ---------------------------------- | ----------------------- |
| 10  | 트랜잭션 lifecycle ≠ HTTP 응답 lifecycle | "요청은 짧다"는 가정            |
| 11  | 동시 만료 → 동시 재연결 burst               | 모든 SSE가 동일 타임아웃이라 같이 끝남 |