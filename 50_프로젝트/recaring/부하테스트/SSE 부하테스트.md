
# SSE 연결 테스트

MVP 단계인 100명이라는 적은 숫자이지만 최소 1000명의 사용자가 동시에 SSE 연결이 되었을 때, 시스템 에러를 미리 방지하고자 부하테스트를 진행했다. 
보호 대상자 1명에 보호자 및 관리자가 최대 5명까지 추가될 수 있어서 500명까지의 연결이 원할하게 되야 하는 것은 물론이고, 그 이상의 사용자가 유입되었을 때 현재 시스템 상 몇명까지 가능한지 미리 알아보기 위함이다.

보호 대상자 -> 보호자에게 GPS값을 끊김없이 제공하기 위해 SSE연결을 지속해야한다.
SSE는 HTTP 연결을 유지하며 Sseemitter 객체를 끊지 않기 때문에 Heap 메모리에 계속 남게 된다.

**엔드포인트**
1. 보호자 - 보호 대상자와의 관계 검증(DB)
2. SseEmitter 객체 반환
3. Redis GPS 캐시 읽기 (첫 SSE 연결 시 DB가 아닌 캐싱으로 값을 바로 반환하기 위해)

참고 - [GPS 데이터 흐름 방식 (Push, Map 방식)](../GPS%20데이터%20흐름%20방식%20(Push,%20Map%20방식).md)

## 테스트 환경

| 항목    | 내용                                                |
| ----- | ------------------------------------------------- |
| 부하 도구 | k6 (ramping-vus: 100 → 500 → 1,000 → 1,500 VU)    |
| 서버    | AWS ECS EC2 launch type, t3.medium (2 vCPU / 4GB) |
| 앱     | Spring Boot 4 / Java 21                           |
| 엔드포인트 | `GET /api/v1/location/stream/{wardKey}` (SSE)     |
| 구조    | 보호대상자(300명) X 보호자(5명)                             |

![](../../../images/스크린샷%202026-06-07%2011.15.54.png)

### 구간 별 VU수를 올리는 구간과 멈추는 구간이 존재하는 이유
SSE 연결을 하는 VU(`connect()`)와 GPS Post(`broadcast()`) 가 함께 섞이면 어떠한 동작 때문에 지연시간 및 에러가 발생하는 지 몰라서 따로 구분해서 테스트를 진행했다.


# 1차 트러블슈팅

## 병목 1 - DB 검증으로 인한 커넥션 풀 고갈

### 현상 
보호자 - 보호대상자와의 관계인지를 검증하기 위해 DB에 접근했다.
모든 연결에 DB 검증을 수행하다가 **커넥션 풀 고갈** 현상이 발생했다.

### 해결
이를 해결하기 위해 로컬 캐시인 **Caffeine Cache**를 이용해서 보호자-보호 대상자 관계를 캐싱하여 DB 검증을 하지 않고 해결했다.

```java
@Cacheable(value = "careRelationship", key = "#wardKey + ':' + #caregiverKey + ':CAREGIVER'")  
public boolean hasCaregiverAccess(String wardKey, String caregiverKey) {  
    return careRelationshipRepository.existsByWardKeyAndCaregiverKeyAndCareRole(wardKey, caregiverKey, CareRole.GUARDIAN)  
            || careRelationshipRepository.existsByWardKeyAndCaregiverKeyAndCareRole(wardKey, caregiverKey, CareRole.MANAGER);  
}
```

## 병목 2 - k6 프로세스 ulimit 및 OOM

### 현상
1. k6가 89VU 근방에서 새 연결을 더 이상 열지 못하고 정체됐다.
2. k6 별도의 인스턴스에서 메모리 부족으로 451VU 근방에서 에러가 발생했다.

### 원인

**1번 현상에 대한 원인**
OS는 프로세스 하나가 열 수 있는 FD 수를 ulimit으로 제한한다.
SSE 연결 1개 = TCP 소켓 = FD 1개이므로, k6 FD 한도가 먼저 소진된다.

**2번 현상에 대한 원인**
SSE 테스트는 일반 HTTP와 달리 모든 VU가 동시에 연결을 열고 유지하므로 메모리 소비가 누적된다.
일반 HTTP 테스트는 요청 후 VU가 해제되지만, SSE는 타임아웃으로 연결이 끊기거나, 다른 이유로 연결이 끊기기 전까지 연결을 붙들고 있어 GC를 동작하지 못하여 메모리에 쌓이게 된다.

### 해결

**1번 현상에 대한 해결**
FD 한도의 기본값은 1024이다. FD의 한도를 늘려 더 많은 작업을 수행하도록 해야한다.
App의 FD의 한도 또한 늘려준다.
참고 - [File Descriptor](../../../10_CS/네트워크/File%20Descriptor.md)

**2번 현상에 대한 해결**
k6를 실행하는 머신의 메모리를 충분히 하기 위해 인스턴스를 medium으로 스케일 업 했다.

## 병목 2 - 앱 서버 JVM Heap OOM(1000 ~ 1500 VU)

### 현상
![](../../../images/스크린샷%202026-06-07%2010.59.27.png)

![](../../../images/스크린샷%202026-06-01%2021.54.49.png)

![](../../../images/스크린샷%202026-06-02%2015.54.01.png)
### 원인
1. SSE 연결 증가
	- Tomcat이 연결마다 SocketBufferHandler 할당
	- 연결이 살아있는 동안 **GC 불가** -> 힙에 영구 축적
2. Old Gen 점유율 상승
	- Short-lived 요청 객체가 Old Gen으로 프로모션될 공간 부족
	- Full GC (STW) 빈번히 발생
3. GC 스레싱
	- GC가 CPU 95% 이상 점유
	- HikariPool housekeeper 2분 정지가 이 상태의 지표
	- 사실상 서버 불능 상태
4. OOM
	- 신규 연결의 ByteBuffer 할당 시도 -> 힙 고갈 -> OOM

### 힙 사용률 구간별 동작
> 힙 사용률 ~60% (평균)
  → Old Gen 여유 많음
  → Minor GC 후 survivor 승격이 Old에 쉽게 들어감
  → Full GC 거의 안 돌고 Minor GC만 짧게 수ms
  → 정상 동작
 힙 사용률 ~80% (평균)
  → Old Gen 거의 참
  → Minor GC가 survivor를 Old로 보내려는데 자리 부족
  → "promotion failure" → Full GC 강제 발생
  → Serial Full GC는 단일 스레드 전체 힙 스캔 → STW 수백ms~수초
 힙 사용률 ~95% (평균)
  → Full GC를 돌려도 살아있는 객체가 너무 많아 회수량이 미미
  → 곧바로 다시 95% 도달 → 또 Full GC → 또 95%
  → CPU의 98%가 GC에 박힘 (= 스래싱)
  → "GC Overhead Limit Exceeded" → OOM
  
### 해결
현재 App Task의 메모리 할당은 512MB였다.

App Task의 메모리를 1GB로 올리고, Task 안의 App Container의 메모리도 1GB로 올렸다.
또한, JVM의 힙 메모리 할당도 올려줘야 한다.

```bash
# 컨테이너 메모리의 75%를 힙으로 동적 할당
JAVA_OPTS=-XX:InitialRAMPercentage=25 -XX:MaxRAMPercentage=75
```

`MaxRAMPercentage`는 컨테이너 메모리 변경 시 힙도 자동으로 비례 조정된다.
현재로선 컨테이너 메모리 1GB의 75%인 750MB으로 JVM 힙 메모리를 할당했다.

또한, SSE 타임아웃을 5분으로 줄였다.

#### ECS 롤링 업데이트에서의 메모리 한계

### 현상
현재 ECS 롤링 업데이트의 무중단 배포 방식을 사용하고 있다.
총 4GB의 메모리를 가진 t3.medium의 메모리를 JVM 힙 메모리에 2GB를 할당하려고 했지만, 아래와 같은 에러가 발생했다.

```
1차 시도 (2,048MB 증설):
  service was unable to place a task — insufficient memory available
2차 시도 (CPU 1 vCPU로 증설):
  insufficient CPU units available
```

### 원인
롤링 업데이트는 구버전과 신버전이 **동시에 실행되는** 구간이 존재한다.

App 컨테이너의 메모리를 2GB이상 할당하게 되면, 동시에 컨테이너가 떠 있는 시간이 존재하기 때문에 메모리 한계를 넘게 된다.
또한, 다른 컨테이너의 메모리 할당도 있기 때문에 2GB 이상의 메모리 할당은 만족될 수 없다.

### 해결 방식 비교

| 방식                                      | 비용 추가  | 다운타임   | 사용 가능 메모리     |
| --------------------------------------- | ------ | ------ | ------------- |
| RECREATE (minHealthy=0, maxPercent=100) | 없음     | 20~30초 | 제한 없음         |
| 롤링 1,260MB 상한 유지                        | 없음     | 없음     | 945MB 힙 (한계치) |
| t3.large 업그레이드                          | ~$30/월 | 없음     | 제한 없음         |
효율적으로 메모리를 사용하기 위해서는 RECREATE 방식이 좋지만, 일단 **무중단 배포를 우선**시하므로 1GB를 상한으로 유지하기로 했다.

## 1차 트러블슈팅 순서 요약

```
		   DB 커넥션 풀 고갈
				해결 : 로컬 캐시 적용
[89 VU]    k6 ulimit 소진
               ↓ 해결: ulimit -n 65535
[451 VU]   k6 프로세스 OOM
               ↓ 해결: k6 머신 메모리 확보
[1,000~1,500 VU] 앱 JVM OOM
               ↓ 원인: JAVA_OPTS -Xmx512m 하드코딩
               ↓ 해결: MaxRAMPercentage=75 + 태스크 메모리 증설
[배포 단계] ECS 롤링 업데이트 실패
               ↓ 원인: 구버전 + 신버전 동시 메모리/CPU 초과
               ↓ 해결: RECREATE 방식 or 1,260MB 이내 유지
```


# 2차 트러블슈팅

## 배경

1차 트러블슈팅에서 컨테이너 메모리를 올려 OOM은 해소했고, 1500 SSE 동시 연결 또한 유지된다.

이제 `broadcast()` 로 GPS값이 제대로 보호 대상자에게 잘 전달되는지를 테스트해야한다.

## 테스트 환경

| 항목    | 내용                                        |
| ----- | ----------------------------------------- |
| JVM   | **Serial GC** (기본값, 768MB 힙)              |
| 프로파일러 | JFR + JDK Mission Control 자동 분석           |
| 부하 패턴 | 1,500 SSE 유지 + GPS POST 트리거로 broadcast 측정 |
| 시작 시간 | 17:20                                     |

## 측정 결과

### 전체 요약

| 항목           | 측정 결과                        |
| ------------ | ---------------------------- |
| 1,500 연결 유지  | ✅                            |
| GPS POST p95 | **950ms (정상 200ms 대비 4.7×)** |
| 최대 STW       | **1.4초**                     |
| JMC 자동 분석 점수 | **83 (빨강)**                  |

### 응답 지연 (P50/P95/P99 그래프)

![](../../../images/스크린샷%202026-06-05%2017.33.53.png)

17:25은 6분 정도가 지난 에 1500VU의 요청을 처리하는 시점이다.
`broadcast()`의 SseEmitter `send()` 의 지연시간이 **4.7배**인 1s가 소요되었다.

### VM Operations (JMC)

![](../../../images/스크린샷%202026-06-06%2012.03.48.png)

| Operation                        | Longest        | Total      | Count | 의미                        |     |
| -------------------------------- | -------------- | ---------- | ----- | ------------------------- | --- |
| **GenCollectForAllocation**      | **786.623 ms** | 3.626 s    | 55    | Serial GC Young+Old 전체 스캔 |     |
| **CollectForMetadataAllocation** | **314.210 ms** | 784.174 ms | 4     | Metaspace 부족으로 인한 STW     |     |

####  GenCollectForAllocation
> 객체 할당 실패로 인한 수집

- **동작 상황** : 새 객체를 Eden에 넣으려는데 자리가 부족할 때
	- Eden만 비우면 충분하면 -> **Minor GC**
	- Minor GC 도중 survivor를 Old로 승격해야 하는데 Old도 자리 부족 -> **Full GC 동작**

#### CollectForMetadataAllocation
> Metaspace에 클래스 메타데이터 할당 실패로 인한 수집

- **동작 상황** : Metaspace가 차면 JVM은 안 쓰는 클래스를 언로드하려고 Full GC를 한 번 돌림


![](../../../images/스크린샷%202026-06-06%2011.58.41.png)

17:25에 Full GC가 동작하게 되어  **900ms의 STW**가 실제로 발생한 것으로 보인다.

### Allocation / Memory (Flight Recorder)

![](../../../images/스크린샷%202026-06-05%2017.56.50.png)

**Allocation** : 새로 만들어진 객체의 총량
- 대부분 Minor GC가 처리하는 short-lived 객체라 만들어지자마자 회수된다.

**Memory Usage** : 실시간 힙 점유량
- 그래프 경향이 위아래로 흔들리는 것은 Minor GC가 정상적으로 회수하게 되며 아래로 내려가고, 새로운 객체가(Eden 채워지는 것) 할당됨으로써 다시 위로 올라가는 것이다.
- 현재 **baseline이 우상향하는 이유**
	- GC는 `reachable`한 객체만 회수할 수 있는데,  `unreachable`한 객체가 쌓이게 되어 GC가 회수하지 못하게 되어서 메모리에 남아있는 객체가 계속 늘어나게 되어 baseline이 우상향 하는 것이다.
		- *baseline* : GC를 실행해도 더는 못 줄어드는 바닥선
			- Old Gen에 승격된 long-lived 객체
			- Survivor에 남은 객체
			- Metaspace 등
			
![](../../../images/스크린샷%202026-06-05%2017.56.39.png)

1. **Heap** : 실시간 힙 사용량 
	- 위의 Memory Usage와 같은 경향
2. **Heap Post GC** : baseline 패널
	- GC를 돌려도 못 지워지는 영역을 나타낸다.
		- Old Gen에 SseEmitter, wardKey 등 영구 누적 중이라 지우지 못하는 상황일 가능성
			- SSE는 연결하는 동안 살아있는 객체라서 지우지 못함
			- 비어있는 wardKey를 Map에서 지우는 로직이 없어서 누적되는 것으로 파악
3. **Longest Pause** : 가장 길었던 STW를 나타냄

### JMC 자동 분석 룰 (빨강)

```
[Red 80] Free Physical Memory
  max 사용량 995 MiB / 1 GiB = 97.2%  → 스와핑 임박

[Red 83] GC Pressure
  17:25:01.312 ~ 17:25:02.076 동안 JVM 100% paused = 764ms STW
```

![](../../../images/스크린샷%202026-06-05%2017.56.21.png)![](../../../images/스크린샷%202026-06-05%2017.56.17.png)

## 원인 분석

> [!note] 분포 framing
> 부하 테스트는 **1 ward × 1,500 보호자**로 단일 hotspot을 시뮬레이션했지만, 실제 운영은 **N개 보호대상자 × M명 보호자(예: 300 × 5 = 1,500)** 분포다. 총 보호자 수가 같으면 SSE 연결 객체의 Old Gen 점유는 동일하므로 GC 압력의 골격은 유지되지만, broadcast의 모양과 시간 분포가 달라진다. 아래 원인 분석은 **운영 분포에 대입한 재구성**이다.

### 분석 프레임 — "SSE라서 GC가 안 된 거 아니냐?"

직관적으로는 "SSE는 연결을 계속 붙잡으니 GC가 안 되는 것 아니냐"고 보기 쉽다. **반은 맞고 반은 다르다.** 이 구분이 처방을 두 갈래로 가른다.

| 분류                       | 예시                                                                            | 수명          | GC 작동 여부                              |
| ------------------------ | ----------------------------------------------------------------------------- | ----------- | ------------------------------------- |
| **연결 객체 (long-lived)**   | `SseEmitter`, Tomcat `SocketBufferHandler` (72KB/연결), `wardEmittersMap` 엔트리   | 구독 동안 영구    | **GC 못 함 — 회수해선 안 되는 살아있는 객체**        |
| **broadcast 트래픽 (short-lived)** | Jackson `byte[]`, JSON 임시 버퍼, 응답 객체                                            | 수 ms~수십 ms  | **Young Gen에서 회수 가능**                 |

"SSE라서 GC가 안 된다"는 직관은 **연결 객체에 대해서만 맞는 말**이다. 1차 트러블슈팅(JVM Heap OOM)에서 다룬 영역 — 보호자 1명당 ~350KB가 Old Gen에 영구 고정되어 1,500명이면 ~500MB가 회수 불가능. 1차의 처방이 GC 튜닝이 아니라 **컨테이너 메모리 증설**이었던 이유다.

### GC는 실제로 작동했는가? — 됐다

그래프가 증거다.

- **Memory Usage 톱니파(64 → 320 MiB) ↑↓** — 톱니가 아래로 떨어지는 게 Young GC가 작동한 증거. GC가 안 됐으면 단조 증가만 보였을 것
- **Heap Post-GC 우상향** — 회수 후 baseline이 올라가는 것. long-lived 객체(SSE 연결 + wardKey 맵 잔류)가 쌓이는 정상 신호. **회수 자체는 매번 됨**

진짜 문제는 회수가 안 된 게 아니라 **회수 빈도 × 회수 1회 비용**이다.

```
[운영 분포, 11분 부하 누적]
  N개 ward가 각자 3초마다 GPS POST → 초당 ~100건 broadcast
                                  → 각 broadcast마다 M명에게 직렬화 + send
                                  → 매초 ~500개 byte[] 신규 할당
                                  → Young Gen 평탄하게 반복 포화

[JMC GenCollectForAllocation 결과 — 분포 무관]
  Count    = 55회        ← Serial GC가 11분 동안 55번 발동
  Longest  = 786 ms      ← 1회 발동 시 단일 스레드로 768MB 힙 스캔
  Total    = 3.626 s     ← 55회 누적 STW
```

부하 테스트(1×1500)는 byte[] 1,500개를 **한 번에** 만들었고, 운영(N×M)은 같은 1,500개를 **시간상 평탄하게 누적**시킨다. 총 할당 throughput이 비슷하므로 **Young Gen 압력과 Minor GC 빈도는 거의 동일**하다.

### 한 줄 진단

> "GC가 안 됐다"가 아니라 **"GC가 너무 자주, 너무 비싸게 됐다"** 가 정확한 진단. 처방은 두 갈래로 갈린다 — **빈도**를 줄이는 쪽(pre-serialization)과 **1회 비용**을 줄이는 쪽(G1GC). 연결 객체 누적은 별도 라인(컨테이너 메모리 + computeIfPresent)으로 따로 끊는다.

---

### ① 메모리 천장 충돌 (분포와 무관 · 근본)

운영 분포에서도 그대로 유효한 가장 깊은 원인이다.

- 컨테이너 1 GB의 **97.2%(995 MiB)** 점유
- Heap Post-GC가 시간이 갈수록 우상향 → 회수해도 baseline이 떨어지지 않는다
- 보호자 1,500명에 해당하는 SseEmitter + Tomcat SocketBufferHandler가 Old Gen에 영구 고정
	- 장기 생존 객체는 Old Gen으로

#### 995 MiB의 정체 — 힙만이 아니다

힙 그래프는 ~200MB 수준인데 Free Physical Memory 룰은 995MB로 잡혔다. **둘은 다른 차원의 측정**이다.

```
컨테이너 1 GB 메모리 구성
├── JVM Heap            ~200~300 MB  (힙 그래프가 보여주는 영역)
├── Metaspace           ~128 MB      (클래스 메타데이터)
├── Code Cache          ~50 MB       (JIT 컴파일된 네이티브 코드)
├── Thread Stacks       ~100 MB      (Tomcat 워커 200개 × 512KB)
├── Direct ByteBuffer   ~100 MB      (NIO 네이티브 버퍼)
├── Tomcat NIO 버퍼     ~80 MB       (소켓 IO용)
├── OS 커널 소켓 버퍼   ~50 MB       (1,500 연결의 TCP rcv/snd 큐)
├── JNI · 기타          ~50 MB
└── 합계                ~800~995 MB
```

SSE 연결 1,500개가 **힙 밖**에 만드는 비힙 메모리만 ~250MB 이상이다 (Direct ByteBuffer + OS 소켓 버퍼 + 워커 스택). 힙은 여유로워 보여도 컨테이너 전체는 천장에 닿는다.

→ **회수할 여유 공간 자체가 부족** → GC를 자주 돌려도 효과 없음.

> 1차에서 `MaxRAMPercentage=75`로 힙을 동적 할당한 것은 OOM은 막았지만, 컨테이너 메모리 1GB 자체가 부족함을 드러낸 셈이다.

### ② Serial GC의 구조적 한계 (분포와 무관 · 증폭)

`GenCollectForAllocation 786ms × 55회 = 3.6초`는 **Serial GC의 단일 스레드 전체 힙 스캔**이다. Old Gen 압박이 분포와 무관하게 같으므로 운영 분포에서도 동일한 STW 길이가 재현된다.

```
Serial GC:   [vCPU 1개 점유] → 768MB 힙 단일 스캔 → 786ms STW
             └ 이 동안 모든 앱 스레드 freeze
             └ vCPU 2 환경에서 사실상 1개 코어 통째로 GC 점유
```

t3.medium은 vCPU 2개다. Serial GC가 한 개를 통째로 잡으면 나머지 1개로 Tomcat·SSE·Lettuce가 다 돌아야 한다.

### ③ broadcast 동시 다발 처리 (운영 분포 특유 · 트리거)

부하 테스트의 "단일 broadcast가 byte[] 1,500개 폭발"이 운영에서는 **"100 RPS broadcast가 시간상 평탄 누적"**으로 형태가 바뀐다.

```
[부하 테스트, 1 ward × 1,500명]
  GPS POST 1건 → broadcast 1회 → byte[] 1,500개 단발 폭발
  단일 broadcast 루프 ~500ms 점유

[운영, 300 ward × 5명]
  GPS POST ~100 RPS (300 ward / 3초 간격) → broadcast ~100 RPS
  각 broadcast = 5명 직렬화 + send (수 ms)
  Tomcat 워커 풀이 100 RPS broadcast를 동시에 받아내며 처리
  byte[] 매초 ~500개 평탄 누적
```

총 byte[] 할당 throughput은 비슷하므로 Young Gen 압력은 동등하지만, **단일 broadcast가 P95 baseline에 기여하던 200ms는 사라진다** (정상 구간 P95는 ~30ms 수준으로 떨어진다).

그러나 broadcast 자체의 **chain 구조는 변하지 않는다**:

```
[GPS POST 도착 → Tomcat 워커 점유]
   ↓ @Transactional 시작
   ↓ DB INSERT (gps_log)
   ↓ 트랜잭션 commit
   ↓ @TransactionalEventListener(AFTER_COMMIT) 호출
   │     ↑ "비동기 호출"처럼 보이지만 사실은 같은 워커에서 동기 실행
   ├─ Redis SET latest:wardKey            ← 동기 chain의 일부
   └─ broadcast():
        for (emitter : M명) em.send(gpsLatest)
                            ↑ 매번 Jackson 직렬화 + 소켓 쓰기
   ↓ 리스너 종료
[HTTP 200 OK 반환 — 워커 해제]
```

M이 5로 줄어도 `AFTER_COMMIT`이 **타이밍만 제어하고 스레딩은 제어하지 않는** 구조는 동일하다. 워커는 chain 전체가 끝날 때까지 잡혀있다.

### ④ STW 후 thundering herd (운영 분포에서 새로 드러나는 병목)

부하 테스트(publisher 1 VU)에선 명시적으로 관찰되지 않았지만, 운영 분포에서 P99 폭주의 핵심 메커니즘이다.

```
Full GC STW 786ms 동안:
  ├─ 진행 중이던 ~10개 broadcast 워커가 모두 freeze
  └─ 그 동안 들어온 GPS POST = 100 RPS × 0.786s ≈ 79건이 Tomcat 큐에 적체

STW 해제 직후:
  ├─ 적체된 79건 + 새로 들어오는 100 RPS가 동시에 broadcast 시도
  ├─ 워커 풀 점유율 급증
  └─ 79건 × byte[] 할당 = Young Gen 재포화 가속
        └─ 다음 GC 사이클 임박 → 또 다른 STW 가능성
```

**P99 응답 지연 폭주는 STW 자체(800ms)에 더해, 이 큐 적체가 풀리는 동안의 대기 시간이 누적되어 만들어진다.** 부하 테스트에선 publisher가 1 VU라 적체될 게 1건뿐이라 이 thundering herd 패턴이 명시적으로 드러나지 않았다.

### ⑤ wardKey 맵 엔트리 누수 (운영 시간 적분 · 부하 테스트로 입증 불가)

```java
// 기존 (누수)
wardEmitters.remove(emitter);   // 리스트에서만 제거, 맵 키는 잔류
```

구독자가 모두 끊겨도 `ConcurrentHashMap<wardKey, List<SseEmitter>>`에 **빈 `CopyOnWriteArrayList`가 영구 잔류**한다. ConcurrentHashMap의 키는 명시적으로 제거되지 않는 한 GC 대상이 아니다.

부하 테스트로는 입증할 수 없는 누수다.

- 부하 테스트(1×1500): wardKey 1개, 11분 내내 disconnect 없음 → 누수 발현 0
- 부하 테스트(300×5 가정): wardKey 300개라도 11분 내내 churn 없으면 → 누수 발현 0

운영에서는 시간에 따라 필연적으로 누적된다:

- 보호대상자 이송·서비스 해지·가족 이탈로 한 ward의 가족 M명이 모두 떠나는 사건이 주기적으로 발생
- `wardEmitters.remove(emitter)`는 빈 리스트를 wardKey와 함께 맵에 그대로 둔다
- 운영 N개월 후 → 죽은 wardKey 수천 개가 Old Gen에 영구 점유
- Heap Post-GC baseline 우상향의 **시간 적분 축**

SseEmitter는 동시 접속자 상한이 정해져 있어 baseline에 cap이 있지만, wardKey 맵 누수는 **단조 증가**라 운영 시간이 길어질수록 더 지배적인 누적원이 된다.

> 이 항목은 부하 결과로 입증된 게 아니라 **코드 리뷰로 사전 방어한 잠재 누수**다.

### 인과 체인 (운영 분포)

```
[보호자 1,500명 SSE 연결] + [wardKey 맵 시간 누수]
   └─ Old Gen 누적 → Heap Post-GC 우상향
        └─ 컨테이너 97.2% 점유 (힙 + 비힙) → 회수 여유 없음
             └─ 100 RPS broadcast의 byte[] 할당이 시간상 평탄 누적
                  └─ Young Gen 평탄 압력 → Minor GC 빈번
                       └─ promotion failure → Full GC
                            └─ 단일 스레드 STW 786ms (vCPU 2 환경)
                                 ├─ 진행 중 ~10건 broadcast freeze
                                 └─ 큐에 GPS POST ~79건 적체
                                      └─ STW 해제 후 thundering herd
                                           └─ 워커 풀 압박 + byte[] 재가속
                                                └─ P95/P99 응답 지연 폭주
```

**핵심:** 원인은 서로 독립이지만 **직렬로 곱해진다**. 메모리 천장(①)과 Serial GC STW(②)는 분포와 무관하게 같은 압력을 만들고, 거기에 운영 분포 특유의 ③ 동시 다발 broadcast + ④ thundering herd가 단위 시간당 압력으로 결합한다. ⑤ 맵 누수는 운영 시간이 길어질수록 ①의 baseline을 끌어올린다. 어느 하나만 고쳐도 부분 개선이지만, **다섯 개를 같이 끊어야** P99가 정상화된다.

### 분포에 따른 원인 강도 비교

| 원인 | 부하 테스트(1×1500) | 운영(300×5) |
|---|---|---|
| ① 메모리 천장 충돌 | 동일 (1,500 보호자 동일) | 동일 |
| ② Serial GC STW | 동일 (Old Gen 압박 동일) | 동일 |
| ③ broadcast 처리 모양 | 단일 폭발 (~500ms 루프) | 평탄 누적 (100 RPS) |
| ④ STW 후 적체 | publisher 1 VU라 1건만 적체 | 100 RPS × STW 시간만큼 적체 |
| ⑤ wardKey 맵 누수 | 발현 안 됨 (churn 0) | 운영 시간에 비례해 폭증 |

부하 테스트에서 관찰된 STW 786ms·GC Pressure 83·Free Memory 80은 운영에서도 그대로 발현되며, 추가로 ④와 ⑤가 운영 특유의 압력으로 더해진다.

---

## 처방 선택 근거

원인별로 정확히 끊는 지점을 매칭한다. 운영 분포에서 가장 효과가 큰 세 처방을 채택했다.

| 원인                                      | 처방                              | 끊는 지점                                          |
| --------------------------------------- | ------------------------------- | ---------------------------------------------- |
| ② Serial GC STW                         | **G1GC + MaxGCPauseMillis=200** | STW 길이를 200ms 상한으로 제어                          |
| ③ broadcast 동시 다발 + ④ STW 후 thundering herd | **Virtual Threads (fire-and-forget)** | Tomcat 워커를 broadcast 작업에서 분리, 워커 점유 ~3ms       |
| ⑤ wardKey 맵 누수                          | **computeIfPresent**            | 빈 리스트 시 키 원자적 제거 → Old Gen 시간 적분 누적 차단        |

원인 ①(메모리 천장)은 1차 트러블슈팅의 컨테이너 메모리 증설로 이미 한 번 끊은 라인이고, ②~⑤가 2차의 추가 처방이다.

> Pre-serialization(Jackson N→1)도 후보였지만 운영 분포에서는 효과가 작다. 부하 테스트(1×1500)에선 byte[] 1,499개 절감으로 임팩트가 컸지만, 운영(M=5)에선 broadcast당 4개 절감에 그친다. 시스템 단위 절대량으론 net positive지만 P99 폭주의 근본 메커니즘(② STW · ④ thundering herd · ⑤ 시간 적분)을 끊지 못해 우선순위에서 제외했다.

### 처방 ① G1GC — Parallel/ZGC가 아닌 이유

#### 후보 비교

| GC                | STW 방식                                | SSE 서버 적합성 | 판정    |
| ----------------- | ------------------------------------- | --------- | ----- |
| Serial (현재)       | 단일 스레드, 전체 힙 스캔 → 500ms+              | ❌         | 문제 자체 |
| Parallel          | 다중 스레드지만 여전히 STW                      | △         | 아래    |
| **G1GC**          | STW 목표 설정 + concurrent marking        | ✅         | **선택** |
| ZGC               | <10ms STW, concurrent compaction      | △         | 아래    |
| Shenandoah        | ZGC와 유사 (concurrent compaction)       | △         | 아래    |

#### Parallel GC를 안 쓰는 이유

```
Serial GC:    [스레드 1개로 힙 스캔]   → 500ms STW
Parallel:     [스레드 2개로 힙 스캔]   → ~250ms STW (절반)
G1GC:         [concurrent marking 병행] → 200ms 이하 목표 제어
```

Parallel은 STW를 **짧게 만들 뿐** 모든 앱 스레드가 멈추는 근본 구조는 동일하다. 운영 분포에서 thundering herd가 발생하는 메커니즘은 그대로다 — STW가 250ms로 줄어도 그 동안 들어온 25건이 큐에 적체된다. 게다가 Parallel은 처리량(throughput) 최적화 GC라 응답 지연이 중요한 서버에는 맞지 않는다.

#### ZGC를 안 쓰는 이유 (3가지)

1. **768MB 힙은 ZGC 최적 구간이 아님** — ZGC는 수 GB 이상 대용량 힙에서 concurrent 작업 이점이 커진다. 768MB에서는 G1GC 대비 이점이 거의 없다.
2. **vCPU 2 환경에서 CPU 경합** — ZGC는 앱 스레드와 동시에 도는 GC 백그라운드 스레드를 쓴다. 코어 2개뿐인 t3.medium에서 ZGC 스레드가 Tomcat 워커의 CPU를 잠식한다.
3. **Over-engineering** — 우리 목표는 "500ms STW를 200ms로 줄이자"다. ZGC의 <10ms는 실시간 트레이딩·게임 서버용이지, SSE GPS push에 필요한 정밀도가 아니다.

#### G1GC를 선택한 이유 (4가지)

1. **환경에 정확히 맞는 설계** — vCPU 2개 중 하나로 concurrent marking을 하면서 다른 하나로 앱을 계속 돌린다. Serial이 코어를 독점하던 구조를 해소.
2. **STW 목표 제어 가능** — `-XX:+UseG1GC -XX:MaxGCPauseMillis=200`으로 목표 시간 안에 수집할 region을 선별한다. "힙 전체 한 번에" 스캔하지 않는다. 운영 분포에서 thundering herd 적체량이 STW × RPS에 비례하므로, STW를 200ms 이하로 제어하면 적체 폭이 79건 → 20건으로 줄어들어 폭주가 자연 완화된다.
3. **SSE 서버 특성에 최적** — SSE 연결 객체는 장기간 살아남아 Old Gen으로 승격된다. G1의 **Mixed GC**(Young + 일부 Old region 선별 수집)는 SerialOld의 전체 Old Gen 스캔과 달리 1,500 연결 + 누적 wardKey 맵 상황에 효율적이다.
4. **한 줄 변경, 검증된 표준** — Java 9부터 서버 기본 GC. Spring Boot 4 + Java 21 조합에서 사실상 표준. 리스크 없이 즉시 적용 가능.

```bash
JAVA_OPTS=-XX:+UseG1GC -XX:MaxGCPauseMillis=200
```

> **단, 메모리 여유 확보가 선행돼야 한다.** 97% 메모리 상황에서는 G1의 region 메타데이터 오버헤드 때문에 오히려 GC 빈도가 늘 수 있다. 1차에서 끊은 메모리 천장 라인이 있기에 G1GC의 효과가 온전히 나온다.

### 처방 ② Virtual Threads — Tomcat 워커 해방 + send 병렬화

가상 스레드의 본질은 **"호출한 platform thread가 기다리지 않고 즉시 다음 일을 받을 수 있게 한다"**는 점이다. 단순히 "병렬로 send해서 빨리 끝낸다"가 아니라, **Tomcat 워커를 broadcast 작업으로부터 완전히 분리**하는 게 핵심이다.

#### 적용 코드 — Fire-and-Forget 패턴

```java
// 공유 vthread executor (클래스 필드, 한 번만 생성)
private final ExecutorService broadcastExecutor =
    Executors.newVirtualThreadPerTaskExecutor();

public void broadcast(String wardKey, Gps gpsLatest) {
    broadcastTimer.record(() -> {
        List<SseEmitter> wardEmitters = emitters.getOrDefault(wardKey, new CopyOnWriteArrayList<>());
        if (wardEmitters.isEmpty()) return;

        String json;
        try {
            json = objectMapper.writeValueAsString(gpsLatest);   // ← Jackson 1회 직렬화 (Tomcat 워커가 실행)
        } catch (JsonProcessingException e) {
            log.error("[SSE 이벤트: GPS 직렬화 실패]: wardKey={} | error={}", wardKey, e.getMessage());
            return;
        }

        // submit만 하고 결과를 기다리지 않음 (try-with-resources 사용 안 함)
        for (SseEmitter emitter : wardEmitters) {
            broadcastExecutor.submit(() -> {
                try {
                    emitter.send(SseEmitter.event().name(EVENT_NAME).data(json));
                } catch (IOException | IllegalStateException e) {
                    log.debug("[SSE 이벤트: broadcast 전송 실패]: wardKey={} | error={}", wardKey, e.getMessage());
                    sendFailures.increment();
                    remove(wardKey, emitter);
                }
            });
        }
        // ← 즉시 반환. Tomcat 워커는 다음 GPS POST를 받으러 감.
    });
}
```

두 가지 설계 결정이 핵심이다:

1. **공유 executor를 클래스 필드로 유지** — broadcast마다 새 executor를 만들고 try-with-resources로 닫는 패턴은 close() 시점에 Tomcat 워커를 다시 block시킨다. 한 번만 만들어 두고 평생 재사용하면 생성/파괴 오버헤드도 없고 워커 점유도 안 한다.
2. **submit 후 즉시 반환 (fire-and-forget)** — vthread가 끝나길 기다리지 않는다. AFTER_COMMIT 리스너는 직렬화·submit만 끝나면 곧바로 종료하고, HTTP 200 OK가 device로 돌아간다.

#### 가상 스레드 동작 메커니즘 — 두 레이어

##### 레이어 ① 호출 스레드의 즉시 해방 (Tomcat 워커 ↔ vthread 분리)

```
[GPS POST 도착]
      ↓
Tomcat 워커 (platform thread)
   ├── 맵 lookup (1ms)
   ├── Jackson 직렬화 (1ms)
   ├── submit × 5 (vthread executor에 작업만 던짐, 1ms)
   └── 메서드 반환 → HTTP 200 OK
                    ↑
                    여기까지 Tomcat 워커 점유 = ~3ms

(이후는 백그라운드)
broadcastExecutor의 vthread 5개가 자동으로 carrier에 마운트되어
emitter.send() 실행 (Tomcat 워커와 무관)
```

Tomcat 워커는 **submit이라는 "작업 등록" 행위만** 하고 곧바로 풀로 반환된다. vthread들이 실제 send를 끝내든 말든 워커는 다음 GPS POST를 받으러 갈 수 있다.

##### 레이어 ② vthread 내부의 carrier 재사용

vthread는 실행될 때 **carrier thread**(JVM 내부 ForkJoinPool의 platform thread, 기본 vCPU 수만큼) 위에 마운트되어 돈다. vthread가 블로킹 I/O(소켓 쓰기)에 진입하면 JVM이 자동으로 carrier에서 unmount시킨다. 그러면 carrier는 다른 vthread를 받아서 일을 시작한다.

```
[carrier 1 (platform)]
  vthread A: send() 시작 → 소켓 I/O 대기 → 자동 unmount
  vthread B: send() 시작 → 소켓 I/O 대기 → 자동 unmount
  vthread C: send() 시작 → 소켓 I/O 대기 → 자동 unmount
  ...

[carrier 2 (platform)]
  vthread D: send() 시작 → ...
  
(A의 I/O가 완료되면 빈 carrier에 재마운트되어 send 마무리)
```

t3.medium의 vCPU 2개 환경에서도 carrier 2개로 수십~수백 개 vthread가 돌아간다. 5명 send를 위해 platform thread 5개를 만드는 게 아니라 **carrier 2개를 5개 vthread가 시간차로 공유**한다.

#### 운영 분포에서의 동작 — 시간 분석

운영(300 ward × 5명, 100 RPS broadcast)에서 매 순간 무슨 일이 일어나는지 추적해 보면:

##### 평상시 (STW 없음)

```
초당 100건 GPS POST 도착
  → Tomcat 워커 100개가 1초 안에 broadcast() 호출
  → 각 워커 ~3ms 점유 후 반환
  → 워커 점유 시간 = 100 × 3ms = 300 ms/sec = 평균 0.3 워커
  → 200 워커 풀 중 0.15% 사용 (사실상 부담 없음)

같은 시각 vthread 측:
  → 100 broadcast × 5 vthread = 500 vthread submit/sec
  → 각 vthread send 평균 10ms
  → 평균 동시 활성 vthread = 500 × 10ms / 1초 = 5개 (carrier 2개로 충분)
```

워커 풀과 vthread 양쪽 다 여유롭게 굴러간다.

##### Full GC STW 786ms 직후

```
STW 동안 (786ms):
  → 모든 platform thread (Tomcat·carrier) freeze
  → 새 GPS POST = 100 × 0.786 ≈ 79건이 Tomcat NIO 큐에 적체
  → 진행 중이던 vthread들도 carrier가 멈춰 같이 freeze
  → submit된 broadcast 작업들도 큐에 누적

STW 해제 직후:
  → Tomcat 워커들이 일제히 적체된 79건 처리 시작
  → 각 워커가 broadcast() 호출 → submit × 5 → 3ms 만에 반환
  → 79건 처리에 필요한 워커 사용량 = 79 × 3ms = 237 ms-of-worker-time
  → 4~5 워커만 잠시 점유 → 워커 풀 고갈 없음

vthread 측:
  → 적체된 79 × 5 = 395 vthread가 추가로 활성
  → carrier 2개가 시간차로 다 처리 (vthread 자체는 매우 가벼움)
  → 1~2초 안에 적체 자연 해소
```

**워커 풀 압박이 해소된 것이 핵심**이다. 만약 동기 broadcast(50ms)였다면 79건 × 50ms = 3,950ms = 80 워커가 동시 점유되어야 했다. 4~5 워커만으로 적체가 풀리는 게 fire-and-forget vthread의 직접 효과다.

#### HTTP 200 응답 의미의 변화

| 항목 | 동기 broadcast | Fire-and-forget vthread |
|---|---|---|
| HTTP 200의 의미 | "GPS 저장 + 가족 전원에게 전송 완료" | "GPS 저장 + 가족 전송 시도 등록" |
| device 응답 시간 | ~50ms | **~3ms** |
| broadcast 실패 시 | 호출 컨텍스트에서 잡힘 | metric/log로만 추적 |
| device가 모르는 broadcast 실패 | 없음 | 일부 발생 가능 |

device 입장에서는 GPS POST가 3초마다 반복되므로 한 번의 broadcast 실패가 치명적이지 않다 (다음 사이클에 새 GPS가 broadcast됨). 따라서 fire-and-forget의 응답 의미 변경은 **이 도메인에서 수용 가능**하다.

다만 broadcast 실패율은 metric으로 추적해야 한다. 현재 코드는 `sendFailures.increment()`로 잡고 있어 Grafana로 모니터링 가능.

#### 주의점 — Carrier pinning

| 트레이드오프 | 대응 |
|---|---|
| STW 중에는 vthread carrier도 같이 freeze된다 | freeze 자체는 G1GC가 따로 해결 → 두 처방이 독립적으로 필요한 이유 |
| send 경로에 `synchronized` 블록이 있으면 carrier pinning | vthread가 unmount되지 못하고 carrier를 잡아둠. `SseEmitter#send` 내부 동기화 구간 검증 필요. JFR `jdk.VirtualThreadPinned` 이벤트로 모니터링 |
| Fire-and-forget이라 백프레셔가 없음 | broadcast가 처리 속도보다 빨리 쌓이면 vthread 수가 무한 증가 가능. 평상시 100 RPS에서는 발생 안 하지만, 비정상 burst 대비 vthread queue 길이 모니터링 권장 |
| vthread N개 생성 비용 | ~수백 bytes 수준이라 무시 가능. 1,500개 동시 활성도 부담 없음 |

#### 한 줄 요약

> **Tomcat 워커가 "GPS 받고 vthread에 작업 등록만 하고 즉시 다음 요청 받으러 가는"** 구조다. vthread는 carrier 2개 위에서 시간차로 돌아가며 실제 send를 처리하고, 평균 3ms 워커 점유로 100 RPS broadcast를 받아낸다. STW 후 적체 79건도 워커 풀 4~5개만 잠시 점유하고 풀린다.

### 처방 ③ computeIfPresent — 운영에서 가장 중요해지는 한 줄

부하 테스트에서는 효과가 거의 없지만 **운영에서는 가장 본질적인 처방**이다.

#### Before

```java
// 빈 리스트가 남아도 키는 그대로 잔류
wardEmitters.remove(emitter);
```

`ConcurrentHashMap<wardKey, List<SseEmitter>>`에서 `List#remove()`만 호출하면 리스트의 원소만 빠진다. 리스트가 비어도 wardKey 키는 맵에 그대로 남는다. ConcurrentHashMap의 키는 명시적으로 제거되지 않는 한 GC 대상이 아니다.

#### After

```java
wardEmittersMap.computeIfPresent(wardKey, (k, list) -> {
    list.remove(emitter);
    return list.isEmpty() ? null : list;   // null 반환 시 키 원자적 제거
});
```

`computeIfPresent`는 함수가 `null`을 반환하면 **키 자체를 원자적으로 제거**한다. 같은 wardKey에 대한 다른 스레드의 동시 작업과도 race 없이 안전하다.

#### 부하 테스트로 입증 불가능한 이유

11분 부하 테스트는 본질적으로 churn(disconnect/reconnect)을 시뮬레이션하지 않는다. k6 `timeout: '24m'`이 11분 테스트 전체를 덮으므로 모든 VU가 끝까지 연결을 유지한다 → 빈 wardKey가 생길 일이 없다.

따라서 이 처방은 **부하 테스트 결과로 정당화되지 않고, 코드 분석과 운영 시나리오 추론으로 사전 방어한 것**이다.

#### 운영에서 폭증하는 이유

| 시점 | 1×1500 부하 테스트 | 운영 (300 ward) |
|---|---|---|
| 11분 churn 0건 | 누수 0 | 누수 0 |
| 1일 운영 (가족 이탈 ~5건) | 해당 없음 | wardKey 누수 5건 |
| 1개월 운영 (~150건) | 해당 없음 | wardKey 누수 150건 |
| 6개월 운영 (~900건) | 해당 없음 | wardKey 누수 900건 → Old Gen 영구 점유 누적 |

SseEmitter 객체는 동시 접속자 상한이 정해져 baseline에 cap이 있지만, **wardKey 맵 누수는 단조 증가**다. 운영 시간이 길어질수록 ① 메모리 천장 압박의 가장 큰 누적원이 된다.

> **이 처방의 진짜 가치는 P99 단축이 아니라 "운영 N개월 후에도 메모리 천장에 다시 닿지 않게 하는 것"이다.** 부하 테스트의 즉시 효과는 없지만 운영 안정성에서 가장 본질적인 한 줄.

### 처방 적용 순서와 근거

| 우선순위 | 처방 | 이유 |
|---|---|---|
| 1 | **G1GC** | 한 줄 변경, STW 직접 해소, 즉시 검증 가능. ②와 ④를 동시 완화 |
| 2 | **virtual threads (fire-and-forget)** | Tomcat 워커 점유 시간 50ms → ~3ms로 단축. STW 후 thundering herd 적체를 워커 4~5개로 흡수 |
| 3 | **computeIfPresent** | 코드 한 줄, 부하 테스트 즉시 효과는 작지만 운영 장기 안정성의 핵심 |

> **서사 한 줄:** "Serial GC STW와 동시 다발 broadcast의 워커 풀 압박은 독립 문제지만, STW가 발생하면 워커 풀 적체가 thundering herd로 증폭된다. G1GC로 STW 길이를 200ms 상한으로 제어하고, virtual threads의 fire-and-forget 패턴으로 Tomcat 워커를 broadcast 작업에서 완전히 분리해 워커 풀 압박을 근본 차단했으며, computeIfPresent로 운영 기간에 비례한 메모리 누수를 사전 방어했다."

### 1. 네트워크 지연 시 스레드가 멈추는 이유 (Blocking I/O)

- **끊어짐 vs 느려짐:** 네트워크가 아예 끊어지면 OS가 소켓을 닫고 예외를 던져 스레드가 즉시 해제됩니다. 진짜 문제는 **네트워크가 극도로 느려질 때** 발생합니다.
    
- **TCP 버퍼 포화:** 클라이언트의 네트워크 상태가 나빠 확인 응답(ACK)이 지연되면, 서버 OS의 '전송 버퍼'가 가득 차게 됩니다.
    
- **스레드 블로킹:** 이 상태에서 스레드가 데이터를 마저 보내려고 하면, 버퍼에 빈자리가 날 때까지 타임아웃(보통 수 분)이 발생하기 전까지 그 자리에 얼어붙은 채 대기(Blocking)하게 됩니다.
    

### 2. Tomcat 워커 스레드 vs @Async 스레드 풀

둘은 **완전히 독립된 별개의 스레드 풀**이지만, 전통적인 자바 환경에서는 둘 다 무거운 '플랫폼 스레드(Platform Thread)'를 사용한다는 공통점이 있습니다.

- **Tomcat 워커 스레드:** 외부 HTTP 요청을 가장 먼저 맞이하고 기본 로직을 처리하는 최전방 스레드입니다.
    
- **@Async 스레드 풀:** 워커 스레드가 넘겨준 무거운 백그라운드 작업(SSE 알림 전송 등)을 전담하는 후방 스레드입니다.
    
- **전통적 @Async의 한계:** `@Async`를 쓰면 Tomcat 워커는 빨리 반환되어 새 요청을 받을 수 있습니다. 하지만, 대규모 네트워크 지연이 발생하면 결국 200개 남짓한 `@Async` 스레드 풀 전체가 줄줄이 블로킹되어 마비되는 현상(풀 고갈)은 막을 수 없습니다. 병목의 위치만 뒤로 미룬 셈입니다.