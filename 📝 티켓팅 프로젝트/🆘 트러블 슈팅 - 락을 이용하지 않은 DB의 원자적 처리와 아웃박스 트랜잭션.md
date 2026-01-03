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

![[../images/db.png]]

- 락을 제거하고 DB 자체의 원자적 연산으로 동시성 제어를 수행한다.
- Outbox 패턴을 도입하여
	- 결제는 성공했지만 DB 반영에 실패하는 경우 방지
	- 결제 성공 이벤트를 mysql 과 CDC 연결로 변경사항을 감지하여 안전하게 Kafka 로 전달
- Kafka 분산, 병렬 처리로 신속하게 DB 에 저장
이를 통해 데이터 정합성과 성능을 모두 개선해보도록 한다.

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

[[DB 차원에서의 동시성 제어]] - 쿼리 동작 방식 참고

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

 ![[../images/test.png]]

기존에 Kafka Consumer Batch 를 이용해서 나온 응답 속도 3.92s
개선 결과 272ms