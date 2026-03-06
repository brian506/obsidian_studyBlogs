#트러블슈팅 

## 문제 상황

기존 시스템은 동시성 환경에서 데이터 정합성과 응답 속도 확보를 위해 다음 방식을 사용했다.
- Redis 분산락 + 비관적 락을 통해 데이터 정합성 보장
- Kafka 배치 처리로 락을 한번만 걸고, 100개의 데이터를 모아 한 번에 DB 에 저장
이로써 데이터 무결성을 확보했지만, 여러가지 한계가 있었다.

## 기존 구조의 한계점

1. 예약 - 결제 트랜잭션 문제
	- 결제 실패 시 예약 정보가 남는 문제를 막기 위해 예약 ~ 결제 과정을 하나의 트랜잭션으로 묶음
	- 이로 인해 트랜잭션 범위가 커지고 복잡도 증가
2. Kafka 의 비효율적 사용
	- Kafka 는 원래 실시간 스트리밍 처리에 강점을 가지지만 배치 형태로 사용하여 이점을 잃음
3. 락으로 인한 성능 저하
	- 락을 사용하는 동안 스레드와 DB 커넥션 풀을 모두 점유하기 때문에 대규모 트래픽 시 응답 속도 저하 발생

## 개선 방향

![[../../images/db.png]]

- 락을 제거하고 DB 자체의 원자적 연산으로 동시성 제어를 수행한다.
- Outbox 패턴을 도입하여
	- 트랜잭션과 이벤트 발행 불일치 문제 방지
	- 결제 성공 이벤트를 mysql 과 CDC 연결로 변경사항을 감지하여 안전하게 Kafka 로 전달
- Kafka 분산, 병렬 처리로 신속하게 DB 에 저장
이를 통해 데이터 정합성과 성능을 모두 개선해보도록 한다.
- DB 작업을 하는 스레드와 CDC로 카프카 메시지를 전달하는 스레드는 독립적으로 실행되기 때문에, 로깅 시스템이 연결적으로(traceId) 보여지지 않는다. 
	- 이를 MDC 로 orderId 로 한명의 사용자가 갖는 시작부터 끝까지의 로깅 모니터링을 위해서 사용한다.

## 코드 구현

### 1. 결제 전 재고 확보 (DB의 원자적 선점)

```java
// TicketService ... 

// 원자적 재고 감소 - 상태 PENDING
public TicketReadyResponse purchaseTicket(TicketRequest ticketRequest){  
    TicketType ticketType = OptionalUtil.getOrElseThrow(ticketTypeRepository.findById(ticketRequest.getTicketTypeId()),"존재하지 않는 티켓입니다.");  
    Member member = OptionalUtil.getOrElseThrow(memberRepository.findById(ticketRequest.getMemberId()),"존재하지 않는 샤용자입니다.");  
    String orderId = new ULID().nextULID();  
    // 재고 차감  
    int updatedRows = ticketTypeRepository.decreaseTicketAtomically(ticketRequest.getTicketTypeId());  
  
    if (updatedRows == 0){  
        throw new ExceededTicketQuantityException(ticketType.getName(),ticketType.getPrice());  
    }  
    // 상태 PENDING 지정  
    Ticket ticket = Ticket.createTicket(ticketType,member);  
    ticketRepository.save(ticket);  
    return TicketReadyResponse.toDto(ticket,orderId);  
}

// TicketRepository ...
@Modifying(clearAutomatically = true)  
@Query("UPDATE TicketType t SET t.issuedQuantity = t.issuedQuantity + 1 "+  
        "WHERE t.id = :id AND t.maxQuantity > t.issuedQuantity")  
int decreaseTicketAtomically(@Param("id") Long id);

// Ticket ...
public static Ticket createTicket(TicketType ticketType,Member member){  
    return Ticket.builder()  
            .member(member)  
            .ticketType(ticketType)  
            .price(ticketType.getPrice())  
            .ticketStatus(TicketStatus.PENDING)  
            .expiredAt(LocalDateTime.now().plusMinutes(10))  
            .build();  
  
}
```

- 사용자가 티켓을 선택하고 결제 버튼을 눌렀을 때 호출
- 티켓 유효 시간을 10분으로 지정
- 먼저 티켓의 상태를 `PENDING` 으로 초기 설정하고, `updatedRows` 가 0 이 되면 재고 마감 (동시성 제어)

[[../../CS 공부/데이터베이스/트랜잭션/DB 차원에서의 동시성 제어]] - 쿼리 동작 방식 참고

### 2. 결제(PG) 호출 후 성공 -> PAID 상태 변경 ->  Outbox Entity에 저장

[[🆘 PG 호출과 트랜잭션 제어]] - 참고

1. PG 사 결제 성공
2. `PENDING`-> `PAID` 로 상태 변경 (expiredAt 이 지났거나 이미 처리된 요청은 실패 응답 - 환불)
3. 아웃박스 엔티티에 저장 - PG 와 트랜잭션 분리

### 3. Outbox Entity 에 INSERT 되면 메시지 발행 (백그라운드 CDC)

[[Debezium CDC]] - 참고

**이후 Kafka 병렬 처리**

```java
// 티켓 최종 저장  
@Transactional  
public void issueTicket(PaymentMessage message){  
    Ticket ticket = OptionalUtil.getOrElseThrow(ticketRepository.findById(message.getTicketId()),"존재하지 않는 티켓입니다.");  
  
    if(ticket.getTicketStatus() == TicketStatus.CONFIRMED){  
        return;  
    }  
    if(ticket.getTicketStatus() != TicketStatus.PAID){  
        throw new IllegalStateException("결제되지 않은 티켓입니다.");  
    }  
    Payment payment = message.toEntity(message);  
    paymentRepository.save(payment);  
    // 최종 승인된 티켓  
    ticket.setTicketStatus(TicketStatus.CONFIRMED);  
}
```

1. `PAID` 인 ticket -> `CONFIRMED` 로 `UPDATE` 
2. ticket,payment 최종 저장


 

## 부하 테스트 결과

 ![[../../images/test.png]]

기존에 Kafka Consumer Batch 를 이용해서 나온 응답 속도 3.92s
개선 결과 272ms![](../../images/스크린샷%202026-01-15%2018.35.38.png)

## 대기열에서 나온 사용자 수를 정한 기준

TPS : 초당 처리 횟수
노트북 스펙 : 맥북 M3 Air

대기열에 있는 사용자 수를 얼마나 받을 지 TPS 를 측정하여 정하고자 한다.
부하 테스트 결과, DB row lock 으로 인해 최대 처리량이 114TPS 에서 포화되었다.

이 이상 유입되면 대기 시간이 1.7초 이상으로 급증하여 초당 100명씩만 대기열 서버에서 입장시키도록 했다.
초당 100명이 요청했을 때 


## 커넥션 풀 고갈 문제

동시 요청자 수를 1000명으로 늘렸을 때 아래와 같이 커넥션 풀 고갈 문제가 발생했다.

`HikariPool-1 - Connection is not available, request timed out after 3000ms (total=50, active=50, idle=0, waiting=58)
`CannotCreateTransactionException: Could not open JPA EntityManager for transaction`

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)  
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)  
public void savePaymentToOutbox(OutboxEvent event){  
    try {  
        String jsonPayload = objectMapper.writeValueAsString(event.message());  
        OutboxMessage message = OutboxMessage.toEntityFromTicket(jsonPayload);  
        log.info("[아웃박스 엔티티]  아웃박스 엔티티 저장 {}",event.message().getOrderId());  
        outboxRepository.save(message);  
    } catch (JsonProcessingException e) {  
        // 직렬화 실패는 심각한 시스템 오류이므로 RuntimeException 처리  
        throw new RuntimeException("Kafka payload 직렬화에 실패했습니다.", e);  
    }  
}
```

**문제 상황**
- PG 결제 이후에 `OutBox`엔티티에 저장은 이벤트 핸들러로 결제 정보 저장 로직이 커밋된 이후에 해당 이벤트 서비스가 동작하도록 구현했다.
	- 이렇게 하면 메인 로직 트랜잭션에 관여하지 않고 별도로 저장되기 때문에 더 빠른 로직 수행이 가능할 거라고 생각했다.
	- 그리고 기존의 트랜잭션이 커밋된 이후니까 커넥션 풀을 반납하는 것으로 생각했다.

[트랜잭션 생명주기](../../CS%20공부/데이터베이스/트랜잭션/트랜잭션%20생명주기.md)의 문서를 참고해서 원인이 무엇인지 파악해보자.

**원인**
기존의 트랜잭션을 점유하고 있는 상태로 새로운 트랜잭션을 생성한다.
기존의 트랜잭션은 3단계인 Cleanup 에 도달하지 못해 커넥션을 반납하지 못하고 있다.
그리고 새로운 트랜잭션은 새로운 커넥션을 할당 받아서 로직을 처리한다.

이렇게 되면 하나의 스레드가 결국 2개의 커넥션을 점유하고 있는 상태가 된다.
트랜잭션의 커넥션을 반납하려면 COMMIT -> 동기화 -> cleanup 단계를 거쳐서 완료해야 하는데,
지금 상황에서는 `afterCommit()` 으로 인해 기존 트랜잭션이 동기화 과정에서 cleanup 단계로 넘어가지 못한 상황이 된다.

그래서 동시에 요청하는 수가 많아질 때 커넥션 풀을 2개씩 잡고 있는 메서드 때문에 커넥션 풀 고갈 문제가 발생하였다.

**해결 방법**
어차피 지금 상황에서 카프카 발행은 CDC 에서 아웃박스 엔티티의 변경 감지를 통해서 이벤트를 소비하고 있다.

처음에 아웃박스 엔티티 저장에 `afterCommit()` 을 한 이유는 무조건 티켓 정보의 데이터 반영이 된 후에 저장이 되어야 정합성을 지킬 수 있다고 판단하였다.
하지만 그냥 아웃박스 엔티티를 저장하는 로직도 하나의 트랜잭션에 포함해서 전체가 실패하면 전체가 실패하는 로직으로 생각해도 데이터 정합성을 맞출 수 있다.

결론적으로 `afterCommit()`으로 인한 불필요한 커넥션 할당을 더 해주는 것이 아닌, 한 트랜잭션 내에서 메서드 실행을 하게 되면 데이터 정합성도 맞출 수 있고, 커넥션 풀 고갈 문제도 해결할 수 있다.

## Kafka 컨슈머 에러는 어떻게 관리하는가?

[Kafka 에러 핸들링](../../인프라%20컴포넌트/kafka/Kafka%20에러%20핸들링.md)- 참고

만약 컨슈머 내에 있는 비즈니스 로직이 실패하면 어떻게 처리해야할까?
컨슈머로 메시지를 전달하기 전에 PG 에서 결제가 완료된 건은 CDC 가 아웃박스 엔티티에 `PAID` 상태로 관리하고 있다.

결제 정보는 아주 중요한 데이터이다.
만약 외부 결제사에서 결제는 했지만, 서버에 결제 완료된 데이터가 없으면 데이터 정합성뿐만 아니라 사용자에 대한 신뢰도 또한 떨어지게 된다.
그래서 결제 완료된 데이터는 무조건적으로 DB 에 남아있어야 한다.

만약 아웃박스 엔티티에 저장하지 않았을 경우에는 유실된 메시지에 한해서 에러 핸들링을 하여 DLT 로 보내졌을 때 DB에도 같이 저장하는 로직이 필요할 것이다.

하지만 지금 상황에서는 아웃박스 테이블에 결제 건에 대한 정보가 있으므로, 데이터 유실에 대해서는 에러 로그로 알림 받고 관리자가 수동으로 처리하는 방식으로 구현했다.

이렇게 DLT로 보내진 메시지는 이전에 재시도 처리를 3번 하도록 설정했다.
특정 예외가 존재하지 않는 이상 재시도 했을 때 성공할 가능성이 높기 때문에 별도로 DB 에 저장하지 않고 로그로 관리하는 형식으로 구현했다.

**결론**
컨슈머 에러 메시지 발생 -> 재시도 3번 처리 -> DLT 토픽으로 전송 -> 로그 알림

## 카프카 로그는 어떻게 관리하는가?




