#database #mysql 

# 개요

JPA 의 1차 캐싱인 영속성 컨텍스트를 마주치지 않고 DB 단에서 직접 `UPDATE` 쿼리를 사용하여 동시성 제어를 할 수 있다.

먼저 서비스단에서 `Setter` 를 이용해서 영속성 컨텍스트 지나쳐서 객체를 어떻게 변경시키게 되는지 알아보자.

## Setter 작동 방식

1. DB 에서 `SELECT 
2. `set()` 로 메모리에 있는 값 변경
3. 트랜잭션이 커밋될 때 1차 캐시의 원본과 변경된 객체를 비교(*ditrtychecking*)
4. 변경이 감지되면 JPA 가 `UPDATE` 쿼리를 생성해서 수정

*한계점*

상황 : 두 명의 사용자가 재고가 1개 남았을 때 재고 감소 요청
- **스레드 A:** 1. 재고 1개 SELECT (통과)
- **스레드 B:** 1. 재고 1개 SELECT (통과)
- **스레드 A:** 2. 메모리에서 재고 0으로 변경
- **스레드 B:** 2. 메모리에서 재고 0으로 변경
- **스레드 A:** 4. UPDATE stock = 0 (성공)
- **스레드** **B:** 4. UPDATE stock = 0 (성공)
이렇게 되면 동시에 요청할 때 한 트랜잭션 내에서 동시 차감이 발생한다.
이를 해결하기 위해 락을 사용할 수 있지만, DB 에 직접 `UPDATE`쿼리를 날려서 동시성 제어를 할 수 있는 방법을 공부해보자.

## DB의 원자적 동시성 제어

`Repository` 단에서 1차 캐싱을 건너뛰고 `@Query`와 `@Modifying` 을 사용하여 구현한다.

*@Query*
- JPA 에서 `@Query` 는 기본적으로 조회(SELECT) 전용
- 근데 `UPDATE` 쿼리를 수행하기 위해 `@Modifying` 을 같이 사용

*@Modifying*
- JPA 가 `@Query`를 변경성 쿼리(UPDATE, DELETE) 로 인식하여 `executeUpdate()` 실행
- 하지만 DB 데이터가 변경되어도, 1차 캐시에 이미 로드된 엔티티가 있으면 그 객체의 재고는 여전히 이전 값을 가지고 있게 됨
- 이를 해결하기 위해 `@Modifying(clearAutomatically = true)` 조건을 붙임
- `clearAutomatically = true` 조건
	- 쿼리 실행 후 영속성 컨텍스트를 자동으로 비움
	- 이로써 남아있는 1차 캐시는 고려하지 않아도 됨
	- 이후 영속성 컨텍스트가 DB 를 조회했을 때 최신반영된 데이터를 읽을 수 있게됨

```java
@Modifying(clearAutomatically = true)  
@Query("UPDATE TicketType t SET t.issuedQuantity = t.issuedQuantity + 1 "+  
        "WHERE t.id = :id AND t.maxQuantity > t.issuedQuantity")  
int decreaseTicketAtomically(@Param("id") Long id);
```

`UPDATE` 쿼리 : 발행된 수량 = 발행된 수량 + 1 
`WHERE`   조건 : 최대 수량 > 발행된 수량 까지만
- 즉 재고가 남아있는 경우에만 감소시키는 조건부 원자 업데이트

쿼리는 DB 내부에서 단일 `UPDATE` 문으로 실행되기 때문에 동시에 여러 스레드가 접근하더라도 DB 가 자동으로 행-락을 잡고 순서대로 처리한다.