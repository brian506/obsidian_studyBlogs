#database #mysql 

# 개요

PG 사 결제 호출과 DB 상태 UPDATE,INSERT 트랜잭션이 서로 분리되게 설계하려고 했다.
`PaymentService` 클래스 내에서 DB UPDATE, INSERT 를 각각 메소드 분리해서 결제 호출 메서드에 호출하는 형식으로 하였다.
하지만 위와 같은 방식은 `savePayment()` 트랜잭션이 독립적으로 실행되지 않았다.

일단 먼저 왜 PG 결제 호출은 트랜잭션을 분리했을까?

## 트랜잭션 내에 PG 호출이 있으면 안되는 이유

### DB Connection pool 부족 현상
#### 트랜잭션 내에서 PG 결제 호출 흐름
1. **트랜잭션 시작** : 트랜잭션이 시작되면 DB Connection Pool 에서 커넥션 1개를 빌려온다.
2. **PG사 호출** : 트랜잭션 내에서 PG사 API를 호출한다.
3. **대기** : 사용자가 결제를 완료하고 PG사에서 성공응답을 줄때까지 DB Connection Pool 은 계속 점유하고 있게 된다.
4. **커넥션 점유** : DB 커넥션을 반납하지  않고 계속 가지고 있는다.
5. **서버 지연** : 쓰레드와 커넥션 할당이 계속해서 점유하고 있게 되므로 다른 요청을 수행하지 못한다.

### 데이터 정합성 문제
#### 만약 결제는 성공했지만 응답을 주기 직전에 내 서버가 다운되었다고 가정해보자
- 서버는 PG사로부터 성공 응답을 받지 못하였으므로 트랜잭션은 타임아웃으로 롤백된다.
- DB 에는 기록이 남지 못하게 되고, 사용자의 돈은 실제로 나갔을 것이다.

## 트랜잭션 문제 상황


```java
// PG 사 호출
public PaymentMessage confirmPayment(PaymentConfirmRequest paymentConfirmRequest) {  
        PaymentConfirmResponse response = paymentGateway.requestPaymentConfirm(paymentConfirmRequest);  
        PaymentMessage message =  response.fromResponse(response,paymentConfirmRequest);  
        savePayment(paymentConfirmRequest,message);  
        return message;  
    }

@Transactional  
public void savePayment(PaymentConfirmRequest request,PaymentMessage message){  
    Ticket ticket = OptionalUtil.getOrElseThrow(ticketRepository.findById(request.getTicketId()),"존재하지 않는 티켓입니다.");  
    // PAID 로 상태변경  
    int updatedRows = ticketRepository.updateTicketStatus(ticket.getId());  
    if(updatedRows == 0){  
        throw new DataNotFoundException("존재하지 않거나 만료된 예약입니다.");  
    }  
    savePaymentToOutbox(message);  
}

// 아웃박스 엔티티에 저장
private void savePaymentToOutbox(PaymentMessage paymentMessage){  
    try {  
        String jsonPayload = objectMapper.writeValueAsString(paymentMessage);  
        OutboxMessage message = OutboxMessage.toEntityFromTicket(jsonPayload);  
        outboxRepository.save(message);  
    } catch (JsonProcessingException e) {  
        // 직렬화 실패는 심각한 시스템 오류이므로 RuntimeException 처리  
        throw new RuntimeException("Kafka payload 직렬화에 실패했습니다.", e);  
    }  
}
```

- 위와 같이 PG 결제 <-> DB UPDATE,INSERT 트랜잭션을 구분하고 하나의 메서드로 실행하고자 했다.
- `savePayment` 안에 `@Transaction` 을 둠으로써 DB UPDATE,INSERT 메서드가 하나의 트랜잭션 안에서 실행되고 PG사 결제는 구분했다.

### 에러

 1. `TransactionRequiredException: Executing an update/delete query`
	-  `@Modifying` `UPDATE` 쿼리가 트랜잭션 없이 실행하게 되어 에러가 발생하는 내용
2. CDC Race Condition
	- CDC Debezium 이 Outbox 엔티티에 데이터가 INSERT 되어도 데이터를 읽지 못한다고 에러 발생
### 원인

1. 클라이언트가 프록시를 통해서 public 메서드만을 호출했을 때 `@Transaction` 이 개입할 수 있다.
	즉 `@Transaction`은 프록시를 거칠 때만 작동한다.
	그래서 한 메서드 안에서 `@Transaction`이 적용된 메서드를 호출해도 `this.savePayment()`와 같이 프록시를 거쳐서 호출되지 않기 때문에 `@Transaction`이 적용되지 않는다.
	
2. DB UPDATE 트랜잭션이 완전히 발행되기 전에 Kafka 가 이를 읽는 과정이 생겨서 시간차로 인해서 에러가 발생했다. 
	그래서 아직 PAID 상태가 아닌 티켓을 읽으려고 하다가 id = null 에러가 발생한 것이다.
	
### 해결 방법

![[../../images/tran.png]]


```java
// PaymetGatewayService ...
public PaymentMessage confirmPayment(PaymentConfirmRequest paymentConfirmRequest) {  
        // 부하테스트용 ( pg 호출 x )        String orderId = new ULID().nextULID();  
        PaymentConfirmResponse response = new PaymentConfirmResponse(orderId,10000,"test-key", OffsetDateTime.now(),OffsetDateTime.now());  
//        PaymentConfirmResponse response = paymentGateway.requestPaymentConfirm(paymentConfirmRequest);  
        PaymentMessage message =  response.fromResponse(response,paymentConfirmRequest);  
        paymentService.savePayment(paymentConfirmRequest,message);  
        return message;  
    }
    
// PaymentService ...
@Transactional  
public void savePayment(PaymentConfirmRequest request,PaymentMessage message){  
    Ticket ticket = OptionalUtil.getOrElseThrow(ticketRepository.findById(request.getTicketId()),"존재하지 않는 티켓입니다.");  
    // PAID 로 상태변경  
    int updatedRows = ticketRepository.updateTicketStatus(ticket.getId());  
    if(updatedRows == 0){  
        throw new DataNotFoundException("존재하지 않거나 만료된 예약입니다.");  
    }  
    message.setExpiredAt(ticket.getExpiredAt());  
    eventPublisher.publishEvent(new OutboxEvent(message));  
}

// OutboxEventService...
@Transactional(propagation = Propagation.REQUIRES_NEW)  
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)  
public void savePaymentToOutbox(OutboxEvent event){  
    try {  
        String jsonPayload = objectMapper.writeValueAsString(event.message());  
        OutboxMessage message = OutboxMessage.toEntityFromTicket(jsonPayload);  
        outboxRepository.save(message);  
    } catch (JsonProcessingException e) {  
        // 직렬화 실패는 심각한 시스템 오류이므로 RuntimeException 처리  
        throw new RuntimeException("Kafka payload 직렬화에 실패했습니다.", e);  
    }  
}
```
1. 클래스를 분리해서 트랜잭션이 적용된 메서드를 호출해야 독립적으로 트랜잭션이 적용된다.

2. `@TransactionalEventListener(phase = TransactionalHpase.AFTER_COMMIT)`
	- 이벤트를 발행하는 첫번째 트랜잭션이 `COMMIT` 된 이후에 `savePaymentOutbox()` 실행
	`@Transactional(propagation = Propagation.REQUIRED_NEW)
	- `savePaymentOutbox()` 를 새로운 트랜잭션으로 실행
