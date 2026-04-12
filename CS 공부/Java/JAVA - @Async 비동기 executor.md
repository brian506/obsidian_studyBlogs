
## Executor

JAVA 에서 @Async 를 사용하여 비동기를 처리할 수 있다.
Spring Boot 에서는 기본적으로 SimpleAsyncTaskExecutor를 사용한다.


| Excutor 종류                    | 설명                                  | 특징                                                 |
| ----------------------------- | ----------------------------------- | -------------------------------------------------- |
| `SimpleAsyncTaskExecutor`     | 각 작업을 새로운 스레드에서 실행하며, 스레드풀을 사용하지 않음 | 많은 작업을 짧은 시간에 실행하면 자원 부족 가능                        |
| `ThreadPoolTaskExecutor`      | 스레드 풀을 사용하여 비동기 작업                  | - 스레드 풀의 크기, 용량, 접두사 이름 등 설정 가능<br>- 과도한 스레스 생성 방지 |
| `ScheduledThreadPoolExecutor` | 주기적 또는 일정 시간 후에 작업 실행               | 주기적 작업이나 지연된 작업을 예약하며 주기적인 작업 스케줄링에 사용             |
| `ForkJoinPool`                | 병렬 처리를 위함                           | 작업을 작은 단위로 분할하여 병렬 처리                              |

![](../../images/스크린샷%202025-12-25%2021.44.19.png)

## `SimpleAsyncTaskExecutor`

- 각 작업을 새로운 스레드에서 실행하며 스레드 풀 사용 X
- 다수의 작업을 실행할 수 있지만, 스레드 수가 많아지면 시스템 리소스를 과도하게 사용하게 되므로 주의

별도의 설정 없이 `@Async`만 사용하면 해당 Executor 로 사용


## `ThreadPoolTaskExecutor`

- 스레드 풀을 사용하여 비동기 작업 실행
- 다수의 작업을 동시에 수행하면서 시스템 리소스 효율적으로 사용

[쓰레드풀(Thread)와 커넥션 풀(Connection pool)](쓰레드풀(Thread)와%20커넥션%20풀(Connection%20pool).md) - 참고

### 주요 메서드

| 메서드                                        | 리턴 타입                               | 설명                                              |
| ------------------------------------------ | ----------------------------------- | ----------------------------------------------- |
| `cancelRemainingTask(Runnable task)`       | `protected void`                    | 실행이 시작되지 않은 남은 작업 취소                            |
| `createQueue(int queueCapacity)`           | `protected BlockingQueue<Runnable>` | BlockingQueue 생성<br>                            |
| `execute(Runnable task)`                   | `void`                              | 주어진 작업 실행                                       |
| `setCorePoolSize(int corePoolSize)`        | `void`                              | 기본 유지 스레드 수 설정                                  |
| `setKeepAliveSeconds(int keepAliveSeconds` | `void`                              | 유지 시간 설정                                        |
| `setMaxPoolSize(int maxPoolSize`           | `void`                              | 최대 스레드 수 설정                                     |
| `setQueueCapacity(int queueCapacity)`      | `void`                              | BlockingQueue 용량 설정                             |
| `submit(Runnable task`                     | `void`                              | 실행을 위해 Runnable 작업을 제출하고 해당 작업을 나타내는 Future를 받음 |

```java
@Configuration  
@EnableAsync  
public class AsyncConfig {  
  
    @Bean(name = "fcmExecutor")  
    public Executor fcmExecutor() {  
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();  
        executor.setCorePoolSize(5);  
        executor.setThreadNamePrefix("Fcm-");  
        executor.initialize();  
        return executor;  
    }  
  
    @Bean(name = "imageExecutor")  
    public Executor imageExecutor() {  
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();  
        executor.setCorePoolSize(5); // 최소한으로 유지할 스레드 수 
        executor.setMaxPoolSize(10); // 스레드 풀이 동시에 사용할 수 있는 최대 스레드 수 
        executor.setQueueCapacity(50);  // 큐에 들어갈 최대 스레드 수(대기 가능 스레드 수)
        executor.setThreadNamePrefix("S3-");  
        executor.initialize();  
        return executor;  
    }  
}
```

- 5개의 요청까지는 스레드 바로 생성 후 처리
- 동시에 10개의 요청까지 스레드 처리 가능
- 11개 부터 스레드 큐에 삽입
- 최대 큐에 들어가는 스레드 총 50개

## `ScheduledThreadPoolExecutor`

- 주기적으로 또는 일정한 지연 후에 작업을 실행할 수 있도록 스레드 풀을 관리
- 스레드 풀을 이용하여 작업이 일정한 간격으로 실행되거나 일정 시간 이후에 실행되도록 구성
- 타이머 작업, 주기적인 데이터 백업, 정기적인 상태 체크 등에 사용


## `ForkJoinPool`

- 큰 작업을 작은 단위로 분할(fork)하고 병렬처리하여 다시 결합(join) 하는데 최적화
- 재귀적인 작업 분할에 적합
- 워크 스틸링 알고리즘 사용하여, 각 스레드가 자신의 작업을 완료하고 다른 스레드의 작업을 훔쳐서 처리하는 방식

### 워크 스틸링(Work - Stealing) 알고리즘

#### 동작 과정

1. **작업 분할** : 메인 스레드가 작업을 `ForkJoinPool`에 제출하면, 여러 개의 작은 작업으로 분할
2. **작업 할당** : 분할된 작업들은 워커 스레드의 작업 큐에 할당되어, 각 워크 스레드는자신의 큐에 있는 작업을 처리
3. **작업 훔치기** : 워커 스레드가 자신의 작업 큐에 더 이상 처리할 작업이 없으면, 다른 워커 스레드의 작업을 훔쳐서 처리
	1. 유휴 상태의 워커 스레드는 다른 워커 스레드의 큐에서 작업을 훔칠 대상을 무작위로 선택
	2. 다른 워커 스레드의 큐에서 가장 오래된 작업을 훔침
	3. 훔쳐온 작업을 자신의 큐에 추가하고 처리
4. **작업 실행** 
5. **작업 완료 및 결합** 
	- 모든 작은 작업이 완료되면 결과를 결합하여 최종 결과 생성