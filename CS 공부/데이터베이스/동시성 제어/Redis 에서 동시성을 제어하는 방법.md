#redis 


락 없이 동시성을 제어하는 방법에 대해서 알아볼 것이다.

Redis 는 일단 **Single Thread**이다.
Redis 는 **Single Thread** 임에도 불구하고 동시성 이슈가 발생하는데 왜 그런지 알아보자.

## Redis 에서 동시성 문제가 발생하는 이유?

Redis 는 싱글 스레드로 동작하여 **명령어 한개씩은 원자적**으로 수행되지만, 여러 작업을 번갈아서 처리하기 때문에 동시성 문제가 발생할 수 있다.

```
A : GET stock (결과 : 1)
B : GET stock (결과 : 1)
A : stock - 1 -> SET stock 0
B : stock - 1 -> SET stock 0
```

한명의 사용자의 명령어(GET,SET)을 수행하고, 다음 명령어를 곧바로 시행하기 때문에 동시성 문제가 발생할 수 있다. 
	- 요청 오는 사용자 기준이 아닌 명령어 기준임

즉, Redis 는 동시성은 있지만, 병렬성은 없다.
- 동시성 : 하나의 CPU 또는 스레드가 여러 작업을 번갈아가며 진행해, 겉보기엔 동시에 처리되는 것처럼 보이는 상태
- 병렬성 : 여러 CPU/코어가 실제로 동시에 작업을 진행하는 상태

이를 해결하기 위해 **Redis 분산락**. **Redis Lua Script**, **Redis Transcation** 로 동시성을 제어할 수 있다.

## Redis 분산 락

 여러 서버, 프로세스가 같은 리소스에 접근해야할 때 순서를 강제하기 위해 **Lock**을 두어 정합성을 지킨다.
 
 __Pub/Sub__ 방식 : 채널을 하나 만들고 락을 점유 중인 스레드가 대기중인 스레드에게 해제를 알려주면 알림을 받은 대기중인 스레드가 락 획득을 시도하는 방식이다.
 - Lettuce 는 **spin Lock** 을 사용하여 시스템에 부하를 줄 수 있다.
 - 락 획득 재시도를 기본으로 제공한다.

	**동작 순서** 
		1. 구독(subscribe) : 락을 획득하기 위해 대기하게 되는데, 이때 락 채널을 구독하고 대기한다.
		2. 알림(publish) : 락을 가지고 있던 클라이언트가 작업을 마치고 락을 해제하면, 채널에 락이 해제되었음을 알린다.
		3. 메시지를 받은 대기 클라이언트들은 다시 락 획득을 시도한다.

**spin lock** : 락을 획득하려는 스레드가 락을 사용할 수 있는 지 반복적으로 확인하면서 락 획득을 시도하는 방식
	- 락을 빠르게 획득할수는 있지만 시스템에 부하를 줄 수 있다.

### 분산락이 필요한 경우

- 데이터가 한 곳에 있는 게 아니라, 여러 서버에 저장되는 분산 DB를 사용하는 경우
- Redis 하나에 락 상태를 두면, 모든 분산된 DB가 같은 락 상태를 바라봄ㅇ 아


## Redis Lua Script

Redis 서버 내부에 **Lua Script**  실행기가 내장되어있는데, 해당 스크립트 전체를 **하나의 거대한 명령어**어로 간주하여 단일 스레드로 해당 스크립트를 실행하는 방식이다.

**특징**
- **Atomic Execution** : 스크립트가 실행되는 동안 다른 어떤 클라이언트의 명령어도 실행되지 않는다.
- **로직 처리 가능** : Redis Transaction과 다르게 스크립트 내에 `GET, IF, SET`등의 조건문을 활용한 로직을 수행할 수 있다.
- **성능 굿** : 스크립트 실행 1회로 네트워크 왕복이 1번으로 줄어든다.

Ex) 티켓 재고 수량 정합성을 지키는 상황

```java
String stockKey = KEY_PREFIX + ticketTypeId;  
Long result = redisTemplate.execute(decreaseStockScript, List.of(stockKey), "1");  
  
if (result == -1L) {  
    throw new IllegalStateException("시스템 오류: 재고 정보가 없습니다.");  
}  
if (result == 0L) {  
    soldOutTicketIds.add(ticketTypeId);  
    return false;  
}
```

```lua
// lua script

local stock = redis.call('get', KEYS[1])  
  
if not stock then  
    return -1  
end  
  
if tonumber(stock) < tonumber(ARGV[1]) then  
    return 0  
end  
  
redis.call('decrby', KEYS[1], ARGV[1])  
return 1
```

- `KEYS[1]`: Redis 키 이름들을 받는다.(현재는 1번째 키)
	- 자바에서 보낸 `List.of(stockKey)` 와 같음
- `ARGV[1]` : 키가 아닌 일반 값들을 받는다.
	- 자바에서 보낸 "1"과 같음
- 레디스는 기본적으로 문자열로 저장하고 반환하기 때문에 `toNumber` 로 변환 후에 비교해야한다.

![](../../../images/스크린샷%202026-02-07%2020.26.21.png)