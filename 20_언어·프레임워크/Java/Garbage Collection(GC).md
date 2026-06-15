## 개념

가비지 컬렉션 (Garbage Collection)은 자바의 메모리 관리 방법 중 하나로 JVM의 Heap 영역에서 동적으로 할당했던 메모리 중 **필요 없게 된 메모리 객체를 모아 주기적으로 제거**하는 프로세스이다. 

Java 에서는 가비지 컬렉션이 자동으로 메모리 관리를 해주므로 메모리 관리, 메모리 누수 문제에 대해서 관리를 하지 않아도 된다.
한번 쓰이고 버려지는 객체들을 주기적으로 비워줌으로써 한정된 메모리를 효율적으로 사용할 수 있게 해준다.

하지만 가비지 컬렉션을 사용할 때의 **단점**이 존재한다.
- 자동으로 처리해준다 해도 메모리가 언제 해제되는지 정확하게 알 수 없어 제어하기 힘들다
- GC 가 동작할 때는 **다른 동작을 멈추기** 때문에 오버헤드 발생 
	- 이를 `Stop-The-World`라고 한다.
		- GC 관련 스레드를 제외한 모든 스레드가 멈추는 것



## 가비지 컬렉션 대상

제거되는 대상 기준
`Reachable`: 객체가 참조되고 있는 상태
`Unreachable` : 객체가 참조되고 있지 않은 상태 (**GC의 대상**)

-> `Unreachable` 한 객체의 상태를 가비지 컬렉션이 제거한다.

![](../../images/스크린샷%202026-01-19%2021.14.47.png)

JVM 메모리에서는 객체들은 실질적으로 **Heap 영역에서 생성**되고, **Stack 과 Method 영역**에서 Heap 영역에 있는 **객체의 주소만 참조**하는 형식으로 구성된다.

만약 여기서 Heap 영역 객체들이 메서드가 끝나서 Heap 영역 객체의 메모리 주소를 가지고 있는 참조 변수(Stack or Method Area)가 삭제된다면, 위의 빨간색 객체와 같이 `Unreachable`한 객체들이 발생한다.

이러한 객체들을 가비지 컬렉션이 주기적으로 제거해준다.

## 가비지 컬렉션 제거 방식

### Mark And Sweep

가바지 컬렉션이 될 대상을 먼저 식별(Mark) 하고,  식별(Mark) 되지 않은 객체를 제거(Sweep)한다.

**Root Space** : Heap 영역을 참조하는 method area, static 변수, stack, native method stack

1. **Mark 과정** : 먼저 **Root Space** 로부터 그래프 순회를 통해 연결된 객체들을 찾아내서 **어떤 객체를 참조하고 있는지** 찾아서 Mark 한다. -> 살아있는 객체 색출
2. **Sweep 과정** : 참조하고 있지 않은 객체 (`Unreachable`)를 Heap 에서 제거한다.
3. **Compact 과정** : Sweep 후에 분산된 객체들을 Heap 의 시작 주소로 모아 메모리가 할당된 부분과 그렇지 않은 부분으로 압축한다.

이 과정으로 Root 로부터 연결이 끊긴 순환 참조되는 객체들을 모두 지울 수 있다.

## Heap 메모리의 구조

Heap 영역에서 **객체가 처음 생성될 때 2가지 전제 조건**을 가지고 생성된다.

- 대부분의 객체는 금방 접근 불가능한 상태가 된다.
- 오래된 객체에서 새로운 객체로의 참조는 아주 적게 존재한다.

-> 즉 객체는 대부분 **일회성**이며, 메모리에 오랫동안 남아있는 경우는 매우 드물다.
![](../../images/스크린샷%202026-01-19%2021.38.50.png)

### [Young Generation]

- 새롭게 생성되는 객체가 할당되는 영역
- 대부분의 객체가 금방 `Unreachable` 되기 때문에, 많은 객체가 생성되었다가 금방 사라진다.
- Minor GC 
- 작은 공간에서 객체를 찾기 때문에 빠르게 제거한다.

#### 1. Eden
- new 를 통해 새로 생성된 객체가 위치
- Eden 영역에 **신규객체가 가득 차면 Minor GC 발생**
	- Minor GC는 Young Generation 영역만 수행
	- YG 영역이 가득 찼을 때만 Minor GC 수행
- 정기적인 쓰레기 수집 후(Mark And Sweep) 살아남은 객체들은 survival 영역으로 보냄
	- survival 0 이 다 쌓이고 survival 1 으로 가는 것이 아니라 Minor GC 가 발생할 때마다 순차적으로 각각 보내게 됨
#### 2.  Survivor 0 / Survivor 1
- 최소 1번 이상의 GC 를 살아남은 객체가 존재하는 영역
- 0, 1 둘 중 하나는 꼭 비어있어야 함 (두 영역 중 무조건 하나만 차있음)
- 살아남은 객체들은 **age**(살아남은 횟수)값이 1씩 증가
- 이 구간도 Minor GC 대상

### [Old Generation]

**Promotion** : Survivor -> Old Generation 으로 이동하는 과정

- Young 영역에서 `Reachable` 상태를 유지하여 살아남은 객체가 복사되는 영역
	- Minor GC 과정 중에 제거되지 않아서 **age 임계값이 한계에 도달해서** 이동된 객체들이 있다.
- Young 영역보다 크게 할당되며, 영역의 크기가 큰 만큼 가비지는 적게 발생
- Major GC or Full Gc
- **객체들이 계속 Promotion 되어 Old 영역의 메모리가 부족해지면 발생
- 큰 공간에서 제거하기 떄문에 제거하는데 걸리는 속도가 느림
	- `Stop-The-World`가 여기서 발생!

#### GC 가 중요한 이유
Old Generation 에서 발생하는 GC에서 시간이 많이 소요되기 때문에 멈추는 동안 사용자의 요청이 큐에 쌓이게 된다.
그리고 GC가 끝난 이후 사용자의 요청이 무더기로 들어오기 떄문에 과부하에 의해 장애가 발생할 수 있다.
따라서 원할한 서비스 운영을 위해서는 GC를 잘 조정해야 한다.


> Old > Young 크기가 더 큰 이유?
: 수명이 짧은 객체들은 큰 공간을 필요로 하지 않으며, 큰 객체들은 주로 Old 영역에 할당되어서


![](../../images/스크린샷%202026-01-19%2021.42.56.png)



## GC Trigger — 언제 작동하는가

GC 종류마다 작동 시점이 다르다. 크게 두 모델로 나뉜다.

| 모델 | trigger 방식 | GC 종류 |
|---|---|---|
| **포화형 (reactive)** | "공간 다 차면 그때 멈춰서 청소" | Serial, Parallel |
| **임계값형 (proactive)** | "사용률 X% 넘으면 미리 백그라운드로 청소" | G1GC, ZGC, Shenandoah |

직관적으로 "기준값 넘으면 동작"이라고 생각하기 쉬운데, **그건 G1GC/ZGC 같은 현대 GC에 맞는 표현**이고 Serial/Parallel은 더 단순한 포화형이다.

### Minor GC — 거의 모든 GC가 동일

**Trigger: Eden 영역이 가득 찼을 때.**

임계값(%) 개념이 없다. **빈 공간이 0이 되는 순간** 트리거된다. 그래서 할당 속도가 빠를수록 Minor GC가 자주 돈다.

### Major / Full GC — GC 종류별로 갈린다

#### Serial / Parallel GC (포화형)
![](../../images/스크린샷%202026-06-07%2012.41.49.png)
다음 조건 중 하나 발생 시 Full GC (STW로 힙 전체 청소):

| Trigger              | 발생 상황                                            |
| -------------------- | ------------------------------------------------ |
| **Old Gen 포화**       | Young → Old 승격하려는데 Old 공간 부족 (Promotion Failure) |
| **Metaspace 포화**     | 클래스 메타데이터 영역 가득 → `CollectForMetadataAllocation` |
| **`System.gc()` 호출** | 코드에서 명시적 호출 (대부분 비권장)                            |

→ **"여유 임계값" 개념 없음**. 무조건 가득 찰 때까지 기다렸다가 한 번에 청소. 그래서 STW가 길다.

#### G1GC (임계값형)

```
Young GC:        Eden region 가득 차면 → STW로 Young region 회수
                 ↓
Concurrent Mark: 힙 사용률 ≥ 45% (-XX:InitiatingHeapOccupancyPercent=45)
                 → 백그라운드로 Old region 마킹 시작 (앱과 동시 실행)
                 ↓
Mixed GC:        마킹 끝나고 회수 가치 높은 region 선별
                 → STW로 Young + 일부 Old region 함께 수집
                 ↓
Full GC:         Young/Mixed가 실패했을 때만 (매우 드묾)
```

`MaxGCPauseMillis=200`는 G1에게 "한 번에 200ms 안에 끝낼 만큼만 region 선별해라"고 지시. 임계값(IHOP 45%)에 도달하면 미리미리 백그라운드로 일을 시작해두기 때문에 STW가 짧게 끝난다.

#### ZGC / Shenandoah

- allocation rate(초당 할당량)를 보고 GC를 **예측적으로** 시작
- 거의 모든 작업이 concurrent → STW < 10ms
- 임계값보단 "압력 곡선 예측"에 가까움

---

## Garbage Collection 알고리즘들

### [Serial GC]

- 서버의 **CPU 코어가 1개일 때** 사용하기 위해 개발된 가장 단순한 GC
- GC를 처리하는 스레드가 1개여서 `Stop-The-World` 시간이 가장 김
- Minor GC 에는 **Mark-Sweep**, Major GC 에는 **Mark-Sweep-Compact**를 사용
- 보통 잘 안사용함
- **단, 컨테이너 환경에서 CPU/메모리가 작으면 JVM이 자동으로 Serial을 선택**한다.
	- CPU : 2개 미만
	- 할당 메모리 : 1792MB(약 1.75GB)미만
		- 컨텍스트 스위칭 비용, 관리용 메타데이터의 메모리 점유 떄문

### [Parallel GC]

- **Java 8**의 디폴트 GC
- Serial GC 와 기본적인 알고리즘은 같지만, Young 영역의 **Minor GC를 멀티스레드**로 수행
- Old 영역은 **싱글 스레드**
- Serial GC에 비해 `Stop-The-World` 시간 감소
- GC 스레드는 기본적으로 **CPU 개수만큼** 할당
- 처리량(Throughput) 최적화 GC. **응답 지연 민감한 서버에는 부적합**.

실행 명령어 : `java -XX:+UseParallelGC -jar Application.java`

### [Parallel Old GC]

- Parallel GC를 개선한 버전
- Young 영역 뿐만 아니라, **Old 영역도 멀티스레드**로 수행
- **Mark-Summary-Compact** 방식 이용

실행 명령어 : `java -XX:+UseParallelOldGC -jar Application.java`

### [CMS GC (Concurrent Mark Sweep)]

- 어플리케이션의 스레드와 GC 스레드가 동시에 실행되어 `Stop-The-World` 시간을 줄이기 위해 고안된 GC
- **GC 과정이 매우 복잡하고** 여러 단계로 수행되기 때문에 다른 GC 대비 **CPU 사용량이 높음**
- Java 9 부터 deprecated 됐고, Java 14 에서는 사용이 중지됨

### [G1 GC (Garbage First)]

![](../../images/스크린샷%202026-01-20%2000.28.40.png)

- **Java 9+** 버전의 디폴트 GC (Spring Boot 4 + Java 21 환경의 사실상 표준)
- Heap 영역을 Young, Old 가 아닌 **Region** 이라는 영역(각 1~32MB)으로 분할하여 역할을 고정이 아닌 **동적으로 부여**
	- 칸의 역할이 계속 바뀜
- Garbage로 가득한 영역을 빠르게 회수하여 빈 공간을 확보하므로, 결국 **GC 빈도가 줄어드는 효과**를 얻음
	- 메모리가 가득 차있는 영역(Region)을 인식해서 **메모리가 많이 차있는 영역을 우선적**으로 GC한다
		- 전체를 한번에 청소하는게 아닌, 필요한 칸만 골라서 청소
- 이전의 GC들과 달리, 객체가 순차적으로 이동하는 것이 아니라 영역(Region) 별 객체가 재할당된다.

#### 동작 순서
1. **Young GC** : 앱이 돌아가면서 Eden 칸들이 차면, STW을 걸고 Eden 칸의 살아있는 객체를 Survivor 칸으로 옮기거나 Old 칸으로 이동
2. **Concurrent Marking** : 힙 베모리가 어느정도 (45%정도) 차오르면, 앱을 멈추지 않고 비동기로 Old 칸들의 살아있는 객체에 마킹을 하고 쓰레기 비율을 계산
3. **Mixed GC** : 마킹이 끝나면, Young GC를 할 때 쓰레기가 가장 많은 Old 칸들을 같이 청소
4. **Full GC** : Young, Mixed GC 실패 시 

#### G1GC의 핵심 단계

| 단계 | 동작 | STW 여부 |
|---|---|---|
| **Young GC** | Eden region 차면 Young region 회수 | STW (짧음) |
| **Concurrent Marking** | 힙 45% 도달 시 백그라운드로 Old region 마킹 | **non-STW** (앱과 동시) |
| **Mixed GC** | 회수 가치 높은 Young+Old region 선별 수집 | STW (목표시간 내) |
| **Full GC** | Young/Mixed 실패 시 fallback | STW (피해야 함) |

#### 주요 튜닝 옵션

```bash
-XX:+UseG1GC                          # G1GC 사용
-XX:MaxGCPauseMillis=200              # STW 목표 시간 (기본 200ms)
-XX:InitiatingHeapOccupancyPercent=45 # Concurrent Marking 시작 임계값 (기본 45%)
-XX:G1HeapRegionSize=8m               # region 크기 (자동 계산이 보통 더 좋음)
-XX:ParallelGCThreads=N               # STW 단계 GC 스레드 수
-XX:ConcGCThreads=N                   # Concurrent 단계 GC 스레드 수
```

`MaxGCPauseMillis`는 **약속이 아니라 목표**. G1이 "이 시간 안에 끝낼 만큼만 region을 선별"하지만 힙 압력이 너무 크면 못 지킬 수 있다.


#### [Shenandoah GC]

- Java 12 에 도입, Java 15에서 production-ready
- G1이 가진 pause의 이슈를 해결
- 강력한 Concurrency와 가벼운 GC로직으로 Heap 사이즈에 영향을 받지 않고 일정한 pause 시간이 소요

#### [ZGC GC]
![](../../images/스크린샷%202026-01-20%2000.40.03.png)
- Java 15 출시 (production-ready)
- **대량의 메모리(8MB ~ 16TB)** 를 low-latency로 잘 처리하기 위해 출시된 GC
- **ZPage** 라는 영역을 사용하고, 크기가 2MB 배수로 동적으로 운영
- 힙 크기가 증가하더라도, `Stop-The-World`시간이 **절대 10ms 를 넘지 않음**
- **단점:** concurrent GC 스레드를 추가로 사용하므로 vCPU가 부족한 환경(t3.medium 등)에서는 오히려 앱 스레드와 경합해 역효과. 수 GB 이상의 큰 힙에서 빛난다.

---

## GC 알고리즘 비교 정리

| GC | 모델 | STW 패턴 | 적합 상황 | 비적합 상황 |
|---|---|---|---|---|
| Serial | 포화형 | 단일 스레드, 긴 STW | 1코어 / 매우 작은 힙 / 배치 | 응답 지연 민감 서버 |
| Parallel | 포화형 | 멀티 스레드, 중간 STW | 처리량 우선 배치 | 응답 지연 민감 서버 |
| **G1GC** | 임계값형 | concurrent marking + 짧은 STW | 일반 서버, 응답 지연 중요 | < 수백 MB 힙 |
| ZGC | 예측형 | < 10ms STW | 수 GB~TB 힙, 실시간 | 코어 적은 소형 인스턴스 |
| Shenandoah | 예측형 | 일정한 짧은 STW | ZGC와 유사 | ZGC와 유사 |

**고를 때 핵심 질문:**
1. 응답 지연이 중요한가? → 임계값/예측형 (G1, ZGC, Shenandoah)
2. 힙 크기는? → < 수백 MB면 Parallel/G1, GB 단위면 G1, TB 단위면 ZGC
3. CPU 여유는? → 부족하면 ZGC/Shenandoah의 concurrent 이점이 사라짐

---

## 부가 개념 — Inverted Parallelism

> JMC 자동 분석에서 빨강으로 잡히는 룰. recaring v3에서 발견된 신호.

병렬 GC의 효율 지표.

```
정상:    user + sys CPU 시간 ≈ wall-clock 시간 × GC 스레드 수
역전:    user + sys < wall-clock 시간
         → GC 스레드가 CPU를 못 받고 대기/스케줄아웃됨
```

`user+sys < real`이면 GC 스레드가 도는 시간보다 멈춰있는 시간이 더 길다는 뜻. 병렬 GC의 의미가 사라진다.

**주요 원인:**
- vCPU 부족으로 다른 스레드(앱·JIT·Tomcat 워커)와 경합
- CPU throttle (AWS t-family 버스트 인스턴스의 크레딧 소진)
- 디스크 I/O 대기 (스와핑 등)
- 너무 많은 GC 스레드 (`ParallelGCThreads`)

**대응:**
- `-XX:ParallelGCThreads=1`로 줄여 경합 자체를 회피 (소형 인스턴스 정석)
- 인스턴스 업그레이드 / unlimited 모드
- non-GC 부하 분산

→ **GC 알고리즘을 바꿔서 푸는 문제가 아니라 환경 튜닝 문제**다.

---


[참고 블로그](https://inpa.tistory.com/entry/JAVA-%E2%98%95-%EA%B0%80%EB%B9%84%EC%A7%80-%EC%BB%AC%EB%A0%89%EC%85%98GC-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%F0%9F%92%AF-%EC%B4%9D%EC%A0%95%EB%A6%AC)
# Java GC 정리

## 1. Old Generation 승격 기준

### Tenuring Threshold (나이 기준)

- 객체는 Minor GC에서 살아남을 때마다 **age +1**
- `MaxTenuringThreshold` (기본값: **15**) 초과 시 Old Generation으로 이동
- G1GC는 이 threshold를 동적으로 조정함

### 즉시 승격되는 경우

|케이스                  |설명                                             |
|---------------------|-----------------------------------------------|
|큰 객체                 |Survivor 공간보다 큰 객체 → 바로 Old로                   |
|Survivor 공간 부족       |Survivor 50% 초과 시 조기 승격 (`TargetSurvivorRatio`)|
|Humongous Object (G1)|Region 크기의 50% 초과 → Humongous Region으로 직행      |

-----

## 2. Minor GC가 객체를 다 지우지 못하는 이유

> GC는 **참조가 있는 객체는 절대 수거하지 못함**

```java
// 못 지움 — static 참조가 살아있음 (GC 루트)
static List<Object> cache = new ArrayList<>();
cache.add(new Object());

// 지워짐 — 메서드 끝나면 참조 소멸
void method() {
    Object temp = new Object();
}
```

### Eden → Survivor → Old 흐름

```
Minor GC 발생
    ↓
Eden의 살아있는 객체 → Survivor로 복사
    ↓
Survivor에서 age 넘은 것 → Old로 승격
    ↓
나머지는 Survivor에서 대기 (다음 GC까지 생존)
```

- “꽉 차면 다 지운다” = **참조 없는 것만** 해당
- 살아있는 참조가 있으면 GC는 건드리지 못함

-----

## 3. 클래스 메타데이터 (Metaspace)

- JVM 8+ 기준 **Heap 외부** (Native Memory) 에 저장
- 꽉 차면 → `java.lang.OutOfMemoryError: Metaspace`

|메타데이터 종류            |구체적 예시                               |
|--------------------|-------------------------------------|
|클래스 구조              |필드 목록, 타입 정보                         |
|메서드 바이트코드           |메서드의 바이트코드, 파라미터 시그니처                |
|상수 풀 (Constant Pool)|문자열 리터럴, 클래스명, 메서드명                  |
|vtable / itable     |가상 메서드 디스패치 테이블                      |
|어노테이션 정보            |`@Override`, `@Autowired` 등 런타임 메타데이터|
|제네릭 타입 정보           |`List<String>` 의 타입 파라미터             |
|ClassLoader 참조      |어느 ClassLoader가 로딩했는지                |

-----

## 4. Serial GC vs G1GC

### 요청 처리 가능 여부

|상황                  |Serial GC|G1GC        |
|--------------------|---------|------------|
|GC 실행 중 (Concurrent)|앱 완전 중단 ❌|앱 동시 실행 가능 ✅|
|STW 구간              |앱 중단 ❌   |앱 중단 ❌      |

### Serial GC 동작

```
[요청 처리] → GC 시작 → [완전 중단] → GC 끝 → [요청 처리 재개]
                              ↑
                  이 동안 들어오는 요청은 큐에서 대기
```

### G1GC 동작

```
[요청 처리]  ──────────────────────────────▶
                  ↕ 동시 실행
[Concurrent Mark] ─────────────▶
                                    ↓
                               STW (짧게 잠깐 중단)
                                    ↓
                               [요청 재개]
```

### STW(Stop The World)

- **Serial GC, G1GC 둘 다 STW 구간에선 완전히 멈춤**
- JVM 내 모든 스레드 정지, GC 스레드만 실행
- G1GC는 STW 시간을 **최소화**하는 것이 목표
- G1GC도 Initial Mark, Remark, Evacuation 구간은 STW 발생

-----

## 5. vCPU 1개 vs 2개 — G1GC 동작 차이

> vCPU 수 = GC가 동시에 일할 수 있는 일꾼 수

### vCPU 2개 이상 → G1GC 정상 동작

```
CPU1: 애플리케이션 스레드 실행
CPU2: Concurrent Mark 실행  ← 동시에 돌아감 (STW 최소화)
```

### vCPU 1개 → G1GC 비효율

```
CPU1: [앱 실행] → [STW! GC] → [앱 실행] → [STW! GC]
       ↑ 번갈아가며 실행, 동시성 없음
```

### JVM이 vCPU에 따라 자동 조정하는 것들

|설정                 |vCPU 1       |vCPU 2+                      |
|-------------------|-------------|-----------------------------|
|`ParallelGCThreads`|1            |`ceil(N * 5/8)`              |
|`ConcGCThreads`    |1            |`ceil(ParallelGCThreads / 4)`|
|GC 자동 선택           |Serial GC로 전환|G1GC 유지                      |
|Concurrent Marking |불가           |앱과 동시 실행 가능                  |


> vCPU 1개이면 `-XX:+UseG1GC` 명시해도 병렬성 이점이 거의 없음

### GC 사이클 비교

```
[vCPU 2] G1GC:
  App  ████░░████░░████   (░ = 짧은 STW)
  GC       ████████       (동시 마킹)

[vCPU 1] G1GC:
  App  ████░░░░░░████     (░ = 긴 STW)
  GC       ████████       (앱 멈추고 GC)
```