컨슈머가 메시지를 처리하는 도중에 에러가 발생하면 어떻게 처리할까?

```java
public void consumePayment(String message, Acknowledgment ack) {  
    try {  		
        ticketService.issueTicket(payment);  
        ack.acknowledge(); // 작업 성공시 브로커에게 완료 메시지 전송  
    } catch (KafkaConsumerException e) {  
        log.error("[kafka 처리] 티켓 최종 발급 처리 실패. payload {}", message);  
    } catch (Exception e){  
        ack.acknowledge();;  
    }
```

컨슈머 안에 DB 에 저장하는 로직이 포함되어 있다.
이 과정에서 DB 에 에러가 생기거나 카프카 브로커, 네트워크에 에러가 생겨서 해당 메시지를 처리를 못하게 되면 해당 처리 못한 메시지를 어떻게 관리해야할까?
지금 로직에서는 성공했을 때 `ack.acknowledge()` 로 정상적으로 처리가 성공되면 브로커에게 완료 메시지를 보내고 있다.
그리고 `KafkaCounsumerException` 이 발생한 경우에는 로그로도 해당 에러 메시지에 대한 처리를 하고 있다.

하지만 이렇게만 하면 메시지가 유실된다.
처리하지 못한 메시지를 재시도하거나, 실패 로그 알림을 보내거나, 데드 레터 큐로 전송하여 안정성을 확보해야 한다.

메시지 유실을 방지하기 위해 실패한 메시지에 대한 처리 방법을 한번 알아보자.

## 에러 핸들러 종류

`DefaultErrorHandler`: 재시도 및 복구 로직에 주로 사용

- **BackOff 정책** : 에러 발생 시 재시도할지, 일정 간격을 두고 시도할지, 시도할 때마다 시간을 늘릴지 등에 대한 세부적인 정책을 정할 수 있음
- **복구** : 지정된 횟수만큼 재시도해도 실패할 경우, 메시지를 **DLT(Dead Letter Topic)** 으로 보내거나 로그만 남기고 다음 메시지로 넘어가는 작업 등 주로 후처리 담당

**DLT** : DLT 에 들어온 메시지는 누군가 처리하기 전까지 카프카에 보관됨

`ContainerStoppingErrorHandler` : 에러가 발생하면 특정 리스너 컨테이너를 중단

- 인프라 등 재시도해도 의미가 없는 로직일 때 리스너 자체를 중단시켜버림

`KafkaListenerErrorHandler` : 특정 메서드에만 적용하는 에러 핸들러

- 특정 리스너 메서드에서만 따로 에러 로직을 관리해야할 때 사용

## Consumer Config 설정

```java
@Bean  
public DefaultErrorHandler defaultErrorHandler(KafkaTemplate<String, String> kafkaTemplate) {  
	// 토픽명 .DLT 로 전송
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate,  
            (r, e) -> new TopicPartition(r.topic() + ".DLT", r.partition()));  
	// 재시도 3초 간격으로 3번 시도
    FixedBackOff backOff = new FixedBackOff(3000L, 3L);  
	
    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);  
  
	handler.addNotRetryableExceptions(DeserializationException.class, MethodArgumentNotValidException.class);  
    handler.setRetryListeners((record, ex, deliveryAttempt) -> {  
        if (deliveryAttempt > 3) {  
            log.error("[Kafka 최종 실패 알림] 메시지 처리에 최종 실패하여 DLT로 전송합니다. " +  
                    "Payload: {}, Reason: {}", record.value(), ex.getMessage());  
        }  
    });  
    return handler;  
}
```

- 재시도 3번 시도해도 안되는 경우 토픽 .DLT 로 전송한다.
- 두 종류의 예외 상황은 재시도하지 않고 바로 .DLT 로 전송한다.
	- `DeserializationException`: 메시지 데이터 자체가 깨져서 객체로 변환 안되는 경우
	- `MethodArgumentNotValidException` : 유효성 검사에서 에러난 경우
- 로그로 DLT 로 전송된 메시지를 식별한다.