## 개념

가비지 컬렉션 (Garbage Collection)은 자바의 메모리 관리 방법 중 하나로 JVM의 Heap 영역에서 동적으로 할당했던 메모리 중 **필요 없게 된 메모리 객체를 모아 주기적으로 제거**하는 프로세스이다. 

Java 에서는 가비지 컬렉션이 자동으로 메모리 관리를 해주므로 메모리 관리, 메모리 누수 문제에 대해서 관리를 하지 않아도 된다.
한번 쓰이고 버려지는 객체들을 주기적으로 비워줌으로써 한정된 메모리를 효율적으로 사용할 수 있게 해준다.

하지만 가비지 컬렉션을 사용할 때의 **단점**이 존재한다.
- 자동으로 처리해준다 해도 메모리가 언제 해제되는지 정확하게 알 수 없어 제어하기 힘들다
- GC 가 동작할 때는 **다른 동작을 멈추기** 때문에 오베헤드 발생 
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

1. **Mark 과정** : 먼저 **Root Space** 로부터 그래프 순회를 통해 연결된 객체들을 찾아내서 어떤 객체를 참조하고 있는 지 찾아서 Mark 한다.
2. **Sweep 과정** : 참조하고 있지 않은 객체 (`Unreachable`)를 Heap 에서 제거한다.
3. **Compact 과정** : Sweep 후에 분산된 객체들을 Heap 의 시작 주소로 모아 메모리가 할당된 부분과 그렇지 않은 부분으로 압축한다.

이 과정으로 Root 로부터 연결이 끊긴 순환 참조되는 객체들을 모두 지울 수 있다.

## Heap 메모리의 구조

Heap 영역에서 **객체가 처음 생성될 때 2가지 전제 조건**을 가지고 생성된다.

- 대부분의 객체는 금방 접근 불가능한 상태가 된다.
- 오래된 객체에서 새로운 객체로의 참조는 아주 적게 존재한다.

-> 즉 객체는 대부분 일회성이며, 메모리에 오랫동안 남아있는 경우는 매우 드물다.
![](../../images/스크린샷%202026-01-19%2021.38.50.png)

### [Young Generation]

- 새롭게 생성되는 객체가 할당되는 영역
- 대부분의 객체가 금방 `Unreachable` 되기 때문에, 많은 객체가 생성되었다가 금방 사라진다.
- Minor GC 
- 작은 공간에서 객체를 찾기 때문에 빠르게 제거한다.

#### 1. Eden
- new 를 통해 새로 생성된 객체가 위치
- Eden 영역에 **신규객체가 가득 차면 Minor GC 발생**
- 정기적인 쓰레기 수집 후(Mark And Sweep) 살아남은 객체들은 survival 영역으로 보냄
	- survival 0 이 다 쌓이고 survival 1 으로 가는 것이 아니라 Minor GC 가 발생할 때마다 순차적으로 각각 보내게 됨
#### 2.  Survivor 0 / Survivor 1
- 최소 1번 이상의 GC 를 살아남은 객체가 존재하는 영역
- 0, 1 둘 중 하나는 꼭 비어있어야 함
- 살아남은 객체들은 **age**(살아남은 횟수)값이 1씩 증가
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

## Gargbage Collection 알고리즘들

### [Serial GC]

- 서버의 **CPU 코어가 1개일 때** 사용하기 위해 개발된 가장 단순한 GC
- GC를 처리하는 스레드가 1개여서 `Stop-The-World` 시간이 가장 김
- Minor GC 에는 **Mark-Sweep**, Major GC 에는 **Mark-Sweep-Compact**를 사용
- 보통 잘 안사용함

### [Parallel GC]

- **Java 8**의 디폴트 GC
- Serial GC 와 기본적인 알고리즘은 같지만, Young 영역의 **Minor GC를 멀티스레드**로 수행
- Old 영역은 **싱글 스레드**
- Serial GC에 비해 `Stop-The-World` 시간 감소
- GC 스레드는 기본적으로 **CPU 개수만큼** 할당

실행 명령어 : `java -XX:+UseParallelGC -jar Application.java 

### [Parallel Old GC]

- Parallel GC를 개선한 버전
- Young 영역 뿐만 아니라, **Old 영역돋 멀티스레드**로 수행
- **Mark-Summary-Compact** 방식 이용

실행 명령어 : `java -XX:+UseParallelOldGC -jar Application.java

### [CMS GC (Concurrent Mark Sweep)]

- 어플리케이션의 스레드와 GC 스레드가 동시에 실행되어 `Stop-The-World`
  시간을 줄이기 위해 고안된 GC
- **GC 과정이 매우 복잡하고** 여러 단계로 수행되기 때문에 다른 GC 대비 **CPU 사용량이 높음**
- Java 9 부터 deprecated 됐고, Java 14 에서는 사용이 중지됨

### [G1 GC (Garbage First)]

![](../../images/스크린샷%202026-01-20%2000.28.40.png)

- **Java 9+** 버전의 디폴트 GC
- **4GB 이상의 힙 메모리**, `Stop-The-World`시간이 0.5초 정도 필요한 상황에 사용 (Heap 이 너무 작으면X)
- Heap 영역을 Young, Old 가 아닌 **Region** 이라는 영역으로 분할하여 역할을 고정이 아닌 **동적으로 부여**
- Garbag로 가득한 영역을 빠르게 회수하여 빈 공간을 확보하므로, 결국 **GC 빈도가 줄어드는 효과**를 얻음
	- 메모리가 가득 차있는 영역(Region)을 인식해서 **메모리가 많이 차있는 영역을 우선적**으로 GC한다
- 이전의 GC들과 달리, 객체가 순차적으로 이동하는 것이 아니라 영역(Region) 별 객체가 재할당된다.

#### [Shenandoah GC]

- Java 12 에 출시
- G1이 가진 pause의 이슈를 해결
- 강력한 Concurrency와 가벼운 GC로직으로 Heap 사이즈에 영향을 받지 않고 일정ㅇ한 pause 시간이 소요

#### [ZGC GC]
![](../../images/스크린샷%202026-01-20%2000.40.03.png)
- Java 15 출시
- **대량의 메모리(8MB ~ 16TB)** 를 low-latency로 잘 처리하기 위해 출시된 GC
- **ZPage** 라는 영역을 사용하고, 크기가 2MB 배수로 동적으로 운영
- 힙 크기가 중가하더라도, `Stop-The-World`시간이 **절대 10ms 를 넘지 않음**

[참고 블로그](https://inpa.tistory.com/entry/JAVA-%E2%98%95-%EA%B0%80%EB%B9%84%EC%A7%80-%EC%BB%AC%EB%A0%89%EC%85%98GC-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%F0%9F%92%AF-%EC%B4%9D%EC%A0%95%EB%A6%AC)
