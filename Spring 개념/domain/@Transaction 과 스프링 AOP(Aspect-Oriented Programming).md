
## AOP 란?

AOP 는 관점(Aspect) 지향 프로그래밍이다.
Aspect 는 흩어진 관심사들을 하나로 모듈화 한 것을 의미한다.

객체 지향 프로그래밍에서는 각 클래스는 하나의 책임을 갖은 상태로 구성된다.
Controller - Service - Entity 의 각 계층에서의 로깅, 보안, 트랜잭션 등 공통적으로 사용하는 부가 기능들이 존재한다.
여기서 AOP는 공통적으로 사용되는 기능들(흩어진 관심사)을 AOP 를 통해 하나로 모듈화하여 관리하게 해준다.

AOP 는 기본적으로 실제 객체를 통해 참조하는 것이 아닌 프록시(대행) 객체를 사용하여 접근한다.
그러면 왜 실제 객체가 아닌 프록시 객체를 사용할까? 트랜잭션을 예로 들어 살펴보자.

```java
@Transactional 
public void registerFood(Food food) { foodRepository.save(food);}
```
위와 같은 형태의 비즈니스 로직을 트랜잭션 없이 구현하게 되면?

```java
public void registerFood(Food food) {
	Connection conn = dataSource.getConnection();
	try {
		conn.setAutoCommit(false); // 트랜잭션 시작
		
		foodRepoistory.save(food);
		
		conn.commit(); // 트랜잭션 커밋
	} catch (Exception e) {
		conn.rollBack(); // 트랜잭션 롤백
		throw e;
	} finally {
		conn.close();// 자원 해제
	}
}
```

트랜잭션 어노테이션을 붙이면 실제 객체가 해야할 비즈니스 로직만을 처리하고, 
트랜잭션 어노테이션을 붙이지 않게 되면 트랜잭션의 관한 모든 로직을 짜야한다.

```java
// [프록시 객체] Spring이 메모리 상에서 자동으로 만든 가상의 코드 (의사 코드)
public class FoodService$$SpringCGLIB extends FoodService {
    
    // 진짜 객체를 품고 있음
    private FoodService target; 

    @Override
    public void registerFood(Food food) {
        try {
            // 1. [전] 트랜잭션 시작 (매니저가 무대 세팅)
            transactionManager.begin();

            // 2. [위임] 실제 객체의 메서드 호출 (가수가 노래 부름)
            target.registerFood(food);

            // 3. [후] 성공하면 커밋 (매니저가 뒷정리)
            transactionManager.commit();
        } catch (Exception e) {
            // 4. [예외] 실패하면 롤백 (문제 생기면 없던 일로)
            transactionManager.rollback();
            throw e;
        }
    }
}
```

AOP 는 빈으로 등록된 클래스의 @Transaction 의 유무를 먼저 확인하고 있으면 프록시 객체를 생성한다.
이렇게 가짜 프록시 객체를 만들고 안에 실제 객체를 주입시킴으로써 번거로운 트랜잭션에 관한 작업을 모두 대행해서 해준다.

## 프록시 객체의 내부 호출 문제

문제 예시 : [🆘 PG 호출과 트랜잭션 제어](../../📝%20티켓팅%20프로젝트/🆘%20PG%20호출과%20트랜잭션%20제어.md)

같은 클래스 내에서 트랜잭션이 붙어 있는 메서드가 2개 있다고 가정했을 때,
트랜잭션이 붙어있는 두개의 메서드 중 하나의 메서드 안에서 다른 메서드를 호출했을 때, 각 트랜잭션은 별도로 적용되지 않는다. 왜 그럴까?

```java

@Transactional
public void register() {
	delete();
	repository.save();
}

@Transactional
public void delete() {
	repository.delete();
}
```

여기서 `register()` 를 호출한다고 가정해보자.
먼저 외부(Controller)에서 `register()`를 호출하게 되면 `register()`에 해당하는 프록시 객체가 생성된다.

그래서 `register()`메서드는 트랜잭션이 프록시 객체를 거쳐서 실행되지만,
`delete()`메서드는 `register()`메서드 내에서 **`this.delete()` 으로 호출**이 되어 프록시 객체가 아닌 **실제 객체가 호출**된다.
이렇게 되면 `delete()` 메서드는 프록시 객체를 거치지 않기 때문에 `@Transaction`이 적용되지 않는다.

그래서 각 메서드에 대해서 트랜잭션을 분리하기 위해서는 메서드별로 클래스를 나눠서 호출해야 한다.



